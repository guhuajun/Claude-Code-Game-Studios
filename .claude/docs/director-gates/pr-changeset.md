> Gate definition. The spawning skill passes this file's path to the director agent; the AGENT reads it — the parent session should not.

# PR-CHANGESET — Change-Set Feasibility Review

Agent: `producer` | Model tier: Opus | Domain: Scope, timeline, dependencies, production risk

**Trigger**: Before finalising a set of in-flight changes (`/create-changes`),
and after any mid-flight scope change

**Context to pass**:
- Proposed change list (titles, estimates, dependencies — `openspec list`)
- The capabilities each change touches
- Team capacity (hours available)
- Outstanding work on already-open changes (if any)
- Milestone constraints

**Prompt**:
> "Review this change set for feasibility. Is the change load realistic for the
> available capacity? Are changes correctly ordered by dependency? Are there
> hidden dependencies between changes that could block work mid-way? Are any
> changes underestimated given their technical complexity? Return REALISTIC
> (set is achievable), CONCERNS [specific risks], or UNREALISTIC [the set must
> be descoped — identify which changes to defer]."

**Verdicts**: REALISTIC / CONCERNS / UNREALISTIC
