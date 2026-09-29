# /change-readiness — Handoff Contract

## Role in Pipeline
Validates that a change directory contains everything a developer needs to begin
implementation and produces a READY / NEEDS WORK / BLOCKED / NOT ASSESSED verdict — sits between
`/create-changes` and `/dev-change`, acting as a blocking gate before any code is written.

## Inputs Required

### Files That Must Exist
| File | Required Fields / Sections | Read-Only? |
|------|---------------------------|-----------|
| `openspec/changes/<change-id>/tasks.md` | All header fields (`Status:`, `Layer:`, `Type:`, `Manifest Version:`), `## Context` (GDD path, TR-ID, ADR reference, Engine Notes, Control Manifest Rules), `## Acceptance Criteria`, `## Test Evidence`, `## Dependencies`, `## Out of Scope` | Yes |
| `docs/architecture/control-manifest.md` | `Manifest Version:` date in header, layer rules | Yes (if exists) |
| `docs/architecture/tr-registry.yaml` | `id`, `status` fields per TR entry | Yes (if exists) |
| `docs/architecture/adr-NNNN-[slug].md` (all ADRs referenced in changes) | `Status:` field | Yes |
| `design/gdd/systems-index.md` | Which systems have approved GDDs | Yes |

### Preconditions
- The change directory being validated was produced by `/create-changes` (or follows the same format)
- For `change set` scope: a change list exists in `openspec/changes/` with change id references
- For `all` scope: `openspec/specs/` directory exists and contains change directorys

## Outputs Produced

### Files Written
None. This skill is strictly read-only and produces no file modifications under any circumstance.

### Output Guarantees
- A verdict of READY, NEEDS WORK, BLOCKED or NOT ASSESSED is produced for every change directory evaluated. `NOT ASSESSED` is not a synonym for `BLOCKED`: BLOCKED names a real, listable obstacle, while NOT ASSESSED means the change could not be evaluated at all. An empty scope yields `NOT ASSESSED — no changes in scope`, never a `Ready: 0 / Needs Work: 0 / Blocked: 0` summary over an empty list
- Every non-READY verdict includes a specific gap list with fix instructions for each failing checklist item
- Every BLOCKED verdict names the specific blocker (missing dependency change id, Proposed ADR ID, or unresolved design question marker)
- For `change set` scope: a change set-level escalation warning is prepended if any Must Have change is not READY
- The skill offers to draft missing sections in conversation but never uses Write or Edit tools

## Immutability Rules
- READS but does NOT modify: change directorys, `docs/architecture/control-manifest.md`, `docs/architecture/tr-registry.yaml`, all ADR files, `design/gdd/systems-index.md`, change lists, any referenced asset files (existence-only Glob checks)
- MODIFIES: nothing

## Hard Constraints (Never Violate)
- Never use Write or Edit tools under any circumstances
- Never draft corrections directly into files — offer drafts in conversation only
- Never mark a change READY if its governing ADR has `Status: Proposed`
- Never mark a change READY if a dependency change directory is missing or has `Status: Draft`
- Never re-read the same ADR file multiple times in one run — cache ADR statuses after the first read
- Never penalize a change for missing `Manifest Version:` if `control-manifest.md` does not exist
- Never penalize a change for missing TR-ID if `tr-registry.yaml` does not exist

## Downstream Skill Expects
**Next skill:** /dev-change

It will rely on this skill's verdict as follows:
- If verdict is READY: `/dev-change` proceeds with implementation using the change directory as its source of truth
- If verdict is NEEDS WORK or BLOCKED: `/dev-change` must not be run until the change is corrected and re-validated
- `/dev-change` reads the same change fields this skill validates — a READY verdict is an implicit guarantee that those fields are present, parseable, and internally consistent:
  - `Type:` field is set to a valid change type
  - `## Acceptance Criteria` contains specific, testable checkbox items
  - `## Test Evidence` specifies a concrete evidence path
  - The governing ADR exists and has `Status: Accepted`
  - `Manifest Version:` matches the current control manifest
  - `TR-ID` in `## Context` is present and resolves in `docs/architecture/tr-registry.yaml` (when the registry exists)
  - `## Dependencies` section is present (may say "None")
  - `## Out of Scope` section is present (checked for existence; `/dev-change` uses it to enforce implementation boundaries)

## Known Fragile Points
- If `control-manifest.md` is regenerated with a new `Manifest Version:` date, every change that embeds the old date will fail the manifest version check — a bulk update of all change `Manifest Version:` fields is required before changes can pass readiness again
- If an ADR is silently renamed or moved after changes reference it by ID, the ADR file-existence check will BLOCK those changes even though the decision content is unchanged
- The manifest version comparison is a string date match, not a semantic version comparison — if the date format changes (e.g., from `2026-01-15` to `Jan 15 2026`), all changes will fail the check regardless of actual currency
- Asset existence checks use Glob — on Windows, path separator mismatches (`\` vs `/`) may cause a Glob miss that incorrectly flags an asset as missing
- The TR-ID status check auto-passes if the registry does not exist, which means changes written before TR tracking was introduced will always pass this check even if they contain invalid or placeholder IDs like `TR-[system]-???`
- For `change set` scope, the skill depends on change list formatting to extract change ids — if the change list format changes, change ids may not be parsed and the scope silently reduces to zero changes validated
