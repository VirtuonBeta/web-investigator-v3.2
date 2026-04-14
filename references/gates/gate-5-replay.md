# Gate 5: Request Replay (P23–P27)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file after completing Gate 4 (deep exploration).

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from Gates 1–4

Before using this file, you should already have:
- From Gate 1 (`references/gates/gate-1-baseline.md`): cookie origin tracking (P7), robots.txt UA rules (P8)
- From Gate 2 (`references/gates/gate-2-pagination.md`): pagination endpoint (P11), pagination replay results (P13)
- From Gate 4 (`references/gates/gate-4-exploration.md`): API endpoints discovered (P17), token lifecycle (P18)

If any prerequisite is missing, return to the appropriate gate file before proceeding.

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 5 | P23 → P27 | Request replay — what does a scraper need? |

**Write gate at P27.** All pending observations must be logged, D2:State updated, D1 Phase Summary written, before proceeding to Gate 6.

---

## Phase 5: Request Replay (~5 cycles)

This phase systematically tests what's required to make API requests work outside the browser. Each test removes or modifies one variable, identifying which factors are required for successful responses.

**Request dependency tracking:**

Throughout Phase 5, track which prior requests are required for each test to succeed. This builds the HTTP request chain that an analyser needs to construct valid scraper requests.

**For each test (P23-P27), add these fields to the EDGE_CASE_TEST entry:**

- `prerequisite_requests`: Array of request entry IDs that must complete before this request can succeed (e.g., `["ent_002", "ent_005"]` — the initial page load and consent acceptance must happen first)
- `cookies_required`: Array of cookie names that this request requires. Cross-reference with P7 cookie origin tracking (from gate-1).
- `headers_required`: Array of header names that this request requires (e.g., `Authorization`, `X-CSRF-Token`, `Referer`)

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

**This chain IS the recipe for building a scraper.** An analyser reading this chain knows exactly what to send, in what order, to get valid responses.

---

### [P23] Modified User-Agent test

Some sites serve different content to different user agents, or block non-browser UAs entirely. This test determines whether the API cares about your User-Agent.

**Before testing:** Check robots.txt for UA-specific rules. If your modified UA matches a blocked bot, the test is INVALID — you're being blocked by robots.txt, not by the API. Log EDGE_CASE_TEST `UA_TEST_SKIPPED_ROBOTS_TXT`.

**Why this matters:** If the API returns different content for different UAs, you need to know which UA gets the full content. If it blocks non-browser UAs, you need to include a browser-like UA in your replay requests.

---

### [P24] Cookie-removal test

Remove all cookies from the request and replay. This determines whether the API requires session state or authentication.

**Possible outcomes:**

- **Same response:** No cookies required. The API is stateless.
- **Different content:** Some cookies affect the response (personalization, A/B testing).
- **Auth error:** Authentication is required. You need valid cookies for access.

**Why this matters:** If the API works without cookies, replay is simple — just fetch the URL. If cookies are required, you need to understand which cookies and how to obtain them (P18).

---

### [P25] Auth-removal test

Remove specific authentication headers (Authorization, X-Auth-Token, etc.) and replay. This isolates whether the auth header is required independent of cookies.

**Why this matters:** Some APIs use header-based auth (Bearer tokens) that are separate from cookie-based sessions. Understanding which auth mechanism is required determines the replay strategy.

---

### [P26] Cursor manipulation test

Modify the pagination cursor to access different pages. This tests whether the cursor is predictable (e.g., incrementing integer) or opaque (e.g., encrypted token).

**Tests:**

- **Increment cursor by 1:** Does it return the next page? → Predictable cursor.
- **Use a future/large cursor value:** Does it skip ahead? → Cursor is an offset.
- **Use an invalid cursor value:** Error? Empty results? → Cursor is validated server-side.
- **Reuse an old cursor:** Same results? → Cursor is deterministic. Different results? → Cursor includes server-side state.

**Why this matters:** Predictable cursors mean you can generate all pagination URLs programmatically. Opaque cursors mean you must follow the chain sequentially — each request returns the cursor for the next request.

---

### [P27] Conditional GET / ETag test

Send a request with `If-None-Match` / `If-Modified-Since` headers based on previous response headers. This determines whether the API supports efficient caching.

**Why this matters:** If the API supports conditional GET, you can poll for changes without re-downloading unchanged content. This is important for incremental extraction — you only download content that has actually changed.

---

## Gate 5 Output

Before proceeding to Gate 6, verify:

☐ Modified User-Agent test completed (P23)
☐ Cookie-removal test completed (P24)
☐ Auth-removal test completed (P25)
☐ Cursor manipulation test completed (P26)
☐ Conditional GET / ETag test completed (P27)
☐ HTTP_REQUEST_CHAIN SYSTEM entry written
☐ Pagination depth requirement documented in HTTP_REQUEST_CHAIN
☐ D2:State updated
☐ D1: Request Replay Phase Summary written
☐ Re-read `references/gates/gate-6-edgecases.md` BEFORE writing first entry of Gate 6
