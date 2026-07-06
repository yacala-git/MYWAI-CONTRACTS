# SHERPA Observability

**One line:** Tracks per-turn costs (Bedrock tokens + guardrails), writes trace events to DynamoDB, streams blocks via AppSync, and surfaces cost + Turn Intel data in the admin UI.

## What it does
- `write_cost` records per-turn token costs (model + guardrail) to the `costs` DDB table with 90-day TTL
- `write_trace` records per-event trace rows to `traces` with 30-day TTL; sampled by SSM-configurable rate
- `appsync.publish` streams structured blocks to the frontend in real time via GraphQL mutation
- `bedrock.py` extracts `cacheReadInputTokens` + `cacheWriteInputTokens` from Bedrock responses to track cache savings
- The Playground Turn Intel panel reads from `costs` + `traces` to show per-turn breakdown

## AppSync streaming
- Mutation: `publishTurnUpdate(turnId, tripId, block, nextAction, done, seq)`
- Published in a background daemon thread (3s timeout) — never blocks the turn
- Silently no-ops if `APPSYNC_URL` or `APPSYNC_API_KEY` env vars are absent
- **`seq` must be unique per turn** — the UI dedupes by `(seq, block.type)`. Duplicate seq+type means the second block is silently dropped
- Block channel is **generic JSON** (`type` + payload over the one `publishTurnUpdate` mutation) — a new block `type` needs NO GraphQL schema change (additive)

## Always-on Turn Diagnostic ("What Was Sent" — 2026-06-28)
A single `TurnDiagnostic` block emitted **exactly once per turn on every exit path** (hotels / refusal / no_inventory / discover / safety_block / error) so the playground can always show what the agent actually did — including the turns where the old hotel-path-only trace went dark.

- **Payload** (`lambda/shared/payloads.py`): `TurnDiagnostic` + nested `IntentBySource`. `type="turn_diagnostic"`. Block type string is a new unique type.
- **Emission**: `lambda_handler` is split into a thin wrapper (builds `td` + `td_ctx`) and `_handle_conversational_turn` (the prior body). The wrapper publishes `td` in a **`finally`** — `_publish_turn_diagnostic(td, turn_id, td_ctx)` — guarded by try/except so a publish failure can never break or mask the turn. `td.outcome` is set at each terminal branch (default `"unknown"`; the wrapper promotes an unhandled exception to `"error"`).
- **Seq**: `SEQ_TURN_DIAGNOSTIC = 11` (handler.py). Collision-free under the `(seq, type)` dedup rule even though the dynamic `_trace_seq` can pass 11 — that carries `type="trace"`.
- **Intent by source** (the headline): `genuine` (forwarded `intent_codons`) · `dna_implied` (`implied_intent_codons`, 0.5×) · `session` (`session_intent_codons`) · `fabricated`. `fabricated = scored_intents − (genuine ∪ dna_implied ∪ session)`, computed in `_apply_diag_to_td` from the dna-shortlist `scored_intents` echo (see `dna_hotel_ranking.md` + `sherpa_intent.md`). **Tri-state**: non-empty → fabrication present (rose); `[]` → "clean — no fabricated intent" (emerald); `None` → "provenance unverified" (slate, when no shortlist ran or the echo isn't deployed — **never** a false "clean").
- **Diag threading**: a mutable `diag` dict is threaded `run_sketch_engine → _compose_one_sketch → _search_and_rank_hotels`, written **only for sketch 0** (single writer — no parallel-build race). No-inventory/refusal early-returns populate intent/route/filters before bailing; route/scored_intents/top_candidates are written only once candidates are non-empty.
- **UI**: a fixed `WhatWasSentStrip` pinned to the **top of the ENGINE tab** (`ui_playground.md`), always rendered when `telemetry.turn_diagnostic` exists. The WS teardown is **diagnostic-aware**: it waits for the diagnostic (which publishes in the `finally`, after the terminal `done=true` block) up to a 2500ms hard cap, closing ~300ms after it arrives — so the strip renders even on refusal/END turns. lucide-react icons only; every field `?? fallback`-guarded.

## Cost model
| Category | Typical per-turn | Monthly at 30k turns |
|----------|-----------------|---------------------|
| Bedrock Sonnet (reasoning) | $0.04–0.10 | $1,200–3,000 |
| Bedrock Guardrails | $0.004–0.020 | $120–600 |
| Bedrock Haiku (fast_classifier) | ~$0.001 | ~$30 |
| Lambda execution | negligible | ~$0.03 |
| AppSync | negligible | ~$0.09 |

## Model tiers
| Tier | Default model | Use |
|------|--------------|-----|
| `fast_classifier` | `claude-haiku-4-5-20251001` | Slot extraction, stroke extraction, translation |
| `reasoning` | `claude-sonnet-4-6-20260201` | Intent extraction, copy generation |
| `vision` | `claude-haiku-4-5-20251001` | Vision tasks |

Model IDs hot-swappable via S3 config key `sherpa/model_config/latest.json`, 5-min TTL cache.

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/observability.py` | `write_cost`, `write_trace` | DDB cost + trace writers |
| `lambda/shared/observability.py` | `bedrock_cost_usd` | Token → USD calculator |
| `lambda/shared/appsync.py` | `publish` | AppSync streaming (background thread) |
| `lambda/shared/bedrock.py` | `converse` | Bedrock Converse API + cache token extraction |

## Critical code
```python
# appsync.py — fire-and-forget; never blocks the turn
def publish(turn_id, trip_id, block, *, seq=0, done=False, ...):
    url = os.environ.get("APPSYNC_URL")
    key = os.environ.get("APPSYNC_API_KEY")
    if not url or not key:
        return          # silent no-op — works in tests / CLI
    def _send():
        try:
            urllib.request.urlopen(req, timeout=3)
        except Exception as _e:
            log.warning("appsync_publish_failed seq=%s error=%s", seq, _e)
    threading.Thread(target=_send, daemon=True).start()
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated observability unit test | Turn Intel visible in Playground UI |

## Gotchas
- `seq` uniqueness is critical — see memory `feedback_appsync_seq_uniqueness.md`. Two blocks sharing `seq + type` means the second is silently dropped by the UI
- `skip_guardrail=True` in eval calls saves ~$2/run — always set it for slot extraction evals unrelated to safety (see memory `feedback_eval_no_guardrails.md`)
- Eval `turn_id` must be unique per run, not shared across runs — shared `turn_id="eval"` shows accumulated costs as a spike on today's dashboard (see memory `feedback_eval_turn_id_per_run.md`)
- Trace sampling rate is an SSM parameter — 1.0 in dev, lower in prod. Always-sample on errors and HITL-flagged events regardless of rate
