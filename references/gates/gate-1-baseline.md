# Gate 1: Baseline (P0–P8a)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file first, before any investigation step. After completing P8a, proceed to `references/gates/gate-2-pagination.md` for content discovery (P9–P13c). After the operator resumes, see `references/gates/gate-3-inspection.md` for item inspection (P14–P16b).

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites

- [ ] site_brief.md provided by operator
- [ ] CDP browser available and functional

## Write Targets

| What | File | Why |
|------|------|-----|
| Raw observations (REQUEST, DOM_SNAPSHOT, COOKIE, LOCAL_STORAGE, EDGE_CASE_TEST, SERVICE_WORKER, UNKNOWN, operational SYSTEM) | `g1d0.log` | Gate-scoped raw data |
| D2:State updates | `state.log` | State checkpoint |
| D1: Baseline Phase Summary | `state.log` | Phase completion record |
| BUDGET_STATUS (at P8) | `state.log` | Budget checkpoint |
| COOKIE_DEPENDENCY_MAP (at P7) | `state.log` | Key synthesis artifact |
| consent_flow_map (at P7c, EU only) | `state.log` | Key synthesis artifact |

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 0 | Pre-Brief → Pre-P2 | Brief ingestion + environment validation |
| Phase 1 | P1 → P8a | Baseline — what IS this site? |
| Phase 2 | P9 → P13c | Content discovery — how is content structured? (see `references/gates/gate-2-pagination.md`) |

**Write gate at P8.** All pending observations must be logged, D2:State updated, D1 Phase Summary written, BUDGET_STATUS written, before proceeding to Gate 2.

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
  max_cycles:         <value or default 60>
  page_limit:         <value or default 15>
  geo_requirements:   <list or NONE>
```

This entry becomes the reference point for P16b verification. If a target_field or question is not in this entry, P16b cannot verify it was answered.

**Note:** The Pre-Brief entry captures the operational extraction — the fields and questions the agent needs to act on. This is NOT a copy of the full site_brief.md. P16b re-reads site_brief.md directly for verification.

**Geo-awareness (conditional):** If `geo_requirements` includes `EU` (or if the target URL is a European domain and no geo_requirements field exists in site_brief.md): Flag for P7c (consent flow mapping) — see P7c for the full procedure. Increase `max_cycles` by 2. Log the adjusted value in the Pre-Brief entry.

**Does NOT consume a decision cycle.**

---

### [Pre-P0] Read site_brief.md known_technology field

Before anything else, read the `known_technology` list from `site_brief.md`. This is the single most impactful optimization you can make — it lets you skip entire investigation branches and focus effort where it matters.

**Adjust your investigation approach based on what's known:**

| Known Technology | Adjustment |
|---|---|
| "Next.js App Router" | Prioritize P5a (RSC detection). Skip the full SSR vs CSR comparison — you already know it's RSC. |
| Specific framework (React, Vue, etc.) | Skip framework fingerprinting. Focus on API and content structure instead. |
| "GraphQL API" | Prioritize P21 (→ gate-4-exploration.md). You already know the API paradigm; focus on documenting operations. |
| "Akamai Bot Manager" | Increase initial delay to 3 seconds. Expect a potential BLOCKER. |
| CMS provider (Sanity, Contentful, etc.) | Look for CMS-specific API patterns during P11 (e.g., Sanity's `_type`/`_key`/`_ref`, Contentful's `sys.type`/`sys.id`). |
| Modern bundler with hashed filenames | Prioritize P19 (→ gate-4-exploration.md) — the bundle names may contain route hints. |
| "SvelteKit" | Flag for SvelteKit fetch binding (see `references/cdp-infrastructure.md` §SvelteKit Fetch Capture). Skip JS-level fetch interception; rely on CDP Network.requestWillBeSent instead. Prioritize JS bundle analysis for request body structure. |

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

Extraction paths follow the taxonomy defined in `references/log-format.md §Extraction Path Taxonomy`.

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
- **Closed mode:** Use CDP DOM piercing to access closed shadow root content (see procedure below). Do NOT log as UNKNOWN — CDP can pierce closed shadow roots.
- **Critical note:** `querySelector` CANNOT reach inside shadow roots. You must use `.shadowRoot.querySelector()` for open roots. Closed roots are accessed via CDP `DOM.getDocument { pierce: true }`.

**CDP piercing procedure for closed shadow roots:**

When the TreeWalker scan finds shadow hosts with `mode: "closed"`, JavaScript cannot access their content. But CDP's `DOM` domain can pierce all shadow boundaries including closed ones. Use this procedure:

1. **Log SYSTEM entry** with event `cdp_dom_enabled`, description "CDP.DOM enabled for closed shadow DOM piercing — N closed root(s) detected"
2. **Enable CDP.DOM:** `CDP.DOM.enable()`
3. **Get full DOM tree with piercing:** `CDP.DOM.getDocument({ depth: -1, pierce: true })`
4. **Locate closed shadow roots** in the returned DOM tree — look for nodes with `shadowRoots` array containing entries with `type: "shadow-root"`
5. **For each closed shadow root:**
   - Capture its content via `CDP.DOM.getOuterHTML({ nodeId: <shadowRootNodeId> })`
   - Log as DOM_SNAPSHOT with context `shadow_dom_closed`
   - Include: host element selector, shadow root mode (`closed`), child count, text content sample
   - Classify extraction paths for any content fields found inside (same taxonomy as P3)
6. **Disable CDP.DOM:** `CDP.DOM.disable()`
7. **Log SYSTEM entry** with event `cdp_dom_disabled`, description "CDP.DOM disabled after shadow DOM capture — N closed root(s) pierced"

**Budget:** 1 decision cycle for the CDP piercing procedure (open roots are free — JS `.shadowRoot.querySelector()` works).

**Buffer safety:** CDP.DOM is enabled for ~30 seconds only. The SYSTEM entries logging enable/disable times let the analyst identify the window where DOM mutation events may appear in the capture. See `references/cdp-infrastructure.md` §1 for full rationale.

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

Extraction paths follow the taxonomy defined in `references/log-format.md §Extraction Path Taxonomy`.

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

**Why this matters:** Preconnect domains become your P17 probing targets (→ gate-4-exploration.md). CSP `connect-src` reveals which API domains are expected. CORS headers determine replay feasibility. Getting these from the right source (CDP headers, not DOM) matters because they can differ.

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

Cookies carry session state, authentication tokens, consent preferences, and tracking identifiers. Understanding the full cookie picture is essential for request replay (Phase 5 → gate-5-replay.md) — if you don't know which cookies are required, you can't construct a valid replay.

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

**Why this matters:** `robots.txt` Disallow paths are essentially a map of the site's sensitive areas (admin, API, internal tools). These are paths you should skip during P17 probing (→ gate-4-exploration.md). Sitemap URLs reveal the full content taxonomy — you'll use this in P8a.

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

## Gate 1 Output

Before proceeding to Gate 2, verify:

☐ CDP capture validated and working
☐ Full DOM snapshot taken with extraction path types classified
☐ Window globals inspected
☐ Embedded JSON/ld+json extracted with extraction map
☐ Rendering classified (SSR/CSR/RSC/hybrid)
☐ Head section analyzed (preconnect domains, CSP, CORS)
☐ SPA route detection completed
☐ All cookies logged with origin tracking
☐ COOKIE_DEPENDENCY_MAP written
☐ LocalStorage captured
☐ Consent flow mapping completed (if EU site)
☐ robots.txt and sitemap.xml parsed
☐ Sitemap URL patterns classified
☐ D2:State updated
☐ D1: Baseline Phase Summary written
☐ BUDGET_STATUS written
☐ Re-read `references/gates/gate-2-pagination.md` BEFORE writing first entry of Gate 2
