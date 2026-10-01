# The `get_fund` payload, field by field

`get_fund` returns `{ input, isin, status, name, cached, generatedAt, expiresAt, data }`. Everything below is under `data`. A field that does not apply to the fund's asset class, or that its documents do not disclose, is `""`, `null` or `[]`: report it as "not disclosed".

## Identity

| Field | Meaning | How to phrase it |
|---|---|---|
| `investmentObjective` | The objective paragraph from the fund's documents | Quote or summarise; it is the issuer's wording |
| `securityType`, `structure` | Share-class type ("ETF", "Mutual Fund", "Money Market Fund"…) | "an ETF", "an open-ended fund" |
| `shareClass` | Share-class label when it can be read from the name | Mention only when it distinguishes share classes |
| `keyFacts.assetClass` | Broad asset class as disclosed | |
| `keyFacts.currency` | Share-class currency (ISO 4217) | Returns and NAV are in this currency |
| `keyFacts.aum` | Fund size as reported, with unit and currency ("USD 151.7bn") | Quote as is; it is often the whole fund, not the share class |
| `keyFacts.inception` | Share-class launch date (YYYY-MM-DD) | |
| `keyFacts.distribution` | Accumulating or distributing | |
| `keyFacts.holdings` | Number of holdings | |
| `keyFacts.manager` | Portfolio manager(s) or management company | |
| `keyFacts.subAsset` | Same as `profile.category` | |
| `managerTenure` | Tenure of the longest-serving manager, when disclosed | |
| `benchmarkName` | The index tracked or the prospectus benchmark | |

## Fees (`headlineMetrics.ter`, `costs`)

- `headlineMetrics.ter`: ongoing charge, from the KID when it states one, otherwise the TER the issuer's documents state. This is the yearly running cost deducted inside the fund.
- `costs.ongoing`: the KID's "management fees and other administrative or operating costs" line (falls back to the TER).
- `costs.transaction`: portfolio transaction costs per year, on top of the ongoing charge.
- `costs.entry`, `costs.exit`: one-off costs as printed in the KID. Many ETFs show 0% here while the broker charges its own fees; FundFacts does not know the broker's fees.
- `costs.performanceFee`: when the fund charges one.
- `costs.riy1y`, `costs.riyRhp`: annual cost impact (reduction in yield) after one year and at the recommended holding period, from the KID's cost table.
- `costs.recommendedHoldingPeriod`: as stated in the KID.

When the user asks "how much does it cost", give the ongoing charge first, then transaction costs and any entry, exit or performance fee, each labelled.

## Risk and classification

- `riskRating`: SRI (PRIIPs KID) or SRRI (UCITS KIID), 1 to 7. Stated by the fund house; `null` means no document read states it, and it is never estimated.
- `sfdrArticle`: 6, 8 or 9 as declared under the EU Sustainable Finance Disclosure Regulation. A disclosure category, not a quality rating or a label.
- `replication`: `"physical"` (holds the index securities, fully or by sampling), `"synthetic"` (gets the index return from a swap) or `null` (not stated; usual for active funds).
- `holdingsBasis`: `"portfolio"` (the holdings are the fund's own positions) or `"substituteBasket"` (a synthetic fund's collateral basket, which backs the swap and does not describe the exposure). `null` without holdings.
- `profile`: FundFacts' classification, recomputable from the payload (`profile.rules` names the rule version):
  - `kind`: equity, fixedIncome, moneyMarket, allocation, alternative, other.
  - `category`: a composed label such as "Global Equity" or "EUR Investment Grade Bond"; a part is left out when the data behind it is not disclosed.
  - `riskBand`: SRRI 1–2 low, 3–4 medium, 5–7 high.
  - `concentration`: weight of the ten largest holdings, ≥50% concentrated, 30–50% balanced, under 30% diversified; `null` for names-only holdings and substitute baskets.
  - `regionFocus` (largest region when ≥80%, otherwise "Global"), `regionTilt` (a Global fund's largest region when 50–80%), `sectorTilt` (equity: largest sector when ≥30%, otherwise "Broad").
  - `valuation` (equity P/E: under 15 value, 15–22 blend, over 22 growth), `creditQuality` (bonds: high, medium, low from the rating buckets), `rateSensitivity` (bonds: duration under 3.5 years limited, 3.5–6 moderate, over 6 extensive), `equityShare` (allocation funds, percent in equity).

## Portfolio

- `topHoldings[]`: `{ name, weight }` in the issuer's order, weight in percent of the fund. Either every weight is a number or every one is `null` (names published without weights). Usually the top 10 to top 50 only, so they add up to less than 100.
- `geography[]` (countries), `region[]` (regions; mirrors geography when there is no separate regional table), `sector[]`, `assetAllocation[]`, `creditQuality[]`, `maturity[]`, `instrument[]`: `{ label, weight }` in percent. For a synthetic ETF these describe the index, which is the fund's exposure.

## Performance

- `annualisedReturns[]`: `{ label, fund, index }`. Labels: "1 Year", "3 Years p.a.", "5 Years p.a.", "10 Years p.a.", "Since Inception". `index` is the benchmark's figure when published. `null` when the share class is younger than the horizon.
- `calendarReturns`: `{ years[], fund[], benchmark[] }`, full calendar years since inception, oldest first; the inception year and the current year are left out.
- `indexedPerformance.points[]`: monthly `{ date: "YYYY-MM", fund, index }` rebased to 100 at the first point; `hasIndex` says whether the index series is filled.
- `cumulativePerformance[]`: the same months as cumulative returns in percent.

## Risk statistics and yields

- `headlineMetrics.volatility3y`: annualised standard deviation of monthly returns over 3 years (the issuer's figure when published).
- `headlineMetrics.sharpe3y`: 3-year Sharpe ratio (the issuer's, otherwise excess return over a same-currency money-market fund divided by volatility). Empty for money-market funds.
- `metrics.maxDrawdown`: largest peak-to-trough fall over the same 3 years.
- `metrics.peRatio` (equity), `metrics.incomeYield` (distribution yield), `metrics.yieldToMaturity`, `metrics.effectiveDuration`, `headlineMetrics.modifiedDuration`, `metrics.averageRating` (bonds), `metrics.sevenDayYield`, `metrics.wam`, `metrics.wal` (money market, days), `metrics.equityCorrelation`, `metrics.equityBondSplit` (allocation funds).

## Freshness

- `dataAsOf`: date of the figures (latest NAV observation, otherwise the factsheet date). Cite it with every answer.
- `generatedAt` / `expiresAt` (envelope): when FundFacts built the payload and when it will refresh it (every 24 hours). `cached: true` means the stored payload was reused.

## Search results (`search_funds`)

`{ query, count, results: [{ isin, name, issuer, currency, assetClass, shareClass, income, category, dataAsOf, url }] }`. `url` is the public FundFacts page when fund pages are public, otherwise `null`. Search covers funds already loaded into FundFacts only.
