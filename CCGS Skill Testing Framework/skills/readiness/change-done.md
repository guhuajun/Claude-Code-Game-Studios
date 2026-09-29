# Skill Test Spec: /change-done

## Skill Summary

`/change-done` closes the loop between design and implementation. Run at the
end of implementing a change, it reads the change directory and verifies each
acceptance criterion against the implementation. It checks for GDD and ADR
deviations, prompts a code review, updates the change status to `Complete`,
logs any tech debt, and surfaces the next ready change from the change set. It
produces a COMPLETE / COMPLETE WITH NOTES / BLOCKED verdict and writes to
the change directory and optionally to `docs/tech-debt-register.md`.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥5 phase headings (complex skill warranting `context: fork` if applicable)
- [ ] Contains verdict keywords: COMPLETE, BLOCKED
- [ ] Contains "May I write" collaborative protocol language (writes to change directory and tech-debt register)
- [ ] Has a next-step handoff (surfaces next change from change set)

---

## Test Cases

### Case 1: Happy Path — All acceptance criteria met, no deviations

**Fixture:**
- Change file at `openspec/changes/core/change-light-pickup.md` with:
  - 3 acceptance criteria, all implemented as described
  - `TR-ID: TR-light-001` referencing a GDD requirement
  - `ADR: docs/architecture/adr-003-inventory.md` (Accepted)
  - `Status: In Progress`
- Implementation files listed in change exist in `src/`
- GDD requirement text at TR-light-001 matches how the feature was implemented
- ADR guidance was followed (no deviations)

**Input:** `/change-done openspec/changes/core/change-light-pickup.md`

**Expected behavior:**
1. Skill reads the change directory and extracts all key fields
2. Skill reads the GDD requirement fresh from `tr-registry.yaml` (not from change's quoted text)
3. Skill reads the referenced ADR to understand implementation constraints
4. Skill evaluates each acceptance criterion (auto where possible, manual prompt where not)
5. Skill checks for GDD requirement deviations
6. Skill checks for ADR guideline deviations
7. Skill prompts user: "Please provide the code review outcome for this change"
8. Skill presents COMPLETE verdict
9. Skill asks "May I update change Status to Complete and add Completion Notes?"
10. If yes: skill updates the change directory
11. Skill surfaces the next `Ready for Dev` change from the change set

**Assertions:**
- [ ] Skill reads `docs/architecture/tr-registry.yaml` for TR-ID requirement text (not just change)
- [ ] Skill reads the referenced ADR file (not just the change reference)
- [ ] Each acceptance criterion is listed with VERIFIED / DEFERRED / FAILED status
- [ ] Skill prompts the user for code review outcome (does not skip this step)
- [ ] Verdict is COMPLETE when all criteria are verified and no deviations exist
- [ ] Skill asks "May I write" before updating the change directory
- [ ] Skill does NOT auto-update change status without user confirmation
- [ ] After completion, skill surfaces the next ready change from `openspec/changes/`

---

### Case 2: Blocked Path — Acceptance criterion cannot be verified

**Fixture:**
- Change file has an acceptance criterion: "Player sees correct animation on pickup"
- No automated test for this criterion exists
- Manual verification has not been performed
- All other criteria are met

**Input:** `/change-done openspec/changes/core/change-light-pickup.md`

**Expected behavior:**
1. Skill processes all acceptance criteria
2. Reaches the animation criterion — cannot auto-verify
3. Skill asks the user: "Acceptance criterion 'Player sees correct animation on
   pickup' cannot be auto-verified. Has this been manually tested?"
4. If user says No: criterion is marked DEFERRED, verdict becomes COMPLETE WITH NOTES
5. Skill records the deferred criterion in completion notes
6. Asks "May I write updated change with deferred criterion noted?"

**Assertions:**
- [ ] Skill asks the user about unverifiable criteria rather than assuming PASS
- [ ] Deferred criteria result in COMPLETE WITH NOTES (not COMPLETE or BLOCKED)
- [ ] The deferred criterion is explicitly named in the completion notes
- [ ] Skill still asks "May I write" before updating the change directory

---

### Case 3: Blocked Path — GDD deviation detected

**Fixture:**
- Change TR-ID points to requirement: "Player can carry max 3 light sources"
- Implementation in `src/` uses a variable `MAX_CARRIED_LIGHTS = 5`
- This is a deliberate deviation from the GDD

**Input:** `/change-done openspec/changes/core/change-light-pickup.md`

**Expected behavior:**
1. Skill reads the GDD requirement text (max 3)
2. Skill detects discrepancy between requirement and implementation value (5)
3. Skill flags this as a GDD deviation and asks the user to classify it:
   - INTENTIONAL: document the deviation and reason
   - ERROR: implementation must be fixed before change can be marked Complete
   - OUT OF SCOPE: requirement changed and GDD needs updating
4. If INTENTIONAL: skill records deviation in completion notes, verdict is COMPLETE WITH NOTES
5. If ERROR: verdict is BLOCKED until implementation is corrected

**Assertions:**
- [ ] Skill detects the mismatch between GDD requirement and implementation value
- [ ] Skill asks the user to classify the deviation (not auto-assumes either way)
- [ ] INTENTIONAL deviation → COMPLETE WITH NOTES (not BLOCKED)
- [ ] ERROR deviation → BLOCKED verdict until fixed
- [ ] Detected deviations are recorded in completion notes or tech debt register

---

### Case 4: Edge Case — No argument, auto-detect current change

**Fixture:**
- `production/session-state/active.md` contains a reference to
  `openspec/changes/core/change-oxygen-drain.md` as the active change
- That change directory exists with `Status: In Progress`

**Input:** `/change-done` (no argument)

**Expected behavior:**
1. Skill reads `production/session-state/active.md`
2. Skill finds the active change reference
3. Skill reads that change directory and proceeds normally
4. Output confirms which change was auto-detected

**Assertions:**
- [ ] Skill reads `production/session-state/active.md` when no argument is given
- [ ] Skill identifies and confirms the auto-detected change before proceeding
- [ ] If no change is found in session state, skill asks the user to provide a path

---

---

### Case 5: Director Gate — LP-CODE-REVIEW behavior across review modes

**Fixture:**
- Change file at `openspec/changes/core/change-light-pickup.md`
- All acceptance criteria verified, no GDD deviations
- `production/session-state/review-mode.txt` exists

**Case 5a — full mode:**
- `review-mode.txt` contains `full`

**Input:** `/change-done openspec/changes/core/change-light-pickup.md` (full mode)

**Expected behavior:**
1. Skill reads review mode — determines `full`
2. After implementation verification, skill invokes LP-CODE-REVIEW gate
3. Lead programmer reviews the implementation
4. If LP verdict is NEEDS CHANGES → change cannot be marked Complete
5. If LP verdict is APPROVED → skill proceeds to mark change Complete

**Assertions (5a):**
- [ ] Skill reads review mode before deciding whether to invoke LP-CODE-REVIEW
- [ ] LP-CODE-REVIEW gate is invoked in full mode after implementation check
- [ ] An LP NEEDS CHANGES verdict prevents change from being marked Complete
- [ ] Gate result is noted in output: "Gate: LP-CODE-REVIEW — [result]"
- [ ] Skill still asks "May I write" before updating change status even if LP approved

**Case 5b — lean or solo mode:**
- `review-mode.txt` contains `lean` or `solo`

**Expected behavior:**
1. Skill reads review mode — determines `lean` or `solo`
2. LP-CODE-REVIEW gate is SKIPPED
3. Output notes the skip: "[LP-CODE-REVIEW] skipped — Lean/Solo mode"
4. Change completion proceeds based on acceptance criteria check only

**Assertions (5b):**
- [ ] LP-CODE-REVIEW gate does NOT spawn in lean or solo mode
- [ ] Skip is explicitly noted in output
- [ ] Skill still requires "May I write" approval before marking change Complete

---

## Protocol Compliance

- [ ] Uses "May I write" before updating the change directory
- [ ] Uses "May I write" before adding entries to `docs/tech-debt-register.md`
- [ ] Presents complete findings (criteria check, deviation check) before asking approval
- [ ] Ends by surfacing the next ready change from the change set plan
- [ ] Does not mark a change Complete if any criteria are in ERROR state
- [ ] Does not skip the code review prompt

---

## Coverage Notes

- The full 8-phase flow of the skill is exercised across Cases 1-3; not all
  edge cases within each phase are covered.
- Tech debt logging (deferred items written to `docs/tech-debt-register.md`)
  is mentioned in Case 2 but not the primary assertion focus; dedicated
  coverage deferred.
- The `openspec status` update (Phase 7 in the skill) is implied by Case 1
  but not the primary assertion; assumed to follow the same "May I write" pattern.
- Changes with multiple TR-IDs or multiple ADRs are not explicitly tested.
