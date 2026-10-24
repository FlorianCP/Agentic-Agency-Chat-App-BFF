# Agency BFF - Update Runbook (for agents)

**Audience:** an AI agent updating an existing `agency-bff` installation from the public release
repository at https://github.com/FlorianCP/Agentic-Agency-Chat-App-BFF. This is a production
service. Inventory first, preserve the keep-alive setup, and keep the previous binary until the
new version is verified. Read ALL version-specific sections newer than the installed version
through the target version before making changes. Sections marked **(required)** are mandatory
migrations, not optional features.

## 1. Inventory the host and current service

```sh
uname -sm
command -v agency-bff
agency-bff version
launchctl print gui/$(id -u)/space.rath.agency-bff 2>/dev/null | head
systemctl --user status agency-bff 2>/dev/null | head
```

Map `Darwin` to `darwin`, `Linux` to `linux`, `arm64` or `aarch64` to `arm64`, and `x86_64` to
`amd64`. Record the installed binary path and whether launchd or systemd owns the process. Do not
replace a working binary until the matching download and checksum are ready. Record the target
version from the public release README or selected release tag, then read ALL sections for
versions newer than the installed one through that target, including every **(required)** section.
For example, a 0.27.2 to 0.28.0 update must complete section 18. Prepare its configuration and
plugin installation before the restart in section 3; obtain explicit operator approval before
any production update, configuration change, or restart.

## 2. Download and verify the matching release

Set `VERSION` to the target version (without the `v` prefix) recorded from the public release
README or selected release tag, then download the binary and checksum file from that same tag.
The examples use a temporary directory so an incomplete download never touches the service.

```sh
: "${VERSION:?Set VERSION to the target release version without the v prefix}"
OS=darwin       # darwin or linux, from the inventory
ARCH=arm64      # arm64 or amd64, from the inventory
REPO=https://github.com/FlorianCP/Agentic-Agency-Chat-App-BFF
TMP=$(mktemp -d)
curl -fL "$REPO/raw/refs/tags/v$VERSION/agency-bff-$OS-$ARCH" -o "$TMP/agency-bff"
curl -fL "$REPO/raw/refs/tags/v$VERSION/SHA256SUMS" -o "$TMP/SHA256SUMS"
(cd "$TMP" && grep "agency-bff-$OS-$ARCH" SHA256SUMS | sed "s#agency-bff-$OS-$ARCH#agency-bff#" | shasum -a 256 -c -)
chmod 755 "$TMP/agency-bff"
"$TMP/agency-bff" version
```

On Linux, use `sha256sum -c -` instead of `shasum -a 256 -c -` if needed. Stop if the checksum
or version does not match. Never install an unverified binary.

## 3. Replace safely and restart the existing service

Set `INSTALLED` to the path found in step 1. Keep a rollback copy until every verification passes.

```sh
INSTALLED=$(command -v agency-bff)
cp -p "$INSTALLED" "$INSTALLED.previous"
cp "$TMP/agency-bff" "$INSTALLED.new"
chmod 755 "$INSTALLED.new"
mv "$INSTALLED.new" "$INSTALLED"
```

After the operator approves the restart, restart the companion service using the manager already
installed. If a required migration also needs a gateway restart (section 18), obtain approval for
that restart separately and complete it too:

```sh
# macOS launchd
launchctl kickstart -k gui/$(id -u)/space.rath.agency-bff

# Linux systemd user service
systemctl --user restart agency-bff
```

Do not replace launchd/systemd with a foreground process. Confirm the existing launchd plist still
has `RunAtLoad` and `KeepAlive`, or the systemd unit still has `Restart=` and remains enabled.

## 4. Verify before removing the rollback copy

```sh
agency-bff version
curl -fsS http://127.0.0.1:8643/healthz
launchctl print gui/$(id -u)/space.rath.agency-bff 2>/dev/null | head
systemctl --user is-active agency-bff 2>/dev/null
```

Confirm `agency-bff version` reports the requested version, `/healthz` returns status `ok` with
that version, and the keep-alive manager reports the service running. Also check its recent logs
for startup or configuration errors and verify the externally reachable companion URL if the user
permits it.

After all checks pass, remove the temporary directory. Keep `agency-bff.previous` until the user
accepts the update or through the next normal service check, then remove it.

## 5. Optional subscription voice in 0.20.0

The update does not enable subscription calls. Existing configs omit
`realtime_subscription.hermes_python`, which is equivalent to the disabled empty default. Enable
the feature only when the operator requests it:

1. Identify the absolute Python executable used by this Hermes installation. Do not rely on
   `PATH`, install a second Python runtime, or change the gateway.
2. Confirm that executable provides Hermes' Codex credential resolver. Voice preview also needs
   the existing environment's `websockets` 15.0+ sync client. The BFF does not install or upgrade
   packages.
3. Back up `~/.agency-bff/config.json`, then set only
   `realtime_subscription.hermes_python` to that absolute executable path. Preserve the other
   fields and keep the file mode at `0600`; never print or copy its contents into logs or chat.
4. Restart only the existing BFF service. The gateway does not need a config change or restart.
5. Check authenticated `GET /v1/realtime/subscription/status`. `ready` means the local Codex
   credential resolver returned a supported subscription credential; `preview_ready` also checks
   the WebSocket dependency. Neither flag establishes model entitlement, quota, or billing
   treatment, and the status request makes no inference call.

Version 0.20.0 adds separate experimental private GPT-Live status, SDP call and typed control
routes under `/v1/live/subscription/`, using that same opt-in Hermes interpreter. Readiness requires
`ready` and `control_ready`; `preview_ready=false` describes the absent host WAV route, while the
native client captures preview audio. Status makes no inference call. The private model/voice and
Codex-only classifier are fixed, with no tools, API-key route or provider fallback. Cove is the
initial testing restriction. Current status does not establish account model entitlement or billing.

To roll back, restore the config backup or clear `realtime_subscription.hermes_python`, then
restart only the BFF service. No Hermes gateway files are involved.

## 6. Request-correlated diagnostics in 0.21.0

Version 0.21.0 adds request-ID echo and validated device/session/run context to access and lifecycle
records, plus filtered cursor-paginated queries for remote device logs. No configuration change or
feature enablement is needed. Existing device log uploads remain compatible. Follow
[`LOGGING.md`](LOGGING.md) for request IDs, query selectors, pagination headers, and full-fidelity
log handling. Version 0.20.0 does not provide these additions.

## 7. Roll back if verification fails

```sh
cp "$INSTALLED.previous" "$INSTALLED.new"
chmod 755 "$INSTALLED.new"
mv "$INSTALLED.new" "$INSTALLED"
launchctl kickstart -k gui/$(id -u)/space.rath.agency-bff 2>/dev/null || systemctl --user restart agency-bff
agency-bff version
curl -fsS http://127.0.0.1:8643/healthz
```

Report the failed version, the verification error, and the restored version. Do not delete the
failed binary or logs until the cause is understood.

## 7. Additive GPT-Live voice catalog in 0.22.0

Version 0.22.0 adds authenticated `GET /v1/live/subscription/voices` for new clients. The legacy
private `/status` schema and its Cove-only `voices` field stay unchanged so TestFlight build 91
can still pass its strict readiness validation. Standard Realtime routes remain unchanged.
The new app requires BFF 0.22.0; keep the previous binary for rollback. The catalog offers Cove,
Juniper, Maple, and Breeze. Bounded provider probes passed startup and nonzero decoded audio
without microphone input; complete preview utterances, perceived identity, and German suitability
remain listening checks.

## 8. Expanded verified GPT-Live catalog in 0.23.0

Version 0.23.0 expands the separate private voice catalog to Cove, Juniper, Maple, Breeze, Vale,
Sol, Ember, Spruce, and Arbor. Each newly admitted voice passed a bounded native provider audio
probe with no microphone input. This establishes feasibility, not perceived voice identity or
German quality. Legacy
Cove-only private status and standard Realtime endpoints remain unchanged. The app requires
BFF 0.23.0; retain the previous binary for rollback.

## 9. Durable GPT-Live voice-reply coordination in 0.24.0

Version 0.24.0 adds authenticated `/v1/voice-replies` registration, lookup, claim, release,
cancel, and audio-publication routes. It stores small immutable coordination records under
`data_dir/state/voice-replies` and app-rendered AAC under `files_dir/outbox/voice-replies`.
No configuration migration, new Python dependency, gateway history mutation, or API-key
fallback is introduced. Existing Realtime/private call routes retain their contracts.

The original durable-job app requires BFF 0.24.0. Retain the previous binary for rollback; rollback disables the new
job API and does not remove its files. Verify the new binary version and authenticated health
before use. Do not create real rendering jobs merely as an update smoke. See
[VOICE_REPLIES.md](VOICE_REPLIES.md) for lease, retention, and foreground-resume limits.


## 10. Optional voice reservations in 0.25.0

Version 0.25.0 adds `reply_mode:"optional"` to voice-reply registration. These jobs start
reserved, remain hidden from ordinary collection GET, and consume the same bounded unresolved
capacity as required replies. Recovery clients opt in with `include_reserved=true`. Only claim
a reservation after canonical final assistant history includes its exact registered managed
media link; cancel an unused reservation after canonical completion without that link. The
BFF stores no transcript and does not synthesize speech or decide whether an answer needs audio.

The optional-reply app originally required BFF 0.25.0. No configuration or Python migration is needed. Required
registration, leases, publication, and Realtime/GPT-Live call routes remain compatible. Retain
the previous binary for rollback; rolling back to 0.24.0 disables optional-reservation clients.
Verify the new binary version and authenticated health; avoid creating real jobs as an update
smoke. See [VOICE_REPLIES.md](VOICE_REPLIES.md) for the exact API and retention limits.


## 11. Bounded optional lifecycle in 0.25.1

The app now requires BFF 0.25.1. Unclaimed optional voice reservations expire to cancelled
seven days after creation. Their managed-reply selection and recovery window ends at that
deadline. Cancelled optional metadata is removed seven days after cancellation, preserving
idempotency during that window while preventing ordinary text turns from accumulating
180 days of tombstones. Late access uses the original expiry deadline, rather than extending
retention. Register/Get/List consistently apply expiry; registration prunes eligible metadata
before checking capacity. Uncancelled claimed optional jobs and all required jobs retain their existing lifecycle. No
configuration migration or service dependency is introduced. See [VOICE_REPLIES.md](VOICE_REPLIES.md).


## 12. Account subscription usage in 0.26.0

The app now requires BFF 0.26.0. The additive authenticated `GET /v1/usage` route reads real
Codex account quota through the existing opt-in Hermes interpreter. Capability
`subscription_usage` reflects interpreter configuration only. No config migration is needed.
OAuth refresh remains owned by Hermes; no API-key fallback is allowed. Update only the BFF
with its existing backup/rollback procedure, then inspect the sanitized quota route without
creating a model turn. See [USAGE.md](USAGE.md) for the exact window and freshness contract.


## 13. Correct pooled account quota in 0.26.1

The app now requires BFF 0.26.1. Protocol 2 reports all configured official Codex OAuth
accounts separately, including quota-exhausted accounts in cooldown. Identity hashes survive
token refresh and round-robin selection; numeric quotas are not combined. The reader uses
read-only account discovery and exact owner-targeted refresh without rotating selection.
No configuration migration or gateway restart is needed. Update only the BFF using the existing
backup procedure, then compare anonymous account IDs and per-account reset times through
`GET /v1/usage`. Rolling back to 0.26.0 restores protocol 1 and disables pooled-quota clients.
See [USAGE.md](USAGE.md) for bounds, refresh, and availability.

## 14. Hermes 0.21.5 GPT-Live compatibility in 0.26.2

Version 0.26.2 repairs the private GPT-Live account identity dependency in call
signaling and readiness. Preserve the existing Hermes installation and config;
replace and restart only the BFF after explicit production approval. Verify
`GET /v1/live/subscription/status`, not only the standard Realtime status route,
then one authorized managed spoken reply on device. Status may refresh OAuth;
include this verification in the approved operation. See `VOICE_REPLIES.md`.

## 15. Claude subscription usage in 0.27.0

Version 0.27.0 adds authenticated, read-only `GET /v1/usage?provider=anthropic` alongside the
existing Codex usage route. It reads existing official Claude OAuth credentials and returns
sanitized credential-slot/window usage without refreshing tokens or changing credential selection;
there is no API-key or custom-route fallback. Anthropic slot labels do not establish distinct
provider accounts, so do not add their percentages together. `capabilities.subscription_usage`
still means only that the Hermes interpreter is configured, not that credentials or quota are
available.

No configuration migration or Hermes restart is needed. After an approved BFF-only update and
backup, verify one GET through the existing private pairing bearer. Inspect only `provider`,
availability, anonymous credential-slot labels, window statuses, remaining percentages, and
reset/fetch times. Do not create a chat turn for verification. Rolling back to 0.26.3 removes
Claude usage while retaining the existing Codex route. See [USAGE.md](USAGE.md).

## 16. Read-only Hermes usage compatibility in 0.27.1

Version 0.27.1 reads usage policy without PyYAML through Hermes' ruamel.yaml parser and fails
closed when Hermes config is unreadable or invalid. No configuration change is needed.

## 17. Hermes data-directory suffix compatibility in 0.27.2

Version 0.27.2 honors literal `HERMES_DATA_DIR_SUFFIX` homes for usage config, auth, and
profile-to-global fallback while explicit `HERMES_HOME` takes precedence. Hermes v0.21.x ignores
`HERMES_DATA_DIR_SUFFIX`, so leave it unset in the BFF service environment on v0.21.x hosts or
the BFF may read a different home than Hermes. No configuration change is needed.

## 18. Hermes plugin channel in 0.28.0 (required)

Updating to BFF **0.28.0 or newer REQUIRES installing and enabling the `agentic_agency`
Hermes plugin and wiring its shared secret**. Without it, model attribution and live thinking
stay missing on Hermes v0.22.3+. Updating the BFF binary alone does not complete this migration.
See [PLUGIN.md](PLUGIN.md#configuration-and-installation) for the private-config script,
channel contract, and update details, and [the public plugin README](https://github.com/FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin)
for plugin installation details.

Plugin release for 0.28.x: **v0.1.0**, commit
`0032b87f68782d573114a8f9fbca04ae7afe76a7`. When updating straight to 0.29.0 or newer, install
the plugin release named in section 21 in step 4 instead and verify its version there.

1. Inventory every Hermes profile the BFF serves, including multiplexed profiles. Resolve each
   profile's `HERMES_HOME` explicitly. Obtain the operator's explicit approval for each production
   configuration change, plugin installation, BFF update, and service restart before performing it.
2. Back up existing private configs. Generate one shared secret with `openssl rand -hex 32`,
   using the script in [PLUGIN.md](PLUGIN.md#configuration-and-installation) to avoid printing it
   or recording it in shell history. Set BFF `plugin.secret` in `~/.agency-bff/config.json`
   (or the installation's configured config path); keep the file at `0600`. The plugin secret
   must differ from the app pairing token.
3. For EVERY served profile, write `$HERMES_HOME/agentic_agency/plugin.json` with mode `0600`
   in an `agentic_agency` directory with mode `0700`:
   `{"bff_url":"http://127.0.0.1:8643","secret":"<same shared secret>"}`.
   Use the actual local BFF port. Reuse the same BFF secret across these profiles; do not rerun
   the secret-generating script separately for each profile and invalidate earlier profiles.
4. With `HERMES_HOME` set to each intended profile, install the pinned release and enable it:

   ```sh
   hermes plugins install FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin --ref 0032b87f68782d573114a8f9fbca04ae7afe76a7 --no-enable
   hermes plugins doctor agentic_agency --ci
   hermes plugins enable agentic_agency
   ```

   In each profile's Hermes config, merge `agentic_agency` into `plugins.enabled` and set
   `plugins.stream_reasoning_deltas: true`, preserving other plugins and settings.
5. Restart the BFF to load `plugin.secret` and restart every affected Hermes gateway to load
   its plugin, only after the operator approves the respective restarts. Preserve existing
   keep-alive service managers. Do not roll back or replace Hermes as part of this migration.
6. Using the existing app pairing bearer through a private secret-handling path, inspect
   `GET /v1/bff/capabilities`. Require `plugin.connected: true` and plugin version `0.1.0`.
   Verify EVERY expected profile in `plugin.profiles` has version `0.1.0`,
   `reasoning_stream: true`, and a fresh `last_seen` (within 120 seconds); require
   `plugin.reasoning_stream: true` too. The top-level connected flag means ANY profile is
   connected, so it does not prove that all multiplexed profiles are ready. Missing or stale
   profiles fail verification. Never print the bearer or plugin secret.

Retain the previous BFF binary until section 4 and these plugin checks pass. For later plugin
updates, follow [PLUGIN.md](PLUGIN.md#update-and-uninstall) and the public plugin README for
intentional pinned reinstallation and coordinated secret rotation across all served profiles.

## 19. Runbook corrections in 0.28.1

No behaviour change; runbook corrections only. The required 0.28.0 migration in section 18 still applies.

## 20. Empty inbox fix in 0.28.2

`GET /v1/inbox` now returns `"entries": []` instead of `null` when nothing is new, which the app
could not read. No configuration change; replace the binary as in sections 2-4.

## 21. Scheduled and proactive deliveries in 0.29.0 (plugin 0.2.0)

BFF 0.29.0 pairs with Hermes plugin **v0.2.0**, commit `f8871f58b66ceb7f45afd1e203b3f1bde47685c8`, on Hermes v0.22.5
or newer. Scheduled jobs created from an app chat then deliver into that chat with a push, jobs
without a target land in an "Assistant" inbox chat, and the agent can message the user on its
own (`agentic_agency_notify`). See [PLUGIN.md](PLUGIN.md#scheduled-and-proactive-deliveries-wp66).

No new secret or BFF config is needed; the 0.28.0 plugin channel from section 18 is reused.

1. Obtain the operator's explicit approval for the BFF update, each plugin reinstallation, and
   each gateway restart on a production host.
2. Replace the BFF binary as in sections 2-4.
3. For EVERY served profile (set `HERMES_HOME` explicitly), reinstall the pinned plugin release:

   ```sh
   hermes plugins install FlorianCP/Agentic-Agency-Chat-App-Hermes-Plugin --ref f8871f58b66ceb7f45afd1e203b3f1bde47685c8 --no-enable --force
   hermes plugins doctor agentic_agency --ci
   hermes plugins enable agentic_agency
   ```

   If a profile restricts toolsets in `platform_toolsets`, merge `agentic_agency` into the
   relevant lists and keep `cronjob` for `api_server`; never replace the operator's lists.
4. Restart every affected Hermes gateway after approval. The platform loads only at gateway start.
5. Verify `GET /v1/bff/capabilities`: `features.deliveries: true`, and for EVERY expected profile in
   `plugin.profiles` version `0.2.0`, `delivery_platform: true` and a fresh `last_seen`.
   `inbox_session_id` stays null until the first delivery to the inbox creates that chat.
6. Optional live check: in an app chat, ask the assistant to send a short test message in one
   minute. It must arrive in that chat with a push and a "Scheduled" label. Remove the test job.

New private state: BFF `<data_dir>/state/deliveries.json` and `<files_dir>/outbox/deliveries/`;
plugin `$HERMES_HOME/agentic_agency/spool/`, `state.json` and lock files.

