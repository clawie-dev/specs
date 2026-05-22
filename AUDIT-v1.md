# Clawie Spec Audit — v1

**Date:** 2026-05-22
**Scope:** Foundation docs + constitution + product spec + 26 feature specs
**Reviewer:** Self-audit (Claude). Findings should be reviewed by the project lead before any are closed.

Findings categorized A–E by severity. Each carries a recommendation. Items marked **(decision needed)** require operator input before they can be resolved — surfaced in the follow-up question set.

---

## A. Contradictions / direct conflicts (MUST fix)

### A1. Pipeline stage names diverge between specs 013 and 016
**Where:** 013 example `flows.yaml` uses `stage: code`; 016 canonical state machine has no `code` — it uses `scaffold`, `implement`, `code-review` as separate stages.
**Risk:** Validators and example configs won't match the real pipeline. New users will be confused.
**Fix:** Rewrite 013's example to use the canonical 016 stage list verbatim. Add a note in 013 that the stage names live in 016 and 013 references them.

### A2. "Operator token" (singular) vs "operator id" (implies multi-user)
**Where:** 023-FR-002 says single operator token. 005-FR-011, 009 audit, 026 force-override all reference "operator id" — implying multiple operators.
**Risk:** v1 promises a single-user surface but the audit/policy code already assumes IDs. Either the docs lie or the design is half-cooked.
**Fix:** Either (a) clarify "single operator, id = constant 'root' in v1" or (b) add a multi-operator spec before v1 ships. **(decision needed)**

### A3. Container reuse: ambiguous between 002 and 015
**Where:** 002-FR-001 says "no reuse of containers across tasks in v1". 015 `mount-rw` mode says "MUST acquire a durable workspace lock (one task at a time)" — sounds compatible (each task = fresh container, holds workspace lock for its duration), but the language could mislead a reader.
**Risk:** Implementer might interpret `mount-rw` as a long-running container.
**Fix:** Add a sentence in 015 explicitly: "lock is held only for the duration of the task's container; the container itself remains ephemeral."

---

## B. Ambiguities / unresolved decisions (SHOULD clarify)

### B1. Frontend framework under Inertia is not chosen
**Where:** 022 says "Inertia + Vue or React". Leaving both possible blocks scaffolding decisions in `clawie.dev` and the dashboard.
**Fix:** Pick one for v1. **(decision needed)**

### B2. Postgres vs SQLite default for v1
**Where:** 004-NFR-003 says "Postgres canonical; SQLite for single-host dev, tested but not load-bearing for production." 025-NFR-001 wants <10 min install on $20 VPS. Postgres adds significant install + ops complexity for indie operators.
**Risk:** The install promise and the production promise are in tension.
**Fix:** Either (a) make SQLite first-class for v1 single-host with documented scale limits, (b) ship managed Postgres in the docker-compose default and accept the heavier install, or (c) auto-promote (start SQLite, migrate to Postgres when scale demands). **(decision needed)**

### B3. Job runner choice
**Where:** 004 Open Question lists BullMQ+Redis vs pg-boss as undecided ("Lean: pg-boss").
**Fix:** Lock the decision so other specs can rely on it. **(decision needed; my recommendation = pg-boss)**

### B4. Marketplace skills mirror semantics
**Where:** 010-FR-006 says marketplace skills "MUST be mirrored as submodules under `skills-registry/`". But agent TOOLS.md references skills as `@version` strings. Mixed semantics — is the registry submodule the source-of-truth at install time, or a cache?
**Fix:** Clarify in 010 + 024: skills resolve via `skills-registry/` submodule (which pins the marketplace snapshot at agent-install-time). Allow `clawie skill upgrade` to bump the pin.

### B5. QA role mentioned but never specified
**Where:** 016 references "e2e-test owner: reviewer (or qa role if defined)". 013's default team has no QA agent. 026's example flow rules include `qa-agent`.
**Fix:** Either add `qa` to the default-agency role list (in 013) or remove "qa role" references from 016 and use `reviewer` for e2e.

### B6. "Post-launch-monitoring" is described as background but not specced
**Where:** 016 lists `post-launch-monitoring` as a pipeline stage ("background"). But background/recurring/cron-style work has no spec. How is "background" different from a one-shot task?
**Fix:** Either add a small scheduler/cron spec (currently in non-goals: deferred) or remove the background stage from v1 pipeline.

### B7. Outcall integration contract not specified
**Where:** Specs 002, 012, 026 mention Outcall heavily but no spec defines the contract: Is Outcall a sidecar with a documented HTTP API? A library Clawie embeds? A versioned protocol?
**Risk:** Implementers can't build without knowing the interface.
**Fix:** Add a thin integration spec or link to an Outcall spec in `clawie-dev/outcall`. **(decision needed)**

### B8. "Secrets-shaped string redaction at sink" (012-FR-010 edge case)
**Where:** 012 edge-cases mention "broker scans logs for known secret prefixes and redacts at sink (defense-in-depth)". This is a real feature but not specified in FR.
**Fix:** Promote to functional requirement with a defined scope.

---

## C. Arbitrary numbers that need justification or parametrization

### C1. Self-modification rate limit: "max 1 per day per agent"
**Where:** 009-FR-010.
**Concern:** Arbitrary. The user wants continuous learning. For agents that propose many small prompt tweaks, 1/day is restrictive. For ones that propose risky structural changes, 1/day might be too permissive.
**Fix:** Default 3/day; configurable per team; per-change-kind sub-limits (prompts/skills/SOUL/AGENTS/TOOLS).

### C2. Hot prompt cost: ≤200 tokens per declared skill
**Where:** 010-NFR-001.
**Concern:** Real-world capability summaries often need 300–500 tokens. ≤200 forces poor summaries.
**Fix:** Raise to ≤500 tokens per skill, with overall hot-prompt cap (≤8k tokens total) per agent.

### C3. Code↔review loop max-iter default 5
**Where:** 016-FR-008, 026-FR-017.
**Concern:** Production codebases often need 8–12 iterations; 5 forces premature escalation.
**Fix:** Default 8; configurable per stage.

### C4. Liveness ping interval 30s + 3 misses = 90s detection
**Where:** 020-FR-001/002.
**Concern:** A single LLM call can run 60–180s. Naive ping interval would mistake "model thinking" for "stuck".
**Fix:** Agent loop must emit pings *during* long calls (between provider stream events, or via a ticker thread). Spec must state ping is separate from "active work" — heartbeat continues even while model is thinking. Add 020-FR explicitly.

### C5. "Copy 1 GB repo ≤30s on commodity hardware"
**Where:** 015-NFR-001.
**Concern:** On a $20 VPS with a slow SSD, 30s for 1 GB is tight. Spinning disks would fail.
**Fix:** State "≤30s on commodity SSD" and add a degraded-mode note.

### C6. Cold container spawn p95 <5s on 1 CPU, 2 GB VPS
**Where:** 002-NFR-001.
**Concern:** Image pull alone can take 5s+ for a 500 MB image on a slow VPS. Number assumes image is pre-pulled.
**Fix:** Restate as "<5s when image cached locally; first-pull excluded".

### C7. Default 24h decision window for approvals
**Where:** 003-FR-009, 005-FR-005.
**Concern:** 24h is fine for sensitive ops but very long for routine ones. A high-velocity team blocks for a day on every minor approval.
**Fix:** Per-action-class default windows (sensitive: 24h, normal: 4h, routine: 30min). Operator overridable.

---

## D. Cross-cutting concerns missing or under-specified

### D1. No spec on scheduling / cron / recurring work
**Where:** Hinted in 016 ("background monitoring"), 020 (reconcile every 5min), Roadmap ("nightly evals"). No dedicated spec.
**Fix:** Either add `027-scheduler-and-recurring-work` or fold into 004 (control plane).

### D2. No spec on backups & disaster recovery
**Where:** 001 says git remote is optional backup. No DR for Postgres, audit log, secrets backend, marketplace artifacts.
**Fix:** Add backup/DR section to 004 or new spec `028-backup-and-disaster-recovery`. **(decision needed: scope for v1?)**

### D3. No spec on upgrades & schema migrations
**Where:** 025 mentions "reversible upgrades" with pre-snapshot. Mechanism not specced.
**Fix:** Add an upgrade spec or expand 025 with concrete mechanism (alembic-style migrations, snapshot format, rollback).

### D4. No multi-operator / RBAC spec
**Where:** See A2. v1 says "no user auth in v1" but audit + policy already imply multi-user.
**Fix:** Either lock single-operator model and remove the multi-user audit fields, or scope a minimal RBAC spec for v1.

### D5. No webhook handling spec (inbound + outbound)
**Where:** 023 mentions outbound webhooks; 026 mentions inbound webhooks from external task systems.
**Fix:** Add a small webhook spec covering: signing, replay protection, delivery retry semantics, ordering guarantees.

### D6. No spec for handling secrets in customer codebase (`.env` files etc.)
**Where:** 015 mentions `.clawieignore` and `sensitive files (e.g., .env, secrets.json) blocked by default`. But no explicit rule for read-protection. An agent reading `/workspace/.env` is a real risk.
**Fix:** Add to 015: default `.clawieignore` includes `.env*`, `*.pem`, `secrets/`, `.aws/`, `.ssh/`. The policy engine refuses reads even via `cat`. **(decision needed: how strict?)**

### D7. Time / timezone handling
**Where:** Implicit in 004 (clock skew), 006 (timestamps). No canonical timezone policy.
**Fix:** Lock to UTC everywhere internally; operator's locale for display only.

### D8. No spec for the "trial/sandbox" path
**Where:** 025-FR-008 mentions `clawie demo` (5-min toy project with free/tiny model). Not specified in detail.
**Fix:** Either expand 025-FR-008 or add a small demo spec.

---

## E. Risk / questionable options

### E1. `Outcall: disabled` mode allowed as default in 002
**Risk:** Operators tempted to use disabled mode "just to get started" and forget to enable. Even with a dashboard warning, this is a footgun.
**Recommendation:** Refuse `disabled` for any team whose flows.yaml references external (non-LLM) connectors. Force operator to consciously opt out per agent rather than per team.

### E2. Auto-merge for self-mod with positive benchmark delta
**Where:** 009-US-004 — "configure auto-merge for `prompts/` changes that improve benchmark".
**Risk:** Agents can game their own benchmark fixture (already flagged in 019-FR-012 as HIGH). Auto-merge with benchmark delta opens that attack vector.
**Recommendation:** Auto-merge requires (delta > threshold) AND (fixture unchanged) AND (changes only to whitelisted paths). Make whitelist explicit per team.

### E3. Unsigned skills via `--unsafe` flag
**Where:** 010-FR-003.
**Risk:** Devs install unsafe skill locally during testing, then publish referencing it as a dep, propagating risk.
**Recommendation:** Unsafe-installed skills can never be referenced by published skills (validator check).

### E4. Force-override percentage as "quality signal" (026-NFR-003)
**Where:** "5% force-overrides per month is healthy".
**Concern:** Arbitrary threshold; could mask legitimate workflow-mismatch.
**Recommendation:** Track per-team baseline; alert on relative spike rather than absolute %.

### E5. Eval worker pool — where does it live?
**Where:** 019-NFR-002 says "separate worker pool / hardware". Not specced how.
**Concern:** Operators may not have a second host. Single-host operators need clarity on resource sharing.
**Recommendation:** Allow eval pool to share host with production but in a separate Docker network + lower priority; large operators run separate hosts.

### E6. Plugin signature roots / trust model
**Where:** 010-FR-003 + 024-FR-002 say "signed". Signed by whom? Marketplace key? Author keys? Web of trust?
**Concern:** Without a clear trust model, signing is theater.
**Recommendation:** Marketplace-signed (gold) vs author-signed (silver) vs unsigned (dev). Operator config specifies minimum tier per agent.

### E7. `clone-branch` mode for workspace requires write credentials
**Where:** 015 + 012.
**Concern:** Granting an agent push rights to a customer's main repo is dangerous; even if PR-only, write rights leak.
**Recommendation:** `clone-branch` operates against a fork by default; PR opens from fork to upstream. Eliminates direct-push permission.

---

## F. Items that look good (no action)

- Constitution v2.0.0 — coherent, principles map cleanly to feature specs.
- Spec 001 monorepo layout — clean, internally consistent.
- Spec 004 task state machine — well-defined.
- Spec 005 approvals — properly layered.
- Spec 006 observability — comprehensive.
- Spec 020 stability — covers known failure modes from research.
- Spec 026 task-management drivers — well-scoped, properly cross-referenced.

---

## Summary

| Category | Count | Action |
|---|---|---|
| A. Contradictions | 3 | Fix before any phase begins |
| B. Ambiguities | 8 | Resolve via operator decision (4 of 8 need operator input) |
| C. Arbitrary numbers | 7 | Tune defaults with reasoning |
| D. Missing concerns | 8 | Add new sections / new specs |
| E. Risk items | 7 | Tighten constraints |

**Most pressing (decision-needed):** A2 (single vs multi-operator), B1 (frontend choice), B2 (SQLite vs Postgres default), B3 (job runner), B7 (Outcall contract), D2 (DR scope for v1), D6 (workspace secret protection strictness).

Once those are answered, the rest can be patched into the specs without further blocking input.
