# 005 — Approvals & Human-in-the-Loop

**Status:** Draft
**Constitutional anchor:** Principle IX (HITL), IV (Default-deny)
**Depends on:** 003, 004
**Required by:** 009, 015, 016, 022

## Purpose

Approvals are how humans steer autonomy. Every unrecognized or sensitive action surfaces in an approval queue. The operator can approve, deny, optionally generate a reusable rule, set decision windows, and pre-authorize patterns.

## Why this matters

Research is unanimous: durable, visible, audited approval flow is the dividing line between "interesting prototype" and "system you'd let touch your codebase". Viktor, Paperclip, Claude Code permissions all converge on this.

## Functional requirements

| ID | Requirement |
|---|---|
| 005-FR-001 | An approval MUST be a first-class durable record tied to a task. |
| 005-FR-002 | Approvals MUST surface via dashboard within 1s of creation; SHOULD also push to configured channels (email, Slack, Telegram, custom webhook). |
| 005-FR-003 | Approval payload MUST include: requesting agent, action, resource, reason from the agent, context summary, suggested decisions. |
| 005-FR-004 | Decision actions: `approve`, `approve_with_rule`, `deny`, `deny_with_rule`, `request_more_info`. |
| 005-FR-005 | Decision window MUST be configurable per team/scope. Per-class defaults (matching spec 003): `sensitive` = 24h, `normal` = 4h, `routine` = 30 min. On expiry, the approval times out to deny with cause `approval_timeout`. |
| 005-FR-006 | `approve_with_rule` / `deny_with_rule` MUST produce a policy rule committed to the relevant repo (PR auto-created for repo merge if remote configured). |
| 005-FR-007 | Approval list views MUST support filter by team, project, agent, urgency, age. |
| 005-FR-008 | Bulk actions (approve N similar, deny N) MUST be supported with confirmation. |
| 005-FR-009 | Pre-authorized patterns: operator MAY pre-approve a class of actions in a team's policy file ahead of time. |
| 005-FR-010 | Pause / resume / abort: every running task MUST have an operator-callable pause, resume, and abort. Pause checkpoints and suspends; resume restarts from checkpoint; abort terminates and cleans up. |
| 005-FR-011 | An approval MUST capture the operator id (v1: constant `"root"` per spec 023's single-operator model; future-proof for multi-operator/RBAC in v2) and an optional free-text reason; reason MUST be logged to audit. |
| 005-FR-012 | Auto-rule generation: when `approve_with_rule` is chosen, dashboard MUST let the operator narrow the rule scope (exact match → pattern). |
| 005-FR-013 | A "kill switch" MUST allow an operator to pause an entire team, an entire project, or the entire platform with one action. Audit logged. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 005-NFR-001 | Approval surface latency p95 < 1s (queue → visible in dashboard). |
| 005-NFR-002 | Notification channel delivery: best-effort with retry; failure must not block in-platform approval. |
| 005-NFR-003 | Operator decision latency target: median < 1 minute for high-urgency, < 1 hour for normal. (UX goal; system enforces only the configured window.) |

## User stories

- **005-US-001 [P0]** — As an operator, I see an approval request within 1s of an agent asking. I approve and the agent continues.
- **005-US-002 [P0]** — As an operator, I approve with "auto-allow this pattern" and the rule is committed to the repo for review.
- **005-US-003 [P0]** — As an operator, I pause an entire project mid-run; on resume, work continues without duplicate effects.
- **005-US-004 [P1]** — As an operator, I configure Slack to receive high-urgency approvals; they show up in #clawie-ops.
- **005-US-005 [P1]** — As an operator, I configure a 1-hour decision window for sensitive ops in production-grade teams.
- **005-US-006 [P2]** — As an operator, I batch-approve 12 similar approvals in one click after reading the pattern summary.

## Acceptance criteria

- Approval request from container reaches dashboard in <1s.
- Approve-with-rule produces a committed rule and an audit entry with operator id.
- Timeout-on-expiry produces deny with `cause=approval_timeout`.
- Pause/abort cleanly terminate without losing audit trail.

## Edge cases & failure modes

- Notification channel returns 500 → retry with backoff; approval still visible in dashboard.
- Operator approves after timeout → operation rejected, error explains timeout already fired.
- Race: two operators approve same request simultaneously → first wins, second sees clear "already decided" status.
- Approval channel exploit (someone replays an old approval URL) → approvals signed/scoped per request; replay rejected.

## Related specs

`003-policy-permissions`, `004-control-plane-durable-state`, `009-agent-self-modification-pr`, `022-web-dashboard`.
