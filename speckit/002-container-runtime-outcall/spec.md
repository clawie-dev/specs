# 002 — Container Runtime + Outcall Integration

**Status:** Draft
**Constitutional anchor:** Principle I (Layered), IV (Default-deny), X (Pluggable); Security constraints 1, 2, 5, 8
**Depends on:** 001
**Required by:** 003, 004, 008, 012, 015

## Purpose

Every agent task runs inside an ephemeral Docker container with optional Outcall-controlled egress. The container is disposable; state lives in the control plane; the workspace mount lets the agent operate on code without persistent runtime state.

## Why this matters

Survey consensus: prompt-only sandboxing fails. Hostile multi-tenancy in one process fails. The only durable security boundary is OS-level isolation. Outcall extends that to the network edge with rule-based egress, which is what NemoClaw approximates with k3s — but Clawie does it natively and configurably.

## Functional requirements

| ID | Requirement |
|---|---|
| 002-FR-001 | Each agent task MUST spawn a fresh Docker container (no reuse of containers across tasks in v1). |
| 002-FR-002 | The container image MUST be a published `clawie/agent-runtime:<version>` image plus optional per-agent overlay. |
| 002-FR-003 | The container MUST mount `/agent` (read-only) from the agent's git worktree pinned to the recorded SHA. |
| 002-FR-004 | The container MUST mount `/workspace` (read-write or read-only per config) for the project codebase. |
| 002-FR-005 | The container MUST mount `/skills` (lazy-populated) for installed skill artifacts. |
| 002-FR-006 | The container MUST receive no host network access by default. |
| 002-FR-007 | All egress MUST traverse Outcall when enabled. When disabled, the container MUST still be on a separate Docker network with no access to other agent containers or the host. |
| 002-FR-008 | Resource limits MUST be set: CPU shares, memory, PIDs, disk, run-time wall clock. |
| 002-FR-009 | The container's filesystem MUST be ephemeral (tmpfs for `/tmp`, no writable layer persisted between runs except via mounts). |
| 002-FR-010 | Lifecycle: `spawn → run → finalize → reap`. On reap, the container's logs, exit code, and resource metrics MUST be captured to the audit and cost ledger. |
| 002-FR-011 | A container MUST emit liveness pings to the control plane on a configurable interval; missed pings beyond threshold trigger stuck-agent recovery (see 020). |
| 002-FR-012 | Multiple container runtimes MUST be pluggable: Docker (default), Podman (supported), containerd (supported), runc/gVisor/Kata (optional security overlays). |
| 002-FR-013 | An operator MUST be able to inspect a running container's live logs and state via dashboard or `clawie task logs <id>`. |
| 002-FR-014 | Killing a container MUST NOT lose task state — state lives in the control plane; the task is marked failed/aborted with cause captured. |
| 002-FR-015 | Outcall integration modes: `disabled`, `sidecar`, `host`. Mode declared per-team in `team.yaml`. |
| 002-FR-016 | When Outcall is `disabled`, the dashboard MUST display a prominent warning per team, and team-level setup logs MUST record the operator's acknowledgment. |
| 002-FR-017 | The base image MUST be reproducible (pinned base, locked dependencies, signed). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 002-NFR-001 | Cold container spawn p95 MUST be <5s on commodity hardware (1 CPU, 2 GB RAM target VPS). |
| 002-NFR-002 | The base image MUST be <500 MB compressed. |
| 002-NFR-003 | Reap MUST be guaranteed even on platform crash (orphan reaper on next boot). |

## Outcall integration modes

| Mode | Topology | Use case |
|---|---|---|
| `disabled` | Docker network isolation only; no egress rules. | Local dev. Hard warning surfaced. |
| `sidecar` | Outcall as a sidecar container per agent network. Per-team Outcall ruleset. | Standard production. |
| `host` | One Outcall daemon on host; agent containers route via host gateway. | Multi-agent on one host sharing one policy. |

Outcall's role within Clawie:
- Egress allow-list per (agent, team, project).
- Credential injection at egress: Clawie hands Outcall the secret reference; Outcall transparently injects on outbound requests.
- Egress audit: every egress flow logged with destination, bytes, decision.

## User stories

- **002-US-001 [P0]** — As an operator, I see every agent container spawn, run, and reap in the audit log.
- **002-US-002 [P0]** — As an operator, when I disable Outcall on a team, the dashboard surfaces a permanent warning until I re-enable it.
- **002-US-003 [P0]** — As an operator, a runaway agent is terminated by its wall-clock limit and shows up in audit with `cause: timeout`.
- **002-US-004 [P1]** — As an operator, I switch from Docker to Podman by editing `clawie.yaml`, and the next task spawns on the new runtime.
- **002-US-005 [P1]** — As an operator, a container that loses heartbeat is marked stuck and recovered per spec 020.

## Acceptance criteria

- Spawn → run → reap round-trip works in <5s for a no-op task.
- A container with no Outcall and no Docker network egress cannot reach any external host.
- A container with Outcall in sidecar can reach only allow-listed hosts and gets credential injection.
- Killing the API mid-task does not orphan containers (verified by orphan reaper on next boot).

## Edge cases & failure modes

- Docker daemon unavailable → control plane marks all queued tasks as `infrastructure_unavailable`; surfaces dashboard banner.
- Image pull failure → task marked failed with cause; not retried infinitely (max 3, then escalates).
- Outcall sidecar crash mid-task → task marked `awaiting_dependency`; resumes when Outcall is healthy.
- Workspace mount race → container holds workspace via lockfile in control plane; concurrent mount denied.
- Container OOM → cause-of-failure `oom`; resource limits surfaced for operator to tune.

## Security notes

- The container MUST NOT have access to the Docker socket.
- The container MUST run as non-root (UID > 1000) by default.
- Capabilities MUST be dropped: `CAP_DROP=ALL` plus only `NET_BIND_SERVICE` if needed.
- Seccomp profile MUST be the Docker default-deny minus needed syscalls.
- Optional: runc → gVisor for higher-isolation profiles per team config.

## Related specs

`003-policy-permissions`, `004-control-plane-durable-state`, `012-credential-broker`, `015-workspace-codebase-mount`, `020-stability-recovery`.
