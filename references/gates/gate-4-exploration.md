# Gate 4: Deep Exploration (P17–P22)

```yaml
gate_id: 4
title: Deep Exploration
steps: P17 → P22
phases: [4]
d0_file: g4d0.log
operator_halt: false
next_gate: references/gates/gate-5-replay.md
```

Read this file after completing Gate 3 (content item inspection). This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] Preconnect hints (P6a), robots.txt rules (P8), cookie origin tracking (P7)
- [ ] Pagination API endpoint (P11), pagination replay results (P13)
- [ ] At least 3 items inspected (P14–P16), extraction maps per item (P15)

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries) | `g4d0.log` |
| D2:State updates | `state.log` |
| D1: Deep Exploration Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 4 | P17 → P22 | Deep exploration — APIs, tokens, bundles |

**Write gate at P22.** All pending observations logged, D2:State updated, D1 Phase Summary written, before proceeding to Gate 5.

---

## Phase 4: Deep Exploration (~10 cycles, conditional)

Phase 4 steps are conditional — only run the ones triggered by your Phase 1-3 observations. Don't burn cycles on irrelevant steps.

---

### [P17] Subdomain/endpoint probing

```yaml
step: P17
cycle: true
condition: preconnect hints (P6a), CSP connect-src, CDP-captured API subdomains, or agent judgment
log: { type: REQUEST, context: endpoint_probe }
```

**Before probing any path, check robots.txt** (from P8). If the path is disallowed:

- Log SYSTEM `PROBE_SKIPPED_ROBOTS_TXT`.
- Skip the path, UNLESS CDP already captured a response for that path.

**Probe these paths on each discovered domain:**

`GET /`, `GET /v1/`, `GET /v2/`, `GET /api/`, `GET /graphql`

**Log every response** — even 404s are informative.

**Feeds into:** P19 (bundle analysis targets), P21 (GraphQL inspection).

---

### [P18] Token/auth lifecycle tracing

```yaml
step: P18
cycle: true
condition: crumb tokens, CSRF tokens, session tokens, Authorization headers detected in CDP captures
log: { type: EDGE_CASE_TEST, test_id: TOKEN_LIFECYCLE }
```

**Trace:**

| Question | How |
|----------|-----|
| Where is the token obtained? | Set-Cookie header? API response? Embedded in page? LocalStorage? |
| What endpoints require it? | Check which requests include the token |
| Does it expire? | Check for expiry timestamps or observe token changes |
| How is it refreshed? | Is there a refresh endpoint? Does navigation refresh it automatically? |

**TOKEN ROTATION TEST:** Make 2 sequential requests to the same endpoint, compare tokens. If token changes between requests, site uses rotating tokens — each request invalidates the previous. Log EDGE_CASE_TEST `TOKEN_ROTATION_TEST`.

**CMS API token detection:** Check RSC payloads (P5a) for embedded CMS tokens or project IDs. Some Next.js sites embed Sanity/Contentful tokens in RSC streaming data — direct API access with no additional authentication.

**Feeds into:** P24 (cookie-removal test), P25 (auth-removal test), HTTP_REQUEST_CHAIN.

---

### [P19] JavaScript bundle analysis

```yaml
step: P19
cycle: true
condition: descriptive filenames (e.g., vendor-analytics.js), >10 bundles loaded, or framework detected
log: { type: SYSTEM, event: bundle_analysis }
```

**Selection priority:**

1. **Descriptive filenames** — most likely to contain relevant logic (e.g., `search-api.js`)
2. **Webpack/chunk/main patterns** — likely to contain routing or data-fetching logic
3. **Largest bundles** — more code, more likely to contain endpoints
4. **Random** — when nothing else distinguishes bundles

**Download up to 500KB each.**

**Regex scan for:**

| Pattern | Reveals |
|---------|---------|
| `fetch()` URLs | Direct API endpoint discovery |
| API endpoint patterns (`/api/`, `/v1/`, `/graphql`) | Endpoint URLs |
| GraphQL operations (`query`, `mutation`, operation names) | Data access patterns |
| WebSocket URLs (`wss://`, `ws://`) | Real-time connections |
| CSS selectors | DOM structure the JS interacts with |

**Feeds into:** P17 (additional probing targets), P21 (GraphQL operations).

---

### [P20] WebSocket inspection

```yaml
step: P20
cycle: true
condition: CDP captures wss:// connections
log: { type: SYSTEM, event: websocket_inspection }
```

**Log:**

| What | Detail |
|------|--------|
| Connection URL | May include auth tokens as query parameters |
| First 20 frames | Reveals message format and protocol |
| Message format | JSON? Binary? Protobuf? Custom? |

**Feeds into:** HTTP_REQUEST_CHAIN (WebSocket protocol for content delivery).

---

### [P21] GraphQL inspection

```yaml
step: P21
cycle: true
condition: /graphql endpoint or .gql file found during probing or bundle analysis
log: { type: SYSTEM, event: graphql_inspection }
```

**Document:**

- All operations (queries and mutations) discovered from CDP captures or bundle analysis.
- Variable schemas — what parameters each operation accepts.
- Test with different variables to explore the response space.

**Feeds into:** P21b (multi-query validation), HTTP_REQUEST_CHAIN.

---

### [P21b] Multi-Query Validation

```yaml
step: P21b
cycle: true
condition: search or filter API endpoint discovered (P13c, P17, or P21)
budget: 2
log: { type: REQUEST, context: multi_query_validation }
```

Test a discovered search/filter API with **at least 3 different queries** before generalizing about its behavior.

**Procedure:**

1. Take the discovered search/filter API endpoint from P13c, P17, or P21.
2. Execute 3 queries:

| Query Type | Parameters | Expected |
|------------|-----------|----------|
| Broad | Minimal or no filters | Many results |
| Specific | Narrow filter | Few or zero results |
| Edge | Boundary value, special character, or empty string | Varies |

3. Compare response schemas across all 3 queries:

| Outcome | Action |
|---------|--------|
| Same schema | API is consistent; one query is representative |
| Different schema | API returns different structures per query; log each variant |
| Errors on edge query | API has input validation; log the error format |

**Log:** REQUEST entries for each query. If schemas differ, also log a SYSTEM entry noting the inconsistency.

**Budget:** 2 additional decision cycles (broad query may already be done; specific and edge are new).

**Feeds into:** Extraction schema reliability, HTTP_REQUEST_CHAIN.

---

### [P22] Cross-item structural comparison

```yaml
step: P22
cycle: true
condition: AFTER completing P14-P16 for N items (minimum 3)
log: { type: DOM_SNAPSHOT, context: stability_matrix }
```

Compare DOM structure across multiple items to identify stable vs brittle selectors.

**Cross-method stability validation:**

For each field identified in P15's extraction map, validate across ≥3 items:

1. **Structured data availability:** Does ld+json contain this field for ALL items?

| Coverage | Flag |
|----------|------|
| All items | `[stable: universal]` |
| Some items | `[stable: partial, coverage X/3]` |
| No items | Must rely on DOM |

2. **Semantic HTML consistency:** Same semantic element (h1, time[datetime], etc.) for this field across ALL items?
3. **Data attribute consistency:** Same `data-testid` for this field across ALL items?
4. **Class-based selector survival:** Same hashed classes for this field across ALL items?

**Final extraction stability matrix:**

```
| Field | structured_data | semantic_html | data_attribute | class_hashed |
|-------|----------------|---------------|----------------|-------------|
| title | universal       | h1            | partial        | .yf-1a2b    |
| author| universal       | partial       | none            | .yf-3c4d    |
| date  | universal       | time[dt]      | none            | .yf-5e6f    |
| body  | none            | article       | none            | .yf-7g8h    |
```

**Fields with NO column showing "universal" are HIGH RISK.** Flag as `stability_risk: HIGH` with explanation.

**Log:** DOM_SNAPSHOT with context `stability_matrix` containing the complete matrix. This is the single most actionable artifact for scraper construction.

**Feeds into:** Extraction schema construction, scraper stability assessment.

---

## Gate 4 Output

Before proceeding to Gate 5, verify:

- [ ] Subdomain/endpoint probing completed (P17)
- [ ] Token/auth lifecycle traced if tokens detected (P18)
- [ ] JavaScript bundle analysis completed if triggered (P19)
- [ ] WebSocket inspected if detected (P20)
- [ ] GraphQL inspected if detected (P21)
- [ ] Multi-query validation completed if search API found (P21b)
- [ ] Cross-item structural comparison completed (P22)
- [ ] Extraction stability matrix produced (P22)
- [ ] D2:State updated
- [ ] D1: Deep Exploration Phase Summary written
- [ ] Re-read `references/gates/gate-5-replay.md` BEFORE writing first entry of Gate 5
