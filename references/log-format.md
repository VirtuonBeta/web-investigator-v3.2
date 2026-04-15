# Log Format Specification `v3.2`

```yaml
purpose: entry_type_definitions
scope: field_specs, shared_conventions, write_rules
read_schedule: [when_writing_any_entry]
```

---

## 1. Shared Rules

### Entry Format

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cycle: N  TYPE

id:                   gN:NNN
phase:                N
source:               cdp_passive | agent_active | system
...type-specific fields...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

1. Every entry is a typed block delimited by `━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━`
2. Every entry starts with `cycle: N  TYPE` where N is the current decision cycle count
3. No free-form prose in data fields. Notes and descriptions are fine. Conclusions are not.
4. D0 files are append-only. state.log: D2 replaced at top; D1 inserted bottom-up. ALL typed entries go in current gate D0 file — state.log is D2+D1 only.
5. Truncation must be marked: `[TRUNCATED at N chars of total M]`

### Shared Fields

All entries include these fields. Not repeated in per-type tables.

| Field | Type | Description |
|---|---|---|
| `id` | string | Gate-qualified ID in `gN:NNN` format |
| `phase` | integer | Phase number |
| `source` | enum | `cdp_passive` \| `agent_active` \| `system` |

### ID Format — `gN:NNN`

- `g` — literal prefix
- `N` — gate number (1–6)
- `:` — separator
- `NNN` — zero-padded local counter (001–999)
- **Self-locating:** `g2:014` = "open g2d0.log, find entry 014"
- Each gate D0 file has its own counter starting at 001
- Counters never reset or renumbered — gaps acceptable after compaction
- D2:State and D1 summaries do NOT have gN:NNN IDs — they are structural sections
- Cross-reference fields use gN:NNN: `set_by_request`, `prerequisite_requests`, `related_entries`, `corrects_entry`

### Cycle Numbers

- `cycle: N` — monotonically increasing integer from budget counter
- NOT a timestamp — derived from budget, not system clock
- Multiple entries may share the same cycle
- Within same gate file: file position = tiebreaker. Across gates: cycle = only ordering signal

### Conditional Fields Rule

Fill conditional fields if ANY of: first entry of this type, gap of 10+ entries since last of this type, field has non-null/interesting data.

### Null Convention

```yaml
null: "checked this field, found nothing (e.g., no cookies, no API endpoints)"
null_not_checked: "did not check this field (e.g., not relevant to current phase, ran out of time)"

rules:
  - Always distinguish between looked-and-found-nothing (null) and didn't-look (null_not_checked)
  - null is an observation: you checked, nothing was there
  - null_not_checked is a signal: this field remains unexplored
  - The review sub-agent uses this distinction to identify INCOMPLETE_INVESTIGATION
  - Never leave a field empty/missing — use null or null_not_checked explicitly
```

### D0 Content Restriction

D0 entries contain ONLY raw observations. No forward-looking content.

```yaml
allowed_in_D0:
  - what you saw (raw data, DOM state, HTTP responses)
  - what happened (navigation, clicks, errors)
  - measurements (sizes, counts, timings)

forbidden_in_D0:
  - next_steps / what_to_do_next / "next I should..."
  - recommendations / reinvestigation_recommendations
  - planning / selection_strategy / "recommended: P13b"
  - conclusions / analysis / "therefore" / "this means" / "this suggests"
  - ANSWERED / PARTIALLY_ANSWERED status on questions
  - any forward-looking statement of any kind

if_you_catch_yourself_writing: "next I should..." or "recommended:"
then: that content belongs in D2:State (Next_steps field), not in a D0 entry
```

---

## 2. Entry Types

### REQUEST

HTTP request/response observed or initiated.

**Core:** method, url, res_status, res_body_type, notes
**Conditional:** trigger, resource_type, req_headers, req_body, res_headers, res_body_sample, res_body_schema, res_body_truncated, res_body_total_size, res_size_bytes, ttlfb_ms, redirect_chain, anomalies

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cycle: 14  REQUEST

id:                   g2:003
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

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `method` | string | HTTP method: GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD |
| `url` | string | Full URL |
| `trigger` | enum | `page_load` \| `scroll` \| `click` \| `manual` \| `rapid_fire_test` \| `edge_case` \| `api_probe` \| `pagination` \| `navigation` \| `unknown` |
| `resource_type` | string | CDP resource type: document, stylesheet, image, font, script, xhr, fetch, websocket, etc. |
| `req_headers` | object \| null | Relevant request headers (auth, cookies, accept, content-type) |
| `req_body` | string \| object \| null | Request body if present |
| `res_status` | integer | HTTP status code |
| `res_headers` | object | Relevant response headers |
| `res_body_type` | enum | `json` \| `html` \| `javascript` \| `css` \| `image` \| `font` \| `binary` \| `websocket` \| `unknown` |
| `res_body_sample` | string \| null | See §3 Response Body Handling |
| `res_body_schema` | object \| null | JSON Schema of response (always included for JSON, even if truncated) |
| `res_body_truncated` | boolean | `true` if sample is incomplete |
| `res_body_total_size` | integer \| null | Total response body size in chars (only when truncated) |
| `res_size_bytes` | integer | Total response size in bytes (headers + body) |
| `ttlfb_ms` | integer \| null | Time to first byte in milliseconds |
| `redirect_chain` | array | Array of `{ status, url }` hops. Main `url` field shows final destination. |
| `anomalies` | array | See §3 Anomaly Convention. Empty `[]` if none. |
| `notes` | string | Factual observations only. No conclusions. |

---

### DOM_SNAPSHOT

DOM state at a specific point in the investigation.

**Core:** context, render_type, notes
**Conditional:** embedded_data_blocks, article_count_visible, root_selector, card_selector, class_type, stable_selectors, brittle_selectors, exclusion_selectors, extraction_map

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
cycle: 5  DOM_SNAPSHOT

id:                   g1:004
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

| Field | Type | Allowed Values / Notes |
|---|---|---|
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
| `extraction_map` | object \| null | Field-to-extraction-path mapping. Keys = field names. Values = { `best_path` (string — CSS selector or JSON path ONLY, no type suffix), `best_type` (enum: `structured_data` \| `semantic_html` \| `aria_role` \| `data_attribute` \| `meta_content` \| `class_semantic` \| `class_hashed`), `fallbacks` (array of {path, type}), `stability_risk` (enum: `LOW` \| `MEDIUM` \| `HIGH`) }. `null` if not yet mapped. Example: `best_path: "ld+json.headline"`, NOT `best_path: "ld+json.headline [structured_data]"` — the type goes in `best_type`, not in the path string. |
| `notes` | string | Factual observations about DOM structure |

---

### COOKIE

Individual cookie observed during investigation.

**Core:** name, domain, notes
**Conditional:** value_sample, path, expires, httpOnly, secure, sameSite, inferred_purpose, set_by, set_by_request

| Field | Type | Allowed Values / Notes |
|---|---|---|
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
| `set_by_request` | string \| null | gN:NNN of REQUEST that set this cookie. `null` if JS-only or unknown. |
| `notes` | string | Factual observations |

---

### LOCAL_STORAGE

localStorage state at a point in the investigation.

**Core:** keys, entry_count, notes
**Conditional:** notable_entries, auth_related_keys, content_related_keys, tracking_keys, changed_since_last

| Field | Type | Allowed Values / Notes |
|---|---|---|
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

Execution and result of an edge case test.

**Core:** test_id, description, result, notes
**Conditional:** method, url, cookies_sent, diff_from_normal, anomalies, prerequisite_requests, cookies_required, headers_required, fingerprint_type, fastest_tested_rate

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `test_id` | enum | `ECT_001` \| `ECT_002` \| `RATE_LIMIT_DETECTED` \| `CONSENT_BANNER` \| `CONSENT_REDIRECT` \| `PAYWALL_DETECTED` \| `UA_TEST` \| `COOKIE_TEST` \| `PAGINATION_REPLAY` \| `CAPTCHA_DETECTED` \| `BLOCKER` \| `I18N_TEST` \| `TOKEN_ROTATION_TEST` \| `WARM_UP_COMPARE` \| `SPA_ROUTE_TEST` \| `CROSS_ORIGIN_IFRAME_TEST` \| `RSC_DETECTED` \| `THIRD_PARTY_CMS_API` \| `CMS_API_AUTH_DETECTED` \| `UA_TEST_SKIPPED_ROBOTS_TXT` \| `WAF_CHALLENGE_DETECTED` \| `SHADOW_DOM_DETECTED` \| `INTERSECTION_OBSERVER_DETECTED` \| `NON_HTTP_REPLAY_FAILURE` \| `HIDDEN_CONTENT_REVEALED` \| `SSR_API_SOURCE_OVERLAP` \| `SITEMAP_HIDDEN_STRUCTURE` \| `DEEP_WEB_ENDPOINT_FOUND` \| `SEARCH_FORM_NAVIGATION` \| `CRAWL_TRAP_DETECTED` \| `CUSTOM` |
| `description` | string | What the test does |
| `method` | enum | `http_raw_request` \| `browser_click` \| `browser_resize_viewport` \| `browser_navigate` \| `browser_execute_js` \| `rapid_fire_test` \| `wait_then_retry` \| `dom_query` \| `dom_query_scan` \| `scroll` \| `custom` |
| `url` | string \| null | URL tested, if applicable |
| `cookies_sent` | array \| null | Cookie names sent with the request |
| `result` | string | What happened — factual, no conclusions |
| `diff_from_normal` | string \| null | How this differs from baseline behavior |
| `anomalies` | array | See §3 Anomaly Convention. Empty `[]` if none. |
| `prerequisite_requests` | array \| null | gN:NNN IDs of prerequisite requests. `null` if no dependencies. |
| `cookies_required` | array \| null | Cookie names required for success. `null` if not tested. |
| `headers_required` | array \| null | Header names required (e.g., `Authorization`, `Referer`). `null` if not tested. |
| `fingerprint_type` | enum \| null | `header` \| `tls` \| `none` \| `null`. Only for `NON_HTTP_REPLAY_FAILURE`. |
| `fastest_tested_rate` | string \| null | `N req/s (tested)` or `N req/s (limited at M req/s)`. Only for rate limit tests. NOT a safe rate guarantee. |
| `notes` | string | Factual observations |

---

### SYSTEM

Infrastructure events and agent lifecycle milestones. Errata = SYSTEM with `event: errata` (→ writing-protocol.md §3).

**Core:** event, description, notes
**Conditional:** details

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `event` | enum | `navigated` \| `consent_handled` \| `budget_status` \| `blocker_detected` \| `investigation_complete` \| `auto_exclude` \| `phase_complete` \| `custom` \| `retry_transient` \| `permanent_error` \| `degraded` \| `robots_txt_disallow` \| `cdp_filter_switch` \| `cdp_capture_limit_reached` \| `cdp_health_check` \| `cdp_dom_enabled` \| `cdp_dom_disabled` \| `geo_requirement_unmet` \| `unexpected_domain_redirect` \| `browser_recovery` \| `empty_content_state` \| `max_pages_reached` \| `ua_blocked_by_robots_txt` \| `probe_skipped_robots_txt` \| `sitemap_classification` \| `search_form_inventory` \| `search_form_skipped_stateful` \| `crawl_trap_boundary` \| `crawl_trap_suspected` \| `errata` \| `cookie_dependency_map` \| `consent_flow_map` \| `http_request_chain` \| `custom` |
| `description` | string | Factual description |
| `details` | object \| null | Structured key-value pairs relevant to the event |
| `notes` | string | Factual observations |

---

### SESSION

Periodic checkpoint of session state.

**Core:** current_url, auth_state, notes
**Conditional:** cookies_count, new_cookies_since_last, auth_method, pages_visited, requests_made

| Field | Type | Allowed Values / Notes |
|---|---|---|
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

Investigation budget consumption and discovery progress.

**Core:** status, cycles_used, cycles_remaining, notes
**Conditional:** cdp_requests_captured, cdp_domains_observed, api_endpoints_detected, js_bundles_captured, js_bundles_analyzed, pagination_endpoint_found, auth_tokens_detected, websocket_connections, graphql_endpoints, content_items_inspected, complexity_assessment, blockers_hit

| Field | Type | Allowed Values / Notes |
|---|---|---|
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
| `notes` | string | Factual budget observations |

---

### UNKNOWN

Observations that don't fit any other type. Use sparingly — prefer a typed entry.

**Core:** description, raw_evidence, notes
**Conditional:** hypothesis, resolution_test, related_entries, confidence_in_hypothesis

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `description` | string | What was observed |
| `raw_evidence` | string | Raw data or representation of the observation |
| `hypothesis` | string \| null | Best guess at explanation (labeled as hypothesis) |
| `resolution_test` | string \| null | Suggested test to confirm or reject the hypothesis |
| `related_entries` | array | gN:NNN IDs of entries that may be related |
| `confidence_in_hypothesis` | enum | `LOW` \| `MEDIUM` \| `HIGH` |
| `notes` | string | Factual observations |

---

### SERVICE_WORKER

Service worker observed during investigation.

**Core:** scope, script_url, state, notes
**Conditional:** intercepts_fetch, cache_names, impact_hypothesis

| Field | Type | Allowed Values / Notes |
|---|---|---|
| `scope` | string | Service worker scope URL |
| `script_url` | string | URL of the service worker script |
| `state` | enum | `activated` \| `installed` \| `waiting` \| `redundant` |
| `intercepts_fetch` | boolean \| enum | `true` \| `false` \| `UNKNOWN` |
| `cache_names` | array | Names of Cache API caches the service worker manages |
| `impact_hypothesis` | string | How the SW may affect scraping (labeled as hypothesis) |
| `notes` | string | Factual observations about the service worker |

---

## 3. Shared Conventions

### Response Body Handling

```
JSON <= 1MB:         full body + full schema, res_body_truncated: false
JSON > 1MB:          first 50,000 chars + "[TRUNCATED at 50000 of N]", full schema, res_body_truncated: true
HTML:                first 5,000 chars, always truncated
JavaScript <= 500KB: full content
JavaScript > 500KB:  first 1,000 chars + "[TOO_LARGE for full capture]", regex results logged separately
WebSocket:           first 200 chars of first 20 frames, always truncated
Images/CSS/fonts/binary: URL, size, content-type only
```

### Anomaly Field Convention

```
LOG (unexpected server behavior):
  200 but empty body, Content-Type mismatch, redirect chain >5 hops,
  10x slower than same endpoint, body structure differs from schema,
  unexpected auth on previously open endpoint

DO NOT LOG (normal variation):
  different article count, different DOM on mobile, minor timing <2x,
  cookie value changes between requests (rotation is normal)
```

### Extraction Path Taxonomy

| Path Type | Priority | Pattern | Survives Deploy? |
|-----------|----------|---------|-----------------|
| `structured_data` | 1 (best) | ld+json, __NEXT_DATA__, embedded JSON | Always |
| `semantic_html` | 2 | `<article>`, `<time datetime>`, `<h1>`, `<main>` | Mostly |
| `aria_role` | 2 | `[role="article"]`, `[role="heading"]` | Mostly |
| `data_attribute` | 3 | `[data-testid]`, `[data-cy]`, `[data-component]` | Often |
| `meta_content` | 3 | `<meta name="description">`, `<meta property="og:*">` | Usually |
| `class_semantic` | 4 | `.article-title`, `.post-body` | Sometimes |
| `class_hashed` | 5 (worst) | `.yf-1a2b3c`, `.css-xyz123`, `._abc123` | Never |

Fields with only `class_hashed` paths → `[brittle]`. Fields with no stable path → `stability_risk: HIGH`.

Referenced by: P3, P5, P15, P22.
