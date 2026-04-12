# Writing Protocol `v3.1`

Reference file for the Web Investigator (Agent 1 v3.1). Defines the writing discipline rules that keep the log pure, incremental, and reliable.

Read this file BEFORE starting the investigation AND at each phase gate.

---

## 1. Phase Gates

Hard write gates exist at the end of these phases: **P8, P13, P16, P22, P27, P32**.

At each gate you MUST:

1. **Stop what you are doing.** Do not proceed to the next phase.
2. **Write all pending observations** to `s1_log.md` as properly typed entries.
3. **Update D2:State** with current phase, key findings, dead ends, and budget.
4. **Update D1:Phase** if the completed phase produced a summary-worthy finding.
5. **Write BUDGET_STATUS** (at P8, P13, P16 gates — other gates only if budget changed significantly).

**Why hard gates exist:** Without forced write points, agents naturally defer writing ("I'll do it after this one more thing") until they batch-dump everything at the end. This destroys the incremental-write property and means:
- If the run is interrupted, all unwritten observations are lost.
- Timestamps are fabricated (backdated to when the observation happened, not when it was written).
- The analyst gets no intermediate data.

**Skipping a gate violates the output contract.** There are no exceptions.

### Gate Checklist

At each gate, verify:

```
☐ All D0 observations from this phase are logged as typed entries
☐ D2:State is updated with current phase and findings
☐ D1:Phase summary is written (if phase is complete)
☐ BUDGET_STATUS is written (P8, P13, P16)
☐ Entry IDs are sequential and no IDs are skipped
☐ No entry contains banned phrases (see §4)
☐ Re-read the next phase section in the appropriate priority-queue
  reference file (prehalt for phases 0–2, posthalt for phases 3–8)
  BEFORE writing the first entry of that phase
```

---

## 2. Output Channel Discipline

The agent has two output channels:

| Channel | Purpose | Content |
|---------|---------|---------|
| `s1_log.md` | Observation log | All observations, state checkpoints, phase transitions |
| Chat | Operator coordination | Status updates, questions, BLOCKER reports, first-pass halt |

### Chat Rules

Chat is for **coordination only**. The following are appropriate chat messages:
- "First pass complete. 23 entries in s1_log.md. Budget remaining: 27 cycles."
- "BLOCKER: CAPTCHA detected on primary URL. Halting."
- "At P17, found /api/v2/ endpoint. Continuing to P18."
- Asking the operator a question when uncertain.

The following are **NOT** appropriate chat messages:
- Narrating your thought process ("I notice that the page seems to use React because...")
- Repeating observations that belong in the log ("The cookie _ga is a Google Analytics cookie")
- Summarizing findings ("So far I've found 3 API endpoints and 2 auth tokens")
- Explaining why you're doing something ("I'm going to check the localStorage now because...")

**Rule of thumb:** If it's an observation, it goes in the log. If it's coordination (telling the operator where you are, asking for guidance, reporting a blocker), it goes in chat.

### Why This Matters

Chat verbosity has three failure modes:
1. **Context pollution** — chat observations get mixed with actual analysis, creating duplicate/conflicting records.
2. **Log starvation** — if you write it in chat, you're less likely to write it in the log, so the log becomes incomplete.
3. **Analyst confusion** — the analyst reads s1_log.md, not chat. If observations only exist in chat, they're invisible to downstream processing.

---

## 3. Reference Read Schedule

You must re-read reference files at these mandatory points:

| When | What to Re-Read | Why |
|------|-----------------|-----|
| Before starting investigation | This file (`references/writing-protocol.md`) | Fresh load of discipline rules |
| Before each phase gate (P8, P13, P16, P22, P27, P32) | This file → §1 Phase Gates | Checklist reminder before writing |
| Before writing any entry | `references/log-format.md` | Ensure entry matches the spec — required fields, correct types |
| When D2:State shows context_risk: HIGH | `s1_log.md` (D2 + relevant D1) | Recover from context loss |
| Before Phase 4 (deep exploration) | `references/priority-queue-posthalt.md` §Phase 4 | Deep exploration steps are conditional — re-read to know which apply |
| When entering Phase 5 (request replay) | `references/log-format.md` → REQUEST type | Replay entries require complete req_headers/res_headers |
| After investigation completes | `references/compaction.md` | Compaction procedure for final log |

### Why Re-Reads Matter

After reading a reference file once, the agent's memory of it degrades over subsequent reasoning turns. By the time you reach P16, your recollection of the exact REQUEST entry fields from P2 is unreliable. Mandatory re-reads ensure you're working from the spec, not from a degraded memory that might omit required fields.

**Real failure case:** In a Yahoo Finance investigation, the agent wrote REQUEST entries at P17 that were missing `req_headers`, `res_headers`, and `res_body_schema` — all required fields. The agent had read log-format.md once at the start, then never re-read it. By P17, the memory had degraded to "REQUEST entries have url, method, status" — missing most required fields.

---

## 4. Banned Phrases

The following phrases indicate analysis leaking into observations. **Do not use them in any log entry's `notes` field or any other field.**

### Banned List

| Phrase | Why It's Banned | Replacement |
|--------|----------------|-------------|
| "suggests" | Analysis, not observation | State the raw fact; let the analyst infer |
| "means" | Interpretation | State what was observed directly |
| "indicates" | Analysis | State the raw fact |
| "required for" | Conclusion about necessity | State what was observed; note absence/presence |
| "implies" | Inference | State the raw fact |
| "therefore" | Logical conclusion | Remove entirely; observations don't conclude |
| "because" | Causal reasoning | State observations separately; no causation |
| "likely due to" | Speculation | Use `?` notation in D0, or UNKNOWN type |
| "probably" | Probability judgment | Use `~` notation in D0 for "likely/probable" |
| "seems to" | Uncertain analysis | State what is directly observable |
| "appears to be" | Uncertain analysis | State what is directly observable |
| "in order to" | Purpose inference | State the action/observation without purpose |
| "is used for" | Purpose inference | State what exists, not its purpose |

### Self-Check After Writing

After writing each entry, scan the `notes` field for banned phrases. If found:
1. Remove the banned phrase.
2. Replace with a raw observation or remove the sentence entirely.
3. If the observation cannot be stated without analysis, use D0 notation: `? hypothesis` or `~ probable`.

### Example Fixes

**Bad:**
```
notes: Cookie _ga suggests Google Analytics tracking; likely set by GTM
```

**Good:**
```
notes: Cookie _ga value starts with GA1.2; set by page_load; not present in initial HTML
```

**Bad:**
```
notes: 403 response indicates TLS fingerprinting because browser succeeds but raw HTTP fails
```

**Good:**
```
notes: 403 on raw HTTP; 200 on browser for same URL; no HTTP status on connection attempt → ? TLS fingerprinting
```

---

## 5. Phase Discipline (Stay in Your Lane)

Work on one phase at a time. Do not:

- **Skip ahead** to later phases because the current phase "seems simple" — you may miss critical observations.
- **Mix phases** — if you're in Phase 2 (P9-P13), don't start Phase 4 (P17-P22) observations.
- **Revisit completed phases** unless you have an explicit reason (e.g., D2:State context_risk: HIGH, or operator instruction).

### Phase Transitions

When moving from one phase to the next:
1. Complete the current phase gate (see §1).
2. Re-read the next phase section: `references/priority-queue-prehalt.md` for phases 0–2, or `references/priority-queue-posthalt.md` for phases 3–8. This is mandatory, even if you have read it before.
3. Then begin the new phase.

Do not "preview" the next phase while still completing the current one. This leads to premature observations that may be wrong because the current phase hasn't established the necessary context.

---

## 6. Cycle Accounting

A **decision cycle** is one LLM reasoning turn that results in an agent action. Specifically:

| Action | Counts as Cycle? | Why |
|--------|-------------------|-----|
| Navigating to a URL | Yes | Agent decided to navigate |
| Clicking a button | Yes | Agent decided to click |
| Executing JS to inspect DOM | Yes | Agent decided to inspect |
| Triggering pagination | Yes | Agent decided to paginate |
| Writing a log entry | No | Recording, not deciding |
| Reading a reference file | No | Information gathering, not action |
| CDP passive capture | No | Automatic, no agent decision |
| Waiting for network idle | No | Passive, no agent decision |
| Updating D2:State | No | State management, not investigation |

### Reporting

In BUDGET_STATUS entries and D2:State, report cycles as:

```
Budget: 14/50 cycles used
```

The numerator must be a count of decision cycles, not a guess. If you're unsure of the exact count, it's better to slightly overcount (conservative) than undercount.

### Why This Matters

In the Yahoo Finance investigation, the agent reported "21/50 cycles" but couldn't explain what counted as a cycle. The number was unverifiable. A clear definition makes the budget accountable and reproducible.

---

## 7. Quick-Write Stubs

At phase gates, you may not have time to write full detailed entries for every observation. Use the stub pattern:

1. Write a minimal entry with required fields only: `id`, `phase`, `source`, `type`, and one key field.
2. Mark it with `stub: true` in the entry.
3. Return to expand it before the next gate.

**Example stub:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:23:45.123Z  REQUEST

id:                   ent_004
phase:                1
source:               cdp_passive
method:               GET
url:                  https://example.com/api/v1/articles?page=2
stub:                 true
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Later, expand it:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
2025-01-15T10:23:45.123Z  REQUEST

id:                   ent_004
phase:                1
source:               cdp_passive
method:               GET
url:                  https://example.com/api/v1/articles?page=2
trigger:              click
resource_type:        fetch
req_headers:          { "accept": "application/json", "cookie": "..." }
req_body:             null
res_status:           200
res_headers:          { "content-type": "application/json; charset=utf-8" }
res_body_type:        json
res_body_sample:      {"articles":[...],"total":42}
res_body_schema:      { type: "object", properties: { articles: { type: "array" }, total: { type: "number" } } }
res_body_truncated:   false
res_body_total_size:  null
res_size_bytes:       15420
ttlfb_ms:             187
redirect_chain:       []
anomalies:            []
notes:                Paginated API endpoint
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Stub expansion is mandatory.** All stubs must be expanded before the investigation completes. Stubs that are not expanded by the final gate are logged as a SYSTEM entry listing incomplete entries.

---

## 8. Observation Protocol

After writing each observation entry, perform this self-check:

```
1. Does this entry contain any banned phrases? (§4)
   → If yes: remove and rewrite.

2. Does this entry match the log-format.md spec for its type?
   → If no: re-read the spec and fix missing/incorrect fields.

3. Is the timestamp the actual time of writing (not backdated)?
   → Backdating is prohibited. Use the current time.

4. Is this an observation, not a conclusion?
   → If it contains causal language ("because", "therefore"), rewrite as raw facts.

5. Would a downstream analyst be able to use this entry without asking me what I meant?
   → If no: add missing context or clarify field values.
```

This 5-point check takes seconds but catches the most common log quality failures.
