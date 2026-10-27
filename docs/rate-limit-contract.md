# Client contract: the two 429s from `api.theintrodb.org`

An integration eventually gets a `429`. There are two different ones — a **burst** ceiling and a
**daily budget** — and the API shapes both identically on purpose: same JSON keys, same
`Retry-After`, so a client only has to parse one `429`. The `code` field is the only thing that
separates "slow down for a few seconds" from "you are out of budget until tomorrow".

Clients must handle them the same way. This is that contract.

## What arrives

Burst — the per-window ceiling. These are the bytes the live endpoint returned:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 10

{"code":"rate_limit_exceeded","error":"Too many requests","retry_after":"10 seconds","retry_after_seconds":10}
```

Daily — the budget for the UTC day is spent:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 50321
X-UsageLimit-Limit: 5000
X-UsageLimit-Remaining: 0
X-UsageLimit-Reset: 50321

{"error":"Usage limit exceeded","retry_after":"14.0 hours","retry_after_seconds":50321,"code":"usage_limit_exceeded"}
```

| | burst | daily budget | one media target |
| --- | --- | --- | --- |
| `code` | `rate_limit_exceeded` | `usage_limit_exceeded` | `specific_media_usage_limit_exceeded` |
| what ran out | requests per 10 s | requests per UTC day | requests per UTC day, for one title |
| `Retry-After` | seconds left in the 10 s window (≤ 10) | seconds to UTC midnight (≤ 86400) | ≤ 86400 |
| counters on the `429` | **none** — `Retry-After` only | `X-UsageLimit-Limit/-Remaining/-Reset` | `X-UsageLimit-SpecificMedia-Limit/-Remaining/-Reset` |

`Retry-After` is always present, always in RFC 9110 delta-seconds, and always the same instant as
the matching `X-…-Reset`. **It is the one piece of the `429` a client may depend on**: the burst
`429` carries no `X-RateLimit-*` counters at all, while the daily one does carry its own. Read the
counters off the last `200` (they are all there) and take the wait from `Retry-After`.

## The rules

**1. The wait is `Retry-After`, whatever the code.** Fall back in this order, and stop at the first
usable value: `Retry-After` → `retry_after_seconds` → `retry_after` (`"14.0 hours"`) →
`X-…-Reset` → 60 s. Nothing else should be consulted to compute a wait.

**2. Clamp only as a ceiling — never down to the burst window.** `0 ≤ wait ≤ 86400`.

Clamping a daily `429` to a burst-shaped wait (5 minutes, 30 seconds) is the one that bites: it
turns "come back tomorrow" into "come back in five minutes", so the client wakes up, spends one
request, is refused again, and works through the next day's budget one request at a time — or,
worse, loops until it is stopped by hand. An implausible-looking wait is real. 14 hours means 14
hours.

**3. Branch on `code` for policy, not for duration.** Both 429s carry the same numbers; what a
client *does about it* is where they differ.

- `rate_limit_exceeded` — sleep the wait, then retry. The run continues. Growing the wait across
  consecutive bursts (×1, ×2, ×4, … capped) is good practice and stays inside rule 2.
- `usage_limit_exceeded` / `specific_media_usage_limit_exceeded` — **stop and report**. Do not sit
  in a request loop until a 24-hour window rolls over; resume on the next scheduled run, or on the
  next thing the user does. A tool with a queue should persist what is left and say when it will
  continue.

**4. A `429` is never an auth verdict.** Only `401` and `403` mean the key is bad. Never report
"invalid key" for a `429`, and never clear a stored key because of one.

**5. Don't earn a `429` to discover a limit you can read from a `200`.** Every successful `/media`
response carries the applicable `X-UsageLimit-*` and `X-RateLimit-*`; pace and budget against
those. (The specific-media trio is on the same responses, so per-title spend is observable before
it is exhausted.)

**6. Say what is happening.** Whatever the surface — a player overlay, an add-on dialog, a log
line, a TUI status field — the user should be told the wait in human terms (`rate limited, retry in
7s`; `daily budget spent, resumes 14:00 UTC`), not handed a bare error. Silence plus an immediate
retry is how a client gets a user's account rate-limited.

## The numbers, so a client can be tested without hammering the API

Burst, per 10 seconds: `/media` 30 (120 signed in), `/data` 200, `/submit` 40, everything else 50.

Daily, per UTC day: `/media` 5000 per IP / 100000 per key, and 2000 for a single media target
whatever the caller; `/data` 8000 / 1500; `/submit` 1000; everything else 400 / 1500.

Limits are per *identity*, so they are not shared between a key and an anonymous run — and mods,
admins and loopback callers are exempt from the burst ceiling.

## Where each client implements this

| client | file | symbols |
| --- | --- | --- |
| plex-sync | `internal/tidb/client.go` | `retryAfter`, `usageReset`, `usageLimited`, `maxUsageWait` |
| kodi-addon | `plugin.video.tidb/introdb.py` | `RATE_RESET_CLAMP`, `USAGE_RESET_CLAMP`, `_wait_rate_limit` |
| jellyfin-plugin | `TheIntroDB/Api/TheIntroDbClient.cs` | `GetRetryAfterSeconds`, `IsUsageLimitResponse`, `RateResetClampSeconds`, `UsageResetClampSeconds` |
| emby-plugin | `TheIntroDB/Api/TheIntroDbClient.cs` | `GetRetryAfterSeconds`, `IsUsageLimitResponse`, `ClampRetryAfter`, `UsageResetClampSeconds` |
| stremio-enhanced-plugin | `tidb.plugin.js` | `MAX_RATE_LIMIT_WAIT` |
| universal-extension | `src/background.ts` | the `status === 429` branch |
| mpv-script | `scripts/theintrodb.lua` | `retry_secs`, `max_retries` |
| iina-package | `src/tidb.js` | — (no 429 branch today) |

Each of those is expected to implement rules 1–6 identically; the file is listed so the next change
to the contract can find them all.
