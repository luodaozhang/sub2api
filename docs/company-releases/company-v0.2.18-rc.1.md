# company-v0.2.18-rc.1 release ledger

## Identity and baseline

- Requested on 2026-10-27. Upstream: `v0.2.18@de86961b627e48db75d61fff8c05535d44a00cbe`.
- Branch: `codex/upgrade-v0.2.18-company`; fork: `https://github.com/luodaozhang/sub2api`.
- Previous company tag: `company-v0.2.15-rc.1@812394afcf398ae7ca3877f67f588e8d0e81ec3c`.
- Its upstream and the target merge-base are both `v0.2.15@f2669c8cf62555cd92389b3f55920e9e6e7c6ff2`.
- Live image: `sub2api-company:0.2.15-rc.1`, ID `sha256:66db15f4007fe031528aa204b69194e1190c720ff44ed155d4547750abf17512`.
- Verified previous BUILD_RECORD: `/opt/sub2api-deploy/builds/company-v0.2.15-rc.1-20261010T022747Z/BUILD_RECORD`. It binds tag, commit, tested commit, source archive, Dockerfile, ledger and image ID.
- Live migration count is 292. PostgreSQL ID `0d154990328e3f250c71eb4107047a2e73cc24818dc0fc2445a5a84680e8291a`, started `2026-06-16T14:37:05.04573087Z`; Redis ID `cc5b8eafb56f2e8526af90a350e10b94469a74e40e26cbe828391d1840eefd1f`, started `2026-05-15T09:27:38.487405299Z`.

## Complete company commit disposition

| Old commit | Final behavior or artifact | Disposition and target evidence | Contract |
| --- | --- | --- | --- |
| `a15b999dc` | 12000-rune moderation excerpt; non-hit retention maximum 7 days, default 3. Backend content_moderation.go, frontend RiskControlView.vue and locale labels. | Preserve with `3d4dd38ca`. Official v0.2.18 still uses 240-rune excerpt and 3-day maximum. | Existing backend truncation/normalization tests and frontend retention input/load/save tests. |
| `02cd4385d` | Secret redaction before truncation and independent cyber-policy error limit; boundary tests. | Preserve with `6afdde46a`. Official config-save change is outside these boundaries and remains present. | Existing backend redaction, rune limit, async persistence and frontend clamp tests. |
| `a9a513e23` | Previous release ledger. | Historical evidence retained by immutable old tag; replaced by this release ledger. | Provenance and diff checks. |
| `812394afc` | Previous release tests record. | Historical evidence retained by immutable old tag; regenerate against this candidate. | Six suites and strict 8-line tests record. |

The v0.2.18 upstream diff changes only configuration persistence inside content_moderation.go and does not change the company frontend paths. No unrelated company behavior is added or removed. Account model mappings and production settings are retained.

## Ordered migration review

Existing SQL files are unchanged. Five new files raise the target count to 297.

| Migration | Effect and lock | Compatibility and acceptance |
| --- | --- | --- |
| `242_group_i18n.sql` | Adds groups.i18n JSONB NOT NULL with constant empty-object default. Brief table DDL lock; no application-side backfill. | Additive. Verify column type/default/nullability. Old app ignores the column. |
| `243_gemini_cached_contents.sql` | Creates a new resource-binding table and 3 indexes. | New independent table. Verify table and all indexes. Old app ignores it. |
| `244_gemini_cached_contents_channel_usage.sql` | Adds channel_usage JSONB NOT NULL default empty object to the new table. | Additive. Verify type/default/nullability. |
| `245_gemini_cached_contents_mutation_lease.sql` | Adds nullable mutation_token and mutation_lease_until plus a partial pending-mutation index. | Additive. Verify columns and index predicate. Do not roll back while live mutation intents exist. |
| `246_enable_gemini_image_generation_groups.sql` | Enables image generation for Gemini/Antigravity groups only. | Production preflight matched 0 rows among 18 groups, so no current permission change. Recheck affected count before deployment and the final predicate afterward. |

Old-application rollback preserves the additive schema. Once new Gemini cached resources are created, an old app cannot manage their lifecycle. Check active cached resources and pending mutation intents before app-only rollback; if either exists, use forward repair or a separately approved restore plan. Never automatically restore PostgreSQL.

## Verification and deployment

Run backend service tests, migration/repository tests, PostgreSQL/Redis integration tests, full frontend tests, frontend typecheck and git diff --check at the fixed candidate commit. Use Go 1.27.3 from the target go.mod. Commit only the structured tests record afterward.

Create an immutable annotated company tag with upstream, commit and ledger SHA256. Build version 0.2.18-rc.1 using the standard company build script. Verify source/test provenance, embedded version, entrypoint and all 297 migration fingerprints.

Create and verify a fresh complete deployment backup. Replace only the application container. Verify health, database reads, image/version/commit, 297 migrations, all migration effects above, preserved model mappings and moderation settings. PostgreSQL and Redis IDs and start times must remain unchanged. No billable model request is part of acceptance.
