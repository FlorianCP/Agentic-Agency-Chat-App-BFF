# Subscription account usage (0.27.0)

`GET /v1/usage` requires the BFF pairing bearer and returns HTTP 200 with the protocol-2 Codex
quota snapshot, including when unavailable. `GET /v1/usage?provider=anthropic` returns the same
protocol-2 shape for Anthropic. An absent provider preserves the Codex contract; unknown or
repeated provider values return HTTP 400. Provider snapshots and caches remain independent. Every
response, including auth errors, has
`Cache-Control: no-store`. Capability `subscription_usage` is true only when the operator has
configured `realtime_subscription.hermes_python`; it indicates configuration, not credentials
or entitlement. The empty default disables usage and optional subscription voice.

## Account identity and scope

For Codex, protocol 2 reports every configured official OAuth account separately, including exhausted
accounts in cooldown. This matters for Hermes credential pools: round-robin selection changes
which account serves a turn, so one default-resolver sample cannot represent the pool. Accounts
remain independent: never add their percentages or treat their limits as a shared pooled balance.
The iOS overview computes each available account's bottleneck as the minimum remaining
percentage across its applicable windows, then shows the maximum of those account bottlenecks.
An unavailable or stale account does not hide a healthy available account; the overview marks
partial availability and retains the per-account details. If no account is available, the
overview is unavailable. Percentages are never added across accounts. These are shared account-wide
subscription allowances, not app-local token counts or a claim about which account served a particular response.

The helper reads Hermes' credential-pool records without calling `select()` or changing priorities
or request counts. It resolves the active `HERMES_HOME` or default `~/.hermes` path directly, then
applies the profile-to-global-root fallback used by Hermes. Bounded, duplicate-key-rejecting reads
inspect the relevant `auth.json`, `.env`, `config.yaml`, Hermes OAuth file, and Claude Code
credential file. The helper does not import Hermes auth, pool, credential, or config modules:
importing them can initialize plugin/config state, and `read_credential_pool()` can create an
`auth.json.corrupt` backup while handling malformed JSON. A pool with no records falls back to
the Hermes singleton token store.
Only OAuth credentials with valid ChatGPT account claims and an official Codex route are queried.
Custom-route and API-key records are excluded, with no API-key fallback. A malformed relevant
OAuth record makes the inventory unavailable rather than silently dropping an account.

Each `id` is SHA-256 of a domain prefix and the OAuth ChatGPT account ID, not the access token.
IDs survive token refresh and pool rotation; duplicate credentials for one account produce one
snapshot. Sorting these IDs gives stable anonymous `Account 1`, `Account 2` labels while the
inventory is unchanged. The iOS client localizes these labels. Raw IDs, email addresses, credential
labels, tokens and plan/account metadata are never returned or logged. At most four distinct
accounts are supported. More returns `account_limit_exceeded`, never a silently truncated pool.

The request goes only to `https://chatgpt.com/backend-api/wham/usage`, without redirects or
inherited HTTP proxies. Resolver base URLs never redirect the request; the helper builds the
quota reader's `codex-cli` User-Agent and JWT-derived account/residency headers, then checks the
account identity before sending. Quota access is
strictly read-only: every helper invocation reads the current Hermes access tokens into memory,
then makes at most one usage request per non-dead account. It never refreshes access tokens,
uses refresh tokens, invokes a credential resolver or pool loader, writes authentication state,
or changes account selection. A 401 or 403 is `auth_rejected`; dead accounts remain visible as
`credentials_unavailable` without a provider request. Expired access tokens are sent as currently
stored and may be rejected by the provider; they never trigger refresh. A later poll automatically
picks up tokens independently rotated by Hermes. The account-header identity must match the
snapshotted token identity before any request. Operator account additions/removals or independently
rotated tokens can take up to 60 seconds to appear in the fresh cache. No quota data or credentials
are persisted by this helper. Hermes retains sole responsibility for credential lifecycle.

For Anthropic, the helper reads OAuth pool entries directly. When the pool has no relevant OAuth
records, it reads the first OAuth token configured through `ANTHROPIC_TOKEN` or
`CLAUDE_CODE_OAUTH_TOKEN`, the Hermes PKCE singleton file, and borrowed Claude Code credentials
from its credential file or macOS Keychain. Claude Code credentials are read only when
`auth.adopt_external_logins` is enabled; its Hermes default is true. Environment API keys and
literal API-key entries are excluded. A pool `api_key` record with the recognized
`sk-ant-oat` OAuth token prefix is treated as OAuth, matching Hermes' credential normalization.
This is a subscription-credential inventory independent of the active chat provider or resolver
precedence: it may show configured OAuth subscription credentials while chat uses an API key, but
it never queries usage with an API key.
The helper does not load or select the pool, resolve a token, inspect refresh state, or refresh
credentials. A malformed relevant OAuth entry makes that provider inventory unavailable. Custom
Anthropic routes are excluded, and a process-level or Hermes `.env` `ANTHROPIC_BASE_URL` override
to a nonofficial route excludes Anthropic usage because the BFF always calls the fixed official
endpoint.

Anthropic's account-usage request goes only to
`https://api.anthropic.com/api/oauth/usage`, without redirects or inherited HTTP proxies. It uses
the stored OAuth access token with the Claude Code `anthropic-beta` and `User-Agent` headers. It
makes one request per non-dead OAuth token and never retries or rotates an expired token. A 401 or
403 is `auth_rejected`; the next poll reads the then-current credential store. Exact duplicate
current tokens are deduplicated in memory for that poll.

Hermes does not expose an Anthropic account UUID through these credential readers. The anonymous
Anthropic `id` is a provider-prefixed hash of a Hermes credential-entry ID, or a fixed singleton
source name for environment, Hermes PKCE, or Claude Code credentials. It identifies a credential
slot, not a verified distinct Anthropic account. Different slots for the same account may therefore appear separately, and a
slot may refer to a different account after reauthentication. The BFF does not retain stale
Anthropic usage across a failed fetch, so old limits cannot be attributed to a repointed slot.
The raw Hermes credential-entry IDs, access tokens, refresh tokens, email addresses, and plan
metadata are never returned or logged; only anonymous slot hashes are returned. At most four
distinct OAuth slots are supported; more returns `account_limit_exceeded`.

## Response

```json
{
  "protocol_version": 2,
  "provider": "openai-codex",
  "scope": "account_pool",
  "strategy": "round_robin",
  "status": "available",
  "reason": "ready",
  "checked_at": "2026-10-04T13:00:00Z",
  "accounts": [
    {
      "id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
      "display_name": "Account 1",
      "status": "available",
      "reason": "ready",
      "checked_at": "2026-10-04T13:00:00Z",
      "fetched_at": "2026-10-04T13:00:00Z",
      "five_hour": {"status": "not_applicable", "window_seconds": 18000},
      "weekly": {
        "status": "available",
        "window_seconds": 604800,
        "remaining_percent": 42.5,
        "reset_at": "2026-10-08T13:00:00Z"
      }
    }
  ]
}
```

`strategy` is `round_robin`, `fill_first`, `least_used`, `random`, or `unknown`. It reports the
configured strategy, not an active-account attribution or a prediction of the next request.
Top-level `checked_at` is the completion time of the last actual BFF fetch attempt. Each account's
`checked_at` is that attempt's completion time; its optional `fetched_at` is the completion time
of the last successfully parsed provider response for the same identity. Cache reads preserve
these timestamps. All are UTC RFC3339, and none is a reset time.

Both window objects are always present per account. Each has `status: available |
not_applicable | unavailable` and exact `window_seconds` 18000 or 604800. Only `available`
includes `remaining_percent` and `reset_at`. For Codex, percentage is exactly `100 - used_percent`;
for Anthropic, it is exactly `100 - utilization`, where Anthropic's `utilization` is already a
percentage on the 0...100 scale. Values are never interpreted as fractions, clamped, or rounded.
Only finite provider values within 0...100 are accepted. Reset times must be in the future and
within the duration plus five minutes of clock tolerance.

Codex classification uses exact duration, never primary/secondary position. Anthropic maps
`five_hour` to the five-hour window and `seven_day` to the weekly window. For either provider,
`not_applicable` requires a successful response with both window fields, exactly one explicit null,
and one valid window. Missing keys, two nulls, malformed values, expired resets, and unknown or
duplicate Codex durations remain unavailable. Both-null responses cannot establish unsupported
account status and conservatively remain unavailable. No fictional zero or 100 percent is emitted.

An account is `available` only with at least one available window and all other windows available
or explicitly not applicable. Codex `stale` retains that same account's previous successful
sample after a transport/rate-limit failure for at most 15 minutes. Anthropic never retains a stale
sample because its credential-slot ID cannot prove the underlying account is unchanged. Auth
failure, changed identity, malformed windows, removed accounts, or unreadable inventory discard
the affected sample. Expired windows lose their numbers. A successful partial response is not
replaced with an older healthy response. Top-level `available` requires all accounts available
and complete inventory; `stale` means at least one stale Codex account and no unavailable accounts;
any unavailable account or incomplete inventory makes the top-level state unavailable. Stale
values must never establish a healthy overview.

Reasons are bounded literals: `ready`, `not_configured`, `credentials_unavailable`,
`account_limit_exceeded`, `account_changed`, `auth_rejected`, `rate_limited`,
`upstream_unavailable`, `timeout`, `busy`, `canceled`, `invalid_response`, `invalid_windows`,
`windows_missing`, `windows_expired`, `cache_expired`, `accounts_stale`, `accounts_unavailable`.

Successful whole snapshots cache for 60 seconds, failures 30 seconds. Concurrent requests to the
same provider coalesce one helper fetch; waiting requests can cancel. A canceled fetch does not
poison other callers' cache. Codex stale retention is keyed only by its stable account ID and
never transfers to another account. The helper has an 18-second deadline with at most four parallel account reads; provider
requests have at most six seconds each. The BFF enforces a 20-second process lifetime and 64 KiB
sanitized output. No authentication refresh work consumes this deadline. Auth remains the existing
constant-time pairing check and failed-auth limiter.

## Upstream compatibility and operators

Model-specific Codex `additional_rate_limits`, Anthropic `seven_day_opus` and `seven_day_sonnet`,
spend controls, banked resets and credits are outside this account-window contract. Available
account quota does not establish model capacity or
entitlement. The Codex endpoint is an internal provider API used by official clients, not a
published stable OpenAI API. Anthropic's OAuth usage endpoint is also provider-owned and subject
to schema drift; unsupported responses become unavailable. Primary references:
[Codex backend client](https://github.com/openai/codex/tree/main/codex-rs/backend-client/src/client),
[Codex usage tests](https://github.com/openai/codex/blob/main/codex-rs/app-server/tests/suite/v2/rate_limits.rs),
[Hermes account usage](https://github.com/NousResearch/hermes-agent/blob/main/agent/account_usage.py),
and [Hermes credential pool](https://github.com/NousResearch/hermes-agent/blob/main/agent/credential_pool.py).

After a backed-up BFF-only update, confirm version `0.27.0`, authenticated capabilities, and one
`GET /v1/usage` plus `GET /v1/usage?provider=anthropic`. Keep the bearer in a private curl config or a process reading the existing private
pairing file; do not put it in shell history or print raw auth state. Inspect only anonymous IDs,
statuses, durations, remaining percentages and reset/fetch timestamps. Codex account IDs remain
stable across token rotation; Anthropic IDs identify credential slots, not verified account
identity. Do not restart or alter the gateway for this feature. Clearing the interpreter disables usage and
optional subscription voice together. Restore the backed-up config or binary to roll back;
protocol-2 clients require BFF 0.26.1 and do not accept the earlier single-account protocol.
