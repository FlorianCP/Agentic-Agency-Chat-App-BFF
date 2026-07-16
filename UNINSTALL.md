# Agency BFF — Uninstall Runbook (for agents)

**Audience:** you are an AI agent (typically the Hermes agent running on this host) that a user
has asked to **remove the Agentic Agency BFF companion service**. Follow this runbook top to
bottom: explain what removal means, inventory what exists, confirm, remove, verify.

**Ground rules:** this is likely a production machine running the user's live Hermes gateway.
Only remove things this runbook names — never touch the Hermes gateway itself, its config, or
its data. Get the user's explicit OK before deleting anything (step 3).

## 1. Tell the user what uninstalling means

- The app on their phone **keeps working** exactly as before the BFF existed — chat, images,
  and on-device voice transcription don't involve the BFF.
- What they **lose**: sending files/documents to the assistant, voice messages transcribed by
  the assistant, media replies, and (if configured) push notifications.
- What gets **deleted from this machine**: the `agency-bff` binary, its background service, and
  its data directory — including the config (with the gateway-key copy and pairing-token hash)
  and any not-yet-cleaned uploads in the inbox/outbox.
- The Hermes gateway and its API key keep working unchanged; the key was shared with the BFF,
  not owned by it. If the user wants the key rotated anyway, that is a separate gateway task.
- Removal is complete: the install touched nothing else on the machine.

## 2. Inventory (read-only)

```sh
command -v agency-bff; ls -la ~/.local/bin/agency-bff 2>/dev/null
ls ~/.agency-bff 2>/dev/null
du -sh ~/.agency-bff/data 2>/dev/null          # anything still in inbox/outbox?
# macOS service:
launchctl print gui/$(id -u)/space.rath.agency-bff 2>/dev/null | head -3
ls ~/Library/LaunchAgents/space.rath.agency-bff.plist 2>/dev/null
# Linux service:
systemctl --user status agency-bff 2>/dev/null | head -3
# Windows service:
sc query agency-bff 2>/dev/null
```

Report to the user what you found (service running or not, data size). If the data directory
holds recent uploads, mention they will be deleted.

## 3. Confirm with the user

Ask explicitly: "Remove the service, the binary, and all of its data (config, pairing, stored
uploads)?" Only proceed on a clear yes.

## 4. Remove

**macOS:**

```sh
launchctl bootout gui/$(id -u)/space.rath.agency-bff 2>/dev/null
rm -f ~/Library/LaunchAgents/space.rath.agency-bff.plist
rm -f ~/.local/bin/agency-bff
rm -rf ~/.agency-bff
```

**Linux:**

```sh
systemctl --user disable --now agency-bff 2>/dev/null
rm -f ~/.config/systemd/user/agency-bff.service
systemctl --user daemon-reload
rm -f ~/.local/bin/agency-bff
rm -rf ~/.agency-bff
```

**Windows:**

```powershell
sc stop agency-bff; sc delete agency-bff    # or remove the Task Scheduler entry
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\agency-bff"
Remove-Item -Recurse -Force "$env:USERPROFILE\.agency-bff"
```

If the install used non-default locations (a different `AGENCY_BFF_DIR`, `/usr/local/bin`, a
reverse-proxy entry the user added for port 8643), remove those too — but back up any file you
did not create before editing it (e.g. a shared proxy config).

## 4a. Remove the learned media capability

If the agent's skills/system prompt/memory were taught the BFF media convention during setup
(saving replies into `~/.agency-bff/data/outbox/`, returning `media://` links, and scoping it to
the `[client: agentic-agency-app]` tag), **remove that instruction too** - after
this uninstall the outbox no longer exists, and a leftover rule would make the agent try to
write media into a deleted directory whenever it sees the tag. If you are the agent performing
this uninstall, that means editing your own persistent instructions; confirm the removal to the
user alongside the file removals.

## 5. Verify and hand off

```sh
curl -s -m 3 http://127.0.0.1:8643/health || echo "port 8643 no longer serving — good"
command -v agency-bff || echo "binary gone — good"
ls ~/.agency-bff 2>/dev/null || echo "data gone — good"
```

Then tell the user:

> The companion service has been removed from this machine. In the chat app, open
> **Settings → Companion service** and remove the pairing there too (it only forgets the URL
> and token on the phone). Everything else in the app keeps working. You can reinstall anytime
> - the setup guide is `SETUP.md` in the public release repository at
> https://github.com/FlorianCP/Agentic-Agency-Chat-App-BFF.
