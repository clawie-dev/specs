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

## Outcall integration interleave

Phases 5+ interleave **numbered phases** (ship a Clawie tag) with **lettered phases** (Outcall integration). Two non-negotiable principles:

1. **Outcall does not depend on Clawie.** Outcall is a general-purpose host firewall for Docker agent containers. Any change Clawie wants in Outcall must stand on its own merit for non-Clawie consumers; Clawie-specific shape lives in Clawie's adapter, not in Outcall.
2. **Clawie ships without Outcall.** Phase 5 introduces an `EgressProvider` abstraction; the default ("null") provider does nothing. Lettered phases add the `outcall` provider opt-in. Every numbered phase remains shippable with `EgressProvider: null` even if the lettered work hasn't landed.

| Phase | Track | Outputs |
|---|---|---|
| 5 | Clawie | `EgressProvider` interface + null provider. Walks back the v0.5.0 sidecar hack. |
| **5a** | Outcall (research + PRs) | Hands-on audit of Outcall vs Clawie's workloads. Bypass/payload suites run; gaps logged; upstream PRs filed where appropriate. |
| **5b** | Clawie + Outcall | `OutcallEgressProvider` implementation. Real connection to `/run/outcall/host.sock`. macOS/CI keep the null provider. |
| 6 | Clawie | Dashboard MVP. |
| **6a** | Outcall + Clawie | Dashboard consumes Outcall's `/api/v1/rules`, recent blocks, per-agent traffic. Upstream PRs for any missing endpoints. |
| 7 | Clawie | Agent files + self-mod. |
| **7a** | Outcall + Clawie | Self-mod permission checks go through `outcall-agent`'s `permissions check` API. |
| 8 | Clawie | Teams + multi-agent. Per-team Outcall networks. |
| **8a** | Outcall + Clawie | Per-team rule scoping ergonomics. |
| 9 | Clawie | Scheduler + crons. |
| **9a** | Outcall | Stress test Outcall under Clawie's planned scheduler load. PR perf fixes upstream. |
| 10 | Clawie | v1.0 release-ready. |
| **10a** | Outcall + Clawie | Joint v1 alignment: Outcall v0.2 (signed artifacts, tighter seccomp) aligned with Clawie v1.0 GA. |

---

## Phase 5 — "Pluggable Egress" → v0.5.1

**Goal:** Introduce the `EgressProvider` abstraction. Walk back the v0.5.0 sidecar work (a parallel-implementation mistake that re-built Outcall instead of consuming it). Keep Clawie shippable with zero egress isolation by default.

**Context:** v0.5.0 shipped a Node.js sidecar that proxied + injected credentials — that's not what Outcall is. Outcall is a Rust host daemon at `Outcall-dev/root`, owns nftables + DNS filter + L7 proxy, is consumed via Docker network + `HTTPS_PROXY` env + `/run/outcall/host.sock`. Clawie is a consumer, not a re-implementer.

**Ships in `clawie-dev/clawie` (v0.5.1):**
- `app/services/egress/provider.ts` — `EgressProvider` interface (single method: `wrapSpawnRequest(req) → req`). Null provider returns the request unchanged.
- Remove `SidecarSpec`, `network: 'sidecar'`, sidecar lifecycle code from `ContainerSpawner`.
- `chat` intent reverts to `network: 'bridge'` + credentials in agent env (Phase 3 model — the stopgap until 5b lands).
- Bump image pin to `clawie/agent-runtime:0.4.1`.
- Tests for the new interface + null provider. Sidecar tests removed.

**Ships in `clawie-dev/agent-runtime` (v0.4.1):**
- Chat handler reverts to Phase 3 direct-call behavior (`https://api.anthropic.com/...` + `x-api-key` from env). Remove `OUTCALL_URL` routing.

**Ships in `clawie-dev/outcall-presets`:**
- README link fix (`clawie-dev/outcall` → `Outcall-dev/outcall`).

**Acceptance:**
- `task:run --intent chat ...` works exactly as it did in v0.3.0.
- No reference to `clawie/outcall:*` anywhere in Clawie's runtime or docs.
- Tests green; CI green; agent-runtime v0.4.1 tagged; clawie v0.5.1 tagged.

---

## Phase 5a — "Outcall integration audit" → no Clawie tag

**Goal:** Understand what Outcall actually does today against Clawie's planned workloads. Identify upstream gaps before writing the connector.

**Repos touched:** `Outcall-dev/outcall`, `Outcall-dev/docs`, `Outcall-dev/specs` (PRs only — no Outcall-version bump driven from Clawie's side).

**Ships:**
- Linux integration matrix (run on a Linux host or CI runner): boot `outcalld`, create `outcall-clawie` network, spawn a fake-Clawie agent, run Outcall's `scripts/test-bypass.sh` + `scripts/test-payloads.sh`, capture results.
- `clawie/docs/integrations/outcall.md` — gap matrix (what Outcall does, what Clawie wants, what's missing).
- Upstream PRs for any blocker. Examples that *might* surface: agent name pattern-matching in rule CEL, batched rule-reload, structured allow-then-reblock for ephemeral agents.

**Exit criteria:**
- Gap matrix landed in Clawie docs.
- Each gap has a decision: (a) implement upstream and wait, (b) work around in Clawie adapter, (c) accept as limitation.
- No Clawie release tag (research phase).

---

## Phase 5b — "Outcall connector" → v0.5.2

**Goal:** Real `OutcallEgressProvider` that talks to a running `outcalld` daemon. Operators who run Outcall get default-deny network isolation for free; operators who don't run Outcall get exactly the v0.5.1 behavior.

**Ships in `clawie/`:**
- `app/services/egress/outcall_provider.ts` — Unix-socket client for `/run/outcall/host.sock`. On boot: probes `outcall bridge status`; on intent registration: drops a rule file at `/etc/outcall/rules.d/clawie-<intent>.yaml` (or POSTs to `/api/v1/rules/reload`).
- `ContainerSpawner.wrapSpawnRequest` calls the active provider, which adds: `--network outcall-clawie`, `--dns 10.200.0.1`, `HTTP(S)_PROXY` env, optional `agent.sock` mount, container name → `clawie-<intent>-<short-id>` so Outcall's `agent.name` resolves to `clawie-<intent>`.
- Config: `CLAWIE_EGRESS=outcall|null` (default `null`).
- macOS/CI: provider self-detects daemon absence; falls back to null with a warning.
- Tests: unit (mocked socket), Linux-only integration gated on `OUTCALL_INTEGRATION=1` env.

**Ships in `clawie-dev/outcall-presets`:**
- `presets/clawie-default.yaml` — Outcall rule pack for Clawie agents (allow LLM providers; deny everything else). Generic, not Clawie-specific identity-wise.

**Acceptance:**
- `CLAWIE_EGRESS=outcall task:run --intent chat ...` actually goes through Outcall's L7 proxy.
- Clawie boot logs the resolved provider (`null` vs `outcall`).
- Outcall blocks an injected exfil attempt in the integration test.
- Linux integration tests gated on env flag — green when flag is set, skipped otherwise.

---

## Phase 6 — "Dashboard MVP" → v0.6.0

**Goal:** React + Inertia dashboard renders tasks, approvals, audit. Approve/deny from the UI.

**Ships:** Inertia pages, basic auth check, real-time updates via WebSocket. Tag `v0.6.0`.

---

## Phase 6a — "Outcall dashboard handoff" → v0.6.1

**Goal:** Clawie dashboard surfaces Outcall enforcement when the operator opted into the Outcall provider.

**Ships in `clawie/`:**
- Dashboard tab "Egress" — lists active rules, recent blocks (with reason: DNS / nftables / L7 proxy), per-agent traffic counters. Read-only via `outcall-api`.
- WebSocket subscription to Outcall's block events (if upstream supports; otherwise polling).

**Ships in `Outcall-dev/outcall` (PR-only, no Clawie-driven version bump):**
- Whatever read endpoints are missing from `outcall-api` to back the dashboard. Justified independently (any UI consumer benefits).

**Acceptance:** with `CLAWIE_EGRESS=outcall`, dashboard shows live rule + block data. With `null`, the tab renders an empty state.

---

## Phase 7 — "Agent Files + Self-Mod" → v0.7.0

**Goal:** agents are defined in git repos (SOUL/AGENTS/TOOLS); self-modifications surface as PRs in dashboard.

**Ships:** spec 008 + spec 009. Tag `v0.7.0`.

---

## Phase 7a — "Outcall agent-shim integration" → v0.7.1

**Goal:** Self-mod actions (write a file, run a tool, exec a shell) are checked against Outcall's `permissions check` API before they happen. Agents that aren't in an Outcall network are unchanged.

**Ships in `clawie/`:**
- Self-mod hook in the agent-runtime that calls `outcall-agent permissions check` over `/run/outcall/agent.sock` for any file write / tool exec / PR open. Decision is allow/deny by rule CEL evaluating `run.tool`, `run.args`, `run.cwd`.
- Dashboard surfaces denials with the matching rule id.

**Ships in `Outcall-dev/outcall` (PR-only):**
- Anything missing from the shim's `permissions check` to support tool-exec semantics (e.g., a `run.purpose` field if needed). Defended independently.

**Acceptance:** an agent attempting to `git push --force` against a deny rule sees the denial in the audit log AND in the dashboard.

---

## Phase 8 — "Teams + Multi-Agent" → v0.8.0

**Goal:** teams orchestrate flows; agents communicate via tickets.

**Ships:** spec 013 + spec 014. When `CLAWIE_EGRESS=outcall`, each team gets a dedicated Outcall network (`outcall-clawie-team-<slug>`) so L2 isolation between teams is structural, not policy-enforced. Tag `v0.8.0`.

---

## Phase 8a — "Per-team rule scoping ergonomics" → v0.8.1

**Goal:** managing N teams × M rules without explosion. CLI + REST helpers to scope rules to a team identity.

**Ships in `clawie/`:**
- `outcall:sync --team <slug>` command — diff and apply Clawie's per-team rule shape to Outcall's `rules.d`.
- Dashboard view: rules per team, conflicts surfaced.

**Ships in `Outcall-dev/outcall` (PR-only):**
- Whatever's needed to make per-team scoping cleaner. Justified independently (any multi-tenant operator benefits).

---

## Phase 9 — "Scheduler + Crons" → v0.9.0

**Goal:** per-agent crons (agent + script kinds) + core ticker.

**Ships:** spec 027 + cost tagging (spec 006/007 source dimension). Tag `v0.9.0`.

---

## Phase 9a — "Outcall scale & hardening under scheduler load" → no Clawie tag

**Goal:** verify Outcall holds up when Clawie's scheduler is producing concurrent agent spawns. Address regressions upstream.

**Ships:**
- Stress harness (Clawie side): spawns N agents simultaneously through Outcall, measures p50/p95/p99 latency on rule eval + proxy connect.
- Findings written to `clawie/docs/integrations/outcall-perf.md`.
- Upstream Outcall PRs for any perf regression hit (memory growth in dynamic rules, conntrack scaling, etc.). Defended independently.

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
- Full docs (including `integrations/outcall.md` finalized).
- Tag `v1.0.0`. Clawie pins `outcall >= <stable version from 10a>` when the Outcall provider is enabled.

---

## Phase 10a — "Joint v1 alignment with Outcall" → no Clawie tag (joint release notes)

**Goal:** Coordinate Outcall v0.2 GA with Clawie v1.0 GA so operators get one coherent stack story.

**Outcall side (independent track, NOT Clawie-version-bumped):**
- Signed release artifacts (Sigstore/SBOM).
- Tighter `outcall-agent` seccomp profile.
- Anything else Outcall needs for its own v0.2.

**Clawie side:**
- Update `OUTCALL_PROVIDER` min-version pin in clawie's `package.json` and docs.
- Joint release notes published on `clawie.dev` + Outcall website.

**Note:** Outcall's v0.2 timeline is set by Outcall maintainers, not Clawie. If Outcall slips, Clawie v1.0 ships with `EgressProvider: null` documented as the default and Outcall integration as an opt-in.

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
