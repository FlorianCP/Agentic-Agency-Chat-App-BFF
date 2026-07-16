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
VERSION=0.11.0
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

## 5. Roll back if verification fails

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
