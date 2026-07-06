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

### Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_flight_integration.py` | 33 tests: `_apply_flight_delta`, `_city_to_primary_iata`, `_nearby_airports`, `_resolve_origin_airports`, `FlightBlock` schema roundtrip |
| `tests/cognitive/test_flight_ranker.py` | Flight ranking with cabin/stops/budget filters |

## Gotchas
- AppSync `seq` must be unique per turn and per block type — the UI dedupes by `(seq, block.type)`. Two blocks with the same seq+type means the second is silently dropped
- The 60s shortlist LRU cache covers the swap loop — N successive calls for the same query reuse the same candidates without re-invoking DNA
- Weather heuristics inject amenity signals: `rain → spa`, `summer → pool`. These appear in filters automatically; no user input needed
- `amenity_groups` in filters tells the shortlist to enforce per-group AND at Aurora level + progressive relaxation. If `amenity_groups` is present, SHERPA post-filter is skipped to avoid double-filtering
- `taste_field.get("codon_affinities")` returns `0.0` (axis lookup), not the dict — always access `taste_field.codon_affinities` directly
- `_compose_hotel_filters` has no `single_amenity_group` param (removed 2026-05-27) — do not re-add it
- `_resolve_flight()` parallel LENS calls use `ThreadPoolExecutor` — always fire all origin airports simultaneously, never wait for primary before firing alternatives
