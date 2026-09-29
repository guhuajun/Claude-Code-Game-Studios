# Skill Test Spec: /regression-suite

## Skill Summary

`/regression-suite` maps test coverage to GDD requirements: it reads the
acceptance criteria from change directorys in the current change set (or a specified capability),
then scans `tests/` for corresponding test files and checks whether each AC has
a matching assertion. It produces a coverage report identifying which ACs are
fully covered, partially covered, or untested, and which test files have no
matching AC (orphan tests).

The skill may write a coverage report to `production/qa/` after a "May I write"
ask. No director gates apply. Verdicts: FULL COVERAGE (all ACs have tests),
GAPS FOUND (some ACs are untested), or CRITICAL GAPS (a critical-priority AC
has no test).

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains verdict keywords: FULL COVERAGE, GAPS FOUND, CRITICAL GAPS
- [ ] Contains "May I write" language (skill may write coverage report)
- [ ] Has a next-step handoff (e.g., `/test-setup` if framework missing, `/qa-plan` if plan missing)

---

## Director Gate Checks

None. `/regression-suite` is a QA analysis utility. No director gates apply.

---

## Test Cases

### Case 1: Full Coverage — All ACs in change set have corresponding tests

**Fixture:**
- `openspec/changes/change set-004.md` lists 3 changes with 2 ACs each (6 total)
- `tests/unit/` and `tests/integration/` contain test files that match all 6 ACs
  (by system name and scenario description)

**Input:** `/regression-suite change set-004`

**Expected behavior:**
1. Skill reads all 6 ACs from change set-004 changes
2. Skill scans test files and matches each AC to at least one test assertion
3. All 6 ACs have coverage
4. Skill produces coverage report: "6/6 ACs covered"
5. Skill asks "May I write to `production/qa/regression-change set-004.md`?"
6. File is written on approval; verdict is FULL COVERAGE

**Assertions:**
- [ ] All 6 ACs appear in the coverage report
- [ ] Each AC is marked as covered with the matching test file referenced
- [ ] Verdict is FULL COVERAGE
- [ ] "May I write" is asked before writing the report

---

### Case 2: Gaps Found — 3 ACs have no tests

**Fixture:**
- Change Set has 5 changes with 8 total ACs
- Tests exist for 5 of the 8 ACs; 3 ACs have no corresponding test file or assertion

**Input:** `/regression-suite`

**Expected behavior:**
1. Skill reads all 8 ACs
2. Skill scans tests — 5 matched, 3 unmatched
3. Coverage report lists the 3 untested ACs by change and AC text
4. Skill asks "May I write to `production/qa/regression-[change set]-[date].md`?"
5. Report is written; verdict is GAPS FOUND

**Assertions:**
- [ ] The 3 untested ACs are listed by name in the report
- [ ] Matched ACs are also shown (not only the gaps)
- [ ] Verdict is GAPS FOUND (not FULL COVERAGE)
- [ ] Report is written after "May I write" approval

---

### Case 3: Critical AC Untested — CRITICAL GAPS verdict, flagged prominently

**Fixture:**
- Change Set has 4 changes; one change is Priority: Critical with 2 ACs
- One of the critical-priority ACs has no test

**Input:** `/regression-suite`

**Expected behavior:**
1. Skill reads all changes and ACs, noting which changes are critical priority
2. Skill scans tests — the critical AC has no match
3. Report prominently flags: "CRITICAL GAP: [AC text] — no test found (Critical priority change)"
4. Skill recommends blocking change completion until test is added
5. Verdict is CRITICAL GAPS

**Assertions:**
- [ ] Verdict is CRITICAL GAPS (not GAPS FOUND)
- [ ] Critical priority AC is flagged more prominently than normal gaps
- [ ] Recommendation to block change completion is included
- [ ] Non-critical gaps (if any) are also listed

---

### Case 4: Orphan Tests — Test file has no matching AC

**Fixture:**
- `tests/unit/save_system_test.gd` exists with assertions for scenarios
  not present in any current change's AC list
- Current changes do not reference save system

**Input:** `/regression-suite`

**Expected behavior:**
1. Skill scans tests and cross-references ACs
2. `save_system_test.gd` assertions do not match any current AC
3. Test file is flagged as ORPHAN TEST in the coverage report
4. Report notes: "Orphan tests may belong to a past or future change set, or AC was renamed"
5. Verdict is FULL COVERAGE or GAPS FOUND depending on overall AC coverage
   (orphan tests do not affect verdict, they are advisory)

**Assertions:**
- [ ] Orphan test is flagged in the report
- [ ] Orphan flag includes the filename and suggestion (past change set / renamed AC)
- [ ] Orphan tests do not cause a GAPS FOUND verdict on their own
- [ ] Overall verdict reflects AC coverage only

---

### Case 5: Director Gate Check — No gate; regression-suite is a QA utility

**Fixture:**
- Change Set with changes and test files

**Input:** `/regression-suite`

**Expected behavior:**
1. Skill produces coverage report and writes it
2. No director agents are spawned
3. No gate IDs appear in output

**Assertions:**
- [ ] No director gate is invoked
- [ ] No gate skip messages appear
- [ ] Verdict is FULL COVERAGE, GAPS FOUND, or CRITICAL GAPS — no gate verdict

---

## Protocol Compliance

- [ ] Reads change ACs from change lists before scanning tests
- [ ] Matches ACs to tests by system name and scenario (not file name alone)
- [ ] Flags critical-priority untested ACs as CRITICAL GAPS
- [ ] Flags orphan tests (exist in tests/ but no AC matches)
- [ ] Asks "May I write" before persisting the coverage report
- [ ] Verdict is FULL COVERAGE, GAPS FOUND, or CRITICAL GAPS

---

## Coverage Notes

- The heuristic for matching an AC to a test (by system name + scenario keywords)
  is approximate; exact matching logic is defined in the skill body.
- Integration test coverage is mapped the same way as unit test coverage; no
  distinction in verdicts is made between the two.
- This skill does not run the tests — it maps AC text to test assertions. Test
  execution is handled by the CI pipeline.
