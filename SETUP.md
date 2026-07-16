# Agency BFF — Setup Runbook (for agents)

**Audience:** you are an AI agent (typically the Hermes agent itself, running on the gateway
host) that a user has pointed at this file with a request like *"install the required companion
for my chat app's live mode."* Follow this runbook top to bottom. It tells you how to **(1) scan the machine
and tell the user whether installing is safe and what it means, (2) install, (3) configure,
(4) verify, and (5) hand the user app-side instructions.**

> **Status: v0.11.0 (2026-07-16).** Pairing, upload, media, foreground events, run relay,
> watches/inbox, remote logging, and optional APNs push are implemented. `upload`, `media`,
> `events`, `run_relay`, and `logs` are the required live-mode baseline; only `push` may
> truthfully report false.

**Ground rules while following this runbook:**

- The host is likely a **production machine** running the user's live Hermes agent. Be surgical.
- **Back up before you modify** anything that existed before you started (see section 7).
- Never print, log, or send anywhere: the gateway API key, the pairing token hash, or APNs keys.
  The **pairing token plaintext** is shown to the user exactly once, at generation.
- Do not stop, restart, or reconfigure the Hermes gateway itself. The BFF sits beside it.
- If any step fails in a way this runbook does not cover, stop and report rather than improvise
  destructive fixes.

---

## 1. What you are installing (tell the user this)

Before doing anything, give the user this assessment in your own words:

- **What it is:** the companion service required by live mode in their Agentic Agency ChatApp. One binary, one
  config file, one new background service listening on one port (default 8643) next to their
  Hermes gateway.
- **What it unlocks in the app:** push notifications when the assistant finishes while the app is
  closed; sending files/documents/voice memos to the assistant; receiving voice/video/file replies.
- **Why it must run here:** these features need software on the same machine as the agent. The
  gateway API has no push channel or file-attachment transport, so the companion closes those
  gaps locally.
- **What it gets access to:** its own data directory (uploads and reply media), a Hermes gateway
  API key you configure for it (used only over localhost, to watch run completion), and — only if
  the user wants push — an Apple push key they provide. It does **not** proxy or store chats.
- **What it does not change:** the app keeps talking to the gateway directly for sessions,
  history, and turn submission. If the BFF stops, history remains readable and accepted text
  turns reconcile, while streaming and upload-dependent sends report the companion outage.
- **The risk surface, honestly:** a new network service holding a gateway key. The gateway key is
  powerful (full agent access), which is why it stays in a `0600` file on this machine, is never
  sent to the phone, and why the BFF's own port should be exposed exactly as narrowly as the
  gateway already is. Uninstall is two commands (section 8).

Then run the scan (section 2) and give a concrete recommendation.

## 2. Scan the server

Collect the following. All commands are read-only.

```sh
# OS + architecture (pick the platform's equivalent)
uname -sm                    # macOS/Linux, e.g. "Darwin arm64"
sw_vers 2>/dev/null          # macOS version, if macOS

# Is a Hermes gateway actually running here, and where?
ps aux | grep -i "hermes" | grep -v grep
# API server port: check the hermes config/env for API_SERVER_* ; default is 8642
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8642/health   # expect 200

# Is the default BFF port free? (expect NO output)
netstat -an | grep -E '[.:]8643\b.*LISTEN'

# Existing install? (expect nothing on first install)
ls ~/.agency-bff 2>/dev/null; command -v agency-bff

# Disk headroom for the data dir (uploads + media; default caps total ~4 GB)
df -h ~

```

**Produce a recommendation** for the user from the results:

- **Safe to proceed** when: a Hermes gateway is running and healthy on this host, port 8643 (or an
  alternative you pick) is free, and there is at least ~5 GB disk headroom. Say which port and
  data directory you will use.
- **Ask first** when: an `~/.agency-bff` directory or running `agency-bff` already exists (this is
  an upgrade/repair, not an install — inventory it and confirm), or disk is tight, or you cannot
  verify the gateway's health endpoint.
- **Do not proceed** when: there is no Hermes gateway on this host (the BFF is useless elsewhere),
  or you cannot obtain a gateway API key legitimately (section 4), or the user declines after your
  assessment.

## 3. Install the binary

The BFF is a single static Go binary, `agency-bff`. Download it from the official public release
repository: https://github.com/FlorianCP/Agentic-Agency-Chat-App-BFF. The head of `main` is the
latest release and every release commit is tagged `v<version>`. Download
`agency-bff-<os>-<arch>` for this host (`uname -sm`), download `SHA256SUMS` from the same tag,
and verify the binary before installing it. Use the `deploy/` service templates in that same
public repository. For an existing installation, follow [UPDATE.md](UPDATE.md) instead.

Install location: `~/.local/bin/agency-bff` (create the directory if needed) or
`/usr/local/bin` if the user prefers system-wide. `chmod 755` the binary. On Windows:
`%LOCALAPPDATA%\agency-bff\agency-bff.exe`.

## 4. Configure

```sh
agency-bff init
```

`init` creates `~/.agency-bff/config.json` (`0600`, directory `0700`), the data/outbox
directories, and generates the **pairing token — it prints once; deliver it to the user verbatim
and do not store it anywhere else** (only its hash lives in the config).

Then fill in `~/.agency-bff/config.json` using the generated structure:

1. **Gateway:** `gateway.base_url` — the local API server address (normally
   `http://127.0.0.1:8642`). `gateway.api_key` — the Hermes `API_SERVER_KEY`. Hermes has exactly
   one key per gateway, so the BFF shares the key the app uses — get the user's explicit OK and
   explain: the key stays in a `0600` file on this machine, and rotating it later means updating
   the gateway, the BFF config, and the app together.
2. **Listen address:** default `127.0.0.1:8643` is only reachable from this machine. For the
   phone to reach the BFF, expose it **the same way the gateway is already exposed** (LAN bind
   like `0.0.0.0:8643`, a Tailscale interface address, or an existing reverse proxy). Do not
   invent a new exposure mechanism; mirror the gateway's. If the user's proxy terminates TLS,
   point it at the BFF; alternatively set `tls_cert_file`/`tls_key_file` for native TLS. The app
   requires `https://` for non-local addresses.
3. **Push (optional, can be added later):** needs an APNs auth key from the user's Apple
   Developer account — `.p8` file into `~/.agency-bff/` (`chmod 600`), plus `key_id`, `team_id`,
   `bundle_id` of the app build, and `environment` (`development` for Xcode installs,
   `production` for TestFlight/App Store). If the user has no Apple Developer membership, skip:
   everything else works, and `capabilities` will simply report `push: false`.
4. **Optional — gateway tools for API sessions (ASK THE USER; their call, either answer is
   fine):** check what the gateway currently exposes to API sessions:

   ```sh
   curl -s -H "Authorization: Bearer <API_SERVER_KEY>" http://127.0.0.1:8642/v1/toolsets
   ```

   On many installs the `terminal` and `code_execution`/`execute_code` tools are disabled or
   blocked for API sessions. Present the trade-off to the user and let them decide:

   - **Why enabling helps:** the features the BFF unlocks lean on the agent doing local work.
     With shell/code tools the agent can transcribe voice messages directly (e.g. run whisper
     on the uploaded audio), convert and inspect the files the user sends (unpack archives,
     process spreadsheets), and produce media replies into the outbox (TTS audio via Piper,
     rendered charts, video clips). Without them, the agent depends on whatever skills happen
     to cover these — voice transcription in particular becomes hit-or-miss.
   - **The risk, honestly:** enabling them means anyone holding the gateway API key can run
     arbitrary commands on this machine through an API session. The key is already powerful
     (full agent access), but shell access removes a containment layer — both against a leaked
     key and against agent mistakes. The key lives in the app's Keychain and this BFF's `0600`
     config; the user must be comfortable with that trust chain.

   If the user says **yes**: hermes resolves tools per platform via `platform_toolsets` in
   `~/.hermes/config.yaml` — add an `api_server` entry there (e.g. the same toolset bundle the
   `cli` platform uses; back the file up first), restart the gateway **only with the user's
   explicit OK** (it interrupts running sessions), then verify with `/v1/toolsets` that
   `terminal` reports enabled **for platform `api_server`** — asking the agent on another
   platform (Telegram etc.) answers for that platform, not this one. Sessions created before
   the change may keep the old toolset; test in a fresh session. If **no**: everything still
   works; recommend a local transcription skill (e.g. whisper) so voice messages transcribe
   reliably.

## 5. REQUIRED: run as a keep-alive service and verify

A foreground `agency-bff run` is for debugging only. The install is incomplete until the service
manager starts it at login/boot and restarts it after a crash. Do not hand the pairing details to
the user before this section passes.

Service templates ship in `deploy/` of the public release repository:

- **macOS:** install `deploy/space.rath.agency-bff.plist` in `~/Library/LaunchAgents/`. Verify the
  plist keeps both `RunAtLoad` and `KeepAlive` enabled, then run
  `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/space.rath.agency-bff.plist`.
- **Linux:** install `deploy/agency-bff.service` in `~/.config/systemd/user/`. Verify its service
  policy includes `Restart=` and an appropriate restart delay, then run
  `systemctl --user enable --now agency-bff` (or a system unit if the user prefers).
- **Windows:** `sc create agency-bff binPath= "<path>\agency-bff.exe run"` or a Task Scheduler
  at-logon task.

**Verify:**

```sh
curl -s http://127.0.0.1:8643/health
# -> {"status":"ok","version":"…"}

curl -s -H "Authorization: Bearer <PAIRING_TOKEN>" http://127.0.0.1:8643/v1/bff/capabilities
# -> {"push":…,"upload":true,"media":true,…}

agency-bff status        # config sanity + gateway reachability, secrets redacted
```

Also verify from off-box (the phone's network path) that `/health` responds at the URL you will
give the user. Kill the process once and prove the service manager restarts it. Confirm reboot
survival when the user permits a reboot test; otherwise report that `RunAtLoad`/enablement was
verified but a production reboot was not performed.

**Logs and retention:** the BFF writes structured JSON to stdout and daily UTC files at
`<data_dir>/logs/bff-<date>.log`. Device batches land under
`<data_dir>/logs/devices/<device-id>/<date>.jsonl`. Existing launchd installs may still redirect
stderr to `~/.agency-bff/log/agency-bff.log`; that legacy path only catches raw stderr and panic
output. When diagnosing a failure, inspect the service manager status and these logs before
changing configuration; never print secrets from the config.

## 6. Hand off to the user

Give the user exactly this, filled in:

> The companion service is installed and healthy.
>
> In the **Agentic Agency ChatApp** live onboarding companion step (or Settings -> Connection
> when editing an existing pairing), enter:
>
> - **URL:** `<the reachable base URL, e.g. https://klaushaus.tail1234.ts.net:8643>`
> - **Pairing token:** `<the token printed by init>`
>
> Tap **Connect and verify**. The app requires version 0.11.0 or newer and will confirm the
> required baseline:
> file & voice-memo sending, rich media replies<if push configured>, and notifications when your
> assistant finishes while the app is closed</if>.
>
> The token is like a house key — the app stores it securely; don't paste it anywhere else. You
> can rotate it any time by asking me to run `agency-bff token rotate` (then re-pair the app).

Optionally, when media replies are wanted, persist an agent instruction that messages ending in
`[client: agentic-agency-app]` may deliver generated media by saving the file under
`~/.agency-bff/data/outbox/` and returning its relative path as `media://<path>`. Scope the rule
to that client tag so other channels are unaffected. Record that you installed this instruction;
[UNINSTALL.md](UNINSTALL.md) removes it again.

## 7. Backups and production respect

- Before editing any pre-existing file (shell profiles, proxy configs, launchd/systemd files not
  created by you): copy it to `<file>.bak-<date>` first and tell the user.
- Never modify the Hermes gateway's own config, database, or service definition.
- The BFF's own state is disposable-by-design: config can be regenerated (`init` + re-pair),
  inbox/outbox are caches. Still, include `~/.agency-bff/config.json` in whatever backup the user
  already runs, and say so in the handoff.

## 8. Uninstall

Follow the dedicated runbook: [UNINSTALL.md](UNINSTALL.md) — it covers explaining the impact to
the user, inventorying what exists, per-OS removal, and verification. Nothing else on the
machine is touched by an install, so removal is complete.
