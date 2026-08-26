# mac-n8n-rms

n8n workflows that poll the Teltonika RMS API for router/device telemetry and load it into BigQuery, so uptime and signal quality can be reported on in Looker Studio.

## Workflows

### `rms-fleet-hourly-to-bigquery.json`
Polls the full device fleet from RMS and appends one row per device per run to BigQuery.

- **Trigger:** cron `0 7-18 * * *` (hourly, 7am-6pm). Downtime outside that window won't be captured — switch the expression to `0 * * * *` for 24/7 coverage.
- **Flow:** `Every Hour1` → `Get Devices` (paginated `GET /api/devices`) → `Split Devices` → `Normalise` (flattens and types each device's fields) → `BigQuery Insert1` (append).
- **Uptime** isn't computed in the workflow — it's derived downstream in Looker Studio from `is_online` across `polled_ts`.

### `rms-single-device-poll-to-bigquery.json`
A higher-frequency variant scoped to a single device (`SD-GEN072` by default). **Disabled by default** — enable the trigger node to run it.

- **Trigger:** cron `*/2 * * * *` (every 2 minutes, 24/7).
- **Flow:** `Every 5 Minutes` → `Get Devices1` → `Split Devices1` → `Normalise1` (filters to the target device name(s); throws if no match, so a rename in RMS surfaces as a failed run instead of a silent gap) → `BigQuery Insert2` (append).
- To change which device(s) are polled, edit the `TARGET_NAMES` array in the `Normalise1` code node.
- Once the numeric RMS id for the target device is known, the URL can be changed to `/api/devices/<id>` to drop pagination — worth doing at this polling cadence.

## Destination

Both workflows append to the same BigQuery table: `.mac_hardware_wh.mac_teltonika_rms`, keyed by `(device_id, polled_ts)`. If a `v2` table is ever created, update the `tableId` in the relevant BigQuery node.
