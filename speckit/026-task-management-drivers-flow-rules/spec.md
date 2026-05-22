# 026 — Task Management Drivers & Flow Rules

**Status:** Draft
**Constitutional anchor:** Principle V (Durable state), IX (HITL), XII (End-to-end agency); Security constraint 4
**Depends on:** 003, 004, 013, 016
**Required by:** none (delivers integration UX)

## Purpose

External task management systems (Linear, Jira, GitHub Projects, Asana, Notion DB, Trello, …) plug in as **drivers**. Clawie layers a **flow-rules** ruleset on top of any driver — status ownership per agent role, allowed transitions, mandatory checkpoints. A dev agent cannot bypass Code Review → QA → Security to push a ticket to "Ready for Release", regardless of what the external system *would* permit, because Clawie's driver refuses the transition at the API boundary.

This is the same idea as Clawie's policy engine (spec 003), specialized to *workflow transitions on external task records*. It complements but does not replace spec 014 (inter-agent tickets) or spec 004 (internal task store).

## Vocabulary (important — three distinct concepts)

| Term | What it is | Spec |
|---|---|---|
| **Internal task** | A row in Clawie's control plane — one agent invocation. State machine: queued → running → … | 004 |
| **Agent-to-agent ticket** | A durable message from one agent to another (e.g., coder asks reviewer a question). | 014 |
| **External work item** | A ticket/issue in Linear/Jira/etc. that represents user-facing work for the agency. | 026 (this spec) |

A single external work item may spawn many internal tasks (research, draft, review, code, …). Drivers map between Clawie's pipeline stages (spec 016) and the external system's statuses.

## Why this matters

Two distinct user needs converge here:

1. **Operators already use Linear/Jira.** Clawie shouldn't fight that. It should be the *worker* against their existing board, not invent yet another board.
2. **Workflow integrity is a security property.** The whole point of having a Code Review and Security Review stage is that they cannot be bypassed. Workflow enforcement at the driver layer makes this structural, not aspirational.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Team config (per-team .yaml in team's git repo)            │
│   ├── task-management.yaml   ← driver + flow rules           │
│   └── ...                                                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Control plane: Task-Mgmt Driver Manager                    │
│   ├── load driver (linear / jira / github-projects / …)      │
│   ├── load flow rules (ownership + transitions)              │
│   └── expose: list, claim, transition, comment, attach       │
└──────────────────────────┬──────────────────────────────────┘
                           │ proxied through Outcall, creds via broker
┌──────────────────────────▼──────────────────────────────────┐
│  External task system (Linear API, Jira API, …)             │
└──────────────────────────────────────────────────────────────┘

When an agent attempts a transition:
   agent → driver.transition(item_id, target_status)
       │
       ├─ load flow rules
       ├─ check: is requesting agent the OWNER of source_status?
       ├─ check: is target_status an ALLOWED next from source for this agent?
       ├─ check: are gates satisfied (e.g., "must have Reviewer approval comment")?
       ├─ if all pass → call external API → audit
       └─ if any fails → DENY with structured reason; audit
```

## Driver interface (canonical)

A driver implements:

```typescript
interface TaskManagementDriver {
  name: string  // 'linear' | 'jira' | …
  capabilities: Capability[]  // list, transition, comment, label, attach, parent-link
  
  listItems(filter: ItemFilter): Promise<ExternalItem[]>
  getItem(id: string): Promise<ExternalItem>
  claim(itemId: string, agent: AgentId): Promise<void>   // optional; e.g., "assigned to"
  release(itemId: string, agent: AgentId): Promise<void>
  transition(itemId: string, targetStatus: string, by: AgentId, reason?: string): Promise<TransitionResult>
  comment(itemId: string, body: string, by: AgentId): Promise<void>
  attachArtifact(itemId: string, artifact: ArtifactRef): Promise<void>
  listStatuses(): Promise<StatusDef[]>
  
  // Webhook ingest: status changed in external system → propagate to Clawie
  onExternalChange(event: ExternalChangeEvent): Promise<void>
}
```

## Flow rules — `task-management.yaml`

Lives in the team's repo. Reviewed via PR like every other config.

```yaml
driver: linear
workspace: "https://linear.app/acme"
auth_secret: linear_main          # alias resolved by credential broker (spec 012)

# Map external statuses → optional Clawie pipeline stage (spec 016) for cross-linking
status_map:
  "Backlog":            { stage: brief }
  "Spec In Progress":   { stage: research }
  "Spec Review":        { stage: spec, approval_required: true }
  "In Development":     { stage: implement }
  "Code Review":        { stage: code-review }
  "QA Review":          { stage: e2e-test }
  "Security Review":    { stage: security-review }
  "Ready for Release":  { stage: deploy-production, approval_required: true }
  "Released":           { stage: completed }
  "Duplicate":          { stage: archived }

# Status ownership: which agent role(s) can act on items in this status
ownership:
  "Backlog":            [triage]
  "Spec In Progress":   [spec-agent]
  "Spec Review":        [reviewer, architect]
  "In Development":     [coder]
  "Code Review":        [reviewer]
  "QA Review":          [qa-agent]
  "Security Review":    [security]
  "Ready for Release":  [devops]
  "Released":           []                # no agent acts further; humans only

# Allowed transitions: source → target, with the agent role authorized.
# A transition not listed here is DENIED.
transitions:
  - { from: "Backlog",          to: "Spec In Progress",  by: spec-agent }
  - { from: "Backlog",          to: "Duplicate",         by: triage }
  - { from: "Spec In Progress", to: "Spec Review",       by: spec-agent }
  - { from: "Spec In Progress", to: "Duplicate",         by: spec-agent }
  - { from: "Spec Review",      to: "In Development",    by: reviewer, requires: [reviewer_approved] }
  - { from: "Spec Review",      to: "Spec In Progress",  by: reviewer }   # send back
  - { from: "In Development",   to: "Code Review",       by: coder }
  - { from: "Code Review",      to: "In Development",    by: reviewer }   # send back
  - { from: "Code Review",      to: "QA Review",         by: reviewer, requires: [tests_pass, lint_clean] }
  - { from: "QA Review",        to: "In Development",    by: qa-agent }
  - { from: "QA Review",        to: "Security Review",   by: qa-agent,    requires: [e2e_pass] }
  - { from: "Security Review",  to: "In Development",    by: security }
  - { from: "Security Review",  to: "Ready for Release", by: security,    requires: [no_high_findings, approval_logged] }
  - { from: "Ready for Release", to: "Released",         by: devops,      requires: [deploy_succeeded, human_approval] }

# Gate definitions — referenced by `requires` in transitions
gates:
  reviewer_approved:
    kind: comment_marker
    pattern: "/clawie approve-spec"
    by_role: reviewer
  tests_pass:
    kind: artifact_check
    artifact: "ci/test-report.json"
    check: "status == 'pass'"
  lint_clean:
    kind: artifact_check
    artifact: "ci/lint-report.json"
    check: "errors == 0"
  e2e_pass:
    kind: artifact_check
    artifact: "ci/e2e-report.json"
    check: "all_pass == true"
  no_high_findings:
    kind: artifact_check
    artifact: "security/findings.json"
    check: "max(severity) < 'high'"
  approval_logged:
    kind: audit_check
    description: "human approval entry within 24h"
  deploy_succeeded:
    kind: artifact_check
    artifact: "devops/deploy.json"
    check: "outcome == 'success'"
  human_approval:
    kind: hitl
    spec_ref: 005-approvals-hitl

# Optional: when external system changes a status outside Clawie,
# how does Clawie respond?
external_change_policy:
  on_unauthorized_change: alert            # alert | revert | accept
  on_status_drift: reconcile

# Sync model
sync:
  mode: bidirectional                       # bidirectional | inbound-only | outbound-only
  webhook_url: "https://clawie.example.com/webhooks/linear"
```

## Functional requirements

### Driver layer

| ID | Requirement |
|---|---|
| 026-FR-001 | Drivers MUST be pluggable: built-in drivers ship for Linear, Jira, GitHub Projects, GitLab Issues; community drivers installable as plugins (signed per spec 010/024). |
| 026-FR-002 | A driver MUST implement the canonical interface (list/get/claim/release/transition/comment/attach/listStatuses/onExternalChange). |
| 026-FR-003 | Drivers MUST use the credential broker (spec 012) — no driver may read its API token from env or config directly. |
| 026-FR-004 | All driver network calls MUST traverse Outcall when enabled. |
| 026-FR-005 | Drivers MUST expose their status definitions to the validator (spec 018), so the operator's `status_map` and `transitions` can be checked at config-merge time against the actual external system. |
| 026-FR-006 | A team MAY use at most one task-management driver in v1. (v2 may allow multi-board.) |
| 026-FR-007 | Teams without a task-management driver continue to work as today — internal task store + dashboard. |

### Flow rules

| ID | Requirement |
|---|---|
| 026-FR-010 | Every external status MUST have explicit `ownership`. Unowned statuses MUST be inert (no agent may act). |
| 026-FR-011 | Every transition an agent attempts MUST match a row in `transitions` with the agent's role in the `by` list AND all `requires` gates satisfied. |
| 026-FR-012 | A transition matching no row MUST be DENIED with `cause=transition_not_allowed` and audited. |
| 026-FR-013 | A transition matching a row but failing a gate MUST be DENIED with `cause=transition_gate_failed`, listing the failed gate. |
| 026-FR-014 | Gates of kind `hitl` MUST route to the approval queue (spec 005). |
| 026-FR-015 | Operator override path: `clawie task transition <item> <status> --force --reason "..."`. Force-transition MUST be audited with operator id, written as a comment to the external item, and counted in a metric for trend visibility. |
| 026-FR-016 | Multiple roles MAY share ownership of a status (e.g., `["reviewer", "architect"]`); any may act. |
| 026-FR-017 | Cyclic transitions are allowed (e.g., Code Review → In Development → Code Review); each iteration counted, max-iterations cap (default 8) escalates to operator per spec 016's loop rules. |

### Sync & reconciliation

| ID | Requirement |
|---|---|
| 026-FR-020 | When external system status changes outside Clawie (human moved a card in Linear), the driver MUST receive the event via webhook (or poll fallback) and reflect it into Clawie's internal task linkage. |
| 026-FR-021 | If a human moves an item past an unauthorized transition (e.g., directly to "Ready for Release"), Clawie MUST act per `external_change_policy`: `alert` (notify operator), `revert` (move back + comment explaining why), or `accept` (record + audit). Default: `alert`. |
| 026-FR-022 | Status drift (external item is in a state that doesn't map to any Clawie stage) MUST be reconciled per policy — usually a `Manual` bucket the operator clears. |
| 026-FR-023 | All driver API calls MUST be retried with exponential backoff on transient failures; permanent failures route through standard cause-of-failure (spec 006/020). |

### Mapping to internal pipeline

| ID | Requirement |
|---|---|
| 026-FR-030 | Each external status MAY map to a Clawie pipeline stage (spec 016). When mapped, the agent claiming an external item gets a properly-initialized internal task with the right stage. |
| 026-FR-031 | When the internal pipeline stage advances, the driver MUST attempt the corresponding external transition (subject to the same flow rules — i.e., the agent attempting the advance must own the source status, etc.). |
| 026-FR-032 | When the pipeline stage advance is blocked by flow rules, the internal task pauses with cause `awaiting_external_transition` and routes to the next-owning agent. |

### Visibility

| ID | Requirement |
|---|---|
| 026-FR-040 | Dashboard MUST visualize the configured flow as a state-machine diagram with: status nodes, allowed transitions, owning roles, gate annotations. |
| 026-FR-041 | Each external item MUST link from the dashboard to its source-of-truth URL and vice versa (driver writes a Clawie comment on the item linking to dashboard). |
| 026-FR-042 | Denied transitions MUST be visible in the dashboard with the failure reason, so the operator can fix rules or unblock the agent. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 026-NFR-001 | Transition evaluation overhead < 100ms (excluding external API latency). |
| 026-NFR-002 | Webhook ingest must converge external→internal within 5s p95. |
| 026-NFR-003 | The platform MUST track per-team force-override baseline rate over a rolling window (default 30 days). Alerts fire on a *relative* spike (e.g., this week's rate is 2× the 30-day baseline) rather than an absolute threshold — what's "healthy" depends on the team's workflow maturity. |

## User stories

- **026-US-001 [P0]** — As an operator, I connect Clawie to my Linear workspace and the next agency task picks up a Backlog item.
- **026-US-002 [P0]** — As an operator, my dev agent tries to push a ticket from "In Development" to "Ready for Release"; the driver denies, the audit logs `transition_not_allowed`, dashboard shows it.
- **026-US-003 [P0]** — As an operator, my reviewer comments `/clawie approve-spec` on a Linear issue; the spec-review gate satisfies; the spec-agent transitions to "In Development".
- **026-US-004 [P0]** — As an operator, someone manually moves a Linear card past Code Review; Clawie alerts me with the unauthorized transition + offers to revert.
- **026-US-005 [P1]** — As an operator, I install a community driver for Notion DB; the marketplace verifies signature; I configure my team to use it.
- **026-US-006 [P1]** — As an operator, I see the configured workflow as a state-machine diagram in the dashboard with owning roles and gates.
- **026-US-007 [P1]** — As an operator, I force-transition with a reason; the override is audited and commented on the external item.
- **026-US-008 [P2]** — As an operator, a rising trend of force-overrides flags my workflow design as misaligned with reality.

## Acceptance criteria

- A team configured with a Linear driver completes a full external-tracked pipeline run end-to-end.
- Unauthorized transitions denied with structured causes.
- Webhook ingest of external changes reaches Clawie within 5s p95.
- Force-override path requires reason + audits.
- The state-machine diagram renders correctly for the configured ruleset.

## Edge cases & failure modes

- External API down → driver enters `degraded`; agents pause, internal queue accumulates; resume on recovery. No silent drift.
- External system has a status Clawie's config doesn't know about → flagged as drift; operator updates `status_map` via PR (validated by 018).
- Two Clawie agents race to claim the same external item → driver-level dedupe via item id + claim TTL; second loses.
- Operator forks a transition rule with conflicting `by` role assignments → validation (spec 018) catches at merge.
- Webhook payload spoofed → drivers MUST verify signatures per the external system's spec (Linear, GitHub, Jira all sign webhooks).
- Linear/Jira API rate limits → backoff per spec 020; queue grows; operator visibility.
- Driver plugin is malicious → mitigated by signature requirement + Outcall-enforced egress allow-list per driver.

## Decision log

- **Multi-board per team** (resolved 2026-05-22): single board per team in v1. Multi-board (e.g., one team driving both Linear + Jira) deferred to v2 with its own spec.
- **Status-mapping conflicts with `flows.yaml`** (resolved 2026-05-22): when a team uses `task-management.yaml`, its flow rules are authoritative for items tracked in the external system; `flows.yaml` covers internal-only work that doesn't have an external counterpart. Validator (spec 018) enforces consistency: every stage referenced in `task-management.yaml.status_map` must exist in `flows.yaml.pipeline_stages`, and vice versa.
- **Impossible-to-automate gates** (resolved 2026-05-22): expected and supported. Gates of kind `hitl` route to the approval queue (spec 005). Gates that no automation can satisfy degrade to manual transitions — the operator performs the move with reason, force-override path audits.

## Related specs

- `003-policy-permissions` — flow rules are essentially a domain-specific extension of permission policy.
- `004-control-plane-durable-state` — external items link to internal tasks via `external_task_id` field added to the schema.
- `005-approvals-hitl` — `hitl` gates route here.
- `006-observability-audit-cost` — every transition (success and denial) emits audit + trace.
- `010-skills-plugins-lazy-load` and `024-plugin-marketplace` — community drivers published as signed plugins.
- `012-credential-broker` — driver API tokens.
- `013-team-orchestration-orgchart` — team's `task-management.yaml` lives next to `team.yaml`; ownership references the team's roles.
- `014-inter-agent-comms-tickets` — clarification that internal *tickets* (agent → agent messages) are NOT the same as external *work items* (Linear/Jira issues).
- `016-software-agency-pipeline` — pipeline stages map to external statuses via `status_map`.
- `018-config-validation-pre-merge` — task-management.yaml validation includes verifying status names actually exist in the external system at config-merge time.
