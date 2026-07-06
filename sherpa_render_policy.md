# SHERPA Render Policy

**One line:** Decides per turn how many sketches to show (N) and along which variation axis — producing COMMIT (1), FORK (2), SWATCH (3), or SWIPER (7) mode.

**Destination picker (2026-06-22):** `RenderMode.DESTINATION_PICKER` is an additional, product-free mode emitted by the light destination finder (not the render-policy decision tree). The `SketchFrame` carries the cards in a new optional `destinations: list[dict]` field — `{name, country, iata, tags[], match (int), climate_note (str)}` — with `sketches=[]`, `variation_axis="destination"`. Gated in `cognitive/handler.py` by `if discover_only or not city` (resolves before the committed-city fast-path short-circuit). Streams inside the existing `block.type == "sketch_frame"` envelope (no AppSync schema change).

**Destination Finder v2 — feasibility gate + climate rank + richer card (2026-06-23):** `_discover_destinations_light` (`cognitive/sketch_engine.py`) now turns the picker from a pure taste-match into a feasibility-aware recommender. **DNA SQL is unchanged** — origin point, budget ceiling, pax, and desired climate are per-user/per-turn values, not catalogue columns, so the DNA `destinations` endpoint stays origin/budget-agnostic. Instead **SHERPA over-fetches** a wider pool (`limit = max(top_n*3, _DEST_POOL_N=12)`) from dna-shortlist and **gates + relaxes + climate-re-ranks + slices** in the finder. Build order is load-bearing: **gate (exclude unaffordable) → relax-backfill stretches → climate soft re-rank → slice to top_n → THEN compute match% over the shown set** (recomputing after the slice preserves the score stretch despite the over-fetch). The dna-shortlist `destination_codon_search(k=limit)` comment notes this (no DNA code change; SHERPA owns the stretch via post-slice recompute).

Feasibility model (all per-turn, finder-local): distance (`_haversine_km_flight` over IATA→coords from the **ti-airports DDB cache index**, never OpenFlights airports_map) → flight hours (`dist/_CRUISE_KMH + _OVERHEAD_H`) → round-trip economy fare (`cabin_fare_estimator.estimate_fares` is **one-way USD** → ×2 round-trip → **USD→EUR via `_USD_PER_EUR`≈1.08** → ×`max(adults+children,1)`). Compared like-for-like against an **all-pax EUR** ceiling (`flight_ceiling_abs`, else `budget_abs*_FLIGHT_BUDGET_FRACTION=0.40`, else None): `≤ceiling`→`within`, `≤ceiling*_STRETCH_MAX(1.4)`→`stretch`, else `unaffordable` (excluded). Climate intent parsed locally (`_parse_climate_intent` keyword→°C band, or warm/beach DNA default) — additive `±_CLIMATE_BONUS/_PENALTY` on top of DNA coverage (DNA dominates). Graceful degrade: no origin resolvable → flight gate skipped (taste+weather only); null lat/lon → card kept, no estimate, affordability null (never excluded on missing data); no budget → keep all (+ optional soft long-haul demote for thrifty BUDG DNA). **Climate JSONB is keyed by 3-letter month abbreviation (`_MONTH_ABBR` "jan".."dec"), not full names** — `climate_note` from `month["notes"]`, weather avg from `month["avg_temp_c"]`.

**Four new card fields (additive, all optional, render-guarded with `?? fallback`):** `period` `{check_in, check_out?, nights?}` (trip dates), `est_flight` `{origin_iata, dest_iata, hours, fare_low, fare_high, currency:"EUR", estimate:true}` (round-trip all-pax estimate from origin), `affordability` `"within"|"stretch"` (budget verdict; `unaffordable` never reaches the card), `weather` `{avg_temp_c, rating?, condition?, months?}` (period-averaged from the climate JSONB). `climate_note` remains the backward-compat fallback. `SketchFrame.destinations` stays `list[dict]` (no Pydantic change). UI: `DestinationCardData` + `DestinationCard` in `mywai-hotel-providers/ui/src/components/AgentPlayground.tsx` render the period line (`CalendarDays`), est-flight+fare line always labelled "est." (`Plane`), affordability badge (`BadgeCheck` within / `Wallet` stretch), and the upgraded weather strip (`Thermometer`, avg °C + condition + rating, falling back to `climate_note`) — lucide-react icons only.

## What it does
- `decide_render` returns `{n, mode, variation_axis, variation_values, rationale}` per turn
- Decision tree: lock primitive → committed trip → DNA conviction → drop-to-commit threshold → axis confidence → cold start default
- Variation values define the spread: FORK [−0.35, +0.35], SWATCH [−0.3, 0.0, +0.3], SWIPER 7 values from −0.5 to +0.5
- LinUCB bandit (`SHERPA_BANDIT=on`) can override the mode hint once it has ≥10 reward updates
- `_choose_variation_axis` switches to a less-settled axis if the pattern default is already determined by DNA (e.g. avoiding budget/luxury fork for a known-luxury user)

## Logical flow (decision tree)
1. **Lock primitive:** any block locked by user → COMMIT immediately
2. **Committed trip:** user picked a sketch → single-sketch modification mode (COMMIT)
3. **DNA conviction > 0.7:** high confidence → COMMIT early
4. **DNA conviction > 0.45:** moderate confidence → FORK
5. **Turn ≥ drop_to_commit_turns + 1:** user has refined enough → COMMIT
6. **Axis confidence > 0.7:** strong signal on variation axis → COMMIT
7. **Axis confidence > 0.4:** decent signal → FORK
8. **Cold start:** use `mode_hint` from pattern (swatch/swiper/fork/commit)
9. **Bandit override:** if `SHERPA_BANDIT=on` and ≥10 updates → use LinUCB hint

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/cognitive/render_policy.py` | `decide_render` | Main decision function |
| `lambda/cognitive/render_policy.py` | `_choose_variation_axis` | Picks most uncertain axis |
| `lambda/shared/bandit.py` | `LinUCB` | LinUCB bandit for mode hint |
| `lambda/shared/pattern_catalogue.py` | `Pattern.render_defaults` | Cold-start N, mode_hint, drop_to_commit_turns |

## Critical code
```python
# render_policy.py — full decision tree (simplified)
def decide_render(pattern, taste_field, turn_number, is_cold_start,
                  bandit_mode_hint=None, dna_conviction_score=0.0,
                  has_committed_trip=False, user_locked_blocks=None):

    if user_locked_blocks and any(v for v in user_locked_blocks.values()):
        return _commit(...)                           # lock primitive
    if dna_conviction_score > 0.7 and not is_cold_start:
        return _commit(...)                           # high conviction
    if dna_conviction_score > 0.45 and not is_cold_start:
        return _fork(variation_axis, ...)             # moderate conviction
    if turn_number >= drop_turns + 1 and not is_cold_start:
        return _commit(...)                           # drop-to-commit
    # ... axis confidence checks, then cold-start mode_hint
    return _swatch(variation_axis, ...)               # default: 3-sketch swatch
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/shared/test_experiments.py` | Bandit mode hint + A/B experiment wiring |

## Gotchas
- `SHERPA_BANDIT=on` must be set in env AND bandit must have ≥10 reward updates for LinUCB to override. Default (shadow mode) collects data but uses rule-based decision
- `_choose_variation_axis` silently switches the axis if the default is settled — rationale string captures why; check it in logs if users are getting unexpected variation axes
- `drop_to_commit_turns` comes from the pattern's `render_defaults` — different patterns have different turn thresholds before converging to COMMIT
