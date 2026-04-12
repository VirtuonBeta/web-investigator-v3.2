# Post-Investigation Compaction `v3.1`

Reference file for the Web Investigator (Agent 1 v3.1). Defines the compaction procedure that runs after the investigation completes, before handoff to the analyst.

Read this file AFTER the investigation completes and BEFORE handing off s1_log.md.

---

## Purpose

The raw s1_log.md accumulated during investigation may contain:
- Duplicate observations (same endpoint logged multiple times)
- Stub entries that were expanded but the stub marker wasn't removed
- CDP passive captures that overlap with agent_active observations of the same event
- Entries with minor errors later corrected via errata entries

Compaction produces a clean, deduplicated log that the downstream analyst (Agent 2) can process efficiently.

---

## Compaction Procedure

### Step 1: Archive the Original

```bash
cp s1_log.md s1_log_full.md
```

The original file is preserved as `s1_log_full.md`. It is never modified. If the analyst needs the raw, unedited log, they read this file.

### Step 2: Apply Errata

Process all SYSTEM entries with `event: errata`. For each:
- Locate the referenced original entry (by `id`).
- Apply the correction to the original entry's fields.
- Remove the errata SYSTEM entry (its content has been applied).

### Step 3: Deduplicate

Identify and merge duplicate observations:

| Duplicate Type | Detection Rule | Merge Strategy |
|---------------|----------------|----------------|
| Same endpoint, multiple REQUEST entries | Same `method` + `url` (ignoring query params) | Keep the most complete entry (most fields filled). If equally complete, keep the one with more recent timestamp. |
| Same cookie, multiple COOKIE entries | Same `name` + `domain` | Keep the most recent entry. Note value changes in `notes`. |
| Same DOM snapshot context | Same `context` enum + same `phase` | Keep the more detailed one. If equivalent, keep one. |
| Stub + expanded version of same entry | Same `id` with `stub: true` and without | Remove the stub. Keep the expanded version. |

**Do NOT deduplicate across phases.** An endpoint observed in Phase 1 and again in Phase 4 may have different behaviors. Keep both, but note in the later entry's `notes` field that it was also observed in an earlier phase.

### Step 4: Trim

Remove entries that provide no analytical value:

| Entry Type | Trimming Rule |
|-----------|---------------|
| REQUEST to excluded domains (ads, tracking) | Remove if `source: cdp_passive` and domain matches exclude list from cdp-infrastructure.md |
| SYSTEM entries for auto-exclusions | Remove (only useful during investigation) |
| EDGE_CASE_TEST with `result` matching baseline | Remove if the test confirmed expected behavior with no anomaly |
| CDP health check entries | Remove (infrastructure-only, no analytical value) |

**Do NOT trim entries just because they seem "unimportant."** The analyst decides what's important. Only trim entries that are provably redundant or purely infrastructural.

### Step 5: Write Compaction Manifest

At the top of the compacted s1_log.md, add a manifest:

```
## Compaction Manifest
Source: s1_log_full.md
Compacted: 2025-01-15T14:30:00Z
Original entries: 87
Compacted entries: 62
Removed: 15 duplicates, 7 infrastructural, 3 stubs
Errata applied: 2 corrections
Dedup rules: §Step 3
Trim rules: §Step 4
```

### Step 6: Validate

1. Verify all entry IDs are sequential and no IDs are skipped (after dedup, IDs should still be sequential — renumber if necessary, noting the original IDs in a mapping comment).
2. Verify D2:State is still accurate (it may reference entry IDs that were renumbered — update references).
3. Verify no required fields are missing from any entry.
4. Verify no banned phrases remain (run a scan per writing-protocol.md §4).

---

## Important Constraints

- **Compaction is a read-modify-write operation.** You read the full log, apply changes, and write the compacted version. You never modify entries in-place during investigation.
- **Compaction runs ONCE, after the investigation.** Do not compact during investigation — you need the full log for context recovery.
- **The analyst receives the compacted log.** The full log is available if they need it, but the compacted version is the default deliverable.
- **Compaction does NOT add analysis.** It only removes duplicates, applies corrections, and trims noise. No new observations or conclusions are added.

---

## When to Skip Compaction

Skip compaction if:
- The log has fewer than 30 entries (deduplication overhead isn't worth it).
- The investigation was interrupted by a BLOCKER (partial logs should remain as-is for forensic value).
- The operator explicitly says "skip compaction."

In these cases, rename `s1_log_full.md` back to the primary (or keep both), and note in a SYSTEM entry that compaction was skipped and why.
