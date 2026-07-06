# SHERPA Intent Extraction

**One line:** Converts a raw user message into a structured IntentPayload using Claude Sonnet — capturing destination, dates, pax, budget, amenities, session intent codons, and sentiment.

## What it does
- `produce_intent` in `intent.py` builds a rich system prompt including the user's DNA profile and calls Bedrock Sonnet
- Extracts 20 trip-level fields: destination, dates, pax_adults/children, budget, nationalities, origin, explicit_stars_min/max, explicit_amenities, urgency, sentiment, intent_type, **session_intent_codons**, etc.
- `intent_enrichment.py` runs enrichment to turn sparse/foreign messages into a strong English query for DNA search, producing **one** codon bucket: `soft_implied` (mood/style hints; DNA blends in). Enrichment also does FAISS query construction. It does NOT own session intent.
- `intent_rebase.py::rebase_intent_codons` is active for the BUDG chromosome — adjusts budget intent toward the user's DNA midpoint (e.g. luxury user saying "cheap" → BUDG#MDST). Upgrade intent (user asking for MORE than their DNA baseline) is now respected as-stated without rebasing.

## Three codon tiers (2026-05-24, updated 2026-05-27)

| Tier | Source | Examples | Weight | LENS penalty |
|------|--------|----------|--------|--------------|
| `session_intent_codons` | **Sonnet (`produce_intent`)** | "family trip", "business trip", "wellness retreat" | 5× (intent_boost) | Bypassed for FAMI/SOC/PURP/WELL/ACTV chromosomes |
| `implied_intent_codons` (soft) | Enrichment | "boutique", "escape", "cultural" | 0.5× | DNA applies normally |
| DNA-derived | Historical profile | Historical profile | 5× via taste_field | Full penalties |

Session intent codons are extracted by Sonnet alongside other IntentPayload fields. They are forwarded through `explicit_hotel_slots["session_intent_codons"]` → `user_constraints` → `sketch_engine` as `session_intent_codons_override`. Enrichment still runs but its `session_entries` output is silently ignored — enrichment owns only FAISS query + soft implied codons.

**Key rule**: amenity requests are NOT session intent. `"needs wifi"` → `[]`. `"with babysitting"` → `[]`. `"has a spa"` → `[]`. Only explicit trip-purpose or social-context statements emit codons.

## Enrichment redesign — the provenance wall (2026-06-28, LIVE)
Fixes the 87→30 match-score collapse. **Root cause**: dna-api `_handle_engine_shortlist` re-derived its own intent codons from the seasonality-decorated FAISS query **text** (`_search_codons_for_signal`) and scored them at full 5× — the seasonal prose ("Bastille Day, Paris Plages, free alternatives") sat semantically near budget, fabricating `BUDG#ECON/BARG` against a luxury DNA → cosine collapse. Deterministic, date-driven (July fabricated, September clean).

Flag-gated via `lambda/shared/enrich_flags.py` (SSM prefix `/mywai-sherpa/prod/enrich/`, 60s TTL, fail-open). All flags promoted LIVE:
| Flag | State | Effect |
|------|-------|--------|
| `forward_intent_codons` | **on** | SHERPA forwards its **genuine** full-weight `intent_codons` to the shortlist; dna-api consumes them and **STOPS** re-extracting from query text (the "wall"). Un-inverts the old weighting (genuine were 0.5×, text-fabricated 5×). Mirrored by DNA flag `/mywai-dna/prod/dna/intent/source=forwarded`. |
| `translate_only` | **on** | Haiku enrichment is **translate-only** (temp 0) — preserves the user's verbatim intent, no reworded prompt. (Contrast handling must move to Sonnet before this is permanent.) |
| `city_base` | **off** | Removed the static Paris/Rome "near landmarks / historic centre" templates — no canned geo prose injected. |
| `seasonal_prose` | **off** | Seasonality/lattice/weather prose is **no longer appended** to the codon-extraction input — it now lives **only** in the ADVISORY channel (chat copy + `turn_diagnostic`), never inferring user intent. |
| `geo_access_nudge` | off | Increment E placeholder (hotel-location → available-activities nudge); not built. |

**Provenance, not codon-type**: a budget codon is fine when the user genuinely asked ("cheap hotel" → still rebases to 3–4★ via `intent_rebase`). The defect was budget codons **fabricated from prose the user never said**. The wall preserves genuine intent and eliminates only fabricated intent. Guarded by `tests/baselines/test_enrichment_invariants*.py` (INV-2 provenance, offline + integration tiers) and the DNA `TestScoreCalibrationProvenance` regression.

**`scored_intents` echo**: dna-shortlist now echoes the intent set it actually scored on the response body; SHERPA reads it (`sketch_engine.py` stashes `candidates[0]["_scored_intents"]`) to compute the genuine-vs-fabricated diff in the `turn_diagnostic` (see `sherpa_observability.md`, `dna_hotel_ranking.md`).

**Finder guard (B-1)**: the destination finder calls `enrich_intent_query(produce_intent=True)` to keep Haiku codon inference; the hotel path uses `produce_intent=False` (translate-only). Prevents the wall from stripping the finder's codons.

## Logical flow
1. Cognitive handler calls `produce_intent(messages, dna_profile, config)` → Bedrock Sonnet with cachePoint
2. System prompt includes `<dna_profile>` block (user codons, archetype, top strokes) — triggers Bedrock prompt cache on turns 2+ (saves ~78% token cost)
3. Bedrock returns structured JSON extracted as IntentPayload — including `session_intent_codons` validated against `_VALID_SESSION_CODONS` whitelist
4. Handler writes `intent.session_intent_codons` into `explicit_hotel_slots["session_intent_codons"]` if non-empty
5. `enrich_intent_query` runs:
   - **Pre-scan**: `_has_contrast()` — detects "usually X but this time Y" pattern
   - **Pass 1**: fast regex rule map → fragments + soft_codons (strong=True rules still run but their session_entries are ignored downstream)
   - **Pass 2**: Haiku LLM call if `_is_foreign()` OR `_has_contrast()` OR pass 1 sparse
   - Returns 4 values: `faiss_query, session_entries, soft_entries, enrichment_log` (session_entries ignored)
6. Sketch engine uses `session_intent_codons_override` (from Sonnet) for session tier; `implied_intent_codons` from enrichment soft_entries
7. `session_intent_codons` bypasses `rebase_intent_codons` — only soft implied codons are rebased
8. Enriched FAISS query + both codon lists forwarded to DNA shortlist

## Durable-taste extraction (2026-07-03, write-side)
`produce_intent` emits `durable_taste_phrases: list[str]` — LASTING personal tastes stated with a genuine preference VERB (love/hate/prefer/…), in the user's own words, any language. A trip REQUEST ("I want a 5-star hotel in Paris", "I need…", "looking for…") returns `[]`. Pydantic validator `IntentPayload._clean_taste_phrases` is fail-safe (non-list/dirty → [], ≤120 chars/phrase, ≤6 phrases). Handler sets `explicit_hotel_slots["durable_taste_phrases"]` UNCONDITIONALLY when produce_intent ran (even []) so the passive-DNA fire site distinguishes "Sonnet ran → fire per phrase" from "fast-path → regex fallback". NOT persisted to `trip_slot`. Each phrase fires ONE `chat.strong` DNA signal with `event_id=turn_id#i`. See `sherpa_dna.md` → "Durable-taste signal gate".

## What is NOT extracted by produce_intent
Service-level slots (`style`, `neighbourhood`, `cabin`, `direct_only`, `theme`, `pace`, `interests`, `physical_level`, `airline_pref`) are filled via MCQ blocks and explicit conversation — intentionally excluded from `produce_intent` to keep the prompt focused.

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/cognitive/intent.py` | `produce_intent` | System prompt builder + Bedrock Sonnet call |
| `lambda/cognitive/intent_enrichment.py` | `enrich_intent_query` | 2-pass English enrichment |
| `lambda/cognitive/intent_enrichment.py` | `_is_foreign` | Detects non-English: accents, FR/IT/ES/DE keywords, Cyrillic/Arabic/CJK scripts |
| `lambda/cognitive/intent_rebase.py` | `rebase_intent_codons` | DNA-aware intent rebase — **active for BUDG** as of 2026-05-23 |

## Critical code
```python
# intent.py — session intent extraction by Sonnet (2026-05-27)
_VALID_SESSION_CODONS: frozenset[str] = frozenset({
    "SOC#COUP", "SOC#SOLO", "SOC#GRUP", "FAMI#KIDS",
    "MOOD#ROMN", "MOOD#ADVN",
    "PURP#WORK", "TIME#FAST",
    "FOOD#FINE", "WELL#SPAS",
})

# IntentPayload (shared/payloads.py):
session_intent_codons: list[str] = Field(default_factory=list)

# _raw_to_intent_payload — validated extraction from Bedrock output:
session_intent_codons=[
    c for c in (raw.get("session_intent_codons") or [])
    if isinstance(c, str) and c in _VALID_SESSION_CODONS
],

# handler.py — threading into user_constraints:
if intent.session_intent_codons:
    explicit_hotel_slots["session_intent_codons"] = intent.session_intent_codons

# sketch_engine.py — _search_and_rank_hotels uses override, ignores enrichment session_entries:
session_intent_codons = session_intent_codons_override or []
implied_intent_codons = [e["codon"] for e in soft_entries]  # enrichment soft only

# dna-shortlist/handler.py — bypass for session intent
def _calculate_dna_boost(..., session_codon_set, session_chromosomes):
    for codon in hotel_codons:
        codon_chrom = codon.split("#")[0]
        if codon in session_codon_set or codon_chrom in session_chromosomes:
            boost_score += intent_boost_val
            continue  # DNA penalty bypassed
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/cognitive/test_intent_enrichment.py` | Rule map, foreign detection, Haiku trigger |
| `tests/cognitive/test_intent_enrichment_live.py` | Live Haiku enrichment calls |
| `tests/cognitive/eval/extractors/produce_intent.py` | Eval harness for produce_intent |
| `tests/cognitive/eval/test_extract.py` | Eval test for extraction correctness |
| `tests/cognitive/eval/cases/session_intent.yaml` | 9 cases: amenity-only → `[]`; explicit trip context → correct codons |

## Intent rebase — BUDG chromosome (2026-05-23)
`intent_rebase.py` is now **active** for the BUDG chromosome. Config seeded in `mywai-dna-prod-dna-config` DynamoDB:

| `config_kind` | `config_key` | `value` |
|--------------|-------------|---------|
| `TIER_LADDERS` | `BUDG` | `["BUDG#ECON","BUDG#THRF","BUDG#MDST","BUDG#PREM","BUDG#LUXR","BUDG#FANC"]` |
| `IS_RELATIVE` | `BUDG` | `"true"` |

Effect: when a user with luxury DNA says "cheap hotel", `rebase_intent_codons` shifts `BUDG#ECON` up the ladder toward their DNA midpoint (e.g. `BUDG#MDST`) so the result is "affordable for them" not "hostel". The rebase is logged as `intent_rebased` in CloudWatch with the full rebase_log.

To add rebase for another chromosome: seed both `TIER_LADDERS/{chrom}` and `IS_RELATIVE/{chrom}` in the same DynamoDB table. The `_CORRELATED` map in `intent_rebase.py` also carries STAY one tier down when BUDG is rebased.

## BUDG synthesis at DNA load time (2026-05-23)

Luxury users often have strong `STAY.TIER#LUXR` affinity but no direct `BUDG#*` codons until they make actual bookings. Without a BUDG baseline, `intent_rebase.py::_baseline_idx` falls through and the rebase doesn't fire — "cheap hotel" keeps returning 5★ hotels.

Fix in `lambda/shared/dna_client.py::_synthesize_budg_from_stay_tier()`: after `codon_weights` is built from DynamoDB, BUDG codons are derived from the strongest STAY.TIER signal and merged in-memory. Formula: `score × codon_weight` where primary weight=0.9, neighbour=0.5–0.6.

Guard: returns `{}` (no synthesis) when any BUDG codon already has score ≥ 1.0 — real booking signals are never overwritten.

For the test user (`STAY.TIER#LUXR = 2.0`): synthesises `BUDG#LUXR = 1.8`, `BUDG#PREM = 1.0`. Rebase then finds `_baseline_idx=4` on the BUDG ladder → "cheap" intent at idx 0 → delta=4 → meet halfway → **rebase lands on BUDG#MDST**. ✓

Log to watch: `budg_derived_from_stay_tier keys=['BUDG#LUXR', 'BUDG#PREM']` in cognitive Lambda CloudWatch.

## `dna_implied_codons` — STAY hint injection (2026-05-23)

`dna_implied_codons(codon_affinities, explicit_codons, dna_scores=None, suppress_tier=False)` in `intent_enrichment.py` now accepts `dna_scores` (raw codon weight dict from `load_user_dna`). STAY codons with score ≥ 1.0 are prepended to the implied list so a luxury user's `STAY.TIER#LUXR` reaches the codon query vector even when the query is generic. `suppress_tier=True` omits STAY.TIER#* (used when the user has stated an explicit budget so the tier hint would contradict the intent).

## Temporal extraction — the IR (2026-07-02, TEMPORAL_REDESIGN)

The old `check_in`/`check_out` string properties + the null-blocklist prose are **DELETED**.
`produce_intent` now emits ONE nested `temporal` object (schema = `TEMPORAL_TOOL_PROPERTY` in
`shared/payloads.py`) — an enum-constrained `TemporalIR`. The LLM does language understanding only
(kind + params, any language); it does NO calendar math. `_raw_to_intent_payload` validates the raw
sub-model (fail-safe: `ValidationError` → `kind=none`), resolves it ONCE via `temporal_resolver` to
populate `trip_params.dates`, and stores the IR on `IntentPayload.temporal`. The IR is threaded to
the sketch engine, which re-resolves with pattern-derived `default_nights` (idempotent).

**The IR emits (examples):**
- Explicit → COMPONENTS, never a resolved string: "22 June" → `{kind:'explicit', start:{day:22,month:6}}`; year left unset unless the user stated one. Ranges → `start` + `end`.
- Relative/fuzzy → the matching kind: "second weekend in July" → `{kind:'nth_weekend', n:2, month:7}`; "in 2 weeks" → `{kind:'in_n_weeks', n:2}`; "this winter" → `{kind:'season', season:'winter'}`; "early June" → `{kind:'month_part', month:6, part:'early'}`; "in June" → `{kind:'in_month', month:6}`.
- Anecdote/reminiscence is NOT a travel request → `{kind:'none'}` ("we loved Paris in May").
- `nights` first-class: "for 5 nights" → 5, "a week" → 7, "a fortnight" → 14. Lead-time ("in 2 weeks") is a kind, never nights.

**Why:** the old split was backwards (Python owned language, the LLM owned arithmetic). Flipping it
gives multilingual + novel-phrasing coverage for free while a strict Pydantic schema keeps the LLM
output as trustworthy as the old deterministic path. All arithmetic lives in `temporal_resolver.py`
(see `contracts/sherpa_sketch_engine.md`).

**Committed fast-path:** `_extract_trip_params` skips the LLM once a destination is committed, so it
obtains the IR from `extract_temporal_ir` (handler.py) — ONE cue-gated (`_TEMPORAL_CUE_RE`),
per-turn-memoized Haiku call forcing a `report_temporal` tool with the SAME schema. `kind != none`
overrides the persisted dates; `kind == none` freezes them (year-normalised in the resolver).

**Flight delta fast-path (2026-06-19):**
Once a destination is committed, `_apply_flight_delta` is called on every turn to detect cabin/direct/origin/dest changes without an LLM call. It receives `trip_shape` and `hotel_locked` context:
- `trip_shape == "hotel_only"` → returns immediately (no flights on this trip)
- `hotel_locked=True` and trip is not `flight_only`/`full` → only cabin/direct extracted; origin/dest skipped
- Trigger regexes (`_FROM_TO_RE`, `_ORIGIN_CHANGE_RE`, `_DEST_CHANGE_RE`) are detection-only; city/airport names are extracted by `city_matcher.find_geo()` from a forward window after the trigger
- `_FROM_TO_RE` requires a flight verb — bare "from X to Y" no longer fires (prevented false positives on hotel sentences)
- `city_matcher.py` builds an Aho-Corasick automaton from `mywai-prod-ti-airports` + `mywai-prod-ti-cities` (population > 100k) at Lambda cold start; handles multi-word cities ("New York", "San Francisco")
- `_CABIN_UPGRADE_RE` handles economy downgrade ("downgrade to economy", "switch to economy")
- `_CABIN_DIRECT_CLEAR_RE` clears `direct_only` ("layovers are ok", "stops are fine")
- `trip_shape` is threaded from `lambda_handler` (resolved at line 1263) into `_extract_trip_params` as a keyword arg

## Gotchas
- The DNA embedder (MiniLM-en-384) is English-only — Haiku translation is the only multilingual layer. If a non-English message reaches DNA without going through enrichment, the vector will be poor
- `intent_rebase.py` is now active for BUDG — see "Intent rebase — BUDG chromosome" section above. Adding rebase for additional chromosomes requires only DynamoDB config; no code change needed
- Service-level slots (pace, cabin, theme) are NOT in IntentPayload — test for them in MCQ/stroke tests, not `produce_intent` tests
- `explicit_amenities` recognises pets ("bringing my dog", "pet-friendly", "travelling with my cat" → "pets"), accessibility ("wheelchair accessible", "disability access" → "accessible"), and parking ("need parking", "with parking" → "parking") in addition to the standard gym/spa/wifi set
- Pets, accessible, and parking are routed as **hard filters** in `_compose_hotel_filters` (→ `required_amenities`, never relaxed). All other explicit amenities are soft (→ `amenities` + `amenity_groups`, subject to progressive relaxation).
