# /dev-change — Handoff Contract

## Role in Pipeline
Bridges planning and code by loading the full context for a single change (change directory, TR registry, ADR, control manifest, engine prefs), routing to the correct programmer agent, implementing source code and tests, and leaving the change ready for `/code-review` then `/change-done`.

## Inputs Required

### Files That Must Exist
| File | Required Fields / Sections | Read-Only? |
|------|---------------------------|-----------|
| `openspec/specs/**/*.md` (the change directory) | `Status: Ready`, `TR-ID`, `Governing ADR`, `Manifest Version`, `## Acceptance Criteria`, `## Test Evidence`, `## Dependencies`, `## Out of Scope` | **Partially mutable.** This skill sets `Status:` to `In Progress` and updates `Last Updated:` before spawning any agent (Phase 2) — at `minimal` the change directory is the only record of progress, since `openspec status` is absent. Everything else in the change directory is read-only to this skill |
| `docs/architecture/tr-registry.yaml` | Entry matching the change's TR-ID with current `requirement` text | Yes |
| `docs/architecture/adr-NNNN-[slug].md` (referenced ADR) | `## Decision`, `## Implementation Guidelines`, `## Engine Compatibility`, `## ADR Dependencies` | Yes |
| `docs/architecture/control-manifest.md` | `Manifest Version:` header, layer rules (required patterns, forbidden patterns, performance guardrails) | Yes |
| `project.yaml` | `engine.name`, `engine.version`, `naming.*`, `performance.*` — the primary source for each of these fields | Yes |
| `.claude/docs/technical-preferences.md` | Legacy fallback for any field above that is absent or empty in `project.yaml`; **sole** source for forbidden patterns and allowed libraries (deliberately never migrated), plus the `Engine Specialists` section | Yes — required only for fields absent from `project.yaml`, and always for forbidden patterns |
| `docs/engine-reference/[engine]/VERSION.md` | Engine version, LLM knowledge cutoff, post-cutoff risk levels | Yes |
| `production/session-state/active.md` | (read to find active change when no argument given; created if absent) | No |

### Preconditions
- The target change must have `Status: Ready` (validated by `/change-readiness` before invocation)
- The ADR referenced in the change must have `Status: Accepted` — if `Status: Proposed`, do not implement; surface this as a blocker and direct user to `/architecture-decision`
- All `## Dependencies` changes listed in the change directory must already have `Status: Complete` (or `Status: Done`)
- An engine must be configured — `engine.name` in `project.yaml`, or as legacy fallback an `**Engine**:` value in `.claude/docs/technical-preferences.md` that is not `[TO BE CONFIGURED]`
- `docs/architecture/tr-registry.yaml` must contain the change's TR-ID
- `docs/architecture/control-manifest.md` must exist (required for layer rule enforcement)

## Outputs Produced

### Files Written
| File | Guaranteed Fields / Sections | Notes |
|------|------------------------------|-------|
| `<code root>/[system]/[file].[ext]` | Doc-commented public APIs; no hardcoded gameplay values; follows ADR Implementation Guidelines and control-manifest required patterns | created or modified by sub-agent; code root resolved per `.claude/docs/code-root-resolution.md` |
| `tests/unit/[system]/[change-slug]_test.[ext]` OR `tests/integration/[system]/[change-slug]_test.[ext]` | One test function per acceptance criterion (Logic/Integration changes only); naming: `test_[scenario]_[expected_outcome]`; no random seeds; no external I/O | created by sub-agent; path matches `## Test Evidence` in change |
| `production/session-state/active.md` | `## Session Extract — /dev-change [date]` block: change id, files changed, test written path, blockers, next steps | appended (created if absent) |

### Output Guarantees
- All source files follow the naming conventions in `project.yaml` (`naming.*`), falling back to `.claude/docs/technical-preferences.md` for any convention absent there
- Test file path matches exactly the path declared in the change's `## Test Evidence` section
- Every acceptance criterion is mapped to a test function (Logic/Integration), an `OBSERVED` retained screenshot (the *look* half of Visual/Feel and UI), or flagged `DEFERRED` (the *feel* half only — timing, weight, responsiveness) in the implementation summary
- No files are written outside the change's `## Out of Scope` boundary without surfacing the conflict to the user first
- Session state records files changed and the next recommended invocations (`/code-review` then `/change-done`)
- The implementation summary states a **verification** result — what was run against the written code (`commands.test`, `commands.smoke`, or the engine's headless import) and its outcome. When the engine binary is unavailable this is **`NOT VERIFIED — <reason>`**, never an inference that the code is fine because it reads correctly
- The implementation summary states a **run result** — `OBSERVED — <what was on screen>` with a retained screenshot under `production/qa/evidence/[change-slug]/`, `NOT VERIFIED — <reason>`, or `N/A — <reason>` (only for a change with genuinely nothing observable). `NOT VERIFIED` is a blocker at the default gate level for Visual/Feel and UI changes. The run is **not waived at `qa.level: minimal`** — tests are, the look is not. Procedure: `.claude/docs/run-and-observe.md`
- At `qa.level: minimal` no test file is written, and the summary says so explicitly (`Test evidence: waived at qa.level: minimal`). A waived run and a run where tests were forgotten must not produce the same artifact
- **`NOT ASSESSED` is an accepted inbound value of the change's `Risk` field** (`/create-changes` emits it when `VERSION.md` assigns no level). This skill treats it as HIGH for the engine-specialist spawn decision
- **Completion is not assumed.** If the programmer agent stopped early or its output fails to parse, the change is reported **INCOMPLETE** with the specific breakage named, and `Implementation Complete` is not emitted

## Immutability Rules
- READS but does NOT modify: `project.yaml`, `docs/architecture/tr-registry.yaml`, `docs/architecture/adr-NNNN-[slug].md`, all GDD files in `design/gdd/`, `docs/architecture/control-manifest.md`, `.claude/docs/technical-preferences.md` (legacy fallback), `docs/engine-reference/[engine]/VERSION.md`
- MODIFIES: `<code root>/**` (new/updated source files, via sub-agent; code root resolved per `.claude/docs/code-root-resolution.md`), `tests/**` (new test file, via sub-agent), `production/session-state/active.md` (append only)
- Updates the change directory's `Status` field to `In Progress` only (Phase 2, before
  spawning any agent). It writes no other `Status` value: advancing a change to
  `Complete` is the exclusive responsibility of `/change-done`

## Hard Constraints (Never Violate)
- Never marks a change `Status: Done` or `Status: Complete` — that belongs to `/change-done`
- Never begins implementation without loading the full context package (change, TR-ID from registry, ADR Decision + Implementation Guidelines, control-manifest layer rules, engine prefs)
- Never implements a change whose referenced ADR has `Status: Proposed` — surface as a blocker
- Never writes a Logic or Integration change as complete without a test file at the path declared in `## Test Evidence`
- Never touches files listed in the change's `## Out of Scope` section without explicit user approval
- Never deviates from the ADR's Implementation Guidelines silently — deviations must be flagged in the summary
- Never modifies ADR files or GDD files under any circumstances
- All file writes are delegated to sub-agents via Task; this orchestrator does not write source or test files directly

## Downstream Skill Expects
**Next skill:** `/change-done`
- It will read: the change directory (to check `Status`, `## Acceptance Criteria`, `## Test Evidence` path, `Type:`, referenced ADR, TR-ID)
- It assumes: the test file declared in `## Test Evidence` already exists on disk (for Logic/Integration changes)
- It assumes: `production/session-state/active.md` records the change id and files changed (used to locate the in-progress change when no argument is given)
- It assumes: source files under the resolved code root are present and match the criterion descriptions, so Grep-based deviation checks can run

## Known Fragile Points
- If the change's embedded `Manifest Version` differs from the current `docs/architecture/control-manifest.md` header date, rules may have changed since the change was authored — the skill surfaces this, but silently proceeding would produce code that violates newer manifest rules
- The test file path in `## Test Evidence` must be exact: `/change-done` uses that literal path for Glob checks; a typo here causes a false BLOCKING result downstream
- If `docs/architecture/tr-registry.yaml` has a stale `requirement` text that diverges from the GDD, the implementation may satisfy the registry but fail the GDD — always treat the registry as authoritative per design, but flag if a visible discrepancy is noticed
- Sub-agent task failures are non-fatal by design (partial reports are surfaced), but a blocked programmer sub-agent means no source files are written — `/change-done` will then fail its file-existence checks
- Config/Data changes produce no test file; `/change-done` must see `Type: Config/Data` in the change header to classify the change correctly — otherwise it resolves the test-evidence gate for the wrong type (e.g. Logic, which is BLOCKING by default) and may flag a missing test
