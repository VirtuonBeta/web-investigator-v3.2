# Gate Review Protocol `v3.2`

```yaml
purpose: quality_gate_review
scope: sub-agent_prompt_template, quality_checks, scoring, loop_closure
trigger: "after D0+D1+D2 written at every gate boundary"
read_schedule: [read_by_sub_agent_when_spawned]
```

---

## 1. Overview

At each gate boundary (P8, P13, P16, P22, P27, P31), after the main agent completes D0, D1, and D2 writes, it spawns a **review sub-agent** using the Task tool. The sub-agent evaluates gate output quality against the framework spec. If quality passes → operator reviews. If quality fails → structured feedback → main agent fixes → re-review → loop.

---

## 2. Main Agent Procedure (Before Spawning)

Before spawning the review sub-agent, the main agent MUST explicitly construct the review context. Think through:

1. **Which gate did I just complete?** (1-6)
2. **What phases does this gate cover?** (e.g., Gate 1 = P0-P8a)
3. **What are my output file names?**
   - D0 file: `g{N}d0.log` — the one I just finished writing
   - state.log — contains D2:State (my latest checkpoint) + D1 summaries (my latest gate summary)
4. **What content must I pass to the reviewer?**
   - The full text of `references/writing-protocol.md`
   - The full text of `references/log-format.md`
   - The full text of `output/state.log` (D2:State + D1 summaries)
   - The full text of `output/g{N}d0.log` — **must be the gate D0 I just completed**, not a different gate
5. **How many review attempts have I made for this gate?** (max 2, then halt for operator)

### Main Agent Spawn Prompt Template

The main agent constructs the following prompt and passes it to the Task tool. The main agent MUST include the actual file contents inline — the sub-agent may not have file access.

```
You are a gate quality reviewer for the Web Investigator framework v3.2.

Your job: review the output of GATE {N} (phases P{start}-P{end}) for quality and completeness.

## Spec Files (provided inline below)

The main agent has read and included these spec files for you. Use them as the quality standard:

### SPEC 1: writing-protocol.md
{main agent pastes full content of references/writing-protocol.md here}

### SPEC 2: log-format.md
{main agent pastes full content of references/log-format.md here}

## Gate Output Files (provided inline below)

These are the actual outputs from the gate you are reviewing:

### OUTPUT 1: g{N}d0.log (Gate {N} D0 entries)
{main agent pastes full content of output/g{N}d0.log here — must be the gate just completed}

### OUTPUT 2: state.log (D2:State + D1 summaries)
{main agent pastes full content of output/state.log here — must contain latest D2 and D1}

## What to Evaluate

You are checking GATE {N} output against the framework spec. Focus on:

### A. Format Compliance (gate-pass minimum)
- D0 entries use correct entry format (delimiters, cycle numbers, gN:NNN IDs, phase, source)
- Core fields present for every entry type
- Conditional fields filled when required (first-of-type or 10-entry gap)
- No banned phrases (see writing-protocol.md Banned Phrases table)
- D0 contains ONLY raw observations — no next steps, no plans, no conclusions, no forward-looking statements
- D1 summary present with correct ents: range
- D2:State present at top of state.log with all required fields

### B. Substance Quality (the real test)
- D0 entries contain meaningful observations, specific and detailed, not bare "checked X, found nothing"
- Count null vs. non-null conditional fields across all entries:
  - If >60% of conditional fields are `null` → flag as SUBSTANCE_DEFICIT
  - If >30% of conditional fields are `null_not_checked` → flag as INCOMPLETE_INVESTIGATION
  - List specific fields that are null_not_checked and COULD have been investigated during this gate's phases
- Findings in D2:State are actually backed by D0 entries (spot-check 2-3 claims against gN:NNN references)
- Dead ends in D2:State have evidence references
- D1 summary references specific gN:NNN IDs with concrete details
- The gate's phases (P{start}-P{end}) are actually covered — no major phase steps skipped

### C. Observation Density
- Minimum entry count expectations:
  - Gate 1 (P0-P8a): ≥6 entries (site is unknown, maximum probing)
  - Gate 2 (P9-P13c): ≥4 entries
  - Gate 3 (P14-P16b): ≥3 entries
  - Gate 4 (P17-P22): ≥4 entries
  - Gate 5 (P23-P27): ≥3 entries
  - Gate 6 (P28-P32+): ≥3 entries
- If entry count is below minimum → flag as THIN_OUTPUT
- Check for mid-gate D0 writes: entries should be distributed across the gate, not all batched at the end

## Output Format

Return EXACTLY one of:

### PASS

```
GATE_REVIEW: PASS
gate: {N}
score: {A|B|C}
notes: {optional brief positive note on what was done well}
```

Scoring:
- **A** = Excellent: rich observations, minimal nulls, all phases covered, good D2 substance
- **B** = Good: adequate observations, some nulls but mostly null (checked, nothing found), phases covered
- **C** = Acceptable: meets minimum requirements but barely — operator should note quality concerns

### FAIL (with structured feedback)

```
GATE_REVIEW: FAIL
gate: {N}
issues:
  - category: {FORMAT|SUBSTANCE_DEFICIT|INCOMPLETE_INVESTIGATION|THIN_OUTPUT|D2_UNBACKED|PHASES_SKIPPED}
    severity: {HIGH|MEDIUM}
    detail: {specific description of the problem}
    fix: {concrete instruction for what the main agent should do}
    fields: {list of specific fields/entries affected, if applicable}
  - category: ...
    severity: ...
    detail: ...
    fix: ...
    fields: ...
```

## Rules
- Be specific. "D0 entry g1:003 is missing the res_body_schema field" is good. "Some entries are incomplete" is useless.
- Prioritize substance over format. A format-perfect entry with no useful observations is worse than a slightly imperfect entry with rich observations.
- If you flag INCOMPLETE_INVESTIGATION, you MUST list the specific null_not_checked fields and explain what the main agent should investigate to fill them.
- Do not be lenient to avoid extra work. The review exists because the main agent cannot self-assess reliably.
- You have ONE job: quality assessment. Do not rewrite entries, do not suggest new investigation directions beyond fixing flagged issues.
```

---

## 3. Sub-Agent Response Handling

### Main Agent Receives PASS

```
1. Log to chat: "Gate {N} review: PASS (score: {A|B|C}). Awaiting operator review."
2. STOP. Do NOT proceed to next gate.
3. Wait for operator to review and explicitly say "continue" or "proceed to gate {N+1}".
4. When operator resumes: re-read next gate file, then begin next phase.
```

### Main Agent Receives FAIL

```
1. Read each issue in the structured feedback.
2. Fix issues in order of severity (HIGH first):
   - FORMAT issues → fix the entry format directly
   - SUBSTANCE_DEFICIT → go back and investigate the flagged areas, write new D0 entries
   - INCOMPLETE_INVESTIGATION → investigate the specific null_not_checked fields listed, write findings as D0 entries
   - THIN_OUTPUT → probe additional areas relevant to this gate's phases, write new D0 entries
   - D2_UNBACKED → add evidence references to D2 or correct unsupported claims
   - PHASES_SKIPPED → execute the skipped phase steps, write D0 entries
3. After fixes: update D2:State and D1 if new entries change the summary
4. Increment review_attempt counter
5. Re-spawn review sub-agent with same gate context
```

### Loop Closure

```yaml
max_review_attempts: 2
on_max_attempts_reached:
  action: "halt for operator regardless of review result"
  message: "Gate {N} review: {PASS|FAIL} after {N} attempts. Max retries reached. Awaiting operator decision."
  operator_options:
    - "continue" → proceed to next gate
    - "fix: {instruction}" → apply operator's fix, then continue
    - "re-review" → spawn one more review attempt (operator overrides the limit)
```

---

## 4. Quality Checklist Summary

For quick reference — what the sub-agent checks:

```
FORMAT:
☐ Entry delimiters present (━━━━━)
☐ cycle numbers present and plausible
☐ gN:NNN IDs present with correct gate prefix
☐ Core fields present per entry type
☐ Conditional fields filled when required
☐ No banned phrases
☐ D0 has no forward-looking content
☐ D1 has ents: range
☐ D2:State has all required fields

SUBSTANCE:
☐ D0 entries contain meaningful observations (not bare "null" or "nothing found")
☐ Null density: <60% null, <30% null_not_checked
☐ D2 findings backed by D0 references (spot-check 2-3)
☐ D2 dead_ends have evidence references
☐ D1 references specific gN:NNN IDs
☐ Gate phases actually covered (no major skips)
☐ Entry count meets minimum for this gate

DENSITY:
☐ Entries distributed across gate (mid-gate writes), not all batched at end
☐ Entry count meets gate minimum
```

---

## 5. Null Convention

```yaml
null: "checked this field, found nothing (e.g., no cookies set, no API endpoints detected)"
null_not_checked: "did not check this field during this gate (e.g., ran out of time, not relevant to current phase)"

reviewer_rule:
  - "null is acceptable — the agent looked and found nothing"
  - "null_not_checked is a quality signal — if many fields are null_not_checked in areas the gate should have covered, flag as INCOMPLETE_INVESTIGATION"
  - "distinguish: null_not_checked on a field irrelevant to this gate's phases is fine; null_not_checked on a field central to this gate's phases is a problem"
```

---

## 6. Interaction with Other Framework Rules

```yaml
gate_review_is:
  - mandatory at every gate boundary
  - happens AFTER D0+D1+D2 writes
  - happens BEFORE operator review
  - enforces per-gate quality gates at every boundary

gate_review_is_not:
  - a replacement for operator judgment (operator always has final say)
  - a way to skip gate procedures (all D0/D1/D2 must be written first)
  - optional (every gate gets reviewed, no exceptions)

budget_impact:
  - review sub-agent spawn: does NOT consume main agent decision cycles
  - fixes after FAIL: DOES consume cycles (counted normally)
  - operator wait: does NOT consume cycles
```
