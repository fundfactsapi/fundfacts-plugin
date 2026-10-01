# Errors, quota and limits

## Error shape

Every error is `{ "error": { "code": string, "message": string, ...details } }`.

| HTTP | `error.code` | When | What the code should do |
|---|---|---|---|
| 400 | `invalid_isin` | Not a valid ISIN (format or check digit), or a batch with no valid ISIN | Fix the input; validate ISINs before calling |
| 400 | `invalid_body` | The POST body is not the expected JSON | Fix the request |
| 400 | `invalid_query` | `/search` with `q` missing or under two characters | Fix the request |
| 400 | `invalid_id` | `/scpi/{id}` with an empty id or one over 80 characters | Fix the request |
| 400 | `batch_too_large` | More ISINs than the plan's batch size; includes `batchMax` | Chunk by `batchMax` |
| 401 | `missing_api_key`, `invalid_api_key` | No key, or an unknown one (also: an OAuth `ffo_` token sent to the REST API) | Check `FUNDFACTS_API_KEY`; ask the user for a valid key |
| 401 | `revoked_api_key` | The key was revoked in the dashboard | Ask the user for a new key |
| 403 | `plan_required` | The endpoint needs a higher plan (portfolio, overlap, extract: Pro; changes, webhooks, export: Scale); includes `required` | Tell the user which plan the feature needs; do not retry |
| 404 | `fund_not_found` | No public data for this ISIN (not a fund, delisted, or not covered yet) | Surface it; do not retry or look elsewhere |
| 404 | `scpi_not_found` | `/scpi/{id}`: the slug, ISIN or name matches no known SCPI | Look the name up with `GET /scpi?q=` first |
| 429 | `rate_limited` | `reason`: `burst` (per-minute limit), `quota_exhausted` (monthly allowance), `overage_cap`; includes `retryAfter`, `resetAt` | Wait `Retry-After` seconds; reduce parallelism |
| 502 | `upstream_error` | The documents could not be retrieved right now (counted on `GET /funds/{isin}`; not counted for batch entries) | Retry after 60 s, at most twice |

## What counts as a request

- One request per ISIN answered (found or not found) on `/funds/{isin}`, `POST /funds`, `/portfolio` and `/overlap`, whether the payload came from the 24-hour store or was loaded fresh.
- One per call on `/scpi/{id}`; one (HTML) or two (PDF) per `POST /factsheets`; five per `/extract` document.
- Free: `/search`, `/scpi?q=`, `/me`, listing and re-reading saved factsheets, share links.
- Not counted: invalid ISINs inside `POST /funds`, `/portfolio` and `/overlap` (they are listed under `invalid`), `pending` entries, batch entries that failed to load, rejected calls (401, 429).
- Counted: a malformed ISIN on `GET /funds/{isin}` (400 `invalid_isin`), a malformed id or an unknown SCPI on `GET /scpi/{id}` (400 `invalid_id`, 404 `scpi_not_found`). Validate ISINs (format and check digit) and resolve SCPI names with `GET /scpi?q=` before calling.
- MCP tool calls count against the same account's quota: one request per fund answered and per SCPI found. Over MCP an invalid ISIN and an unknown SCPI are not counted, unlike the REST calls above.

## Headers

| Header | Meaning |
|---|---|
| `X-RateLimit-Limit` | Requests per month |
| `X-RateLimit-Remaining` | Requests left this month after this call |
| `X-RateLimit-Reset` | Unix time (seconds) of the monthly reset (UTC calendar month) |
| `X-Burst-Limit`, `X-Burst-Remaining` | Requests allowed per minute, and left |
| `Retry-After` | On 429 only: seconds to wait |

The same numbers are in the `quota` object of each response. Stop before `remaining` reaches 0 in batch jobs.

## Plans (limits only)

| Plan | Requests per month | Burst per minute | ISINs per batch call | Portfolio / overlap | API keys |
|---|---|---|---|---|---|
| Free | 15 | 10 | 1 | no | 1 |
| Starter | 500 | 30 | 10 | no | 2 |
| Pro | 9,000 | 120 | 50 | yes | 10 |
| Scale | 60,000 | 600 | 200 | yes | 10 |
| Enterprise | metered | 1,200 | 1,000 | yes | 50 |

Plan changes happen in the dashboard (`https://fundfactsapi.com/dashboard`), never from code. The current plan's figures are in `GET /me`; prefer them over this table in code.

## Timing

- Warm ISIN (loaded by anyone in the last 24 hours): well under a second, `cached: true`.
- Cold ISIN: 15 seconds to 3 minutes; the server allows 300 seconds. Two callers asking for the same cold ISIN share one load.
- A cold SCPI: 10 to 60 seconds.
- Payloads refresh when older than 24 hours (`expiresAt`); `data.dataAsOf` is the date of the underlying figures, usually monthly.
