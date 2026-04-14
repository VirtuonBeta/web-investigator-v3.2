# Gate 2: Content Discovery & Pagination (P9–P13c)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file after completing Gate 1 (P0–P8a). After completing P13c, HALT and present log to operator. When operator resumes, read `references/gates/gate-3-inspection.md` for content item inspection (P14–P16b).

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from Gate 1

Before using this file, you should already have (from `references/gates/gate-1-baseline.md`):
- [ ] DOM snapshot (P3) with extraction path types classified
- [ ] Rendering classification determined (P6/P5a)
- [ ] Cookie/localStorage data captured (P7/P7b)
- [ ] If EU site: consent flow mapping completed (P7c)

If any prerequisite is missing, return to `references/gates/gate-1-baseline.md` before proceeding.

## Write Targets

| What | File | Why |
|------|------|-----|
| Raw observations | `g2d0.log` | Gate-scoped raw data |
| D2:State updates | `state.log` | State checkpoint |
| D1: Content Discovery Phase Summary | `state.log` | Phase completion record |
| BUDGET_STATUS (at P13) | `state.log` | Budget checkpoint |
| INVESTIGATION_FIRST_PASS_COMPLETE | `state.log` | Lifecycle milestone |

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 2 | P9 → P13c | Content discovery — how is content structured? |

**HALT after P13c.** Present log to operator. Do not proceed without instruction.

---

## Phase 2: Content Discovery (~5 cycles)

> **Phase gate reminder:** Before starting Phase 2, verify that Phase 1 completed successfully (see `references/gates/gate-1-baseline.md`). You should have: a DOM snapshot (P3), a rendering classification (P6), and cookie/localStorage data (P7/P7b). If any of these are missing, return to `references/gates/gate-1-baseline.md` before proceeding — Phase 2 depends on understanding the page's baseline state.
>
> ☐ Write D1 phase summary for completed phase before proceeding.

---

### [P9] Content structure identification

This step identifies the fundamental content units on the page — what items exist, how many are visible, and how they're structured. This drives every subsequent content-related step.

**Identify:**

- **What items are present?** (articles, products, listings, posts, etc.)
- **How many visible on first load?**
- **Shadow DOM check:** If P3a detected shadow hosts with content items, note which fields need shadow DOM piercing to access.
- **Different item types?** (featured vs standard, pinned vs organic, sponsored vs editorial)

**Log:** DOM_SNAPSHOT with context `content_structure`.

**If ZERO items visible:**

- Log SYSTEM event `empty_content_state`.
- Check: loading spinner? "No results" message? Content after scroll/wait?
- **If RSC detected (P5a):** Wait for streaming completion + `readyState` check.
- **If content never appears:** This is a BLOCKER — the page may require authentication, may be geo-restricted, or may be blocking your UA.

**Date field check:** For each content card, inspect the date/time element. If the text is relative ("3h ago", "2d ago"), check for: (1) `<time datetime="...">` with ISO 8601, (2) `data-datetime` or `data-timestamp` attribute, (3) JSON in embedded data with absolute date. If none found, log `FEED_TIME: relative-only` in the extraction_map and flag in D2:Open that date filtering on the feed page requires relative-time parsing or detail-page visits.

**Why this matters:** If you can't identify content items, you can't click into them (P14), can't test pagination (P10-P13), and can't do structural comparison (P22). This step gates the entire content pipeline.

---

### [P9a] IntersectionObserver detection

Many sites use IntersectionObserver (IO) for lazy-loading content — items appear only when they scroll into the viewport. This is different from click-based pagination and requires different interaction patterns to trigger.

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
7. If count unchanged BUT scrollHeight grew → IO may have loaded off-screen
   content. Scroll again to new bottom, wait 3s, re-check.
8. After confirming IO, continue scrolling in stages to map full pagination
   depth (batch count, total items, exhaustion point).

**IO endpoint extraction:** When IO loading is detected, CDP will have captured the XHR/fetch requests that loaded the new items. Extract these from CDP captures and log them as part of the P9a observation:

- **Endpoint URL(s):** The XHR/fetch URLs that IO triggered (e.g., `/v2/articles?offset=24`)
- **Batch size:** Number of items returned per IO request
- **Total count signal:** Any "Showing 1-N of M" text in the DOM or `total`/`count` fields in the response

This extraction is critical because P11 (pagination trigger) assumes a button/scroll action. If the mechanism is pure IO, P11 needs to know which endpoint to probe — and P9a already triggered it.

**Log:** EDGE_CASE_TEST with test_id `INTERSECTION_OBSERVER_DETECTED`. Include `io_endpoints` array with URL, batch_size, and total_count (if found) in the entry details.

**Why viewport simulation matters:** IO-based loading responds to elements entering the viewport, not to scroll events. Just firing `scroll` events won't trigger IO observers — you need to actually change the viewport position. This is a common pitfall that leads to false "no more content" conclusions.

---

### [P10] Pagination mechanism identification

Pagination determines how you access content beyond the first page. Different mechanisms require different interaction strategies, and some have traps that can waste your entire budget.

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

**Log:** All signal types detected (not just the first one found), classified mechanism(s) with priority ranking, trigger element selectors. If a button is found, log its exact selector — you'll need it for P11.

**Why this matters:** Pagination is how you determine the full scope of available content. Without understanding the pagination mechanism, you can't estimate content volume, can't test API replay, and risk falling into crawl traps that consume your entire budget. The scan-first approach prevents the common failure mode where the agent identifies IO lazy-loading and stops, missing the underlying cursor API.

---

### [P11] Trigger pagination with depth probing — WITH CDP

Now that you know the pagination mechanism, trigger it while CDP is listening. Depth probing (up to 5 pages) catches "soft caps" where pagination works for the first few pages then degrades — a pattern that a single trigger would miss.

**Procedure:**

- Click button / scroll / follow page link according to the mechanism identified in P10.
- **Respect crawl trap boundaries** from P10 — don't trigger beyond safe limits.
- CDP captures the XHR/fetch request automatically.

**Depth probing (up to 5 pages):**

1. Trigger pagination for page 2. Log the request/response.
2. Trigger page 3. Compare response structure and item count to page 2.
3. **Fast path:** If pages 2 and 3 return consistent structure (same schema, similar item count, no errors), note "pagination consistent through page 3" and skip pages 4-5.
4. **Full path:** If ANY structural change appears (fewer items, missing fields, errors, truncated content), continue to pages 4 and 5.
5. **Soft cap detection:** If a page returns partial data or errors while earlier pages were fine, log EDGE_CASE_TEST `CRAWL_TRAP_DETECTED` with detail: "Soft cap at page N — full data through page N-1, degraded at page N."

**Budget:** 2-5 decision cycles depending on fast/full path. Most sites hit the fast path at 3 pages.

**Log:** URL, method, params, response schema for EACH page triggered. Note structural consistency or changes across pages.

**If request goes to a THIRD-PARTY domain:** Log EDGE_CASE_TEST `THIRD_PARTY_CMS_API`.

- Note the CMS provider if identifiable from the response structure:
  - **Sanity:** Look for `_type`, `_key`, `_ref` fields.
  - **Contentful:** Look for `sys.type`, `sys.id` fields.
  - **Strapi:** Look for REST-style format with `id`, `attributes` structure.
- **A third-party CMS response is the PRIMARY data source** — it's the raw content before any frontend transformation.

**Why this matters:** The pagination API call is often the most valuable single observation in the entire investigation. It reveals the data endpoint, the response schema, the cursor/page mechanism, and potentially the CMS provider. Depth probing catches soft caps that a single trigger would miss — some sites return full data for pages 1-3 then degrade at page 4. Without probing, you'd log "pagination works fine" and the scraper would break in production.

---

### [P12] Examine pagination response

The pagination response schema reveals how the site structures its data, how it signals "no more content," and what fields are available. This is the blueprint for constructing replay requests.

**Examine:**

- **Schema:** What fields exist? What are their types? What's the nesting structure?
- **Cursor fields:** How does the site track position? (`cursor`, `offset`, `after`, `page`, etc.)
- **Has-more signal:** How does the site indicate more content is available? (`hasNextPage`, `hasMore`, `totalPages > currentPage`, absence of signal, etc.)

**Log:** Full response schema.

**Why this matters:** Understanding the response schema is prerequisite for P13 (replay) and for estimating total content volume. If you can't parse the cursor, you can't replay. If you can't find the has-more signal, you can't determine when to stop.

---

### [P12b] Source overlap check

When a site uses both SSR and API-pagination, the initial page load may contain some content items that also appear in the API response. Understanding the overlap tells you whether you have one data source or two, which affects extraction strategy.

**Procedure:**

- Compare initial SSR content (P9) against pagination response items.
- Match by URL or item ID.
- Count: SSR-only, API-only, overlap, total unique.

**Interpretation:**

| Overlap | Meaning | Action |
|---|---|---|
| Zero (disjoint) | Dual source — SSR and API serve different items | Log EDGE_CASE_TEST `SSR_API_SOURCE_OVERLAP` |
| Partial | Some items appear in both, some are unique to each source | Log EDGE_CASE_TEST `SSR_API_SOURCE_OVERLAP` with percentages |
| Complete | SSR content is a subset of the API | Log in worklog "SSR subset of API" |

**Why this matters:** If SSR and API are disjoint, you need both sources for complete coverage. If SSR is a subset of API, you only need the API. This determination directly affects extraction strategy and completeness.

---

### [P12c] Search form discovery

Search and filter forms are deep web entry points — they expose content that isn't reachable by browsing or pagination alone. Identifying these forms early lets you probe them for API endpoints.

**Procedure:**

- Scan all visited pages for `<form>` elements.
- Cross-reference with sitemap deep web patterns from P8a (query-parameter URLs).

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

**Log:** SYSTEM entry `SEARCH_FORM_INVENTORY`.

**If no search/filter forms:** Log in worklog "No deep web search forms detected."

**Why this matters:** Search forms are the gateway to the deep web — content that exists in the site's database but isn't linked from any page. Probing these forms (P13c) can reveal API endpoints that return structured data for any query.

---

### [P13] Test pagination replay

The ultimate goal is to determine whether the pagination API can be called directly, without a browser. If it can, extraction is trivial — just replay the request in a loop. If it can't, you need to understand what's required (cookies, tokens, headers) and whether those requirements can be met.

**Procedure:**

- Take the exact pagination XHR URL from P11.
- Replay it via raw HTTP (outside the browser, using `fetch` or equivalent).
- Compare the response to the browser-captured response.

**Log:** EDGE_CASE_TEST for `pagination_replay`.

**Possible outcomes:**

- **Same response:** Clean replay. The API is freely accessible.
- **Different response (but valid):** Some server-side variation (A/B testing, personalization). Still usable.
- **Error/blocked:** The API requires something the browser provides that raw HTTP doesn't. Move to Phase 5 (→ gate-4-exploration.md) to diagnose what.

**Why this matters:** Replay success determines the entire extraction architecture. A replayable API means simple HTTP fetching. A non-replayable API means you need browser automation, which is slower and more expensive.

---

### [P13b] TLS fingerprint diagnostic

This step is triggered when P13 (or P24/P28/P29) raw HTTP replay fails with NO HTTP status code — meaning the connection itself was rejected, not the request. This pattern strongly suggests TLS fingerprinting: the server is checking the TLS handshake characteristics and rejecting non-browser clients.

**Trigger conditions (all must be true):**

1. Raw HTTP replay failed with NO HTTP status code (connection-level failure).
2. The same URL succeeded when accessed via the browser.
3. The failure is not a TRANSIENT error (retry once to confirm).

**Active probe — distinguish TLS vs header fingerprinting:**

When P13b fires, do NOT just log "probably TLS fingerprinting." Run 2 additional requests to determine the fingerprint type:

1. **Minimal headers request:** Send raw HTTP with only `Host` and `User-Agent` headers. If this also fails with no HTTP status → confirms fingerprinting, continue to test 2.
2. **Full browser headers request:** Send raw HTTP with a complete browser header set (`Accept`, `Accept-Language`, `Accept-Encoding`, `Sec-Fetch-Dest`, `Sec-Fetch-Mode`, `Sec-Fetch-Site`, `Sec-Ch-Ua`, `Sec-Ch-Ua-Mobile`, `Sec-Ch-Ua-Platform`, `Upgrade-Insecure-Requests`). If this SUCCEEDS, the fingerprinting is **header-based** — the server checks header presence/ordering, not TLS. If this also FAILS, the fingerprinting is **TLS-based** — the server checks the TLS handshake.

**Why this distinction matters:**
- **Header fingerprinting** → Solvable by sending the right headers in the scraper. Trivial fix.
- **TLS fingerprinting** → Requires `curl-impersonate`, `got-scraping`, or similar TLS-mimicking tools. Hard constraint.

**Log:** EDGE_CASE_TEST `NON_HTTP_REPLAY_FAILURE` with `fingerprint_type: "header" | "tls" | "none"` field.

**Implications:** TLS fingerprinting means raw HTTP replay is impossible without TLS impersonation. This is a significant constraint on extraction strategy — you may need to use the browser for all data access, or use a tool that can impersonate browser TLS fingerprints. Header fingerprinting is much easier to work around — the analyst just needs to include the right headers.

---

### [P13c] Search form probe

If P12c found search or filter forms, probe up to 2 of them to discover deep web API endpoints. Prioritize forms that appear to be the PRIMARY content access method (e.g., a product search on an e-commerce site).

**Procedure:**

1. Enter a minimal test query in text fields.
2. Leave selects/checkboxes at their default values.
3. Submit via browser click (not raw HTTP — let CDP capture the request).
4. Wait for network idle.

**Interpret the response:**

| Outcome | Meaning | Log |
|---|---|---|
| XHR/fetch triggered | Deep web endpoint found — the form submits to an API | EDGE_CASE_TEST `DEEP_WEB_ENDPOINT_FOUND` |
| Full page navigation | Form navigates to a search results page | EDGE_CASE_TEST `SEARCH_FORM_NAVIGATION` |
| Form appears stateful (POST to `/api/create`) | This is a write endpoint, not a search — DO NOT SUBMIT | SYSTEM `SEARCH_FORM_SKIPPED_STATEFUL` |
| CAPTCHA | Bot detection triggered | BLOCKER |

**Budget:** Max 2 form probes.

**Why this matters:** Search form API endpoints often accept arbitrary query parameters and return structured data. They're the most powerful deep web access point — a single search endpoint can expose the entire content catalog.

---


**Pre-halt steps complete.** After the operator resumes, switch to `references/gates/gate-3-inspection.md` for Phase 3 (P14–P16b), then `references/gates/gate-4-exploration.md` for Phases 4–8 (P17–P32+).

## Gate 2 Output

Before halting and presenting to operator, verify:

☐ Content items identified with selectors (P9)
☐ Date field check completed (P9)
☐ IntersectionObserver detection completed (P9a)
☐ Pagination mechanism classified — ALL 7 signal types scanned (P10)
☐ Pagination triggered with depth probing (P11)
☐ Pagination response examined (P12)
☐ Source overlap checked (P12b)
☐ Search forms discovered and probed (P12c)
☐ Pagination replay tested (P13)
☐ TLS fingerprint diagnostic completed if needed (P13b)
☐ Search form probe completed (P13c)
☐ D2:State updated
☐ D1: Content Discovery Phase Summary written
☐ BUDGET_STATUS written
☐ INVESTIGATION_FIRST_PASS_COMPLETE SYSTEM entry written
☐ Awaiting operator instruction to continue
☐ When resuming: read `references/gates/gate-3-inspection.md`
