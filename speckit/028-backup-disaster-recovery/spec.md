# 028 — Backup & Disaster Recovery

**Status:** Draft
**Constitutional anchor:** Principle V (Durable state), VI (Observable), VIII (Stability)
**Depends on:** 004, 006, 027
**Required by:** 025 (onboarding promises restore path)

## Purpose

Built-in scheduled backups to operator-configured object storage (S3, Cloudflare R2, or any S3-compatible). Covers: control plane DB (SQLite or Postgres), audit log integrity proof, secrets backend metadata, git config repos, marketplace artifact references. Restore is one command. No hand-rolled cron, no operator-written scripts in the happy path.

## Why this matters

Indie operators won't write backup scripts; they'll skip them entirely. Producing a working DR posture out of the box is the difference between "ran the demo" and "trusts the platform with real work". Aligns with the user's stated stability priority.

## What gets backed up

| Artifact | Frequency | Rationale |
|---|---|---|
| Control plane DB (SQLite or Postgres) | hourly incremental, daily full | Tasks, approvals, audit, ledger live here |
| Audit hash-chain checkpoint | on every backup | Tamper-evidence root preserved separately |
| Root git config repo | on commit (via post-commit hook) + nightly | Captures `clawie.yaml` and submodule pointer state |
| Each team & agent git repo | nightly (mirror push to backup remote) | Versioned config |
| Secrets backend metadata only (aliases, scopes, kind — NEVER values) | nightly | Restore can reconstruct slots; values re-injected by operator |
| Workspace project artifacts (`projects/<id>/artifacts/`) | daily | Pipeline outputs |
| Outcall rulesets in git | covered by root git backup | — |
| Marketplace plugin mirror state | nightly | Pinned versions; binaries re-fetched from marketplace on restore |

## What is explicitly NOT backed up

- **Secret values.** Backup contains aliases + metadata only. Restore process requires operator to re-inject values (via Vault, env, etc.).
- **Customer codebase working trees (`/workspace/` mounts).** These belong to the customer, not Clawie.
- **Plugin artifact binaries.** Re-fetched from the marketplace on restore. (Tarballs ARE backed up if `backup.include_plugin_artifacts: true` is set, for air-gapped recovery.)
- **Container images.** Re-pulled from registry on restore.

## Architecture

Backup runs as a platform schedule (spec 027 — `platform_schedules`). Two operating modes:

| Mode | What happens |
|---|---|
| **Built-in** (default) | Platform writes a tarball + manifest to configured object storage. Encrypted with operator-provided key. |
| **External** | Operator configures their own backup tool against the documented file paths; platform provides only the dump + checksum command. |

```
┌──────────────────────────────────────────┐
│  Backup schedule (via spec 027)          │
│  cron: "0 2 * * *"  handler: backup.run   │
└────────────────────────┬─────────────────┘
                         ↓
┌──────────────────────────────────────────┐
│  Backup runner                            │
│   1. quiesce: stop new claims, drain     │
│      in-flight to safe checkpoint         │
│   2. dump: SQLite snapshot or            │
│      pg_dump (if Postgres)               │
│   3. include: audit chain root,          │
│      git refs, secrets metadata          │
│   4. encrypt: age/sops with operator key  │
│   5. upload: PUT to S3/R2 with retention │
│   6. verify: read back, checksum, log    │
│   7. resume: control plane unfrozen      │
└──────────────────────────────────────────┘
```

Quiesce window is brief (typically <5s on healthy SQLite, <30s on Postgres with WAL). All in-flight tasks survive — only new task claims are paused during quiesce.

## Restore

```
clawie restore --from s3://clawie-backups/2026-05-22/ --key <age-key>
   → verify checksum + audit chain root
   → stop platform if running
   → restore DB, git refs, secrets metadata
   → prompt operator for missing secret values
   → run validator (spec 018)
   → run smoke test (spec 025)
   → start platform if smoke passes
```

## Functional requirements

### Configuration

| ID | Requirement |
|---|---|
| 028-FR-001 | Backup destination MUST be configurable: S3, R2, GCS (S3 API), MinIO, local filesystem. Connection via standard env (AWS_*, CF_R2_*, etc.). |
| 028-FR-002 | Backup encryption MUST use an operator-provided key (age or sops-compatible). Platform MUST refuse to backup without one. |
| 028-FR-003 | Retention policy MUST be configurable: `keep_daily: N` / `keep_weekly: N` / `keep_monthly: N` (defaults 7/4/12). Old backups pruned by the runner. |
| 028-FR-004 | Backup schedule (spec 027 platform_schedules) defaults to `0 2 * * *` (02:00 UTC daily); configurable. |
| 028-FR-005 | An operator MAY trigger an ad-hoc backup via `clawie backup now` or dashboard button. |

### Execution

| ID | Requirement |
|---|---|
| 028-FR-010 | Backup MUST quiesce the control plane (pause new task claims) for the duration of the DB dump only. Quiesce MUST log start/end. |
| 028-FR-011 | Backup MUST capture: DB dump, audit chain root hash, git ref state for all configured repos, secrets backend metadata (aliases/scopes/kinds only), platform version, manifest with checksums. |
| 028-FR-012 | Each backup artifact MUST be encrypted with the operator key before upload. |
| 028-FR-013 | Backup MUST verify-on-completion: read back, decrypt, compute checksums, compare to manifest. Verification failure marks the backup invalid + alerts operator. |
| 028-FR-014 | Failed backups MUST be audit-logged with cause; the platform MUST NOT silently fail. |
| 028-FR-015 | Backup runner MUST NOT exceed configured `max_runtime_minutes` (default 30). Exceeding aborts the backup with cause. |

### Restore

| ID | Requirement |
|---|---|
| 028-FR-020 | `clawie restore --from <uri> --key <keyfile>` MUST be the single canonical command. |
| 028-FR-021 | Restore MUST verify checksum + audit chain root before applying anything. Mismatch aborts cleanly. |
| 028-FR-022 | Restore MUST run the validator (spec 018) and smoke test (spec 025) before declaring success. |
| 028-FR-023 | Restore MUST prompt for any required secret values that backup metadata says existed but were not stored. |
| 028-FR-024 | An operator MAY run `clawie restore --dry-run` to verify a backup without applying. |

### Observability

| ID | Requirement |
|---|---|
| 028-FR-030 | Dashboard MUST show: last successful backup time, last attempt time, last failure cause, retention status, configured destination, configured retention policy. |
| 028-FR-031 | An operator MUST receive an alert when a scheduled backup fails or the time since last successful backup exceeds threshold (default 36h). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 028-NFR-001 | Backup of a typical install (≤100k tasks, ≤10k audit events) MUST complete in <5 min. |
| 028-NFR-002 | Quiesce window p95 < 30s on Postgres, < 5s on SQLite. |
| 028-NFR-003 | Backup runner overhead during normal operation < 5% CPU on the platform host. |
| 028-NFR-004 | Restore of a typical backup MUST complete in <10 min. |

## User stories

- **028-US-001 [P0]** — As an operator, I configure an S3 bucket and an age key during install; nightly backups just happen.
- **028-US-002 [P0]** — As an operator, my platform host dies; I provision a new one, run `clawie restore --from ... --key ...`, and within 10 minutes I'm back up.
- **028-US-003 [P0]** — As an operator, a backup fails; dashboard alerts me with the cause.
- **028-US-004 [P1]** — As an operator, I run `clawie backup now` before a risky upgrade.
- **028-US-005 [P1]** — As an operator, restore prompts me to re-inject secret values; I do, validator passes, smoke test passes, platform starts.
- **028-US-006 [P2]** — As an operator, I configure 1y retention and the runner prunes old backups automatically.

## Acceptance criteria

- End-to-end backup → restore round-trip succeeds on a sample install.
- Failed backups produce alerts and audit entries.
- Restore prompts for missing secrets, runs validator + smoke test, and refuses on failure.
- Encryption key is mandatory; platform refuses to back up without one.
- Quiesce window stays within target.

## Edge cases & failure modes

- **Object storage unreachable** — backup queued for retry; alert after N consecutive failures.
- **Encryption key lost** — backups become unrecoverable; documented prominently in install + docs.
- **Restore to a different platform version** — restore command MUST detect and either auto-migrate (forward) or refuse (backward) with clear guidance.
- **Schema drift between backup and current code** — restore runs schema migrations forward; refuses backward-incompatible restores without `--force`.
- **Partial backup (DB dump succeeded, git mirror failed)** — entire backup marked invalid; alert; operator decides.
- **Backup during heavy load** — quiesce extends; alert if exceeds 60s. Operator may configure low-traffic window.

## Open questions

- **Air-gapped operator** — should backup support local-disk destination cleanly? (Lean: yes, with rotation handled by the runner.)
- **Backup encryption: age vs sops?** Both work; standardize one for v1 to reduce surface. (Lean: age — simpler.)
- **Differential / incremental backups?** Out of scope v1; full + retention is simpler. Revisit if backup size becomes a problem.

## Related specs

`004-control-plane-durable-state`, `006-observability-audit-cost`, `012-credential-broker`, `018-config-validation-pre-merge`, `025-onboarding-install`, `027-scheduler-recurring-work`.
