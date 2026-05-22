# 006 — Observability, Audit, Cost Ledger

**Status:** Draft
**Constitutional anchor:** Principle VI (Observable by default)
**Depends on:** 004
**Required by:** 007, 019, 022, 023

## Purpose

If you can't see it, you can't run it. Clawie produces a unified, queryable record of everything: state transitions, tool calls, model calls, approvals, config changes, container lifecycle, egress decisions, errors, and costs. Three views: traces (per-task), audit (immutable history), cost ledger (FinOps).

## Why this matters

Research consensus: the difference between toy frameworks and production frameworks is observability. Paperclip's audit, LangSmith's traces, Claude Code's cost ledger all converge. You can't be autonomous without being legible.

## Functional requirements

### Audit log

| ID | Requirement |
|---|---|
| 006-FR-001 | Audit MUST be append-only, tamper-evident (hash chain or signed entries). |
| 006-FR-002 | Every state transition (task, approval, config-change, container-lifecycle, policy-decision, egress-decision) MUST produce an audit event. |
| 006-FR-003 | Audit events MUST capture: timestamp, actor (agent/operator/system), action verb, subject, outcome, optional reason, optional related entity ids. |
| 006-FR-004 | Audit MUST be queryable by entity (task, agent, project), time range, action type. |
| 006-FR-005 | Retention is configurable; default 1 year for production, infinite for dev. |
| 006-FR-006 | Export to S3/cold storage supported as a periodic job. |

### Traces

| ID | Requirement |
|---|---|
| 006-FR-010 | Every task MUST produce a structured trace: tree of spans for runtime, tool calls, model calls, child tasks. |
| 006-FR-011 | Traces MUST follow OpenTelemetry semantics; exporter pluggable (Jaeger, Tempo, Honeycomb, Datadog, OTLP). |
| 006-FR-012 | Each span MUST carry: name, start/end, latency, status, error, attributes (model, prompt size, token counts). |
| 006-FR-013 | Traces MUST link to audit events so an operator can pivot from a trace to its audit context. |

### Cost ledger

| ID | Requirement |
|---|---|
| 006-FR-020 | Every model call MUST emit a cost entry: (provider, model, in-tokens, out-tokens, USD cost, task id, **source**, **schedule_id** if applicable). |
| 006-FR-021 | Every container minute MUST emit a cost entry with configurable rate per host (defaults to 0 for self-hosted; operator can attribute a host cost). Includes script-cron containers, not just agent ones. |
| 006-FR-022 | Every external API call (via connector) MAY emit a cost entry if the connector declares pricing. |
| 006-FR-023 | The ledger MUST roll up: per-agent, per-team, per-project, per-day, **per-source** (`brief`/`cron`/`eval`/`platform`/`adhoc`), **per-schedule** (when source = cron). |
| 006-FR-024 | Currency unit is USD; configurable display in dashboard. |
| 006-FR-025 | Every cost entry MUST carry a `source` discriminator. Defined values: `brief` (operator-initiated project task), `cron` (scheduled via spec 027 — `schedule_id` populated), `eval` (run via spec 019), `platform` (control-plane internal — reconcile, backup, etc.), `adhoc` (manual trigger). Source enables rollup queries like "how much did crons cost this month?" or "which schedule is the most expensive?". |

### Errors & cause-of-failure

| ID | Requirement |
|---|---|
| 006-FR-030 | Every failure MUST carry a structured `cause` enum: `oom`, `timeout`, `policy_denied`, `approval_timeout`, `tool_error`, `provider_rate_limit`, `provider_error`, `infrastructure_unavailable`, `deadlock`, `validation_failed`, `unknown`. |
| 006-FR-031 | `unknown` is a code smell; the platform MUST expose count over time to drive triage. |
| 006-FR-032 | Each cause MUST carry a "what to try next" pointer in the docs. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 006-NFR-001 | Audit write MUST NOT block task progress; async fanout with persistence guarantees. |
| 006-NFR-002 | Trace overhead < 5% of task latency. |
| 006-NFR-003 | Cost calculation MUST update within 10s of the underlying event. |

## User stories

- **006-US-001 [P0]** — As an operator, I open a task in dashboard and see its full span tree + every related audit event.
- **006-US-002 [P0]** — As an operator, I query "every config change by agent X this week" and get a clean list.
- **006-US-003 [P0]** — As an operator, I see live cost per project; clicking drills into per-LLM-call breakdown.
- **006-US-004 [P1]** — As an operator, I export the audit log to S3 for compliance.
- **006-US-005 [P1]** — As an operator, I plug Honeycomb in as the trace backend via config.
- **006-US-006 [P1]** — As an operator, I see "unknown cause" trend going up and dig in.

## Acceptance criteria

- Every action across surfaces produces audit + trace.
- Cost dashboard updates within seconds of an LLM call completing.
- Pivot from trace → audit → cost is one click.
- Tamper-evidence verifiable via CLI: `clawie audit verify`.

## Edge cases & failure modes

- Audit storage full → platform refuses to start new tasks; alerts; existing tasks finish.
- Trace exporter unreachable → buffer with bounded queue; drop oldest with a clear metric.
- Provider doesn't report token counts → estimate, mark estimate flag in ledger.
- Clock drift between hosts → audit uses monotonic id + wall clock; sort stable.

## Related specs

`004-control-plane-durable-state`, `007-budget-governance`, `019-agent-benchmarks-evals`, `022-web-dashboard`, `023-rest-api`.
