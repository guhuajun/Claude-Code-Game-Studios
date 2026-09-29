---
name: create-changes
description: "Break one capability into OpenSpec change proposals embedding TR-ID, ADR guidance, acceptance criteria. Reads the control manifest. After /design-system."
argument-hint: "[capability-slug | capability-path] [--review full|lean|solo]"
user-invocable: true
---

!`bash "${CLAUDE_SKILL_DIR}/../../../../hooks/yaml-helper.sh" resolve_config --keys review_mode,automation,workflow,docs.density,change_granularity,system_overrides`

Resolved above — use as-is; `--review` overrides `review_mode`. No block →
defaults in `.claude/docs/config-resolution.md`.


# Create Changes

A change is a single implementable behaviour — small enough to complete in one
focused session, self-contained, and fully traceable to a capability requirement
and an ADR decision. Changes are what developers pick up. Capabilities are what
architects define.

**Run this skill per capability**, not per layer. Run it for Foundation
capabilities first, then Core, and so on — matching the dependency order.

**Output:** one OpenSpec change directory per change —
`openspec/changes/<change-id>/` containing `proposal.md`, `specs/<system>/spec.md`
(the delta), and `tasks.md`. `design.md` is added only when the change warrants
one (see Step 6).

**Previous step:** `/design-system [system]`
**Next step after changes exist:** `/change-readiness [change-id]` then
`/dev-change [change-id]`

---

## 1. Parse Argument


See `.claude/docs/director-gates.md` for the full check pattern. Individual gate definitions live in `.claude/docs/director-gates/[gate-id].md` — the spawned agent reads its own gate file; do not read it in the parent session.


Every `AskUserQuestion` call follows `.claude/docs/automation-modes.md`
(collaborative asks always · guided major-only · autonomous logs and proceeds;
`automation_always_ask` categories always prompt).

**`workflow`** for this capability's system (per `.claude/docs/workflow-modes.md`) —
use the `system_overrides` row for `<system>` if the block lists one, else the
project value. `<system>` is the capability slug. The tier sets which
prerequisites block — see the note in Step 2.

**`change_granularity`** — it sets each
change's acceptance-criteria load: **5–10 covering a whole feature** at `coarse`,
**2–4 covering one task** at `balanced` (default), **1** at `fine` (the change
name is the criterion restatement). Group or split criteria into changes to hit
the target.

**`docs.density`** — it controls the *depth* of each change's prose (context,
implementation notes, ADR summary), not the acceptance-criteria count (that is
`change_granularity`) and never the criteria text itself. `modes.rigor` sets it
alongside `workflow`; set `docs.density` explicitly to vary change prose alone:
`terse` = notes as bullets, no preamble; `balanced` = short context paragraph +
notes (default); `thorough` = full context, implementation guidance, and ADR
rationale. The embedded TR-ID reference, ADR Version stamp, and acceptance
criteria are structural and are never trimmed by density.

- `/create-changes [capability-slug]` — e.g. `/create-changes combat`
- `/create-changes openspec/specs/combat/spec.md` — full path also accepted
- No argument — at `minimal` there are no capabilities yet (Option A): skip to Step 2's
  `minimal` branch and synthesize the capability from `design/game-brief.md`. At
  `standard`/`full`, ask "Which capability would you like to break into changes?" and
  run `openspec list --specs` to list available capabilities with their status.

  > **If that command lists nothing at `standard`/`full`, stop — do not build a
  > question with no options.** Report:
  > "No capabilities found under `openspec/specs/`. Run `/design-system layer: foundation`
  > first — a capability is what this skill decomposes."
  >
  > **The zero-capability path is load-bearing.** Asking which capability *and*
  > listing them leaves an `AskUserQuestion` with nothing to offer when the list is
  > empty. Route to `/design-system` instead — it is named as **Previous step**
  > in this skill's own header.
  >
  > Note what this skill guarded and what it did not. Step 2's ADR validation is
  > thorough: three tiers, each with its own stop condition, and an explicit
  > message naming the missing file. That is the **deepest** input. The **first**
  > input — does a capability exist at all — went unchecked. Guarding the far end of a
  > chain while leaving the near end open is the shape to watch for.
  >
  > At `minimal` this does not apply: there are deliberately no capabilities, and the
  > branch above synthesizes one from the brief.

---

## 2. Load Everything for This Capability

> **`minimal` tier — synthesize the capability from the brief** (Option A). At
> `minimal` there is no `/design-system` step and no spec. Instead:
> 1. Read `design/game-brief.md` in full (it is one page).
> 2. Synthesize an implicit capability: write a lightweight
>    `openspec/specs/<slug>/spec.md` with a `## Purpose` (the brief's one-sentence
>    pitch), the brief's MVP feature list as requirements, and ordering = its
>    **Build order**. Keep it terse; this is the capability `/dev-change` and
>    `openspec list --specs` expect.
>    **Write this spec BEFORE archiving anything** — see the archive warning in
>    Step 6.
> 3. Generate **one coarse change per MVP feature** (Step 3+), in Build-order
>    sequence, each traced to the brief (not a GDD/TR-ID). Leave changes unblocked
>    on ADR grounds — none exist at this tier.
> Skip the GDD, control-manifest, TR-registry, and ADR reads below (none exist at
> `minimal`), then continue to Step 3 with the synthesized capability.

For `standard`/`full` (a `/design-system` capability exists), read in full (these are small):

- `openspec/specs/<system>/spec.md` — the capability GDD: Purpose, Player
  Fantasy, Detailed Design, Formulas, Edge Cases, Tuning Knobs, Dependencies, and
  the requirements + scenarios. At `full` read all sections; at `standard` the 5
  required sections (+ conditional Formulas). Always prioritise the Requirements
  and their scenarios, Formulas, and Edge Cases where present.
- `docs/architecture/control-manifest.md` — grep only this capability's layer (`Grep pattern="^## <layer> Layer Rules" path="docs/architecture/control-manifest.md" output_mode="content" -A 40`) plus the header Manifest Version date, not a full read of all layers
- `docs/architecture/tr-registry.yaml` — grep only this system's entries (`Grep pattern="system: <slug>" path="docs/architecture/tr-registry.yaml" output_mode="content" -B1 -A5`, or `id: TR-<slug>-`), not the whole cross-system registry

**Load each governing ADR by section — never with an unbounded full read.** A
substantial ADR exceeds the 25k-token `Read` cap, and a capped read's only
recovery is paging through the remainder — the most expensive possible way to
read a file (measured at 103k tokens on a 34k-token ADR vs ~54k for targeted
reads of the same file). Per ADR:

1. **Map the headings** (cheap — line numbers only):
   ```
   Grep pattern="^## |^### Implementation Guidelines" path="docs/architecture/[adr-file].md" output_mode="content" -n
   ```
2. **Bounded-read exactly the sections this skill consumes**, using the line
   numbers from the map to set `Read(offset, limit)` spans that end where the
   next section begins:
   - `## Summary` and `## Decision` (including its `### Implementation
     Guidelines` subsection) — these feed the change's `design.md` Decisions
     section and the tasks' implementation notes.
   - `## Engine Compatibility` — feeds the change's Engine, Risk, and Engine
     Notes fields. (Engine Notes is a *change* field derived from this
     section — it is not an ADR section name; do not search for one.)
3. **Capture the `## Last Verified` date**:
   ```
   Grep pattern="^## (Last Verified|Date)" path="docs/architecture/[adr-file].md" output_mode="content" -A 1
   ```
   Use `Last Verified`, falling back to `Date`, then to `unversioned` if both
   are absent. This becomes the change's `ADR Version` stamp — `/dev-change`
   uses it to decide whether it can trust this change's distilled summary
   instead of re-opening the ADR.

Skip Context, Alternatives Considered, Consequences, Risks, and any
Amendments Log unless a section you loaded explicitly cross-references one of
their entries — then take only the referenced entry with one more bounded
read. If the heading map comes back empty (a nonstandard ADR predating the
template), fall back to one full `Read` — and if that read truncates at the
cap, do **not** page through the remainder; grep for the change-relevant
content directly and flag the ADR for `/architecture-decision [file]
retrofit`.

**ADR existence validation** (tier-gated — resolved in Step 1): After reading the governing ADRs list from the capability, confirm each referenced ADR file exists on disk.

- **`full`** — if **any** referenced ADR file cannot be found, **stop immediately** before decomposing any change.
- **`standard`** — stop only if a **critical (Foundation-layer) ADR** is missing; for a missing non-critical ADR, **warn and continue** (the change embeds the ADR reference and its `design.md` records the gap).
- **`minimal`** — no ADR requirement; do **not** stop. Embed any ADR references that do exist; otherwise decompose against the brief + acceptance criteria and leave changes unblocked on ADR grounds.

When stopping (full / standard-critical):

> "Capability references [ADR-NNNN: title] but `docs/architecture/[adr-file].md` was not found.
> Check the filename in the capability's Governing ADRs list, or run `/architecture-decision`
> to create it. Cannot create changes until all referenced ADR files are present."

At `full`, do not proceed to Step 3 until all referenced ADR files are confirmed present.

Report: "Loaded capability [name], spec [path], [N] governing ADRs [ADR status], [manifest status]." State the **actual** situation for the resolved tier — e.g. "all confirmed present, control manifest v[date]" at full; "M present, K missing non-critical (embedded, gap recorded)" at standard; "no ADRs / manifest required" at minimal. Do not assert "all confirmed present" if any referenced ADR was missing, or name a manifest version when none exists.

---

## 3. Classify Changes by Type

**Change Type Classification** — assign each change a type based on its acceptance criteria:

| Change Type | Assign when criteria reference... |
|---|---|
| **Logic** | Formulas, numerical thresholds, state transitions, AI decisions, calculations |
| **Integration** | Two or more systems interacting, signals crossing boundaries, save/load round-trips |
| **Visual/Feel** | Animation behaviour, VFX, "feels responsive", timing, screen shake, audio sync |
| **UI** | Menus, HUD elements, buttons, screens, dialogue boxes, tooltips |
| **Config/Data** | Balance tuning values, data file changes only — no new code logic |

Mixed changes: assign the type that carries the highest implementation risk.
The type determines what test evidence is required before `/change-done` can close the change.

---

## 4. Decompose the Capability into Changes

For each requirement/scenario in the capability spec:

1. Group related criteria that require the same core implementation
2. Each group = one change
3. Order changes: foundational behaviour first, edge cases last, UI last

**Change sizing rule:** size each change to the resolved `modes.change_granularity`
target (above). The "~2-4 hours / one focused session" heuristic is the
`balanced` default — at `coarse` a change spans a whole feature (5–10 criteria,
multi-day), at `fine` a change is a single criterion. Split or group criteria to hit the
resolved target, not a fixed session length.

For each change, determine:
- **Change id**: kebab-case, unique under `openspec/changes/` (e.g. `add-combat-parry`). This is the directory name and what `/opsx:propose` would derive.
- **Capability requirement**: which requirement / scenario does this satisfy?
- **TR-ID**: look up in `tr-registry.yaml`. Use the stable ID. If no match, use `TR-[system]-???` and warn.
- **Governing ADR**: which ADR governs how to implement this?
  - `Status: Accepted` → embed normally
  - `Status: Proposed` → record the gap in the change's `design.md` Open Questions: "BLOCKED: ADR-NNNN is Proposed — run `/architecture-decision` to advance it"
  - **Multiple ADRs apply**: List all governing ADRs in the change's `design.md`.
    Designate the one most directly controlling the implementation pattern as
    primary (first in the list). Others are listed as secondary references.
  - **No ADR applies at all**: Write `ADR: N/A — [brief reason, e.g. "pure data configuration, no architectural pattern required"]`. Do NOT leave the field blank — a blank ADR field means "not checked", not "not applicable".
- **Change Type**: from Step 3 classification
- **Engine risk**: from the ADR's Knowledge Risk field

---

## 4b. QA Lead Change Readiness Gate

**Review mode check** — apply before spawning QL-CHANGE-READY:
- `solo` → skip. Note: "QL-CHANGE-READY skipped — Solo mode." Proceed to Step 5 (present changes for review).
- `lean` → skip (not a PHASE-GATE). Note: "QL-CHANGE-READY skipped — Lean mode." Proceed to Step 5 (present changes for review).
- `full` → spawn as normal.

After decomposing all changes (Step 4 complete) but before presenting them for write approval, spawn `qa-lead` **once** via `Agent` using gate **QL-CHANGE-READY** (`.claude/docs/director-gates/ql-change-ready.md`). A single call returns **both** the readiness verdict and the test-case specs — do not spawn `qa-lead` a second time to generate specs.

Pass: the full change list with acceptance criteria, change types, and TR-IDs; the capability spec's requirements for reference. Require in the return:
1. The QL-CHANGE-READY verdict per change (ADEQUATE / GAPS / INADEQUATE).
2. For every change it marks **ADEQUATE**, its test-case spec block (formats below) — one Given/When/Then per acceptance criterion for Logic and Integration changes, or manual verification steps for Visual/Feel and UI changes.

Present the assessment. For each change flagged GAPS or INADEQUATE, revise the acceptance criteria before proceeding — untestable criteria cannot be implemented correctly; those changes carry no specs until they reach ADEQUATE (re-request specs for just those in a follow-up call only if a revision was needed). Once all changes are ADEQUATE, proceed with the returned specs.

**Prefer an existing QA plan when one already covers a change** — this substitutes for the qa-lead's specs, it does not add a spawn. Glob `production/qa/qa-plan-*.md` for the most recent file; if it holds test specs for changes in this capability (match titles/slugs in its Automated Tests Required section) that differ from the qa-lead's, use `AskUserQuestion` (Use QA-plan specs / Use qa-lead specs / Skip and leave `*Test cases not yet defined — run /qa-plan to generate them.*`). Either way no additional `qa-lead` spawn occurs.

The spec block formats — Logic/Integration:

```
Test: [criterion text]
  Given: [precondition]
  When: [action]
  Then: [expected result / assertion]
  Edge cases: [boundary values or failure states to test]
```

For Visual/Feel and UI changes, produce manual verification steps instead:
```
Manual check: [criterion text]
  Setup: [how to reach the state]
  Verify: [what to look for]
  Pass condition: [unambiguous pass description]
```

These test case specs are embedded into each change's `tasks.md` as verification
steps. The developer implements against these cases. The programmer does not write
tests from scratch — QA has already defined what "done" looks like.

---

## 5. Present Changes for Review

Before writing any files, present the full change list:

```
## Changes for Capability: [name]

Change `add-combat-parry`: [title] — Logic — ADR-NNNN
  Covers: TR-[system]-001 ([1-line summary of requirement])
  Test required: tests/unit/[system]/[slug]_test.[ext]

Change `add-combat-crit`: [title] — Integration — ADR-MMMM
  Covers: TR-[system]-002, TR-[system]-003
  Test required: tests/integration/[system]/[slug]_test.[ext]

Change `add-parry-vfx`: [title] — Visual/Feel — ADR-NNNN
  Covers: TR-[system]-004
  Evidence required: production/qa/evidence/[slug]-evidence.md

[N changes total: N Logic, N Integration, N Visual/Feel, N UI, N Config/Data]
```

Use `AskUserQuestion`:
- Prompt: "May I write these [N] changes to `openspec/changes/`?"
- Options: `[A] Yes — write all [N] changes` / `[B] Not yet — I want to review or adjust first`

---

## 6. Write Change Directories

For each change, create `openspec/changes/<change-id>/` with these files.

**`proposal.md`** — why the change exists:

```markdown
# Proposal

## Why

[1-2 sentences: the problem or opportunity this change addresses.]

## What Changes

- [Specific new capability, modification, or removal]

## Capabilities

### New Capabilities

### Modified Capabilities
- `<system>`: [what requirement is changing]

## Impact

- Governing ADRs: [ADR-NNNN: title] | **ADR Version**: [Last Verified date, else Date, else `unversioned`]
- Engine: [name + version] | **Risk**: [LOW / MEDIUM / HIGH]
- Engine Notes: [from the ADR's Engine Compatibility section]
- Control manifest rules (this layer): Required [..] · Forbidden [..] · Guardrail [..]
- Code root: [resolved from engine.name — `src/` Godot, `lssets/` Unity, `Source/<Module>/` Unreal]
```

**`specs/<system>/spec.md`** — the delta against the capability spec. Use the
OpenSpec delta format (`## ADDED Requirements` / `## MODIFIED Requirements`).
**Do not put the game-design sections here.** Player Fantasy, Formulas, Tuning
Knobs and the rest live in the *main* spec (`openspec/specs/<system>/spec.md`)
and are already written by `/design-system`. A delta carries requirements only —
`/opsx:archive` merges it into the main spec, preserving that spec's GDD sections
untouched.

> **Never let a change be the first thing that creates a capability's spec.**
> `openspec archive` creates a missing spec through `buildSpecSkeleton()`, which
> keeps only `## Purpose` and `## Requirements` — every hand-written GDD section
> is silently discarded. Verified against openspec 1.13.1. Write the main spec
> first via `/design-system`, then archive changes against it.

**`tasks.md`** — the implementation checklist. Group related tasks under `## N.`
headings; each task is `- [ ] N.M <description> and verify <how>`. The checkbox
markers are the progress record `openspec status` reads — a line with no checkbox
is not tracked, and only `- [x]` counts as done.

**`design.md`** — **only when the change warrants one**: a cross-cutting change,
a new architectural pattern, a new external dependency, a significant data-model
change, or security/performance/migration complexity. Otherwise omit it. When
present, it records Context, Goals/Non-Goals, Decisions (with alternatives), and
Risks/Trade-offs.

> **At `minimal` tier the Context/traceability inputs do not exist** (no ADR,
> TR registry, or control manifest). Fill the fields from the brief instead —
> apply this mapping exactly, so every run is deterministic rather than improvised:
> - **Capability spec** → `openspec/specs/<slug>/spec.md` (the one you wrote in Step 2)
> - **Requirement** → `Brief MVP feature N` (the feature this change implements — NOT a `TR-[system]-NNN` ID)
> - **Governing ADRs / ADR Version** → `N/A (minimal — no ADRs)`
> - **Manifest Version** and **Control manifest rules** → `N/A (minimal — no control manifest)`
> - **Engine** and **Risk** → read `docs/engine-reference/<engine>/VERSION.md`
>   (engine from `engine.name`). **Engine** is `engine.name` + `engine.version`.
>   **Risk** is the risk level that file assigns to the pinned version — its
>   post-cutoff timeline row, or its stated overall risk. If the file is missing
>   or assigns no level, write `NOT ASSESSED (no VERSION.md risk rating)` — never
>   guess a level.
>
>   > **This field is load-bearing and had no rule, so it was improvised.**
>   > `/dev-change` Phase 3 spawns the engine specialist as a mandatory secondary
>   > "when engine risk is HIGH (from the ADR or VERSION.md)". At `minimal` there
>   > is no ADR, so `VERSION.md` is the *only* source — and nothing here told this
>   > skill to read it. A change written with an invented `Risk: MEDIUM` against a
>   > `VERSION.md` rating of HIGH silently disables the specialist review. In the
>   > run that found this, that review was what caught two wrong engine defaults.
>   > Treat `NOT ASSESSED` as HIGH for the spawn decision: an unknown risk is not
>   > a low one.
> - **Engine Notes** → `none (no ADR engine-compatibility analysis at minimal)`
> - The acceptance criteria → derive concrete, testable criteria from the brief's
>   **Player goal & fail state** field plus the MVP feature this change implements,
>   rather than inventing them from a bare MVP bullet
> - At any tier where the QL-CHANGE-READY / qa-lead gate is skipped (`minimal`, or
>   `lean`/`solo` review mode) no qa-lead specs are authored; write
>   "*N/A — no qa-lead specs at this tier; implement against the criteria above*"
>   in `tasks.md` rather than improvising test cases.
> - Any **Test Evidence / DoD** line is governed by `qa.level`, not this template —
>   at `qa.level: minimal` it is **waived** (advisory, never "must exist and pass").

### After writing

Run `openspec validate <change-id>` and report the result. A change with zero
requirement deltas is rejected unless it sets `skip_specs: true` in its
`.openspec.yaml` — set that only for a change that genuinely alters no
spec-level behaviour (pure refactor, tooling, docs), never to silence a
validation error about missing requirements.

---

## 7. After Writing

Use `AskUserQuestion` to close with context-aware next steps:

Check:
- Are there other capabilities under `openspec/specs/` without changes yet? List them.
- Is this the last capability? If so, include planning the next batch as an option.

Widget:
- Prompt: "[N] changes written under `openspec/changes/`. What next?"
- Options (include all that apply):
  - `[A] Start implementing — run /change-readiness [first-change-id]` (Recommended)
  - `[B] Create changes for [next-capability-slug] — run /create-changes [slug]` (only if other capabilities have no changes yet)
  - `[C] Propose a change — run /opsx:propose` (only if all capabilities have changes)
  - `[D] Stop here for this session`

Note in output: "Work through changes in order — each change's dependency field tells you what must be DONE before you can start it."

---

## Collaborative Protocol

**Applies in `collaborative` mode (the default).** For `guided` and
`autonomous` modes, see `.claude/docs/automation-modes.md` — the rules below
describe what collaborative mode requires, not universal behavior.

1. **Read before presenting** — load all inputs silently before showing the change list
2. **Ask once** — present all changes for the capability in one summary, not one at a time
3. **Warn on blocked changes** — flag any change with a Proposed ADR before writing
4. **Ask before writing** — get approval for the full change set before writing files
5. **No invention** — acceptance criteria come from the capability spec, implementation notes from ADRs, rules from the manifest
6. **Never start implementation** — this skill stops at the change directory level

After writing (or declining):

- **Verdict: COMPLETE** — [N] changes written under `openspec/changes/`. Run `/change-readiness` → `/dev-change` to begin implementation.
- **Verdict: BLOCKED** — user declined. No change directories written.
