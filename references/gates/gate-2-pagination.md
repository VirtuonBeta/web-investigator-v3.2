# Gate 2: Content Discovery & Pagination (P9–P13c)

```yaml
gate_id: 2
title: Content Discovery & Pagination
steps: P9 → P13c
phases: [2]
d0_file: g2d0.log
operator_halt: true
next_gate: references/gates/gate-3-inspection.md
```

Read this file after completing Gate 1 (P0–P8a). After completing P13c, HALT and present log to operator. When operator resumes, read `references/gates/gate-3-inspection.md`. This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] DOM snapshot (P3) with extraction path types classified
- [ ] Rendering classification determined (P6/P5a)
- [ ] Cookie/localStorage data captured (P7/P7b)
- [ ] If EU site: consent flow mapping completed (P7c)

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries incl. BUDGET_STATUS, INVESTIGATION_FIRST_PASS_COMPLETE) | `g2d0.log` |
| D2:State updates | `state.log` |
| D1: Content Discovery Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 2 | P9 → P13c | Content discovery — how is content structured? |

**HALT after P13c.** Present log to operator. Do not proceed without instruction.

---

## Phase 2: Content Discovery (~5 cycles)

---

### [P9] Content structure identification

```yaml
step: P9
cycle: true
condition: ALWAYS
log: { type: DOM_SNAPSHOT, context: content_structure }
```

**Identify:**

- **What items are present?** (articles, products, listings, posts, etc.)
- **How many visible on first load?**
- **Shadow DOM check:** If P3a detected shadow hosts with content items, note which fields need shadow DOM piercing to access.
- **Different item types?** (featured vs standard, pinned vs organic, sponsored vs editorial)

**If ZERO items visible:**

| Observation | Action |
|------------|--------|
| Loading spinner | Wait for content to load |
| "No results" message | Log as empty state |
| Content after scroll/wait | Proceed with items found |
| RSC detected (P5a) | Wait for streaming completion + `readyState` check |
| Content never appears | **BLOCKER** — may require auth, geo-restricted, or blocking UA |

Log SYSTEM event `empty_content_state`.

**Date field check:** For each content card, inspect the date/time element. If the text is relative ("3h ago", "2d ago"), check for: (1) `<time datetime="...">` with ISO 8601, (2) `data-datetime` or `data-timestamp` attribute, (3) JSON in embedded data with absolute date. If none found, log `FEED_TIME: relative-only` in the extraction_map and flag in D2:Open that date filtering on the feed page requires relative-time parsing or detail-page visits.

**Feeds into:** P14 (click-through), P10-P13 (pagination), P22 (structural comparison).

---

### [P9a] IntersectionObserver detection

```yaml
step: P9a
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: INTERSECTION_OBSERVER_DETECTED }
```

**Detect IO-based lazy loading via behavior, not API inspection:**

1. Record `document.body.scrollHeight` and visible item count BEFORE scroll.
2. Scroll to `(scrollHeight - viewportHeight)`, i.e. the absolute page bottom.
3. Wait 3 seconds (some IO implementations debounce).
4. Record NEW `scrollHeight` and item count AFTER scroll.
5. If count grew → IO confirmed. Record:
   - Trigger position: scroll position when IO fired
   - Batch size: item count delta
   - New scrollHeight (for depth estimation)
6. If count unchanged AND scrollHeight unchanged → NO_IO_DETECTED.
7. If count unchanged BUT scrollHeight grew → IO may have loaded off-screen content. Scroll again to new bottom, wait 3s, re-check.
8. After confirming IO, continue scrolling in stages to map full pagination depth (batch count, total items, exhaustion point).

**IO endpoint extraction:** When IO loading is detected, CDP will have captured the XHR/fetch requests that loaded the new items. Extract from CDP captures:

- **Endpoint URL(s):** The XHR/fetch URLs that IO triggered (e.g., `/v2/articles?offset=24`)
- **Batch size:** Number of items returned per IO request
- **Total count signal:** Any "Showing 1-N of M" text in the DOM or `total`/`count` fields in the response

This extraction is critical because P11 (pagination trigger) assumes a button/scroll action. If the mechanism is pure IO, P11 needs to know which endpoint to probe — and P9a already triggered it.

Include `io_endpoints` array with URL, batch_size, and total_count (if found) in the entry details.

**Feeds into:** P11 (pagination trigger — IO endpoints), P10 (mechanism classification).

---

### [P10] Pagination mechanism identification

```yaml
step: P10
cycle: true
condition: ALWAYS
log: null
```

**CRITICAL: Scan ALL signal types before classifying.** Decision-tree pagination identification is the #1 cause of "stuck at first page" failures — the agent finds one mechanism (e.g., IO lazy-loading), classifies it, and stops looking. On Yahoo Finance, this caused the agent to identify infinite scroll and miss the underlying cursor API that was the real pagination mechanism.

**Scan for ALL 7 signal types before classifying:**

| # | Signal Type | Detection Method |
|---|---|---|
| 1 | **XHR/fetch pagination API** | Review CDP captures from P2 for XHR/fetch requests with pagination parameters (`page`, `offset`, `cursor`, `after`, `start`, `skip`) |
| 2 | **Infinite scroll (IO)** | Cross-reference with P9a — IO batches with pauses indicate viewport-triggered loading |
| 3 | **"Load More" button** | `document.querySelectorAll('button, a')` matching text patterns: "Load More", "Show More", "See More", "More Results" |
| 4 | **Numbered page links** | `document.querySelectorAll('a[href*="page="], a[href*="/page/"], nav a')` with numeric text |
| 5 | **URL-based pagination** | Current URL contains `?page=`, `?p=`, `/page/N/`, `?offset=`, `?start=` |
| 6 | **Cursor parameter** | CDP captures show requests with `cursor=`, `after=`, `startAfter=`, `continuation=`, `token=` parameters |
| 7 | **Session/tracking param pagination** | Same URL returns different content on reload with session cookies — server-side pagination without visible UI controls |

**After scanning all 7, classify the mechanism(s):**

A site can have MULTIPLE pagination mechanisms (e.g., IO for initial load + cursor API for deep pagination). Log EACH mechanism found, ranked by priority for scraper construction:

- **Primary:** The mechanism that provides the most complete data access (usually the XHR/fetch API if found)
- **Secondary:** Any additional mechanism that provides access to different content or different pages

**If multiple mechanisms exist:** Test the API-based mechanism first (P11) — it's always the better data source. IO/button mechanisms are fallback.

**If no mechanism and single-page content:** Log `pagination_mechanism_identification: NONE_SINGLE_PAGE`. Skip P11-P13 — there's nothing to paginate.

**CRAWL TRAP DETECTION — critical for budget safety:**

| Trap Type | Detection Rule |
|---|---|
| Date/calendar parameters | BOUNDARY DATE = latest content date minus 1 month. Do NOT paginate past this. |
| Session/tracking parameters with identical content | Stop after 3 consecutive identical pages. Log EDGE_CASE_TEST `CRAWL_TRAP_DETECTED`. |
| Opaque cursor with no terminal signal | Cap at 5 pages. You can't tell when it ends, so don't chase it forever. |

Crawl trap thresholds per SKILL.md §Rate Limiting & Safety.

**Brief contradiction check:** If site_brief mentions pagination/infinite scroll as a known or likely feature, AND P9a reports NO_IO_DETECTED, AND P10 finds NO pagination mechanism at all → flag as potential false negative in D2:Open. Recommend P-X re-investigation with deeper scroll in `reinvestigation_recommendations`.

**Log:** All signal types detected (not just the first one found), classified mechanism(s) with priority ranking, trigger element selectors.

**Feeds into:** P11 (trigger), P12 (response), P13 (replay).

---

### [P11] Trigger pagination with depth probing — WITH CDP

```yaml
step: P11
cycle: true
condition: P10 identified a pagination mechanism (not NONE_SINGLE_PAGE)
log: { type: EDGE_CASE_TEST, test_id: pagination_depth_probing }
```

**Procedure:**

1. Click button / scroll / follow page link according to the mechanism identified in P10.
2. **Respect crawl trap boundaries** from P10 — don't trigger beyond safe limits.
3. CDP captures the XHR/fetch request automatically.

**Depth probing (up to 5 pages):**

1. Trigger pagination for page 2. Log the request/response.
2. Trigger page 3. Compare response structure and item count to page 2.
3. **Fast path:** If pages 2 and 3 return consistent structure (same schema, similar item count, no errors), note "pagination consistent through page 3" and skip pages 4-5.
4. **Full path:** If ANY structural change appears (fewer items, missing fields, errors, truncated content), continue to pages 4 and 5.
5. **Soft cap detection:** If a page returns partial data or errors while earlier pages were fine, log EDGE_CASE_TEST `CRAWL_TRAP_DETECTED` with detail: "Soft cap at page N — full data through page N-1, degraded at page N."

**Budget:** 2-5 decision cycles depending on fast/full path. Most sites hit the fast path at 3 pages.

**Log:** URL, method, params, response schema for EACH page triggered. Note structural consistency or changes across pages.

**If request goes to a THIRD-PARTY domain:** Log EDGE_CASE_TEST `THIRD_PARTY_CMS_API`.

| CMS Provider | Signature Fields |
|---|---|
| **Sanity** | `_type`, `_key`, `_ref` fields |
| **Contentful** | `sys.type`, `sys.id` fields |
| **Strapi** | REST-style format with `id`, `attributes` structure |

Note the CMS provider if identifiable from the response structure. A third-party CMS response is the **PRIMARY data source** — it's the raw content before any frontend transformation.

**Feeds into:** P12 (response examination), P13 (replay), D2:State (CMS classification).

---

### [P12] Examine pagination response

```yaml
step: P12
cycle: true
condition: ALWAYS (after P11)
log: { type: DOM_SNAPSHOT, context: pagination_response_schema }
```

**Examine:**

1. **Schema:** What fields exist? What are their types? What's the nesting structure?
2. **Cursor fields:** How does the site track position? (`cursor`, `offset`, `after`, `page`, etc.)
3. **Has-more signal:** How does the site indicate more content is available? (`hasNextPage`, `hasMore`, `totalPages > currentPage`, absence of signal, etc.)

**Log:** Full response schema.

**Feeds into:** P13 (replay), P12b (overlap), content volume estimation.

---

### [P12b] Source overlap check

```yaml
step: P12b
cycle: true
condition: ALWAYS (after P12)
log: null
```

**Procedure:**

1. Compare initial SSR content (P9) against pagination response items.
2. Match by URL or item ID.
3. Count: SSR-only, API-only, overlap, total unique.

| Overlap | Meaning | Action |
|---|---|---|
| Zero (disjoint) | Dual source — SSR and API serve different items | Log EDGE_CASE_TEST `SSR_API_SOURCE_OVERLAP` |
| Partial | Some items appear in both, some are unique to each source | Log EDGE_CASE_TEST `SSR_API_SOURCE_OVERLAP` with percentages |
| Complete | SSR content is a subset of the API | Log in worklog "SSR subset of API" |

**Feeds into:** Extraction strategy (single vs dual source), completeness assessment.

---

### [P12c] Search form discovery

```yaml
step: P12c
cycle: true
condition: ALWAYS
log: { type: SYSTEM, event: SEARCH_FORM_INVENTORY }
```

**Procedure:**

1. Scan all visited pages for `<form>` elements.
2. Cross-reference with sitemap deep web patterns from P8a (query-parameter URLs).

**Classify each form:**

| Type | Examples |
|---|---|
| SEARCH | Site search, product search, article search |
| FILTER | Category filters, date range, price range |
| LOGIN | Sign in, authentication |
| NEWSLETTER | Email subscription |
| CONTACT | Contact form, feedback |
| OTHER | Anything that doesn't fit above |

**For SEARCH and FILTER forms only,** log: selector, action URL, method, all input fields, estimated item count, whether it's a PRIMARY content access method.

**If no search/filter forms:** Log in worklog "No deep web search forms detected."

**Feeds into:** P13c (search form probe).

---

### [P13] Test pagination replay

```yaml
step: P13
cycle: true
condition: ALWAYS (after P12)
log: { type: EDGE_CASE_TEST, test_id: pagination_replay }
```

**Procedure:**

1. Take the exact pagination XHR URL from P11.
2. Replay it via raw HTTP (outside the browser, using `fetch` or equivalent).
3. Compare the response to the browser-captured response.

| Outcome | Meaning | Action |
|---|---|---|
| Same response | Clean replay — API is freely accessible | Proceed |
| Different response (but valid) | Server-side variation (A/B testing, personalization) | Still usable |
| Error/blocked | API requires something the browser provides | → Phase 5 (gate-4-exploration.md) to diagnose |

**Feeds into:** Extraction architecture (HTTP vs browser), P13b (TLS diagnostic if blocked).

---

### [P13b] TLS fingerprint diagnostic

```yaml
step: P13b
cycle: true
condition: P13 (or P24/P28/P29) raw HTTP replay failed with NO HTTP status code
log: { type: EDGE_CASE_TEST, test_id: NON_HTTP_REPLAY_FAILURE }
```

**Trigger conditions (all must be true):**

1. Raw HTTP replay failed with NO HTTP status code (connection-level failure).
2. The same URL succeeded when accessed via the browser.
3. The failure is not a TRANSIENT error (retry once to confirm).

**Active probe — distinguish TLS vs header fingerprinting:**

When P13b fires, do NOT just log "probably TLS fingerprinting." Run 2 additional requests to determine the fingerprint type:

| # | Request | If Fails | If Succeeds |
|---|---------|----------|-------------|
| 1 | Minimal headers (only `Host` + `User-Agent`) | → Confirms fingerprinting, continue to test 2 | → Not fingerprinting |
| 2 | Full browser headers (`Accept`, `Accept-Language`, `Accept-Encoding`, `Sec-Fetch-Dest`, `Sec-Fetch-Mode`, `Sec-Fetch-Site`, `Sec-Ch-Ua`, `Sec-Ch-Ua-Mobile`, `Sec-Ch-Ua-Platform`, `Upgrade-Insecure-Requests`) | → **TLS-based** — server checks TLS handshake | → **Header-based** — server checks header presence/ordering |

| Fingerprint Type | Constraint | Solution |
|---|---|---|
| Header-based | Solvable by sending the right headers | Trivial fix — include correct headers in scraper |
| TLS-based | Raw HTTP replay impossible without TLS impersonation | Requires `curl-impersonate`, `got-scraping` — hard constraint |

**Log:** EDGE_CASE_TEST `NON_HTTP_REPLAY_FAILURE` with `fingerprint_type: "header" | "tls" | "none"`.

**Feeds into:** Extraction strategy (browser requirement), D2:Open (TLS constraint flag).

---

### [P13c] Search form probe

```yaml
step: P13c
cycle: true
condition: P12c found search or filter forms
log: null
```

Probe up to 2 forms from P12c to discover deep web API endpoints. Prioritize PRIMARY content access methods.

**Procedure:**

1. Enter a minimal test query in text fields.
2. Leave selects/checkboxes at their default values.
3. Submit via browser click (not raw HTTP — let CDP capture the request).
4. Wait for network idle.

**Interpret the response:**

| Outcome | Meaning | Log |
|---|---|---|
| XHR/fetch triggered | Deep web endpoint found — form submits to an API | EDGE_CASE_TEST `DEEP_WEB_ENDPOINT_FOUND` |
| Full page navigation | Form navigates to a search results page | EDGE_CASE_TEST `SEARCH_FORM_NAVIGATION` |
| Form appears stateful (POST to `/api/create`) | Write endpoint, not a search — DO NOT SUBMIT | SYSTEM `SEARCH_FORM_SKIPPED_STATEFUL` |
| CAPTCHA | Bot detection triggered | BLOCKER |

**Budget:** Max 2 form probes.

**Feeds into:** Deep web extraction endpoints, D2:State (search API classification).

---

**Pre-halt steps complete.** After the operator resumes, switch to `references/gates/gate-3-inspection.md` for Phase 3 (P14–P16b), then `references/gates/gate-4-exploration.md` for Phases 4–8 (P17–P32+).

## Gate 2 Output

Before halting and presenting to operator, verify:

- [ ] Content items identified with selectors (P9)
- [ ] Date field check completed (P9)
- [ ] IntersectionObserver detection completed (P9a)
- [ ] Pagination mechanism classified — ALL 7 signal types scanned (P10)
- [ ] Pagination triggered with depth probing (P11)
- [ ] Pagination response examined (P12)
- [ ] Source overlap checked (P12b)
- [ ] Search forms discovered and probed (P12c)
- [ ] Pagination replay tested (P13)
- [ ] TLS fingerprint diagnostic completed if needed (P13b)
- [ ] Search form probe completed (P13c)
- [ ] D2:State updated
- [ ] D1: Content Discovery Phase Summary written
- [ ] BUDGET_STATUS written (to g2d0.log)
- [ ] INVESTIGATION_FIRST_PASS_COMPLETE SYSTEM entry written (to g2d0.log)
- [ ] Awaiting operator instruction to continue
- [ ] When resuming: read `references/gates/gate-3-inspection.md`
