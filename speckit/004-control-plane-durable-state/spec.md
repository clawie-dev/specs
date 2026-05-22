# 004 — Control Plane & Durable State

**Status:** Draft
**Constitutional anchor:** Principle V (Durable, idempotent), VI (Observable)
**Depends on:** 001
**Required by:** 003, 005, 006, 014, 016, 017, 020

## Purpose

The control plane is the durable brain. Every agent invocation is a row in a database with a state machine, idempotency key, and recovery semantics. No work lives only in memory; no subagent is "in process". Restarting any node leaves the system recoverable.

## Why this matters

Survey: Paperclip moved to durable kanban for this exact reason. LangGraph's durable execution is the headline feature. Hermes shipped a board model after losing work to crashes. OpenClaw's instability cited by the user is largely state-in-memory.

## Functional requirements

| ID | Requirement |
|---|---|
| 004-FR-001 | All tasks MUST be persisted in a relational store (Postgres in prod, SQLite for single-host dev). |
| 004-FR-002 | Each task MUST have an idempotency key derived from (project, agent, intent, payload-hash). Re-queuing an identical task MUST not produce duplicate side effects. |
| 004-FR-003 | Each task MUST follow the canonical state machine: `created → queued → claimed → running → awaiting_approval → running → completing → completed` (or `failed/aborted/timed_out` as terminal states from any non-terminal state). |
| 004-FR-004 | State transitions MUST be atomic (single-row update with optimistic locking) and logged to audit. |
| 004-FR-005 | A claim MUST include a worker id and a TTL; expired claims MUST be reclaimable by another worker. |
| 004-FR-006 | Each task MUST emit checkpoints: structured progress markers from the agent runtime. On restart, the task SHOULD resume from the last checkpoint (best-effort; agent decides what's safe). |
| 004-FR-007 | A pending approval MUST suspend the task with the model context preserved; on approval, the task resumes without re-running prior steps. |
| 004-FR-008 | Child tasks (spawned by a parent agent's tool call) MUST be first-class durable tasks, not in-process subagents. |
| 004-FR-009 | The control plane MUST expose a typed event stream (WebSocket / SSE) of state transitions for live dashboards and tooling. |
| 004-FR-010 | Recovery: on platform start, the control plane MUST scan for orphaned tasks (claimed but worker dead) and re-queue them with reclaim policy. |
| 004-FR-011 | Locks (workspace, agent repo, project) MUST be expressed as durable rows; lease-based with TTL; auto-released on lease expiry. |
| 004-FR-012 | No task may be silently dropped. Every queued task either completes, fails with cause, or is explicitly aborted. |
| 004-FR-013 | Tasks MUST be indexed for query: by project, team, agent, status, time range. Dashboard queries MUST be sub-second on 100k tasks. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 004-NFR-001 | Enqueue p99 < 50ms. Claim p99 < 100ms. Transition p99 < 100ms. |
| 004-NFR-002 | Recovery sweep at boot completes in <30s for ≤10k orphaned tasks. |
| 004-NFR-003 | Postgres is canonical. SQLite mode is for single-host dev, tested but not load-bearing for production. |

## Task schema (sketch)

```
tasks
├── id (uuid)
├── idempotency_key
├── project_id, team_id, agent_id
├── parent_task_id (nullable)
├── intent (string, e.g., "research-brief", "implement-feature")
├── payload (jsonb)
├── status (enum)
├── claimed_by, claim_expires_at
├── created_at, started_at, finished_at
├── result (jsonb, nullable)
├── failure_cause (enum), failure_detail (text)
├── checkpoints[] (jsonb array, append-only)
└── cost_total (numeric)
```

## User stories

- **004-US-001 [P0]** — As an operator, I kill the API mid-task and on restart the task continues from its last checkpoint.
- **004-US-002 [P0]** — As an operator, I view a project's tasks as a kanban (queued, running, awaiting, completed) live-updating.
- **004-US-003 [P0]** — As an agent, I emit checkpoints during a long task; if I crash, my next incarnation reads them.
- **004-US-004 [P1]** — As an operator, I query tasks by status across 100k history and get results <1s.
- **004-US-005 [P1]** — As an operator, I see a graph of parent-child tasks for a project run.

## Acceptance criteria

- Kill -9 the API → restart → no lost tasks, no duplicate side-effects.
- Two workers cannot claim the same task; second loses optimistic lock cleanly.
- All transitions visible in audit and live event stream.
- Approval pause + resume restores the task with no replay of completed work.

## Edge cases & failure modes

- Database unreachable → control plane refuses to claim new tasks; running tasks finish their current step and gracefully await reconnect.
- Clock skew → leases use monotonic+wall hybrid; expiry honored conservatively.
- Storm of failures (rate-limit from LLM) → exponential backoff at task level; circuit breaker per provider.
- Checkpoint payload too large → cap with explicit error; agent must summarize.

## Open questions

- Job runner: BullMQ (Redis-backed) for queue + Postgres for state, or fully-in-Postgres (pg-boss)? Decision in plan phase. (Lean: pg-boss for simplicity; one fewer dependency.)

## Related specs

`002-container-runtime-outcall`, `005-approvals-hitl`, `006-observability-audit-cost`, `014-inter-agent-comms-tickets`, `017-multi-project-juggling`, `020-stability-recovery`.
