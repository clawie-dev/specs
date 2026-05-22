# 027 — Scheduler & Recurring Work

**Status:** Draft
**Constitutional anchor:** Principle II (Git source of truth), V (Durable state), VI (Observable)
**Depends on:** 001, 004, 008
**Required by:** 016, 019, 020

## Purpose

Some work is not initiated by an operator brief — it recurs. Nightly evals (spec 019), container reconcile (spec 020), post-launch monitoring (spec 016), agent self-reflection cycles, hourly domain-expiry checks, etc. Clawie ships a scheduler that:

1. Lets **each agent declare its own cron schedules in its own git repo**, so schedule changes are reviewable via the same PR flow as any other agent change (spec 009).
2. Supports **two schedule kinds**:
   - **`agent`** — invokes the agent in the normal task loop. Costs LLM tokens. Use when the work benefits from the agent's judgment.
   - **`script`** — runs a deterministic script (no LLM client available). **Costs ~$0 in LLM tokens**, only container minutes. The script's structured output is treated as the task result and can directly trigger notifications/messages. Use for cheap recurring checks (RSS polling, deploy-health pings, "any new expired domain on this list?"). On failure, optionally escalates to an agent for diagnosis.
3. Runs a single core ticker (`* * * * *`, every minute) that consults the registry and dispatches due work as durable tasks (spec 004).
4. Treats every scheduled invocation as a first-class task — idempotent, recoverable, observable, budget-bound, **cost-attributed per cron** (tokens + container time + external API costs).

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
  # ── AGENT schedules: invoke the agent loop, cost LLM tokens ──────────────────
  - id: nightly-self-reflect
    kind: agent                    # default; may be omitted
    cron: "0 3 * * *"             # 03:00 UTC daily
    intent: self-reflect
    description: Daily reflection on last 24h work; may produce a self-mod PR.
    timezone: UTC                   # only UTC supported in v1
    on_failure: log                 # log | escalate
    catch_up: false                 # if platform was down, run missed? default false
    max_concurrent: 1               # never run two instances of this schedule at once
    budget_usd: 0.50                # hard cap; over-budget aborts with cause

  - id: eval-suite
    kind: agent
    cron: "30 4 * * *"
    intent: run-eval-suite
    description: Run my own benchmark fixtures, report drift.
    catch_up: true                   # missed runs trigger one catch-up at next tick

  - id: post-launch-health-check
    kind: agent
    cron: "*/15 * * * *"
    intent: project-health-check
    description: Check deployed prod service; raise ticket on drift.
    bound_to: project
    when_project_in_stage:
      - post-launch-monitoring

  # ── SCRIPT schedules: no LLM, deterministic, ~$0 in tokens ──────────────────
  - id: expired-domains-check
    kind: script
    cron: "0 * * * *"               # hourly
    script: scripts/check-expired-domains.ts
    description: |
      Polls domain-expiry source, formats a notification on new finds.
      Zero LLM cost. Agent only invoked on script failure.
    timeout_seconds: 60
    on_success:
      # Script returns structured JSON: { action, payload }
      # Action mapped to platform side-effect — no agent involved.
      route_actions: true
      allowed_actions:
        - notify_operator          # sends to dashboard + configured channels
        - open_ticket              # creates an internal ticket (spec 014)
        - external_transition      # transitions a Linear/Jira item via spec 026
    on_failure:
      escalate_to: researcher       # spawn an agent task with the debug bundle
      prompt_template: |
        The recurring script `{schedule_id}` failed at {when} with exit code {exit_code}.
        Last stdout (truncated to 2000 chars):
        {stdout}
        Last stderr:
        {stderr}
        Investigate and either propose a fix to the script or escalate to operator.

  - id: deploy-health-ping
    kind: script
    cron: "*/5 * * * *"
    script: scripts/ping-prod.sh
    description: Curl prod healthcheck; alert on non-200. Pure check, no LLM.
    timeout_seconds: 30
    on_success:
      route_actions: true
      allowed_actions: [notify_operator]
    on_failure:
      escalate_to: devops
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
| 027-FR-002 | Each schedule MUST declare: `id` (unique per agent), `kind` (`agent` or `script`; defaults to `agent`), `cron` (standard 5-field UTC), `description`. **Kind-specific required fields:** `agent` requires `intent`; `script` requires `script` (path relative to the agent repo). |
| 027-FR-003 | Cron expressions MUST be in UTC. v1 does NOT support arbitrary timezones (UTC-only); local-time scheduling deferred. |
| 027-FR-004 | Optional schedule fields (all kinds): `on_failure` (`log`/`escalate`/escalate-config, default `log`), `catch_up` (bool, default false), `max_concurrent` (int, default 1), `budget_usd` (per-invocation budget), `bound_to` (project-scoped), `when_project_in_stage` (only fire while project is in listed stages), `timeout_seconds` (int, default 300). |
| 027-FR-005 | Schedule changes MUST go through the agent's standard PR review flow (spec 009). A merged change updates the registry within 10s of the merge being detected. |
| 027-FR-006 | Project-bound schedules MUST be associated with a project and MUST stop firing when that project is aborted, completed, or paused. |
| 027-FR-007 | Platform-internal schedules MUST live in `clawie.yaml` under `platform_schedules:`, validated by spec 018, and reviewable as PRs to the root config repo. |

### Script schedules (kind: script)

| ID | Requirement |
|---|---|
| 027-FR-008 | Script schedules MUST run the declared script inside a container based on the same agent-runtime image as agent tasks (same security posture, Outcall egress, credential broker), with **no LLM client available** (provider routing layer blocked at the runtime). Tampering attempts to invoke an LLM from a script MUST raise a `script_llm_attempt` cause and abort. |
| 027-FR-009 | The script's stdout (last N=2000 chars) MUST be captured as structured JSON when the script declares `route_actions: true`. Schema: `{action: <string>, payload: <object>}` or `[{action, payload}, ...]`. Unparseable output on a route-actions schedule = failure with cause `script_output_invalid`. |
| 027-FR-010 | `allowed_actions` restricts which platform side-effects the script's JSON output may trigger. Default empty (no actions). Built-in actions: `notify_operator`, `open_ticket`, `external_transition`, `set_metric`, `attach_artifact`. Actions outside `allowed_actions` are silently ignored and audit-logged. |
| 027-FR-011 | On script failure (non-zero exit, timeout, OOM, output schema mismatch), the scheduler MUST capture a **debug bundle**: exit code, stdout (≤8 kB), stderr (≤8 kB), wall time, peak memory, env snapshot (no secret values — broker redacts). The bundle MUST be attached to the task's audit entry. |
| 027-FR-012 | `on_failure.escalate_to` (script schedules only) MUST dispatch an `agent`-kind task to the named agent, with the debug bundle interpolated into `prompt_template`. This is the only path that converts a $0 script failure into an LLM-costed agent task — and it's explicit, opt-in per schedule. |
| 027-FR-013 | Script schedules MUST NOT exceed `timeout_seconds`. Default 300s; max 1800s (30 min). Exceeding triggers `script_timeout` cause + escalation per config. |
| 027-FR-014 | A script schedule's container MUST be reaped immediately on script exit; no lingering state. |

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

### Observability + cost attribution

| ID | Requirement |
|---|---|
| 027-FR-020 | Dashboard MUST render: per-agent schedule list with next-fire-time, recent invocations + outcomes; platform schedule list; missed/skipped invocations. Per-schedule rollup view: invocations/period, success rate, p95 duration, **token cost (rollup)**, **container-minute cost**, **external-API cost**. |
| 027-FR-021 | Audit MUST log: schedule registered (on merge), dispatched, skipped (with reason), failed (with cause). |
| 027-FR-022 | Operator MUST be able to pause/resume individual schedules without editing the git repo (audit-logged; resume restores normal cadence). Pause persists across restarts via the registry. |
| 027-FR-023 | Operator MAY trigger a schedule manually from the dashboard or CLI; manual trigger is audit-logged with cause `manual_trigger`. Same authorization model (must own the agent or be platform admin). |
| 027-FR-024 | Every scheduled invocation MUST emit a cost ledger entry tagged with `source: cron` and `schedule_id`, so the cost dashboard can roll up "money spent by crons" per agent, team, project, and globally. (Cross-references and complements spec 006/007 cost ledger.) |
| 027-FR-025 | The dashboard MUST highlight crons that are persistently failing (e.g., 5+ consecutive failures) with the cause and the most recent debug bundle. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 027-NFR-001 | Ticker overhead < 50ms per tick on ≤1000 active schedules. |
| 027-NFR-002 | Schedule registry MUST scale to ≥10k active schedules. |
| 027-NFR-003 | Drift: a schedule's actual dispatch time MUST be within +/- 30s of its scheduled time on a healthy platform. |

## User stories

- **027-US-001 [P0]** — As an operator, I add a `schedules.yaml` to my coder agent declaring nightly self-reflection; on merge, it shows up in the dashboard as "next fire-time: 03:00 UTC".
- **027-US-002 [P0]** — As an operator, a schedule fails repeatedly with the same cause; the dashboard alerts me with the debug bundle for script schedules.
- **027-US-003 [P0]** — As an operator, I pause a schedule mid-week without touching git; it resumes when I unpause.
- **027-US-004 [P1]** — As an operator, my project enters `post-launch-monitoring` and a `*/15 *` health-check schedule begins firing; ends when the project is archived.
- **027-US-005 [P1]** — As an operator, the platform was down 6 hours; on recovery, `catch_up: true` schedules fire one catch-up; `catch_up: false` ones skip and log.
- **027-US-006 [P1]** — As an operator, I roll back the coder agent to last Tuesday and its `schedules.yaml` rolls with it; the registry updates automatically.
- **027-US-007 [P0]** — As an operator, I want an hourly check for expired domains; I write a script that costs ~0 tokens, attached to a `kind: script` schedule. It runs hourly, sends me a notification if it finds matches, and only escalates to an agent if the script itself fails. Over a month: pennies in compute, zero LLM cost.
- **027-US-008 [P0]** — As an operator, the cost dashboard shows "crons spent $0.40 this month" and breaks down per schedule — I can see that `nightly-self-reflect` cost $0.32 (mostly tokens) while `expired-domains-check` cost <$0.01 (just container minutes).
- **027-US-009 [P1]** — As an operator, a script schedule fails 5 times in a row; the dashboard shows the debug bundle (exit code, stderr) and offers a one-click "ask the researcher agent to investigate" button that uses the configured escalation template.
- **027-US-010 [P1]** — As an operator, my script tries to invoke the LLM (a bug); the runtime blocks the attempt, logs `script_llm_attempt`, and fails the invocation cleanly without spending tokens.

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

## Decision log

- **Catch-up cap** (resolved 2026-05-22): per-schedule, default 10 missed periods. Configurable per schedule.
- **Manual fire-now trigger** (resolved 2026-05-22): supported via dashboard + `clawie schedule fire <agent>/<id>`. Audit-logged with cause `manual_trigger`. See 027-FR-023.
- **Script-kind schedules** (resolved 2026-05-22): introduced as a first-class schedule kind alongside agent-kind. Zero LLM cost when the work is deterministic; escalation path to an agent on failure makes them safe even when the script becomes brittle.

## Related specs

`001-monorepo-git-config`, `004-control-plane-durable-state`, `006-observability-audit-cost`, `007-budget-governance`, `008-agent-definition`, `009-agent-self-modification-pr`, `016-software-agency-pipeline`, `018-config-validation-pre-merge`, `019-agent-benchmarks-evals`, `020-stability-recovery`, `028-backup-disaster-recovery`.
