# Gate 5: Request Replay (P23–P27)

```yaml
gate_id: 5
title: Request Replay
steps: P23 → P27
phases: [5]
d0_file: g5d0.log
operator_halt: false
next_gate: references/gates/gate-6-edgecases.md
```

Read this file after completing Gate 4 (deep exploration). This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] Cookie origin tracking (P7), robots.txt UA rules (P8)
- [ ] Pagination endpoint (P11), pagination replay results (P13)
- [ ] API endpoints discovered (P17), token lifecycle (P18)

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries including HTTP_REQUEST_CHAIN) | `g5d0.log` |
| D2:State updates | `state.log` |
| D1: Request Replay Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 5 | P23 → P27 | Request replay — what does a scraper need? |

**Write gate at P27.** All pending observations logged, D2:State updated, D1 Phase Summary written, before proceeding to Gate 6.

---

## Phase 5: Request Replay (~5 cycles)

Systematically test what's required to make API requests work outside the browser. Each test removes or modifies one variable.

**Request dependency tracking:**

For each test (P23-P27), add these fields to the EDGE_CASE_TEST entry:

- `prerequisite_requests`: Array of gate-qualified request IDs that must complete first (e.g., `["g1:003", "g1:007"]`)
- `cookies_required`: Array of cookie names this request requires. Cross-reference with P7 (gate-1).
- `headers_required`: Array of header names this request requires (e.g., `Authorization`, `X-CSRF-Token`, `Referer`)

**At the end of Phase 5 (after P27), write a SYSTEM entry `HTTP_REQUEST_CHAIN`:**

```
Step 1: GET {target_url}
  Sets cookies: [A1, A1S, GUCS, A3]
  Returns: HTML page with initial content + crumb token
  Required for: Steps 2, 3, 4

Step 2: GET {pagination_endpoint}
  Requires cookies: [A1, A1S, GUCS] (from Step 1)
  Requires headers: [Referer: {target_url}]
  Returns: JSON pagination data
  Required for: Step 5

Step 3: GET {item_detail_url}
  Requires cookies: [A1, A1S, GUCS] (from Step 1)
  Returns: HTML detail page with ld+json

Step 4: GET {pagination_endpoint}?cursor={next_cursor}
  Requires cookies: [A1, A1S, GUCS] (from Step 1)
  Requires: cursor from Step 2 response
  Returns: Next page of JSON pagination data
```

---

### [P23] Modified User-Agent test

```yaml
step: P23
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: UA_MODIFIED_TEST }
```

**Before testing:** Check robots.txt for UA-specific rules. If your modified UA matches a blocked bot, the test is INVALID — blocked by robots.txt, not by the API. Log EDGE_CASE_TEST `UA_TEST_SKIPPED_ROBOTS_TXT`.

**Feeds into:** P28 (empty UA test), HTTP_REQUEST_CHAIN.

---

### [P24] Cookie-removal test

```yaml
step: P24
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: COOKIE_REMOVAL_TEST }
```

Remove all cookies from the request and replay.

| Outcome | Implication |
|---------|-------------|
| Same response | No cookies required; API is stateless |
| Different content | Some cookies affect response (personalization, A/B testing) |
| Auth error | Authentication required; need valid cookies (see P18) |

**Feeds into:** P29 (raw HTTP cookie-less), HTTP_REQUEST_CHAIN.

---

### [P25] Auth-removal test

```yaml
step: P25
cycle: true
condition: auth headers detected (Authorization, X-Auth-Token, etc.)
log: { type: EDGE_CASE_TEST, test_id: AUTH_REMOVAL_TEST }
```

Remove specific authentication headers (Authorization, X-Auth-Token, etc.) and replay. Isolates whether the auth header is required independent of cookies.

**Feeds into:** HTTP_REQUEST_CHAIN.

---

### [P26] Cursor manipulation test

```yaml
step: P26
cycle: true
condition: pagination uses cursor-based navigation
log: { type: EDGE_CASE_TEST, test_id: CURSOR_MANIPULATION_TEST }
```

| Test | Observation | Conclusion |
|------|-------------|------------|
| Increment cursor by 1 | Returns next page? | Predictable cursor |
| Use future/large cursor value | Skips ahead? | Cursor is an offset |
| Use invalid cursor value | Error? Empty? | Cursor is validated server-side |
| Reuse old cursor | Same results? | Cursor is deterministic |

**Feeds into:** HTTP_REQUEST_CHAIN (pagination depth requirement).

---

### [P27] Conditional GET / ETag test

```yaml
step: P27
cycle: true
condition: previous responses included ETag or Last-Modified headers
log: { type: EDGE_CASE_TEST, test_id: CONDITIONAL_GET_TEST }
```

Send a request with `If-None-Match` / `If-Modified-Since` headers based on previous response headers. Determines whether the API supports efficient caching.

**Feeds into:** Extraction strategy (incremental extraction feasibility).

---

## Gate 5 Output

Before proceeding to Gate 6, verify:

- [ ] Modified User-Agent test completed (P23)
- [ ] Cookie-removal test completed (P24)
- [ ] Auth-removal test completed (P25)
- [ ] Cursor manipulation test completed (P26)
- [ ] Conditional GET / ETag test completed (P27)
- [ ] HTTP_REQUEST_CHAIN SYSTEM entry written
- [ ] Pagination depth requirement documented in HTTP_REQUEST_CHAIN
- [ ] D2:State updated
- [ ] D1: Request Replay Phase Summary written
- [ ] Re-read `references/gates/gate-6-edgecases.md` BEFORE writing first entry of Gate 6
