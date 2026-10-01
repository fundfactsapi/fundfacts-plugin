# Endpoints

Base `https://fundfactsapi.com/api/v1`, header `Authorization: Bearer $FUNDFACTS_API_KEY` on every call. The OpenAPI document at `https://fundfactsapi.com/openapi.json` is the machine-readable reference; the field reference is at `https://fundfactsapi.com/docs/fields`.

## `GET /funds/{isin}`

The envelope:

```json
{
  "isin": "IE00B4L5Y983",
  "name": "iShares Core MSCI World UCITS ETF",
  "cached": true,
  "generatedAt": "2026-09-02T20:32:58.872Z",
  "expiresAt": "2026-09-03T20:32:58.872Z",
  "plan": "pro",
  "quota": { "limit": 9000, "remaining": 8123, "resetAt": "2026-10-01T00:00:00.000Z" },
  "data": { "...": "the factsheet, see below" }
}
```

`data` groups (full list at `/docs/fields`):

- Identity: `investmentObjective`, `securityType`, `structure`, `shareClass`, `keyFacts.{assetClass, currency, aum, inception, subAsset, distribution, holdings, manager}`, `managerTenure`, `benchmarkName`, `replication`.
- Fees and risk: `headlineMetrics.ter`, `costs.{entry, exit, ongoing, transaction, performanceFee, riy1y, riyRhp, recommendedHoldingPeriod}`, `riskRating` (1–7 or null), `sfdrArticle` (6, 8, 9 or null).
- Profile: `profile.{kind, category, riskBand, concentration, regionFocus, regionTilt, sectorTilt, valuation, creditQuality, rateSensitivity, equityShare, rules}`.
- Portfolio: `topHoldings[{ name, weight|null }]`, `holdingsBasis`, `geography`, `region`, `sector`, `creditQuality`, `assetAllocation`, `instrument`, `maturity` (all `[{ label, weight }]`, percent).
- Performance: `calendarReturns.{years, fund, benchmark}`, `annualisedReturns[{ label, fund, index }]`, `indexedPerformance.{points[{ date, fund, index }], hasIndex}`, `cumulativePerformance[{ date, fund, benchmark }]`.
- Statistics: `headlineMetrics.{volatility3y, sharpe3y, yieldToMaturity, modifiedDuration, aum}`, `metrics.{maxDrawdown, peRatio, incomeYield, effectiveDuration, effectiveMaturity, averageRating, sevenDayYield, wam, wal, equityCorrelation, equityBondSplit, ...}`.
- Freshness: `dataAsOf` (date of the figures), `generatedAt`.

## `POST /funds`

Body `{ "isins": ["IE00B4L5Y983", "IE00BK5BQT80"], "wait": true }`, up to the plan's batch size.

```json
{
  "count": 2,
  "results": [{ "input": "...", "isin": "...", "status": "ok", "name": "...", "cached": true, "generatedAt": "...", "expiresAt": "...", "data": {} }],
  "pending": [],
  "plan": "pro",
  "quota": {}
}
```

`status`: `ok`, `not_found` (no data; counted), `pending` (loading in the background; not counted; re-send later), `invalid` (bad ISIN; not counted), `error` (retrieval failed). `wait: false` answers from the store only and returns every cold ISIN as `pending` while it is warmed in the background.

## `GET /search?q=msci world&limit=10`

Free. `{ query, count, results: [{ isin, name, issuer, currency, assetClass, shareClass, income, category, dataAsOf, url }] }`. Covers funds already loaded into FundFacts; an empty result does not mean the fund does not exist.

## `POST /portfolio` (Pro, Scale, Enterprise)

Body `{ "positions": [{ "isin": "IE00B4L5Y983", "weight": 60 }, { "isin": "IE00BK5BQT80", "weight": 40 }], "wait": true }`. Weights are normalised to 100. Returns `positions[]`, `coverage`, `fees.{weightedTer, coverage, annualCostPer10k}`, `risk.{weightedSrri, band, coverage}`, `kinds`, `assetAllocation`, `sector`, `geography`, `region`, `creditQuality` (each `{ items: [{ label, weight }], coverage }`, weights in percent of the whole portfolio), `currency`, `topHoldings.{items, coverage, note, excluded, namesOnly}`, `rules`, `pending`.

## `GET /overlap?isins=A,B,C` (Pro, Scale, Enterprise)

`{ funds: [{ isin, name, holdings, holdingsBasis, weighted }], pairs: [{ a, b, overlap, sharedCount, shared: [{ label, a, b }], sharedNames, disclosed: { a, b }, comparable }], excluded, namesOnly, note, rules }`. `overlap` is null (not 0) when a fund publishes names without weights. `comparable` is false when a fund is swap-based (overlap 0, not a measurement) and when a fund has no weighted holdings to compare (still loading, not covered, or none disclosed: `holdings` 0, `holdingsBasis` null); such pairs have `overlap: null`, and neither kind is a real 0.

## SCPI

- `GET /scpi?q=epargne pierre&limit=10`: free; `results: [{ slug, isin, name, manager, managerName, category, loaded, dataAsOf, url }]`.
- `GET /scpi/{slugOrIsin}`: one request; a cold SCPI takes 10–60 seconds. Returns the SCPI record (`keyFacts`, `performance`, `portfolio`, `liquidity`, `documents`, `dataAsOf`, `bulletinPeriod`). Field guide: `https://fundfactsapi.com/docs/scpi`.

## `POST /factsheets`

Body `{ "isin": "...", "format": "html" | "pdf", "deliver": "file" | "link", "theme", "accent", "title", "branding", "logo" }` (`logo` is an https URL, Pro and up). Returns the document (share links in `X-Factsheet-Url` / `X-Factsheet-Pdf-Url`) or, with `deliver: "link"`, `{ id, isin, name, issuer, format, options, favorite, createdAt, dataAsOf, urls: { html, pdf, embed } }`; the `urls` need no key. `GET /factsheets` lists, `GET /factsheets/{id}?format=pdf` re-renders, `PATCH` `{ favorite }`, `DELETE`. One request for HTML, two for PDF; list, get and share links are free.

## Other endpoints

| Call | Plan | Purpose |
|---|---|---|
| `GET /me` | all | `{ email, plan, quota, usage }`; call once at start-up, not per request |
| `GET /demo/funds/{isin}` | none (no key) | Keyless read of a fund already in the store, 30 per minute per IP; for checking the shape only |
| `POST /extract` | Pro and up | Any KID / factsheet (multipart `file`, or `{ url }` / `{ text }`) → the same `data` shape; 5 requests per document |
| `GET /changes?since=&isins=` | Scale and up | Change feed (`fund.changed`, `fund.refreshed`), paged with `nextSince` |
| `/webhooks` | Scale and up | Push the same events, signed with `X-FundFacts-Signature` |
| `GET /export?issuer=&assetClass=&currency=&since=` | Scale and up | NDJSON of every stored fund matching the filter |

## MCP server

`https://fundfactsapi.com/api/mcp`, Streamable HTTP. Tools: `get_fund`, `search_funds`, `compare_funds`, `analyze_portfolio`, `fund_overlap`, `get_scpi`, `search_scpi`; same accounting as the REST API.

- Claude Code with OAuth (sign in with `/mcp`): `claude mcp add --transport http fundfacts https://fundfactsapi.com/api/mcp`, or install the FundFacts plugin: `/plugin marketplace add fundfactsapi/fundfacts-plugin` then `/plugin install fundfacts@fundfacts`.
- Claude Code with a key: `claude mcp add --transport http fundfacts https://fundfactsapi.com/api/mcp --header "Authorization: Bearer $FUNDFACTS_API_KEY"`.
- Cursor, Windsurf, VS Code and other clients: the URL plus the header `Authorization: Bearer <ffk_ key>`, or OAuth where the client supports it (Cursor does: leave out `headers` and sign in when it asks). With a key, e.g. `.cursor/mcp.json`:

```json
{ "mcpServers": { "fundfacts": { "url": "https://fundfactsapi.com/api/mcp", "headers": { "Authorization": "Bearer ${env:FUNDFACTS_API_KEY}" } } } }
```

- Claude web and desktop (Customize → Connectors → Add custom connector), ChatGPT (developer mode): add a custom connector with the URL and sign in (OAuth); these apps cannot send an API key.
