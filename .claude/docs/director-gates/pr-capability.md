> Gate definition. The spawning skill passes this file's path to the director agent; the AGENT reads it — the parent session should not.

# PR-CAPABILITY — Capability Structure Feasibility Review

Agent: `producer` | Model tier: Opus | Domain: Scope, timeline, dependencies, production risk

**Trigger**: After capabilities are defined by `/design-system`, before changes
are broken out — validates the capability structure is producible before
`/create-changes` is invoked

**Context to pass**:
- Capability spec paths (all capabilities just designed,
  `openspec/specs/<system>/spec.md`)
- The capability inventory (`openspec list --specs`)
- Milestone timeline and target dates
- Team capacity (solo / small team / size)
- Layer being specified (Foundation / Core / Feature / etc.)

**Prompt**:
> "Review this capability structure for production feasibility before change
> breakdown begins. Are the capability boundaries scoped appropriately — could
> each capability realistically be built before a milestone deadline? Are
> capabilities correctly ordered by system dependency — does any capability
> require another's output before work can start? Are any capabilities
> underscoped (too small, should merge) or overscoped (too large, should split
> into 2-3 focused capabilities)? Are the Foundation-layer capabilities scoped
> to allow Core-layer capabilities to begin once Foundation completes? Return
> REALISTIC (capability structure is producible), CONCERNS [specific structural
> adjustments before changes are written], or UNREALISTIC [capabilities must be
> split, merged, or reordered — change breakdown cannot begin until resolved]."

**Verdicts**: REALISTIC / CONCERNS / UNREALISTIC
