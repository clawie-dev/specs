# 016 — End-to-End Software Agency Pipeline

**Status:** Draft (FLAGSHIP)
**Constitutional anchor:** Principle XII (End-to-end agency)
**Depends on:** 004, 005, 007, 008, 010, 011, 013, 014, 015
**Required by:** 017

## Purpose

The defining workflow. A single command — `clawie project new "Build me a server monitoring SaaS"` — kicks off a pipeline that takes an idea from brief to launched product, with humans approving the few decisions that matter.

This is the spec that turns Clawie from "agent framework" into "autonomous software agency".

## Pipeline state machine

```
brief
  ↓
research          ← researcher agent
  ↓
spec              ← architect agent
  ↓ (approval gate: spec sign-off)
architecture      ← architect agent (deeper)
  ↓ (approval gate: architecture sign-off, OPTIONAL)
scaffold          ← coder agent (repo + skeleton)
  ↓
implement         ← coder agent (iterates)
  ↓  ↑
  ↓  └── code-review ← reviewer agent (loops back on failure)
  ↓
security-review   ← security agent
  ↓ (loop to implement on fail)
performance-review ← perf agent
  ↓ (loop to implement on fail)
documentation     ← doc-writer agent
  ↓ (approval gate: ship-decision)
devops-setup     ← devops agent
  ↓
deploy-staging    ← devops agent
  ↓
e2e-test          ← reviewer agent (or qa role if defined)
  ↓ (loop on failure)
deploy-production ← devops agent
  ↓ (approval gate: launch-go)
marketing-site    ← marketer agent
  ↓
launch-announcement ← marketer agent
  ↓
post-launch-monitoring ← devops + reviewer (background)
  ↓
completed
```

## Functional requirements

| ID | Requirement |
|---|---|
| 016-FR-001 | A project MUST be initiated by a brief (CLI / API / dashboard). Brief = free-form text + optional structured fields (target platform, language, constraints, budget). |
| 016-FR-002 | The pipeline state machine MUST be the canonical orchestration for a project. Custom team `flows.yaml` MAY override stages. |
| 016-FR-003 | Each stage MUST emit canonical artifacts to `projects/<name>/artifacts/<stage>/`. |
| 016-FR-004 | Approval gates: spec sign-off (post-spec), ship-decision (post-docs), launch-go (pre-marketing). Configurable. |
| 016-FR-005 | The pipeline MUST be pausable, resumable, abortable at any stage. |
| 016-FR-006 | Inter-stage handoffs MUST happen via tickets (spec 014) — not in-process calls. |
| 016-FR-007 | Each stage MUST have a "definition of done" checklist that the responsible agent and reviewer agree on before transition. |
| 016-FR-008 | Loops (code → review → code) MUST be bounded by max-iteration (default 5) and a fallback to operator escalation. |
| 016-FR-009 | The project workspace (spec 015) MUST be the working dir for all code-touching stages. |
| 016-FR-010 | Budgets (spec 007) cascade from project → stages → tasks. |
| 016-FR-011 | A project MUST surface a single dashboard "project view" with stage progress, current approvals, cost, artifacts. |
| 016-FR-012 | Reference implementation: a `default-agency` team that fulfills every stage out of the box. |
| 016-FR-013 | Stages MAY be skipped explicitly in the brief (e.g., "skip marketing"). |
| 016-FR-014 | When a team has a task-management driver configured (spec 026), the pipeline stages MUST map to external statuses via `status_map`; pipeline transitions MUST go through the driver's flow rules. The pipeline cannot bypass the rules: a stage advance is just a transition attempt subject to ownership + gates. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 016-NFR-001 | A trivial sample brief MUST complete the pipeline in ≤2h wall-clock and ≤$5 cost on default settings. |
| 016-NFR-002 | Project view dashboard renders 100 active projects without lag. |

## Stage definitions of done (canonical)

| Stage | DoD |
|---|---|
| research | Market analysis doc, competitor list, target user list committed to `artifacts/research/`. |
| spec | Spec file with FRs/NFRs/users/use cases committed; auditor (reviewer agent) signs off. |
| architecture | Component diagram, data model, deployment topology committed. |
| scaffold | Repo created; CI green; smoke test passes. |
| implement | Acceptance tests pass; coverage ≥ team threshold; lint clean. |
| code-review | Reviewer's checklist passes; PR-style commentary committed. |
| security-review | OWASP-style review committed; no high/critical findings unresolved. |
| performance-review | Perf budget defined and measured; no regressions. |
| documentation | README, API docs, runbook committed. |
| devops-setup | CI/CD pipeline defined; staging environment provisioned (or stubbed). |
| deploy-staging | Service reachable in staging; smoke tests pass. |
| e2e-test | E2E suite passes against staging. |
| deploy-production | Service reachable in prod; smoke tests pass. |
| marketing-site | Landing page deployed; copy committed. |
| launch-announcement | Announcement copy committed; channels configured. |
| post-launch-monitoring | Monitoring dashboards defined; alerts configured. |

## User stories

- **016-US-001 [P0]** — As an operator, I run `clawie project new "Build me a server monitoring SaaS"` and walk away. I get 3 approval requests over the next hour and end up with a shipped product.
- **016-US-002 [P0]** — As an operator, I pause a project mid-pipeline; on resume, work continues without duplicate effects.
- **016-US-003 [P0]** — As an operator, I see the current stage, owner, cost, and pending approvals at a glance.
- **016-US-004 [P1]** — As an operator, I skip the marketing stage by adding `--skip marketing` to the new command.
- **016-US-005 [P1]** — As an operator, a code→review loop hits its iteration cap; the project pauses and escalates to me.

## Acceptance criteria

- A sample brief end-to-end succeeds with the `default-agency` team.
- Pause/resume/abort work at every stage boundary.
- Approval gates trigger correctly.
- Artifacts committed per stage.
- Cost stays within budget for the sample.

## Edge cases & failure modes

- Stage agent missing from the team → validator catches at install; runtime fails fast with operator-friendly cause.
- Code → review loop never converges → max-iteration triggers escalation.
- Customer codebase exists already (brownfield) → researcher stage analyzes existing code first; subsequent stages branch from current state.
- Cost runs over → spec 007 aborts with state preserved; operator can extend.
- Approval gate timed out → project pauses; resumes on operator action.

## Open questions

- Default-agency team contents: ship as a starter pack in v1, or curate via marketplace? (Lean: ship in v1; iterate via marketplace.)
- Brief parsing: free-form vs structured? (Lean: free-form + LLM-driven extraction by researcher agent.)

## Related specs

`004-control-plane-durable-state`, `005-approvals-hitl`, `007-budget-governance`, `013-team-orchestration-orgchart`, `014-inter-agent-comms-tickets`, `015-workspace-codebase-mount`, `017-multi-project-juggling`, `026-task-management-drivers-flow-rules`.
