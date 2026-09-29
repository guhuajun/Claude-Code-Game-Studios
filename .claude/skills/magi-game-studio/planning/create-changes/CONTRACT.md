# /create-changes — Handoff Contract

## Role in Pipeline
Decomposes a single capability into implementable change directorys by tracing GDD acceptance
criteria through ADR decisions and manifest rules — sits between `/design-system`
and `/change-readiness` + `/dev-change`.

## Inputs Required

### Files That Must Exist
| File | Required Fields / Sections | Read-Only? |
|------|---------------------------|-----------|
| `openspec/specs/<system>/spec.md` | `Layer:`, `GDD:` path, `Architecture Module:`, `## Governing ADRs` table, `## GDD Requirements` table with TR-IDs | Yes |
| `design/gdd/[filename].md` | All 8 required sections, especially `## Acceptance Criteria`, `## Formulas`, `## Edge Cases` | Yes |
| `docs/architecture/adr-NNNN-[slug].md` (all governing ADRs) | `Status:`, `Decision`, `Implementation Guidelines`, `Engine Compatibility`, `Engine Notes` | Yes |
| `docs/architecture/control-manifest.md` | Layer rules (required patterns, forbidden patterns, guardrails), `Manifest Version:` date in header | Yes |
| `docs/architecture/tr-registry.yaml` | All TR-IDs for the capability's system, with stable `id` values | Yes |

### Preconditions
- `/design-system` has been run and the target `spec.md` has `Status: Ready`
- The spec.md contains a populated `## GDD Requirements` table (not empty)
- All governing ADRs listed in the spec.md exist as files
- Foundation capabilities have changes created before Core capabilities are started; Core before Feature; Feature before Presentation

## Outputs Produced

### Files Written
| File | Guaranteed Fields / Sections | Notes |
|------|------------------------------|-------|
| `openspec/changes/<change-id>/tasks.md` | `Capability:`, `Status:`, `Layer:`, `Type:`, `Manifest Version:`, `## Context` (GDD path + `TR-[system]-NNN` + ADR reference + ADR Decision Summary + Engine/Risk + Engine Notes + Control Manifest Rules), `## Acceptance Criteria`, `## Implementation Notes`, `## Out of Scope`, `## Test Evidence`, `## Dependencies` | Created per change |
| `openspec/specs/<system>/spec.md` | `## Changes` table replacing the "Not yet created" placeholder | Updated in-place |

### Output Guarantees
- Every change directory contains a `TR-[system]-NNN` reference in its `## Context` section (or `TR-[system]-???` with a warning if the registry has no match)
- Every change directory contains an `ADR Governing Implementation:` line referencing at least one ADR, or an explicit `No ADR applies` note
- Every change directory has a `Status:` field — either `Ready` or `Blocked` (if the governing ADR is `Proposed`)
- Every change directory has a `Type:` field — one of: Logic, Integration, Visual/Feel, UI, Config/Data
- Every change directory has a `Manifest Version:` field matching the date from `docs/architecture/control-manifest.md`'s header at time of writing
- Every change directory has a `## Test Evidence` section stating the expected evidence location for its type
- Every change directory has a `## Acceptance Criteria` section with at least one checkbox item copied from the GDD
- Every change directory has a `## Dependencies` section (may say "None" but is never omitted)
- The spec.md `## Changes` table is updated with a row per change including `#`, `Change` title, `Type`, `Status`, and `ADR`
- Changes with a `Proposed` ADR have `Status: Blocked` and a note: `BLOCKED: ADR-NNNN is Proposed — run /architecture-decision to advance it`
- Every change directory has an `Engine` and a `Risk` field. `Risk` is taken from `docs/engine-reference/<engine>/VERSION.md`; when that file is missing or assigns no level, `Risk` is **`NOT ASSESSED (no VERSION.md risk rating)`** — never a guessed level
- **`NOT ASSESSED` is an emittable value of the `Risk` field**, and downstream readers must handle it. `/dev-change` treats it as HIGH when deciding whether to spawn the engine specialist: an unknown risk is not a low one

## Immutability Rules
- READS but does NOT modify: the capability's GDD, all ADR files, `docs/architecture/tr-registry.yaml`, `docs/architecture/control-manifest.md`
- MODIFIES: `openspec/changes/<change-id>/tasks.md` (creates), `openspec/specs/<system>/spec.md` (updates Changes section only)

## Hard Constraints (Never Violate)
- Never start implementation — this skill stops at the change directory level
- Never invent acceptance criteria — all criteria must come from the GDD
- Never invent implementation notes — all guidance must come from the referenced ADR
- Never write change directorys without presenting the full change list for approval first
- Never mark a change `Status: Ready` if its governing ADR has `Status: Proposed`
- Never omit the `Manifest Version:` field — downstream readiness checks depend on it
- Never reuse a change number (NNN) that already exists in the capability directory

## Downstream Skill Expects
**Next skill:** /change-readiness (then /dev-change)

It will read:
- Each `change-NNN-[slug].md` file — specifically: `Status:`, `Type:`, `Manifest Version:` (header), `TR-[system]-NNN` (in `## Context`), `ADR Governing Implementation:` line, `## Acceptance Criteria` section, `## Test Evidence` section, `## Dependencies` section
- `docs/architecture/control-manifest.md` — to compare its `Manifest Version:` against the change's embedded version
- The referenced ADR file — to verify its `Status:` field is still `Accepted`

It assumes:
- `Manifest Version:` in the change header is a date string that can be compared against the manifest's current `Manifest Version:` date
- `TR-[system]-NNN` IDs are resolvable in `docs/architecture/tr-registry.yaml`
- The ADR referenced by name in `ADR Governing Implementation:` exists as a file in `docs/architecture/` (pattern: `adr-NNNN-[slug].md`)
- `Status: Ready` means the change is a candidate for assignment (not Draft, not in progress)
- `## Acceptance Criteria` contains checkbox items that are directly testable
- `## Test Evidence` specifies an exact file path or evidence doc path where proof will be stored

## Known Fragile Points
- If the change directory format (header field names, section names, checkbox syntax) changes, `/change-readiness` will silently skip checks for fields it cannot parse — changes may pass readiness that should fail
- If `control-manifest.md` is regenerated with a new `Manifest Version:` date after changes are written, all existing changes will show a stale manifest version and fail the readiness manifest check — every change in the capability will need its `Manifest Version:` updated
- If a governing ADR is renumbered or its file is renamed after changes are written, the `ADR Governing Implementation:` reference in change directorys will point to a missing file and `/change-readiness` will BLOCK those changes
- If `tr-registry.yaml` deprecates or supersedes a TR-ID that is already embedded in change directorys, `/change-readiness` will flag those changes as NEEDS WORK
- The spec.md Changes table update is append-style — if `/create-changes` is run twice for the same capability, duplicate rows will appear in the table
