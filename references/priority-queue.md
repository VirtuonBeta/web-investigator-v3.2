# Priority Queue — Detailed Steps (P1–P32+)

Reference file for the Web Investigator (Agent 1 v3.1). Read the relevant section before starting each phase.

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

---

## Phase 0: Pre-Flight

These steps run before the investigation proper begins. They ensure the environment is healthy and primed, so later observations are reliable. None of these consume a decision cycle.

---

### [Pre-P0] Read site_brief.md known_technology field

Before anything else, read the `known_technology` list from `site_brief.md`. This is the single most impactful optimization you can make — it lets you skip entire investigation branches and focus effort where it matters.

**Adjust your investigation approach based on what's known:**

| Known Technology | Adjustment |
|---|---|
| "Next.js App Router" | Prioritize P5a (RSC detection). Skip the full SSR vs CSR comparison — you already know it's RSC. |
| Specific framework (React, Vue, etc.) | Skip framework fingerprinting. Focus on API and content structure instead. |
| "GraphQL API" | Prioritize P21. You already know the API paradigm; focus on documenting operations. |
| "Akamai Bot Manager" | Increase initial delay to 3 seconds. Expect a potential BLOCKER. |
| CMS provider (Sanity, Contentful, etc.) | Look for CMS-specific API patterns during P11 (e.g., Sanity's `_type`/`_key`/`_ref`, Contentful's `sys.type`/`sys.id`). |
| Modern bundler with hashed filenames | Prioritize P19 — the bundle names may contain route hints. |

**Log:** SYSTEM entry with `known_technology_adjustment: {adjustments}` so downstream analysis can see what was skipped and why.

**Does NOT consume a decision cycle.**

---

### [Pre-P1] CDP Health Validation

If CDP isn't capturing, every subsequent step that relies on network observations is silently broken. This step catches that failure early, before you waste cycles on bad data.

**Procedure:**

1. After enabling CDP domains, navigate to `about:blank` or a minimal test page.
2. Make one explicit `fetch` request (e.g., `fetch('https://httpbin.org/get')`).
3. Verify the request appears in CDP capture.

**Outcomes:**

- **If NO requests captured:** CDP is broken. Log as **BLOCKER**. Do not proceed — nothing you observe from network traffic can be trusted.
- **If requests captured:** CDP is working. Proceed to P2.

**Log:** SYSTEM entry with event `cdp_health_check`.

---

### [Pre-P2] Warm-Up Request

The first load to any site primes CDN caches and establishes TLS sessions. This means the first load is often slower and may produce different network behavior than subsequent loads. Discard observations from this load to avoid skewed data.

**Procedure:**

1. Navigate to `target_url`, wait for network idle.
2. **IMMEDIATELY** navigate away (e.g., to `about:blank`).
3. Discard all observations from this load.

**Log:** EDGE_CASE_TEST with test_id `WARM_UP_COMPARE`.

Then proceed to P2 (the real investigation load).

---

## Phase 1: Baseline (~8 cycles)

This phase captures the fundamental nature of the page: what loads, how it renders, what data is embedded, and what the server reveals about its structure. Everything in later phases builds on these observations.

---

### [P1] Enable CDP capture

Enable the CDP Network, Runtime, Console, and Page domains. See `references/cdp-infrastructure.md` for the full domain enablement details and error handling.

**Why this matters:** Without CDP domains active, you're flying blind — no network requests, no console messages, no runtime exceptions will be captured.

**Not a decision cycle.**

---

### [P2] Navigate to target_url

Navigate to the target URL and wait for the page to settle. This is the single most important observation point — it captures the full page load waterfall, initial cookies, and the DOM state as the user would first see it.

**Procedure:**

- Navigate to `target_url`.
- Wait for network idle: 500ms with ≤2 in-flight requests.
- If navigation hangs beyond **15 seconds**: fall back to `DOMContentLoaded` event. Some pages never reach full idle (long-polling, SSE, etc.).

**Edge cases:**

- **Consent/gate wall detected:** Handle before proceeding (see SKILL.md §Consent Walls). A consent wall can block access to the real content entirely.
- **Redirected to completely different domain:** Log SYSTEM event `unexpected_domain_redirect`. This may indicate geo-routing, bot detection, or a misconfigured target.

**Captures:** All page load requests, cookies, DOM state.

---

### [P3] Full DOM snapshot

Take a complete DOM snapshot of the initial page state. This is your reference point — you'll compare against it throughout the investigation to detect dynamic changes, lazy-loaded content, and structural patterns.

**Log:** DOM_SNAPSHOT with context `initial_load`.

**Capture:** Article count, root selectors, card selectors, embedded JSON blocks.

**Why this matters:** Later steps (P9, P22) need a baseline DOM to compare against. Without this snapshot, you can't detect what changed or what's dynamic.

---

### [P3a] Shadow DOM detection

Modern web components use Shadow DOM to encapsulate their internals. Standard `querySelector` cannot pierce shadow boundaries, which means content living inside shadow roots is invisible to your normal DOM inspection. Detect shadow DOM early so you know which selectors need piercing.

**Execute this JS:**

```js
const shadowHosts = [];
const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_ELEMENT);
while (walker.nextNode()) {
  const el = walker.currentNode;
  if (el.shadowRoot) {
    shadowHosts.push({
      tag: el.tagName, id: el.id || null, classes: el.className || null,
      mode: el.shadowRoot.mode, childCount: el.shadowRoot.children.length,
      textContent: el.shadowRoot.textContent?.substring(0, 200) || ''
    });
  }
}
JSON.stringify(shadowHosts);
```

**If shadow hosts found (`shadowHosts.length > 0`):**

- Log EDGE_CASE_TEST with test_id `SHADOW_DOM_DETECTED`.
- For each shadow host, record: tag, selector, mode (open/closed), child count.
- **Open mode:** Snapshot shadow DOM contents (DOM_SNAPSHOT with context `shadow_dom_content`). You can access these via `.shadowRoot.querySelector()`.
- **Closed mode:** Log as UNKNOWN — content is inaccessible to JavaScript. You may need CDP DOM inspection as a fallback.
- **Critical note:** `querySelector` CANNOT reach inside shadow roots. You must use `.shadowRoot.querySelector()` for open roots. Closed roots require CDP-level access.

**If no shadow hosts (`shadowHosts.length === 0`):** No action needed.

---

### [P4] Window globals inspection

Many frameworks and third-party tools stash configuration, state, or data on the `window` object. These globals are often the richest source of embedded data — they can reveal the framework, the CMS, the analytics setup, and even the full page data.

**Check for these known globals:**

| Global | What it reveals |
|---|---|
| `window.YAHOO` | Legacy Yahoo framework |
| `window.__NEXT_DATA__` | Next.js Pages Router — full page props, routing, and data |
| `self.__next_f` | Next.js App Router — RSC streaming payload (NOT `__NEXT_DATA__`) |
| `window.__NUXT__` | Nuxt.js — state and data |
| `window.CONFIG` | Generic site configuration |
| `window.dataLayer` | Google Tag Manager — event tracking and e-commerce data |
| `window.OnetrustActiveGroups` | OneTrust consent state — which cookie categories are active |

**Also check:** Any global object with site-specific naming patterns (e.g., `window.MYAPP_CONFIG`, `window.__SITE_DATA__`).

**Why this matters:** These globals often contain the exact same data that the page renders, already structured as JSON. Finding them can eliminate the need for fragile DOM scraping.

---

### [P5] Embedded JSON/scripts extraction

Pages frequently embed structured data in `<script>` tags that isn't visible in the DOM but contains rich, machine-readable content. This is often the best data source on the page — cleaner than DOM scraping and more complete than what's visible.

**Extract from:**

- `<script type="application/json">` — frequently used by frameworks for hydration data.
- `<script type="application/ld+json">` — structured data for search engines (Schema.org). Often contains article metadata, product info, or organization data.
- `<script id="...">` with JSON content — site-specific embedded config or data.

**Why this matters:** Embedded JSON is typically the "source of truth" that the framework uses to render the page. It's more reliable than scraping rendered text because it's the structured input, not the formatted output.

---

### [P5a] RSC (React Server Components) detection

Next.js App Router uses React Server Components (RSC), which deliver content via a streaming protocol that is fundamentally different from traditional SSR or CSR. If you don't detect RSC, you'll misclassify the rendering strategy and potentially miss the primary data source.

**Check for:**

- `self.__next_f` — the RSC streaming buffer on the window object.
- `self.__next_f.push` calls in page scripts — these are RSC payload chunks.
- `<script src="*/_next/static/chunks/*">` with RSC-related patterns.

**If RSC detected:**

- Log EDGE_CASE_TEST with test_id `RSC_DETECTED`.
- **Override SSR vs CSR classification:** Classify as `"RSC"` — this is neither traditional SSR nor CSR.
- **RSC payloads ARE the embedded data source.** Parse `self.__next_f.push()` calls to extract structured data.
- **For P9 content structure:** Wait for RSC streaming to complete before counting items. Completion criteria: `document.readyState === 'complete'` AND no new `push` calls in 2 seconds (max wait 10s).

**Why this matters:** RSC content arrives incrementally after the initial HTML. If you snapshot the DOM too early, you'll see an empty or partial page. If you treat it as SSR, you'll miss the streaming data. If you treat it as CSR, you'll miss the server-rendered shell.

---

### [P6] Raw HTML sample

The raw HTML (before JavaScript execution) reveals the server's output directly. Comparing this against the post-JS DOM tells you whether content is server-rendered, client-rendered, or delivered via RSC streaming.

**Execute JS:** First 5000 chars of `document.documentElement.outerHTML`.

**Classification logic:**

1. **If P5a detected RSC:** Classify as `"RSC"` — content is delivered via RSC streaming, not traditional SSR or CSR.
2. **For non-RSC sites**, compare content item count in raw HTML vs post-JS count:
   - **Zero content items in raw HTML** → CSR (but check P5a first — RSC is expected to have an HTML shell).
   - **Some items but fewer than post-JS** → Hybrid (SSR shell + CSR hydration).
   - **Same items in both** → Pure SSR.

**Log:** The comparison counts — how many items in raw HTML, how many in post-JS DOM.

**Why this matters:** This classification determines your entire data extraction strategy. SSR means you can scrape raw HTML. CSR means you need JS execution. RSC means you need to parse streaming payloads.

---

### [P6a] \<head\> section analysis

The `<head>` section contains critical infrastructure signals: preconnect hints tell you which domains the page expects to contact, CSP headers tell you what's allowed, and meta tags reveal configuration. These signals guide later probing steps.

**Extract:**

- All `<link rel="preconnect" href="...">` → log these domains for the capture filter. Preconnect hints are a reliable signal of which third-party domains the page will contact.
- All `<meta>` tags → log names and content values. These can reveal CMS fingerprints, viewport settings, and SEO configuration.
- **CSP (Content-Security-Policy):** Retrieve from CDP response headers — NOT from the DOM. `<meta>` CSP is a fallback only; the HTTP header is authoritative.
- **CORS headers:** Retrieve from CDP response headers. These determine whether you can replay API requests outside the browser.

**Log:** DOM_SNAPSHOT with context `head_analysis`.

**Why this matters:** Preconnect domains become your P17 probing targets. CSP `connect-src` reveals which API domains are expected. CORS headers determine replay feasibility. Getting these from the right source (CDP headers, not DOM) matters because they can differ.

---

### [P6b] SPA route detection

Single-page applications handle navigation via JavaScript (History API) rather than full page loads. If you don't detect SPA routing, you'll miss that "navigation" doesn't trigger network requests — and you'll wonder why CDP shows no new page loads when the URL changes.

**Procedure:**

- Monitor `history.pushState` and `history.replaceState` calls.
- Install `popstate` listener, override `pushState`/`replaceState`.
- Detects SPA navigation without network page loads.

**Log:** EDGE_CASE_TEST with test_id `SPA_ROUTE_TEST`.

**Why this matters:** In an SPA, clicking a link might change the URL and update the DOM without any network activity. If you're only watching CDP network events, you'll think nothing happened. Detecting SPA routing ensures you know to inspect the DOM instead of waiting for network responses.

---

### [P7] Cookie logging

Cookies carry session state, authentication tokens, consent preferences, and tracking identifiers. Understanding the full cookie picture is essential for request replay (Phase 5) — if you don't know which cookies are required, you can't construct a valid replay.

**Procedure:**

- Capture all cookies from the page load (via CDP Network response headers).
- After navigation, use `CDP.Network.getCookies()` to capture JS-set cookies (these won't appear in response headers).
- Compare against `Set-Cookie` headers to identify JS-only cookies.

**Why JS-only cookies matter:** They're often set by analytics or consent platforms and may be required for the server to return full content. If a cookie exists in JS but not in response headers, it's a client-side signal that the server might check on subsequent requests.

---

### [P7b] LocalStorage capture

LocalStorage often stores auth tokens, user preferences, and application state that isn't in cookies. It's a parallel storage mechanism that can contain critical data not visible in the cookie jar.

**Execute JS:** `Object.keys(window.localStorage)`

For each key: record `value_sample` (first 30 chars only — don't log full values for privacy/size).

**Classify each key:**

- **Auth-related:** tokens, session IDs, user identifiers
- **Content-related:** cached data, user preferences, filters
- **Tracking/other:** analytics IDs, A/B test assignments

**Cross-reference with cookies:** If a key exists in BOTH localStorage and cookies, that redundancy signals importance — the site is storing the same data in two places, which usually means it's critical for functionality.

**Log:** LOCAL_STORAGE entry.

---

### [P8] robots.txt and sitemap.xml

`robots.txt` reveals which paths the site owner considers off-limits to automated access. `sitemap.xml` reveals the full URL structure — often exposing content patterns that aren't visible from navigation alone.

**Procedure:**

- Fetch via raw HTTP (if not already captured by CDP).
- Log full content per log format rules.
- **Parse robots.txt:** Disallow paths, Allow paths, Crawl-delay, Sitemap directives.
- Log parsed rules as SYSTEM entry.

**Why this matters:** `robots.txt` Disallow paths are essentially a map of the site's sensitive areas (admin, API, internal tools). These are paths you should skip during P17 probing. Sitemap URLs reveal the full content taxonomy — you'll use this in P8a.

---

### [P8a] Sitemap URL pattern classification

The sitemap often contains hundreds or thousands of URLs that reveal structural patterns invisible from the homepage alone. Classifying these patterns tells you about the site's content architecture, API endpoints, and deep web entry points.

**Procedure:**

1. Extract ALL URLs from sitemap.xml. Handle sitemap index files recursively (a sitemap can point to other sitemaps).
2. Classify URLs by pattern cluster:

| Pattern | Examples | Signal |
|---|---|---|
| Content pages | `/articles/`, `/posts/`, `/products/`, `/items/`, `/listings/` | Core content structure |
| Category/tag | `/category/`, `/tag/`, `/topic/`, `/search?` | Taxonomy / navigation |
| API-like | `/api/`, `/v1/`, `/v2/`, `/graphql`, `/rest/` | Machine-accessible endpoints |
| Date-structured | `/2024/`, `/2024/01/`, `/archive/` | Temporal organization |
| Query-parameter | Any URL containing `?` | Deep web entry points |

3. For each cluster: record pattern, count, first 3 sample URLs, and `deep_web` flag.
4. **Log as SYSTEM entry** `SITEMAP_CLASSIFICATION`.

**Edge cases:**

- **Sitemap reveals URL patterns NOT visible from navigation:** Log EDGE_CASE_TEST `SITEMAP_HIDDEN_STRUCTURE`. This means the sitemap exposes content that the UI doesn't link to — a significant finding for coverage.
- **Skip if:** no sitemap, empty sitemap, or fetch failed.

**Why this matters:** The sitemap is the site owner's declaration of what exists. It's more comprehensive than what you can discover by clicking around. API-like patterns in the sitemap are direct endpoints to test. Query-parameter URLs are deep web entry points that may accept filter parameters.

---

## Phase 2: Content Discovery (~5 cycles)

> **Phase gate reminder:** Before starting Phase 2, verify that Phase 1 completed successfully. You should have: a DOM snapshot (P3), a rendering classification (P6), and cookie/localStorage data (P7/P7b). If any of these are missing, go back and complete them — Phase 2 depends on understanding the page's baseline state.

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

**Why this matters:** If you can't identify content items, you can't click into them (P14), can't test pagination (P10-P13), and can't do structural comparison (P22). This step gates the entire content pipeline.

---

### [P9a] IntersectionObserver detection

Many sites use IntersectionObserver (IO) for lazy-loading content — items appear only when they scroll into the viewport. This is different from click-based pagination and requires different interaction patterns to trigger.

**Detect IO-based lazy loading via behavior, not API inspection:**

1. Note visible item count at current scroll position.
2. Scroll 500px, wait 2 seconds.
3. If new items appear: scroll another 500px, wait 2 seconds.
4. **Interpret the pattern:**
   - **Batches with pauses** → IntersectionObserver (items load in discrete batches as elements enter viewport).
   - **Continuous loading** → Scroll event listener (items load continuously during scroll).
   - **Only loads when scroll stops** → IO + debounce (items load after scrolling pauses).

**Log:** EDGE_CASE_TEST with test_id `INTERSECTION_OBSERVER_DETECTED`.

**Why viewport simulation matters:** IO-based loading responds to elements entering the viewport, not to scroll events. Just firing `scroll` events won't trigger IO observers — you need to actually change the viewport position. This is a common pitfall that leads to false "no more content" conclusions.

---

### [P10] Pagination mechanism identification

Pagination determines how you access content beyond the first page. Different mechanisms require different interaction strategies, and some have traps that can waste your entire budget.

**Check for:**

- Infinite scroll
- "Load More" button
- Numbered pages
- URL-based pagination (`?page=2`, `/page/2/`)

**Cross-reference with P9a:**

- If P9a detected IO: the pagination trigger is "element enters viewport" — not a button click.
- If multiple mechanisms exist: log each separately, test the primary one first.

**If no mechanism and single-page content:** Log `pagination_mechanism_identification: NONE_SINGLE_PAGE`. Skip P11-P13 — there's nothing to paginate.

**CRAWL TRAP DETECTION — critical for budget safety:**

| Trap Type | Detection Rule |
|---|---|
| Date/calendar parameters | BOUNDARY DATE = latest content date minus 1 month. Do NOT paginate past this. |
| Session/tracking parameters with identical content | Stop after 3 consecutive identical pages. Log EDGE_CASE_TEST `CRAWL_TRAP_DETECTED`. |
| Opaque cursor with no terminal signal | Cap at 5 pages. You can't tell when it ends, so don't chase it forever. |

**Log:** Mechanism type and trigger element. If a button is found, log its exact selector — you'll need it for P11.

**Why this matters:** Pagination is how you determine the full scope of available content. Without understanding the pagination mechanism, you can't estimate content volume, can't test API replay, and risk falling into crawl traps that consume your entire budget.

---

### [P11] Trigger pagination once — WITH CDP

Now that you know the pagination mechanism, trigger it once while CDP is listening. This captures the network request that fetches the next page of content — which is often an API call that reveals the site's data endpoint.

**Procedure:**

- Click button / scroll / follow page link according to the mechanism identified in P10.
- **Respect crawl trap boundaries** from P10 — don't trigger beyond safe limits.
- CDP captures the XHR/fetch request automatically.

**Log:** URL, method, params, response schema.

**If request goes to a THIRD-PARTY domain:** Log EDGE_CASE_TEST `THIRD_PARTY_CMS_API`.

- Note the CMS provider if identifiable from the response structure:
  - **Sanity:** Look for `_type`, `_key`, `_ref` fields.
  - **Contentful:** Look for `sys.type`, `sys.id` fields.
  - **Strapi:** Look for REST-style format with `id`, `attributes` structure.
- **A third-party CMS response is the PRIMARY data source** — it's the raw content before any frontend transformation.

**Why this matters:** The pagination API call is often the most valuable single observation in the entire investigation. It reveals the data endpoint, the response schema, the cursor/page mechanism, and potentially the CMS provider. This single request can replace hundreds of DOM scraping operations.

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

- Compare initial SSR content (from P9) against pagination response items.
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
- **Error/blocked:** The API requires something the browser provides that raw HTTP doesn't. Move to Phase 5 to diagnose what.

**Why this matters:** Replay success determines the entire extraction architecture. A replayable API means simple HTTP fetching. A non-replayable API means you need browser automation, which is slower and more expensive.

---

### [P13b] TLS fingerprint diagnostic

This step is triggered when P13 (or P24/P29/P30) raw HTTP replay fails with NO HTTP status code — meaning the connection itself was rejected, not the request. This pattern strongly suggests TLS fingerprinting: the server is checking the TLS handshake characteristics and rejecting non-browser clients.

**Trigger conditions (all must be true):**

1. Raw HTTP replay failed with NO HTTP status code (connection-level failure).
2. The same URL succeeded when accessed via the browser.
3. The failure is not a TRANSIENT error (retry once to confirm).

**Log:** EDGE_CASE_TEST `NON_HTTP_REPLAY_FAILURE`.

**Implications:** TLS fingerprinting means raw HTTP replay is impossible without TLS impersonation. This is a significant constraint on extraction strategy — you may need to use the browser for all data access, or use a tool that can impersonate browser TLS fingerprints.

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

**Triggered by:** preconnect hints (P6a), CSP `connect-src`, CDP-captured API subdomains, or agent judgment.

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

**CMS API token detection:** Check RSC payloads (P5a) for embedded CMS tokens or project IDs. Some Next.js sites embed Sanity/Contentful tokens in the RSC streaming data, which gives you authenticated API access.

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
