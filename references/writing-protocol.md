# Writing Protocol `v3.2`

Reference file for the Web Investigator (Agent 1 v3.2). Defines the writing discipline rules that keep the log pure, incremental, and reliable.

Read this file BEFORE starting the investigation AND at each phase gate.

---

## Quick Reference (read at phase gates; full document available for deep reference)

### Gate Triggers
P8, P13, P16, P22, P27, P31 — STOP, write all pending observations, update D2:State, write D1 phase summary, write BUDGET_STATUS (at P8, P13, P16).

### Banned Phrases
| Banned | Use Instead |
|--------|-------------|
| appears to be / seems to be | Describe the observation directly |
| it is likely that | State the observation + uncertainty marker |
| the site uses | Name the specific technology observed |
| this suggests that | State the raw observation; leave inference to D1/D2 |
| potentially | UNKNOWN or NOT_OBSERVED |
| seems like | Describe what you see |
| probably | State the observation; mark uncertainty explicitly |
| note that | Remove preamble; state the note directly |
| interestingly | Remove; no editorial commentary |
| importantly | Remove; all observations are equally important |
| in order to | to |
| the fact that | Remove; restructure sentence |
| it should be noted | Remove; state directly |
| suggests | State the raw fact; let the analyst infer |
| means | State what was observed directly |
| indicates | State the raw fact |
| required for | Note absence/presence |
| implies | State the raw fact |
| therefore | Remove entirely; observations don't conclude |
| because | State observations separately; no causation |
| likely due to | Use `?` notation in D0, or UNKNOWN type |
| is used for | State what exists, not its purpose |

### Self-Check (after every entry)
1. Is observation separated from analysis? (D0 = raw, D1 = synthesized)
2. Are all mandatory core fields present for this entry type?
3. Any conditional fields that should be filled? (first-of-type or 10-entry gap)
4. Any banned phrases? Search your entry before finalizing.
5. Does this entry add new information, or repeat what's already logged?

### Output Discipline
- Log goes to gate D0 files and state.log ONLY. Never to chat, never to other files.
- One entry per observation/action. Never bundle.
- D0 = observations. D1 = phase summaries. D2 = living state.
- Errata for corrections. Never edit existing entries.
- If any BUDGET_STATUS field contradicts D2:State, D2 takes precedence. Flag the contradiction in the notes field and update the field to match D2.

---

## 1. Phase Gates

Hard write gates exist at the end of these phases: **P8, P13, P16, P22, P27, P31**.

At each gate you MUST:

1. **Stop what you are doing.** Do not proceed to the next phase.
2. **Write all pending observations** to the current gate's D0 file (`g{N}d0.log`) as properly typed entries.
3. **Replace D2:State** at the top of `state.log` with current phase, key findings, dead ends, budget, and `current_d0`.
4. **Write D1 Phase Summary** in `state.log` for the completed phase (see §9 — Mandatory D1 Phase Summaries). Include `ents:` index range (format — pending addition pass).
5. **Write BUDGET_STATUS** — append to `state.log` (at P8, P13, P16 gates — other gates only if budget changed significantly).

**Why hard gates exist:** Without forced write points, agents naturally defer writing ("I'll do it after this one more thing") until they batch-dump everything at the end. This destroys the incremental-write property and means:
- If the run is interrupted, all unwritten observations are lost.
- (Timing rule — pending addition pass)
- The analyst gets no intermediate data.

**Skipping a gate violates the output contract.** There are no exceptions.

### Gate Checklist

At each gate, verify:

```
☐ All D0 observations from this phase are logged as typed entries in the gate D0 file
☐ D2:State is replaced at the top of state.log with current phase and findings
☐ D1 Phase Summary is appended to state.log for the completed phase (MANDATORY — see §9)
☐ BUDGET_STATUS is appended to state.log (P8, P13, P16)
☐ No entry contains banned phrases (see Quick Reference above)
☐ For each entry: all core fields present; conditional fields filled if first-of-type or 10-entry gap
☐ Re-read the next gate file (see SKILL.md → Reference Files for the phase-to-file map)
  BEFORE writing the first entry of that phase
```

---

## 2. Output Channel Discipline

The agent has two output channels:

| Channel | Purpose | Content |
|---------|---------|---------|
| Gate D0 file (`g{N}d0.log`) | Observation log | All raw observations |
| `state.log` | State & summaries | D2:State, D1 summaries, BUDGET_STATUS, key synthesis artifacts |
| Chat | Operator coordination | Status updates, questions, BLOCKER reports, first-pass halt |

### Chat Rules

Chat is for **coordination only**. The following are appropriate chat messages:
- "First pass complete. state.log + g1d0.log + g2d0.log written. Budget remaining: 27 cycles."
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
3. **Analyst confusion** — the analyst reads state.log and gate D0 files, not chat. If observations only exist in chat, they're invisible to downstream processing.

---

## 3. Reference Read Schedule

Read ALL reference files once at investigation start (Phase 0). After that, re-reads follow this schedule:

| When | What to Re-Read | Why |
|------|-----------------|-----|
| At each phase gate (P8, P13, P16, P22, P27, P31) | This file → Quick Reference + §1 Phase Gates | Checklist reminder before writing |
| At each phase gate | This file → §9 Mandatory D1 Phase Summaries | Format reminder for D1 writing |
| Before writing an entry type you haven't written in the last 5 entries | `references/log-format.md` → section for that entry type | Ensure entry matches the spec |
| Before writing an entry type for the FIRST time | `references/log-format.md` → section for that entry type | Full spec compliance on first use |
| At each gate: "Any entry types I've written fewer than 3 times this investigation?" | Re-read those log-format.md sections now | Reinforce rare entry format |
| When context pressure is felt (uncertain about prior state) | `state.log` (last D2:State + relevant D1) | Recover from context loss |
| Before Phase 4 (deep exploration) | `references/gates/gate-4-exploration.md` | Deep exploration steps are conditional — re-read to know which apply |
| When entering Phase 5 (request replay) | `references/log-format.md` → REQUEST type | Replay entries require complete req_headers/res_headers |
| After investigation completes | `references/compaction.md` | Compaction procedure for final log |

**On-demand re-reads are always allowed.** If you're unsure about a field or format, re-read the relevant spec section.

### Why Gate-Based Re-Reads

After reading a reference file once, the agent internalizes the format. What degrades with context distance is compliance on checklist items (banned phrases, self-check) and rare entry types. Gate re-reads (~6 per investigation) target both problems at ~15% of the token cost of per-entry re-reads (~40 per investigation).

**Real failure case:** In a Yahoo Finance investigation, the agent wrote REQUEST entries at P17 that were missing `req_headers`, `res_headers`, and `res_body_schema` — all required fields. The agent had read log-format.md once at the start, then never re-read it. By P17, the memory had degraded to "REQUEST entries have url, method, status" — missing most required fields. Gate-based re-reads with the Quick Reference checklist prevent this.

---

## 4. Banned Phrases

The following phrases indicate analysis leaking into observations. **Do not use them in any log entry's `notes` field or any other field.**

See the Quick Reference at the top of this file for the complete banned phrases list and replacements.

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
- **Revisit completed phases** unless you have an explicit reason (e.g., operator instruction).

### Phase Transitions

When moving from one phase to the next:
1. Complete the current phase gate (see §1).
2. Re-read the next gate file (see SKILL.md → Reference Files for the phase-to-file map). This is mandatory, even if you have read it before.

   | Transition | Read This File |
   |------------|---------------|
   | Starting investigation → Phase 0 | `references/gates/gate-1-baseline.md` |
   | Gate 1 (P8) → Phase 2 | `references/gates/gate-2-pagination.md` |
   | Gate 2 (P13) + operator resumes → Phase 3 | `references/gates/gate-3-inspection.md` |
   | Gate 3 (P16) → Phase 4 | `references/gates/gate-4-exploration.md` |
   | Gate 4 (P22) → Phase 5 | `references/gates/gate-5-replay.md` |
   | Gate 5 (P27) → Phase 6 | `references/gates/gate-6-edgecases.md` |
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
{CYCLE_AND_TYPE}  REQUEST

id:                   {ID}
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
{CYCLE_AND_TYPE}  REQUEST

id:                   {ID}
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
1. Does this entry contain any banned phrases? (Quick Reference)
   → If yes: remove and rewrite.

2. Does this entry match the log-format.md spec for its type?
   → If no: re-read the spec and fix missing/incorrect fields.

3. Are all core fields present? Are conditional fields filled if
   this is the first entry of this type, or there's a 10-entry gap
   since the last entry of this type?
   → If no: fill missing core fields. Fill conditional fields if required.

4. {Time/ordering rule TBD — pending addition pass}

5. Is this an observation, not a conclusion?
   → If it contains causal language ("because", "therefore"), rewrite as raw facts.

6. Would a downstream analyst be able to use this entry without asking me what I meant?
   → If no: add missing context or clarify field values.

7. If writing a BUDGET_STATUS: do any fields contradict D2:State?
   → If yes: D2 takes precedence. Update the field and note the previous value.
```

This 7-point check takes seconds but catches the most common log quality failures.

---

## 9. Mandatory D1 Phase Summaries

At EVERY phase gate (P8, P13, P16, P22, P27, P31), BEFORE proceeding to the next phase, write a D1 section summarizing the completed phase. Format:

```
## D1: {Phase Name} (P{start}-P{end})

{2-5 sentence summary of key findings, dead ends, and open questions from
this phase. Include {ID references — format TBD} for the most
important observations.}
```

This is MANDATORY, not optional. The D1 summary is the primary context recovery artifact. It must be written while context about the phase is fresh — not retroactively.

**Why mandatory:** D1 summaries are the most cost-effective recovery artifact. Written at gate time (when context is fresh), they capture the essence of a phase in ~50-100 tokens. On recovery, reading D2 + all D1 summaries (~500-800 tokens total) gives near-complete state reconstruction vs reading the full D0 log (~7-8K tokens for a 1900-line file).

---

## Context Maintenance Trigger

Every 6 DECISION cycles (not entries — a decision cycle is any entry where the agent chose a non-obvious action: probing, testing, changing direction), the agent MUST:

1. Re-read the last D2:State entry in `state.log`
2. Re-read the most recent D1 phase summary in `state.log`
3. Verify: "Does my next planned action align with D2:State and site_brief?"

This is a mechanical trigger. No self-assessment required. Do it at cycle 6, 12, 18, 24, 30, 36, 42, 48, 54, 60 regardless of how you "feel" about your context state.

What counts as a decision cycle:
- EDGE_CASE_TEST entries (agent-initiated only, not passive detection)
- SYSTEM entries where agent chose to investigate something
- DOM_SNAPSHOT entries triggered by agent choice (not routine)
- REQUEST entries where agent actively probed an endpoint

What does NOT count:
- COOKIE entries (passive capture)
- BUDGET_STATUS entries (routine)
- SYSTEM entries that are automatic (errata, dependency maps)
- CDP passive captures

Total cost: ~200 tokens every 6 cycles (D2 + D1 re-read).

---

## Back-Edit Protection

The log is APPEND-ONLY. Never modify an existing entry.

Errata (the only mechanism for corrections) requires:
- `corrects_entry`: the ID of the original entry (format — pending addition pass)
- `field`: which field is being corrected
- `original_value`: the exact text of the original
- `corrected_value`: the corrected text
- `reason`: MUST include a raw observation that supports the correction

Critical rule: **"If you cannot articulate WHY the original was wrong using a specific raw observation (not a feeling, not 'that doesn't look right'), do NOT write errata. When in doubt, leave the original and add a new observation entry instead."**

This rule ensures corrections are grounded in evidence. If the correction itself is wrong, the original survives untouched.
