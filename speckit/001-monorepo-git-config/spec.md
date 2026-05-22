# 001 — Monorepo Git Configuration

**Status:** Draft
**Constitutional anchor:** Principle II (Git Is the Source of Truth)
**Depends on:** none (foundation feature)
**Required by:** 008, 009, 013, 015, 016, 024

## Purpose

Every piece of Clawie configuration — platform settings, teams, agents, skills, projects — lives in git. Each unit is its own repository, referenced as a submodule from a single root repo. This makes every change reviewable as a PR, every state rollback-able, and every configuration independently versionable.

## Why this matters

The research shows that frameworks treating configuration as opaque ("agent state is in memory" / "config is mutated in place") suffer from:
- No rollback when an update degrades behavior.
- No way to attribute changes to a person or reason.
- No way to compare "what does this agent look like today vs last month".

Clawie's defining differentiator from the survey is **per-component VCS rollback**.

## Functional requirements

| ID | Requirement |
|---|---|
| 001-FR-001 | A single root repo (`clawie-config`) MUST hold platform-level config (`clawie.yaml`), references to teams, projects, and marketplace mirror. |
| 001-FR-002 | Each team MUST be a git submodule of the root repo. |
| 001-FR-003 | Each agent within a team MUST be a git submodule of the team repo. |
| 001-FR-004 | Each project workspace MUST be a git submodule of the root repo under `projects/`. |
| 001-FR-005 | Skills installed from the marketplace MUST be pinned by signed tag, recorded in the team or agent repo. |
| 001-FR-006 | Submodule pointer updates MUST be the canonical event surfaced in the dashboard as "config change". |
| 001-FR-007 | A CLI command (`clawie repo init`, `clawie repo sync`) MUST manage the submodule graph. |
| 001-FR-008 | An operator MUST be able to roll back any single submodule independently: `clawie agent rollback <name> --to <ref>`. |
| 001-FR-009 | The platform MUST refuse to start when the root repo or any required submodule is dirty (uncommitted changes), unless `--dev-mode` is set. |
| 001-FR-010 | An operator MUST be able to attach an optional remote (GitHub, Gitea, self-hosted) for backup; remote is not required. |
| 001-FR-011 | Branch protection: agents and pipeline tasks MUST never commit to `main` of their repo. Only the control plane, on operator-approved merge, MUST be allowed to fast-forward main. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 001-NFR-001 | Submodule operations MUST complete in <2s for a typical (≤20 agents) configuration. |
| 001-NFR-002 | The platform MUST verify all submodules are present and at the recorded SHA at boot. |
| 001-NFR-003 | Git LFS support: not required in v1. Documented as future option. |

## Layout (canonical)

```
~/.clawie/config/                      # root repo
├── clawie.yaml                       # platform config
├── outcall.yaml                      # optional, integration mode + global rules
├── .clawie/                          # platform-private (gitignored)
│   ├── runtime.lock
│   └── audit.snapshot
├── teams/
│   ├── default-agency/               # submodule
│   │   ├── team.yaml
│   │   ├── budgets.yaml
│   │   ├── flows.yaml
│   │   └── agents/
│   │       ├── researcher/           # submodule
│   │       ├── architect/            # submodule
│   │       ├── coder/                # submodule
│   │       ├── reviewer/             # submodule
│   │       ├── security/             # submodule
│   │       ├── perf/                 # submodule
│   │       ├── doc-writer/           # submodule
│   │       ├── devops/               # submodule
│   │       └── marketer/             # submodule
├── skills-registry/                  # submodule (marketplace mirror)
└── projects/
    ├── server-monitoring-saas/       # submodule
    └── ...
```

## Per-agent repo contents

```
<agent-name>/
├── SOUL.md                  # identity, voice, motivation
├── AGENTS.md                # role, responsibilities, allowed peers
├── TOOLS.md                 # tool list + permission requests
├── MODEL.yaml               # provider preference
├── skills/                  # locally installed or vendored skills
├── benchmarks/              # eval fixtures
├── prompts/                 # versioned prompts / templates
└── README.md
```

## User stories

- **001-US-001 [P0]** — As an operator, I run `clawie repo init` once and get a root repo, a default-agency team submodule with all roles, and an empty projects/ tree.
- **001-US-002 [P0]** — As an operator, I rollback the `coder` agent to last week with one command, and the next task uses the rolled-back definition.
- **001-US-003 [P0]** — As an operator, I push the root repo to a private GitHub remote for backup, and pulling on a new machine restores the entire agency.
- **001-US-004 [P1]** — As an operator, I see a dashboard view of "what changed in the agency this week" computed from submodule pointer bumps.

## Acceptance criteria

- `clawie repo init` produces a working, validated config tree in one command.
- Rolling back any agent does not affect any other agent.
- The platform refuses to boot with dirty submodules unless `--dev-mode`.
- All submodules verified at startup; missing one fails fast with a clear cause-of-failure.

## Edge cases & failure modes

- **Detached HEAD on a submodule** — boot fails with explicit remediation pointer.
- **Submodule URL changes** (e.g., GitHub → Gitea) — handled by `clawie repo sync --remap`.
- **Merge conflict in root repo on submodule bump** — operator resolves; platform refuses to run during conflict.
- **Operator manually mutates `~/.clawie/config/` without commit** — boot warns; `--dev-mode` allows, audit notes.
- **Deleted agent** — submodule removal commit; running tasks gracefully drain; no orphan containers.

## Open questions

- Should marketplace mirror be a submodule or a separate cached directory? (Lean: submodule with shallow clone.)
- LFS for large benchmark fixtures? (Defer.)

## Related specs

`008-agent-definition`, `009-agent-self-modification-pr`, `015-workspace-codebase-mount`, `018-config-validation-pre-merge`, `024-plugin-marketplace`.
