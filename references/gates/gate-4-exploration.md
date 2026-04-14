# Gate 4: Deep Exploration (P17–P22)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file after completing Gate 3 (content item inspection).

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from Gates 1–3

Before using this file, you should already have:
- From Gate 1 (`references/gates/gate-1-baseline.md`): preconnect hints (P6a), robots.txt rules (P8), cookie origin tracking (P7)
- From Gate 2 (`references/gates/gate-2-pagination.md`): pagination API endpoint (P11), pagination replay results (P13)
- From Gate 3 (`references/gates/gate-3-inspection.md`): at least 3 items inspected (P14–P16), extraction maps per item (P15)

If any prerequisite is missing, return to the appropriate gate file before proceeding.

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 4 | P17 → P22 | Deep exploration — APIs, tokens, bundles |

**Write gate at P22.** All pending observations must be logged, D2:State updated, D1 Phase Summary written, before proceeding to Gate 5.

---

## Phase 4: Deep Exploration (~10 cycles, conditional)

> **Phase gate reminder:** Before starting Phase 4, verify you have: completed P14-P16 for at least 3 items, identified the pagination API (P11-P12), and tested replay (P13). Phase 4 steps are conditional — only run the ones triggered by your Phase 1-3 observations. Don't burn cycles on steps that aren't relevant.

These steps are triggered by specific observations from earlier phases. They dig deeper into infrastructure, authentication, and advanced patterns. Only execute the ones that are relevant to the site you're investigating.

---

### [P17] Subdomain/endpoint probing

**Triggered by:** preconnect hints (P6a, from gate-1), CSP `connect-src`, CDP-captured API subdomains, or agent judgment.

**Before probing any path, check robots.txt** (from P8). If the path is disallowed:

- Log SYSTEM `PROBE_SKIPPED_ROBOTS_TXT`.
- Skip the path, UNLESS CDP already captured a response for that path (i.e., the page itself accessed it).

**Probe these paths on each discovered domain:**

`GET /`, `GET /v1/`, `GET /v2/`, `GET /api/`, `GET /graphql`

**Log every response** — even 404s are informative (they confirm the path doesn't exist).

**Why this matters:** Subdomains and API endpoints discovered through headers, CSP, or network traffic are often undocumented but functional. A single `/api/` endpoint can expose the entire site's data. Probing is cheap and can yield high-value discoveries.

---

### [P18] Token/auth lifecycle tracing

**Triggered by:** crumb tokens, CSRF tokens, session tokens, Authorization headers detected in CDP captures, or agent judgment.

Understanding the token lifecycle is essential for replay — if you don't know where tokens come from, when they expire, or how they're refreshed, you can't construct valid authenticated requests.

**Trace:**

- **Where is the token obtained?** (Set-Cookie header? API response? Embedded in page? LocalStorage?)
- **What endpoints require it?** (Check which requests include the token.)
- **Does it expire?** (Check for expiry timestamps or observe token changes.)
- **How is it refreshed?** (Is there a refresh endpoint? Does navigation refresh it automatically?)

**TOKEN ROTATION TEST:** Make 2 sequential requests to the same endpoint, compare tokens in the responses. If the token changes between requests, the site uses rotating tokens — each request invalidates the previous token. Log EDGE_CASE_TEST `TOKEN_ROTATION_TEST`.

**CMS API token detection:** Check RSC payloads (P5a, from gate-1) for embedded CMS tokens or project IDs. Some Next.js sites embed Sanity/Contentful tokens in the RSC streaming data, which gives you authenticated API access.

**Why this matters:** Token rotation means you can't simply replay a captured token — it's already been consumed. You need to understand the full lifecycle to construct valid requests. Embedded CMS tokens are a goldmine — they may give you direct API access with no additional authentication.

---

### [P19] JavaScript bundle analysis

**Triggered by (any one of):** descriptive filenames (e.g., `vendor-analytics.js`), >10 bundles loaded, framework detected.

JavaScript bundles often contain hardcoded API endpoints, GraphQL operations, and configuration that isn't visible from network traffic alone. This is especially valuable for SPAs where all API calls are constructed in JavaScript.

**Selection priority:**

1. **Descriptive filenames** — most likely to contain relevant logic (e.g., `search-api.js`, `product-list.js`).
2. **Webpack/chunk/main patterns** — likely to contain routing or data-fetching logic.
3. **Largest bundles** — more code, more likely to contain endpoints.
4. **Random** — when nothing else distinguishes bundles.

**Download up to 500KB each.**

**Regex scan for:**

- `fetch()` URLs — direct API endpoint discovery.
- API endpoint patterns — `/api/`, `/v1/`, `/graphql`.
- GraphQL operations — `query`, `mutation`, operation names.
- WebSocket URLs — `wss://`, `ws://`.
- CSS selectors — reveals DOM structure the JavaScript interacts with.

**Why this matters:** Bundle analysis is the "look inside the source code" step. It can reveal endpoints that are never called during normal browsing (e.g., admin APIs, internal tools, debugging endpoints). It can also reveal the data model by showing what fields the JavaScript expects from API responses.

---

### [P20] WebSocket inspection

**Triggered by:** CDP captures `wss://` connections.

WebSockets carry real-time data that doesn't appear in normal HTTP request/response flows. This data can include live updates, streaming content, or real-time event notifications.

**Log:**

- Connection URL (may include auth tokens as query parameters).
- First 20 frames — reveals the message format and protocol.
- Message format/protocol — JSON? Binary? Protobuf? Custom?

**Why this matters:** WebSocket data is invisible to standard HTTP replay. If the site uses WebSockets for content delivery, you need to understand the protocol to extract that content. The connection URL may also reveal authentication mechanisms.

---

### [P21] GraphQL inspection

**Triggered by:** `/graphql` endpoint or `.gql` file found during probing or bundle analysis.

GraphQL APIs expose a single endpoint with a rich query language. Understanding the available operations and their schemas gives you comprehensive data access — you can query any combination of fields the schema allows.

**Document:**

- All operations (queries and mutations) discovered from CDP captures or bundle analysis.
- Variable schemas — what parameters each operation accepts.
- Test with different variables to explore the response space.

**Why this matters:** A GraphQL endpoint is essentially an open API to the site's entire data model. If introspection is enabled, you can discover the full schema. Even without introspection, capturing the operations the frontend uses gives you a complete set of data access patterns.

---

### [P21b] Multi-Query Validation

When you discover a search or filter API endpoint, test it with **at least 3 different queries** before generalizing about its behavior. A single query may produce a result that doesn't represent the endpoint's full behavior.

**Procedure:**

1. Take the discovered search/filter API endpoint from P13c, P17, or P21.
2. Execute 3 queries with different parameters:
   - **Broad query:** minimal or no filters — should return many results.
   - **Specific query:** narrow filter — should return few or zero results.
   - **Edge query:** boundary value, special character, or empty string.
3. Compare response schemas across all 3 queries:
   - Same schema? → The API is consistent; one query is representative.
   - Different schema? → The API returns different structures for different queries; log each variant.
   - Errors on edge query? → The API has input validation; log the error format.

**Log:** REQUEST entries for each query. If schemas differ, also log a SYSTEM entry noting the inconsistency.

**Why this matters:** In the Yahoo Finance investigation, the agent tested the search API with one query ("bitcoin"), got results, and generalized that the API works for all queries. A second query ("xyznotreal") would have revealed that the API returns a different schema for zero-result queries. Without multi-query validation, you risk building an extraction schema that only works for one type of response.

**Budget:** 2 additional decision cycles (the broad query may already be done; the specific and edge queries are new).

---

### [P22] Cross-item structural comparison

**Triggered by:** After completing P14-P16 for N items (minimum 3).

Comparing the DOM structure across multiple items reveals which selectors are stable (work for all items) and which are brittle (work for some items but not others). This is critical for building a reliable extraction schema.

**Procedure:**

- Execute JS to diff DOM structures across the items you've visited.
- Identify: which selectors appear in ALL items? Which appear in only SOME?

**Log:**

- **Stable selectors** — reliable for extraction, work across all items.
- **Brittle selectors** — only work for some items, need fallbacks.
- **Content structure differences** — what varies between item types.

**Why this matters:** An extraction schema based on a single item's structure will fail when it encounters items with different structures. Cross-item comparison catches these failures before they happen, ensuring your schema handles the full variety of content.

**Cross-method stability validation:**

Beyond comparing selectors across items, compare EXTRACTION METHODS across the same item. This is the critical test that determines which fields are safe to extract and which will break.

**Procedure:**

For each field identified in P15's extraction map, validate across >=3 items:

1. **Structured data availability:** Does ld+json contain this field for ALL items? Or only some?
   - All items → `structured_data` path is universal. Flag as `[stable: universal]`
   - Some items → partial coverage. Flag as `[stable: partial, coverage X/3]`
   - No items → Must rely on DOM.

2. **Semantic HTML consistency:** Does the same semantic element (h1, time[datetime], etc.) appear for this field across ALL items?
   - All items → `semantic_html` path is universal
   - Some items use different elements → Log the variants

3. **Data attribute consistency:** Does the same `data-testid` appear for this field across ALL items?
   - All items → `data_attribute` path is universal
   - Different data-testids → Log the variants

4. **Class-based selector survival:** Do the same hashed classes appear for this field across ALL items?
   - All items → stable across items (but may still break on deploy)
   - Different classes → brittle even across items

**Final extraction stability matrix:**

```
| Field | structured_data | semantic_html | data_attribute | class_hashed |
|-------|----------------|---------------|----------------|-------------|
| title | universal       | h1            | partial        | .yf-1a2b    |
| author| universal       | partial       | none            | .yf-3c4d    |
| date  | universal       | time[dt]      | none            | .yf-5e6f    |
| body  | none            | article       | none            | .yf-7g8h    |
```

**Fields with NO column showing "universal" are HIGH RISK.** Flag them as `stability_risk: HIGH` with an explanation.

**Log:** DOM_SNAPSHOT with context `stability_matrix` containing the complete matrix. This matrix is the single most actionable artifact for scraper construction — it tells the analyser exactly which fields are safe and which will break.

---

## Gate 4 Output

Before proceeding to Gate 5, verify:

☐ Subdomain/endpoint probing completed (P17)
☐ Token/auth lifecycle traced if tokens detected (P18)
☐ JavaScript bundle analysis completed if triggered (P19)
☐ WebSocket inspected if detected (P20)
☐ GraphQL inspected if detected (P21)
☐ Multi-query validation completed if search API found (P21b)
☐ Cross-item structural comparison completed (P22)
☐ Extraction stability matrix produced (P22)
☐ D2:State updated
☐ D1: Deep Exploration Phase Summary written
☐ Re-read `references/gates/gate-5-replay.md` BEFORE writing first entry of Gate 5
