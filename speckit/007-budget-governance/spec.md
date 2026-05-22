# 007 — Budget & Cost Governance

**Status:** Draft
**Constitutional anchor:** Principle VI (Observable), IX (HITL)
**Depends on:** 006
**Required by:** 016, 017

## Purpose

Hard budget caps at every scope. Soft alerts before caps. Automatic abort at hard cap, with state preserved. Inspired by Paperclip's budget governance.

## Functional requirements

| ID | Requirement |
|---|---|
| 007-FR-001 | Budgets MUST be configurable at: platform, team, project, agent, single-task scope. |
| 007-FR-002 | A budget MUST specify: amount (USD), period (one-shot / per-day / per-week / per-month), soft-alert thresholds (defaults 50%, 75%, 90%), hard-cap behavior (`abort` / `pause_for_approval`). |
| 007-FR-003 | The control plane MUST evaluate budget before each LLM call against current spend; on hard cap, reject the call with `cause=budget_exceeded`. |
| 007-FR-004 | Soft alerts MUST emit to the operator (dashboard + configured channels). |
| 007-FR-005 | `pause_for_approval` MUST suspend the task and create an approval to extend or abort. |
| 007-FR-006 | Budgets MUST cascade: a per-task abort frees its budget; an aborted project releases its remaining budget. |
| 007-FR-007 | An operator MUST be able to temporarily override a budget (with audit log entry and reason). |
| 007-FR-008 | A "budget burndown" view MUST be available per scope. |
| 007-FR-009 | Free-quota providers (local models, free-tier APIs) MUST be modeled with $0 cost but rate-limit awareness. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 007-NFR-001 | Budget check overhead < 5ms per LLM call. |
| 007-NFR-002 | Budget update must converge within 10s of underlying ledger update. |

## User stories

- **007-US-001 [P0]** — As an operator, I set $20/day per project; a runaway task gets aborted at the cap.
- **007-US-002 [P0]** — As an operator, I see live burndown for each project.
- **007-US-003 [P1]** — As an operator, an approaching cap triggers `pause_for_approval` so I can extend rather than restart.
- **007-US-004 [P1]** — As an operator, an override is audited with my id and reason.

## Acceptance criteria

- Hard cap aborts task cleanly with state preserved and cause logged.
- Soft alerts deliver to dashboard within seconds.
- Cascading aborts release reserved budget.
- Override paths require audit entry.

## Edge cases & failure modes

- Concurrent tasks racing past cap — control plane reserves cost per call; collisions resolved by deny-on-overshoot.
- Provider misreports cost (negative or huge) — clamp + flag for review.
- Budget set to $0 → all paid calls denied; free-tier still works.

## Related specs

`006-observability-audit-cost`, `005-approvals-hitl`, `016-software-agency-pipeline`, `017-multi-project-juggling`.
