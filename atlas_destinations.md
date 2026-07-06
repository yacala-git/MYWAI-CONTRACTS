# ATLAS Destinations Enrichment (ETL contract)

**One line:** A monthly ETL builds a DNA-scorable destination catalog — gates cities on a Wikivoyage article, derives vibe codons from the `ti-landmarks` POI histogram (in the SAME 320-dim codon space as hotels), enriches text/climate/images, writes `mywai-prod-ti-city-enrichment` (DDB), and the DNA side syncs it into Aurora `destinations` one day later.

**Owners:** ATLAS (`mywai-travel-info-insight`) = enrichment ETL + write-side DDB · DNA (`mywai-dna`) = Aurora + sync. Single account `252842590549`, region `eu-west-3`.

## Pipeline (monthly)
```
ti-cities ETL (15th @ 2am)
        ↓
scanner   (16th @ 2am)  box3  ti-box3-etl DestinationsEnrichScannerRule, cron(0 2 16 * ? *)
   gate + enqueue one SQS msg per promotable city
        ↓
SQS  mywai-prod-ti-destinations-queue  (box8; visibility 960s; redrive → DLQ after maxReceiveCount=3)
        ↓
worker (container, box8)  mywai-prod-ti-destinations-worker-eu-west-3
   stages: wikivoyage → vibe_codons → blurb → climate → images → decode_destination → ddb_writer
        ↓
DDB  mywai-prod-ti-city-enrichment   (PK countryCode / SK cityId)
        ↓
destinations-sync (17th @ 2am)  DNA box17, cron(0 2 17 * ? *)
        ↓
Aurora  destinations  (read-side; see _dna.md + dna_hotel_search.md)
```

## The gate (relevance)
- A city is promotable iff Wikidata has BOTH a `P1566` (GeoNames ID, bridges to `ti-cities.cityId`) AND an `enwikivoyage` sitelink. ~20–35k cities post-gate.
- Backfill: Wikidata + Wikivoyage **dumps** parsed in a Glue Python-Shell job (`glue/destinations_dump_parse.py`, deps `mwxml`+`lxml`) → promotable manifest in S3 (`PROMOTABLE_MANIFEST_KEY`). Incremental: live SPARQL / MediaWiki API with a descriptive User-Agent.
- No Wikivoyage article → not enqueued. Vibe is grounded in real POI + text data, NEVER in hotel inventory and never in a single hero photo.

## Worker stages
| Stage | Source | Writes (DDB attrs) |
|-------|--------|--------------------|
| `wikivoyage` | MediaWiki article (or dump) | `source_text` (≤2 KB intro), `wikivoyage`, `attribution` (CC-BY-SA); full multi-section text → S3 raw |
| `vibe_codons` | `ti-landmarks` feature-code histogram | `vibe_codons[]`, `vibe_weights{}` |
| `blurb` | Haiku (`DESTINATIONS_BEDROCK_MODEL_ID`, `eu.anthropic.claude-haiku-4-5-…`) grounded in histogram + Wikivoyage | `blurb` (≤240 chars) |
| `climate` | Open-Meteo archive (ported from SHERPA seasonality) | `climate{jan..dec}` |
| `images` | fork of `etl-image-worker` → Commons P18 → Pillow variants → S3 media | `hero_image` (S3 key), `image_credit`, `attribution` |
| `decode_destination` | mirror of the MobileNet harness (shredder); scene tag + P18 reject filter | (fallback scene tag / reject only) |
| `ddb_writer` | assembles + stamps | `confidence`, `source_version`, `updated_at`, `partial`, `failed_stages[]` |

- Images are gated by `DESTINATIONS_IMAGES_ENABLED` (default `"false"` until the shared `mobilenet.onnx` is published to artifacts and the box8 image is rebuilt; `decode_destination` returns `None` → reject when the model is missing).

## Codon validation (CRITICAL — fail-loud)
- Vibe codons come from a deterministic feature-code→codon map (e.g. `MUSE/MUS → CULT#MUSC`; `MT/PK/VOL → DEST#MTNS, DEST#NATL, MOOD#ADV`; `BCH/ISL → DEST#BEAC, DEST#ISLD`). Weights: `log1p(count)` → L1-normalise → keep `w ≥ 0.05`, cap top ~8.
- The single source of truth is the live `mywai-prod-dna-codons` registry + Aurora `codon_index_map` — NOT `taste_field._AXES` (carries legacy + canonical IDs).
- Every emitted codon MUST already be registered. The DNA sync validates the union against `get_codon_index_map()` and **fails loud** on any unknown ID — it NEVER calls `ensure_codon_index_map` on destination codons (that mints a permanent dimension in the map shared with hotels and can push the count past 320, silently corrupting hotel vectors). `len(index_map) ≤ 320` hard-gated. A genuinely new codon dimension is a human escalation.
- Do NOT inherit `destinations.json`'s non-canonical IDs (`DEST#CITY`, `DEST#ISLE`, `MOOD#EXPL`, etc.) — the catalog re-derives codons from `ti-landmarks`.

## DDB write semantics (disjoint writers, no clobber)
- `mywai-prod-ti-city-enrichment`: PK `countryCode`, SK `cityId` (1:1 with `ti-cities`). NOT co-located on `ti-cities` — its monthly full `put_item` replace would clobber enrichment.
- The **image fork** and the **main worker** write **disjoint attribute sets** on the same item, so write order doesn't matter — **but ONLY because both use a targeted `UpdateItem`/`SET` (never `put_item`, which replaces the whole item)**.
- `partial: true` + `failed_stages[]`: a non-fatal stage failure still writes the rest of the record (re-attempted next month). Fatal failures redrive and are NOT written.
- `source_version` is stamped per item; the DNA sync compares it (with `updated_at`) to skip unchanged rows → idempotent re-run.

## SQS / failure handling
- Worker returns `{"batchItemFailures":[…]}`; event-source mapping has `ReportBatchItemFailures` (box8). `BatchSize: 5`.
- DLQ `mywai-prod-ti-destinations-dlq`, `maxReceiveCount=3`, 14-day retention.
- **DLQ reprocessor** `mywai-prod-ti-destinations-dlq-reprocessor-eu-west-3` (box3, plain zip, `cron(5/15 * * * ? *)`): moves messages DLQ → work queue, tracks a `RetryCount` message attribute, drops after `MAX_TOTAL_RETRIES=5`. Fork of `etl-image-dlq-reprocessor`; CloudWatch namespace `MyWai/Destinations`. Reuses the box8 `DestinationsWorkerRole` (already grants SQS on both queues).

## S3 layouts
- Raw text + artifacts: `s3://mywai-artifacts-euw3-252842590549/destinations/raw/{countryCode}/{cityId}.json` (full Wikivoyage article + raw stage artifacts).
- Media: `DNA_MEDIA_BUCKET` under `destinations/{geoname_id}/` (Pillow variants; store the S3 key, not the Commons URL).
- Model (shared): `s3://mywai-artifacts-euw3-252842590549/models/mobilenet/v1/mobilenet.onnx` — published once, `aws s3 cp` at Docker build.

## Monitoring (box6)
- `mywai-prod-ti-destinations-dlq-depth` — SQS `ApproximateNumberOfMessagesVisible > 0` on the DLQ.
- `mywai-prod-ti-destinations-worker-errors` — Lambda `Errors > 0` on the box8 worker.
- (Mirrors the existing image-pipeline DLQ + worker-error alarms.)

## Key files
| File | Purpose |
|------|---------|
| `lambda/etl/etl-destinations-enrich/` | scanner (zip): gate + SQS enqueue |
| `lambda/etl/etl-destinations-worker/` | containerized worker + `stages/` + `ddb_writer.py` + `Dockerfile` |
| `lambda/etl/etl-destinations-image/` | fork of `etl-image-worker` (MySQL stack deleted; Commons URL arrives in SQS payload) |
| `lambda/etl/etl-destinations-dlq-reprocessor/` | DLQ → work-queue redrive (fork of `etl-image-dlq-reprocessor`) |
| `glue/destinations_dump_parse.py` | one-time dump parse → promotable manifest |
| `infra/ti-box1-…yaml` | enrichment DDB table (10th) |
| `infra/ti-box2-iam.yaml` | NET-NEW `ti-destinations-worker-role` (DDB RW + S3 + SQS + first-ever ATLAS Bedrock grant) |
| `infra/ti-box3-etl.yaml` | scanner Fn + 16th rule; DLQ reprocessor Fn + 15-min rule |
| `infra/ti-box8-destinations-worker.yaml` | container worker + SQS queue + DLQ |
| `infra/ti-box6-monitoring.yaml` | destinations DLQ + worker-error alarms |

## Gotchas
- `ti-cities` stores longitude as `lng` (not `lon`/`longitude`). The DNA sync reads `(countryCode, cityId)` via `BatchGetItem` for lat/lng.
- Container base is `python:3.13` (matches the proven shredder harness) — intentionally differs from ATLAS's 3.12 default; do not "correct" it.
- ATLAS had ZERO Bedrock grants before this; the worker role needs BOTH the `foundation-model` ARN AND the `inference-profile` ARN (wildcard region) for the cross-region Haiku profile.
- Deploy order: DNA `./deploy.sh all` first (creates `destinations` + sync Lambda + Aurora ARNs) → ATLAS `./deploy.sh box1 box2 box3 box8 box6` → Glue backfill → scanner once → sync once → EventBridge steady state.
- Worker is reserved-concurrency capped (`WorkerReservedConcurrency`, default 2) — demo cap, do not raise without instruction.
- The SHERPA picker does NOT consume the catalog yet — see `sherpa_dna.md` "consumption seam (FUTURE — NOT YET WIRED)".
