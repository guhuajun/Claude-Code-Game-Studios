# Skill Test Spec: /estimate

## Skill Summary

`/estimate` estimates task or change effort using a relative-size scale (S / M /
L / XL) based on change complexity, acceptance criteria count, and historical
change set velocity from past change lists. Estimates are advisory and are never
written automatically. No director gates are invoked. Verdicts are effort ranges,
not pass/fail — every run produces an estimate.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains size labels: S, M, L, XL (the "verdict" equivalents for this skill)
- [ ] Does NOT require "May I write" language (advisory output only)
- [ ] Has a next-step handoff (how to use the estimate in change set planning)

---

## Director Gate Checks

None. Estimation is an advisory informational skill; no gates are invoked.

---

## Test Cases

### Case 1: Happy Path — Clear change with known tech stack

**Fixture:**
- `openspec/changes/combat/change-hitbox-detection.md` exists with:
  - 4 clear Acceptance Criteria
  - ADR reference (Accepted status)
  - No "unknown" or "TBD" language in change body
- `openspec/changes/change set-003.md` through `change set-005.md` exist with velocity data
- Tech stack is GDScript (well-understood by team per change set history)

**Input:** `/estimate openspec/changes/combat/change-hitbox-detection.md`

**Expected behavior:**
1. Skill reads the change directory — assesses clarity, AC count, tech stack
2. Skill reads change set history to determine average velocity
3. Skill outputs estimate: M (1–2 days) with reasoning
4. No files are written

**Assertions:**
- [ ] Estimate is M for a clear, well-scoped change with known tech
- [ ] Reasoning references AC count, tech stack familiarity, and velocity data
- [ ] Estimate is presented as a range (e.g., "1–2 days"), not a single point
- [ ] No files are written

---

### Case 2: High Uncertainty — Unknown system, no ADR yet

**Fixture:**
- `openspec/changes/online/change-lobby-matchmaking.md` exists with:
  - 2 vague Acceptance Criteria (using "should" and "TBD")
  - No ADR reference — matchmaking architecture not yet decided
  - References new subsystem ("online/matchmaking") with no existing source files

**Input:** `/estimate openspec/changes/online/change-lobby-matchmaking.md`

**Expected behavior:**
1. Skill reads change — finds vague AC, no ADR, no existing source
2. Skill flags multiple uncertainty factors
3. Estimate is L–XL with an explicit risk note: "Estimate range is wide due to architectural unknowns"
4. Skill recommends creating an ADR before development begins

**Assertions:**
- [ ] Estimate is L or XL (not S or M) when significant unknowns exist
- [ ] Risk note explains the specific unknowns driving the wide range
- [ ] Output recommends resolving architectural questions first
- [ ] No files are written

---

### Case 3: No Change Set Velocity Data — Conservative defaults used

**Fixture:**
- Change file exists and is well-defined
- `openspec/changes/` is empty — no historical change sets

**Input:** `/estimate openspec/changes/core/change-save-load.md`

**Expected behavior:**
1. Skill reads change — assesses complexity
2. Skill attempts to read change set velocity data — finds none
3. Skill notes: "No change set history found — using conservative defaults for velocity"
4. Estimate is produced using default assumptions (e.g., 1 change point = 1 day)
5. No files are written

**Assertions:**
- [ ] Skill does not error when no change set history exists
- [ ] Output explicitly notes that conservative defaults are being used
- [ ] Estimate is still produced (not blocked by missing velocity)
- [ ] Conservative defaults produce a higher (not lower) estimate range

---

### Case 4: Multiple Changes — Each estimated individually plus change set total

**Fixture:**
- User provides a change list: `openspec/changes/change set-007.md` with 4 changes
- Change Set history exists (3 previous change sets)

**Input:** `/estimate openspec/changes/change set-007.md`

**Expected behavior:**
1. Skill reads change list — identifies 4 changes
2. Skill estimates each change individually: S, M, M, L
3. Skill computes change set total: approximately 6–8 change points
4. Skill presents per-change estimates followed by change set total
5. No files are written

**Assertions:**
- [ ] Each change receives its own estimate label
- [ ] Change Set total is presented after individual estimates
- [ ] Total is a sum range derived from individual ranges
- [ ] Skill handles change lists (not just single change directorys) as input

---

### Case 5: Gate Compliance — No gate; estimates are informational

**Fixture:**
- Change file exists with medium complexity
- `review-mode.txt` contains `full`

**Input:** `/estimate openspec/changes/core/change-item-pickup.md`

**Expected behavior:**
1. Skill reads change and change set history; computes estimate
2. No director gate is invoked in any review mode
3. Estimate is presented as advisory output only
4. Skill notes: "Use this estimate in /change set-plan when selecting changes for the next change set"

**Assertions:**
- [ ] No director gate is invoked regardless of review mode
- [ ] Output is purely informational — no approval or write prompt
- [ ] Next-step recommendation references `/change set-plan`
- [ ] Estimate does not change based on review mode

---

## Protocol Compliance

- [ ] Reads change directory before estimating
- [ ] Reads change set velocity history when available
- [ ] Produces effort range (S/M/L/XL), not a single number
- [ ] Does not write any files
- [ ] No director gates are invoked
- [ ] Always produces an estimate (never blocked by missing data; uses defaults instead)

---

## Coverage Notes

- The skill does not produce PASS/FAIL verdicts; the "verdict" here is the
  effort range itself. Test assertions focus on the accuracy of the range
  and the quality of the reasoning, not a binary outcome.
- Team-specific velocity calibration (what "M" means for this team) is an
  implementation detail not tested here; it is configured via change set history.
