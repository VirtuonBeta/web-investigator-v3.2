# Writing Protocol v3.2

```yaml
purpose: writing_discipline
scope: when_to_write, where_to_write, how_to_write, what_not_to_write
read_schedule: [investigation_start, each_phase_gate]
```

---

## 1. Gate Procedure

```yaml
gate_steps: [P8, P13, P16, P22, P27, P31]
action: HARD_STOP
```

At each gate boundary, execute in order — do NOT proceed to next gate until all steps complete:

1. Write any remaining D0 observations → current gate D0 file (`g{N}d0.log`) as typed entries
2. Replace D2:State → top of `state.log` (structured format: phase, findings, dead_ends, open, next_steps, budget)
3. Write D1 Gate Summary → `state.log`, inserted ABOVE previous D1 (bottom-up ordering)
4. Write BUDGET_STATUS → current gate D0 file (P8, P13, P16 only; other gates only if budget changed significantly)
5. Re-read `references/writing-protocol.md` and `references/log-format.md` ⚠️ even if you believe you remember them — they must be at top of context
6. Re-read next gate file (mandatory even if read before)

**D0 is written DURING the gate, not batched at the boundary.** Write D0 entries 2-4 times per gate — after every 2-3 P-steps or after any significant observation. The gate boundary step 1 catches any final pending observations only. If step 1 produces more than 3-4 entries, you're batching too much — write more frequently during the gate.

Skipping a gate boundary procedure violates the output contract. No exceptions.

### D1 Format

```yaml
trigger: every_gate_boundary
ordering: bottom-up  # newest above oldest
location: state.log
```

```yaml
## D1: {Gate Name} (P{start}-P{end}) | ents: g{N}:{first}-g{N}:{last}

rendering: {SSR/CSR/RSC/hybrid}
data_sources: [{list of embedded data blocks found}]
api_endpoints: [{list with key properties}]
cookies: {summary with key dependency refs}
sitemap: {findings}
consent: {findings}
key_selectors: {root, card, type classification}
dead_ends: [{ruled-out paths with gN:NNN evidence}]
open: [{unresolved questions}]
budget_at_gate: {N}/{total}
```

Include only fields relevant to this gate. Reference key D0 entries with gN:NNN. `ents:` is the index — `ents: g1:001-g1:011` means "open g1d0.log, entries g1:001 through g1:011." Insert new D1 ABOVE previous D1. Most recent gate = first D1 after D2:State.

D1 must carry enough substance that a reader can understand the gate's findings without reading D0. Count substance, not lines.

### BUDGET_STATUS at Gates

- Append to current gate D0 file (NOT state.log)
- Included in D1 summary's `ents:` range
- If any BUDGET_STATUS field contradicts D2:State → D2 takes precedence. Flag contradiction in BUDGET_STATUS `notes` and update the field to match D2.

### Gate Checklist

```
☐ All D0 observations logged as typed entries in gate D0 file
☐ D2:State replaced at top of state.log
☐ D1 Phase Summary inserted above previous D1 (ents: range included)
☐ BUDGET_STATUS appended to gate D0 file (P8, P13, P16)
☐ No banned phrases in any entry
☐ Core fields present; conditional fields filled if first-of-type or 10-entry gap
☐ Re-read writing-protocol.md and log-format.md ⚠️ even if you remember them
☐ Re-read next gate file BEFORE first entry of next phase
```

---

## 2. Entry Rules

Applied to every entry written.

### Output Channels

| Channel | Content | File |
|---------|---------|------|
| Gate D0 | All typed entries (REQUEST, DOM_SNAPSHOT, COOKIE, BUDGET_STATUS, SYSTEM, EDGE_CASE_TEST, COOKIE_DEPENDENCY_MAP, consent_flow_map, etc.) | `g{N}d0.log` |
| state.log | D2:State (replaced at top) + D1 summaries (bottom-up) | `state.log` |
| Chat | Coordination only: status, blockers, questions, first-pass halt | — |

No typed entries belong in state.log. Chat is for coordination only — never narrate, summarize, or explain reasoning in chat. Observations → log. Status/blockers/questions → chat.

### Self-Check (after every entry)

```
1. Observation separated from analysis? (D0 = raw, D1 = synthesized)
2. All core fields present for this entry type?
3. Conditional fields filled if first-of-type or 10-entry gap?
4. No banned phrases?
5. Adds new information, not repeating what's already logged?
6. cycle: N matches current decision cycle count? id: gN:NNN has correct gate prefix?
7. If BUDGET_STATUS: any field contradicts D2:State? → D2 takes precedence
```

### Banned Phrases

Do not use in any entry field.

| Banned | Use Instead |
|--------|-------------|
| appears to be / seems to be | Describe observation directly |
| it is likely that | Observation + uncertainty marker |
| the site uses | Name specific technology observed |
| this suggests that | Raw observation; inference goes in D1/D2 |
| potentially | UNKNOWN or NOT_OBSERVED |
| seems like | Describe what you see |
| probably | Observation + explicit uncertainty |
| note that | Remove preamble; state directly |
| interestingly / importantly | Remove; no editorial |
| in order to | to |
| the fact that | Remove; restructure |
| it should be noted | Remove; state directly |
| suggests / indicates / implies | State raw fact |
| means | State what was observed |
| required for | Note absence/presence |
| therefore | Remove; observations don't conclude |
| because | State observations separately; no causation |
| likely due to | `?` notation in D0, or UNKNOWN |
| is used for | State what exists, not purpose |

**Fix pattern:** Remove banned phrase → replace with raw observation or remove sentence. If observation cannot be stated without analysis → use `? hypothesis` or `~ probable`.

### Cycle Accounting

| Action | Cycle? |
|--------|--------|
| Navigate to URL | yes |
| Click button | yes |
| Execute JS to inspect DOM | yes |
| Trigger pagination | yes |
| Write log entry | no |
| Read reference file | no |
| CDP passive capture | no |
| Wait for network idle | no |
| Update D2:State | no |

Report format: `Budget: 14/50 cycles used`. If unsure of exact count → slightly overcount. Never undercount.

### Null Convention

Conditional fields that are checked but yield nothing → `null`. Conditional fields that were not checked → `null_not_checked`.

| Value | Meaning | When to Use |
|-------|---------|-------------|
| `null` | Checked, nothing found | Tested/observed and the result was empty or absent |
| `null_not_checked` | Not checked, no data | Field was not evaluated during this observation |

Example: `req_body: null` means "request had no body." `req_body: null_not_checked` means "I didn't check the request body." Every `null_not_checked` is a gap — fill it on next observation of the same type if possible.

### Quick-Write Stubs

When time-pressured at a gate:

1. Write minimal entry: `cycle: N TYPE`, `id`, `phase`, `source`, one key field
2. Mark with `stub: true`
3. Expand before next gate

All stubs must be expanded before investigation completes. Unexpanded stubs → SYSTEM entry listing incomplete entries.

---

## 3. Corrections

Log is append-only. Never modify an existing entry.

Errata = SYSTEM entry with `event: errata`:

| Field | Content |
|-------|---------|
| `corrects_entry` | Gate-qualified ID (`gN:NNN`) of original |
| `field` | Field being corrected |
| `original_value` | Exact text of original |
| `corrected_value` | Corrected text |
| `reason` | MUST include a raw observation supporting the correction |

### Errata Rules

1. One correction per errata entry. Multiple fields → multiple entries.
2. `corrects_entry` must use gN:NNN format.
3. `reason` must include a specific raw observation supporting the correction. (e.g., "Cookie rotates on each request" ✅; "Cookie is clearly a session cookie" ❌)
4. Original entry unchanged during investigation. Errata applied during compaction (→ references/compaction.md).
5. Missing required field: include the field's value in `corrected_value`.

If you cannot articulate WHY the original was wrong using a specific raw observation → do NOT write errata. Add a new observation entry instead.
