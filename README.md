# Web Investigator

An AI agent that explores websites and produces structured observation logs for scraper builders. Give it a target, it investigates, and hands you raw facts — not opinions.

## How It Works

```
You write a site brief → Agent investigates → It produces s1_log.md → You (or Agent 2) build a scraper
```

The investigator observes only. It never analyzes, concludes, or recommends. Every claim in the log is a verifiable fact with an entry ID.

## Quick Start

### 1. Create a site brief

```bash
cp site-brief-template.md site_brief.md
```

Fill in the target URL, what data you need, and anything you already know about the site. The `geo_requirements` and `known_technology` fields let the agent skip work and adjust its approach.

### 2. Run the investigation

Provide the agent with `SKILL.md`, your `site_brief.md`, and access to the `references/` folder.

The agent runs through baseline and content discovery (P1–P13), then **halts for your review**.

### 3. Review the first pass

The agent produces `s1_log.md`. Check:

- Did it find the content structure and pagination API?
- Were there any BLOCKERs (CAPTCHAs, 403s)?
- Is there enough budget remaining for deeper investigation?

### 4. Continue or stop

Tell the agent to:
- **Continue** — runs item inspection, deep exploration, request replay, and edge cases (P14–P32)
- **Stop** — the first-pass log may be enough for simple sites
- **Adjust** — change budget or add known technology, then continue

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
| Deep web forms | Finds search/filter forms and probes them for API endpoints |
| CMS backends | Identifies third-party CMS (Sanity, Contentful, Strapi) from API responses |

## File Structure

```
web-investigator/
├── SKILL.md                         # Agent identity and flow overview (WHY)
├── site-brief-template.md           # Per-site input template
├── references/
│   ├── priority-queue-prehalt.md    # Steps P0–P13c — before first-pass halt (HOW)
│   ├── priority-queue-posthalt.md   # Steps P14–P33+ — after operator resumes (HOW)
│   ├── writing-protocol.md          # Phase gates, banned phrases, cycle accounting
│   ├── log-format.md                # Entry types, fields, body capture rules
│   ├── compaction.md                # Post-investigation log cleanup
│   └── cdp-infrastructure.md        # CDP setup, capture filter, volume management
├── examples/
│   └── s1_log_example.md            # Example first-pass output
└── README.md                        # This file
```

The agent reads `SKILL.md` first for context, then consults the `references/` files as needed during the investigation. You don't need to read the reference files — they're for the agent.

## Investigation Pipeline

| Phase | Steps | What Happens |
|-------|-------|-------------|
| Pre-flight | Pre-Brief → Pre-P2 | Read your brief, validate CDP, warm up the browser |
| Baseline | P1 → P8a | Map the DOM, find embedded data, catalog cookies, parse robots/sitemap |
| Content Discovery | P9 → P13c | Find content items, identify pagination, probe search forms |
| **First-pass halt** | — | **You review before continuing** |
| Item Entry | P14 → P16b | Click into items, map extraction paths, detect hidden content |
| Deep Exploration | P17 → P22 | API probing, token tracing, JS bundle analysis, stability matrix |
| Request Replay | P23 → P27 | What headers, cookies, and tokens does a scraper need? |
| Edge Cases | P28 → P32 | Mobile, empty UA, cookie-less, rate limits |
| Re-Investigation | P33+ | Targeted follow-ups from Agent 2 |

## EU Sites

If your site brief includes `geo_requirements: ["EU"]` (or if the target is a European domain):

- The agent automatically runs P7c consent flow mapping
- Baseline budget increases by 2 cycles
- Consent state is tested for content visibility impact, not just cookie state

This matters because on sites like DN.se or SVT.se, rejecting consent truncates article text. Without P7c, the log would document a preview as if it were full content.

## Key Artifacts in the Log

| Artifact | Where | What It Tells You |
|----------|-------|-------------------|
| Extraction Stability Matrix | P22 | Which fields have stable extraction paths vs. which will break on deploy |
| HTTP Request Chain | P27 | Step-by-step recipe for constructing valid scraper requests |
| Cookie Dependency Map | P7 | Which cookies to obtain and in what order |
| Extraction Map | P3, P5, P15 | Field-to-path mapping with best and fallback paths |
| Consent Flow Map | P7c (EU only) | Which consent categories gate which content zones |
| Fingerprint Type | P13b | Whether fingerprinting is header-based or TLS-based |

## Limitations

| Limitation | What Happens |
|------------|-------------|
| CAPTCHA-protected sites | Detects and stops. Cannot solve CAPTCHAs. |
| Geo-restricted content | Logs the restriction. Cannot change location. |
| Auth-gated sites | Investigates with whatever auth the brief provides |
| Closed Shadow DOM | Content is inaccessible. Logged as UNKNOWN. |
| TLS fingerprinting | Detected and classified. Requires impersonation tools to work around. |
| Deep web behind login | Does NOT probe LOGIN forms — provide credentials instead. |

## License

MIT
