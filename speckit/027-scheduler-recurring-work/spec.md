# 027 — Scheduler & Recurring Work

**Status:** Draft
**Constitutional anchor:** Principle II (Git source of truth), V (Durable state), VI (Observable)
**Depends on:** 001, 004, 008
**Required by:** 016, 019, 020

## Purpose

Some work is not initiated by an operator brief — it recurs. Nightly evals (spec 019), container reconcile (spec 020), post-launch monitoring (spec 016), agent self-reflection cycles. Clawie ships a scheduler that:

1. Lets **each agent declare its own cron schedule in its own git repo** (declared in the agent's `CRONS.md` or `schedules.yaml`), so schedule changes are reviewable via the same PR flow as any other agent change.
2. Runs a single core ticker (`* * * * *`, every minute) that consults the registry of active schedules and dispatches due work as ordinary durable tasks.
3. Treats every scheduled invocation as a first-class task (spec 004) — idempotent, recoverable, observable, budget-bound.

## Why this matters

Scheduling is often hidden in agent code or platform internals; bug-prone, opaque, hard to govern. Clawie's commitment to git-as-source-of-truth (Principle II) means schedules must live with the agent they belong to, be reviewable as PRs (spec 009), and roll back with the agent (spec 001).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  AGENT REPO  /agents/<name>/                                │
│  ├── SOUL.md, AGENTS.md, TOOLS.md, ...                       │
│  └── schedules.yaml   ←── per-agent recurring jobs           │
└─────────────────────────────────────────────────────────────┘
                          ↓ (loaded into control plane on agent boot/update)
┌─────────────────────────────────────────────────────────────┐
│  SCHEDULE REGISTRY (DB table in control plane)              │
│   one row per (agent, schedule_id) with active cron expr     │
└─────────────────────────────────────────────────────────────┘
                          ↑
┌─────────────────────────────────────────────────────────────┐
│  CORE TICKER  (* * * * *)                                   │
│   1. wake every minute                                       │
│   2. query schedules due in this minute                      │
│   3. dispatch each as a durable task (spec 004)              │
│   4. record dispatch in audit + ledger                       │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  REGULAR TASK FLOW  → claim → run → reap → metrics           │
└─────────────────────────────────────────────────────────────┘
```

## Per-agent schedule file

`agents/<name>/schedules.yaml`:

```yaml
schedules:
  - id: nightly-self-reflect
    cron: "0 3 * * *"            # 03:00 UTC daily
    intent: self-reflect
    description: Daily reflection on last 24h work; may produce a self-mod PR.
    timezone: UTC                  # only UTC supported in v1
    on_failure: log                # log | escalate
    catch_up: false                # if platform was down, run missed? default false
    max_concurrent: 1              # never run two instances of this schedule at once
    budget_usd: 0.50

  - id: eval-suite
    cron: "30 4 * * *"
    intent: run-eval-suite
    description: Run my own benchmark fixtures, report drift.
    catch_up: true                  # missed runs trigger one catch-up at next tick

  - id: post-launch-health-check
    cron: "*/15 * * * *"            # every 15 minutes
    intent: project-health-check
    description: Check deployed prod service; raise ticket on drift.
    bound_to: project              # MUST be bound to a project (see FR-006)
    when_project_in_stage:
      - post-launch-monitoring
```

## Platform-internal schedules

Beyond per-agent schedules, certain core jobs are platform-internal (not in any agent repo). These live in `clawie.yaml`'s `platform_schedules:` block — also git-versioned:

```yaml
# clawie.yaml
platform_schedules:
  - id: container-reconcile
    cron: "*/5 * * * *"
    handler: control-plane.reconcile-containers
    description: Spec 020 — orphans/ghost reaping.
  - id: ledger-rollup
    cron: "0 * * * *"
    handler: control-plane.cost-rollup
  - id: backup-run
    cron: "0 2 * * *"
    handler: backup.run                  # spec 028
  - id: nightly-eval
    cron: "0 4 * * *"
    handler: eval-harness.nightly        # spec 019
```

## Functional requirements

### Schedule definition

| ID | Requirement |
|---|---|
| 027-FR-001 | Each agent MAY ship a `schedules.yaml`; absence is equivalent to no schedules. |
| 027-FR-002 | Each schedule MUST declare: `id` (unique per agent), `cron` (standard 5-field UTC), `intent` (intent name resolved against the agent's task handlers), `description`. |
| 027-FR-003 | Cron expressions MUST be in UTC. v1 does NOT support arbitrary timezones (UTC-only); local-time scheduling deferred. |
| 027-FR-004 | Optional schedule fields: `on_failure` (log/escalate, default log), `catch_up` (bool, default false), `max_concurrent` (int, default 1), `budget_usd` (per-invocation budget), `bound_to` (project-scoped), `when_project_in_stage` (only fire while project is in listed stages). |
| 027-FR-005 | Schedule changes MUST go through the agent's standard PR review flow (spec 009). A merged change updates the registry within 10s of the merge being detected. |
| 027-FR-006 | Project-bound schedules MUST be associated with a project and MUST stop firing when that project is aborted, completed, or paused. |
| 027-FR-007 | Platform-internal schedules MUST live in `clawie.yaml` under `platform_schedules:`, validated by spec 018, and reviewable as PRs to the root config repo. |

### Ticker + dispatch

| ID | Requirement |
|---|---|
| 027-FR-010 | The platform MUST run one canonical ticker that fires every minute (`* * * * *`). Failover-safe: at most one ticker active platform-wide via durable lock. |
| 027-FR-011 | On each tick, the scheduler MUST query schedules whose next fire-time falls within the previous minute and dispatch each as a durable task (spec 004). |
| 027-FR-012 | Dispatch MUST be idempotent: the schedule's `(schedule_id, scheduled_for)` MUST be a unique key on the dispatch ledger to prevent double-dispatch under failover. |
| 027-FR-013 | `catch_up: false` means a missed minute is skipped (only fire if scheduled_for is "now"). `catch_up: true` means one catch-up task is dispatched per missed period (capped at the most recent N=10 missed periods to avoid storms). |
| 027-FR-014 | `max_concurrent` MUST be enforced: dispatching a schedule with `max_concurrent: 1` while an existing instance is still running results in skip (audit-logged with reason). |
| 027-FR-015 | Dispatched tasks MUST be billed per spec 007 against the agent's (or platform's) configured budget. Over-budget skips are audit-logged. |
| 027-FR-016 | Schedule failures MUST capture cause per spec 006/020. `on_failure: escalate` raises an approval-style ticket to operator. |

### Observability

| ID | Requirement |
|---|---|
| 027-FR-020 | Dashboard MUST render: per-agent schedule list with next-fire-time, recent invocations + outcomes; platform schedule list; missed/skipped invocations. |
| 027-FR-021 | Audit MUST log: schedule registered (on merge), dispatched, skipped (with reason), failed (with cause). |
| 027-FR-022 | Operator MUST be able to pause/resume individual schedules without editing the git repo (audit-logged; resume restores normal cadence). Pause persists across restarts via the registry. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 027-NFR-001 | Ticker overhead < 50ms per tick on ≤1000 active schedules. |
| 027-NFR-002 | Schedule registry MUST scale to ≥10k active schedules. |
| 027-NFR-003 | Drift: a schedule's actual dispatch time MUST be within +/- 30s of its scheduled time on a healthy platform. |

## User stories

- **027-US-001 [P0]** — As an operator, I add a `schedules.yaml` to my coder agent declaring nightly self-reflection; on merge, it shows up in the dashboard as "next fire-time: 03:00 UTC".
- **027-US-002 [P0]** — As an operator, a schedule fails repeatedly with the same cause; the dashboard alerts me.
- **027-US-003 [P0]** — As an operator, I pause a schedule mid-week without touching git; it resumes when I unpause.
- **027-US-004 [P1]** — As an operator, my project enters `post-launch-monitoring` and a `*/15 *` health-check schedule begins firing; ends when the project is archived.
- **027-US-005 [P1]** — As an operator, the platform was down 6 hours; on recovery, `catch_up: true` schedules fire one catch-up; `catch_up: false` ones skip and log.
- **027-US-006 [P1]** — As an operator, I roll back the coder agent to last Tuesday and its `schedules.yaml` rolls with it; the registry updates automatically.

## Acceptance criteria

- Per-agent schedules round-trip end-to-end (declare → merge → register → dispatch).
- Project-bound schedules stop with the project.
- Failover under load: kill the ticker; another node picks it up; no double-dispatches.
- Catch-up and skip semantics work as documented.

## Edge cases & failure modes

- **Two nodes both think they're the ticker** — durable lock + idempotency key prevents duplicate dispatches; second node loses the lock cleanly.
- **Cron expression invalid** — validation (spec 018) refuses merge.
- **`when_project_in_stage` fires while the project is paused** — scheduler treats paused as "not in stage"; skips and logs.
- **Schedule budget exceeded for the day** — dispatch refused with `cause=budget_exceeded`; logged; not retried until the next budget period.
- **Ticker missed a full minute (slow query, GC pause)** — next tick scans the missed window; honors per-schedule catch_up semantics.

## Open questions

- Should `catch_up: true` support a configurable catch-up cap per schedule (default 10), or is one global cap fine? (Lean: per-schedule, default 10.)
- Allow human-triggered "fire now" via dashboard? (Lean: yes, audit-logged as `manual_trigger`.)

## Related specs

`001-monorepo-git-config`, `004-control-plane-durable-state`, `006-observability-audit-cost`, `007-budget-governance`, `008-agent-definition`, `009-agent-self-modification-pr`, `016-software-agency-pipeline`, `018-config-validation-pre-merge`, `019-agent-benchmarks-evals`, `020-stability-recovery`, `028-backup-disaster-recovery`.
