# Agent Playground UI

**One line:** Interactive SHERPA conversation tester — sends turns via WebSocket / API, renders AppSync-streamed sketch blocks in real time, and shows per-turn DNA profile, cost, and match inspector.

## What it does
- `AgentPlayground.tsx` in `mywai-hotel-providers/ui/src/components/` provides a chat interface for testing SHERPA
- Sends messages via `POST /turns`, receives streaming blocks via AppSync GraphQL subscription
- Renders hotel sketch cards (FORK/SWATCH/SWIPER/COMMIT layouts), anchor blocks, flight blocks, and pricing progress indicators
- FlightCard (2026-06-14): airport full names, per-slice times (↑/↓, date, hhmm→hhmm, duration, Direct/stop badge), airline+cabin+price, lock/swap buttons; `~` prefix when `real_pricing=false`
- FlightSwapCarousel (2026-06-14): slide-up overlay (`absolute inset-0 z-20`) using `flight.alternatives[]` — no API call; tapping selects via `onBlockStroke('flight', offerId, true)`
- Inline seat map (2026-06-14): shown below FlightCard when `seat_map` AppSync block arrives; compact grid with seat dots (amber=recommended, green=extra-legroom, grey=taken), extras chips, party note, expiry countdown, "see all rows" toggle
- Inspector panel (replaces old DnaPanel + TurnTelemetry) shows: archetype roulette, top codons, per-turn cache tokens, cost breakdown, match scores per hotel
- Warm-up button fires 5 parallel `_ping_aurora` invokes to warm dna-api containers before a search

## Key UI components
| Component | Purpose |
|-----------|---------|
| Hotel sketch card | Shows hotel name, stars, match score, amenity pills, why text |
| FlightCard | Airport full names, city/country, per-slice times (↑↓), Direct badge, lock/swap buttons; estimated price with `~` when `real_pricing=false` |
| FlightSwapCarousel | Slide-up overlay from `flight.alternatives[]` — airline/cabin/price/stops per row, tapping confirms selection |
| Inline seat map | Below FlightCard on `seat_map` AppSync block: compact seat grid, amber recommended dots, extras chips, `ExpiryBadge` countdown, party note |
| ExpiryBadge | Ticking countdown from `expires_at` ISO string; turns amber <5 min, red at expiry |
| ArchetypeRoulette | Spins through 8 archetypes on each `archetype_match` block, lands on user's archetype |
| Inspector > DNA tab | User's archetype, top codons, OPPOSITES hits, rebase log |
| Inspector > TURN tab | Latency, guardrail cost, Bedrock prompt cache hit%, total turn cost |
| Inspector > ENGINE tab > What Was Sent (2026-06-28) | **Always-on** `WhatWasSentStrip` pinned to the TOP of the ENGINE tab — driven by `telemetry.turn_diagnostic` (block type `turn_diagnostic`, seq 11), renders on EVERY turn incl. refusals. Verbatim prompt + `faiss_query` + intent-by-source (genuine/dna-soft/session/**fabricated**) with tri-state provenance verdict (clean=emerald / fabricated=rose / unverified=slate, never a false clean) + route/advisory/result. WS teardown is diagnostic-aware (waits for the post-`done` diagnostic, 2500ms cap). See `sherpa_observability.md`. |
| Inspector > ENGINE tab > Search Trace | Intent codons sent to box10, user DNA top-5, ranked candidate table (name/stars/match/BUDG codons) |
| Inspector > MATCH tab | Per-hotel codon contributions, axis signals (only shown when codon_contribs present) |
| PricingProgress block | Shows provider search status while LENS fetches live prices |

## ENGINE tab — Search Trace section (2026-05-23)
Shows the full search dispatch per turn. Data path: `sketch_engine._search_and_rank_hotels` attaches `_search_trace` to the returned pick dict → `compose_trip_shape` writes it to `lattice_summary.search_trace` → `sketch_meta` AppSync block → `telemetry.sketch_meta.lattice_summary.search_trace` in UI.

Fields displayed:
- **Intent codons sent** — `routing_codons` (merged raw + implied + DNA-injected STAY codons sent to box10)
- **User DNA top-5** — user's top codon affinities by absolute weight at search time
- **Ranked candidates** — top-10 hotels from box10: rank, stars, name, RAW `match_score` + calibrated display in parens (`54 (disp 88)`), BUDG codons present. The raw value is what the pipeline gates on (`_MIN_SKETCH_MATCH=35`); the card MatchPill shows the calibrated `matchScoreDisplay` curve — labelling both here stops the two reading as a mismatch.

The same data is also logged to CloudWatch as `search_dispatch` (sherpa) and `hotel_scored` / `ranked_top10` (box10).

## Turn flow
1. User types message → `POST /turns` → SHERPA returns `{turn_id}` 202
2. AppSync subscription fires as cognitive Lambda publishes blocks
3. Each block appended to conversation: hotel cards, anchor cards, copy blocks, pricing progress
4. Inspector panel populated from `cost_meta` / `trace` / `archetype_match` debug blocks (hidden from main chat)

## Flight lock + seat map flow (2026-06-14)
1. User taps Lock on FlightCard → `onBlockStroke('flight', offerId, true)` + `triggerSeatMap(offerId)` fire simultaneously
2. `triggerSeatMap` opens an independent AppSync WebSocket subscribed to a pre-generated `seatsTurnId`
3. `triggerSeatMap` fires `POST /build/flight-seats` with that `seatsTurnId` — fire-and-forget
4. SHERPA creates an ARGO cart item from `offer_id`, fetches seat map + extras in parallel, publishes `seat_map` block to AppSync under `seatsTurnId`
5. SketchCard's `seatWsRef` WebSocket receives the block, calls `setSeatMapBlock(block)`
6. Seat map panel renders inline below FlightCard

## DNA panel (Inspector > DNA tab)
- Archetype: user's matched archetype with confidence (e.g. "luxury_traveler 0.87")
- ArchetypeRoulette: animated spin on each new `archetype_match` event, `spinKey` incremented
- Top codons: highest-affinity codon IDs and scores from user DNA profile
- Rebase log: which codons were rebased (e.g. cheap intent → MIDR tier instead of BUDG)

## Cost / Turn Intel (Inspector > TURN tab)
- Tokens in / out: from Bedrock response
- Cache read %: `cache_read / (tokens_in + cache_read)` — 78% on turns 2+
- LLM cost + guardrail cost: from `costs` DDB table via `bedrock_cost_usd`
- All costs shown as USD per turn (5 decimal places)

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `ui/src/components/AgentPlayground.tsx` | Full playground | 4547 lines — chat, blocks, inspector |
| `ui/src/components/AgentPlayground.tsx:2098` | Inspector panel | DNA + Turn Intel + Match tabs |
| `ui/src/components/AgentPlayground.tsx:1718` | ArchetypeRoulette | Animated archetype spinner |

## Critical code
```typescript
// AgentPlayground.tsx — prompt cache hit percentage
const cacheHitPct = telemetry && telemetry.tokens_in > 0
  ? Math.round(
      (telemetry.cache_read / (telemetry.tokens_in + telemetry.cache_read)) * 100
    )
  : 0;

// match_score display (uses display-calibrated 0-100 int)
score={hotel.match_score ?? Math.min(100, Math.round(hotel.dna_boost * 100))}
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No automated UI tests | Manual via Amplify deploy |

## Gotchas
- `ReservedConcurrentExecutions: 1` on cognitive + dna-api means a second parallel turn from the playground gets a 429. This is demo-only and must be removed before production
- The Inspector tab shows match data only when `codon_contribs` is non-empty — if the shortlist returns candidates without codon contributions (e.g. scripted fallback), the MATCH tab will be empty
- Warm-up fires 5 parallel pings but does NOT guarantee all containers are warm before the first turn — Aurora cold connect still adds ~2-10s to first search
