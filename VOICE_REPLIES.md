# Durable voice-reply jobs - BFF 0.24.0

All routes require the existing pairing bearer token. The BFF stores identifiers/digests and
native iOS-rendered AAC; it does not synthesize speech, hold transcripts, invoke Piper, change
call signaling, or provide a paid API fallback. The app registers immutable GPT-Live intent
before sending Hermes. Only the registered media link in final assistant history identifies
the job; the app never guesses its association by text/time/latest message.

## Routes

| Method/path | Request | Successful response |
|---|---|---|
| PUT `/v1/voice-replies/{uuid}` | `{session_id,model,voice}` | Job; identical registration is idempotent |
| GET `/v1/voice-replies/{uuid}` | None | Job without lease token |
| GET `/v1/voice-replies?session_id=...` | Session filter | `{data:[Job]}` unresolved pending/rendering/failed only |
| POST `/v1/voice-replies/{uuid}/claim` | `{message_id,answer_sha256}` | `{job,lease_token,lease_generation,lease_expires_at}` |
| POST `/v1/voice-replies/{uuid}/release` | `{lease_token,lease_generation}` | Failed Job; binding retained |
| POST `/v1/voice-replies/{uuid}/cancel` | None | Cancelled Job; lease invalidated, binding retained |
| PUT `/v1/voice-replies/{uuid}/audio` | Raw AAC/ISO BMFF body | Ready Job |

UUIDs are canonical lowercase hyphenated values. Model is exactly `gpt-live-1-codex`; voices
are Cove, Juniper, Maple, Breeze, Vale, Sol, Ember, Spruce, Arbor using lowercase wire names.
The server derives `media_path` as `voice-replies/<uuid>.m4a`. Registration has no autoplay
field; autoplay is device-local. Session/message IDs are nonempty, trimmed, at most 256 UTF-8
bytes, and reject control/format characters. SHA-256 values are 64 lowercase hexadecimal bytes.

Job fields: `id`, `session_id`, `model`, `voice`, `media_path`, `status`, optional `message_id`,
`answer_sha256`, `audio_sha256`, `lease_generation`, optional `lease_expires_at`, `created_at`,
`updated_at`. Dates are UTC RFC3339 without fractions. Status is pending, rendering, failed,
ready, unavailable, or cancelled. GET never returns the lease token.

Publication headers: `Content-Type: audio/mp4`, `X-Lease-Token`, `X-Lease-Generation`, and
`X-Audio-SHA256`. Validate the current random token and generation, body digest, and bounded
ISO BMFF box framing with mandatory ftyp/moov/mdat. This is container validation, not AAC
sample decoding, playable-audio proof, or voice-identity verification.

## Conflicts, recovery, and bounds

Errors use `{error:code}`. Invalid input returns 400 `invalid_request`; unknown jobs return
404 `job_not_found`; audio over limit returns 413 `audio_too_large`. Conflict codes (409) are
`intent_conflict`, `binding_conflict`, `lease_busy`, `lease_stale`, `publish_conflict`,
`audio_unavailable`, `job_limit`, and `storage_pressure`. Filesystem failures return 500
`voice_reply_storage_failed` without exposing path or credentials.

Leases last ten minutes; claim binds message ID/answer digest once. Expired claims can be
retaken with a new generation. Stale owners cannot publish/release. A published identical-body
retry succeeds only with the original publishing token/generation; another digest cannot
overwrite the winner. Cancel ends unresolved work and frees pending capacity. Ready jobs
cannot be cancelled; use existing media deletion. Missing/changed ready files become unavailable
and cannot silently regenerate. List returns every unresolved job for the session, ordered by
creation time then ID, and excludes ready/unavailable/cancelled history.

Limits: JSON bodies 16 KiB, individual metadata records 32 KiB, audio the smaller of 8 MiB and
the configured upload cap, 32 unresolved jobs/session, 256 unresolved jobs globally, and 10,000
retained job records. Failed jobs remain unresolved until completed/cancelled. Metadata is
pruned after 180 days. Metadata enumeration streams bounded pages and refuses directories over
11,024 entries, including foreign names/recent crash artifacts, rather than allocating or
scanning without limit. Valid regular `.job-<64-hex>` crash files older than ten minutes are
cleaned during enumeration; symlinks/foreign files are preserved. Publication also removes
regular `.upload-<64-hex>` crash artifacts older than ten minutes from the managed outbox,
using the same bounded directory-scan limit. Such a heavily polluted
metadata directory requires operator cleanup before registration/listing can continue.

Publication checks combined inbox/outbox budget and configured minimum disk free space before
writing. Host audio remains under the existing combined 20 GB/default 180-day retention policy.
Metadata/file publication uses restrictive permissions, containment-safe paths, synced files,
atomic rename/create-only hard link, and directory fsync on macOS/Linux. Windows has process-
restart recovery without directory-fsync power-loss guarantees. On restart, a matching published
file reconciles the publication/metadata seam. The phone resumes unfinished rendering while
foregrounded; host job persistence does not provide background synthesis. Existing cached
files remain playable offline, while new rendering/publication requires connectivity.
