---
name: bug-triage
description: "Re-evaluate open bugs — priority vs severity, assign to change sets, surface systemic trends. Run when the count grows."
argument-hint: "[change set | full | trend]"
user-invocable: true
---

!`bash "${CLAUDE_SKILL_DIR}/../../../../hooks/yaml-helper.sh" resolve_config --keys automation`

**Automation mode**: Resolve `modes.automation` (`project.local.yaml` →
`project.yaml` → default `collaborative`). Every `AskUserQuestion` call and
every file write follows `.claude/docs/automation-modes.md`
(collaborative asks always · guided major-only · autonomous logs and proceeds;
`automation_always_ask` categories always prompt).

# Bug Triage

This skill processes the open bug backlog into a prioritised, change set-assigned
action list. It distinguishes between **severity** (how bad is the impact?) and
**priority** (how urgently must we fix it?), detects systemic trends, and
ensures no critical bug is lost between change sets.

**Output:** `production/qa/bug-triage-[date].md`

**When to run:**
- Change Set start — assign open bugs to the new change set or backlog
- After `/team-qa` completes and new bugs have been filed
- When the bug count crosses 10+ open items

---

## 1. Parse Arguments

**Modes:**
- `/bug-triage change set` — triage against the current change set; assign fixable bugs
  to the open change backlog; defer the rest
- `/bug-triage full` — full triage of all bugs regardless of change set scope
- `/bug-triage trend` — trend analysis only (no assignment); read-only report
- No argument — run change set mode if a current change set exists, else full mode

---

## 2. Load Bug Backlog

### Step 2a — Discover bug files

Glob for bug reports in priority order:
1. `production/qa/bugs/*.md` — individual bug report files (preferred format)
2. `production/qa/bugs.md` — single consolidated bug log (fallback)
3. Any `production/qa/qa-plan-*.md` "Bugs Found" table (last resort)

If no bug files found:
> "No bug files found in `production/qa/bugs/`. If bugs are tracked in a
> different location, adjust the glob pattern. If no bugs exist yet, there is
> nothing to triage."

Stop and report. Do not proceed if no bugs exist.

**In `trend` mode, do not read full bug bodies.** Trend metrics (volume, severity
mix, by-system, by-date) are computable from the header fields alone:
```
Grep pattern="\*\*(Severity|Priority|Status|System|Category|Reported)\*\*" glob="production/qa/bugs/*.md" output_mode="content"
```
(Bug-report fields are bolded — `**Severity**:`, `- **System**:` — so match the
`**field**` form, not a bare line-start `Field:`.)
Full bug bodies are needed only for the priority-vs-severity **re-evaluation** in
`change set`/`full` modes; `trend` is a read-only report and skips it. (The one
deviation check that needs a change's status — "bug filed against a Complete
change" — is a targeted change-status grep either way, not a bug-body read.)

### Step 2b — Load change set context

Read the most recently modified file in `openspec/changes/` to understand:
- Current change set number / name
- Changes in scope (for assignment target)
- Change Set capacity constraints (if noted)

If no change list exists: note "No change list found — assigning to backlog only."

### Step 2c — Load severity reference

Read `.claude/docs/coding-standards.md` for severity/priority definitions if they
exist. If they do not exist, use the standard definitions in Step 3.

---

## 3. Classify Each Bug

For each bug, extract or infer:

### Severity (impact of the bug)

| Severity | Definition |
|----------|-----------|
| **S1 — Critical** | Game crashes, data loss, or complete feature failure. Cannot proceed past this point. |
| **S2 — High** | Major feature broken but game is still playable. Significant wrong behaviour. |
| **S3 — Medium** | Feature degraded but a workaround exists. Minor wrong behaviour. |
| **S4 — Low** | Visual glitch, cosmetic issue, typo. No gameplay impact. |

### Priority (urgency of the fix)

| Priority | Definition |
|----------|-----------|
| **P1 — Fix this change set** | Blocks QA, blocks release, or is regression from last change set |
| **P2 — Fix soon** | Should be resolved before the next major milestone |
| **P3 — Backlog** | Would be good to fix, but no active blocking impact |
| **P4 — Won't fix / Deferred** | Accepted risk or out of scope for current product scope |

### Assignment

For each P1/P2 bug in `change set` mode:
- Identify which change or capability the fix belongs to
- Check whether the current change set has remaining capacity
- If capacity exists: assign to change set (`Change Set: [current]`)
- If capacity is full: flag as `Priority overflow — consider pulling from change set`

For `full` mode: assign all P1 to current change set, P2 to next change set estimate,
P3+ to backlog.

### Deviation check

Flag bugs that suggest **systematic problems**:
- 3+ bugs from the same system in the same change set → "Potential design or
  implementation quality issue in [system]"
- 2+ S1/S2 bugs in the same change → "Change may need to be reopened and
  re-reviewed before shipping"
- Bug filed against a change marked Complete → "Regression in completed change —
  change should be re-opened in change set tracking"

---

## 4. Trend Analysis

After classifying all bugs, generate trend metrics:

### Volume trends
- Total open bugs: [N]
- Opened this change set: [N]
- Closed this change set: [N]
- Net change: [+N / -N]

### System hot spots
- Which system has the most open bugs?
- Which system has the highest S1/S2 ratio?

### Age analysis
- How many bugs are older than 2 change sets?
- Are any S1/S2 bugs un-assigned (change set = none)?

### Regression indicator
- Any bugs filed against previously-completed changes?
- Count: [N] regression bugs (change reopened implied)

---

## 5. Generate Triage Report

```markdown
# Bug Triage Report

> **Date**: [date]
> **Mode**: [change set | full | trend]
> **Generated by**: /bug-triage
> **Open bugs processed**: [N]
> **Change Set in scope**: [change set name, or "N/A"]

---

## Triage Summary

| Priority | Count | Notes |
|----------|-------|-------|
| P1 — Fix this change set | [N] | [N] assigned to change set, [N] overflow |
| P2 — Fix soon | [N] | Scheduled for next change set |
| P3 — Backlog | [N] | Deferred |
| P4 — Won't fix | [N] | Accepted risk |

**Critical (S1/S2) unfixed count**: [N]

---

## P1 Bugs — Fix This Change Set

| ID | System | Severity | Summary | Assigned to | Change |
|----|--------|----------|---------|-------------|-------|
| BUG-NNN | [system] | S[1-4] | [one-line description] | [change set] | [change id] |

---

## P2 Bugs — Fix Soon

| ID | System | Severity | Summary | Target Change Set |
|----|--------|----------|---------|---------------|
| BUG-NNN | [system] | S[1-4] | [one-line description] | Change Set [N+1] |

---

## P3/P4 Bugs — Backlog / Won't Fix

| ID | System | Severity | Summary | Disposition |
|----|--------|----------|---------|-------------|
| BUG-NNN | [system] | S4 | [one-line description] | Backlog |

---

## Systemic Issues Flagged

[List any patterns from Step 3 deviation check, or "None identified."]

---

## Trend Analysis

**Volume**: [N] open / [+N] net change this change set
**Hot spot**: [system with most bugs]
**Regressions**: [N] bugs against completed changes
**Aged bugs (>2 change sets old)**: [N]

[If N aged S1/S2 bugs > 0:]
> ⚠️ [N] high-severity bugs have been open for more than 2 change sets without
> assignment. These represent accepted risk that should be explicitly reviewed.

---

## Recommended Actions

1. [Most urgent action — usually "fix P1 bugs before QA hand-off"]
2. [Second action — usually "investigate [hot spot system] quality"]
3. [Third action — optional improvement]
```

---

## 6. Write and Gate

Present the report in conversation, then ask:

"May I write this triage report to `production/qa/bug-triage-[date].md`?"

Write only after approval.

After writing:
- If any S1 bugs are unassigned: "S1 bugs must be assigned before the change set
  can be considered healthy. Run `openspec status` to see current capacity."
- If regression bugs exist: "Regressions found — consider re-opening the
  affected changes in change set tracking and running `/smoke-check` to re-gate."
- If no P1 bugs exist: "No P1 bugs — build is in good shape for QA hand-off." Verdict: **COMPLETE** — triage report written.

If user declined write: Verdict: **BLOCKED** — user declined write.

---

## Collaborative Protocol

- **Never close or mark bugs Won't Fix without user approval** — surface them
  as P4 candidates and ask: "Are these acceptable as Won't Fix?"
- **Never auto-assign to a change set at capacity** — flag overflow and let the
  change set owner decide what to pull
- **Severity is objective; priority is a team decision** — present severity
  classifications as recommendations, not mandates
- **Trend data is informational** — do not block work on trend findings alone;
  surface them as observations
