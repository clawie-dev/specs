# 012 — Credential Broker

**Status:** Draft
**Constitutional anchor:** Principle IV (Default-deny); Security constraint 3
**Depends on:** 002, 003
**Required by:** 011, 015, 016

## Purpose

The agent runtime MUST NEVER see raw long-lived credentials. The credential broker injects them at the network edge (preferred via Outcall) or via a secret-mount sidecar. Secrets are referenced by alias inside the agent; the broker resolves and injects.

## Why this matters

Research is emphatic: NemoClaw's egress proxy, Viktor's "credentials don't touch the AI", Claude Code's scoped credential proxies all converge. Any framework that hands keys to the agent process loses on prompt injection or log leakage eventually.

## Architecture

```
Agent container ─► tool call needs credential alias ─► Outcall (or broker sidecar)
                                                       │
                                                       ├── looks up alias in secrets store
                                                       ├── injects into outbound request (header / query / body)
                                                       └── audits the use
```

The agent code calls e.g., `github.create_pr(...)` referencing alias `github_main`. The broker resolves and inserts the token transparently. The agent never reads the token's value.

## Functional requirements

| ID | Requirement |
|---|---|
| 012-FR-001 | Secrets MUST be stored in a backend pluggable: encrypted-at-rest file (default), system keychain, HashiCorp Vault, AWS Secrets Manager, env-injected (dev only). |
| 012-FR-002 | Each secret MUST have: alias, kind (api_key / oauth_token / ssh_key / generic), scope (which agents/teams may reference). |
| 012-FR-003 | Agents reference secrets by alias only. The runtime MUST refuse to materialize secret values into prompts, logs, or stdout. |
| 012-FR-004 | The broker MUST validate the agent's scope before injecting; cross-scope refusal logged. |
| 012-FR-005 | Outbound requests MUST be matched to a credential-injection rule (host pattern, header name, value template). |
| 012-FR-006 | Every credential use MUST emit an audit event: (agent, alias, destination, outcome). |
| 012-FR-007 | Outcall mode: the broker is part of Outcall's egress proxy. Disabled mode: broker is a sidecar in the agent's Docker network. |
| 012-FR-008 | Rotation: secrets MAY be rotated server-side; agents continue working without restart. |
| 012-FR-009 | Operator CLI: `clawie secret add`, `secret rotate`, `secret revoke`, `secret scope` — all audited. |
| 012-FR-010 | Secret material MUST NEVER appear in dashboards, audit details, exports, traces, or error messages. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 012-NFR-001 | Broker overhead MUST be < 20ms per request. |
| 012-NFR-002 | Backend pluggability tested end-to-end for at least the default file backend and Vault. |

## User stories

- **012-US-001 [P0]** — As an operator, I add a GitHub PAT as `github_main` scoped to the `coder` agent; the coder uses it without ever seeing it.
- **012-US-002 [P0]** — As an operator, I rotate the secret; running tasks continue.
- **012-US-003 [P0]** — As an operator, I revoke a secret; the next use returns an explicit refusal audit-entry.
- **012-US-004 [P1]** — As an operator, I switch the backend to Vault by editing `clawie.yaml`.

## Acceptance criteria

- Agent prompt and stdout audited; no secret material present.
- Audit entries for every credential use.
- Cross-scope use refused.
- Rotation transparent.

## Edge cases & failure modes

- Agent tries to log secret-shaped string → broker scans logs for known secret prefixes and redacts at sink (defense-in-depth).
- Broker unreachable → outbound credential-required call fails; task pauses with `cause=credentials_unavailable`.
- Vault backend latency spike → broker caches short-lived; cache invalidated on rotation.

## Related specs

`002-container-runtime-outcall`, `003-policy-permissions`, `011-model-router-providers`, `006-observability-audit-cost`.
