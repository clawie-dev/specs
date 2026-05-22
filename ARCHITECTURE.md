# Clawie — System Architecture

**Date:** 2026-05-22
**Status:** Foundation document
**Pairs with:** `RESEARCH-NOTES.md`, `ROADMAP.md`, `speckit/constitution.md`

---

## 1. Architectural thesis

Clawie is a **layered, git-versioned, durable agency platform**. Four layers, each independently swappable, all observable, all benchmarked.

```
┌─────────────────────────────────────────────────────────────────┐
│ SURFACES                                                        │
│   AdonisJS CLI (ace)   Web Dashboard (Inertia)   REST + WS API   │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│ CONTROL PLANE  (AdonisJS app + SQLite/Postgres)                 │
│   ┌─────────────┐  ┌────────────┐  ┌──────────────┐             │
│   │ Task Store  │  │ Approvals  │  │ Cost Ledger  │             │
│   │ (durable)   │  │ HITL       │  │ Budgets      │             │
│   └─────────────┘  └────────────┘  └──────────────┘             │
│   ┌─────────────┐  ┌────────────┐  ┌──────────────┐             │
│   │ Audit Log   │  │ Org Chart  │  │ Benchmark    │             │
│   │ (immutable) │  │ Team Flows │  │ Tracker      │             │
│   └─────────────┘  └────────────┘  └──────────────┘             │
│   ┌─────────────┐  ┌────────────┐                               │
│   │ Scheduler   │  │ Backup     │                               │
│   │ (* * * * *) │  │ Runner     │                               │
│   └─────────────┘  └────────────┘                               │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│ POLICY + CREDENTIAL LAYER                                       │
│   Permission engine (Clawie rules)                              │
│   Credential broker (Outcall egress proxy)                      │
│   Plugin signature verifier                                     │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│ AGENT RUNTIME (Docker containers, ephemeral)                    │
│   ┌──────────────────────────────────────────┐                  │
│   │ Container: one agent task               │                  │
│   │   /agent       (SOUL.md, AGENTS.md, …)  │                  │
│   │   /workspace   (mounted customer code)  │                  │
│   │   /skills      (lazy-loaded)            │                  │
│   │   agent-loop   (model router, MCP)      │                  │
│   └──────────────────────────────────────────┘                  │
│      ↓ all egress through Outcall (when enabled)                │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│ TOOLS / CONNECTORS / MODEL ROUTER                               │
│   MCP servers • GitHub • Linear • Slack • SSH • DNS • LLMs      │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│ EVAL HARNESS (separate from production)                         │
│   Internal regression suite • External benchmarks • Score over  │
│   commits • Per-agent quality trend                             │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Component responsibilities

### Surfaces

- **CLI (`ace`)** — AdonisJS Ace commands: `clawie agent new`, `clawie team start`, `clawie project new`, etc. Powers all scripting and CI.
- **Web Dashboard** — Inertia + Vue/React. Approvals queue, change-review (per-agent PRs), benchmark trends, fleet view, project status, cost dashboard.
- **REST + WebSocket API** — OpenAPI-documented. Everything in the CLI/Dashboard is callable. WebSocket for live container/agent streams.

All three surfaces are **thin clients of the API**. The API is the only writer to the control plane.

### Control plane

Single AdonisJS application. SQLite for v1 single-host (canonical default per spec 004), Postgres for scale. No Redis dependency — queue + locks live in the same store via pg-boss (Postgres) or a lightweight tabled queue (SQLite).

- **Task store** — one row per agent invocation. State machine: `queued → running → awaiting_approval → completed/failed/aborted`. Idempotency keys.
- **Approvals** — pending decision queue. Tied to a task. Decision rules can be auto-generated from approvals ("always allow X for agent Y").
- **Cost ledger** — every LLM call, container minute, and external API call logged with cost. Budgets at agent/team/project levels enforce hard caps.
- **Audit log** — immutable append-only table. Captures every state transition, every tool call, every approval, every config change.
- **Org chart / team flows** — declarative team config (YAML in the team's git repo) describing roles and who-talks-to-whom.
- **Benchmark tracker** — scores per agent over commits/time. Surfaces regression alerts.
- **Scheduler** (spec 027) — single core ticker firing every minute. Consults per-agent schedules (declared in each agent's git repo) and platform schedules, dispatches due work as durable tasks. **Two schedule kinds:** `agent` (invokes the agent, costs LLM tokens) and `script` (deterministic script, ~$0 LLM cost, escalates to an agent only on failure).
- **Backup runner** (spec 028) — scheduled via the scheduler; writes age-encrypted snapshots to S3/R2/local.
- **Upgrade runner** (spec 029) — versioned upgrade flow with pre-snapshot, smoke test, automatic rollback on failure.
- **Webhook dispatcher** (spec 030) — durable outbound delivery with HMAC signing, retry, dead-letter; inbound ingest with signature verification, replay protection, ACK-then-handle.

### Policy + credential layer

- **Permission engine** — Clawie's own rules (what an agent *may* attempt: tool calls, files, commands). Evaluated before runtime even tries.
- **Credential broker** — secrets never touch the container. Outcall (or a lightweight broker sidecar when Outcall is disabled) injects credentials at egress.
- **Plugin verifier** — skills/plugins are signed; verifier rejects unsigned ones unless explicitly trusted in config.

### Agent runtime

- **One Docker container per agent task.** Ephemeral. No state lives inside.
- **Mounts:**
  - `/agent` — read-only mount of the agent's git repo (SOUL.md, AGENTS.md, TOOLS.md, skills/, etc.).
  - `/workspace` — read-write mount or copy of the customer codebase (e.g., a Laravel project).
  - `/skills` — lazy-loaded skill artifacts.
- **Agent loop** — small TypeScript runtime that reads SOUL/AGENTS/TOOLS, talks to model router, executes tools.
- **Egress** — all network calls go through Outcall when enabled. Without Outcall, container has Docker-native network isolation only (still containerized, but no rule-based egress).

### Outcall integration modes

| Mode | Description | Use when |
|---|---|---|
| **Disabled** | Plain Docker network. No egress rules. | Local dev, fully trusted agents. |
| **Sidecar** | Outcall runs as a sidecar container in the same Docker network. Agent egresses through it. | Standard production. |
| **Host** | Outcall daemon on host; agents route via host DNS/iptables. | Multi-agent on one host with shared policy. |

### Tools, connectors, model router

- **Model router** — abstracts Anthropic, OpenAI, Gemini, local (Ollama / llama.cpp), with fallback, per-agent preference, cost-aware routing.
- **MCP server registry** — agents declare MCPs they need; control plane validates and grants.
- **Connector SDK** — community can write connectors (publish to marketplace). Connector is a thin tool adapter, runs inside the agent container, talks to a tool's API.
- **Task-management drivers** — Linear, Jira, GitHub Projects, etc., plug in here as pluggable drivers (spec 026). Drivers + per-team flow rules enforce status ownership and allowed transitions so an agent literally cannot push a ticket past Code Review/QA/Security to Ready-for-Release; the driver refuses the API call. Drivers use the credential broker and egress through Outcall like any other tool.

### Eval harness

- **Lives in a separate process from production.** Runs nightly + on every agent self-modification merge.
- **Internal regression suite** — task fixtures per agent role (researcher, coder, reviewer, etc.). Scores: correctness, cost, latency, safety violations.
- **External benchmarks** — SWE-bench-style runs (opt-in).
- **Per-commit trend** — dashboard chart of score over time. Regression below threshold blocks merge.

## 3. Data model (high level)

```
projects ──┬── tasks (state machine)
           ├── budgets
           ├── audit_events
           └── workspaces

teams ─────┬── agents (FK → agent_repos)
           └── flows (org chart + comms)

agent_repos ─── commits ─── benchmarks
                          └── changes_pending_review

plugins ─── versions ─── signatures
         └── installations (per team / per agent)

approvals ─── decisions ─── auto_rules

cost_ledger ─── llm_calls
             ├── container_minutes
             └── external_api_calls
```

## 4. Git monorepo layout

Root repo at `~/.clawie/config/` (or wherever the operator points to):

```
clawie-config/                     # ROOT REPO
├── .git/
├── clawie.yaml                    # platform config
├── outcall.yaml                   # Outcall global config (optional)
├── teams/
│   ├── alpha-team/                # SUBMODULE (its own git repo)
│   │   ├── team.yaml              # roles, flows, org chart
│   │   ├── agents/
│   │   │   ├── researcher/       # SUBMODULE per agent
│   │   │   │   ├── SOUL.md
│   │   │   │   ├── AGENTS.md
│   │   │   │   ├── TOOLS.md
│   │   │   │   ├── skills/
│   │   │   │   └── benchmarks/
│   │   │   ├── coder/             # SUBMODULE
│   │   │   ├── reviewer/          # SUBMODULE
│   │   │   └── ...
│   │   └── budgets.yaml
│   └── ...
├── skills-registry/               # SUBMODULE (mirrored marketplace)
├── projects/
│   ├── server-monitoring-saas/    # SUBMODULE (per-project state)
│   │   ├── brief.md
│   │   ├── pipeline-state.json
│   │   └── artifacts/
│   └── ...
└── workspaces/                    # mounted customer code (gitignored, individual repos)
```

A team submodule may also contain `task-management.yaml` (spec 026) declaring its external task-system driver and status-flow rules. That file is reviewed via PR like every other config and is enforced by the driver at every transition attempt.

**Why submodules per agent:** rollback granularity. `cd teams/alpha-team/agents/researcher && git revert HEAD` rolls back ONLY that agent's last self-modification, without touching anyone else.

**Why root repo:** captures the relationships and platform config. Operators see one PR history of "what changed in the agency" via submodule pointer bumps.

## 5. The end-to-end software agency pipeline

The flagship workflow. A single command (`clawie new-project "Build me a server monitoring SaaS"`) kicks off a pipeline whose state machine looks like:

```
brief → research → spec → architecture → code-iterate → 
  ├── code-review (loop)
  ├── security-review
  ├── performance-review
  ├── docs-writing
devops-setup → deploy-staging → e2e-test → deploy-prod → 
  marketing-site → launch-announcement → post-launch-monitoring
```

Each stage maps to one or more agent roles in a team. The control plane owns the state machine; the team's `flows` config maps state transitions to agent invocations. Approvals are pluggable at any transition.

## 6. Non-functional priorities (in order)

1. **Stability.** Catch failure modes that bite OpenClaw/Hermes: stuck agents, runaway costs, dead workflows. Liveness + timeouts + cause-of-failure capture is P0.
2. **Security.** Default-deny egress (Outcall), default-deny actions (permissions). Optional but recommended for any non-local install.
3. **Observability.** You can't run an autonomous agency you can't see. Trace + audit + cost are first-class.
4. **Recoverability.** Every state survives restart; resumable from any point.
5. **Performance.** Optimization comes after the above. Avoid premature optimization; focus on correctness.

## 7. Anti-architecture (things we explicitly are not)

- **Not a chat UI for an LLM.** Clawie orchestrates teams. The chat is incidental.
- **Not an MCP server itself.** It hosts agents that *use* MCPs.
- **Not a model provider.** BYOK, always.
- **Not a CI/CD replacement.** Agents can invoke CI/CD; Clawie is not GitHub Actions.
- **Not a single-binary "just works" magic.** Configuration is explicit, validated, and git-tracked.
- **Not multi-tenant in the SaaS sense (v1).** Self-hosted per organization. Multi-team within one host yes; multi-customer-on-one-Clawie-instance not in v1.

---

See `ROADMAP.md` for delivery phases and `speckit/` for full feature specs.
