# 020 — Stability, Liveness, Recovery

**Status:** Draft
**Constitutional anchor:** Principle VIII (Stability is a P0 feature)
**Depends on:** 002, 004, 006
**Required by:** every running spec

## Purpose

Catch the failure modes that bite OpenClaw, Hermes, Paperclip. Stuck agents, runaway costs, dead workflows, deadlocks, ghost containers, infinite retry loops. Surface causes; recover automatically where safe; escalate where not.

## Why this matters

The user's direct statement: "we should see failures all the time, like OpenClaw. My openclaw often stops responding etc, cause of various reasons". Stability is the difference between "demo" and "framework".

## Failure modes catalog

| Mode | Detection | Recovery |
|---|---|---|
| Container OOM | Cgroup OOM signal | Mark task `failed`, cause `oom`, surface resource recommendation |
| Wall-clock timeout | Per-task TTL | Reap container, mark `timed_out` |
| Lost heartbeat | No ping for N×interval | Mark task `stuck`, attempt graceful, then force reap |
| Deadlock (waiting on ticket/approval that won't come) | Pending state > threshold, no progress | Alert operator, optionally auto-abort |
| Provider rate-limit storm | Counters per provider | Backoff, fall back per spec 011 |
| Cost runaway | Live ledger check | Abort per spec 007 |
| Infinite agent loop (max iter exceeded) | Pipeline iteration counter | Escalate to operator |
| Ghost container (running, not tracked) | Periodic reconciliation between Docker state and DB | Reap |
| Orphan claim (worker died holding claim) | Lease TTL expired | Reclaim by another worker |
| Validation loop (config keeps failing validation) | Repeated validation rejections | Block agent's self-mod budget; alert |
| Unknown cause | Default bucket | Increment metric; alert if rising |

## Functional requirements

| ID | Requirement |
|---|---|
| 020-FR-001 | Every running task MUST emit liveness ping at configurable interval (default 30s). |
| 020-FR-002 | Missed N pings (default 3) MUST mark the task `stuck` and trigger recovery. |
| 020-FR-003 | Every task MUST have a wall-clock TTL (default 60 min, configurable per intent). |
| 020-FR-004 | Container reconciliation MUST run periodically (default 5 min): list Docker containers, cross-check DB; orphans reaped; ghost containers reaped. |
| 020-FR-005 | Lease TTL on task claims (default 5 min); expired leases reclaimable. |
| 020-FR-006 | Deadlock detection: any waiting state (ticket pending, approval pending, lock pending) past a threshold without progress alerts the operator. |
| 020-FR-007 | Cause-of-failure MUST be captured for every non-success terminal state. Codes per spec 006. |
| 020-FR-008 | Retries MUST be bounded with exponential backoff; "kill switch" budget per task to prevent infinite loops. |
| 020-FR-009 | Cause-rising metric: any cause class above baseline triggers an alert. |
| 020-FR-010 | Graceful shutdown: SIGTERM-handling control plane drains tasks (no new claims), waits for in-flight to checkpoint, then exits. |
| 020-FR-011 | Boot reconcile: on startup, recover orphans + scan for ghost containers + verify submodule SHAs + replay any queued audit events. |
| 020-FR-012 | Dashboard MUST surface "stuck task count", "ghost containers reaped this hour", "cause distribution" as platform health. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 020-NFR-001 | Detection of stuck task < (interval × N) + 30s. |
| 020-NFR-002 | Recovery actions MUST be idempotent — repeated recovery is safe. |
| 020-NFR-003 | Failure metric retention: 90 days minimum for cause-trending. |

## User stories

- **020-US-001 [P0]** — As an operator, a container that stops responding is reaped within 2 minutes; task marked `stuck` with cause.
- **020-US-002 [P0]** — As an operator, a deadlocked approval (no operator response) times out at decision-window expiry; task fails with `cause=approval_timeout`.
- **020-US-003 [P0]** — As an operator, a ghost container leftover from a crashed run is auto-reaped at next reconcile.
- **020-US-004 [P1]** — As an operator, dashboard shows a spike in `cause=provider_rate_limit` — I see the trend and react.
- **020-US-005 [P1]** — As an operator, I issue SIGTERM to the API; running tasks checkpoint and the platform restarts cleanly with zero lost work.

## Acceptance criteria

- Every failure mode in the catalog has a detection mechanism and a documented recovery path.
- Cause-of-failure captured for 100% of failures.
- Boot reconcile is idempotent.
- Graceful shutdown works under load.

## Edge cases & failure modes (meta)

- Detection mechanism itself fails → outer watchdog on the control plane (e.g., systemd unit / supervisor restart).
- Recovery loop (auto-recover keeps retrying a failing task) → bounded retry; escalate after.
- Reconciler kills a container that was about to ping → mitigated by grace window in reconcile.

## Related specs

`002-container-runtime-outcall`, `004-control-plane-durable-state`, `006-observability-audit-cost`, `005-approvals-hitl`, `007-budget-governance`.
