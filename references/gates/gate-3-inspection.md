# Gate 3: Content Item Inspection (P14–P16b)

```yaml
gate_id: 3
title: Content Item Inspection
steps: P14 → P16b
phases: [3]
d0_file: g3d0.log
operator_halt: false
next_gate: references/gates/gate-4-exploration.md
```

Read this file ONLY after the operator resumes the investigation following the first-pass halt. This file provides the HOW. The WHY lives in SKILL.md.

---

## Prerequisites

- [ ] Content items identified (P9) with selectors
- [ ] Pagination mechanism classified (P10) and endpoint captured (P11)
- [ ] Pagination replay tested (P13) — know if raw HTTP works
- [ ] Search forms discovered (P12c) if any exist
- [ ] Rendering classification determined (P6/P5a)
- [ ] If EU site: consent flow mapping completed (P7c)

## Write Targets

| What | File |
|------|------|
| Raw observations (all typed entries including BUDGET_STATUS, site brief verification) | `g3d0.log` |
| D2:State updates | `state.log` |
| D1: Item Inspection Phase Summary | `state.log` |

## Phase Map

| Phase | Steps | Purpose |
|-------|-------|---------|
| 3 | P14 → P16b | Content item entry — verify item structure |

**Write gate at P16.** All pending observations logged, D2:State updated, D1 Phase Summary written, BUDGET_STATUS written to g3d0.log, before proceeding to Gate 4.

---

## Phase 3: Content Item Entry (~3 cycles/item, min 3 items)

**Item selection strategy:** Choose items with **variety** — different positions (first, middle, last), different publishers/sources, different content types. Variety ensures structural observations generalize.

---

### [P14] Click into item — CDP captures all requests

```yaml
step: P14
cycle: true
condition: ALWAYS
log: null
```

Click on a content item and let CDP capture everything.

**Observe:**

| Signal | Reveals |
|--------|---------|
| Item-specific API calls | Dedicated detail endpoint — often cleaner than DOM scraping |
| Analytics/tracking requests | Events fired on item view |
| Third-party content loads | Ads, embeds, social widgets |

**Feeds into:** P18 (token tracing), P23+ (request replay).

---

### [P15] Full DOM snapshot of item detail page

```yaml
step: P15
cycle: true
condition: ALWAYS
log: { type: DOM_SNAPSHOT, context: article_entry_N, required_fields: [extraction_map] }
```

Snapshot the item detail page DOM, organized by zone.

**Zones to capture:**

| Zone | What to look for |
|------|------------------|
| **Header zone** | Title, author, timestamp, publisher |
| **Body zone** | Content container, paragraph types, exclusions (ads, related content mixed in) |
| **Footer zone** | Related items, comments, social sharing |

**Log:** Unique item ID (from URL, data attribute, or embedded JSON). Needed for P22 cross-item comparison.

**Detail page extraction path mapping:**

For each zone, identify every extractable field and its best extraction path. Same taxonomy as P3 (structured_data → semantic_html → aria_role → data_attribute → meta_content → class_semantic → class_hashed).

**Header zone fields to map:**

| Field | Check These Paths (in priority order) |
|-------|---------------------------------------|
| Title/headline | ld+json.headline → h1 → [data-testid*="title"] → .headline → [class_hashed] |
| Author | ld+json.author.name → [rel="author"] → [data-testid*="author"] → .byline → [class_hashed] |
| Publish date | ld+json.datePublished → time[datetime] → [data-testid*="date"] → .date → [class_hashed] |
| Modified date | ld+json.dateModified → time[datetime] (2nd) → [data-testid*="updated"] → [class_hashed] |
| Category/tag | ld+json.articleSection → [data-testid*="category"] → .category → [class_hashed] |
| Image | ld+json.image.url → img[src] within header → [data-testid*="hero"] → [class_hashed] |

**Body zone fields to map:**

| Field | Check These Paths (in priority order) |
|-------|---------------------------------------|
| Article body | ld+json.articleBody → article → [role="main"] → .article-body → [class_hashed] |
| Sub-headings | h2, h3 within body → [data-testid*="subtitle"] → [class_hashed] |
| Embedded media | figure > img, iframe, video → [data-testid*="media"] → [class_hashed] |
| Links | a[href] within body → [data-testid*="link"] → [class_hashed] |

**Footer zone fields to map:**

| Field | Check These Paths (in priority order) |
|-------|---------------------------------------|
| Related articles | a[href] within related section → [data-testid*="related"] → .related → [class_hashed] |
| Tags/categories | a[href*="/tag/"] or a[href*="/category/"] → [data-testid*="tag"] → .tags → [class_hashed] |

For each field, log the BEST available path and ALL fallback paths. If only `[class_hashed]` available, flag as `[brittle: no stable path]`.

**Log:** DOM_SNAPSHOT with context `article_entry_N` MUST include `extraction_map` field containing the complete field-to-path mapping (see log-format.md).

**Feeds into:** P22 (stability matrix).

---

### [P15b] Hidden content element detection

```yaml
step: P15b
cycle: true
condition: ALWAYS
log: { type: EDGE_CASE_TEST, test_id: HIDDEN_CONTENT_REVEALED }
```

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

| Result | Log |
|--------|-----|
| Paywall signals (subscription prompt, login wall, "upgrade to read more") | EDGE_CASE_TEST `PAYWALL_DETECTED` |
| Genuine hidden content revealed (more paragraphs, full article) | EDGE_CASE_TEST `HIDDEN_CONTENT_REVEALED` |

**Safety rules:**

- **Do NOT click `<a>` tags with `href`** — those navigate, they don't expand content.
- **Budget:** Max 3 click attempts per detail page.

**Feeds into:** P30 (ad/sponsored content identification), extraction feasibility assessment.

---

### [P16] Navigate back — note if page re-fetches or serves from cache

```yaml
step: P16
cycle: true
condition: ALWAYS
log: null
```

**What to note:**

| Observation | Implication |
|-------------|-------------|
| Page re-fetches all content | No caching; each navigation is expensive |
| Page serves from cache | bfcache or HTTP cache active; navigation is cheap |
| Page renders differently | State was lost (SPA without scroll restore, or server returns different content) |

**Feeds into:** P23+ (replay strategy), extraction strategy (back-navigation API calls are additional data sources).

---

### [P16b] Site Brief Field Verification

```yaml
step: P16b
cycle: false
condition: AFTER ≥3 items inspected (P14-P16)
log: { type: SYSTEM, event: custom, description: "Site brief field verification" }
```

**Procedure:**

1. Read the Pre-Brief SYSTEM entry (first SYSTEM entry with description `"site_brief read"`). For each `target_field` and `open_question`, check: did your observations address it?
2. If a field is unanswered: note it in D0 as an open question.
3. If a field is answered: note the entry ID(s) that provide the answer.
4. If the Pre-Brief entry is missing: re-read `site_brief.md` directly — specifically `target_data` and `questions` fields.

**Log:** SYSTEM entry with event `custom`, description "Site brief field verification", details containing `{brief_field: entry_id_or_OPEN}`.

---

## Gate 3 Output

Before proceeding to Gate 4, verify:

- [ ] At least 3 content items inspected (P14–P16 per item)
- [ ] Item-specific API calls captured (P14)
- [ ] Detail page extraction map created for each item (P15)
- [ ] Hidden content detection completed per item (P15b)
- [ ] Navigation back behavior noted per item (P16)
- [ ] Site brief field verification completed (P16b)
- [ ] D2:State updated
- [ ] D1: Item Inspection Phase Summary written
- [ ] BUDGET_STATUS written (to g3d0.log)
- [ ] Re-read `references/gates/gate-4-exploration.md` BEFORE writing first entry of Gate 4
