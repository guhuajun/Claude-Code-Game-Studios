# Claude Code Game Studios -- Game Studio Agent Architecture

Indie game development managed through 49 coordinated Claude Code subagents.
Each agent owns a specific domain, enforcing separation of concerns and quality.

## Workflow — OpenSpec (read this first)

Work is driven by **OpenSpec**, not by sprints or stories. The OpenSpec root is
`openspec/`; it is authoritative for what exists and what is in flight.

| Concept | Where it lives |
|---|---|
| **Capability** — a game system's design | `openspec/specs/<system>/spec.md` — this IS the GDD |
| **Change** — a unit of implementable work | `openspec/changes/<change-id>/` (proposal, delta spec, tasks) |
| **Completed work** | `openspec/changes/archive/`, merged into `specs/` |
| **Lifecycle phase** | `project.stage` in `project.yaml` (Concept … Release) |

- **An epic maps to a capability; a story maps to a change. A sprint has no
  equivalent** — in-flight changes are the unit of work. Do not look for
  `production/sprints/`, `production/epics/`, or `sprint-status.yaml`; they do
  not exist in any project this template produces.
- **Progress lives in the change's `tasks.md` checkbox markers**, read by
  `openspec status`. Never write a status field into a capability spec.
- The `ccgs-game` schema (`openspec/schemas/ccgs-game/`) adds the game-design
  sections — Player Fantasy, Detailed Design, Formulas, Edge Cases, Tuning
  Knobs, Dependencies — to the standard OpenSpec spec structure.
- **Never let a change be the first thing that creates a capability's spec.**
  `openspec archive` creates a missing spec skeleton containing only Purpose and
  Requirements, silently discarding those GDD sections. Write the spec first via
  `/design-system`, then archive changes against it. Verified against
  openspec 1.13.1.
- Useful commands: `openspec list` (in-flight changes), `openspec list --specs`
  (capability inventory), `openspec status`, `openspec validate <change-id>`.

## Technology Stack

- **Engine**: [CHOOSE: Godot 4 / Unity / Unreal Engine 5]
- **Language**: [CHOOSE: GDScript / C# / C++ / Blueprint]
- **Version Control**: Git with trunk-based development
- **Build System**: [SPECIFY after choosing engine]
- **Asset Pipeline**: [SPECIFY after choosing engine]

> **Note**: Engine-specialist agents exist for Godot, Unity, and Unreal with
> dedicated sub-specialists. Use the set matching your engine.

## Project Structure

@.claude/docs/directory-structure.md

## Engine Version Reference

<!-- ENGINE-REFERENCE-IMPORT: the line below is engine-specific. /setup-engine
     rewrites it to @docs/engine-reference/<engine>/VERSION.md for the chosen
     engine, so a Unity or Unreal project stops loading the Godot reference every
     session. It defaults to Godot (the template's example engine); skills that
     need the pinned version read docs/engine-reference/<engine>/VERSION.md on
     demand regardless of this import. -->
@docs/engine-reference/godot/VERSION.md


## Technical Preferences

`project.yaml` at the repo root is the primary config store — engine, specialists,
naming, platform, performance, modes. Skills resolve it via `resolve_config`
(see `.claude/docs/config-resolution.md`).

`.claude/docs/technical-preferences.md` is the **legacy fallback**, read on demand
when a key is absent from `project.yaml`. It is no longer imported here: before
`/setup-engine` runs it is almost entirely `[TO BE CONFIGURED]` placeholders, and
after it runs `project.yaml` holds the real values.

## Coordination Rules

@.claude/docs/coordination-rules.md

## Collaboration Protocol

**User-driven collaboration, not autonomous execution.**
Every task follows: **Question -> Options -> Decision -> Draft -> Approval**

- Agents MUST ask "May I write this to [filepath]?" before using Write/Edit tools
- Agents MUST show drafts or summaries before requesting approval
- Multi-file changes require explicit approval for the full changeset
- No commits without user instruction

See `docs/COLLABORATIVE-DESIGN-PRINCIPLE.md` for full protocol and examples.

> **First session?** If the project has no engine configured and no game concept,
> run `/start` to begin the guided onboarding flow.

## Coding Standards

@.claude/docs/coding-standards.md

## Context Management

Read `.claude/docs/context-management.md` on demand — it is a reference, not
session context. Two of its conventions are load-bearing and cited by name
elsewhere in the repo, so they are restated here rather than lost:

- **`production/session-state/active.md` is the session checkpoint.** The file is
  the memory, not the conversation. Read it first after any compaction, crash, or
  `/clear`.
- **Helpers in `.claude/scripts/` emit observations, never verdicts.** A script
  that scores or judges will eventually contradict a mode or override it cannot
  see. (Cited by `artifact-check.sh` and `adr-dep-graph.sh`.)
