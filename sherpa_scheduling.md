# SHERPA Scheduling

**One line:** Two scheduled Lambda jobs — a feedback scheduler that prompts users 48h after checkout, and an adversarial runner that tests SHERPA daily against a suite of attack cases.

## Feedback Scheduler (box8)
- EventBridge hourly rule fires `lambda/feedback/handler.py`
- Scans the `trips` DDB table for confirmed trips (stage CHECKOUT/END) with `last_updated` in the 48–168h window
- Writes a `feedback_due` funnel event per qualifying trip (deduped — won't write twice for the same trip)
- In production this triggers email/push notification; in PoC it just marks the trip as feedback-due for the Outcome Dashboard

## Adversarial Runner
- Daily EventBridge trigger fires `lambda/adversarial/handler.py`
- Runs a fixed suite of attack messages through SHERPA's cognitive Lambda (sync invoke)
- Tests safety blocks (restricted destinations, minor-solo, keyword attacks), off-topic deflection, and prompt injection attempts
- Results stored in DDB; latest pass/fail visible at `GET /admin/adversarial`
- Can also be triggered on-demand via `POST /admin/adversarial` from the admin UI

## Logical flow (feedback)
1. EventBridge rule (hourly) → feedback Lambda
2. Scan `trips` table: stage ∈ {CHECKOUT, END} AND last_updated BETWEEN 48h ago AND 168h ago
3. For each trip: check if `feedback_due` funnel event already exists
4. Write `feedback_due` event if not → downstream notification handler fires

## Logical flow (adversarial)
1. EventBridge rule (daily) or `POST /admin/adversarial` → adversarial Lambda
2. Load attack case list from DDB (`adversarial-cases` table)
3. Invoke cognitive Lambda sync for each attack (RequestResponse, 90s timeout)
4. Evaluate response: did SHERPA refuse correctly? Did it stay on topic?
5. Write pass/fail record per case to `adversarial-results` table

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/feedback/handler.py` | `lambda_handler` | 48h post-trip feedback scheduler |
| `lambda/adversarial/handler.py` | `lambda_handler` | Daily safety + off-topic test runner |

## Critical code
```python
# feedback/handler.py — window: 48h after booking, give up after 7 days
_FEEDBACK_AFTER_H  = int(os.environ.get("FEEDBACK_AFTER_HOURS",  "48"))
_FEEDBACK_WINDOW_H = int(os.environ.get("FEEDBACK_WINDOW_HOURS", "168"))

# Scan trips in the window
cutoff_early = (now - timedelta(hours=_FEEDBACK_AFTER_H)).isoformat()
cutoff_late  = (now - timedelta(hours=_FEEDBACK_WINDOW_H)).isoformat()
resp = _trips.scan(FilterExpression=..., ...)  # stage + last_updated filter
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/eval/adversarial.py` | Adversarial case definitions and eval harness |

## Gotchas
- box8 is a stub in CloudFormation — EventBridge rules must be deployed and enabled separately before either scheduler fires
- The adversarial runner uses sync Lambda invoke (90s timeout) — if the cognitive Lambda cold-starts, the total run time for a large suite can exceed Lambda's 15-min limit. Keep the suite under ~50 cases or paginate
- Feedback window is configurable via env vars `FEEDBACK_AFTER_HOURS` / `FEEDBACK_WINDOW_HOURS` — defaults are 48h and 168h
