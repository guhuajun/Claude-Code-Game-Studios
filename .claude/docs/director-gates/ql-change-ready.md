> Gate definition. The spawning skill passes this file's path to the director agent; the AGENT reads it — the parent session should not.

# QL-CHANGE-READY — QA Lead Change Readiness Check

Agent: `qa-lead` | Model tier: Sonnet (Tier 2 lead — invoked when a domain specialist's feasibility sign-off is needed)

**Trigger**: Before a change is accepted for implementation — invoked by
`/create-changes` and `/change-readiness` during change selection

**Context to pass**:
- Change id / path (`openspec/changes/<change-id>/`)
- Change type (Logic / Integration / Visual/Feel / UI / Config/Data)
- Acceptance criteria list (verbatim from the change's `tasks.md` and delta)
- The capability requirement (TR-ID and text) the change covers

**Prompt**:
> "Review this change's acceptance criteria for testability before it enters
> implementation. Are all criteria specific enough that a developer would know
> unambiguously when they are done? For Logic-type changes: can every criterion
> be verified with an automated test? For Integration changes: is each criterion
> observable in a controlled test environment? Flag criteria that are too vague
> to implement against, and flag criteria that require a full game build to test
> (mark these DEFERRED, not BLOCKED). Return ADEQUATE (criteria are implementable
> as written), GAPS [specific criteria needing refinement], or INADEQUATE
> [criteria are too vague — the change must be revised before implementation]."

**Verdicts**: ADEQUATE / GAPS / INADEQUATE
