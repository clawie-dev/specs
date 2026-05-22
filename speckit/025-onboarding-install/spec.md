# 025 — Onboarding & Installation

**Status:** Draft
**Constitutional anchor:** Principle III (Validated), VIII (Stability)
**Depends on:** 001, 018, 020, 021, 023
**Required by:** none (delivers UX)

## Purpose

First-run experience that gets a working Clawie + default agency in under 10 minutes on a fresh machine. Resumable. Validated. Zero silent failures. Specifically counters NemoClaw's "non-resumable onboarding" criticism in the research.

## Install paths

| Path | Audience | Steps |
|---|---|---|
| **One-line script** | Indie operator | `curl -sSL clawie.dev/install \| sh` → script installs Docker (if missing), pulls images, initializes config repo, runs validator, starts. |
| **Docker compose** | Self-host with controlled stack | Provided compose file with Clawie + Postgres + Redis + Outcall. |
| **Package manager** | Linux operators | `apt`, `brew`, `dnf` packages pinned to releases. |
| **Manual** | Customization | Clone source, follow docs. |

## Functional requirements

| ID | Requirement |
|---|---|
| 025-FR-001 | First-run wizard MUST: detect/install dependencies, ask for LLM provider + key, initialize config repo, install `default-agency` starter pack, validate, smoke-test, start. |
| 025-FR-002 | Every step MUST be resumable. State persisted under `~/.clawie/install-state.json`. Re-running the installer picks up where it left off. |
| 025-FR-003 | First-boot validator MUST run automatically (spec 018) and refuse to start if anything fails — operator sees structured remediation. |
| 025-FR-004 | First-run smoke test: a no-op task on a sample agent MUST complete end-to-end before declaring "ready". |
| 025-FR-005 | Default Outcall mode MUST be `sidecar` if Outcall is available, `disabled` otherwise (with prominent warning). |
| 025-FR-006 | The wizard MUST default to a secure local-host bind (not 0.0.0.0). Operator explicitly opts in to remote access. |
| 025-FR-007 | Tokens MUST be generated locally; first operator token written to `~/.clawie/cli.token` with 0600 perms. |
| 025-FR-008 | Sample brief: `clawie demo` runs a 5-minute toy project to demonstrate the full pipeline without real cost (uses tiny local model or capped budget). |
| 025-FR-009 | `clawie doctor` MUST be available before / during / after install to diagnose. |
| 025-FR-010 | Uninstall MUST be supported and clean: `clawie uninstall` removes container, images (opt-in), data (opt-in), config (opt-in). Per-component prompts. |
| 025-FR-011 | Upgrades MUST be reversible: pre-upgrade snapshot of config + DB schema; rollback command provided. |
| 025-FR-012 | The wizard MUST surface a TL;DR of what's about to happen and ask for confirmation before destructive steps. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 025-NFR-001 | Install on a fresh Ubuntu 22.04 VM, no Docker preinstalled → ready in <10 min on a $20 VPS. |
| 025-NFR-002 | Install MUST work without internet access if all images are pre-staged (air-gapped path). |
| 025-NFR-003 | Failure mid-install MUST leave the system in a recoverable state — never partial. |

## User stories

- **025-US-001 [P0]** — As an operator, I run the install script on a fresh VPS; in <10 min I have a working dashboard + sample agent.
- **025-US-002 [P0]** — As an operator, the install fails halfway due to DNS; I fix DNS and re-run; it resumes from the failure.
- **025-US-003 [P0]** — As an operator, `clawie demo` shows me the full pipeline on a toy brief.
- **025-US-004 [P1]** — As an operator, I upgrade Clawie; the upgrade is reversible.
- **025-US-005 [P1]** — As an operator, I uninstall; nothing is left behind I didn't explicitly opt to keep.

## Acceptance criteria

- Fresh-VM install completes within target time.
- Resumable validated.
- Smoke test gates "ready" state.
- Uninstall + upgrade tested.
- `clawie demo` reliably succeeds without real cost.

## Edge cases & failure modes

- Docker version too old → installer offers upgrade path or aborts with clear message.
- LLM provider unreachable during install → wizard offers offline-mode (local model) or pauses.
- Disk full → fails fast, points to docs.
- Re-running install on already-installed system → detected; offers upgrade vs. re-init.

## Related specs

`001-monorepo-git-config`, `018-config-validation-pre-merge`, `020-stability-recovery`, `021-cli`, `024-plugin-marketplace`.
