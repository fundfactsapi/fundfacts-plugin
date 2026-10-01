---
name: portfolio-analysis
description: Analyses a portfolio of funds and ETFs with the FundFacts MCP server's analyze_portfolio (look-through of blended fees, weighted risk indicator, asset class, sector, country and region exposure, currency split and combined top holdings) and fund_overlap (how much two or more funds hold in common). Use when the user lists several funds with weights or amounts and asks what they really hold, their overall fees or risk, their exposure to a country or sector, whether their ETFs overlap or are redundant, or how diversified the portfolio is.
license: MIT
metadata:
  author: FundFacts API
  version: "1.0.4"
---

# Portfolio look-through and overlap with FundFacts

Two tools on the FundFacts MCP server work across several funds. Both read the same per-fund data as `get_fund` and count one request per ISIN; they are not included in every FundFacts plan (see section 5). Explicit instructions from the user take precedence over this skill.

| Tool | Input | Answers |
|---|---|---|
| `analyze_portfolio` | `positions: [{ isin, weight }]` | Blended TER, weighted risk indicator, asset / sector / country / region / credit exposure, currency split, combined top holdings, each with a coverage figure |
| `fund_overlap` | `isins: [...]` (two or more) | For every pair: overlap percentage, shared holdings, and whether the pair can be compared at all |

## 1. Collect the positions

- Weights can be percentages or amounts in one currency ("€12,000 in X, €8,000 in Y"): the tool normalises them so they sum to 100. Pass the numbers as the user gave them; do not convert. If every weight is zero, the tool weights the funds equally; ask for weights instead when they matter.
- Each position needs an ISIN. Resolve names with `search_funds` (not counted) and confirm the matches with the user before the counted call: a wrong share class uses a request and changes the answer. The same ISIN listed twice is summed.
- Ask for anything missing in one question (the missing ISIN, the weights) rather than guessing.
- The account's plan caps the positions per call, and the tool says so when a list is too long. Above the cap, see "Large portfolios" in [references/reading-results.md](references/reading-results.md).

## 2. Call and check what was covered

Call `analyze_portfolio` once. Then read, before any figure:

- `coverage`: the share of the portfolio (percent) with fund data. `positions[].covered` is false for funds that are not covered or still loading.
- `pending`: ISINs being loaded for the first time. Their weight is in the portfolio but not in the figures. A second call would count every position again, so tell the user and offer to re-run in a minute rather than re-running on your own.
- `invalid`: inputs that are not valid ISINs (not charged). They are dropped **before** the weights are normalised, so every figure, `coverage` included, describes the valid positions rescaled to 100. Name the excluded inputs and their weight in the answer ("IE00B4L5Y98 (20%) is not a valid ISIN and is left out; the figures describe the other 80%, rescaled") and ask the user to correct them.

State the coverage in the answer whenever it is below 100: "Figures cover 85% of the portfolio; X is not covered."

## 3. Read the results

Full field guide: [references/reading-results.md](references/reading-results.md). The rules that most often go wrong:

- **Breakdowns are shares of the whole portfolio.** `sector.items[].weight` is the percent of the total portfolio in that sector, from the funds that disclose a sector split. The items add up to about `sector.coverage`, not to 100. To express them relative to the covered part, divide by `coverage / 100`, and say which you did.
- **Fees.** `fees.weightedTer` is the weight-averaged ongoing charge over the positions that disclose one (`fees.coverage`). `fees.annualCostPer10k` is the same in currency units per 10,000 invested per year (0.25% → 25). Transaction costs and broker fees are not included.
- **Risk.** `risk.weightedSrri` is a weighted average of the funds' 1–7 risk indicators, with `risk.band`. It is a rough summary, not the portfolio's own SRI: a mix of funds can be less volatile than its parts. Say so.
- **Currency** (`currency`) is the split by share-class currency, not the currency exposure of the underlying assets.
- **Top holdings are a lower bound.** Funds disclose their largest positions only, so combined holdings understate the true exposure. Swap-based ETFs are left out of `topHoldings` (listed in `topHoldings.excluded`: their holdings are a collateral basket) while their sector and country exposure still counts; funds that publish names without weights are left out too (`topHoldings.namesOnly`). Quote `topHoldings.note` in substance.

## 4. Overlap

Call `fund_overlap` with the ISINs. For each pair:

- `overlap`: sum over shared holdings of the smaller weight in either fund, in percent. It is measured over the holdings each fund discloses (`disclosed.a`, `disclosed.b`), so with top-10 disclosures it is a lower bound.
- `comparable: false`: the pair was not measured. Either one fund is swap-based (its collateral basket is not compared) or one fund has no weighted holdings to compare: it is still loading (`pending`), not covered (`notFound`), or discloses no holdings (`funds[].weighted: false`, `holdingsBasis: null`). Such a pair comes back with `overlap: null` (a swap-based pair with `overlap: 0`, which is not a measurement either); never report it as "no overlap" or 0. Say which fund could not be compared and why; for a swap-based fund, compare sector and country exposure with `get_fund` or `analyze_portfolio` instead.
- `overlap: null` with `comparable: true`: a fund publishes names without weights. Report `sharedCount` and `sharedNames` ("they share 7 of their top 10 names"), not a percentage.
- An `overlap` of 0 is a measurement only when `comparable` is true and both funds have `weighted: true`. (Older server versions returned 0 with `comparable: true` for a pending, not-covered or undisclosed fund: check `pending`, `notFound` and `funds[]` before calling any 0 real.)
- `shared[]` lists the shared positions with both weights; show the five largest.
- `errored`: ISINs whose documents could not be read this time. Their pairs are not measured either; offer to run the overlap again in a few minutes rather than saying the fund discloses nothing.

Explain what the number means in plain words ("about 60% of fund A's disclosed holdings are also in fund B at the same or higher weight"). High overlap means the funds hold largely the same securities; it does not say which to keep.

## 5. Account limits

When the account's plan does not include look-through and overlap, both tools answer with a short message saying so, with an informational link (https://fundfactsapi.com/docs/requests). Relay it neutrally in one sentence, with that link. Do not quote prices, suggest changing plan, or point to billing or checkout pages. If the user still wants something useful, offer to fetch each fund with `get_fund` (one request each) and describe their exposures side by side; say that this is not an aggregated look-through.

The same applies to the allowance: when a message says the monthly requests are used up, say when they reset if the message gives the date, and stop calling tools that count requests.

## What to say and not say

- Describe the portfolio; do not tell the user to buy, sell or rebalance. If they ask whether it is "well diversified" or "too concentrated", give the facts that bear on it (largest country and sector weights, combined top holdings, overlap between funds, fees) and leave the judgement to them or a licensed adviser.
- Show the numbers in a compact table (exposure, weight), then two or three sentences of reading. Round to one decimal.
- End with the coverage and the date line: "Source: FundFacts, from the funds' documents; figures cover N% of the portfolio." When the funds' `dataAsOf` dates matter (a question about recent changes), fetch them with `get_fund` only if the user asks.
- Answer in the user's language.
