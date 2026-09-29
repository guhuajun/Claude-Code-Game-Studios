# Skill Test Spec: /scope-check

## Skill Summary

`/scope-check` is a Haiku-tier read-only skill that analyzes a feature, change set,
or change for scope creep risk. It reads change set and change directorys and compares them
against the active milestone goals. It is designed for fast, low-cost checks
before or during planning. No director gates are invoked. No files are written.
Verdicts: ON SCOPE, CONCERNS, or SCOPE CREEP DETECTED.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains verdict keywords: ON SCOPE, CONCERNS, SCOPE CREEP DETECTED
- [ ] Does NOT require "May I write" language (read-only skill)
- [ ] Has a next-step handoff (what to do based on verdict)

---

## Director Gate Checks

None. Scope check is a read-only advisory skill; no gates are invoked.

---

## Test Cases

### Case 1: Happy Path — Changes align with milestone goals

**Fixture:**
- `production/milestones/milestone-03.md` lists 3 goals: combat system, enemy AI, level loading
- `openspec/changes/change set-006.md` contains 5 changes, all tagged to one of the 3 goals
- `production/session-state/active.md` references milestone-03 as the active milestone

**Input:** `/scope-check`

**Expected behavior:**
1. Skill reads active milestone goals from milestone-03
2. Skill reads change set-006 changes and checks each against milestone goals
3. All 5 changes map to one of the 3 goals
4. Skill outputs a mapping table: change → milestone goal
5. Verdict is ON SCOPE

**Assertions:**
- [ ] Each change is mapped to a milestone goal in the output
- [ ] Verdict is ON SCOPE when all changes map to milestone goals
- [ ] No files are written
- [ ] Skill does not modify change set or milestone files

---

### Case 2: Scope Creep Detected — Changes introducing systems not in milestone

**Fixture:**
- `production/milestones/milestone-03.md` goals: combat, enemy AI, level loading
- `openspec/changes/change set-006.md` contains 5 changes:
  - 3 changes map to milestone goals
  - 2 changes reference "online leaderboard" and "achievement system" (not in milestone-03)

**Input:** `/scope-check`

**Expected behavior:**
1. Skill reads milestone goals and changes
2. Skill identifies 2 changes with no matching milestone goal
3. Skill names the out-of-scope changes: "Online Leaderboard Feature", "Achievement System Setup"
4. Verdict is SCOPE CREEP DETECTED

**Assertions:**
- [ ] Out-of-scope changes are named explicitly in the output
- [ ] Verdict is SCOPE CREEP DETECTED when any change has no milestone goal match
- [ ] Skill does not automatically remove the changes — findings are advisory
- [ ] Output recommends deferring the out-of-scope changes to a later milestone

---

### Case 3: No Milestone Defined — CONCERNS; scope cannot be validated

**Fixture:**
- `production/session-state/active.md` has no milestone reference
- `production/milestones/` directory exists but is empty
- `openspec/changes/change set-006.md` has 4 changes

**Input:** `/scope-check`

**Expected behavior:**
1. Skill reads active.md — finds no milestone reference
2. Skill checks `production/milestones/` — no milestone files found
3. Skill outputs: "No active milestone defined — scope cannot be validated"
4. Verdict is CONCERNS

**Assertions:**
- [ ] Skill does not error when no milestone is defined
- [ ] Output explicitly states that scope validation requires a milestone reference
- [ ] Verdict is CONCERNS (not ON SCOPE or SCOPE CREEP DETECTED without data)
- [ ] Output suggests running `/milestone-review` or creating a milestone

---

### Case 4: Single Change Check — Evaluated against its parent capability

**Fixture:**
- User targets a single change: `openspec/changes/combat/change-parry-timing.md`
- Change references parent capability: `capability-combat.md`
- `openspec/changes/combat/capability-combat.md` has scope: "melee combat mechanics"
- Change title: "Implement parry timing window" — matches capability scope

**Input:** `/scope-check openspec/changes/combat/change-parry-timing.md`

**Expected behavior:**
1. Skill reads the specified change directory
2. Skill reads the parent capability to get scope definition
3. Skill evaluates change against capability scope — "parry timing" matches "melee combat"
4. Verdict is ON SCOPE

**Assertions:**
- [ ] Single-file argument is accepted (change id, not change set)
- [ ] Skill reads the parent capability referenced in the change directory
- [ ] Change is evaluated against capability scope (not milestone scope) in single-change mode
- [ ] Verdict is ON SCOPE when change matches capability scope

---

### Case 5: Gate Compliance — No gate; PR may be consulted separately

**Fixture:**
- Change Set has 2 SCOPE CREEP changes and 3 ON SCOPE changes
- `review-mode.txt` contains `full`

**Input:** `/scope-check`

**Expected behavior:**
1. Skill reads milestone and change set; identifies 2 scope creep items
2. No director gate is invoked regardless of review mode
3. Skill presents findings with SCOPE CREEP DETECTED verdict
4. Output notes: "Consider raising scope concerns with the Producer before change set begins"
5. Skill ends without writing any files

**Assertions:**
- [ ] No director gate is invoked in any review mode
- [ ] Producer consultation is suggested (not mandated)
- [ ] No files are written
- [ ] Verdict is SCOPE CREEP DETECTED

---

## Protocol Compliance

- [ ] Reads milestone goals and change set/change directorys before analysis
- [ ] Maps each change to a milestone goal (or flags as unmapped)
- [ ] Does not write any files
- [ ] No director gates are invoked
- [ ] Runs on Haiku model tier (fast, low-cost)
- [ ] Verdict is one of: ON SCOPE, CONCERNS, SCOPE CREEP DETECTED

---

## Coverage Notes

- The case where the change list itself does not exist is not tested; the
  skill would output a CONCERNS verdict with a message about missing change set data.
- Partial scope overlap (change touches a milestone goal but also introduces
  new scope) is not explicitly tested; implementation may classify this as
  CONCERNS rather than SCOPE CREEP DETECTED.
