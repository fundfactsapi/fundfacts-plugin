---
type: llm
---
PASS if the code uses @fundfactsapi/sdk (getFund) or calls https://fundfactsapi.com/api/v1/funds/{isin} with an Authorization Bearer header, reads the key from the FUNDFACTS_API_KEY environment variable, and returns data.headlineMetrics.ter; a timeout, if one is set, is at least 300 seconds (the SDK default counts).
FAIL if the key is hard-coded, the endpoint or field path is invented, or a timeout under 300 seconds is set.
