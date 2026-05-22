# 014 — Inter-Agent Communication (Tickets)

**Status:** Draft
**Constitutional anchor:** Principle V (Durable), VI (Observable)
**Depends on:** 004, 013
**Required by:** 016

## Purpose

Agents communicate via durable tickets — never direct container-to-container. Tickets are first-class rows in the control plane. Inspired by Paperclip's ticketing; tightened with auditable rules from spec 013.

**Disambiguation:** "Ticket" in this spec means an *agent-to-agent message* — a coder asking a reviewer a question, a manager pinging a security agent about a finding. These are **not** the same as external work items (Linear issues, Jira tickets, GitHub Projects cards); those are covered by spec 026. The two systems coexist: an external work item may spawn many internal agent-to-agent tickets during its lifecycle.

## Why this matters

Direct in-process subagent calls are the most reliably-broken pattern in the survey. Ticket-based comms means restart-safe, audit-visible, rate-limitable, and approval-gateable communication.

## Functional requirements

| ID | Requirement |
|---|---|
| 014-FR-001 | A ticket MUST be a durable record with: id, from (agent or system), to (agent or role or `*`), topic, body, attachments (workspace files, related task ids), state. |
| 014-FR-002 | Ticket states: `open → claimed → in_progress → resolved` (or `rejected/dropped`). |
| 014-FR-003 | Ticket delivery MUST respect `flows.yaml` messaging rules; forbidden destinations denied + audited. |
| 014-FR-004 | Rate-limiting per (from, to, topic) MUST prevent ticket storms (configurable defaults). |
| 014-FR-005 | A ticket MAY require approval to deliver (configurable per topic). |
| 014-FR-006 | An agent that receives a ticket MUST spawn a new task (durable) — tickets never invoke a child agent in-process. |
| 014-FR-007 | Ticket history per project MUST be queryable and visible as a thread view in the dashboard. |
| 014-FR-008 | Tickets MUST timeout if not claimed within their TTL; timeout escalates to the team manager role (when present). |
| 014-FR-009 | An operator MUST be able to read, redirect, or close tickets manually from the dashboard. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 014-NFR-001 | Ticket enqueue p99 < 50ms. |
| 014-NFR-002 | Thread view renders 1000-message thread in < 1s. |

## User stories

- **014-US-001 [P0]** — As an operator, I see coder→reviewer tickets in the dashboard thread view.
- **014-US-002 [P0]** — As an operator, an unauthorized cross-team ticket is denied.
- **014-US-003 [P0]** — As an operator, I close a stuck ticket manually with a reason.
- **014-US-004 [P1]** — As an operator, ticket storms (10+ per second from one agent) trigger throttle + alert.

## Acceptance criteria

- All comms durable, audit-visible, rule-enforced.
- Thread view performant.
- Manual override possible.

## Edge cases & failure modes

- Ticket loops (agent A → B → A → B ...) → cycle detector + retry budget per topic.
- Recipient agent rolled back / deleted → ticket dead-letters with operator notification.
- Attachment too large → caps with explicit error.

## Related specs

`004-control-plane-durable-state`, `013-team-orchestration-orgchart`, `016-software-agency-pipeline`, `022-web-dashboard`, `026-task-management-drivers-flow-rules` (external task systems; distinct from agent tickets).
