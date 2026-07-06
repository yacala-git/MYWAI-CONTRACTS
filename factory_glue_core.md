# Contract — Factory glue_core extraction (activity-first)

Status: BUILT (not deployed) 2026-07-01. Repo: `mywai-hotel-providers`. Design:
`docs/factory-glue-core-extraction-design.md` + `docs/factory-glue-core-packaging.md`.

## What exists
- `lambda/glue_core/` — product-NEUTRAL shared Glue library. ZERO provider/product literals.
  Imports nothing from `factory/` or `activity_factory/` (one-way dep). Modules:
  - `udfs.py` — UDF library, extracted VERBATIM from the hotel factory. Two literals parametrised:
    Splink path via `configure_splink(model_path, logger) -> bool` (call once on the driver;
    returns True iff a model loaded = the cold-start NO-OP gate); ONNX local dir standardised to
    `/tmp/match-model`. The model artifact path `models/hotel-match-model` is a shared STORAGE
    location, intentionally kept.
  - `skeleton.py` — `Ctx` + pure Spark helpers: `extract_core`, `stars_lookup_join` (hotel-only,
    guarded), `translate_geo`, `apply_geofence(geofence_table, enabled)`, `composite_address`
    (hotel-only), `cdc_delta(cdc_state_table) -> (df, count)`, `build_h3_candidates`,
    `checkpoint`/`cleanup_checkpoint`, `assemble_dna` (incl. no-DNA→description→0.0 fallback),
    `write_slim_canonical(product_type, master_id_col, dna_cols, id_cols)`.
  - `identity.py` — `Cols` struct, `MatcherStrategy` ABC, `resolve_identity(...)` orchestrator
    (registry lookup → canonical pool + self-match exclusion → H3 blocking → 3-phase scoring →
    checkpoint → row_number best → tier split → match-table write-back → apply MATCHED/PENDING).
    Module scope is pyspark-FREE (lazy Spark inside `resolve_identity`) so recipe/sql import
    without a Spark runtime.
  - `recipe.py` — `ProductRecipe` dataclass (all D1-D8 fields incl. `cdc_state_db`; `matcher`
    Optional; `golden_merge_sql` callable; `pre_canonical_hook`/`post_upsert_hook`).
  - `sql.py` — `registry_upsert_sql`, `cdc_state_sql` (bare `glue_incremental_state` = shared infra
    name), `upsert_via_temp` (generic match/staging write mechanics).
- `lambda/activity_factory/` (SHIPPABLE package): `__init__.py`, `matcher.py`
  (`ActivityMatcher(MatcherStrategy)`), `recipe_activity.py` (`make_activity_recipe`,
  `activity_golden_merge_sql`, geo-guard, 7b projection prep, RDS/pending builders, 7c projection +
  dirty-trigger `post_upsert_hook`), and the THIN `activity_etl_script.py` entrypoint.

## Invariants (enforced)
- Hotel factory `lambda/factory/glue_etl_script.py` is BYTE-IDENTICAL (sha256
  `f59181ad047468bdcb7a75d31867ada0b4a149cf373b2d5a56900c81836fa8aa`); does NOT import glue_core.
- glue_core has zero provider/product literals + no product-package imports — CI gate
  `tests/glue_core/test_glue_core_agnosticism_gate.py`.
- Activity golden-merge/registry/CDC SQL == the pre-rewrite clone SQL (whitespace-normalized) —
  `tests/glue_core/test_gate_c_sql_equivalence.py`.
- D8: activity `cdc_state_db == canonical_db` (co-located); hotel would be `hotel_hub` (deferred).

## Packaging (deploy.sh `box11b` + `box11b-activity-factory.yaml`)
- Two DISTINCT artifacts: `glue/glue_core.zip` + `glue/activity_factory.zip` (package dir at zip
  root; MD5-verified). box11b NO LONGER writes `glue/core_layer.zip` (removed the parity landmine).
- `--extra-py-files` = `s3://.../glue/glue_core.zip,s3://.../glue/activity_factory.zip`.
- `--enable_geo_fence: "true"` added (required by getResolvedOptions); `--splink_model_path ""` +
  `--enable_image_pipeline "false"` retained. Hotel box11 block + Section-8 rebuild untouched.

## Verification state
- Gate A (hotel untouched), Gate C (SQL), CI agnosticism gate: RUN + PASS locally.
- Gate B (Spark UDF/skeleton units): written, importorskip-guarded — SKIPS without a Spark runtime,
  runs in a Glue-equipped CI.
- Gate D (end-to-end clone≡refactored parity on a scratch DB): DEFERRED — needs the Glue runtime.
- NOT deployed. Hotel migration to glue_core is deferred + human-gated.
