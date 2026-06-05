---
name: synth-data
description: "Generate statistically realistic synthetic star-schema datasets on Snowflake for demos, testing, and PoCs. Covers 12 verticals, 46 use cases, 26 regions. Use when: user wants synthetic data, demo data, fake data, sample dataset, star schema generation, test data, or realistic data generation for any vertical. Triggers: synthetic data, synth data, generate demo data, fake data, sample dataset, star schema generator, test data generation, realistic data, demo dataset, generate data for."
log_marker: SKILL_USED_SYNTH_DATA
skill_version: "2026-06-02"
validated_syntax_date: "2026-06-02"
---

# Synthetic Data Generation Skill

## When to Use

Invoke this skill when the user needs synthetic data for:
- Demos or PoCs (star-schema with realistic distributions)
- Testing pipelines, dashboards, or ML models
- Any of the 12 verticals (healthcare, retail, financial services, insurance, SaaS, manufacturing, telecom, energy, logistics, media, life sciences, public sector)

Do NOT use for: single-table mock data (use simple GENERATOR instead), data masking/anonymization, or migrating real data.

---

## Goal

Generate a complete star-schema dataset with statistically realistic distributions for a specified vertical use case. Two output modes: `live` executes SQL directly against Snowflake; `sql_files` writes numbered .sql scripts to a local directory for later execution. Output: populated tables (live) or SQL project (sql_files), plus data dictionary and rollback script.

---

## Inputs

| Parameter | Type | Example | Required |
|-----------|------|---------|----------|
| output_mode | enum | `live` (execute on Snowflake), `sql_files` (write .sql to disk) | Yes |
| output_dir | string | `./synth_output/healthcare_claims` | No (required if output_mode=sql_files) |
| vertical | string | `healthcare`, `retail-cpg`, `financial-services`, `insurance`, `technology-saas`, `manufacturing`, `telecom`, `energy-utilities`, `logistics-transportation`, `media-entertainment`, `life-sciences-pharma`, `public-sector` | Yes |
| use_case | string | `claims-analytics`, `demand-forecasting`, `customer-360` | Yes |
| region | string | `us`, `ca`, `eu-west`, `latam-chile`, `latam-brazil`, `apac-japan`, `mena-uae` (+ 19 more LATAM) | Yes |
| target_database | string | `DEMO_DB` | Yes |
| target_schema | string | `SYNTH_HEALTHCARE` | Yes |
| date_range_start | date | `2024-01-01` | No (default: CURRENT_DATE - 2 years) |
| date_range_end | date | `2025-12-31` | No (default: CURRENT_DATE - 1 day) |
| row_scale | enum | `small` (10K fact rows), `medium` (100K), `large` (1M) | Yes |
| currency | string | `USD` | No (default: from region) |
| include_anomalies | boolean | `true` | No (default: true) |

---

## Output Mode Behavior

### `live` — Execute directly on Snowflake
- Requires active Snowflake connection with CREATE TABLE privileges
- Executes all SQL via `sql_execute`, verifies results inline
- Gates run real queries to check preconditions
- Phase 7 validates populated tables

### `sql_files` — Write SQL scripts to disk
- No Snowflake connection required
- Creates numbered .sql files in `{output_dir}/`
- Gates become comments in the setup script (informational, not blocking)
- Verification queries written to `06_verify.sql` for user to run later
- Date range defaults resolved to literal dates

**File structure (`sql_files` mode):**
```
{output_dir}/
├── 00_setup.sql            -- CREATE DATABASE/SCHEMA IF NOT EXISTS, USE statements
├── 01_dim_date.sql         -- Date dimension + holiday calendar
├── 02_dim_{entity1}.sql    -- One file per entity dimension
├── 03_dim_{entity2}.sql
├── 04_fact_{name}.sql      -- Fact table generation
├── 05_anomalies.sql        -- Anomaly/data quality injection (if enabled)
├── 06_verify.sql           -- All validation queries
├── 99_rollback.sql         -- DROP SCHEMA CASCADE
└── README.md               -- Data dictionary
```

**Canonical numbering (strict):**
| # | File | Content |
|---|------|---------|
| 00 | `00_setup.sql` | CREATE DATABASE/SCHEMA, USE statements |
| 01 | `01_dim_date.sql` | Date dimension + holiday_calendar helper |
| 02 | `02_dim_{first}.sql` | First entity dimension |
| 03 | `03_dim_{second}.sql` | Second entity dimension (if needed) |
| 04 | `04_fact_{name}.sql` | Fact table CTAS |
| 05 | `05_anomalies.sql` | Anomaly injection (UPDATE statements only) |
| 06 | `06_verify.sql` | All validation queries |
| 99 | `99_rollback.sql` | DROP SCHEMA CASCADE |
| — | `README.md` | Data dictionary |

For >2 entity dims: use 02, 03, 03b — never push fact past 04.

---

## Context Setup

Before starting, load into session context:
1. This file (SKILL.md) — always full
2. `references/SYNTH_USE_CASE_SCHEMA_MAP.md` — load ONLY the matching vertical > use_case entry (not the entire file)
3. `references/SYNTH_SQL_PATTERNS.md` — load full (13 patterns, fits in context)
4. `references/SYNTH_REGIONS.md` — load ONLY the matching region section

This selective loading keeps total context under 500 lines.

---

## Stopping Points

| Phase | Checkpoint | Action |
|-------|-----------|--------|
| Phase 1 | After input validation | Confirm resolved parameters with user |
| Phase 2, Step 5 | Plan summary table | **Wait for user approval** before executing |
| Phase 3 Gate | Schema non-empty (live mode) | HALT — ask user to confirm DROP or pick new schema |
| Phase 7/8 | Final report | Present results, offer rollback script |

---

## Phase 1 — Explore (max 7 steps)

### Entry criteria
Required inputs provided by user. Optional inputs use defaults.

### Steps
1. **Resolve defaults**: If `date_range_start`/`date_range_end` not provided, compute: `SELECT CURRENT_DATE - INTERVAL '2 years' AS start_date, CURRENT_DATE - INTERVAL '1 day' AS end_date;` (live mode) or use today's literal date minus 2 years (sql_files mode). Confirm resolved range with user.
2. **Staleness check** *(live mode only)*: Run `SELECT ZIPF(2.0, 100, RANDOM()) FROM TABLE(GENERATOR(ROWCOUNT => 1));` — confirms GENERATOR and ZIPF available. *(sql_files mode: skip.)*
3. **Validate target** *(live mode only)*: Run `SHOW SCHEMAS IN DATABASE {target_database}` — confirm database exists. *(sql_files mode: skip.)*
4. **Check privileges** *(live mode only)*: Run `SELECT CURRENT_ROLE();` and confirm CREATE TABLE privilege. *(sql_files mode: skip.)*
5. **Validate inputs**: Confirm `vertical` + `use_case` combination exists in `references/SYNTH_USE_CASE_SCHEMA_MAP.md`. If not found, suggest closest match.
6. **Load parameters**: From schema map, extract: fact table(s), dimensions, grain, volume, distributions, seasonality. From regions reference, extract: holiday calendar, monthly activity index, currency.
7. **Prepare output directory** *(sql_files mode only)*: Create `{output_dir}/` if it doesn't exist. *(live mode: skip.)*

### Exit criteria
All inputs validated. Schema definition, distribution params, and regional params loaded.

---

## Phase 2 — Plan (max 5 steps)

### Steps
1. **Derive schema**: From the use case's schema map entry, declare all tables to create (1 fact + N dims + dim_date).
2. **Calculate row counts**: Apply `row_scale` multiplier to the schema map's volume baseline:
   - `small` = baseline / 10
   - `medium` = baseline
   - `large` = baseline * 10
3. **Plan dimensions**: For each dimension, declare row count (typically 1/50th to 1/200th of fact volume).
4. **Plan seasonality**: Map regional monthly activity index + use-case-specific seasonal patterns to the date range. Identify anomaly injection dates (if include_anomalies=true).
5. **Present plan to user**: Output a summary table:
   | Table | Rows | Key Distributions | Notes |

   Include cost guidance:
   | Scale | Approx Fact Rows | Warehouse Rec. | Est. Credits |
   |-------|-----------------|----------------|--------------|
   | small | 10K | X-SMALL | <0.01 |
   | medium | 100K | SMALL | ~0.05 |
   | large | 1M | MEDIUM | ~0.3 |

   **⚠️ STOP**: Show plan and wait for user confirmation before executing.

### Exit criteria
User confirms the generation plan.

---

## Phase 3 — Execute: Schema + Date Dimension (max 4 steps)

### Gate *(live mode only; sql_files mode: emit as comment in 00_setup.sql)*
```
GATE: SELECT COUNT(*) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = '{target_schema}' AND TABLE_CATALOG = '{target_database}';
EXPECTED: 0 (schema empty or doesn't exist)
ON FAIL: HALT — "Target schema {target_schema} contains {n} existing tables. Confirm DROP SCHEMA CASCADE or choose a different schema."
```

### Steps
1. Create database and schema:
   - `CREATE DATABASE IF NOT EXISTS {target_database};`
   - `USE DATABASE {target_database};`
   - `CREATE SCHEMA IF NOT EXISTS {target_database}.{target_schema};`
   - *sql_files mode*: Write to `00_setup.sql` with header comment including gate check, USE DATABASE/SCHEMA statements. Every table reference in generated SQL must be fully qualified (`{target_database}.{target_schema}.{table_name}`). Do NOT rely on `USE SCHEMA` — it creates implicit context that breaks when scripts are run out of order.
2. Generate date dimension using Pattern #1 from `references/SYNTH_SQL_PATTERNS.md`, parameterized with `date_range_start` / `date_range_end`. Add holiday flags from the region's calendar.
   - *sql_files mode*: Write to `01_dim_date.sql`.
3. Create `holiday_calendar` helper table with event dates, names, `volume_multiplier`, and `value_multiplier` from `references/SYNTH_REGIONS.md`. Filter to only include events whose date range overlaps with `date_range_start`..`date_range_end`. For single-day events, set `start_date = end_date` (both columns must be populated).
   - *sql_files mode*: Append to `01_dim_date.sql` (always co-located with dim_date, never in fact file).
4. **Verify** *(live mode)*: `SELECT COUNT(*), MIN(full_date), MAX(full_date) FROM {target_database}.{target_schema}.dim_date;` — confirm row count = expected days, min/max match date range.
   - *sql_files mode*: Write verification query to `06_verify.sql`.

### Exit criteria
dim_date and holiday_calendar verified.

---

## Phase 4 — Execute: Entity Dimensions (max 6 steps)

### Gate *(live mode only; sql_files mode: skip — execution order guarantees dependency)*
```
GATE: SELECT COUNT(*) FROM {target_database}.{target_schema}.dim_date;
EXPECTED: > 0 (date dimension populated)
ON FAIL: HALT — "Date dimension is empty. Phase 3 did not complete successfully."
```

### Steps
For each dimension table declared in Phase 2 plan:

1. Generate using CTAS from GENERATOR with:
   - Surrogate keys via `ROW_NUMBER() OVER (ORDER BY SEQ4())`
   - Size/revenue tiers via `ZIPF()` (Pattern #3)
   - Entity names via ARRAY + UNIFORM indexing (Pattern #4) — never MOD
   - Geographic distribution via weighted UNIFORM (Pattern #7)
   - *sql_files mode*: Write each dimension to `02_dim_{name}.sql`, `03_dim_{name}.sql`, etc.
2. Apply correlated NULL patterns (Pattern #5) for optional fields.
3. **Verify each** *(live mode)*: `SELECT COUNT(*), COUNT(DISTINCT {pk_column}) FROM {table}` — confirm row count and uniqueness.
   - *sql_files mode*: Append verification queries to `06_verify.sql`.

### Exit criteria
All dimension tables created and verified. Foreign key values ready for fact table.

---

## Phase 5 — Execute: Fact Table (max 8 steps)

### Gate *(live mode only; sql_files mode: skip)*
```
GATE: SELECT COUNT(*) FROM {target_database}.{target_schema}.{first_dim_table};
EXPECTED: > 0 (dimensions populated)
ON FAIL: HALT — "Dimension tables not populated. Phase 4 did not complete successfully."
```

### Steps
1. Generate base fact rows using GENERATOR with row count from plan.
2. Assign date FK using weighted date assignment (Pattern #12):
   - Apply monthly_activity_index to weight date distribution
   - Apply volume_multiplier from holiday_calendar to spike row counts on event dates
3. Assign entity FKs:
   - Entity FKs: ZIPF(1.0–1.2) for FK assignment — ensures 99%+ dimension key coverage. Use schema map alpha values only for metric columns, not FK columns.
   - Correlated FKs: When dimensions have parent-child relationships (e.g., account → customer), generate ONLY the child FK via ZIPF, then JOIN to the child dimension to derive the parent FK. Never generate both independently.
   - **Adaptive FK coverage**: Compute `draw_ratio = fact_rows / dim_rows` per FK column:
     - `draw_ratio >= 100`: Use ZIPF(1.1-1.2) per schema map
     - `draw_ratio 40-99`: Use ZIPF(1.0) (minimum skew)
     - `draw_ratio < 40`: Use UNIFORM (common at small scale for large dimensions)
4. Assign categorical columns (e.g., `channel`, `txn_type`, `status`, `payment_method`):
   - **Must use Pattern #11** (single-seed categorical): ONE `UNIFORM(0::FLOAT, 1::FLOAT, RANDOM())` seed per row in the base CTE, then cumulative CASE thresholds in the final SELECT.
   - Never use multiple independent RANDOM() calls per CASE branch.
5. Generate metric columns using time-series composition (Pattern #2):
   - Trend component: growth rate from plan
   - Seasonal component: combine regional monthly index + use-case seasonality
   - Noise component: NORMAL(0, noise_pct)
6. Apply holiday value_multiplier via Pattern #10 (compound pre-aggregated join).
7. Generate metric amounts via ZIPF:
   - **FK columns**: `ZIPF(1.0-1.2, dim_row_count, RANDOM())` — N = dimension size
   - **Metric columns**: `ZIPF(alpha, fact_row_count, RANDOM())` — N = fact table row count (NOT dim size). This ensures sufficient rank spread at all scales.
   - Formula: `base_value * POWER(zipf_rank, 1.0/alpha)` — low rank (majority) = low value
8. **Optional — Intraday timestamps** (Pattern #13): For verticals where time-of-day matters (retail, telecom, media), generate `transaction_timestamp` with bimodal hour distribution.
   - *sql_files mode*: Write entire fact generation to `04_fact_{name}.sql`.
9. **Verify** *(live mode)*:
   - Row count matches plan (±1%)
   - Referential integrity: Pattern #8 anti-join for each FK → dim (expect 0 orphans)
   - Metric sanity: `SELECT MIN(metric), AVG(metric), MAX(metric), STDDEV(metric) FROM fact`
   - *sql_files mode*: Append all verification queries to `06_verify.sql`.

### Exit criteria
Fact table populated. Zero orphan FKs. Metrics within expected ranges.

---

## Phase 6 — Execute: Anomalies & Data Quality (max 4 steps)

*Skip this phase if `include_anomalies = false`.*

### Steps
1. **Inject positive outliers** (2-3 events): Use Pattern #6 to multiply values for specific date ranges. Events sourced from vertical use case context (e.g., flash sale, viral product, emergency surge).
2. **Inject negative outliers** (2-3 events): Dip/outage events — multiply by 0.0-0.3 for specific dates. Apply to ALL rows on affected dates (not a probabilistic subset). Target months with normally-high activity so dips are detectable against baseline. Never use DELETE — simulate outages by setting metric columns to 0 via UPDATE.
3. **Apply data quality artifacts**: Pattern #5 for additional correlated NULLs on fact table optional fields (1-3% rate).
   - *sql_files mode*: Write all anomaly/quality SQL to `05_anomalies.sql`.
   - **Implementation method**: For `sql_files` mode, use UPDATE statements in `05_anomalies.sql` (idempotent on re-run). For `live` mode, also prefer UPDATE for idempotency. Do NOT include anomaly logic in the original Phase 5 CTAS — separation ensures anomalies are independently verifiable and the phase is retryable.
4. **Verify anomalies detectable** *(live mode)*:
   ```sql
   SELECT DATE_TRUNC('week', d.full_date) AS week, SUM(f.{metric})
   FROM fact f JOIN dim_date d ON f.date_id = d.date_key
   GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
   ```
   Confirm top/bottom weeks correspond to injected events.
   - *sql_files mode*: Append to `06_verify.sql`.

### Exit criteria
Anomalies verifiable via simple aggregation. Data quality patterns applied.

---

## Phase 7 — Verify & Commit (max 6 steps)

### Steps

**Live mode:**
1. **Distribution validation**: Run Pattern #9 (ROW_NUMBER Gini) on primary revenue/amount metric. Expected: ZIPF metrics → 0.20-0.85 (small scale) / 0.45-0.85 (medium) / 0.55-0.85 (large); NORMAL metrics → 0.15-0.45. Skip for UNIFORM metrics.
2. **Seasonality spot-check**: Monthly aggregation — confirm expected peaks/troughs from regional index.
3. **Referential integrity sweep**: Run Pattern #8 for ALL foreign key relationships. All must return 0.
4. **Row count summary**: `SELECT table_name, row_count FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = '{target_schema}'`
5. **Generate data dictionary**: Markdown table for each table with column name, type, distribution, business meaning.
6. **Generate rollback script**: Output the cleanup SQL (see Rollback section below).

**sql_files mode:**
1. Write `06_verify.sql` containing all validation queries (already accumulated). Add Gini and seasonality checks.
2. Write `99_rollback.sql` with DROP SCHEMA CASCADE.
3. Write `README.md` as data dictionary (table descriptions, column definitions, distribution notes, execution instructions).
4. Report final file manifest to user.

### Exit criteria
`live`: All validations pass. Data dictionary and rollback script delivered to user.
`sql_files`: All .sql files written. README.md generated. File manifest reported.

---

## Phase 8 — Optional: Cortex Analyst Semantic Model (max 3 steps)

*Skip unless user requests Cortex Analyst integration or explicitly wants a semantic model.*

### Steps
1. **Generate semantic_model.yaml**: Map generated tables to a Cortex Analyst semantic model:
   - `tables` → fact + all dimensions with physical names
   - `dimensions` → all VARCHAR/DATE/BOOLEAN columns + FK relationships
   - `measures` → all NUMBER metric columns with appropriate aggregations (SUM for amounts, AVG for rates/scores)
   - `time_dimensions` → date_key relationship to dim_date
   - `relationships` → JOIN paths between fact and dims
2. **Validate** *(live mode)*: Run `cortex reflect semantic_model.yaml --target-schema {target_database}.{target_schema}` to check YAML validity.
3. **Deliver**: Output file path or content to user.

### Exit criteria
Valid semantic_model.yaml generated and (if live) validated via cortex reflect.

---

## Guardrails

| Don't | Do Instead |
|-------|-----------|
| Use UNIFORM for revenue/spend metrics | Use ZIPF(1.5-2.2) for power-law behavior |
| Generate raw INSERT statements | Always use CTAS from GENERATOR — scales to millions |
| Create more than 3 subagents | Keep execution in main thread; use subagent only for schema map lookup if context is tight |
| Skip referential integrity checks | Run Pattern #8 anti-join after every fact table creation |
| Hardcode dates in seasonality | Always derive from region's holiday_calendar table |
| Use RANDOM() alone for metrics | Compose: ZIPF/NORMAL base * time-series formula * holiday multiplier |
| Assume column names from table name | Always reference schema map for exact column definitions |
| Generate entity names inline | Use ARRAY pattern (#4) — reusable, consistent, no hallucinated geography |
| Use `USE DATABASE` before creating DB | Use `CREATE DATABASE IF NOT EXISTS {db}; USE DATABASE {db};` — never assume DB exists |
| Use bare identifiers in VALUES clauses | Use SQL literals only: strings, numbers, booleans, NULL. For unbounded values use sentinel 99999 or -1 |
| Use `ABS(NORMAL(...))` for non-negative | Use `GREATEST(0, NORMAL(...))` — clips without folding the distribution |
| Use different seeds per CASE branch | Use a single seed per row with cumulative thresholds (Pattern #11) |
| Use ZIPF alpha >1.3 for FK assignment | Use ZIPF(1.0-1.2) for FK columns — higher alpha leaves keys unreferenced |
| JOIN holiday_calendar directly to fact | Pre-aggregate multipliers per date_key before joining (Pattern #10) — overlapping events cause fanout |
| Generate correlated FKs independently | Derive parent FK from child dimension via JOIN |
| Use ZIPF rank for categorical splits | Use UNIFORM with cumulative thresholds (Pattern #11) |
| Use MOD for ARRAY indexing or categorical splits | Use `UNIFORM(0, N-1, RANDOM())` for arrays; Pattern #11 for categoricals — MOD creates deterministic cycles |
| Use DELETE in anomaly injection | Use UPDATE to set metrics to 0/NULL — DELETE breaks idempotency and row counts |
| Assume ZIPF(1.1) covers dimension | Require draw:key ratio >= 40:1 for 99% coverage. Below 10:1 use UNIFORM or hybrid |
| Use `max / POWER(rank, 1/alpha)` | Use `base * POWER(rank, 1/alpha)` — low rank = low value |
| Use ZIPF for FK when draw:key ratio < 40 | Check ratio at runtime; use UNIFORM when `fact_rows / dim_rows < 40` (common at small scale) |
| Use `SET var = ARRAY_CONSTRUCT(...)` | Use inline CTE with CROSS JOIN — SET only supports scalar values in Snowflake |
| Use `UNIFORM(0, ARRAY_SIZE(arr)-1, ...)` | Hardcode array bounds as integer literals — UNIFORM requires constant bounds |

---

## Known Warnings

These Snowflake warnings are expected and non-blocking during execution:

| Warning | Context | Why It's OK |
|---------|---------|-------------|
| "Object already exists" | `CREATE DATABASE/SCHEMA IF NOT EXISTS` | Idempotent DDL — expected on re-runs |
| "Statement succeeded" with 0 rows | `CREATE OR REPLACE TABLE ... AS SELECT` on empty GENERATOR | Won't happen — GENERATOR always produces rows. If it does → treat as failure |
| WAREHOUSE_SIZE advisory | Running `large` scale on X-SMALL | Not a SQL warning — user was warned in Phase 2 cost table |

Any unlisted Snowflake warning or error = treat as failure. Retry once; if persists, halt and report.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `GENERATOR(ROWCOUNT => N)` fails | N exceeds session GENERATOR limit (10B) | Split into batches with UNION ALL, or reduce row_scale |
| `Insufficient privileges` on CREATE TABLE | Current role lacks CREATE TABLE on target schema | `GRANT CREATE TABLE ON SCHEMA ... TO ROLE {current_role}` — ask user |
| Query timeout during Phase 5 fact CTAS | Warehouse too small for row_scale=large | Recommend MEDIUM+ warehouse; offer to temporarily resize |
| ZIPF returns unexpected distribution | Wrong alpha direction (high alpha = extreme skew) | Verify alpha from schema map. FK columns: 1.0-1.2. Metrics: 1.5-2.2 |
| Holiday calendar JOIN produces duplicate fact rows | Overlapping events without pre-aggregation | Must use Pattern #10 (EXP/SUM/LN compound). Never direct JOIN |
| Phase 3 gate fires unexpectedly | Schema has leftover tables from prior run | Offer user: DROP CASCADE + retry, or pick new schema name |
| `sql_files` mode: files reference wrong schema | output_dir context lost between phases | Use `{target_database}.{target_schema}` literal in every file — no session variables |

---

## Verification Checklist

**Live mode:**
- [ ] Staleness check passed (GENERATOR + ZIPF available)
- [ ] All dimension tables have correct row count and unique PKs
- [ ] Fact table row count matches plan (+-1%)
- [ ] Zero orphan foreign keys across all relationships
- [ ] Gini coefficient for primary metric in expected range (ZIPF: 0.20-0.85 small, 0.45-0.85 medium, 0.55-0.85 large; NORMAL: 0.15-0.45)
- [ ] Monthly aggregation shows regional seasonal pattern
- [ ] At least 3 anomaly events detectable in weekly aggregation (if enabled)
- [ ] No NULL values in required/PK columns
- [ ] Data dictionary generated with all tables documented
- [ ] Rollback script generated and verified syntactically

**sql_files mode:**
- [ ] All .sql files written and non-empty
- [ ] Files numbered correctly (00-06, 99)
- [ ] SQL is syntactically valid (spot-check via `only_compile=true` if connection available)
- [ ] `06_verify.sql` contains all validation queries (row counts, Gini, FK integrity, seasonality)
- [ ] `99_rollback.sql` contains DROP SCHEMA CASCADE
- [ ] `README.md` documents all tables, columns, and distributions
- [ ] No hardcoded session-specific values (all parameterized via {database}.{schema} references)

---

## Rollback & Idempotency

### Cleanup Script

```sql
-- ROLLBACK: Synthetic Data Generation
-- Run this to remove all objects created by this skill.
-- WARNING: This is destructive. Review before executing.

DROP SCHEMA IF EXISTS {target_database}.{target_schema} CASCADE;
```

### Idempotency Contract

| Behavior | Implementation |
|----------|----------------|
| Error-if-exists (default) | Phase 3 gate halts if schema contains tables |
| Replace-always (user-confirmed) | User confirms DROP CASCADE, then re-run from Phase 3 |

### Mid-Phase Recovery

| Phase Interrupted At | Orphaned State | Recovery |
|---------------------|----------------|----------|
| Phase 3 | dim_date exists, no dims | Safe — re-run from Phase 4 |
| Phase 4, mid-dimension | Some dims created, others missing | Re-run Phase 4 (CTAS is idempotent with OR REPLACE) |
| Phase 5 | Fact table partially generated | DROP fact table, re-run Phase 5 |
| Phase 6 | Anomalies partially applied | Re-run Phase 6 (UPDATE is idempotent on same dates) |

---

## Session Notes

- **Context budget**: Load schema map entry only for the selected use case (not entire file). Load region section only for the selected region.
- **Subagent usage**: Max 1 subagent — only if context is tight, delegate entity name array generation or data dictionary writing.
- **Compaction**: After Phase 5 verification, safe to compact. Phases 6-8 only need fact table name and verification queries.
- **Multi-use-case**: To generate data for multiple use cases in same schema, run Phases 4-6 repeatedly with different use case params. Shared dim_date avoids duplication.

---

## References

- `references/SYNTH_SQL_PATTERNS.md` — 14 reusable SQL generation patterns
- `references/SYNTH_REGIONS.md` — 26 regional parameter sets (holidays, seasonality, currency)
- `references/SYNTH_USE_CASE_SCHEMA_MAP.md` — 46 use case -> schema mappings across 12 verticals
