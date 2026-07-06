# SHERPA Shadow ML

**One line:** Fire-and-forget Lambda invoked after every sketch render to compare fast stroke extraction against a Haiku LLM reference, predict conversion probability, and feed the LinUCB bandit.

## What it does
- Invoked async (InvocationType=Event) by cognitive Lambda after each turn — never blocks the turn
- **Stroke agreement:** runs Haiku stroke extraction on the same message and compares axis/delta against the fast keyword extractor to surface fast-path blind spots
- **Conversion score:** rule-based P(session converts to booking) from taste field confidence, render mode, dates known, city known. LightGBM replaces this after 500+ turns of collected data
- Writes every prediction to `shadow-predictions` DDB (90-day TTL) — even on partial failure
- SQS DLQ catches Lambda invocation failures so no prediction is silently lost
- LinUCB bandit updated with render outcome signals to improve future mode selection

## Phase roadmap
| Phase | Trigger | What happens |
|-------|---------|-------------|
| Phase 1 (now) | Always | Data collection — rule-based conversion + Haiku stroke comparison |
| Phase 2 | ≥500 turns | Train LightGBM on features; hot-swap model via S3 |
| Phase 3 | ≥1k turns | Train stroke axis weight predictor |

## Input schema (from cognitive Lambda)
```
trace_id, turn_id, trip_id, user_id, user_message,
render_mode (COMMIT/FORK/SWATCH/SWIPER),
n_sketches, taste_field_snapshot, strokes_fast,
lattice_status, city, turn_number, dates_known
```

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shadow/handler.py` | `lambda_handler` | Shadow prediction entry point |
| `lambda/shared/bandit.py` | `LinUCB`, `load_bandit`, `save_bandit` | LinUCB bandit state management |

## Critical code
```python
# shadow/handler.py — LightGBM feature columns (must match train_lgbm.py)
FEATURE_COLUMNS = [
    "turn_number", "stroke_count",
    "taste_field_confidence_mean", "taste_field_axes_filled",
    "city_known", "dates_known",
    "render_commit", "render_fork", "render_swatch", "render_swiper",
    "lattice_ok", "lattice_blocked",
    "luxe_value", "luxe_confidence",
    "family_mode_value", "romance_mode_value", "pace_value",
    "stroke_agreement", "pattern_top_weight",
]
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/shadow/test_shadow_handler.py` | Shadow handler input/output, DDB write |

## Gotchas
- The cognitive Lambda fires shadow async and does not wait — shadow failure does not affect the user's turn. Monitor DLQ depth via `/admin/shadow` endpoint
- LightGBM is not loaded until Phase 2 — current conversion score is rule-based. The `_lgbm_loaded` flag tells you which path is active
- LinUCB bandit has no effect on render mode until `SHERPA_BANDIT=on` is set AND it has ≥10 reward updates. In shadow mode it records but does not influence decisions
