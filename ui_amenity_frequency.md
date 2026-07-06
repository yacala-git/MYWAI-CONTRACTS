# Amenity Frequency Panel UI

**One line:** Admin panel that displays per-city amenity frequency rankings and triggers a refresh — loads data when the tab is first activated and polls for completion after "Run Now" is clicked.

## What it does
- `AmenityFrequencyPanel` component at line 2785 of `AgentSetupDashboard.tsx`
- Loads frequency data on tab activation via `GET /admin/amenity-frequency/{city}` → box16 → S3
- Shows each amenity code, its frequency (% of hotels that have it), and hotel count
- "Run Now" button calls `POST /admin/amenity-frequency/{city}/refresh` → box16 → fires `dna-api` Lambda async
- City selector lets admin switch between Paris and Rome

## Logical flow
1. User clicks "Amenity Frequency" in sidebar → `AmenityFrequencyPanel` mounts
2. On mount: `fetchAmenityData(activeCity)` → `GET /admin/amenity-frequency/{city}` via SHERPA admin API
3. If 404 (no data yet): shows "No data for {city} yet — click Run Now to generate."
4. If success: renders frequency table sorted by `frequency` descending
5. User clicks "Run Now": sends `POST /admin/amenity-frequency/{city}/refresh`
6. Endpoint returns `{"status": "triggered", "city": city, "eta_seconds": 30}`
7. UI shows "Triggering…" while POST is in flight, then "Triggered — check back in ~30s"

## Frequency data format (from S3)
```json
{
  "city": "paris",
  "total_hotels": 2827,
  "computed_at": "2026-05-22T10:00:00Z",
  "frequencies": [
    {"code": "am_wifi", "hotel_count": 2524, "frequency": 0.893},
    {"code": "am_parking", "hotel_count": 812, "frequency": 0.287},
    ...
  ]
}
```

## Key files
| File | Function | Purpose |
|------|----------|---------|
| `ui/src/components/AgentSetupDashboard.tsx:2785` | `AmenityFrequencyPanel` | Full panel component |
| `lambda/dna-config-admin/handler.py` | `_amenity_freq_get` | GET → S3 read |
| `lambda/dna-config-admin/handler.py` | `_amenity_freq_refresh` | POST → fire dna-api Lambda async |
| `lambda/dna-api/handler.py` | `_compute_amenity_frequency` | Aurora query + S3 upload |

## Critical code
```typescript
// AgentSetupDashboard.tsx:2797 — load on tab activation
const fetchAmenityData = async (city: string) => {
  const d = await adminFetch(`/admin/amenity-frequency/${city}`);
  if (!d || d.error) {
    setNote(`No data for ${city} yet — click Run Now to generate.`);
    return;
  }
  setData(d);
};

// Run Now trigger
await adminFetch(`/admin/amenity-frequency/${activeCity}/refresh`, { method: 'POST' });
// Response: { status: "triggered", city: "paris", eta_seconds: 30 }
```

## Tests
| Test file | What it covers |
|-----------|---------------|
| No automated test | Tested manually via admin UI |

## Gotchas
- The "Run Now" trigger is fire-and-forget — the UI does NOT poll for completion. User must manually refresh the frequency data after ~30 seconds by switching away and back to the tab (or clicking a reload button if added)
- If the panel shows "No data" after Run Now + 30s wait: check CloudWatch for `dna-config-admin` (POST received?), then `dna-api` (action branch triggered?), then Aurora (query ran successfully?)
- Both box16 API Gateway routes for amenity-frequency must be declared in CloudFormation — without explicit route declarations, API GW returns 404 before Lambda is invoked
