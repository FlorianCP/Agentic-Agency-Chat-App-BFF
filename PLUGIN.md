# Hermes companion channel (WP56 first slice)

Release: BFF 0.29.3 with Hermes plugin 0.2.3 (the channel itself since BFF 0.28.0 and plugin 0.1.0).

The stdlib-only `agentic_agency` Hermes plugin restores model attribution and live
reasoning when the gateway omits them. The BFF continues to own app pairing,
files, push, and relay delivery. The plugin never receives the app pairing token.
See [the plugin README](https://github.com/FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin) for its hooks and installation.

## Configuration and installation

Generate a separate random secret with `openssl rand -hex 32`. Store it in both
locations, without printing it or adding it to shell history:

- BFF config: `"plugin": {"secret": "<random secret>"}`. The object is optional;
  absent means plugin routes return 404. A configured secret must be at least 32
  characters and must differ from the app pairing token.
- `$HERMES_HOME/agentic_agency/plugin.json` (0600, parent directory 0700):
  `{"bff_url":"http://127.0.0.1:8643","secret":"<same random secret>"}`.
  Resolve `HERMES_HOME` for the intended Hermes profile before writing this file.

For example, this local operator script updates both private config files without
showing the secret. Set `HERMES_HOME` to the intended profile first. Preserve
existing BFF configuration and keep both services on the same host. For multiple served or
multiplexed profiles, generate the shared secret only once, then write the same secret and
BFF URL into every profile's private plugin config. Do not rerun this secret-generating
script for each profile, which would invalidate the earlier profiles. Install/enable the
plugin and set `plugins.stream_reasoning_deltas: true` in every served profile.

```sh
python3 - <<'PY'
import json, os, pathlib, subprocess
bff = pathlib.Path(os.environ.get('AGENCY_BFF_DIR', '~/.agency-bff')).expanduser() / 'config.json'
hermes = pathlib.Path(os.environ['HERMES_HOME']).expanduser()
secret = subprocess.check_output(['openssl', 'rand', '-hex', '32'], text=True).strip()
config = json.loads(bff.read_text())
config['plugin'] = {'secret': secret}
plugin = hermes / 'agentic_agency' / 'plugin.json'
plugin.parent.mkdir(mode=0o700, parents=True, exist_ok=True)
os.chmod(plugin.parent, 0o700)
for path, value in [(bff, config), (plugin, {'bff_url': 'http://127.0.0.1:8643', 'secret': secret})]:
    temp = path.with_name(path.name + '.tmp')
    fd = os.open(temp, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    os.fchmod(fd, 0o600)
    with os.fdopen(fd, 'w') as stream:
        json.dump(value, stream, indent=2)
        stream.write('\n')
    os.replace(temp, path)
PY
```

Install the reviewed public plugin release for the intended profile, pinned by
its full commit (replace `FULL_40_CHARACTER_COMMIT` with that release's commit):

Current release: **v0.2.3**, commit `d3978f9fc56c55c437bf14d78450d45ee10ba6db`
(paired with BFF 0.29.3).

```sh
hermes plugins install 'https://github.com/FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin.git' --ref FULL_40_CHARACTER_COMMIT --no-enable
hermes plugins doctor agentic_agency --ci
hermes plugins enable agentic_agency
```

Set `HERMES_HOME` explicitly to the intended profile before these commands.
Plain local paths are not Git install sources; see
[the plugin README](https://github.com/FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin) for `file://` or local-copy
installation. In the profile's Hermes config merge:

```yaml
plugins:
  enabled:
    - agentic_agency
  stream_reasoning_deltas: true
```

Reload the BFF configuration by restarting the BFF and restart Hermes to load the
plugin. On a production host, every configuration write, BFF update/restart, and Hermes
restart needs explicit approval for that specific production action. Installation
instructions do not authorize those actions. No gateway rollback is part of this
channel setup.

## HTTP surface and security

`POST /v1/plugin/events` uses `Authorization: Bearer <plugin secret>`, independently
of app auth. The app pairing token is rejected here; the plugin secret is rejected
on every app-authenticated route. The fixed-size secret hash comparison is
constant-time. Keep this channel on localhost and keep config files private.
Secrets are never included in event bodies, capabilities, responses, or logs.
The reusable `requirePluginAuth` middleware also protects WP66's implemented
`POST /v1/plugin/deliveries` and `POST /v1/plugin/media` routes.

A batch contains `plugin_version`, optional `hermes_version`, `profile`, and an
`events` array of `hello` (also accepted as `heartbeat`), `runtime`, and `reasoning`
events. Hello reports `hooks` and `stream_reasoning_deltas`; runtime reports
`session_id`, `turn_id`, `api_call`, `provider`, `model`, and optional
`response_model`; reasoning reports `session_id`, `turn_id`, `seq`, and `delta`.
Limits are 256 events and 256 KiB per request. Responses: 204 accepted, 400 invalid
JSON/events, 401 wrong credential, 413 body/event limit exceeded. Whole batches
are validated before ingestion.

`GET /v1/bff/capabilities` remains app-authenticated and adds `plugin` with
`connected`, `version`, `hermes_version`, `profile`, `hooks`, `reasoning_stream`,
and `last_seen` (server Unix seconds). A `profiles` array retains each profile's
`profile`, `version`, `reasoning_stream`, and `last_seen`. Connected means any
profile sent hello within 120 seconds. The legacy top-level fields prefer a
connected `default` profile, then the first connected profile by name (or the
same preference among stale profiles when none is connected). Version and last_seen are null before hello. `run_runtime: true` is
advertised both alongside existing flat flags and under `features.run_runtime`.
Existing app decoders ignore these additive keys.

`GET /v1/runs/{run_id}/runtime` uses app pairing-token auth. It returns 200 with
`run_id`, `provider`, `model`, `attribution` (`plugin` or `gateway`), or 404 without
a receipt. Receipts are written atomically with 0600 permissions under
`<data_dir>/state/runtime_receipts.json`, alongside persisted watches. They survive
BFF restart and expire after seven days. Gateway receipts take precedence.

## Correlation and relay behavior

Runtime and reasoning require a nonempty `turn_id`. For each session, the first
event of an unseen turn binds to the oldest non-terminal watch without a turn.
Later events for that turn keep the same run, even while its terminal SSE is held
or the next run has already started. A new plugin turn proves execution, so stale
gateway queued state does not block binding. Registration order is restored from
watch persistence. Bindings are written atomically to
`<data_dir>/state/plugin_bindings.json`, survive restart, and are removed with the
watch (or expire after seven days).

Events with no unbound watch retain their turn identity in a ten-second buffer,
up to 2000 events per session and 128 unmatched sessions. Later watch registrations
drain buffered turns in arrival order, without assigning another turn's events to
an already-bound run. Older/excess events are dropped.

Plugin reasoning enters the normal live relay and replay path, preserving its
coalescing and truncation limits. Once native gateway reasoning is observed for a
run, further plugin reasoning is suppressed. Previously relayed deltas cannot be
retracted. Runtime uses the latest plugin model call. Missing/empty provider or
model in `run.completed` or `run.failed` carrying output is filled individually;
other gateway runtime fields are preserved. A connected plugin gets up to 1500 ms
to supply runtime before the terminal event is relayed. Fully populated gateway
runtime is never overwritten. Polling also records gateway runtime and terminal state.

## Update and uninstall

On update, rotate the secret in both files using the installation script, then
update/enable the repository plugin according to its README. Set
`plugins.stream_reasoning_deltas: true` and restart both services after obtaining
any required production approvals. Confirm plugin Connected/version in BFF
capabilities and exercise a watched run's relay and runtime endpoint.

On uninstall, run `hermes plugins disable agentic_agency` and
`hermes plugins remove agentic_agency` for the intended profile,
remove `$HERMES_HOME/agentic_agency/plugin.json`, and remove the `plugin` object
from BFF config. Remove `plugins.stream_reasoning_deltas` only if no other plugin
uses it. Restart the BFF and Hermes after the operator approves when on a production host. Remove
`runtime_receipts.json` and `plugin_bindings.json` when deleting BFF data, or retain
them for their normal seven day recovery window. Never delete other plugins' files or configuration.

## Scheduled and proactive deliveries (WP66)

Release: BFF 0.29.0+ with Hermes plugin 0.2.0+ (0.2.1 drops Hermes host notices), Hermes v0.22.5 or later. On older
hosts the plugin keeps its telemetry role and does not register the platform.

Hermes cron delivers only through messaging platforms, and the `api_server`
platform the app uses is request/response only. The plugin therefore registers
an `agentic_agency` messaging platform. Cron jobs (and the plugin's
`agentic_agency_notify` tool) deliver through it like through Telegram.

### Targets

- `agentic_agency:<hermes session id>` delivers into that app chat. A job created
  from an app chat without `deliver` (or with `origin`) is rewritten to the
  current chat by the plugin's `tool_request` middleware on `cronjob_manage`; an
  update is rewritten only for an explicit `origin`, so it never moves a job that
  another channel created.
- `agentic_agency` (bare) or `agentic_agency:inbox` delivers into the profile's
  "Assistant" inbox chat. The plugin creates that `api_server` session on first
  use and remembers its id in `$HERMES_HOME/agentic_agency/state.json`.
- Unknown, foreign-profile, or non-`api_server` session ids are rejected by the
  platform's target validation, so a job cannot write into another channel.
- Only cron results (and their attachments) and the notify tool become deliveries. Hermes'
  own host notices to the inbox home channel (gateway online/restart pings, warnings) are
  dropped by the adapter (plugin 0.2.1), since the app's users are not operators.

### Transcript row (source of truth for the timeline)

The adapter writes each delivery into the target session's Hermes transcript with
`SessionDB.append_completion_note` (idempotent per delivery id, refused during a
live turn and retried by the plugin's spool afterwards). The row is
`role=user`, `display_kind=agentic_agency_delivery`, and its content is a header
line followed by the delivered text, which may contain `media://` references:

```text
[Delivered by the assistant: scheduled job "Morning briefing"]
Good morning! ...
```

Agent-initiated messages use the header `[Delivered by the assistant]`. A file
attachment that Hermes hands over separately (cron `MEDIA:` tags) becomes its own
row with the header `[Delivered by the assistant: attachment]` and a `media://`
reference as its text. The
model sees the row on the next turn, so the agent can refer to what it sent. The
app reads it through the normal session history: `GET /api/sessions/{id}/messages`
exposes `display_kind`. The app strips the header and renders the row as an
assistant message labeled with the job name. Cron's own `Cronjob Response:`
wrapper is removed by the plugin before writing.

### Plugin to BFF

The plugin spools each delivery durably on disk
(`$HERMES_HOME/agentic_agency/spool/`, 0600) before `send()` returns, then a worker
writes the transcript row and notifies the BFF. Both steps are idempotent on the
delivery id and retried with backoff; Hermes cron itself is at-most-once.

`POST /v1/plugin/deliveries` (plugin secret, 16 KiB body limit):

```json
{
  "delivery_id": "8b1d...",
  "profile": "default",
  "session_id": "20261025_071500_ab12cd",
  "inbox": false,
  "source": "cron",
  "job_id": "a1b2c3",
  "job_name": "Morning briefing",
  "preview": "Good morning! ...",
  "created_at": 1761376500
}
```

- `delivery_id`: 1-128 characters `[A-Za-z0-9._:-]`. `source`: `cron` or `agent`.
  `job_id`/`job_name` are optional (absent for agent messages). `preview` is plain
  text, at most 512 characters; it is used only for the push body.
- Responses: 200 `{"delivery_id", "cursor", "duplicate": bool}`; a repeated
  delivery id returns the original cursor with `duplicate: true` and sends no push.
  400 invalid fields, 401 wrong credential, 404 plugin channel disabled, 413 too large.
- `notify` (optional bool, default true): `false` records the receipt, inbox entry
  and event but sends no push. Attachment rows use it so a text delivery and its
  files produce one banner.
- `POST /v1/plugin/media?name=<file name>` (plugin secret, raw body up to the BFF
  upload limit) copies a file into the media outbox and returns
  `{"ref": "media://deliveries/<id>/<name>"}`.

Media: Hermes v0.22.5 hands cron `MEDIA:` attachments to the adapter as separate
calls without job or execution identity, so the plugin cannot attach them to the
preceding text row. The plugin's prompt section therefore asks the agent to share
files through the app's `media://` outbox convention inside the answer text (one
atomic row). Stray `MEDIA:` attachments still arrive, each as its own attachment
row with `notify: false`.

The BFF keeps a receipt store `<data_dir>/state/deliveries.json` (0600, 500
receipts, 30 days). For each new delivery it:

1. appends an inbox entry `{"kind": "delivery", "delivery_id", "session_id",
   "profile", "status": "delivered", "completed_at", "cursor"}` (`run_id` is
   empty for deliveries; run entries keep `kind: "run"`);
2. broadcasts `delivery.received` on `GET /v1/events` with `delivery_id`,
   `session_id`, `profile`, `inbox`, `source`, `job_name`, `notify`;
3. pushes to every registered device with category `delivery`, top-level
   `session_id`, `delivery_id`, `profile`, title = job name (or "Assistant"), body =
   preview when the device opted into previews, else a static line. Pushes for the
   same job collapse (`apns-collapse-id`).

Anti-spam: at most one delivery push per job per 5 minutes and 20 delivery pushes
per hour overall. Held-back deliveries still create the inbox entry and event, so
the chat turns unread without another banner. Hermes already skips empty and
`[SILENT]` cron results.

Capabilities add `features.deliveries: true` and, per plugin profile,
`delivery_platform` (bool) and `inbox_session_id` (string or null), reported by
the plugin's `hello`.

### Delivery correlation and diagnostics (0.29.2 / 0.2.2)

`delivery.received` events carry `delivery_id`, `session_id`, `profile`, `inbox`, `notify`,
`source`, `job_name` and optional `job_id`. Delivery inbox entries and APNs payloads carry the same
stable `delivery_id`, session/profile and optional `job_id`; the app preserves them in diagnostics.
Media uploads may include `X-Delivery-ID`, `X-Session-ID`, `X-Delivery-Profile` and
`X-Delivery-Job-ID` headers for correlation. Each plugin HTTP attempt supplies `X-Request-ID`.
These additions are optional and older clients ignore them. See [LOGGING.md](LOGGING.md) for
content-free lifecycle events and [DEBUGGING.md](DEBUGGING.md) for reconstruction commands.

## System prompt sections (WP76)

The plugin registers `agentic_agency.delivery` (cross-platform schedule/media
guidance) and `agentic_agency.approvals` (plain-language explanation before a
potentially gated tool call). The latter renders only for `api_server` sessions,
including app chats and other API clients; Telegram, CLI, and cron delivery
sessions receive no approval section. Hermes v0.22.5 exposes platform metadata,
not an app-client marker. Sections are frozen into persisted session prompts,
so check rollout in a new chat. Approvals register independently of private
configuration and delivery setup, with their own failure warning and unload
cleanup. Delivery guidance retains delivery's host-API checks and cleanup. See [the plugin prompt contract and pinned source](../hermes-plugin/README.md#system-prompt-sections).

The plugin callback is the sole approval instruction source. No duplicate rule
is added to `agent-snippet.md`, no new BFF/event field is introduced, and approval
detection and approve/deny behavior remain Hermes-owned. Model compliance and
language quality need live checks; this guidance cannot guarantee an explanation.
