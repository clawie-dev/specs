# Clawie — Research Synthesis

**Source:** Private deep-research report on autonomous AI agent frameworks (172 lines, Danish) — synthesized into this document.
**Date:** 2026-05-22
**Status:** Foundation document — informs constitution & all feature specs

---

## 1. The single most important takeaway

Existing "agent frameworks" are not one category. They are four overlapping layers — and most failures come from building them as a monolith:

| Layer | What it does | Best examples in survey |
|---|---|---|
| **Runtime** | Executes agent loops, owns the channels | OpenClaw, Hermes, Claude Code |
| **Control plane** | Tasks, state, approvals, budgets, audit | Paperclip, Viktor, LangGraph |
| **Security wrapper** | Sandboxing, egress control, credential isolation | NemoClaw (over OpenClaw), Outcall |
| **Eval harness** | Reproducible benchmarks, regression detection | SWE-bench, SWE-agent |

**Clawie must explicitly architect around these four layers, with each layer independently swappable.** That is the structural advantage. Every framework in the survey that tried to be all four ended up shipping a single layer well and the others poorly.

## 2. Patterns to steal (high consensus across sources)

| Pattern | Source(s) | Clawie translation |
|---|---|---|
| **Durable state over ephemeral subagents** | Paperclip kanban, Hermes board, LangGraph durable execution | Every task is a first-class row in Postgres with state machine + idempotency keys. Restart-safe. |
| **Lazy-loaded skills, not all-tools-in-prompt** | Viktor (3,200+ tools), Claude Code skills | Hot capability summaries in system prompt; full skill loaded on demand. |
| **Credential brokering at the network edge** | NemoClaw egress-proxy, Viktor, Claude Code web | Secrets injected by Outcall at egress. Agent never sees raw keys. |
| **Composable safety in layers** | NemoClaw (4 layers), Hermes (7 layers) | Permissions + sandbox + approvals + audit + rate limits + trust boundaries + budgets. Don't trust one. |
| **First-class observability** | Paperclip audit log, LangSmith traces | Immutable audit log, cost ledger, traces. Answer "who/what/why/cost" for every action. |
| **HITL for destructive / unrecognized actions** | Viktor approvals, Paperclip board, Claude Code permissions | Default-deny new actions. Approval generates a re-usable rule (with audit). |
| **Bring-your-own-runtime / pluggable** | Paperclip, LangGraph | LLM provider, container runtime, storage, message channels all abstracted behind interfaces. |
| **Ephemeral workspaces per task** | OpenHands, NemoClaw sandbox-pod | Each agent invocation gets a fresh container with a mounted workspace; state lives in control plane, not the container. |

## 3. Anti-patterns to avoid (high consensus)

| Anti-pattern | Where it bit | Clawie mitigation |
|---|---|---|
| **Prompt-only security** | Claude Code SDK subagent permission filter issues; OpenClaw single-trust gateway | Actual OS/network boundaries via Docker + Outcall. Prompts are advisory, not enforcing. |
| **Hostile multi-tenancy in one gateway** | OpenClaw docs explicitly disclaim it | One trust boundary per container. Multi-tenant = multi-host or multi-VM. |
| **Non-resumable onboarding / hidden state** | NemoClaw early friction, Paperclip stale-lock bugs | All install steps idempotent + resumable. State machine surfaced in dashboard. |
| **Benchmark overfitting** | SWE-bench Verified contamination | Internal evals separate from public benchmarks. Diverse eval suite. |
| **Aggressive proactivity too early** | Viktor's own warning | Proactivity is opt-in per agent. Default: reactive. |
| **In-process subagent sprawl** | Hermes/Paperclip both moved to durable kanban | Every "subagent invocation" is a task row, not a thread. |
| **Hooks running with host privileges** | Claude Code hooks | Hooks run inside the agent container, never on host. |

## 4. Where each surveyed system gets it right (and what to copy specifically)

- **OpenClaw** — channel breadth (Slack/Telegram/Discord/Web/CLI). Multi-channel surface is good, but it should be a thin adapter over the API, not the runtime. *Steal:* the channel adapter SDK pattern.
- **NemoClaw** — egress proxy, credential broker, blueprint YAML, Landlock/seccomp/netns. *Steal:* nearly the entire security architecture, but implemented via our own Outcall + Docker rather than k3s-in-a-box.
- **Claude Code** — skills as lazy artifacts, hooks, MCP, GitHub Actions integration. *Steal:* skill packaging convention; agent can author skills on a branch and propose merge.
- **Viktor** — approvals UX, code-as-tool-call, 3,200 connector pattern. *Steal:* the lazy-loading principle + approval flow shape.
- **Paperclip** — org chart, budgets, immutable audit, durable tickets, heartbeats, plugin workers. *Steal:* the whole control-plane mental model. This is closest to what Clawie wants.
- **SWE-agent / SWE-bench** — YAML-driven agent config, Docker-based reproducible eval. *Steal:* the eval harness pattern (Docker-isolated, leaderboard internal).
- **Hermes** — learning loop, persistent memory, kanban, cron, MCP, checkpoints/rollback. *Steal:* checkpoint + rollback semantics for agent self-modifications.
- **OpenHands** — ephemeral workspace SDK + sandboxed agent loop. *Steal:* the codebase mount/copy pattern.
- **LangGraph** — durable execution, HITL, streaming. *Steal:* the state-machine-first task model.

## 5. What Clawie does that the survey does NOT do

These are the differentiation points — none of the eight frameworks in the survey does all of these together:

1. **Every configuration (root + per-team + per-agent + per-skill) is its own git repo, individually rollback-able.** No system in the survey has this granularity. Hermes has checkpoints; Paperclip has audit; nobody has per-component VCS rollback.
2. **Agent self-modification flows through PR review by default.** Hermes and OpenClaw allow self-mod but with weaker review. Clawie's default is *every* SOUL/AGENTS/TOOLS edit is a branch → dashboard PR.
3. **Continuous benchmarking with per-commit regression detection on agent quality.** SWE-bench is external. Clawie embeds an eval suite that runs after every merge to detect "this agent got dumber".
4. **End-to-end software agency pipeline.** From `clawie new-project "Build me a server monitoring SaaS"` → research → spec → code → review → security → docs → devops → launch → marketing. Other multi-agent frameworks (MetaGPT, crewAI) simulate a software company; Clawie ships one end-to-end with real artifacts (PRs, deploys, landing pages).
5. **Outcall-grade egress isolation as a configurable layer.** Optional but recommended. No survey system has rule-based, deny-by-default egress with the granularity Outcall provides.
6. **Validation before merge — for everything.** Agent configs, team flows, plugin manifests, skills are all schema- and smoke-test-validated before reaching main. "OpenClaw stops responding" is the explicit anti-goal.

## 6. Risks and how the research tells us to mitigate

| Risk class | Specific failure | Mitigation (per research) |
|---|---|---|
| **Security** | Prompt injection, supply-chain plugin, leaked secrets | Outcall egress policy, signed plugins, low-priv workers, broker-injected credentials, red-team eval suite. |
| **Operational** | Stuck workflows, stale locks, lost state | Durable task store + idempotency + recovery semantics + structured traces. |
| **UX** | Over-proactive agent → user fatigue | Proactivity opt-in; default-reactive; approval visibility in dashboard. |
| **Eval** | Overfitting public benchmarks | Internal regression suite + cost/speed/quality dashboard, not single number. |
| **Stability** | Agent stops responding (OpenClaw class) | Liveness heartbeats, deadlock detection, per-task timeouts, auto-restart with cause-of-failure capture. |

## 7. Open architectural questions the research surfaces (decisions Clawie must make)

1. **Local-first vs. server-first?** Most survey systems are server-first (Paperclip, Viktor). OpenClaw is single-host. Clawie target: server-first, but the server can run on a $5 VPS or your laptop. Single binary that scales out.
2. **Bring-your-own-LLM vs. opinionated default?** Bring-your-own with sensible default (Claude/OpenAI/local). Required at runtime — not configurable at install time.
3. **One agent per container vs. agent pool?** One per task invocation (ephemeral). Pool reuse is a later optimization.
4. **Inter-agent comm: in-memory bus or via control plane?** Always via control plane (durable). No direct container-to-container comm.
5. **Plugin distribution model?** Signed git tags pulled from `market.clawie.dev`. Local install possible. Review required for the public marketplace.

---

## TL;DR for the spec

Clawie is **the layered competitor**: explicitly four layers, each swappable, all git-versioned, all observable, all benchmarked. The closest single referenced point is "Paperclip control plane + NemoClaw security + Claude Code DX + Hermes durability + SWE-bench eval — all glued by a git-everything substrate." That's the spec.
