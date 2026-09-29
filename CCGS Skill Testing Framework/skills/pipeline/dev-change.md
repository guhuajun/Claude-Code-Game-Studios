# Skill Test Spec: /dev-change

## Skill Summary

`/dev-change` reads a change directory, loads all required context (referenced ADR,
TR-ID from the registry, control manifest, engine preferences), implements the
change, verifies that all acceptance criteria are met, and marks the change
Complete. The skill routes implementation to the correct specialist agent based
on the engine and file type — it does not write source code directly.

In `full` review mode, an LP-CODE-REVIEW gate runs before marking the change
Complete. In `lean` or `solo` mode, LP-CODE-REVIEW is skipped and the change is
marked Complete after the user confirms all criteria are met. The skill asks
"May I write" before updating change status and before writing code files.

---

## Static Assertions (Structural)

Verified automatically by `/skill-test static` — no fixture needed.

- [ ] Has required frontmatter fields: `name`, `description`, `argument-hint`, `user-invocable`, `allowed-tools`
- [ ] Has ≥2 phase headings
- [ ] Contains verdict keywords: COMPLETE, BLOCKED, IN PROGRESS, NEEDS CHANGES
- [ ] Contains "May I write" collaborative protocol language (change status + code files)
- [ ] Has a next-step handoff at the end (`/change-done`)
- [ ] Documents LP-CODE-REVIEW gate: active in full mode, skipped in lean/solo
- [ ] Notes that implementation is delegated to specialist agents (not done directly)

---

## Director Gate Checks

In `full` mode: LP-CODE-REVIEW gate runs after implementation is complete and all
criteria are verified, before marking the change Complete.

In `lean` mode: LP-CODE-REVIEW is skipped. Output notes:
"LP-CODE-REVIEW skipped — lean mode". Change is marked Complete after user confirms.

In `solo` mode: LP-CODE-REVIEW is skipped with equivalent notes.

---

## Test Cases

### Case 1: Happy Path — Change implemented and marked Complete (full mode)

**Fixture:**
- A change directory exists at `openspec/changes/[layer]/change-[name].md` with:
  - `Status: Ready`
  - A TR-ID referencing a registered requirement
  - At least 2 Given-When-Then acceptance criteria
  - A test evidence path
- Referenced ADR has `Status: Accepted`
- `docs/architecture/control-manifest.md` exists
- `.claude/docs/technical-preferences.md` has engine and language configured
- `production/session-state/review-mode.txt` contains `full`

**Input:** `/dev-change openspec/changes/[layer]/change-[name].md`

**Expected behavior:**
1. Skill reads the change directory and all referenced context
2. Skill verifies the ADR is Accepted (no block)
3. Skill routes implementation to the correct specialist agent
4. All acceptance criteria are verified as met
5. LP-CODE-REVIEW gate spawns and returns APPROVED
6. Skill asks "May I update change status to Complete?"
7. Change status is updated to Complete

**Assertions:**
- [ ] Skill reads change before spawning any agent
- [ ] ADR status is checked before implementation begins
- [ ] Implementation is delegated to a specialist agent (not done inline)
- [ ] All acceptance criteria are confirmed before LP-CODE-REVIEW
- [ ] LP-CODE-REVIEW appears in output as a completed gate
- [ ] Change status is updated to Complete only after gate approval and user consent
- [ ] Test file is written as part of implementation (not deferred)

---

### Case 2: Failure Path — Referenced ADR is Proposed

**Fixture:**
- A change directory exists with `Status: Ready`
- The change's TR-ID points to a requirement covered by an ADR with `Status: Proposed`

**Input:** `/dev-change openspec/changes/[layer]/change-[name].md`

**Expected behavior:**
1. Skill reads the change directory
2. Skill resolves the TR-ID and reads the governing ADR
3. ADR status is Proposed — skill outputs a BLOCKED message
4. Skill names the specific ADR blocking the change
5. Skill recommends running `/architecture-decision` to advance the ADR
6. Implementation does NOT begin

**Assertions:**
- [ ] Skill does NOT begin implementation with a Proposed ADR
- [ ] BLOCKED message names the specific ADR number and title
- [ ] Skill recommends `/architecture-decision` as the next action
- [ ] Change status remains unchanged (not set to In Progress or Complete)

---

### Case 3: Ambiguous Acceptance Criteria — Skill asks for clarification

**Fixture:**
- A change directory exists with `Status: Ready`
- Referenced ADR is Accepted
- One acceptance criterion is ambiguous (not Given-When-Then; uses subjective language like "feels responsive")

**Input:** `/dev-change openspec/changes/[layer]/change-[name].md`

**Expected behavior:**
1. Skill reads the change and identifies the ambiguous criterion
2. Before routing to the specialist, skill asks the user to clarify the criterion
3. User provides a concrete, testable restatement
4. Skill proceeds with implementation using the clarified criterion
5. Skill does NOT guess at the intended behavior

**Assertions:**
- [ ] Skill surfaces the ambiguous criterion before implementation starts
- [ ] Skill asks for user clarification (not auto-interpretation)
- [ ] Implementation begins only after clarification is provided
- [ ] Clarified criterion is used in the test (not the original vague version)

---

### Case 4: Edge Case — No argument; reads from session state

**Fixture:**
- No argument is provided
- `production/session-state/active.md` references an active change directory
- That change directory exists with `Status: In Progress`

**Input:** `/dev-change` (no argument)

**Expected behavior:**
1. Skill detects no argument is provided
2. Skill reads `production/session-state/active.md`
3. Skill finds the active change reference
4. Skill confirms with user: "Continuing work on [change title] — is that correct?"
5. After confirmation, skill proceeds with that change

**Assertions:**
- [ ] Skill reads session state when no argument is provided
- [ ] Skill confirms the active change with the user before proceeding
- [ ] Skill does NOT silently assume the active change without confirmation
- [ ] If session state has no active change, skill asks which change to implement

---

### Case 5: Director Gate — LP-CODE-REVIEW returns NEEDS CHANGES; lean mode skips gate

**Fixture (full mode):**
- Change is implemented and all criteria appear met
- `production/session-state/review-mode.txt` contains `full`
- LP-CODE-REVIEW gate returns NEEDS CHANGES with specific feedback

**Full mode expected behavior:**
1. LP-CODE-REVIEW gate spawns after implementation
2. Gate returns NEEDS CHANGES with 2 specific issues
3. Change status remains In Progress — NOT marked Complete
4. User is shown the gate feedback and asked how to proceed

**Assertions (full mode):**
- [ ] Change is NOT marked Complete when LP-CODE-REVIEW returns NEEDS CHANGES
- [ ] Gate feedback is shown to the user verbatim
- [ ] Change status stays In Progress until issues are resolved and gate passes

**Fixture (lean mode):**
- Same change, `production/session-state/review-mode.txt` contains `lean`

**Lean mode expected behavior:**
1. Implementation completes
2. LP-CODE-REVIEW gate is skipped — noted in output
3. User is asked to confirm all criteria are met
4. Change is marked Complete after user confirmation

**Assertions (lean mode):**
- [ ] "LP-CODE-REVIEW skipped — lean mode" appears in output
- [ ] Change is marked Complete after user confirms criteria (no gate required)
- [ ] Skill does NOT block on a gate that is skipped

---

## Protocol Compliance

- [ ] Does NOT write source code directly — delegates to specialist agents
- [ ] Reads all context (change, TR-ID, ADR, manifest, engine prefs) before implementation
- [ ] "May I write" asked before updating change status and before writing code files
- [ ] Skipped gates noted by name and mode in output
- [ ] Updates `production/session-state/active.md` after change completion
- [ ] Ends with next-step handoff: `/change-done`

---

## Coverage Notes

- Engine routing logic (Godot vs Unity vs Unreal) is not tested per engine —
  the routing pattern is consistent; engine selection is a config fact.
- Visual/Feel and UI change types (no automated test required) have different
  evidence requirements and are not covered in these cases.
- Integration change type follows the same pattern as Logic but with a different
  evidence path — not independently fixture-tested.
