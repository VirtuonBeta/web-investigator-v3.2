# Gate 1: Baseline (P0–P8a)

```yaml
gate_id: 1
title: Baseline
steps: Pre-Brief → P8a
phases: [0, 1]
d0_file: g1d0.log
operator_halt: false
next_gate: references/gates/gate-2-pagination.md
```

Read this file first, before any investigation step. This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] site_brief.md provided by operator
- [ ] CDP browser available and functional

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries incl. BUDGET_STATUS, COOKIE_DEPENDENCY_MAP, consent_flow_map) | `g1d0.log` |
| D2:State updates | `state.log` |
| D1: Baseline Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 0 | Pre-Brief → Pre-P2 | Brief ingestion + environment validation |
| 1 | P1 → P8a | Baseline — what IS this site? |
| 2 | P9 → P13c | Content discovery (→ gate-2-pagination.md) |

**Write gate at P8.** All pending observations logged, D2:State updated, D1 Phase Summary written, before proceeding to Gate 2.

---

## Phase 0: Pre-Flight

None of these consume a decision cycle.

---

### [Pre-Brief] Read site_brief.md in full

```yaml
step: Pre-Brief
cycle: false
condition: ALWAYS
```

Read the entire site_brief.md before any other action.

**Log:** SYSTEM entry:
```yaml
event: custom
description: "site_brief read"
details:
  target_url: <url>
  content_type: <type>
  target_fields: [<list all fields from target_data>]
  open_questions: [<list all questions from questions field>]
  known_issues: [<list>]
  known_technology: [<list>]
  auth_required: <bool>
  max_cycles: <value or default 60>
  page_limit: <value or default 15>
  geo_requirements: <list or NONE>
```

This entry is the reference point for P16b verification — if a target_field or question is not in this entry, P16b cannot verify it was answered. This captures the operational extraction, NOT a copy of the full site_brief.md. P16b re-reads site_brief.md directly for verification.

**Geo-awareness (conditional):**

| Condition | Action |
|-----------|--------|
| `geo_requirements` includes `EU` | Flag P7c (consent flow mapping). Increase `max_cycles` by 2. Log adjusted value in Pre-Brief entry. |
| Target URL is European domain AND no `geo_requirements` field | Same as above — flag P7c, +2 cycles. |
| Neither | No action. |

---

### [Pre-P0] Read known_technology and adjust investigation

```yaml
step: Pre-P0
cycle: false
condition: ALWAYS
```

Read the `known_technology` list from site_brief.md. Adjust investigation approach:

| Known Technology | Adjustment |
|-----------------|------------|
| "Next.js App Router" | Prioritize P5a (RSC detection). Skip full SSR vs CSR comparison — already known RSC. |
| Specific framework (React, Vue, etc.) | Skip framework fingerprinting. Focus on API and content structure. |
| "GraphQL API" | Prioritize P21 (→ gate-4). Already know the API paradigm; focus on documenting operations. |
| "Akamai Bot Manager" | Increase initial delay to 3 seconds. Expect potential BLOCKER. |
| CMS provider (Sanity, Contentful, etc.) | Look for CMS-specific API patterns during P11 (e.g., Sanity `_type`/`_key`/`_ref`, Contentful `sys.type`/`sys.id`). |
| Modern bundler with hashed filenames | Prioritize P19 (→ gate-4) — bundle names may contain route hints. |
| "SvelteKit" | Flag for SvelteKit fetch binding (→ cdp-infrastructure.md §SvelteKit Fetch Capture). Skip JS-level fetch interception; rely on CDP Network.requestWillBeSent. Prioritize JS bundle analysis for request body structure. |

**Log:** SYSTEM entry with `known_technology_adjustment: {adjustments}`.

---

### [Pre-P1] CDP Health Validation

```yaml
step: Pre-P1
cycle: false
condition: ALWAYS
log: { type: SYSTEM, event: cdp_health_check }
```

**Procedure:**
1. After enabling CDP domains, navigate to `about:blank` or a minimal test page
2. Make one explicit `fetch` request (e.g., `fetch('https://httpbin.org/get')`)
3. Verify the request appears in CDP capture

| Outcome | Action |
|---------|--------|
| NO requests captured | CDP broken. Log as **BLOCKER**. Do not proceed — network observations are untrustworthy. |
| Requests captured | CDP working. Proceed to Pre-P2. |

---

### [Pre-P2] Warm-Up Request

```yaml
step: Pre-P2
cycle: false
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: WARM_UP_COMPARE }
```

**Procedure:**
1. Navigate to `target_url`, wait for network idle
2. **IMMEDIATELY** navigate away (e.g., to `about:blank`)
3. Discard all observations from this load

First load primes CDN caches and TLS sessions, producing skewed data. Discard it. Then proceed to P2 (the real investigation load).

---

## Phase 1: Baseline (~8 cycles)

Captures the fundamental nature of the page: what loads, how it renders, what data is embedded, what the server reveals. Everything in later phases builds on these observations.

---

### [P1] Enable CDP capture

```yaml
step: P1
cycle: false
condition: ALWAYS
log: null
```

Enable the CDP Network, Runtime, Console, and Page domains. See `references/cdp-infrastructure.md` for full domain enablement details and error handling.

---

### [P2] Navigate to target_url

```yaml
step: P2
cycle: true
condition: ALWAYS
log: null  # observations captured by CDP and logged in subsequent steps
```

**Procedure:**
- Navigate to `target_url`
- Wait for network idle: 500ms with ≤2 in-flight requests
- If navigation hangs beyond **15 seconds**: fall back to `DOMContentLoaded` event (some pages never reach full idle — long-polling, SSE, etc.)

**Edge cases:**

| Observation | Action |
|------------|--------|
| Consent/gate wall detected | Handle before proceeding (→ SKILL.md §Consent Walls). Can block access to real content entirely. |
| Redirected to completely different domain | Log SYSTEM `unexpected_domain_redirect`. May indicate geo-routing, bot detection, or misconfigured target. |

**Captures:** All page load requests, cookies, DOM state.

---

### [P3] Full DOM snapshot

```yaml
step: P3
cycle: true
condition: ALWAYS
log: { type: DOM_SNAPSHOT, context: initial_load }
```

Take a complete DOM snapshot of the initial page state. This is the reference point for the entire investigation — compare against it to detect dynamic changes, lazy-loaded content, and structural patterns.

**Capture:** Article count, root selectors, card selectors, embedded JSON blocks.

**Extraction path type tagging:**

Extraction paths follow the taxonomy in `references/log-format.md §Extraction Path Taxonomy`. Run this JS after snapshot to classify each field's best available path:

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

Include `extraction_path_types` array and `class_type` field in the DOM_SNAPSHOT entry. For each identified field, note its best available path type.

**Feeds into:** P5 (field mapping), P15 (detail page mapping), P22 (stability matrix).

---

### [P3a] Shadow DOM detection

```yaml
step: P3a
cycle: true  # 1 cycle for CDP piercing procedure (open roots are free)
condition: ALWAYS
```

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

**If `shadowHosts.length > 0`:**

| Mode | Access Method | Log |
|------|--------------|-----|
| Open | `.shadowRoot.querySelector()` | EDGE_CASE_TEST `SHADOW_DOM_DETECTED` + DOM_SNAPSHOT `shadow_dom_content` |
| Closed | CDP `DOM.getDocument { pierce: true }` (see procedure below) | EDGE_CASE_TEST `SHADOW_DOM_DETECTED` + DOM_SNAPSHOT `shadow_dom_closed` |

For each shadow host, record: tag, selector, mode (open/closed), child count.

`querySelector` CANNOT reach inside shadow roots — use `.shadowRoot.querySelector()` for open, CDP for closed. Do NOT log closed roots as UNKNOWN — CDP can pierce them.

**CDP piercing procedure for closed shadow roots:**

1. **Log SYSTEM** event `cdp_dom_enabled`, description "CDP.DOM enabled for closed shadow DOM piercing — N closed root(s) detected"
2. **Enable:** `CDP.DOM.enable()`
3. **Get full DOM tree:** `CDP.DOM.getDocument({ depth: -1, pierce: true })`
4. **Locate closed shadow roots** — nodes with `shadowRoots` array containing `{ type: "shadow-root" }`
5. **For each closed shadow root:**
   - Capture via `CDP.DOM.getOuterHTML({ nodeId: <shadowRootNodeId> })`
   - Log as DOM_SNAPSHOT context `shadow_dom_closed`
   - Include: host element selector, shadow root mode (`closed`), child count, text content sample
   - Classify extraction paths for content fields found inside (same taxonomy as P3)
6. **Disable:** `CDP.DOM.disable()`
7. **Log SYSTEM** event `cdp_dom_disabled`, description "CDP.DOM disabled after shadow DOM capture — N closed root(s) pierced"

**Buffer safety:** CDP.DOM enabled for ~30 seconds only. SYSTEM entries logging enable/disable times let the analyst identify the window where DOM mutation events may appear in capture. → cdp-infrastructure.md §1.

**If `shadowHosts.length === 0`:** No action needed.

---

### [P4] Window globals inspection

```yaml
step: P4
cycle: true
condition: ALWAYS
```

**Check for these known globals:**

| Global | Reveals |
|--------|---------|
| `window.__NEXT_DATA__` | Next.js Pages Router — full page props, routing, data |
| `self.__next_f` | Next.js App Router — RSC streaming payload (NOT `__NEXT_DATA__`) |
| `window.__NUXT__` | Nuxt.js — state and data |
| `window.YAHOO` | Legacy Yahoo framework |
| `window.CONFIG` | Generic site configuration |
| `window.dataLayer` | Google Tag Manager — event tracking, e-commerce data |
| `window.OnetrustActiveGroups` | OneTrust consent state — active cookie categories |

**Also check:** Site-specific naming patterns (e.g., `window.MYAPP_CONFIG`, `window.__SITE_DATA__`).

---

### [P5] Embedded JSON/scripts extraction

```yaml
step: P5
cycle: true
condition: ALWAYS
log: { type: DOM_SNAPSHOT, context: embedded_json_N, required_fields: [extraction_map] }
```

**Extract from:**

| Source | Typical Content |
|--------|----------------|
| `<script type="application/json">` | Framework hydration data |
| `<script type="application/ld+json">` | Schema.org structured data (article metadata, product info, organization data) |
| `<script id="...">` with JSON content | Site-specific embedded config or data |

**Field-to-extraction-path mapping:**

When ld+json or other structured data is found, map EVERY field to its JSON path. Extraction paths follow the taxonomy in `references/log-format.md §Extraction Path Taxonomy`.

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
3. **Cross-reference with P3 DOM snapshot** — for each field, identify the DOM element that renders it. Create a dual-path extraction map:
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

DOM_SNAPSHOT entry MUST include `extraction_map` field containing the complete field-to-path mapping (→ log-format.md).

**Feeds into:** P15 (detail page mapping), P22 (stability matrix). #1 most valuable output for the downstream analyst.

---

### [P5a] RSC (React Server Components) detection

```yaml
step: P5a
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: RSC_DETECTED }
```

**Check for:**

| Signal | Meaning |
|--------|---------|
| `self.__next_f` | RSC streaming buffer on window object |
| `self.__next_f.push` calls in page scripts | RSC payload chunks |
| `<script src="*/_next/static/chunks/*">` with RSC patterns | RSC-related script loading |

**If RSC detected:**

- Override SSR vs CSR classification → Classify as `"RSC"` — neither traditional SSR nor CSR
- RSC payloads ARE the embedded data source → Parse `self.__next_f.push()` calls to extract structured data
- For P9 content structure: Wait for RSC streaming to complete before counting items. Completion: `document.readyState === 'complete'` AND no new `push` calls in 2 seconds (max wait 10s)

**Feeds into:** P6 (rendering classification), P9 (content counting), P5 (data source selection).

---

### [P6] Raw HTML sample

```yaml
step: P6
cycle: true
condition: ALWAYS
```

**Execute JS:** First 5000 chars of `document.documentElement.outerHTML`.

**Classification logic:**

| Condition | render_type |
|-----------|-------------|
| P5a detected RSC | `RSC` |
| Zero content items in raw HTML | `CSR` (but check P5a first — RSC has HTML shell) |
| Some items but fewer than post-JS | `hybrid` (SSR shell + CSR hydration) |
| Same items in both raw and post-JS | `SSR` |

**Log:** The comparison counts — how many items in raw HTML, how many in post-JS DOM.

---

### [P6a] \<head\> section analysis

```yaml
step: P6a
cycle: true
condition: ALWAYS
log: { type: DOM_SNAPSHOT, context: head_analysis }
```

**Extract:**

| Source | What | Note |
|--------|------|------|
| `<link rel="preconnect" href="...">` | Third-party domains the page will contact | Log for capture filter |
| `<meta>` tags | CMS fingerprints, viewport, SEO config | Log names and content values |
| CDP response headers | CSP (Content-Security-Policy) | **NOT from DOM** — HTTP header is authoritative |
| CDP response headers | CORS headers | Determines replay feasibility outside browser |

**Feeds into:** P17 probing targets (preconnect domains, CSP connect-src), P13 replay (CORS).

---

### [P6b] SPA route detection

```yaml
step: P6b
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: SPA_ROUTE_TEST }
```

**Procedure:**
- Monitor `history.pushState` and `history.replaceState` calls
- Install `popstate` listener, override `pushState`/`replaceState`
- Detects SPA navigation without network page loads

**Feeds into:** All subsequent navigation steps — if SPA detected, inspect DOM instead of waiting for network on navigation.

---

### [P7] Cookie logging

```yaml
step: P7
cycle: true
condition: ALWAYS
```

**Procedure:**
1. Capture all cookies from page load (via CDP Network response headers)
2. After navigation, use `CDP.Network.getCookies()` to capture JS-set cookies (won't appear in response headers)
3. Compare against `Set-Cookie` headers to identify JS-only cookies

**Cookie origin tracking:**

1. During P2 (initial navigation), CDP captures all `Set-Cookie` response headers. Map each cookie to the URL that set it
2. After navigation, use `CDP.Network.getCookies()` to find JS-only cookies (no corresponding `Set-Cookie` header). Note as `set_by: javascript`

**Log each cookie with:** `set_by_request` — gate-qualified ID of the REQUEST that set this cookie (e.g., `g1:003`). Creates a linkable chain.

**After all cookies logged**, write a SYSTEM entry `COOKIE_DEPENDENCY_MAP`:
```
Step 1: GET / → sets cookies [A1, A1S, GUCS, A3]
Step 2: GET /consent → sets cookies [consentStatus, euconsent-v2]
Step 3: GET /api/data → requires cookies [A1, A1S] → sets cookie [apiSession]
```

This chain tells the analyser exactly which cookies to obtain and in what order. Without it: WHICH cookies exist is known, WHERE they come from and in what ORDER is not.

**Feeds into:** Phase 5 request replay (→ gate-5-replay.md), P18 token lifecycle.

---

### [P7b] LocalStorage capture

```yaml
step: P7b
cycle: true
condition: ALWAYS
log: { type: LOCAL_STORAGE }
```

**Execute JS:** `Object.keys(window.localStorage)`

For each key: record `value_sample` (first 30 chars only — don't log full values for privacy/size).

**Classify each key:**

| Category | Examples |
|----------|----------|
| Auth-related | tokens, session IDs, user identifiers |
| Content-related | cached data, user preferences, filters |
| Tracking/other | analytics IDs, A/B test assignments |

**Cross-reference with cookies:** If a key exists in BOTH localStorage and cookies, that redundancy signals importance — the site stores the same data in two places, usually critical for functionality.

---

### [P7c] Consent flow mapping (EU-only, conditional)

```yaml
step: P7c
cycle: true
budget: 2  # consent state check + content comparison; does NOT count toward P11 pagination budget
condition: Pre-Brief flagged geo_requirements: EU
log: { type: SYSTEM, event: consent_flow_map }
```

**If no EU flag from Pre-Brief:** Skip this step entirely. Log nothing.

**Procedure:**

**1. Detect consent platform:**

| Signal | Platform |
|--------|----------|
| `window.OnetrustActiveGroups` | OneTrust |
| `window.__tcfapi` or `window.__tcfapiLocator` | TCF v2 (IAB Europe) |
| `window.__cmp` or `window.__cmpLocator` | CMP (older IAB) |
| `window._sp_` | Sourcepoint |
| `window.consentData` or `window.UC_UI` | Usercentrics |
| Cookies containing `euconsent`, `eupubconsent`, `consentStatus` | Generic EU consent |

**2. Map consent categories** — for each detected platform:
- OneTrust: Parse `OnetrustActiveGroups` string (comma-separated category IDs like `C0001,C0003`)
- TCF v2: Call `__tcfapi('getTCData', 2, callback)` to get purpose consents
- Log: which categories exist, which are active, which are required vs optional

**3. Content availability test:**

| Phase | Action |
|-------|--------|
| Before consent | Note visible article text length, DOM structure, any truncated elements with `aria-hidden` or `display:none` on content areas |
| Click "Accept All" | Wait for consent handlers to complete (2 seconds) |
| After consent | Re-snapshot affected areas, note any content that appeared or expanded |

If consent banner was already handled in P2: this test still runs — check whether P2 consent state is "full consent" or "essential only".

**4. Map consent-dependent content** — for each content zone that changed:
- Which consent category controls it (e.g., "C0003 functional cookies" vs "C0004 targeting cookies")
- What content is gated vs always visible
- Whether the gating is client-side (CSS hide/show, DOM removal) or server-side (different response body)

**Log:** SYSTEM entry `consent_flow_map` containing: platform detected, categories and states, content zones affected by consent state, gating type (client-side/server-side).

---

### [P8] robots.txt and sitemap.xml

```yaml
step: P8
cycle: true
condition: ALWAYS
```

**Procedure:**
- Fetch via raw HTTP (if not already captured by CDP)
- Log full content per log format rules
- **Parse robots.txt:** Disallow paths, Allow paths, Crawl-delay, Sitemap directives
- Log parsed rules as SYSTEM entry

**Feeds into:** P17 probing (Disallow paths → skip these during probing, → gate-4), P8a (sitemap URLs for pattern classification).

---

### [P8a] Sitemap URL pattern classification

```yaml
step: P8a
cycle: true
condition: sitemap.xml found in P8
log: { type: SYSTEM, event: sitemap_classification }
```

**Procedure:**
1. Extract ALL URLs from sitemap.xml. Handle sitemap index files recursively (a sitemap can point to other sitemaps)
2. Classify URLs by pattern cluster:

| Pattern | Examples | Signal |
|---------|----------|--------|
| Content pages | `/articles/`, `/posts/`, `/products/`, `/items/`, `/listings/` | Core content structure |
| Category/tag | `/category/`, `/tag/`, `/topic/`, `/search?` | Taxonomy / navigation |
| API-like | `/api/`, `/v1/`, `/v2/`, `/graphql`, `/rest/` | Machine-accessible endpoints |
| Date-structured | `/2024/`, `/2024/01/`, `/archive/` | Temporal organization |
| Query-parameter | Any URL containing `?` | Deep web entry points |

3. For each cluster: record pattern, count, first 3 sample URLs, and `deep_web` flag
4. Log as SYSTEM entry `SITEMAP_CLASSIFICATION`

**Edge cases:**

| Observation | Action |
|------------|--------|
| Sitemap reveals URL patterns NOT visible from navigation | Log EDGE_CASE_TEST `SITEMAP_HIDDEN_STRUCTURE` — sitemap exposes content the UI doesn't link to |
| No sitemap, empty sitemap, or fetch failed | Skip this step |

---

## Gate 1 Output

Before proceeding to Gate 2, verify:

- [ ] CDP capture validated and working
- [ ] Full DOM snapshot taken with extraction path types classified
- [ ] Window globals inspected
- [ ] Embedded JSON/ld+json extracted with extraction map
- [ ] Rendering classified (SSR/CSR/RSC/hybrid)
- [ ] Head section analyzed (preconnect domains, CSP, CORS)
- [ ] SPA route detection completed
- [ ] All cookies logged with origin tracking
- [ ] COOKIE_DEPENDENCY_MAP written (to g1d0.log)
- [ ] LocalStorage captured
- [ ] Consent flow mapping completed (if EU site)
- [ ] robots.txt and sitemap.xml parsed
- [ ] Sitemap URL patterns classified
- [ ] D2:State updated
- [ ] D1: Baseline Phase Summary written
- [ ] BUDGET_STATUS written (to g1d0.log)
- [ ] Re-read `references/gates/gate-2-pagination.md` BEFORE writing first entry of Gate 2
