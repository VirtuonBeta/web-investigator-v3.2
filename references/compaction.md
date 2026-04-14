# Post-Investigation Compaction `v3.2`

Reference file for the Web Investigator (Agent 1 v3.2). Defines the compaction procedure that runs after the investigation completes, before handoff to the analyst.

Read this file AFTER the investigation completes and BEFORE handing off the log files.

---

## Purpose

The raw gate D0 files accumulated during investigation may contain:
- Duplicate observations (same endpoint logged multiple times)
- Stub entries that were expanded but the stub marker wasn't removed
- CDP passive captures that overlap with agent_active observations of the same event
- Entries with minor errors later corrected via errata entries

Compaction produces clean, deduplicated gate D0 files for the downstream analyst. The deliverable remains the set of files (state.log + g1d0.log through g6d0.log) — there is no merge into a single file.

---

## Compaction Procedure

### Step 1: Archive the Originals

```bash
mkdir s1_log_full/
cp state.log s1_log_full/
cp g*d0.log s1_log_full/
```

The original files are preserved in `s1_log_full/`. They are never modified. If the analyst needs the raw, unedited log, they read these files.

### Step 2: Apply Errata

Process all SYSTEM entries with `event: errata` across all gate D0 files. For each:
- Locate the referenced original entry (by `id`).
- Apply the correction to the original entry's fields.
- Remove the errata SYSTEM entry (its content has been applied).

### Step 3: Deduplicate Per Gate

For each gate D0 file, identify and merge duplicate observations:

| Duplicate Type | Detection Rule | Merge Strategy |
|---------------|----------------|----------------|
| Same endpoint, multiple REQUEST entries | Same `method` + `url` (ignoring query params) | Keep the most complete entry (most fields filled). If equally complete, keep the one with the higher cycle number. |
| Same cookie, multiple COOKIE entries | Same `name` + `domain` | Keep the entry with the highest cycle number. Note value changes in `notes`. |
| Same DOM snapshot context | Same `context` enum + same `phase` | Keep the more detailed one. If equivalent, keep one. |
| Stub + expanded version of same entry | Same `id` with `stub: true` and without | Remove the stub. Keep the expanded version. |

**Do NOT deduplicate across gate files.** An endpoint observed in Gate 1 and again in Gate 4 may have different behaviors. Keep both, but note in the later entry's `notes` field that it was also observed in an earlier gate.

### Step 4: Trim Per Gate

For each gate D0 file, remove entries that provide no analytical value:

| Entry Type | Trimming Rule |
|-----------|---------------|
| REQUEST to excluded domains (ads, tracking) | Remove if `source: cdp_passive` and domain matches exclude list from cdp-infrastructure.md |
| SYSTEM entries for auto-exclusions | Remove (only useful during investigation) |
| EDGE_CASE_TEST with `result` matching baseline | Remove if the test confirmed expected behavior with no anomaly |
| CDP health check entries | Remove (infrastructure-only, no analytical value) |

**Do NOT trim entries just because they seem "unimportant."** The analyst decides what's important. Only trim entries that are provably redundant or purely infrastructural.

### Step 5: Clean state.log

D2:State is a single entry (replaced, not appended). No trimming needed.

Keep all D1 summaries — they are the primary context recovery artifact for the analyst.

state.log contains ONLY D2:State and D1 summaries. All typed entries (BUDGET_STATUS, SYSTEM, etc.) are in the gate D0 files — they are compacted in Steps 2-4 above.

### Step 6: Write Compaction Manifest

At the top of state.log, add a manifest:

```
## Compaction Manifest
Source: s1_log_full/ directory
Compacted: cycle {N} (final investigation cycle)
Gate files: g1d0.log through g6d0.log
Removed: 15 duplicates, 7 infrastructural, 3 stubs
Errata applied: 2 corrections
Dedup rules: §Step 3
Trim rules: §Step 4
```

### Step 7: Validate

1. Verify that all entry IDs (`gN:NNN`) still exist in their respective gate files. Gaps in local numbering are acceptable — do NOT renumber IDs. Renumbering risks breaking `gN:NNN` cross-reference chains (e.g., `set_by_request: g1:003`, `corrects_entry: g1:005`).
2. Verify the final D2:State in state.log is valid and current.
3. Verify all D1 summaries have correct `ents:` ranges that match the surviving entries in their gate D0 files.
4. Verify no required fields are missing from any entry in any file.

---

## Important Constraints

- **Compaction is a per-file operation.** Each gate D0 file is compacted independently. state.log needs no compaction — it only contains D2:State (single entry) and D1 summaries (all kept).
- **Compaction runs ONCE, after the investigation.** Do not compact during investigation — you need the full log for context recovery.
- **The analyst receives the set of files** (state.log + g1d0.log through g6d0.log). The originals are available in `s1_log_full/` if they need them, but the compacted versions are the default deliverable.
- **Compaction does NOT add analysis.** It only removes duplicates, applies corrections, and trims noise. No new observations or conclusions are added.

---

## When to Skip Compaction

Skip compaction if:
- The total entries across all gate D0 files are fewer than 30 (deduplication overhead isn't worth it).
- The investigation was interrupted by a BLOCKER (partial logs should remain as-is for forensic value).
- The operator explicitly says "skip compaction."

In these cases, keep the files as-is and note in state.log that compaction was skipped and why.
