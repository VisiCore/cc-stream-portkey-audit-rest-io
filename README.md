# Portkey Audit Logs Collector

Pulls **Portkey** organisation audit logs (admin actions on API keys, workspaces, integrations, configs, prompts, and so on) into Cribl Stream on a schedule via the Portkey Admin API. The pack ships a REST Collector, an event breaker, an empty pipeline, and a route. One event is produced per audit record.

## About this Pack

- Polls `GET /v1/audit-logs` on the Portkey control plane (`https://api.portkey.ai/v1` by default; set `vct_portkey_api_base` for a self-hosted control plane).
- Authenticates with the `x-portkey-api-key` header, read from a group text secret at run time with `C.Secret()`.
- Pages 100 records at a time with `current_page` / `page_size`, using the `total` field to know when to stop.
- Each run asks for records newer than the last collected timestamp (collector state tracking) and the run filter `_time > __collectible.state.latestTime` drops the boundary record, so overlapping runs do not duplicate.
- `_time` comes from the record `timestamp` (ISO 8601 UTC).
- The pipeline `vct_portkey_audit_logs` is intentionally empty. `request_body`, `query_params`, and `request_headers` arrive as JSON strings; parse or clean them downstream as you see fit. Portkey already masks header values (for example `authorization`).
- Events are tagged `datatype='portkey_audit_log'`, `sourcetype='portkey:audit'`, and `source='portkey:audit-logs'`.

## Deployment

### 1. Get an Admin API key and your organisation id (Portkey side)

In Portkey go to **Settings > API Keys** and create an **Admin** key with audit-log read access. Your organisation id is shown under **Settings > Organisation** and also appears as `organisation_id` on every audit record.

Verify before configuring the pack (needs curl and python3):

```bash
curl -sS -G https://api.portkey.ai/v1/audit-logs \
  -H "x-portkey-api-key: $PORTKEY_API_KEY" \
  --data-urlencode "organisation_id=$PORTKEY_ORG_ID" \
  --data-urlencode "start_time=$(python3 -c 'import datetime as d;print((d.datetime.now(d.timezone.utc)-d.timedelta(hours=24)).strftime("%Y-%m-%dT%H:%M:%SZ"))')" \
  --data-urlencode "end_time=$(python3 -c 'import datetime as d;print(d.datetime.now(d.timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"))')" \
  --data-urlencode page_size=2 | python3 -m json.tool
```

A 401 means the key is wrong or not an Admin key. A 200 with `"object": "audit-log"` and a `records` array means you are ready.

### 2. Create the group secret (Cribl side)

Packs cannot contain secrets. In the Worker Group, create a **text secret** with id `vct-portkey-rest-secret` holding the Admin API key. The id must match the `vct_portkey_secret_id` pack variable.

### 3. Set the pack variables

| Variable | Value |
|---|---|
| `vct_portkey_org_id` | Your Portkey organisation id (quoted string). Ships as the placeholder `'11111111-1111-1111-1111-111111111111'`. |
| `vct_portkey_secret_id` | `'vct-portkey-rest-secret'` (the group text secret holding the API key) |
| `vct_portkey_api_base` | `'https://api.portkey.ai/v1'` (change only for a self-hosted control plane) |

Variable values are JavaScript expressions, so keep the single quotes around strings.

### 4. Point the route at a destination

The pack route `vct_portkey_audit_logs` matches `datatype=='portkey_audit_log'` and sends to `devnull` by default. Change the output to your SIEM or Lake destination.

### 5. Run once, then enable the schedule

Open the collector `vct_portkey_audit_logs` inside the pack, click **Run** in preview mode with a wide time range (for example earliest `-30d`) to backfill, and confirm events appear with `_time` matching `timestamp`. Then enable the schedule (default every 15 minutes, 1 hour lookback for the first run), commit, and deploy.

### Collector settings worth knowing

- `page_size` is capped at 100 by Portkey; the collector uses 100.
- Results come newest first. Because `end_time` is fixed at job start, records arriving mid-run do not shift pages.
- Retries use exponential backoff on 429 and 503 and honour `retry-after`.
- Audit volume is low (tens of records per day for a small org), so a single task per run is expected.

## Upgrades

Upgrading in place keeps your pack variables and route output. If you add your own functions to the `vct_portkey_audit_logs` pipeline, copy the pipeline rather than editing it so upgrades do not overwrite your changes.

## References

- Portkey Admin API, List Audit Logs: https://portkey.ai/docs/api-reference/admin-api/control-plane/audit-logs/list-audit-logs
- Portkey documentation home: https://portkey.ai/docs

## Authors

- Andrew Hendrix <Andrewh@visicoretech.com>
- VisiCore Tech <CriblPacks@VisiCoreTech.com>

## Release Notes

### Version 0.1.0 - 2026-09-09

- Initial release: audit-log REST collector, `vct-portkey-audit-logs` event breaker, empty pipeline, placeholder route.
