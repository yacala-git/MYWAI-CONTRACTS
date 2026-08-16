# Contract — the hotel media pipeline

**What this covers:** how a hotel photograph gets from a provider to the traveller's screen, which
job writes what, and the traps. Read this before touching image ingestion, classification, the
gallery, or `content_hash`.

**Status:** written 15 Aug 2026 from live measurement. Every figure states where it came from.
Things not established say so.

---

## 1. The stages, in order

Two repos. **The order matters and is not obvious** — the classifier runs before the embedder, in a
different service.

| # | stage | where | trigger | writes |
|---|---|---|---|---|
| 1 | **provider ingest** | `mywai-hotel-providers` box4b | crawl | `hotel_hub.<provider>_*` raw tables |
| 2 | **factory / identity** | Glue `glue_etl_script.py` | manual / scheduled | `canonical_hotels`, `master_hotel_registry` |
| 3 | **image processing** | ECS Fargate Spot `ecs/image-processor/processor.py` | SQS from a 5-min scanner | S3 `original.jpg` + `quality_score`, `dhash`, `status` in MySQL |
| 4 | **merge + cap** | Glue `lambda/media_aggregator/media_aggregator.py` | manual | `canonical_hotel_media` |
| 5 | **classify + gallery** | container Lambda `lambda/shredder/main.py` | SQS FIFO (watchdog) | `category` back to `canonical_hotel_media`; DDB golden `images_s3`; `content_hash` |
| 6 | **DNA + CLIP embed** | `mywai-dna` box6 `lambda/dna-labeler/handler.py` | SQS `dna-product-queue` | Aurora `hotels`, `hotels_segments_image` |
| 7 | **serve** | `mywai-sherpa/lambda/cognitive/sketch_engine.py` | a turn | the card |

⚠️ **Stage 5 classifies. Stage 6 embeds.** The CLIP vector does **not** exist when the subject label
is decided. Anything that wants CLIP at classification time must run it in-process at stage 5 — the
stored vectors are downstream and cannot be used there.

## 2. The three image artifacts — do not confuse them

This is the single easiest mistake to make; it has been made repeatedly.

| artifact | where | holds | who reads it |
|---|---|---|---|
| `canonical_hotel_media` | MySQL `hotel_canonical_data_model` | the merged pool, capped ~50/hotel | the shredder |
| **`images_s3`** | DDB golden + Aurora `hotels.images_s3` | **≤30 entries, `{key, caption}`** | **SHERPA — this is what the traveller sees** |
| `hotels_segments_image` | Aurora | 512-d CLIP vectors | retrieval / ranking |

**The card reads `images_s3`, not `canonical_hotel_media`.** But `images_s3` is a *projection* of it,
so changing the serve layer alone re-derives the same set from the same 50.

The caption format is `"{TAG} - {ROLE}"`, e.g. `"BEDROOM - HERO"` — the TAG is the classifier's
category. **SHERPA reads the key and discards the caption** (`sketch_engine.py:1444-1464`, cap 10 at
`:231`).

## 3. Traps — each verified in source

### Selection is blind to subject
- `media_aggregator.py:216-224` — the cap is `ROW_NUMBER() … ORDER BY quality_score DESC` cut at 50.
  **Sharpness only.** A well-lit boardroom outranks a softer bedroom and is *selected in*.
- `quality_score` is technical image quality — 55% BRISQUE + brightness/contrast/sharpness + a face
  adjustment (`ecs/image-processor/processor.py:115-205`). **No subject term whatsoever.**

### The classifier sees fewer images than the gallery keeps
- `shredder/main.py:490-496` classifies `LIMIT 30`; `media_aggregator.py:224` keeps **50**.
- Result: every hotel with >30 images carries a permanently unclassifiable tail *inside its gallery*.
  Measured: rank ≤30 → 15 NULLs of 96,130; rank >30 → **32,407 of 38,600 (84%)**. 24% of the corpus.

### The hero rule exists and has never run
- `shredder/main.py:161` `HERO_PRIORITY` (`EXTERIOR/BEACH/POOL 1 · LOBBY 2 · BEDROOM 3 · SPA/ACTV 4 ·
  RESTAURANT 5 · INTERIOR 6 · BATHROOM 9`, commented *"Never show toilet first"*) is **defined twice
  and read zero times**.
- The hero is actually `"role": "HERO" if i == 0` (`:514`) over rows ordered by `quality_score` —
  i.e. **the sharpest photo**.
- `is_hero` in `canonical_hotel_media` is **0 rows**; the aggregator writes it as
  `COALESCE(is_golden,0)` and clobbers it with `VALUES(is_hero)` each run (`:128`, `:229`), and
  nothing ever sets `is_golden`.

### The merge never deletes
- `INSERT … ON DUPLICATE KEY UPDATE`, no demotion. Rows that fall out of the top 50 stay forever —
  **669 hotels hold >50 rows, max 95**, against a "cap" of 50.
- `category` is **not** in the INSERT column list (declared at `:190`), so re-running the aggregator
  does not blank existing tags.
- `STAGE_TABLE = stage_canonical_media_load` is DROPped and rebuilt each run (`:77`) — staging is
  transient; the durable inputs are the provider tables and S3.

### The classifier cannot see hotel scenes
- MobileNetV2 / **ImageNet-1k** (`shredder/main.py:87`), proven from the ONNX classifier weight
  `[1000,1280,1,1]`.
- Measured on 400 supplier-labelled photos: **7 of 10 categories score 0/40**; the `LOBBY`, `POOL`
  and `SPA` tags **never fire once**.
- `POOL` is mapped to ImageNet 900/898 = *water tower, water bottle*; `LOBBY` to letter opener,
  poncho, dogsled, tandem bicycle. **The tag names were matched on words.**
- Its apparent 72% recall on rooms is **21% precision** — 138 photos called "room", 29 correct — because
  `decode_prediction` accepts BEDROOM from anywhere in the top-3 and that tag fires on `studio couch`
  / `window shade` / `desk`.

### Provider labels exist and are better than any model
- `hotelbeds_hotels_list_images.imagetypecode` — 10 classes incl. **`CON` Conferences**, on
  13.6M rows, zero NULL.
- Join to canonical is **89,699 of 89,699 = 100%** (via `hotelbeds_dna_media.dhash` = `media_id` +
  filename), with a 1.23× fan-out to resolve by modal code.
- **Netstorming publishes no taxonomy** — its `type` column is the filename ordinal, 90% NULL.

### ⚠️ Facility codes are `(code, group)`, never code alone
- `amenities_etl_script.py:99/:175` joins on `provider_raw_code` alone. A Hotelbeds facility is
  identified by `(facilityCode, facilityGroupCode)`.
- Consequence: **3,788,607 of 6,844,004 facility rows (55%)** have no exact `(code, group)` match and
  resolve to the wrong label. This is how a Paris townhouse "has" archery. See ticket **116**.
- Booking.com's API has the identical hazard (their facility id 125 = minimarket at property scope,
  linen at room scope). **Any provider's facility id is meaningless without its scope.**

### ⚠️ `content_hash` — the dirty-marker, and both its failure modes
> **✅ SPLIT 15 Aug 2026 (commit `4ad4090`).** `needs_reshred = 1` is now the ONLY work marker;
> `content_hash` is purely a content fingerprint that writers never blank. The column
> `canonical_hotels.needs_reshred BOOLEAN NOT NULL DEFAULT 0` is live. **Never `OR` the two signals
> together in a query** — they mean different things and reading both sweeps the same hotels twice.
> Everything below is retained because it explains WHY, and because the activity clone
> (`activity_watchdog/watchdog.py:64` and friends) still uses the old conflated convention and
> inherits this landmine when activities go live.

`content_hash IS NULL` used to mean **"needs re-shredding"**. It was a designed signal, not corruption.

- **It is blanked wholesale by the factory.** `glue_etl_script.py:1478-1484` blanks its entire delta
  batch unconditionally on every run, even when zero media rows are inserted. On 14 May 2026 that
  blanked 4,344 hotels in one statement at 12:34:23; the shredder drained 641 and stopped
  (its last log event ever is 12:47:53 that day), leaving **3,703 stranded for three months**.
- **The image work triggers it.** The aggregator's "only mark genuinely changed hotels" filter is
  defeated by its own `updated_at = NOW()` on every staged row, so the change-window matches
  essentially every media hotel → corpus-wide blank.
- **`force_refresh=True`** (`watchdog.py:87`) bypasses the `old_hash == new_hash` skip, so a sweep
  is full-price — which is why it collides with the **$0.50/day Bedrock cap** (the shredder role is
  in its target list; ~150 hotels trips it and denies Bedrock platform-wide until midnight).
- ⚠️ **The silent failure is worse than the loud one.** The shredder writes the hash at `main.py:1377`
  **before** the DDB persist at `:1458`. A DDB failure leaves a *valid* marker over stale or missing
  DNA — permanently, and invisible to any NULL-count telemetry.

### The fingerprint mixes text and images
> **✅ SPLIT 15 Aug 2026.** Two version-tagged halves in the same 32-char column (`t…`/`i…`), read by
> `split_content_hash`. Text half changed → full shred. Image half only → `refresh_gallery_only`,
> which republishes **both** projections and never calls Bedrock. The image half includes the
> **classifier identity**, so improving the model makes affected hotels detectably stale instead of
> needing a hand-driven pass.

An image-only change used to re-run the Haiku DNA pass, because the fingerprint included the first
five gallery URLs. **Photo work cost Bedrock money it should not.**

⚠️ **THE HALVES ARE UNREADABLE ON EVERY PRE-SPLIT ROW.** All 5,639 stored hashes are bare v1 MD5s, so
`split_content_hash` returns `(None, None)` and both halves read as UNKNOWN — which is deliberately
treated as *changed*. **The cheap gallery-only path is therefore unreachable for the current corpus**:
every hotel falls through to a full shred until it has been shredded once under the new scheme. A
deliberate re-label job must call `refresh_gallery_only` DIRECTLY rather than inferring its way in
through the hash comparison — and when it does, it must write **neither** `needs_reshred` **nor**
`content_hash` (pass the stored hash through unchanged), or a hotel flagged for a genuine text change
is silently unflagged and that change is lost. Note `update_content_hashes` clears the flag in the
same UPDATE, so calling it at all breaks this.

### ⚠️ `canonical_hotel_media.updated_at` is the aggregator's change signal
The column is `on update CURRENT_TIMESTAMP`, and `media_aggregator.py` dirty-marks every hotel whose
media row moved inside a 10-minute window. **Any bulk UPDATE of that table therefore looks like a
corpus-wide media change** and queues a full-price re-shred of ~5,639 hotels — enough to trip the
$0.50/day cap and deny Bedrock platform-wide for weeks.

Any job that writes to `canonical_hotel_media` for reasons that are **not** a media change must
preserve it explicitly: `SET …, updated_at = updated_at`. `scripts/import_provider_image_labels.py`
does this on every write and is the reference implementation.

### Aurora is the copy that is served, and it was truncated to 10
SHERPA reads `aurora.images_s3 or hive.images_s3` — **Aurora first**, DynamoDB only as a fallback.
Until 15 Aug the full shred wrote `gallery[:10]` to the published record while DynamoDB received the
whole gallery, so the served copy was capped at 10 photos even though **3,430 of 5,641 hotels carry
more**. Both now go through the single uncapped `build_image_projection`, which is also the image
fingerprint's input — so what is hashed is what is written.

## 4. Measured state (15 Aug 2026)

| | |
|---|---|
| `canonical_hotel_media` | 134,730 rows · 4,715 hotels · avg 28.6, max 95 |
| category NULL | 32,422 (24%) → **0 for hotelbeds** after `cb028b5` |
| `is_hero` set | **0** |
| provider split | hotelbeds 89,699 · netstorming 45,031 |
| `subject_source` | provider 89,699 · none 45,031 (netstorming, awaiting the classifier) |
| Paris / Rome | 107,938 imgs (38.2/hotel) · 26,792 (9.5/hotel) |
| Rome from hotelbeds | **0** — 97,993 ingested, never merged; the job has not run since 30 Apr |
| hotels with a blank marker | 3,703 of 5,639 — **99.4% already have DNA** |

## 5. Rules of thumb

1. **The provider's label beats every model.** Use it first; classify only what has none.
2. **Never treat a facility or image code without its group/scope.**
3. **Absence is a state.** An unmatched code maps to nothing, never to a best-effort neighbour.
4. **Sharpness is not subject.** Any selection that ranks on `quality_score` alone will surface
   boardrooms.
5. **Check which artifact you are changing** — fixing `images_s3` without fixing the merge re-derives
   the same set.
6. **Assume any factory or aggregator run will mass-dirty the corpus.** The marker split (`4ad4090`)
   makes the signal honest, not harmless — the writers are change-scoped, but a *corpus-wide* media
   refresh still flags everything.
7. **Preserve `updated_at`** on any write to `canonical_hotel_media` that is not a real media change.
8. **The provider's label is revisable, the model's is not.** `subject_raw` stores the supplier's own
   code verbatim, so changing our minds about a mapping is a re-map — no re-fetch, no re-classify.
   Per-provider mappings live in `providers.media_config` (a column the Analyst does not regenerate),
   never in code.
9. **A cheap path is only cheap if it is reachable.** Check what the current corpus actually stores
   before assuming a skip/fast path applies to it — every stored hash predating a scheme change reads
   as "unknown", and unknown routes to the expensive branch by design.

## 6. Not established
- Whether the 50-cap and the 30-classify limit were ever tuned (no comment, commit or doc explains
  either; the 20-point gap is certainly unintended).
- Who disabled the watchdog and the shredder on 14 May, or whether that factory run was manual —
  **there is no CloudTrail trail in this account.**
- Netstorming's media taxonomy beyond "no subject signal".

## Reaching the data
⚠️ The `mysql` CLI **fails against this cluster** since the MySQL 8.4 upgrade (5 Aug) with
`ERROR 2026 (HY000): SSL connection error: unknown error number`. Use `python3` + `pymysql`. Bash also
needs `dangerouslyDisableSandbox: true` — the local sandbox blocks port 3306. Credentials:
SSM `/mywai/prod/giata/mysql/{username,password}`, host
`dev-post-local-news-new.cxzkwcclfdvr.eu-west-3.rds.amazonaws.com`.

The Bedrock daily cap is CloudFormation stack **`mywai-prod-b22-bedrock-cost-cap`** containing Lambda
**`mywai-prod-bedrock-cost-cap`** — two different names for the same thing, and querying the stack
name as if it were a resource returns nothing. It is not an EventBridge Scheduler schedule.

## Related
- `4ad4090` marker/fingerprint split · `cb028b5` provider image labels
- `.scratch/booking-flow/IMAGE_BUILD_PLAN_2026-08-15.md` — the nine-step build plan
- `.scratch/booking-flow/IMAGE_PIPELINE_GROUND_TRUTH_2026-08-15.md` — the measurements
- `.scratch/booking-flow/IMAGE_PIPELINE_STUDY_2026-08-15.md` — the study, independently validated
- `.scratch/booking-flow/CONTENT_HASH_BLANKING_STUDY_2026-08-15.md` — the marker mechanism
- `.scratch/booking-flow/IMAGE_PLAN_DECIDED_2026-08-15.md` — the agreed plan
- Tickets **104** (gallery subjects), **116** (facility codes), **118** (the stranded markers)
