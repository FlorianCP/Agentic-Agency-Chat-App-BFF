# BFF debugging playbook

This is an agent-facing incident runbook. Follow it cold on the companion host. Commands are
read-only and do not expose configured secrets. Device diagnostic messages and BFF request
paths/queries are full fidelity and may contain message content or PII, so keep any copied output
inside the customer's support boundary. Request bodies and authorization headers are not logged.
The request-ID echo, request/chunk metadata, and paginated query examples require BFF `0.21.0` or
newer. BFF `0.20.0` does not provide those additions; confirm the installed version before relying
on them. Existing local JSONL files remain readable on older versions.

## 1. Locate the files

The default config is `~/.agency-bff/config.json`; `AGENCY_BFF_DIR` overrides its directory. Read
the configured data directory rather than assuming the default:

```sh
LOG_ROOT="$(jq -r '.data_dir' "${AGENCY_BFF_DIR:-$HOME/.agency-bff}/config.json")/logs"
find "$LOG_ROOT" -type f -maxdepth 4 -print
```

The BFF stream is `$LOG_ROOT/bff-YYYY-MM-DD.log`. Device streams are
`$LOG_ROOT/devices/<device-id>/YYYY-MM-DD.jsonl`.

## 2. Establish the incident and upload windows

Ask for the approximate incident time and timezone. Convert it to UTC and expand the event window
by at least five minutes on both sides. Filter device records by their `ts` value: the query's
default `clock=device` preserves the historical `since`/`until` behavior. Include the reported
device and, when known, its session:

```sh
START='2026-07-12T00:00:00Z'
END='2026-07-13T00:00:00Z'
curl -fsS -D /tmp/bff-log-headers \
  -H "Authorization: Bearer $PAIRING_TOKEN" \
  "$BFF_URL/v1/logs?device=$DEVICE_ID&session=$SESSION_ID&since=$START&until=$END&limit=500"
```

`ts` is the phone's event time and can be skewed. `received_at` is when the BFF received the batch,
not when the phone event occurred. If the phone was offline, incident records may arrive hours or
days later. For BFF access and service records, filter the server-clock `time` separately:

```sh
jq -c --arg start "$START" --arg end "$END" 'select(.time >= $start and .time <= $end)' "$LOG_ROOT"/bff-*.log
```

If device records are missing from the event-time query, use a wider window covering when the
phone reconnected or logs were uploaded, and query by BFF receive time:

```sh
UPLOAD_START='2026-07-12T00:00:00Z'
UPLOAD_END='2026-07-15T00:00:00Z'
curl -fsS -D /tmp/bff-log-headers \
  -H "Authorization: Bearer $PAIRING_TOKEN" \
  "$BFF_URL/v1/logs?device=$DEVICE_ID&session=$SESSION_ID&clock=received&since=$UPLOAD_START&until=$UPLOAD_END&limit=500"
```

Compare each result's `ts` and `received_at` to distinguish a delayed upload from an event that
occurred during the later window. `seq` is the reliable order within one device stream even if its
clock changes.

## 3. Correlate the run and session

Once any record gives you a `run_id`, one recursive search shows both the device action and BFF
watch/relay lifecycle. Substitute the reported id:

```sh
grep -RhF '"run_id":"run-demo-123"' "$LOG_ROOT"
```

If there is no run id, repeat with the exact `session_id`. Keep `received_at`/`time`, `device_id`,
and `seq` visible when manually ordering results.

New app requests carry an `X-Request-ID`; the BFF echoes it and includes it on request-start and
request-completion records. The app's matching shipped entries carry that same `request_id`. Query
the remote store by it when a particular HTTP attempt is in question:

```sh
curl -fsS -D /tmp/bff-log-headers \
  -H "Authorization: Bearer $PAIRING_TOKEN" \
  "$BFF_URL/v1/logs?request=request-id-from-access-log&clock=received&limit=500"
```

The array response stays backwards compatible. Follow `X-Log-Next-Cursor` while
`X-Log-Has-More: true`; inspect `X-Log-Skipped-Records` if records appear missing. The BFF emits a
single warning with the aggregate skipped count, so check the matching raw JSONL file if that count
is nonzero. A `bff.endpoint_started` without a completion record can identify a request that was
still hung when the file was inspected; its completion is written only when the handler returns or
its client context ends.

## 4. Symptom queries

### Send failed

Start with warning/error entries from the request-related app categories, then take their run or
session id into section 3. Categories are `conversation`, `bff`, `media`, and `voice`:

```sh
jq -c 'select((.category == "conversation" or .category == "bff" or .category == "media" or .category == "voice") and (.level == "error" or .level == "warning")) | {received_at,device_id,seq,session_id,run_id,request_id,message}' "$LOG_ROOT"/devices/*/*.jsonl
```

### Push missing

Search the run id across both streams using section 3. Expect device send/watch registration and
BFF `watch started`, `watch terminal`, and `watch push completed` records. Missing terminal state
points to gateway observation; a `failed` count points to APNs/device registration.

### Upload failed

Show upload storage records and HTTP status for the file route:

```sh
jq -c 'select((.msg // "") | test("upload|http")) | select(.path == "/v1/files" or ((.msg // "") | test("upload"))) | {time,level,msg,event,route,path,query,status,status_class,duration_ms,response_bytes,upload_id,bytes}' "$LOG_ROOT"/bff-*.log
```

`413` means the configured upload cap. `401` points to pairing credentials. `500` plus
`upload failed` points to host storage or permissions.

### Pairing or authentication issue

```sh
jq -c 'select(((.msg // "") | test("authentication failed")) or (.path == "/v1/bff/capabilities" and .status >= 400)) | {time,level,msg,path,status,remote_ip}' "$LOG_ROOT"/bff-*.log
```

Repeated `401` means the app's bearer token does not match the configured token hash. `429` means
the per-IP failed-auth window is active. Never ask the user to paste a token into a log or ticket.

### App cannot reach companion

The BFF cannot receive a request that never reaches it. Compare device-side `bff` entries with the
BFF access stream in the same server-clock window:

```sh
jq -c 'select(.category == "bff") | {received_at,device_id,seq,level,message}' "$LOG_ROOT"/devices/*/*.jsonl
```

If the device records connection failures and the BFF has no matching access record, inspect DNS,
TLS, reverse proxy, Tailscale/LAN reachability, and whether the service is running. If access is
present, use its HTTP status to stay at the application/auth layer.

### Filter by device metadata

Use hardware, OS, locale, or timezone to separate a multi-device report:

```sh
jq -c 'select(.device_model_id == "iPhone14,5" or .locale == "de-DE") | {received_at,device_id,device_model,device_model_id,os_version,locale,timezone,message}' "$LOG_ROOT"/devices/*/*.jsonl
```

## 5. Retention caveats

Defaults are 14 days and 256 MiB for the entire log tree. Age eviction happens before
oldest-first size eviction, so an old incident may already be gone even if the tree is small. If GC
unlinks the active BFF daily file on POSIX, the writer reopens the path on its next write, so later
records remain retrievable. Records written before the unlink are still lost with that file.
Confirm `limits.logs_max_age_days` and `limits.logs_max_total_bytes` in config before concluding
that missing records were never emitted.
