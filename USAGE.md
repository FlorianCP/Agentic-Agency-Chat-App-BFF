# Subscription account usage (0.26.0)

`GET /v1/usage` requires the BFF pairing bearer and returns HTTP 200 with a bounded quota
snapshot, including when unavailable. Every response, including auth errors, has
`Cache-Control: no-store`. Capability `subscription_usage` is true only when the operator has
configured `realtime_subscription.hermes_python`; it indicates configuration, not credentials
or entitlement. The empty default disables both the usage reader and optional subscription voice.

The embedded helper invokes the Hermes-owned Codex OAuth runtime resolver in that configured
Python environment. Hermes owns credential selection, token refresh and its atomic auth-store
writes. Only `provider: openai-codex` and `auth_mode: chatgpt` are accepted. There is no API-key
fallback. The request goes only to `https://chatgpt.com/backend-api/wham/usage`, without redirects
or inherited HTTP proxies; resolver base URLs are ignored. Hermes generates account identity
headers. OAuth tokens and account identifiers never leave the host in API responses or logs.

This is the active account selected by the Hermes runtime resolver, including its supported
credential-pool selection. It does not promise a stable identity when operators rotate pool
accounts, nor that the quota account served a previous message. A fresh successful response
replaces the previous sample; authentication or credential failures discard retained samples.
The in-memory cache may reflect the previous selected account for at most its 60-second fresh
interval after an external account change. No quota data is persisted.

## Response

```json
{
  "protocol_version": 1,
  "provider": "openai-codex",
  "scope": "account",
  "status": "available",
  "reason": "ready",
  "checked_at": "2026-10-04T13:00:00Z",
  "fetched_at": "2026-10-04T13:00:00Z",
  "five_hour": {
    "status": "not_applicable",
    "window_seconds": 18000
  },
  "weekly": {
    "status": "available",
    "window_seconds": 604800,
    "remaining_percent": 42.5,
    "reset_at": "2026-10-08T13:00:00Z"
  }
}
```

`checked_at` is the completion time of the last actual BFF fetch attempt, including failures;
a cached read preserves it. `fetched_at` is the completion time of the last successfully parsed
provider response, or absent when none exists. Both are UTC RFC3339. Neither is a reset time.

Both window objects are always present. Each has `status: available | not_applicable |
unavailable` and its exact `window_seconds`. Only `available` includes `remaining_percent`
and `reset_at`. Remaining percentage is exactly `100 - used_percent`, with no clamping or
rounding, and only finite provider values within 0...100 are accepted. Reset times must be in
the future and within the window duration plus five minutes of clock tolerance.

Window classification uses exact duration, never primary/secondary position. `not_applicable`
requires a successful response with both primary and secondary keys, exactly one explicit null,
and one valid known-duration window. Missing keys, two nulls, malformed values, expired resets,
unknown durations and duplicate durations stay unavailable. The upstream endpoint does not
unambiguously identify unsupported accounts when both windows are null; these conservatively
remain unavailable. No fictional zero or 100 percent is emitted.

Top-level `available` requires at least one available window and every other window available
or explicitly not applicable. `stale` means a failed refresh retained a previously available
sample for at most 15 minutes. Stale data must never drive a healthy quick indicator; expired
windows lose their numbers. Successful partial/malformed window responses are unavailable and
are not replaced with older healthy data. `unavailable` means the snapshot cannot establish
current account quota. Reasons are bounded literals: `not_configured`, `credentials_unavailable`,
`auth_rejected`, `rate_limited`, `upstream_unavailable`, `timeout`, `busy`, `canceled`,
`invalid_response`, `invalid_windows`, `windows_missing`, `windows_expired`, `cache_expired`.

Successful snapshots are cached 60 seconds, failures 30 seconds. Concurrent requests coalesce
one fetch; waiting requests can cancel. A canceled fetch does not poison other callers' cache.
The process helper is bounded to 20 seconds, the provider request to 10 seconds, and provider JSON
to 64 KiB. Auth remains the existing constant-time pairing check and failed-auth limiter.

## Scope and upstream compatibility

These are account-wide subscription allowances, shared by every tool consuming the same account,
not app-local token usage or per-message counts. Model-specific `additional_rate_limits`, spend
controls, banked resets and credits are intentionally outside this account-window contract.
Account quota availability does not establish capacity or entitlement of any particular model.

The endpoint is an internal provider API used by official Codex clients, not a published stable
OpenAI API contract. Primary source references:
[Codex backend client](https://github.com/openai/codex/tree/main/codex-rs/backend-client/src/client),
[Codex usage contract tests](https://github.com/openai/codex/blob/main/codex-rs/app-server/tests/suite/v2/rate_limits.rs),
and [Hermes account usage](https://github.com/NousResearch/hermes-agent/blob/main/agent/account_usage.py).
Schema drift degrades to unavailable rather than invented quota.

## Operator verification

After a backed-up BFF-only update, confirm binary version `0.26.0`, authenticated capabilities,
and a single authenticated `GET /v1/usage`. Keep the bearer in a private curl config or a process
that reads the existing private pairing file; do not put it in shell history or print raw auth
state. Inspect only status, durations, remaining percent, and reset/fetch timestamps. If
unavailable, verify the configured absolute interpreter and Hermes OAuth login without dumping
credentials. Do not restart or alter the gateway for this feature. Clearing the interpreter
setting disables usage and optional subscription voice together; restore the backed-up config
or binary to roll back.
