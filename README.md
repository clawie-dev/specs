# Clawie — Specifications

This folder holds the canonical, end-goal specification for **Clawie, the Autonomous Software Agency Framework**.

It is split across foundation documents and Spec-Kit feature folders. Phases of delivery are captured in `ROADMAP.md`. Nothing here mandates implementation order beyond what the roadmap and feature `depends_on` lines describe.

> **Use:** Read `RESEARCH-NOTES.md` → `ARCHITECTURE.md` → `ROADMAP.md` → `speckit/constitution.md` → `speckit/000-product/spec.md`, then dive into any feature spec that interests you.

## Foundation documents

| File | Purpose |
|---|---|
| `RESEARCH-NOTES.md` | Distilled insights from the deep-research report — what to steal, what to avoid, where Clawie differentiates. |
| `ARCHITECTURE.md` | Layered system design, component responsibilities, git monorepo layout, end-to-end pipeline. |
| `ROADMAP.md` | Phased delivery plan (Phase 0 skeleton → Phase 6 ecosystem). |

## Speckit

| File | Purpose |
|---|---|
| `speckit/constitution.md` | v2 constitution. Non-negotiable principles, security constraints, governance. |
| `speckit/000-product/spec.md` | Product vision, target users, top-level scenarios, success criteria for v1. |

### Feature specs

Each feature is its own folder. v1 contains only `spec.md`; `plan.md` and `tasks.md` are added per feature as it enters delivery per ROADMAP.

| # | Feature | Constitutional anchor |
|---|---|---|
| 001 | Monorepo Git Configuration | II Git source of truth |
| 002 | Container Runtime + Outcall Integration | I, IV, X, Security 1/2/5/8 |
| 003 | Policy & Permissions Engine | IV, IX, Security 4/7 |
| 004 | Control Plane & Durable State | V, VI |
| 005 | Approvals & HITL | IX, IV |
| 006 | Observability, Audit, Cost Ledger | VI |
| 007 | Budget & Cost Governance | VI, IX |
| 008 | Agent Definition (SOUL/AGENTS/TOOLS) | II, X |
| 009 | Agent Self-Modification via PR | II, III, IX |
| 010 | Skills & Plugins (Lazy-Loaded) | X, III, Security 7 |
| 011 | Model Router & Provider Abstraction | X |
| 012 | Credential Broker | IV, Security 3 |
| 013 | Team Orchestration & Org Chart | XII |
| 014 | Inter-Agent Communication (Tickets) | V, VI |
| 015 | Workspace & Customer Codebase Mount | I, IV |
| 016 | End-to-End Software Agency Pipeline | XII (FLAGSHIP) |
| 017 | Multi-Project Juggling & Scheduling | V, VIII |
| 018 | Configuration Validation Before Merge | III, VIII |
| 019 | Agent Benchmarks & Evals | VII |
| 020 | Stability, Liveness, Recovery | VIII |
| 021 | CLI (AdonisJS Ace) | I surfaces |
| 022 | Web Dashboard | I, VI, IX |
| 023 | REST + WebSocket API | I, X |
| 024 | Plugin Marketplace (market.clawie.dev) | XI, III |
| 025 | Onboarding & Installation | III, VIII |
| 026 | Task Management Drivers & Flow Rules | III, V, IX, XII, Security 4 |

## Where this came from

- The deep-research report on autonomous AI agent frameworks (`/Users/mark/Downloads/deep-research-report.md`) — synthesized in `RESEARCH-NOTES.md`.
- The user's directive in the `/goal` invocation: fully autonomous software agency, AdonisJS surfaces, Outcall integration, git-everything with per-component rollback, continuous benchmarking, validated configurations, stability-first.
- Previous attempts under `../previous-attempts/` — `clawie-attempt-3` contained an earlier v1 constitution and 8 feature specs that this v2 supersedes (referenced where appropriate).
- The follow-up directive on task-management drivers (Linear/Jira/etc.) with workflow-enforcement rules — produced spec 026 and minor amendments to specs 013, 014, 016 plus `ARCHITECTURE.md` and `ROADMAP.md`.

## What's intentionally NOT here

- **Implementation code.** This is spec, not build.
- **`plan.md` / `tasks.md` per feature.** Added per feature when its delivery phase begins.
- **Gherkin features.** The previous attempt had these; v2 defers them until features enter delivery to avoid spec rot.
- **Dated milestones.** ROADMAP.md describes phase scope, not calendar dates.

## Status

| Item | Status |
|---|---|
| Foundation documents | Complete |
| Constitution | Ratified v2.0.0 |
| Product spec | Complete |
| Feature specs 001–026 | Complete (drafts; each carries a Draft status header) |
| `plan.md` per feature | Pending (per phase) |
| `tasks.md` per feature | Pending (per phase) |

This satisfies the directive: **spec the end goal in full; deliver in phases**.
