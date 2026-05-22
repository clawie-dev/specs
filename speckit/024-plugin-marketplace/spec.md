# 024 — Plugin Marketplace (market.clawie.dev)

**Status:** Draft
**Constitutional anchor:** Principle XI (Open source first), III (Validated)
**Depends on:** 010
**Required by:** none (delivers UX + ecosystem)

## Purpose

A public registry for community contributions: skills, task-management drivers, model providers, channel adapters, connector tools, validator plugins, starter-pack teams. Signed and reviewed. Operators discover, install with one command, manage via dashboard/CLI.

## Plugin types

| Type | Purpose | Spec |
|---|---|---|
| **Skill** | Capability bundle for agents | 010 |
| **Task-mgmt driver** | Linear/Jira/etc. integration | 026 |
| **Model provider** | LLM provider adapter | 011 |
| **Channel adapter** | Slack/Telegram/Discord/email surface | (future) |
| **Connector** | Tool adapter (Stripe, GitHub, AWS, …) | 010 |
| **Validator plugin** | Custom config validator | 018 |
| **Starter pack** | Pre-built team(s) | 013 |
| **Eval fixture** | Benchmark fixture | 019 |

## Marketplace pieces

- **Public registry** at `market.clawie.dev` — searchable index.
- **Author workflow** — submit a signed release to a repo, marketplace runs CI (build, scan, test, signature check), reviewer approves, listing goes live.
- **Operator workflow** — search/browse, see permissions + signature status, install with one command. Subsequent upgrades reviewable.

## Functional requirements

| ID | Requirement |
|---|---|
| 024-FR-001 | Every plugin MUST have: name, version, type, description, author, signature, repo URL, permission declarations, eval/test report. |
| 024-FR-002 | Plugins MUST be signed by their author. Marketplace MUST validate signatures and surface "trusted by marketplace" status. |
| 024-FR-003 | Marketplace CI MUST run: signature verification, security scan (deps + secrets), smoke test, schema validation. |
| 024-FR-004 | A human reviewer MUST approve a plugin before its first listing OR an automated trust path (e.g., reputation-based) is established with audit. v1 is human-approval-only. |
| 024-FR-005 | Operators MUST be able to install via CLI (`clawie plugin install <name>@<version>`) and dashboard. |
| 024-FR-006 | Installed plugins MUST be pinned by version in the team or agent repo. |
| 024-FR-007 | Upgrades MUST be reviewable as PRs (diff between versions). |
| 024-FR-008 | The operator MUST see all permissions a plugin declares before install and confirm. |
| 024-FR-009 | An operator MAY pin to a private alternative registry (self-hosted or org-internal). |
| 024-FR-010 | Yanked / vulnerable plugins MUST emit alerts to operators with affected installs; install of yanked version blocked. |
| 024-FR-011 | Marketplace MUST expose a stats view: install counts, upgrade rates, eval score distribution per skill. |
| 024-FR-012 | License field MUST be required; SPDX identifier preferred. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 024-NFR-001 | Marketplace search results < 500ms p95. |
| 024-NFR-002 | Install (typical 10 MB plugin) < 10s. |
| 024-NFR-003 | Yanking propagates to operators within 24h via the platform's daily registry sync. |

## User stories

- **024-US-001 [P0]** — As an operator, I install a signed Stripe connector and confirm declared permissions.
- **024-US-002 [P0]** — As a plugin author, I submit a new skill, marketplace CI runs, reviewer approves, listing live.
- **024-US-003 [P1]** — As an operator, an upgrade of a skill is queued as a PR for me to review.
- **024-US-004 [P1]** — As an operator, a yanked plugin in my install triggers a dashboard alert.

## Acceptance criteria

- Signature verification mandatory.
- Permission declarations visible pre-install.
- Yank propagates with alerts.
- Per-version pinning enforced.

## Edge cases & failure modes

- Author signing key compromised → marketplace yanks affected versions; alerts ripple.
- Plugin works locally but fails in CI → submission rejected with logs.
- Operator on air-gapped network → mirror registry workflow supported.
- Plugin declares more permissions than it actually uses → drift detection over time; flag.

## Related specs

`010-skills-plugins-lazy-load`, `011-model-router-providers`, `018-config-validation-pre-merge`, `026-task-management-drivers-flow-rules`.
