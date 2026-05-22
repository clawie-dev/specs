# 015 — Workspace & Customer Codebase Mount

**Status:** Draft
**Constitutional anchor:** Principle I (Layered), IV (Default-deny)
**Depends on:** 002, 003, 004
**Required by:** 016

## Purpose

Agents operate on real customer codebases (a Laravel app, a Next.js repo, a Python service). Clawie gives each task either a *mounted* live worktree or a *copied* snapshot. Writes go through policy. Changes ship back as a branch+PR to the customer's repo (or stay as a diff for review).

## Why this matters

The user's requirement: "spin ups of a agent should mount (or copy, based on configurations) a codebase over, like a Laravel codebase etc, where agent can do changes to the codebase etc."

This is also OpenHands' core pattern, refined for the durable-state world.

## Workspace modes

| Mode | Behavior | Use |
|---|---|---|
| `mount-rw` | Live worktree mounted read-write. Changes happen in place. | Trusted, fast iteration on local code. |
| `mount-ro` | Live worktree mounted read-only. | Inspection-only agents (e.g., security audit). |
| `copy-ephemeral` | Worktree copied into container; changes returned as diff. | Default for production. Safe. |
| `clone-branch` | Container clones a fresh branch from the customer's remote git; pushes back the branch on completion. | When the codebase lives at a remote and operator wants a PR opened. |

## Functional requirements

| ID | Requirement |
|---|---|
| 015-FR-001 | Each project MUST declare its workspace source (local path or remote git URL) and default mode in `projects/<name>/project.yaml`. |
| 015-FR-002 | A task MAY override the default mode per its declared intent. |
| 015-FR-003 | `mount-rw` MUST acquire a durable workspace lock (one task at a time). |
| 015-FR-004 | `copy-ephemeral` MUST capture a tree hash before/after and emit a diff to the project's artifacts. |
| 015-FR-005 | `clone-branch` MUST commit changes, push to a feature branch, and optionally open a PR via configured tool. |
| 015-FR-006 | File writes MUST be evaluated by the policy engine: writes outside `/workspace` blocked; sensitive files (e.g., `.env`, `secrets.json`) blocked by default. |
| 015-FR-007 | Workspace MUST be quota-limited (disk size). Exceeding cap fails the task with cause. |
| 015-FR-008 | Diffs MUST be reviewable in the dashboard before any merge to customer main. |
| 015-FR-009 | `.clawieignore` (similar to .gitignore) MUST allow customers to declare paths off-limits to agents. |
| 015-FR-010 | Workspace state survives container reap (the diff persists in the project artifacts). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 015-NFR-001 | Copy for a 1 GB repo MUST be ≤30s on commodity hardware. |
| 015-NFR-002 | Diff capture overhead < 5s for a typical changeset. |
| 015-NFR-003 | Workspace quota default: 5 GB per project; configurable. |

## User stories

- **015-US-001 [P0]** — As an operator, I point Clawie at `/home/me/myapp` (Laravel) and an agent makes changes I review as a diff before merge.
- **015-US-002 [P0]** — As an operator, an agent tries to write `/etc/passwd`; policy denies; audit captures.
- **015-US-003 [P1]** — As an operator, I configure `clone-branch` mode against a private GitHub repo; the agent opens a PR.
- **015-US-004 [P1]** — As a customer, I add `.clawieignore` listing my secrets folder; agent cannot read or write it.

## Acceptance criteria

- Each mode round-trips correctly for a sample repo.
- Sensitive paths refused.
- Lock contention surfaces in dashboard with queue.
- Diff visible in dashboard for review.

## Edge cases & failure modes

- Customer repo too large to copy → operator switches to `mount-rw` with lock; or excludes paths.
- Customer repo has submodules → handled (recursive checkout); merge conflicts flagged.
- Customer remote rejects push (auth) → task fails with clear `cause`; credential needs grant.
- Concurrent `mount-rw` attempt → second task queues for the lock.

## Related specs

`002-container-runtime-outcall`, `003-policy-permissions`, `005-approvals-hitl`, `012-credential-broker`, `016-software-agency-pipeline`.
