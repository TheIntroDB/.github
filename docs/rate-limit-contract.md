# Client contract: the two 429s from `api.theintrodb.org`

An integration eventually gets a `429`. There are two *kinds* of them — a **burst** ceiling and a
**daily budget** — and the API shapes both identically on purpose: same JSON keys, same
`Retry-After`, so a client only has to parse one `429`. The `code` field is the only thing that
separates "slow down for a few seconds" from "you are out of budget until tomorrow".

Clients must handle them the same way. This is that contract.

The daily budget reports under **two codes, not one**, and the second is not a separate regime:
`specific_media_usage_limit_exceeded` is the same 24-hour budget for the caller's per-title bucket,
described in the third column of the table below. The API charges the per-title counter first
(`middleware/usage.go`), so when the title itself is what ran out, that is the code returned —
*instead of* `usage_limit_exceeded`. A client that branches on `usage_limit_exceeded` alone will
miss it exactly when it matters, so both codes take the same branch. Rules 1 and 2 do not care
which code arrived; only rule 3's policy differs.

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

The last two columns are the *same* daily budget at two scopes, not two regimes: column three is
column two for one title. Both reset at UTC midnight and report their own `X-UsageLimit-*` trio on
the `429`, and a client treats both as "out of budget until tomorrow" — the only difference is what
it may keep doing meanwhile (rule 3).

`Retry-After` is always present, always in RFC 9110 delta-seconds, and always the same instant as
the matching `X-…-Reset`. **It is the one piece of the `429` a client may depend on**: the burst
`429` carries no `X-RateLimit-*` counters at all, while the daily one does carry its own. Take the
wait from `Retry-After` and read the counters off a `200`.

Scope what you cache, because the buckets are not shared. A counter means nothing without the
request that produced it: these limits are per endpoint *and* per identity, so only carry counters
from the last successful response **for the same endpoint and the same caller** — a key's budget is
not an anonymous caller's, and `/media` counters do not describe `/submit`. `X-UsageLimit-SpecificMedia-*`
is narrower still: it describes one media target, so use it only against a response for that same
target and keep it in its own cache keyed by target — one title's remaining budget says nothing
about another's.

## The rules

**1. The wait is `Retry-After`, whatever the code.** Fall back in this order, and stop at the first
usable value: `Retry-After` → `retry_after_seconds` → `retry_after` (`"14.0 hours"`) →
`X-…-Reset` → 60 s. That value is the **base wait**: it is a floor, not a suggestion. A client may
lengthen it under rule 3 and must never shorten it, and nothing outside this contract should be
consulted to compute one.

**2. Clamp only as a ceiling — never down to the burst window.** `0 ≤ wait ≤ 86400`.

Clamping a daily `429` to a burst-shaped wait (5 minutes, 30 seconds) is the one that bites: it
turns "come back tomorrow" into "come back in five minutes", so the client wakes up, spends one
request, is refused again, and works through the next day's budget one request at a time — or,
worse, loops until it is stopped by hand. An implausible-looking wait is real. 14 hours means 14
hours.

**3. Branch on `code` for policy, not for duration.** Both 429s carry the same numbers; what a
client *does about it* is where they differ.

- `rate_limit_exceeded` — sleep the wait, then retry. The run continues. Growing the wait across
  consecutive bursts (×1, ×2, ×4, … capped) is good practice, and it is the **only** thing allowed
  to move the rule 1 value: it multiplies that value upward, and rule 2 still caps the result. A
  client never lengthens the wait for a daily-budget `429` — the budget is spent, and the next
  bullet is what to do instead.
- `usage_limit_exceeded` / `specific_media_usage_limit_exceeded` — **stop and report**. Do not sit
  in a request loop until a 24-hour window rolls over; resume on the next scheduled run, or on the
  next thing the user does. A tool with a queue should persist what is left and say when it will
  continue.

  **Hold the two differently.** `specific_media_usage_limit_exceeded` is spent for one exact media
  target — the same id *and* season and episode — so hold it against that target and keep asking
  about everything else: a spent title must not stop the rest of the library. `usage_limit_exceeded`
  is the caller's own budget and stops every request until it resets. Clients store the first keyed
  by media target and the second as a single deadline, and check the deadline first.

When the body carries no `code` at all — a proxy's `429`, say — read a wait **beyond 300 s** as a
daily budget and anything shorter as a burst. That line is not arbitrary: it is what jellyfin-plugin,
emby-plugin, kodi-addon and the extension all use, so an unrecognised `429` is classified the same
way by every client.

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
| theintrodb-npm | `src/funcs.ts` | `readRateLimitHeaders`, `RateLimitDetails`, and the `rateLimit` on the thrown error — parses all three counter trios, no `429` branch today |
| mpv-script | `scripts/theintrodb.lua` | `retry_secs`, `max_retries` |
| iina-package | `src/tidb.js` | — (no 429 branch today) |

Each client that branches on a `429` is expected to implement rules 1–6 identically; the file is
listed so the next change to the contract can find them all. Two rows are not there yet, and they
are listed for the same reason: `theintrodb-npm` reads the counters off every response and hands
them to the caller on the thrown error without ever acting on them, and `iina-package` throws on any
non-2xx. Neither follows rules 1–3 today — which is the gap this contract exists to close, since
rule 5 makes the counters on a `200` the thing a client paces against.
