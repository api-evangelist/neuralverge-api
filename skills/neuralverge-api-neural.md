---
name: neural
description: Use when building AI-powered research, extraction, and data enrichment workflows. Reach for this skill when you need to run multi-step research tasks, extract structured data from webpages, enrich contact/company information, or integrate reusable AI agents into products. Common tasks: company intelligence, lead enrichment, data validation, structured web scraping, and operational automation.
metadata:
    mintlify-proj: neural
    version: "1.0"
---

# NeuralVerge Skill Reference

## Product summary

NeuralVerge is an API-first platform for AI research, extraction, reusable agents, and premium data source execution. It combines multi-step research workflows, schema-driven extraction, reusable agent configurations, web search, and targeted data source endpoints (LinkedIn, email, phone, Crunchbase, Amazon, eBay, Walmart, Best Buy, AliExpress) into one system.

**Key endpoints:**
- `POST /functions/v1/run-research` — multi-step research workflow
- `POST /functions/v1/run-extract` — structured extraction from URLs
- `POST /functions/v1/run-agent` — execute saved agent configuration
- `POST /functions/v1/run-search` — fast synchronous web search
- `GET /functions/v1/get-session-status` — poll async job status
- `POST /functions/v1/run-[data-source]` — targeted lookups (email, phone, LinkedIn, etc.)

**Authentication:** Bearer token in `Authorization: Bearer YOUR_ACCESS_TOKEN` header.

**Primary docs:** https://docs.neuralverge.ai

## When to use

Reach for NeuralVerge when:

- **Research task requires multi-step exploration** — searching multiple sources, analyzing findings, synthesizing a structured answer (use AI Research)
- **Extracting structured data from a known URL** — converting webpage content into JSON schema (use AI Extract)
- **Running the same workflow repeatedly** — save configuration once, execute with different instructions (use AI Agents)
- **Need fast ranked search results** — synchronous web search without synthesis (use Search)
- **Looking up specific contact or company data** — email enrichment, LinkedIn profiles, phone validation, Crunchbase (use Data Sources)
- **Building product features** — embedding research, enrichment, or extraction into your application
- **Automating operational workflows** — lead qualification, company risk review, contact verification

Do not use NeuralVerge for: simple text generation, chat interfaces, or tasks that don't require external data retrieval or structured extraction.

## Quick reference

### Product areas and when to use each

| Product Area | Use When | Async? | Example |
|---|---|---|---|
| **AI Research** | Task needs exploration, multi-source synthesis, analyst-style summary | Yes | Investigate company risks, compile market overview |
| **AI Extract** | You know the URL, need structured schema-based output | No | Parse company website, extract product data |
| **AI Agents** | Same workflow runs repeatedly with different instructions | Yes | Lead qualification agent, risk review agent |
| **Search** | Need ranked web results fast, no synthesis needed | No | Pull search results for display, fact-check |
| **Data Sources** | Lookup target is known, need provider-specific enrichment | Varies | Email enrichment, LinkedIn profile, phone validation |

### Common API parameters

**AI Research / AI Extract / AI Agents:**
- `instructions` (string, required) — task description or extraction goal
- `settings.country_code` (string) — localization hint (e.g., "us", "gb")
- `settings.search_enabled` (boolean) — enable web search in workflow
- `settings.deepsearch_model` (string) — model selection for deep analysis
- `settings.finalizer_model` (string) — model for final synthesis
- `settings.extract_schema_json` (string) — JSON schema for structured output

**Search:**
- `query` (string, required) — search query
- `settings.country` (string) — bias results toward country
- `settings.language` (string) — bias results toward language
- `settings.max_results` (number) — cap results returned

**Data Sources (examples):**
- Email enrichment: `email` (required)
- Phone enrichment: `phone` (required)
- LinkedIn profile: `url` (required)
- Crunchbase: `url` (required)

### Rate limits (per organization)

| Limit | Default | Notes |
|---|---|---|
| Concurrent requests | 50 | Across all endpoints |
| Requests per second | 10 | Across all endpoints |
| Per-endpoint RPS | 20 | Default per endpoint |
| Email/phone enrichment | 10 RPS | Specific endpoint limits |

Requests queue automatically; only fail with `429` if waiting >60 seconds. Use exponential backoff on retry.

### Response codes

| Code | Meaning | Retry? |
|---|---|---|
| 200 | Success | N/A |
| 400 | Bad request (invalid params, format) | No |
| 401 | Auth failed (missing/invalid token) | No |
| 402 | Payment required (usage limit exceeded) | No |
| 404 | Resource not found (invalid session_id, agent) | No |
| 429 | Queue timeout (rate limit) | Yes, with backoff |
| 500 | Internal error (upstream/service failure) | Yes, with backoff |

## Decision guidance

### When to use AI Research vs Search

| Scenario | Use AI Research | Use Search |
|---|---|---|
| Need ranked list of links only | ❌ | ✅ |
| Need synthesis across sources | ✅ | ❌ |
| Need structured final answer | ✅ | ❌ |
| Latency/cost critical | ❌ | ✅ |
| Task is exploratory | ✅ | ❌ |

### When to use AI Extract vs AI Research

| Scenario | Use AI Extract | Use AI Research |
|---|---|---|
| Know exact URL to inspect | ✅ | ❌ |
| Need to search for sources first | ❌ | ✅ |
| Output is predefined schema | ✅ | ❌ |
| Need free-form summary | ❌ | ✅ |
| Single page extraction | ✅ | ❌ |

### When to use Data Sources vs AI Research

| Scenario | Use Data Sources | Use AI Research |
|---|---|---|
| Lookup target already known | ✅ | ❌ |
| Need exploration/discovery | ❌ | ✅ |
| Provider-specific enrichment | ✅ | ❌ |
| Need cross-source synthesis | ❌ | ✅ |
| Fast, narrow result needed | ✅ | ❌ |

### When to use AI Agents vs one-off calls

| Scenario | Use AI Agents | Use one-off call |
|---|---|---|
| Workflow runs once | ❌ | ✅ |
| Same task, many entities | ✅ | ❌ |
| Multiple teams need same logic | ✅ | ❌ |
| Configuration drifts easily | ✅ | ❌ |
| Embedded in product | ✅ | ❌ |

## Workflow

### Typical async research task (AI Research, AI Agents, or async Data Sources)

1. **Prepare the request** — write clear `instructions`, set `settings` (country, model, search enabled, schema if needed)
2. **Start the run** — POST to `/run-research`, `/run-agent`, or relevant endpoint; save the returned `session_id`
3. **Poll for completion** — GET `/get-session-status?session_id=...` every 2–5 seconds
4. **Check status** — stop polling when `status` is `complete` or `failed`
5. **Parse results** — extract `human` (summary) and `machine` (structured data) from response
6. **Handle errors** — if `failed`, check error details; if `429`, back off and retry

### Typical synchronous task (Search, AI Extract, or sync Data Sources)

1. **Prepare the request** — write `query` or `instructions`, set `settings`
2. **Call the endpoint** — POST to `/run-search`, `/run-extract`, or data source endpoint
3. **Parse response immediately** — no polling needed
4. **Handle errors** — check status code; retry on `429` or `500` with backoff

### Integrating into a product

1. **Authenticate** — store API token securely in backend environment
2. **Choose product area** — map user request to AI Research, Extract, Agents, Search, or Data Sources
3. **Build request** — construct JSON with required fields and settings
4. **Handle async** — for async endpoints, persist `session_id` in database, poll in background job
5. **Render results** — use `human` for UI display, `machine` for downstream automation
6. **Track usage** — monitor `total_points` for billing and quota management

## Common gotchas

- **Missing Authorization header** — requests fail with `401` if token is missing or malformed. Always include `Authorization: Bearer YOUR_ACCESS_TOKEN`.
- **Polling too fast** — don't poll every millisecond; use 2–5 second intervals to avoid rate limits and wasted requests.
- **Ignoring 429 responses** — don't retry immediately; use exponential backoff. The request is queued and will run when capacity frees up.
- **Confusing async and sync endpoints** — AI Research, AI Agents, and some Data Sources are async (return `session_id`); Search and AI Extract are sync (return result immediately). Check the endpoint docs.
- **Invalid session_id** — if you lose the `session_id`, you cannot retrieve the result. Persist it in your database before polling.
- **Forgetting required fields** — `instructions` is required for research/extract/agents; `query` for search; specific fields for data sources (email, phone, URL, etc.). Missing fields return `400 Bad Request`.
- **Not setting `settings` object** — even if empty, `settings: {}` is required for most endpoints. Omitting it causes `400`.
- **Exceeding 402 Payment Required** — if you hit usage limits, requests fail with `402`. Check your plan and quota before scaling.
- **Assuming all data sources are async** — most are sync (return immediately); only AI Research, AI Agents, and some complex data source operations return `session_id`.
- **Not handling upstream failures** — `500` errors can happen from provider issues. Implement retry logic with exponential backoff.
- **Hardcoding agent IDs** — agent IDs are UUIDs; if an agent is deleted or moved, requests fail with `404`. Store agent IDs in configuration, not code.

## Verification checklist

Before submitting work with NeuralVerge:

- [ ] **Authentication** — token is valid and stored securely; `Authorization` header is present in all requests
- [ ] **Correct endpoint** — mapped user request to the right product area (Research vs Extract vs Agents vs Search vs Data Sources)
- [ ] **Required fields** — all required parameters are present (`instructions`, `query`, `email`, `url`, etc.)
- [ ] **Settings object** — `settings: {}` is included even if empty
- [ ] **Async handling** — for async endpoints, `session_id` is persisted and polling is implemented with 2–5 second intervals
- [ ] **Error handling** — code handles `400`, `401`, `402`, `404`, `429`, and `500` responses appropriately
- [ ] **Rate limit strategy** — requests are spread out; exponential backoff is used for `429` and `500`
- [ ] **Response parsing** — code extracts both `human` and `machine` outputs when available
- [ ] **Usage tracking** — `total_points` is logged for quota and billing monitoring
- [ ] **Timeout handling** — async polling has a reasonable overall timeout (e.g., 5 minutes) to avoid infinite loops
- [ ] **Test with real data** — verified with actual company names, emails, URLs, or queries before production

## Resources

**Comprehensive page listing:** https://docs.neuralverge.ai/llms.txt

**Critical documentation pages:**
- [Introduction & Product Overview](https://docs.neuralverge.ai) — understand which product area to use
- [Authentication](https://docs.neuralverge.ai/authentication) — how to obtain and use API tokens
- [Errors & Rate Limits](https://docs.neuralverge.ai/errors-and-rate-limits) — error codes, polling guidance, rate limit handling

---

> For additional documentation and navigation, see: https://docs.neuralverge.ai/llms.txt