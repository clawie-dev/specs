# 017 — Multi-Project Juggling & Scheduling

**Status:** Draft
**Constitutional anchor:** Principle V (Durable), VIII (Stability)
**Depends on:** 004, 007, 013, 016
**Required by:** none (delivers UX on multi-project setups)

## Purpose

Multiple concurrent projects share a team's agents. The scheduler ensures fairness, respects priorities, surfaces contention, and prevents starvation. Operator sees per-project queues and can rebalance.

## Functional requirements

| ID | Requirement |
|---|---|
| 017-FR-001 | Each project MUST have an explicit priority (`low/normal/high/urgent`) and a fair-share weight (default per-team config). |
| 017-FR-002 | Agents MUST claim tasks across projects per a configurable strategy: `weighted-fair-share` (default), `priority-strict`, or `round-robin`. |
| 017-FR-003 | The scheduler MUST surface starvation alerts: a project with queued tasks aging beyond threshold triggers operator notification. |
| 017-FR-004 | Per-project concurrency caps MUST be configurable (e.g., max 2 simultaneous tasks per project). |
| 017-FR-005 | Operator MAY pause or boost individual projects without affecting others. |
| 017-FR-006 | Cross-project workspace locks MUST be respected (a workspace is per-project, but a single agent across two projects must not deadlock). |
| 017-FR-007 | Dashboard MUST show per-project queue depth, average wait, current spend, and progress to next stage. |
| 017-FR-008 | "Project view" and "Team view" complement each other: team view shows agents and current load; project view shows pipeline and bottleneck. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 017-NFR-001 | Scheduler decision overhead < 50ms even with 100 active projects. |
| 017-NFR-002 | Starvation threshold default: 1h queued without claim → alert. |

## User stories

- **017-US-001 [P0]** — As an operator, I run 3 projects concurrently and none starves.
- **017-US-002 [P0]** — As an operator, I boost project X to `urgent` and the scheduler shifts agent attention.
- **017-US-003 [P1]** — As an operator, an alert tells me project Y has been queued 2h with no claim — I see the cause (e.g., budget paused, agent at capacity).

## Acceptance criteria

- Fair share enforced under load.
- Starvation alerts fire correctly.
- Operator can boost / pause per project.
- Dashboard reflects scheduling decisions in real-time.

## Edge cases & failure modes

- All projects high priority → degrade to round-robin; surface warning.
- Workspace cross-locks → deadlock detector unwinds with cause logged.
- Agent crashes during multi-project run → recovery (spec 020) restarts with original project's task.

## Related specs

`004-control-plane-durable-state`, `007-budget-governance`, `013-team-orchestration-orgchart`, `016-software-agency-pipeline`, `020-stability-recovery`.
