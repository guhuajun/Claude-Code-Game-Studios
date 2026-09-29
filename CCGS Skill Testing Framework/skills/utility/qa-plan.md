# Skill Test Spec: /qa-plan

## Skill Summary

`/qa-plan` generates a structured QA test plan for a feature or change set milestone.
It reads change directorys for the specified change set, extracts acceptance criteria from
each change, cross-references test standards from `coding-standards.md` to assign
the appropriate test type (unit, integration, visual, UI, or config/data), and
produces a prioritized QA plan document.

The skill asks "May I write to `production/qa/qa-plan-change set-NNN.md`?" before
persisting the output. If an existing test plan for the same change set is found, the
skill offers to update rather than replace. The verdict is COMPLETE when the plan
is written. No director gates are used — gate-level change readiness is handled by
`/change-readiness`.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains verdict keyword: COMPLETE
- [ ] Contains "May I write" collaborative protocol language before writing the plan
- [ ] Has a next-step handoff (e.g., `/smoke-check` or `/change-readiness`)

---

## Director Gate Checks

None. `/qa-plan` is a planning utility. Change readiness gates are separate.

---

## Test Cases

### Case 1: Happy Path — Change Set with 4 changes generates full test plan

**Fixture:**
- `openspec/changes/change set-003.md` lists 4 changes with defined acceptance criteria
- Changes span types: 1 logic (formula), 1 integration, 1 visual, 1 UI
- `coding-standards.md` is present with test evidence table

**Input:** `/qa-plan change set-003`

**Expected behavior:**
1. Skill reads change set-003.md and identifies 4 changes
2. Skill reads each change's acceptance criteria
3. Skill assigns test types per coding-standards.md table:
   - Logic change → Unit test (BLOCKING)
   - Integration change → Integration test (BLOCKING)
   - Visual change → Retained screenshot + lead sign-off (BLOCKING)
   - UI change → Retained screenshot of each screen (BLOCKING)
4. Skill drafts QA plan with change-by-change test type breakdown
5. Skill asks "May I write to `production/qa/qa-plan-change set-003.md`?"
6. File is written on approval; verdict is COMPLETE

**Assertions:**
- [ ] All 4 changes are included in the plan
- [ ] Test type is assigned per coding-standards.md (not guessed)
- [ ] Gate level (BLOCKING vs ADVISORY) is noted for each change
- [ ] "May I write" is asked with the correct file path
- [ ] Verdict is COMPLETE

---

### Case 2: Change With No Acceptance Criteria — Flagged as UNTESTABLE

**Fixture:**
- `openspec/changes/change set-004.md` lists 3 changes; one change has empty
  acceptance criteria section

**Input:** `/qa-plan change set-004`

**Expected behavior:**
1. Skill reads all 3 changes
2. Skill detects the change with no AC
3. Change is flagged as `UNTESTABLE — Acceptance Criteria required` in the plan
4. Other 2 changes receive normal test type assignments
5. Plan is written with the UNTESTABLE change flagged; verdict is COMPLETE

**Assertions:**
- [ ] UNTESTABLE label appears for the change with no AC
- [ ] Plan is not blocked — the other changes are still planned
- [ ] Output suggests adding AC to the flagged change (next step)
- [ ] Verdict is COMPLETE (the plan is still generated)

---

### Case 3: Existing Test Plan Found — Offers update rather than replace

**Fixture:**
- `production/qa/qa-plan-change set-003.md` already exists from a previous run
- Change Set-003 has 2 new changes added since the last plan

**Input:** `/qa-plan change set-003`

**Expected behavior:**
1. Skill reads change set-003.md and detects 2 changes not in the existing plan
2. Skill reports: "Existing QA plan found for change set-003 — offering to update"
3. Skill presents the 2 new changes and their proposed test assignments
4. Skill asks "May I update `production/qa/qa-plan-change set-003.md`?" (not overwrite)
5. Updated plan is written on approval

**Assertions:**
- [ ] Skill detects the existing plan file
- [ ] "update" language is used (not "overwrite")
- [ ] Only new changes are proposed for addition — existing entries preserved
- [ ] Verdict is COMPLETE

---

### Case 4: No Changes Found for Change Set — Error with guidance

**Fixture:**
- `openspec/changes/change set-007.md` does not exist
- No other change list matching change set-007

**Input:** `/qa-plan change set-007`

**Expected behavior:**
1. Skill attempts to read change set-007.md — file not found
2. Skill outputs: "No change list found for change set-007"
3. Skill suggests running `/change set-plan` to create the change set first
4. No plan is written; no "May I write" is asked

**Assertions:**
- [ ] Error message names the missing change list
- [ ] `/change set-plan` is suggested as the remediation step
- [ ] No write tool is called
- [ ] Verdict is not COMPLETE (error state)

---

### Case 5: Director Gate Check — No gate; QA planning is a utility

**Fixture:**
- Change Set with valid changes and AC

**Input:** `/qa-plan change set-003`

**Expected behavior:**
1. Skill generates and writes QA plan
2. No director agents are spawned
3. No gate IDs appear in output

**Assertions:**
- [ ] No director gate is invoked
- [ ] No gate skip messages appear
- [ ] Skill reaches COMPLETE without any gate check

---

## Protocol Compliance

- [ ] Reads coding-standards.md test evidence table before assigning test types
- [ ] Assigns BLOCKING or ADVISORY gate level per change type
- [ ] Flags changes with no AC as UNTESTABLE (does not silently skip them)
- [ ] Detects existing plan and offers update path
- [ ] Asks "May I write" before creating or updating the plan file
- [ ] Verdict is COMPLETE when plan is written

---

## Coverage Notes

- The case where `coding-standards.md` is missing (skill cannot assign test types)
  is not fixture-tested; behavior would follow the BLOCKED pattern with a note
  to restore the standards file.
- Multi-change set planning (spanning 2 change sets) is not tested; the skill is designed
  for one change set at a time.
- Config/data change type (balance tuning → smoke check) follows the same
  assignment pattern as other types in Case 1 and is not separately tested.
