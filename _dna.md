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
| `lambda/dna-api/handler.py` | `_signal_already_processed` | Per-turn idempotency: conditional put on `(userId, eventId)` in `SIGNAL_DEDUP_TABLE` (fail-open when unset) |
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

## Text-signal codon mapping — explicit tier/budget override (write-side, 2026-07-03)
The chat/planner/media text-signal path (`handle_signals_http` → `_process_signal_message` → `_resolve_text_to_deltas`) previously embedded the WHOLE sentence and took the nearest-codon guess, so "I want a 5-star hotel" wrote STAY.TIER#MIDR / BUDG#MDST (mid-tier from a 5★ ask — backwards). Fix (all text-signal callers):
- **Deterministic parse before embedding.** `_parse_explicit_tier_budget` mirrors the canonical `shared/codon_utils.tier_from_stars` (≤2★→BUDG, 3★→MIDR, 4★→UPSC, 5★→LUXR) + `budg_from_stay_tier` neighbour; "cheap/budget/affordable/…"→BUDG#ECON(+THRF). Matches "5-star","5 star","five-star","3-star", etc.
- **Override, not blend.** When an explicit token is present the parsed codon is emitted authoritatively; embedding-NN hits on `STAY.TIER#` / `BUDG#` are SUPPRESSED (a literal user word never loses to a cosine guess). Other sub-axes (STAY.CAT/STAY.STY) are untouched.
- **Span-strip.** The matched token span is removed from the text before embedding so the residual NN search isn't skewed (whole-token chunk → NN search skipped entirely).
- **Idempotency.** `_signal_already_processed(userId, eventId)` conditional-puts to `SIGNAL_DEDUP_TABLE` (composite key `userId` HASH + `eventId` RANGE, TTL attr `expiresAt`) before applying; a duplicate returns `{"note":"duplicate_event"}`. Fail-open when the env var is unset. Keyed on eventId, never text. Semantics are **at-most-once**: the marker is written BEFORE the signal is applied, so a crash mid-apply drops that one signal — accepted tradeoff for a soft, continuously-accumulating taste profile (a single missed low-weight signal is immaterial; a double-write is worse). Composite key is load-bearing — a sole-PK table would dedupe two distinct eventIds for the same user.
- **HTTP boundary.** `handle_signals_http` now accepts text OR `codons[]` OR `entities[]` (was `text` required) so stroke/Box6 explicit-codon signals reach the trusted branch instead of being forced through `text` (embedded).

## Destination catalog (read-side; DNA-scored cities)
- Aurora `destinations` table mirrors `hotels.geo` + `hotels.codon_vector` so a city ranks in the **same shared 320-dim codon space** as hotels (`tools/schema_migration.sql`). PK `geoname_id` (== `ti-cities.cityId`). Cosine arm only.
- `pg_client.upsert_destination(row, index_map)` / `upsert_destination_batch(rows, index_map)` — caller passes `get_codon_index_map()` ONCE. Every emitted codon is validated as already-registered and **fails loud** on any unknown ID; the map is NEVER mutated (`ensure_codon_index_map` is forbidden — would mint a permanent dimension shared with hotels). `len(index_map) ≤ 320` hard-gated. NULL-geo / NULL-vector guarded by `COALESCE` so a partial re-sync never clobbers.
- `destinations-sync` Lambda (box17, monthly — EventBridge `cron(0 2 17 * ? *)`, 17th @ 02:00 UTC, one day after the ATLAS enrich scanner on the 16th) Scans the ATLAS `mywai-prod-ti-city-enrichment` table → `BatchGetItem` `ti-cities` by `(countryCode, cityId)` for lat/lng (note: ti-cities stores longitude as `lng`) → upserts changed rows. Idempotency via `destinations.source_version` (compares item `source_version`/`updated_at`; skips unchanged). Per-row errors are isolated (batch failure → single-row retry), counted, and surfaced; `errors>0` in the return payload is the alarm signal. RDS Data API path — no VPC. Dedicated `destinations-sync-role` (the `ti-cities` grant on dna-api is not reusable).

## Gotchas
- box6 is the only writer to `hotels.codons` — direct writes bypass chromosome caps and corrupt the corpus
- destination sync must call `get_codon_index_map()` (read-only), never `ensure_codon_index_map()` — minting a dimension can push the shared map past 320 and silently corrupt every hotel vector
- `provider_id` in Aurora is the same value as `master_hotel_id` in DDB — 32-char hex, never use the 12-char display form in SQL
- Deploy order: `./deploy.sh all` first in mywai-dna so box12 exports `AURORA_PROXY_ENDPOINT` and other env vars before box4/10 are deployed
- Stars are a hard SQL WHERE filter only when user explicitly states them; `STAY.TIER#*` codons are ranking-only — never add a TIER WHERE clause
- STAY.TIER codons are collapsed to a single signed dimension in `build_codon_vector()` — all four map to `STAY.TIER#LUXR` index (258) with values BUDG=-0.5, MIDR=-0.33, UPSC=+0.5, LUXR=+1.0. Luxury users actively repel budget hotels (negative dot product). See `dna_hotel_search.md` for details.
