# 003 — Policy & Permissions Engine

**Status:** Draft
**Constitutional anchor:** Principle IV (Default-deny), IX (HITL); Security constraints 4, 7
**Depends on:** 001, 002
**Required by:** 005, 010, 011, 014, 015, 016

## Purpose

A single policy engine decides whether an agent's requested action is allowed, denied, or escalated to a human. Policy is layered with Outcall (network egress), but operates on a broader surface: tool calls, file writes, command execution, peer-agent communication, model selection, secret access.

## Why this matters

Research shows that frameworks relying on prompt-side restrictions ("don't run rm -rf") consistently leak. Clawie pushes enforcement into the policy engine, which lives in the control plane and is consulted by the agent runtime *before* any side-effect.

## Architecture

```
Agent runtime ─► requests tool/action ─► Policy engine (control plane)
                                          │
                                          ├── allowed by static rule? → allow
                                          ├── denied by static rule? → deny
                                          └── unknown? → approval queue (HITL)
                                                            ↓
                                                       operator decision
                                                            ↓
                                                   optionally auto-generate rule
```

## Functional requirements

| ID | Requirement |
|---|---|
| 003-FR-001 | Policy MUST be evaluated *before* the agent runtime performs any side-effecting action: tool call, file write, command exec, network request, peer message. |
| 003-FR-002 | Policies MUST be declarative (YAML or equivalent), versioned in the team or agent repo, and validated on merge. |
| 003-FR-003 | Policy scope MUST support: per-platform (global), per-team, per-agent, per-project, per-task. More-specific scopes override broader ones. |
| 003-FR-004 | Each policy rule MUST specify: subject (agent role/id), action verb (call_tool, write_file, exec_command, send_message, …), resource pattern (path, host, tool name), decision (allow/deny), optional condition (time, cost-remaining, prior-approval). |
| 003-FR-005 | The policy engine MUST surface a structured response: `{decision, matched_rule_id?, reason}`. |
| 003-FR-006 | When no rule matches, the default MUST be `escalate` (route to approval). Operator MAY override default to `deny` per scope. |
| 003-FR-007 | Approval queue MUST be visible in the dashboard within 1 second of escalation. |
| 003-FR-008 | An approval decision MAY produce an auto-generated rule (`approve once / approve always for this pattern`); auto-rules MUST be audit-logged with operator id, reason, and easy revocation. |
| 003-FR-009 | Unhandled escalation MUST time out per a configurable decision window (default 24h) and resolve to deny with a clear cause. |
| 003-FR-010 | The policy engine MUST emit traces of evaluation for every decision (rule match path, latency, decision). |
| 003-FR-011 | Policies MUST support a dry-run mode where decisions are logged but not enforced — used for migrating from permissive to strict. |
| 003-FR-012 | Outcall handles network egress; the Clawie policy engine MUST coordinate with Outcall by handing it the relevant ruleset on container spawn and consuming its decisions in the audit. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 003-NFR-001 | Policy evaluation p99 MUST be <50ms (in-process; not the human approval). |
| 003-NFR-002 | The policy ruleset MUST scale to ≥10k rules across all scopes. |
| 003-NFR-003 | Auto-generated rules MUST be reviewable as PRs (committed to the appropriate repo). |

## Rule example

```yaml
# teams/default-agency/policies/coder.yaml
- id: coder-allow-workspace-writes
  subject: { team: default-agency, agent: coder }
  action: write_file
  resource: "/workspace/**"
  decision: allow

- id: coder-deny-system
  subject: { team: default-agency, agent: coder }
  action: exec_command
  resource: ["rm -rf *", "sudo *", "shutdown *"]
  decision: deny

- id: coder-escalate-new-tools
  subject: { team: default-agency, agent: coder }
  action: call_tool
  resource: "*"
  condition: "not in trusted_tools"
  decision: escalate
```

## User stories

- **003-US-001 [P0]** — As an operator, an agent's attempt to write outside `/workspace` is denied and audited.
- **003-US-002 [P0]** — As an operator, when an agent requests an unknown tool, I see the request in dashboard within 1s.
- **003-US-003 [P0]** — As an operator, I approve once and choose "auto-allow this pattern"; the next identical request bypasses approval.
- **003-US-004 [P1]** — As an operator, I revert an auto-rule via `clawie policy revert <rule-id>`, with audit reason.
- **003-US-005 [P1]** — As an operator, I dry-run a stricter policy for a week before enforcing.

## Acceptance criteria

- Every side-effecting action results in a policy evaluation visible in audit.
- Default-deny: an action with no matching rule blocks pending approval or denies on timeout.
- Auto-rule generation produces a reviewable commit + audit entry.
- Outcall decisions appear in the unified audit alongside Clawie policy decisions.

## Edge cases & failure modes

- Policy engine slow → degrade to deny + queue, never silently allow.
- Rule conflict (two rules match, opposite decisions) → most-specific scope wins; tie-break by deny.
- Approval channel down (dashboard offline) → policy engine queues; timeout still applies; once dashboard recovers, queue replays.
- Operator disabled approvals via misconfig → boot validator catches it.

## Related specs

`002-container-runtime-outcall`, `005-approvals-hitl`, `006-observability-audit-cost`, `012-credential-broker`, `018-config-validation-pre-merge`.
