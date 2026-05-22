# Clawie — Implementation Phases

**Date:** 2026-05-22
**Status:** Plan-of-record for the implementation track
**Companion to:** `ROADMAP.md` (spec delivery plan), `speckit/` (spec set)

`ROADMAP.md` describes when each spec is "delivered" in scope. **This document** describes the order in which a working framework is **built**, version by version, with each phase shipping a tagged release.

## Guiding principles

1. **Vertical slices, not horizontal layers.** Each phase touches every architectural layer, but only does one thing through them. Get task → audit → CLI working with the simplest possible intent before adding Docker, before adding LLMs, before adding policy. This keeps every release end-to-end working.
2. **Tests-first discipline (spec 031).** No phase exits without its mirror tests + coverage gate green.
3. **Ship → check → iterate.** Each phase ships a release tag. CI is the source of truth on whether the release is good. Failures (in our control) trigger patch versions.
4. **No phase skips a layer.** Every phase has: code + tests + docs + CI gates.
5. **Cumulative, not destructive.** Phase N+1 extends Phase N; it doesn't rewrite.

---

## Phase 1 — "Hello Task" → v0.1.0

**Goal:** Prove the durable task lifecycle works end-to-end through the simplest possible code path. **No LLMs. No Docker. No Outcall. No agent files yet.** Just: a task is created, claimed, executed by a built-in intent handler, completed, and audit-logged. All durable in SQLite. All testable.

The smallest possible vertical slice.

**Ships in `clawie-dev/clawie`:**
- AdonisJS app retained from scaffold (auth scaffold kept — fine for Phase 1).
- `app/models/task.ts` — Lucid model with state machine columns.
- `app/models/audit_event.ts` — append-only audit table.
- `database/migrations/create_tasks_table.ts`, `create_audit_events_table.ts`.
- `app/services/task_state_machine.ts` — transition logic with optimistic locking.
- `app/services/intents/registry.ts` — intent handler registry.
- `app/services/intents/echo.ts` — built-in `echo` intent: returns `"hello: <payload>"`.
- `app/services/audit_logger.ts` — append-only writes with hash chain seed (full chain in later phase).
- `commands/task_run.ts` — Ace command: `node ace task:run --intent echo --payload "world"`.
- `app/controllers/tasks_controller.ts` — `POST /v1/tasks`, `GET /v1/tasks/:id`, `GET /v1/tasks`.
- Tests under `tests/unit/` and `tests/integration/` per spec 031 mirror convention.
- `scripts/check-mirror-tests.ts` — the CI gate.
- `.github/workflows/ci.yml` — lint + typecheck + test + mirror-check.
- README updates with quick-start.
- `CHANGELOG.md` initial entry.
- Tag `v0.1.0`.

**Acceptance:**
- `node ace task:run --intent echo --payload "world"` prints `hello: world`, creates a task row in SQLite, emits two audit events (created → completed).
- `GET /v1/tasks/<id>` returns the task as JSON.
- All tests pass; mirror-check passes; coverage gates pass.
- CI green on the push.

**Exit:** v0.1.0 tagged, pushed, GitHub Actions green.

---

## Phase 2 — "Task in a Container" → v0.2.0

**Goal:** the intent runs **inside a Docker container** based on the `clawie/agent-runtime` image, not in the control-plane process.

**Ships:**
- `clawie-dev/agent-runtime`: minimal Dockerfile + entrypoint that reads task payload from env, runs the intent, returns the result via stdout JSON.
- In `clawie/`: `app/services/container_spawner.ts`, `app/services/intents/dispatch.ts` (sends to container instead of in-process).
- Lifecycle: spawn → wait for exit → capture stdout → reap. Resource limits applied.
- Tests: spawn-claim-exit smoke test; mock Docker for unit tests; real Docker for integration.
- Tag `v0.2.0`.

**Acceptance:** same `task:run` command now executes inside Docker; audit captures container lifecycle events.

---

## Phase 3 — "Real LLM" → v0.3.0

**Goal:** an intent that uses an LLM (via credential broker stub for now).

**Ships:**
- Model router with Anthropic + OpenAI adapters.
- Credential broker stub (env-injected dev backend; real broker comes with spec 012 later).
- New intent: `chat` — prompts the model with the payload, returns the completion.
- Cost ledger entries per call.
- Tag `v0.3.0`.

---

## Phase 4 — "Policy + Approval" → v0.4.0

**Goal:** default-deny semantics enforced. Approval queue surfaces in CLI + API.

**Ships:** policy engine, approval model, decision windows, auto-rule generation. Tag `v0.4.0`.

---

## Phase 5 — "Outcall" → v0.5.0

**Goal:** containers egress only through Outcall (sidecar mode).

**Ships:** Outcall sidecar wiring in compose, credential broker fully integrated, egress rules per-team. Tag `v0.5.0`.

---

## Phase 6 — "Dashboard MVP" → v0.6.0

**Goal:** React + Inertia dashboard renders tasks, approvals, audit. Approve/deny from the UI.

**Ships:** Inertia pages, basic auth check, real-time updates via WebSocket. Tag `v0.6.0`.

---

## Phase 7 — "Agent Files + Self-Mod" → v0.7.0

**Goal:** agents are defined in git repos (SOUL/AGENTS/TOOLS); self-modifications surface as PRs in dashboard.

**Ships:** spec 008 + spec 009. Tag `v0.7.0`.

---

## Phase 8 — "Teams + Multi-Agent" → v0.8.0

**Goal:** teams orchestrate flows; agents communicate via tickets.

**Ships:** spec 013 + spec 014. Tag `v0.8.0`.

---

## Phase 9 — "Scheduler + Crons" → v0.9.0

**Goal:** per-agent crons (agent + script kinds) + core ticker.

**Ships:** spec 027 + cost tagging (spec 006/007 source dimension). Tag `v0.9.0`.

---

## Phase 10 — "v1 release-ready" → v1.0.0

**Goal:** ship-grade Clawie.

**Ships:**
- Spec 016 — full pipeline state machine.
- Spec 026 — Linear/Jira drivers.
- Spec 028 — Backup/DR.
- Spec 029 — Upgrades.
- Spec 030 — Webhooks.
- Spec 024 — Marketplace v0.
- `default-agency` starter pack.
- `clawie.dev` landing site.
- Full docs.
- Tag `v1.0.0`.

---

## Patch / bugfix protocol

Within a phase:
- Bug discovered → patch version (e.g., v0.1.0 → v0.1.1)
- CI failure (in our control) → fix → patch version
- Failures NOT in our control (Cloudflare deploy access, external service outages) → documented in CHANGELOG.md, not auto-fixed
- Each patch: commit, tag, push, sleep + check Actions, iterate until green

## Cross-cutting (every phase, mandatory)

- **Spec 031 — Test Discipline & Coverage**: applies from Phase 1 onward. Mirror-file convention enforced by CI gate. Per-area coverage targets. PR cannot merge without tests for new code.
- **Spec 018 — Validation gate**: configs validated before merge.
- **Documentation**: each phase updates `clawie-dev/docs` with what's now possible.
- **Foundation docs**: each phase updates the relevant phase doc in `clawie-dev/clawie` repo (PHASE-N-NOTES.md).
- **Repos updated together**: when a phase touches a sibling repo (agent-runtime, schemas, sdk-typescript, default-agency, outcall-presets), tag those repos with matching minor versions.

## Phase exit checklist

Before tagging a phase:
1. ✅ All planned features implemented.
2. ✅ Tests pass locally + in CI.
3. ✅ Coverage gates green.
4. ✅ Mirror-file convention satisfied.
5. ✅ Documentation updated.
6. ✅ CHANGELOG.md updated.
7. ✅ README quick-start verified manually.
8. ✅ `git tag v<X>.<Y>.0 && git push --tags`.
9. ✅ GitHub Actions green (post-tag).
10. ✅ Sibling repos (if touched) tagged consistently.
