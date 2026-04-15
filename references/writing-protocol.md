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
action: STOP
```

At each gate, execute in order:

1. Write all pending observations → current gate D0 file (`g{N}d0.log`) as typed entries
2. Replace D2:State → top of `state.log` (structured YAML: Phase, Gate, Findings, Dead_ends, Open, Next_steps, Budget, current_d0, Last_checkpoint)
3. Write D1 Phase Summary → `state.log`, inserted ABOVE previous D1 (bottom-up ordering)
4. Write BUDGET_STATUS → current gate D0 file (P8, P13, P16 only; other gates only if budget changed significantly)
5. **HARD_STOP** — think explicitly: "I am at gate {N}, my files are g{N}d0.log and state.log"
6. Spawn review sub-agent → references/gate-review.md prompt template
7. Handle review result (PASS → operator halt; FAIL → fix → re-review; max 2 attempts)
8. Wait for operator to explicitly say "continue" before proceeding to next gate

Skipping a gate violates the output contract. No exceptions. The review sub-agent is NOT optional.

### D1 Format

```yaml
trigger: every_phase_gate
ordering: bottom-up  # newest above oldest
location: state.log
```

```
## D1: {Phase Name} (P{start}-P{end}) | ents: g{N}:{first}-g{N}:{last}

rendering: {SSR|CSR|hybrid|RSC|UNKNOWN}
data_sources: [{source}, ...]  # e.g., ["__NEXT_DATA__", "ld+json", "API fetch"]
api_endpoints: [{count} found | null]  # null if not checked this gate
cookies: {count} observed | null
sitemap: {found|not_found|null}
consent: {none|banner_handled|platform_name|null}
key_selectors: [root, card, ... | null]
dead_ends: [{description} (gN:NNN ✗) | none]
open: [{unresolved question} | none]
budget_at_gate: {cycles_used}/{total}
```

`ents:` is the index — `ents: g1:001-g1:011` means "open g1d0.log, entries g1:001 through g1:011." Insert new D1 ABOVE previous D1. Most recent phase = first D1 after D2:State.

Fields that were not checked during this gate use `null`. Fields checked but nothing found use `none` or `0 found` as appropriate.

### BUDGET_STATUS at Gates

- Append to current gate D0 file (NOT state.log)
- Included in D1 summary's `ents:` range
- If any BUDGET_STATUS field contradicts D2:State → D2 takes precedence. Flag contradiction in BUDGET_STATUS `notes` and update the field to match D2.

### Gate Checklist

```
☐ All D0 observations logged as typed entries in gate D0 file
☐ D0 entries written at 2 scheduled high-density steps per gate (see SKILL.md mid_gate_D0_writes schedule)
☐ D0 contains NO forward-looking content (no next steps, plans, conclusions)
☐ D2:State replaced at top of state.log (structured YAML with all fields)
☐ D1 Phase Summary inserted above previous D1 (ents: range + structured fields included)
☐ BUDGET_STATUS appended to gate D0 file (P8, P13, P16)
☐ No banned phrases in any entry
☐ Core fields present; conditional fields filled if first-of-type or 10-entry gap
☐ Null convention followed: null = checked nothing found, null_not_checked = didn't check
☐ HARD_STOP: explicitly identified gate number and file names
☐ Review sub-agent spawned and result handled
☐ Operator approval received before proceeding
☐ Re-read writing-protocol.md and log-format.md at gate boundary
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
