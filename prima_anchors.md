# PRIMA Anchors

**One line:** Curated POI library for Rome and Paris — 409 landmarks that SHERPA attaches to sketch cards as anchor blocks, resolved from user messages by proximity scan.

## What it does
- `lookup_anchors` in `sketch_engine.py` fetches nearby POIs from DDB (`anchors` table) filtered by city and anchor type
- `landmark_resolver.py` resolves place mentions in user messages to lat/lon coordinates for geo-filtered hotel search
- The anchor library is city-specific (Paris GeoNames ID 2988507, Rome 3169070) — only monuments, museums, churches, parks, and similar feature codes are indexed
- Anchors are stored in DDB `mywai-sherpa-prod-anchors` table; 409 seeded from SHERPA's anchor library via ETL to Aurora `landmarks_registry` (used by DNA for geo queries)
- Each anchor has `BlockStatus` (SOFT by default, upgrades to LOCKED when user confirms)

## Logical flow
1. Intent extraction produces `required_sites` (e.g. "Vatican", "Eiffel Tower")
2. `landmark_resolver.extract_landmark_geo(user_message, city)` → scans in-memory city cache → returns `{lat, lon, radius_m, name}` or None
3. Geo coords passed to `_compose_hotel_filters` → `anchor_lat/lon` in filters → DNA shortlist uses PostGIS to rank hotels by proximity
4. `_lookup_anchors(city, anchor_types, ...)` queries DDB for nearby POIs → returned as `AnchorBlock` list in SketchCard

## Landmark resolver
- City cache loaded once per warm container from DDB (`mywai-prod-ti-landmarks`), filtered to monument-class GeoNames feature codes
- Cache stores pre-lowercased name + altNames for ~1ms substring scan — zero DDB calls after first load
- Supported feature codes: MNMT, MUS, CSTL, CH, AMTH, RUIN, MNS, PAL, SQR, PRK, PLZA, BRDG, TOWR, GDNS, and others (24 total)
- City mapping: Paris → GeoNames 2988507, Rome → 3169070

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/cognitive/landmark_resolver.py` | `extract_landmark_geo` | User message → lat/lon coords |
| `lambda/cognitive/sketch_engine.py:1430` | `_lookup_anchors` | DDB query for nearby POIs |
| `lambda/cognitive/sketch_engine.py:1531` | `_load_local_anchors` | Fallback: reads anchor JSON from Lambda zip |
| `lambda/tools/lookup_anchors/` | tool contract | Tool contract for anchor lookups |
| `tools/etl_landmarks_registry.py` | ETL script | Syncs DDB anchors → Aurora landmarks_registry |

## Critical code
```python
# landmark_resolver.py — supported city GeoNames IDs
_CITY_TO_GEONAMES_ID: dict[str, str] = {
    "paris": "2988507",
    "rome":  "3169070",
}
# In-memory substring scan (after first DDB load)
# extract_landmark_geo("near the Colosseum", "rome")
# → {"lat": 41.8902, "lon": 12.4923, "radius_m": 1000, "name": "Colosseum"}
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated anchor unit test | Anchors visible via GET /admin/anchors |

## Gotchas
- Anchors are locked to Rome + Paris — do not seed additional cities until V2 is planned (see memory `feedback_demo_cities.md`)
- `_load_local_anchors` is the fallback when DDB is unavailable — it reads from a JSON file bundled in the Lambda zip. This file must be kept in sync if anchors are updated
- 409 POIs seeded in Aurora `landmarks_registry` via `tools/etl_landmarks_registry.py` — rerun this ETL whenever anchors are added to DDB to keep Aurora geo distances accurate
