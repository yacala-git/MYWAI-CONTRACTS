# Seeder invoke contract (`mywai-prod-seeder`) — FROZEN

The box5 seeder (`mywai-prod-seeder`, stack `mywai-prod-hotel-seeder`, `infra/box5-seeder.yaml`) is invoked cross-repo by the **aic-etl-data-pipeline `api` Lambda** (#2 on-demand fill) and by EventBridge (#1 cadence). The seeder is inline-ZipFile CFN with **no version/alias**, so a `deploy.sh box5` redeploy swaps the live contract immediately. **This interface is frozen — changing a request/response key here breaks the booking API. If you must change it, version it and update both callers.**

Dispatch order in `lambda_handler` (first match wins):
1. `event.mode == "fill_status"` → `handle_fill_status`
2. `event.mode == "cadence_tick"` → `handle_cadence_tick`
3. `event.refetch_entity` present → `handle_refetch`
4. else → **FULL root crawl of all active providers** (the dangerous default — callers MUST always set one of the above; a future hardening should require an explicit root-crawl marker).

## Modes

### `handle_refetch` — targeted refetch (aic `FILL_CATALOGUE_HOTEL`, and manual backfills)
Request: `{"provider_id": str, "refetch_entity": str, "ids": [str, …]}` (≤ `MAX_REFETCH_IDS`=5000). Renders the entity body server-side from `providers` config+auth (secret stays in-Lambda), enqueues one job per id to `queue_url_for(provider_id)`.
Response: `{"enqueued": int, "worker_wake": {…}}` (`worker_wake` present only, and inert, unless `WAKE_WORKER=true`).

### `handle_fill_status` — read-only verdict (aic `GET_FILL_STATUS`)
Request: `{"mode": "fill_status", "provider_id": str, "entity": str, "ids": [str, …]}` (`entity` = the detail data_type, e.g. `hotel_details`).
Response: `{"statuses": {id: "FILLED"|"DEFUNCT"|"PENDING"}}` — FILLED = a real detail row; DEFUNCT = a `provider_defunct_registry` row; PENDING = neither. Tolerates the registry table not existing yet.

### `handle_cadence_tick` — per-provider cron (EventBridge, #1)
Request: `{"mode": "cadence_tick"}`. Scans `provider_schedules` for due rows, atomically claims + enqueues, advances `next_run_at` + the incremental cursor. No caller args.

## Invariants callers rely on
- **Single-egress:** aic reaches the catalogue ONLY via this invoke — never a direct netstorming call. The seeder owns auth, body-render, rate governance, lane routing.
- **Read-only-ish for booking:** FILL enqueues catalogue writes (details land in the mywai catalogue), never a booking/money write, no payment path.
- **Async:** FILL returns after the fast enqueue, not the fetch; poll `fill_status`.

## Enable-time rules (systems-guru)
- `WAKE_WORKER` stays **OFF** until the netstorming lane cutover completes; then set `WORKER_SERVICE` to the `-provider-worker-b` fleet (enabling it against legacy box4 = 250-QPM double-poll ban). The wake now describes-first and only scales 0→1 (never scales a running crawl down).
- Don't `deploy.sh box5` mid-crawl-drain without eyeballing one enqueued payload (the root-crawl render was refactored — proven byte-identical, but it's the live path).
