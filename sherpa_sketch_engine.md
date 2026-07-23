# SHERPA Sketch Engine

**One line:** Assembles a complete SketchFrame — from enriched query to DNA shortlist call to hotel blocks with copy, anchors, and lattice validation — ready to stream to the UI.

## What it does
- Enriches the user query (rule map + Haiku translation) and calls the DNA shortlist Lambda via Lambda-to-Lambda invoke
- Hydrates raw hotel candidates with DDB metadata (name, images, amenities, landmarks) from `hotel_canonical_data_txt_media`
- Applies Render Policy to decide N sketches and variation axis, then assembles N distinct SketchCards
- Validates each sketch through the Reality Lattice (dates, visa, capacity, seasonal closures)
- Generates Sonnet copy (headline + paragraph) per sketch via Bedrock
- Publishes each block to AppSync in sequence (seq must be unique per turn — UI dedupes by seq+type)

## Logical flow
1. `_compose_hotel_filters(intent, taste_field)` → builds stars, amenities, geo, amenity_groups filters
2. `_get_seasonality_context(city, check_in, check_out)` → demand_surge, price_pressure, crowd_level signals
3. `_dna_shortlist_candidates(city, user_id, ..., extra_filters, seasonality_context)` → POST to DNA API Lambda, 8s timeout, 60s LRU cache
4. `_search_and_rank_hotels(...)` picks N candidates at different variation axis values
5. Per candidate: `_hotel_from_candidate` builds HotelBlock; `_lookup_anchors` fetches nearby POIs
6. Lattice.evaluate runs; auto-fixes applied, confirms surfaced to user
7. Sonnet generates headline + copy; `appsync.publish` streams the block

## Key functions
| Function | Location | Purpose |
|----------|----------|---------|
| `_dna_shortlist_candidates` | `sketch_engine.py:783` | Lambda invoke to DNA API with 60s LRU cache |
| `_compose_hotel_filters` | `sketch_engine.py:537` | Builds filter dict from intent + taste field |
| `_search_and_rank_hotels` | `sketch_engine.py:1018` | Main assembly loop: N sketches × axis values |
| `_hotel_from_candidate` | `sketch_engine.py:1344` | Converts raw candidate to HotelBlock |
| `_lookup_anchors` | `sketch_engine.py:1430` | Fetches nearby POIs from DDB anchor library |
| `_required_pill_groups` | `sketch_engine.py:140` | Groups amenity display names into pill groups |
| `_dna_conviction_score` | `sketch_engine.py:195` | 0–1 score driving render mode convergence |

## Critical code
```python
# sketch_engine.py — hard vs soft amenity routing in _compose_hotel_filters
# _HARD_PILL_NAMES = frozenset({"pets", "accessible", "parking"})
# Hard amenities → required_amenities (Tier 1 CTE, never relaxed)
# Soft amenities → amenities + amenity_groups (progressive relaxation)
if explicit_amenities:
    hard_codes, soft_dns = [], []
    for dn in explicit_amenities:
        pill = next((p for p, dns in _PILL_GROUPS.items() if dn in dns), None)
        if pill in _HARD_PILL_NAMES:
            hard_codes.append(_am_code(dn))
        else:
            soft_dns.append(dn)
    if hard_codes:
        filters["required_amenities"] = hard_codes
    if soft_dns:
        filters["amenities"] = soft_dns
        _pill_groups = _required_pill_groups(soft_dns, city=city.lower())
        if _pill_groups:
            filters["amenity_groups"] = [sorted(g) for g in _pill_groups]

# sketch_engine.py — amenity badge pills on hotel cards
# requested_codes includes BOTH soft (display names → am_codes) and hard (already am_codes)
requested_codes: set[str] = (
    {_am_code(dn) for dn in (filters.get("amenities") or [])}
    | set(filters.get("required_amenities") or [])
)
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/golden/test_pipeline_golden.py` | End-to-end pipeline golden cases |
| `tests/smoke/test_prima_turns.py` | Prima sketch turn smoke test |

## Hotel query chromosome filter (2026-05-22)

Hotels only emit codons from certain chromosomes. User DNA from outside that set (DEST, ACTV, BUDG, etc.) cannot match any hotel codon — it only dilutes the query vector. `_HOTEL_QUERY_CHROMS` (module-level frozenset in `sketch_engine.py`) defines the allowed set:

```
STAY, MOOD, FOOD, WELL, CULT, SOC, PURP
```

BUDG is intentionally excluded: the `luxe_threshold` axis emits `BUDG#LUXR`/`BUDG#PREM`, but hotels never emit BUDG codons. STAY.TIER#LUXR is the hotel-side tier signal — it enters `_routing_codons` via the STAY injection block (line ~1237), which reads `taste_field.codon_affinities` directly and injects any STAY codon with affinity ≥ 1.5.

**Critical:** The injection must use `taste_field.codon_affinities` (the dict attribute), NOT `taste_field.get("codon_affinities")`. `TasteField.get()` is an axis-lookup method that returns a float, silently injecting nothing.

`_MIN_SKETCH_MATCH = 35` (module-level constant) — candidates below this score are excluded before the variation picker. Falls back to the full pool when no candidate passes so the engine never returns None.

## Hotel deck pick — triptych rank mapping (2026-07-03)
The 3-card SWATCH deck decouples the hotel pick from the `variation_value` sign so the **hero is always the #1 DNA match** and the cards never duplicate a hotel.

- `_search_and_rank_hotels(..., hotel_rank_position: int | None)`. When a position is supplied (SWATCH only), the pick is `pool[_triptych_rank_depths(n)[position]]`; `variation_value` is then used ONLY for anchor/copy flavor. When `None` (FORK / SWIPER / COMMIT), the legacy `variation_value` index picker runs verbatim — **unchanged**.
- `_TRIPTYCH_BAND = (0, 2, 5)` — hero + narrow top-band ranks (~1/3/6). `_triptych_rank_depths(n)` clamps to the pool, de-dupes, and back-fills lower unused ranks so up to **3 DISTINCT** hotels show whenever the pool has them; returns a monotone list. n=1→[0], n=2→[0,1], n=3→[0,1,2], n=4→[0,2,3], n=5→[0,2,4], n≥6→[0,2,5].
- `run_sketch_engine` assigns each FRESH-search card its position among fresh cards (`_fresh_positions`, mirroring the `_build_sketch` locked/committed branch). Every fresh card in a deck queries the same city/filters → an identical ranked pool, so per-card slots are collision-free. A card whose position exceeds the distinct-plan length returns `None` → the sketch is dropped downstream (render fewer, never duplicate).
- **Defense-in-depth dedupe:** after the cards are composed and before the `hotel=None` filter, `run_sketch_engine` walks the deck and nulls the hotel of any LATER card that resolved to a `hotel_id` already seen (`sketch_hotel_dup_dropped` log). The nulled card then drops via the existing `hotel=None` path. This only bites under a cross-invocation score-tie ordering divergence — the primary dedup backstop is the handler's `_used_hotel_ids` substitution.
- Byte-identical preserved: the LENS `hotel_ids_callback` fires pre-pick on the full id set (pricing coverage unchanged); flight resolve-once; trip_shape hotel filter; anchor ranking.

## Hotel swap — carry the real DNA score (2026-07-03)
Swap alternatives are bounded by the submitted DNA-50 — every one is a scored candidate WITH codons. `swap_block` wave-2 branch now carries the real `match_score` / `codon_contribs` / `dna_boost` into `_hive_to_candidate(...)` from the persisted price record instead of hard-zeroing. Those three fields are persisted alongside each wave-2 alternative in `handler.py` (`_wave2_alts`, looked up from `_candidates_meta`) → `save_lens_alternatives` (schemaless DDB attribute `lens_wave2_alternatives`, no Aurora change). Missing scores fail-safe to 0. The internal off-screen counter is `_tail_offscreen` (the AppSync `provider_status.new_hotels` wire field is retained for UI compat but means "off-screen priced DNA candidate", not "new to DNA").

**Persisted-pool cap (`_SWAP_POOL_CAP = 15`):** before `save_lens_alternatives`, `_wave2_alts` is sorted by `match_score` desc and truncated to the top 15. The swap only ever consumes `count<=5`, so 15 is ample headroom while bounding the DDB write (~15×1.5KB ≈ 22KB, well under the 400KB item limit) and making it independent of the shortlist limit. Full `codon_contribs` are kept on the 15 survivors — the swap MATCH tab renders that codon trace.

**Non-finite guard:** the numeric fields carried into `_wave2_alts` are sanitised in `handler.py` — `match_score` / `dna_boost` via `_finite()` and every float inside `codon_contribs` via `_sanitize_codon_contribs()` (non-finite → 0.0). This stops a fabricated `NaN`/`inf` from making `state._py_to_ddb`'s `Decimal(str(x))` emit `Decimal('NaN')`, which would trip the bare-except in `save_lens_alternatives` and drop the ENTIRE alternatives list for the turn.

## Server-side match display % — `HotelBlock.match_display` (2026-07-06)
The card match % was computed by a 7-anchor piecewise-linear curve that lived ONLY in the playground UI (`AgentPlayground.tsx` `MATCH_CURVE_ANCHORS`/`matchCurve`), reading the pre-rounded int `match_score` (drifts ±1% vs truth). It now lives server-side.
- **`shared/match_display.py::match_display_pct(match_01: float) -> int`** — pure, byte-parity port of the UI curve (same anchors, JS `Math.round`-equivalent `floor(x+0.5)`, clamp [0,100]). Reads FULL-PRECISION `match_01` (no rounding drift). Fail-safe → 0 on NaN/inf/non-numeric.
- **`match_01` threading:** `match_01` (`dna-shortlist/handler.py:1902`, `round(match_01,6)`) survives the dna-shortlist → dna-api → sketch_engine hop verbatim (dna-api forwards the raw shortlist body). `_dna_shortlist_candidates` reads it (`float(r.get("match_01") or match_score/100)`) and threads it onto every candidate dict via `_hive_to_candidate(match_01=...)` and the Aurora-fallback dict.
- **`_hotel_from_candidate`** sets `match_display=match_display_pct(raw["match_01"])`. `HotelBlock.match_display: int = 0` is ADDITIVE — `match_score` stays RAW (ENGINE gate + inspector). The UI renders `match_display`; its own `matchCurve` becomes a no-op fallback.
- **Swap/rebuild paths:** `match_01` is persisted alongside `match_score` in `_wave2_alts` (finite-guarded) and threaded into `_hive_to_candidate` on the swap rebuild so `match_display` isn't reset to 0 on a swap. Substitution/alternatives in `handler.py` rebuild via `_hotel_from_candidate` off `_candidates_meta` (which carries `match_01`) → display correct. `_apply_price` updates in place → `match_display` preserved.
- **INVARIANT (untouched):** scoring/ranking, `match_score`, `match_01`, sort order, and the `_MIN_SKETCH_MATCH=35` gate are all byte-identical — only the additive display field is new.

## Confirmed trip total — `SketchFooter.total_all_in` (2026-07-06; day-plan activities folded in 2026-07-08 B3)
`footer.total_estimated_price` is hotel-only; `TripConfirmationCard` used to compute the all-in total client-side (`flight.total_price + hotel.price_per_night × nights`). Now server-side.
- **`_footer_total_all_in(hotel, flight, day_plan=None) -> Optional[float]`** = hotel `total_price` (price_per_night×nights) + flight `total_price` (selected-offer all-pax total) **+ booked PAID day-plan activities** (`kind=="activity"` AND `price>0`, via `_day_plan_activities_total`). **trip_shape-aware by presence** — the composition already nulls the omitted component (hotel_only → flight None; flight_only → hotel None). Only PRESENT hotel/flight components are expected; if any expected component's price is missing → `None` (UI keeps its fallback). Rounded 2 dp.
- **B3 (Pierre-decided): `total_all_in` = hotel + flight + booked PAID activities ONLY.** Free landmark **anchors are NEVER in `total_all_in`**; their entrance-fee estimates go in the SEPARATE soft field `anchor_estimate_total` (a "≈ €X entrance fees" line the UI shows, not the authoritative total). Paid activities are counted **exactly ONCE, from the day-plan** — the legacy singular `SketchCard.activity` is **NO LONGER summed** by `_build_footer` (was in `total_estimated_price`), so there is no double-count. The `SketchCard.activity`/`anchors[]` FIELDS stay populated for the current UI and get **deleted in a fast follow-up** once the dev-branch UI switches to `day_plan`.
- **New `SketchFooter` fields (additive):** `activities_total: Optional[float]` = sum of paid day-plan activities (the constant the UI re-adds in its selectionChanged re-sum; `None` when the activities layer is off / no day-plan, `0.0` when the layer is on but no paid activity survived the ceiling); `anchor_estimate_total: Optional[float]` = SOFT sum of per-anchor `entrance_fee_estimate` (kind=="anchor"; legacy `price`-only anchors reconcile via fallback) — shown as "≈ entrance fees", **never** in `total_all_in`. **Header == rows** (see the design-v2 BACKEND fields note above).
- Set in `_build_footer` at compose (correct for scripted/priced), then **recomputed in `handler.py` post-LENS** per sketch once the live hotel price (and any substitution) is final — `_footer_total_all_in(_sk.hotel, _sk.flight, _sk.day_plan)` at both handler call sites (post-LENS recompute + tail undercut). Activities were priced at compose time, so only the hotel/flight legs change on recompute — the day-plan sum is constant.
- `SketchFooter.total_all_in: Optional[float] = None` is ADDITIVE.
- **CLIENT seat/extras reconciliation (2026-07-13, finish-page redesign — design (a)):** user-selected **seat fees** (and future bags/extras) are CLIENT-known only (chosen in the seat stage, held in React state, never POSTed) and are **NOT in `total_all_in`**. The finish surfaces therefore fold them on TOP: the UI's displayed trip total = `total_all_in` + client all-pax seat/extras total (shared `resolveTripTotal({..., seatsExtrasTotal})` in `mywai-hotel-providers/ui/src/components/tripTotal.ts`, surfaced as a "Seats & extras €X" receipt line). The finish bar and the confirmation card read the SAME resolved total so they cannot disagree. **KNOWN FUTURE MIGRATION → design (b) when `mywai-booking` lands:** the server total must then include seats to hand to the cart (the agent never pays, but the cart needs the accurate charge), so the client will POST selected seats/extras and `total_all_in` becomes server-authoritative incl. seats. Until then the server contract above is unchanged.

## `SketchFooter.pax_adults` — authoritative (verified 2026-07-06)
`pax_adults` is fully resolved in `handler.py` (produce_intent → `_extract_trip_params`, then trip-type/family defaults) BEFORE `run_sketch_engine`, threaded through `_compose_one_sketch` → `_build_footer(pax_adults=...)`, and persisted to `trip_slot["pax"]["adults"]` (restored on the committed fast-path). So `footer.pax_adults` always carries the resolved value — the UI can drop its `TRIP_TYPE_PAX` pill re-derivation. No code change (verification task).

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_hotel_query_filter.py` | `_codon_chrom`, whitelist, STAY injection contract, match floor |
| `tests/cognitive/test_triptych_ranks.py` | `_triptych_rank_depths` invariants + deck pick (hero=pool[0], distinct, no dup) + variation_value picker unchanged |
| `tests/cognitive/test_swap_score_carry.py` | swap alternatives carry persisted match_score/codon_contribs/dna_boost (not 0) |

## Landmark proximity resolution (2026-05-25)

`landmark_resolver.py::extract_landmark_geo(user_message, city)` is called inside `_search_and_rank_hotels` before `_compose_hotel_filters`. When a proximity phrase is found ("near", "close to", "next to", etc.) it returns `{lat, lon, radius_m, name}` which becomes a PostGIS geo filter in the DNA shortlist call — forcing the result set to hotels within `radius_m` metres of that point.

**Two-tier resolution:**

1. **DDB registry** — `mywai-prod-ti-landmarks` seeded from GeoNames. Covers monuments, museums, parks, bridges, towers, squares with ≥1 alt name. In-memory cache per container (no TTL — warm container retains it indefinitely). Monuments (MNMT) always present; squares (SQR) only if GeoNames had alt names.

2. **Haiku fallback** (`_haiku_resolve_landmark`) — fires when the DDB scan finds no match but a proximity phrase was detected. Handles streets, squares the ETL missed, neighbourhoods, addresses. Uses 1000m radius (vs 1500m for DDB) to account for coordinate imprecision. Result cached with 3600s TTL.

**Key behaviour:**
- No proximity phrase → `None` immediately (no Haiku call, no geo filter)
- DDB hit → return as-is (no Haiku call)
- DDB miss + proximity phrase → Haiku → `{lat, lon, radius_m=1000, name}` or `None` if unknown
- Haiku `None` → geo filter skipped, search falls back to codon/BM25 path

**Logs to watch:** `landmark_geo_detected` (DDB hit), `landmark_haiku_resolved` (Haiku hit), `landmark_haiku_failed` (Haiku error — silently ignored).

| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_landmark_resolver.py` | 10 unit tests (mocked DDB + Bedrock): no-phrase guard, DDB hit, DDB miss→Haiku, Haiku null, Haiku error, TTL cache |
| `tests/cognitive/test_landmark_resolver_live.py` | 5 live tests (`SHERPA_INTEGRATION=1`): Arc de Triomphe DDB hit, Place Dauphine Haiku fallback, Rue de Rivoli, Paris bbox validation, cache latency |

## Session amenity injection (2026-05-27, updated 2026-05-27)

When `session_intent_codons` contains family or business codons, `_dna_shortlist_sketch` loads category-specific display names from the S3 amenity frequency file and appends them as a separate OR group in `extra_filters["amenity_groups"]`.

**Codon triggers:**
- `FAMI#KIDS`, `SOC#FAM`, `SOC#GRUP` → `_load_category_amenities(city, "fami_kids")`, filtered to **Family Rooms only**
- `STAY.STY#BIZN`, `PURP#BUSN`, `PURP#WORK` → `_load_category_amenities(city, "purp_busn")` (full list)

**Family filter:** Only "Family Rooms" is used as the SQL gate (231 Paris / 89 Rome hotels). The other 6 fami_kids amenities (Babysitting Service 518 Paris, Water Slides 493) are commodity-tier and qualify too many hotels. Graceful fallback to full list if the city frequency file has no "Family Rooms" entry.

**Key design decisions:**
- **No guard** — injection fires even when the user also typed explicit amenities (e.g. "family hotel with a pool"). Old guard `if not explicit_amenities` was removed.
- **Separate group** — session codes are packed into one frozenset and appended to `amenity_groups`, not merged with user pill groups. Result: AND between groups, OR within each group.
- **`_compose_hotel_filters` is unchanged** — user's explicit amenities still go through the normal `_required_pill_groups` pill path. Session codes bypass `_required_pill_groups` entirely (no `single_amenity_group` param — that was removed 2026-05-27).

**Example outcomes:**
- "family trip in Paris" → `[["am_7f2e60475e"]]` — Family Rooms only, OR within group
- "family hotel with a pool" → `[[pool codes], ["am_7f2e60475e"]]` — two AND'd groups
- "business hotel with wifi" → `[[wifi codes], [business codes]]` — two AND'd groups

```python
# Family injection (sketch_engine.py ~line 1313):
_family_primary = [dn for dn in _ams if dn.lower() == "family rooms"]
_session_ams = _family_primary if _family_primary else _ams  # graceful fallback
```

**Logs:** `session_amenities_injected` (event in DDB turn log) with `context="family"` or `context="business"` and `amenities=[...]`. Family context will now always log `amenities=["Family Rooms"]` (or full list if fallback). `dna_shortlist_filters` logs `amenity_groups` showing all groups.

| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_session_amenity_injection.py` | 13 tests: signature guard, pill routing regression, group assembly, cross-contamination, combined user+session groups |

## Date + duration resolution — temporal_resolver (2026-07-02, TEMPORAL_REDESIGN)

The regex/dateparser `resolve_dates` ladder is **DELETED**. All temporal understanding now flows
through an enum-constrained `TemporalIR` (`shared/payloads.py`) emitted by the LLM, resolved by the
single pure `shared/temporal_resolver.py`. See `docs/TEMPORAL_REDESIGN.md`.

**The flip:** the LLM does language understanding only (WHAT kind of temporal expression + params,
in any language) — it does NO calendar math. `temporal_resolver.resolve(ir, *, today, tz,
existing_check_in, existing_check_out, default_nights) -> (check_in, check_out, confidence)` owns
ALL arithmetic: `today_in(tz)` anchoring, year rollover, `_nth_weekend_of_month`, `_SEASON_BANDS`,
`_NIGHTS_MAP`, nights precedence, the travel-window clamp (`[today.year, today.year+3]`), the
follow-up-freeze rule, and `CONFIDENCE_BY_KIND`. Pure, deterministic, idempotent; **never raises** —
any calendar error on a branch degrades that branch to `none` (freeze existing / next-Friday default).

`run_sketch_engine(..., temporal_ir=...)` threads the IR and calls `resolve` once (site 5807) with
`default_nights=_NIGHTS_MAP.get(pattern.id, 3)`. sketch_engine contains no date parser.

**Kind → confidence** (LOAD-BEARING at the discover 0.2 gate): explicit range 0.90 / single 0.80;
nth_weekend 0.90; tomorrow/this·next·long-weekend/weekdays 0.85; weekend_in_month 0.80;
next_week/week_after_next/in_n_weeks/in_n_days 0.75; next_month 0.65; month_part early·late 0.60;
in_month + month_part mid 0.50; season 0.55; `none`+existing (freeze) 0.95; `none`+no-existing 0.20.

**Follow-up-freeze (on the IR, not text):** `kind == none` (incl. invalid-IR degrade) ⇒ freeze
existing dates (0.95) or default; a bare `kind:none, nights:5` re-derives the span without moving
check-in. `kind != none` ⇒ override from the IR. This structurally kills the anecdote-override class
("we loved Paris in May" → `kind:none`) — there is no substring month scan anymore.

**nights precedence:** `ir.nights` → existing span (freeze only) → `default_nights` → 3, clamped 1..30.

**Weekend anchoring (`this_weekend` vs `next_weekend`):** `this_weekend` uses `lattice.next_weekend(today)` — the nearest bookable Fri–Sun, which SKIPS the in-progress weekend to the next Friday on Fri/Sat/Sun. `next_weekend` is exactly ONE weekend past the CURRENT calendar weekend: it anchors on `today + (4 - today.weekday())` (the coming Friday Mon–Thu, the in-progress weekend's Friday Fri–Sun) **+ 1 week** — it does NOT add a week to `lattice.next_weekend`, which already advanced on Fri/Sat/Sun and would double-advance to the next-NEXT weekend (bug fixed 2026-07-03). `lattice.next_weekend` is shared and unchanged; the fix lives in `temporal_resolver`. Both are 0.85 confidence, `_weekend_co` applies `ir.nights` when present (else natural Sunday). On Fri/Sat/Sun `this_weekend == next_weekend` (both point to the next bookable weekend), a benign consequence of the lattice skip.

**Fail-safe:** every site materialising an IR from raw LLM output wraps `TemporalIR.model_validate`
in `try/except ValidationError` → degrades to `TemporalIR()` (kind=none), logged, never throws the
turn. The IR is a strict trust boundary (ranged fields + cross-field validators reject incoherent
combos, e.g. `nth_weekend` with no month, `day=45`, `n=9`).

**Committed fast-path:** Sonnet is skipped, so `extract_temporal_ir` (handler.py) makes ONE cue-gated
(`_TEMPORAL_CUE_RE`), per-turn-memoized Haiku call forcing a `report_temporal` tool whose schema is
the SAME `TEMPORAL_TOOL_PROPERTY` as produce_intent. Mirrors `extract_contrast_clear`.

**Dependency removed:** `dateparser` is dropped from `lambda/requirements.txt` (no remaining importer).

| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_temporal_resolver.py` | Golden corpus (~150 cases): ladder parity, the deleted intent.py null-list now resolving, this session's currency/duration/lead-time/anecdote bugs, confidence-boundary (0.2 gate), invalid-IR fail-safe, freeze/override + nights precedence, `_nth_weekend_of_month` math. Pure/offline (frozen `today=2026-07-02`). Live multilingual extraction battery (§10.7) is a separate guru harness. |

## Flight path in sketch engine (2026-06-14)

When `_flight_needed` is True (trip has "flight" in `products_needed`), `_compose_one_sketch()` calls `_resolve_flight()` in parallel with hotel search.

### `_resolve_flight(origin_city, destination_city, check_in, check_out, pax_adults, exclude_ids, user_constraints)`

**Airport resolution priority (origin):**
1. `user_constraints["origin_override"]` — chat override ("fly from Heathrow not Luton") → IATA lookup
2. `user_constraints["resolved_iata"]` — from `trip_init` DDB slot
3. `user_constraints["home_airports"]` — DNA `identity.homeAirports` list (e.g. `["LTN"]`)
4. City name → `_city_to_primary_iata()` lookup against full OpenFlights dataset

Up to 3 origin airports fired in parallel via `ThreadPoolExecutor` — primary + up to 2 nearby within 120 km (haversine from OpenFlights `airports.dat`). Results merged, deduped by `offer_id`, ranked via `rank_flight_offers_from_lens()`.

**Budget ceiling:** `user_constraints["flight_ceiling_abs"]` (from `budget_allocator.py` output, persisted in DDB `trip_slot.budget_allocation`). Post-filter after ranking: if all offers exceed ceiling → `budget_warning=True`, show cheapest anyway.

**Timing check:** Haiku call after flight + anchors resolved. Inputs: flight arrival datetime, hotel check-in, Day-1 anchors with `time_of_day`, pax composition, domestic vs international. Output: `anchors_to_defer[]` (Day-1 anchors pushed to Day 2), `lattice_confirms[]` (e.g. "Late arrival 22:45 — hotel will need late check-in notice").

**Return value:** `FlightBlock` with `real_pricing=True`, `offer_id` (Duffel), `slices[]`, `alternatives[]` (up to 4 for swap carousel), `origin_airports_tried[]`.

### Helper functions (sketch_engine.py, ~line 2071+)
| Function | Purpose |
|----------|---------|
| `_get_flight_airports_map()` | Lazy-loads OpenFlights dataset from `flight_routes` contract |
| `_city_to_primary_iata(city, airports_map)` | City/airport name → IATA code (partial match) |
| `_nearby_airports(iata, airports_map, radius_km, max_count)` | Haversine filter for nearby airports |
| `_resolve_origin_airports(...)` | 4-tier priority chain → list of up to 3 IATAs |
| `_resolve_destination_airports(city, airports_map)` | Primary + 1 nearby for destination |
| `_airport_display_name(iata, airports_map)` | "London Heathrow" full name |
| `_airport_city_country(iata, airports_map)` | "London, United Kingdom" |
| `_check_flight_timing(flight, anchors, pax_adults, pax_children)` | Haiku timing check |
| `_flight_offer_to_alt_dict(offer)` | Converts LENS offer to alternatives[] dict shape |

## Activity pricing spine — real DNA→LENS path (2026-07-08, increment A)

Replaces the dead `search_activities` stub + hardcoded placeholder prices with a real path: **DNA activity retrieval → LENS live pricing → `ActivityBlock`**. Internal only — `ActivityBlock`/`SketchCard.activity` already ride the wire as opaque AWSJSON, **no public-schema change**. Ships behind the existing `comp.activity_required` gate. (At increment-A time the budget predicate stayed False; the increment-B FLIP later made `activities_layer_active(trip_shape)` True for full/None trips — see the FLIP section. The single-activity path here is unchanged by the flip.)

### LENS activity client (`cognitive/lens_client.py`)

- `stream_lens_activities(activities, from_date, to_date, paxes, language="en", timeout=20)` — mirrors `stream_lens_flights` via the shared `_stream_lens` transport, POSTing `/activity-search-stream`. Payload `{"activities", "from", "to", "paxes", "language", "type": "activity"}`.
- `fetch_lens_activities(...)` — blocking collector mirroring `fetch_lens_flights`. Keeps `type=="result"` chunks, returns `dict[activity_code -> {price, currency, name, free_cancellation, duration_min, rate_key, pax_amounts, operation_dates, sessions}]`. `{}` on any error / no inventory (never raises).
- **JOIN key = `activity_code`** (== the candidate's `provider_ids[].external_id`). Empirically verified 2026-07-08: there is **NO `offer_id`** on activity result chunks. `price` = the FULL BOOKING TOTAL for the exact `paxes` sent (scales sub-linearly 1→2 adults; private tours have a fixed base) — trust it directly, do NOT reconstruct from `pax_amounts`. ~15% availability hit-rate; only priced activities return.

### DNA retrieval (`_dna_activity_candidates(user_id, city, taste_field, limit=20)`)

Mirrors `_dna_destination_candidates`: synth API-GW v2 event (JWT sub = user_id), direct-invoke `DNA_API_FUNCTION_NAME`, path `/dna/engine/activities`, body `{"query", "filters": {"city": city.lower(), "type": "activities"}, "limit", "intent_codons"}` → dna-shortlist `mode=="activities"`. Parses `json.loads(payload["body"])["results"]`. Each row: `provider_id` (canonical master_activity_id), `provider_ids` `[{provider, external_id}]`, `title`, `category`, `duration_minutes`, `badges`, `usps`, `landmarks`, `codons`, `geo`, `images_s3`, `from_price`=NULL. Any error → `[]` (failure isolated, never breaks the caller).

### `_resolve_activity(city, taste_field, family_mode, user_id, exclude_ids, check_in, check_out, adults, children)`

Signature widened with `check_in, check_out, adults, children`. The stub tool (`tools/search_activities/contract.py`) is **bypassed entirely** (not fixed). Flow:
1. `/dna/engine/activities` retrieval for `city` (limit 20, taste-ranked).
2. Apply `exclude_ids` BEFORE pricing (drop excluded master_ids).
3. Build LENS payload `[{"master_id": row.provider_id, "provider_ids": row.provider_ids}]`.
4. Real party: `paxes = [{"age":30}]*adults + [{"age":8}]*children` — **never a single hardcoded adult** (price depends on the real party).
5. `fetch_lens_activities(...)` → prices keyed on `activity_code`.
6. JOIN in DNA (taste-rank) order: for each row look up `priced[external_id]` where `external_id = row.provider_ids[0].external_id`. The **FIRST priced row wins the block** ("best priced by taste" — no re-sort); **every priced row (taste order) becomes a `priced_pool` entry**.
7. ZERO priced → return `(None, [])` (the card simply does not render — NEVER a placeholder, NEVER an invented price).

**Return (increment B1):** `(Optional[ActivityBlock], list[dict])`. The **first element is byte-identical to the pre-B1 selection** (first priced by taste) — no behaviour change to the selected block, `_build_footer`, `total_all_in`, or the AppSync payload. The **second element is the FULL priced pool** in taste-rank order: one raw dict per PRICED candidate (unpriced excluded), **retained for the B2 day-plan composer** and currently **DORMANT (unread)**. Every degrade path (missing dates / no rows / no payload / no prices / zero priced / blanket `except`) yields `(None, [])`.

**Priced-pool dict shape** (per priced candidate) — carries everything the composer needs that `_resolve_activity` previously computed then discarded:
`master_id` (= row `provider_id`), `external_id`, `activity_code` (both = LENS join key), `provider_ids` (optional/additive — `[{provider, external_id}]`, the join-keys carried onto the saved cart line + `DayPlanItem` for a later LENS re-price; see the activity-recheck path), `title`, `price` (RAW LENS booking total), `currency`, `lat`, `lng` (anchor `lng` convention; sourced from the live top-level row `lat`/`lon` with a nested-`geo` fallback), `duration_min` (raw LENS), `duration_minutes` (DNA row), `sessions`, `operation_dates`, `free_cancellation`, `rate_key` (all from the LENS priced dict), `category`, `codons`, `usps`, `landmarks` (`metadata.landmarks` → top-level `landmarks` fallback → `[]`), `images_s3`.

> **⚠️ Row-shape drift (flagged for B2):** the live activity row from `activity_codon_search` carries `lat`/`lon` **top-level** (`ST_Y`/`ST_X`) and `primary_landmark` (a single string) — **not** a nested `geo` dict nor a `landmarks` list. B1 sources geo/landmarks defensively (top-level-first for geo, `metadata.landmarks`→top-level for landmarks, else empty) so it is correct against the live shape today; `landmarks` will typically be `[]` until backend fill lands. B2 should confirm the canonical geo/landmark source before consuming these.

Blanket `except → (None, [])`; the whole resolution is collected once-per-turn under a hard 15s timeout, so any DNA/LENS failure yields "no activity card", never a broken hotel deck.

**Price mapping invariant:** `ActivityBlock.total_price = price` (the LENS booking total for the real party) stored **RAW**. `price_per_person = round(price / max(1, adults+children), 2)` is derived for display only. `activity_id` = the canonical `provider_id` (master_activity_id) — stable for exclude/lock. `rate_key` is NOT persisted on the block (booking re-quotes later) but IS retained in the priced-pool dict. `price_note=""` (increment A does not claim the activity in the trip total).

### Concurrency — resolved ONCE per turn (mirrors the flight pattern)

`_resolve_activity` is NOT called inline per-sketch (DNA-api `ReservedConcurrency=5` is sized for the hotel fan-out; N inline invokes would 429 the deck). `run_sketch_engine` submits the DNA+LENS work to a `max_workers=1` `ThreadPoolExecutor` **before** the sketch loop (so the 2–13s LENS stream overlaps the hotel search), collects it once via `_collect_shared_activity()` (`.result(timeout=15)`, `except → (None, [])`), and shares the single resolved block across sketches via `model_copy()`. `_compose_one_sketch` takes an `activity_provider` callback exactly like `flight_provider`. Executor is `shutdown(wait=False)` after the sketch loop.

**Priced-pool retention (B1, dormant):** `_collect_shared_activity` now unpacks the `(block, pool)` tuple, stashes **both** turn-level in `_activity_cache` (`"value"` = block, `"pool"` = priced pool), and still returns the block (callback contract unchanged). A sibling `_collect_activity_pool() -> list[dict]` materialises the future (via `_collect_shared_activity`, lock released across the call — `threading.Lock` is non-reentrant) and returns a shallow copy of the stashed pool. **Post-B3 the once-per-turn `_compose_shared_day_plan` reads it** (via `_collect_activity_pool`) to build the shared placement — see the "Increment B3" section. (At B1 it was threaded-but-unread; the `activity_pool_provider` param on `_compose_one_sketch` was removed in B3 since the composer now runs at turn level, not per sketch.)

## Day-plan composer — `cognitive/day_plan_engine.py` (2026-07-08, increment B2 + FLIP)

Unifies the free landmark **anchors** + the paid **activity pool** (the B1 priced pool) into ONE sequenced, sparse, budget-constrained **day-plan**, emitted as an **additive** `SketchCard.day_plan` field. **Post-FLIP the composer is LIVE for full trips** (see the FLIP section below). **Post-B3 the day-plan is the authoritative activities structure: it is PLACED once per turn + its booked PAID activities are folded into `total_all_in`** (see the "Increment B3" section). The dev-branch UI that renders `day_plan` is a SEPARATE later step; the backend `SketchCard.day_plan`/`activity`/`anchors` FIELDS coexist until that UI switch.

### Payload models (live in `shared/sketch_types.py`, alongside every other AppSync block type — so the api Lambda, which bundles `shared` but NOT `cognitive`, can validate a SketchCard without importing the composer)
```
DayPlanItem   { item_id, kind: "activity"|"anchor", day_index, day_part: "morning"|"afternoon"|"evening",
                title, why, lat?, lng?, price?, entrance_fee_estimate?, start_time?, image_url?,
                usps: list[str], currency, duration_hours, booking_required,
                provider_ids?, activity_code?, rate_key?,   # optional/additive — LENS re-price join-keys (activity items only)
                alternatives: list[DayPlanItem] }
DayPlanDay    { day_index, date?: ISO-str, items: list[DayPlanItem] }
DayPlan       { days: list[DayPlanDay] }
```
**design-v2 BACKEND fields (2026-07-09, ADDITIVE — default `None`, existing callers/tests byte-unaffected; UI reads them next):**
- **`price: Optional[float]`** is now the BOOKABLE total for a PAID activity **only** (RAW LENS booking total). An **anchor carries no `price`** (`None`) — its soft entrance-fee estimate lives in `entrance_fee_estimate`. So a non-null `price` unambiguously means "a booked activity".
- **`entrance_fee_estimate: Optional[float]`** — SOFT entrance-fee estimate for an **anchor** (museum ticket etc.), sourced in the composer from the anchor row's `price_estimate`, else a `price_band` midpoint (`LOW 8 / MID 20 / HIGH 45`; FREE/unknown → `None`). `None` for free anchors (UI shows "free") and for paid activities. Card shows "≈ €X". **Header == rows:** the footer `anchor_estimate_total` is now the **sum of these per-item `entrance_fee_estimate` values** (`_day_plan_anchor_estimate_total`; a legacy anchor that only set `price` still reconciles via a `price` fallback). This fixes the "header total not matching rows" design flaw.
- **`start_time: Optional[str]`** — 24h clock "HH:MM". A **paid activity with a parseable HotelBeds session** (`sessions=[{code,name}]`, name/code like "20:15"/"2015") uses the **earliest** real session start; every other item (anchors, sessionless activities) uses the **day-part DEFAULT** — one place: `_DAYPART_DEFAULT_TIME = {morning "09:30", afternoon "14:00", evening "18:30"}`. Keyed on the already-assigned `day_part`, so an evening item resolves to 18:30, never 09:30, and arrival/departure suppression is respected transitively. `None` only when genuinely unknown (unrecognised day_part) — the UI falls back to the `day_part` word; **never a fabricated precise time** (a NAMED session like "Morning" does not parse → falls to the default, not to a made-up clock).
- **`booking_required`** is populated HONESTLY in the composer: paid activity → `True`; free anchor → the row's `reservation_required` (`False` in practice). Not derived from price. The UI drives its badge off this.
- **`image_url: Optional[str]`** — a resolved CDN URL, **MIRRORING the hotel path** (`sketch_engine.py` ~750-771): same `IMAGE_HANDLER_DOMAIN` env (default `d2lajevoh0giji.cloudfront.net`) + same `https://{domain}/fit-in/600x0/{key}` transform. Activity: first resolvable `images_s3` key (dict `{key, caption}` / Aurora repr-string, exactly like hotels). Anchor: a full `photo_url` used as-is, else a bare `photo_url`/`photo_key` S3 key resolved through the same prefix. `None` → UI degrades (no image). No invented URL scheme.
- **`usps: list[str]` + richer `why`** — selling copy. Activity `usps` = up to 3 highlight bullets from the pool row's already-retained `usps`; `why` = the first 1-2 bullets joined with " · " (a real pitch, not the old `usps[0]`-only string), falling back to the row `description`. Anchors keep `why` = their description (`usps` empty). **No new retrieval/pool field was added** — `usps` was already retained on the priced-pool row.
- **`alternatives: list[DayPlanItem]`** — a per-item **SWAP pool** so the UI swaps a slot with **zero round-trip**, mirroring `SketchCard.alternatives` (full same-shape blocks, not compact dicts). **Pool-sourced — NO extra LENS call.** Built by `_attach_alternatives` → `_alternatives_for` on the FINAL placed set (after dedupe/density/budget/cleared), using the **same `_to_item` builder** so an alternative is byte-shape-identical to a primary and lands in the primary's `(day_index, day_part)`. Rules: other facts of the **same `kind`**, **not already placed**, excluded ids already gone from `facts`. **CAP is kind-specific (2026-07-09):** ACTIVITIES `_MAX_ALTERNATIVES` (4); **ANCHORS `_MAX_ANCHOR_ALTERNATIVES` (3)** — dropped from 4 because a 4-of-a-kind list ("Villa Borghese, Parco Lineare, Parco Yitzhak Rabin, Giardino…") is a weak like-for-like. **ORDER within the eligible set is kind-specific:** **ACTIVITIES** keep **taste order** (the B1 pool is already taste/DNA-ranked); **ANCHORS** are ordered by **PROMINENCE-then-CATEGORY-DIVERSITY**. First `_anchor_prominence_key` sorts so recognizable landmarks lead over obscure ghost places (fixes the sparsity/prominence bias where swapping a famous park surfaced "Parco Yitzhak Rabin"-class neighbourhood gardens). The anchor library has **no explicit prominence/rank field**, so the proxy is DERIVED from the two recognition signals every anchor row DOES carry — `crowd_level` (1-10, higher = better-known) and `hidden_gem` (True = obscure) — carried into the anchor fact by `_build_facts`. Key (ascending → smaller = more prominent): `(signal-less last, hidden_gem last, higher crowd_level first, taste_rank tie-break)`. THEN `_diversify_by_type` round-robins the prominence-sorted list across `type_hint` (= `anchor_type`: monument/museum/viewpoint/park) buckets so the set reads **varied** (a monument, a viewpoint, a garden) instead of four of the same type — after the top item, the next-best item of a DIFFERENT type is preferred before a 2nd of the same, then remaining slots fill by prominence. Within each bucket prominence order is preserved (so the most-prominent overall still leads); anchors with a MISSING type share one bucket that drains LAST. **Both prominence AND diversity are ORDER, not a filter** — every eligible anchor still appears (an all-same-type set falls straight through as top-3 by prominence; a low-prominence or signal-less anchor is never dropped-to-zero, just later); same-kind / not-placed / budget / cap constraints are unchanged. For a **paid** slot a candidate is offered only if the swap keeps booked activities within `activities_ceiling_abs` (`paid_total − primary.price + alt.price ≤ ceiling`; ceiling `None` → uncapped) — the same budget constraint the primary obeys. Alternatives **never nest** their own `alternatives` (`[]`), so the payload stays bounded.
- **`price` basis / pax (assessed 2026-07-09, no field added):** `DayPlanItem.price` is the RAW LENS booking **total for the party sent** (pax = `pax_adults + pax_children`). The UI can derive **per-person / "total for N"** WITHOUT a new field — `SketchFooter` already carries `pax_adults` + `pax_children` and the footer ships on every `SketchCard`, so `price ÷ (pax_adults + pax_children)` is reachable UI-side. Per the "minimal thing" rule, **no `price_basis`/`pax_count` passthrough was added** to `DayPlanItem`.
`SketchCard.day_plan: Optional[DayPlan] = None` — **additive**; does NOT touch `activity` / `anchors` / `footer` / `total_all_in`. Rides the existing opaque AWSJSON payload.

### LLM-internal IR (LEAN — placement only, in `day_plan_engine.py`)
`DayPlanIR { days: [{ day_index, items: [{ item_id, kind, day_part, why }] }] }` — Sonnet emits ONLY this (no prices/durations); `kind`/`day_part` are loose strings validated + repaired at join time. Code joins everything else by `item_id`.

### Addressing primitive — `(day_index, day_part, item_id)`
`item_id` = activity **master_id** / anchor **anchor_id** — **NEVER a rate_key**. Designed here so a later strip-back increment (B4) is purely additive.

### `compose_day_plan(activity_pool, anchors, hotel_geo, nights, dates, arrival_window, departure_window, activities_ceiling_abs, pax, converse_fn, taste_field, excluded_item_ids=None, cleared_parts=None) -> DayPlan | None`
> `excluded_item_ids` + `cleared_parts` are **ADDITIVE optional kwargs** (default no-op — every existing caller/test is byte-unaffected). See the "Increment B4" section for their suppression semantics.
Pipeline:
1. **(a) CODE computes facts** — per-item duration (activity: from `duration_min`, treating the `1440` all-day sentinel as **UNKNOWN**, never a 24h day-eater, fallback `duration_minutes`→2h default; anchor: `typical_duration_hours`), geo (activities normalised to `lat`/`lng` in B1; anchors read `lat`/`lng` or nested `geo`), taste rank = input order (the B1 pool is already taste-ranked; activities rank before anchors), `arrival_window`/`departure_window` → suppressed day-parts (evening arrival suppresses all of day-0; early departure suppresses last-day slots).
2. **(b) SONNET places → `DayPlanIR`** via `converse_fn` (max ONE call + ONE retry on invalid JSON, then fall through). Strict `DayPlanIR.model_validate_json`; markdown fences stripped; first-`{…}` fallback.
3. **(c) CODE joins IR item_ids → full `DayPlanItem`s** (drops hallucinated ids + dupes, repairs invalid `day_part`), then VALIDATES + REPAIRS: dedupe by item_id → sparse **density cap ≤2 per (day, day_part)** → **budget ceiling on BOOKED ACTIVITIES ONLY** (greedy-drop the lowest-taste paid item over `activities_ceiling_abs`, backfill a free anchor into the vacated slot when capacity allows; free anchors never count against the ceiling and are never dropped for budget).
4. **(d) DETERMINISTIC FLOOR** — if Sonnet fails/empties, a taste-order plan is built purely in code (activities-first, anchors fill; mirrors `_assign_time`'s day-part assignment; per-day-part + per-day caps + window suppression) so a plan **ALWAYS renders**. Same repair pass applied.

Returns `None` **only** when there is nothing to place (no pool AND no anchors) — anchors are the fallback, never dropped for lack of a paid activity. Never raises (any error → `None`), mirroring the pipeline's LLM-output discipline.

`reorder_by_hotel_distance(day_plan, hotel_lat, hotel_lng)` — cheap deterministic per-sketch post-pass reordering items WITHIN each (day, day_part) by distance from the hotel (`model_copy`; no-geo items sort last). No LLM cost.

### Increment B FLIP — single-source activation (2026-07-08)

The FLIP makes the composer LIVE on full trips + has the budget FUND activities, with ONE predicate so funding and composition can never diverge. **Only user-visible effect = the BUDGET SHIFT** (a full trip's hotel/flight ceilings shrink as activities gets a carve-out). The day-plan stays additive + UI-hidden (B3); footers/`total_all_in` are NOT touched; the single-activity `_resolve_activity`/`SketchCard.activity`/`_build_footer` path is **byte-for-byte unchanged** and coexists during the soak.

- **Single-source predicate** — `shared/activities_layer.py::activities_layer_active(trip_shape) -> bool` = `trip_shape in (None, "full")`. SHERPA-only module (NOT one of the byte-identical cross-repo mirrors `dna_config.py`/`error_tracker.py`, so zero drift-gate impact). Imported by BOTH `handler` and `sketch_engine` — one function object (unit-tested for identity). It replaces the old `handler._activities_layer_active(trip)` (deleted).
- **THREE consumers, one predicate:**
  1. **FUNDING** — `handler._handle_trip_init` (reads `event.get("trip_shape")`) and `handler._run_budget_v2` (has `trip_shape` in scope) join `"activities"` to the priced product set iff `activities_layer_active(trip_shape)`.
  2. **COMPOSITION** — the composer gate is `activities_layer_active(trip_shape)`. **Post-B3 the Sonnet PLACEMENT runs ONCE per turn** in `run_sketch_engine` (see "Increment B3"); each sketch only applies the deterministic per-hotel reorder. Full/None → composes; every narrower explicit shape → skipped (no pool fetch, no Sonnet call, `day_plan` stays `None` on every card). **Decoupled from `comp.activity_required`** — that field still gates ONLY the single-activity path, and `_apply_trip_shape("full")` does **not** force it True.
  3. **GUARD** — `run_sketch_engine._activity_funding_guard(trip_shape, user_constraints, ...)` — backstop for a STALE persisted allocation. When a budget is set (⟺ `activities_ceiling_abs` was threaded, present even at `0.0`), it compares `activities_layer_active(trip_shape)` (composition intent) against `"activities" ∈ allocation.products` (persisted funding). On **compose-WITHOUT-funding** → structured **ERROR** `activity_funding_composition_diverged` (direction `compose_unfunded`) + **DEGRADE to free-landmarks-only** (returns a shallow copy with the composer's `activities_ceiling_abs` forced to `0.0` → paid activities suppressed, anchors kept; caller dict untouched). The benign **fund-without-compose** direction logs **INFO** (`fund_uncomposed`), no degrade. Aligned/no-budget → passthrough.

### Increment B3 — day-plan is the authoritative activities structure (2026-07-08)

Makes the day-plan the source of truth for the trip total + fixes the two FLIP pre-ship observations. Backend-only slice; the dev-branch UI that renders `day_plan` is a SEPARATE later step (flip+B3 deploy together).

- **FIX A — paid pool sourced on the predicate.** The activity-future SUBMIT gate in `run_sketch_engine` is re-keyed from `shape_composition.activity_required` to **`activities_layer_active(trip_shape)`** (the SAME predicate that funds the carve-out + gates composition). So EVERY full trip that funds an activities carve-out also fetches the ~20-item PAID pool that feeds the day-plan (previously a full trip whose pattern had `activity_required=False` funded a carve-out but never fetched the pool → empty paid day-plan). The predicate is a strict superset of the old gate (every narrower shape where it's False already has `activity_required` forced False by `_apply_trip_shape`). Exclude-before-price + the 15%-hit degrade are unchanged; LENS prices the pool ONCE/turn (intended cost). The single-activity BLOCK is still only ATTACHED when `comp.activity_required` — this only adds the pool fetch, not a new card.
- **FIX B — compose ONCE per turn, reorder per sketch.** The Sonnet PLACEMENT (hotel-agnostic item selection + day/day_part) is hoisted into `run_sketch_engine` behind a lazy, thread-safe `_collect_shared_day_plan()` (mirrors the flight/activity once-per-turn collectors: sketch 0 triggers the compose, sketches 1+2 read the cache). Net: **exactly ONE Sonnet call/turn** (was up to one per sketch). The shared placement uses a **hotel-agnostic anchor lookup** (`_lookup_anchors(hotel_lat=0, hotel_lng=0, exclude_anchor_ids=None, _raw_out=...)` → city-centre geo fallback) + the turn-level PAID pool + the shared flight's arrival/departure windows + the activities ceiling. `_compose_one_sketch` no longer composes: it takes a `day_plan_provider` callback and applies ONLY `reorder_by_hotel_distance(shared_plan, hotel_lat, hotel_lng)` (`model_copy`, no LLM) so the three sketches share one placement, reordered within each (day, day_part) by their own hotel. `compose_day_plan` itself is called UNCHANGED — only WHERE it runs moved. (`activity_pool_provider` on `_compose_one_sketch` is removed — the turn-level composer reads the pool via `_collect_activity_pool` directly.)
- **TOTAL — paid activities folded in (Pierre).** `_footer_total_all_in(hotel, flight, day_plan)` adds `_day_plan_activities_total(day_plan)` (kind=="activity", price>0). `_build_footer` no longer sums the legacy singular `activity.total_price` (was in `total_estimated_price`) — the day-plan is the sole activity source, so each paid activity is counted **exactly once, never double**. New `SketchFooter` fields `activities_total` (paid-activities constant; None off-layer, 0.0 when on but no paid survived) + `anchor_estimate_total` (SOFT per-anchor `entrance_fee_estimate` sum, "≈ entrance fees"; header==rows). **Anchors are NEVER in `total_all_in`.** Both handler recompute sites pass `_sk.day_plan` / `_owning.day_plan`.
- **Legacy fields (deleted in a fast follow-up).** `SketchCard.activity` + `anchors[]` stay POPULATED as-is this slice (the current UI still reads them); they get deleted once the dev-branch UI switches to `day_plan`.

### Increment B4 — day-part strip-back ("nothing that morning") (2026-07-09)

Lets a user CONVERSATIONALLY empty a day-plan day-part ("nothing that morning", "clear day 1 afternoon"), leaving it as free time, and un-do it ("actually keep the morning"). ADDITIVE — reuses the B2 `(day_index, day_part, item_id)` addressing; no `compose_day_plan` signature break (two optional kwargs appended). Backend-only; the per-item ✕ UI channel already exists.

- **Persisted state — the `cleared_parts` constraint cell** (`shared/constraint_state.py`). A new `ConstraintCell` on `ConstraintState` whose value is a **list of `{"day_index": int | null, "day_part": "morning"|"afternoon"|"evening"}`** slots (`day_index: null` = the day-part on ALL days, e.g. "no mornings"). It **carries forward like any cell** via the `slots["constraints"]` round-trip — **no legacy bridge** (absent from `state._LEGACY_*_KEYS`, so `_seed` never touches it; absent from `_CLEARABLE_TO_CONSTRAINT`, so `detect_clears`/`apply_clears` can never full-clear it). A whole-state **RESET wipes it** (re-composes fresh). An empty set collapses the cell to `None` (`is_no_signal` → carries nothing → composes normally). `default_factory` empty cell covers older persisted payloads (no migration).
- **Merge — `apply_dayparts(state, *, add, remove, turn_count)`** (`constraint_state.py`, pure, never raises). `add` = slots to clear (union IN); `remove` = slots to un-clear (difference OUT). SET keyed on `(day_index, day_part)` → **idempotent** (re-clear = no-op; un-clear-unset = no-op). Stamped `source="user", explicit=True, set_turn` (a USER act — like a CLEAR — so `_set_cell`/`_seed` leave it alone and it survives to next turn). No-op turn (both empty) preserves the carried cell + provenance verbatim.
- **Edit-IR — `FollowupEditIR.clear_dayparts` + `unclear_dayparts`** (`shared/payloads.py`), each `list[DayPartClear]` (`{day_index: int|null 0-based, day_part: DayPart}`). A `mode="before"` validator drops malformed slots (unknown `day_part`, out-of-range `day_index`) so a stray entry never rejects the whole IR (mirrors `AmenityEdit._filter_pills`); de-dupes on `(day_index, day_part)`. The `_coerce` model-validator resolves a same-turn conflict (a slot in BOTH lists) as **un-clear wins** (restore beats clear, mirrors SET-beats-CLEAR). Prompt: `EDIT_TOOL_INSTRUCTION` + `EDIT_TOOL_PROPERTY` gained both fields (multilingual-native, no up-front translation; `day_index` is **0-based**, day 1 → 0, omit for all days). Extracted via the SAME cue-gated + memoized `extract_edit_ir` Haiku call (`_EDIT_CUE_RE` already fires on "nothing"/"morning"/"clear"/"keep").
- **Apply + persist wiring** — `handler._dayparts_from_ir(ir) -> (add, remove)` unwraps the IR enums; `_build_shadow_state` calls `apply_dayparts(state, add, remove, turn_count)` right after `apply_clears`, from the same memoized `_edit_ir`. The cell is projected to the hotel path via `_PROJECTION["cleared_parts"] → explicit_hotel_slots` (real consumer: the composer), so the carryfwd-authoritative hotel-path injection (`handler` §4.1) lands it in `user_constraints["cleared_parts"]`.
- **Composer honours it** — `compose_day_plan(excluded_item_ids, cleared_parts)`:
  - `excluded_item_ids` (per-item ✕ — reuses the existing exclude channel) are **dropped from `facts` up front**, so neither Sonnet nor the floor can place them. In `run_sketch_engine._compose_shared_day_plan` these = `user_excluded_blocks["activity"] ∪ ["anchor"]`.
  - `cleared_parts` → `_expand_cleared(cleared_parts, n_days)` = concrete `set[(day_index, day_part)]` (`day_index=null` fans out to every day). Threaded into `_join_ir` (skips Sonnet items in a cleared slot) and `_deterministic_floor` (folds cleared slots into `_suppressed_dayparts` → **never backfills** them — the point of "nothing that morning"). A final belt-and-suspenders filter drops any item that slipped into a cleared slot (e.g. a budget backfill). Clearing **every** day-part → an empty-but-valid `DayPlan` (the safety re-floor also honours `cleared`, so it does not re-populate).
- **Tests** — `tests/cognitive/test_day_plan_b4.py` (14): all-mornings/single-slot suppression, un-clear repopulates, excluded_item_id, floor-no-backfill, all-cleared empty plan, `apply_dayparts` add/un-clear/idempotence + carry-forward round-trip + no-op provenance, `cleared_parts` projection, edit-IR shape incl. malformed-drop + conflict-resolution + a multilingual (FR) mapping. `test_carryfwd_phase2.py::test_projection_no_inert_targets` CONSUMER_PATHS gained `cleared_parts → {explicit_hotel_slots}`.

### Ceiling threading — `activities_ceiling_abs` + `alloc_products` (THREE-WAY semantics)
Threaded into `user_constraints` at the two handler twins (post-`_persisted_budget_alloc` reads) whenever a **budget is set** — the first twin gates on a non-empty persisted allocation, the v2 twin on `_effective_budget_abs > 0`:
- **no budget** → `activities_ceiling_abs` **ABSENT** (composer → uncapped). The no-budget CLEAR path pops both `activities_ceiling_abs` **and** `alloc_products`.
- **budget set, activities got 0** → `activities_ceiling_abs = 0.0` (present, NOT popped) → composer shows **FREE LANDMARKS ONLY**.
- **budget set, activities funded** → `activities_ceiling_abs > 0` → greedy-drop over the ceiling.

`alloc_products` = the persisted allocation's `products` list, threaded beside the ceiling — the guard's funding signal + a clean "budget set" signal. `user_constraints` already threads `run_sketch_engine → _build_sketch → _compose_one_sketch`, so the composer reads the ceiling via `user_constraints.get("activities_ceiling_abs")` — no new parameters for the ceiling.

`day_plan_engine._enforce_activity_budget(items, ceiling, facts)` implements the three-way: `ceiling is None` → uncapped; `ceiling <= 0` → drop ALL paid activities (anchors kept); `ceiling > 0` → greedy-drop the lowest-taste paid item over the ceiling, backfilling a free anchor. **Anchors are NEVER dropped by the budget step.**

### Composer gate (dormant seam DELETED 2026-07-12)
The composer is LIVE. `run_sketch_engine` calls `compose_day_plan(...)` directly under the `activities_layer_active(trip_shape)` gate — the **trip shape is the gate**, there is no feature flag. The dead `_DAY_PLAN_COMPOSER_LIVE` flag + `compose_day_plan_if_live(...)` seam (production-unreferenced; only two tests exercised it) were deleted per the pre-launch build-clean rule, along with those two tests.

### Pre-block anchor geo — additive `_lookup_anchors(..., _raw_out=None)`
`AnchorBlock` drops geo; the composer needs it. `_lookup_anchors` gains an optional `_raw_out: list[dict]` that, when passed, is `extend`ed with the SELECTED pre-block raw anchor dicts (geo intact) in the SAME order as the blocks. Behaviour-neutral: only populated on the composer path. **Post-B3 the composer's anchor lookup is turn-level + hotel-agnostic** (city-centre geo, no per-sketch exclude) inside `_compose_shared_day_plan`; the per-sketch `_lookup_anchors` call for `SketchCard.anchors` no longer passes `_raw_out`.

### Measurement harness — `tools/measure_activities_budget_shift.py`
Deploy-free before/after decks for the go/no-go. Runs the REAL allocator (Bedrock stubbed → deterministic rule-based fallback + real `_validate_and_fix` + `_derive_activities_ceiling`) over budgets {1500,3000,5000} × full × dna_budg {0.0, 1.5}, printing flight/hotel/activities ceilings BEFORE (products without activities) vs AFTER (with activities) + deltas. No AWS, no deploy: `PYTHONPATH=lambda python3 tools/measure_activities_budget_shift.py`.

### B1 hardening (per-row isolation)
The B1 priced-pool append loop in `_resolve_activity` now wraps each row in `try/except → log warning + continue`, so a single malformed LATER row skips only that pool entry instead of degrading an already-selected block + accumulated pool back to `(None, [])`. The outer `except` is reserved for pre-loop failures.

### Footer

**Post-B3:** `_build_footer(hotel, flight, activity, day_plan, ...)` folds booked PAID **day-plan** activities into `total_all_in` (via `_footer_total_all_in(hotel, flight, day_plan)`) and no longer sums the legacy singular `activity.total_price` (avoids double-count — day-plan is the sole activity source). New `activities_total` + `anchor_estimate_total` fields; anchors never enter `total_all_in`. See the "Confirmed trip total" + "Increment B3" sections. (Historic: increment A summed the singular activity into `total_estimated_price` ONCE — regression-fixed from `* pax_adults` — and kept `total_all_in` hotel+flight-only; B3 supersedes both.)

### Route pre-validation gate — hybrid source (2026-06-24)

Before the Duffel fan-out, `_pairs_with_routes(pairs)` filters `(origin, dest)` pairs to those with confirmed scheduled service. **It no longer takes a pre-loaded full `routes_map`** — it resolves routes lazily, **per distinct origin (1–3/turn)**, via the hybrid `flight_routes` contract:

- **PRIMARY:** AeroDataBox per-origin daily routes (`get_routes_for_origin(orig)`), fetched on demand, **DDB-cached ~45 days** (`mywai-sherpa-prod-flight-routes-cache`, `pk=ROUTE#<IATA>`), guarded by a **monthly unit budget** (`pk=BUDGET#<YYYY-MM>`; 600 free units/month ÷ 6/req → ceiling **540 units = 90 lookups**). Cache hits cost 0 units. Atomic conditional `ADD units_used :6` reserves budget *before* the HTTP call; refunded on 429/5xx/timeout.
- **FALLBACK:** the static 2014 OpenFlights `routes.dat` map (`get_routes_map()`), used on over-budget / uncovered (HTTP 204) / API error / missing SSM key / DDB error. **Never goes dark, never exceeds the free tier.** Resolution order, failure matrix, and shapes: `docs/AERODATABOX_ROUTES_PLAN.md` §7–8.

`_pairs_with_routes` is called via the `importlib.import_module("tools.flight_routes.contract")` path (preserves the `tools.flight_routes` shadowing fix). Call site (sketch_engine.py ~3886) is now `pairs, _route_carriers = _pairs_with_routes(pairs)` (the `get_routes_map()` arg was dropped). With a 45-day cache and ≤3 origins/turn, **steady state ≈ 0 AeroDataBox calls/turn**.

`flight_routes/contract.py` public surface:
- `get_routes_for_origin(iata) -> list[tuple[str, str]]` — legacy `(dst, carrier)` tuples, hybrid+fallback (primary entry point).
- `get_routes_for_origin_rich(iata) -> list[dict]` — full rows `{dst, carriers[], avg_daily_flights, dst_name, dst_municipality, dst_country, dst_lat, dst_lon}`. AeroDataBox rows carry coords/frequency; OpenFlights-sourced rows carry only `dst`+`carriers` (rest `None`) — callers must null-check.
- `get_routes_map()` — unchanged OpenFlights bulk map, now **fallback-only**.
- `_handle()` (the `flight_routes` tool) uses `get_routes_for_origin_rich`: AeroDataBox `avg_daily_flights` → `frequency="~N/day"` + inline coords for distance (skips `airports.dat` haversine); `data_source` is `"aerodatabox"` or `"openflights"` per-origin.

Env: `FLIGHT_ROUTES_CACHE_TABLE` (default `mywai-sherpa-prod-flight-routes-cache`), `AERODATABOX_MAX_LOOKUPS_PER_MONTH` (90), `FLIGHT_ROUTES_CACHE_TTL_DAYS` (45), SSM key `/mywai/prod/aerodatabox/api_key`. Structured logs: `aerodatabox_cache_hit` / `_live_fetch` / `_budget_exhausted` / `_uncovered` / `_http_error` / `_fallback`.

### `swap_block()` flight branch
When `block_type="flight"`: if `flight_alternatives` already pre-fetched → pick next non-excluded offer. If exhausted → fresh `_resolve_flight()` call with updated `exclude_ids`. Origin city passed via `origin_city` parameter.

### DDB schema for budget/origin (handler.py, ~line 1335)
`_handle_trip_init` saves slots FLAT (no "trip" key). Main turns save under `"trip"` key. `save_trip()` does a COMPLETE REPLACE. Fix: read budget_allocation and origin_iata from BOTH paths:
```python
_nested = _raw_slots.get("trip") or {}
_persisted_budget_alloc = _nested.get("budget_allocation") or _raw_slots.get("budget_allocation") or {}
_origin_iata = _nested.get("origin_iata") or _raw_slots.get("origin_iata") or ""
```
These values are then carried forward in `trip_slot` construction so they survive turn-1 overwrite.

### `budget_allocation` shape + derivation (budget_allocator.py / budget_resolver.py)

`BudgetAllocation.to_dict()` (persisted in `trip_slot.budget_allocation`, echoed in the `sketch_meta` `trip_slots` block):

```
flight_pct, hotel_pct, activities_pct, dining_pct   # ints, sum to 100 over ACTIVE products
flight_ceiling_abs                                   # all-pax TOTAL (NOT per-person)
hotel_nightly_ceiling_abs                            # per-night
activities_ceiling_abs                               # per-trip activities envelope (2026-07)
products                                             # active product set this allocation covers (2026-07)
reasoning, confidence
```

**Per-trip activities envelope (`activities_ceiling_abs`, 2026-07 — activities day-plan step 1).**
- Single source of truth: `_derive_activities_ceiling(activities_pct, budget_total, dna_budg)` in `budget_allocator.py` = `activities_pct × budget_total / 100`, then `× 0.80` when `dna_budg > 1.0` (`dna_budg>1.0` means budget-CONSCIOUS — the same haircut `flight_ceiling_abs`/`hotel_nightly_ceiling_abs` take). Rounded to 2dp. EVERY ceiling site calls this helper — `_validate_and_fix`, the rule-based fallback, and the cascade reconstruction (`budget_resolver.cascade_allocation`) — so the ×0.80 can never be applied on one path and skipped on another. In the cascade the activities ceiling passes through UNSCALED by the cascade itself (like the flight ceiling); the ×0.80 lives inside the helper.
- Granularity is **PER-TRIP** — a share of the fixed total. The day-plan composer derives per-day-part pacing internally (`pot / bookable_day_parts`); no per-day figure belongs in this shape.
- **Additive + backward-compatible:** in-flight persisted trips predate `activities_ceiling_abs`/`products`; every consumer reads with `.get("activities_ceiling_abs", 0.0)` / `.get("products", [])`.

**Activation predicate (`shared/activities_layer.py::activities_layer_active(trip_shape)`) — the SINGLE-SOURCE activation point (post-FLIP).**
`= trip_shape in (None, "full")`. When True, `"activities"` joins the priced product set at BOTH allocator entries (`_handle_trip_init.products_needed`, threading `event.get("trip_shape")` + `_run_budget_v2.products_full`, threading its `trip_shape`), so the ceiling is derived and priced; and it is the composer gate + divergence-guard intent. Full/None trips fund + compose; every narrower explicit shape does neither. Replaced the old dormant `handler._activities_layer_active(trip)` (deleted). See the "Increment B FLIP" section for the full three-consumer wiring.

**Products-set reuse gate (`_run_budget_v2`, generic root-cause fix).**
A carried (no-change) budget reuses the persisted allocation ONLY when `set(products_full) == set(prior_allocation.get("products", []))`. A missing `products` key (in-flight legacy row) reads as `[]` → mismatch → recompute ONCE (self-heal — the re-persisted allocation then carries `products`, and steady-state reuse resumes). Covers any future product join (activities today, dining later) without a product-specific branch.

**Fail-loud guard (`_run_budget_v2`, the ONE convergence point AFTER reuse/cascade/fresh resolve — NOT `_validate_and_fix`, which the cascade bypasses).**
Invariant: `activities_active AND effective_budget > 0 AND remainder_after_locks > 0 ⟹ activities_ceiling_abs > 0`. Both-hotel-and-flight-locked consuming the whole budget legitimately zeros activities (a viability state, gated on `remainder_after_locks > 0`). On violation: structured `activities_budget_zeroed` ERROR + RESIDUAL-based repair (`activities_ceiling_abs = remainder_after_locks`, NOT pct×effective which can overshoot the remainder on a locked turn).

### Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_flight_integration.py` | 33 tests: `_apply_flight_delta`, `_city_to_primary_iata`, `_nearby_airports`, `_resolve_origin_airports`, `FlightBlock` schema roundtrip |
| `tests/cognitive/test_flight_ranker.py` | Flight ranking with cabin/stops/budget filters |
| `tests/cognitive/test_day_plan.py` | 33 tests: B2 day-plan composer — valid compose (sequenced/sparse ≤2/day-part), budget repair drop+anchor-backfill, deterministic floor on junk/empty LLM, anchors-only fallback, None when nothing to place, stable master_id/anchor_id addressing (never rate_key), `duration_min==1440`→unknown, arrival-window suppression, `reorder_by_hotel_distance`, `DayPlanIR` strict parse + retry + fence-strip; **design-v2 times/fees (7): session→`start_time` (earliest), sessionless→day-part default, evening-not-morning default, `start_time` None on named/garbage session + unknown day_part (no fabrication), `entrance_fee_estimate` priced-vs-free + header==rows reconciliation, `booking_required` honest paid-True/free-False, paid activity keeps `price` + no entrance-fee**; **design-v2 images/copy/swap (7): activity `image_url` mirrors hotel CDN + None when no image, anchor `image_url` from bare `photo_key` + full `photo_url` as-is, `why`/`usps` joins bullets (not single-truncation), `alternatives` populated same-shape+capped+no-nesting, anchor-slot alternatives are anchors, paid-slot alternatives budget-respected**; **prominence ordering (4, 2026-07-09): anchor alternatives ordered by `_anchor_prominence_key` (high crowd_level + non-hidden lead, hidden gem trails), prominence-missing anchor sorts LAST not dropped, activity alternatives keep taste order unchanged, `_anchor_prominence_key` unit order**; **prominence-then-category-diversity + cap (4, 2026-07-09): anchor alternatives capped at 3 (`_MAX_ANCHOR_ALTERNATIVES`), diverse set preferred ([park9,park8,museum7,viewpoint6]→leads park9 then museum+viewpoint before a 2nd park), all-same-type falls back to top-3 by prominence, activity alternatives cap unchanged at 4 + taste order** |
| `tests/cognitive/test_activities_flip.py` | 19 tests: the increment-B FLIP — predicate semantics (full/None on, narrower off), same-function-object at both import sites, composer gate mirrors the predicate, three-way ceiling (None/0.0/>0) at both `compose_day_plan` + `_enforce_activity_budget` unit level, divergence guard (no-budget noop / compose-unfunded ERROR+degrade / aligned noop / fund-uncomposed INFO), single-activity gate decoupled from the predicate (`_apply_trip_shape` does not force `activity_required` on full) |
| `tests/cognitive/test_activities_ceiling.py` | fresh/cascade/carried funding on a full trip (no monkeypatch — the predicate funds `full`), budg haircut, fail-loud residual repair, NON-full shape never prices activities |
| `tests/cognitive/test_day_plan_b3.py` | 8 tests: increment B3 — FIX A (full trip fetches the paid pool on the predicate, not `activity_required`; narrower shape fetches none), FIX B (exactly ONE Sonnet compose/turn across 3 sketches; per-sketch `reorder_by_hotel_distance` differs by hotel, same item set), TOTAL (paid day-plan activities folded into `total_all_in`; anchors excluded → `anchor_estimate_total`; no double-count vs a duplicate singular activity; €0 ceiling → `activities_total` 0 + total = hotel+flight; no day-plan → `activities_total` None; `_footer_total_all_in` positional `day_plan`) |
| `tests/cognitive/test_activity_pricing.py` | B3-updated: the singular `SketchCard.activity` is NO LONGER summed into any total (field stays populated); `_build_footer` takes `day_plan` |

## Gotchas
- AppSync `seq` must be unique per turn and per block type — the UI dedupes by `(seq, block.type)`. Two blocks with the same seq+type means the second is silently dropped
- The 60s shortlist LRU cache covers the swap loop — N successive calls for the same query reuse the same candidates without re-invoking DNA
- Weather heuristics inject amenity signals: `rain → spa`, `summer → pool`. These appear in filters automatically; no user input needed
- `amenity_groups` in filters tells the shortlist to enforce per-group AND at Aurora level + progressive relaxation. If `amenity_groups` is present, SHERPA post-filter is skipped to avoid double-filtering
- `taste_field.get("codon_affinities")` returns `0.0` (axis lookup), not the dict — always access `taste_field.codon_affinities` directly
- `_compose_hotel_filters` has no `single_amenity_group` param (removed 2026-05-27) — do not re-add it
- `_resolve_flight()` parallel LENS calls use `ThreadPoolExecutor` — always fire all origin airports simultaneously, never wait for primary before firing alternatives

## Day-plan quality pass (2026-07-12 — commit ce3e210)
- **Over-fetch:** `_ACTIVITY_OVERFETCH = 80` (was hardcoded 20) into `_dna_activity_candidates`. Latency-neutral: the LENS HotelBeds activity worker batches ≤100 codes into ONE availability POST. Priced pool ~4 → ~12-16 (test-host hit rate ~12-20%; the real ceiling is the HotelBeds TEST host — prod-host swap is a separate gated track).
- **Geo-bounded retrieval:** `_dna_activity_candidates` sends `filters.geo_lat/geo_lon/geo_radius_m` (radius `_ACTIVITY_GEO_RADIUS_KM = 80` km) from the `_CITY_META` centroid; unknown city → unbounded (logged). DNA side pre-existed (dna-shortlist → `activity_codon_search` ST_DWithin on ::geography, meters). ACTIVITIES-ONLY — hotel/destination shortlist paths untouched. **80 km deliberately exceeds the 50 km day-trip split so day-trips are retrievable; the "city-core" tightness is the composer's job.** ⚠️ NULL-geo rows are EXCLUDED when the bound fires (Rome ~54 no-coord rows) — geocoding backfill is the upstream fix.
- **Day-trip split:** `_build_facts(city_centroid)` marks `is_day_trip` = `_haversine_km(centroid, item) > _DAY_TRIP_KM (50)`; activities only (anchors never); missing geo/centroid never promotes. `_enforce_day_trip_isolation` = unconditional FINAL chokepoint after every adding pass (Sonnet join, floor, re-floor, budget anchor-backfill): a day holding a day-trip keeps ONLY its single highest-taste day-trip; logs `day_plan_daytrip_isolated`. Deterministic floor is day-trip-aware (day-trip marks the day full).
- **No-repeat invariant:** `_enforce_no_repeats` (replaces silent `_dedupe`): any item_id at most once across the whole trip, keep FIRST in (day_index, day_part) order, runs on BOTH exit paths (main + safety re-floor), logs `day_plan_dupes_dropped`. Prompt also states the rule; the deterministic guard is the invariant. Guard composition is order-dependent in one benign corner (dup'd day-trip + higher-taste day-trip same day → one lost item, both invariants hold).
- **Why sanitizer:** `_clean_why` in `_to_item` — a bare codon or codon token (`CULT#HIST` etc.) never reaches the UI; deterministic plain-English fallback by item kind, else empty. Known benign false-positive: an all-caps 3-5-letter legit why falls back.
- **Locked-hotel gate:** `if locked_hotel and comp.hotel_required:` — hotel-less shapes (`activities_only`/`diy_anchors`/`flight_only`) no longer pin a previously-locked hotel. `hotel_required` is a non-null bool on every composition.
- **Client-side backlog (guru):** `_alternatives_for` can still offer an unplaced day-trip as a Swap candidate for an in-city slot (compose-time invariant only) — filter `is_day_trip` from non-day-trip-slot alternatives in a follow-up.

## Weather-aware day-plan (2026-07-12 — sherpa a6ceb9f + dna 5d0d492, both deployed)
- **Signal:** `weather_setting` (indoor|outdoor|mixed) — factual facet from the activity shredder (DDB golden `metadata.weather_setting`), NOT a codon, no scoring impact. Read side: dna `activity_codon_search` returns `raw_metadata->>'weather_setting'` (4 activity SELECTs); write side: `_UPSERT_ACTIVITY_SQL` now writes raw_metadata (mirrors hotel upsert — it previously never did, so the column was NULL on all rows; field flows only from the next sync/reshred touch). Fill 3/627 today — reshred pending upstream (content_hash clearing owned by the hotel-providers session).
- **Rule (Pierre, revised same day — DOWNGRADE not drop):** on a WET day outdoor items are DEPRIORITIZED, not banned: indoor/mixed rank first at floor/backfill/alternatives (stable sort, indoor/mixed placement byte-identical to pre-weather); outdoor places as LAST RESORT rather than under-filling a day, flagged `rain_risk: true` (additive DayPlanItem field). HARD drop fires ONLY when the wet signal is a REAL short-horizon forecast (`wet_source == "forecast"`; archive/historical/unavailable ⇒ always soft — climatology is a prior) AND the day keeps ≥1 item post-drop (survivor rule — a day can never empty). Swap pool offers outdoor-on-wet ranked last + flagged (day-trip swap exclusion stays HARD). `mixed` NEVER suppressed; missing/NULL/unknown/anchors ⇒ `mixed`. Logs: day_plan_weather_downgraded (soft) / day_plan_weather_dropped (hard). FUTURE: if a swap-pin channel is ever added, the pin must bypass the hard drop or user choices silently revert on recompose (guru note).
- **Wet days:** `_fetch_weather` emits `wet_dates` (forecast daily list, same >3mm threshold as has_rain; archive path un-shifts +365 so dates land in the trip year); `_compose_shared_day_plan` maps to day indices via calendar-date arithmetic (DST-free), out-of-range clamped.
- **Enforcement (downgrade semantics — see the revised Rule bullet above):** `_rainy_outdoor` + `_apply_weather` at the chokepoint slot (order: cleared → weather → isolation → overlap → backfill → final no-repeats+isolation); floor/backfill/alternatives ORDER outdoor last on wet days (last-resort placement, `rain_risk` flagged); hard drop only forecast-sourced + survivor-guaranteed. Swap pool: day-trip exclusion HARD; weather SOFT (outdoor-on-wet offered, ranked last, flagged).
- **Watch item:** ~1/3 of full-shape Paris runs composed an absent day 0 (arrival day) — no guard events fired, consistent with arrival-window suppression by a late-arriving flight variant (correct by design). If an EARLY-arrival trip shows an empty first day, investigate `_backfill_empty_days` coverage of days absent from the Sonnet IR.
- **deploy.sh (dna):** test gate pinned to `python3.12 -m pytest` (23cdaff) — bare pytest resolved to 3.9 and aborted on pre-existing PEP-604 annotations.

## Trip-map stop glyph category — server-truth `meta.category` (2026-07-12)
- **What:** each trip-map STOP entity may carry `meta.category` — the server's authoritative glyph classification for the disc icon. The client (`ui/src/tripmap/render/glyphs.ts` `iconForEntity`, "option C") honours it over its own label/why text heuristic when present; absent ⇒ the client heuristic guesses. Hotels never need it (`kind==='hotel'` ⇒ BedDouble in the client).
- **Enum (server may emit ONLY these):** must stay in lockstep with the UI `CATEGORY_MAP` — `hotel, cruise, boat, food, dining, restaurant, landmark, museum, culture, sight, park, nature, tour, walk, show, nightlife` (`day_plan_engine._TRIPMAP_CATEGORIES`).
- **Normalisation + gate:** `day_plan_engine._map_category(raw) -> Optional[str]` lowercases, passes through an enum value, else maps a confident alias (`monument/tower→landmark`, `viewpoint→sight`, `food_stop/cafe→food`, `bar→nightlife`, `_CATEGORY_ALIASES`), else **None**. Vacuous provider values (`ACTIVITIES`/`GENERAL`/`OTHER`/`''`) and anything unplaceable ⇒ None — the server NEVER guesses (the client heuristic is the guesser). Aurora `activities.category` is known-poor (71/100 = `ACTIVITIES`), so None is the common path today.
- **Flow:** source (`activities.category` / anchor `anchor_type`) → fact `type_hint` → `_to_item` sets the additive `DayPlanItem.category = _map_category(type_hint)` (single chokepoint; placed items AND swap alternatives) → `trip_map_composer._thin_meta(category=...)` emits `meta.category` omit-empty. Additive/opaque (`MapEntity.meta` is a free-form dict); no AppSync change.
- **Memo:** `_signature` is NOT extended — category is stable per `item_id` (already keyed), so no memo churn (same treatment as `why`/`image_url`).
- **Tests:** `test_day_plan.py` (`_map_category` enum/alias/junk→None; `_to_item` category from activity + anchor); `test_trip_map_composer.py` (stop `meta.category` emitted when present, omitted when None).
- **NOTE (stale line above):** the "Enforcement" bullet's `_weather_eligible`/`_enforce_weather` naming predates the weather DOWNGRADE conversion — those are now `_rainy_outdoor` + `_apply_weather(wet_hard)` (soft-downgrade default, hard-drop only on real forecast with a survivor; `day_plan_weather_downgraded` vs `_dropped`), per the revised bullet at the top of this section.

## Plan-edit channel — POST /build/plan-edit (2026-07-12, staged; ships with the visual-pass set)
- **Purpose:** swap/remove on the day-plan updates the MAP sub-second (the old path fired a full /turns stroke regenerate — LENS+Sonnet, new sketch_ids, chosen alternative never honored). Deterministic end-to-end: NO LENS, NO Sonnet.
- **Wire:** POST /build/plan-edit {user_id, trip_id, sketch_id, _turn_id, edit:{op:"swap"|"remove"|"rebase", item_id?, to_item_id?}} → 202; async cognitive `_action` (flight-seats pattern) publishes ONE `sketch_patch` block {sketch_id, day_plan, trip_map, footer, next_action:"PLAN_UPDATED"} seq=10 done — fresh turn_id per edit (server falls back to planedit-<uuid>), so (seq,type) dedupe can't collide. UI adopts in place at STABLE epoch (main-turn city changes still remount via new epoch).
- **Semantics (Pierre):** user owns WHAT, agent owns WHEN/ORDER. swap = exclude outgoing + HARD-PIN the chosen alternative; remove = reflow-only (free time, exclude); rebase (fired at Confirm-trip) = re-rank free anchors for the confirmed hotel (its own variation_value + hotel geo, deterministic _lookup_anchors) preserving ALL paid activities (compose-local pins, NOT persisted) + full re-sequence.
- **Pins (`user_plan_pins`, constraint cell, cleared_parts family):** HARD membership honored on every later compose — bypass weather-hard (rain_risk badge stays), budget drop, day-trip isolation; pin-aware overlap arbiter (pin shifts → cornered pin evicts the non-pinned clasher (no cascade, day never empties) → pin-vs-pin kept); `_ensure_pins` belt has backfill-grade slot eligibility (never cleared/suppressed/day-trip-day). A pin fixes membership, never position. Released only by whole-state RESET (family behavior).
- **Pin pricing dropout (guru condition, closed):** the main-turn compose pool unions persisted plan_state.pool rows for pinned ids absent from fresh pricing (`_carry_pinned_stale_rows`) — carried rows and their items flag `price_stale: true` (additive DayPlanItem field); logs day_plan_pin_stale_price. A factless, non-excluded pin logs day_plan_pin_unavailable (user-choice loss is never silent).
- **plan_state (persisted per trip, single shared blob):** {day_plan, pool, anchors, params, per_sketch:{sketch_id:{variation_value, hotel_geo}}} captured at compose (SketchFrame._plan_state PrivateAttr — never serializes to AppSync), persisted via the single-slots-arg path (UpdateExpression proof-test guards the duplicate-path ValidationException). TODO(cap-removal): conditional/versioned write before parallel turns.
- **Known deferred:** route trusts body user_id/trip_id like its siblings — folded into the pre-launch multi-tenancy fix (now a WRITE path).
