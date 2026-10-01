# Changelog

## 1.0.3 (2026-10-01)

- OpenAI package: the review walkthrough video (`review.demo_recording_url`) is included. No change to the skills or the server.

## 1.0.2 (2026-10-01)

- `fundfacts-api` skill: no plan names left. Look-through, overlap, `/extract`, `/changes`, `/webhooks`, `/export` and custom factsheet logos are described as "not included in every plan" with the link https://fundfactsapi.com/docs/requests; the `GET /funds/{isin}` example shows a neutral `plan` and a round `quota.limit`; a 403 `plan_required` is relayed neutrally with that link.
- Skills and evals follow the server's neutral tool messages ("not included in this account's plan. Details: …"); search tools are described as not counted.
- README: no plan tiers; searches are "not counted".

## 1.0.1 (2026-10-01)

- Skills: plan and allowance messages are relayed as information only, with the link to https://fundfactsapi.com/docs/requests and no prices, plan changes or billing links; plan tiers, quotas and batch sizes are no longer listed (read them from `GET /me`); "free" labels become "not counted".

## 1.0.0 (2026-09-30)

- First release: the FundFacts MCP server (`https://fundfactsapi.com/api/mcp`, OAuth sign-in) and four skills: `fund-research`, `portfolio-analysis`, `scpi-research` and `fundfacts-api`.
- Evals under `evals/` for `claude plugin eval`, answered by MCP mocks.
