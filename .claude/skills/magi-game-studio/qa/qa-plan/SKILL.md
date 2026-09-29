---
name: qa-plan
description: "QA test plan for a change set — classifies changes by Logic/Integration/Visual/UI, covers automated tests, manual cases, smoke scope."
argument-hint: "[change set | feature: system-name | change: path]"
user-invocable: true
---

!`bash "${CLAUDE_SKILL_DIR}/../../../../hooks/yaml-helper.sh" resolve_config --keys automation,workflow,qa.level,system_overrides`

Resolved above — use as-is. No block → defaults in
`.claude/docs/config-resolution.md`.


# QA Plan

This skill generates a structured QA plan for a change set, feature, or individual
change. It reads all in-scope change directorys and their referenced GDDs, classifies
each change by test type, and produces a plan that tells developers exactly what
to automate, what to verify manually, what the smoke test scope is, and when
to bring in a playtester.

Run this before implementation begins so the team knows upfront what testing work
is required. A test plan written after implementation is a post-mortem, not a
plan.

**Output:** `production/qa/qa-plan-[scope]-[date].md`

---

Every `AskUserQuestion` call follows `.claude/docs/automation-modes.md`
(collaborative asks always · guided major-only · autonomous logs and proceeds;
`automation_always_ask` categories always prompt).

**Workflow tier**: resolve per the GDD's system as each is read (per
`.claude/docs/workflow-modes.md`): `workflow_overrides.system_overrides.<system>`
if the block lists one, else the project value. It sets how many GDD sections
the test plan is mined from — see Phase 2.

**`qa.level`**: at `minimal`, produce only a minimal smoke
plan — drop the automated-test-required rows and the test-file DoD; at `standard`,
a full plan per change type; at `full`, also add per-system coverage targets.
Distinct axis from `workflow` (which sets how many GDD sections are mined).

## Phase 1: Parse Scope

**Argument:** `$ARGUMENTS` (blank = ask user via AskUserQuestion)

Determine scope from the argument:

- **`change set`** — read the most recent file in `openspec/changes/`, extract
  every change directory path referenced. Use `openspec list` for the change list and `openspec status` for task progress — both always available:
  use it as the primary change list and fall back to the change list for change
  metadata.
- **`feature: [system-name]`** — glob `openspec/changes/*/tasks.md`, filter
  to changes whose file path or title contains the system name. Also check the
  capability index file (`spec.md`) in that system's directory.
- **`change: [path]`** — validate that the path exists and load that single file.
- **No argument** — use `AskUserQuestion`:
  - "What is the scope for this QA plan?"
  - Options: "Current change set", "Specific feature (enter system name)",
    "Specific change (enter path)", "Full capability"

After resolving scope, report: "Building QA plan for [N] changes in [scope]."

If a change directory path is referenced but the file does not exist, note it as
MISSING and continue with the remaining changes. Do not fail the entire plan
for one missing file.

> **If the resolved scope contains ZERO changes, stop — do not build a plan.**
> Report `NOT ASSESSED — no changes in scope`, naming which scope was searched
> and which path was empty, and route:
> - no file in `openspec/changes/` → "No change list found. Run `/create-changes`."
> - a change list exists but references no changes → "Change list `[path]` lists no
>   changes. Run `/create-changes [capability-slug]`."
> - `feature:`/`change:` scope matched nothing → name the glob that came back empty.
>
> **The N=0 guard is mandatory.** Without it an empty scope reports "Building QA
> plan for 0 changes" and continues into Phase 4, producing a plan document with
> empty tables — and **a QA plan for zero changes looks exactly like a completed
> QA plan.** This skill gates the hand-off to manual QA, so a false-clean here
> sends a build to QA on the strength of a plan that tested nothing.
>
> Note the shape, because this skill already had the harder half of the rule.
> Phase 2 states **"Never treat an absent section as an absent change"** — the
> sophisticated inner case, correctly handled. The outer boundary, no changes at
> all, had nothing. The same shape appears in `/change-readiness`, where `NOT ASSESSED`
> existed for per-change failures and the empty scope could not reach it.
>
> The rule at the top of this block still stands: one missing file among several
> is MISSING-and-continue. This is the different case where there is no *several*.

---

## Phase 2: Load Inputs

Establish the denominator first — glob the in-scope change directorys and count **N** —
then collect the fields below with **targeted section greps, not a full read of
each change**. A QA plan needs each change's type and acceptance criteria; it does
not need its implementation notes, out-of-scope boundaries or ADR rationale, and
reading N changes whole to reach two sections is where this phase's cost lives:

```
Grep pattern="^## Acceptance Criteria" glob="openspec/changes/**/tasks.md" output_mode="content" -A 15
Grep pattern="^> \*\*(Type|Status|Estimate)\*\*" glob="openspec/changes/**/tasks.md" output_mode="content"
Grep pattern="^## (Context|Dependencies)" glob="openspec/changes/**/tasks.md" output_mode="content" -A 8
```

(Scope the globs to the change list's change ids in `change set` mode.) From those:

- **Change title** and change ID — from the file name and path; no read at all
- **Change Type** field — from the header grep (e.g., `Type: Logic`)
- **Acceptance criteria** — the complete numbered/bulleted list, from the first grep
- **GDD / ADR reference** and **Dependencies** — from the `## Context` grep
- **Estimate** — from the header grep if present
- **Implementation files** and **Engine notes** — only needed for changes whose
  test plan actually turns on them; full-read those individual changes

**Never treat an absent section as an absent change.** If a change matched no
`## Acceptance Criteria`, full-read that one and say so — a change with no
testable criteria is a QA finding in its own right, not a change to skip.

After reading changes, load supporting context once (not per change):

- `design/gdd/systems-index.md` — to understand system priorities and which
  GDDs are approved
- For each unique GDD referenced across all changes: mine test material per that
  system's workflow tier (resolved above) — at `full`, mine all 8 sections; at
  `standard`, mine the test-relevant subset of the required sections
  (**Acceptance Criteria** and **Edge Cases** always, plus **Formulas** for any
  system that defines numeric rules); at `minimal`, **Acceptance
  Criteria only**. Do not load the full GDD
  text. These sections contain the testable requirements, the math to verify, and
  the boundary conditions tests must cover. If an Edge Cases section is absent (or
  not expected at minimal), note per GDD: "No Edge Cases section found — edge case
  coverage will be inferred from acceptance criteria only."
- `docs/architecture/control-manifest.md` — scan for forbidden patterns that
  automated tests should guard against (if the file exists)

If no GDD is referenced in a change, note it as a gap but do not block the plan.
The change will be classified using acceptance criteria alone.

---

## Phase 3: Classify Each Change

For each change, assign a Change Type:

- **If the change already has a `Type:` field in its header**: accept it as-is. Do NOT re-classify or validate against the criteria below — the Type was set by lead-programmer at change creation and is authoritative. Record it as-is.
- **If the `Type:` field is missing**: infer the type from the acceptance criteria using the table below, and note in the report that the type was inferred (not declared). Flag this as a gap — the change should have its Type declared explicitly before implementation begins.

| Change Type | Classification Indicators |
|---|---|
| **Logic** | Acceptance criteria reference calculations, formulas, numerical thresholds, state transitions, AI decisions, data validation, buff/debuff stacking, economy transactions, or any testable computation |
| **Integration** | Criteria involve two or more systems interacting, signals or events propagating across system boundaries, save/load round-trips, network sync, or persistence |
| **Visual/Feel** | Criteria reference animation behaviour, VFX, shader output, "feels responsive", perceived timing, screen shake, particle effects, audio sync, or visual feedback quality |
| **UI** | Criteria reference menus, HUD elements, buttons, screens, dialogue boxes, inventory panels, tooltips, or any player-facing interface element |
| **Config/Data** | Changes are limited to balance tuning values, data files, or configuration — no new code logic is involved |

**Mixed changes** (e.g., a change that adds both a formula and a UI display):
assign the primary type based on which acceptance criteria carry the highest
implementation risk, and note the secondary type. Mixed Logic+Integration or
Visual+UI combinations are the most common.

After classifying all changes, produce a classification summary table in
conversation before proceeding to Phase 4. This gives the user visibility into
how tests will be allocated.

---

## Phase 4: Generate Test Plan

Assemble the full QA plan document. Use this structure:

````markdown
# QA Plan: [Change Set/Feature Name]
**Date**: [date]
**Generated by**: /qa-plan
**Scope**: [N changes across [N systems]]
**Engine**: [engine name — `engine.name` from project.yaml if present and non-empty, else the Engine field from .claude/docs/technical-preferences.md, else "Not configured"]
**Change Set File**: [path to change list if applicable]

---

## Test Summary

| Change | Type | Automated Test Required | Manual Verification Required |
|-------|------|------------------------|------------------------------|
| [change title] | Logic | Unit test — `tests/unit/[system]/` | None |
| [change title] | Integration | Integration test — `tests/integration/[system]/` | Smoke check |
| [change title] | Visual/Feel | None (not automatable) | Screenshot + lead sign-off |
| [change title] | UI | Interaction walkthrough | Manual step-through |
| [change title] | Config/Data | Data validation test | Spot-check in-game values |

---

## Automated Tests Required

*(At `qa.level: minimal`, omit this entire section and blank the Test Summary's
"Automated Test Required" column — automated tests are not required there; list
manual/smoke verification only.)*

### [Change Title] — [Type]
**Test file path**: `tests/[unit|integration]/[system]/[change-slug]_test.[ext]`
**What to test**:
- [Specific formula or rule from the GDD Formulas section]
- [Each named state transition or decision branch]
- [Each side effect that should or should not occur]

**Edge cases to cover**:
- Zero/minimum input values (e.g., 0 damage, empty inventory)
- Maximum/boundary input values (e.g., max level, stat cap)
- Invalid or null input (e.g., missing target, dead entity)
- [Any edge case explicitly called out in the GDD Edge Cases section]

**Estimated test count**: ~[N] unit tests

[If no GDD formula reference was found for this change, note:]
*No formula found in referenced GDD — test cases must be derived from acceptance
criteria directly. Review the GDD Formulas section before writing tests.*

---

## Manual QA Checklist

### [Change Title] — [Type]
**Verification method**: [Screenshot + designer sign-off | Playtest session |
Manual step-through | Comparison against reference footage]
**Who must sign off**: [designer / lead-programmer / qa-lead / art-lead]
**Evidence to capture**: [screenshot of X | video clip of Y | written playtest
notes | side-by-side comparison]

Checklist:
- [ ] [Specific observable condition — concrete and falsifiable]
- [ ] [Another condition]
- [ ] [Every acceptance criterion translated into a manual check item]

*If any criterion uses subjective language ("feels", "looks", "seems"), it must
be supplemented with a specific benchmark or a playtest protocol note.*

---

## Smoke Test Scope

Critical paths to verify before any QA hand-off for this change set:

1. Game launches to main menu without crash
2. New game / new session can be started
3. [Primary mechanic introduced or changed this change set]
4. [Any system with a regression risk from this change set's changes]
5. Save / load cycle completes without data loss (if save system exists)
6. Performance is within budget on target hardware (no new frame spikes)

*Smoke tests are verified by the developer via `/smoke-check`. Reference this
list when running that skill.*

---

## Playtest Requirements

| Change | Playtest Goal | Min Sessions | Target Player Type |
|-------|--------------|--------------|-------------------|
| [change] | [What question must the session answer?] | [N] | [new player / experienced] |

**Sign-off requirement**: Playtest notes must be written to
`production/session-logs/playtest-[change set]-[change-slug].md` and reviewed by
the [designer / qa-lead] before the change can be marked COMPLETE.

If no changes require playtest validation: *No playtest sessions required for
this change set.*

---

## Definition of Done — This Change Set

A change is DONE when ALL of the following are true (at `qa.level: minimal`, drop
the test-file / evidence-document / smoke rows below — only acceptance-criteria
verification is required):

- [ ] All acceptance criteria verified — via automated test result OR documented
      manual evidence (screenshot, video, or playtest notes with sign-off)
- [ ] Test file exists at the specified path for all Logic and Integration changes *(qa.level standard/full)*
- [ ] Manual evidence document exists for all Visual/Feel and UI changes *(qa.level standard/full)*
- [ ] Smoke check passes (run `/smoke-check` before QA hand-off) *(qa.level standard/full)*
- [ ] No regressions introduced
- [ ] Code reviewed (via `/code-review` or documented peer review)
- [ ] Change directory updated to `Status: Complete` (via `/change-done`)
````

When generating content, use the actual change titles, GDD formula text, and
acceptance criteria extracted in Phase 2. Do not use placeholder text — every
test entry should reflect the real requirements of these specific changes.

---

## Phase 5: Write Output

Show the complete plan in conversation (or a summary if the plan is very long),
then ask two questions together using `AskUserQuestion`:

```
question: "Ready to write the QA plan. Choose output options:"
multiSelect: true
options:
  - "Write QA plan to production/qa/qa-plan-[scope]-[date].md"
  - "Also back-fill test case specs into each change directory's ## QA Test Cases section (Recommended — enables /dev-change and /code-review traceability)"
```

If "Write QA plan" is selected: write the plan file exactly as generated — do not truncate.

If "Also back-fill change directorys" is selected: for each Logic and Integration change in scope, edit the change directory at its path. Find the `## QA Test Cases` section and replace its content with the test case specs generated in Phase 4 for that change. If a change has no `## QA Test Cases` section, append it before `## Test Evidence`. For Visual/Feel and UI changes, write the manual verification steps instead of test specs.

After writing:

"QA plan written to `production/qa/qa-plan-[scope]-[date].md`.

Next steps:
- Share this plan with the team before change set implementation begins
- Once all changes are implemented, run `/smoke-check` to gate QA hand-off — not yet, only after implementation is complete
- For Logic/Integration changes, create the test files at the listed paths
  before marking changes done — `/change-done` checks for them"

Silently append to `production/session-state/active.md` (create the file if it does not exist):

```
<!-- QA-PLAN: [date] | System: [system/change set identifier] | Plan written: production/qa/qa-plan-[identifier]-[date].md -->
```

---

## Collaborative Protocol

**Applies in `collaborative` mode (the default).** For `guided` and
`autonomous` modes, see `.claude/docs/automation-modes.md` — the rules below
describe what collaborative mode requires, not universal behavior.

- **Never write the plan without asking** — Phase 5 requires explicit approval.
- **Classify conservatively**: when a change is ambiguous between Logic and
  Integration, classify it as Integration — it requires both unit and
  integration tests.
- **Do not invent test cases** beyond what acceptance criteria and GDD formulas
  support. If a formula is absent from the GDD, flag it rather than guessing.
- **Playtest requirements are advisory**: the user decides whether a playtest
  is warranted for borderline Visual/Feel changes. Flag the case; do not mandate.
- Use `AskUserQuestion` for scope selection when no argument is provided.
  Keep all other phases non-interactive — present findings, then ask once to
  approve the write.
