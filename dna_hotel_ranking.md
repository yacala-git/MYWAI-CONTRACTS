# DNA Hotel Ranking

**One line:** Scores each hotel candidate with a weighted formula: 60% DNA cosine + 28% intent Jaccard + 12% quality, plus optional axis bonus and amenity boost (capped 20 points).

## What it does
- `_calculate_match_score` in `dna-shortlist/handler.py` computes a 0–1 score and calibrates it to a 0–100 int for display
- DNA cosine = similarity between user's normalised codon vector and the hotel's codon vector
- Intent Jaccard = overlap between intent codons and hotel codons (unweighted)
- Quality = normalised hotel star rating (0.0–1.0)
- Axis bonus = up to +0.15 from vibe/stay/food/wellness chromosome overlap (from Aurora `dna_full` column)
- Archetype bonus = **RETIRED 2026-08-29** per `mywai-dna/docs/ARCHETYPE_REDESIGN.md` §4/§7-P3.
  `_ARCHETYPE_CODONS` and `_archetype_fit_score` are deleted. It had been disarmed (not inert) by
  the DDB value `ARCHETYPE_BONUS_CAP = 0.0`, while the in-code default was `0.08` — one failed
  config read from going live with 15 dead codon ids of 34. Removal measured at zero delta.
- Amenity boost = `min(matched_codes × 0.03, 0.20)` — additive, capped at 20 points — added AFTER the base score is computed

## Score formula
```
# Default weights
base = _W_DNA(0.60) × dna_cosine + _W_INTENT(0.28) × intent_jacc + _W_QUALITY(0.12) × quality
     + axis_bonus (up to +0.15)

# Session intent conflict override (2026-05-24):
# When the hotel's dominant chromosome matches the user's session intent AND
# the user's DNA score for that chromosome is negative, weights shift to
# let explicit intent beat historical DNA for that hotel:
#   w_dna=0.38, w_intent=0.50, w_quality=0.12

match_01 = clamp(base, 0.0, 1.0)
match_score = round(match_01 × 100)   # displayed as "87% match"

# Amenity boost applied after base (preserves DNA alignment signal)
if amenity_pills_selected:
    am_boost = min(matched_codes × 0.03, 0.20)
    match_score = min(100, match_score + round(am_boost × 100))
```

## Score calibration (2026-06-28, LIVE) — flag-gated
Restores strong DNA matches to an honest high band with sharper discrimination, without breaking the tuned "cheap → 3–4★" behaviour. Gated via `lambda/shared/score_flags.py` (SSM `/mywai-dna/prod/dna/intent/score_calibration`, default off, fail-open) — promoted **on**. Pairs with the intent forwarding flag `/mywai-dna/prod/dna/intent/source` (`forwarded` | `legacy`, read by `lambda/dna-api/intent_flags.py`); under `forwarded`, `_handle_engine_shortlist` consumes SHERPA's `intent_codons` and skips `_search_codons_for_signal` re-extraction (see `sherpa_intent.md`).

- **Lever A — un-halve genuine DNA tier signal** (`_implied_weight` in `dna-shortlist/handler.py`, provenance gate ~:768): an implied `STAY.TIER#*` codon is promoted from 0.5× to full `_INTENT_BOOST` **only when it is GENUINE** — i.e. present in the user's `dna_profile` with score > 0 — and **only in the `tier_target is None` branch** (an explicit cheap `tier_target` forces `promote_tier=False`, so Lever A stays inert and the Gaussian tier_fit still demotes a luxury hotel). A **fabricated/implied** `STAY.TIER#MIDR` absent from `dna_profile` stays 0.5× and cannot tank a luxury score. This is the gate that blocks the 87→30 regression — covered by `tests/test_session_intent_scoring.py::TestScoreCalibrationProvenance` (proven to bite: removing the gate collapses a luxury match 0.68→0.45).
- **Lever B — display curve** (UI, `AgentPlayground.tsx` `MATCH_CURVE_ANCHORS` / `matchCurve()`): replaces the flat `÷0.65` display calibration with a piecewise-linear, **bimodal** curve (5★ core ≈0.5, sub-5★ ≈0.3, empty gap 0.36–0.44). Display-only — does NOT change `match_01` or the sort.

## `scored_intents` echo (2026-06-28) — observability contract
The shortlist **response body** now carries a top-level `scored_intents: list[str]` (default `[]`) — the intent codon set the scorer actually weighted. Written by `dna-api/handler.py` `_handle_engine_shortlist` via `attach_scored_intents` in `lambda/shared/shortlist_contract.py`. **Typed with a stdlib `TypedDict`, NOT Pydantic** — pydantic is not in the DNA Lambda runtime (no `pip install` in `deploy.sh`); importing it would crash dna-api at cold start. The import is unconditional, so `shortlist_contract.py` MUST be packaged on every deploy (committed). Echoes the forwarded set under `forwarded` mode, the `_search_codons_for_signal` set under `legacy`. SHERPA consumes it to compute genuine-vs-fabricated provenance (see `sherpa_observability.md`).

## Seasonality adjustments (applied before base score)
| Signal | Adjustment |
|--------|-----------|
| `demand_surge=True` + free cancellation | +0.06 boost |
| `price_pressure=high` + ≤2 star hotel | −0.04 penalty |

## V2 parallel scoring (dark, A/B)
- Per-chromosome scores computed in parallel via `_cosine_dna_per_chromosome`
- Chromosome weights derived from intent + profile: `α×intent + (1-α)×profile`, α=0.70
- `match_score_v2` stored in candidate but NOT used for ranking — collected for offline A/B comparison
- Flip criterion: ≥2 weeks of A/B logs showing v2 top-1 better-aligns to user follow-up signal

## Scoring trace (2026-05-23, shape fixed 2026-08-12)
`_calculate_match_score` returns a 6-tuple — the 6th element is a `_trace` dict exposing the intermediate components:
```python
_trace = { "dna_cos": float, "taste": float, "tier": float, "pen": float,
           "qual": float, "ax": float, "no_codons": bool }
```
This is used by the per-hotel CloudWatch log `hotel_scored` (see Observability section) and is NOT added to the API response body.

**The trace carries these keys on EVERY path.** A hotel with no codons (a skeleton row — ~241 exist in
Aurora) cannot be taste-scored and returns `0, 0.0, False, [], {}, <zeroed trace with no_codons=True>`.
It used to return an empty dict there, which the `hotel_scored` log line indexed unconditionally — one
skeleton row raised `KeyError: 'dna_cos'` and 500'd the whole shortlist (booking-flow ticket 51). Any
new consumer of the 6th element may index it directly; do not reintroduce a second shape.

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/dna-shortlist/handler.py:249` | `_W_DNA`, `_W_INTENT`, `_W_QUALITY` | Weight constants |
| `lambda/dna-shortlist/handler.py:575` | `_calculate_match_score` | Main scoring function — returns 6-tuple inc. `_trace` |
| `lambda/dna-shortlist/handler.py:1455` | Amenity boost | `min(matches × 0.03, 0.20)` |
| `lambda/dna-shortlist/handler.py:399` | V2 dark scoring | Per-chromosome A/B parallel path |

## Critical code
```python
# dna-shortlist/handler.py — weights and amenity boost
_W_DNA     = 0.60
_W_INTENT  = 0.28
_W_QUALITY = 0.12

# Base score
raw = _W_DNA * dna_cosine + _W_INTENT * intent_jacc + _W_QUALITY * quality
    + axis_bonus

# Amenity boost (post-base, capped to preserve proportionality)
_am_boost = min(_am_matches * _cfg_magic("amenity_match_boost", 0.03), 0.20)
match_score = min(100, match_score + round(_am_boost * 100))
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/test_shortlist_core.py` | Score formula, weight application |
| `tests/test_stay_v2_scoring.py` | STAY v2 codon scoring |
| `tests/test_shortlist_codonless_hotel.py` | A codon-less hotel is scored/logged/ranked, never raises (ticket 51) |
| `tests/integration/test_t3_retrieval.py` | End-to-end ranking quality |

## CloudWatch log events (search traceability — 2026-05-23)
Per-search events emitted by `dna-shortlist/handler.py`:

| Event | When | Key fields |
|-------|------|------------|
| `hotel_scored` | once per candidate | `pid`, `name`, `match`, `dna_cos`, `taste`, `tier`, `pen`, `budg` (BUDG codon list), `codons` (usable codon count — `codons=0` is a data gap, not a bad match) |
| `hotel_no_codons` | WARNING, per codon-less candidate | `pid`, `name` — a skeleton row reached the ranker (retrieval widened, or the labeller never ran on it) |
| `ranked_top10` | once per request after sort | array of top-10 `{rank, pid, name, stars, match, budg}` |

Per-search events emitted by `mywai-sherpa/cognitive/sketch_engine.py`:

| Event | When | Key fields |
|-------|------|------------|
| `intent_codons_raw` | before enrichment | `codons` (LLM-extracted), `user_message[:120]` |
| `search_dispatch` | before calling shortlist | `routing_codons`, `dna_top5`, `pool`, `query[:80]` |

## Intent provenance — which ask reached THIS hotel (2026-09-24)

**Read by no ranking term.** It is an evidence channel for the card's because-clause, added because no
hotel card could ever say the traveller's request was why a hotel ranked (ticket 196).

- `affinity_graph.expand()` returns `list[IntentEdge]` — `{source, target, weight, graph}`. It used to
  return `{target: weight}`, which collapsed two sources onto one target and destroyed *which* input
  caused the hit (52 live edges → 47 targets). The flat return is DELETED, no shim.
- `fold_weights(edges)` rebuilds the weight map and MUST be called **per graph**: the clamp in `expand()`
  is per-edge against a running sum, so folding one combined list yields different weights than the
  graphs produced separately. It raises `ValueError` on a mixed list rather than drift quietly.
- `attach_intent_provenance(item, edges, hotel_codons)` runs at `handler.py:2510`, **after** every score
  mutation and before both sorts, which read only `final_score` / `match_01`. It writes
  `item["intent_provenance"]` = the edges whose `target` is in that hotel's own codons, capped at 8.
- **The key is ABSENT when nothing intersects** — a turn with no stated ask serialises exactly as before.
- **Ordering: positives first**, then by descending magnitude, with `(source, target)` as the tiebreak.
  32 of 132 live edges are negative; ordering by magnitude alone let "avoid" edges fill all 8 slots and
  push the real positive support out.
- A malformed codon list degrades to no key plus one WARNING — never an exception. The call sits in an
  un-wrapped per-hotel loop, so a raise here would 500 the whole deck for a caption.

**Proof the shape change cost no ranking** (24 Sep 2026): old tree vs new, same body, real Aurora, same
minute — all 50 `hotel_scored` lines identical and in the same order, every payload identical apart from
the new key. Re-verify with `tests/test_affinity_graph_edges.py` (exact `==` against a frozen copy of the
old `expand`, real 274-codon prod index map).

### Rules for anything that RENDERS this
- **Voice the `source`, never the `target`.** The target is our inference. Voicing it produces
  "you asked for design-led boutiques" on a turn where they typed "a romantic hotel" — a fabricated quote.
- **A negative weight may NEVER be voiced as "because you asked for…".**
- **A `source` is not necessarily something they said.** `handler.py:1660` folds
  `intent_codons + implied_intent_codons`, so a source can be DNA-derived. The traveller's verbatim words
  are joined SHERPA-side; no raw traveller text travels down to DNA.

## Gotchas
- Amenity boost is additive post-base — a 5-pill search with 5/5 matches adds 0.15 (15 pts). A 2-pill search with 2/2 matches adds 0.06 (6 pts). The cap at 20 pts prevents large pill selections from overwhelming the DNA alignment signal
- `match_score` (0–100 int) is display-calibrated; `match_01` (float) is the raw score. Don't confuse them — the UI shows `match_score`, the sort key is `match_01`
- Axis bonus reads from `dna_full` in Aurora — hotels without vibe/stay/food/wellness data get 0 axis bonus. Check `dna_full IS NOT NULL` before debugging low scores for new hotels
- `quality` = 0 when `stars=0` (unrated) — unrated boutiques get 0 quality score and are ranked purely on DNA. This is correct behaviour, not a bug
