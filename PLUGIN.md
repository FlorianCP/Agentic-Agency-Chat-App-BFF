# Hermes companion channel (WP56 first slice)

Release: BFF 0.28.0 with Hermes plugin 0.1.0.

The stdlib-only `agentic_agency` Hermes plugin restores model attribution and live
reasoning when the gateway omits them. The BFF continues to own app pairing,
files, push, and relay delivery. The plugin never receives the app pairing token.
See [the plugin README](../hermes-plugin/README.md) for its hooks and installation.

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
existing BFF configuration and keep both services on the same host.

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

```sh
hermes plugins install 'https://github.com/FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin.git' --ref FULL_40_CHARACTER_COMMIT --no-enable
hermes plugins doctor agentic_agency --ci
hermes plugins enable agentic_agency
```

Set `HERMES_HOME` explicitly to the intended profile before these commands.
Plain local paths are not Git install sources; see
[hermes-plugin/README.md](../hermes-plugin/README.md) for `file://` or local-copy
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
The reusable `requirePluginAuth` middleware is also the intended gate for WP66's
future `POST /v1/plugin/deliveries`; that route is not implemented in this slice.

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
