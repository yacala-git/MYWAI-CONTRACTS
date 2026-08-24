# SHERPA API

**One line:** HTTP API Gateway (box7) entry point — routes all inbound requests, handles admin auth, and dispatches async/sync Lambda invocations.

## What it does
- Handles 48+ routes: user-facing (`/turns`, `/feedback`), admin (`/admin/*`), atlas services (`/atlas/*`), and account management
- Admin routes require `x-admin-key` header matching `ADMIN_KEY` env var; missing key in env = open (dev mode)
- `POST /turns` invokes the cognitive Lambda asynchronously (`InvocationType=Event`) and returns 202 immediately
- Eval endpoints use a second sync Lambda client with 90s timeout for blocking calls
- CORS headers on every response; `CORS_ORIGIN` env var controls the allowed origin

## Logical flow
1. API Gateway HTTP v2 event → `lambda_handler` in `lambda/api/handler.py`
2. Admin check via `_admin_auth(event)` on protected routes
3. Route matched by method + path prefix
4. For `/turns`: async fire-and-forget invoke, return `{turn_id}` 202
5. For admin data routes: DDB/S3 reads and proxy calls to DNA Config API
6. Response via `_resp(status, body)` with CORS headers

## Key routes
| Method | Path | What it does |
|--------|------|--------------|
| POST | /turns | Fire cognitive Lambda async, return turn_id |
| GET | /health | 200 {status: ok} |
| GET | /admin/config | Slot schemas + model config from S3 |
| PUT | /admin/config | Update config item in S3 |
| GET | /admin/safety | Safety policies + recent refusal stats |
| POST | /admin/safety/refresh | Trigger safety feed refresher Lambda async |
| GET | /admin/conversations | Recent conversation traces |
| GET | /admin/costs | Cost aggregations per model / category / trend |
| GET | /admin/funnel | Conversion funnel + daily trend |
| GET/POST/PUT | /admin/experiments | A/B experiment management |
| POST | /feedback | Post-trip feedback (no auth) |
| GET | /admin/adversarial | Attack list + pass/fail |
| POST | /admin/adversarial | Trigger adversarial runner async |
| POST | /admin/eval/retrieval | Fire retrieval eval async (returns job_id) |
| GET | /admin/eval/result/{job_id} | Poll S3 for eval result |
| GET | /admin/shadow | Shadow ML stats + DLQ depth |
| GET/POST/DELETE | /admin/anchors | Anchor CRUD |
| GET/PUT | /admin/patterns | Pattern read + override |
| GET/POST/DELETE | /admin/lattice | Lattice rule management |
| GET | /images | Unsplash photo search proxy (`?city=&n=`) — destination-picker carousel; reads SSM key, returns `[{url,thumb,alt,photographer,photographer_url,download_location}]`; warm-cache per city; `[]` (HTTP 200) on Unsplash error/rate-limit. Origin-gated (allowlist → `(403, [])` for external callers, before the key is touched) + API-GW per-route throttle (5/s, burst 10) to protect the demo key |
| GET | /atlas/weather | Weather forecast (open-meteo) |
| GET | /atlas/currency | Currency conversion (ECB Frankfurter) |
| GET | /atlas/seasonality | Destination × month seasonality rating |
| GET/POST | /atlas/briefing | Destination briefing composite |
| POST | /atlas/entities | Entity resolution |
| POST | /build/flight-search | Async: search Duffel flights, stream FlightOfferBlock via AppSync |
| POST | /build/flight-seats | Async: create ARGO cart item from offer_id, fetch seat map + extras, publish `seat_map` AppSync block |
| POST | /trip/init | Sync: save trip slots (destination, dates, budget, origin_iata, products_needed) + run budget allocation; returns {trip_id, budget_allocation, slots} |
| POST | /trips | Save-for-Later upsert (JWT). Body = ARGO Cart/Order pre-image (§0.5): `{trip_id, status ∈ saved\|demo_paid, cart, order?, plan_context?, last_check?, plan_version}`. PK forced to caller sub (body `user_id` ignored). Server invariants: `order.mock≠false`, `order_id` in `DEMO-` namespace, per-line `source=="demo_mock" ⇒ provider_order_id null`, no client `provider` line, ≤100KB. `saved_at` set once, `updated_at` always. Returns `{trip_id, status, plan_version, updated_at, booking_ref}`. **`booking_ref`** = the customer-quotable reference (ticket 37) — six characters from the 27-char confusable-free alphabet `234679ACDEFGHJKMNPQRTUVWXYZ`, ISSUED server-side (`secrets`, never derived from price/date/id) and stored, **only** when `status=="demo_paid"` AND `cart.cart_total_minor > 0` (post-payment only, §20 HTML:5360; a free plan is confirmed but not paid, so it gets `null`). Idempotent per TRIP: re-confirming returns the same reference, never a second one. Uniqueness is enforced at issue time by a conditional `PutItem` (`attribute_not_exists(ref)`) into `…-booking-refs` (PK `ref` → `{user_id, trip_id, issued_at}`), retried on collision; a lost attach race adopts the winner and retires its own row. A client-supplied `booking_ref` in the body is ignored. Issue failure is non-fatal — the save stands and returns `booking_ref: null`. |
| GET | /trips | List the caller's saved trips (JWT). Single `Query` on caller sub, `Limit=100`. Returns media-rich summary rows (`{trip_id, status, plan_version, schema_version, currency, cart_total_minor, destination:{city,flag,image}, date_range, nights, party_size, product_types[], has_order, booking_ref, saved_at, updated_at}`; `booking_ref` is the issued customer reference or `null` — never a fallback slice of the trip_id) — NOT the full cart/plan blobs. `destination`/`date_range`/`nights`/`party_size` are derived server-side from `plan_context.plan` (`image` ← plan `img`; `nights` = `plan.nights` else derived from resolved check-in/out); `product_types` = distinct `cart.items[].product_type`. (Earlier the summary read `plan_context` directly — one level too shallow — so destination/dates were always null; fixed 2026-07-22.) |
| GET | /trips/{id} | Return one full stored trip (JWT). `GetItem` keyed on caller sub → foreign id misses → 404. Returns `{trip}` (Decimal money → int/float) |
| DELETE | /trips/{id} | Delete one saved trip (JWT). Keyed on caller sub. Idempotent (missing row → 200 `{deleted}`) |
| POST | /build/trip-last-check | Stamp the classified outcome of a live re-check onto the caller's own saved trip so GO surfaces CONFIRMED problems + self-clears (JWT; WRITE surface, `allow_link=False`, caller-owns). Body `{trip_id, status ∈ ok\|gone\|price_up\|price_down\|unverified, delta_minor: int\|null}` (`delta_minor` = signed minor-unit move fresh−saved, for price_up/down; null otherwise). **ADDITIVE:** writes only `last_check = {at, status, delta_minor}` — `updated_at` is deliberately NOT bumped (GO resets staleness off `max(updated_at, last_check.at)`). `UpdateItem` guarded by `attribute_exists(trip_id)` (reuses the saved-trips-crud `UpdateItem` IAM, no new grant), so a missing/foreign trip → **404**, never a bare-row create. Returns `{trip_id, last_check}`. GET /trips now exposes `last_check` on each summary row. |
| POST | /build/hotel-recheck | Re-price saved hotels before the (mock) pay step (JWT; stateless, no user data). Body `{hotelIds[≤20, 32-char], checkIn, checkOut, city?, adults?, rates?}`. Returns `{results: {hotelId: {available, total_price, nightly, currency, is_refundable, cancellation_deadline, rate, comparison}}}`; **THE one stay re-price path** — tickets 82 (stale-reveal gate) and 32 (the tail) come through here rather than growing a second "this price may have moved" mechanism. **Ticket 106 (2026-08-15) widened it:** the fresh `is_refundable`/`cancellation_deadline` were previously fetched and DISCARDED, so a caller could show a moved price beside the terms of a rate that no longer existed (ticket 53's untruth through the re-check door); `rate` is the fresh `StayRate` (identity + INFERRED expiry — see `sherpa_sketch_engine.md`); `comparison` is `compare_stay_rates(saved, fresh)` naming what moved (`provider`/`terms`/`price`/`window`/`party`/…), present only when the caller sent `rates: {hotelId: <StayRate>}`. A malformed saved rate is logged and dropped, never a 400 — re-pricing is the safe action. `adults` is the party to re-price for; absent → read off the supplied `rates` when they agree (a rate is quoted FOR a party, and re-pricing a couple's rate for one adult would manufacture a price move), else 1 as before. The route parses its own body in-process, so it has **no cognitive forwarder allowlist** to update; degrades to `{results: {}}` on a LENS miss. **Omit-on-unverified:** the vendored `shared/lens_recheck.fetch_lens_prices(...)` returns `(prices, verified)` — `verified=True` only on a CLEAN full sweep (a real `complete` with no `degraded` marker). When `verified=False` (degraded / never completed), an un-priced hotel is OMITTED from `results` so the client reads it as `unverified` (couldn't-check), NEVER a false "gone"; hotels it DID price are still returned. Re-check payload forces `firstNWindowMs=8000` (== overall timeout) so the hub runs the sweep to completion instead of its 1.5s first-win early-exit (which would false-`degraded` a clean-but-slow 2-provider sweep → every sold-out hotel stuck "unverified"). LENS is a public Function URL (`LENS_HUB_STREAM_URL`, no IAM/VPC) |
| POST | /build/flight-recheck | Re-quote a saved trip's flight (same route/dates/pax/cabin) before the (mock) pay step (JWT; stateless). Body `{origin, destination, departureDate, returnDate?, adults, children?, cabin}` (`children` defaults 0, clamped ≤8). Children are searched as child fares: `passengers = [{type:adult}]*adults + [{age:8}]*children` (default child age 8, matching the sketch/activity party — the caller backs children out of the total party so they are NOT double-counted as adults). Returns `{result: {checked, available, total_price, offer_id, expires_at, carrier, currency, stops, route:{origin,destination}, departureDate, returnDate, adults, children, cabin}}`. **Tri-state** `checked`/`available` (via vendored `shared/flight_recheck.fetch_lens_flight_quote(...)`): a priced offer → `checked:true, available:true`; a CLEAN complete with zero (or no usable-total) offers → `checked:true, available:false` (**gone**); a degraded / non-completing / no-config search → `checked:false` (**unverified**, NEVER a false gone). A returned offer with a missing/zero total is treated as unverified (never a misleading `0.0` fare). Re-quote payload forces `firstNWindowMs=8000` (twin of hotel-recheck; harmless today with one flight provider, future-proofs a 2nd). Flight is INFORMATIONAL in Phase-1 (not bookable here) |
| POST | /build/activity-recheck | Re-price a saved trip's day-plan activities before the (mock) pay step (JWT; stateless, no user data). Body `{activities: [{activity_code, provider_ids: [{provider, external_id}]}] (≤12), fromDate, toDate, adults, children}`. Real party is priced: `paxes = [{age:30}]*adults + [{age:8}]*children` (adults MUST be adults-only — the caller backs children out of the total party). Returns `{results: {activity_code: {available:true, total_price, currency, free_cancellation, cancellation_policies, rate_key}}}` — **PRICED rows ONLY**. `free_cancellation` is **TRI-STATE** (`null` = the supplier said nothing — it used to default to `false`, asserting "non-refundable" on activities nobody checked) and `cancellation_policies` is the supplier's graduated refund schedule or `null` (ticket 18; see `contracts/sherpa_sketch_engine.md` → "Activity cancellation terms"). **No `gone` state, no `available:false` row:** LENS emits no positive "unavailable" signal and activity availability hit-rate is ~15%, so an absent/un-priced code is a WEAK signal read client-side as `unverified` ("couldn't reconfirm"), NEVER "gone" — it can never block the pay path. Degrades to `{results: {}}` on any LENS miss (server never throws for a pricing miss). Lines without join-keys are filtered out BEFORE calling (can't be re-priced → stay unverified). Vendored `shared/activity_recheck.py` (box7 packages `shared/`; NOT a drift-gated mirror). Re-price payload forces `firstNWindowMs=8000` (twin of hotel/flight-recheck). Activity is INFORMATIONAL in Phase-1 (not bookable here) |
| POST | /children | Create a parent-declared kid sub-profile (JWT; caller-bound to `claims.sub`). Body `{name, dob_month, dob_year, diet[], accessibility[], interests[]}` (strict Pydantic; `interests[]` must be from the closed `child_interests.INTEREST_TAGS` set → 400 otherwise). Mints `child_<uuid>`, writes its OWN facts partition (`user_id=child_<uuid>`: `diet#/special_need#` adult-shaped rows + `_profile#child_meta` DOB/name + `_profile#interests` tags) and a parent pointer `_profile#child#<child_id>` under the caller. No behavioural DNA for a minor (D7). Returns 201 `{child_id}`. Storage = the existing `user-facts` table (no new table/grant beyond `BatchWriteItem`) |
| GET | /children | List the caller's children, hydrated (JWT). Reads the caller's `_profile#child#` pointers, hydrates each child partition. Returns `{children: [{child_id, name, dob_month, dob_year, diet[], accessibility[], interests[]}]}` |
| PATCH | /children/{child_id} | Edit a child (JWT; ownership = the pointer exists under the caller → IDOR-free, foreign/unknown id → 404). Partial body (any of `name, dob_month, dob_year, diet[], accessibility[], interests[]`). `diet`/`accessibility`/`interests` use REPLACE-set semantics (a `batch_writer` = **BatchWriteItem**). Returns `{status:"updated", child_id}` |
| DELETE | /children/{child_id} | GDPR erase a child (JWT; ownership-checked, foreign id → 404). Deletes EVERY row of the child partition (`batch_writer` = **BatchWriteItem**) + the parent pointer. Returns `{status:"deleted", child_id}` |
| GET | /green-light | **The server-issued gate (ticket 100): may the client start this trip shape right now?** JWT-gated, stateless, no user data — ONE route for BOTH audiences (the traveller app and Mission Control's operator light) so the two can never hold different opinions about the same lane. Query `?shape=full\|full_trip\|hotel_flight\|hotel_only\|flight_only\|activities_only\|diy_anchors` (default `full`), `?refresh=1` forces a synchronous probe and is honoured ONLY for a `sherpa-dev` caller (the operator light) — otherwise silently ignored (still 200, served from cache), so no authenticated client can turn the gate into a way to hammer the hub the real searches share. Returns `{shape, can_run, unavailable[], lanes:{stay\|flight\|activity: {state, detail, latency_ms, degraded, checked_at}}, shapes:{shape: can_run}, checked_at, age_seconds, stale, source}`. **Three states per lane, never two:** `ok` (200 + ≥1 priced result), `down` (200 + a real `complete` + zero results), `unknown` (non-200, timeout, throw, or `LENS_HUB_STREAM_URL` unset). **`unknown` NEVER blocks** — blocking on "we could not tell" is the defect class this exists to remove — and neither does the ACTIVITY lane, ever, for any shape (free DIY anchors are date-independent, so the day plan degrades honestly). `can_run=false` only when a lane the shape genuinely REQUIRES came back `down`. Cache-first (`shared/lane_probe.py` → the one-row `-lane-health` table, fresh for 30 s, stale-while-revalidate) so the traveller never waits behind the ~12 s probe; an error anywhere in the gate answers `can_run:true`. Probes the PRICING PATH via the public LENS Function URL — never ECS state, never Aurora |

**`/build/flight-seats` request shape:**
```json
{"user_id": "...", "trip_id": "...", "offer_id": "off_000...", "_turn_id": "seats-xxx", "pax_adults": 2, "pax_children": 0}
```
Returns 202 `{turn_id, status: "processing"}`. Publishes `seat_map` block to AppSync under `turn_id`. Playground opens an independent WebSocket subscription to that `turn_id` and stores the block in local state.

### Internal `_action` events (cognitive Lambda only, not via HTTP API)
These are sent directly to the cognitive Lambda from `/build/flight-seats` route (not the turns channel).

| `_action` | Handler | What it does |
|-----------|---------|--------------|
| `build_flight_seats` | `_handle_build_flight_seats(event)` | Creates ARGO cart item via `POST /carts/{trip_id}/items` if `item_id` absent (uses `ARGO_API_URL` env var), then fetches seat map + extras in parallel, runs seating optimiser (window pairs, 3-across for families), publishes `seat_map` AppSync block |

**`build_flight_seats` event shape:**
```json
{"_action": "build_flight_seats", "trip_id": "...", "item_id": "...", "pax_adults": 2, "pax_children": 0}
```

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/api/handler.py` | `lambda_handler` | Full router — 2286 lines |
| `lambda/api/handler.py` | `_resp` | Builds response dict with CORS headers |
| `lambda/api/handler.py` | `_admin_auth` | Checks x-admin-key header |
| `lambda/api/handler.py` | `_lam` / `_lam_sync` | Async (5s timeout) vs sync (90s) Lambda clients |

## Critical code
```python
# Two Lambda clients — async for turns, sync for evals
_lam = boto3.client("lambda",
    config=Config(read_timeout=5))         # Event invocation — never waits
_lam_sync = boto3.client("lambda",
    config=Config(read_timeout=90))        # RequestResponse — eval jobs only

def _admin_auth(event):
    admin_key = os.environ.get("ADMIN_KEY", "")
    if not admin_key:
        return True          # dev: key not set = open
    return event.get("headers", {}).get("x-admin-key", "") == admin_key
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_api_handler.py` | Route matching, admin auth, 202 response shape; `GET /green-light` dispatch (401 without a claim, shape + `refresh` forwarding, a gate error still answering `can_run:true`) |
| `tests/shared/test_lane_probe.py` | The green light itself: the three outcomes per lane off real hub chunk shapes, a non-200 never read as "no inventory", a priced result outranking `degraded`, `unknown` never blocking any shape, the activity lane never required, fresh-cache-hit-does-not-probe, stale-while-revalidate, and the response carrying no infrastructure vocabulary |
| `tests/cognitive/test_saved_trips.py` | Save-for-Later: claims.sub isolation, allow_link=False write surface, order realness invariants, save→get round-trip (money float↔Decimal), hotel-recheck / flight-recheck / activity-recheck degrade paths (activity: priced-only results, `{}` on LENS miss, never `gone`); trip-last-check stamp (writes `last_check`, exposed on summary, `attribute_exists`→404, caller-owns, ok self-clears, additive/no updated_at bump); flight-recheck children (forwarded/clamped/echoed, `{age:8}` child passengers) |

## Gotchas
- **The green light's probe ids are STATIC CONFIG on purpose** (`shared/lane_probe.py`). Resolving them from Aurora would put a 14–25 s scale-to-zero resume in front of the gate, and a cold database would make the gate itself the outage. The probe WINDOW rolls forward (today + 7 days) — a hardcoded date would silently become a past date and every lane would read `down` forever. Never probe ECS/service state: a running count flaps and a task that exists but cannot answer is still a dead lane
- Route matching is string-based (`path.startswith(...)`) — order matters. More specific routes must come before catch-alls
- `_lam` has `read_timeout=5` — using it for a sync call will always timeout. Always use `_lam_sync` for eval endpoints
- DNA Config API URL is `https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com` — set via `DNA_CONFIG_API_URL` env var
- **`/turns` forwards `pre_context` WHOLESALE, and that is load-bearing.** The rest of the cognitive payload is built FIELD BY FIELD, so a key not on that allowlist is silently dropped — `trip_shape`, `base_location`, `participants` and ticket 36's dates each died that way. `pre_context` is the one exception: `pre_context = body.get("pre_context")` copies the whole object, so a NEW key inside it needs no forwarder change. Do not "tidy" this into per-key copying. (Truthiness gate: an EMPTY `pre_context` is not forwarded; every real sender carries at least one key.)
- **`pre_context` stay-settings dials (booking-flow ticket 52).** The stay sheet's four non-amenity controls travel as typed fields beside `amenities`: `stars` (0/3/4/5 — a floor, 0 = none), `area` (a label or null; the server geocodes it, the client never sends coordinates), `rate` (`{free_cancellation_only, breakfast_included}`), `nightly_cap` (EUR a night or null). The SHEET always sends all four (an off dial is `0`/`null`/`false`, never an omitted key — that is how a dial is turned OFF); a chat refine sends none and the server keeps the set it persisted on `slots.trip.stay_dials`. **Key presence is load-bearing beyond replace-vs-carry:** it is also what tells the server the dials were just set BY HAND, which decides whether they outrank what the traveller typed this turn (precedence table in `contracts/sherpa_sketch_engine.md`). Parsed by `lambda/shared/stay_dials.py` (total, never raises). Stars → the Aurora `stars_min` floor; area → the same `geo_lat/lon/radius_m` anchor a "near the Colosseum" message produces; the other two cut post-pricing, on `nightly_price` and the tri-state `is_refundable`. The nightly cap is HARD (no `_BUDGET_FLEX`), and "free cancellation only" drops `is_refundable = None` — a rate nobody vouched for is not a free-to-cancel rate
- `/turns` passes `discover_only` (bool) through to the cognitive event when present in the body — mirrors the `trip_shape`/`budget_abs` passthrough; routes the turn to the product-free destination picker (gated `if discover_only or not city`)
- `/images` IAM is already covered by the api role's `parameter/${Project}/${Env}/*` SSM wildcard (key at `/mywai-sherpa/prod/unsplash/access_key`); SecureString on AWS-managed `alias/aws/ssm` decrypts with `WithDecryption=true` alone (no `kms:Decrypt`)
- Saved-trips is a WRITE surface: `_resolve_effective_user_id(..., allow_link=False)` — a linked companion (account-link) can NOT write to another user's shelf; only the caller's own sub or a real `sherpa-dev` login. Contrast the facts panel which uses the `allow_link=True` default
- Exact-suffix `path.endswith("/trips")` (list/upsert) MUST be dispatched before the `"/trips/" in path` (by-id) branches, else the by-id handlers shadow the list. `/trips` is NOT shadowed by `/trip/init` (different suffix/substring)
- `/build/hotel-recheck` + `/build/flight-recheck` + `/build/activity-recheck` need `LENS_HUB_STREAM_URL` on the API Lambda (box7) — it is NOT there by default (only box4-cognitive had it). Deploy passes `LensHubStreamUrl` in `deploy_box7`. Unset → hotel-recheck returns `{}` / flight-recheck returns `checked:false` / activity-recheck returns `{results: {}}`. The LENS clients are vendored as `lambda/shared/lens_recheck.py` (hotel) + `lambda/shared/flight_recheck.py` (flight) + `lambda/shared/activity_recheck.py` (activity) (box7 packages `shared/`); none is a drift-gated mirror. Deploy order: box1 (table) → box7 (IAM + routes + env + code)
- **Re-check payloads force `firstNWindowMs=8000`** (both hotel + flight). The hub's default 1.5s first-win early-exit marks nearly every clean-but-slow 2-provider hotel sweep `degraded` → `verified=False` → a genuinely sold-out hotel would read "unverified" forever and never "gone". Setting `firstNWindowMs` == the overall timeout makes the sweep run to completion before deciding. TRAP: the hub streaming loop IGNORES `firstNReady` (it compares a global constant) — ONLY `firstNWindowMs` takes effect. This is bounded to the re-check payloads; main-search payloads are untouched
- **`order.bookings[].documents[]` — forward-compat client contract (booked-trip Documents tab).** The mywai-demo booked-trip view (SavedTripPage → `pages/savedTrip/documents.ts`) reads an OPTIONAL per-booking `documents` field: `Array<{ kind: string; url: string; wallet_url?: string|null }>`. It is a **LIST** (not a scalar) because one booking emits many docs — a flight returns a **boarding pass AND an e-ticket**, and a multi-passenger flight returns one `.pkpass` **per passenger** (Duffel returns docs per-passenger/per-kind). The client matches by `kind` with `?? null`, so today (demo, field absent/null) every doc row renders as a clearly-marked DEMO placeholder. When ARGO fulfilment lands, the **server** writes the real `booking_reference`/`provider_order_id` + `documents[]` onto each `order.bookings[]` line and the Documents tab goes live with NO client reshape — only the status derivation flips `demo → issued` (rule: `demo_mock→demo` checked FIRST, then `provider && (booking_reference ?? provider_order_id)!=null → issued`, else `pending`). `wallet_url` assumes a `.pkpass` Add-to-Wallet capability that exists nowhere in the stack yet → harmless null, NOT a backend commitment. This write-back is the only path allowed to mint `source:'provider'`; it is server-written (a real PNR can't be trusted from the UI), security-sensitive, and specced later — it does not block the demo build.
