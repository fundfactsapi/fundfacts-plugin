---
name: fund-research
description: Looks up and explains factual data on UCITS funds, ETFs and money-market funds by ISIN or name with the FundFacts MCP server (search_funds, get_fund, compare_funds). Use when the user gives an ISIN or a fund or ETF name and asks about its fees (TER, ongoing charges, KID costs), SRRI / SRI risk indicator, SFDR article, replication (physical or synthetic), top holdings, sector or country exposure, returns, volatility, Sharpe ratio, drawdown or fund size, or wants two or more funds compared side by side.
license: MIT
metadata:
  author: FundFacts API
  version: "1.0.3"
---

# Fund research with FundFacts

The FundFacts MCP server reads the documents each fund house publishes (KID, factsheet, holdings file, NAV history) and returns one structured factsheet per ISIN. Your job is to pick the right tool, read the payload correctly, and report the figures with their date. Explicit instructions from the user take precedence over this skill.

## Tools

| Tool | Use it for | Requests counted on the user's account |
|---|---|---|
| `search_funds` | Turn a name, issuer or ISIN prefix into ISINs | None |
| `get_fund` | One fund's full factsheet | One request |
| `compare_funds` | Two or more ISINs side by side (a compact row per fund) | One request per ISIN |

Do not call `get_fund` or `compare_funds` speculatively or twice for the same ISIN in a conversation: each call counts on the user's account, so reuse the result you already have.

## Workflow

1. **Get an ISIN.**
   - The user gave an ISIN (12 characters: 2 letters, 9 letters or digits, 1 check digit): use it as is, upper-cased, spaces removed.
   - The user gave a name: call `search_funds` with the distinctive words ("msci world ishares", "fundsmith equity"). Search only covers funds already loaded into FundFacts, so an empty result does not mean the fund does not exist. If the user can give the ISIN (it is on the KID and on every broker page), use it with `get_fund` directly.
   - Several share classes match (currency, accumulating or distributing, hedged): pick the one the user described; otherwise list the candidates with their `currency`, `income` and `shareClass` and ask. Each share class has its own ISIN, TER and returns.
   - A ticker (IWDA, VWCE, CSPX) is not an ISIN. Search by the fund's name. Use an ISIN you know from memory only if the `name` in the result then matches the fund the user meant; say so if it does not.
2. **Fetch.** One fund: `get_fund`. Two or more: `compare_funds` (leave `includeFullData` false unless you need fields beyond the comparison row, such as the holdings or the sector split; it makes the result much larger).
3. **Handle the status** (next section) before reading any figure.
4. **Answer** with the figures the user asked for, each read as described in "Reading the payload", and close with the date line: "Source: FundFacts, from the fund's documents, data as of {data.dataAsOf}." Name the fund and ISIN once.

## Statuses and errors

- **ok**: read `data`.
- **Being loaded for the first time**: FundFacts had never loaded this ISIN and the first load did not finish within the call. The tool already waited several minutes. This is not charged. Tell the user it is loading and ask again once, about a minute later; do not loop. In `compare_funds` the row has `status: "pending"` and empty fields; report the others and say which one is still loading.
- **No published fund documents found** (`not_found`): the ISIN is not a fund FundFacts covers (a share, a bond, a structured product, a closed fund, or an issuer not covered yet). It counts as a request. Say so plainly; do not guess figures, do not present data from another source as FundFacts data, and do not retry.
- **Not a valid ISIN**: a typo or a wrong check digit. Not charged. Ask the user to check it.
- **Documents could not be read right now** (`error`): a temporary failure. Retry at most once, later.
- **Account limit messages** (the account allows one ISIN per call, its monthly requests are used up, or a tool is not included in its plan): relay them in one neutral, informational sentence with the link the message gives (https://fundfactsapi.com/docs/requests). Do not quote prices, suggest changing plan, or point to billing or checkout pages. When the account allows one ISIN per call, `compare_funds` with several ISINs is refused: call `get_fund` for each fund instead (one request each, as with `compare_funds`) and build the comparison yourself.
- **Sign-in or authorization errors**: the user must connect (or reconnect) their FundFacts account in the assistant's connector or plugin settings. Never ask the user to paste an API key or password into the chat.

## Reading the payload

The `get_fund` result is `{ isin, name, status, cached, generatedAt, expiresAt, data }`. All the facts are in `data`. The essentials (full field-by-field guide in [references/fields.md](references/fields.md)):

- **Empty is not zero.** Fields that do not apply or are not disclosed are `""`, `null` or `[]`. Say "not disclosed", never "0" and never an estimate.
- **Formatted strings stay as published**: `headlineMetrics.ter` `"0.20%"`, `keyFacts.aum` `"USD 151.7bn"`, `headlineMetrics.volatility3y` `"11.8%"`. Breakdown weights and returns are numbers in percent.
- **Fees.** `headlineMetrics.ter` is the ongoing charge from the KID (preferred) or the issuer's other documents. `costs.*` are the PRIIPs KID lines as printed (entry, exit, transaction, performance fee, reduction in yield). Transaction costs and entry or exit costs come on top of the ongoing charge; say which figure you quote.
- **Risk indicator.** `riskRating` is the KID's SRI / SRRI on a 1 (lowest) to 7 (highest) scale. `null` means no document states it: it is never estimated. `profile.riskBand` groups it (1–2 low, 3–4 medium, 5–7 high).
- **SFDR.** `sfdrArticle` 6, 8 or 9 is a disclosure category the fund declares (9: sustainable investment objective; 8: promotes environmental or social characteristics; 6: neither). It is not a rating. `null`: not disclosed.
- **Replication and holdings.** `replication` is `"physical"`, `"synthetic"` or `null` (not stated, usual for active funds). When `holdingsBasis` is `"substituteBasket"`, the fund is swap-based and `topHoldings` lists the collateral basket that backs the swap, not what the fund is exposed to: never present those names as the fund's investments. Use `sector`, `geography` and `region` (which describe the index) for exposure, and say why.
- **Names-only holdings.** When every `topHoldings[].weight` is `null`, the fund house publishes its largest positions without weights. List the names, say the weights are not published, and never invent or estimate them. `profile.concentration` is then `null`.
- **Returns.** `annualisedReturns[]` has `label` ("1 Year", "3 Years p.a.", "5 Years p.a.", "10 Years p.a.", "Since Inception"), `fund` and `index`, in percent; multi-year figures are annualised; `null` when the share class is younger than the horizon. `calendarReturns` holds full calendar years only. Returns are the share class's own, in its currency (`keyFacts.currency`). Past performance says nothing certain about the future: say so when you quote returns.
- **Risk statistics.** `headlineMetrics.volatility3y`, `headlineMetrics.sharpe3y` and `metrics.maxDrawdown` cover the last 3 years. The Sharpe ratio is empty for money-market funds.
- **Profile.** `data.profile` (kind, category, riskBand, concentration, regionFocus, regionTilt, sectorTilt, valuation, creditQuality, rateSensitivity) is FundFacts' own classification, computed from the disclosed data with published rules. Present it as "FundFacts classifies it as…", not as the issuer's label.
- **Dates.** `data.dataAsOf` is the date of the figures (latest NAV or factsheet date): always cite it. `generatedAt` is only when FundFacts produced the payload. A `dataAsOf` more than about two months old means the fund house has not published newer figures; mention it.

## compare_funds rows

Each row carries `isin`, `status`, `name`, `category`, `assetClass`, `ter` (as printed) and `terPct` (number), `riskRating`, `aum`, `distribution`, `currency`, `return1y`, `return3yPa`, `return5yPa`, `volatility3y`, `sharpe3y`, `maxDrawdown`, `replication`, `top10Weight`, `dataAsOf` and `page`. `top10Weight` is `null` for a synthetic fund's substitute basket, for names-only holdings, and for a fund with no holdings to measure (still loading, not found, or none disclosed): that is "not measurable", not "low concentration". Present the comparison as a table, one fund per column or row, with a `dataAsOf` line per fund when the dates differ. Compare like with like (same asset class, same currency when you compare returns) and point out when the funds differ in a way that makes a figure incomparable.

## What to say and not say

- The data is informational. Describe, compare and explain; do not recommend buying, selling or holding a fund, and do not rank funds as "best" for the user. When the user asks which to choose, set out the factual differences that matter for such a choice (costs, risk indicator, what they hold, replication, size, track record length) and say that the decision depends on their situation; for personal advice, a licensed adviser.
- Attribute every figure to the fund's documents via FundFacts, with the date. Do not mix in figures from memory without saying so.
- Answer in the user's language. Keep fund names, ISINs and the published strings as they are.
- Some assistants display a FundFacts card with the result. Still write the key figures in the text: not every user sees the card.

## References

- [references/fields.md](references/fields.md): every field of `data`, what it means and how to phrase it.
- [references/examples.md](references/examples.md): worked answers for a plain ETF, a synthetic ETF, a names-only fund, a comparison and a fund that is still loading.
