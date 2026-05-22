# 023 — REST + WebSocket API

**Status:** Draft
**Constitutional anchor:** Principle I (Surfaces), X (Pluggable)
**Depends on:** 003, 004
**Required by:** 021, 022

## Purpose

A single API is the only writer to the control plane. CLI and Dashboard both consume it. Third parties can automate against it. OpenAPI documented. Versioned.

## Resource model (high level)

```
/v1/projects                CRUD
/v1/projects/:id/tasks
/v1/projects/:id/artifacts
/v1/projects/:id/pipeline
/v1/projects/:id/pause       (POST)
/v1/projects/:id/resume      (POST)
/v1/projects/:id/abort       (POST)

/v1/teams                    CRUD
/v1/teams/:id/agents
/v1/teams/:id/flows
/v1/teams/:id/task-management   (spec 026)

/v1/agents/:id
/v1/agents/:id/runs
/v1/agents/:id/self-mods      (PRs)
/v1/agents/:id/benchmarks
/v1/agents/:id/rollback       (POST {to: ref})

/v1/tasks/:id                read; transition; abort
/v1/tasks/:id/logs           streamable
/v1/tasks/:id/trace
/v1/tasks/:id/checkpoints

/v1/approvals                list/decide/bulk-decide
/v1/policies                 CRUD
/v1/secrets                  add/rotate/revoke/scope/list (no value retrieval)
/v1/skills                   list/install/link
/v1/plugins                  list/install/verify

/v1/audit                    query/export/verify
/v1/cost                     query/budgets

/v1/eval/:agent              run/results/trends

/v1/health                   liveness/readiness/diagnostic
/v1/outcall                  status/rules/test
/v1/external-tasks           list/transition (driver-mediated, spec 026)

/v1/events                   WebSocket: server-pushed events (task transitions, approvals, costs, audit)
```

## Functional requirements

| ID | Requirement |
|---|---|
| 023-FR-001 | API MUST be OpenAPI-described; spec auto-generated and published. |
| 023-FR-002 | Auth via single operator token (Bearer) in v1. v1 supports exactly one operator; every API write is attributed to constant `operator_id = "root"` in audit. Multi-operator + RBAC is a v2 spec, not delivered in v1. OAuth/SSO planned with v2. |
| 023-FR-003 | All write operations MUST be idempotent via `Idempotency-Key` header. |
| 023-FR-004 | Versioning via URL path (`/v1/...`); breaking changes increment major. |
| 023-FR-005 | Pagination cursor-based; default page size 50, max 500. |
| 023-FR-006 | Filter parameters MUST be consistent across resources (`status`, `project`, `team`, `agent`, `since`, `until`). |
| 023-FR-007 | Errors MUST follow a structured shape: `{error: {code, message, details, related_audit_id?}}`. |
| 023-FR-008 | WebSocket `/v1/events` MUST support topic subscription (per-project, per-team, per-task, all). |
| 023-FR-009 | Rate limits: per token, per resource class. Returns 429 with Retry-After. |
| 023-FR-010 | Audit endpoint MUST allow tamper-verification (`/v1/audit/verify` returns chain status). |
| 023-FR-011 | Secret values MUST NEVER be returned via API. Aliases and metadata only. |
| 023-FR-012 | All responses MUST include a `request_id` for cross-referencing with traces. |
| 023-FR-013 | Webhooks (outbound) for: approvals created, projects-state-changed, cost-threshold-crossed. Operator-configurable per integration. |
| 023-FR-014 | The API MUST never bypass policy/permission engine for state-changing operations originating from non-operator subjects. (Operators can be granted admin-bypass scopes; agents cannot.) |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 023-NFR-001 | Common read p99 < 200ms. |
| 023-NFR-002 | WebSocket event delivery < 500ms p95 from internal event publication. |
| 023-NFR-003 | OpenAPI spec MUST validate via standard tooling on every PR. |

## User stories

- **023-US-001 [P0]** — As an operator, I script daily reports via the API.
- **023-US-002 [P0]** — As an operator, I subscribe to WebSocket events filtered to one project for a custom dashboard.
- **023-US-003 [P1]** — As an operator, I configure an outbound webhook to Slack on every approval.
- **023-US-004 [P1]** — As an integrator, I generate a typed client from the OpenAPI spec.

## Acceptance criteria

- All endpoints documented + validated against OpenAPI.
- Idempotency-Key honored.
- WebSocket events delivered with low latency.
- Audit verify endpoint returns chain status.
- Secret values never appear in any response body.

## Edge cases & failure modes

- Concurrent updates → optimistic locking; 409 conflict with hint.
- WebSocket disconnect → client reconnect with last-event-id for replay.
- API outage → CLI degrades to local-only diagnostic; cached state read-only.
- Token compromise → revocation immediate; refresh required.

## Related specs

`021-cli`, `022-web-dashboard`, `004-control-plane-durable-state`, `006-observability-audit-cost`, all other functional specs.
