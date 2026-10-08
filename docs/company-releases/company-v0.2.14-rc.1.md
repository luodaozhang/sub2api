# company-v0.2.14-rc.1 release ledger

## Release identity

- Upstream: `v0.2.14@0363b8cdba8cec3e2ba4b2dbd49c4481143fa55d`
- Previous company release: `company-v0.2.7-rc.1@68c82ad701b2e6ec4612117640929b14d31602a6`
- Upgrade branch: `codex/upgrade-v0.2.14-company`
- Requested by the company operator on 2026-10-08.

## Previous image provenance

Production runs `sub2api-company:0.2.7-rc.1` with image ID `sha256:1f0be2aa91935d315675ffb07f46af3c7614d23ff9e06b61c594645360f8a4fc`.

- Production has 286 applied migrations before this release.
- PostgreSQL container ID is `0d154990328e3f250c71eb4107047a2e73cc24818dc0fc2445a5a84680e8291a` and Redis container ID is `cc5b8eafb56f2e8526af90a350e10b94469a74e40e26cbe828391d1840eefd1f`.
- The application, PostgreSQL, and Redis were healthy before build and deployment work began.

## Company patch inventory and disposition

| Previous behavior | Disposition | Contract |
| --- | --- | --- |
| Persist up to 12000 redacted runes in the moderation audit excerpt. | Preserve. Upstream `v0.2.14` still persists only 240 runes. | Backend tests cover rune truncation, redaction, async persistence, and the separate cyber-policy error limit. |
| Allow non-hit retention up to 7 days while keeping the default at 3 days. | Preserve. Upstream `v0.2.14` still caps the value at 3. | Backend normalization and frontend input, load, save, and clamping tests cover the 7-day limit. |
| Enforce the configured group model allowlist on requests. | Keep upstream behavior. | Requests outside an enabled group allowlist remain rejected before model mapping. |

No additional company behavior is introduced by this release.

## Migration review

The target has 291 migrations. It adds five migration files to the 286 currently applied in production.

| Migration | Production effect | Compatibility decision |
| --- | --- | --- |
| `238b_content_moderation_engine_meta.sql` | Adds nullable JSONB `content_moderation_logs.engine_meta`. Existing rows remain unchanged. | Additive and old-application compatible. |
| `239_channel_reasoning_effort_multipliers.sql` | Adds JSONB reasoning-effort multiplier maps and migrates explicit legacy maximum multipliers. Removes the consumed legacy JSON key from group pricing entries. | Forward data migration; backup required before deployment. |
| `240_affiliate_ledger_operation_id.sql` | Adds nullable idempotency operation ID and a partial unique index for affiliate ledger entries. | Additive; duplicate non-null operation IDs would fail index creation, so preflight must verify none exist. |
| `241_add_payment_order_bonus_amount.sql` | Adds non-null `payment_orders.bonus_amount` with default zero. | Additive. |
| `241_add_typesafe_platform.sql` | Replaces two platform CHECK constraints with supersets that add `typesafe`. | Forward schema change; existing values remain valid. |

Returning to the old application after migrations would require a separately approved database restore from the pre-upgrade backup.

## Verification

- Run backend service tests, migration and repository tests, PostgreSQL integration migration tests, frontend tests, frontend typecheck, and `git diff --check`.
- Generate `company-v0.2.14-rc.1.tests.tsv` against the fixed candidate commit.
- Build from the immutable annotated tag with explicit `VERSION=0.2.14-rc.1` and the full commit.
- Back up Compose, `.env`, application data, Redis RDB, PostgreSQL dump, migrations, and the old image.
- Rebuild only `sub2api`; preserve the PostgreSQL and Redis container IDs and start times.
- Verify all 291 migrations, `/health`, database reads, embedded version, new columns and indexes, expanded platform constraints, and company moderation settings.
