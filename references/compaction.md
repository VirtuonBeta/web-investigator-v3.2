---
version: "3.2"
type: terminal_procedure
read_when: after gate 6 completes, before handoff
deliverable: "state.log + g1d0.log through g6d0.log (compacted, no merge)"
---

# Post-Investigation Compaction `v3.2`

```yaml
purpose:
  - deduplicate gate D0 files
  - apply errata corrections
  - trim provably redundant entries
does_not:
  - add analysis or new observations
  - merge files into single output
  - modify entries during investigation
```

---

## Procedure

### Step 1: Archive

```bash
mkdir s1_log_full/
cp state.log s1_log_full/
cp g*d0.log s1_log_full/
```

Originals preserved in `s1_log_full/`. Never modified.

### Step 2: Apply Errata

```yaml
process: SYSTEM entries with event: errata across all gate D0 files
actions:
  - locate referenced original entry by id
  - apply correction to original entry's fields
  - remove the errata SYSTEM entry
```

### Step 3: Deduplicate Per Gate

| Duplicate Type | Detection Rule | Merge Strategy |
|---------------|----------------|----------------|
| Same endpoint, multiple REQUEST | Same `method` + `url` (ignoring query params) | Keep most complete entry; if equal, keep higher cycle number |
| Same cookie, multiple COOKIE | Same `name` + `domain` | Keep highest cycle number; note value changes in `notes` |
| Same DOM snapshot context | Same `context` enum + same `phase` | Keep more detailed one; if equivalent, keep one |
| Stub + expanded version | Same `id` with `stub: true` and without | Remove stub, keep expanded |

```yaml
rule: do NOT deduplicate across gate files
reason: "same endpoint in different gates may have different behaviors"
action_on_cross_gate: "note in later entry's notes field that it was also observed in earlier gate"
```

### Step 4: Trim Per Gate

| Entry Type | Trimming Rule |
|-----------|---------------|
| REQUEST to excluded domains (ads, tracking) | Remove if `source: cdp_passive` and domain matches exclude list from cdp-infrastructure.md |
| SYSTEM entries for auto-exclusions | Remove |
| EDGE_CASE_TEST with `result` matching baseline | Remove if confirmed expected behavior with no anomaly |
| CDP health check entries | Remove |

```yaml
rule: "do NOT trim entries just because they seem unimportant"
arbiter: "the analyst decides what's important; only trim provably redundant or purely infrastructural"
```

### Step 5: Clean state.log

```yaml
D2_State: "single entry (replaced, not appended); no trimming needed"
D1_summaries: "keep all — primary context recovery artifact for the analyst"
typed_entries: "none in state.log; all compacted in Steps 2-4 via gate D0 files"
```

### Step 6: Write Compaction Manifest

Add manifest **above D2:State** at the top of state.log (D2:State remains the first operational entry the agent reads on recovery):

```
## Compaction Manifest
Source: s1_log_full/ directory
Compacted: cycle {N} (final investigation cycle)
Gate files: g1d0.log through g6d0.log
Removed: {X} duplicates, {Y} infrastructural, {Z} stubs
Errata applied: {W} corrections
```

### Step 7: Validate

```yaml
checks:
  - all entry IDs (gN:NNN) still exist in their gate files; do NOT renumber (breaks cross-reference chains like set_by_request, corrects_entry)
  - final D2:State is valid and current
  - all D1 summaries have correct ents: ranges matching surviving entries
  - no required fields missing from any entry in any file
```

---

## Constraints

```yaml
scope: per-file (each gate D0 compacted independently; state.log needs no compaction)
timing: once, after investigation (not during — need full log for context recovery)
deliverable: "set of files (state.log + g1d0.log through g6d0.log); originals in s1_log_full/"
no_analysis: true  # only removes duplicates, applies corrections, trims noise
```

---

## When to Skip

```yaml
skip_if:
  - total entries across all gate D0 files < 30
  - investigation interrupted by BLOCKER (partial logs retain forensic value)
  - operator says "skip compaction"
action_if_skipped: "keep files as-is; note in state.log that compaction was skipped and why"
```
