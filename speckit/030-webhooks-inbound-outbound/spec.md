# 030 — Webhooks (Inbound & Outbound)

**Status:** Draft
**Constitutional anchor:** Principle IV (Default-deny), V (Durable), VI (Observable), Security constraints 9, 10
**Depends on:** 002, 003, 004, 006, 012, 023, 026
**Required by:** none

## Purpose

A single, defensible specification for **webhook delivery** (Clawie → external) and **webhook ingest** (external → Clawie). Signed, replay-protected, retried with backoff, ordered per-resource, delivered durably or dead-lettered. Resolves audit finding D5.

## Why this matters

Spec 023 mentions outbound webhooks; spec 026 mentions inbound from Linear/Jira. Without a unified spec, every integration reinvents signing, retries, ordering, dead-lettering. The result: silent failures (a Linear event lost in transit) or duplicate side-effects (a Stripe event processed twice). This spec is small but load-bearing.

## Vocabulary

| Term | Meaning |
|---|---|
| **Outbound webhook** | Clawie POSTs an event to an operator-configured URL (Slack, Discord, custom listener). |
| **Inbound webhook** | An external system POSTs an event to Clawie (Linear status change, GitHub PR merge, Stripe payment). |
| **Endpoint** | A registered destination (outbound) or source (inbound) with associated config: URL, signing key, allowed event types, retry policy. |
| **Event** | A structured message. Schema-versioned. |

## Outbound webhooks

### Architecture

```
Internal event → outbound dispatcher
                  ↓
               sign (HMAC-SHA256 over body + headers)
                  ↓
               attempt delivery (retry with jitter)
                  ↓ ↓ ↓
               2xx / 4xx / 5xx
                  │   │   └── retry per policy
                  │   └────── dead-letter (no retry on client error)
                  └────────── success (audit)
```

### Functional requirements (outbound)

| ID | Requirement |
|---|---|
| 030-FR-001 | Outbound endpoints MUST be configured per-team (in the team repo) or platform-wide (in `clawie.yaml`). Per-team config reviewed via standard PR flow. |
| 030-FR-002 | Each endpoint declares: URL, signing-key alias (resolved via credential broker spec 012), subscribed event types, retry policy (default: 5 attempts, exponential backoff 1s/4s/16s/64s/256s with ±20% jitter), max payload size (default 1 MB). |
| 030-FR-003 | Every outbound delivery MUST be signed: header `X-Clawie-Signature: sha256=<hex>` = HMAC-SHA256(signing_key, request_body). Header `X-Clawie-Timestamp: <unix-seconds>`. Header `X-Clawie-Event: <event-type>`. Header `X-Clawie-Delivery: <uuid>` for idempotency. |
| 030-FR-004 | Delivery state MUST be a durable record: `(endpoint_id, event_id, attempt, status, last_response_code, last_attempted_at, next_retry_at)`. State survives platform restart. |
| 030-FR-005 | Retries MUST use exponential backoff with jitter. Permanent failures (4xx client errors except 408/429) dead-letter immediately; transient failures (5xx, network, 408, 429) retry per policy. After max attempts → dead-letter. |
| 030-FR-006 | A dead-letter queue MUST be visible in the dashboard with one-click redrive. Dead-lettered events retain payload + cause for N=30 days (configurable). |
| 030-FR-007 | Ordering: events for the same resource (project_id, agent_id) MUST deliver in order. Cross-resource events MAY parallelize. Per-endpoint concurrency cap configurable (default 4). |
| 030-FR-008 | Egress MUST traverse Outcall (when enabled). Endpoint URLs MUST resolve to operator-allow-listed hosts; refused otherwise with cause `egress_not_allowed`. |
| 030-FR-009 | Outbound dispatcher MUST never block the originating internal event. Dispatch is asynchronous; the internal task continues without waiting. |
| 030-FR-010 | Payload schema MUST be versioned: every event carries `schema_version: "v1"`. Schema changes are SemVer; v2 events delivered alongside v1 during a deprecation window. |
| 030-FR-011 | An endpoint MAY declare `event_filter` (boolean expression over event fields) to subscribe to a subset. |

## Inbound webhooks

### Architecture

```
external system → POST /v1/webhooks/<source>/<endpoint_id>
                                                ↓
                                  signature verify (per source contract)
                                                ↓
                                  replay check (nonce / timestamp window)
                                                ↓
                                  persist raw event (audit)
                                                ↓
                                  dispatch to handler as durable task (spec 004)
                                                ↓
                                  200 ACK to sender (before handler runs)
```

### Functional requirements (inbound)

| ID | Requirement |
|---|---|
| 030-FR-020 | Inbound endpoints MUST be registered per source (Linear, Jira, GitHub, Stripe, custom). Each endpoint has: source (driver identifier), signing-key alias, allowed event types, timestamp-skew tolerance (default ±5 min), nonce window (default 10 min). |
| 030-FR-021 | Signature verification MUST match the source's documented scheme (Stripe HMAC, GitHub HMAC, Linear HMAC, etc.). The platform delegates to the driver's verifier (spec 026 driver interface) for systems with their own conventions. |
| 030-FR-022 | Replay protection: every accepted event MUST be recorded by its `(source, delivery_id_or_nonce)`. Repeat receipt within the nonce window returns `200 OK (duplicate)` without re-dispatching. |
| 030-FR-023 | Timestamp-skew check: events with timestamps outside the tolerance window MUST be refused with `400` and audited as `webhook_timestamp_skew`. |
| 030-FR-024 | The platform MUST `200 ACK` the sender **before** running the handler. Handler runs as a durable task (spec 004); failures do not propagate to the sender (would cause source to retry). |
| 030-FR-025 | Raw payload + headers MUST be persisted to audit for diagnostic purposes (with secret-shape redaction per spec 012-FR-011). Retention follows audit retention policy (spec 006). |
| 030-FR-026 | Handler dispatch MUST be idempotent on `(source, delivery_id)`. Source retries reaching the same event MUST not double-process. |
| 030-FR-027 | Unknown or malformed events MUST `400` with audit entry, not silently 200. |
| 030-FR-028 | Inbound URLs MUST be operator-configurable (path scheme `/v1/webhooks/<source>/<endpoint_id>` standard). Unknown source paths return `404`. |
| 030-FR-029 | An inbound endpoint MAY require an additional shared-secret header (`X-Clawie-Token`) beyond the source's own signing, for defense in depth. |

## Cross-cutting

### Observability

| ID | Requirement |
|---|---|
| 030-FR-040 | Dashboard MUST surface per-endpoint: total delivered, retried, dead-lettered, pending, average latency, last successful delivery time. |
| 030-FR-041 | Audit MUST log every: registration, delivery attempt, success, failure, dead-letter, redrive, signature-verify result. |
| 030-FR-042 | An operator MUST be able to inspect any single delivery: request/response, headers (redacted), retry trail, final outcome. |
| 030-FR-043 | Alert thresholds: per-endpoint dead-letter rate > N per hour, signature-verify failure rate > M per minute, oldest pending delivery > T minutes. All configurable. |

### Security

| ID | Requirement |
|---|---|
| 030-FR-050 | Signing keys MUST NEVER be returned by any API endpoint. Aliases and metadata only (per spec 012). |
| 030-FR-051 | Inbound endpoints MUST be rate-limited per `(source, source_ip)` to prevent abuse. Default: 100 req/s per source IP; exceedance returns `429`. |
| 030-FR-052 | Outbound endpoint URLs MUST NOT resolve to private/link-local/loopback addresses unless explicitly opted in (`allow_private_targets: true` per endpoint with audit). Default: SSRF protection enabled. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 030-NFR-001 | Outbound dispatch latency p95 < 200ms from event publication to first delivery attempt. |
| 030-NFR-002 | Inbound 200-ACK p95 < 100ms (handler runs asynchronously after ACK). |
| 030-NFR-003 | Delivery state MUST scale to ≥1M historical delivery records without query degradation. |

## User stories

- **030-US-001 [P0]** — As an operator, every approval triggers a Slack webhook signed with my Slack secret; replay protection prevents duplicates.
- **030-US-002 [P0]** — As an operator, a Slack outage causes deliveries to retry; after 5 attempts they dead-letter; I redrive from the dashboard once Slack is back.
- **030-US-003 [P0]** — As an operator, Linear sends a status-change webhook; signature verifies, the driver picks it up, the matching internal task transitions.
- **030-US-004 [P1]** — As an operator, a malformed webhook arrives with bad signature; the platform 400s, audit logs the attempt with cause `signature_invalid`, no handler runs.
- **030-US-005 [P1]** — As an operator, I add a custom inbound endpoint for our internal CI system; declared signing key, audit shows verified deliveries.
- **030-US-006 [P2]** — As an operator, my outbound endpoint URL points to a `192.168.x.x` host (misconfiguration); platform refuses with SSRF protection cause.

## Acceptance criteria

- End-to-end signed delivery + verification works for at least: Slack outbound, GitHub inbound, Linear inbound, Stripe inbound.
- Retry + dead-letter + redrive cycle proven.
- Replay protection prevents double-handling.
- SSRF protection blocks private targets unless explicitly opted in.
- Ordering guarantees verified for same-resource events.

## Edge cases & failure modes

- **Outbound endpoint URL resolves slowly (DNS)** — per-attempt timeout; counts as transient.
- **Inbound payload >1 MB** — refused with 413; audit-logged.
- **Source clock drift causes timestamp skew** — operator raises tolerance per endpoint.
- **Signing key rotation mid-flight** — broker maintains both old and new key for grace window; both accepted during overlap.
- **Network partition between control plane and external target** — events queue durably; resume on recovery.
- **Operator deletes an endpoint with pending deliveries** — pending dead-lettered with cause `endpoint_deleted`; not silently dropped.

## Decision log

- **Signing algorithm** (resolved 2026-05-22): HMAC-SHA256 universally for Clawie-emitted webhooks. Inbound matches whatever the source specifies (HMAC variants are standard across sources).
- **ACK-then-handle for inbound** (resolved 2026-05-22): yes — sender sees fast 200 ACK; handler runs as durable task. Prevents source retry storms during handler issues.
- **Dead-letter redrive UX** (resolved 2026-05-22): dashboard one-click + CLI `clawie webhook redrive <delivery_id>`. Bulk redrive supported with confirmation.

## Related specs

`002-container-runtime-outcall`, `003-policy-permissions`, `004-control-plane-durable-state`, `006-observability-audit-cost`, `012-credential-broker`, `023-rest-api`, `026-task-management-drivers-flow-rules`.
