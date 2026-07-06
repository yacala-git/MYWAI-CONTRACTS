# DNA Hotel Search

**One line:** Aurora PostgreSQL hybrid search — routes queries to either a codon cosine path (DNA-rich) or BM25 hybrid path (text + vector + RRF), with a 2-tier CTE for hard-filtered queries.

## What it does
- `_is_sparse_query` in `dna-shortlist/handler.py` decides the search path: codon cosine when ≥2 intent codons and no geo; BM25 hybrid otherwise
- Codon cosine path: 320-dim L2-normalised hotel vector vs combined user DNA + intent codon vector (5× boost on intent)
- BM25 hybrid path: 4-lane RRF (BM25 text / vibe vector / image vector / codon vector) from `pg_client.hybrid_hotel_search`
- 2-tier CTE: when hard filters present (stars, amenities, geo), a SQL CTE materialises candidate hotel IDs first, then outer query runs vector ops against the real table column (prevents pgvector HNSW horizon miss)
- Per-group AND for amenity pills enforced after Aurora query (Aurora uses OR semantics — any one code matches)

## Search path routing
```
Intent has ≥2 codons AND no geo filter?
  YES → codon cosine path (pure DNA alignment, no BM25)
  NO  → BM25 hybrid path (text + HNSW vector + image + RRF)
```

## 2-tier CTE (always used — city is always Tier 1)
- **Tier 1 (CTE):** `SELECT provider_id FROM hotels WHERE lower(city)=... AND <other hard filters>` — materialises IDs only. City is ALWAYS in Tier 1 so the HNSW ANN search in Tier 2 runs on the city-scoped subset, not globally. Without this, HNSW explores ef_search=40 candidates across all cities and only ~20 survive the city filter.
- **Tier 2 (outer):** `JOIN hotels ON provider_id = t.provider_id ORDER BY codon_vector <=> :qvec::vector(320) LIMIT k` — vector ops against real table column. Must NOT apply `<=>` in Tier 1 because pgvector's `<=>` operator returns 0 rows silently when applied to a materialised `vector(320)` column via RDS Data API.

## Amenity search
- Aurora: `amenities && ARRAY[...]` is OR (any one code matches)
- Per-group AND (hotel must have ≥1 code from EACH pill group) enforced in shortlist handler after Aurora returns results
- Progressive relaxation: if filtered results < 5, drop most-common pill group first (WiFi ~89% dropped before Pool ~9%), retry. Logs `codon_amenity_relax_step`
- **Hard amenities (pets, accessible, parking):** ONE am_* code per pill sent in `required_amenities` (`h.amenities @> ...` in Tier 1 CTE). Must be one code per pill — `@>` requires ALL listed codes simultaneously, so sending all display names for a pill (e.g. all 3 pet variants) would require a hotel to have every variant at once (8 Paris hotels vs 1,090 with any one). Never passed to the relaxation loop — `_caller_required_amenities` is preserved unchanged in every retry call.
- **Soft amenities (all others):** sent in `amenities` + `amenity_groups`. Subject to progressive relaxation.

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/shared/pg_client.py` | `hybrid_hotel_search` | 4-lane RRF entry point |
| `lambda/shared/pg_client.py` | `codon_cosine_search` / `codon_dot_search` | Codon vector search with 2-tier CTE branch |
| `lambda/dna-shortlist/handler.py:730` | `_is_sparse_query` | Path routing decision |
| `lambda/dna-shortlist/handler.py:1191` | Progressive relaxation loop | Per-group AND + drop-and-retry logic |

## Critical code
```python
# pg_client.py — 2-tier CTE branch
def codon_cosine_search(qvec, min_stars, max_stars, req_ams, has_geo, ...):
    use_filtered = bool(min_stars > 0 or max_stars > 0 or req_ams or has_geo)
    sql = _CODON_VECTOR_SEARCH_FILTERED_SQL if use_filtered else _CODON_VECTOR_SEARCH_SQL
    # FILTERED_SQL: WITH t AS (SELECT provider_id FROM hotels WHERE ...)
    #               SELECT ... FROM hotels h JOIN t ON h.provider_id = t.provider_id
    #               ORDER BY h.codon_vector <=> :qvec::vector(320)
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| `tests/integration/test_t3_retrieval.py` | 4 trouble queries: codon path quality vs BM25 |
| `tests/integration/test_t4_hnsw_health.py` | HNSW index health check |
| `tests/test_shortlist_core.py` | Search routing + filter logic |
| `tests/test_destination_codon_search.py` | Destination coverage ranking + prior + vector-trap guards |

## Destination ranking — query-anchored COVERAGE (2026-06-23)

The **destination** picker (mode="destinations") does NOT use the hotel cosine arm.
L2-normalised cosine has a sparsity bias: a city carrying ONLY the query's codons (a
3-codon ghost town) aligns perfectly and outranks a rich, genuinely-iconic city whose
extra codons dilute its unit-vector direction (mountain query surfaced Chico/Everett
over Boulder/Grenoble; historic surfaced Savannah/Amiens over Rome/Vienna). See memory
`project_destination_cosine_sparsity`.

`destination_codon_search(query_codons, k, geo_filter)` in `pg_client.py` ranks by:

```
coverage = Σ(query_weight[c] for c in query ∩ city.vibe_codons) / Σ(query weights)
order    = coverage + β·log1p(pop)/log1p(POP_MAX) + γ·log1p(lc)/log1p(LC_MAX)  DESC
```

- **query_codons** is the dna-shortlist `_codon_q` WEIGHTED map (intent + DNA + graph
  expansion), NOT the 320-dim vector. STAY.TIER#* and non-positive-weight codons are
  dropped; empty query → `[]` (no SQL). Threaded to SQL as two parallel arrays
  (`query_codons text[]`, `query_weights float8[]`) + scalar `query_mass`.
- **Coverage is ASYMMETRIC** — denominator is the QUERY mass only, so a broad/rich city
  is never penalised for codons outside the query. (Overlap-coefficient ties ghost vs
  rich; Jaccard penalises rich — both rejected.)
- **No vector(320) anywhere** — coverage is array set-membership over the stored
  `vibe_codons text[]` (GIN-indexed `&&` prefilter), sidestepping the Data-API trap.
- **Prominence prior ADDS** (coverage is a similarity, bigger=better; sort DESC) and is
  bounded, so it only breaks ties WITHIN a coverage band — it cannot invert a genuine
  coverage winner. Defaults β=0.06 / γ=0.04 / pool_k=50 (env-overridable).
- `codon_score` returned = the raw coverage (SHERPA match% math sees the genuine signal).
- Geo path: 2-tier CTE prefilters by `ST_DWithin` ∧ array-overlap before the coverage rank.

## STAY.TIER → BUDG codon derivation (2026-05-23)

Every hotel now receives BUDG codons derived deterministically from its STAY.TIER in `stay_v2_codons_from_vectors()` (`pg_client.py`). This ensures the BUDG chromosome has full corpus coverage so intent Jaccard scoring works for budget queries.

```python
_TIER_TO_BUDG = {
    "STAY.TIER#BUDG": [("BUDG#ECON", 0.9), ("BUDG#THRF", 0.5)],
    "STAY.TIER#MIDR": [("BUDG#THRF", 0.6), ("BUDG#MDST", 0.9)],
    "STAY.TIER#UPSC": [("BUDG#MDST", 0.6), ("BUDG#PREM", 0.9)],
    "STAY.TIER#LUXR": [("BUDG#PREM", 0.5), ("BUDG#LUXR", 0.9)],
}
```

- Primary codon score 0.9, neighbour 0.5–0.6 — both beat typical RRF-derived BUDG scores (0.3–0.5)
- The neighbour codon allows gradient matching: a 3★ hotel partially matches "midrange" and "budget" queries
- `_CHROM_CAPS["BUDG"]` raised from 1 → **2** to accommodate both codons per hotel
- Requires re-enqueuing all hotels through box6 after the code change (use `tools/reenqueue_codon_rescore.py`)
- Single-writer rule applies — never update `codons_weighted` directly in SQL; always go through box6

## Signed STAY.TIER dimension (2026-05-22)

STAY.TIER codons are collapsed onto a single signed dimension at the `STAY.TIER#LUXR` index (258) in `build_codon_vector()` (`pg_client.py:1658`):

| Codon | Signed value |
|-------|-------------|
| `STAY.TIER#BUDG` | **-0.5** |
| `STAY.TIER#MIDR` | **-0.33** |
| `STAY.TIER#UPSC` | **+0.5** |
| `STAY.TIER#LUXR` | **+1.0** |

A luxury user (LUXR=+2.0) vs a budget hotel (BUDG=-0.5) produces a negative dot-product contribution — active repulsion rather than silence. Applies to both cosine (`codon_vector`) and raw (`codon_vector_raw`) arms. New hotels are automatically correct when their STAY.TIER codons are written to `codons_weighted` — no extra column or step needed. To rebuild vectors after a schema change: `tools/backfill_tier_signed_vector.py`.

## Gotchas
- 2-tier CTE must select only `provider_id` in Tier 1 — selecting `h.*` causes pgvector's `<=>` operator to silently return 0 rows (RDS Data API materialises the vector column, which breaks ANN)
- When `tier1_thin: true` is in the response (< 5 candidates after hard filters), the shortlist does NOT silently fall back to unfiltered search — it returns the thin set and logs `codon_search_tier1_thin` to CloudWatch
- DOT product path is available via `X-Experiment-Codon-Method: dot` header but NOT promoted to production — COSINE wins DNA-merged queries (see memory `project_dot_vs_cosine_finding.md`)
- `user_message` is forwarded from SHERPA → DNA API → shortlist so BM25 preserves words like "conference" or "WiFi" that enrichment may not turn into codons
- After any mass `codon_vector` UPDATE, run `REINDEX INDEX hotels_codon_vector_hnsw; REINDEX INDEX hotels_codon_vector_raw_ip;` — the HNSW graph fragments silently and returns degraded results without error
