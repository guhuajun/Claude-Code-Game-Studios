# Skill Test Spec: /create-changes

## Skill Summary

`/create-changes` breaks a single capability into developer-ready change directorys. It reads
the spec.md, the corresponding GDD, governing ADRs, the control manifest, and the
TR registry. Each change gets structured frontmatter including: Title, Capability, Layer,
Priority, Status, TR-ID, ADR references, Acceptance Criteria, and Definition of
Done. Changes are classified by type (Logic / Integration / Visual/Feel / UI /
Config/Data) which determines the required test evidence path.

In `full` review mode, a QL-CHANGE-READY check runs per change after creation. In
`lean` or `solo` mode, QL-CHANGE-READY is skipped. The skill asks "May I write"
before writing each change directory. Changes are written to
`openspec/changes/[layer]/change-[name].md`.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains verdict keywords: COMPLETE, BLOCKED, NEEDS WORK
- [ ] Contains "May I write" collaborative protocol language (per-change approval)
- [ ] Has a next-step handoff at the end (`/change-readiness`, `/dev-change`)
- [ ] Documents change Status: Blocked when governing ADR is Proposed
- [ ] Documents QL-CHANGE-READY gate: active in full mode, skipped in lean/solo

---

## Director Gate Checks

In `full` mode: QL-CHANGE-READY check runs per change after creation. Changes that
fail the check are noted as NEEDS WORK before the "May I write" ask.

In `lean` mode: QL-CHANGE-READY is skipped. Output notes:
"QL-CHANGE-READY skipped — lean mode" per change.

In `solo` mode: QL-CHANGE-READY is skipped with equivalent notes.

---

## Test Cases

### Case 1: Happy Path — Capability with 3 changes, all ADRs Accepted

**Fixture:**
- `openspec/changes/[layer]/EPIC-[name].md` exists with 3 GDD requirements
- Corresponding GDD exists with matching acceptance criteria
- All governing ADRs have `Status: Accepted`
- `docs/architecture/control-manifest.md` exists
- `docs/architecture/tr-registry.yaml` has TR-IDs for all 3 requirements
- `production/session-state/review-mode.txt` contains `lean`

**Input:** `/create-changes [capability-name]`

**Expected behavior:**
1. Skill reads spec.md, GDD, governing ADRs, control manifest, and TR registry
2. Classifies each requirement into a change type (Logic / Integration / Visual/Feel / UI / Config/Data)
3. Drafts 3 change directorys with correct frontmatter schema
4. QL-CHANGE-READY is skipped (lean mode) — noted in output
5. Asks "May I write" before writing each change directory
6. Writes all 3 change directorys after approval

**Assertions:**
- [ ] Each change's frontmatter contains: Title, Capability, Layer, Priority, Status, TR-ID, ADR reference, Acceptance Criteria, DoD
- [ ] Change types are correctly classified (at least one Logic type in fixture)
- [ ] "May I write" is asked per change (not once for the entire batch)
- [ ] QL-CHANGE-READY skip is noted in output
- [ ] All 3 change directorys are written with correct naming: `change-[name].md`
- [ ] Skill does NOT start implementation

---

### Case 2: Failure Path — No capability file found

**Fixture:**
- The capability path provided does not exist in `openspec/changes/`

**Input:** `/create-changes nonexistent-capability`

**Expected behavior:**
1. Skill attempts to read the spec.md file
2. File not found
3. Skill outputs a clear error with the path it searched
4. Skill suggests checking `openspec/changes/` or running `/create-capabilities` first
5. No change directorys are created

**Assertions:**
- [ ] Skill outputs a clear error naming the missing file path
- [ ] No change directorys are written
- [ ] Skill recommends the correct next action (`/create-capabilities`)
- [ ] Skill does NOT create changes without a valid spec.md

---

### Case 3: Blocked Change — ADR is Proposed

**Fixture:**
- spec.md exists with 2 requirements
- Requirement 1 is covered by an Accepted ADR
- Requirement 2 is covered by an ADR with `Status: Proposed`

**Input:** `/create-changes [capability-name]`

**Expected behavior:**
1. Skill reads the ADR for Requirement 2 and finds Status: Proposed
2. Change for Requirement 2 is drafted with `Status: Blocked`
3. Blocking note references the specific ADR: "BLOCKED: ADR-NNN is Proposed"
4. Change for Requirement 1 is drafted normally with `Status: Ready`
5. Both changes are shown in the draft — user asked "May I write" for both

**Assertions:**
- [ ] Change 2 has `Status: Blocked` in its frontmatter
- [ ] Blocking note names the specific ADR number and recommends `/architecture-decision`
- [ ] Change 1 has `Status: Ready` — blocked status does not affect non-blocked changes
- [ ] Blocked status is shown in the draft preview before writing
- [ ] Both change directorys are written (blocked changes are still written — just flagged)

---

### Case 4: Edge Case — No argument provided

**Fixture:**
- `openspec/changes/` directory exists with ≥2 capability subdirectories

**Input:** `/create-changes` (no argument)

**Expected behavior:**
1. Skill detects no argument is provided
2. Outputs a usage error: "No capability specified. Usage: /create-changes [capability-name]"
3. Skill lists available capabilities from `openspec/changes/`
4. No change directorys are created

**Assertions:**
- [ ] Skill outputs a usage error when no argument is given
- [ ] Skill lists available capabilities to help the user choose
- [ ] No change directorys are written
- [ ] Skill does NOT silently pick an capability without user input

---

### Case 5: Director Gate — Full mode runs QL-CHANGE-READY; changes failing noted as NEEDS WORK

**Fixture:**
- spec.md exists with 2 requirements
- Both governing ADRs are Accepted
- `production/session-state/review-mode.txt` contains `full`
- QL-CHANGE-READY check finds one change has ambiguous acceptance criteria

**Input:** `/create-changes [capability-name]`

**Expected behavior:**
1. Both changes are drafted
2. QL-CHANGE-READY check runs for each change
3. Change 1 passes QL-CHANGE-READY
4. Change 2 fails QL-CHANGE-READY — noted as NEEDS WORK with specific feedback
5. Both changes are shown to user with pass/fail status before "May I write"
6. User can proceed (change written as-is with NEEDS WORK note) or revise first

**Assertions:**
- [ ] QL-CHANGE-READY results appear per change in the output
- [ ] Change 2 is flagged as NEEDS WORK with the specific failing criteria
- [ ] Change 1 shows as passing QL-CHANGE-READY
- [ ] User is given the choice to proceed or revise before writing
- [ ] Skill does NOT auto-block writing of changes that fail QL-CHANGE-READY without user input

---

## Protocol Compliance

- [ ] All context (EPIC, GDD, ADRs, manifest, TR registry) loaded before drafting changes
- [ ] Change drafts shown in full before any "May I write" ask
- [ ] "May I write" asked per change (not once for the entire batch)
- [ ] Blocked changes flagged before write approval — not discovered after writing
- [ ] TR-IDs reference the registry — requirement text is not embedded inline in change directorys
- [ ] Control manifest rules quoted per-change from the manifest, not invented
- [ ] Ends with next-step handoff: `/change-readiness` → `/dev-change`

---

## Coverage Notes

- Integration change test evidence (playtest doc alternative) follows the same
  approval pattern as Logic changes — not independently fixture-tested.
- Change ordering (foundational first, UI last) is validated implicitly via
  Case 1's multi-change fixture.
- The change sizing rule (splitting large requirement groups) is not tested here
  — it is addressed in the `/create-changes` skill's internal logic.
