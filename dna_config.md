# DNA Config Service (box16)

**One line:** CRUD + audit trail for all DNA ranking knobs — chromosome weights, codon overrides, tier ladders, archetypes, and magic numbers — served via its own HTTP API and cached 300s in SHERPA.

## What it does
- DynamoDB table `mywai-dna-prod-dna-config` (PK: `config_kind`, SK: `config_key`) stores all tunable scoring parameters
- Audit table `mywai-dna-prod-dna-config-audit` logs every change: actor, reason, prev_value, new_value (365-day TTL)
- box16 exposes an HTTP API (`https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com`) with full CRUD
- SHERPA reads config via `shared/dna_config.py` — full-table scan cached 300s, refreshed on miss
- Config kinds: `CHROMOSOME_WEIGHTS`, `CODON_OVERRIDES`, `OPPOSITES`, `TIER_LADDERS`, `IS_RELATIVE`, `MAGIC_NUMBERS`, `ARCHETYPES`, `RISKY_INTENTS`

## Config kinds explained
| Kind | Purpose |
|------|---------|
| `CHROMOSOME_WEIGHTS` | Per-chromosome contribution weight in scoring |
| `CODON_OVERRIDES` | Manual score adjustments for specific codons |
| `OPPOSITES` | Codon pairs that cancel each other out (e.g. BUDG#ECON + BUDG#LUXR) |
| `TIER_LADDERS` | Budget → luxury upgrade path per chromosome |
| `IS_RELATIVE` | Flag: score relative to user's DNA vs absolute |
| `MAGIC_NUMBERS` | Scalar constants (e.g. amenity_match_boost default = 0.03) |
| `ARCHETYPES` | Weighted codon lists for 8 traveller archetypes |
| `RISKY_INTENTS` | Intent signals that trigger extra caution |

## Key schema
- Global value: `config_key = "{key}"` — applies to all product types
- Product-type override: `config_key = "{product_type}#{key}"` — overrides global for that product type
- Lookup order in `dna_config.get_config`: `{product_type}#{key}` → `{key}` → `__default__`

## Logical flow (write)
1. PUT `https://h9we9bv5r4.execute-api.eu-west-3.amazonaws.com/admin/config/{kind}/{key}` with `{value, reason, actor}`
2. box16 reads existing value, writes new value to `ConfigTable`, writes audit record to `AuditTable`
3. SHERPA's 300s cache expires naturally; `reload_config()` forces immediate refresh

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `lambda/dna-config-admin/handler.py` | `lambda_handler` | Full CRUD router + amenity-frequency routes |
| `lambda/dna-config-admin/handler.py` | `_write_audit` | Audit trail writer (365-day TTL) |
| `infra/box16-dna-config.yaml` | All resources | API GW, Lambda, DDB tables, IAM |
| `lambda/shared/dna_config.py` | `get_config` | SHERPA read-side helper (300s TTL cache) |
| `tools/seed_dna_config.py` | CLI script | Idempotent seed of all config entries |

## Critical code
```python
# dna_config.py — lookup order with product-type override
def get_config(kind, key=None, product_type=None):
    kind_data = _cache.get(kind, {})
    if product_type:
        val = kind_data.get(f"{product_type}#{key}")  # product-specific override first
        if val is not None:
            return val
    return kind_data.get(key) or kind_data.get("__default__")

# box16 handler routes — amenity-frequency BEFORE generic config catch-all
if method == "GET" and path.startswith("/admin/amenity-frequency/"):
    return _amenity_freq_get(city)
# ... then generic config routes below
```

## API routes (box16)
| Method | Path | Purpose |
|--------|------|---------|
| GET | /admin/config | All kinds summary |
| GET | /admin/config/{kind} | All keys for a kind |
| GET | /admin/config/{kind}/{key} | Single entry |
| PUT | /admin/config/{kind}/{key} | Upsert (body: value, reason, actor) |
| DELETE | /admin/config/{kind}/{key} | Soft-delete with audit |
| GET | /admin/config/{kind}/{key}/history | Audit trail (last 50) |
| GET | /admin/amenity-frequency/{city} | Read frequency JSON from S3 |
| POST | /admin/amenity-frequency/{city}/refresh | Trigger compute async |

## Tests
| Test file | What it covers |
|-----------|---------------|
| No dedicated box16 test | Config visible via DnaConfigPage admin UI |

## Gotchas
- box16 API Gateway routes must be explicitly declared in CloudFormation — API GW HTTP APIs have no proxy fallback. A missing route returns 404 before Lambda is ever invoked
- Amenity-frequency routes must appear BEFORE the generic config routes in the Lambda router — the catch-all `if not kind` fires for any path where the `kind` path param is absent
- `MEDIA_BUCKET` env var and `s3:GetObject` IAM must be set on the box16 Lambda — both were missing initially
- The `dna-config-admin` Lambda also needs `lambda:InvokeFunction` on `dna-api` to trigger the async amenity-frequency compute
