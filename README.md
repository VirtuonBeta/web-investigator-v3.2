# Web Investigator

AI agent that explores websites and produces structured, replayable observation logs. Tell it what to investigate via a site brief; it produces a raw fact log that downstream analysts can trust.

## Core Principle

**Observation ≠ Analysis.** The investigator observes raw facts. It never concludes, recommends, or summarizes. Every claim in the log is an observable fact with an entry ID — the analyst draws conclusions from evidence, not from the investigator's opinions.

## Quick Start

### 1. Create a site brief

Copy `site-brief-template.md` and fill in the target URL, content type, desired fields, and any known technology. Save as `site_brief.md`.

### 2. Run the investigator

Provide the agent with:
1. `SKILL.md` — the main config
2. `site_brief.md` — your target description
3. Access to `references/` — detailed procedures

The investigator runs through baseline + content discovery (P1–P13), then **halts** for your review.

### 3. Review the first pass

The agent produces `s1_log.md` with structured observations. Check:
- Did it find the content structure and pagination?
- Did it detect any BLOCKERs?
- Is the budget remaining sufficient for deeper investigation?

### 4. Continue or stop

Tell the agent to:
- **Continue** — run content item entry, deep exploration, edge cases (P14–P32)
- **Stop** — the first-pass log may be sufficient for simple sites
- **Adjust** — change budget, add known technology, then continue

## Architecture

### File Structure

```
web-investigator/
├── SKILL.md                      # Main agent config — identity, output contract, flow overview
├── references/
│   ├── priority-queue-prehalt.md # Detailed P0–P13c steps — before first-pass halt (the HOW)
│   ├── priority-queue-posthalt.md # Detailed P14–P33+ steps — after operator resumes (the HOW)
│   ├── writing-protocol.md       # Phase gates, banned phrases, cycle accounting
│   ├── log-format.md             # Entry types, fields, body capture rules
│   ├── compaction.md             # Post-investigation log compaction
│   └── cdp-infrastructure.md     # CDP setup, capture filter, volume management
├── site-brief-template.md        # Template for per-site input
├── examples/
│   └── s1_log_example.md         # Example first-pass output
└── README.md                     # This file
```

### Progressive Disclosure

The architecture uses three levels of information:

| Level | File | When to Read |
|-------|------|-------------|
| Overview | SKILL.md | Once at startup — tells you WHO you are, WHAT you produce, and WHY |
| Procedure | references/priority-queue-prehalt.md (P0–P13c), references/priority-queue-posthalt.md (P14–P33+) | Before each phase — tells you HOW to execute each step |
| Specification | references/log-format.md, cdp-infrastructure.md | When writing entries or configuring CDP — tells you the exact format |

### Key Design Decisions (v3.0)

This version was rebuilt from scratch based on production run analysis and research into skill architecture:

| Decision | v2.2 (old) | v3.0 (new) | Why |
|----------|-----------|-----------|-----|
| Worklog vs. deliverable | Two disconnected systems: internal `work.log` + deliverable `s1_log.md` | **Merged:** s1_log.md IS the worklog | Production runs showed agents never produced s1_log.md — the config never clearly connected the two systems |
| Write cadence | "Write throughout investigation" (vague) | **Trigger-based:** write at phase boundaries, context pressure, blockers | Batch-write drift: agents deferred all writing until the end, losing context recovery and risking data loss |
| Native behaviors | Required custom D2:State tracking | **Works with native behaviors:** agent's own todo/planning is fine; D2:State is a checkpoint, not a replacement | Agents naturally used their own todo systems and ignored D2:State — leverage this instead of fighting it |
| Config format | Custom markdown config (~1500 lines) | **SKILL.md format** (<500 lines) + reference files | Too-long configs get skimmed; progressive disclosure keeps the core in context and details available on demand |
| Tone | "MUST/NEVER" enforcement | **Guidance:** explain WHY, not just WHAT | Skills research shows agents comply better with understanding than with rigid commands |
| Context management | No context awareness | **context_risk field** in D2:State | Agents lose track of what's in their context window — explicit risk signaling prevents drift |

## Investigation Pipeline

The investigator works through a priority queue top to bottom. Each phase has a decision-cycle budget.

| Phase | Steps | Cycles | Purpose |
|-------|-------|--------|---------|
| Pre-flight | Pre-P0 → Pre-P2 | 0 | CDP health, warm-up, tech adjustments |
| Baseline | P1 → P8a | ~8 | What IS this site? Rendering, globals, cookies, robots, sitemap |
| Content Discovery | P9 → P13c | ~5 | Content structure, pagination, deep web forms |
| *First-pass halt* | — | — | *Operator reviews before continuing* |
| Item Entry | P14 → P16 | ~3/item | Verify item structure, find hidden content/paywalls |
| Deep Exploration | P17 → P22 | ~10 | APIs, tokens, JS bundles, WebSocket, GraphQL |
| Request Replay | P23 → P27 | ~5 | What does a scraper need to send? |
| Edge Cases | P28 → P32 | ~5 | Mobile, empty UA, cookie-less, rate limits |
| Re-Investigation | P33+ | 5/request | Targeted follow-ups from Agent 2 |

## Capabilities

| Capability | How it Works |
|------------|-------------|
| Deep web discovery | Finds search/filter forms, probes them to discover API endpoints behind form-driven content |
| Crawl trap protection | Caps date-based pagination at 1 month, stops after 3 identical pages, caps opaque cursors at 5 pages |
| Sitemap intelligence | Parses sitemap.xml URLs into pattern clusters, flags hidden architecture and deep web entry points |
| TLS fingerprint detection | Detects when raw HTTP replay fails at connection level while browser succeeds |
| Shadow DOM piercing | Detects open/closed shadow roots, snapshots open root contents |
| RSC streaming detection | Identifies Next.js App Router with React Server Components |
| Hidden content detection | Finds collapsed/hidden elements, distinguishes from paywalls |
| SSR/API source overlap | Checks whether initial page and API pagination return the same items or different sets |
| IntersectionObserver detection | Identifies IO-based lazy loading (viewport intersection, not scroll events) |
| CMS API detection | Identifies third-party CMS backends (Sanity, Contentful, Strapi) |
| robots.txt UA blocking | Detects site-wide UA-specific blocks before making raw HTTP probes |

## Limitations

| Limitation | What Happens | Workaround |
|------------|-------------|------------|
| CAPTCHA-protected sites | Detects and backs off. Does not solve. | Provide session cookies from manual browser |
| Geo-restricted content | Noted in log. Cannot change location. | Run from appropriate geography |
| Auth-gated sites | Investigates with whatever auth the brief provides | Provide credentials in site_brief.md |
| Very large sites | Budget caps ensure bounded investigation | Increase max_cycles in site brief |
| Sites blocking automated browsers | Logged as BLOCKER. Partial results still valuable. | Use residential proxies |
| Closed shadow DOM | Content inaccessible. Logged as UNKNOWN. | No workaround — DOM spec prevents access |
| Deep web behind login forms | Does NOT probe LOGIN forms | Provide credentials so content is visible |

## Output

The investigator produces `s1_log.md` — a structured observation log containing:
- **D2:State** — current phase, key findings, budget, context risk level
- **D1:Phase** — per-phase summaries (persists across context resets)
- **D0:Recent** — raw observations, patterns, dead ends, hypotheses
- **Typed entries** — REQUEST, DOM_SNAPSHOT, COOKIE, LOCAL_STORAGE, EDGE_CASE_TEST, SYSTEM, SESSION, BUDGET_STATUS, UNKNOWN, SERVICE_WORKER

See `references/log-format.md` for the complete entry specification.

## Version History

| Version | Date | Summary |
|---------|------|---------|
| 3.0 | 2026-04-12 | Complete rebuild: SKILL.md format, merged worklog+deliverable, trigger-based writes, context_risk, progressive disclosure |
| 2.2 | 2026-04-12 | Deep web surfacing (P8a, P12c, P13c), crawl trap protection (P10) |
| 2.1 | 2026-04-12 | Gate enrichment, first-pass halt, geo/fingerprint distinction |
| 2.0 | 2026-04-12 | Worklog drift prevention — phase transition gates + operator spot-checks |
| 1.9 | 2026-04-11 | Log tier assessment + confidence floor gate |
| 1.8 | 2026-04-11 | LocalStorage entry type, stress test audit |
| 1.5 | 2026-04-10 | GLM-5 optimization — notation, scope declarations |
| 1.3 | 2026-04-10 | Initial framework |

## License

MIT
