# Subscription account usage (0.26.1)

`GET /v1/usage` requires the BFF pairing bearer and returns HTTP 200 with a bounded quota
snapshot, including when unavailable. Every response, including auth errors, has
`Cache-Control: no-store`. Capability `subscription_usage` is true only when the operator has
configured `realtime_subscription.hermes_python`; it indicates configuration, not credentials
or entitlement. The empty default disables usage and optional subscription voice.

## Account identity and scope

Protocol 2 reports every configured official Codex OAuth account separately, including exhausted
accounts in cooldown. This matters for Hermes credential pools: round-robin selection changes
which account serves a turn, so one default-resolver sample cannot represent the pool. Accounts
remain independent: never add their percentages or treat their limits as a shared pooled balance.
A client can form a summary only from a complete inventory and available account snapshots; one
unavailable or stale account makes the pool summary unknown. Apply window comparisons within each
account before applying a product-specific account-selection rule, and retain the per-account
details. These are shared account-wide subscription allowances, not app-local token counts or a
claim about which account served a particular response.

The helper reads Hermes' credential-pool records without calling `select()` or changing priorities
or request counts. A pool with no records falls back to the Hermes singleton token store.
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
inherited HTTP proxies. Resolver base URLs never redirect the request; Hermes generates the
canonical account headers and the helper checks their identity before sending. Expiring tokens
or a 401 trigger only Hermes-owned targeted refresh of that exact credential. The helper supplies
both row ID and token hint, checks the current account before refresh, and checks the returned
account afterward. It never calls a pool refresh without an exact target. A changed identity is
unavailable. Hermes owns cross-process locking and atomic token-store writes; quota reading never
rotates selection. Operator account additions/removals can take up to 60 seconds to appear in the
fresh cache. No quota data is persisted.

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
includes `remaining_percent` and `reset_at`. Percentage is exactly `100 - used_percent`, with
no clamping or rounding. Only finite provider values within 0...100 are accepted. Reset times
must be in the future and within the duration plus five minutes of clock tolerance.

Classification uses exact duration, never primary/secondary position. `not_applicable` requires
a successful response with both primary and secondary keys, exactly one explicit null, and one
valid known-duration window. Missing keys, two nulls, malformed values, expired resets, unknown
or duplicate durations remain unavailable. Both-null upstream responses cannot establish
unsupported-account status and conservatively remain unavailable. No fictional zero or 100
percent is emitted.

An account is `available` only with at least one available window and all other windows available
or explicitly not applicable. `stale` retains that same account's previous successful sample
after a transport/rate-limit failure for at most 15 minutes. Auth failure, changed identity,
malformed windows, removed accounts, or unreadable inventory discard the affected sample.
Expired windows lose their numbers. A successful partial response is not replaced with an older
healthy response. Top-level `available` requires all accounts available and complete inventory;
`stale` means at least one stale account and no unavailable accounts; any unavailable account or
incomplete inventory makes the top-level state unavailable. Stale values must never establish a
healthy overview.

Reasons are bounded literals: `ready`, `not_configured`, `credentials_unavailable`,
`account_limit_exceeded`, `account_changed`, `auth_rejected`, `rate_limited`,
`upstream_unavailable`, `timeout`, `busy`, `canceled`, `invalid_response`, `invalid_windows`,
`windows_missing`, `windows_expired`, `cache_expired`, `accounts_stale`, `accounts_unavailable`.

Successful whole snapshots cache for 60 seconds, failures 30 seconds. Concurrent requests coalesce
one helper fetch; waiting requests can cancel. A canceled fetch does not poison other callers'
cache. Per-account stale retention is keyed only by its stable ID and never transfers to another
account. The helper has an 18-second deadline with at most four parallel account reads; provider
requests have at most six seconds each. The BFF enforces a 20-second process lifetime and 64 KiB
sanitized output. Targeted Hermes refresh can consume the budget and degrade to unavailable
rather than exceed that deadline. Auth remains the existing constant-time pairing check and
failed-auth limiter.

## Upstream compatibility and operators

Model-specific `additional_rate_limits`, spend controls, banked resets and credits are outside
this account-window contract. Available account quota does not establish model capacity or
entitlement. The endpoint is an internal provider API used by official Codex clients, not a
published stable OpenAI API. Schema drift becomes unavailable. Primary references:
[Codex backend client](https://github.com/openai/codex/tree/main/codex-rs/backend-client/src/client),
[Codex usage tests](https://github.com/openai/codex/blob/main/codex-rs/app-server/tests/suite/v2/rate_limits.rs),
[Hermes account usage](https://github.com/NousResearch/hermes-agent/blob/main/agent/account_usage.py),
and [Hermes credential pool](https://github.com/NousResearch/hermes-agent/blob/main/agent/credential_pool.py).

After a backed-up BFF-only update, confirm version `0.26.1`, authenticated capabilities, and one
`GET /v1/usage`. Keep the bearer in a private curl config or a process reading the existing private
pairing file; do not put it in shell history or print raw auth state. Inspect only anonymous IDs,
statuses, durations, remaining percentages and reset/fetch timestamps. Compare anonymous account
IDs before/after a normal gateway turn to prove rotation does not change quota identity. Do not
restart or alter the gateway for this feature. Clearing the interpreter disables usage and
optional subscription voice together. Restore the backed-up config or binary to roll back;
protocol-2 clients require BFF 0.26.1 and do not accept the earlier single-account protocol.
