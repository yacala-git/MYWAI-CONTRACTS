# DNA (TravelDNA Engine)

**One line:** Builds a personal travel preference profile (TravelDNA) for each user from image swipes and signals, then uses it to rank hotels via Aurora PostgreSQL hybrid search.

## What it does
- Maps user signals (image swipes, ratings, searches) to codons — atomic preference units like "beach", "luxury", "adventure"
- Groups codons into chromosomes — thematic clusters like MOOD, STAY, FOOD, DEST (15 chromosomes total)
- Stores per-user codon affinities (0.0–1.0) in DynamoDB; hotel codons in Aurora PostgreSQL
- Runs hybrid search (BM25 text + pgvector cosine + RRF fusion) against Aurora to find matching hotels
- Serves SHERPA: the shortlist Lambda returns ranked candidates with match scores for sketch assembly

## Codon/Chromosome model
- **Chromosome**: thematic travel dimension (STAY, BUDG, MOOD, FOOD, ACTV, DEST, CULT, WELL, SOC, PURP, TIME, SHOP, SUST, SAFE, TRAN)
- **Codon**: atomic preference unit within a chromosome (e.g. `MOOD#ADVN`, `STAY.STY#BOUT`, `DEST#BEAC`)
- **v2 STAY axis**: emits up to 5 codons: `STAY.CAT#*` (property type) + `STAY.TIER#*` (price tier) + `STAY.STY#*` (0–3 style overlays)
- Each hotel gets up to 12 codons total (bucketed by chromosome cap — STAY:5, FOOD:3, ACTV:3, rest:1-2)

## Aurora schema (key tables)
| Table | Purpose |
|-------|---------|
| `hotels` | Master hotel table: provider_id, city, stars, amenities[], codon_vector, geo |
| `hotels_segments` | 384-dim vibe vectors (BM25+HNSW search target) |
| `hotels_segments_image` | 512-dim image vectors |
| `codons_segments` | Text embeddings for user DNA codons |
| `amenities_registry` | 234 approved amenity codes with codon category mappings |
| `landmarks_registry` | 409 Rome+Paris POIs for geo distance queries |

## 16 Boxes
| Box | Purpose |
|-----|---------|
| 0 | SSM seed |
| 1 | DynamoDB control tables |
| 2 | IAM roles |
| 3 | DynamoDB user tables |
| 4 | DNA API Lambda (core brain) |
| 5 | Media processor + pre-signer |
| 6 | Vision AI codon labeler (box6 is the ONLY writer to hotel codons) |
| 7 | Embedding pipeline (ECS) |
| 8 | FAISS router — **retired** |
| 9 | DNA profile refiner |
| 10 | Candidate shortlist Lambda |
| 11 | Analytics event writer |
| 12 | Aurora PostgreSQL Serverless v2 + RDS Proxy |
| 13 | Product type ingestion |
| 14 | Auth API + MFA + Cognito triggers |
| 15 | Batch indexing ingest |
| 16 | Config Admin CRUD + audit (box16) |

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/pg_client.py` | `hybrid_hotel_search` | 4-lane RRF: BM25 + vector + image + codon |
| `lambda/dna-shortlist/handler.py` | `lambda_handler` | Shortlist entry: codon or BM25 routing, scoring, relaxation |
| `lambda/dna-api/handler.py` | `lambda_handler` | Core brain: DNA read/write, codon mapping, shortlist proxy |
| `lambda/dna-api/handler.py` | `_resolve_text_to_deltas` | Text-signal → codon deltas. Parses explicit tier/star/budget tokens DETERMINISTICALLY before embedding; the parsed codon overrides any cosine-NN guess on STAY.TIER/BUDG |
| `lambda/dna-api/handler.py` | `_parse_explicit_tier_budget` | "5-star"/"five-star"/"3-star"→`tier_from_stars`→STAY.TIER (+BUDG neighbour); "cheap/budget/affordable"→BUDG#ECON(+THRF). Returns (parsed, residual_text, suppress_prefixes) |
| `lambda/dna-api/handler.py` | `_signal_already_processed` | Per-turn idempotency: conditional put on `(userId, eventId)` in `SIGNAL_DEDUP_TABLE` (fail-open when unset); claim is released by `_release_signal_claim` only on `DnaWriteConflictError` |
| `lambda/dna-api/handler.py` | `_update_user_dna` / `_run_versioned_dna_write` | The ONLY way DNA is written: one targeted, version-conditioned `UpdateItem`, ≤3 attempts with re-read + recompute (see "DNA write concurrency") |
| `lambda/dna-api/handler.py` | `_set_user_fields` | Targeted SET of non-DNA profile fields on the dna-users row; refuses `dna`/`version`/`summary`, never bumps `version` |
| `lambda/dna-labeler/handler.py` | `_handle_product_upsert` | Only writer to hotel codons via box6 |
| `lambda/dna-config-admin/handler.py` | `lambda_handler` | box16 CRUD for all ranking knobs |

## Critical code
```python
# pg_client.py — codon path triggered when ≥2 intent codons and no geo filter
# handler.py:577
def _is_sparse_query(intent_codons, has_geo):
    return len(set(intent_codons)) >= 2 and not has_geo

# When True → codon cosine search (320-dim L2-normalised vector, COSINE wins DNA-merged)
# When False → BM25 hybrid (user_message text + HNSW vector + RRF)
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/integration/test_t3_retrieval.py` | Codon retrieval quality (4 trouble queries) |
| `tests/integration/test_t2_corpus_invariants.py` | Corpus health: cap violations, null persona |
| `tests/test_shortlist_core.py` | Shortlist scoring logic |
| `tests/test_codon_vector.py` | Codon vector build + normalisation |
| `tests/test_dna_write_versioning.py` | DNA write concurrency (moto): single-signal parity goldens vs the pre-versioning handler, interleaved writers, retry/exhaustion, claim-first dedup + release, affinity-once, SQS partial batch failure, 409, decay/reset/profile/onboarding targeted writes |

## Text-signal codon mapping — explicit tier/budget override (write-side, 2026-07-03)
The chat/planner/media text-signal path (`handle_signals_http` → `_process_signal_message` → `_resolve_text_to_deltas`) previously embedded the WHOLE sentence and took the nearest-codon guess, so "I want a 5-star hotel" wrote STAY.TIER#MIDR / BUDG#MDST (mid-tier from a 5★ ask — backwards). Fix (all text-signal callers):
- **Deterministic parse before embedding.** `_parse_explicit_tier_budget` mirrors the canonical `shared/codon_utils.tier_from_stars` (≤2★→BUDG, 3★→MIDR, 4★→UPSC, 5★→LUXR) + `budg_from_stay_tier` neighbour; "cheap/budget/affordable/…"→BUDG#ECON(+THRF). Matches "5-star","5 star","five-star","3-star", etc.
- **Override, not blend.** When an explicit token is present the parsed codon is emitted authoritatively; embedding-NN hits on `STAY.TIER#` / `BUDG#` are SUPPRESSED (a literal user word never loses to a cosine guess). Other sub-axes (STAY.CAT/STAY.STY) are untouched.
- **Span-strip.** The matched token span is removed from the text before embedding so the residual NN search isn't skewed (whole-token chunk → NN search skipped entirely).
- **Idempotency.** `_signal_already_processed(userId, eventId)` conditional-puts to `SIGNAL_DEDUP_TABLE` (composite key `userId` HASH + `eventId` RANGE, TTL attr `expiresAt`) before applying; a duplicate returns `{"note":"duplicate_event"}`. Fail-open when the env var is unset. Keyed on eventId, never text. Semantics are **at-most-once**: the marker is written BEFORE the signal is applied, so a crash mid-apply drops that one signal — accepted tradeoff for a soft, continuously-accumulating taste profile (a single missed low-weight signal is immaterial; a double-write is worse). One exception (2026-09-15): when the DNA write loses every versioned retry (`DnaWriteConflictError`, nothing written) the claim is DELETED so the SQS redelivery applies — see "DNA write concurrency". Composite key is load-bearing — a sole-PK table would dedupe two distinct eventIds for the same user.
- **HTTP boundary.** `handle_signals_http` now accepts text OR `codons[]` OR `entities[]` (was `text` required) so stroke/Box6 explicit-codon signals reach the trusted branch instead of being forced through `text` (embedded).

## DNA write concurrency — versioned conditional writes (DNA ticket 0, 2026-09-15)
**Why.** Every DNA writer used to get the whole `dna-users` row, change it in memory and `put_item` the whole row back, unconditioned. Two invocations for one user (two SQS pollers, or SQS + an HTTP GET/profile edit) interleaved and the later put erased the earlier one's deltas, AND rewrote `persona`/`vectors_v2`/`synthesis`/`matchCeiling` from its stale snapshot. Design: `mywai-demo/.scratch/discover-share/issues/IDGO-RETRACT-DESIGN.md` (TICKET 0); build + guru record: `DNA-T0-BUILD.md`, `DNA-T0-GURU.md` in the same folder.

**1. The versioned write (`_update_user_dna`, `_run_versioned_dna_write`).**
- ONE `UpdateItem` that SETs only the attributes the writer owns plus `version`, and REMOVEs only what it names. No DNA writer does a whole-row `put_item` any more (`_put_user` remains only for a test harness).
- Condition, from the row AS READ:

| Row read | ConditionExpression | Version written |
|---|---|---|
| no row | `attribute_not_exists(userId)` | 1 (row created) |
| legacy row, no `version` | `attribute_not_exists(version)` | 1 |
| versioned row, `version = v` | `version = :v` | v+1 |

- On `ConditionalCheckFailedException`: sleep 50–150 ms (random), re-read (strongly consistent from attempt 2), RECOMPUTE from the fresh row, retry — max 3 attempts. Any other error propagates unretried. Exhaustion logs `dna_write_conflict_exhausted` and raises `DnaWriteConflictError` — nothing was written.
- `version` now increments on every DNA write (it was a constant 1). `GET /dna/{userId}` returns it; no consumer relied on the constant (grep of sherpa + both UIs, 2026-09-15).

**2. What each writer owns.**

| Writer | SET | REMOVE | On exhaustion |
|---|---|---|---|
| `handle_mutations` (SQS `dna_signal`, HTTP `POST /dna/{userId}/mutations`, onboarding bubbles, media undo) | `dna`, `summary`, `updatedAt`, `version`; `+refinerStats` only when a refiner action applied; `+lastDecayAt` only when creating the row | — | raises (see callers below) |
| `handle_get` scaffold / decay / missing-summary (`GET /dna/{userId}`) | scaffold: `dna`,`summary`,`lastDecayAt`; decay: `dna`,`lastDecayAt`; summary: `summary`; always `updatedAt`,`version` | — | the GET still returns 200 with the computed view, UNPERSISTED; `lastDecayAt` did not advance, so the next GET decays the whole gap |
| `handle_reset_dna` (`POST /dna/reset`) | `dna={}`, `summary`, `updatedAt`, `version`; `refinerStats` reset if present | `synthesis`, `persona` | 500 `Reset failed`, before any mutations/media are deleted |

Scoring is unchanged: the old loop is `_compute_mutations` verbatim; a single uncontended signal produces byte-identical `dna`/`summary`/audit rows (parity goldens in `tests/test_dna_write_versioning.py`). Audit rows (`dna-mutations`) are written only after the DNA write lands.

**3. Non-DNA profile writers — targeted field sets (`_set_user_fields`).** They never write `dna`, `version` or `summary` and never bump `version` (so they neither conflict with nor revert — nor roll back the version of — a concurrent DNA write).

| Handler | Owns (SET) | Notes |
|---|---|---|
| `handle_update_user_profile` (dna-users part) | `firstName`, `lastName`, `title`, `preferredLocale`, `passportNumber` (each only if truthy), `identity.dob` (key-level; `identity={dob}` when identity is missing/not a map) | only if the dna-users row exists (`attribute_exists(userId)`); no `updatedAt` stamp (as before). The auth-users `USERS_TABLE` put in the same handler is unchanged (different table, no DNA). |
| `handle_post_onboarding` | `identity` (whole map, replaced as before), `identityCompleted=true`, `status.preferencesCompleted` (key-level; other `status` keys kept), `updatedAt` | creates the row if missing. Onboarding seeds DNA ONLY via `_apply_bubble_preferences` → `handle_mutations` (the versioned write). A brand-new row with no bubble selections still has no `dna` attribute (pre-existing; `handle_get` KeyErrors on it). |

Both handlers re-raise `DnaWriteConflictError` (→ 409) instead of their blanket `except → 401`.

**4. Signal dedup — claim first, release on conflict** (`_process_signal_message`).
1. `_signal_already_processed` conditional-puts the `(userId, eventId)` claim BEFORE any work → of two concurrent duplicate deliveries only one applies.
2. Entity affinity (`affinity_store.update_affinity`, additive) is STAGED and applied only AFTER `handle_mutations` returns, so a redelivery can never add it twice. If that affinity write fails after the DNA write: logged `affinity_update_failed_after_dna_write`, response `entities_applied=[]`, claim kept (the audit row still lists `entityDeltas`).
3. On `DnaWriteConflictError`: `_release_signal_claim` deletes the claim (never raises; failure logged `signal_dedup_release_failed` = that one signal lost), then re-raises.
Residual (unchanged): a crash between claim and write leaves the claim → that signal is dropped (at-most-once).
IAM: dna-api role needs `GetItem`/`PutItem`/`DeleteItem` on `signal-dedup` — live role verified allowed 2026-09-15 (`simulate-principal-policy`).

**5. Failure surfacing by caller.**

| Caller | On `DnaWriteConflictError` |
|---|---|
| SQS `dna_signal` (`_from_sqs`) | record reported in `batchItemFailures` → SQS redelivers it alone; claim released. ONLY this error is reported — every other per-record failure is still logged (`sqs_process_failed`) and dropped, so a poison message cannot loop (queue has no DLQ, 14-day retention). |
| HTTP `POST /dna/{userId}/mutations`, `POST /dna/signals/v1?sync=true`, profile update, onboarding | `409 {"error":"dna_write_conflict","retryable":true}` (`lambda_handler`). The refiner worker treats ≥300 as failure and redrives. |
| Media undo (`DELETE /dna/media/{mediaId}`) | `500 {"error":"delete_media_failed"}` from its existing handler — the undo was not applied and the mutation row + media are NOT deleted, so the user can retry. |
| Onboarding bubbles (`_apply_bubble_preferences`) | swallowed by its existing `except` (logged `apply_bubble_preferences_failed`); the preference diff is not re-applied on retry — known gap. |
| `GET /dna/{userId}` decay | 200, unpersisted (self-healing). |
| Reset | 500. |

**6. Queue config (`infra/box4-dna-api.yaml`).**
- `DnaSignalsEventSource`: `FunctionResponseTypes: [ReportBatchItemFailures]` — required, otherwise `batchItemFailures` is ignored and a conflict is silently dropped. `_from_sqs` always returns `{"ok", "processed", "failed", "batchItemFailures": [...]}`.
- `DnaSignalsQueue.VisibilityTimeout: 540` (was 60) = 6× the function `Timeout: 90`, per AWS SQS event-source guidance (reserved concurrency 5 → throttling). At 60 s < 90 s a slow batch reappeared and a second container processed the same records in parallel. Trade-off: a record reported in `batchItemFailures` is redelivered ~9 min later.
- **`DnaSignalsDeadLetterQueue`** (`infra/box4-dna-api.yaml:524-537`): `DnaSignalsQueue`'s `RedrivePolicy` points at it — `maxReceiveCount: 5` (a record in `batchItemFailures`, or a failed/timed-out invocation, is redelivered every 540 s and moves to the DLQ after 5 receives, ~45 min). `MessageRetentionPeriod: 1209600` (14 days, the SQS max — cannot exceed the source queue's own 14-day retention). `SqsManagedSseEnabled: true`, matching the source queue's encryption. A standard queue keeps the original enqueue time on redrive, so a DLQ'd record still ages out ~13.9 days after its original enqueue, not 14 days after arriving in the DLQ.

## Signal source: `search.tapped` (mywai-social AGORA search, PLANNED — T-S10 not yet built)
Owner-ruled signal for a tile tap on the AGORA search results page (`mywai-social` design:
`mywai-demo/.scratch/discover-share/issues/SEARCH-MIRROR-DESIGN-v3.md` §6.2, ticket T-S10). DNA
needs no new code for this — `_source_weight()` (`dna-api/handler.py:877-900`) already resolves
any source by exact match against the live `MUTATION_WEIGHTS_TABLE` (`refs.lookup("weights", source)`),
falling back to a `group.` prefix match, then `1.0`. Only a weight ROW needs to exist.
- **Weight 0.2, exact-match key `search.tapped`.** ASSUMPTION/PLANNED: not present in the checked-in
  bulk seed script (`tools/seed-dna-mutation-weights.sh`, verified 2026-09-17 — no `search.tapped`
  entry) or anywhere else in this repo's code/config. The design doc (§6.2) asserts it is already a
  live DDB row seeded out-of-band (e.g. via box16 config-admin CRUD); this contract cannot confirm a
  live DynamoDB value from the checked-in source, so treat the weight as PLANNED until T-S10 ships
  and the row is confirmed live (`GET` via box16 or a `describe`/`get-item` on the mutation-weights
  table).
- **Explicit codons only, never text.** The message is `{"kind":"dna_signal","userId":sub,"body":
  {"source":"search.tapped","direction":"like","strength":"medium","codons":[from the tapped post's
  `post_codons`, capped],"eventId":"srchtap:{sub}:{post_id}","timestampMs"}}` — no `text` key, so it
  never enters the Haiku taste-codon embedding path; codons are trusted as-is like any other
  explicit-codon signal.
- **Never retracted.** No opposite transition exists for a tap (unlike a like/unlike pair), so there
  is deliberately no retract path for this source — accepted (guru-agreed) because DNA's claim-first
  `(userId, eventId)` dedup absorbs a duplicate/re-tap.
- **Deterministic eventId `srchtap:{sub}:{post_id}`.** One tap per `(user, post)` ever produces a
  DNA write — a second tap on the same post hits `_signal_already_processed`'s conditional put and is
  a no-op (see "Text-signal codon mapping" idempotency above, and the claim-first dedup in "DNA write
  concurrency" §4). The `mywai-social` side additionally short-circuits before sending: a
  `SRCH#<sub>/TAP#<post_id>` claim row in its own `SeenTable` prevents even attempting a duplicate
  send.

## Destination catalog (read-side; DNA-scored cities)
- Aurora `destinations` table mirrors `hotels.geo` + `hotels.codon_vector` so a city ranks in the **same shared 320-dim codon space** as hotels (`tools/schema_migration.sql`). PK `geoname_id` (== `ti-cities.cityId`). Cosine arm only.
- `pg_client.upsert_destination(row, index_map)` / `upsert_destination_batch(rows, index_map)` — caller passes `get_codon_index_map()` ONCE. Every emitted codon is validated as already-registered and **fails loud** on any unknown ID; the map is NEVER mutated (`ensure_codon_index_map` is forbidden — would mint a permanent dimension shared with hotels). `len(index_map) ≤ 320` hard-gated. NULL-geo / NULL-vector guarded by `COALESCE` so a partial re-sync never clobbers.
- `destinations-sync` Lambda (box17, monthly — EventBridge `cron(0 2 17 * ? *)`, 17th @ 02:00 UTC, one day after the ATLAS enrich scanner on the 16th) Scans the ATLAS `mywai-prod-ti-city-enrichment` table → `BatchGetItem` `ti-cities` by `(countryCode, cityId)` for lat/lng (note: ti-cities stores longitude as `lng`) → upserts changed rows. Idempotency via `destinations.source_version` (compares item `source_version`/`updated_at`; skips unchanged). Per-row errors are isolated (batch failure → single-row retry), counted, and surfaced; `errors>0` in the return payload is the alarm signal. RDS Data API path — no VPC. Dedicated `destinations-sync-role` (the `ti-cities` grant on dna-api is not reusable).

## Similar travellers (`POST /dna/engine/similar-travelers`, box4 `dna-api`)
"Travellers like you": an IDF-weighted cosine of the caller's DNA against the seed-traveller pool. Code: `handle_similar_travelers` in `lambda/dna-api/handler.py`. Tests: `tests/test_similar_travelers.py`. Design: `mywai-demo/.scratch/discover-share/issues/PROFILE-MATCH-DESIGN.md`.
- **Auth.** JWT required; the viewer is ALWAYS the JWT subject (`_resolve_user_id`) — no body `userId` is read. No JWT → 401.
- **Pool.** The seed subs in SSM `/mywai-agora/prod/seed-subs` (cached 300 s), minus the viewer. Every call loads the whole pool, because the IDF is computed over it.
- **Floors.** A profile counts only if it has ≥ `SIMILAR_CARD_FLOOR` (8) graded codons (> 0.01) AND an L1 mass ≥ `SIMILAR_L1_FLOOR` (6.0) — applied to the viewer and to each candidate. A pair needs ≥ `SIMILAR_MIN_SHARED` (2) shared codons. IDF = `log(1 + n / max(df, SIMILAR_DF_FLOOR=5))`.
- **Buckets** come from the pair's absolute sim, never its rank: ≥ 0.20 → 1 "Strong match", ≥ 0.08 → 2 "Good match", else 3 "Some overlap".
- **Taste switch (`dna_public`), server-enforced.** Before any candidate DNA is loaded or the IDF is computed, the pool is filtered against AGORA's "show my taste" switch: a live, uncached `BatchGetItem` on `AGORA_POSTS_TABLE` (chunks of 100, one bounded retry of `UnprocessedKeys`), keys `{pk: "FOL#{sub}", sk: "PROFILE"}` (mirrors AGORA's `_profile_key`), `ProjectionExpression="pk, sk, dna_public"`, `ConsistentRead=True`. Per candidate: `dna_public is False` → dropped (their choice); attribute absent on a returned row → kept (mirrors AGORA's own default-ON, `is not False`); row absent, or still unresolved after the retry → dropped, fail-closed (this deliberately differs from AGORA's own default-public-on-absent-row rule, so a drifted key layout can't silently open the pool); any raise (env unset, AccessDenied, timeout) → the WHOLE pool is hidden, `{"results": [], "hidden": true, "reason": "unavailable"}`, never served partial. No cache — a cache would reopen the fail-open window. List mode and target mode share this one filter. Design: `mywai-demo/.scratch/discover-share/issues/DNA-PUBLIC-DECISION.md`. This is a cross-stack READ into AGORA's `mywai-agora-prod-posts` table — see `mywai-social/contracts/social-block.md` for the AGORA-side contract on `FOL#{sub}/PROFILE` and `dna_public`.

**Request body**
| Body | Mode |
|---|---|
| `{}`, no body, a non-dict JSON body, or any dict without `target` | **List mode** — the top `SIMILAR_MAX_RESULTS` rows by sim (code default 6; box4 sets 12) |
| `{"target": "<16 lowercase hex>"}` | **Target mode** — only that traveller's row, whatever their rank |
| `target` present but not a string matching `^[0-9a-f]{16}$` (incl. `null`, uppercase, trailing newline) | `400 {"error": "invalid target"}`, checked before any read |

The `target` is the traveller's opaque handle, `sha256(sub)[:16]` — the same value as a list row's `handle` and AGORA's `_author_handle`. A sub never crosses the API in either direction.

**Response** (always 200 except the 400/401 above): `{"results": [row, ...], "hidden": false}` or `{"results": [], "hidden": true, "reason": "<reason>"}`. A row is exactly `{handle, founding, bucket, bucketLabel, reason, sharedTags (≤3 names), sharedCount}` — no raw score, no codon id, no sub.

| Reason | List mode | Target mode |
|---|---|---|
| `viewer_below_mass_floor`, `no_candidates`, `unavailable` (any DDB/codon-load error, fail-open) | yes | yes (they describe the viewer only) |
| `no_qualifying_candidates`, `no_shared_taste` | yes | never — collapses to `no_match` |
| `no_match` | never | the target is not a seed, has hidden their taste (`dna_public=false` or unreadable), is below the mass floor, shares < 2 codons, or is the viewer |

- **Same score, byte for byte.** Target mode scores the same pool with the same IDF and builds the row with the same function (`_similar_row`) as list mode; it only skips the truncation. So a target row equals that traveller's row in an untrimmed list.
- **No membership oracle.** Every target-side miss returns the one `no_match` reason. The granular cause is logged server-side only (`similar_travelers_target`, field `target_reason`: `matched` / `self` / `not_in_pool` / `not_public` / `below_mass_floor` / `below_min_shared`). `not_public` covers a private target, a missing PROFILE row, and an unreadable one alike — a private target and a target that was never a seed are indistinguishable to the caller, both `no_match` (the "hid-their-taste oracle" rule AGORA applies with its own empty-200).
- **T1 resolved (was the known gap).** This endpoint now reads AGORA's `dna_public` switch server-side, in both modes (see "Taste switch" above) — no longer a client-side-only gate. The demo profile page's client gate stays as defence in depth.

## Gotchas
- box6 is the only writer to `hotels.codons` — direct writes bypass chromosome caps and corrupt the corpus
- Never `put_item` a whole `dna-users` row, and never SET `dna`/`version`/`summary` outside `_update_user_dna` — a stale whole-row write reverts concurrent DNA deltas and rolls `version` back (ABA), defeating every versioned writer. Non-DNA fields go through `_set_user_fields`.
- destination sync must call `get_codon_index_map()` (read-only), never `ensure_codon_index_map()` — minting a dimension can push the shared map past 320 and silently corrupt every hotel vector
- `provider_id` in Aurora is the same value as `master_hotel_id` in DDB — 32-char hex, never use the 12-char display form in SQL
- Deploy order: `./deploy.sh all` first in mywai-dna so box12 exports `AURORA_PROXY_ENDPOINT` and other env vars before box4/10 are deployed
- Stars are a hard SQL WHERE filter only when user explicitly states them; `STAY.TIER#*` codons are ranking-only — never add a TIER WHERE clause
- STAY.TIER codons are collapsed to a single signed dimension in `build_codon_vector()` — all four map to `STAY.TIER#LUXR` index (258) with values BUDG=-0.5, MIDR=-0.33, UPSC=+0.5, LUXR=+1.0. Luxury users actively repel budget hotels (negative dot product). See `dna_hotel_search.md` for details.
