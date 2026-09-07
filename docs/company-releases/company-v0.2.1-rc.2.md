# company-v0.2.1-rc.2 release ledger

## Release identity

- Upstream: `v0.2.1@578785ee7fb35030b094b69624efe25670a36f5f`
- Previous company release: `company-v0.1.183-rc.1@f5429d2019b1b0c873e3fc6203e319aa5e6e2312`
- Upgrade branch: `codex/upgrade-v0.2.1-company`
- Requested by the company operator on 2026-09-07. Existing company behavior must remain; no removal was approved.

## Previous image provenance

Production runs `sub2api-company:0.1.183-rc.1` with image ID `sha256:15a5e6ca595869ab0dac2f7773d59d2d1285935aff6aa26b078e32fa5ea99fb0`.

- The immutable tag resolves to `f5429d2019b1b0c873e3fc6203e319aa5e6e2312` and records upstream `v0.1.183@e8cb019fabf8b55199436229044cbf9aa7a82564`.
- The target tag merge-base with the previous company tag is exactly that upstream commit.
- The verified build artifact is `/opt/sub2api-deploy/builds/company-v0.1.183-rc.1-20260827T013831Z`.

## Company patch inventory and disposition

| Previous commits | Final behavior | Disposition | Contract |
| --- | --- | --- | --- |
| `d088db966`, `14856bf40` | Persist up to 12000 redacted runes in the moderation audit excerpt. Moderation input remains 12000 runes and cyber-policy errors remain capped at 960 runes. | Preserve. Upstream `v0.2.1` still persists only 240 runes. | Backend tests cover rune truncation, redaction across the input boundary, async persistence, and the independent cyber-policy error limit. |
| `d088db966` | Allow non-hit retention up to 7 days while keeping the default at 3 days. | Preserve. Upstream `v0.2.1` still caps the value at 3. | Backend normalization and frontend input/load/save tests cover 7-day acceptance and clamping. |

No other previous company behavior is carried forward. The candidate must differ from `v0.2.1` only in the six implementation/test/locale files above, this ledger, and the generated test evidence file.

## Migration review

Production currently has 270 migrations. The target has 281. All 11 new migrations are forward-only and additive.

| Migration | Production effect | Old application compatibility |
| --- | --- | --- |
| `231_add_usage_log_native_compaction_v2.sql` | Adds a non-null boolean with default false to `usage_logs`. | Compatible while default remains false. |
| `231_add_usage_log_requested_reasoning_effort.sql` | Adds a nullable text field to `usage_logs`; metadata-only on supported PostgreSQL versions. | Compatible; old code ignores it. |
| `231_user_restrict_public_groups.sql` | Adds a non-null user flag with default false. | Rollback requires every row to remain false. |
| `232_add_usage_log_upstream_request_id.sql` | Adds a nullable upstream request ID field. | Compatible; old code ignores it. |
| `232_channel_cache_write_1h_pricing.sql` | Adds four nullable price fields. | Rollback requires every new price field to remain null, otherwise old billing ignores configured prices. |
| `232_group_force_openai_fast.sql` | Adds a non-null group flag with default false. | Rollback requires every row to remain false. |
| `232_group_reasoning_effort_over_limit.sql` | Adds a group policy with default `downgrade`. | Rollback requires every row to remain `downgrade`. |
| `233_add_usage_log_upstream_request_id_index_notx.sql` | Builds a partial index concurrently on the hot usage table. | Compatible; the old app ignores the index. |
| `233_group_free_openai_fast.sql` | Adds a non-null group flag with default false. | Rollback requires every row to remain false. |
| `234_channel_max_reasoning_effort_multiplier.sql` | Adds a nullable positive multiplier and check constraint. | Rollback requires every value to remain null, otherwise old billing ignores it. |
| `234_group_codex_models_manifest_config.sql` | Adds a JSON config with default empty object. | Rollback requires every value to remain an empty object. |

Before app-only rollback, every guard below must be zero:

```sql
SELECT
  (SELECT COUNT(*) FROM users WHERE restrict_public_groups) AS restricted_public_group_users,
  (SELECT COUNT(*) FROM channel_model_pricing WHERE cache_write_1h_price IS NOT NULL) +
  (SELECT COUNT(*) FROM channel_pricing_intervals WHERE cache_write_1h_price IS NOT NULL) +
  (SELECT COUNT(*) FROM channel_account_stats_model_pricing WHERE cache_write_1h_price IS NOT NULL) +
  (SELECT COUNT(*) FROM channel_account_stats_pricing_intervals WHERE cache_write_1h_price IS NOT NULL) AS cache_write_1h_price_rows,
  (SELECT COUNT(*) FROM groups WHERE force_openai_fast) AS forced_fast_groups,
  (SELECT COUNT(*) FROM groups WHERE max_reasoning_effort_over_limit <> 'downgrade') AS non_default_reasoning_over_limit_groups,
  (SELECT COUNT(*) FROM groups WHERE free_openai_fast) AS free_fast_groups,
  (SELECT COUNT(*) FROM channel_model_pricing WHERE max_reasoning_effort_multiplier IS NOT NULL) AS max_effort_multiplier_rows,
  (SELECT COUNT(*) FROM groups WHERE codex_models_manifest_config <> '{}'::jsonb) AS codex_manifest_config_groups;
```

Any nonzero guard blocks app-only rollback. Do not clear or rewrite production data to make a rollback pass.

## Verification

- Run backend service tests, migration and repository tests, PostgreSQL integration migration tests, frontend tests, frontend typecheck, and `git diff --check`.
- Generate `company-v0.2.1-rc.2.tests.tsv` against the fixed candidate commit.
- Build from the immutable annotated tag with explicit `VERSION=0.2.1-rc.1` and the full commit.
- Back up Compose, `.env`, application data, Redis RDB, PostgreSQL dump, migrations, and the old image.
- Rebuild only `sub2api`; preserve the Postgres and Redis container IDs and start times.
- Verify all 281 migrations, the new schema objects, `/health`, database reads, embedded version, and rollback guards.
