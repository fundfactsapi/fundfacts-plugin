---
name: scpi-research
description: Looks up French SCPIs (sociétés civiles de placement immobilier, "pierre-papier" real-estate funds) with the FundFacts MCP server's search_scpi and get_scpi tools and explains their figures. Use when the user names an SCPI or its management company (Corum, Perial, Atland Voisin, Paref, Iroko, La Française, AEW, Praemia, Sofidy…) or asks about a prix de souscription or de retrait, taux de distribution, TRI, frais de souscription or de gestion, TOF (taux d'occupation financier), valeur de reconstitution, parts en attente de retrait, collecte, patrimoine or the latest bulletin trimestriel.
license: MIT
metadata:
  author: FundFacts API
  version: "1.0.0"
---

# SCPI research with FundFacts

An SCPI is an unlisted French real-estate company whose shares (parts) are bought from and sold back through its management company (société de gestion). FundFacts reads the management company's own documents (the SCPI's page, the bulletin trimestriel, the DIC, the annual report) and returns one structured record. Explicit instructions from the user take precedence over this skill.

| Tool | Use it for | Cost |
|---|---|---|
| `search_scpi` | Find an SCPI by name, former name, management company or ISIN prefix; returns its `slug` | Free |
| `get_scpi` | The full record for one SCPI, by slug (preferred), ISIN or exact name | One request |

## Workflow

1. **Identify the SCPI.** Call `search_scpi` with the name as the user wrote it ("epargne pierre", "corum origin", "pfo2"). Former names are searchable (Interpierre France is now PAREF Hexa, PFO2 is now Perial O2). Several matches: pick the exact one or ask. No match: the SCPI is not in FundFacts' list yet; say so.
2. **Fetch** with `get_scpi` and the `slug` from the search result. Do not show the search result's `url` to the user: it is an API address that needs a key.
3. **Answer** the question from `data`, then the source line: "Source: FundFacts, from {manager}'s documents, bulletin {data.bulletinPeriod}, data as of {data.dataAsOf}."

Errors: "not a known SCPI" → search again with other words; "no published documents could be read" → the management company's site did not yield documents this time, say so; "could not be retrieved right now" → retry once later. Plan or allowance messages: relay neutrally with their link, without pressing for an upgrade. Sign-in errors: the user reconnects FundFacts in the assistant's settings; never ask for keys or passwords in the chat.

## Reading the record

`get_scpi` returns `{ slug, isin, name, manager, cached, generatedAt, data }`. Amounts are in euros, rates in percent, and missing values are `null` or `[]` ("not published", never 0). Field guide and French vocabulary: [references/glossary.md](references/glossary.md).

- **Prices.** `keyFacts.sharePriceEur` is the prix de souscription (what a buyer pays per part, fees included). `keyFacts.withdrawalPriceEur` is the prix de retrait (what a seller receives in a variable-capital SCPI, i.e. the subscription price minus the subscription fee). `performance.sharePriceHistory` lists price changes; a cut in the price is a revaluation downwards.
- **Distribution.** `performance.distributionRateByYear` is the taux de distribution per calendar year (oldest first). `performance.distributionPerShareByYear` is the gross dividend per part in euros. `performance.lastQuarterDistributionPerShareEur` is the latest quarterly payment. Past distributions are not guaranteed to recur, and an SCPI's capital is not guaranteed: say so when you quote them.
- **TRI.** `performance.tri.y5`, `y10`, `y15`: annualised internal rate of return over 5, 10, 15 years (price change plus distributions), when published.
- **Valuations.** `performance.reconstitutionValueEur` (valeur de reconstitution per part) and `realisationValueEur` (valeur de réalisation). You may compute the gap between the subscription price and the reconstitution value (price ÷ reconstitution − 1): a positive gap is a premium (surcote), a negative one a discount (décote). Label it as your calculation.
- **Fees.** `keyFacts.subscriptionFeesPct` (commission de souscription, percent of the price, TTI or TTC as the document prints it) and `keyFacts.managementFeesPct` (commission de gestion, percent of rental income, HT or TTC as printed). Say that management fees are charged on rents, not on the invested amount.
- **Portfolio.** `portfolio.numberOfProperties`, `surfaceSqm`, `numberOfTenants`, `financialOccupancyRatePct` (TOF), `physicalOccupancyRatePct` (TOP), `walbYears` / `waltYears` (remaining lease length), `sectorSplit`, `geographicSplit`, `topTenants`.
- **Liquidity.** `liquidity.sharesAwaitingWithdrawal` (parts en attente de retrait) against `liquidity.sharesOutstanding` shows how many sellers are waiting; you may compute the percentage and label it as your calculation. `netInflowLastQuarterEur` is the latest quarter's net inflow (collecte nette).
- **Identity.** `keyFacts.category` (bureaux, commerces, santé, logistique, diversifiée, résidentiel, européenne, autre), `capitalType` (fixe or variable), `creationYear`, `capitalisationEur`, `minimumSubscriptionShares` / `minimumSubscriptionEur`, `delayOfJouissance` (délai de jouissance as printed), `riskIndicator` (DIC, 1–7), `sfdrArticle`, `labelIsr`.
- **Documents.** `documents.bulletinUrl`, `annualReportUrl`, `kidUrl`, `noteInformationUrl`, `statutsUrl`, `productUrl`, `factsheetUrl`: the management company's own links. Offer them when the user wants the source.
- **Dates.** `data.dataAsOf` is the bulletin date and `data.bulletinPeriod` its period ("T2 2026"). Always cite them: SCPI figures move quarterly.

## Answering

- Answer in the user's language. For French users, use the French terms (taux de distribution, prix de part, TOF). For others, give the English meaning and the French term once in brackets: "distribution rate (taux de distribution) of 5.1% for 2025".
- The data is informational: describe and compare, do not recommend buying or selling an SCPI or say one is a "good investment". When asked, list the facts that bear on a decision (price history, distribution history, occupancy, withdrawal queue, fees, the délai de jouissance) and leave the decision to the user or a licensed adviser.
- Comparing SCPIs: call `get_scpi` for each (one request each), then a table with the same fields and one date line per SCPI. Point out when the bulletins are from different quarters.
- An OPCI, an SCI sold inside life insurance or a listed real-estate fund is not an SCPI. If it has an ISIN, `get_fund` may cover it.
