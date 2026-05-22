# 031 — Test Discipline & Coverage

**Status:** Draft
**Constitutional anchor:** Principle III (Validated before merged), VIII (Stability is a P0 feature)
**Depends on:** none (cuts across every framework spec)
**Required by:** every code-shipping feature

## Purpose

The Clawie framework MUST be very thoroughly tested. The user's explicit directive: "we dont want bugs like OpenClaw has on every new update". Spec 031 sets the **non-negotiable engineering rules** for how Clawie's code is tested — mirror-file convention, coverage targets, test types and where each applies, PR gates, what gets fixtured vs. what hits real infrastructure.

This spec governs the **framework's code quality**, distinct from spec 019 which governs **agent quality** (eval fixtures). 019 is about whether the agent gets smarter; 031 is about whether the framework breaks.

## Why this matters

The OpenClaw "breaks on every update" pattern is a structural failure: it's not a single bug, it's a tested-discipline gap. The fix is not heroics during releases — it's a non-negotiable test surface that prevents broken code from ever reaching main.

## File layout convention (MANDATORY)

For every production code file in `clawie/app/` or any other source root, a corresponding test file MUST exist at the mirrored path under `clawie/tests/`:

```
clawie/
├── app/
│   ├── controllers/
│   │   ├── tasks_controller.ts
│   │   └── approvals_controller.ts
│   ├── services/
│   │   ├── policy_engine.ts
│   │   ├── credential_broker.ts
│   │   └── scheduler/
│   │       ├── ticker.ts
│   │       └── dispatcher.ts
│   └── models/
│       ├── task.ts
│       └── approval.ts
├── tests/
│   ├── unit/
│   │   ├── controllers/
│   │   │   ├── tasks_controller.test.ts          ← mirrors app/controllers/tasks_controller.ts
│   │   │   └── approvals_controller.test.ts
│   │   ├── services/
│   │   │   ├── policy_engine.test.ts
│   │   │   ├── credential_broker.test.ts
│   │   │   └── scheduler/
│   │   │       ├── ticker.test.ts
│   │   │       └── dispatcher.test.ts
│   │   └── models/
│   │       ├── task.test.ts
│   │       └── approval.test.ts
│   ├── integration/                              ← real DB, real Docker, real Outcall sidecar
│   │   ├── task_lifecycle.test.ts
│   │   ├── approval_flow.test.ts
│   │   └── scheduler_dispatch.test.ts
│   ├── e2e/                                      ← Playwright over CLI + Dashboard + API
│   │   ├── cli/
│   │   │   ├── project_new.test.ts
│   │   │   └── agent_rollback.test.ts
│   │   ├── dashboard/
│   │   │   ├── approval_decision.test.ts
│   │   │   └── self_mod_pr_review.test.ts
│   │   └── api/
│   │       ├── projects.test.ts
│   │       └── webhooks.test.ts
│   ├── smoke/                                    ← per-agent spawn-claim-exit
│   │   └── default_agency/
│   │       ├── researcher.test.ts
│   │       └── ...
│   └── chaos/                                    ← kill -9 + recovery scenarios
│       ├── api_crash_mid_task.test.ts
│       └── container_oom_during_run.test.ts
```

**Discoverability rule:** a file `app/<path>/<name>.ts` MUST have a unit test at `tests/unit/<path>/<name>.test.ts`. This makes "which files are covered" answerable by simple `ls`. Files without a test are visible by absence.

## Test pyramid

| Tier | Tooling | What it covers | When it runs | Speed |
|---|---|---|---|---|
| **Unit** | Japa (AdonisJS native) | Single function/class/module; deps mocked at module boundary | Every save (watch), every commit (pre-commit hook), every PR | <1 ms each, <30 s suite |
| **Integration** | Japa + real services in Docker | Multi-component flows: API → service → DB; agent runtime → Outcall sidecar → broker | Every PR, nightly full sweep | <5 s each, <10 min suite |
| **E2E** | Playwright + Japa runner | Through real surfaces: CLI commands, Dashboard interactions, REST/WS API calls | Every PR, on release branches | <30 s each, <30 min suite |
| **Smoke** | Japa + per-agent container | Each shipped agent spawns, claims a no-op task, exits cleanly. One per agent. | Every PR touching agent-runtime or default-agency; nightly | <2 min per agent |
| **Chaos** | Japa + scripted failure injection | Kill -9 mid-task, OOM during run, Outcall sidecar crash, DB unavailability — verify recovery semantics from spec 020 | Nightly, before release | <10 min suite |
| **Benchmark / load** | k6 or custom | Sustained-load characterization (NFR validation) | Weekly, before release | hours |

## Functional requirements

### Layout & discoverability

| ID | Requirement |
|---|---|
| 031-FR-001 | Every production code file in any source root MUST have a mirrored unit-test file under `tests/unit/`. The mirroring convention is `app/foo/bar.ts → tests/unit/foo/bar.test.ts`. |
| 031-FR-002 | A CI check (`scripts/check-mirror-tests.ts`) MUST run on every PR, listing files without a mirror test. New code without a mirror test FAILS the gate. Exemptions require an `// @no-test: <reason>` comment on the first line of the source file and an audit log entry on merge. |
| 031-FR-003 | Test file extension convention: `.test.ts` (or `.spec.ts` for legacy compatibility — pick one repo-wide, default `.test.ts`). |

### Coverage targets

| ID | Requirement |
|---|---|
| 031-FR-010 | **Control plane code** (`app/services/`, `app/models/`, policy engine, scheduler, dispatcher, broker, audit, ledger): line coverage ≥ **90%**, branch coverage ≥ **85%**. |
| 031-FR-011 | **Controllers / surfaces** (`app/controllers/`, API handlers): line ≥ **85%**, branch ≥ **80%**. |
| 031-FR-012 | **Plumbing / utilities**: ≥ **70%** line. |
| 031-FR-013 | **Critical path code** (anything touching audit, secrets, policy decisions, container lifecycle): **100%** line + branch. Listed explicitly in `tests/CRITICAL_PATHS.md`. |
| 031-FR-014 | Coverage regression of >1% on a PR blocks merge. The PR author may override with `--accept-coverage-regression` (CI gate flag) + audit-logged reason. |

### Test types — what hits real services

| ID | Requirement |
|---|---|
| 031-FR-020 | Unit tests MUST NOT touch real DB, real Docker, real Outcall, or real LLM. Module-boundary mocks only. |
| 031-FR-021 | Integration tests MUST use real services (real SQLite/Postgres, real Docker, real Outcall sidecar, real broker). Spun up via `docker-compose.test.yml`. Single LLM is the only mock-allowed (a deterministic stub provider for reproducibility). |
| 031-FR-022 | E2E tests via Playwright MUST drive the actual Dashboard in a real browser, the actual CLI as a subprocess, and the actual REST API over HTTP. End-to-end as in: same code paths the user hits. |
| 031-FR-023 | Smoke tests MUST spawn the agent's actual container with its actual definition and exercise the spawn-claim-exit path. No mocks. |
| 031-FR-024 | Chaos tests MUST inject real failures: SIGKILL the API, drop the DB connection, crash the Outcall sidecar, OOM-kill a running container. The platform's spec-20 recovery is what's being tested. |

### Test data + fixtures

| ID | Requirement |
|---|---|
| 031-FR-030 | Test database MUST be reset between tests (transaction-rollback for unit, schema-recreate for integration). No cross-test leakage. |
| 031-FR-031 | Fixtures live under `tests/fixtures/`. Each fixture is small, self-contained, and named for what it represents (not `fixture-1.json` — `task-running-with-pending-approval.json`). |
| 031-FR-032 | Mock LLM (the deterministic stub provider) MUST be deterministic given inputs: same prompt → same response. Used in integration + E2E tests for repeatability. |
| 031-FR-033 | Test secrets MUST NEVER be real secrets. Broker uses a test backend with synthetic values; assertions on "secret value was injected" check shape, not material. |

### CI gates

| ID | Requirement |
|---|---|
| 031-FR-040 | Every PR MUST pass: lint, type-check, unit suite, integration suite, mirror-test check, coverage check, schema validation (spec 018), security scan. Failure blocks merge. |
| 031-FR-041 | E2E + smoke run on every PR but in a separate stage (slow lane). PR may merge if fast lane passes and slow lane is queued; slow-lane failure post-merge auto-reverts via revert PR. |
| 031-FR-042 | Chaos + load run nightly on `main`. Failures open auto-tickets via spec 014. |
| 031-FR-043 | Release branches additionally run the full chaos + load suite before tag. Tag fails if any chaos check fails. |

### Test authorship

| ID | Requirement |
|---|---|
| 031-FR-050 | Every PR implementing a Functional Requirement (e.g., spec 004-FR-006) MUST include tests that exercise that FR. Test annotation: `// covers: 004-FR-006` (machine-greppable). |
| 031-FR-051 | A separate report (`scripts/check-fr-coverage.ts`) maps FRs → tests and flags FRs with zero covering tests. Surfaces as a dashboard metric. |
| 031-FR-052 | Critical-path FRs (per CRITICAL_PATHS.md) MUST have at least one integration test AND at least one chaos test. |
| 031-FR-053 | Tests MUST be named after what they verify, not what they are. `tests/unit/services/policy_engine.test.ts` body: `test('default-deny applies when no rule matches')`, not `test('test policy engine 3')`. |

### Speed & developer experience

| ID | Requirement |
|---|---|
| 031-FR-060 | Watch mode (`pnpm test:watch`) MUST re-run only affected unit tests on save. Re-run latency p95 < 2s for a typical change. |
| 031-FR-061 | Pre-commit hook MUST run lint + type-check + affected unit tests in < 30s. Slower checks run in CI. |
| 031-FR-062 | Failing tests MUST surface a clear assertion + actionable diff (Japa's snapshot/assertion conventions). Stack traces include source maps. |
| 031-FR-063 | Flaky tests are NOT allowed to live. A test that fails non-deterministically twice in a week is quarantined (moved to `tests/quarantine/`) and a ticket opened. Quarantined tests run but do not gate. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 031-NFR-001 | Unit test suite MUST complete in < 30s on commodity hardware. Integration < 10 min. E2E < 30 min. Slow tests are budgeted, not unbounded. |
| 031-NFR-002 | Coverage report MUST be machine-readable (LCOV) and uploaded as a CI artifact. |
| 031-NFR-003 | Flaky-test detection MUST be automated; a test failing >X% of runs over a window MUST be flagged. |

## User stories

- **031-US-001 [P0]** — As a contributor, I open a PR; CI shows me exactly which files lack mirror tests and what coverage moved.
- **031-US-002 [P0]** — As a contributor, my PR introduces a coverage regression; the gate refuses merge until I add tests or use the audited override.
- **031-US-003 [P0]** — As a contributor, my PR implements an FR; I annotate the test with `// covers: 027-FR-012`; the FR-coverage report shows it.
- **031-US-004 [P1]** — As a maintainer, a nightly chaos run fails (recovery semantics regressed); an auto-ticket opens with the failing scenario.
- **031-US-005 [P1]** — As a contributor, my E2E test is flaky; after two failures in a week, it's auto-quarantined and a ticket is opened.
- **031-US-006 [P1]** — As a maintainer, the FR-coverage dashboard shows 3 FRs with no covering test; I prioritize closing them.

## Acceptance criteria

- Mirror-test gate enforces 1:1 file mapping with `@no-test:` opt-out.
- Coverage gates enforce per-area thresholds.
- E2E, smoke, chaos suites all exist and run.
- FR-to-test mapping is generated and reported.
- Pre-commit + CI gates operate as specified.

## Edge cases & failure modes

- **Pure-type file with no runtime code** (e.g., `types.ts`) → `@no-test: pure types` allowed.
- **Generated code** (e.g., from OpenAPI) → `@no-test: generated` allowed; provenance recorded.
- **Test that touches both unit and integration responsibilities** → split it; unit covers the function, integration covers the flow.
- **Coverage tool false-negative on TypeScript type-narrowing** → known issue; suppress per-line with comment + reason.

## Decision log

- **Test framework**: Japa (AdonisJS native) — chosen 2026-05-22. Matches the framework runtime; no additional framework dep.
- **E2E framework**: Playwright — chosen 2026-05-22. Industry standard, multi-browser, screenshots/video on failure for triage.
- **Mock provider for LLM in integration tests**: deterministic stub bundled with the test harness. Real providers excluded from integration; opt-in `--with-real-llm` flag for one-off verification only.
- **Mirror convention**: `app/<path>/<name>.ts → tests/unit/<path>/<name>.test.ts`. Resolved 2026-05-22 (per user directive).

## Related specs

This spec depends on no other spec but governs how every other spec is implemented. Most-related: constitution (Principle III & VIII), 018-config-validation, 020-stability-recovery, 019-agent-benchmarks-evals (different domain — agent quality vs framework quality), 025-onboarding-install (smoke gate).
