---
name: fundfacts-api
description: Guides writing code against the FundFacts REST API (https://fundfactsapi.com/api/v1), its official SDKs (@fundfactsapi/sdk for JavaScript / TypeScript, fundfacts for Python) and the @fundfactsapi/widgets React components. Use when the user is building an app, script, notebook, dashboard, screener or back-office job that fetches fund, ETF or SCPI data by ISIN, handles the FUNDFACTS_API_KEY, renders a fund factsheet, calls /funds, /search, /portfolio, /overlap, /scpi or /factsheets, or asks how the API counts requests, caches data or reports errors.
license: MIT
metadata:
  author: FundFacts API
  version: "1.0.3"
---

# Building on the FundFacts API

FundFacts API turns a fund or ETF ISIN into one structured JSON factsheet (key facts, fees, risk indicator, holdings, exposure, returns, risk statistics), plus search, batch, portfolio look-through, holdings overlap, SCPIs and rendered factsheets. This skill is for code. To answer a question about a fund in the conversation, use the FundFacts MCP tools and the `fund-research` skill instead. Explicit instructions from the user take precedence over this skill.

## Setup

- Base URL: `https://fundfactsapi.com/api/v1`. OpenAPI: `https://fundfactsapi.com/openapi.json`. Docs: `https://fundfactsapi.com/docs`.
- Auth: `Authorization: Bearer <key>` on every call (`X-Api-Key: <key>` also works). Keys start with `ffk_` and are created in the dashboard (`https://fundfactsapi.com/dashboard`).
- Read the key from the environment variable `FUNDFACTS_API_KEY`. If it is not set, ask the user to create a key and put it in `.env` / `.env.local` (listed in `.gitignore`). Never write a key into source code, logs, commits or a browser bundle: call the API from server code or a backend route.
- The MCP server's OAuth tokens (`ffo_…`) only work on `/api/mcp`, not on the REST API.

```bash
curl -s https://fundfactsapi.com/api/v1/funds/IE00B4L5Y983 -H "Authorization: Bearer $FUNDFACTS_API_KEY"
```

Prefer the SDKs over hand-written HTTP; they handle the timeout, the cache and the error shape ([references/sdks.md](references/sdks.md)):

```ts
import { FundFacts } from "@fundfactsapi/sdk"; // npm install @fundfactsapi/sdk
const ff = new FundFacts(); // reads FUNDFACTS_API_KEY
const fund = await ff.getFund("IE00B4L5Y983");
console.log(fund.name, fund.data.headlineMetrics.ter, fund.data.riskRating, fund.data.dataAsOf);
```

```python
from fundfacts import FundFacts  # pip install fundfacts
ff = FundFacts()  # reads FUNDFACTS_API_KEY
fund = ff.get_fund("IE00B4L5Y983")
```

## Endpoints at a glance

| Call | Purpose | Counts |
|---|---|---|
| `GET /funds/{isin}` | One factsheet | 1 request |
| `POST /funds` `{ isins, wait }` | Batch; per-item `status` | 1 per ISIN answered |
| `GET /search?q=&limit=` | Names → ISINs, over funds already loaded | not counted |
| `POST /portfolio` `{ positions: [{ isin, weight }] }` | Look-through (not in every plan) | 1 per position |
| `GET /overlap?isins=A,B` | Pairwise overlap (not in every plan) | 1 per ISIN |
| `GET /scpi/{slugOrIsin}`, `GET /scpi?q=` | French SCPIs | 1 / not counted |
| `POST /factsheets` | HTML or PDF one-pager, or keyless share links | 1 (HTML) / 2 (PDF) |
| `GET /me` | Plan and quota | not counted |

Request and response shapes, and the `/changes`, `/webhooks`, `/export` and `/extract` endpoints: [references/endpoints.md](references/endpoints.md). Look-through, overlap and those four endpoints are not included in every plan (https://fundfactsapi.com/docs/requests); an account without them gets 403 `plan_required`.

## Rules the code must follow

1. **Slow first loads.** A fund nobody loaded in the last 24 hours is read from its documents on demand: 15 seconds to 3 minutes. Use a 300-second timeout on `/funds/{isin}` (the SDKs default to it), show a loading state in UIs, and never fire the same ISIN twice at once.
2. **Batches and `pending`.** `POST /funds` answers HTTP 200 with a per-item `status`: `ok`, `not_found`, `pending`, `invalid` or `error`, and a top-level `pending` list. `pending` ISINs are still loading in the background (not charged): re-send just those later, once, after a minute or more. With `wait: false` every cold ISIN comes back `pending` at once, which suits background warm-up jobs.
3. **Cache by ISIN until `expiresAt`.** Payloads refresh at most once per 24 hours; re-fetching earlier costs a request for the same data.
4. **Quota.** One request per ISIN answered (found or not found). Read `X-RateLimit-Remaining` and `X-Burst-Remaining`; on 429 wait `Retry-After` seconds (`reason`: `burst`, `quota_exhausted`, `overage_cap`). Batch size per call depends on the account's plan: read it from `GET /me` or from `batchMax` in a 400 `batch_too_large`, and chunk larger lists.
5. **Errors** are `{ "error": { "code", "message", ... } }`. Retry only 502 `upstream_error`, at most twice, 60 s apart. `404 fund_not_found` means "not a covered fund": surface it, do not retry. Full table: [references/errors-and-limits.md](references/errors-and-limits.md).
6. **Data shape.** Fields that do not apply are `""`, `null` or `[]` (not disclosed, never zero). Fee, size and statistic fields are formatted strings (`"0.20%"`, `"USD 151.7bn"`); parse them only when the user needs numbers. Breakdown weights and returns are numbers in percent. Show `data.dataAsOf` next to figures. Use `data.profile` to classify and filter.
7. **Holdings pitfalls.** `topHoldings[].weight` is `null` for every row when the fund house publishes names only: never estimate. When `data.holdingsBasis` is `"substituteBasket"` (a synthetic ETF, `data.replication: "synthetic"`), the holdings are the swap's collateral, not the exposure: use `sector` / `geography` / `region`, and keep such holdings out of look-through or overlap code (the `/portfolio` and `/overlap` endpoints already do).
8. **Sourcing.** Do not scrape fundfactsapi.com, fund house sites or data vendors as a workaround; the API is the source. Name the product "FundFacts" in UIs.

## Rendering

`@fundfactsapi/widgets` renders an API response as React components with inline styles (server-component safe, light and dark): `FundFactsheet` for the whole page, or `GrowthChart`, `DonutChart`, `WeightBars`, `KeyFacts`, `RiskScale`, `ProfileChips`, `CalendarReturns`, `AnnualisedReturns`, `RiskMetrics`, `FreshnessDial`, `WidgetCard`. For a PDF or a shareable link without building a page, `POST /factsheets`. Details: [references/sdks.md](references/sdks.md).

## MCP server for coding agents

The same data is available to agents at `https://fundfactsapi.com/api/mcp` (Streamable HTTP; seven tools: `get_fund`, `search_funds`, `compare_funds`, `analyze_portfolio`, `fund_overlap`, `get_scpi`, `search_scpi`). Clients that support OAuth sign in with the FundFacts account; others send `Authorization: Bearer <ffk_ key>`. Setup snippets: [references/endpoints.md](references/endpoints.md#mcp-server).
