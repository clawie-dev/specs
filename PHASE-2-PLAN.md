# Phase 2 — "Task in a Container" Plan

**Target version:** `clawie v0.2.0` + `agent-runtime v0.2.0`
**Goal (one sentence):** an intent executes inside an ephemeral Docker container instead of in-process, with the same audit + state machine + CLI surface as Phase 1.

**Status:** Shipped. Phase 2 landed as `clawie v0.2.1` (container execution via `ContainerSpawner` + `clawie/agent-runtime`) and remains in production through v1.0.0. This document is retained as historical planning context — operator-facing references should point to `PHASES.md`, the v1.0 docs, and the [v0.2.1 release notes](https://github.com/clawie-dev/clawie/releases/tag/v0.2.1).

**Publish strategy decided:** image stays **local-only** for now (built from the `clawie-dev/agent-runtime` repo via `docker build`); we defer pushing to GHCR / Docker Hub until v0.5.x, when more agents start needing it. Tests gate on local Docker availability.

---

## What Phase 2 ships

### In `clawie-dev/agent-runtime`

| Deliverable | Notes |
|---|---|
| `Dockerfile` | Node 24 alpine; non-root user (uid 1000); `WORKDIR /agent`; minimal entrypoint script |
| `src/entrypoint.ts` | Reads task spec from stdin JSON, dispatches to a built-in handler set (initially just `echo`), writes result JSON to stdout, exits 0/1 |
| `src/handlers/echo.ts` | Same semantics as Phase 1's in-process echo; ported into the container |
| `tests/entrypoint.test.ts` | Run the entrypoint subprocess; assert stdout is correct JSON |
| `Makefile` + `scripts/build.sh` | `make build` → `docker build -t clawie/agent-runtime:dev .`; `make test` runs the entrypoint tests |
| `README.md` | "How to build locally", "What the image guarantees", "Future GHCR publish path" |
| Tag `v0.2.0` | local-only at this point; no registry push |

### In `clawie-dev/clawie`

| Deliverable | Notes |
|---|---|
| `app/services/container_spawner.ts` | Interface + Docker implementation. Spawn via `docker run --rm -i clawie/agent-runtime:dev <task-spec>`. Capture stdout, parse JSON result. |
| `app/services/intents/containerized_echo.ts` | New intent that delegates to `ContainerSpawner` instead of running in-process |
| `app/services/task_executor.ts` (modified) | Branches: if intent metadata says `runtime: container`, use spawner; otherwise in-process. Phase 1 in-process path unchanged. |
| `database/migrations/add_runtime_to_tasks.ts` | Adds nullable `runtime` column (`inproc` default; `container` for Phase 2 intents) |
| `commands/task_run.ts` | New `--runtime` flag (`inproc` default, `container` for Phase 2 intents) |
| `tests/unit/services/container_spawner.test.ts` | Mocked `child_process.spawn` — no real Docker |
| `tests/integration/container_lifecycle.test.ts` | **Gated on `docker info` succeeding.** Builds the local image, runs a real container, asserts result. Skipped when Docker is unavailable. |
| `scripts/docker-available.ts` | Returns exit 0 if Docker daemon is up; CI uses this to decide whether to run integration. |
| `.github/workflows/ci.yml` (modified) | Adds an optional `docker` job that runs when Docker is set up; uses `docker/setup-buildx-action` |
| README + CHANGELOG | Document `v0.2.0` capabilities |
| Tag `v0.2.0` | |

### In `clawie-dev/docs`

| Deliverable | Notes |
|---|---|
| `concepts/intents-and-runtimes.md` | Explain `inproc` vs `container`; how to choose |
| `reference/cli.md` (update) | Document `--runtime` flag |
| `install/quick-start.md` (update) | "Try a containerized intent" section, gated on Docker installed |

### In `clawie-dev/clawie.dev`

| Deliverable | Notes |
|---|---|
| `content/landing.md` (update) | Roadmap row v0.2.0 marked ✅ |

---

## Concrete task breakdown

In dependency order. Each is independently testable.

1. **agent-runtime: scaffold Dockerfile + entrypoint** (~45 min real-time)
2. **agent-runtime: echo handler + tests** (~30 min)
3. **agent-runtime: Makefile + build script + README** (~20 min)
4. **agent-runtime: tag v0.2.0** (~5 min)
5. **clawie: ContainerSpawner interface + Docker implementation** (~60 min)
6. **clawie: containerized_echo intent + task_executor branch** (~30 min)
7. **clawie: migration for `runtime` column** (~15 min)
8. **clawie: CLI `--runtime` flag** (~15 min)
9. **clawie: unit tests with mocked spawner** (~45 min)
10. **clawie: integration tests gated on Docker** (~60 min)
11. **clawie: CI workflow extension (separate docker job)** (~30 min)
12. **docs updates** (~30 min)
13. **clawie: tag v0.2.0 + verify CI green** (~15 min) — likely 1-2 patch tags

**Estimate:** ~7 hours of focused work in a single agent session, assuming no Docker oddities. Realistic: 1.5-2 sessions including CI iteration.

---

## Risks

| Risk | Mitigation |
|---|---|
| Docker daemon flaky in CI runners | Make integration tests opt-in via env flag; mock-only path always green |
| Image pull races between parallel CI runs | Use deterministic image tag (sha-based); cache via setup-buildx |
| Container hangs (no liveness yet — that's Phase 5 / spec 020) | Wall-clock timeout via `docker run --stop-timeout`; surface as task `cause=timeout` |
| Stdin/stdout JSON framing | Lock to single JSON object on stdin, single JSON object on stdout. Multi-line JSON not yet supported. |
| AdonisJS test runner already flaky around connection pool | Phase 1 demonstrated this is manageable; Phase 2 doesn't make it worse |
| Image size creep | Stick to alpine + only Node 24 + entrypoint. Goal: <100 MB compressed. |

---

## Acceptance criteria

Phase 2 ships when:

1. `clawie task:run --intent containerized-echo --payload '"hi"' --runtime container` returns `{"message":"hello: hi"}` produced by a Docker container.
2. The audit trail captures the container lifecycle (`container.spawned`, `container.exited`) alongside the existing task transitions.
3. Killing the container mid-task marks the task `failed` with `cause=container_died`. No orphaned containers after reap.
4. Unit tests pass without Docker installed.
5. Integration tests pass with Docker installed; skipped cleanly without.
6. CI on `main` green for both jobs (lint+typecheck+unit always; docker-integration when env permits).
7. Existing v0.1.x echo path (in-process) still works unchanged — no regressions.

---

## Out of scope for Phase 2 (deferred)

- Multiple language overlays (Python, PHP) — Phase 2 ships Node 24 only.
- LLM client inside the container — Phase 3.
- Outcall egress isolation — Phase 5.
- Credential broker — Phase 5.
- GHCR / Docker Hub push pipeline — Phase 5 (v0.5.x).
- Health-checked liveness ping during long-running intents — Phase 5+ per spec 020.
- runc/gVisor/Kata variants — later phase.
- `docker compose` for the full platform (server + db + agent-runtime + outcall) — Phase 5.

---

## Greenlight checklist

Before starting Phase 2 implementation:

- [ ] Confirm scope above is correct (cut anything? add anything?)
- [ ] Confirm session capacity / availability for ~1.5–2 sessions of work
- [ ] Confirm we're OK with local-only image until v0.5.x
- [ ] Confirm Docker dev requirement on the operator's machine for end-to-end Phase 2 testing

Once those are green, the per-task breakdown above is the execution order.
