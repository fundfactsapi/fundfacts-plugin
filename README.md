# FundFacts plugin

Fund, ETF and SCPI data inside Claude. Ask about any fund or ETF by ISIN or name and Claude answers from the documents its fund house publishes, read by [FundFacts](https://fundfactsapi.com): ongoing charges and KID costs, the SRRI risk indicator, SFDR article, physical or synthetic replication, top holdings, sector and country exposure, calendar and annualised returns, volatility, Sharpe ratio and drawdown, always with the date of the figures. Give it a portfolio and it shows what you really hold (blended fees, weighted risk, combined exposure) and how much your funds overlap. French SCPIs are covered too: prix de part, taux de distribution, TOF, parts en attente de retrait and the latest bulletin.

The data is informational. Claude describes and compares funds; it does not tell you what to buy or sell.

## What is inside

- `.mcp.json`: the FundFacts MCP server, `https://fundfactsapi.com/api/mcp` (Streamable HTTP, seven read-only tools: `get_fund`, `search_funds`, `compare_funds`, `analyze_portfolio`, `fund_overlap`, `get_scpi`, `search_scpi`).
- `skills/fund-research`: when to call which tool and how to read each field (fees, risk indicator, synthetic ETFs' substitute baskets, holdings published without weights, funds still loading).
- `skills/portfolio-analysis`: look-through and overlap, coverage figures and their limits.
- `skills/scpi-research`: SCPIs, with the French vocabulary, answered in your language.
- `skills/fundfacts-api`: for developers writing code against the REST API, the JavaScript and Python SDKs and the React widgets.

## Install

You need a FundFacts account (the Free plan works, no card): https://fundfactsapi.com/signup. Tool calls count toward your plan's monthly requests like API calls; searches are free.

**Claude Code**

```
/plugin marketplace add fundfactsapi/fundfacts-plugin
/plugin install fundfacts@fundfacts
```

Then run `/mcp`, pick `plugin:fundfacts:fundfacts` and sign in: a browser window opens on fundfactsapi.com, you approve the connection, and Claude Code stores the token. From a shell: `claude plugin marketplace add fundfactsapi/fundfacts-plugin` and `claude plugin install fundfacts@fundfacts`.

To use an API key instead of signing in (for CI or scripts), skip the plugin's server and add it yourself with the key from your dashboard:

```bash
claude mcp add --transport http fundfacts https://fundfactsapi.com/api/mcp --header "Authorization: Bearer $FUNDFACTS_API_KEY"
```

**Claude web, desktop and Cowork**

Install the plugin from the Claude directory once it is listed, or add the server as a custom connector (one on the Free plan; on Team and Enterprise an Owner adds it under Organization settings → Connectors): Customize → Connectors → Add custom connector → paste `https://fundfactsapi.com/api/mcp` → Connect, then sign in and approve. Answers come with a FundFacts card for funds, comparisons, portfolios and SCPIs.

**ChatGPT**

Settings → Security and login → turn on Developer mode (Plus, Pro, Business, Enterprise and Edu, on the web). Then Plugins → + → paste `https://fundfactsapi.com/api/mcp`, choose OAuth, Create, and sign in. The skills in this repository are also packaged for the OpenAI plugin directory.

## Your data

FundFacts receives what the assistant sends its tools (ISINs, fund or SCPI names, portfolio weights) and the FundFacts account that approved the connection. It stores hashed tokens and a request log, as for API calls; it never receives your conversation. You can see and disconnect every connected app in your dashboard (https://fundfactsapi.com/dashboard); its tokens stop working at once. Privacy policy: https://fundfactsapi.com/privacy. Terms: https://fundfactsapi.com/terms. Support: hello@fundfactsapi.com.

## Licence

MIT (see LICENSE). The FundFacts service and its data are covered by their own terms.
