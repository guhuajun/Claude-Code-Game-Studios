---
name: test-evidence-review
description: "Quality review of test files and evidence — goes beyond existence, evaluates assertion coverage. ADEQUATE/INCOMPLETE/MISSING/NOT ASSESSED per change."
argument-hint: "[change-id | change set | system-name]"
user-invocable: true
---

!`bash "${CLAUDE_SKILL_DIR}/../../../../hooks/yaml-helper.sh" resolve_config --keys automation`

**Automation mode**: Resolve `modes.automation` (`project.local.yaml` →
`project.yaml` → default `collaborative`). Every `AskUserQuestion` call and
every file write follows `.claude/docs/automation-modes.md`
(collaborative asks always · guided major-only · autonomous logs and proceeds;
`automation_always_ask` categories always prompt).

# Test Evidence Review

`/smoke-check` verifies that test files **exist** and **pass**. This skill
goes further — it reviews the **quality** of those tests and evidence documents.
A test file that exists and passes may still leave critical behaviour uncovered.
A manual evidence doc that exists may lack the sign-offs required for closure.

**Output:** Summary report (in conversation) + optional `production/qa/evidence-review-[date].md`

**When to run:**
- Before QA hand-off sign-off (`/team-qa` Phase 5)
- On any change where test quality is in question
- As part of milestone review for Logic and Integration change quality audit

---

## 1. Parse Arguments

**Modes:**
- `/test-evidence-review [change-id]` — review a single change's evidence
- `/test-evidence-review change set` — review all changes in the current change set
- `/test-evidence-review [system-name]` — review all changes in an capability/system
- No argument — ask which scope: "Single change", "Current change set", "A system"

---

## 2. Load Changes in Scope

Based on the argument:

**Single change**: Read the change directory directly. Extract: Change Type, Test
Evidence section, change slug, system name.

**Change Set**: Read the most recently modified file in `openspec/changes/`; extract
the list of change directory paths from the change list.

**System**: Glob `openspec/specs/[system-name]/change-*.md`.

> **If the resolved scope contains ZERO changes, stop here.** Report
> `NOT ASSESSED — no changes in scope`, name which scope was searched and which
> path was empty, and route: no change list → `/create-changes`; a change list
> listing no changes → `/create-changes [capability-slug]`; a `[system-name]` glob that
> matched nothing → name the glob. Do not continue to Section 3.
>
> **Guard the empty scope, not just the per-change unknown.** The verdict
> vocabulary here — ADEQUATE / INCOMPLETE / MISSING — needs a "could not check"
> value, or an unverifiable change acquires a verdict claiming somebody verified
> it; that is what `NOT ASSESSED` is for. **But giving the per-change unknown a
> home does nothing for the empty-scope unknown.** With no changes
> the Section 6 report renders an empty Summary table and ends
> `BLOCKING items: 0 / ADVISORY items: 0` — which reads as *everything reviewed,
> all fine*. This skill gates change closure, and `coding-standards.md` marks Logic,
> Integration, Visual/Feel and UI evidence BLOCKING, so a false-clean closes changes nobody
> reviewed.
>
> This is a recurring shape: the sophisticated inner rule present, the outer
> boundary unguarded. Ask it of any skill that aggregates —
> **what does this emit when the set is empty?**

For the resulting change set, collect the fields below with **targeted section
greps, not a full read of each change**:
```
Grep pattern="## Test Evidence" glob="openspec/changes/**/tasks.md" output_mode="content" -A 8
Grep pattern="## Acceptance Criteria" glob="openspec/changes/**/tasks.md" output_mode="content" -A 15
```
- **Change Type** (Logic / Integration / Visual/Feel / UI / Config/Data) and the
  stated evidence path — both live under `## Test Evidence`, so the first grep's
  `-A 8` captures them.
- Acceptance Criteria list — the `## Acceptance Criteria` block from the second grep.
- Change slug (from the file name) and System (from the directory path) — no read.
Full-read a change only when its Test Evidence section is missing or ambiguous.
(In Change Set mode, scope the globs to the change list's change ids.)

---

## 3. Locate Evidence Files

For each change, find the evidence:

**Logic changes**: Glob `tests/unit/[system]/[change-slug]_test.*`
  - If not found, also try: Grep in `tests/unit/[system]/` for files
    containing the change slug

**Integration changes**: Glob `tests/integration/[system]/[change-slug]_test.*`
  - Also check `production/session-logs/` for playtest records mentioning the change

**Visual/Feel and UI changes**: Glob `production/qa/evidence/[change-slug]-evidence.*`

**Config/Data changes**: Glob `production/qa/smoke-*.md` (any smoke check report)

Note what was found (path) or not found (gap) for each change.

---

## 4. Review Automated Test Quality (Logic / Integration)

For each test file found, read it and evaluate:

### Assertion coverage

Count the number of distinct assertions (lines containing assert, expect,
check, verify, or engine-specific assertion patterns). Low assertion count is
a quality signal — a test that makes only 1 assertion per test function may
not cover the range of expected behaviour.

Thresholds:
- **3+ assertions per test function** → normal
- **1-2 assertions per test function** → note as potentially thin
- **0 assertions** (test exists but no asserts) → flag as BLOCKING — the
  test passes vacuously and proves nothing

### Edge case coverage

For each acceptance criterion in the change that contains a number, threshold,
or "when X happens" conditional: check whether a test function name or
test body references that specific case.

Heuristics:
- Grep test file for "zero", "max", "null", "empty", "min", "invalid",
  "boundary", "edge" — presence of any is a positive signal
- If the change has a Formulas section with specific bounds: check whether
  tests exercise at minimum/maximum values

### Naming quality

Test function names should describe: the scenario + the expected result.
Pattern: `test_[scenario]_[expected_outcome]`

Flag functions named generically (`test_1`, `test_run`, `testBasic`) as
**naming issues** — they make failures harder to diagnose.

### Formula traceability

For Logic changes where the GDD has a Formulas section: check that the test
file contains at least one test whose name or comment references the formula
name or a formula value. A test that exercises a formula without mentioning
it by name is harder to maintain when the formula changes.

---

## 5. Review Manual Evidence Quality (Visual/Feel / UI)

For each evidence document found, read it and evaluate:

### Criterion linkage

The evidence doc should reference each acceptance criterion from the change.
Check: does the evidence doc contain each criterion (or a clear rephrasing)?
Missing criteria mean a criterion was never verified.

### Sign-off completeness

Check for three sign-off lines (or equivalent fields):
- Developer sign-off
- Designer / art-lead sign-off (for Visual/Feel)
- QA lead sign-off

If any are missing or blank: flag as INCOMPLETE — the change cannot be fully
closed without all required sign-offs.

### Screenshot / artefact completeness

For Visual/Feel changes: Glob `production/qa/evidence/` for a retained image
(`*.png`, `*.jpg`, `*.gif`) belonging to this change. A doc that describes a
visual check but retains no image is INCOMPLETE — this gate is BLOCKING by
default, and a description is an assertion rather than evidence.

For UI changes: require the same retained screenshot of each screen touched,
plus a walkthrough sequence (step-by-step interaction log).

### Date coverage

Evidence doc should have a date. If the date is earlier than the change's
last major change (heuristic: compare against change set start date from the change set
plan), flag as POTENTIALLY STALE — the evidence may not cover the final
implementation.

---

## 6. Build the Review Report

For each change, assign a verdict:

| Verdict | Meaning |
|---------|---------|
| **ADEQUATE** | Test/evidence exists, passes quality checks, all criteria covered |
| **INCOMPLETE** | Test/evidence exists but has quality gaps (thin assertions, missing sign-offs) |
| **MISSING** | No test or evidence found for a change type that requires it |
| **NOT ASSESSED** | The review could not be performed for this change — see below |

> **`NOT ASSESSED` is required, and it is not a softer `MISSING`.**
> The other three verdicts all presume the review actually ran. `MISSING` means
> *I looked and there was nothing there* — a real, reportable failure. It must
> never be used for *I could not look*, which is not a finding about the change
> at all. Use `NOT ASSESSED` when:
>
> - the change's **type cannot be determined**, so the required evidence is
>   unknown (the type→evidence mapping in `.claude/docs/coding-standards.md` is
>   what makes any other verdict meaningful);
> - the evidence path is named but **unreadable or outside this run's scope**;
> - the change directory itself could not be parsed.
>
> **Say which of those it was, per change.** A reader cannot act on a bare
> `NOT ASSESSED`, and the whole point of separating it from `MISSING` is that
> the two have different fixes: one needs a test written, the other needs the
> reviewer given access or the change classified.

The overall change set/system verdict is the worst change verdict present, and
**`NOT ASSESSED` outranks `ADEQUATE`**: a run that could not assess part of its
scope has not established that the scope is adequate. It does not outrank
`INCOMPLETE` or `MISSING` — a known failure is more actionable than an unknown,
and demoting a real failure behind an access problem would bury it.

> Why this needed saying: evidence review **gates change completion**, and
> `coding-standards.md` makes Logic, Integration, Visual/Feel and UI evidence BLOCKING. A verdict
> vocabulary with no way to express "I could not check" forces every unknown
> into one of three claims about the change — which is how a change nobody
> verified acquires a verdict that says somebody did.

```markdown
## Test Evidence Review

> **Date**: [date]
> **Scope**: [single change id | Change Set [N] | [system name]]
> **Changes reviewed**: [N]
> **Overall verdict**: ADEQUATE / INCOMPLETE / MISSING / NOT ASSESSED

---

### Change-by-Change Results

#### [Change Title] — [Type] — [ADEQUATE/INCOMPLETE/MISSING/NOT ASSESSED]

**Test/evidence path**: `[path]` (found) / (not found)

**Automated test quality** *(Logic/Integration only)*:
- Assertion coverage: [N per function on average] — [adequate / thin / none]
- Edge cases: [covered / partial / not found]
- Naming: [consistent / [N] generic names flagged]
- Formula traceability: [yes / no — formula names not referenced in tests]

**Manual evidence quality** *(Visual/Feel/UI only)*:
- Criterion linkage: [N/M criteria referenced]
- Sign-offs: [Developer ✓ | Designer ✗ | QA Lead ✗]
- Artefacts: [screenshots present / missing / N/A]
- Freshness: [dated [date] — current / potentially stale]

**Issues**:
- BLOCKING: [description] *(prevents change-done)*
- ADVISORY: [description] *(should fix before release)*

---

### Summary

| Change | Type | Verdict | Issues |
|-------|------|---------|--------|
| [title] | Logic | ADEQUATE | None |
| [title] | Integration | INCOMPLETE | Thin assertions (avg 1.2/function) |
| [title] | Visual/Feel | INCOMPLETE | QA lead sign-off missing |
| [title] | Logic | MISSING | No test file found |

**BLOCKING items** (must resolve before change can be closed): [N]
**ADVISORY items** (should address before release): [N]
```

---

## 7. Write Output (Optional)

Present the report in conversation.

Ask: "May I write this test evidence review to
`production/qa/evidence-review-[date].md`?"

This is optional — the report is useful standalone. Write only if the user
wants a persistent record.

After the report:

- For BLOCKING items: "These must be resolved before `/change-done` can mark the
  change Complete. Would you like to address any of them now?"
- For thin assertions: "Consider running `/test-helpers [system]` to see
  scaffolded assertion patterns for common cases."
- For missing sign-offs: "Manual sign-off is required from [role]. Share
  `[evidence-path]` with them to complete sign-off."

Verdict: **COMPLETE** — evidence review finished. Use CONCERNS if BLOCKING items were found.

---

## Collaborative Protocol

- **Report quality issues, do not fix them** — this skill reads and evaluates;
  it does not modify test files or evidence documents
- **ADEQUATE means adequate for shipping, not perfect** — avoid nitpicking
  tests that are functioning and comprehensive enough to give confidence
- **BLOCKING vs. ADVISORY distinction is important** — only flag BLOCKING when
  the gap leaves a change criterion genuinely unverified
- **Ask before writing** — the report file is optional; always confirm before writing
