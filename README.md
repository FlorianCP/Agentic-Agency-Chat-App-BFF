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

## Contents

- `agency-bff-<os>-<arch>` - static binaries for `darwin`/`linux` (`arm64`, `amd64`) and
  `windows` (`amd64`).
- `SHA256SUMS` - checksums for all binaries in this release.
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
