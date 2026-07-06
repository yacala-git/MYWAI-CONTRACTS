# SHERPA Slots and Strokes

**One line:** Two complementary input mechanisms — slots fill structured trip requirements via MCQ blocks; strokes adjust the TasteField (preference axes) via swipes, sliders, or chat.

## Slots
- **What they do:** Present multi-choice questions (MCQ blocks) to gather missing trip facts (style, pace, theme, neighbourhood) that produce_intent does not extract
- **Schema source:** S3 is runtime truth; `_CANONICAL_SCHEMAS` in `slots.py` is a hard-coded fallback. SSM version gates the cache so a deploy-time schema bump triggers a reload
- **Priority + required:** Each slot has a `priority` int and `required` flag; slots.py orders MCQ blocks so high-priority required slots appear first
- **When slots fire:** After intent extraction, if required slots are still empty the cognitive handler returns MCQ blocks before showing sketches

## Strokes
- **What they do:** Adjust the TasteField (16 preference axes, range −1.0 to +1.0) in response to user actions
- **Sources:** `StrokeSource` enum: CHAT, BUTTON, SLIDER, SWIPER_PICK, MAP, VOICE
- **Axes (16 valid):** `luxe_threshold`, `boutique_preference`, `pace`, `adventure_tolerance`, `cultural_depth`, `local_immersion`, `food_sophistication`, `nightlife_appetite`, `family_mode`, `solo_mode`, `romance_mode`, `wellness_priority`, `beach_affinity`, `outdoor_affinity`, `eco_consciousness`, `flexibility`
- **Extraction:** `stroke_handler.py` runs a fast keyword→delta map first; falls back to a Haiku LLM call for complex sentences. Delta values are 0.3–0.5
- **Validation:** `Stroke.axis` validated against `_VALID_STROKE_AXES` in `sketch_types.py` — invalid axis is nulled with a `slot_dropped` log warning

## Logical flow (strokes)
1. User sends a message or clicks a UI control
2. `extract_strokes(message, source)` → keyword scan (fast path)
3. If no axis matched and message > 4 words → Haiku LLM fallback extracts axis + delta
4. `Stroke` objects collected, validated against `_VALID_STROKE_AXES`
5. TasteField updated: each axis value clamped to [−1.0, 1.0], confidence updated

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/cognitive/slots.py` | `get_slot_schemas` | Loads slot schemas from S3 with SSM version gate |
| `lambda/cognitive/slots.py` | `_CANONICAL_SCHEMAS` | Hardcoded fallback when S3 is unavailable |
| `lambda/cognitive/stroke_handler.py` | `extract_strokes` | Keyword map + Haiku fallback stroke extraction |
| `lambda/shared/sketch_types.py` | `_VALID_STROKE_AXES` | Frozenset of 16 valid axis names |
| `lambda/shared/sketch_types.py` | `Stroke` | Pydantic model with axis validator |
| `lambda/shared/taste_field.py` | `TasteField` | 16-axis preference state with per-axis confidence |

## Critical code
```python
# sketch_types.py — axis validation prevents hallucinated axes from corrupting TasteField
_VALID_STROKE_AXES = frozenset({
    "luxe_threshold", "boutique_preference", "pace", "adventure_tolerance",
    "cultural_depth", "local_immersion", "food_sophistication", "nightlife_appetite",
    "family_mode", "solo_mode", "romance_mode", "wellness_priority",
    "beach_affinity", "outdoor_affinity", "eco_consciousness", "flexibility",
})

class Stroke(BaseModel):
    axis: Optional[str] = None
    @field_validator("axis")
    @classmethod
    def _validate_axis(cls, v):
        if v is not None and v not in _VALID_STROKE_AXES:
            _log.warning("slot_dropped field=axis value=%r", v)
            return None   # null the axis, keep the rest of the stroke
        return v
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_slots.py` | Schema loading, MCQ ordering, required/priority |
| `tests/golden/test_slot_golden.py` | Golden slot extraction cases |

## Date-shift strokes (relative date changes in follow-up turns)

`stroke_handler.py` detects relative date phrases in follow-up messages and emits `date_shift` strokes applied in `handler.py` (lines ~1173–1203) before sketch engine runs. These work even when the destination is already committed (fast-path short-circuit in `_extract_trip_params`).

| Stroke type | Keywords | Effect |
|-------------|----------|--------|
| `{"type": "weekend", "subtype": "this"}` | "this weekend", "coming weekend" | → next Friday–Sunday from today |
| `{"type": "weekend", "subtype": "next"}` | "next weekend", "the following weekend" | → Friday–Sunday one week out |
| `{"type": "weekend", "subtype": "long"}` | "long weekend", "four-day weekend" | → Friday + 3 nights |
| `{"type": "weekday", "weekday": N}` | "next monday", "for friday", "depart tuesday" | → next occurrence of weekday N (0=Mon) |
| `{"type": "duration", "nights": N}` | "for 3 nights", "make it 5 nights" | → check_out = check_in + N days |
| `{"direction": "later", "weeks": N}` | "push it back", "a week later" | → shift both dates +N weeks |

**Explicit date override in follow-ups (2026-05-27):**
`stroke_handler` only handles relative shifts. Explicit new dates ("26-06-26 to 28-06-26") are detected in `_extract_trip_params` via a regex gate (pattern: `\d{1,2}[-/]\d{1,2}[-/]\d{2,4}` + month-name patterns) before the fast-path return. If dateparser finds valid future dates, they override the persisted `check_in`/`check_out`. The regex gate prevents importing `dateparser` on every turn (only fires when the message looks like it contains a date).

## Gotchas
- S3 is the runtime schema truth — a schema deploy without an SSM version bump means the old schema stays cached until TTL expires
- Service-level slots (pace, cabin, theme) do NOT come from `produce_intent` — they come only from slot MCQ answers or strokes. Don't test for them in the intent eval harness
- Delta range for strokes is 0.3–0.5 — larger deltas would overfit a single signal and suppress the user's existing DNA priors
