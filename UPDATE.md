# Agency BFF - Update Runbook (for agents)

**Audience:** an AI agent updating an existing `agency-bff` installation from the public release
repository at https://github.com/FlorianCP/Agentic-Agency-Chat-App-BFF. This is a production
service. Inventory first, preserve the keep-alive setup, and keep the previous binary until the
new version is verified.

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
replace a working binary until the matching download and checksum are ready.

## 2. Download and verify the matching release

Set the required version, then download the binary and checksum file from the same public tag.
The examples use a temporary directory so an incomplete download never touches the service.

```sh
VERSION=0.26.1
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

Restart only the companion service, using the manager already installed:

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
