# company-v0.2.15-rc.1 release ledger

## Release identity

- Upstream: `v0.2.15@f2669c8cf62555cd92389b3f55920e9e6e7c6ff2`
- Previous company release: `company-v0.2.14-rc.1@4447ad44bd85717882e4e1c43f03ed527697b4eb`
- Upgrade branch: `codex/upgrade-v0.2.15-company`
- Requested by the company operator on 2026-10-10.

## Previous image provenance

Production runs `sub2api-company:0.2.14-rc.1` with image ID `sha256:e58103b6b38125f88013646ec06f5ff38a61d81dcb2605759c7f53faebc9533b`.

- Production has 291 applied migrations before this release.
- PostgreSQL container ID is `0d154990328e3f250c71eb4107047a2e73cc24818dc0fc2445a5a84680e8291a` and Redis container ID is `cc5b8eafb56f2e8526af90a350e10b94469a74e40e26cbe828391d1840eefd1f`.
- The application, PostgreSQL, and Redis were healthy before build and deployment work began.

## Company patch inventory and disposition

| Previous behavior | Disposition | Contract |
| --- | --- | --- |
| Persist up to 12000 redacted runes in the moderation audit excerpt. | Preserve. Upstream `v0.2.15` still persists only 240 runes. | Backend tests cover rune truncation, redaction, async persistence, and the separate cyber-policy error limit. |
| Allow non-hit retention up to 7 days while keeping the default at 3 days. | Preserve. Upstream `v0.2.15` still caps the value at 3. | Backend normalization and frontend input, load, save, and clamping tests cover the 7-day limit. |
| Temporary frontend test alignment from `company-v0.2.14-rc.1`. | Drop the company patch. | Upstream `v0.2.15` contains the complete official test update. |
| Existing account model mappings and group model allowlists. | Preserve production data. | The deployment replaces only the application container and does not recreate PostgreSQL or Redis. |

No additional company behavior is introduced by this release.

## Migration review

The target has 292 migrations. It adds one migration to the 291 currently applied in production.

| Migration | Production effect | Compatibility decision |
| --- | --- | --- |
| `242_drop_platform_check_constraints.sql` | Drops `user_platform_quotas_platform_check` and `composite_model_routes_target_platform_check`. Platform validation moves to the application catalog. | Metadata-only schema change. It briefly requires table locks. The old application can still read and write its known platform set, but rollback after new platform values are stored requires validation or database restore. |

The preflight must verify the named constraints exist. The post-deployment check must verify both constraints are absent and all stored platform values remain present in the application catalog.

## Verification

- Run backend service tests, migration and repository tests, PostgreSQL integration migration tests, frontend tests, frontend typecheck, and `git diff --check`.
- Generate `company-v0.2.15-rc.1.tests.tsv` against the fixed candidate commit.
- Build from the immutable annotated tag with explicit `VERSION=0.2.15-rc.1` and the full commit.
- Back up Compose, `.env`, application data, Redis RDB, PostgreSQL dump, migrations, and the old image.
- Rebuild only `sub2api`; preserve the PostgreSQL and Redis container IDs and start times.
- Verify all 292 migrations, `/health`, database reads, embedded version, removed constraints, application platform validation, company moderation settings, and the existing model mapping counts.
