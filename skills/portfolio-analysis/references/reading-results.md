# Reading `analyze_portfolio` and `fund_overlap`

Both tools follow a fixed, published rule set; the result's `rules` field names the version ("fundfacts-portfolio/3"). Weights are percentages.

## `analyze_portfolio`

```
{
  positions: [{ isin, name, weight, covered, ter, riskRating, category, kind }],
  coverage,
  fees: { weightedTer, coverage, annualCostPer10k },
  risk: { weightedSrri, band, coverage },
  kinds: [{ label, weight }],
  assetAllocation | sector | geography | region | creditQuality: { items: [{ label, weight }], coverage },
  currency: [{ label, weight }],
  topHoldings: { items: [{ label, weight }], coverage, note, excluded: [isin], namesOnly: [isin] },
  rules,
  pending: [isin],
  invalid: [input]
}
```

| Field | Meaning |
|---|---|
| `positions[].weight` | The position's weight after normalisation (all valid positions sum to 100; inputs in `invalid` are dropped before normalising) |
| `positions[].covered` | Whether FundFacts had data for the fund in this call |
| `positions[].ter`, `riskRating`, `category`, `kind` | The fund's ongoing charge (number, percent), 1–7 risk indicator, FundFacts category and kind (equity, fixedIncome, moneyMarket, allocation, alternative, other) |
| `coverage` | Percent of the portfolio with fund data |
| `fees.weightedTer` | Ongoing charge averaged over the positions that disclose one, weighted by position; percent |
| `fees.coverage` | Percent of the portfolio behind `weightedTer` |
| `fees.annualCostPer10k` | `weightedTer` expressed per 10,000 invested per year, in the portfolio's currency units |
| `risk.weightedSrri`, `risk.band`, `risk.coverage` | Weighted average risk indicator (1–7), its band (under 2.5 low, under 4.5 medium, otherwise high) and its coverage |
| `kinds` | Split by fund kind, percent of the portfolio |
| `assetAllocation` | Asset classes; a single-asset fund without an allocation table counts fully in its class |
| `sector`, `geography` (countries), `region`, `creditQuality` | Look-through exposure. `items[].weight` is the percent of the whole portfolio; `coverage` is the share of the portfolio whose funds disclose that table. Up to 30 items |
| `currency` | Split by the share classes' currencies, not by the currency of the underlying assets |
| `topHoldings` | Combined largest positions across funds (matched by issuer name), percent of the whole portfolio, up to 25; a lower bound |
| `topHoldings.excluded` | Swap-based funds whose holdings are a collateral basket, left out of the holdings (their sector and country tables still count) |
| `topHoldings.namesOnly` | Funds that publish holdings without weights, left out of the holdings and of their coverage |
| `pending` | ISINs still loading for the first time; not in the figures |
| `invalid` | Inputs that are not valid ISINs. They are left out before the weights are normalised, so the other positions are rescaled to 100 and `coverage` does not reflect them: name them and their weight in the answer |

### Worked reading

Coverage 100, `sector.coverage` 90 (one fund, 10% of the portfolio, publishes no sector split), technology item 22.5:

> About 22.5% of the whole portfolio is in technology companies (25% of the part whose funds publish a sector split; fund X publishes none).

Coverage 80 with one fund in `pending`:

> These figures cover 80% of the portfolio: X is being loaded by FundFacts for the first time. I can run the analysis again in a minute to include it (it counts each position again).

## Large portfolios

The account's plan caps positions per call; the tool says so when a list is too long. Above the cap, with the user's agreement:

1. Split the positions into groups under the cap and note each group's share of the total (sum of its weights over the grand total).
2. Call `analyze_portfolio` per group.
3. Combine: for each label, total weight = Σ (group share × group item weight). Do the same for `fees.weightedTer` and `risk.weightedSrri`, weighting by group share × group coverage and dividing by the combined coverage.
4. Say that the figures were combined from several calls.

## `fund_overlap`

```
{
  funds: [{ isin, name, holdings, holdingsBasis, weighted }],
  pairs: [{ a, b, overlap, sharedCount, shared: [{ label, a, b }], sharedNames, disclosed: { a, b }, comparable }],
  excluded: [isin],
  namesOnly: [isin],
  note,
  rules,
  pending: [isin],
  notFound: [isin]
}
```

| Field | Meaning |
|---|---|
| `funds[].holdings` | Number of disclosed holdings used |
| `funds[].holdingsBasis` | `portfolio`, `substituteBasket` (swap-based ETF's collateral) or `null` (no holdings disclosed) |
| `funds[].weighted` | Whether the fund's holdings carry weights |
| `pairs[].overlap` | Σ min(weight in a, weight in b) over shared holdings, percent; `null` when either fund publishes names only, or when the pair is not comparable (a swap-based pair has 0, still not a measurement) |
| `pairs[].sharedCount`, `sharedNames` | How many holdings both disclose, and their names (up to 25) |
| `pairs[].shared` | Shared holdings with each fund's weight, largest common weight first; empty when `overlap` is null |
| `pairs[].disclosed` | Sum of each fund's disclosed holding weights (a top-10 list may sum to 25): the overlap cannot exceed the smaller one |
| `pairs[].comparable` | false when either fund is swap-based (its basket is not compared) or has no weighted holdings: still loading (`pending`), not covered (`notFound`) or no holdings disclosed (`holdingsBasis: null`, `weighted: false`). Not a measurement; `overlap` is null, or 0 for a swap-based pair |
| `excluded`, `namesOnly` | ISINs in each situation |
| `pending`, `notFound` | ISINs still loading, and ISINs FundFacts has no data for |
| `errored` | ISINs whose documents could not be read this time (retry in a few minutes); their pairs are not measured |

### Reading overlap

- Two MSCI World trackers typically overlap very strongly; a world fund and an S&P 500 fund overlap by roughly the US share of the world fund. Use such context only to explain the number the tool returns, never instead of it.
- With top-10 disclosures (`disclosed` around 20–40), say the figure covers the disclosed positions only.
- Pairs with `comparable: false`: say the overlap was not measured and why (one fund is swap-based, still loading, not covered, or discloses no holdings). For a swap-based fund, compare the sector and country splits instead; for a pending fund, offer to run the overlap again in a minute.
- A 0 is a real "no shared holdings" only when `comparable` is true and both funds have `weighted: true`. If an older server version returns 0 for a pair that involves a `pending` or `notFound` ISIN, or a fund with `holdingsBasis: null`, treat it as not measured.
