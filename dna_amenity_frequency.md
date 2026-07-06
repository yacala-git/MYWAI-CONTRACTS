# DNA Amenity Frequency

**One line:** Per-city frequency table showing what fraction of hotels have each amenity code — used to sort progressive relaxation so the most common amenity (WiFi ~89%) is dropped first.

## What it does
- Frequency data is a JSON dict mapping `am_*` codes to `{hotel_count, total_hotels, frequency}`, cached in S3 at `config/amenity_frequency/{city}.json`
- The frequency JSON also includes a `pill_codes` section: `{pill_name: [am_code, ...]}` derived from `amenities_registry` categories — this is the source of truth SHERPA uses to map UI pills (Pool, Restaurant, etc.) to `am_*` codes
- `pill_codes.json` (city-agnostic) is written to both `MEDIA_BUCKET/config/pill_codes.json` and `SHERPA_CONFIG_BUCKET/config/pill_codes.json` on every compute run
- Progressive relaxation in the shortlist Lambda reads this data to sort pill groups most-common-first before dropping them one by one when < 5 hotels match
- `POST /admin/amenity-frequency/{city}/refresh` in box16 fires the compute function async via Lambda invoke
- Compute function queries Aurora `hotels.amenities` column, counts occurrences per code, uploads JSON to S3
- Admin UI "Run Now" button in the Amenity Frequency panel triggers this refresh

## Logical flow (compute)
1. Admin UI → `POST /admin/amenity-frequency/{city}/refresh` → box16 handler
2. box16 invokes `dna-api` Lambda async with `{"action": "compute_amenity_frequency", "city": city}`
3. `dna-api` handler routes to `_compute_amenity_frequency(city)` (action branch at top of handler)
4. Aurora query: `SELECT code, COUNT(*) FROM hotels, unnest(amenities) AS code WHERE lower(city) = :city AND code LIKE 'am_%' GROUP BY code`
5. Result uploaded to `MEDIA_BUCKET/config/amenity_frequency/{city}.json`

## Logical flow (consumption)
1. Shortlist handler selects amenity groups from `filters["amenity_groups"]`
2. If < 5 candidates after per-group AND: read frequency data from S3
3. Sort pill groups by average frequency (highest first)
4. Drop top group, retry search with remaining groups
5. Repeat until ≥5 candidates or all groups dropped
6. Logs `codon_amenity_relax_step drop=N remaining_groups=M hits_before=K`

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/dna-config-admin/handler.py` | `_amenity_freq_get` | Read JSON from S3 |
| `lambda/dna-config-admin/handler.py` | `_amenity_freq_refresh` | Fire compute Lambda async |
| `lambda/dna-api/handler.py` | `_compute_amenity_frequency` | Aurora query + S3 upload |
| `lambda/dna-shortlist/handler.py:67` | `_sort_groups_by_commonness` | Sort groups by frequency for relaxation |
| `tools/compute_amenity_frequency.py` | CLI script | Manual compute outside Lambda (for initial seed) |

## Critical code
```python
# dna-config-admin/handler.py — router ordering (amenity routes BEFORE generic config catch-all)
if method == "GET" and path.startswith("/admin/amenity-frequency/"):
    city = path.split("/admin/amenity-frequency/")[-1].split("/")[0]
    return _amenity_freq_get(city)

# dna-api/handler.py — action branch at top of lambda_handler
if event.get("action") == "compute_amenity_frequency":
    city = event.get("city", "")
    result = _compute_amenity_frequency(city)
    return {"status": "done", "city": city, "total_hotels": result["total_hotels"]}
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated test | Verified via admin UI "Run Now" + curl |

## Pill code derivation (pill_codes.json)
Pills are derived from `amenities_registry` at compute time, not hardcoded:
- Category-derived pills (`PILL_TO_CATEGORIES`): pool → `stay_pool` (9 codes), spa → `well_spas`/`well_spa`, gym → `well_fitn`, pets → `soc_pets`, restaurant → `food_hall`
- Commodity-name pills (`PILL_COMMODITY_NAMES`): wifi → "wi-fi", parking → "parking - auto", bar → "pub", breakfast → "breakfast buffet" etc., restaurant also includes 3 commodity dining codes
- `pill_codes.json` shape: `{"generated_at": "...", "pill_codes": {"pool": ["am_xxx", ...], ...}}`
- SHERPA reads this from `CONFIG_BUCKET/config/pill_codes.json` at first request per city (cached for Lambda container lifetime). Falls back to hardcoded `_PILL_GROUP_CODES` if S3 read fails.
- **To refresh**: run `python3 tools/compute_amenity_frequency.py --city paris --city rome` with `MEDIA_BUCKET` and `SHERPA_CONFIG_BUCKET` set, then redeploy SHERPA box4

## Gotchas
- The amenity-frequency routes in box16 API Gateway MUST be declared before the generic `/admin/config` routes — the config catch-all fires when `kind` path param is empty, which matches amenity-frequency paths too (see prior bug where `GET /admin/config` was returned for amenity requests)
- `MEDIA_BUCKET` env var must be set on box16 Lambda — it was missing initially, causing the read endpoint to return 503
- box16's IAM policy needs `s3:GetObject` on `config/amenity_frequency/*` in MEDIA_BUCKET — added to `box16-dna-config.yaml`
- `SHERPA_CONFIG_BUCKET` is not in `.env` (only set at compute time); its value is `mywai-sherpa-prod-config-252842590549-eu-west-3` (from `mywai-sherpa-prod-b3-config` CF stack output `ConfigBucketName`)
- If the "Run Now" button does nothing: check CloudWatch for `dna-config-admin` (did box16 receive the POST?), then `dna-api` (did it receive the action event?), then Aurora query results
