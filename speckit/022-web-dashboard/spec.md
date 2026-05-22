# 022 — Web Dashboard

**Status:** Draft
**Constitutional anchor:** Principle I (Surfaces), VI (Observable), IX (HITL)
**Depends on:** 023 (consumes API)
**Required by:** none (delivers UX)

## Purpose

The visual command center. Approvals, project status, agent health, fleet view, cost, audit, benchmarks, workflow diagrams, change review. AdonisJS + Inertia (Vue or React); SSR with client-side reactivity.

## Top-level navigation

1. **Home** — health overview, recent activity, pending approvals count, cost snapshot.
2. **Projects** — per-project view: pipeline progress, current stage, owners, cost, artifacts, approvals.
3. **Teams** — per-team view: org chart, agent health, task-management driver state, flow diagram.
4. **Agents** — per-agent: definition, recent runs, self-mod PRs, benchmark trend, cost history.
5. **Tasks** — global task list and detail view (trace + audit).
6. **Approvals** — pending queue + history, with bulk actions.
7. **Audit** — query interface; integrity verification UI.
8. **Cost** — ledger views, budget management, burndowns.
9. **Marketplace** — installable skills, plugins, starter packs; signature status.
10. **Settings** — operator profile, tokens, integration configs (Slack, Telegram, channels).

## Functional requirements

| ID | Requirement |
|---|---|
| 022-FR-001 | Every state in the system MUST be visible in the dashboard (no hidden state). |
| 022-FR-002 | Approvals MUST surface within 1s of creation (WebSocket push). |
| 022-FR-003 | Project view MUST show the pipeline state machine (spec 016) as a live diagram. |
| 022-FR-004 | Team view MUST show org chart (spec 013) and current flow diagram (spec 026 if task driver configured). |
| 022-FR-005 | Agent view MUST show: definition files, recent runs, self-mod PRs (diff + benchmark delta), benchmark trend chart, cost over time. |
| 022-FR-006 | Self-mod PR view MUST support: diff, CHANGE.md, benchmark delta, related task links, merge / reject / force-merge (with reason). |
| 022-FR-007 | Approval card MUST show: requesting agent, action, resource, context, decision actions, channel-of-origin. |
| 022-FR-008 | Cost views MUST support roll-up by agent / team / project / day, with drill-down to LLM-call detail. |
| 022-FR-009 | Audit view MUST support text search + structured filter (actor, action, time range). |
| 022-FR-010 | Real-time logs / traces for active tasks MUST stream via WebSocket. |
| 022-FR-011 | Dashboard MUST be responsive (works on desktop and mobile browser, though desktop-first). |
| 022-FR-012 | Dark mode + light mode. |
| 022-FR-013 | Outcall integration status (mode + recent decisions) MUST be visible on the platform health view. |
| 022-FR-014 | Force-override actions (force-merge, force-transition, force-deploy) MUST require an inline reason field in the modal, persisted to audit. |
| 022-FR-015 | Operator profile MUST allow per-operator notification preferences (channel routing for high vs low urgency). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 022-NFR-001 | First load on home page < 2s on commodity broadband. |
| 022-NFR-002 | WebSocket reconnection MUST be transparent. |
| 022-NFR-003 | All UI strings MUST be i18n-ready (single language v1 = English; structure permits future translations). |

## User stories

- **022-US-001 [P0]** — As an operator, I see the project's pipeline diagram with the current stage highlighted.
- **022-US-002 [P0]** — As an operator, I click "Approve with rule" and see the auto-generated rule preview before committing.
- **022-US-003 [P0]** — As an operator, I scroll a task's trace tree and pivot to its audit events.
- **022-US-004 [P1]** — As an operator, I see the team's flow diagram with denied transitions surfaced as red edges with reasons.
- **022-US-005 [P1]** — As an operator, the dashboard shows me "12 force-overrides this month, 3% — healthy".

## Acceptance criteria

- All major views render and update in real time.
- Approval surfacing latency < 1s.
- WebSocket reconnect is invisible.
- Force-override forces a reason field.

## Related specs

`016`, `013`, `019`, `026`, `005`, `006`, `009`, and `023` (everything renders through the API).
