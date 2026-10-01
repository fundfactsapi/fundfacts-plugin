# Evals

Cases for `claude plugin eval` (Claude Code 2.1.269+). Every FundFacts tool is answered by the mocks in `mocks/fundfacts/` (suite-wide) or in a case's own `mocks/fundfacts/`, so no run reaches the real server or spends FundFacts requests. The fund and SCPI figures in the mocks are fixtures for the graders, not reference data. `get_fund` answers per ISIN from `mocks/fundfacts/fixtures/<ISIN>.json`, consistent with the `compare_funds`, `analyze_portfolio` and `fund_overlap` mocks; its `expect:` lists the ISINs that have a fixture, so add one there when a case uses a new ISIN.

```bash
claude plugin eval . --runs 1 --ablation none   # one cheap pass
claude plugin eval .                             # 3 runs with and without the plugin
```

Every run is a model call billed to the account running it. `results/` is written by each run and ignored by git.
