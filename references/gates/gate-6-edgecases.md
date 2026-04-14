# Gate 6: Edge Cases & Completion (P28–P32+)

```yaml
gate_id: 6
title: Edge Cases & Completion
steps: P28 → P32+
phases: [6, 7, 8]
d0_file: g6d0.log
operator_halt: false
next_gate: null
final_gate: true
```

Read this file after completing Gate 5 (request replay). This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] Full cookie catalog, robots.txt, CSP headers, consent mapping (Gate 1)
- [ ] Pagination endpoint, replay feasibility (Gate 2)
- [ ] Content item structure, hidden content findings (Gate 3)
- [ ] All discovered API endpoints, token lifecycles (Gate 4)
- [ ] Request replay results, HTTP request chain (Gate 5)

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries including BUDGET_STATUS) | `g6d0.log` |
| D2:State updates | `state.log` |
| D1: Edge Cases Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 6 | P28 → P31 | Edge case battery — boundary conditions |
| 7 | P-X | Open exploration — unexpected observations |
| 8 | P32+ | Re-investigation (if s2_gaps.md provided) |

**Write gate at P31.** This is the final gate.

---

## Phase 6: Edge Case Battery (~5 cycles)

Quick tests probing unusual conditions that may reveal different content, behavior, or access patterns.

---

### [P28] Empty User-Agent request via raw HTTP

```yaml
step: P28
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: EMPTY_UA_TEST }
```

Send a request with an empty or missing `User-Agent` header via raw HTTP.

**Feeds into:** P23 (UA comparison), HTTP_REQUEST_CHAIN.

---

### [P29] Cookie-less request via raw HTTP

```yaml
step: P29
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: RAW_HTTP_COOKIELESS }
```

Send requests without any cookies via raw HTTP — both for the index page and a detail page.

**Feeds into:** P24 (cookie-removal comparison), HTTP_REQUEST_CHAIN.

---

### [P30] Sponsored/ad content identification

```yaml
step: P30
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: SPONSORED_CONTENT_IDENTIFICATION }
```

**Detection methods:**

| Method | Signal |
|--------|--------|
| Visual labels | "Sponsored", "Ad", "Promoted", "Paid Partnership" in DOM |
| Structural differences | Different selectors, different parent containers, `data-ad-*` attributes |
| Different API sources | Separate endpoint (e.g., `ad.*.com` domains in CDP captures) |
| Google Ad Manager | `googlesyndication.com`, `doubleclick.net` in network requests |

**Feeds into:** Extraction schema (filter/flag sponsored content).

---

### [P31] Rate limit test (2-tier)

```yaml
step: P31
cycle: true
condition: ALWAYS
budget: 2
log: { type: EDGE_CASE_TEST, test_id: RATE_LIMIT_DETECTED }
```

Fire the discovered API endpoint in 2 tiers to find the fastest tested rate. NOT a guarantee of safe production speed.

**Tier 1 (standard):** 5 requests at 1-second intervals.

**If Tier 1 passes (all 200, no degradation):** Run Tier 2.

**Tier 2 (fast):** 5 requests at 0.3-second intervals.

**Watch for (both tiers):**

| Signal | Meaning |
|--------|---------|
| HTTP 429 | Explicit rate limiting |
| HTTP 503 | Rate limiting or load shedding |
| Gradual content reduction | Silent reduction under load |
| CAPTCHA challenge | Rate limiting via CAPTCHA instead of 429 |
| Consistent responses | No rate limiting at this volume |

**Result classification:**

| Outcome | `fastest_tested_rate` |
|---------|----------------------|
| Tier 1 hits rate limit | `1 req/s (limited)` |
| Tier 1 passes, Tier 2 hits rate limit | `1 req/s (limited at 3.3 req/s)` |
| Both tiers pass | `3.3 req/s (tested)` |
| Tier 1 passes, Tier 2 not run (budget) | `1 req/s (tested only)` |

**Log:** EDGE_CASE_TEST with test_id `RATE_LIMIT_DETECTED` (if limited) or `CUSTOM` (if no limit found). Include `fastest_tested_rate` field. This field is NOT a safe rate guarantee — it's the fastest speed that returned normal responses during this test.

**Budget:** 1-2 decision cycles. Tier 2 only runs if Tier 1 passes cleanly.

**Feeds into:** Extraction strategy (request pacing).

---

## Phase 7: Open Exploration (lowest priority, ~15% remaining budget)

---

### [P-X] Open exploration

```yaml
step: P-X
cycle: true
condition: agent observes something potentially relevant not covered by P1–P31
budget: 15% of remaining decision cycles
log: varies
```

**Requirements:**

- Must log what triggered the exploration and why.
- Must use standard log entry types (DOM_SNAPSHOT, EDGE_CASE_TEST, SYSTEM, etc.).
- Must still follow the general investigation principles from SKILL.md.

**Example triggers:**

| Trigger | Example |
|---------|---------|
| Unusual response header | Not seen before in this investigation |
| Uninvestigated DOM element | Comment section, sharing widget, dark mode toggle revealing different content |
| Unexpected domain in CDP | Not matching known API/ad domains |
| Non-standard dynamic loading | Doesn't fit IO/pagination patterns from P9a/P10 |
| WebSocket schema change | Different schema than documented in P20 |

---

## Phase 8: Re-Investigation (only if s2_gaps.md provided)

---

### [P32+] Execute each specific request from s2_gaps.md in order

```yaml
step: P32+
cycle: true
condition: s2_gaps.md provided by second-pass analysis
budget: 5 cycles per request item
log: varies
```

**Procedure:**

- Execute each request from `s2_gaps.md` in order.
- Each request is a decision cycle.
- **Budget:** 5 cycles per request item, maximum.

---

## Gate 6 Output (Final Gate)

Before concluding the investigation, verify:

- [ ] Empty User-Agent test completed (P28)
- [ ] Cookie-less request test completed (P29)
- [ ] Sponsored/ad content identified (P30)
- [ ] Rate limit test completed (P31)
- [ ] Open exploration completed if triggered (P-X)
- [ ] Re-investigation completed if s2_gaps.md provided (P32+)
- [ ] D2:State updated with final state
- [ ] D1: Edge Cases Phase Summary written
- [ ] Final BUDGET_STATUS written (to g6d0.log)
- [ ] Investigation complete — proceed to compaction (see `references/compaction.md`)
