# Clawie Constitution

**Version:** 2.1.0
**Ratified:** 2026-05-22
**Supersedes:** v1.0.0 (`previous-attempts/clawie-attempt-3/specs/speckit/constitution.md`); v2.0.0 (same-day amendment for test discipline)

This constitution is the highest-priority specification for Clawie. Every feature spec MUST reference it. Security and stability constraints are NON-NEGOTIABLE; feature specs MUST NOT relax them.

---

## Mission

Clawie is the open-source framework for running an **autonomous software agency** — teams of AI agents that take ideas from brief to launched product, end to end, under explicit human governance, in fully isolated environments, with every configuration version-controlled in git.

---

## Core Principles

### I. Layered, Not Monolithic

Clawie MUST maintain four explicitly separated layers — **surfaces**, **control plane**, **policy + credential**, and **runtime** — plus a fifth, **eval harness**, that runs out-of-band. Each layer MUST be swappable without rewriting the others. Feature specs MUST identify which layer they belong to. Cross-layer leakage (e.g., a runtime that talks to Postgres directly, or a CLI that bypasses the API) is a defect.

### II. Git Is the Source of Truth for Everything Configurable

Every configuration — the root platform config, each team, each agent, each skill, each project state — MUST live in a git repository. The root Clawie config repo MUST reference per-team and per-agent repos as submodules so each can be rolled back independently. Any change to an agent's identity (SOUL.md, AGENTS.md, TOOLS.md, skills) MUST go through a branch and a dashboard PR review before reaching that agent's main. No agent runs from a dirty working tree.

### III. Validated Before Merged

Every config, manifest, plugin, skill, team flow, and pipeline change MUST pass a validation gate before it reaches main: schema check, dependency check, lint, smoke test (where applicable), and benchmark delta (for agent self-modifications). Validation is mandatory; the operator MAY override only with explicit `--force-merge` *and* an audit-log reason. "OpenClaw stops responding because someone merged a broken skill" is an explicit anti-state.

### IV. Default-Deny Security

No agent, tool, plugin, network destination, or user is trusted by default. Tool calls MUST be evaluated by the permission engine; network egress MUST be evaluated by Outcall (when enabled). Approvals route to a human via the dashboard or a configured channel. Unhandled requests time out into deny within a configurable decision window. Approved requests MAY auto-generate a reusable allow-rule; auto-rules MUST be auditable and revocable.

### V. Durable State, Idempotent Tasks

Every agent invocation MUST be a first-class durable task in the control plane: a row in Postgres with a state machine, idempotency key, and recovery semantics. Ephemeral subagent sprawl is forbidden — child invocations MUST also be durable tasks. Restarting any process at any time MUST leave the system recoverable without lost work or duplicate side-effects.

### VI. Observable by Default

Every state transition, tool call, model call, approval, cost event, and config change MUST be logged to an immutable audit log and emit a structured trace. The cost ledger MUST track LLM tokens, container time, and external API costs per agent/team/project. The operator MUST be able to answer "who did what, why, when, at what cost, and what blocked them" for any past action.

### VII. Benchmarked Continuously

Every agent MUST ship with at least one eval fixture. The eval harness MUST run nightly and on every agent self-modification merge. Score regressions beyond a configurable threshold MUST block merge or trigger an alert. Quality trend is visible per-commit on the dashboard. Benchmarks live in the agent's own repo so they roll back with the agent.

### VIII. Stability Is a P0 Feature

The platform MUST detect stuck agents, deadlocked workflows, runaway costs, and dead processes. Liveness heartbeats, per-task timeouts, and automatic cause-of-failure capture are required, not optional. "It works most of the time" is not acceptable. Any failure mode observed in OpenClaw, Paperclip, or Hermes that this constitution can pre-empt SHOULD be pre-empted by spec, not deferred.

### IX. Human-in-the-Loop by Default

Destructive, irreversible, or unrecognized actions MUST require human approval. Agents MAY request escalation; the operator MAY pre-authorize patterns. Auto-approval rules are explicit, audited, and revocable. The dashboard MUST expose pause, resume, and abort for every running task. No agent is fully autonomous without an explicit, audited decision to make it so.

### X. Pluggable & Vendor-Neutral

Container runtime, LLM provider, storage backend, message channel, secrets backend, and MCP servers MUST be abstracted behind interfaces. BYOK is required at runtime — Clawie ships no provider credentials. Vendor lock-in is a defect. Reference implementations target Docker + Postgres + Anthropic/OpenAI/local, but no spec MAY assume them.

### XI. Open Source First

Clawie's core platform is open source. All core features (orchestration, security, audit, evals, marketplace install) MUST be available without a paid tier. Future enterprise additions MUST be strictly additive — they MUST NOT gate core functionality. Self-hosting MUST remain first-class forever.

### XII. End-to-End Agency, Not Just Coding

Clawie's flagship workflow is the full software agency pipeline: brief → research → spec → architecture → code → review → security → performance → docs → devops → deploy → marketing → post-launch. Specs MUST consider this end-to-end view; isolated "single agent doing one thing" is supported but not the headline.

---

## Security Constraints (NON-NEGOTIABLE)

These cannot be relaxed by any feature spec. Override requires constitutional amendment.

1. Agent runtimes MUST execute in OS-isolated containers (Docker by default; runc/gVisor/Kata optional). No agent code runs on the host.
2. Network egress from agent containers MUST be controllable. When Outcall is enabled, all egress MUST flow through Outcall's policy engine, deny-by-default.
3. Credentials (LLM keys, customer-system tokens, secrets) MUST be brokered at the network edge or via secret-mount sidecars. The agent process MUST NOT see raw long-lived credentials in its environment, files, or memory.
4. Tool/permission decisions MUST be enforced at the runtime/policy layer, not solely by prompt instruction.
5. Container resource limits (CPU, memory, PIDs, disk, time) MUST be enforced via cgroups v2 or equivalent.
6. Agent git repositories MUST enforce branch protection — agents MUST NOT push directly to main of their own repo or any other repo without an approval.
7. Plugins, skills, and connectors MUST be signature-verified before install. Unsigned local installs allowed for development with an explicit flag.
8. Hooks MUST run inside the agent container under the same isolation as the agent — never on the host with operator privileges.
9. All API endpoints MUST validate and sanitize input; all SQL MUST use parameterized statements.
10. All data in transit MUST be TLS 1.2+.
11. Audit logs MUST be append-only and tamper-evident (hash chained or equivalent).
12. Multi-tenancy on a single Clawie instance is NOT a supported security boundary in v1. One Clawie instance = one trust boundary. Multi-org = multi-instance.
13. Self-modification merges to an agent's main MUST require the validation gate to pass AND a human approval (auto-merge requires explicit per-rule pre-authorization in the team config).
14. Outcall integration mode MUST be declared per-team. Operators MAY opt out of Outcall, but doing so MUST emit a clear startup warning and be visible in the dashboard.

---

## Quality Standards

- All code MUST have automated tests before merge. Coverage targets (per spec 031): **≥90% line / ≥85% branch on control plane code**, **≥85% line on controllers/surfaces**, **100% on critical-path code** (audit, secrets, policy, container lifecycle). Coverage regressions block PR merge.
- The framework MUST follow the mirror-file convention from spec 031 (`app/foo/bar.ts → tests/unit/foo/bar.test.ts`). A CI gate enforces this. "Files without a covering test" is answerable by `ls`, not by guess.
- Test pyramid (per spec 031): unit (Japa) → integration with real DB/Docker/Outcall sidecar → E2E with Playwright over CLI + Dashboard + API → smoke per agent → chaos for recovery semantics → nightly load. All tiers run on PRs (slow lanes parallel; chaos + load nightly).
- Every PR implementing a Functional Requirement MUST annotate the relevant test (`// covers: XXX-FR-NNN`) so FR-to-test traceability is greppable.
- Flaky tests are NOT allowed to live: a test failing non-deterministically twice in a week is auto-quarantined; quarantine ticket opened.
- All public APIs MUST have OpenAPI documentation and a typed client (spec 023).
- All specs MUST use RFC 2119 keywords (MUST, SHOULD, MAY) consistently.
- All PRs to the Clawie platform MUST reference the spec they implement.
- Error messages exposed to users MUST NOT leak internal implementation details.
- All database migrations MUST be reversible (or explicitly marked `irreversible: true` with reason, per spec 029).
- All long-running operations MUST have configurable timeouts and surface progress.
- Every operator-facing failure MUST include a cause-of-failure code and a "what to try next" pointer.

---

## Governance

- Constitution changes require explicit approval from the project lead and an open RFC.
- Security constraints (above) MUST NOT be overridden in feature specs; they MAY only be amended in this constitution.
- Amendments are tracked via semantic version and ratification date.
- Each feature spec MUST include a "Constitutional alignment" section identifying which principles it implements or interacts with.
- A spec that violates this constitution is invalid and MUST be revised before implementation begins.

---

## Amendment log

- **v2.0.0 — 2026-05-22** — Initial v2. Adds layered architecture (I), end-to-end agency mission (XII), benchmarking principle (VII), stability principle (VIII), validation-before-merge (III), expanded git-everything (II). Inherits v1's permission/sandbox/HITL principles in stronger form.
- **v2.1.0 — 2026-05-22** — Elevated test discipline from SHOULD to MUST. Added mirror-file convention, per-area coverage targets, test-pyramid mandate, FR-to-test traceability requirement, and flaky-test quarantine policy. New spec 031 (Test Discipline & Coverage) is the source of truth for engineering test rules.
