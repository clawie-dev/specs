# Audit v1 — Resolutions

**Date:** 2026-05-22
**Companion to:** `AUDIT-v1.md`
**Effect:** All resolved findings have been patched into the relevant specs in the same session. This document records what changed and why.

Status legend: ✅ resolved · 🆕 new spec added · 📋 deferred (with rationale) · ⏭ explicitly out-of-scope

---

## A. Contradictions

| ID | Finding | Resolution | Status |
|---|---|---|---|
| A1 | Stage names diverge between 013 and 016 | Rewrote 013's `pipeline_stages` example to use the canonical 016 stage list verbatim (16 stages including post-launch-monitoring as recurring). Added a header comment in 013 stating "Stage names below MUST match spec 016". | ✅ |
| A2 | "operator token" vs "operator id" — single vs multi-user mismatch | Locked single-operator v1. `operator_id` is constant `"root"` in v1 audit. Multi-operator + RBAC moved to v2. Updated 023-FR-002, 005-FR-011 with explicit notes. | ✅ |
| A3 | Container reuse ambiguity between 002 and 015 | Clarified in 015-FR-003: lock held only for container lifetime; container itself remains ephemeral per spec 002. | ✅ |

## B. Ambiguities / unresolved decisions

| ID | Finding | Resolution | Status |
|---|---|---|---|
| B1 | Vue vs React under Inertia | **Vue 3** locked for v1 dashboard (matches AdonisJS Inertia ecosystem). React used for `clawie.dev` static site (Next.js App Router). Spec 022 and clawie.dev/README updated. | ✅ |
| B2 | Postgres vs SQLite default | **SQLite for v1 single-host** (canonical); Postgres for scale. Same schema via Lucid ORM. Migration tool: `clawie migrate-store --to postgres`. Updated 004-FR-001, 004-NFR-003, 025-NFR-001. | ✅ |
| B3 | Job runner — BullMQ vs pg-boss | **In-database queue.** On Postgres = pg-boss. On SQLite = lightweight tabled queue with row-locking. No Redis dependency. Added to 004 "Decision log". | ✅ |
| B4 | Marketplace skills mirror semantics | Clarified in 010-FR-006: `skills-registry/` submodule pins a marketplace snapshot at a known SHA; agent `name@version` references resolve against the mirror at install/boot. `clawie skill upgrade` bumps the pin via PR. | ✅ |
| B5 | QA role referenced but missing | Added `qa` to default-agency roles in 013. Removed "(or qa role if defined)" parenthetical from 016. Updated 013 `pipeline_stages` example to use `qa` as e2e-test owner. | ✅ |
| B6 | Post-launch-monitoring "background" not specced | Resolved by spec 027 (Scheduler & Recurring Work). 016 pipeline marks `post-launch-monitoring` as `recurring: true`, dispatched via 027 schedules. | ✅ + 🆕 |
| B7 | Outcall integration contract undefined | Added a full "Outcall integration contract" section to spec 002: REST API on `/v1/...` for control plane, transparent HTTP/HTTPS proxy for agent traffic, gRPC stream for audit, version compatibility gate. Sidecar mode is the production default. | ✅ |
| B8 | Log-sink redaction in 012 edge case | Promoted to 012-FR-011. Broker maintains registry of secret prefixes/shapes and redacts at write time across all sinks. | ✅ |

## C. Arbitrary numbers

| ID | Finding | Resolution | Status |
|---|---|---|---|
| C1 | Self-mod rate 1/day arbitrary | Raised to **3/day** with **per-change-kind sub-limits**: SOUL 1/week, AGENTS 1/day, TOOLS 1/day, prompts 3/day, skills unlimited. (009-FR-010) | ✅ |
| C2 | Hot prompt ≤200 tokens too tight | Raised to **≤500 tokens per skill**, total **≤8 000 tokens per agent**. Validator enforces. (010-NFR-001) | ✅ |
| C3 | Loop max-iter 5 too low | Raised default to **8**, per-stage overrides allowed (e.g., security-review stricter at 3). (016-FR-008, 026-FR-017) | ✅ |
| C4 | Liveness ping kills long LLM calls | Made explicit in 020-FR-001/002: heartbeat thread runs independently of active work. Per-intent overrides for long-running intents (e.g., deep-research = 5 missed = 150s). Added 020-FR-008a: base image ships heartbeat library that wraps long calls automatically. | ✅ |
| C5 | 1 GB repo copy ≤30s assumes fast disk | Clarified: ≤30s on commodity SSD; ≤120s on HDD/network-storage with `clawie doctor` warning. (015-NFR-001) | ✅ |
| C6 | Cold spawn p95 <5s assumes pre-pulled image | Clarified: <5s **with image cached locally**. Platform warm-pulls on install/upgrade. (002-NFR-001) | ✅ |
| C7 | 24h decision window too long for routine | Replaced with **per-class defaults**: sensitive 24h, normal 4h, routine 30min. Applied to both 003-FR-009 and 005-FR-005. | ✅ |

## D. Missing concerns

| ID | Finding | Resolution | Status |
|---|---|---|---|
| D1 | No scheduling spec | **New spec 027** — Scheduler & Recurring Work. Each agent declares own crons in its git repo; core runs `* * * * *` ticker. | 🆕 |
| D2 | No backup / DR | **New spec 028** — Backup & Disaster Recovery. Built-in scheduled backups to S3/R2, encrypted, with restore command. | 🆕 |
| D3 | Upgrade / migration mechanism | Captured in 028 restore (forward migrations on restore; refuse backward-incompatible without `--force`). Will expand in 025 detail when delivery starts. | 📋 |
| D4 | Multi-operator / RBAC | Locked: single-operator v1 (see A2). RBAC = v2 spec, not v1. | ⏭ v2 |
| D5 | Webhook handling | **Resolved via new spec 030** — unified: HMAC signing both directions, replay protection (nonce + timestamp window), retry with jitter+backoff, dead-letter queue + redrive, ordering per-resource, SSRF protection, ACK-then-handle for inbound. | ✅ |
| D6 | Workspace secret protection strictness | Added 015-FR-009a: **strict default-deny list** for workspace mounts. Customer `.clawieignore` is additive-only (narrower, never wider). Both reads and writes blocked. | ✅ |
| D7 | Time / timezone | Locked: UTC internally everywhere; locale display only. 027 scheduler explicitly UTC-only in v1. | ✅ |
| D8 | `clawie demo` path under-specified | Deferred to delivery phase 6 (025). Sample 5-min toy project with capped budget remains the goal. | 📋 |

## E. Risk / questionable options

| ID | Finding | Resolution | Status |
|---|---|---|---|
| E1 | `outcall: disabled` footgun | Added 002-FR-018: refused at boot when any non-LLM connector declared. LLM-only allowed with permanent dashboard warning. | ✅ |
| E2 | Auto-merge with benchmark delta gameable | Tightened 009-FR-004: auto-merge requires (delta > threshold) AND (fixture files unchanged) AND (whitelist paths). Refuses if same PR touches `benchmarks/`. | ✅ |
| E3 | Unsigned skill `--unsafe` supply-chain risk | 010-FR-003 updated: unsigned skills cannot be referenced as dependencies by published skills; validator enforces. | ✅ |
| E4 | 5% force-override absolute threshold arbitrary | 026-NFR-003 rewritten: per-team baseline with rolling window; alert on relative spike (e.g., 2× the 30-day baseline). | ✅ |
| E5 | Eval pool host not specced | 019-NFR-002 clarified: default shares host at nice 10, capped 30%. Operators with multi-host can assign dedicated. | ✅ |
| E6 | Signature trust model vague | New 010-FR-003a: trust tiers — `marketplace` (gold), `author` (silver), `unsigned` (dev). Per-team minimum tier. 024-FR-002 updated to counter-sign on approval. | ✅ |
| E7 | `clone-branch` push rights dangerous | 015-FR-005 updated: default operates against a fork, PRs cross-fork. Direct push to upstream requires explicit per-project config + approval gate. | ✅ |

---

## Summary

| Bucket | Count | Status |
|---|---|---|
| Contradictions (A) | 3 | All ✅ |
| Ambiguities (B) | 8 | All ✅ (one resolved by new spec) |
| Arbitrary numbers (C) | 7 | All ✅ |
| Missing concerns (D) | 8 | 5 ✅, 2 📋 (deferred), 1 ⏭ (v2) |
| Risk items (E) | 7 | All ✅ |
| **Total** | **33** | **31 resolved / 2 deferred / 1 v2-scoped** |

**New specs added:** 027 (scheduler with dual-mode crons), 028 (backup/DR), 029 (upgrades/migrations), 030 (webhooks), 031 (test discipline). Total feature spec count: 26 → 31. Constitution amended v2.0.0 → v2.1.0 to elevate test discipline.

**Documents touched:** constitution (no change), product spec (000), 002, 003, 004, 005, 009, 010, 012, 013, 015, 016, 019, 020, 022, 023, 024, 025, 026 plus ARCHITECTURE.md, ROADMAP.md, README.md, AUDIT-v1.md (existing), AUDIT-v1-resolutions.md (this file). New: 027 and 028. clawie.dev/README also updated to reflect React for static site.

**Deferred items** (need separate decisions or later delivery phases):
- D3 upgrade migrations (will expand in 025 plan phase)
- D5 dedicated webhook spec (current handling in 023 + 026 adequate for v1)
- D8 `clawie demo` detail (delivery-phase concern)
- D4 RBAC (explicitly v2)

The spec set is now internally consistent on the dimensions audited. Remaining items are honest-defer or v2-scope rather than overlooked.
