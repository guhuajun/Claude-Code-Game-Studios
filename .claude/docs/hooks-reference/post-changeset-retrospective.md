# Hook: post-changeset-retrospective

## Trigger

Manual trigger at the end of a milestone or a change set (typically invoked by
the producer agent or the human developer).

## Purpose

Automatically generates a retrospective starting point by analyzing the
completed changes: what was planned vs completed, throughput changes, bug
trends, and common blockers. This is not a git hook but a workflow hook invoked
through the `producer` agent.

> There is no sprint. OpenSpec's in-flight changes are the unit of work, and a
> retrospective now covers a *change set* — the changes closed in a period — or
> a milestone, rather than a time box.

## Implementation

This is a workflow hook, not a git hook. It is invoked by running:

```
@producer Generate a retrospective for the changes closed since [date]
```

The producer agent should:

1. **Read the completed changes** under `openspec/changes/archive/`, and the
   capability specs they touched under `openspec/specs/`.
2. **Calculate metrics**:
   - Changes planned vs completed
   - Tasks planned vs completed (checkbox markers in each change's `tasks.md`)
   - Carryover items from a previous change set
   - New changes opened mid-period
   - Average change completion time
3. **Analyze patterns**:
   - Most common blockers
   - Which agent/area had the most incomplete work
   - Which estimates were most inaccurate
4. **Generate the retrospective**:

```markdown
# Change-Set Retrospective — [date range]

## Metrics
| Metric | Value |
|--------|-------|
| Changes Planned | [N] |
| Changes Completed | [N] |
| Completion Rate | [X%] |
| Carryover from Previous | [N] |
| New Changes Opened | [N] |
| Bugs Found | [N] |
| Bugs Fixed | [N] |

## Throughput Trend
[Period N-2]: [X] | [Period N-1]: [Y] | [Period N]: [Z]
Trend: [Improving / Stable / Declining]

## What Went Well
- [Automatically detected: changes completed ahead of estimate]
- [Facilitator adds team observations]

## What Went Poorly
- [Automatically detected: changes that were carried over or cut]
- [Automatically detected: areas with significant estimate overruns]
- [Facilitator adds team observations]

## Blockers
| Blocker | Frequency | Resolution Time | Prevention |
|---------|-----------|----------------|-----------|

## Action Items for Next Period
| # | Action | Owner | Priority |
|---|--------|-------|----------|

## Estimation Accuracy
| Area | Avg Planned | Avg Actual | Accuracy |
|------|------------|-----------|----------|
```

5. **Save** to `production/retrospectives/retro-[date]-[slug].md`
