# CDP Infrastructure Setup

Reference file for the Web Investigator (Agent 1 v3.0). CDP passive capture layer configuration and management.

Read this during Phase 0 setup and when CDP issues arise.

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
- Mobile viewport comparison (if not already done)
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
       - For all subsequent raw HTTP probes (P13, P17, P23-P27, P29-P30):
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

**Detection:** if raw HTTP request gets **403** but same URL works in browser → likely TLS/header fingerprinting. Log as `UNKNOWN` with hypothesis.

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
