# BFF logging reference

This document defines remote logging for Agency BFF. For an active incident, follow
[DEBUGGING.md](DEBUGGING.md).

The request-correlation and paginated-query contract below is available in BFF `0.21.0`. BFF
`0.20.0` accepts the existing log-ingest contract, but does not echo request IDs, retain the new
request/chunk fields, or provide the query headers and selectors described here. App uploads remain
additive and compatible across both versions.

## Architecture

The app's diagnostics shipper batches its structured entries and sends them to authenticated
`POST /v1/logs`. The BFF validates each batch, preserves the message exactly, stamps the server
receive time, re-marshals each entry, and appends one JSON object per line to a device-specific
daily file. Metadata fields have bounded lengths and control characters are replaced in metadata;
message content is never redacted or rewritten. Messages above 16 KiB are rejected with `413`.
The BFF's own `log/slog` stream is JSON written simultaneously to stdout and a daily file in the
same tree. Go's standard `log.Printf` is bridged through the configured slog handler and reaches
both sinks as well.

Every HTTP request gets a validated `X-Request-ID` or a generated opaque ID, echoed in the response.
When supplied, valid `X-Diagnostics-Device-ID`, `X-Session-ID`, and `X-Run-ID` headers are included
in the access record. The access stream records start and completion with method, normalized route,
escaped path, raw query, declared request length, response status, duration, response bytes,
cancellation, and write/panic details. SSE and long-poll requests have a start record while active
and a completion record when they return. Request bodies and authorization headers are never
logged by the access middleware. Path and query values are retained for incident reconstruction, so
future routes must not put credentials or other secrets there. Log call sites must not deliberately
include credential values. Because diagnostic messages and arbitrary error payloads are full fidelity,
there is no technical guarantee that every log is free of sensitive substrings; treat the complete
log tree as sensitive.

The app keeps a rolling local log, a bounded remote queue, and a bounded local-only shipper-health
file. Export includes the local log, queued device entries, health records, and retained corrupt
queue bytes. Health records explain local append, read, clear, and upload failures without being
fed back into the remote queue. Clearing diagnostics removes those files and resets loss counters;
the persistent per-device sequence remains monotonic and is shared by ordinary, crash, and health
entries. The Settings action then writes a fresh clear-success event.

The Settings toggle controls remote diagnostics collection and upload. While off, the local rolling
app log continues, but no new remote entries are queued and crash/MetricKit capture is stopped;
already queued entries remain on device. Turning it back on resumes capture and queue delivery.
The queue retries failed batches with backoff. If the BFF stored a batch but its HTTP response was
lost, the app can upload those entries again; `Store.Append` does not deduplicate, so duplicate
entries are possible under this at-least-once retry behavior. Queue or local-storage losses are
reported separately through loss markers and shipper-health records.

## Directory layout

All dates are UTC. `data_dir` comes from `config.json`.

```text
<data_dir>/logs/
|-- bff-2026-07-12.log
`-- devices/
    |-- device-0001/
    |   `-- 2026-07-12.jsonl
    `-- device-0002/
        `-- 2026-07-12.jsonl
```

Set the log root once when investigating:

```sh
LOG_ROOT="$(jq -r '.data_dir' "${AGENCY_BFF_DIR:-$HOME/.agency-bff}/config.json")/logs"
```

Read every retained record in server receive order:

```sh
jq -cs 'sort_by([.received_at // .time, .device_id // "", .seq // 0])[]' "$LOG_ROOT"/bff-*.log "$LOG_ROOT"/devices/*/*.jsonl
```

Find a known run across both device and BFF files:

```sh
grep -RhF '"run_id":"run-demo-123"' "$LOG_ROOT"
```

For an authenticated remote query, set `BFF_URL` and `PAIRING_TOKEN` in the operator's shell, then
query a bounded incident by at least one selector. The JSON response remains an array for existing
clients. `X-Log-Has-More`, `X-Log-Next-Cursor`, and `X-Log-Skipped-Records` report pagination and
corrupt lines without changing that array shape:

```sh
curl -fsS -D /tmp/bff-log-headers \
  -H "Authorization: Bearer $PAIRING_TOKEN" \
  "$BFF_URL/v1/logs?session=session-demo-456&clock=received&limit=500"
```

The query accepts one or more of `session`, `run`, `request`, and `device`; an unconstrained query
is rejected. `since` and `until` remain compatible with the historical device-clock (`ts`) behavior.
Set `clock=received` to filter and order by the BFF receipt time when device clocks may be skewed.
Historic device entries without `received_at` fall back to their `ts` value.
Pages return the newest matching entries, ordered oldest-to-newest within each page. Follow the
opaque `cursor` from `X-Log-Next-Cursor` until `X-Log-Has-More` is `false`. The cursor keeps one
receive-time snapshot and includes stable device/sequence/file-position tie-breakers so equal-time
entries are not skipped when another batch arrives during retrieval. The cursor does not deduplicate
entries that were stored again after an upload retry. For counts and chunk reconstruction, identify
entries by `(device_id, seq)`, then group distinct entries by `message_id` and order by
`chunk_index`; compare duplicate payloads and do not concatenate repeated copies of the same chunk.

## Device entry schema

Every device file is JSONL. One line is one server-remarshaled object.

| JSON key | Meaning | Example |
|---|---|---|
| `device_id` | Stable, validated device/install id and directory key. | `device-0001` |
| `seq` | Monotonic per-device sequence and authoritative local order. | `42` |
| `ts` | Device-clock timestamp, retained as supplied. | `2026-07-12T20:00:00.123Z` |
| `level` | App log severity. | `error` |
| `category` | App diagnostics category. | `conversation` |
| `app_version` | User-facing app version. | `2.4.0` |
| `app_build` | App build number. | `187` |
| `os_version` | Operating-system name and version. | `iOS 26.0` |
| `device_model` | Human-readable hardware name. | `iPhone 13` |
| `device_model_id` | Raw Apple hardware identifier. | `iPhone14,5` |
| `locale` | Device locale identifier. | `de-DE` |
| `timezone` | Device time-zone identifier. | `Europe/Berlin` |
| `message` | Full-fidelity diagnostic message. | `send failed: companion unavailable` |
| `request_id` | Optional originating app request correlation ID. | `af81f00d-91de-4f40-b876-56cd0c6f09d1` |
| `message_id` | Optional stable ID shared by chunks of one message. | `message-demo-123` |
| `chunk_index` / `chunk_count` | Optional zero-based chunk index and total chunk count. | `0` / `3` |
| `correlation_provided` | Optional marker that dedicated correlation fields are authoritative, even when empty. | `true` |
| `run_id` | Optional gateway run correlation id. | `run-demo-123` |
| `session_id` | Optional Hermes session correlation id. | `session-demo-456` |
| `received_at` | BFF receive time in UTC, stamped on ingest. | `2026-07-12T20:00:02.456789Z` |

## Wire contract

- Endpoint: `POST /v1/logs` with `Authorization: Bearer <pairing-token>` and
  `Content-Type: application/json`.
- Capability: authenticated `GET /v1/bff/capabilities` reports `"logs": true`.
- Body: one JSON array, 1 through 500 entries, at most 4 MiB total.
- Identity: every entry must have the same `device_id`, matching
  `^[a-z0-9][a-z0-9-]{7,63}$`.
- Metadata: every string except `message` is bounded to 256 bytes. `request_id` is bounded to 128
  safe characters. Device/session/run/message IDs use a bounded printable identifier alphabet.
- Content: message is preserved byte-for-byte as a JSON string, including newlines and control
  characters, up to 16 KiB. Larger entries receive `413` and are not silently truncated. Client
  JSON bytes are never appended directly; BFF re-marshalling prevents JSONL framing injection.
- Optional chunk fields: `message_id`, `chunk_index`, and `chunk_count` let the app split long log
  messages losslessly while keeping each entry under the per-entry cap. Query response entries
  include these fields and all device/app metadata.
- `correlation_provided: true` means the app resolved correlation for this entry and its dedicated
  `session_id`, `run_id`, and `request_id` fields are authoritative even when omitted/empty. New app
  entries set it; missing or false markers preserve legacy message-token fallback for unchunked
  entries. Chunks are never parsed individually for fallback correlation.
- Responses: `204` stored; `400` invalid JSON, empty batch, invalid id, or mixed ids; `401`
  unauthorized; `413` too many entries, an oversized body, or a message over 16 KiB; `500` storage
  failure.

Remote diagnostic messages and request paths/queries are full fidelity and may contain customer
content or PII. They remain within the customer's device and companion-host boundary. JSON
re-marshalling protects JSONL integrity, not privacy. The BFF access middleware does not log request
bodies or authorization headers. Do not deliberately log gateway keys, pairing tokens, APNs keys,
or other credentials, and do not place them in paths or queries. Arbitrary error/message text may
still contain sensitive values, so no content-level secrecy guarantee is made.

## Retention and GC

The existing startup and hourly GC walks the complete log tree, including BFF daily files and all
device streams. It removes over-age files first, then evicts the oldest files until the total is
within budget, and prunes empty device directories.

| Config key | Default | Meaning |
|---|---:|---|
| `limits.logs_max_age_days` | `14` | Maximum retained file age. |
| `limits.logs_max_total_bytes` | `268435456` | Maximum total log-tree size, 256 MiB. |

If GC unlinks the currently open BFF daily file on POSIX, `DailyWriter` detects the missing or
replaced pathname on the next write and reopens it, so later records remain retrievable. A record
written before an unlink is still lost with that file. Increase the size budget if current-day
eviction is possible in normal operation.

## Scheduled and proactive deliveries (BFF 0.29.2, plugin 0.2.2)

Delivery diagnostics contain identifiers, job names, counts, lengths, statuses and fixed reasons.
They never include message/preview text, file bytes, credentials, or exception text from the plugin.
The primary join key is `delivery_id`; also retain `session_id`, `profile`, `job_id` when known.
BFF records include `request_id` through `addRequestLogAttrs`, including asynchronous push completion.
The plugin generates an `X-Request-ID` for each HTTP attempt and logs it before sending.

| Component | Events | Additional fields |
|---|---|---|
| Plugin | `delivery.target_rewritten` | `action`, `old_target`, `new_target`, `reason`; session/profile/job when known, before a delivery id exists |
| Plugin | `delivery.inbox_created`, `delivery.target_resolved`, `delivery.spooled`, `delivery.spool_duplicate` | `previous_session_id` and reason for drain-time rotation, `target_kind`, `ordering_key`, `text_bytes`, `spool_count`, `attachment`, `notify`, `notify_reason` |
| Plugin | `delivery.send_accepted`, `delivery.send_rejected`, `delivery.notify_accepted`, `delivery.notify_rejected` | target kind, `requested_target`, `resolution_reason` (profile inbox, explicit session, compression continuation), or fixed rejection reason and error type |
| Plugin | `delivery.transcript_committed`, `delivery.transcript_deferred` | `message_id` on the real host commit, `outcome`, `stage`, `error_type`, `reason` (`active_turn_lease` or `append_refused`) |
| Plugin | `delivery.post_attempt`, `delivery.post_response`, `delivery.post_failed` | `endpoint` without query, `attempt`, `status`, `request_id`, `server_request_id`, `error_type` |
| Plugin | `delivery.retry_scheduled`, `delivery.attempt_failed`, `delivery.dropped` | attempt/budget, age/budget, `backoff_seconds`, `retry_at`, stage and fixed reason |
| Plugin | `delivery.media_rejected`, `delivery.media_uploaded` | fixed reason, error type, `bytes`, returned `ref`; never host paths |
| BFF | `delivery.accepted`, `delivery.duplicate`, `delivery.inbox_appended` | `cursor`, `duplicate`, `source`, `inbox`, `notify`, `job_name`, `preview_bytes` on acceptance |
| BFF | `delivery.rejected`, `delivery.persistence_failed`, `delivery.event_published` | fixed validation/read reason or storage error; cursor and `event_name=delivery.received` after broadcast |
| BFF | `push.delivery_suppressed`, `push.delivery_scheduled`, `push.delivery_completed` | suppression `reason`, limiter `identity`, `hourly_count`, `hourly_limit`, `job_window_seconds`, `retry_after_seconds`; device count or attempted/sent/failed/invalidated counts |
| BFF | `push.delivery_device_result`, `push.delivery_device_load_failed`, `push.delivery_device_removal_failed` | hashed `device_key` (never a token), environment, sent/invalidated, HTTP status/reason; storage error for device store failures |
| BFF | `delivery.media_stored`, `delivery.media_rejected`, `delivery.media_failed` | `ref`, `declared_bytes`, fixed reason, upload limit or storage error; correlation headers are logged only when their delivery/session/profile validators pass |
| App | `delivery.received`, `delivery.inbox_processed` | notify/inbox, cursor, refresh decision and previous foreground refresh |
| App | `delivery.push_received`, `delivery.push_tapped`, `delivery.push_routed` | source, category, presentation suppression, action and routing outcome |
| App | `delivery.routed`, `delivery.refresh`, `delivery.activity_recorded`, `delivery.unread_updated`, `delivery.pending_toast_resolved` | active/known chat, notify, toast/unread decisions, reason, history/catalog refresh decision, activity disposition |
| App | `delivery.transcript_decoded` | `display_kind=agentic_agency_delivery`, total `count`, last 20 numeric gateway ids in comma-separated `message_ids`; one record per nonempty fetch, join to plugin `delivery.transcript_committed` by session and message id |

App events are JSON inside the normal diagnostic `message`, with dedicated session correlation,
using `LogClient` and its existing local/remote sinks. They ship to `POST /v1/logs` when diagnostics
are enabled. Empty keys mean unavailable, not a fabricated profile/job. Hermes' messages API exposes
`display_kind` and numeric row id, but omits `display_metadata`; therefore transcript decode logs
join through the plugin's `message_id`, rather than inventing a delivery id from the numeric row id.
Events, inbox entries and APNs payloads retain `delivery_id`/`profile`/`session_id` and add `job_id`;
events also retain `job_name`.

Hermes v0.22.5 `hermes_logging.py` filters `gateway.log` to gateway component logger prefixes,
including `hermes_plugins`, and writes INFO by default. The delivery logger uses
`hermes_plugins.agentic_agency.delivery`, so these records reach `$HERMES_HOME/logs/gateway.log`
and `agent.log` without a custom handler, global level change or debug switch. Profile routing uses
Hermes' current profile context; background spool workers may log in the launch profile's directory,
so search all served profile homes and use each record's explicit `profile` field.

The host append API returns the stable row id for both an insert and an existing row. The first
successful append logs `written_or_duplicate`; retries after a durable `transcript_written` marker
log `duplicate`. A crash after the DB commit but before that marker cannot be distinguished from
a first write by this API. `message_id` still proves the exact committed row. A `delivery.refresh`
or activity record describes the reducer decision; subsequent history/organization logs prove its
completion. `event_published` proves a broadcast, not receipt by a disconnected phone. APNs `sent`
proves provider acceptance, not presentation on a device.
