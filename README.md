# Agentic Agency BFF - binary releases

Prebuilt binaries of `agency-bff`, the companion service for the Agentic Agency Chat iOS app.
It is installed on the same host as the Hermes Agent Gateway and is required for the app's
live mode.

This repository distributes **binaries only** (no source). Its audience is the AI agent that
installs or updates the BFF on the gateway host, following the setup runbook that pointed it
here. That runbook (`bff/SETUP.md` in the app repository) remains the authoritative
step-by-step guide; this README only covers getting the right binary.

## Versioning

- Every commit here is one BFF release, tagged `v<version>` (e.g. `v0.8.1`).
- The head of `main` is always the latest release.
- The installed binary reports its version with `agency-bff version`; the app enforces a
  minimum version, so always install from the latest commit unless instructed otherwise.
- Apps using durable GPT-Live voice-message replies require BFF `0.24.0` or newer.
- Version 0.19.0 adds an optional Codex-subscription Realtime broker. It stays disabled unless
  `realtime_subscription.hermes_python` is explicitly set in the BFF config. Voice preview also
  requires the Hermes environment's `websockets` 15.0+ package. See `SETUP.md` and `UPDATE.md`.
- Version 0.20.0 adds experimental private GPT-Live status, WebRTC SDP exchange and typed
  no-tools control classification under `/v1/live/subscription/`. It uses the same opt-in Hermes
  interpreter, fixed private model/Cove voice and fixed Codex-only classifier. OAuth stays on
  the host, and returned control candidates never execute actions in the BFF. Private
  `preview_ready=false` describes the absent host WAV route; native clients capture audio.
- Version 0.21.0 adds request-ID echo, request/device/session/run correlation in access and
  lifecycle records, richer diagnostic metadata and chunk storage, and cursor-paginated
  `/v1/logs` queries filtered by session, run, request, or device. Query pagination is reported in
  `X-Log-Has-More`, `X-Log-Next-Cursor`, and `X-Log-Skipped-Records`. Request bodies and
  authorization headers are not logged, but paths, queries, and diagnostic messages may contain
  sensitive information; treat the log tree as private. See `LOGGING.md` and `DEBUGGING.md`.
- Version 0.22.0 adds authenticated `/v1/live/subscription/voices` with Cove, Juniper, Maple,
  and Breeze. Legacy private status remains Cove-only for strict older clients, including
  TestFlight build 91; standard Realtime routes stay unchanged. Bounded new-voice provider probes
  decoded nonzero audio without microphone input; complete preview utterances, perceived voice
  identity, and German suitability remain listening checks.
- Version 0.23.0 expands the private voice catalog to Cove, Juniper, Maple, Breeze, Vale, Sol,
  Ember, Spruce, and Arbor. Each new voice passed one bounded native provider audio probe
  without microphone input. Legacy Cove-only status and standard Realtime endpoints remain
  unchanged. This evidence does not establish perceived voice identity or German quality.
- Version 0.24.0 adds bounded `/v1/voice-replies` jobs with immutable request-time voice and
  final-message bindings, ten-minute rendering leases, cancellation, and create-only AAC
  publication into the existing outbox. The iOS app renders through GPT-Live; the BFF stores
  identifiers/digests and audio, without transcripts, speech generation, or a Piper/API-key
  fallback. Unfinished rendering resumes while the app is foregrounded. See
  [VOICE_REPLIES.md](VOICE_REPLIES.md) for API, storage limits, and recovery.
- Subscription voice uses the existing Hermes Codex credential resolver and has no API-key fallback.
  Local credential readiness does not establish model entitlement, quota, or billing treatment.

## Contents

- `agency-bff-<os>-<arch>` - static binaries for `darwin`/`linux` (`arm64`, `amd64`) and
  `windows` (`amd64`).
- `SHA256SUMS` - checksums for all binaries in this release.
- `SETUP.md`, `UPDATE.md`, and `UNINSTALL.md` - current agent-facing service runbooks.
- `LOGGING.md` and `DEBUGGING.md` - request correlation, remote log query, and incident workflows.
- `VOICE_REPLIES.md` - durable native voice-reply coordination and publication API.
- `THIRD_PARTY_NOTICES.md` - third-party notices that must accompany the subscription broker.
- `deploy/` - service templates: `space.rath.agency-bff.plist` (macOS launchd, RunAtLoad +
  KeepAlive) and `agency-bff.service` (Linux systemd user unit). Running the BFF under one of
  these is a required part of the install, so it survives crashes and reboots.

## Install / update (summary)

1. Pick the binary for the host platform (`uname -sm`), download it from this repository, and
   verify its checksum against `SHA256SUMS`.
2. Install it as `~/.local/bin/agency-bff` and make it executable (`chmod +x`).
3. First install: run `agency-bff init` and follow the runbook for configuration. Then set up
   the keep-alive service from `deploy/` (do not leave it running only in a foreground shell).
4. Update: replace the binary, restart the service, and confirm the new version with
   `agency-bff version` and `GET /health`.
