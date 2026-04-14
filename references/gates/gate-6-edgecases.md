# Gate 6: Edge Cases & Completion (P28–P32+)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file after completing Gate 5 (request replay).

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from All Prior Gates

Before using this file, you should already have:
- From Gate 1 (`references/gates/gate-1-baseline.md`): full cookie catalog, robots.txt, CSP headers, consent mapping
- From Gate 2 (`references/gates/gate-2-pagination.md`): pagination endpoint, replay feasibility
- From Gate 3 (`references/gates/gate-3-inspection.md`): content item structure, hidden content findings
- From Gate 4 (`references/gates/gate-4-exploration.md`): all discovered API endpoints, token lifecycles
- From Gate 5 (`references/gates/gate-5-replay.md`): request replay results, HTTP request chain

If any prerequisite is missing, return to the appropriate gate file before proceeding.

## Write Targets

| What | File | Why |
|------|------|-----|
| Raw observations (all typed entries including BUDGET_STATUS, etc.) | `g6d0.log` | Gate-scoped D0 — all typed entries go here |
| D2:State updates | `state.log` | State checkpoint |
| D1: Edge Cases Phase Summary | `state.log` | Phase completion record |

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 6 | P28 → P31 | Edge case battery — boundary conditions |
| Phase 7 | P-X | Open exploration — unexpected observations |
| Phase 8 | P32+ | Re-investigation (if s2_gaps.md provided) |

**Write gate at P31.** All pending observations must be logged, D2:State updated, D1 Phase Summary written. This is the final gate.

---

## Phase 6: Edge Case Battery (~5 cycles)

These tests probe unusual conditions that may reveal different content, different behavior, or different access patterns. They're quick to execute and often reveal important constraints.

---

### [P28] Empty User-Agent request via raw HTTP

Send a request with an empty or missing `User-Agent` header via raw HTTP.

**Why this matters:** Some sites have default behavior for unknown UAs that differs from both browser and bot UAs. An empty UA might bypass bot detection (if it only matches known bot UAs) or might trigger a block (if it requires a UA).

---

### [P29] Cookie-less request via raw HTTP

Send requests without any cookies via raw HTTP — both for the index page and a detail page.

**Why this matters:** This is the "can anyone access this content?" test. If the cookie-less request returns full content, the site is publicly accessible with no authentication. If it returns limited content or a login wall, you know exactly what's behind the authentication barrier.

---

### [P30] Sponsored/ad content identification

Identify which content items are sponsored, promoted, or advertisements rather than organic content.

**How to detect:**

- **Visual labels:** "Sponsored", "Ad", "Promoted", "Paid Partnership" in the DOM.
- **Structural differences:** Ad items may have different selectors, different parent containers, or `data-ad-*` attributes.
- **Different API sources:** Ads may come from a separate endpoint (e.g., `ad.*.com` domains in CDP captures).
- **Google Ad Manager:** Look for `googlesyndication.com`, `doubleclick.net` in network requests.

**Why this matters:** Including sponsored content in extraction results can be misleading or undesirable. Identifying it allows you to filter it out or flag it separately.

---

### [P31] Rate limit test (2-tier)

Fire the discovered API endpoint in 2 tiers to find the fastest tested rate. This is NOT a guarantee of safe production speed — it's the fastest speed tested during the investigation.

**Tier 1 (standard):** 5 requests at 1-second intervals. Observe whether responses degrade, return rate-limit errors (429), or remain consistent.

**If Tier 1 passes (all 200, no degradation):** Run Tier 2.

**Tier 2 (fast):** 5 requests at 0.3-second intervals. Observe same criteria.

**What to watch for (both tiers):**

- **HTTP 429 Too Many Requests:** Explicit rate limiting.
- **HTTP 503 Service Unavailable:** May indicate rate limiting or load shedding.
- **Gradual content reduction:** Some sites silently reduce content under load.
- **CAPTCHA challenge:** Some sites respond to rate limiting with CAPTCHAs instead of 429s.
- **Consistent responses:** No rate limiting detected at this request volume.

**Result classification:**

| Outcome | `fastest_tested_rate` |
|---|---|
| Tier 1 hits rate limit | `1 req/s (limited)` |
| Tier 1 passes, Tier 2 hits rate limit | `1 req/s (limited at 3.3 req/s)` |
| Both tiers pass | `3.3 req/s (tested)` |
| Tier 1 passes, Tier 2 not run (budget) | `1 req/s (tested only)` |

**Log:** EDGE_CASE_TEST with test_id `RATE_LIMIT_DETECTED` (if limited) or `CUSTOM` (if no limit found). Include `fastest_tested_rate` field with the result string above. This field is NOT a safe rate guarantee — it's the fastest speed that returned normal responses during this test.

**Budget:** 1-2 decision cycles. Tier 2 only runs if Tier 1 passes cleanly.

**Why this matters:** Understanding rate limits determines how fast you can extract data. The 2-tier approach finds whether the API tolerates faster requests without the budget cost of 9-request burst tests. The analyst should treat `fastest_tested_rate` as a ceiling on tested speed, not a production recommendation.

---

## Phase 7: Open Exploration (lowest priority, ~15% remaining budget)

---

### [P-X] Open exploration

**Trigger:** The agent observes something potentially relevant that isn't covered by steps P1–P31.

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

### [P32+] Execute each specific request from s2_gaps.md in order

This phase only activates if the agent receives an `s2_gaps.md` file from a second-pass analysis. Each item in that file represents a specific gap or question that the first investigation didn't resolve.

**Procedure:**

- Execute each request from `s2_gaps.md` in order.
- Each request is a decision cycle.
- **Budget:** 5 cycles per request item, maximum.

**Why this matters:** The second-pass investigation is targeted — it only looks at specific gaps rather than re-running the full investigation. This budget cap prevents a single complex gap from consuming the entire re-investigation budget.

---

*End of Priority Queue reference. Return to SKILL.md for overall investigation philosophy and decision framework.*

---

## Gate 6 Output (Final Gate)

Before concluding the investigation, verify:

☐ Empty User-Agent test completed (P28)
☐ Cookie-less request test completed (P29)
☐ Sponsored/ad content identified (P30)
☐ Rate limit test completed (P31)
☐ Open exploration completed if triggered (P-X)
☐ Re-investigation completed if s2_gaps.md provided (P32+)
☐ D2:State updated with final state
☐ D1: Edge Cases Phase Summary written
☐ Final BUDGET_STATUS written (to g6d0.log)
☐ Investigation complete — proceed to compaction (see `references/compaction.md`)
