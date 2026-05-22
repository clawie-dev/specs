# 029 — Upgrades & Migrations

**Status:** Draft
**Constitutional anchor:** Principle III (Validated before merged), VIII (Stability)
**Depends on:** 004, 018, 025, 028
**Required by:** none

## Purpose

Upgrading Clawie is not a side project. Every release ships with: pre-upgrade snapshot, schema migration plan, agent-runtime compatibility check, smoke test on the new version, and a one-command rollback. Goal: no operator left stranded by a bad upgrade. Resolves audit finding D3.

## Why this matters

The user's stated quality bar: "we should see failures all the time, like OpenClaw. My openclaw often stops responding etc, cause of various reasons." Upgrades are a top-five failure mode in long-lived agent platforms. Specced now, defended forever.

## Versioning

| Layer | Scheme | Cadence | Compatibility |
|---|---|---|---|
| Control plane (the AdonisJS app) | SemVer (`MAJOR.MINOR.PATCH`) | Monthly minor, weekly patch | Breaking changes only on MAJOR. Stale-skew tolerated within MINOR. |
| Agent-runtime image | SemVer pinned per minor of control plane | Per release | Compatibility matrix published; validator enforces. |
| Spec set | SemVer (constitution.version) | Quarterly review | Constitutional changes require RFC. |
| Schemas (`@clawie-dev/schemas`) | SemVer | Per change | Major bump = breaking schema. |
| Plugins / skills / drivers | SemVer per-plugin | Author-driven | Marketplace enforces minimum tier. |

## Upgrade flow (canonical)

```
clawie upgrade [--to <version>] [--dry-run]
   ↓
1. resolve target version + agent-runtime + schema matrix
2. compatibility-check vs current install
   ├── agents using deprecated APIs flagged
   └── plugins requiring newer host flagged
3. pre-upgrade snapshot (spec 028 — adhoc backup)
4. read changelog + breaking notes; require operator OK if breaking
5. quiesce control plane (drain claims, no new ones)
6. pull new agent-runtime image (warm cache before swap)
7. apply forward migrations (Lucid migrations on SQLite/Postgres)
8. swap control plane binary
9. boot validator (spec 018) — full validation pass
10. smoke test (spec 025 — toy task end-to-end)
11. if any step fails → rollback automatically (see below)
12. unquiesce; log upgrade success
```

## Rollback

```
clawie upgrade rollback [--to <version>]
   ↓
1. verify rollback target is the previous snapshot (spec 028)
2. quiesce
3. apply backward migrations (only if marked "reversible")
4. restore from snapshot
5. swap binary
6. validator + smoke
7. unquiesce
```

If a migration is irreversible (rare, but real — e.g., dropped column), rollback restores the snapshot instead of replaying inverse migrations. Operator is told this in the changelog.

## Functional requirements

### Upgrade preconditions

| ID | Requirement |
|---|---|
| 029-FR-001 | `clawie upgrade` MUST check: target version exists, signed by the release key, present in the version registry, with a published compatibility matrix. |
| 029-FR-002 | The platform MUST refuse to upgrade if the current state is unhealthy: any task in `running` for >24h, any pending approval older than its decision window, validator errors on current config. Operator may `--force` with audit reason. |
| 029-FR-003 | Compatibility check MUST inspect: agent-runtime image matches, all plugins resolve against the new host version, all schemas backward-compatible (or operator-acknowledged breaking). |
| 029-FR-004 | Breaking changes MUST require explicit operator confirmation (`--accept-breaking`) and surface the changelog excerpt. |

### Migrations

| ID | Requirement |
|---|---|
| 029-FR-010 | All DB schema changes MUST use Lucid migrations under `database/migrations/`. Migration files MUST be timestamped, immutable once shipped, and individually reversible OR explicitly marked `irreversible: true` with a reason. |
| 029-FR-011 | Forward migrations MUST be idempotent (re-applying is a no-op). Backward migrations same. |
| 029-FR-012 | Migrations MUST work identically on SQLite and Postgres. CI runs the full migration suite against both backends on every PR. |
| 029-FR-013 | Long-running migrations (estimated >30s) MUST be marked and split into an additive phase + a cleanup phase (online-schema-migration pattern). |
| 029-FR-014 | The platform MUST refuse to start if pending migrations exist; operator runs `clawie migrate` explicitly. Auto-migrate on boot is dev-mode only. |

### Pre-upgrade snapshot

| ID | Requirement |
|---|---|
| 029-FR-020 | Upgrade MUST trigger an adhoc backup (spec 028) before applying any migration or binary swap. Backup MUST verify successfully before proceeding. |
| 029-FR-021 | If pre-upgrade backup fails, upgrade aborts cleanly with no state changes. |

### Smoke + rollback

| ID | Requirement |
|---|---|
| 029-FR-030 | After binary swap + migration, the platform MUST run validator + a tiny smoke task end-to-end. Failure triggers automatic rollback. |
| 029-FR-031 | Automatic rollback restores from the pre-upgrade snapshot and reverts the binary swap. Must complete in <15 min total. |
| 029-FR-032 | `clawie upgrade rollback` MUST be available manually for up to N=14 days (configurable) post-upgrade. After the window, manual restore from a snapshot is the path. |

### Version skew (control plane vs agent-runtime)

| ID | Requirement |
|---|---|
| 029-FR-040 | The control plane MUST publish, for each released minor, the set of compatible agent-runtime versions (e.g., `clawie-control-plane@1.4.x` supports `agent-runtime@1.4.x` and `agent-runtime@1.3.x`). |
| 029-FR-041 | Validator MUST refuse to spawn a container with an incompatible agent-runtime; clear remediation pointer. |
| 029-FR-042 | Agent-runtime supports running against control plane up to one minor newer (forward compat) and one minor older (back compat). Beyond that = upgrade required. |

### Plugin / skill compatibility

| ID | Requirement |
|---|---|
| 029-FR-050 | Each plugin declares `requires_clawie: ">=X.Y, <Z.0"` in its manifest. Marketplace and CLI refuse install/upgrade if incompatible. |
| 029-FR-051 | Yanked-due-to-incompatibility plugins surface in the dashboard with a clear "incompatible with current version" badge. |

### Documentation

| ID | Requirement |
|---|---|
| 029-FR-060 | Every release MUST ship a `CHANGELOG.md` entry. Breaking changes MUST appear in a dedicated `BREAKING.md` section with: what broke, why, migration steps, rollback steps. |
| 029-FR-061 | The dashboard MUST surface a banner whenever a new version is available, including a one-line summary of breaking changes (if any). |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 029-NFR-001 | A non-breaking upgrade MUST complete in <10 min on a default install. |
| 029-NFR-002 | Migrations MUST not lose audit data or cost ledger entries under any circumstances. |
| 029-NFR-003 | Compatibility matrix lookups MUST be <50ms in dashboard / CLI. |

## User stories

- **029-US-001 [P0]** — As an operator, `clawie upgrade` checks compatibility, snapshots, migrates, smoke-tests, and reports success — all without touching git or remembering steps.
- **029-US-002 [P0]** — As an operator, an upgrade fails the smoke test; the platform rolls back automatically and tells me what broke.
- **029-US-003 [P0]** — As an operator, I'm warned ahead of a breaking change; the dashboard banner links to the migration guide.
- **029-US-004 [P1]** — As an operator, I run `clawie upgrade rollback` two days post-upgrade because of an issue; restore completes and I'm back on the previous version.
- **029-US-005 [P1]** — As an operator, my plugin is incompatible with the new minor; install/upgrade refuses with a clear pointer to the plugin's release timeline.
- **029-US-006 [P1]** — As a contributor, my PR adds a migration; CI runs it forward + backward on both SQLite and Postgres.

## Acceptance criteria

- Upgrade path tested end-to-end across minor and patch bumps.
- Rollback works for any non-irreversible migration.
- Smoke failure triggers automatic rollback.
- Compatibility matrix is published per release, queryable via API and CLI.
- Audit data + cost ledger survive every upgrade unchanged.

## Edge cases & failure modes

- **Pre-upgrade backup destination unreachable** — upgrade refused; clear remediation.
- **Migration succeeds but smoke fails** — automatic rollback (restore from snapshot).
- **Operator skips multiple minor versions** — multi-hop upgrade unsupported in v1; operator must upgrade minor-by-minor with audit gate at each step.
- **In-flight tasks during quiesce** — drained to safe checkpoint; tasks resume post-upgrade.
- **Plugin author yanked a version that's installed** — dashboard shows it; upgrade may continue or refuse based on policy.

## Decision log

- **Multi-hop upgrades** (resolved 2026-05-22): not supported in v1. Operators upgrade minor-by-minor. Simpler test surface; reduces compatibility-matrix combinatorics.
- **Automatic boot-time migrations** (resolved 2026-05-22): dev mode only. Production requires explicit `clawie migrate` step so operators know what's happening.
- **Snapshot retention for rollback** (resolved 2026-05-22): minimum N=14 days post-upgrade, configurable. Older snapshots subject to the retention policy in spec 028.

## Related specs

`004-control-plane-durable-state`, `018-config-validation-pre-merge`, `024-plugin-marketplace`, `025-onboarding-install`, `028-backup-disaster-recovery`.
