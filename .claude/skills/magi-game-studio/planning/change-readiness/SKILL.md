---
name: change-readiness
description: "Is a change implementation-ready? Checks clear acceptance criteria, open questions, ADR refs. READY/NEEDS WORK/BLOCKED/NOT ASSESSED."
argument-hint: "[change-file-path or 'all' or 'change set']"
user-invocable: true
---

!`bash "${CLAUDE_SKILL_DIR}/../../../../hooks/yaml-helper.sh" resolve_config --keys review_mode,automation,workflow,qa.level,testing.strict,system_overrides`

Resolved above — use as-is; `--review` overrides `review_mode`. No block →
defaults in `.claude/docs/config-resolution.md`.


# Change Readiness

This skill validates that a change file contains everything a developer needs
to begin implementation — no mid-flight design interruptions, no guessing,
no ambiguous acceptance criteria. Run it before assigning a change.

**This skill is read-only.** It never edits change files. It reports findings
and asks whether the user wants help filling gaps.

**Output:** Verdict per change (READY / NEEDS WORK / BLOCKED / NOT ASSESSED) with a specific

> **`NOT ASSESSED` is not a synonym for `BLOCKED`.** `BLOCKED` is a
> finding about the change — a Proposed ADR, an unresolved dependency — and it
> tells the reader exactly what to clear. Use `NOT ASSESSED` when the change could
> not be evaluated at all: the file is unreadable or unparseable, or a referenced
> ADR or design document cannot be located, so the checks below cannot run.
> Collapsing that into `BLOCKED` reports a blocker that does not exist and hides
> the one that does — the reader chases a phantom ADR instead of a missing file.
> `READY` must never be reachable for a change that was not actually evaluated.
>
> **Precedence — first matching rule wins**, in this order: **BLOCKED**, then
> **NEEDS WORK**, then **NOT ASSESSED**, then **READY**. `NOT ASSESSED` outranks
> `READY` (a change that could not be evaluated has not been shown ready) and
> ranks **below** both failure verdicts (a known blocker is more actionable than
> an unknown, and demoting it behind an access problem buries it). A change with
> both a real blocker and an unevaluable check is `BLOCKED` — the blocker is the
> actionable finding. This half of the rank has to be stated: the rule above
> establishes only that `READY` is unreachable, which would leave the ordering
> against `BLOCKED` to inference.
gap list for each non-ready change.

---

## Phase 0: Resolve Review Mode


See `.claude/docs/director-gates.md` for the full check pattern and mode definitions. Individual gate definitions live in `.claude/docs/director-gates/[gate-id].md` — the spawned agent reads its own gate file; do not read it in the parent session.


Every `AskUserQuestion` call follows `.claude/docs/automation-modes.md`
(collaborative asks always · guided major-only · autonomous logs and proceeds;
`automation_always_ask` categories always prompt).

**Resolve the workflow tier per change** (per `.claude/docs/workflow-modes.md`):
for the system a change belongs to (**the GDD filename stem** of its `GDD:` path;
the `[system]` segment of a `TR-[system]-NNN` ID is a fallback alias only), use the
`system_overrides` row for that system if the block lists one, else the
project value. When validating multiple changes (`all` / `change set` scope),
resolve **per change** — different systems may sit at different tiers. The tier
sets which checklist sections block — see the note in Section 3.

**`qa.level`**: controls whether a test requirement is
validated. At `minimal`, the "Test evidence requirement is clear" item
auto-passes (no requirement validated); at `standard`, validate the per-type test
requirement (strictness from `testing.strict`); at `full`, also validate a
coverage target. Distinct axis from `workflow`.

---

## 1. Parse Arguments

**Scope:** `$ARGUMENTS[0]` (blank = ask user via AskUserQuestion)

- **Specific path** (e.g., `/change-readiness openspec/changes/combat/change-001-basic-attack.md`):
  validate that single change file.
- **`change set`**: read the current change set from `openspec/changes/` (most
  recent file), extract every change path it references, validate each one.
- **`all`**: glob `openspec/changes/**/*.md`, exclude `spec.md` index files,
  validate every change file found.
- **No argument**: ask the user which scope to validate.

If no argument is given, use `AskUserQuestion`:
- "What would you like to validate?"
  - Options: "A specific change file", "All changes in the current change set",
    "All changes in openspec/changes/", "Changes for a specific capability"

Report the scope before proceeding: "Validating [N] change files."

> **If the scope resolves to ZERO change files, stop and report
> `NOT ASSESSED — no changes in scope`.** Name which scope was searched and
> which path was empty (`openspec/changes/**/*.md`, the change list's change
> list, or the specific path given), and route: `/design-system [layer]` then
> `/create-changes [capability-slug]`.
>
> **The zero-scope path is mandatory.** Without it an empty glob falls through
> to the Section 5 aggregate template and renders `Ready: 0 /
> Needs Work: 0 / Blocked: 0` above an empty list — **indistinguishable from
> "I checked every change and none needed work"**. It is the core failure of this
> framework exactly: a scan that finds nothing because there was nothing to
> scan, reported the same way as a clean result. Three zeros read as a healthy
> change set.
>
> Note what made this survive: `NOT ASSESSED` was already in this skill's
> vocabulary, but the body scoped it to **per-change** evaluation failures (an
> unreadable or unparseable file, a missing referenced ADR). The verdict existed;
> the case that most needs it had no route to it. It is a recurring shape — a
> correct fix that did not reach one surface.
>
> **A zero-change change set scope is not the same as an absent change list.** If
> `openspec/changes/` has no file at all, say that instead — "no change set
> found" and "the change set lists no changes" send the reader to different
> fixes, and Section 7 already draws that distinction for the handoff block.

---

## 2. Load Supporting Context

Before checking any changes, load reference documents once (not per-change):

- `design/gdd/systems-index.md` — to know which systems have approved GDDs
- `docs/architecture/control-manifest.md` — change-readiness needs only the header
  `Manifest Version:` date (its manifest check is existence + version, not the rule
  bodies), so grep it (`Grep pattern="Manifest Version" path="docs/architecture/control-manifest.md"`)
  rather than a full read. If the file does not exist, note it as missing once; do not
  re-flag per change.
- `docs/architecture/tr-registry.yaml` — index all entries by `id`. Used to
  validate TR-IDs in changes. If the file does not exist, note it once; TR-ID
  checks will auto-pass for all changes (registry predates changes, so missing
  registry means changes are from before TR tracking was introduced).
- All ADR status fields — resolve these with **one scan, not one read per ADR**.
  A `Status:` value is a single line; reading whole ADR files to find it costs the
  entire architecture corpus, and in `all` scope that multiplies across every
  change in the repo:
  ```
  Grep pattern="^## Status" glob="docs/architecture/adr-*.md" output_mode="content" -A 3
  ```
  Establish the denominator first (glob `docs/architecture/adr-*.md`, count **N**)
  and interpret against it: **0 matches with N > 0 means malformed ADRs, not
  "no Accepted ADRs"** — report "run `/architecture-decision [file] retrofit`"
  rather than failing every change's ADR check. Never treat an unreadable status as
  a failed one. Cache the resulting map; do not re-scan per change.
- The current change list (if scope is `change set`) — to identify Must Have /
  Should Have priority for escalation decisions

---

## 3. Change Readiness Checklist

For each change file, evaluate every item below. A change is READY only if all
items pass or are explicitly marked N/A with a stated reason.

> **Workflow tier adjustment** (resolved in Phase 0, per the change's system). The
> full checklist below is the `full` baseline:
> - **`full`** — every item is blocking (TR registry + ADR + control manifest
>   fully validated).
> - **`standard`** — **Design Completeness** and **Scope Clarity** stay blocking.
>   In **Architecture Completeness**, only a **critical (Foundation-layer) ADR**
>   that is missing or `Proposed` BLOCKS; a missing non-critical ADR, or an absent
>   "ADR referenced" note, is **advisory** (NEEDS WORK note, not BLOCKED). The
>   TR-ID and manifest items are advisory.
> - **`minimal`** — **acceptance-criteria check only**: evaluate Design
>   Completeness and Scope Clarity. Treat the entire **Architecture Completeness**
>   section as N/A — do not flag missing ADR / TR-ID / manifest references.

### Design Completeness

- [ ] **GDD requirement referenced**: The change includes a `design/gdd/` path
  and quotes or links a specific requirement, acceptance criterion, or rule from
  that GDD — not just the GDD filename. A link to the document without tracing
  to a specific requirement does not pass.
- [ ] **Requirement is self-contained**: The acceptance criteria in the change
  are understandable without opening the GDD. A developer should not need to
  read a separate document to understand what DONE means.
- [ ] **Acceptance criteria are testable**: Each criterion is a specific,
  observable condition — not "implement X" or "the system works correctly".
  Bad example: "Implement the jump mechanic." Good example: "Jump reaches
  max height of 5 units within 0.3 seconds when jump is held."
- [ ] **No acceptance criteria require judgment calls** *(auto-pass for `Type: Visual/Feel`)*: Criteria like
  "feels responsive" or "looks good" are not testable without a defined
  benchmark. For Logic, Integration, UI, and Config/Data changes, these must be
  replaced with specific observable conditions. For Visual/Feel changes, subjective
  criteria are expected and this check auto-passes — instead verify that each
  subjective criterion has a paired playtest protocol or evidence requirement
  (e.g., "evidence doc required at `production/qa/evidence/[slug]-evidence.md`").
  PASS if the acceptance criterion ends with or is accompanied by an explicit reference to a file path such as `production/qa/evidence/[slug]-evidence.md`. NEEDS WORK if the criterion is purely subjective with no evidence file path specified.

### Architecture Completeness

> The `BLOCKED` / fail outcomes in this section are the `full` baseline. Apply the
> tier note above: at `standard` only a missing/Proposed **critical** ADR BLOCKS
> (other items advisory); at `minimal` treat this entire section as N/A.

- [ ] **ADR referenced or N/A stated**: The change references at least one ADR,
  OR explicitly states "No ADR applies" with a brief reason.
  A change with no ADR reference and no explicit N/A note fails this check.
- [ ] **ADR is Accepted (not Proposed)**: For each referenced ADR, check its
  `Status:` field using the cached ADR statuses loaded in Section 2.
  - If `Status: Accepted` → pass.
  - If `Status: Proposed` → **BLOCKED**: the ADR may change before it is accepted,
    and the change's implementation guidance could be wrong.
    Fix: `BLOCKED: ADR-NNNN is Proposed — wait for acceptance before implementing.`
  - If the ADR file does not exist → **BLOCKED**: referenced ADR is missing.
  - Auto-pass if change has an explicit "No ADR applies" N/A note.
- [ ] **TR-ID is valid and active**: If the change contains a `TR-[system]-NNN`
  reference, look it up in the TR registry loaded in Section 2.
  - If the ID exists and `status: active` → pass.
  - If the ID exists and `status: deprecated` or `status: superseded-by: ...` →
    NEEDS WORK: the requirement was removed or replaced.
    Fix: update the change to reference the current requirement ID or remove if no longer applicable.
  - If the ID does not exist in the registry → NEEDS WORK: ID was not registered
    (change may predate registry, or registry needs an `/architecture-review` run).
  - Auto-pass if the change has no TR-ID reference OR if the registry does not exist.
- [ ] **Manifest version is current**: If the change has a `Manifest Version:` date
  in its header AND `docs/architecture/control-manifest.md` exists:
  - If change version matches current manifest `Manifest Version:` → pass.
  - If change version is older than current manifest → NEEDS WORK: new rules may
    apply. Fix: review changed manifest rules, update change if any forbidden/required
    entries changed, then update the change's `Manifest Version:` to current.
  - Auto-pass if either the change has no `Manifest Version:` field OR the manifest
    does not exist.
- [ ] **Engine notes present**: For any post-cutoff engine API this change
  is likely to touch, implementation notes or a verification requirement are
  included. If the change clearly does not touch engine APIs (e.g., it is a
  pure data/config change), "N/A — no engine API involved" is acceptable.
- [ ] **Control manifest rules noted**: Relevant layer rules from the control
  manifest are referenced, OR "N/A — manifest not yet created" is stated.
  This item auto-passes if `docs/architecture/control-manifest.md` does not
  exist yet (do not penalize changes written before the manifest was created).

### Scope Clarity

- [ ] **Estimate present**: The change includes a size estimate (hours,
  points, or a t-shirt size). A change with no estimate cannot be planned.
- [ ] **In-scope / Out-of-scope boundary stated**: The change states what
  it does NOT include, either in an explicit Out of Scope section or in
  language that makes the boundary unambiguous. Without this, scope creep
  during implementation is likely.
- [ ] **Change dependencies listed**: If this change depends on other changes
  being DONE first, those change IDs are listed. If there are no dependencies,
  "None" is explicitly stated (not just omitted).

### Open Questions

- [ ] **No unresolved design questions**: The change does not contain text
  flagged as "UNRESOLVED", "TBD", "TODO", "?", or equivalent markers in
  any acceptance criterion, implementation note, or rule statement.
- [ ] **Dependency changes are not in DRAFT**: For each change listed as a
  dependency, check if the file exists and does not have a DRAFT status. A
  change that depends on a DRAFT or missing change is BLOCKED, not just
  NEEDS WORK.

### Asset References Check

- [ ] **Referenced assets exist**: Scan the change text for asset path patterns
  (paths containing `assets/`, or file extensions `.png`, `.jpg`, `.svg`,
  `.wav`, `.ogg`, `.mp3`, `.glb`, `.gltf`, `.tres`, `.tscn`, `.res`).
  - For each asset path found: use Glob to check whether the file exists.
  - If any referenced asset does not exist: **NEEDS WORK** — note the missing
    path(s). (The change references assets that have not been created yet.
    Either remove the reference, create a placeholder, or mark it as an
    explicit dependency on an asset creation change.)
  - If all referenced assets exist: note "Referenced assets verified:
    [count] found."
  - If no asset paths are referenced in the change: note "No asset references
    found in change — skipping asset check." This item auto-passes.
  - This is an existence-only check. Do not validate file format or content.

### Definition of Done

- [ ] **Minimum testable acceptance criteria by change type**:
  - Logic / Integration changes: at least 3
  - Visual/Feel and UI changes: at least 2
  - Config/Data changes: at least 1
  Apply the threshold matching the change's `Type:` field. If the change has fewer than the minimum, mark as NEEDS WORK.
- [ ] **Performance budget noted if applicable**: If this change touches any
  part of the gameplay loop, rendering, or physics, a performance budget or
  a "no performance impact expected — [reason]" note is present.
- [ ] **Change Type declared**: The change includes a `Type:` field in its header
  identifying the test category (Logic / Integration / Visual/Feel / UI / Config/Data).
  Without this, test evidence requirements cannot be enforced at change close.
  Fix: Add `Type: [Logic|Integration|Visual/Feel|UI|Config/Data]` to the change header.
- [ ] **Test evidence requirement is clear** *(auto-pass at `qa.level: minimal` — no
  test evidence is required, so this item never blocks)*: If the Change Type is set,
  the change includes a `## Test Evidence` section stating where evidence will be stored
  (test file path for Logic/Integration, or evidence doc path for Visual/Feel/UI).
  Resolve this item's gate level from the `testing.strict` block **resolved in
  Phase 0** — not by reading `project.yaml`, which would ignore a developer's
  locally-overridden value. Map the Change Type to a key (Logic→`logic`,
  Integration→`integration`, Visual/Feel→`visual`, UI→`ui`,
  Config/Data→`config`) and take
  `testing.strict.<key>`; use it only if its value is `true` or `false`
  (case-insensitive). If the key is absent, empty, or holds any other value, read
  `testing.strict` as a plain boolean (legacy single-value form); if that too is
  absent or invalid, default to strict for Logic, Integration, Visual/Feel and UI,
  and advisory for Config/Data. Surface any unrecognized value to the user.
  - At a **strict** gate level, a missing `## Test Evidence` section marks the
    change **NEEDS WORK**.
  - At an **advisory** gate level, a missing section is listed as a gap but does
    not by itself downgrade the verdict from READY.
  Fix: Add `## Test Evidence` with the expected evidence location for the change's type.

---

## 4. Verdict Assignment

Assign one of three verdicts per change:

**READY** — All checklist items pass, have explicit N/A justifications, or are
advisory-level gaps (a checklist item whose gate level resolved to advisory via
`testing.strict`). Advisory gaps are still listed under Gaps in the output.
The change can be assigned immediately.

**NEEDS WORK** — One or more checklist items fail, but all dependency changes
exist and are not DRAFT. The change can be fixed before assignment.

**BLOCKED** — One or more dependency changes are missing or in DRAFT state,
OR a critical design question (flagged UNRESOLVED in a criterion or rule) has
no owner. The change cannot be assigned until the blocker is resolved. Note:
a change that is BLOCKED may also have NEEDS WORK items — list both.

---

## 5. Output Format

### Single change output

```
## Change Readiness: [change title]
File: [path]
Verdict: [READY / NEEDS WORK / BLOCKED]

### Passing Checks (N/[total])
[list passing items briefly]

### Gaps
- [Checklist item]: [exact description of what is missing or wrong]
  Fix: [specific text needed to resolve this gap]

### Blockers (if BLOCKED)
- [What is blocking]: [change ID or design question that must resolve first]
```

### Multiple change aggregate output

```
## Change Readiness Summary — [scope] — [date]

Ready:      [N] changes
Needs Work: [N] changes
Blocked:    [N] changes

### Ready Changes
- [change title] ([path])

### Needs Work
- [change title]: [primary gap — one line]
- [change title]: [primary gap — one line]

### Blocked Changes
- [change title]: Blocked by [change ID / design question]

---
[Full detail for each non-ready change follows, using the single-change format]
```

### Change Set escalation

If the scope is `change set` and any Must Have changes are NEEDS WORK or BLOCKED,
add a prominent warning at the top of the output:

```
WARNING: [N] Must Have changes are not implementation-ready.
[List them with their primary gap or blocker.]
Resolve these before the change set begins or replan with `/create-changes`.
```

---

## 6. Collaborative Protocol

This skill is read-only. It never proposes edits or asks to write files.

After reporting findings, offer:

"Would you like help filling in the gaps for any of these changes? I can
draft the missing sections for your approval."

If the user says yes for a specific change, draft only the missing sections
in conversation. Do not use Write or Edit tools — the user (or
`/create-changes`) handles writing.

**Redirect rules:**
- If a change file does not exist at all: "This change file is missing entirely.
  Run `/design-system [layer]` then `/create-changes [capability-slug]` to generate changes from the GDD and ADR."
- If a change has no GDD reference and the work appears small: "This change has
  no GDD reference. If the change is small (under ~4 hours), run
  `/quick-design [description]` to create a Quick Design Spec, then reference
  that spec in the change."
- If a change's scope has grown beyond its original sizing: "This change appears
  to have expanded in scope. Consider splitting it or escalating to the producer
  before implementation begins."

---

## 7. Next-Change Handoff

After completing a single-change readiness check (not `all` or `change set` scope):

1. Read the current change list from `openspec/changes/` (most recent).
2. Find changes that are:
   - Status: READY or NOT STARTED
   - Not the change just checked
   - Not blocked by incomplete dependencies
   - In the Must Have or Should Have tier

If any are found, surface up to 3:

```
### Other Ready Changes in This Change Set

1. [Change name] — [1-line description] — Est: [X hrs]
2. [Change name] — [1-line description] — Est: [X hrs]

Run `/change-readiness [path]` to validate before starting.
```

If no change list exists, or no other ready changes are found, say which — `Next ready changes: no change list found` or `Next ready changes: none ready in [change set]` — rather than omitting the section. The two mean different things (nothing to read versus nothing ready) and an omitted section reads as neither.

---

## Phase 8: Director Gate — Change Readiness Review

Apply the review mode resolved in Phase 0 before spawning QL-CHANGE-READY:

- `solo` → skip. Note: "QL-CHANGE-READY skipped — Solo mode." Proceed to close.
- `lean` → skip. Note: "QL-CHANGE-READY skipped — Lean mode." Proceed to close.
- `full` → spawn as normal.

Spawn `qa-lead` via `Agent` using gate **QL-CHANGE-READY** (`.claude/docs/director-gates/ql-change-ready.md`).

Pass the following context:
- Change title
- Acceptance criteria list (all items from the change's acceptance criteria section)
- Dependency status (all dependencies listed and their current state: exist / DRAFT / missing)
- Overall verdict (READY / NEEDS WORK / BLOCKED) from Phase 4

Handle the verdict per standard rules in `director-gates.md`:
- **ADEQUATE** → change is cleared. Proceed to close.
- **GAPS [list]** → surface the specific gaps to the user via `AskUserQuestion`:
  options: `Update change with suggested gaps` / `Accept and proceed anyway` / `Discuss further`.
- **INADEQUATE** → surface the specific gaps; ask user whether to update the change or proceed anyway.

---

## Recommended Next Steps

- Run `/dev-change [change-path]` to begin implementation once the change is READY
- Run `/change-readiness change set` to check all changes in the current change set at once
- Run `/create-changes [capability-slug]` if a change file is missing entirely
