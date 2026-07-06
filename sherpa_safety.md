# SHERPA Safety

**One line:** Hard-block layer that prevents SHERPA from planning trips to restricted destinations, processing minor-solo travel, or responding to flagged keywords — before any LLM is called.

## What it does
- `evaluate_hard_blocks` checks every turn against active policies from DynamoDB before LLM processing
- Three categories: `restricted_destination`, `minor_solo`, `keyword_block`
- Blocked turns return a `RefusalBlock` to the UI; events written to `refusal-events` table (hour-bucketed PK)
- Policies cached 300s in-process; `invalidate_cache()` forces reload
- Bedrock Guardrails (box3b) adds a second layer: denied topics, toxicity, prompt-attack blocking, PII anonymisation
- **Destination finder reuse:** `sketch_engine._filter_safe_candidate_rows` runs every catalogue candidate (over-fetched ~15) through the SAME `evaluate_hard_blocks` before the affordability gate/enrichment — restricted candidates are dropped from the picker. Country ISO-2 codes are resolved to display names first (policies store "Syria", not "SY"); the city name is checked too. Policy set is loaded once via the 300s cache (no per-candidate IO). Evaluator error → fail OPEN-but-SAFE (keep all, log `exc_info`). Logs `destination_safety_filtered` + `safety_dropped` in `destination_picker_emit`.

## Logical flow
1. Cognitive handler calls `evaluate_hard_blocks(intent, participants, user_message, destination=...)`
2. All active policies loaded from DDB (5-min TTL cache)
3. For `restricted_destination`: checks explicit destination arg → intent field → raw message keyword scan (in that priority order)
4. For `minor_solo`: checks if any participant is under 18 with no adults
5. For `keyword_block`: checks user message for flagged terms
6. First match wins → returns `RefusalDecision(blocked=True, user_message=...)` and writes to `refusal-events`
7. If no match → `RefusalDecision(blocked=False)`; turn proceeds to Bedrock

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/safety.py` | `evaluate_hard_blocks` | Main hard-block evaluator |
| `lambda/shared/safety.py` | `log_refusal` | Writes to refusal-events DDB table |
| `lambda/safety_refresher/handler.py` | `lambda_handler` | Daily feed refresh (OFAC programs + FCDO advisories) |
| `lambda/safety_refresher/handler.py` | `_write_policies` | Writes auto rows; skips `source==manual` OR `locked` rows |
| `lambda/adversarial/handler.py` | `lambda_handler` | Daily red-team runner against all attack cases |

## Critical code
```python
# safety.py — destination check priority order
def evaluate_hard_blocks(*, intent, participants, user_message, destination=None):
    _dest_ai     = (destination or "").strip().lower()          # arg takes priority
    _dest_intent = intent.get("trip_params",...).strip().lower() # then intent
    # Falls back to raw message keyword scan if both are empty
    for policy in _load_active_policies():
        if policy["category"] == "restricted_destination":
            blocked = {c.lower() for c in rule.get("countries", [])} | \
                      {c.lower() for c in rule.get("cities", [])}
            # structured check first, then raw scan
            matched = next((t for t in [_dest_ai, _dest_intent] if t in blocked), None)
            if not matched:
                matched = next((t for t in blocked if t in user_message.lower()), None)
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/shared/test_safety.py` | Policy loading, all three categories, cache TTL |
| `tests/golden/test_safety_golden.py` | Golden test cases: blocked vs allowed |

## Safety feed refresher (`safety_refresher/handler.py`)
- Runs daily 03:00 UTC. Sources (`SAFETY_SOURCES`, default `ofac,fcdo_travel`):
  - **OFAC** — `_parse_ofac_basic` maps sanctions PROGRAMS → target countries via `_PROGRAM_TO_COUNTRY` (Cuba, Iran, North Korea, Syria, Russia, Belarus, Burma, Venezuela, Crimea/Donetsk/Luhansk). Correct data shape.
  - **FCDO** — `_parse_fcdo_advisories` reads gov.uk alert_status: level-4 (`avoid_all_travel_to_whole_country`) → `restricted_destination` hard-block; level-3 → `travel_advisory` (show, not block).
- **UN/EU sanctions feeds REMOVED as destination sources (2026-06).** `_parse_un`/`_parse_eu` and the `un`/`eu`/`eu_fallback` feed URLs are gone. Those lists give the geography of sanctioned PERSONS/ENTITIES (individual nationality, entity-address country / `countryIso2Code`), NOT travel restrictions — they swept ~100 mainstream tourist countries (Belgium, Egypt, Morocco, Indonesia…) into the blocklist, producing false `restricted_destination` hard-blocks. The live `un-restricted-destinations` row had a 1,100-char country list blocking real destinations.
- **Hard-block destination coverage now = OFAC programs + FCDO level-4 + manually curated rows.**
- **`_write_policies` lock semantics:** an existing row is never overwritten when `source == "manual"` (logs `skip_manual_override`) OR it carries a truthy `locked` flag (logs `skip_locked_override`). An operator can set `locked=true` (+ `active=false`) on any auto row to permanently disable it — the ETL can no longer resurrect it.

## Gotchas
- Tehran, Moscow, and similar cities in restricted countries are NOT yet in the DDB policy `cities` array — they need to be seeded (tracked in memory: `project_safety_cities.md`)
- Bedrock Guardrails costs ~6-20% on top of model cost — `skip_guardrail=True` on eval calls to avoid burning budget (see memory: `feedback_eval_no_guardrails.md`)
- Destination priority: explicit arg beats intent field beats raw message scan. Always pass the AI-extracted destination as the `destination=` arg to catch indirect references that keyword scan misses
