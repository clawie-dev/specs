# 008 — Agent Definition (SOUL, AGENTS, TOOLS)

**Status:** Draft
**Constitutional anchor:** Principle II (Git source of truth), X (Pluggable)
**Depends on:** 001
**Required by:** 009, 010, 011, 013, 015, 016

## Purpose

An agent's identity, role, voice, allowed tools, model preferences, and skills are all declared as files in a git repository. No agent state lives outside this repo (memory is a separate concern).

## Why this matters

The repo IS the agent. No magic. You can read it. You can diff it. You can roll it back. You can fork it.

## Required files

Each agent submodule MUST contain:

### `SOUL.md` — identity

The motivational layer. Voice, mission, "why I exist". This is the high-level system-prompt scaffold the runtime prepends to every model call. Free-form Markdown. Recommended sections: Identity, Mission, Voice, Boundaries.

### `AGENTS.md` — role and behavior

Operational definition: role, primary responsibilities, allowed peers (which other agents may message), preferred communication style, work cadence. Schema:

```yaml
---
name: coder
role: Implementation engineer
team: default-agency
peers: [reviewer, architect, devops]
communication_style: concise, technical
cadence: claims work from queue; max 3 concurrent tasks
escalates_to: architect
---
# Free-form behavior notes for the model
```

### `TOOLS.md` — tools and permission requests

What tools/skills/connectors this agent declares it needs. Each entry MUST be approvable by the policy engine. Schema:

```yaml
tools:
  - name: github
    why: open PRs against project repos
  - name: bash
    why: run tests, build, lint
  - name: filesystem
    why: edit /workspace
permissions_requested:
  - action: write_file
    resource: "/workspace/**"
  - action: exec_command
    resource: ["npm *", "git *", "node *"]
skills:
  - name: stripe-billing-implementation
    version: "1.2.0"
    pinned: true
```

### `MODEL.yaml` — provider preference

```yaml
default:
  provider: anthropic
  model: claude-sonnet-4-6
  max_tokens: 64000
fallback:
  - { provider: openai, model: gpt-5 }
  - { provider: local, model: llama-3.2-70b }
cost_cap_per_task_usd: 2.50
```

### `benchmarks/` — eval fixtures

At least one fixture. Each fixture is a YAML describing a task and its grading rubric. Used by spec 019.

### `skills/` — vendored or installed skills

Pinned to versions. Lazy-loaded at runtime.

### `prompts/` — versioned prompt templates

Any prompts the runtime uses (initial system, reflection, summarization). Versioned so changes are reviewable.

### `README.md`

Human description of the agent.

## Functional requirements

| ID | Requirement |
|---|---|
| 008-FR-001 | Validation gate MUST refuse to merge an agent commit lacking required files or with schema errors. |
| 008-FR-002 | The runtime MUST assemble the system prompt from SOUL.md + AGENTS.md + the active task's context. |
| 008-FR-003 | Skill load MUST be lazy: declared skills are listed by capability summary in the prompt; full skill content loaded only when the model decides to use it. |
| 008-FR-004 | Permission requests in TOOLS.md MUST be evaluated against the policy engine at runtime; declaration does not equal grant. |
| 008-FR-005 | Model fallback MUST be triggered on provider errors with consistent decision logic across agents. |
| 008-FR-006 | Each agent commit on `main` MUST be tagged with a benchmark score (from spec 019). |
| 008-FR-007 | An agent definition MUST be human-readable in <10 minutes for any reviewer. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 008-NFR-001 | Agent loading at container spawn < 1s for a typical agent (≤20 skills declared, ≤5 vendored). |

## User stories

- **008-US-001 [P0]** — As an operator, I clone a starter agency and see exactly how the coder agent is configured by reading 4 files.
- **008-US-002 [P0]** — As an operator, I edit SOUL.md on a branch, run validation, push, and the dashboard surfaces the PR for review.
- **008-US-003 [P1]** — As an agent, I declare a need for the `stripe` skill in TOOLS.md; the skill is loaded lazily when I attempt a billing task.

## Acceptance criteria

- Every agent has SOUL, AGENTS, TOOLS, MODEL, README, ≥1 benchmark fixture.
- Validation rejects schemaless edits.
- Lazy loading verifiable: prompt budget shows only summary unless skill activated.

## Edge cases & failure modes

- Conflicting peer declaration (peer doesn't exist) → validation fails.
- Model not available at provider → fallback chain consulted.
- Skill version yanked from marketplace → operator warned; agent runs without it (degrading gracefully) unless `pinned: true` (then refuse to spawn).

## Related specs

`001-monorepo-git-config`, `009-agent-self-modification-pr`, `010-skills-plugins-lazy-load`, `011-model-router-providers`, `019-agent-benchmarks-evals`.
