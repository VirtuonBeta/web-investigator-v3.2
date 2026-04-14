# Gate 3: Content Item Inspection (P14–P16b)

Reference file for the Web Investigator (Agent 1 v3.2). Read this file ONLY after the operator resumes the investigation following the first-pass halt.

This file provides the HOW for each investigation step. The WHY lives in SKILL.md.

## Prerequisites from Gates 1–2

Before using this file, you should already have (from `references/gates/gate-1-baseline.md` and `references/gates/gate-2-pagination.md`):
- [ ] Content items identified (P9) with selectors
- [ ] Pagination mechanism classified (P10) and endpoint captured (P11)
- [ ] Pagination replay tested (P13) — know if raw HTTP works
- [ ] Search forms discovered (P12c) if any exist
- [ ] Rendering classification determined (P6/P5a)
- [ ] If EU site: consent flow mapping completed (P7c) — consent state may affect content visibility

If any prerequisite is missing, return to the appropriate gate file before proceeding.

## Write Targets

| What | File | Why |
|------|------|-----|
| Raw observations | `g3d0.log` | Gate-scoped raw data |
| D2:State updates | `state.log` | State checkpoint |
| D1: Item Inspection Phase Summary | `state.log` | Phase completion record |
| BUDGET_STATUS (at P16) | `state.log` | Budget checkpoint |
| Site brief field verification (P16b) | `state.log` | Verification artifact |

## Quick Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| Phase 3 | P14 → P16b | Content item entry — verify item structure |

**Write gate at P16.** All pending observations must be logged, D2:State updated, D1 Phase Summary written, BUDGET_STATUS written, before proceeding to Gate 4.

---

## Phase 3: Content Item Entry (~3 cycles/item, min 3 items)

> **Phase gate reminder:** Before starting Phase 3, verify you have: identified content items (P9), understood pagination (P10-P13), and have at least one successful item click path. If content items aren't identifiable, this phase is blocked — return to P9.

This phase explores individual content items in detail. Each item takes ~3 cycles (click in, snapshot, navigate back). The goal is to understand item structure, detect hidden content, and build a reliable extraction schema.

**Item selection strategy:** Choose items with **variety** — different positions in the list (first, middle, last), different publishers/sources (if applicable), different content types (if applicable). Variety ensures your structural observations generalize rather than being specific to one item type.

---

### [P14] Click into item — CDP captures all requests

Click on a content item and let CDP capture everything that happens. The requests triggered by clicking into an item can reveal:

- **Item-specific API calls** — some sites fetch item detail via a separate API endpoint when you click.
- **Analytics/tracking requests** — what events fire on item view.
- **Third-party content loads** — ads, embeds, social widgets that load on the detail page.

**Why this matters:** If clicking into an item triggers a dedicated API call, that API is often a cleaner data source than scraping the detail page DOM. You may be able to replay this API call directly.

---

### [P15] Full DOM snapshot of item detail page

Snapshot the item detail page's DOM structure, organized by zone. This zone-based approach ensures you capture each functional area independently, which makes selector identification more reliable.

**Zones to capture:**

| Zone | What to look for |
|---|---|
| **Header zone** | Title, author, timestamp, publisher |
| **Body zone** | Content container, paragraph types, exclusions (ads, related content mixed in) |
| **Footer zone** | Related items, comments, social sharing |

**Log:** Unique item ID (from URL, data attribute, or embedded JSON). You'll need this for P22 cross-item comparison.

**Why this matters:** The body zone is where the actual content lives, but it's often polluted with ads, related articles, and other noise. Identifying the clean content container and its exclusions is critical for reliable extraction.

**Detail page extraction path mapping:**

For each zone, identify every extractable field and its best extraction path. This is the detail-page equivalent of the P3 extraction path tagging from gate-1. Use the same extraction path type taxonomy (structured_data → semantic_html → aria_role → data_attribute → meta_content → class_semantic → class_hashed).

**Header zone fields to map:**
| Field | Check These Paths (in priority order) |
|---|---|
| Title/headline | ld+json.headline → h1 → [data-testid*="title"] → .headline → [class_hashed] |
| Author | ld+json.author.name → [rel="author"] → [data-testid*="author"] → .byline → [class_hashed] |
| Publish date | ld+json.datePublished → time[datetime] → [data-testid*="date"] → .date → [class_hashed] |
| Modified date | ld+json.dateModified → time[datetime] (2nd) → [data-testid*="updated"] → [class_hashed] |
| Category/tag | ld+json.articleSection → [data-testid*="category"] → .category → [class_hashed] |
| Image | ld+json.image.url → img[src] within header → [data-testid*="hero"] → [class_hashed] |

**Body zone fields to map:**
| Field | Check These Paths (in priority order) |
|---|---|
| Article body | ld+json.articleBody → article → [role="main"] → .article-body → [class_hashed] |
| Sub-headings | h2, h3 within body → [data-testid*="subtitle"] → [class_hashed] |
| Embedded media | figure > img, iframe, video → [data-testid*="media"] → [class_hashed] |
| Links | a[href] within body → [data-testid*="link"] → [class_hashed] |

**Footer zone fields to map:**
| Field | Check These Paths (in priority order) |
|---|---|
| Related articles | a[href] within related section → [data-testid*="related"] → .related → [class_hashed] |
| Tags/categories | a[href*="/tag/"] or a[href*="/category/"] → [data-testid*="tag"] → .tags → [class_hashed] |

For each field, log the BEST available path and ALL fallback paths. If only `[class_hashed]` is available, flag as `[brittle: no stable path]`.

**Log:** DOM_SNAPSHOT with context `article_entry_N` MUST include `extraction_map` field containing the complete field-to-path mapping (see log-format.md).

---

### [P15b] Hidden content element detection

Many sites hide full content behind "Read More" buttons, collapsible sections, or paywall mechanisms. These hidden elements contain content that's in the DOM but not visible — and they're critical for understanding whether you're getting the full content or just a preview.

**Execute this JS:**

```js
const bodyZone = document.querySelector('[role="main"], main, article, .article-body, .post-body, .entry-content, .story-body');
const container = bodyZone || document.body;
const hiddenElements = [];
const candidates = container.querySelectorAll(
  '[style*="display:none"], [style*="display: none"],' +
  '[style*="visibility:hidden"], [style*="visibility: hidden"],' +
  '[style*="height:0"], [style*="height: 0"],' +
  '[aria-hidden="true"],' +
  '[class*="collapsed"], [class*="expand"],' +
  '[class*="read-more"], [class*="continue"]'
);
candidates.forEach(el => {
  const text = el.textContent?.trim() || '';
  if (text.length > 100) {
    hiddenElements.push({
      tag: el.tagName,
      selector: el.id ? '#' + el.id : el.className ? '.' + el.className.split(' ').filter(c=>c).join('.') : el.tagName.toLowerCase(),
      hiddenMechanism: el.style.display === 'none' ? 'display:none' :
        el.style.visibility === 'hidden' ? 'visibility:hidden' :
        el.getAttribute('aria-hidden') === 'true' ? 'aria-hidden' :
        el.classList.contains('collapsed') ? 'class:collapsed' :
        'class-based (' + Array.from(el.classList).join(',') + ')',
      textLength: text.length,
      textSample: text.substring(0, 200),
      isLink: el.tagName === 'A' && el.hasAttribute('href'),
      isClickable: window.getComputedStyle(el).cursor === 'pointer' ||
        el.tagName === 'BUTTON' || el.getAttribute('role') === 'button'
    });
  }
});
JSON.stringify(hiddenElements);
```

**For each clickable non-navigation element (up to 3 per detail page):**

1. Click the element.
2. Wait 2 seconds (or network idle if XHR detected).
3. Re-snapshot the area.

**Interpret the result:**

- **Paywall signals detected** (e.g., subscription prompt, login wall, "upgrade to read more"): Log EDGE_CASE_TEST `PAYWALL_DETECTED`.
- **Genuine hidden content revealed** (more paragraphs, full article): Log EDGE_CASE_TEST `HIDDEN_CONTENT_REVEALED`.

**Safety rules:**

- **Do NOT click `<a>` tags with `href`** — those navigate to a new page, they don't expand content.
- **Budget:** Max 3 click attempts per detail page.

**Why this matters:** Hidden content detection is the difference between extracting a 50-word preview and the full 2000-word article. It also detects paywalls, which fundamentally change extraction feasibility.

---

### [P16] Navigate back — note if page re-fetches or serves from cache

After examining a detail page, navigate back to the listing page. This observation reveals the site's caching strategy and navigation behavior.

**What to note:**

- **Does the page re-fetch all content?** → No caching, or cache-control headers prevent it. Each navigation is expensive.
- **Does the page serve from cache?** → bfcache or HTTP cache is active. Navigation is cheap.
- **Does the page render differently?** → State was lost (SPA that doesn't restore scroll position, or server that returns different content on revisit).

**Why this matters:** If the listing page re-fetches on every back navigation, you know that repeated visits are expensive for the server. This affects how aggressively you can navigate back and forth during investigation. It also affects extraction strategy — if back-navigation triggers new API calls, those calls are additional data sources to capture.

---

### [P16b] Site Brief Field Verification

After completing P14-P16 for at least 3 items, cross-reference your findings against the operator's requirements. This ensures the investigation actually covers what the operator asked about.

**Procedure:**

1. Read the Pre-Brief SYSTEM entry (the first SYSTEM entry with description `"site_brief read"`). For each `target_field` and `open_question` listed in its details, check: did your observations address it?
2. If a field is unanswered: note it in D0 as an open question.
3. If a field is answered: note the entry ID(s) that provide the answer.
4. If the Pre-Brief entry is missing (investigation started before Pre-Brief was added): re-read `site_brief.md` directly — specifically the `target_data` and `questions` fields.

**Log:** SYSTEM entry with event `custom`, description "Site brief field verification", details containing a mapping of `{brief_field: entry_id_or_OPEN}`.

**Why this matters:** Without this check, it's easy to complete the investigation and realize you never answered the operator's actual question. The brief may ask about auth mechanisms while you spent all your budget on content structure. This step ensures alignment between investigation output and operator needs. The Pre-Brief entry is the reliable reference point because it was written at startup when context was fresh — it captures every field and question the operator cared about.

**Does NOT consume a full decision cycle** — it's a verification step that references a single log entry.

---

## Gate 3 Output

Before proceeding to Gate 4, verify:

☐ At least 3 content items inspected (P14–P16 per item)
☐ Item-specific API calls captured (P14)
☐ Detail page extraction map created for each item (P15)
☐ Hidden content detection completed per item (P15b)
☐ Navigation back behavior noted per item (P16)
☐ Site brief field verification completed (P16b)
☐ D2:State updated
☐ D1: Item Inspection Phase Summary written
☐ BUDGET_STATUS written
☐ Re-read `references/gates/gate-4-exploration.md` BEFORE writing first entry of Gate 4
