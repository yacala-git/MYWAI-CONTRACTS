# PRIMA Data Model

**One line:** All shared types for the sketchpad output — every block the agent sends to the UI is defined here in Pydantic models.

## What it does
- Defines the full type hierarchy for sketch output: SketchFrame → SketchCard → HotelBlock / AnchorBlock / FlightBlock / ActivityBlock
- Status enums control how the UI renders each piece: LOCKED (user confirmed), SOFT (best match, swappable), PLACEHOLDER (estimated)
- `RenderMode` drives the layout: COMMIT = single card, FORK = 2 cards, SWATCH = 3 cards, SWIPER = 7 cards
- `Stroke` carries user preference adjustments from chat, buttons, sliders, and swipes — validated against 16 known axes
- All models serialise to JSON for AppSync; all fields have defaults so partial data never crashes the client

## Key types

### Status enums
| Enum | Values | Use |
|------|--------|-----|
| `SketchStatus` | DRAFT / VALID / ADJUSTED / BLOCKED / COMMITTED | Lifecycle of a SketchCard |
| `BlockStatus` | LOCKED / SOFT / PLACEHOLDER | Per-block state for UI rendering |
| `ConfidenceLevel` | LOW / MEDIUM / HIGH / COMMITTED | Overall sketch confidence |
| `RenderMode` | COMMIT / FORK / SWATCH / SWIPER | How many sketches to show |
| `StrokeSource` | CHAT / BUTTON / SLIDER / SWIPER_PICK / MAP / VOICE | Where a stroke came from |

### FlightBlock (key fields, updated 2026-06-14)
| Field | Type | Purpose |
|-------|------|---------|
| `origin` | str | IATA code of origin airport |
| `destination` | str | IATA code of destination airport |
| `origin_airport_name` | str | Full English name e.g. "London Heathrow" |
| `destination_airport_name` | str | Full English name e.g. "Paris Charles de Gaulle" |
| `origin_city` | str | "London, United Kingdom" |
| `destination_city` | str | "Paris, France" |
| `offer_id` | str? | Duffel offer ID when `real_pricing=True` |
| `real_pricing` | bool | False = scripted placeholder; True = live Duffel price |
| `expires_at` | str? | ISO datetime — Duffel offer hold expiry |
| `airline` | str | Full airline name |
| `airline_iata` | str | Airline IATA code e.g. "BA" |
| `airline_logo` | str? | Duffel `owner_carrier.logo_symbol_url`; null when absent. UI renders it as `<img>` next to the airline name, falling back to the Plane glyph on missing/failed load. |
| `cabin` | str | "economy" / "economy_plus" / "business" / "first" |
| `price_per_person` | float? | Per-adult price |
| `total_price` | float? | All passengers |
| `currency` | str | ISO currency code |
| `duration_minutes` | int? | Total journey time |
| `stops` | int? | 0 = direct |
| `slices` | list[FlightSliceSummary] | Outbound + return slices |
| `alternatives` | list[dict] | Up to 4 pre-fetched alternatives for swap carousel |
| `origin_airports_tried` | list[str] | Debug: ["LHR", "LTN"] — all origins searched |
| `budget_warning` | bool | True when cheapest offer exceeds flight_ceiling_abs |
| `status` | BlockStatus | SOFT (real offer) / PLACEHOLDER (scripted) |

**FlightSliceSummary fields:** `origin`, `destination`, `origin_airport_name`, `destination_airport_name`, `departing_at` (ISO datetime), `arriving_at` (ISO datetime), `duration_min`, `stops`, `via` (list of IATA connection codes).

### HotelBlock (52 fields, key subset)
| Field | Type | Purpose |
|-------|------|---------|
| `hotel_id` | str | Provider hotel ID |
| `stars` | int 1-5 | Hotel star rating |
| `match_score` | int 0-100 | Calibrated DNA match for display |
| `cancellation_deadline` | str? | ISO-8601, THREE states: instant `"…T18:00:00+02:00"` / bare date `"2026-09-01"` / `null`. Never a `date` — see `sherpa_sketch_engine.md` |
| `free_cancellation` | bool? | TRI-STATE; `None` = not known → renders as silence |
| `status` | BlockStatus | LOCKED/SOFT/PLACEHOLDER |
| `km_from_anchor` | float? | PostGIS distance from queried landmark |
| `photo_urls` | list[str] | Gallery, ≤10 CDN urls (B5(a), raised from 4) |
| `amenities` | list[str] | Canonical amenity codes (am_*) |
| `amenity_labels` | list[str] | Human-readable pill labels — **REQUESTED subset only** (playground match pills; do NOT widen) |
| `amenity_labels_all` | list[str] | B5(b) EVERY amenity as a label, deduped + case-insensitively sorted, ≤50 |
| `usps` | list[str] | B5(c) ≤3 selling bullets (Aurora `usps`, else DDB `metadata.usps`), each ≤240 chars |
| `description` | str? | B5(d) property blurb — `dna_full.stay`, else `.vibe`, else None; ≤400 chars, word-boundary trim + `…` |
| `dna_full` | dict | vibe/stay/food/wellness/persona/archetype from Aurora |
| `codon_contribs` | list | Per-codon score trace |
| `why` | str | DNA-match rationale sentence |

**B5 richer hotel block (2026-07-25)** — the four fields above are ADDITIVE with safe
defaults (`[]`/`[]`/`None`); consumers that ignore them are byte-unaffected. All populated
at compose time from data already in scope (the Aurora hit + the DDB canonical record) —
**no extra reads, no new IAM**. Amenity *labels* resolve from `_AMENITY_CODE_MAP` first,
then from `config/amenity_frequency/{city}.json` in `CONFIG_BUCKET` (generated from Aurora
`amenities_registry`), loaded **lazily from inside the compose path** and cached per
container — never at module import (Aurora auto-stops overnight; an import-time remote read
would hang cold starts). A registry failure degrades to the hardcoded map and skips unknown
codes; it never fails the block. Every field is size-bounded because the demo persists the
hotel sketch verbatim into a DynamoDB row.

### Stroke (preference signal)
| Field | Type | Note |
|-------|------|------|
| `source` | StrokeSource | Chat, button, slider, swipe, map, voice |
| `axis` | str? | Validated against `_VALID_STROKE_AXES` (16 values); nulled if invalid |
| `delta` | float | Range 0.3–0.5; direction of preference change |
| `confirmed` | bool | True = user locked this block |

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/sketch_types.py` | All types | Complete type hierarchy (296 lines) |
| `lambda/shared/sketch_types.py` | `_VALID_STROKE_AXES` | Frozenset of 16 axis names — any other value is nulled |
| `lambda/shared/taste_field.py` | `TasteField` | 16-axis preference state accumulator |

## Critical code
```python
# sketch_types.py — SketchFrame is the top-level published object
class SketchFrame(BaseModel):
    mode: RenderMode                 # COMMIT/FORK/SWATCH/SWIPER
    sketches: list[SketchCard]       # 1, 2, 3, or 7 cards
    variation_axis: str              # axis that varies across sketches
    no_inventory: bool = False       # UI "no rooms found" state
    no_inventory_reason: str | None  # WHY the zero: no_supply | lookup_failed | not_priced
                                     #             | no_availability | same_city
    hotel_lane_failed: bool = False  # the stay LOOKUP broke (not "nothing available")
    alternatives: list[HotelBlock]  # in-budget swaps not in primary set

class SketchCard(BaseModel):
    hotel: HotelBlock | None
    stay_missing_reason: str | None  # set ONLY on a card published with hotel=None on a shape
                                     # that wanted one: lookup_failed | not_priced
                                     #   | no_availability | filtered_out

# 16 valid TasteField axes — any stroke axis not in this set is nulled
_VALID_STROKE_AXES = frozenset({
    "luxe_threshold", "boutique_preference", "pace", "adventure_tolerance",
    "cultural_depth", "local_immersion", "food_sophistication", "nightlife_appetite",
    "family_mode", "solo_mode", "romance_mode", "wellness_priority",
    "beach_affinity", "outdoor_affinity", "eco_consciousness", "flexibility",
})
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/shared/test_payloads.py` | Payload type validation + serialisation |

## Gotchas
- `no_inventory: True` and empty `sketches: []` are different UI states — `no_inventory` shows a "no rooms found" message; empty sketches is an error state
- **`no_inventory` alone is not enough to phrase anything** — read `no_inventory_reason` first. Only `no_supply` (the city produced no candidate) may ever be spoken as a coverage claim. `lookup_failed` and `not_priced` are facts about US; `no_availability` is a fact about the DATES. Conflating them is how a Paris search printed *"We don't have hotel coverage in Paris yet"* twice on 13 Aug — once from a timed-out lookup, once from a degraded pricing sweep over three real Paris hotels (tickets 73/76)
- **`degraded` on the LENS `complete` chunk is the hub telling us it failed** — its own words: *"an UNPRICED hotel is 'unverified', not 'gone'"*. `cognitive/lens_client.fetch_lens_prices(meta=…)` surfaces it as `meta["verified"]`, the same contract `shared/lens_recheck.py` has always had. Never treat an empty price dict as evidence of supply without checking it
- **A card may arrive with `hotel: None` on a hotel shape.** That card kept a flight or a day plan the turn had already paid for — never render it as an empty stay or as "no rooms". **Read `SketchCard.stay_missing_reason`, NOT `hotel_lane_failed`, to phrase it**: the frame flag says only that the stay SEARCH did not answer, and it is frame-wide, whereas a deck can be mixed — one card keeps a substitute while its sibling is left without. Ticket 98: the shortlist SUCCEEDED (50 Paris hotels) and the pricing sweep returned nothing, so `hotel_lane_failed` was False and a card holding 100 live ranked fares was dropped, and the traveller was told the *flights* failed. The four words are `no_inventory_reason`'s plus `filtered_out` (stays WERE priced; none met the traveller's nightly cap / free-cancellation dial / budget — the only one of the four they can act on, and the one that must never read as "we couldn't get prices")
- `_dna_shortlist_candidates` RAISES `ShortlistUnavailable` on any dependency failure and returns `[]` only for a genuine empty answer. Never re-collapse the two
- `match_score` is display-calibrated (0–100 int) — it is NOT the raw floating-point match score from DNA shortlist. The raw score lives in the candidate dict before conversion
- Stroke axis validation is silent — an invalid axis name from Haiku gets nulled with a log warning, not an exception. The rest of the stroke is still processed
