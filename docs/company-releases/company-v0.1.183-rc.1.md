# company-v0.1.183-rc.1 release ledger

## Release identity

- Upstream: `v0.1.183@e8cb019fabf8b55199436229044cbf9aa7a82564`
- Previous company release: `company-v0.1.178-rc.1@d62e45fa3aca7fbfe4a765f6f99d0a46aaf2eded`
- Upgrade branch: `codex/upgrade-v0.1.183-company`
- Company behavior implementation: `d088db96657f9ee2bd57807cb71c09ba51315735`, with audit-boundary correction `14856bf408d4f788a3366b7a0bed215dca5b68d0`
- Requested by the company operator on 2026-08-26 with the instruction to upgrade the company Sub2API instance to the latest release. The operating rule requires preserving current company behavior unless its removal is explicitly approved; no removal was approved.

## Previous image provenance

The running production image before this upgrade is `sub2api-company:0.1.178-rc.1` with image ID `sha256:e131f0f8ef7a3a1f2c8103a90151f49a8ce1586c23f0de315d2db9e4ad10e8cf`.

- The original build log records a clean checkout of `company-v0.1.178-rc.1`, build arguments `VERSION=0.1.178-rc.1` and `COMMIT=d62e45fa3aca7fbfe4a765f6f99d0a46aaf2eded`, and the resulting image ID above.
- The running binary independently reports `0.1.178-rc.1`, commit `d62e45fa3aca7fbfe4a765f6f99d0a46aaf2eded`, built at `2026-08-19T02:20:45Z`.
- The immutable fork tag and branch both resolve to that commit. A fresh full clone is not shallow, and `v0.1.178` commit `e0c48a19ed794a565e3858662520afe0a1f9f0ba` is the unique merge-base with the company tag.
- Recomputed from the immutable tag on 2026-08-26: `git archive` SHA256 `9408b7934329b2c66f598f5bb750b2a0b8ffb7bdbaef9253492d6cf6d00605aa`; root `Dockerfile` SHA256 `41c750711dc576cd1c7cc085218b5705adab94965ec69ea167c32fd598b461bb`.

## Company patch inventory and disposition

| Old commit | Final behavior | Files and dependencies | Disposition | Target evidence and contract |
| --- | --- | --- | --- | --- |
| `4ed00b459` | Increased the persisted moderation excerpt from 240 to 2000 runes. | `backend/internal/service/content_moderation.go` and its tests; depends on existing secret redaction before truncation. | Superseded. Do not replay the intermediate value because `d62e45fa3` replaced it with the final 12000-rune behavior. | The complete old-tag diff proves the later commit overrides this value. |
| `d62e45fa3` | Persist up to 12000 redacted runes and allow non-hit retention to be configured up to 7 days while keeping the default at 3 days. | Six files: the moderation service and tests, Risk Control view and tests, plus Chinese and English labels. The existing database field is `TEXT`; no migration is required. The old implementation truncated moderation input before building the log excerpt and reused the excerpt limit for `cyber_policy` errors. | Preserve the company contract, but correct the old implementation: derive the log excerpt from the complete normalized text before truncating input for the moderation API, and retain the independent `cyber_policy` error limit of 960 runes. | Upstream `v0.1.183` still has excerpt `240` and maximum non-hit retention `3`, so upstream does not replace the company contract. Tests cover a secret crossing the 12000-rune boundary through `Check` and async persistence, the independent 960-rune error limit, seven-day normalization, UI max/load/save behavior, and the unchanged three-day default. |

No other file differs between `v0.1.178` and `company-v0.1.178-rc.1`. The new company branch differs from `v0.1.183` only in the same six files plus this ledger.

## Verification

- Backend focused tests, Go 1.27.0 container: `go test ./internal/service -run "Test(BuildContentModerationLog_(RedactsInputExcerpt|UsesExcerptLimit)|ContentModerationConfigNormalize_(NonHitRetentionMaxSevenDays|NonHitRetentionKeepsDefaultThreeDays)|ContentModerationRecordCyberPolicyEvent_KeepsIndependentErrorLimit|ContentModerationCheck_RedactsSecretCrossingInputLimitBeforeAsyncLog|RedactContentModerationSecrets_LongHexAndTokens)$"` — passed.
- Backend service package, Go 1.27.0 container: `go test ./internal/service` — passed.
- Migration tests, Go 1.27.0 container: `go test ./migrations ./internal/repository` — passed.
- PostgreSQL integration migrations, Go 1.27.0 container: `go test -tags integration ./internal/repository -run 'TestMigrationsRunner_(ConcurrentInstancesSerializeOnSessionLock|IsIdempotent_AndSchemaIsUpToDate)'` — passed against a temporary PostgreSQL 18.1 container.
- Frontend full suite before the audit-boundary follow-up: `pnpm test:run` — 244 files and 1746 tests passed, including the then-current 5 `RiskControlView.spec.ts` tests.
- Updated Risk Control view tests: `pnpm exec vitest run src/views/admin/__tests__/RiskControlView.spec.ts` — 6 of 6 passed.
- Frontend type check: `pnpm typecheck` — passed with no errors.
- `git diff --check` — passed.

## Migration and rollback review

The production database has 265 `schema_migrations` rows. `v0.1.183` has 270 non-empty SQL migrations. A filename and `SHA256(strings.TrimSpace(content))` comparison found no production file missing from the target and no checksum mismatch. Five files are new:

| Migration | Production impact | Old application compatibility |
| --- | --- | --- |
| `226_add_usage_log_effective_model_indexes_notx.sql` | Builds two indexes concurrently on `usage_logs` (about 880,496 live rows and 1.11 GB total before release). It avoids the normal writer-blocking index build but can consume I/O and take time. | Compatible; indexes are invisible to old application behavior. |
| `227_composite_routes_add_cn_providers.sql` | Replaces one check constraint to allow `kimi`, `zhipu`, and `deepseek`. The table currently has zero rows, so validation and lock duration should be short. | Schema is readable by the old app. Data is conditionally compatible: the old admin API rejects the three new values. Before any app-only rollback, `target_platform IN ('kimi','zhipu','deepseek')` must still be zero. A nonzero result blocks app-only rollback and requires a forward fix or the approved restore path. |
| `228_channel_pricing_multipliers.sql` | Adds nullable numeric columns and positive-value checks. The affected tables currently have 4 and 0 live rows. | Schema-compatible, but data is conditionally compatible. The old app silently ignores all six new multiplier columns and `time_pricing.weekdays_only`; app-only rollback is blocked if any multiplier is non-null or any row has `weekdays_only=true`, because old billing would calculate a different price. |
| `229_plugins.sql` | Creates new plugin tables and indexes; plugins default disabled. | Schema-compatible. App-only rollback is blocked while any plugin or plugin binding is enabled, because the old app cannot preserve enabled plugin routing behavior. Disabled installation metadata can remain. |
| `230_plugin_artifacts.sql` | Adds a nullable `BYTEA` column to the new plugin table. | Compatible; no backfill is required. |

Before any app-only rollback, all of the following query results must be zero:

```sql
SELECT
  (SELECT COUNT(*)
   FROM composite_model_routes
   WHERE target_platform IN ('kimi', 'zhipu', 'deepseek')) AS cn_composite_routes,
  (SELECT COUNT(*)
   FROM channel_model_pricing
   WHERE fast_multiplier IS NOT NULL
      OR flex_multiplier IS NOT NULL) AS channel_multiplier_rows,
  (SELECT COUNT(*)
   FROM channel_pricing_intervals
   WHERE input_multiplier IS NOT NULL
      OR output_multiplier IS NOT NULL
      OR cache_write_multiplier IS NOT NULL
      OR cache_read_multiplier IS NOT NULL) AS interval_multiplier_rows,
  (SELECT COUNT(*)
   FROM channel_model_pricing
   WHERE time_pricing -> 'weekdays_only' = 'true'::jsonb) AS weekdays_only_rows,
  (SELECT COUNT(*)
   FROM sub2api_plugin_installations
   WHERE state IN ('starting', 'enabled')) AS active_plugins,
  (SELECT COUNT(*)
   FROM sub2api_plugin_bindings
   WHERE enabled) AS active_plugin_bindings;
```

Immediate rollback is app-only only while all six guards remain zero. PostgreSQL and Redis must not be restarted. Any nonzero guard blocks app-only rollback and requires the approved restore path or a forward fix; those values must not be silently deleted or rewritten. After successful migration, long-term rollback must re-evaluate these guards and current data rather than relying on the pre-release snapshot.
