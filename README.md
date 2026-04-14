# Web Investigator

An AI agent that explores websites and produces structured observation logs for scraper builders. Give it a target, it investigates, and hands you raw facts — not opinions.

## How It Works

```
You write a site brief → Agent investigates → It produces state.log + g1d0.log through g6d0.log → You (or Agent 2) build a scraper
```

The investigator observes only. It never analyzes, concludes, or recommends. Every claim in the log is a verifiable fact with an entry ID.

## Output Files

The investigation produces **7 files** — one state file and one raw-observation file per gate:

```
output/
├── state.log          ← D2:State (replaced at top) + D1 summaries (bottom-up)
├── g1d0.log           ← Raw observations from Gate 1 (P0–P8a)
├── g2d0.log           ← Raw observations from Gate 2 (P9–P13c)
├── g3d0.log           ← Raw observations from Gate 3 (P14–P16b)
├── g4d0.log           ← Raw observations from Gate 4 (P17–P22)
├── g5d0.log           ← Raw observations from Gate 5 (P23–P27)
└── g6d0.log           ← Raw observations from Gate 6 (P28–P32+)
```

**Why split files?** At each gate, the agent re-reads state to recover context. With a single monolithic log, that re-read grows to 2000+ lines. With split files, reading `state.log` (~150-250 lines) gives the agent its full state + summaries, and it only opens a specific gate D0 file when it needs raw observations.

## Quick Start

### 1. Create a site brief

```bash
cp site-brief-template.md site_brief.md
```

Fill in the target URL, what data you need, and anything you already know about the site. The `geo_requirements` and `known_technology` fields let the agent skip work and adjust its approach.

### 2. Run the investigation

Provide the agent with `SKILL.md`, your `site_brief.md`, and access to the `references/` folder.

The agent runs through baseline and content discovery (P1–P13), then **halts for your review**.

prompt example: 
```
Clone the repo https://github.com/VirtuonBeta/web-investigator and save as xyzzy.

site_brief.md is located at {EXACT_PATH} — you will read it during the Pre-Brief step.

1. Read SKILL.md, then writing-protocol.md.
2. Run the Pre-Brief step from references/gates/gate-1-baseline.md (read the whole file first — it covers Phase 0 through Phase 1).
3. Read other reference files only when SKILL.md, writing-protocol.md, or a gate file tells you to.
4. Begin the investigation from Phase 0 (Pre-Flight).
5. Stop at the first-pass halt (after P13c) and output the full halt message per SKILL.md §First-Pass Halt — include entry count, cycle breakdown, budget remaining, key findings, and next steps.

Write state.log and g1d0.log, g2d0.log to {OUTPUT_DIRECTORY}.

Rules:
- After writing each entry, do the 5-point self-check from writing-protocol.md §8
- Re-read log-format.md before writing ANY entry
- Re-read writing-protocol.md at every phase gate (P8, P13, P16, P22, P27, P31)
- Never proceed past a phase gate without writing to the log files first
- If you're unsure what phase you're in, re-read D2:State in state.log
- Chat is for coordination only. All observations go to gate D0 files, all state to state.log
```

### 3. Review the first pass

The agent produces `state.log` + `g1d0.log` + `g2d0.log`. Check:

- Did it find the content structure and pagination API?
- Were there any BLOCKERs (CAPTCHAs, 403s)?
- Is there enough budget remaining for deeper investigation?

### 4. Continue or stop

Tell the agent to:
- **Continue** — runs item inspection, deep exploration, request replay, and edge cases (P14–P31)
- **Stop** — the first-pass log may be enough for simple sites
- **Adjust** — change budget or add known technology, then continue
prompt example
```You are Agent 1 (Web Investigator v3.2). You previously ran an investigation through the first-pass halt (P13c). Your context was reset. Your prior work is in state.log and g1d0.log/g2d0.log at {PATH}.

Resume procedure:
1. Read SKILL.md
2. Read state.log — start with the last D2:State entry, then read all D1 sections. This IS your context recovery.
3. Read references/gates/gate-3-inspection.md — this is your instruction set for Phase 3 (subsequent phases have their own gate files).
4. Read references/writing-protocol.md — re-fresh the discipline rules.
5. Resume from where D2:State says you left off. Continue writing to the SAME files — state.log for D2:State (replaced at top) and D1 summaries, g3d0.log for Gate 3 raw observations.

The operator has approved continuing past the first-pass halt. Begin Phase 3 (P14-P16b).

Rules still apply:
- 5-point self-check after each entry (writing-protocol.md §8)
- Re-read log-format.md before writing ANY entry
- Re-read writing-protocol.md at every phase gate (P16, P22, P27, P31)
- Chat is for coordination only. Raw observations go to gate D0 files, state goes to state.log
```

### 5. Continue or stop
it should come back with example:
```
Remaining unexplored (optional, 23 cycles available):
Rate limit testing on /xhr/news (P31)
Empty UA test (P28)
Cookie-less request test (P29)
JS bundle analysis (lower priority given /xhr/news discovery)
Log files (state.log + g1d0.log through g6d0.log) are ready for handoff to Agent 2.
```
either let it explore the missing pieces on its own or if all gotten feed to agent 2:


## What It Discovers

| What | How |
|------|-----|
| Pagination APIs | Scans all 7 signal types before classifying — won't miss cursor APIs behind IO loading |
| Extraction stability | Classifies every field by path type (structured data → hashed classes), produces stability matrix |
| Request recipes | Builds a sequenced HTTP request chain: what to send, in what order, with which cookies |
| EU consent impact | Maps consent platforms (OneTrust, TCF, Sourcepoint), tests whether content is gated |
| Fingerprinting type | Distinguishes header-based (easy fix) from TLS-based (hard constraint) |
| Rate limits | 2-tier test finds the fastest speed that returns normal responses |
| Hidden content | Detects collapsed sections, paywalls, consent-gated text |
| Closed Shadow DOM | Pierces closed shadow roots via CDP DOM.getDocument — no content invisible |
| Deep web forms | Finds search/filter forms and probes them for API endpoints |
| CMS backends | Identifies third-party CMS (Sanity, Contentful, Strapi) from API responses |

## File Structure

```
web-investigator/
├── SKILL.md                         # Agent identity and flow overview (WHY)
├── site-brief-template.md           # Per-site input template
├── references/
│   ├── gates/                        # Gate-aligned step procedures (HOW)
│   │   ├── gate-1-baseline.md        # Steps P0–P8a — pre-flight + baseline
│   │   ├── gate-2-pagination.md      # Steps P9–P13c — content discovery + pagination
│   │   ├── gate-3-inspection.md      # Steps P14–P16b — content item inspection
│   │   ├── gate-4-exploration.md     # Steps P17–P22 — deep exploration
│   │   ├── gate-5-replay.md          # Steps P23–P27 — request replay
│   │   └── gate-6-edgecases.md       # Steps P28–P32+ — edge cases + re-investigation
│   ├── writing-protocol.md          # Phase gates, banned phrases, cycle accounting
│   ├── log-format.md                # Entry types, fields, body capture rules
│   ├── compaction.md                # Post-investigation log cleanup
│   └── cdp-infrastructure.md        # CDP setup, capture filter, volume management
├── examples/
│   ├── state.log                    # Example state + summaries output
│   ├── g1d0.log                     # Example Gate 1 raw observations
│   └── g2d0.log                     # Example Gate 2 raw observations
└── README.md                        # This file
```

The agent reads `SKILL.md` first for context, then consults the `references/` files as needed during the investigation. You don't need to read the reference files — they're for the agent.

## Investigation Pipeline

| Phase | Steps | What Happens | Output File |
|-------|-------|-------------|-------------|
| Pre-flight | Pre-Brief → Pre-P2 | Read your brief, validate CDP, warm up the browser | g1d0.log |
| Baseline | P1 → P8a | Map the DOM, find embedded data, catalog cookies, parse robots/sitemap | g1d0.log |
| Content Discovery | P9 → P13c | Find content items, identify pagination, probe search forms | g2d0.log |
| **First-pass halt** | — | **You review before continuing** | — |
| Item Entry | P14 → P16b | Click into items, map extraction paths, detect hidden content | g3d0.log |
| Deep Exploration | P17 → P22 | API probing, token tracing, JS bundle analysis, stability matrix | g4d0.log |
| Request Replay | P23 → P27 | What headers, cookies, and tokens does a scraper need? | g5d0.log |
| Edge Cases | P28 → P31 | Empty UA, cookie-less, rate limits | g6d0.log |
| Re-Investigation | P32+ | Targeted follow-ups from Agent 2 | g6d0.log |

All phases also write to `state.log` for D2:State updates and D1 summaries. All typed entries (BUDGET_STATUS, SYSTEM, COOKIE_DEPENDENCY_MAP, etc.) go to the current gate D0 file.

## EU Sites

If your site brief includes `geo_requirements: ["EU"]` (or if the target is a European domain):

- The agent automatically runs P7c consent flow mapping
- Baseline budget increases by 2 cycles
- Consent state is tested for content visibility impact, not just cookie state

This matters because on sites like DN.se or SVT.se, rejecting consent truncates article text. Without P7c, the log would document a preview as if it were full content.

## Key Artifacts in the Log

| Artifact | Where | What It Tells You |
|----------|-------|-------------------|
| Extraction Stability Matrix | P22 (g4d0.log) | Which fields have stable extraction paths vs. which will break on deploy |
| HTTP Request Chain | P27 (g5d0.log) | Step-by-step recipe for constructing valid scraper requests |
| Cookie Dependency Map | P7 (g1d0.log) | Which cookies to obtain and in what order |
| Extraction Map | P3, P5, P15 (g1d0, g3d0) | Field-to-path mapping with best and fallback paths |
| Consent Flow Map | P7c (g1d0.log, EU only) | Which consent categories gate which content zones |
| Fingerprint Type | P13b (g2d0.log) | Whether fingerprinting is header-based or TLS-based |

## Limitations

| Limitation | What Happens |
|------------|-------------|
| CAPTCHA-protected sites | Detects and stops. Cannot solve CAPTCHAs. |
| Geo-restricted content | Logs the restriction. Cannot change location. |
| Auth-gated sites | Investigates with whatever auth the brief provides |
| Closed Shadow DOM | Pierced via CDP DOM.getDocument { pierce: true }. Content is captured and logged. |
| TLS fingerprinting | Detected and classified. Requires impersonation tools to work around. |
| Deep web behind login | Does NOT probe LOGIN forms — provide credentials instead. |

## License

MIT
