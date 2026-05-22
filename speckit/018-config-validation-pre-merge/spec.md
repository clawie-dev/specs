# 018 — Configuration Validation Before Merge

**Status:** Draft
**Constitutional anchor:** Principle III (Validated before merged), VIII (Stability)
**Depends on:** 001, 008, 010, 013
**Required by:** every spec that ships config

## Purpose

Every configuration that touches production passes a gate: schema check, dependency check, semantic lint, optional smoke test. The user's hard requirement: "we should see failures all the time, like OpenClaw […] all configurations should be validated before allowing merge."

## What gets validated

| Artifact | Checks |
|---|---|
| `clawie.yaml` (root) | Schema, referenced submodule paths exist |
| `team.yaml` | Schema, all referenced agent submodules exist, no role-name conflicts |
| `flows.yaml` | Schema, no cycles (unless `cycle_safe`), all referenced roles exist, stages all map to roles |
| `budgets.yaml` | Schema, no negative budgets, soft thresholds < hard caps |
| Agent SOUL.md, AGENTS.md | Required sections present, AGENTS frontmatter schema |
| TOOLS.md | Schema, declared tools exist in registry, permissions parse |
| MODEL.yaml | Schema, providers known, models known (or marked custom) |
| Skill manifest | Schema, signature, declared dependencies resolvable |
| Policy rules | Schema, no conflicts at same scope, regex patterns compile |
| Outcall rules | Schema (per Outcall's own validator) |
| Project briefs | Min length, no obvious PII leakage (heuristic warning) |

## Functional requirements

| ID | Requirement |
|---|---|
| 018-FR-001 | A pre-commit hook MUST be installed by `clawie repo init`. |
| 018-FR-002 | The `clawie validate` command MUST run all applicable validators against the current tree. |
| 018-FR-003 | Validation MUST also run as a server-side check at merge time. The platform refuses to update a submodule pointer to a SHA whose tree fails validation. |
| 018-FR-004 | Smoke tests MUST run for: new agents (spawn-and-exit), new skills (manifest test), new teams (cold-start). |
| 018-FR-005 | Validation reports MUST be structured: `{passed, failed_checks[], warnings[]}`. |
| 018-FR-006 | An operator MAY override with `--force-merge` + reason; override is audited. |
| 018-FR-007 | Validators MUST be extensible — plugins can register their own checks. |
| 018-FR-008 | Validation MUST be fast: typical validation under 5s for a single agent change; full repo validation under 30s. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 018-NFR-001 | Validator output MUST be machine-readable (JSON) for CI integration. |
| 018-NFR-002 | Validator MUST have its own test suite ensuring no false negatives on known-bad fixtures. |

## User stories

- **018-US-001 [P0]** — As an operator, I edit AGENTS.md with a syntax error; pre-commit blocks me with the exact issue.
- **018-US-002 [P0]** — As an operator, my CI runs `clawie validate` against the config repo and blocks bad PRs.
- **018-US-003 [P1]** — As an operator, I add a custom validator for "no agent named after a Greek god" via a plugin (yes, it's a test case).
- **018-US-004 [P1]** — As an operator, an override is captured in audit with my reason.

## Acceptance criteria

- All listed artifacts have validators.
- Pre-commit + server-side checks both enforce.
- Smoke tests gate at least agent and team installs.
- Override requires explicit flag + reason.

## Edge cases & failure modes

- Validator itself buggy → false rejection. Mitigated by validator test suite and `--force-merge` escape hatch.
- Schema evolves → versioned schemas; old configs upgraded by migration tool with diff for review.
- Smoke test flaky → retried N times; persistent failure surfaces with logs.

## Related specs

All other specs (config shape) — but especially `001-monorepo-git-config`, `008-agent-definition`, `010-skills-plugins-lazy-load`, `013-team-orchestration-orgchart`, `024-plugin-marketplace`.
