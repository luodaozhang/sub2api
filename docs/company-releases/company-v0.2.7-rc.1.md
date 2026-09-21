# company-v0.2.7-rc.1 release ledger

## Release identity

- Upstream: `v0.2.7@aea725f2ea644d5592d0bbb1d63b607efa7e200a`
- Previous company release: `company-v0.2.1-rc.2@9e0c3180a1d71e08685fab040172d4b8f179369e`
- Upgrade branch: `codex/upgrade-v0.2.7-company`
- Requested by the company operator on 2026-09-21.
- The operator accepted the upstream model allowlist request-enforcement behavior and explicitly waived old-application and database rollback compatibility.

## Previous image provenance

Production runs `sub2api-company:0.2.1-rc.2` with image ID `sha256:0763a0a4e79988175157063a3bbc60ecb3c6d08f06fa6b6737b3ed6db2b1b01e`.

- The previous company tag resolves to `9e0c3180a1d71e08685fab040172d4b8f179369e`.
- Production has 281 applied migrations before this release.
- PostgreSQL container ID is `0d154990328e3f250c71eb4107047a2e73cc24818dc0fc2445a5a84680e8291a` and Redis container ID is `cc5b8eafb56f2e8526af90a350e10b94469a74e40e26cbe828391d1840eefd1f`.

## Company patch inventory and disposition

| Previous behavior | Disposition | Contract |
| --- | --- | --- |
| Persist up to 12000 redacted runes in the moderation audit excerpt. | Preserve. Upstream `v0.2.7` still persists only 240 runes. | Backend tests cover rune truncation, redaction, async persistence, and the separate cyber-policy error limit. |
| Allow non-hit retention up to 7 days while keeping the default at 3 days. | Preserve. Upstream `v0.2.7` still caps the value at 3. | Backend normalization and frontend input, load, save, and clamping tests cover the 7-day limit. |
| Treat the configured model list as display-only. | Do not preserve. Use upstream `v0.2.7` request enforcement. | Requests outside an enabled group allowlist are rejected before model mapping. The operator reviewed the 30-day production impact on 2026-09-21. |

No additional company behavior is introduced by this release.

## Migration review

The target has 286 migrations. It adds five migration files to the 281 currently applied in production.

| Migration | Production effect | Compatibility decision |
| --- | --- | --- |
| `235_group_model_allowlist.sql` | Renames `groups.models_list_config` to `groups.model_allowlist`; the same configuration now controls listing and request admission. | Old application compatibility is intentionally not retained. |
| `236_group_model_allowlist_repair.sql` | Repairs partial rename states and guarantees a non-null JSONB `model_allowlist`. | Forward repair only. |
| `237_add_minimax_platform.sql` | Adds `minimax` to platform check constraints. | Forward schema change. |
| `238_opencode_go.sql` | Adds `opencode_go` to platform and OAuth provider constraints. | Forward schema change. |
| `238_purge_unlimited_user_platform_quotas.sql` | Deletes quota rows whose daily, weekly, and monthly limits are all null. Production preflight found 281 matching rows. | Deletion is accepted. No compatibility preservation or app-only rollback is promised. |

The release does not add compatibility migrations. Returning to the old application would require a separately approved database restore from the pre-upgrade backup.

## Verification

- Run backend service tests, migration and repository tests, PostgreSQL integration migration tests, frontend tests, frontend typecheck, and `git diff --check`.
- Generate `company-v0.2.7-rc.1.tests.tsv` against the fixed candidate commit.
- Build from the immutable annotated tag with explicit `VERSION=0.2.7-rc.1` and the full commit.
- Back up Compose, `.env`, application data, Redis RDB, PostgreSQL dump, migrations, and the old image. This is a deployment safety record, not an app-only rollback guarantee.
- Rebuild only `sub2api`; preserve the PostgreSQL and Redis container IDs and start times.
- Verify all 286 migrations, `/health`, database reads, embedded version, `model_allowlist`, the new platform constraints, and deletion of all-null unlimited quota rows.
