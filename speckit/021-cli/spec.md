# 021 — CLI (AdonisJS Ace)

**Status:** Draft
**Constitutional anchor:** Principle I (Layered — surfaces)
**Depends on:** 023 (CLI calls API; never bypasses it for state changes)
**Required by:** none (delivers UX)

## Purpose

A first-class CLI built on AdonisJS Ace. Every interactive operation an operator can do in the dashboard MUST also be doable via CLI. Scriptable end-to-end. Tab completion, JSON output flag, human-readable default.

## Commands (canonical surface)

```
clawie repo init                                  # initialize root config repo
clawie repo sync                                   # sync submodules
clawie validate                                    # run all validators
clawie up                                          # start the platform
clawie down                                        # stop the platform
clawie status                                      # cluster status, health, queue depth

clawie team list / install / fork / remove / pause / resume
clawie team flows show <team>                      # render flow diagram (ascii / svg)

clawie agent list / show / rollback / suspend / resume / benchmark <name>
clawie agent edit <name>                           # opens repo in $EDITOR; on save, validates

clawie project new "<brief>" [--team <name>] [--budget USD] [--priority urgent]
clawie project list / show / pause / resume / abort / boost <name>
clawie project artifacts <name>                    # list and fetch artifacts

clawie task list [--status <s>] [--project <p>] [--agent <a>]
clawie task get <id>
clawie task logs <id> [--follow]
clawie task abort <id> --reason "..."
clawie task transition <external-id> <status> [--force --reason "..."]    # spec 026

clawie approval list                                # pending approvals
clawie approval decide <id> --action approve [--with-rule "..."] [--reason "..."]

clawie policy add / list / revert / show
clawie secret add / rotate / revoke / scope / list  # never prints values

clawie skill install / link / unlink / list
clawie plugin install / list / verify

clawie eval run <agent> [--fixture <f>]
clawie eval trends [--agent <a>]                   # ASCII chart

clawie audit query "..." [--from <date>] [--to <date>] [--actor <a>]
clawie audit verify                                 # checks tamper-evident chain
clawie audit export --to s3://...

clawie outcall status / rules / test               # outcall integration
clawie tm connect <driver> / list / show           # task-management driver mgmt (spec 026)

clawie doctor                                      # diagnose common platform issues
```

## Functional requirements

| ID | Requirement |
|---|---|
| 021-FR-001 | All commands MUST be implemented as Ace commands in the AdonisJS app. |
| 021-FR-002 | All state-changing CLI commands MUST call the REST API (spec 023), not the database directly. Read-only commands MAY use either. |
| 021-FR-003 | Every command MUST support `--json` for machine-readable output. |
| 021-FR-004 | Every command MUST support `--verbose` (more detail) and `--quiet` (errors only). |
| 021-FR-005 | Tab completion installable for bash, zsh, fish. |
| 021-FR-006 | Destructive operations (`agent rollback`, `task abort`, `project abort`, `policy revert`, `--force` anywhere) MUST require explicit confirmation OR `--yes`. |
| 021-FR-007 | Authentication: operator token in `~/.clawie/cli.token` or `CLAWIE_TOKEN` env. Remote install supported via `--host`. |
| 021-FR-008 | Help text MUST include examples for every non-trivial command. |
| 021-FR-009 | `clawie doctor` MUST diagnose: missing submodules, dirty repos, expired tokens, unreachable API, Outcall health, Docker availability, disk space, dependency versions. |
| 021-FR-010 | Exit codes MUST be conventional (0 = success, non-zero = failure class). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 021-NFR-001 | Common commands MUST respond in <500ms p95 (excluding the underlying operation). |
| 021-NFR-002 | CLI MUST work offline for `validate`, `agent edit`, `doctor` (local-only diagnostics). |

## User stories

- **021-US-001 [P0]** — As an operator, I run `clawie project new "..."` and the project starts.
- **021-US-002 [P0]** — As an operator, I script approval responses via `--json`.
- **021-US-003 [P1]** — As an operator, `clawie doctor` finds my expired GitHub token.
- **021-US-004 [P1]** — As an operator, tab completion saves me typing across all commands.

## Acceptance criteria

- All listed commands implemented.
- JSON output verified.
- Tab completion installs for bash/zsh/fish.
- Doctor catches each documented failure mode.

## Related specs

`022-web-dashboard`, `023-rest-api`, and basically every functional spec — the CLI is the universal entry point.
