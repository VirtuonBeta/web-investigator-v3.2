# CDP Infrastructure Setup

Reference file for the Web Investigator (Agent 1 v3.2). CDP passive capture layer configuration and management.

**Version:** 3.2

Read this during Phase 0 setup and when CDP issues arise.

**Phase reference:** Setup steps (Pre-P1) are in `references/gates/gate-1-baseline.md`. Steps that use CDP data are distributed across gate files (see SKILL.md → Reference Files).

---

## 1. CDP Domains to Enable

```
CDP.Network.enable({
  maxTotalBufferSize: 100_000_000,   // 100MB buffer for large pages
  maxResourceBufferSize: 50_000_000, // 50MB per resource
})

CDP.Runtime.enable()
CDP.Console.enable()
CDP.Page.enable()
```

**Do NOT enable `CDP.DOM.enable()` at startup.** This domain fires a CDP event for every DOM mutation (child added, attribute changed, text updated). On dynamic sites, this floods the CDP buffer with thousands of events per second, pushing out important Network captures.

### Conditional CDP.DOM Enable (for Shadow DOM piercing only)

Enable `CDP.DOM` only when P3a detects **closed shadow roots** that JavaScript cannot access. Disable it immediately after capturing the shadow DOM content.

**Procedure:**

```
1. Enable: CDP.DOM.enable()
2. Log SYSTEM entry: event "cdp_dom_enabled", description "CDP.DOM enabled for closed shadow DOM piercing"
3. Run: CDP.DOM.getDocument({ depth: -1, pierce: true })
   - This returns the full DOM tree, piercing ALL shadow roots including closed ones
4. For each closed shadow root found:
   - Snapshot its contents via CDP.DOM.getOuterHTML({ nodeId })
   - Log as DOM_SNAPSHOT with context "shadow_dom_closed"
5. Disable: CDP.DOM.disable()
6. Log SYSTEM entry: event "cdp_dom_disabled", description "CDP.DOM disabled after shadow DOM capture"
```

**Why conditional:** CDP.DOM events consume the same 2000-entry buffer as Network events. Leaving it enabled would cause premature filter switches and potentially miss API calls and pagination endpoints. The ~30 seconds of DOM events from a targeted piercing is negligible; continuous DOM event streaming is not.

**Buffer impact:** During the ~30 seconds CDP.DOM is enabled, some DOM mutation events may appear in the capture. The SYSTEM entries logging enable/disable times let the analyst identify this window and discount any DOM mutation events that appear during it.

---

## 2. Anti-Detection Note

The CDP browser used for investigation has anti-detection flags enabled (webdriver hidden, automation flags suppressed). These signals will **NOT** appear in the log — the investigator cannot observe checks for `navigator.webdriver`, Chrome automation flags, or `chrome.runtime`. If the target site relies on these checks, they are not observable by this investigator.

---

## 3. Capture Filter

```
CAPTURE EVERYTHING:
  - Resource types: xhr, fetch, document, websocket

CAPTURE URLS MATCHING THESE DOMAINS:
  - The target domain and all its subdomains (auto-detected from target_url)
  - First-party CDN assets matching the target's CDN pattern (auto-detected from first page load)
  - Any domain referenced in <link rel="preconnect"> tags on the target page
  - Any domain referenced in CSP headers (connect-src directive)

EXCLUDE URLS MATCHING THESE PATTERNS:
  - *google-analytics*
  - *googletagmanager*
  - *doubleclick*
  - *googlesyndication*
  - *facebook.com/tr*
  - *taboola*
  - *outbrain*
  - *scorecardresearch*
  - *amazon-adsystem*
  - *criteo*
  - *rubiconproject*
  - *pubmatic*
  - *bidswitch*
  - *casalemedia*
  - Other patterns matching known ad/tracking/analytic services

NOTE: The exclude list is extended automatically when you encounter new
tracking domains during the investigation. Log each auto-exclusion as a
SYSTEM entry.
```

---

## 4. What Gets Logged Per Request

Every request passing the filter is logged as a `REQUEST` entry per log-format.md. Key fields captured:

| Field | Description |
|---|---|
| `source` | `cdp_passive` |
| `url` | Full URL with all query parameters |
| `method` | HTTP method |
| `resource_type` | `xhr` \| `fetch` \| `document` \| `websocket` |
| `req_headers` | All headers (redact cookie values to first 20 chars) |
| `res_status` | HTTP status code |
| `res_headers` | All response headers |
| `res_body_type` | Per log-format.md body rules |
| `res_body_sample` | Per log-format.md body rules |
| `res_body_schema` | Per log-format.md body rules |
| `res_size_bytes` | Response body size |
| `ttfb_ms` | Time to first byte |
| `redirect_chain` | Array of `{url, status}` for every hop |
| `trigger` | Auto-detected: `page_load` \| `scroll` \| `click` \| `timer` \| `websocket_message` \| `unknown` |

---

## 4b. SvelteKit Fetch Capture Limitation

SvelteKit (detected by `data-svelte-h` attributes or `_app/immutable/` asset paths) binds `window.fetch` at module import time, before any runtime monkey-patching can take effect. Standard fetch interception (overriding `window.fetch`) will NOT capture SvelteKit network requests.

For SvelteKit sites, use CDP `Network.requestWillBeSent` events instead of JS-level interception, or reverse-engineer request bodies from JS bundle analysis. This limitation cost ~4-5 decision cycles on a Yahoo Finance investigation before the agent discovered it.

---

## 5. DOM Snapshot Rules

```
Max depth:              15 levels from document.body
Max nodes logged:       500 per snapshot
Max text length:        200 chars per node

Element attributes to log:
  class, id, href, src, data-*, type, role, aria-*, rel

Skip content (log tag existence only):
  <script>, <style>, <svg>, <noscript>

Skip elements with class containing:
  'ad', 'banner', 'footer', 'cookie', 'popup'
  (log element existence only)

Element classification during snapshots:

When taking a DOM snapshot, classify each significant element by its extraction path type. This classification feeds into the `extraction_map` field in DOM_SNAPSHOT entries and the extraction stability taxonomy used throughout the investigation.

```js
// Run after DOM snapshot to classify elements
function classifyElement(el) {
  const classes = Array.from(el.classList || []);
  const hashedPattern = /^[a-z]{1,3}-[a-z0-9]{4,}$|^_[a-z0-9]{4,}$|^css-[a-z0-9]+$/;
  const hasTestId = el.hasAttribute('data-testid') || el.hasAttribute('data-cy');
  const hasAriaRole = el.hasAttribute('role');
  const isSemantic = ['ARTICLE','MAIN','NAV','HEADER','FOOTER','SECTION','ASIDE','TIME'].includes(el.tagName);

  return {
    tag: el.tagName,
    class_type: classes.filter(c => hashedPattern.test(c)).length > classes.length * 0.5 ? 'hashed' :
                classes.length === 0 ? 'none' : 'semantic',
    best_extraction_type: isSemantic ? 'semantic_html' :
                          hasAriaRole ? 'aria_role' :
                          hasTestId ? 'data_attribute' :
                          classes.filter(c => hashedPattern.test(c)).length > 0 ? 'class_hashed' :
                          'class_semantic'
  };
}
```

---

## 6. CDP Capture Volume Management

```
MAX_CDP_ENTRIES: 2000 (total REQUEST entries from CDP passive capture)
```

**When approaching the limit:**

- Switch CDP filter from `CAPTURE_EVERYTHING` to `FILTERED` mode
- `FILTERED` mode captures only:
  - Requests to target domain and API subdomains
  - Requests with non-2xx status codes
  - WebSocket connections
  - Requests matching URL patterns discovered during investigation
- Log the filter switch as `SYSTEM` entry with event `"CDP_FILTER_SWITCH"`

**When the limit is reached:**

- Stop CDP passive capture entirely
- Log `SYSTEM` entry: `"CDP_CAPTURE_LIMIT_REACHED: {N} entries captured"`
- Continue investigation using `agent_active` observations only

---

## 7. DOM Snapshot Budget

```
MAX_DOM_SNAPSHOTS: 30 per investigation
```

**After 30 snapshots, only log DOM snapshots for:**

- Content item entry pages (always log these)
- Specific DOM comparison requested by operator
- Any DOM snapshot explicitly requested in `s2_gaps.md`

For other observations, use targeted JS queries instead of full snapshots.

---

## 8. robots.txt UA-Specific Blocking Check

Some sites use UA-specific rules in robots.txt to block specific bots site-wide.

```
After parsing robots.txt:
  1. Extract ALL User-agent rules (not just the wildcard *)
  2. For EACH User-agent rule that has Disallow: / (site-wide block):
     - Check if the investigator's raw HTTP User-Agent matches (case-insensitive substring)
     - If match found:
       - Log SYSTEM entry with event "UA_BLOCKED_BY_ROBOTS_TXT"
       - Set flag: raw_http_ua_blocked = true
       - For all subsequent raw HTTP probes (P13, P17, P23-P27, P28-P29):
         - Use a browser-like User-Agent instead
         - OR skip the request and log as DELIBERATE_VIOLATION
  3. This check must complete BEFORE any raw HTTP requests are made
```

---

## 9. Content Delivery Architecture Detection

During investigation, distinguish between CDN edge and origin:

- **Check for CDN headers:** `CF-Ray`, `X-Cache`, `Age`, `Via`, `X-Served-By`
- **If CDN detected:** log provider and cache behavior, test conditional requests
- **Log the `Vary` header** — tells what dimensions change the cached response
- **Matters for scraper:** if CDN caches based on `Cookie`, scraper must send correct cookies

---

## 10. Request Fingerprinting Detection

Some sites fingerprint requests to detect automation:

- TLS fingerprinting (JA3/JA4)
- HTTP/2 fingerprinting
- Header ordering
- Cookie presence/absence
- Header count anomalies

**Detection and classification:** When raw HTTP replay fails while browser succeeds, P13b (in `references/gates/gate-2-pagination.md`) runs an active probe to determine whether the fingerprinting is header-based (solvable with correct headers) or TLS-based (requires TLS impersonation tools like `curl-impersonate`). P13b logs the result as an EDGE_CASE_TEST with `fingerprint_type: "header" | "tls" | "none"`.

**Do NOT log fingerprinting as UNKNOWN with a hypothesis** — use the P13b active probe instead.

---

## 11. A/B Testing Detection

After second visit to same URL:

- Compare DOM snapshot to first visit
- **Look for:** different layout, selectors, content ordering
- **If differences found:** log `EDGE_CASE_TEST` with `test_id: "a_b_compare"`
- **Common platforms:** Optimizely, VWO, Google Optimize

---

## 12. Cross-Origin iframe Detection

During DOM snapshots, check for `<iframe>` elements pointing to different origins:

- Log `EDGE_CASE_TEST` `"CROSS_ORIGIN_IFRAME_TEST"`
- CDP Network domain **WILL** still capture iframe network requests
- If primary content appears inside cross-origin iframe: log `UNKNOWN` hypothesis

---

## 13. Service Worker Detection

After initial page load:

```
Execute JS: navigator.serviceWorker.getRegistrations()
Log: scope, script URL, state
```

- If service worker intercepts fetch: log `UNKNOWN` hypothesis about cached responses
- If SW scope matches target: test clearing SW and log behavior change

---

## 14. Internationalization Detection

During baseline:

- **Check:** `<html lang>`, `hreflang` tags, `Accept-Language` influence, URL path prefixes, `Content-Language` header
- **If multi-language detected:** test different `Accept-Language`, log `EDGE_CASE_TEST` `"I18N_TEST"`
