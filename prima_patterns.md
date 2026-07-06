# PRIMA Patterns

**One line:** Trip pattern templates that drive cold-start sketch behaviour — how many sketches to show, which axis to vary, and what taste offsets to apply for this trip type.

## What it does
- Each pattern maps a trip intent (solo weekend, family city break, bleisure) to a set of render defaults and taste offsets
- Patterns blend (top-2 weighted) — a trip that matches both "solo" and "wellness" signals gets a weighted mix of the two patterns
- Render defaults include: `cold_start_n` (N sketches), `variation_axis`, `drop_to_commit_turns`, `mode_hint`
- `sketch_composition` defines which blocks are required vs optional per sketch
- `taste_offset` pre-applies axis nudges before the user's own TasteField signals take over

## Loading priority
1. DynamoDB table `{project}-{env}-patterns` — admin-editable at runtime without deploy
2. Local YAML files under `lambda/data/patterns/` — bundled in Lambda zip
3. Hard-coded `_BUILTIN_PATTERNS` in `pattern_catalogue.py` — last-resort fallback, never None

## Built-in patterns (key subset)
| Pattern ID | Trip type | mode_hint | variation_axis |
|------------|-----------|-----------|---------------|
| `solo_city_weekend` | Independent solo city break | swatch | boutique_preference |
| `family_city_weekend` | Family with kids | swatch | family_mode |
| `couple_romantic_escape` | Romantic couple trip | swatch | romance_mode |
| `bleisure_extension` | Business + leisure | commit | pace |
| `cultural_immersion` | Museum/heritage focus | swatch | cultural_depth |
| `wellness_retreat` | Spa/wellness focus | swatch | wellness_priority |
| `adventure_weekend` | Active/outdoor | swatch | adventure_tolerance |

## Pattern fields
| Field | Purpose |
|-------|---------|
| `selection_keywords` | Trigger words in user message |
| `selection_party_hints` | Detected party type (solo, family, couple) |
| `taste_offset` | Pre-applied TasteField deltas for this pattern |
| `render_defaults.cold_start_n` | How many sketches at turn 1 |
| `render_defaults.variation_axis` | Which axis sketches vary along |
| `render_defaults.drop_to_commit_turns` | After this many turns → COMMIT |
| `render_defaults.mode_hint` | swatch / swiper / fork / commit |
| `sketch_composition.anchors_per_day` | How many POI anchors per day |
| `archetype_hints` | Weighted archetype list forwarded to DNA shortlist |
| `city_overrides` | Pattern overrides for specific cities |

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/pattern_catalogue.py` | `get_pattern` | Returns blended pattern for current turn |
| `lambda/shared/pattern_catalogue.py` | `_BUILTIN_PATTERNS` | 7+ hard-coded fallback patterns |
| `lambda/shared/pattern_catalogue.py` | `Pattern`, `RenderDefaults`, `SketchComposition` | Pydantic models |

## Critical code
```python
# pattern_catalogue.py — RenderDefaults defines cold-start behaviour
class RenderDefaults(BaseModel):
    cold_start_n: int = 3
    variation_axis: str = "boutique_preference"
    drop_to_commit_turns: int = 3
    mode_hint: str = "swatch"   # swatch | swiper | commit | fork

# taste_offset example (solo_city_weekend)
taste_offset = {
    "solo_mode": 0.4,
    "local_immersion": 0.3,
    "family_mode": -0.6,     # suppressed — solo traveller
    "nightlife_appetite": 0.1,
}
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated pattern unit test | Patterns visible via GET /admin/patterns |

## Gotchas
- Patterns blend (top-2), not hard-select — a "solo wellness" message may produce a blend of `solo_city_weekend` (60%) and `wellness_retreat` (40%). `pattern_blend` dict in SketchFrame traces the weights
- DynamoDB overrides take precedence over built-ins at runtime — a pattern deployed via admin UI without a Lambda redeploy takes immediate effect
- `city_overrides` allows per-city render tuning (e.g. fewer anchors per day in a smaller city) without creating a separate pattern
