# Priority Queue — Pre-Halt (P0–P13c)

Reference file for the Web Investigator (Agent 1 v3.1). Read this file before the first-pass halt. After the operator resumes the investigation, switch to `references/priority-queue-posthalt.md` for Phases 3–8.

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 0 | Pre-Brief → Pre-P2 | Brief ingestion + environment validation |
| Phase 1 | P1 → P8a | Baseline — what IS this site? |
| Phase 2 | P9 → P13c | Content discovery — how is content structured? |

**HALT after P13c.** Present log to operator. Do not proceed without instruction.

---

---

## Phase 0: Pre-Flight

These steps run before the investigation proper begins. They ensure the environment is healthy and primed, so later observations are reliable. None of these consume a decision cycle.

---

### [Pre-Brief] Read site_brief.md in full

Read the entire site_brief.md before any other action.

Log a SYSTEM entry with:
```
event:                custom
description:          "site_brief read"
details:
  target_url:         <url>
  content_type:       <type>
  target_fields:      [<list all fields from target_data>]
  open_questions:     [<list all questions from questions field>]
  known_issues:       [<list>]
  known_technology:   [<list>]
  auth_required:      <bool>
  max_cycles:         <value or default 50>
  page_limit:         <value or default 15>
  geo_requirements:   <list or NONE>
```

This entry becomes the reference point for P16b verification. If a target_field or question is not in this entry, P16b cannot verify it was answered.

**Geo-awareness (conditional):** If `geo_requirements` includes `EU` (or if the target URL is a European domain and no geo_requirements field exists in site_brief.md):
1. Flag the investigation for P7c (consent flow mapping) — this step becomes mandatory, not optional.
2. Increase `max_cycles` by 2. Log the adjusted value in the Pre-Brief entry.
3. EU consent walls can gate content visibility, not just cookie state. Without P7c, the log may document truncated content as "full content."

**Does NOT consume a decision cycle.**

---

### [Pre-P0] Read site_brief.md known_technology field

Before anything else, read the `known_technology` list from `site_brief.md`. This is the single most impactful optimization you can make — it lets you skip entire investigation branches and focus effort where it matters.

**Adjust your investigation approach based on what's known:**

| Known Technology | Adjustment |
|---|---|
| "Next.js App Router" | Prioritize P5a (RSC detection). Skip the full SSR vs CSR comparison — you already know it's RSC. |
| Specific framework (React, Vue, etc.) | Skip framework fingerprinting. Focus on API and content structure instead. |
| "GraphQL API" | Prioritize P21 (→ priority-queue-posthalt.md). You already know the API paradigm; focus on documenting operations. |
| "Akamai Bot Manager" | Increase initial delay to 3 seconds. Expect a potential BLOCKER. |
| CMS provider (Sanity, Contentful, etc.) | Look for CMS-specific API patterns during P11 (e.g., Sanity's `_type`/`_key`/`_ref`, Contentful's `sys.type`/`sys.id`). |
| Modern bundler with hashed filenames | Prioritize P19 (→ priority-queue-posthalt.md) — the bundle names may contain route hints. |

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

**Extraction path type tagging:**

For each content item field visible in the initial DOM, classify its extraction path type. This classification determines extraction stability — which selectors survive site redesigns and which break on every deploy.

| Path Type | Priority | Pattern | Survives Deploy? |
|-----------|----------|---------|-----------------|
| `structured_data` | 1 (best) | ld+json, __NEXT_DATA__, embedded JSON | Always |
| `semantic_html` | 2 | `<article>`, `<time datetime>`, `<h1>`, `<main>` | Mostly |
| `aria_role` | 2 | `[role="article"]`, `[role="heading"]` | Mostly |
| `data_attribute` | 3 | `[data-testid]`, `[data-cy]`, `[data-component]` | Often |
| `meta_content` | 3 | `<meta name="description">`, `<meta property="og:*">` | Usually |
| `class_semantic` | 4 | `.article-title`, `.post-body` | Sometimes |
| `class_hashed` | 5 (worst) | `.yf-1a2b3c`, `.css-xyz123`, `._abc123` | Never |

**How to detect:** Run this JS after snapshot to classify each field's best available path:

```js
const allClasses = new Set();
document.querySelectorAll('[class]').forEach(el => {
  el.classList.forEach(c => allClasses.add(c));
});
const hashedPattern = /^[a-z]{1,3}-[a-z0-9]{4,}$|^_[a-z0-9]{4,}$|^css-[a-z0-9]+$/;
const hashedCount = [...allClasses].filter(c => hashedPattern.test(c)).length;
const class_type = hashedCount > allClasses.size * 0.5 ? 'hashed' :
                   hashedCount > allClasses.size * 0.2 ? 'mixed' : 'semantic';
// For each content field: check ld+json (P5) → semantic tag → aria role → data-* → meta → semantic class → hashed class
```

**Log:** Include `extraction_path_types` array in the DOM_SNAPSHOT entry and set `class_type` field. For each identified field, note its best available path type.

**Why this matters:** Yahoo Finance uses `yf-*` hashed classes that break on deploy. Wikipedia uses semantic HTML. The Guardian uses mixed. Without this classification, the agent logs `.yf-1a2b3c` as "stable" because it appears consistently — but it will break on the next deploy. This tagging feeds into P5 (field mapping), P15 (detail page mapping), and P22 (stability matrix).

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

**Field-to-extraction-path mapping:**

When ld+json or other structured data is found, map EVERY field to its JSON path. This mapping is the single most valuable artifact for building stable scrapers — it tells the analyser exactly which path to use for each field, with fallback ordering.

**For each ld+json block:**

1. Parse the JSON fully. Extract `@type` to identify the schema (Article, NewsArticle, Product, etc.)
2. List ALL fields with their JSON paths:

```
Schema: NewsArticle
  headline → ld+json.headline [structured_data]
  datePublished → ld+json.datePublished [structured_data]
  author.name → ld+json.author.name [structured_data]
  image.url → ld+json.image.url [structured_data]
```

3. **Cross-reference with P3 DOM snapshot** — for each field in the structured data, identify the DOM element that renders it. This creates a dual-path extraction map:

```
Field: headline
  Best: ld+json.headline [structured_data]
  Fallback: h1 [semantic_html]
  Last resort: .yf-1abc23d [class_hashed] [brittle]

Field: publish_date
  Best: ld+json.datePublished [structured_data]
  Fallback: time[datetime] [semantic_html]
  Last resort: span.yf-date [class_hashed] [brittle]
```

4. **If ld+json is NOT present**, explicitly log: `"No structured data found. All extraction paths are DOM-based. Stability risk: HIGH."`

**Log:** DOM_SNAPSHOT with context `embedded_json_N` MUST include `extraction_map` field containing the complete field-to-path mapping (see log-format.md).

**Why this matters:** Without this mapping, an analyser building from the log would use hashed class selectors that break on deploy, when ld+json would have been stable forever. This is the #1 most valuable output for the downstream analyst.

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

**Why this matters:** Preconnect domains become your P17 probing targets (→ priority-queue-posthalt.md). CSP `connect-src` reveals which API domains are expected. CORS headers determine replay feasibility. Getting these from the right source (CDP headers, not DOM) matters because they can differ.

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

Cookies carry session state, authentication tokens, consent preferences, and tracking identifiers. Understanding the full cookie picture is essential for request replay (Phase 5 → priority-queue-posthalt.md) — if you don't know which cookies are required, you can't construct a valid replay.

**Procedure:**

- Capture all cookies from the page load (via CDP Network response headers).
- After navigation, use `CDP.Network.getCookies()` to capture JS-set cookies (these won't appear in response headers).
- Compare against `Set-Cookie` headers to identify JS-only cookies.

**Why JS-only cookies matter:** They're often set by analytics or consent platforms and may be required for the server to return full content. If a cookie exists in JS but not in response headers, it's a client-side signal that the server might check on subsequent requests.

**Cookie origin tracking:**

For each cookie, record which HTTP response or JS execution set it. This is essential for constructing the HTTP request chain later — the analyser needs to know not just WHICH cookies exist, but WHERE they come from and in what ORDER they must be obtained.

**Tracking method:**
1. During P2 (initial navigation), CDP captures all `Set-Cookie` response headers. Map each cookie to the URL that set it.
2. After navigation, use `CDP.Network.getCookies()` to find JS-only cookies (no corresponding `Set-Cookie` header). Note as `set_by: javascript`.

**Log each cookie with additional field:**
- `set_by_request`: The entry ID of the REQUEST that set this cookie (e.g., `ent_002`). Creates a linkable chain: "Cookie A was set by the request in ent_002, which was the initial page load."

**Cookie dependency chain notation:**

After all cookies are logged, write a SYSTEM entry `COOKIE_DEPENDENCY_MAP` that summarizes the acquisition order:

```
Step 1: GET / → sets cookies [A1, A1S, GUCS, A3]
Step 2: GET /consent → sets cookies [consentStatus, euconsent-v2]
Step 3: GET /api/data → requires cookies [A1, A1S] → sets cookie [apiSession]
```

This chain tells the analyser exactly which cookies to obtain and in what order. Without it, the analyser knows WHICH cookies exist but not WHERE they come from or in what ORDER they must be obtained.

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

### [P7c] Consent flow mapping (EU-only, conditional)

**Triggered by:** Pre-Brief flagged `geo_requirements: EU`.

On EU sites, consent state can control what content is visible — not just what cookies are set. On sites like DN.se or SVT.se, rejecting consent truncates article text to a preview. This step maps the consent mechanism and its content impact.

**If no EU flag from Pre-Brief:** Skip this step entirely. Log nothing.

**Procedure:**

1. **Detect consent platform** — check for these known consent signals:

   | Signal | Platform |
   |---|---|
   | `window.OnetrustActiveGroups` | OneTrust |
   | `window.__tcfapi` or `window.__tcfapiLocator` | TCF v2 (IAB Europe) |
   | `window.__cmp` or `window.__cmpLocator` | CMP (older IAB) |
   | `window._sp_` | Sourcepoint |
   | `window.consentData` or `window.UC_UI` | Usercentrics |
   | Cookies containing `euconsent`, `eupubconsent`, `consentStatus` | Generic EU consent |

2. **Map consent categories** — for each detected platform, identify which cookie/content categories it manages:
   - OneTrust: Parse `OnetrustActiveGroups` string (comma-separated category IDs like `C0001,C0003`)
   - TCF v2: Call `__tcfapi('getTCData', 2, callback)` to get purpose consents
   - Log: which categories exist, which are active, which are required vs optional

3. **Content availability test** — compare content visibility before and after consent:
   - **Before consent acceptance:** Note visible article text length, DOM structure, any truncated elements with `aria-hidden` or `display:none` on content areas
   - **Click "Accept All" or equivalent:** Wait for consent handlers to complete (2 seconds)
   - **After consent acceptance:** Re-snapshot affected areas, note any content that appeared or expanded
   - **If consent banner was already handled in P2:** This test still runs — check whether the P2 consent state is "full consent" or "essential only"

4. **Map consent-dependent content** — for each content zone that changed:
   - Which consent category controls it (e.g., "C0003 functional cookies" vs "C0004 targeting cookies")
   - What content is gated vs always visible
   - Whether the gating is client-side (CSS hide/show, DOM removal) or server-side (different response body)

**Log:** SYSTEM entry with event `consent_flow_map` containing:
- Platform detected (OneTrust/TCF/Sourcepoint/Usercentrics/other/none)
- Categories and their states
- Content zones affected by consent state
- Whether gating is client-side or server-side

**Budget:** 2 decision cycles (consent state check + content comparison). Does NOT count toward P11 pagination budget.

**Why this matters:** On European news sites, the consent state is not just about cookies — it controls whether you see full article text or a truncated preview. A scraper that doesn't handle consent will get different content than one that does. Without this mapping, the analyst has no way to know that the content they see in the log is incomplete.

---

### [P8] robots.txt and sitemap.xml

`robots.txt` reveals which paths the site owner considers off-limits to automated access. `sitemap.xml` reveals the full URL structure — often exposing content patterns that aren't visible from navigation alone.

**Procedure:**

- Fetch via raw HTTP (if not already captured by CDP).
- Log full content per log format rules.
- **Parse robots.txt:** Disallow paths, Allow paths, Crawl-delay, Sitemap directives.
- Log parsed rules as SYSTEM entry.

**Why this matters:** `robots.txt` Disallow paths are essentially a map of the site's sensitive areas (admin, API, internal tools). These are paths you should skip during P17 probing (→ priority-queue-posthalt.md). Sitemap URLs reveal the full content taxonomy — you'll use this in P8a.

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
- **Error/blocked:** The API requires something the browser provides that raw HTTP doesn't. Move to Phase 5 (→ priority-queue-posthalt.md) to diagnose what.

**Why this matters:** Replay success determines the entire extraction architecture. A replayable API means simple HTTP fetching. A non-replayable API means you need browser automation, which is slower and more expensive.

---

### [P13b] TLS fingerprint diagnostic

This step is triggered when P13 (or P24/P29/P30) raw HTTP replay fails with NO HTTP status code — meaning the connection itself was rejected, not the request. This pattern strongly suggests TLS fingerprinting: the server is checking the TLS handshake characteristics and rejecting non-browser clients.

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


**Pre-halt steps complete.** After the operator resumes, switch to `references/priority-queue-posthalt.md` for Phases 3–8 (P14–P33+).
