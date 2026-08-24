# Architect Genesis (box6) — marker protocol, circuit breaker, cost controls

**One line:** How box6 designs a provider's schema from real samples, and the fail-CLOSED accounting that guarantees a broken or looping genesis can never quietly burn Bedrock.

Repo: `mywai-hotel-providers`. Code: `lambda/architect/`. Marker SSOT: `lambda/architect/modules/genesis_markers.py`.

## Why this contract exists

A PROVEN production spend loop. `viator/products` genesis hit a column-sizing bug → MySQL 1406 on every row → the per-row handler swallowed and continued → the invocation ran the full 300s Lambda timeout and was **KILLED**. A timeout is **not** a Python exception, so the `except Exception` catch-all never ran, the failure marker was never written, and the breaker (which counts those markers) saw 0 and never tripped. **Fail-OPEN.** The queue had no DLQ, so the message redelivered forever, each redelivery paying ~28 Sonnet calls (~$0.06). Confirmed live: ZERO marker rows existed for a type that had been looping.

Everything below exists so that failure mode cannot recur, in that form or any adjacent one.

## The invariant

> **No marker => no Bedrock.**

The attempt marker is written and **durably committed BEFORE** the first possible spend. A timeout kill can then only ever leave the marker PRESENT, so a killed attempt is counted exactly like a failed one.

Corollary — **fail-closed must also be fail-LOUD**: if the marker cannot be written, genesis RAISES (→ SQS retry → DLQ → page). A plain `return` would be a clean Lambda exit, SQS would DELETE the message, and genesis for that type would silently never happen.

## Marker protocol

A **marker** is an `is_active=0` `provider_schemas` row that records an ATTEMPT rather than a schema. `ddl_sql` is the marker channel (`mapping_json` is `json NOT NULL`, so a marker carries the valid placeholder `'{}'`).

| Prefix | Meaning |
|---|---|
| `-- GENESIS_ATTEMPT` | attempt started; written before any spend |
| `-- GENESIS_FAILED_ATTEMPT` | legacy prefix (pre-inversion rows) — still counted |

**Lifecycle — ONE row per attempt, never two:**

| Step | Action |
|---|---|
| 1 | `_record_genesis_attempt` INSERTs `ddl_sql = "<ATTEMPT>: started req=<aws_request_id>"` |
| 2a. success | row is LEFT AS IS; the breaker's anchor moves past it. Nothing to clean up. |
| 2b. failure | `_finalize_failed_genesis` **UPDATEs the same row** → `"<ATTEMPT>\n<FAILED>: {reason}\n{ddl_log}"`. ATTEMPT prefix stays LEADING so the LIKE still matches. |
| 2c. known-transient | `_release_genesis_attempt` **DELETEs** the row — see *Cost cap* below. |

A row whose `ddl_sql` matches NEITHER prefix is a REAL schema. That test is `is_marker_ddl()`, and it is what anchors the breaker.

**Rules:**
- The marker is written on its **OWN connection with an explicit `commit()`**. `@@autocommit = 0` on this DB — an uncommitted marker is rolled back by the timeout kill, silently reinstating the fail-open bug. It must never share a connection with a transaction spanning the Bedrock work.
- Version is assigned inside ONE atomic statement (`INSERT … SELECT COALESCE(MAX(version),0)+1`). Racing workers can only collide on `idx_provider_type_version` (1062) → bounded retry that converges (MAX advances). 1062-exhaustion ⇒ a peer owns the attempt ⇒ clean skip (**return**, not raise).
- Any OTHER marker-write failure ⇒ `GenesisMarkerError` (**raise**). `get_db_connection()` (SSM + `pymysql.connect`) is INSIDE the try, or a connect blip escapes as a raw error, bypasses the handler, and becomes a clean exit.
- **Placement:** after the idempotency check, after the breaker, after `fetch_samples`, and immediately BEFORE the `AUTO_PARSING_CONFIG` block (`generate_parsing_config` is the first spend; the flag is `true` in prod). Exits above it ("already active", "circuit open", "no data found") cost **zero** markers.

**LIKE escaping:** `_` and `%` are LIKE wildcards and both prefixes contain `_`. Patterns are escaped (`-- GENESIS\_ATTEMPT%`) so SQL `LIKE` and Python `startswith` are exact mirrors. Always bind `GENESIS_ATTEMPT_LIKE` / `GENESIS_FAILED_LIKE` — never re-type a pattern locally.

## Every reader of `provider_schemas` (keep current)

| File | Site | Requirement |
|---|---|---|
| `lambda/architect/lambda_function.py` | `run_architect` sticky breaker | COUNTS markers; ANCHORS on the last non-marker row |
| `lambda/architect/lambda_function.py` | `run_architect_additive` cap | EXCLUDE markers (A5(i)) |
| `lambda/control_plane/mission_control.py` | `GET_PROVIDER_SCHEMAS` | EXCLUDE markers (A5(ii)) |
| `lambda/control_plane/connector_handler.py` | `get_schema_history` (Diff View) | EXCLUDE markers (W1) |

Any NEW reader must decide explicitly whether it wants markers. A marker rendered as a schema is a lie told during an incident.

## F8-v2 sticky circuit breaker

Counts `is_active=0` marker rows (either prefix) created **since the ANCHOR**; at/above `GENESIS_FAILED_STICKY_CAP` returns `{"status": "circuit_open"}` **without any Bedrock**.

- **ANCHOR = MAX(created_at) of the last row that is a REAL schema (matches neither prefix), ACTIVE OR NOT.**
- It is *not* "the latest `is_active=1` row". That was a bug: the documented recovery flow is "deactivate the broken schema, then re-genesis", which destroyed the anchor → COALESCE fell to the epoch → every historical marker counted → the breaker tripped instantly on the very type the operator had just asked to re-run.
- Safe to anchor on `created_at`: `provider_schemas.created_at` has **no** `ON UPDATE CURRENT_TIMESTAMP` (only `updated_at` does), so finalizing a marker in place cannot move it.
- Sticky, not a rolling window: a hopeless type stops and STAYS stopped. The original rolling-hour cap re-opened every hour — that WAS the bleed.
- `COALESCE(ddl_sql,'')` everywhere: `ddl_sql` is nullable and `NULL LIKE x` is NULL, not false.

`GENESIS_FAILED_STICKY_CAP` (default 3) is env-driven and **recorded in `infra/box6-architect.yaml`** — set it there, or the next `deploy.sh box6` silently resets it.

## Genesis insert: fail fast, but ONLY under genesis

The verification insert runs `sql_mode='STRICT_ALL_TABLES'` on its own connection so oversize values ERROR (1406) instead of silently truncating — which is what makes the F1 row-count check truthful.

| Errno | Under `strict_widen=True` (genesis) | Under `strict_widen=False` (Processor / Doctor) |
|---|---|---|
| 1406 | `ColumnTooLongError` → auto-widen ladder (…→MEDIUMTEXT→LONGTEXT) → retry | logged + skipped (byte-identical) |
| any other | `GenesisRowError` → **abort genesis** | logged + skipped (byte-identical) |

- `ColumnTooLongError` and `GenesisRowError` share the base `GenesisInsertError`. **`except ColumnTooLongError` MUST be listed before `except GenesisInsertError`** or the widen ladder silently stops working.
- The non-strict swallow is **LOAD-BEARING** for the Processor (`bulk_load_optimized` fallback) and Schema Doctor: per-row offender isolation, the deliberate 2026-07-06 batch-poisoning fix where one bad item must not poison a 3,269-item row. Scope any change by `strict_widen`, never remove it.
- Widen-exhausted / refused / row-error exit **explicitly** (`gate_blocked` with reason `autowiden_exhausted: {table}.{column}` or `genesis_row_error: {table} errno={n}`). They must not fall through to the F1 gate, which would block for a misattributed row-count reason and hide the real cause.

## Cost controls

**Attribution.** `invoke_bedrock_json(prompt, _retries=0, *, provider=None, label='box6-architect')`.
- `provider`/`label` are **KEYWORD-ONLY** and the retry recursion passes them **BY NAME**. The function recurses positionally (`invoke_bedrock_json(prompt, _retries+1)`); a `provider` added as the 2nd positional param binds the retry counter → `_retries` resets forever → **infinite paid retry loop**.
- Labels: `box6-architect` (default), `box6-architect-sniper`. (`box6-architect-additive` is RETIRED — the additive heal path is now deterministic; the only additive Bedrock call left is the per-child Sniper under the `box6-architect-sniper` label.) Distinct labels are the point — they answer "how much did the Sniper burn on retries" in `mywai-prod-ai-cost-log`.

**A cost cap is not a schema verdict.**
- `check_daily_cap()` is called OUTSIDE `invoke_bedrock_json`'s try, so `BedrockCapExceeded` propagates to callers. `identify_unique_constraints` MUST re-raise it **before** its bare `except Exception: return []`. Swallowing it returns an empty key list, and `generate_ddl` reads `if composite_keys:` → the child table is emitted with **NO UNIQUE KEY**, the F1 gate cannot see it, and `is_active=1` is written: a structurally wrong schema activates silently and its loader upsert can never dedupe.
- The Sniper degrading to "no composite key" (transport error → `None`, or a parse failure) is a real schema-quality decision and is logged loudly (`alert="sniper_degraded"`). It must never be silent.
- `run_architect` catches `BedrockCapExceeded` → `_release_genesis_attempt` (marker DELETED, breaker NOT charged) → `{"status": "deferred", "reason": "bedrock_daily_cap"}`. Rationale: box8 re-fires every ~5 min, so a cap day would otherwise burn the whole sticky budget in ~15 min and brick the type forever. The retry is already guaranteed once the cap resets, so this returns rather than raising.

## Additive heal path (`run_architect_additive`) — DETERMINISTIC (no LLM authors schema)

The Doctor-triggered additive upgrade no longer asks an 8000-token Bedrock call to "design the
extension". It re-runs the SAME signed structural induction genesis uses and diffs the result
against the active mapping. The `box6-architect-additive` LLM label is retired (grep → 0 non-test
refs). This is the last LLM-authors-schema path removed.

**Pipeline:** guard (params + `ADDITIVE_CAP_PER_HOUR` breaker, A5(i) marker filter) → load the
active mapping → build the DRIFT corpus → `induce_schema(..., resolve_key=False)` → `diff_additive`
→ over-budget width guard → `generate_additive_ddl` → `execute_ddl` (F2 tolerances) → merge the
delta into the mapping → `deactivate_and_insert_new_version` → re-feed the quarantined rows.

| Piece | Contract |
|---|---|
| **Corpus** | `∪` of the quarantined `original_payload`s (status `QUARANTINED`/`RECOVERING` — the bodies that CARRY the drift) **and** `fetch_samples` (baseline shape), cleaned through the SHARED `_build_clean_samples` (same smart_unwrap / root_path / stray-body lens as genesis, so the induced tree is comparable to the mapping). `blocking_fields` is now only the Doctor's drift HINT (logged). |
| **Root key** | INHERITED from the active mapping — `induce_schema` is called with `resolve_key=False`, so **NO Haiku root-key pick** and no `GenesisKeyResolutionError` fire on the drift corpus. A new root column is never a natural key. |
| **`diff_additive`** (in `modules/schema_inducer.py`, DB-free, unit-tested) | ADD-ONLY. Root/child columns match by lowercased `sql_name`; child tables by `table_name`/`xml_root_path`. **HARD INVARIANT:** a path already mapped to ANY active node (column / `_json` tail / child table) is UNTOUCHED — never dropped, renamed, retyped, or re-classified. A value that no longer fits an existing column is a WIDEN (the loader's strict-widen ladder), never an additive change. |
| **Emitter** (`db_manager.generate_additive_ddl`) | typed column → `ALTER … ADD COLUMN … NULL`; `_json` tail → `ALTER … ADD COLUMN <node>_json MEDIUMTEXT NULL`; new child table → the `generate_ddl` subtree (CREATE IF NOT EXISTS + Sniper key-pick + F9 + key-safety render). Every added column is NULLABLE + appended (INSTANT in-place InnoDB ADD). One statement each. |
| **Typing SSOT** | `db_manager.resolve_column_type(col, rows, wide_table, text_default_non_key)` — factored out of `generate_ddl` so ALTER and CREATE size a column IDENTICALLY. `generate_ddl` calls it with `text_default_non_key=False` (its FIX-d non-key TEXT default stays a separate post-Sniper pass that exempts Sniper composite-key columns) → **CREATE output byte-identical** (proven by `test_induce_parity` / `test_shape_conformance` / `test_f2_f3_ddl`). |
| **Over-budget drift** | `apply_table_column_width_guard(live_cols + delta tails, typed candidates, TABLE_COLUMN_BUDGET)` per table; a demoted candidate is emitted as a `<node>_json` MEDIUMTEXT tail instead of ADD COLUMN (fold-to-JSON, never a rebuild). |
| **Idempotency** | `_existing_table_columns` pre-check (root + each existing child) skips an already-present ADD COLUMN; `execute_ddl` tolerates 1050 (CREATE)/1060-1061 (ALTER). Partial-failure retry = no-op. |
| **Guards KEPT** | `ADDITIVE_CAP_PER_HOUR` breaker + A5(i) marker filter unchanged. **No** genesis-style fail-closed marker is added (the 8000-token loop that justified it is gone; the hour cap bounds the remaining bounded per-child Sniper). `_drop_stale_genesis_tables_or_fail` / `_truncate_genesis_tables_or_fail` are **NOT** reused — additive is CREATE-IF-NOT-EXISTS + ADD-COLUMN only, never clears live data. |

Genesis (CREATE) and additive (ALTER) SHARE `_build_clean_samples` / `induce_schema` /
`materialize_policy` / `resolve_column_type` / `generate_ddl` / `deactivate_and_insert_new_version`;
only `diff_additive` + `generate_additive_ddl` are additive-specific. Genesis is NEVER routed through
the diff.

## Queue + alarms

| Control | Where |
|---|---|
| genesis DLQ, `maxReceiveCount` 2 | `infra/box3-queues.yaml` |
| DLQ-depth alarm | `infra/box3-queues.yaml` |
| `genesis-circuit-open` alarm | `infra/box6-architect.yaml` |
| `architect-invocation-rate` alarm (healthy steady state is ZERO) | `infra/box6-architect.yaml` |
| Timeout 300s, `ReservedConcurrentExecutions: 2` | `infra/box6-architect.yaml` |

A message in the DLQ means genesis **RAISED** (e.g. `GenesisMarkerError` — could not record the attempt at all). That is a *different* failure from a tripped breaker: it means we could not even account for the attempt, and it pages rather than looping.

## Recovery runbook

Full runbook (with the exact SQL) lives in `lambda/architect/modules/genesis_markers.py`. Summary:

1. **Decide whether the breaker is RIGHT** before deleting anything — read the markers; their `ddl_sql` carries the reason and attempted DDL.
   - `autowiden_exhausted` / `genesis_row_error` / a gate reason ⇒ the schema IS broken. **Fix the cause.** Clearing markers just restarts the loop and re-spends Bedrock.
   - cost cap / throttle / RDS transient ⇒ should self-retract; if they accumulated anyway, clearing is correct.
2. **Clear the counter** — the only supported reset:
   ```sql
   DELETE FROM provider_schemas
    WHERE provider_id = %s AND data_type = %s AND is_active = 0
      AND (COALESCE(ddl_sql,'') LIKE '-- GENESIS\_ATTEMPT%'
           OR COALESCE(ddl_sql,'') LIKE '-- GENESIS\_FAILED\_ATTEMPT%');
   ```
   `is_active = 0` + the marker prefixes are load-bearing: this can never delete a real schema. **Never** use a bare `DELETE … WHERE provider_id = …`. box8 re-fires every ~5 min → genesis restarts on its own, no redeploy.
3. **Alternative (no DB write):** a successful activation resets the count implicitly (the anchor moves past the markers).
4. **To raise the budget** instead of clearing: `GENESIS_FAILED_STICKY_CAP` in `infra/box6-architect.yaml`.

## ⚠️ Prod-hardening trigger — DO BEFORE THE FIRST REAL TENANT

`_drop_stale_genesis_tables_or_fail` DROPs the genesis-owned tables (root + children + `{table}_view`)
whenever **no `is_active=1` schema exists** for the type, so a corrected DDL can actually land
(`CREATE TABLE IF NOT EXISTS` + tolerated 1050 silently discards it otherwise — this was the real
root cause of the 2026-07-17 viator/products block: a stale `varchar(255)` unprefixed key from a
previous run defeating a correct `TEXT` + `(255)`-prefix design).

This is **deliberately destructive and was approved on the explicit basis that mywai has ZERO real
users** (dev / build-clean doctrine, 2026-07-17). The predicate is byte-identical to the one that
already authorized the pre-existing TRUNCATE, so the *data* loss is not new — but the table shells
are, and it means:

> **The documented Wave-2 `deactivate → re-genesis` recovery flow is NO LONGER REVERSIBLE by
> reactivating the old schema row — the tables are gone.** (The deactivated row still carries its
> executed `ddl_sql`, so manual recreation has a recipe.)

**TRIGGER: before onboarding the first real tenant, either env-gate this drop or make it
archive-instead-of-drop (e.g. `RENAME TABLE` to a timestamped shell).** An env-gate was deliberately
NOT added now — with no users it would be dead code guarding nobody. Adjudicated by logic-guru
(2026-07-17): acceptable as shipped ONLY with this written trigger recorded.

## Agnosticism

Keyed only on `(provider, data_type)` + schema state. No provider literals anywhere in the marker, breaker, or insert paths — per the repo's agnostic-pipeline rule, fix the GENERIC root cause.
