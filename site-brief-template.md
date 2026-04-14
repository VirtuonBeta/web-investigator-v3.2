# Site Brief Template `v3.2`

This file is the per-site input that tells the investigator WHAT to investigate.

Copy this template, fill in the fields, and save as `site_brief.md`. The investigator reads this file at startup to understand the target and adjust its approach.

---

## Target

```yaml
url: https://example.com/target-page
name: "Short human-readable name for this target"
```

## What I'm After

Describe what content you need to extract from this site. Be specific about fields.

```yaml
content_type: news_articles | product_listings | forum_posts | directory_listings | social_feed | search_results | custom
description: |
  I need to scrape [content type] from this page.

  For each item in the list, I need:
    - title: string
    - url: string
    - [field]: [type]

  For each item's detail page, I need:
    - headline: string
    - author: string (nullable)
    - publish_date: datetime (nullable)
    - body: HTML
    - [field]: [type]

  I need pagination to get all items, not just the first page.
  I need to know the minimum request (headers, cookies, auth) to get valid data.
```

## Authentication (if any)

```yaml
auth_required: false

# If auth is required, provide credentials or instructions:
# auth_type: cookie | bearer_token | api_key | login_flow | custom
# credentials:
#   username: ""
#   password: ""
# Or provide cookies directly:
# cookies:
#   - name: "session_id"
#     value: "..."
#     domain: "example.com"
```

## Special Considerations

Note anything you know about the site that might affect the investigation.

```yaml
known_issues: []
# Examples:
# - "Site shows EU consent wall for European IPs"
# - "Content behind soft paywall — first 3 articles free"
# - "Site uses Cloudflare protection"
# - "Rate limits aggressively — slow down"

known_technology: []
# Examples:
# - "Built with Next.js App Router (RSC streaming, NOT Pages Router)"
# - "Uses GraphQL API"
# - "Content served from Sanity CMS (cdn.sanity.io)"
# - "Uses Akamai Bot Manager for anti-bot"
# - "Modern bundler with hashed filenames (no descriptive names)"

geo_requirements: []
# Examples:
# - "Must access from US IP — content restricted by region"
# - "EU IP triggers GDPR consent wall"

additional_domains: []
# Examples:
# - "api.example.com — known API subdomain"
# - "cdn.example.com — static assets"
```

## Budget Override (optional)

Leave blank to use the default from the agent config.

```yaml
budget:
  max_cycles: null        # null = use default (60)
  max_pages: null         # null = use default (15). Counts unique page navigations only.
  priority_overrides: null  # null = use default priority queue
  # Example override:
  # max_cycles: 80
  # priority_overrides:
  #   - item: "P19"
  #     priority: HIGH
```

## Re-Investigation (optional — only for round 2+)

Re-investigation is handled via a separate file (`s2_gaps.md`), not by modifying this target file. The workflow is:

1. After first investigation, Agent 2 may produce `s2_gaps.md` with specific targeted requests.
2. Agent 1 reads `s2_gaps.md` as a separate input and executes the requests, appending results to the current gate D0 file and updating D2:State + D1 in state.log.
3. Agent 2 re-reads the full log and produces an updated `s2_analysis.md`.

This target file is never modified after the initial investigation begins.
