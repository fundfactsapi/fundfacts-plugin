# SCPI record and French vocabulary

`get_scpi` returns `{ slug, isin, name, manager, cached, generatedAt, data }`. Amounts in euros (`…Eur`), rates in percent (`…Pct`), years as `"YYYY"` strings in the yearly series. `null` or `[]` means the documents read do not publish it.

## `data.keyFacts`

| Field | French term | Meaning |
|---|---|---|
| `name`, `managementCompany` | nom, société de gestion | The SCPI and the company that manages it |
| `isin` | code ISIN | The shares' ISIN when the SCPI has one; some managers use an internal code instead |
| `category` | catégorie / typologie | bureaux (offices), commerces (retail), santé (healthcare), logistique, diversifiée, résidentiel, européenne, autre |
| `capitalType` | capital fixe / variable | Variable: the manager issues and redeems parts continuously. Fixed: parts change hands on a secondary market (marché des parts) organised by the manager |
| `creationYear` | date de création | Year the SCPI was created |
| `capitalisationEur` | capitalisation | Subscription price × parts in issue |
| `sharePriceEur` | prix de souscription / prix de part | Price a buyer pays per part, subscription fee included |
| `withdrawalPriceEur` | prix de retrait / valeur de retrait | What a seller receives per part in a variable-capital SCPI (the subscription price less the subscription fee) |
| `subscriptionFeesPct` | commission (frais) de souscription | Percent of the subscription price, TTI or TTC as printed |
| `managementFeesPct` | commission (frais) de gestion | Percent of the rental income collected (HT or TTC as printed), not of the amount invested |
| `minimumSubscriptionShares`, `minimumSubscriptionEur` | minimum de souscription | Minimum first purchase, in parts or euros |
| `delayOfJouissance` | délai de jouissance | Time between buying parts and the first income they earn, as printed ("1er jour du 6e mois") |
| `riskIndicator` | indicateur de risque (DIC) | 1 (lowest) to 7 |
| `sfdrArticle` | classification SFDR | 6, 8 or 9 as declared |
| `labelIsr` | label ISR | Whether the SCPI holds the French SRI label, when stated |

## `data.performance`

| Field | French term | Meaning |
|---|---|---|
| `distributionRateByYear[]` | taux de distribution | Gross distribution of the year as a percent of the part price, as the manager publishes it, per calendar year, oldest first (up to 5 years) |
| `distributionPerShareByYear[]` | dividende brut par part | Gross distribution per part in euros, per calendar year |
| `tri.y5`, `tri.y10`, `tri.y15` | TRI (taux de rentabilité interne) | Annualised return over 5, 10, 15 years, including price change and distributions |
| `reconstitutionValueEur` | valeur de reconstitution | What it would cost to rebuild the portfolio (appraised value plus fees and costs), per part |
| `realisationValueEur` | valeur de réalisation | Appraised value of the buildings plus other net assets, per part |
| `sharePriceHistory[]` | historique du prix de part | `{ date, priceEur }` at each price change |
| `lastQuarterDistributionPerShareEur` | acompte trimestriel | Latest quarterly distribution per part |

Computations you may offer, labelled as yours: premium or discount of the price to the reconstitution value (price ÷ reconstitution − 1; positive: surcote, negative: décote); the change in price over the history; the withdrawal queue as a share of parts in issue.

## `data.portfolio`

| Field | French term | Meaning |
|---|---|---|
| `numberOfProperties` | nombre d'actifs / d'immeubles | |
| `surfaceSqm` | surface (m²) | |
| `numberOfTenants` | nombre de locataires (baux) | |
| `financialOccupancyRatePct` | TOF, taux d'occupation financier | Rents invoiced as a share of the rents if everything were let |
| `physicalOccupancyRatePct` | TOP, taux d'occupation physique | Surface let as a share of total surface |
| `walbYears` | WALB, durée résiduelle des baux jusqu'à la prochaine échéance | Years until tenants can first leave |
| `waltYears` | WALT, durée ferme résiduelle | Years until leases end |
| `sectorSplit[]`, `geographicSplit[]` | répartition sectorielle / géographique | `{ label, weight }` in percent, by value or rent as the bulletin states |
| `topTenants[]` | principaux locataires | `{ name, weight }`; weight may be null |

## `data.liquidity`

| Field | French term | Meaning |
|---|---|---|
| `sharesAwaitingWithdrawal` | parts en attente de retrait | Parts whose holders asked to sell and are still waiting. A rising queue means sellers are not being matched with new buyers |
| `netInflowLastQuarterEur` | collecte nette | New subscriptions minus withdrawals over the latest quarter |
| `sharesOutstanding` | nombre de parts | Parts in issue |
| `numberOfAssociates` | nombre d'associés | Number of shareholders |

## `data.documents` and dates

- `productUrl`, `bulletinUrl` (bulletin trimestriel d'information), `annualReportUrl` (rapport annuel), `kidUrl` (DIC), `noteInformationUrl` (note d'information), `statutsUrl` (statuts), `factsheetUrl` (plaquette).
- `dataAsOf`: bulletin date (YYYY-MM-DD). `bulletinPeriod`: its period as printed ("T2 2026").

## Standard facts you may state

These are general features of SCPIs that every DIC states; phrase them neutrally:

- Neither the capital nor the income is guaranteed; distributions can fall and the part price can be cut.
- Liquidity is not guaranteed: selling depends on buyers (or on the withdrawal queue in a variable-capital SCPI).
- The DIC states a recommended holding period; quote the one from the SCPI's own documents if the user asks, and link `documents.kidUrl`.
