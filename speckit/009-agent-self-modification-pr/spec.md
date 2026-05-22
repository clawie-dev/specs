# 009 — Agent Self-Modification via PR Review

**Status:** Draft
**Constitutional anchor:** Principle II, III, IX
**Depends on:** 001, 005, 008, 019
**Required by:** 016

## Purpose

Agents can learn from their work. Clawie captures every proposed change to an agent's own identity, role, tools, prompts, or skills as a *branch* on the agent's repo and a *PR* in the dashboard. Operator reviews diff + benchmark delta + reason; merge advances; reject closes.

## Why this matters

The user's defining requirement: "Changes to itself, like updating SOUL.md, AGENTS.md, TOOLS.md and other agent specific details, should become a branch and any changes should be commited and listed in Clawie dashboard for review — that way we have control over their learnings and changes."

## Functional requirements

| ID | Requirement |
|---|---|
| 009-FR-001 | An agent MUST NEVER commit to its repo's `main` directly. It commits to a working branch named `selfmod/<task-id>/<short-desc>`. |
| 009-FR-002 | A self-modification branch MUST include: changed files, a `CHANGE.md` describing what + why + before/after expected behavior, and a benchmark delta (run before opening the PR). |
| 009-FR-003 | The dashboard MUST list pending self-mod PRs per agent with: diff view, CHANGE.md, benchmark delta, related tasks, "who proposed" (which task/run), and merge/reject buttons. |
| 009-FR-004 | Merge requires explicit operator approval OR a pre-authorized rule. Auto-merge rules MUST be conservative by default: a rule like "auto-merge if benchmark delta > +X% AND fixture files unchanged AND changes only in path whitelist" MUST satisfy all three predicates. Auto-merge MUST refuse if the same PR also modifies the agent's `benchmarks/` directory ("agent rewriting its own test" defense per spec 019-FR-012). |
| 009-FR-005 | Rejected branches remain in history for 30 days then are auto-pruned. |
| 009-FR-006 | If a benchmark regresses below threshold, merge MUST be blocked (operator can override with `--force-merge` + audit reason). |
| 009-FR-007 | After merge, the next agent task MUST use the new definition; running tasks finish on the old. |
| 009-FR-008 | Rollback: `clawie agent rollback <name> --to <ref>` updates the submodule pointer; produces audit entry. |
| 009-FR-009 | Per-agent rollback MUST NOT affect any other agent. |
| 009-FR-010 | Self-modifications MUST be rate-limited per agent (default: max 3 per day, configurable per team). Sub-limits by change kind: SOUL.md max 1/week (high-risk identity drift), AGENTS.md max 1/day, TOOLS.md max 1/day, prompts/ max 3/day, skills/ unlimited (always reviewable). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 009-NFR-001 | PR view in dashboard MUST render diff for ≥1000 line changes within 2s. |
| 009-NFR-002 | Benchmark delta computation parallel with PR open, results within 60s for typical fixture suite. |

## CHANGE.md schema

```markdown
# Change: <slug>

**Type:** SOUL | AGENTS | TOOLS | MODEL | skills | prompts | benchmarks
**Proposed by:** task <id> during <project> on <date>
**Reason:** <free text from the agent>

## What changed
- ...

## Expected behavior delta
- ...

## Benchmark delta
- Pre: <score>
- Post: <score>
- Regression risk: low / medium / high
```

## User stories

- **009-US-001 [P0]** — As an operator, I see "coder proposes change to prompts/review.md" in dashboard with diff + benchmark.
- **009-US-002 [P0]** — As an operator, merge requires my click; reject closes the branch.
- **009-US-003 [P0]** — As an operator, I rollback coder to last Tuesday's version with one command; other agents unaffected.
- **009-US-004 [P1]** — As an operator, I configure auto-merge for `prompts/` changes that improve benchmark.
- **009-US-005 [P1]** — As an operator, a regression below threshold blocks merge; dashboard surfaces it as "blocked: benchmark regression".

## Acceptance criteria

- Every self-mod creates a branch + PR + CHANGE.md + benchmark delta.
- Operator decision visible in audit with timestamp + reason.
- Rollback affects only the targeted agent.
- Auto-merge rules are explicit and auditable.

## Edge cases & failure modes

- Agent proposes infinite changes → rate limit per agent.
- Branch conflict with concurrent self-mod → second proposal rebased or rejected.
- Operator merges, then immediately regrets → rollback with one command, audit captures both.
- Benchmark fixture itself changed in the same PR → flagged as "agent rewriting its own test"; requires extra scrutiny.

## Related specs

`001-monorepo-git-config`, `005-approvals-hitl`, `008-agent-definition`, `019-agent-benchmarks-evals`, `022-web-dashboard`.
