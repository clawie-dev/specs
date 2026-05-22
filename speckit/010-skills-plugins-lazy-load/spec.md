# 010 — Skills & Plugins (Lazy-Loaded)

**Status:** Draft
**Constitutional anchor:** Principle X (Pluggable), III (Validated); Security constraint 7
**Depends on:** 001, 003
**Required by:** 008, 016, 024

## Purpose

Skills are the unit of extensibility — packaged capability that an agent can declare it needs. Hot capability summaries live in the prompt; full skill content loads only when activated. This is the Viktor lesson distilled.

## Why this matters

"All tools in prompt" doesn't scale past ~50 tools. Viktor has 3,200 connectors and explicitly designed for lazy loading. Claude Code skills work similarly. Clawie adopts this from day one.

## Skill anatomy

A skill is a directory:

```
skills/stripe-billing-implementation@1.2.0/
├── manifest.yaml        # name, version, capability summary, signature
├── capability.md        # short hot-loadable summary (≤500 tokens)
├── full.md              # detailed instructions (lazy-loaded)
├── tools/               # tool definitions (optional)
├── prompts/             # prompt fragments
├── examples/            # few-shot examples
└── tests/               # smoke tests
```

`manifest.yaml`:

```yaml
name: stripe-billing-implementation
version: 1.2.0
author: clawie-community/payments
description: Adds Stripe checkout + webhook flow to a Node/Laravel/Rails app
capability_summary: |
  Use this skill when the task involves adding Stripe payments.
  Inputs: codebase path, plan tier definitions.
  Outputs: PR with Stripe client config, checkout flow, webhook handler, tests.
required_tools: [bash, filesystem, github]
required_permissions:
  - { action: write_file, resource: "/workspace/**" }
  - { action: call_tool, resource: "github.create_pr" }
signature: sha256:...
trusted_signers: [marketplace, your-team]
```

## Functional requirements

| ID | Requirement |
|---|---|
| 010-FR-001 | The runtime MUST hot-load `capability.md` for every declared skill into the system prompt scaffold. |
| 010-FR-002 | `full.md` and detailed prompts MUST be lazy-loaded by an explicit "load skill" tool call. |
| 010-FR-003 | Skills MUST be signature-verified at install per the trust tiers in 010-FR-003a. Unsigned skills require explicit `--unsafe` flag for local install. A skill published to the marketplace MUST be at minimum author-signed; unsigned local skills MUST NOT be referenced as dependencies by any published skill (validator-enforced). |
| 010-FR-003a | **Signature trust tiers:** `marketplace` (signed by marketplace key after review — gold tier, the default trust required for any new marketplace install); `author` (author-signed only — silver tier, requires operator explicit opt-in via `--trust author` per author key); `unsigned` (dev mode only — requires `--unsafe`, dashboard banner, never auto-installable). Per-team config declares the minimum acceptable tier. |
| 010-FR-004 | Skills MUST declare required tools and permissions; install warns the operator about new permission asks. |
| 010-FR-005 | Skills MUST be pinned by version (`@1.2.0`) in the agent's TOOLS.md. |
| 010-FR-006 | Marketplace skills MUST be mirrored as a submodule under `skills-registry/` in the root config repo. This mirror pins the marketplace state at a known SHA; agent TOOLS.md references skills by `name@version`, which the platform resolves against the mirror at agent install/boot time. `clawie skill upgrade` bumps the mirror pin and stages the agent-side version bumps as a PR. |
| 010-FR-007 | Skill smoke tests MUST pass before merge to the agent that declares them. |
| 010-FR-008 | Skill upgrades MUST be reviewable as PRs (diff between versions). |
| 010-FR-009 | A skill MAY require other skills (dependency graph). Cyclic deps rejected at install. |
| 010-FR-010 | Skills can ship MCP server specs; loading a skill MAY also register an MCP. |
| 010-FR-011 | Local development: an operator MAY symlink a skill from disk for iteration (`clawie skill link /path/to/skill`). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 010-NFR-001 | Hot prompt cost: ≤500 tokens per declared skill (capability.md summary). Per-agent hot-prompt cap: ≤8 000 tokens total across all declared skills. Skills exceeding the per-skill cap fail validation; agents whose total exceeds the cap require operator review at boot. |
| 010-NFR-002 | Lazy load latency < 200ms per skill. |
| 010-NFR-003 | Marketplace mirror sync nightly or on-demand. |

## User stories

- **010-US-001 [P0]** — As an operator, I install a skill from the marketplace and approve its declared permissions.
- **010-US-002 [P0]** — As an agent, I have 30 skills declared but only `capability.md` summaries in my prompt; full content loaded on use.
- **010-US-003 [P1]** — As a skill author, I publish a signed version to the marketplace.
- **010-US-004 [P1]** — As an operator, I link a local skill for development, run an agent, iterate, then publish.

## Acceptance criteria

- Lazy loading verified: prompt budget stays bounded as declared skill count grows.
- Unsigned skills require explicit flag, with clear warning.
- Skill upgrades reviewable as PRs.
- Smoke tests gate skill install per agent.

## Edge cases & failure modes

- Skill signature broken / unknown signer → install fails with clear remediation.
- Skill declares dangerous permission (e.g., write `/etc`) → install warns; explicit operator confirmation required.
- Skill version yanked → existing pinned installs continue; new installs blocked with reason.
- Capability summary too long (>500 tokens) → validator rejects.

## Related specs

`003-policy-permissions`, `008-agent-definition`, `018-config-validation-pre-merge`, `024-plugin-marketplace`.
