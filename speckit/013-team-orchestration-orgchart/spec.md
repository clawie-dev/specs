# 013 — Team Orchestration & Org Chart

**Status:** Draft
**Constitutional anchor:** Principle XII (End-to-end agency)
**Depends on:** 001, 003, 004, 008
**Required by:** 014, 016, 017

## Purpose

A team is a versioned collection of agents with declared roles, an org chart, and message-flow rules. Teams are the unit of "an agency" — installable as a starter pack, customizable, observable as a whole.

## Why this matters

Paperclip's "OpenClaw is a worker, Paperclip is the company" framing crystallizes the value. Clawie ships this as a first-class abstraction.

## Team anatomy

A team repo contains:

```
<team-name>/
├── team.yaml             # name, description, charter
├── flows.yaml            # who-talks-to-whom; pipeline stage assignments
├── budgets.yaml          # team-level + per-agent budgets
├── policies/             # team-wide policy rules
├── task-management.yaml  # OPTIONAL: external task driver + status-flow rules (spec 026)
├── agents/               # submodules per agent
└── README.md
```

When `task-management.yaml` is present, the external driver-enforced flow rules (spec 026) layer on top of `flows.yaml`. `flows.yaml` describes Clawie-internal pipeline; `task-management.yaml` describes how that pipeline maps onto an external system like Linear/Jira and which agent roles may transition which statuses. Validator (spec 018) enforces consistency between the two.

### `team.yaml`

```yaml
name: default-agency
charter: "End-to-end software agency: research → spec → code → review → deploy → market"
roles:
  - { id: researcher, agent: researcher }
  - { id: architect, agent: architect }
  - { id: coder, agent: coder }
  - { id: reviewer, agent: reviewer }
  - { id: qa, agent: qa }
  - { id: security, agent: security }
  - { id: perf, agent: perf }
  - { id: doc_writer, agent: doc-writer }
  - { id: devops, agent: devops }
  - { id: marketer, agent: marketer }
manager:
  agent: manager
  responsibilities: triage, schedule, escalate
```

### `flows.yaml`

```yaml
# Stage names below MUST match the canonical state machine in spec 016.
pipeline_stages:
  - { stage: research,              owner: researcher, on_complete: spec }
  - { stage: spec,                  owner: architect,  on_complete: architecture, approval_required: true }
  - { stage: architecture,          owner: architect,  on_complete: scaffold }
  - { stage: scaffold,              owner: coder,      on_complete: implement }
  - { stage: implement,             owner: coder,      on_complete: code-review }
  - { stage: code-review,           owner: reviewer,   on_change: implement, on_pass: security-review }
  - { stage: security-review,       owner: security,   on_fail: implement,  on_pass: performance-review }
  - { stage: performance-review,    owner: perf,       on_fail: implement,  on_pass: documentation }
  - { stage: documentation,         owner: doc_writer, on_complete: devops-setup, approval_required: true }
  - { stage: devops-setup,          owner: devops,     on_complete: deploy-staging }
  - { stage: deploy-staging,        owner: devops,     on_complete: e2e-test }
  - { stage: e2e-test,              owner: qa,         on_fail: implement,  on_pass: deploy-production }
  - { stage: deploy-production,     owner: devops,     on_complete: marketing-site, approval_required: true }
  - { stage: marketing-site,        owner: marketer,   on_complete: launch-announcement }
  - { stage: launch-announcement,   owner: marketer,   on_complete: post-launch-monitoring, approval_required: true }
  - { stage: post-launch-monitoring, owner: devops,    recurring: true }   # scheduled via spec 027
messaging:
  - { from: coder,    to: [reviewer, architect], topic: implementation-questions }
  - { from: reviewer, to: coder,                 topic: feedback }
  - { from: '*',      to: manager,               topic: blockers }
```

## Functional requirements

| ID | Requirement |
|---|---|
| 013-FR-001 | A team MUST have at least one declared role. |
| 013-FR-002 | Each role MUST map to an agent submodule. |
| 013-FR-003 | Flows MUST be a directed graph; cycles allowed only with explicit `cycle_safe: true` + max-iterations cap. |
| 013-FR-004 | The control plane MUST drive pipeline transitions based on flows.yaml. |
| 013-FR-005 | Messaging rules MUST be enforced: cross-team or cross-role messages outside declared rules are denied + audited. |
| 013-FR-006 | A team MUST be installable from the marketplace as a starter pack (signed). |
| 013-FR-007 | Operators MAY fork starter packs into their own team repo. |
| 013-FR-008 | A team's org chart MUST render in the dashboard. |
| 013-FR-009 | A team-wide pause MUST stop all member agents at their next safe checkpoint. |
| 013-FR-010 | Team config changes go through PR review like agent changes. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 013-NFR-001 | Flow evaluation per transition < 50ms. |
| 013-NFR-002 | Up to 20 agents per team supported in v1. |

## User stories

- **013-US-001 [P0]** — As an operator, I install the `default-agency` starter pack and have a working 9-agent team in one command.
- **013-US-002 [P0]** — As an operator, I see the org chart and active messages on the dashboard.
- **013-US-003 [P1]** — As an operator, I fork the starter pack into `my-devops-team` and edit roles.
- **013-US-004 [P1]** — As an operator, a forbidden cross-role message is denied and audited.

## Acceptance criteria

- Team install one-shot.
- Org chart rendered, messages rule-enforced.
- Pipeline transitions driven by `flows.yaml`.
- Team-level pause / abort works.

## Edge cases & failure modes

- Agent referenced in role but submodule missing → validation fails.
- Cycle without `cycle_safe` → validation fails.
- Two roles claim same pipeline stage → validation fails.
- Manager agent crashes → other agents pause for next-claim; recovery per spec 020.

## Related specs

`001-monorepo-git-config`, `008-agent-definition`, `014-inter-agent-comms-tickets`, `016-software-agency-pipeline`, `017-multi-project-juggling`, `026-task-management-drivers-flow-rules`.
