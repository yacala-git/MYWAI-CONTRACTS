# SHERPA ↔ DNA Bridge

**One line:** SHERPA reads the user's TravelDNA from DynamoDB (300s cache) and calls the DNA API Lambda for signal writes and shortlist queries.

## What it does
- `load_user_dna` reads the user's codon affinities from DDB, caches the result 300s (matches Bedrock prompt-cache TTL)
- `fire_signal` writes a DNA signal (e.g. a hotel view or swipe) via Lambda invoke to the DNA API; also invalidates the local cache
- `get_config` reads ranking knobs from the DNA Config table (box16) — chromosome weights, tier ladders, archetypes, magic numbers
- DNA profile is injected into the Bedrock system prompt as a `<dna_profile>` block; Bedrock caches it across turns 2+ saving ~78% tokens
- Archetype matching uses weighted codon overlap (8 archetypes × codon list with weights) — falls back to hard-coded list if config service is unreachable

## Logical flow
1. Cognitive handler calls `load_user_dna(user_id)` → checks in-process `_DNA_CACHE`
2. Cache miss → reads `DNA_USERS_DNA_TABLE` and `DNA_AFFINITY_TABLE` from DDB (same account, direct IAM access)
3. `_match_archetype(codon_weights)` scores user against 8 archetypes from config service (or fallback)
4. `produce_intent` receives the DNA profile as a structured dict; injects into system prompt
5. After sketches are shown, `fire_signal(user_id, signal_type, ...)` invokes DNA API Lambda → deletes cache key
6. Next turn: cache miss forces fresh DDB read → picks up signal effects

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/dna_client.py` | `load_user_dna` | DDB read with 300s TTL in-process cache |
| `lambda/shared/dna_client.py` | `fire_signal` | Lambda invoke to DNA API + cache invalidation; `event_id` (idempotency) + `codons[]` (explicit-codon trusted branch) params |
| `lambda/shared/dna_client.py` | `fire_signals_batch` | Explicit codon IDs → `codons[]` (NOT text); text signals → text |
| `lambda/shared/dna_client.py` | `_STRONG_SIGNAL_RE` | Fast-path FALLBACK preference-verb gate; "I want/I need" dropped |
| `lambda/shared/dna_client.py` | `_match_archetype` | Weighted codon overlap → archetype name + confidence |
| `lambda/shared/dna_config.py` | `get_config` | Read-side helper for box16 config table (300s TTL) |
| `lambda/shared/dna_config.py` | `_normalize_archetype` | Normalises legacy flat codon lists to [{codon, weight}] |
| `lambda/tools/load_dna/` | tool contract | Tool contract for DNA load (used by tool_gates) |

## Critical code
```python
# dna_client.py — cache keeps Bedrock prompt-cache hit alive
_DNA_CACHE: dict[str, tuple[float, dict]] = {}  # user_id → (ts, data)
_DNA_CACHE_TTL = 300  # seconds — matches Bedrock 5-min cache window

def load_user_dna(user_id: str) -> dict:
    ts, data = _DNA_CACHE.get(user_id, (0.0, {}))
    if time.time() - ts < _DNA_CACHE_TTL:
        return data          # cache hit → same DNA block → Bedrock cache hit
    data = _read_from_ddb(user_id)
    _DNA_CACHE[user_id] = (time.time(), data)
    return data

def fire_signal(user_id, ...):
    _invoke_dna_api(...)
    _DNA_CACHE.pop(user_id, None)   # invalidate so next turn reads fresh
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/shared/test_dna_archetype.py` | Archetype scoring, weighted overlap |
| `tests/shared/test_dna_merge.py` | DNA merge logic for linked travellers |
| `tests/tools/test_load_dna_contract.py` | Tool contract validation |

## Durable-taste signal gate (write-side, 2026-07-03)
Durable TravelDNA is mutated ONLY by genuine preference statements — never by trip requests.
- **Rule:** durable taste fires only on a genuine preference verb (love/hate/adore/despise/loathe/like/enjoy/prefer/dislike/always want/always prefer/must-have/can't-stand/never want/would love/would never). "I want"/"I need"/"looking for"/"book me"/"find me" are trip logistics and write **ZERO** durable taste.
- **Primary path (Sonnet):** `produce_intent` emits a validated `durable_taste_phrases: list[str]` (Pydantic fail-safe → [], capped 6). `handler.py` fires ONE `chat.strong` signal **per phrase** (not the raw sentence). Zero phrases → zero writes. The field rides `_explicit_hotel_slots → user_constraints` (NOT persisted to `trip_slot`).
- **Fallback (fast-path only):** when `produce_intent` was skipped (committed-destination follow-up), the key is absent → the tightened `_STRONG_SIGNAL_RE` (`dna_client.py`) runs on the raw message and fires once. The old "I want/I need/definitely need/really want/absolutely" alternatives were REMOVED from that regex.
- **Per-turn idempotency:** each fire carries `event_id` (`turn_id#i` per phrase; `turn_id` for the fallback). `fire_signal(..., event_id=...)` forwards it as body `eventId`; DNA dedupes on `(userId, eventId)` (conditional put), so a retried turn applies once while distinct turns/phrases accumulate. Dedup is NEVER on text. Guarantee is **at-most-once**: the dedup marker is written before apply, so a crash mid-apply drops that one signal — an accepted tradeoff for a soft accumulating taste profile (double-writes are worse than a rare dropped low-weight signal).
- **Stroke contract:** `fire_signals_batch` POSTs explicit codon IDs as `codons=[{id,score}]` (DNA trusted branch), NOT as `text` — passing a codon ID as text embedded the literal string. `confidence → score`. The one remaining explicit-codon caller is the `signal_dna` tool (`tools/signal_dna/contract.py`, `req.confidence → score`).
- **User-declared facts write ZERO DNA (revised 2026-07-21).** The old user-facts fact→DNA dual-write (`shared/user_facts.py` `_FACT_TO_CODON` → `fire_signal`, score 1.0, `strength="high"`) was **DELETED** — it re-fired a max-strength `BUDG#*` signal every turn a mapped fact recurred, ratcheting the profile (creep study `docs/DNA_BUDGET_CREEP_STUDY_2026-07-20.md`). Facts are now permanent-attribute-only (`diet`/`allergy`/`special_need`/`seat_preference` + an identity/passport row), persisted to `…-user-facts` and fanned out DETERMINISTICALLY (`load_facts_constraints` → sketch-engine `dietary`/`wheelchair`, plus `seat_preference`/`nationality` exposure). Facts are NEVER re-injected into the intent prompt (pollution guard) and write **NO** DNA codon. DNA learns durable taste ONLY via the `chat.strong` path above. See SHERPA `CONTRACT.md` §11.

## Destination catalog consumption seam (FUTURE — NOT YET WIRED)
**Status: design-only.** The destination catalog (Aurora `destinations`, populated monthly by DNA box17 `destinations-sync` from the ATLAS enrichment pipeline — see `atlas_destinations.md` and `_dna.md`) exists, but SHERPA does not query it yet. `_discover_destinations_light` in `lambda/cognitive/sketch_engine.py` still reads the 11 hand-authored cities from `lambda/data/destinations.json`.

The planned (not-yet-built) swap:
- `_discover_destinations_light` will call a DNA-side `destination_codon_search(query_vector, k, geo_filter)` — structurally identical to hotel `codon_cosine_search` (`pg_client.py`) but ranking the `destinations` table by `codon_vector <=> :qvec::vector(320)`.
- It will pass the **same** 320-dim query vector that hotel codon search already builds from `taste_field.to_codon_hints()` + the user's `codon_affinities`. No translation layer: because the catalog is encoded with the shared `codon_index_map` + `build_codon_vector` (DNA side), cities live in the same codon space as hotels — zero adapter between the SHERPA query vector and the catalog.
- Geo prefilter (`ST_DWithin`) selects `geoname_id` only in a CTE, then `<=>` ranks in the outer query (the documented Data-API `vector(320)`-through-the-wire trap).

Until that change ships, nothing in SHERPA reads `destinations`. This entry marks the boundary so the catalog work and the picker swap stay decoupled.

## Child-preferences overlay — activities lane (2026-07-24)
A parent's DECLARED child interests steer ACTIVITY retrieval without profiling the minor (no child DNA; UK Children's Code). SHERPA sends declared interest tags projected to codons; DNA runs a SEPARATE floored per-interest lane over its own whole-city fetch (NOT the adult deck) and returns matches SHERPA interleaves. Endpoint: `POST /dna/engine/activities` (mode `activities`), the SAME call `_dna_activity_candidates` already makes.

**Request (additive on the existing activities body):**
- `child_overlays: [{ "interest": str, "codons": [codon_id, ...] }]` — one entry per DISTINCT owned child interest, projected SHERPA-side via `shared/child_interests.TAG_TO_CODONS` (distinctive codons only). **Present ONLY when the trip has ≥1 owned child with a mapped interest** — absent/empty → the key is OMITTED and the request + response are byte-identical to today (parity).
- `child_floor: float` (optional) — per-interest abstain floor override. SHERPA does NOT send it today; DNA reads its box16 `MAGIC_NUMBERS` `CHILD_INTEREST_FLOOR` knob (default 0.4).

**Response (additive):**
- `child_matches: [{ "interest": str, "activities": [ { ...full activity row..., "score": float }, ... ] }]` — per-interest, best-first. Each activity is the SAME dict shape `results` carries (`provider_id`, `provider_ids` [{provider, external_id}] LENS join key, `title`, `category`, `duration_minutes`, `codons`/`codons_weighted`, geo, `images_s3`, `usps`, …) PLUS an additive `score` ∈ [0,1] (the child-interest cosine, post-floor). Child rows do NOT carry `match_01`/`match_score` — that stretch is adult-deck-relative and meaningless across lanes; use `score` only for a child-side transform, NEVER compared to adult match%. Key is present ONLY when `child_overlays` was non-empty. An interest with nothing ≥ floor returns `activities: []` (honest abstain, distinct from "no match").

**SHERPA-side consumption** (`sketch_engine.py`, see `sherpa_sketch_engine.md`): child-ONLY matches (provider_id not in the adult deck) are merged into the candidate rows BEFORE LENS pricing so they price identically and RENDER; the priced pool is then interleaved (tag `why="child_interest:<interest>"`, promote, dedupe, cap). Threaded through BOTH the initial turn (`run_sketch_engine`) and the deferred hotel-compose op (`compose_activities_for_hotel`, via `child_ids` persisted on `compose_seed`) so the demo's visible day-plan steers identically.

## Gotchas
- Three env vars are required with NO defaults: `DNA_USERS_DNA_TABLE`, `DNA_AFFINITY_TABLE`, `DNA_API_FUNCTION_NAME`. Missing any one silently breaks DNA loading at cold start
- `dna_config.py` requires `DNA_CONFIG_TABLE` — if unset, `get_config` always returns None (config reads silently no-op, defaults apply)
- Archetype list has 8 specific keys: `business_pro`, `culture_urban`, `family_vacation`, `wellness_relax`, `budget_packer`, `adventure_nature`, `luxury_traveler`, `social_party`. Do not assume others exist
- 300s TTL is not a coincidence — it matches the Bedrock prompt-cache TTL. If the DNA cache expires before Bedrock's cache, the system prompt changes and Bedrock must re-cache
