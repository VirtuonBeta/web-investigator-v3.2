# Log Format Specification `v3.2`

Reference file for the Web Investigator (Agent 1 v3.2). Defines the typed entry format and file structure for the investigation output.

Agent 1 writes these entries. Agent 2 reads them. Every entry must follow this spec exactly.

---

## File Structure

The investigation output is a set of files, not a single monolithic log:

```
state.log          ← D2:State + D1 summaries + BUDGET_STATUS + key SYSTEM artifacts
g1d0.log           ← Raw observations from Gate 1 (P0–P8a)
g2d0.log           ← Raw observations from Gate 2 (P9–P13c)
g3d0.log           ← Raw observations from Gate 3 (P14–P16b)
g4d0.log           ← Raw observations from Gate 4 (P17–P22)
g5d0.log           ← Raw observations from Gate 5 (P23–P27)
g6d0.log           ← Raw observations from Gate 6 (P28–P32+)
```

### state.log — State & Summaries

Append-only. Contains the D2/D1/BUDGET hierarchy and key synthesis SYSTEM entries. Never contains raw observation entries (REQUEST, DOM_SNAPSHOT, COOKIE, etc.).

**Write targets for state.log:**
- D2:State updates (at trigger points)
- D1 phase summaries (at gate boundaries)
- BUDGET_STATUS entries (at P8, P13, P16, and when budget exhausted)
- COOKIE_DEPENDENCY_MAP (at P7)
- HTTP_REQUEST_CHAIN (at P27)
- consent_flow_map (at P7c, EU only)
- INVESTIGATION_FIRST_PASS_COMPLETE (at P13c)
- Site brief field verification (at P16b)

### gNd0.log — Gate Raw Observations

Append-only per gate. Contains all typed observation entries (REQUEST, DOM_SNAPSHOT, COOKIE, LOCAL_STORAGE, EDGE_CASE_TEST, SERVICE_WORKER, UNKNOWN, SESSION, and operational SYSTEM entries). A gate D0 file is naturally frozen when the gate completes — no explicit freeze step needed.

---

## General Rules

1. Every entry is a typed block delimited by `━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`
2. Every entry starts with a UTC ISO 8601 timestamp and a TYPE declaration
3. Every entry has a unique `id` — format: `ent_NNN` (sequential) or `ent_auto_NNN` (CDP passive). IDs are global across all files — sequential numbering continues across state.log and gate D0 files.
4. Every entry has a `phase` indicating which priority queue step produced it
5. Source field: `source: cdp_passive | agent_active | system`
6. No free-form prose in data fields. Notes and descriptions are fine. Conclusions are not.
7. Append only. Never modify or delete entries in any file.
8. Truncation must be marked: `[TRUNCATED at N chars of total M]`
9. Raw observation entries go in the current gate's D0 file. State and synthesis entries go in state.log.

---

## Extraction Path Taxonomy

For each content item field, classify its extraction path type. This classification determines extraction stability — which selectors survive site redesigns and which break on every deploy.

| Path Type | Priority | Pattern | Survives Deploy? |
|-----------|----------|---------|-----------------|
| `structured_data` | 1 (best) | ld+json, __NEXT_DATA__, embedded JSON | Always |
| `semantic_html` | 2 | `<article>`, `<time datetime>`, `<h1>`, `<main>` | Mostly |
| `aria_role` | 2 | `[role="article"]`, `[role="heading"]` | Mostly |
| `data_attribute` | 3 | `[data-testid]`, `[data-cy]`, `[data-component]` | Often |
| `meta_content` | 3 | `<meta name="description">`, `<meta property="og:*">` | Usually |
| `class_semantic` | 4 | `.article-title`, `.post-body` | Sometimes |
| `class_hashed` | 5 (worst) | `.yf-1a2b3c`, `.css-xyz123`, `._abc123` | Never |

Fields with only `class_hashed` paths are flagged as `[brittle]`. Fields with NO column showing "universal" in the stability matrix are `stability_risk: HIGH`.

This taxonomy is referenced by: P3 (initial extraction tagging), P5 (field-to-path mapping), P15 (detail page mapping), P22 (stability matrix). All extraction path references in the gate files point to this canonical definition.

---

## Entry Types

### REQUEST

Captures every HTTP request/response observed or initiated.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:23:45.123Z  REQUEST

id:                   ent_004
phase:                1
source:               cdp_passive
method:               GET
url:                  https://example.com/api/v1/articles?page=2
trigger:              click
resource_type:        fetch
req_headers:          { "accept": "application/json", "cookie": "..." }
req_body:             null
res_status:           200
res_headers:          { "content-type": "application/json; charset=utf-8", "x-request-id": "abc123" }
res_body_type:        json
res_body_sample:      {"articles":[...],"total":42}
res_body_schema:      { type: "object", properties: { articles: { type: "array" }, total: { type: "number" } } }
res_body_truncated:   false
res_body_total_size:  null
res_size_bytes:       15420
ttlfb_ms:             187
redirect_chain:       []
anomalies:            []
notes:                Paginated API endpoint; returns full article list
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, method, url, res_status, res_body_type, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** trigger, resource_type, req_headers, req_body, res_headers, res_body_sample, res_body_schema, res_body_truncated, res_body_total_size, res_size_bytes, ttlfb_ms, redirect_chain, anomalies

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number (see Phase Numbering Convention) |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `method` | string | HTTP method: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD |
| `url` | string | Full URL |
| `trigger` | enum | `page_load` \| `scroll` \| `click` \| `manual` \| `rapid_fire_test` \| `edge_case` \| `api_probe` \| `pagination` \| `navigation` \| `unknown` |
| `resource_type` | string | CDP resource type: document, stylesheet, image, font, script, xhr, fetch, websocket, etc. |
| `req_headers` | object \| null | Relevant request headers (auth, cookies, accept, content-type) |
| `req_body` | string \| object \| null | Request body if present |
| `res_status` | integer | HTTP status code |
| `res_headers` | object | Relevant response headers |
| `res_body_type` | enum | `json` \| `html` \| `javascript` \| `css` \| `image` \| `font` \| `binary` \| `websocket` \| `unknown` |
| `res_body_sample` | string \| null | See Response Body Handling Rules |
| `res_body_schema` | object \| null | JSON Schema of response (always included for JSON, even if truncated) |
| `res_body_truncated` | boolean | `true` if sample is incomplete |
| `res_body_total_size` | integer \| null | Total response body size in chars (only when truncated) |
| `res_size_bytes` | integer | Total response size in bytes (headers + body) |
| `ttlfb_ms` | integer \| null | Time to first byte in milliseconds |
| `redirect_chain` | array | Array of `{ status, url }` hops. Main `url` field shows final destination. |
| `anomalies` | array | See Anomaly Field Convention. Empty `[]` if none. |
| `notes` | string | Factual observations only. No conclusions. |

---

### DOM_SNAPSHOT

Captures the state of the DOM at a specific point in the investigation.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:24:12.456Z  DOM_SNAPSHOT

id:                   ent_007
phase:                1
source:               agent_active
context:              content_structure
render_type:          hybrid
embedded_data_blocks: [{ "selector": "script#__NEXT_DATA__", "format": "json", "size_estimate": "24KB" }]
article_count_visible: 12
root_selector:        main#main-content
card_selector:        div.article-card
class_type:           hashed
stable_selectors:     ["article[data-article-id]", "h2.article-title", "time[datetime]"]
brittle_selectors:    ["div.css-1a2b3c4", "span._styledComponent_xyz"]
exclusion_selectors:  ["nav.sidebar", "footer", "div.ad-container"]
notes:                Card layout uses hashed Tailwind classes; data-attribute selectors are stable
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, context, render_type, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** embedded_data_blocks, article_count_visible, root_selector, card_selector, class_type, stable_selectors, brittle_selectors, exclusion_selectors, extraction_map

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `context` | enum | `initial_load` \| `post_pagination_N` \| `content_entry_N` \| `article_entry_N` \| `pagination_mechanism_identification` \| `content_structure` \| `page_chrome` \| `window_globals` \| `embedded_json_N` \| `raw_html_sample` \| `service_workers` \| `framework_fingerprint` \| `analytics_payload` \| `timestamp_comparison` \| `url_type_analysis` \| `duplicate_detection` \| `encoding_check` \| `compression_check` \| `fingerprinting` \| `head_analysis` \| `csp_analysis` \| `shadow_dom_content` \| `shadow_dom_closed` \| `spa_state_change` \| `a_b_compare` \| `hidden_content_revealed` \| `stability_matrix` \| `custom` |
| `render_type` | enum | `SSR` \| `CSR` \| `hybrid` \| `RSC` \| `UNKNOWN` \| `N/A` |
| `embedded_data_blocks` | array | Objects with `selector`, `format`, `size_estimate` |
| `article_count_visible` | integer \| null | Number of visible content items |
| `root_selector` | string \| null | CSS selector for the main content container |
| `card_selector` | string \| null | CSS selector for repeated content cards |
| `class_type` | enum | `semantic` \| `hashed` \| `mixed` \| `UNKNOWN` \| `N/A` |
| `stable_selectors` | array | Selectors likely to survive redesigns |
| `brittle_selectors` | array | Selectors likely to break on redesigns |
| `exclusion_selectors` | array | Selectors for non-content chrome to exclude |
| `extraction_map` | object \| null | Field-to-extraction-path mapping. Keys are field names (e.g., "title", "author"). Values are objects with `best_path` (string), `best_type` (enum: `structured_data` \| `semantic_html` \| `aria_role` \| `data_attribute` \| `meta_content` \| `class_semantic` \| `class_hashed`), `fallbacks` (array of {path, type}), and `stability_risk` (enum: `LOW` \| `MEDIUM` \| `HIGH`). `null` if not yet mapped. |
| `notes` | string | Factual observations about DOM structure |

---

### COOKIE

Captures an individual cookie observed during the investigation.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:25:00.789Z  COOKIE

id:                   ent_012
phase:                1
source:               cdp_passive
name:                 _ga
value_sample:         GA1.2.1234567890.123456789
domain:               .example.com
path:                 /
expires:              2027-01-15T10:25:00Z
httpOnly:             false
secure:               true
sameSite:             Lax
inferred_purpose:     analytics
set_by:               page_load
notes:                Google Analytics cookie; set on initial page load
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, name, domain, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** value_sample, path, expires, httpOnly, secure, sameSite, inferred_purpose, set_by, set_by_request

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `name` | string | Cookie name |
| `value_sample` | string | First 30 characters only — never log full cookie values |
| `domain` | string | Cookie domain |
| `path` | string | Cookie path |
| `expires` | string \| null | ISO 8601 or `session` |
| `httpOnly` | boolean | Whether cookie is HTTP-only |
| `secure` | boolean | Whether cookie requires HTTPS |
| `sameSite` | string | `Strict` \| `Lax` \| `None` \| `unknown` |
| `inferred_purpose` | enum | `tracking` \| `session` \| `crumb` \| `consent` \| `auth` \| `analytics` \| `unknown` |
| `set_by` | enum | `page_load` \| `consent_flow` \| `auth_flow` \| `api_response` \| `javascript` \| `unknown` |
| `set_by_request` | string \| null | Entry ID of the REQUEST that set this cookie (e.g., `ent_002`). Creates a linkable chain for cookie origin tracking. `null` if source is JS-only or unknown. |
| `notes` | string | Factual observations |

---

### LOCAL_STORAGE

Captures localStorage state at a point in the investigation.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:26:30.012Z  LOCAL_STORAGE

id:                   ent_018
phase:                1
source:               agent_active
keys:                 ["auth_token", "user_prefs", "consent_settings", "session_id", "cache_v2"]
entry_count:          5
notable_entries:
  - key:              auth_token
    value_sample:     eyJhbGciOiJIUzI1NiIsInR5cCI6...
    inferred_purpose: auth
  - key:              consent_settings
    value_sample:     {"analytics":true,"marketing":fal
    inferred_purpose: consent
auth_related_keys:    ["auth_token"]
content_related_keys: []
tracking_keys:        []
changed_since_last:   ["auth_token"]
notes:                auth_token appears to be a JWT; consent_settings controls analytics tracking
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, keys, entry_count, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** notable_entries, auth_related_keys, content_related_keys, tracking_keys, changed_since_last

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `keys` | array | All localStorage key names |
| `entry_count` | integer | Total number of entries |
| `notable_entries` | array | Objects with `key`, `value_sample` (first 30 chars), `inferred_purpose` |
| `auth_related_keys` | array | Keys related to authentication |
| `content_related_keys` | array | Keys related to content state |
| `tracking_keys` | array | Keys related to tracking/analytics |
| `changed_since_last` | array \| null | Keys that changed since previous LOCAL_STORAGE entry; `null` if first |
| `notes` | string | Factual observations |

---

### EDGE_CASE_TEST

Records the execution and result of an edge case test from the battery.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T11:45:00.000Z  EDGE_CASE_TEST

id:                   ent_042
phase:                6
source:               agent_active
test_id:              RATE_LIMIT_DETECTED
description:          Rapid-fire 10 requests to /api/v1/articles in 2s
method:               rapid_fire_test
url:                  https://example.com/api/v1/articles
cookies_sent:         ["session_id", "auth_token"]
result:               HTTP 429 after 7th request; Retry-After: 30
diff_from_normal:     Normal: 200 with JSON body. Edge: 429 with {"error":"rate_limit"}.
anomalies:            ["Rate limit returns JSON error body instead of HTML"]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, test_id, description, result, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** method, url, cookies_sent, diff_from_normal, anomalies, prerequisite_requests, cookies_required, headers_required, fingerprint_type, fastest_tested_rate

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `test_id` | enum | `ECT_001` \| `ECT_002` \| `RATE_LIMIT_DETECTED` \| `CONSENT_BANNER` \| `CONSENT_REDIRECT` \| `PAYWALL_DETECTED` \| `UA_TEST` \| `COOKIE_TEST` \| `PAGINATION_REPLAY` \| `CAPTCHA_DETECTED` \| `BLOCKER` \| `I18N_TEST` \| `TOKEN_ROTATION_TEST` \| `WARM_UP_COMPARE` \| `SPA_ROUTE_TEST` \| `CROSS_ORIGIN_IFRAME_TEST` \| `RSC_DETECTED` \| `THIRD_PARTY_CMS_API` \| `CMS_API_AUTH_DETECTED` \| `UA_TEST_SKIPPED_ROBOTS_TXT` \| `WAF_CHALLENGE_DETECTED` \| `SHADOW_DOM_DETECTED` \| `INTERSECTION_OBSERVER_DETECTED` \| `NON_HTTP_REPLAY_FAILURE` \| `HIDDEN_CONTENT_REVEALED` \| `SSR_API_SOURCE_OVERLAP` \| `SITEMAP_HIDDEN_STRUCTURE` \| `DEEP_WEB_ENDPOINT_FOUND` \| `SEARCH_FORM_NAVIGATION` \| `CRAWL_TRAP_DETECTED` \| `CUSTOM` |
| `description` | string | What the test does |
| `method` | enum | `http_raw_request` \| `browser_click` \| `browser_resize_viewport` \| `browser_navigate` \| `browser_execute_js` \| `rapid_fire_test` \| `wait_then_retry` \| `dom_query` \| `dom_query_scan` \| `scroll` \| `custom` |
| `url` | string \| null | URL tested, if applicable |
| `cookies_sent` | array \| null | Cookie names sent with the request |
| `result` | string | What happened — factual, no conclusions |
| `diff_from_normal` | string \| null | How this differs from baseline behavior |
| `anomalies` | array | See Anomaly Field Convention. Empty `[]` if none. |
| `prerequisite_requests` | array \| null | Entry IDs of requests that must complete before this one. `null` if no dependencies. |
| `cookies_required` | array \| null | Cookie names required for this request to succeed. `null` if not tested. |
| `headers_required` | array \| null | Header names required for this request to succeed (e.g., `Authorization`, `Referer`). `null` if not tested. |
| `fingerprint_type` | enum \| null | `header` \| `tls` \| `none` \| `null`. Only for test_id `NON_HTTP_REPLAY_FAILURE`. Distinguishes header-based fingerprinting (solvable with correct headers) from TLS-based (requires TLS impersonation). `null` if not applicable. |
| `fastest_tested_rate` | string \| null | Fastest request rate that returned normal responses during rate limit test. Format: `N req/s (tested)` or `N req/s (limited at M req/s)`. Only for test_id `RATE_LIMIT_DETECTED` or rate limit tests. NOT a safe rate guarantee — ceiling on tested speed. `null` if not applicable. |

---

### SYSTEM

Infrastructure-level events and agent lifecycle milestones.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:20:00.000Z  SYSTEM

id:                   ent_001
phase:                0
source:               system
event:                navigated
description:          Navigated to https://example.com
details:              { "url": "https://example.com", "loadTime_ms": 1432 }
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, event, description, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** details

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `event` | enum | `navigated` \| `consent_handled` \| `budget_status` \| `blocker_detected` \| `investigation_complete` \| `auto_exclude` \| `phase_complete` \| `custom` \| `retry_transient` \| `permanent_error` \| `degraded` \| `robots_txt_disallow` \| `cdp_filter_switch` \| `cdp_capture_limit_reached` \| `cdp_health_check` \| `cdp_dom_enabled` \| `cdp_dom_disabled` \| `geo_requirement_unmet` \| `unexpected_domain_redirect` \| `browser_recovery` \| `empty_content_state` \| `max_pages_reached` \| `ua_blocked_by_robots_txt` \| `probe_skipped_robots_txt` \| `sitemap_classification` \| `search_form_inventory` \| `search_form_skipped_stateful` \| `crawl_trap_boundary` \| `crawl_trap_suspected` \| `errata` \| `cookie_dependency_map` \| `consent_flow_map` \| `http_request_chain` \| `custom` |
| `description` | string | Human-readable factual description |
| `details` | object \| null | Structured key-value pairs relevant to the event |

---

### SESSION

Periodic checkpoint of overall session state.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T11:00:00.000Z  SESSION

id:                   ent_035
phase:                2
source:               system
current_url:          https://example.com/news?page=3
cookies_count:        8
new_cookies_since_last: 2
auth_state:           authenticated
auth_method:          cookie
pages_visited:        5
requests_made:        47
notes:                Auth confirmed via session cookie; no token rotation observed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, current_url, auth_state, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** cookies_count, new_cookies_since_last, auth_method, pages_visited, requests_made

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `current_url` | string | URL the browser is currently on |
| `cookies_count` | integer | Total cookies for the domain |
| `new_cookies_since_last` | integer | Cookies added since previous SESSION entry |
| `auth_state` | enum | `authenticated` \| `unauthenticated` \| `unknown` |
| `auth_method` | enum | `cookie` \| `bearer_token` \| `crumb` \| `none` \| `unknown` |
| `pages_visited` | integer | Number of distinct pages navigated to |
| `requests_made` | integer | Total HTTP requests observed |
| `notes` | string | Factual session observations |

---

### BUDGET_STATUS

Tracks investigation budget consumption and discovery progress.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T11:30:00.000Z  BUDGET_STATUS

id:                   ent_040
phase:                4
source:               system
status:               content_discovery_complete
cycles_used:          14
cycles_remaining:     21
cdp_requests_captured: 89
cdp_domains_observed:  3
api_endpoints_detected: 5
js_bundles_captured:   2
js_bundles_analyzed:   1
pagination_endpoint_found: true
auth_tokens_detected:  1
websocket_connections: 0
graphql_endpoints:     0
content_items_inspected: 0
complexity_assessment: MEDIUM
blockers_hit:          0
remaining_unexplored:  ["JS bundle #2 analysis", "GraphQL probe", "shadow DOM inspection"]
reinvestigation_recommendations: []
notes:                 Pagination found; API surface is moderate
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, status, cycles_used, cycles_remaining, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** cdp_requests_captured, cdp_domains_observed, api_endpoints_detected, js_bundles_captured, js_bundles_analyzed, pagination_endpoint_found, auth_tokens_detected, websocket_connections, graphql_endpoints, content_items_inspected, complexity_assessment, blockers_hit, remaining_unexplored, reinvestigation_recommendations

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `status` | enum | `baseline_complete` \| `content_discovery_complete` \| `item_inspection_complete` \| `deep_exploration_complete` \| `budget_exhausted` \| `investigation_complete` |
| `cycles_used` | integer | Priority queue cycles consumed |
| `cycles_remaining` | integer | Priority queue cycles left |
| `cdp_requests_captured` | integer | Total CDP network requests logged |
| `cdp_domains_observed` | integer | Distinct domains seen in CDP traffic |
| `api_endpoints_detected` | integer | Distinct API endpoints discovered |
| `js_bundles_captured` | integer | JS bundles saved for analysis |
| `js_bundles_analyzed` | integer | JS bundles fully analyzed |
| `pagination_endpoint_found` | boolean | Whether pagination mechanism was identified |
| `auth_tokens_detected` | integer | Auth-related tokens/cookies found |
| `websocket_connections` | integer | Active or observed WebSocket connections |
| `graphql_endpoints` | integer | GraphQL endpoints discovered |
| `content_items_inspected` | integer | Individual content items entered and inspected |
| `complexity_assessment` | enum | `LOW` \| `MEDIUM` \| `HIGH` |
| `blockers_hit` | integer | Number of blockers encountered |
| `remaining_unexplored` | array | Description of unfinished investigation areas |
| `reinvestigation_recommendations` | array | Items warranting a second pass |
| `notes` | string | Factual budget observations |

---

### UNKNOWN

Catches observations that don't fit any other type. Use sparingly — prefer a typed entry.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T12:00:00.000Z  UNKNOWN

id:                   ent_050
phase:                4
source:               cdp_passive
description:          Received a binary WebSocket frame with non-UTF8 content
raw_evidence:         "[154 bytes of binary data, first 4 bytes: 0x89 0x50 0x4E 0x47]"
hypothesis:           May be a PNG image streamed over WebSocket
resolution_test:      Capture next frame with base64 encoding for MIME detection
related_entries:      ["ent_048", "ent_049"]
confidence_in_hypothesis: LOW
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, description, raw_evidence, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** hypothesis, resolution_test, related_entries, confidence_in_hypothesis

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `description` | string | What was observed |
| `raw_evidence` | string | Raw data or representation of the observation |
| `hypothesis` | string \| null | Best guess at explanation (clearly labeled as hypothesis) |
| `resolution_test` | string \| null | Suggested test to confirm or reject the hypothesis |
| `related_entries` | array | IDs of entries that may be related |
| `confidence_in_hypothesis` | enum | `LOW` \| `MEDIUM` \| `HIGH` |

---

### SERVICE_WORKER

Captures a service worker observed during the investigation.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:28:00.000Z  SERVICE_WORKER

id:                   ent_022
phase:                1
source:               cdp_passive
scope:                https://example.com/
script_url:           https://example.com/sw.js
state:                activated
intercepts_fetch:     true
cache_names:          ["workbox-precache-v2-https://example.com/", "runtime-cache"]
impact_hypothesis:    Service worker caches HTML shell; may serve stale content on revisit
notes:                Uses Workbox; precache manifest includes 14 assets
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Core fields (ALWAYS present):** id, phase, source, scope, script_url, state, notes

**Conditional fields (fill if FIRST entry of this type, or gap of 10+ entries since last of this type, or the field has non-null/interesting data):** intercepts_fetch, cache_names, impact_hypothesis

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `id` | string | `ent_NNN` or `ent_auto_NNN` |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |
| `scope` | string | Service worker scope URL |
| `script_url` | string | URL of the service worker script |
| `state` | enum | `activated` \| `installed` \| `waiting` \| `redundant` |
| `intercepts_fetch` | boolean \| enum | `true` \| `false` \| `UNKNOWN` |
| `cache_names` | array | Names of Cache API caches the service worker manages |
| `impact_hypothesis` | string | How the SW may affect scraping (clearly labeled as hypothesis) |
| `notes` | string | Factual observations about the service worker |

---

## Phase Numbering Convention

```
Phase 0: Pre-flight (robots.txt, sitemap, environment, warm-up)
Phase 1: Baseline (initial load, DOM, globals, embedded data, head analysis)
Phase 2: Content discovery (content structure, pagination)
Phase 3: Content item inspection (enter N items, DOM snapshots)
Phase 4: Deep exploration (API probing, JS analysis, token tracing)
Phase 5: Request replay experiments
Phase 6: Edge case battery
Phase 7: Re-investigation (only if reinvestigation requests exist)
```

### Phase Mapping Table

Priority queue step → Phase number:

| Priority Queue Steps | Phase | Description | Reference File | D0 File |
|---|---|---|---|---|
| Pre-Brief | 0 | Full site brief ingestion | gate-1 | state.log |
| P1 | 0 | Pre-flight | gate-1 | g1d0.log |
| P2–P8 | 1 | Baseline | gate-1 | g1d0.log |
| P9–P13 | 2 | Content discovery | gate-2 | g2d0.log |
| P14–P16 | 3 | Content item inspection | gate-3 | g3d0.log |
| P17–P22 | 4 | Deep exploration | gate-4 | g4d0.log |
| P23–P27 | 5 | Request replay | gate-5 | g5d0.log |
| P28–P31 | 6 | Edge case battery | gate-6 | g6d0.log |
| P32+ | 7 | Re-investigation | gate-6 | g6d0.log |

---

## Source Field Convention

```
cdp_passive:   Automatically captured by CDP. No agent decision involved.
agent_active:  Agent explicitly decided to perform this action.
system:        Infrastructure event (budget, navigation, session state). No observation content.
```

---

## Response Body Handling Rules

```
JSON responses <= 1MB:
  res_body_sample:    full response body
  res_body_schema:    full schema
  res_body_truncated: false

JSON responses > 1MB:
  res_body_sample:    first 50,000 chars + "[TRUNCATED at 50000 of 2345678]"
  res_body_schema:    full schema (always included)
  res_body_truncated: true
  res_body_total_size: actual size

HTML responses:
  res_body_sample:    first 5,000 chars
  res_body_truncated: true (always for HTML)

JavaScript <= 500KB:
  Full content for regex analysis

JavaScript > 500KB:
  First 1,000 chars + "[TOO_LARGE for full capture]"
  Always log regex scan results separately

WebSocket:
  First 200 chars of first 20 frames
  Always truncated. Log total frame count in notes.

Images, CSS, fonts, binary:
  URL, size, content-type only — no body sample
```

---

## Anomaly Field Convention

```
GOOD anomalies (unexpected server behavior — LOG THESE):
  - 200 but empty body
  - Content-Type mismatch with body
  - Redirect chain >5 hops
  - 10x slower than previous request to same endpoint
  - Response body structure differs from schema
  - Unexpected auth requirement on previously open endpoint

BAD anomalies (normal variation — do NOT log):
  - Different article count from expected
  - Different DOM on mobile viewport
  - Minor timing variation (<2x)
  - Cookie value changes between requests (rotation is normal)
```

---

## Redirect Chain Format

When redirects occur, the `redirect_chain` array captures every hop. The main `url` field always shows the **final** destination.

```json
redirect_chain: [
  { "status": 301, "url": "https://example.com/old-path" },
  { "status": 302, "url": "https://example.com/intermediate" },
  { "status": 200, "url": "https://example.com/final-path" }
]
url: "https://example.com/final-path"
```

Each hop includes the redirect status code and the URL redirected to. The final entry in the chain shows the terminal status and URL, which must match the top-level `url` and `res_status` fields.

---

## Entry Corrections (Errata)

Entries are append-only — you never modify or delete a previous entry. If you discover an error in a previously written entry, write a correction using a SYSTEM entry with `event: errata`.

### Errata Entry Format

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T12:30:00.000Z  SYSTEM

id:                   ent_055
phase:                4
source:               system
event:                errata
description:          Correction to ent_012
details:
  corrects_entry:     ent_012
  field:              inferred_purpose
  original_value:     tracking
  corrected_value:    session
  reason:             Cookie rotates on each request; matches session pattern, not tracking
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Errata Rules

1. **One correction per errata entry.** If multiple fields are wrong, write multiple errata entries.
2. **Always reference the original entry ID** in `corrects_entry`.
3. **Always state the reason.** The reason must be a raw observation, not analysis. (e.g., "Cookie rotates on each request" is valid; "Cookie is clearly a session cookie" is analysis.)
4. **Errata must cite a raw observation.** The `reason` field must include a specific raw observation that supports the correction, not a feeling or intuition. If you cannot articulate WHY the original was wrong using a specific raw observation, do NOT write errata — add a new observation entry instead.
5. **Errata are applied during compaction.** During the investigation, the original entry remains unchanged. See `references/compaction.md`.
6. **Errata for missing required fields** should include the missing field's value in `corrected_value`.

### Common Errata Scenarios

| Scenario | Example |
|----------|--------|
| Wrong `inferred_purpose` on COOKIE entry | Originally tagged as `tracking`, but subsequent observation showed it rotates (session) |
| Missing required field in REQUEST entry | `req_headers` was omitted at P2; adding it now after re-reading spec |
| Wrong `render_type` in DOM_SNAPSHOT | Originally classified as `SSR`, but P6 raw HTML comparison showed it's `hybrid` |
| Incorrect `source` classification | Logged as `cdp_passive` but was actually `agent_active` |
| Timestamp error | Entry was written with incorrect timestamp (rare; use current time as corrected_value) |
