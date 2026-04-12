# Priority Queue — Post-Halt (P14–P33+)

Reference file for the Web Investigator (Agent 1 v3.1). Read this file ONLY after the operator resumes the investigation following the first-pass halt.

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from Pre-Halt

Before using this file, you should already have (from `references/priority-queue-prehalt.md`):
- [ ] Content items identified (P9) with selectors
- [ ] Pagination mechanism classified (P10) and endpoint captured (P11)
- [ ] Pagination replay tested (P13) — know if raw HTTP works
- [ ] Search forms discovered (P12c) if any exist
- [ ] Rendering classification determined (P6/P5a)

If any prerequisite is missing, return to `references/priority-queue-prehalt.md` before proceeding.

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 3 | P14 → P16b | Content item entry — verify item structure |
| Phase 4 | P17 → P22 | Deep exploration — APIs, tokens, bundles |
| Phase 5 | P23 → P27 | Request replay — what does a scraper need? |
| Phase 6 | P28 → P32 | Edge case battery — boundary conditions |
| Phase 7 | P-X | Open exploration — unexpected observations |
| Phase 8 | P33+ | Re-investigation (if s2_gaps.md provided) |

---

## Phase 3: Content Item Entry (~3 cycles/item, min 3 items)

> **Phase gate reminder:** Before starting Phase 3, verify you have: identified content items (P9), understood pagination (P10-P13), and have at least one successful item click path. If content items aren't identifiable, this phase is blocked — return to P9.

This phase explores individual content items in detail. Each item takes ~3 cycles (click in, snapshot, navigate back). The goal is to understand item structure, detect hidden content, and build a reliable extraction schema.

**Item selection strategy:** Choose items with **variety** — different positions in the list (first, middle, last), different publishers/sources (if applicable), different content types (if applicable). Variety ensures your structural observations generalize rather than being specific to one item type.

---

### [P14] Click into item — CDP captures all requests

Click on a content item and let CDP capture everything that happens. The requests triggered by clicking into an item can reveal:

- **Item-specific API calls** — some sites fetch item detail via a separate API endpoint when you click.
- **Analytics/tracking requests** — what events fire on item view.
- **Third-party content loads** — ads, embeds, social widgets that load on the detail page.

**Why this matters:** If clicking into an item triggers a dedicated API call, that API is often a cleaner data source than scraping the detail page DOM. You may be able to replay this API call directly.

---

### [P15] Full DOM snapshot of item detail page

Snapshot the item detail page's DOM structure, organized by zone. This zone-based approach ensures you capture each functional area independently, which makes selector identification more reliable.

**Zones to capture:**

| Zone | What to look for |
|---|---|
| **Header zone** | Title, author, timestamp, publisher |
| **Body zone** | Content container, paragraph types, exclusions (ads, related content mixed in) |
| **Footer zone** | Related items, comments, social sharing |

**Log:** Unique item ID (from URL, data attribute, or embedded JSON). You'll need this for P22 cross-item comparison.

**Why this matters:** The body zone is where the actual content lives, but it's often polluted with ads, related articles, and other noise. Identifying the clean content container and its exclusions is critical for reliable extraction.

---

### [P15b] Hidden content element detection

Many sites hide full content behind "Read More" buttons, collapsible sections, or paywall mechanisms. These hidden elements contain content that's in the DOM but not visible — and they're critical for understanding whether you're getting the full content or just a preview.

**Execute this JS:**

```js
const bodyZone = document.querySelector('[role="main"], main, article, .article-body, .post-body, .entry-content, .story-body');
const container = bodyZone || document.body;
const hiddenElements = [];
const candidates = container.querySelectorAll(
  '[style*="display:none"], [style*="display: none"],' +
  '[style*="visibility:hidden"], [style*="visibility: hidden"],' +
  '[style*="height:0"], [style*="height: 0"],' +
  '[aria-hidden="true"],' +
  '[class*="collapsed"], [class*="expand"],' +
  '[class*="read-more"], [class*="continue"]'
);
candidates.forEach(el => {
  const text = el.textContent?.trim() || '';
  if (text.length > 100) {
    hiddenElements.push({
      tag: el.tagName,
      selector: el.id ? '#' + el.id : el.className ? '.' + el.className.split(' ').filter(c=>c).join('.') : el.tagName.toLowerCase(),
      hiddenMechanism: el.style.display === 'none' ? 'display:none' :
        el.style.visibility === 'hidden' ? 'visibility:hidden' :
        el.getAttribute('aria-hidden') === 'true' ? 'aria-hidden' :
        el.classList.contains('collapsed') ? 'class:collapsed' :
        'class-based (' + Array.from(el.classList).join(',') + ')',
      textLength: text.length,
      textSample: text.substring(0, 200),
      isLink: el.tagName === 'A' && el.hasAttribute('href'),
      isClickable: window.getComputedStyle(el).cursor === 'pointer' ||
        el.tagName === 'BUTTON' || el.getAttribute('role') === 'button'
    });
  }
});
JSON.stringify(hiddenElements);
```

**For each clickable non-navigation element (up to 3 per detail page):**

1. Click the element.
2. Wait 2 seconds (or network idle if XHR detected).
3. Re-snapshot the area.

**Interpret the result:**

- **Paywall signals detected** (e.g., subscription prompt, login wall, "upgrade to read more"): Log EDGE_CASE_TEST `PAYWALL_DETECTED`.
- **Genuine hidden content revealed** (more paragraphs, full article): Log EDGE_CASE_TEST `HIDDEN_CONTENT_REVEALED`.

**Safety rules:**

- **Do NOT click `<a>` tags with `href`** — those navigate to a new page, they don't expand content.
- **Budget:** Max 3 click attempts per detail page.

**Why this matters:** Hidden content detection is the difference between extracting a 50-word preview and the full 2000-word article. It also detects paywalls, which fundamentally change extraction feasibility.

---

### [P16] Navigate back — note if page re-fetches or serves from cache

After examining a detail page, navigate back to the listing page. This observation reveals the site's caching strategy and navigation behavior.

**What to note:**

- **Does the page re-fetch all content?** → No caching, or cache-control headers prevent it. Each navigation is expensive.
- **Does the page serve from cache?** → bfcache or HTTP cache is active. Navigation is cheap.
- **Does the page render differently?** → State was lost (SPA that doesn't restore scroll position, or server that returns different content on revisit).

**Why this matters:** If the listing page re-fetches on every back navigation, you know that repeated visits are expensive for the server. This affects how aggressively you can navigate back and forth during investigation. It also affects extraction strategy — if back-navigation triggers new API calls, those calls are additional data sources to capture.

---

### [P16b] Site Brief Field Verification

After completing P14-P16 for at least 3 items, cross-reference your findings against the fields in `site_brief.md`. This ensures the investigation actually covers what the operator asked about.

**Procedure:**

1. Re-read `site_brief.md` — specifically the `target_data` and `questions` fields.
2. For each field/question in the brief, check: did your observations so far address it?
3. If a field is unanswered: note it in D0 as an open question.
4. If a field is answered: note the entry ID(s) that provide the answer.

**Log:** SYSTEM entry with event `custom`, description "Site brief field verification", details containing a mapping of `{brief_field: entry_id_or_OPEN}`.

**Why this matters:** Without this check, it's easy to complete the investigation and realize you never answered the operator's actual question. The brief may ask about auth mechanisms while you spent all your budget on content structure. This step ensures alignment between investigation output and operator needs.

**Does NOT consume a full decision cycle** — it's a verification step that takes one re-read of the brief.

---

## Phase 4: Deep Exploration (~10 cycles, conditional)

> **Phase gate reminder:** Before starting Phase 4, verify you have: completed P14-P16 for at least 3 items, identified the pagination API (P11-P12), and tested replay (P13). Phase 4 steps are conditional — only run the ones triggered by your Phase 1-3 observations. Don't burn cycles on steps that aren't relevant.

These steps are triggered by specific observations from earlier phases. They dig deeper into infrastructure, authentication, and advanced patterns. Only execute the ones that are relevant to the site you're investigating.

---

### [P17] Subdomain/endpoint probing

**Triggered by:** preconnect hints (P6a, from prehalt), CSP `connect-src`, CDP-captured API subdomains, or agent judgment.

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

**CMS API token detection:** Check RSC payloads (P5a, from prehalt) for embedded CMS tokens or project IDs. Some Next.js sites embed Sanity/Contentful tokens in the RSC streaming data, which gives you authenticated API access.

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

---

## Phase 5: Request Replay (~5 cycles)

This phase systematically tests what's required to make API requests work outside the browser. Each test removes or modifies one variable, identifying which factors are required for successful responses.

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

## Phase 6: Edge Case Battery (~5 cycles)

These tests probe unusual conditions that may reveal different content, different behavior, or different access patterns. They're quick to execute and often reveal important constraints.

---

### [P28] Mobile viewport

Resize the browser to 375×812 (iPhone X dimensions), reload the page, and compare the DOM against the desktop version.

**What to look for:**

- **Different content:** Mobile may show fewer items, different ad placements, or mobile-specific features.
- **Different DOM structure:** Mobile may use different selectors, different component hierarchy, or AMP-specific markup.
- **Different API calls:** Mobile may hit different endpoints or return different response formats.

**Why this matters:** Many sites serve fundamentally different experiences on mobile — sometimes with simpler DOM structures that are easier to extract from. Mobile-specific APIs may also be less protected than desktop APIs.

---

### [P29] Empty User-Agent request via raw HTTP

Send a request with an empty or missing `User-Agent` header via raw HTTP.

**Why this matters:** Some sites have default behavior for unknown UAs that differs from both browser and bot UAs. An empty UA might bypass bot detection (if it only matches known bot UAs) or might trigger a block (if it requires a UA).

---

### [P30] Cookie-less request via raw HTTP

Send requests without any cookies via raw HTTP — both for the index page and a detail page.

**Why this matters:** This is the "can anyone access this content?" test. If the cookie-less request returns full content, the site is publicly accessible with no authentication. If it returns limited content or a login wall, you know exactly what's behind the authentication barrier.

---

### [P31] Sponsored/ad content identification

Identify which content items are sponsored, promoted, or advertisements rather than organic content.

**How to detect:**

- **Visual labels:** "Sponsored", "Ad", "Promoted", "Paid Partnership" in the DOM.
- **Structural differences:** Ad items may have different selectors, different parent containers, or `data-ad-*` attributes.
- **Different API sources:** Ads may come from a separate endpoint (e.g., `ad.*.com` domains in CDP captures).
- **Google Ad Manager:** Look for `googlesyndication.com`, `doubleclick.net` in network requests.

**Why this matters:** Including sponsored content in extraction results can be misleading or undesirable. Identifying it allows you to filter it out or flag it separately.

---

### [P32] Rate limit test

Fire the discovered API endpoint 5 times with 1-second delays. Observe whether responses degrade, return rate-limit errors (429), or remain consistent.

**What to watch for:**

- **HTTP 429 Too Many Requests:** Explicit rate limiting.
- **HTTP 503 Service Unavailable:** May indicate rate limiting or load shedding.
- **Gradual content reduction:** Some sites silently reduce content under load.
- **CAPTCHA challenge:** Some sites respond to rate limiting with CAPTCHAs instead of 429s.
- **Consistent responses:** No rate limiting detected at this request volume.

**Why this matters:** Understanding rate limits determines how fast you can extract data. If the API rate-limits at 5 requests/second, you need to throttle extraction accordingly. If there's no rate limiting at this volume, you may be able to extract faster — but should still be respectful.

---

## Phase 7: Open Exploration (lowest priority, ~15% remaining budget)

---

### [P-X] Open exploration

**Trigger:** The agent observes something potentially relevant that isn't covered by steps P1–P32.

**Budget cap:** 15% of remaining decision cycles.

**Requirements:**

- Must log what triggered the exploration and why.
- Must use standard log entry types (DOM_SNAPSHOT, EDGE_CASE_TEST, SYSTEM, etc.).
- Must still follow the general investigation principles from SKILL.md.

**Example triggers:**

- An unusual response header not seen before.
- A DOM element suggesting a feature not yet investigated (e.g., a comment section, a sharing widget, a dark mode toggle that reveals different content).
- An unexpected domain request in CDP captures.
- Dynamic content loading that doesn't fit the IO/pagination patterns detected in P9a/P10.
- A WebSocket message with a different schema than documented in P20.

**Why this exists:** No fixed procedure can anticipate every site's unique characteristics. Open exploration gives the agent room to follow up on observations that don't fit the standard framework, while keeping budget consumption bounded.

---

## Phase 8: Re-Investigation (only if s2_gaps.md provided)

---

### [P33+] Execute each specific request from s2_gaps.md in order

This phase only activates if the agent receives an `s2_gaps.md` file from a second-pass analysis. Each item in that file represents a specific gap or question that the first investigation didn't resolve.

**Procedure:**

- Execute each request from `s2_gaps.md` in order.
- Each request is a decision cycle.
- **Budget:** 5 cycles per request item, maximum.

**Why this matters:** The second-pass investigation is targeted — it only looks at specific gaps rather than re-running the full investigation. This budget cap prevents a single complex gap from consuming the entire re-investigation budget.

---

*End of Priority Queue reference. Return to SKILL.md for overall investigation philosophy and decision framework.*
