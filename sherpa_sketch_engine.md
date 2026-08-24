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
# _HARD_PILL_NAMES = frozenset({"pets", "accessible", "parking"})  == the design's NEEDS
# Hard amenities → required_amenities (Tier 1 CTE, never relaxed — enforced on BOTH
#   dna-shortlist retrieval paths since booking-flow ticket 03; classifier is
#   mywai-dna/lambda/shared/amenity_needs.py, which mirrors _PILL_GROUPS)
# Soft amenities → amenities + amenity_groups (progressive relaxation, one at a time)
# A relaxed search comes back with HotelBlock.failed_requirements naming, per hotel,
#   which requirements THAT hotel fails. Empty = full match. Ticket 26 renders it.
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

## Cancellation deadline — an ISO STRING with three wire states (2026-08-11, ticket 40)

`HotelBlock.cancellation_deadline` and its `SketchFooter` mirror are `Optional[str]`, **not** `Optional[date]`. The renderer's terms line branches on whether the supplier gave a TIME, so the wire must say three things apart:

| State | Wire value | Meaning |
|-------|-----------|---------|
| instant | `"2026-09-01T18:00:00+02:00"` | Deadline with a time. An offset/`Z` makes it absolute → render it in the DESTINATION's clock. **No** offset = the supplier quoted a naive local time at the property → already the destination's clock, do NOT re-zone. |
| bare date | `"2026-09-01"` | Day known, time NOT known. A complete, legitimate state — never filled in with a start/end of day. What a bare date MEANS is a separate live-probe question; until then the renderer prints nothing. |
| no deadline | `null` | The supplier quoted none. Distinct from a bare date. |

**Consumer test:** `null` → none; else `"T" in value` → instant; else → bare date.

- **`shared/sketch_types.py::normalize_supplier_deadline(raw) -> Optional[str]`** is the single normalizer. Value-preserving only: never adds a time to a date, never adds a zone to a naive instant, never derives a deadline from `check_in` (ticket 06 stands). `date`/`datetime` objects → `isoformat()`. Unparseable → `None` + a `cancellation_deadline_unparseable` WARNING (logged, not swallowed). Parsing is explicit rather than leaning on `date.fromisoformat` leniency, which differs across interpreters.
- **Producers must call it** — `handler._apply_price`, the two AppSync publishes (`hotel_alternative_added`, `hotel_price_update`), the persisted `_wave2_alts` pool, `sketch_engine.swap_block`'s rebuild, and `merge_rate_terms` on the undercut's frame write (ticket 81, below). A `field_validator(mode="before")` on both models is the construction/`model_validate` backstop (these models do NOT set `validate_assignment`, so a plain attribute assignment bypasses it). The footer keeps mirroring the composed hotel verbatim.
- **Wire compat:** a bare date and an absent deadline serialize byte-identically to the old `date` field, so no existing consumer or persisted row changes. Only the previously-destroyed instant is new.
- **Fixed in passing:** `swap_block` used `model_dump()` (not `mode="json"`) and `_handle_swap_block` returns straight out of the Lambda, whose runtime `json.dumps` has no `default=str` — so the old `date` OBJECT made every /swap response with a bare-date alternative fail to serialize. A string cannot.
- ~~**Not carried:** `/build/hotel-recheck` still returns availability + price only, no terms.~~ **Superseded 2026-08-15 by ticket 106** — the re-check now returns the fresh terms and the fresh rate (below).

## Terms on the KEPT frame — an undercut carries them (2026-08-13, ticket 81)

Ticket 53 fixed what a supplier swap **sends**; this is what it **keeps**. The wave-2 tail used to
mutate `hotel.price_per_night` + `total_price` in place and stop, so the persisted frame could hold
the NEW price beside the OLD supplier's refundability and deadline — and ticket 79's terms-only pass
touched the frame not at all, so the answer just published was never written down. Dormant, not
unreachable: ticket 52's undercut gate forces frame and event to agree only with *"Free cancellation
only"* **ON**, and the dial is OFF by default.

- **`shared/sketch_types.py::merge_rate_terms(chunk, *, previous_provider, previous_free_cancellation, previous_deadline) -> (Optional[bool], Optional[str])`** — the server mirror of the client's
  `mergeRateTerms` (playground `rateTerms.ts`), same three branches: a chunk that **states terms**
  (either field, via `states_supplier_terms`) is the authority for BOTH; a **silent** chunk from the
  SAME supplier retains; a silent chunk from a **different** supplier CLEARS both to `null`. Cleared
  is `null`, never `false` — ticket 06's third state. An empty provider id on either side is the wire
  declining to say, not a switch (the buffered hub path omits `provider` on an update).
- **`handler._carry_undercut_onto_sketch(owning, chunk, *, previous_provider, nightly, total, rate_ctx)`** applies
  it to the owning sketch on BOTH tail passes: terms + `hotel.provider` (only when the wire named one)
  + the footer's mirror of the terms, plus the price when it moved. `nightly`/`total` are `None` on a
  terms-only pass — the stay is already on screen at that figure and the publish restates it, so a
  frame write may not reintroduce a price movement. `previous_provider` is read from
  `_published_rate[hid]["provider"]` BEFORE `_remember_publication` overwrites it.
- **The frame is re-persisted after the tail joins.** `save_last_sketch_map` runs ~300 lines BEFORE
  the tail, so an undercut used to reach the traveller's screen and never their stored trip. A second
  targeted save fires only when the tail rewrote a sketch AND the tail actually finished (an
  overrunning tail is still mutating those blocks). Log line `undercut_sketch_map_resaved`.
- **Consumers of that frame, all now reading one rate:** the plan-edit footer + trip-map recompose
  (`handler.py` `_handle_build_plan_edit`), the deferred day-plan compose (`_handle_plan_compose`) —
  both rehydrate `last_sketch_map[sketch_id].hotel` and mirror its terms through `_build_footer` — the
  hotel-lock rehydration fallback (map scan → `user_locked_hotel`), and the budget-v2 locked-nightly
  fallback. The `/swap` pool is a SEPARATE store (`lens_alternatives`) whose price and terms come from
  one chunk, so it is consistent by construction and untouched here.

## A stay rate's IDENTITY and its EXPIRY (2026-08-15, ticket 106)

**Ground truth first.** A live bounded probe of the LENS hub (`POST {LENS_HUB_STREAM_URL}/search-stream`,
3 sweeps, Rome, 15 Aug 2026) shows a stay `result` chunk carries exactly
`type · hotel_id · nightly_eur · nights · total_eur · currency · is_refundable ·
cancellation_deadline · available · provider · latency_ms · seq` — **no `rate_key`, no `rate_code`,
no room code, no board code, no validity window.** (The ACTIVITY lane genuinely does return
`rate_key`/`rate_code`; stays do not.) The same sweep re-run ~7s later returned a different total
(223.52 → 237.60) **and** a different cancellation deadline for the same hotel + provider, so a stay
rate has **no supplier-side continuity at all** — every sweep mints a fresh one. "Is this still the
same rate?" therefore cannot be asked of the supplier, only answered by comparing what we recorded
against what came back.

- **`shared/sketch_types.py::StayRate`** — the record. Both invented fields are LABELLED as invented
  and nothing may drop the label: `id_source="derived"` (the id is a `derived-v1-<16 hex>` hash of
  OUR facets, **never** a booking token — the `"supplier"` member is reserved for the day a real
  `rate_key` exists) and `expiry_source="inferred"` (`expires_at = checked_at + STAY_RATE_TTL_SECONDS`;
  contrast `FlightBlock.expires_at`, which IS the supplier's own offer expiry — the two look identical
  on the wire and only this label separates them). `room_code`/`board_code` are always `null` today:
  recorded-absent, not invented, because "the supplier does not say which room" is itself a fact.
- **Identity = hotel · provider · check_in · check_out · pax_adults · room · board · free_cancellation ·
  cancellation_deadline.** Terms are IN, because ticket 53 found a card inheriting terms after the rate
  changed hands — a rate whose terms moved is a *different rate* and its id says so. `total`/`currency`
  are OUT: "the price moved" is the event the re-price path exists to report, and currency is routinely
  `null` = NOT TOLD (ticket 86), so hashing it would churn the id while saying nothing.
- **`STAY_RATE_TTL_SECONDS = 1800`** — 30 min, §17 HTML:4265's *"gate the reveal at half an hour or so"*
  taken literally. It is **ours**, a policy about how long we will show an unchecked price, not a window
  anybody promised. **Ticket 82's reveal gate must READ this constant, never copy the number.**
- **`build_stay_rate(price_rec, *, hotel_id, check_in, check_out, pax_adults, nights, checked_at?)`** —
  THE one builder, called at all four LENS parse sites so they cannot drift: `cognitive/lens_client._process_chunk`
  (both `result` and `update`), the `cognitive/handler` Phase-1 drain, the wave-2 tail's persisted
  `_wave2_alts` pool row, and `shared/lens_recheck`. Takes the price-record shape those sites already
  build (NOT a raw chunk). Non-finite `total` → `None` (a `Decimal('NaN')` would make DynamoDB reject
  the whole swap-pool write). Never raises out of a parse loop; a failed stamp leaves the record
  **unstamped**, which reads as stale.
- **`stay_rate_is_expired(rate, *, now?)`** — `None`, unstamped and unparseable all read **expired**.
  Deliberate direction: the cost is a re-check we did not need; the other direction serves a price
  nobody checked.
- **`compare_stay_rates(saved, fresh) -> StayRateComparison`** — `{same_stay, same_rate, changed[],
  price_delta, price_comparable}`. Reports, never merges: there is no path through it by which the
  saved rate's terms reach a rate that no longer states them. `price_comparable=false` (and
  `price_delta=null`) whenever either side has no price or the units differ / are NOT TOLD — ticket 86
  applied to arithmetic.
- **Carriage:** price record → `HotelBlock.rate` (additive, default `null`, so every persisted sketch
  and swap-pool row still parses) via `handler._apply_price`, which **copies** rather than rebuilds so
  the card and its pool row share one `rate_id` and one `checked_at`. `_carry_undercut_onto_sketch`
  **re-stamps from the MERGED block** (so a retained term is inside the new identity and a cleared one
  is not); with no `rate_ctx` it drops `rate` to `null` rather than leaving a superseded stamp beside a
  moved price. `sketch_engine.swap_block` **carries the persisted pool rate** instead of re-minting it —
  re-stamping would reset the clock and make a 40-minute-old rate look freshly checked, which is exactly
  what ticket 82's age gate must be able to see. `rate` is NOT in `_SKETCH_MAP_STRIP`.
- **THE one re-price path** stays `POST /build/hotel-recheck`; tickets 82 and 32 come through it rather
  than growing a second mechanism. See `sherpa_api.md` for its widened body/response.

## Activity cancellation terms — a graduated SCHEDULE (2026-08-12, ticket 18)

A stay quotes one deadline; an **experience quotes a schedule**. HotelBeds returns
`operationDates[].cancellationPolicies[] = [{dateFrom, amount}]` — a list of breakpoints
("from this moment, this amount applies"). Three hops discarded it: the Go adapter kept only
`from`/`to`, `_build_facts` dropped even the surviving `free_cancellation` flag, and neither
`ActivityBlock` nor `DayPlanItem` had a field for it.

**Wire shape** — `cancellation_policies: Optional[list[{date_from, amount}]]`, in the
**supplier's own order** (never re-sorted — a schedule read out of order is a different
schedule).

- `date_from` obeys the deadline contract above (instant / bare date), normalized by
  `normalize_supplier_deadline`. HotelBeds quotes naive instants (`"2026-08-18T00:00:00"`),
  which are the OPERATOR's own clock and are never re-zoned.
- `amount` is the supplier's figure carried **verbatim**. Whether it is a charge retained or a
  sum refunded is the supplier's convention and is **deliberately not asserted here** — a
  renderer that wants to print "full refund" must establish that meaning first (one live probe).
  The adapter and this contract guarantee only the numbers and their order.
- **`None` is a value**: no schedule quoted, or one we could not carry. Never `[]` (which reads
  as "no penalty, ever") and never collapsed to a single derived deadline.

**Two normalizers, both in `shared/sketch_types.py`:**

- `normalize_cancellation_schedule(operation_dates)` — reads the schedule off a priced LENS
  offer. Returns `None` when: no operation date carried one; **any** breakpoint is unusable
  (unparseable `date_from`, non-numeric `amount`) — a schedule is carried WHOLE or not at all,
  because a graduated schedule missing its middle tier understates what a late cancellation
  costs; or the operation dates carry **different** schedules (no single answer → we do not
  invent one). Every drop logs `cancellation_schedule_dropped reason=…`. **Known and accepted:** an
  operation date with NO policies is skipped rather than counted as a disagreement, so an offer with
  one quoted and one silent window asserts the quoted schedule for the whole offer. Left as is —
  such windows nearly always differ on `date_from` and drop anyway, and the schedule prints its own
  dates.
- `coerce_cancellation_schedule(raw)` — the same rules over an already-normalized raw list (a
  persisted `plan_state.pool` row, where `amount` is a DDB `Decimal`). Never raises: a malformed
  row costs the cancellation LINE, never the day-plan.

**The four hops, end to end:**

1. `fargate/activity-worker/main.go` → `mapCancellationPolicies` carries the supplier's list
   verbatim onto each canonical `OperationDate` (`cancellation_policies`, `omitempty` so absence
   emits no key). It filters and re-formats nothing — the adapter that discarded this data does
   not get a second opinion about it. **`FreeCancellation` is a `*bool` here and on `hbRate`, not
   a `bool`**: a plain bool decodes an OMITTED `freeCancellation` key to Go's zero value and emits
   an asserted "non-refundable", which no later hop can tell from a supplier that actually said no.
   With `*bool` + `omitempty` the three states reach the wire as key-absent / `false` / `true`, and
   `chunk.get("free_cancellation")` resolves absence to `None`. `copyBool` detaches the flag from
   the decoded response, which offers outlive.
2. `lens_client.fetch_lens_activities` → derives `cancellation_policies` from `operation_dates`
   (which are still carried untouched) and makes `free_cancellation` **TRI-STATE** — the old
   `chunk.get(..., False)` asserted "non-refundable" on every activity whose supplier said
   nothing (ticket 06's defect, mirrored on the experience).
3. `sketch_engine._resolve_activity` → both fields onto the priced-pool row **and** onto
   `ActivityBlock` (new `free_cancellation` / `cancellation_policies`, both default-null).
4. `day_plan_engine._build_facts` (the hand-written key list that was dropping the flag) →
   `_to_item` → `DayPlanItem`. **Anchors force both null** — a free landmark is never booked.

Additive/opaque AWSJSON throughout — **no AppSync schema change** (same pattern as
`rain_risk`/`price_stale`), default-null so every persisted record still parses.

**Acceptance counter:** `lens_activities_done` now logs `terms=` (priced activities with a
refundability flag at all) and `schedules=` (with a carryable schedule). `schedules=0` alongside
`priced>0` means the supplier quoted none — not that the plumbing broke.

**NOT built, recorded rather than silently omitted:**

- **The operator-cancellation clause** §20 prints ("*also fully refunded if the operator cancels
  for weather, or if too few travellers book*", HTML:5198-5199) has **no field in the HotelBeds
  Activities availability response** — not in `rates`, `rateDetails`, `operationDates` or the
  embedded `content` factsheet (whose fields are enumerated in
  `mywai-product-search/docs/hotelbeds-first-feasibility.md:126-140`). Nothing was added for it:
  a permanently-null field would be dead plumbing, and asserting the clause on our own authority
  is exactly what §20 forbids at HTML:5346-5353. Ticket 29 renders silence for that line.
- **The 25k harvested Viator rows.** §20 step one ("a cancellation role in the mapper's fixed
  list") is a **different repo and a schema change**: the role list is
  `analyst_agent._FACTORY_IDENTITY_OPTIONAL` in `mywai-hotel-providers`, and the data is
  `viator_products_cancellationpolicy_refundeligibility` (25,232 rows). Reaching it needs a new
  role + a canonical column + a re-shred, and Viator is not a live LENS pricing provider today.

## `SketchFooter.pax_adults` — authoritative (verified 2026-07-06)
`pax_adults` is fully resolved in `handler.py` (produce_intent → `_extract_trip_params`, then trip-type/family defaults) BEFORE `run_sketch_engine`, threaded through `_compose_one_sketch` → `_build_footer(pax_adults=...)`, and persisted to `trip_slot["pax"]["adults"]` (restored on the committed fast-path). So `footer.pax_adults` always carries the resolved value — the UI can drop its `TRIP_TYPE_PAX` pill re-derivation. No code change (verification task).

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_hotel_query_filter.py` | `_codon_chrom`, whitelist, STAY injection contract, match floor |
| `tests/cognitive/test_triptych_ranks.py` | `_triptych_rank_depths` invariants + deck pick (hero=pool[0], distinct, no dup) + variation_value picker unchanged |
| `tests/cognitive/test_swap_score_carry.py` | swap alternatives carry persisted match_score/codon_contribs/dna_boost (not 0) |
| `tests/cognitive/test_cancellation_deadline_instant.py` | the deadline's three wire states (instant / bare date / none), the normalizer, model rehydration of legacy rows, /swap producer + JSON-serializability, ticket-06 non-regression |
| `tests/cognitive/test_activity_cancellation_terms.py` | ticket 18: a graduated schedule survives as a schedule (order preserved), the deadline obeys ticket 40, all-or-nothing drops (unusable breakpoint / disagreeing windows / non-numeric amount) log and yield `None`, DDB-Decimal round-trip, both models' wire shape + legacy rehydration, the pool→fact→item hops, anchors state no terms, LENS + re-check tri-state |

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

## Stay settings dials — the four non-amenity filters + their echo (2026-08-13, ticket 52)

The stay sheet's STARS / AREA / RATE / nightly-cap controls are typed `pre_context` fields (shape: `contracts/sherpa_api.md`), applied here and **echoed back on the frame** so the sheet renders the search rather than its own memory (§17, HTML:4143 — "the chip state coming from what the server echoes back").

**Where each one cuts** — three doors, and every one of them is guarded, because §17's invariant is that *every stay shown satisfies every chip shown as active*:

| Dial | Applied | Where |
|------|---------|-------|
| `stars` | `user_constraints["stars_min"]` → `_compose_hotel_filters` → Aurora `WHERE stars BETWEEN :min AND :max` | retrieval |
| `area` | geocoded server-side (`handler._resolve_stay_area_geo` → `landmark_resolver.resolve_base_location_geo`, >50 km-from-city guard) → passed to `_search_and_rank_hotels(area_geo=…, area_geo_wins=…)` → the SAME `geo_lat/lon/radius_m` filter a proximity message produces | retrieval |
| `rate.free_cancellation_only` | post-pricing on the tri-state `is_refundable`; `None` (nobody told us) is DROPPED, not treated as free | LENS merge |
| `nightly_cap` | post-pricing on `nightly_price`, **HARD — no `_BUDGET_FLEX`**. The tier cap beside it carries 20% because the tier is an inference; a number the traveller dragged is not | LENS merge |

**PRECEDENCE — it turns on where the dial came from, and getting this wrong is a shipped defect class.** A dial the traveller just moved AT THE SHEET is an explicit control and wins outright. A dial read back out of the persisted slot is a MEMORY, and a memory may never outrank what the traveller just said — §17 gives the chat *"anything the pills do not cover — 'nearer the river'"* (HTML:4104), and the interface has no way to show that it ignored you (same shape as ticket 56).

- **stars:** `shared.stay_dials.resolve_star_floor(dial, already_set, from_sheet=…)`. From the sheet → overwrite. Carried → FILL only what nothing else set; when something else did, this turn wins and the DIAL FOLLOWS it, so the slot and the echo describe the search rather than the memory.
- **area:** three-way, in `_search_and_rank_hotels` — sheet-this-turn (`area_geo_wins=True`, message resolver not consulted at all, since two anchors would AND into an intersection neither describes) > a place named in THIS TURN's message > the carried area as fallback. The slot then follows the geo the search ACTUALLY used (read off `lattice_summary["landmark_geo"]` at the trip_slot build, which is after the engine ran), so a chat "near the Colosseum" that outranked a carried area is what the sheet reopens on.

The two post-pricing dials run through ONE predicate (`shared.stay_dials.price_row_passes`, wired as `handler._stay_price_ok`) at **all FOUR doors** a stay can reach the traveller through: the primary sketch, the substitution/alternatives pool, the wave-2 `hotel_alternative_added` that feeds `/swap`, and the wave-2 **UNDERCUT** (`hotel_price_update`). The fourth is the easy one to miss — it sits fifty lines below the third in the same loop, and it re-prices a stay that is ALREADY ON SCREEN while publishing the new rate's terms, so an ungated undercut lands a cheaper NON-refundable rate under an active "free cancellation only" chip (ticket 53's defect through another door; ~20% of simulated supply is non-refundable). It also drops terms-less `update` chunks, which carry no terms at all. `tests/cognitive/test_stay_dials.py::test_every_door_a_stay_reaches_the_traveller_through_is_gated` is a structural pin against a fifth door opening ungated.

**The echo — `SketchFrame.stay_filters`** (`shared/stay_dials.StayFiltersEcho`, one `StayDialEcho` per dial: `{requested, applied, state, note}`). `applied` is the only field a chip may light up from; it is `null` for every non-`applied` state. Four states, and deliberately no fifth: `applied` · `unresolved` (an area nothing could place) · `unsupported` (`breakfast_included` — LENS returns a total, a nightly rate, a provider and cancellation terms, and **no board/meal plan from any provider**, so nothing could answer it) · `off`. There is NO `relaxed`: §17 relaxes a WANT amenity visibly below five results, but a stars floor or a cap that empties the pool leaves an honestly short list with the true denominator, never a quietly widened one.

`stars` and `area` echo the EFFECTIVE filter, not the sheet's request — so a floor or an anchor the traveller set in the CHAT is echoed too (`requested: null, applied: …`). That is the other half of the invariant: *a chip the user cannot see may never cut the results*.

**Carry-forward:** persisted at `slots.trip.stay_dials` (written from the parsed object, all-off writes no key). A turn whose `pre_context` carries any of the four keys REPLACES the set whole; a chat refine carrying none keeps it. Logs: `stay_dials_resolved`, `stay_area_geo_applied`, `stay_area_out_of_city`, `stay_dial_stars_yielded_to_turn`, `stay_dial_area_followed_search`, `lens_undercut_dropped_stay_dials`, and `lens_sketch_substituted reason=stay_dials`.

| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_stay_dials.py` | parse/degrade, both price-gate dials, echo states, the retrieval filters, the area>message precedence, and the REAL `/turns` forwarder round-trip |

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

### `_resolve_flight(origin_city, destination_city, check_in, check_out, pax_adults, exclude_ids, user_constraints, progress=None)`

**Airport resolution priority (origin):**
1. `user_constraints["origin_override"]` — chat override ("fly from Heathrow not Luton") → IATA lookup
2. `user_constraints["resolved_iata"]` — from `trip_init` DDB slot
3. `user_constraints["home_airports"]` — DNA `identity.homeAirports` list (e.g. `["LTN"]`)
4. City name → `_city_to_primary_iata()` lookup against full OpenFlights dataset

Up to 3 origin airports fired in parallel via `ThreadPoolExecutor` — primary + up to 2 nearby within 120 km (haversine from OpenFlights `airports.dat`). Results merged, deduped by `offer_id`, ranked via `rank_flight_offers_from_lens()`.

**Budget ceiling:** `user_constraints["flight_ceiling_abs"]` (from `budget_allocator.py` output, persisted in DDB `trip_slot.budget_allocation`). Post-filter after ranking: if all offers exceed ceiling → `budget_warning=True`, show cheapest anyway.

**Timing check:** Haiku call after flight + anchors resolved. Inputs: flight arrival datetime, hotel check-in, Day-1 anchors with `time_of_day`, pax composition, domestic vs international. Output: `anchors_to_defer[]` (Day-1 anchors pushed to Day 2), `lattice_confirms[]` (e.g. "Late arrival 22:45 — hotel will need late check-in notice").

**Return value:** `FlightBlock` with `real_pricing=True`, `offer_id` (Duffel), `slices[]`, `alternatives[]` (up to 4 for swap carousel), `origin_airports_tried[]`.

### Flight progress frames + the ONE seq allocator (2026-08-11, ticket 35)

The ~25s fan-out used to emit nothing. It now reports on itself — no extra supplier call, only frames. Design: booking-flow §14, `BOOKING_FLOW_DESIGN_2026-08-07.html:3464-3478`.

- **`cognitive/flight_progress.py`** — `FlightSearchCounter` (THE one counter) + `FlightProgressEmitter` (the ladder). Block type **`flight_progress`**, payload `{type, phase, line, routes_total, routes_checked, fares_in, sources_unreached, elapsed_ms}`. `phase ∈ {fanout, searching, sources, slow, ranking}`. The `line` is composed server-side, verbatim from the doc, so no client can paraphrase it; the raw numbers ride alongside.
- **Rules:** the first frame names ROUTES only (a fare count has no denominator yet); climbing frames are batched at 1.5s and their numbers only ever climb (high-water clamp, never re-derived); the >30s line is emitted only while a lane is genuinely outstanding; a lane that failed or passed the collection budget is **unreached, not empty** — excluded from `routes_checked`, never counted as "returned no fares".
- **A ROUTE = one (origin, destination) pair; its LANES = economy + one call per elevated cabin.** A route is `checked` only when every one of its lanes is back and none failed.
- **`_LANE_COLLECT_BUDGET_S = 21.0`** — one shared deadline for the whole collection (replaced a serial `future.result(timeout=21)` per lane, which could stack). Lanes are observed as they land; their offers are **replayed in submission order**, so the accumulated pool and every stable-sort tie-break downstream are unchanged.
- **ONE COUNTER (HTML:3474).** `FlightBlock.offers_evaluated` = `counter.fares_in` (the wait line's closing number AND the reason panel's "Evaluated N options"); `FlightBlock.fares_held` = `1 + len(alternatives)` (the tail door's "All N fares"). Both stamped at the single construction site in `_resolve_flight`. Nothing downstream may recount either. Without a counter (direct invoke / swap / tests) `offers_evaluated` falls back to the ranker's own `len(parsed)` exactly as before.
- **EVERY fetch feeds the counter, or the number is wrong and hidden.** The direct-relax (`:5856`) and alternative-destination (`:5875`) fallbacks rank offers the first fan-out never produced; they call `progress.add_fares(n)` — which adds to `fares_in` WITHOUT touching route accounting, since they re-query routes already reported. They use `progress`, not `_prog`, so the counter is fed even when the route gate emptied `pairs` and no frames were sent. **`fanout_close()` is called after both fallbacks**, so the ranking frame prints the same closed denominator that is stamped on the block. There is deliberately **no `max(counter, ranker)` reconciliation** — two numbers arbitrated at the read site is what HTML:3474 forbids, and it would hide this defect class rather than surface it. Instead a **tripwire** logs `flight_counter_undercount` at ERROR (persists to the error-log table) whenever `fares_in < best.offers_evaluated`; both UIs hide the line at `offers_evaluated <= 1`, so an uncounted path is otherwise invisible.
- **Zero fares emits no ranking frame** — "Ranking 0 fares against your taste profile" is not honest; the caller falls back to a scripted placeholder. The dead-source frame is still emitted when a lane died.
- **PRE-EXISTING, not fixed here:** an empty `pairs` (route gate filtered every pair) makes `_n_workers = 0` and `ThreadPoolExecutor(max_workers=0)` raises, so the alt-destination fallback below it is unreachable and a permanent "no scheduled route" surfaces as "Flight search is temporarily unavailable — please try again". Characterised by `test_an_empty_route_gate_raises_before_any_of_this_PRE_EXISTING_DEFECT`; fixing it changes what a route-gated turn returns, which is a product call.
- **`shared/turn_seq.py` — `TurnSeq` / `SEQ_STREAM_BASE=9`.** Both clients de-duplicate streamed blocks on `(seq, block.type)`, so a flight frame that reused a hotel frame's pair would be **silently dropped** on any turn that runs both lanes (HTML:3478). `handler._do_sketch` creates ONE `TurnSeq` per turn before the engine runs; the flight ladder and the LENS hotel-price tail (formerly a private `_seq_ctr = [9]`) and the trace/done seqs all draw from it. Seq 0-8 stay hand-assigned; the allocator starts above them. Order does not matter — neither client sorts on seq.
- **Inert on its own.** `flight_progress` is not in the playground's `SHOWABLE_TYPES` and the demo narrows to `sketch_frame`, so shipping it before ticket 14 renders nothing anywhere. Absence of frames degrades to the previous silent wait.

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
- `fetch_lens_activities(...)` — blocking collector mirroring `fetch_lens_flights`. Keeps `type=="result"` chunks, returns `dict[activity_code -> {price, currency, name, free_cancellation (TRI-STATE), cancellation_policies, duration_min, rate_key, pax_amounts, operation_dates, sessions}]`. `{}` on any error / no inventory (never raises). See "Activity cancellation terms" above for the two cancellation fields.
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
`master_id` (= row `provider_id`), `external_id`, `activity_code` (both = LENS join key), `provider_ids` (optional/additive — `[{provider, external_id}]`, the join-keys carried onto the saved cart line + `DayPlanItem` for a later LENS re-price; see the activity-recheck path), `title`, `price` (RAW LENS booking total), `currency`, `lat`, `lng` (anchor `lng` convention; sourced from the live top-level row `lat`/`lon` with a nested-`geo` fallback), `duration_min` (raw LENS), `duration_minutes` (DNA row), `sessions`, `operation_dates`, `free_cancellation` (TRI-STATE), `cancellation_policies` (the graduated refund schedule — ticket 18), `rate_key` (all from the LENS priced dict), `category`, `codons`, `usps`, `landmarks` (`metadata.landmarks` → top-level `landmarks` fallback → `[]`), `images_s3`.

> **⚠️ Row-shape drift (flagged for B2):** the live activity row from `activity_codon_search` carries `lat`/`lon` **top-level** (`ST_Y`/`ST_X`) and `primary_landmark` (a single string) — **not** a nested `geo` dict nor a `landmarks` list. B1 sources geo/landmarks defensively (top-level-first for geo, `metadata.landmarks`→top-level for landmarks, else empty) so it is correct against the live shape today; `landmarks` will typically be `[]` until backend fill lands. B2 should confirm the canonical geo/landmark source before consuming these.

Blanket `except → (None, [])`; the whole resolution is collected once-per-turn under a hard 15s timeout, so any DNA/LENS failure yields "no activity card", never a broken hotel deck.

**Price mapping invariant:** `ActivityBlock.total_price = price` (the LENS booking total for the real party) stored **RAW**. `price_per_person = round(price / max(1, adults+children), 2)` is derived for display only. `activity_id` = the canonical `provider_id` (master_activity_id) — stable for exclude/lock. `rate_key` is NOT persisted on the block (booking re-quotes later) but IS retained in the priced-pool dict. `price_note=""` (increment A does not claim the activity in the trip total).

### Concurrency — resolved ONCE per turn (mirrors the flight pattern)

`_resolve_activity` is NOT called inline per-sketch (DNA-api `ReservedConcurrency=5` is sized for the hotel fan-out; N inline invokes would 429 the deck). `run_sketch_engine` submits the DNA+LENS work to a `max_workers=1` `ThreadPoolExecutor` **before** the sketch loop (so the 2–13s LENS stream overlaps the hotel search), collects it once via `_collect_shared_activity()` (`.result(timeout=15)`, `except → (None, [])`), and shares the single resolved block across sketches via `model_copy()`. `_compose_one_sketch` takes an `activity_provider` callback exactly like `flight_provider`. Executor is `shutdown(wait=False)` after the sketch loop.

**Priced-pool retention (B1, dormant):** `_collect_shared_activity` now unpacks the `(block, pool)` tuple, stashes **both** turn-level in `_activity_cache` (`"value"` = block, `"pool"` = priced pool), and still returns the block (callback contract unchanged). A sibling `_collect_activity_pool() -> list[dict]` materialises the future (via `_collect_shared_activity`, lock released across the call — `threading.Lock` is non-reentrant) and returns a shallow copy of the stashed pool. **Post-B3 the once-per-turn `_compose_shared_day_plan` reads it** (via `_collect_activity_pool`) to build the shared placement — see the "Increment B3" section. (At B1 it was threaded-but-unread; the `activity_pool_provider` param on `_compose_one_sketch` was removed in B3 since the composer now runs at turn level, not per sketch.)

### Child-preferences overlay — activity lane (2026-07-24)

A parent's DECLARED child interests steer the activity pool WITHOUT profiling the minor (no child DNA). Interest tags → distinctive codons (`shared/child_interests.TAG_TO_CODONS`) → `child_overlays` on the activities request; DNA runs a SEPARATE floored per-interest lane and returns `child_matches` (full activity rows + `score`) — see `sherpa_dna.md` "Child-preferences overlay". SHERPA-side wiring:

- **Threading:** `child_overlays: Optional[list[dict]]` param added to `_dna_activity_candidates` (adds `child_overlays` to the request body ONLY when present — absent → byte-identical request; NO `child_floor` sent, DNA uses its box16 knob), `_resolve_activity`, `compose_activities_for_hotel`, and `run_sketch_engine`. `_dna_activity_candidates` return widened `list → (results, child_matches)`.
- **Pre-LENS insert (`_merge_child_only_rows`):** a child-ONLY match (provider_id NOT already in the adult deck) is APPENDED to the candidate `rows` BEFORE the LENS pricing loop, so it is priced through the SAME `fetch_lens_activities` call (provider_ids join) and RENDERS as a real slot — a below-adult-horizon zoo actually appears instead of being dropped. Best (highest-score) child-only row per interest, deduped, excludes user-excluded ids, skips a row with no `provider_ids` (can't price → no invented price), bounded to `_CHILD_DECK_INSERT_CAP` (6). Appends (never prepends) so the headline `selected` block stays the adult's best.
- **Post-LENS interleave (`_interleave_child_matches`):** over the priced pool, best match per interest → tag `why="child_interest:<interest>"`, PROMOTE to the pool front (day-plan composer reads taste-rank order), DEDUPE by provider_id (a match in both lanes keeps ONE slot), cap 6. Deck-relative `score` is NEVER compared across lanes (slot, don't compare). A match not in the priced pool (LENS couldn't price / beyond cap) is counted + logged, no invented price.
- **Trigger (D5):** gated on the trip having ≥1 owned child with a mapped interest (`child_ids` non-empty), NOT `is_group` — a solo parent + child fires. Overlays are built by `handler._build_child_overlays(user_id, child_ids)` (ownership-checked via `shared.children.get_child`, IDOR-safe; unowned dropped + logged).
- **Both compose paths:** the initial turn (`run_sketch_engine`) AND the deferred hotel-compose op (`_handle_plan_compose` → `compose_activities_for_hotel`) thread the overlays — the latter rebuilds them from `child_ids` persisted on `compose_seed` (re-derived, never a cached projection, D7), so the day-plan the demo build flow actually SHOWS steers identically to the initial turn.
- **Parity:** no `child_ids` → no `child_overlays` on the wire, `_merge`/`_interleave` are no-ops, priced pool byte-identical. **D6:** a `child_` id never enters `_derive_group_ids`/`participants`/`merge_dna` (dropped by an explicit `is_child_id` guard).

## Day-plan composer — `cognitive/day_plan_engine.py` (2026-07-08, increment B2 + FLIP)

Unifies the free landmark **anchors** + the paid **activity pool** (the B1 priced pool) into ONE sequenced, sparse, budget-constrained **day-plan**, emitted as an **additive** `SketchCard.day_plan` field. **Post-FLIP the composer is LIVE for full trips** (see the FLIP section below). **Post-B3 the day-plan is the authoritative activities structure: it is PLACED once per turn + its booked PAID activities are folded into `total_all_in`** (see the "Increment B3" section). The dev-branch UI that renders `day_plan` is a SEPARATE later step; the backend `SketchCard.day_plan`/`activity`/`anchors` FIELDS coexist until that UI switch.

### Payload models (live in `shared/sketch_types.py`, alongside every other AppSync block type — so the api Lambda, which bundles `shared` but NOT `cognitive`, can validate a SketchCard without importing the composer)
```
DayPlanItem   { item_id, kind: "activity"|"anchor", day_index, day_part: "morning"|"afternoon"|"evening",
                title, why, lat?, lng?, price?, entrance_fee_estimate?, start_time?, image_url?,
                usps: list[str], currency, duration_hours, booking_required,
                provider_ids?, activity_code?, rate_key?,   # optional/additive — LENS re-price join-keys (activity items only)
                free_cancellation?, cancellation_policies?, # optional/additive — supplier terms, ticket 18 (activity items only; anchors forced null)
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

### `compose_day_plan(activity_pool, anchors, nights, dates, arrival_window, departure_window, activities_ceiling_abs, pax, converse_fn, taste_field, excluded_item_ids=None, cleared_parts=None, city_centroid=None, wet_days=None, wet_hard=False, pins=None) -> DayPlan | None`
> `excluded_item_ids` + `cleared_parts` are **ADDITIVE optional kwargs** (default no-op — every existing caller/test is byte-unaffected). See the "Increment B4" section for their suppression semantics.
> **S3 (2026-08-03) deleted the `hotel_geo` parameter — it was never read by a single line of the function.** The plan's hotel binding is carried by `city_centroid`, which is the composer's **DISTANCE ORIGIN** (every `dist_km`, hence day clustering + the day-trip split): the trip's bound hotel when there is one, else the user's chosen base, else the per-city centroid — resolved by `sketch_engine.day_plan_geo_anchor`.
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

### S3 — COMPOSER UNIFICATION (activities chat-edit, 2026-08-03)

One composer for both triggers. `sketch_engine._compose_day_plan_core` is now the ONLY route to `compose_day_plan`: the deferred op (`compose_activities_for_hotel`) and the `/turns` closure (`_compose_shared_day_plan`) each own **retrieval only** and hand the core a pool + anchors. The core owns the pinned-row carry, the wet-day derivation, the shape-derived €0 `diy_anchors` ceiling, the Sonnet `converse_fn` (and its `day_plan_sonnet_compose` line, now emitted from one place with a `path` field), and the `plan_state = {pool, anchors, params}` capture. The pre-S3 "LOCK-STEP: mirror any change in the other" comment is deleted — there is nothing left to mirror.

- **The binding.** `run_sketch_engine(day_plan_hotel_ctx=…)` carries the trip's bound day-plan hotel, resolved by `handler._handle_conversational_turn` from PERSISTED state **before the sketch engine starts** — the pinned contract: a hotel this turn's own search produced can never feed this turn's compose. Same resolver as the button edit (`_resolve_plan_edit_hotel_ctx`, extended with a turn-start `locked_hotel` rung: per_sketch → locked/committed hotel → trip-keyed binding → empty; the button path passes no `locked_hotel`, so its chain is byte-unchanged).
- **What the binding does.** `day_plan_geo_anchor(base_geo, hotel_ctx) -> (geo, source)` — user base > bound hotel > none — feeds all three consumers together: the activity retrieval's geo window (same `_ACTIVITY_GEO_RADIUS_KM`, the window MOVES, it does not widen), `_lookup_anchors`, and `compose_day_plan(city_centroid=…)`. Logged once per compose as `day_plan_geo_anchor {path, source, hotel_id}` — the canary invariant `day_plan_geo_source` asserts on it (a hotel-bound and a city-centred compose both return a plausible plan; nothing else distinguishes them).
- **Scope.** `full` DEFERS its day-plan, so the live turn-path binding applies to the shape-`None` default (hotel deck + day-plan on the same turn). `activities_only` / `diy_anchors` are EXCLUDED: they have no hotel of their own, a lock carried in from a prior shape is stale there, and a user base is the explicit answer anyway. **Unbound refines are byte-unchanged.**
- The compose now writes the trip-keyed `hotel_ctx` itself (it knows what it composed around); the post-sketch-loop finalize only fires when the compose was unbound.

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
- **plan_state (persisted per trip, single shared blob):** {day_plan, pool, anchors, params, per_sketch:{sketch_id:{variation_value, hotel_geo, hotel_id}}} captured at compose (`hotel_id` added 2026-08-03 by S1 — it is what lets the finalize step match the trip's locked/committed hotel into a trip-keyed `params.hotel_ctx`, and what makes an exact per-sketch hit identifiable) (SketchFrame._plan_state PrivateAttr — never serializes to AppSync), persisted via the single-slots-arg path (UpdateExpression proof-test guards the duplicate-path ValidationException). TODO(cap-removal): conditional/versioned write before parallel turns.
- **The refusal SENTENCE rides the patch — `sketch_patch.reply_text` (2026-08-14, booking-flow ticket 92; built, NOT deployed).** A refused edit publishes the unchanged plan AND the reason on **ONE** mutation: the optional `reply_text` string is present only when there is something to say (absent, never `""`, on a successful edit, so a UI cannot render a blank bubble). It replaces the separate `text` block that used to go out at seq 9 ahead of the patch + `done` at seq 10 — AppSync orders nothing across mutations and both UIs settle + close the socket at `done`, so the sentence could reach a socket that no longer existed and the traveller was refused in SILENCE (four frames measured landing `s7, s4, s6, s5` within 8ms on a live turn). Same rule on the CHAT path (`_dispatch_chat_plan_removals`): when the turn produced a patch the reply rides it as `reply_text` at seq 3 done — including a SUCCESSFUL "Removed X from your plan"; when it produced no patch (ambiguous ask, name matched nothing) the `text` block is itself the closing frame and was never in a race. **UI contract: read `reply_text` off any block IN ADDITION to `text` blocks** — the demo (`ui/src/lib/assistantReplies.ts`) and the playground (`AgentPlayground.tsx` plan-edit socket + the /turns stream) must both do this, and clients ship BEFORE the server (the field is additive and inert until then).
- **Known deferred:** route trusts body user_id/trip_id like its siblings — folded into the pre-launch multi-tenancy fix (now a WRITE path).

## `add` — the fourth day-plan op (2026-08-12, booking-flow ticket 19 / §19; built, NOT deployed)
Design: `mywai-demo/docs/booking-flow-design/BOOKING_FLOW_DESIGN_2026-08-07.html` §19 — *"Adding needs a real operation, not a client trick… Add has to exist server-side, drawing on the same pool, or the control is a facade"* (HTML:4806).
- **Wire (same route, TWO phases):** POST /build/plan-edit `edit:{op:"add", day_index:int, day_part:"morning"|"afternoon"|"evening", to_item_id?}` → 202. **Phase 1** (no `to_item_id`, the traveller tapped an open slot) publishes ONE `plan_add_candidates` block `{sketch_id, day_index, day_part, candidates:[DayPlanItem…], next_action:"PLAN_UNCHANGED"}` seq=10 done — read-only, persists nothing. **Phase 2** (with `to_item_id`) runs the normal edit core and publishes the usual `sketch_patch`. Fresh turn_id per request, so (seq,type) cannot collide.
- **The candidate pool is the PLANNER'S OWN, server-side.** `day_plan_engine.add_candidates` reads `plan_state["pool"]` + `plan_state["anchors"]` — literally the two lists `compose_day_plan` was called with, returned by the same statement that composed the plan (`sketch_engine._compose_day_plan_core`). The swap lists (`_alternatives_for`) cannot serve add: each hangs off an ALREADY-PLACED item, so an empty slot has none. Admission is the composer's own three calls — `_admissible` + `_slot_under_cap` + `_slot_clock_free` — against the day's current occupancy; order is `taste_rank`, the floor's own key (no second ranking policy), with `_alternatives_for`'s wet-day outdoor demotion. Withheld: anything already placed, anything on `user_excluded_blocks` (remove stays irreversible — an add that re-offered a removed item would be an undo), and any paid activity that would breach `activities_ceiling_abs` (so a `diy_anchors` trip at €0 offers free anchors only). Capped at `_MAX_ADD_CANDIDATES` (8).
- **DAY-TRIP ISOLATION is enforced at ADMISSION, both directions** (gate finding, 2026-08-12 — the first pass had this hole). The edit `SlotCtx` models no day-trip days, so `add_candidates` seeds `ctx.day_trip_days` from the PLACED plan exactly as the two membership belts do (`_ensure_pins:1583`, `_ensure_child_slot:1662`) — without it `_admissible`'s clause 4 read an empty set and in-city items were offered into the other parts of a Versailles day. And an `is_day_trip` candidate is offered ONLY into an EMPTY day: the floor may place a day-trip on an occupied day and let `_enforce_day_trip_isolation` DROP the cohabitants afterwards, a repair an edit must never make. **The claim the function makes is "would have placed it there AND KEPT it", not merely "placed"** — those differ, and the difference is where this was wrong. Non-self-healing if breached: an added item is a HARD PIN and isolation exempts pins (`membership > isolation`), so a violation would survive every later rebase. Both halves are independently falsifiable (each alone leaves 2 of the 4 day-trip tests red).
- **Weather is deliberately an ORDERING, not a filter** — the discriminator being that `_apply_weather` exempts PINS from its hard drop and an added item becomes a pin (as a swap-in does), while day-trip isolation bends for nobody at any edit site. Every other composer repair pass was checked against the offer: `_enforce_no_repeats` (covered by the placed-id filter), `_resolve_time_overlaps` (`_slot_clock_free`), `_enforce_density` (`_slot_under_cap`), `_enforce_activity_budget` (the ceiling filter), `_normalize_parts` (`_admissible` clause 5 + `_to_item`); `_ensure_pins` / `_ensure_child_slot` / `_backfill_empty_days` are fillers, not admission rules.
- **The id is re-validated inside the op.** `apply_plan_edit("add")` recomputes `add_candidates` for that exact slot and refuses anything not in it (`notes.unplaceable_reason="not_offered"`, plan returned untouched, membership guard refuses, nothing persisted, the traveller gets an add-specific sentence via `PlanEditRefusal.incoming_op="add"`). One function is both the offer and the gate, so a client-supplied id can never enter the plan.
- **Slots are preserved.** `_insert_in_place` — the mirror of `_drop_in_place` — puts the built card at its chronological position on the NAMED day and touches nothing else; a day the plan omits (item-less days are not emitted) is re-created in day order. It deliberately does NOT use `_place_item`: that walks the trip's days, which is right for a swap and would make add a day-search — one generalisation from the move op §19 cuts permanently (HTML:4808 / SETTLED HTML:5516). Membership: `_expected_membership` gains a conserving `add` arm (`current + to_item_id`). The added item becomes a HARD PIN, like a swap-in.
- **BOTH whitelists extended** (`api/handler.py` `/build/plan-edit` 400s an unknown op; `cognitive/handler.py` `_handle_build_plan_edit` drops one) and `day_index`/`day_part` added to the api forwarder's field-by-field `_edit_out` allowlist — `day_index` forwarded as Optional-int because 0 is Day 1, `day_part` dropped to `""` unless it is a real part.
- **Chat is NOT wired (ruling).** `add` is a typed op only. The edit-IR resolver matches titles against `plan_state.day_plan` and never the pool, so an add target (by definition not on the plan) is unreachable by it; and a chat add carries no slot, which would force the server to choose a day — the exact inference the move cut forbids.
- **Refactor rider:** the swap branch's inline edit `SlotCtx` is now `_edit_slot_ctx(params)`, shared verbatim with add so the offered set and the accepted set are judged against one calendar.
- **Owed:** one live turn + a DDB turn-log grep for `plan_add_candidates` before any UI is built on it (the forwarder-drop class has fired four times here).

## ComposeParams rail + trip-keyed hotel binding + edit-IR state bypass (2026-08-03, activities chat-edit S1 — built, NOT deployed)
Spec: `mywai-sherpa/docs/ACTIVITIES_CHAT_EDIT_STUDY_2026-07-26.md` (§B findings A2a/A2b/A5a, the guru convene verdict + the second pass's two mandatory conditions). Backend-only; **no AppSync shape change**.
- **`shared/compose_params.py` — `ComposeParams` (+ nested `HotelCtx`) is now the ONE rail.** Both composers (`compose_activities_for_hotel`, `run_sketch_engine._compose_shared_day_plan`) build it and persist `.to_state()` as `plan_state["params"]`; `handler._handle_build_plan_edit` rehydrates it with `.from_state()` and passes it WHOLE into `day_plan_engine.apply_plan_edit(params=…)`, whose ten former compose kwargs (`nights/dates/arrival_window/departure_window/activities_ceiling_abs/pax/city_centroid/wet_days/wet_hard/hotel_geo`) are **replaced** by it. Serialization is deterministic (field order + `wet_days` sorted + `cleared_parts` deduped/sorted by `(day_index, day_part)`) so persisted state is byte-stable; `from_state` is tolerant (unknown keys ignored, junk → defaults, never raises — a plan-edit publishes to a live UI).
- **A2b FIXED:** `cleared_parts` now rides the rail into `apply_plan_edit → compose_day_plan`. Previously the plan-edit hop never passed it, so clearing a morning then removing any item could re-place into the cleared morning. Adding a future compose param (S4 `pace`, S5 `pinned_slots`) is one field here + its consumer in `compose_day_plan` — there is no transport hop left to forget.
- **A5a FIXED — hotel binding is TRIP-KEYED, not sketch-keyed.** New `params.hotel_ctx` = `{hotel_id, lat, lng, variation_value, price_per_night, total_price}`. Written by the deferred compose (the hotel it ranked around; price legs folded in by `_handle_plan_compose`) and, on the `/turns` path, from the trip's **locked/committed** hotel matched into `_per_sketch_ctx` (which now also carries `hotel_id`). `handler._resolve_plan_edit_hotel_ctx(plan_state, params, sketch_id)` is the ONE precedence chain: **`per_sketch[sketch_id]` (with geo) → trip-keyed ctx (with geo) → empty**. The EXACT per-sketch entry wins because on a soft-lock swatch turn the three cards carry three DIFFERENT hotels — an edit posted against card 2 must order by card 2's hotel, and the trip binding points at the locked/committed one (card 0). The trip binding is the fallback for the re-minted-sketch ORPHAN case, which is precisely "the exact lookup missed". Consumers: `reorder_by_hotel_distance` + the `rebase` anchor re-rank (which no longer silently runs at `variation_value=0.0` after a refine). `compose_day_plan(hotel_geo=…)` stays `(None, None)` on the edit path — the hotel only ever RE-ORDERS, never re-PLACES.
- **A2a FIXED — cue gate = STATE BYPASS + widened vocabulary.** `handler.trip_has_live_day_plan(trip_item)` (plan_state.day_plan with ≥1 PLACED item) is evaluated once per turn just before pool 2 and registers the turn in `_EDIT_IR_BYPASS_TURNS`; `extract_edit_ir` then skips the `_EDIT_CUE_RE` prerequisite for that turn. A per-TURN marker, not a call-site kwarg — `extract_edit_ir` has 7 consumers and a kwarg would let them disagree about whether the gate applied. The study's §B1 miss corpus lives in a SEPARATE `_DAYPLAN_CUE_RE` (intensity word + a day-plan noun within 30 chars; "too much/many"; empty-a-day-part either word order; reschedule verbs `move/shift/reschedule/swap/switch/reorder/rearrange/bump`), OR-ed in by the single gate function **`handler.edit_cue_hit(message)`** and vetoed by `_DAYPLAN_INFO_VETO_RE` for INFORMATIONAL phrasings ("how many days should I stay", "tell me more about the museums", "are the evenings free in Rome", a leading wh-word / copula / "more info") — a question ABOUT the plan is not an edit OF it, and the veto is scoped to the day-plan arms so "how much for a room with a pool?" still fires on the base arms. Both corpora are pinned (assert-HIT / assert-MISS) in `test_edit_resolver.py`. `edit_ir_fired` logs `gate="state_bypass"` only when the bypass was the SOLE admit path (`bypass AND not edit_cue_hit`), else `"cue"`, plus `has_dayparts` + `leashed`; it no longer logs the message text (the bypass makes that population unfiltered free text — same privacy reasoning as `edit_ir_gate_miss`).
- **BYPASS LEASH (S1 gate condition).** A turn admitted ONLY by the state bypass has its IR restricted, at the single post-extraction point in `extract_edit_ir` (so all 7 consumers and the memo see the same object), to the DAY-PLAN cells `_BYPASS_ONLY_IR_FIELDS = {clear_dayparts, unclear_dayparts}`. Rationale: with no edit vocabulary in the message the population is largely smalltalk ("yes", "thanks", "sounds great"), and a hallucinated `budget`/`pax`/`reset`/`dest_switch` would silently re-carve the trip. Nothing legitimate is lost — every leashed field carries vocabulary that fires the cue anyway ("cheaper", "pool", "kids", "business class", "fly from", "start over"), so a real edit is admitted by the CUE and is never leashed. `clears` is leashed too despite the name: its enum is budget/stars/visa_free/climate/business, i.e. trip-wide constraints. The dropped field names ride the `edit_ir_fired` line as `leashed=[…]` — the metric for whether the leash is holding back strays or is over-tight.
- **Condition A — prewarm.** `_pool2` workers 4→5; on bypass turns the module-level `handler.edit_ir_prewarm(message, trip_id, turn_id)` (module level so tests exercise the real function, not a re-typed copy) is submitted alongside `_do_dna`/`_do_facts`/`_do_strokes`/`_check_off_topic`, so the added wall is `max(branches)`, not a serial addition. All consumers read the same `(turn_id, message)` memo → ONE Bedrock call. `log.info("edit_ir_prewarm", call_ms, added_wall_ms)` measures it in prod. **Measured live 2026-08-03 (n=5, prod Bedrock eu-west-3): edit-IR p50 810 ms (max 2441) vs slowest incumbent Haiku branch (strokes) p50 1024 ms → added wall p50 = 0 ms** (criterion ≤ 300 ms); `_do_dna` is normally the binding branch and is slower still.
- **Condition B — tier-scoped `fast_classifier` read timeout.** `shared/bedrock.py` now holds TWO clients: `_bedrock` (60 s, reasoning/vision — untouched) and `_bedrock_fast` (`FAST_CLASSIFIER_READ_TIMEOUT_S`, default **10 s**), selected by `_client_for_tier(tier, read_timeout=None)` in both `converse` and `converse_stream`; `FAST_CLASSIFIER_READ_TIMEOUT_S` is a real CFN parameter/env var on `infra/box4-cognitive.yaml` (tunable without a code change). Rationale: a hung 60 s Haiku eats the whole 60 s Lambda budget. `ReadTimeoutError` is not a `ClientError`, so it propagates to each caller's existing fail-safe — verified caller-by-caller (edit/temporal IR, off-topic, intent, strokes, budget allocator, user_facts, weather summary, vaccine populater, adversarial, shadow all wrap the call and degrade to no-signal). **Per-call override:** `converse(..., read_timeout=30)` for the two fast-tier callers that are GENERATIVE rather than classifiers and sit OFF the turn path — `tools/vaccine_requirement/populater.py` (max_tokens=1000; a timeout there would PERSIST "no requirements") and `cognitive/eval_retrieval.py` (max_tokens=1024; a timeout would score as an extraction miss and poison the eval baseline). One cached client per distinct timeout (`_client_overrides`), never a client per call.
- **Canary:** new invariants `day_plan_nonempty` (in `_PROVIDER_PRESENCE` — an empty plan can be an empty LENS pool / the far-future paid-lane behaviour), `day_plan_items_floor` / `day_plan_has` / `day_plan_lacks` (new `_DAY_PLAN_DEPENDENT` → env on an empty plan) and `edit_ir_fired` + `edit_ir_gate_is` (new `_TURN_LOG_DEPENDENT` → env on a 0-event turn-log read). `run_canary._extract_outcome` reads `sketches[].day_plan` off the SAME invoke response plus the `edit_ir_fired`/`edit_ir_gate_miss` turn-log events. Golden entry `dayplan_cue_bypass` (full tier, SCRATCH user, `just activities in Rome for 3 days` → **`i would be dead on my feet by lunchtime`**, asserting `edit_ir_fired` + `edit_ir_gate_is=state_bypass` + `day_plan_nonempty`). The final turn is deliberately OUTSIDE the vocabulary: S1 widened it, so an in-vocabulary phrase would be admitted by the CUE and would pass with the bypass entirely broken — the widened vocabulary is pinned for free by the unit corpus, while the bypass depends on live trip state and can only be proven live.
