# SDKs and widgets

All three packages are MIT-licensed and documented at `https://fundfactsapi.com/docs/sdks`.

## JavaScript / TypeScript: `@fundfactsapi/sdk`

Zero dependencies; Node 18+, Bun, Deno and edge runtimes.

```bash
npm install @fundfactsapi/sdk
```

```ts
import { FundFacts, FundFactsError } from "@fundfactsapi/sdk";

const ff = new FundFacts(); // or new FundFacts({ apiKey, baseUrl, timeoutMs, fetch, cache })

const fund = await ff.getFund("IE00B4L5Y983");                  // 1 request; cached in memory until expiresAt
const batch = await ff.getFunds(["IE00B4L5Y983", "IE00BK5BQT80"]); // POST /funds; batch.pending lists ISINs still loading
const hits = await ff.search("msci world");                       // not counted
const look = await ff.portfolio([{ isin: "IE00B4L5Y983", weight: 60 }, { isin: "IE00BK5BQT80", weight: 40 }]);
const ov = await ff.overlap(["IE00B4L5Y983", "IE00BK5BQT80"]);
const me = await ff.me();                                        // plan and quota
const html = await ff.factsheet("IE00B4L5Y983", { format: "html" }); // string
const pdf = await ff.factsheet("IE00B4L5Y983", { format: "pdf" }); // ArrayBuffer
const link = await ff.factsheet("IE00B4L5Y983", { deliver: "link" }); // { urls: { html, pdf, embed } }
// Not included in every plan: ff.changes({ since }), for await (const f of ff.export({ issuer: "iShares" })) { ... }

try {
  await ff.getFund(isin);
} catch (e) {
  if (e instanceof FundFactsError) console.error(e.status, e.code, e.message, e.retryAfter);
}
```

- The constructor throws when neither `apiKey` nor `FUNDFACTS_API_KEY` is set.
- Default timeout 300 s, matching the slowest first load.
- There is no SCPI method yet: call `GET /scpi/{slug}` and `GET /scpi?q=` with `fetch` and the same header.

## Python: `fundfacts`

Standard library only, Python 3.9+.

```bash
pip install fundfacts
```

```python
from fundfacts import FundFacts, FundFactsError

ff = FundFacts()  # reads FUNDFACTS_API_KEY; FundFacts(api_key=..., timeout=300.0, cache=True, retry_on_burst=True)

fund = ff.get_fund("IE00B4L5Y983")
batch = ff.get_funds(["IE00B4L5Y983", "IE00BK5BQT80"], wait=True)
hits = ff.search("msci world")
look = ff.portfolio([{"isin": "IE00B4L5Y983", "weight": 60}, {"isin": "IE00BK5BQT80", "weight": 40}])
ov = ff.overlap(["IE00B4L5Y983", "IE00BK5BQT80"])
me = ff.me()
pdf = ff.factsheet("IE00B4L5Y983", format="pdf")  # bytes
```

- Raises `FundFactsError` (`status`, `code`, `message`, `retry_after`); a burst 429 is retried once automatically.
- For analysis: `pandas.json_normalize(fund["data"]["topHoldings"])`; a holding's `weight` is `None` when the fund house publishes names only.
- Also: `changes(...)`, `export(...)` (iterator; `changes` and `export` are not included in every plan (https://fundfactsapi.com/docs/requests)), `factsheets(...)`.

## React: `@fundfactsapi/widgets`

```bash
npm install @fundfactsapi/widgets @fundfactsapi/sdk
```

```tsx
import { FundFacts } from "@fundfactsapi/sdk";
import { FundFactsheet, FundFactsTheme } from "@fundfactsapi/widgets";

export default async function Page() {
  const fund = await new FundFacts().getFund("IE00B4L5Y983"); // server side: the key never reaches the browser
  return (
    <FundFactsTheme dark={false}>
      <FundFactsheet fund={fund} />
    </FundFactsTheme>
  );
}
```

Components take the raw slice of `data` and render nothing misleading when a field is not disclosed:

| Component | Reads |
|---|---|
| `FundFactsheet` | the whole envelope |
| `GrowthChart` | `data.indexedPerformance.points`, `data.benchmarkName` |
| `DonutChart` | any `[{ label, weight }]` |
| `WeightBars` | `data.topHoldings`, `data.geography`, … (names-only holdings listed without bars) |
| `KeyFacts`, `RiskMetrics`, `StatTiles` | `data.keyFacts`, `data.headlineMetrics`, `data.metrics` |
| `RiskScale` | `data.riskRating` |
| `ProfileChips` | `data.profile` |
| `CalendarReturns`, `AnnualisedReturns` | `data.calendarReturns`, `data.annualisedReturns` |
| `FreshnessDial` | `generatedAt`, `expiresAt`, `data.dataAsOf` |
| `WidgetCard` | a titled card around any of the above |

Inline styles only (no CSS import), server-component safe. Theme with CSS variables (`--ff-accent`, `--ff-text`, `--ff-surface`, …) or `FundFactsTheme`. Every visible label is a prop, and `locale` (e.g. `"fr-FR"`) formats numbers on the bars and returns tables.
