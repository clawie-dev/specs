# Clawie — Phased Roadmap

**Date:** 2026-05-22
**Status:** Plan-of-record
**Source of truth:** Each phase ships specific spec folders from `specs/speckit/`.

The roadmap is **specced end-to-end** but **delivered in phases**. We do not promise dates; we promise scope per phase.

---

## Guiding principles

1. **Foundation before features.** No agent autonomy until containers, control plane, audit, and git substrate are rock-solid.
2. **Single agent before multi-agent.** Get one agent working perfectly end-to-end before adding org charts.
3. **Manual before automatic.** Approval gates default-on; remove gates only with evidence (rule auto-generation).
4. **Validated before merged.** Every config that touches production passes a validation gate. No exceptions.
5. **Benchmark from day 1.** Even a trivial benchmark beats no benchmark. Trend data only accumulates if it starts early.

---

## Phase 0 — Skeleton (the boring stuff)

**Goal:** AdonisJS app skeleton, monorepo git layout, container build pipeline, CI.

**Ships:**
- AdonisJS application scaffold (CLI + API + Dashboard skeleton, no features)
- Root git repo + submodule scripts
- Dockerfile for the agent runtime base image
- CI pipeline (lint, type-check, test)
- Basic auth (operator token)

**Specs:** None (infrastructure). Captured in `clawie/README.md` and `clawie/docs/development.md`.

**Exit criteria:** `clawie --version` works. `ace serve` runs. Empty dashboard renders. Tests run in CI.

---

## Phase 1 — Foundation (sandbox + control plane)

**Goal:** One agent can run in a sandboxed Docker container, do one thing, log everything, recover from restart.

**Ships:**
- `001-monorepo-git-config` — root repo + submodule per agent
- `002-container-runtime-outcall` — Docker spawn, mount agent + workspace, lifecycle
- `003-policy-permissions` — basic permission engine (Clawie rules)
- `004-control-plane-durable-state` — task store, state machine, idempotency, recovery
- `006-observability-audit-cost` — audit log, traces, cost ledger
- `008-agent-definition` — SOUL.md, AGENTS.md, TOOLS.md schema
- `011-model-router-providers` — Anthropic + OpenAI providers, fallback
- `020-stability-recovery` — liveness, timeouts, stuck-agent detection
- `021-cli` — minimum: `agent new`, `agent run`, `task list`, `task get`

**Exit criteria:**
- One agent can be defined in git, spun up in a container, run a single task, log to the audit, terminate cleanly.
- Killing the API mid-run leaves the task recoverable.
- Every cost is logged.

---

## Phase 2 — Safety & approvals

**Goal:** Default-deny everything sensitive. HITL works. Outcall integration is real.

**Ships:**
- `005-approvals-hitl` — approval queue, decision windows, auto-rule generation
- `007-budget-governance` — budget caps per agent/team/project
- `012-credential-broker` — secrets via Outcall (sidecar mode)
- `018-config-validation-pre-merge` — schema + lint + smoke test for agent configs
- `022-web-dashboard` — approvals queue, task view, cost view (minimum viable dashboard)
- `023-rest-api` — approvals, tasks, agents endpoints
- Outcall integration: sidecar mode wired up with example ruleset

**Exit criteria:**
- An agent that tries to `curl` an unapproved host is blocked.
- Operator sees the request in dashboard, approves, agent retries, succeeds.
- Repeat request without approval is auto-allowed (rule generated).
- Budget overrun aborts the task with a clear reason.
- Invalid agent config cannot be merged to its branch's main.

---

## Phase 3 — Self-modification + benchmarks

**Goal:** Agents can propose changes to themselves. Quality is measurable.

**Ships:**
- `009-agent-self-modification-pr` — branch-on-edit, dashboard PR view, per-agent rollback
- `010-skills-plugins-lazy-load` — skill packaging, lazy load, install/upgrade
- `015-workspace-codebase-mount` — mount/copy modes for customer code
- `019-agent-benchmarks-evals` — eval harness, per-commit scoring, regression detection

**Exit criteria:**
- An agent edits its SOUL.md → branch created → dashboard shows diff + benchmark delta → operator merges or rejects.
- Daily eval run produces a score per agent. Dashboard shows trend.
- Score regression beyond threshold blocks the merge.

---

## Phase 4 — Teams & multi-agent

**Goal:** Multi-agent flows. Org chart. Inter-agent comms.

**Ships:**
- `013-team-orchestration-orgchart` — team.yaml, roles, flows
- `014-inter-agent-comms-tickets` — durable ticket-based messages

**Exit criteria:**
- A two-agent team (researcher → coder) can complete a task with a handoff visible in the dashboard.
- Tickets survive a restart.
- Cyclic flows detected and rejected.

---

## Phase 5 — End-to-end software agency

**Goal:** `clawie new-project "Build me a server monitoring SaaS"` produces a deployable artifact with minimal human input. Optionally connected to Linear/Jira so the work tracks where the operator already lives.

**Ships:**
- `016-software-agency-pipeline` — the canonical pipeline state machine
- `017-multi-project-juggling` — scheduler across projects
- `026-task-management-drivers-flow-rules` — Linear + Jira drivers; status-flow rules
- A reference team: `default-agency` (researcher, architect, coder, reviewer, security, devops, doc-writer, marketer)
- Project workspace template + reference run

**Exit criteria:**
- A new-project command kicks off the pipeline.
- Operator sees stage transitions in dashboard with approval gates at the obvious points (post-spec, post-deploy, post-launch).
- Pipeline completes for a sample brief and produces: spec → repo → tests → staging deploy → docs → landing page.
- Running multiple projects concurrently does not starve or block each other.
- When configured with a Linear driver, the same pipeline drives Linear status transitions and refuses bypass attempts.

---

## Phase 6 — Distribution & ecosystem

**Goal:** Others can install, contribute, and trust.

**Ships:**
- `024-plugin-marketplace` — market.clawie.dev with signed plugins
- `025-onboarding-install` — one-command install + first-run validator
- Production-grade docs site
- Reference agencies as installable starter packs

**Exit criteria:**
- A new user runs the install script and sees a working dashboard with a working sample agent within ~5 minutes.
- A community plugin can be submitted, reviewed, signed, and installed.
- All starter packs pass benchmark gate.

---

## Cross-cutting (every phase)

These run as continuous tracks, not phases:

- **Stability tax:** Every phase must keep recovery + liveness green. Regressions block phase exit.
- **Benchmark tax:** Every new agent role ships with an eval fixture.
- **Security review:** Each phase gets a threat-model checkpoint and a red-team session.
- **Docs:** Every spec that ships has user-facing docs.

---

## Out-of-scope until further notice

- Cloud-hosted SaaS Clawie (we ship self-hosted only)
- Mobile dashboard (web is responsive; native deferred)
- Voice surfaces (defer behind Phase 5)
- Multi-tenant in the customer-of-clawie sense (defer behind Phase 6+)
- Training/fine-tuning agents (we run them; we don't train them)
- Proactive agents that wake themselves without explicit schedule/event (re-evaluate post-Phase 5)
