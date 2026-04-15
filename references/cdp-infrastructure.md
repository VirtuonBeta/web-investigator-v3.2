---
version: "3.2"
type: on_demand_reference
read_when:
  - gate 1 Phase 0 setup
  - CDP errors during investigation
phase_reference: "setup steps in gate-1-baseline.md; CDP data usage distributed across gate files"
---

# CDP Infrastructure `v3.2`

---

## CDP Domains

```yaml
enable_at_startup:
  CDP.Network:
    maxTotalBufferSize: 100_000_000   # 100MB
    maxResourceBufferSize: 50_000_000  # 50MB per resource
  CDP.Runtime: {}
  CDP.Console: {}
  CDP.Page: {}

do_not_enable_at_startup: CDP.DOM  # fires event per DOM mutation; floods buffer on dynamic sites
```

### Shadow DOM Piercing (Conditional)

```yaml
trigger: "P3a detects closed shadow roots that JS cannot access"
procedure:
  - enable: CDP.DOM.enable()
  - log SYSTEM: event "cdp_dom_enabled", description "CDP.DOM enabled for closed shadow DOM piercing"
  - run: CDP.DOM.getDocument({ depth: -1, pierce: true })
  - for each closed shadow root found:
      - snapshot: CDP.DOM.getOuterHTML({ nodeId })
      - log: DOM_SNAPSHOT with context "shadow_dom_closed"
  - disable: CDP.DOM.disable()
  - log SYSTEM: event "cdp_dom_disabled", description "CDP.DOM disabled after shadow DOM capture"
constraint: "disable immediately after capture; DOM events share 2000-entry buffer with Network events"
```

---

## Anti-Detection

```yaml
anti_detection_flags: enabled  # webdriver hidden, automation flags suppressed
observable: false  # investigator cannot observe checks for navigator.webdriver, Chrome automation, chrome.runtime
implication: "if target site relies on these checks, they are not observable"
```

---

## Capture Filter

```yaml
capture:
  resource_types: [xhr, fetch, document, websocket]
  include_domains:
    - target domain + all subdomains (auto-detected from target_url)
    - first-party CDN assets matching target CDN pattern (auto-detected from first page load)
    - domains in <link rel="preconnect"> tags
    - domains in CSP headers (connect-src directive)

exclude_patterns:
  - "*google-analytics*"
  - "*googletagmanager*"
  - "*doubleclick*"
  - "*googlesyndication*"
  - "*facebook.com/tr*"
  - "*taboola*"
  - "*outbrain*"
  - "*scorecardresearch*"
  - "*amazon-adsystem*"
  - "*criteo*"
  - "*rubiconproject*"
  - "*pubmatic*"
  - "*bidswitch*"
  - "*casalemedia*"
  - "other known ad/tracking/analytic services"

auto_extend: "log each new tracking domain as SYSTEM entry, add to exclude list"
```

---

## CDP Request Fields

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

## SvelteKit Fetch Capture

```yaml
detection: "data-svelte-h attributes or _app/immutable/ asset paths"
limitation: "binds window.fetch at module import time before monkey-patching; standard fetch interception fails"
workaround:
  - use CDP Network.requestWillBeSent events instead of JS-level interception
  - OR reverse-engineer request bodies from JS bundle analysis
```

---

## DOM Snapshot Rules

```yaml
limits:
  max_depth: 15 levels from document.body
  max_nodes: 500 per snapshot
  max_text_length: 200 chars per node

attributes_to_log: [class, id, href, src, "data-*", type, role, "aria-*", rel]

skip_content_tag_only: [script, style, svg, noscript]

skip_element_class_contains: [ad, banner, footer, cookie, popup]  # log existence only
```

### Element Classification

```yaml
priority_order:
  1. semantic_html: "tag in [ARTICLE, MAIN, NAV, HEADER, FOOTER, SECTION, ASIDE, TIME]"
  2. aria_role: "has role attribute"
  3. data_attribute: "has data-testid or data-cy"
  4. class_semantic: "classes are human-readable"
  5. class_hashed: "classes match hashed pattern"

hashed_pattern: "/^[a-z]{1,3}-[a-z0-9]{4,}$|^_[a-z0-9]{4,}$|^css-[a-z0-9]+$/"

class_type_rule:
  if: "hashed classes > 50% of all classes"
  then: class_type = hashed
  else_if: "no classes"
  then: class_type = none
  else: class_type = semantic
```

---

## Capture Volume Management

```yaml
max_cdp_entries: 2000

approaching_limit:
  action: "switch CDP filter from CAPTURE_EVERYTHING to FILTERED"
  filtered_captures:
    - requests to target domain and API subdomains
    - requests with non-2xx status codes
    - WebSocket connections
    - requests matching URL patterns discovered during investigation
  log: SYSTEM entry with event "CDP_FILTER_SWITCH"

limit_reached:
  action: "stop CDP passive capture entirely"
  log: SYSTEM entry "CDP_CAPTURE_LIMIT_REACHED: {N} entries captured"
  fallback: "continue with agent_active observations only"
```

---

## DOM Snapshot Budget

```yaml
max_per_investigation: 30

after_limit_only_log_for:
  - content item entry pages (always)
  - operator-requested DOM comparison
  - explicit requests in s2_gaps.md

alternative_after_limit: "targeted JS queries instead of full snapshots"
```

---

## robots.txt UA Blocking

```yaml
when: "after parsing robots.txt, BEFORE any raw HTTP requests"
procedure:
  - extract ALL User-agent rules (not just wildcard *)
  - for EACH rule with Disallow: / (site-wide block):
      - check if investigator raw HTTP User-Agent matches (case-insensitive substring)
      - if match:
          - log SYSTEM: event "UA_BLOCKED_BY_ROBOTS_TXT"
          - set flag: raw_http_ua_blocked = true
          - for subsequent raw HTTP probes (P13, P17, P23-P27, P28-P29):
              - use browser-like User-Agent
              - OR skip and log as DELIBERATE_VIOLATION
```

---

## CDN Detection

```yaml
check_headers: [CF-Ray, X-Cache, Age, Via, X-Served-By]
if_detected:
  - log provider and cache behavior
  - test conditional requests
  - log Vary header (tells what dimensions change cached response)
  - if CDN caches based on Cookie: scraper must send correct cookies
```

---

## Request Fingerprinting

```yaml
types: [TLS/JA3/JA4, HTTP/2, header_ordering, cookie_presence, header_count_anomalies]

detection: "raw HTTP replay fails while browser succeeds → P13b active probe"
classification: EDGE_CASE_TEST with fingerprint_type: "header" | "tls" | "none"
  header_based: "solvable with correct headers"
  tls_based: "requires curl-impersonate or similar"

rule: "do NOT log as UNKNOWN with hypothesis; use P13b active probe"
```

---

## Detection Procedures

### A/B Testing

```yaml
when: "second visit to same URL"
check: "compare DOM snapshot to first visit"
look_for: [different layout, different selectors, different content ordering]
if_found: "log EDGE_CASE_TEST with test_id: a_b_compare"
common_platforms: [Optimizely, VWO, Google Optimize]
```

### Cross-Origin iframe

```yaml
when: "DOM snapshots"
check: "<iframe> elements pointing to different origins"
if_found: "log EDGE_CASE_TEST CROSS_ORIGIN_IFRAME_TEST"
note: "CDP Network domain WILL still capture iframe network requests"
if_primary_content_in_iframe: "log UNKNOWN hypothesis"
```

### Service Worker

```yaml
when: "after initial page load"
execute_js: "navigator.serviceWorker.getRegistrations()"
log: [scope, script URL, state]
if_intercepts_fetch: "log UNKNOWN hypothesis about cached responses"
if_scope_matches_target: "test clearing SW, log behavior change"
```

### Internationalization

```yaml
when: "during baseline"
check: ["<html lang>", hreflang tags, Accept-Language influence, URL path prefixes, Content-Language header]
if_multi_language: "test different Accept-Language, log EDGE_CASE_TEST I18N_TEST"
```
