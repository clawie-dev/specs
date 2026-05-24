# 000 — Product: Clawie, the Autonomous Software Agency Framework

**Feature ID:** 000
**Type:** Product overview
**Status:** Draft
**Spec version:** 2.0.0
**Constitution:** `specs/speckit/constitution.md` v2.1.0
**Owner:** Mark Diderichsen

---

## Vision

> A small operator can point Clawie at an idea — *"Build me a server monitoring SaaS"* — and get back a deployable, documented, marketed product, while watching every decision in a dashboard, approving the few things that matter, and rolling back any specific agent's behavior to last Tuesday with one click.

Clawie is the substrate. It does not write the code itself — agents do, in containers, with rules. Clawie is what makes that auditable, recoverable, version-controlled, benchmarked, and safe enough that you can let it run.

## Positioning vs. the market

| Category | Example | What they do well | Where Clawie wins |
|---|---|---|---|
| Personal agent runtime | OpenClaw, Hermes | UX, channel breadth | Multi-team, durable state, granular git rollback |
| Coding agent | Claude Code, OpenHands | Coding UX | End-to-end pipeline (spec→launch), not just coding |
| Control plane | Paperclip, Viktor | Governance, budgets | Open source, git-versioned, with security wrapper |
| Security wrapper | NemoClaw | Hardened sandbox | Native, Outcall-integrated, configurable per team |
| Eval | SWE-bench | Reproducible measurement | Embedded continuous regression detection |
| Multi-agent | crewAI, MetaGPT, AutoGen | Patterns | Production-grade durability, real PR/deploy artifacts |

Clawie is the only framework that explicitly **combines all four layers and ships an end-to-end agency workflow** with per-component git rollback.

## Target users

| Persona | What they need from Clawie | Primary surface |
|---|---|---|
| **Solo operator / indie hacker** | One-command install, run an agency on a $20 VPS, approve key decisions, see what's happening, ship a side product per month | Web dashboard + CLI |
| **Small SaaS founder** | Run a team of agents on their existing Laravel/Rails/Next codebase, get features shipped, see audit | Dashboard + Git PR review |
| **Platform operator at a small studio** | Multi-project, multi-team, budget control per client | Dashboard + API |
| **Plugin / skill author** | Publish skills/connectors to marketplace, get installs | CLI + marketplace |
| **Researcher / red-teamer** | Reproducible benchmarks, isolated runs, eval suite control | CLI + API |

Explicitly **not** targeted in v1:
- Enterprise procurement teams (no SaaS, no SOC2 in v1)
- Mobile-first users (web only)
- Non-technical end-customers (the operator is technical)

## Top-level user scenarios

### PROD-US-001 [P0] — Start a new project from a brief

**As an** operator
**I want** to run `clawie project new "Build me a server monitoring SaaS"`
**So that** a fresh team picks up the brief, drafts a spec, asks me clarifying questions, and stays in a paused-awaiting-approval state until I unblock the spec.

**Acceptance:**
- A project workspace is created (git submodule under `projects/`).
- The default agency team is assigned.
- A pipeline task in `research` state appears in the dashboard.
- The first approval (spec sign-off) is queued before any code is written.

### PROD-US-002 [P0] — Approve / reject an agent's self-modification

**As an** operator
**I want** to see every change an agent proposes to its own SOUL/AGENTS/TOOLS/skills as a PR in the dashboard with diff + benchmark delta
**So that** I can merge improvements and reject regressions without trusting the agent's self-assessment.

**Acceptance:**
- Every self-modification commits to a branch in the agent's repo.
- Dashboard shows: diff, benchmark before/after, change reason, related tasks.
- Merge requires explicit operator action (or pre-authorized rule).
- Reject closes the branch; merge advances the agent.

### PROD-US-003 [P0] — Approve a sensitive action

**As an** operator
**I want** to be notified when an agent tries an action outside its current policy (new domain, new tool, file write outside workspace)
**So that** I can approve once and optionally generate a reusable rule.

**Acceptance:**
- Within 1 second of the agent's request, the dashboard surfaces it.
- Default decision window is 24 hours; configurable per team.
- Approve | Approve + Auto-rule | Deny | Deny + Block-rule options.
- Outcome logged to audit with reason.

### PROD-US-004 [P0] — See cost and budget

**As an** operator
**I want** to see live cost per agent, team, and project
**So that** I can stop a runaway project before it bankrupts me.

**Acceptance:**
- Live cost streaming in dashboard.
- Soft alerts at 75% of budget; hard stop at 100% (configurable).
- Stop triggers task abort, not container kill — state preserved.
- Cost broken down by LLM tokens, container minutes, external APIs.

### PROD-US-005 [P0] — Pause / resume / abort a project

**As an** operator
**I want** to pause an in-flight project mid-pipeline
**So that** I can stop activity, inspect, and either resume or abort cleanly.

**Acceptance:**
- Pause halts active tasks at their next safe checkpoint.
- Resume picks up from the checkpoint with no duplicate side-effects.
- Abort cleans up containers and marks the project archived with reason.
- All transitions audited.

### PROD-US-006 [P1] — Define a custom team

**As an** operator
**I want** to write a `team.yaml` (org chart + flows + budgets) and `clawie team install`
**So that** I can stand up a specialized agency (e.g., DevOps consultancy with a different role mix) without writing Clawie code.

**Acceptance:**
- `team.yaml` validated against schema before merge.
- Smoke test: each role spawns, claims a no-op task, exits.
- Team appears in dashboard with org chart visualization.

### PROD-US-007 [P1] — Mount a customer codebase

**As an** operator
**I want** to point Clawie at a Laravel repo on disk (or a remote git URL)
**So that** an agent gets a fresh worktree, makes changes, and opens a PR against my repo.

**Acceptance:**
- Mount or copy mode configurable per agent.
- Read-only by default; write requires permission grant.
- Agent's changes captured as a commit + branch in the customer repo.
- Optional auto-push + PR open (requires credential grant).

### PROD-US-008 [P1] — Roll back a single agent

**As an** operator
**I want** to revert the `coder` agent to its state from 2 weeks ago
**So that** I can isolate a quality regression without rolling back other agents.

**Acceptance:**
- `clawie agent rollback coder --to <commit>` updates the submodule pointer.
- Other agents unaffected.
- Audit captures the rollback with operator id + reason.
- Next task uses the rolled-back agent definition.

### PROD-US-009 [P1] — Benchmark trend

**As an** operator
**I want** to see a per-agent quality score over the last N commits
**So that** I can tell whether the agent is improving, drifting, or regressing.

**Acceptance:**
- Dashboard chart per agent with score history.
- Annotated with significant commits (self-mods, skill installs).
- Regression alerts surface at threshold cross.

### PROD-US-010 [P1] — Multi-project juggling

**As an** operator
**I want** to run three projects concurrently and have the scheduler share agents fairly
**So that** none starves and contention is visible.

**Acceptance:**
- Scheduler assigns tasks across projects according to priority.
- Dashboard shows queue depth per project.
- Fair-share configurable per team.

### PROD-US-010b [P1] — Connect to my existing Linear / Jira

**As an** operator already using Linear
**I want** to point Clawie at my Linear workspace and have agents work my Backlog → Spec → Code → Review → Release flow there
**So that** I don't have to maintain a parallel board, and my dev agent literally cannot push tickets past Code Review / QA / Security to Ready-for-Release because the driver refuses the unauthorized transition.

**Acceptance:**
- Configurable per team via `task-management.yaml` (spec 026).
- Status ownership + allowed transitions enforced at the driver, not just in prompt.
- Unauthorized transitions denied with structured cause, visible in audit + dashboard.
- Force-override path requires reason + audited.

### PROD-US-011 [P2] — Install a marketplace plugin

**As an** operator
**I want** to install a signed plugin from `market.clawie.dev`
**So that** I extend my team with community skills/connectors safely.

**Acceptance:**
- Signature verified before install.
- Plugin scoped to team or agent.
- Permissions surface clearly; operator confirms.

### PROD-US-012 [P2] — Stop a stuck agent

**As an** operator
**I want** the platform to detect when an agent has stopped responding
**So that** I do not babysit it.

**Acceptance:**
- Liveness heartbeat per running task.
- Configurable timeout per task type.
- Auto-restart attempts limited; on exceed, escalate to operator.
- Cause-of-failure (oom, deadlock, infinite loop, missing tool, model timeout) captured.

## Headline use cases

### UC-A [P0] — Single-agent task

The smallest unit: one agent, one task, one container, full audit. Foundation for everything else.

### UC-B [P0] — Approve / rollback agent self-modification

The defining workflow. Agents can learn; humans gate the learning.

### UC-C [P0] — End-to-end project pipeline

The flagship: brief → spec → code → review → deploy → docs → marketing. Multiple agents, multiple stages, multiple approval points.

### UC-D [P1] — Customer codebase work

A real Laravel/Rails/Next repo gets a feature added by an agent and ships a PR back.

### UC-E [P1] — Multi-project agency

Three concurrent projects, one team, fair scheduling, no starvation.

### UC-F [P1] — Benchmark regression caught and reverted

Agent's self-mod regresses score → merge blocked → operator sees diff → reject or override with audit.

### UC-G [P2] — New community plugin published

Author writes a skill, signs it, submits to marketplace, gets reviewed, published, installed by another operator.

## Non-goals

- **Not a managed SaaS.** Self-hosted only in v1.
- **Not a foundation model trainer.** BYOK.
- **Not a hosted CI/CD.** Can invoke CI/CD; not a replacement.
- **Not a user authentication platform for end-users.** Operator-token auth only in v1.
- **Not multi-tenant in the SaaS sense.** One instance = one trust boundary.
- **Not a chat interface for an LLM.** Chat is an incidental channel.
- **Not a vector-DB / memory-store wrapper.** Memory lives in agent repos or external stores; Clawie doesn't ship its own.

## Success criteria for v1

A small SaaS founder can:

1. Install Clawie on a $20 VPS in under 10 minutes.
2. Define an agency in YAML (or use a starter pack), have it validated, and start it.
3. Run `clawie project new "Add Stripe billing to my Laravel app at /home/me/myapp"` and walk away.
4. Receive 1–3 approval requests over the next hour (spec sign-off, sensitive permission, deploy gate).
5. End with a merged PR, a documented feature, a deployed staging environment, and a cost under $5.
6. See every decision in the dashboard, including which agent did what.
7. Roll back to last week with one command if anything degraded.

If any of these breaks, v1 isn't done.

## Open product questions (defer to discuss-phase or specific spec)

- Default eval-suite contents (which fixtures ship with the platform?) — defer to `019-agent-benchmarks-evals`
- Marketplace moderation model — defer to `024-plugin-marketplace`
- Multi-operator (team of humans, not agents) — deferred past v1
- Distributed Clawie (multiple hosts) — deferred past v1

## Constitutional alignment

This spec is the integration point for every principle. Each feature spec under `001-`...`025-` implements specific principles; see the respective specs.

---

## Related specs

| Spec | Principle anchor |
|---|---|
| `001-monorepo-git-config` | II Git-source-of-truth |
| `002-container-runtime-outcall` | I Layered, IV Default-deny, X Pluggable |
| `003-policy-permissions` | IV, IX HITL |
| `004-control-plane-durable-state` | V Durable, VI Observable |
| `005-approvals-hitl` | IX, IV |
| `006-observability-audit-cost` | VI |
| `007-budget-governance` | VI, IX |
| `008-agent-definition` | II |
| `009-agent-self-modification-pr` | II, III, IX |
| `010-skills-plugins-lazy-load` | X, III |
| `011-model-router-providers` | X |
| `012-credential-broker` | IV, security constraints 3 |
| `013-team-orchestration-orgchart` | XII end-to-end |
| `014-inter-agent-comms-tickets` | V, VI |
| `015-workspace-codebase-mount` | I, IV |
| `016-software-agency-pipeline` | XII |
| `017-multi-project-juggling` | V, VIII |
| `018-config-validation-pre-merge` | III |
| `019-agent-benchmarks-evals` | VII |
| `020-stability-recovery` | VIII |
| `021-cli` | I surfaces |
| `022-web-dashboard` | I surfaces, IX |
| `023-rest-api` | I surfaces, X |
| `024-plugin-marketplace` | XI, III |
| `025-onboarding-install` | III, VIII |
| `026-task-management-drivers-flow-rules` | III, V, IX, XII (Linear/Jira drivers + status-flow rules) |
| `027-scheduler-recurring-work` | II, V, VI (per-agent crons + core ticker; dual-mode: agent + script) |
| `028-backup-disaster-recovery` | V, VI, VIII (built-in scheduled backups + restore) |
| `029-upgrades-and-migrations` | III, VIII (versioned upgrades with rollback) |
| `030-webhooks-inbound-outbound` | IV, V, VI (signed, retried, ordered, dead-letterable) |
| `031-test-discipline-and-coverage` | III, VIII (mirror tests, coverage gates, pyramid) |
