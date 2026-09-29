> Gate definition. The spawning skill passes this file's path to the director agent; the AGENT reads it — the parent session should not.

# QL-TEST-COVERAGE — QA Lead Test Coverage Review

Agent: `qa-lead` | Model tier: Sonnet (Tier 2 lead — invoked when a domain specialist's feasibility sign-off is needed)

**Trigger**: After implementation changes are complete, before marking a
capability done, or at `/gate-check` Production → Polish

**Context to pass**:
- List of implemented changes with their types (Logic / Integration / Visual / UI / Config)
- Test file paths in `tests/`
- The capability spec's requirements and scenarios (`openspec/specs/<system>/spec.md`)

**Prompt**:
> "Review the test coverage for these implementation changes. Are all Logic changes
> covered by passing unit tests? Are Integration changes covered by integration
> tests or documented playtests? Is each requirement scenario in the capability
> spec mapped to at least one test? Are there untested edge cases from the spec's
> Edge Cases section? Return ADEQUATE (coverage meets standards), GAPS [specific
> missing tests], or INADEQUATE [critical logic is untested — do not advance]."

**Verdicts**: ADEQUATE / GAPS / INADEQUATE
