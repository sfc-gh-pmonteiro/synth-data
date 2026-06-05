# Synth SQL Patterns

Reusable Snowflake SQL generation patterns for synthetic data (14 patterns).
Each pattern uses GENERATOR(ROWCOUNT=>N) with Snowflake's native distribution functions.

---

### 1. Date Dimension Generator

Complete date dimension CTAS with fiscal quarter (Feb FY start), ISO week, and day attributes.

```sql
-- First: SET rowcount = DATEDIFF('day', '{start_date}'::DATE, '{end_date}'::DATE) + 1;
-- GENERATOR requires a constant. Use a session variable or hardcode the computed value.
-- For 2024-01-01 to 2025-12-31 = 731 rows.
CREATE OR REPLACE TABLE {database}.{schema}.dim_date AS
WITH date_spine AS (
  SELECT DATEADD('day', ROW_NUMBER() OVER (ORDER BY SEQ4()) - 1, '{start_date}'::DATE) AS full_date
  FROM TABLE(GENERATOR(ROWCOUNT => {row_count}))  -- Precompute: DATEDIFF('day', start, end) + 1
)
SELECT
  TO_NUMBER(TO_CHAR(full_date, 'YYYYMMDD'))  AS date_key,
  full_date,
  YEAR(full_date)                             AS year,
  QUARTER(full_date)                          AS quarter,
  MONTH(full_date)                            AS month,
  MONTHNAME(full_date)                        AS month_name,
  DAYOFWEEK(full_date)                        AS day_of_week,
  DAYNAME(full_date)                          AS day_name,
  IFF(DAYOFWEEK(full_date) IN (0, 6), TRUE, FALSE) AS is_weekend,
  WEEKOFYEAR(full_date)                       AS iso_week,
  CEIL(MOD(MONTH(full_date) + 10, 12) / 3.0) AS fiscal_quarter,
  DAYOFYEAR(full_date)                        AS day_of_year
FROM date_spine;
```

### 2. Time-Series Composition

Core formula combining trend + seasonality + noise for realistic time-series values.

```sql
{base_value}
  * (1 + {growth_rate} * DATEDIFF('month', '{start_date}'::DATE, d.full_date) / 12.0)
  * (1 + {seasonal_amplitude} * SIN(2 * 3.14159 * DAYOFYEAR(d.full_date) / 365.0))
  * (1 + NORMAL(0, {noise_pct}, RANDOM()))
AS generated_value
```

### 3. ZIPF Revenue Distribution

Generate power-law revenue values (~20% of entities generate ~80% of value).

```sql
CREATE OR REPLACE TABLE {database}.{schema}.dim_customer AS
WITH base AS (
  SELECT
    SEQ4() + 1 AS customer_id,
    ZIPF({alpha}, {n_customers}, RANDOM()) AS zipf_rank,
    UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS segment_seed
  FROM TABLE(GENERATOR(ROWCOUNT => {n_customers}))
)
SELECT
  customer_id,
  -- Categorical assignment: use Pattern #11 (UNIFORM seed), NOT zipf_rank thresholds
  CASE
    WHEN segment_seed < 0.05 THEN 'enterprise'
    WHEN segment_seed < 0.20 THEN 'mid_market'
    ELSE 'smb'
  END AS tier,
  -- Revenue: low rank (majority) = low value. High rank (rare) = high value.
  ROUND({base_value} * POWER(zipf_rank, 1.0 / {alpha}), 2) AS base_annual_revenue
FROM base;
```

**Formula direction**: ZIPF concentrates outputs at LOW rank numbers (P50~1, P90~8 for alpha=1.9, N=10000). Therefore:
- `base * POWER(rank, 1/alpha)` -> low rank = low value (majority get small amounts) CORRECT
- `max / POWER(rank, 1/alpha)` -> low rank = HIGH value (majority get max amounts) NEVER USE

**Alpha guidance:**
| Use Case | Alpha Range | N Parameter | Rationale |
|----------|-------------|-------------|-----------|
| FK assignment (customer_id, subscriber_id in fact tables) | 1.0-1.2 | dim_row_count | Ensures 99%+ of dimension keys are referenced. Alpha >1.5 leaves most FKs orphaned. |
| Revenue/amount metrics (revenue, cost, spend) | 1.5-2.2 | fact_row_count | Sufficient rank spread at all scales. Produces realistic Pareto 80/20. |
| Count metrics (er_visits, defect_count, complaints) | 2.0-2.5 | fact_row_count | Extreme skew is realistic for rare-event counts. |

> **Tested**: ZIPF(1.2, 5000) with 400K draws -> 4,955 distinct values (99%). ZIPF(1.6, 5000) -> only 2,739 (55%). Never use alpha >1.3 for FK assignment.

> **Coverage requires sufficient draws**: ZIPF(1.1, N) needs >= 40x draws to cover 99% of N keys. At 2.5:1 ratio only ~38% covered. When draw:key ratio < 10:1, use UNIFORM or hybrid (first N rows = sequential guarantee, remainder = ZIPF).

### 4. Entity Name Arrays

Realistic naming using ARRAY_CONSTRUCT in a CTE + random indexing.

```sql
-- Entity names via inline CTE (Snowflake SET only supports scalar values, NOT arrays)
WITH names AS (
  SELECT
    ARRAY_CONSTRUCT(
      'Apex Corp','Nebula Inc','Horizon Ltd','Vanguard Co','Summit LLC',
      'Meridian Group','Pinnacle Tech','Atlas Partners','Zenith Solutions','Nova Industries'
    ) AS company_names,  -- 10 elements → index 0..9
    ARRAY_CONSTRUCT(
      'New York|NY','Los Angeles|CA','Chicago|IL','Houston|TX','Phoenix|AZ',
      'Dallas|TX','Atlanta|GA','Miami|FL','Seattle|WA','Denver|CO'
    ) AS city_state      -- 10 elements → index 0..9
)
SELECT
  n.company_names[UNIFORM(0, 9, RANDOM())]::VARCHAR AS company_name,
  SPLIT_PART(n.city_state[UNIFORM(0, 9, RANDOM())]::VARCHAR, '|', 1) AS city,
  SPLIT_PART(n.city_state[UNIFORM(0, 9, RANDOM())]::VARCHAR, '|', 2) AS state
FROM TABLE(GENERATOR(ROWCOUNT => {n_rows})) CROSS JOIN names n;
```

> **Never use `SET` for arrays** — Snowflake `SET` only supports scalar values. `SET x = ARRAY_CONSTRUCT(...)` throws a syntax error. Always use a single-row CTE with `CROSS JOIN` instead.

> **UNIFORM bounds must be integer literals** — `UNIFORM(0, ARRAY_SIZE(arr)-1, RANDOM())` fails because UNIFORM's first two arguments must be constants. Pre-count your array elements and hardcode N-1 (e.g., 10 elements → `UNIFORM(0, 9, RANDOM())`).

> **Never use MOD for array indexing** — `MOD(SEQ4(), N)` creates deterministic cycles (N entries produce exactly N unique outputs regardless of row count). Use `UNIFORM(0, N-1, RANDOM())` for realistic name diversity (N*M combinations from two arrays).

### 5. Correlated NULL Injection

NULLs correlate across columns using a shared random seed per row.

```sql
SELECT
  *,
  CASE WHEN null_seed < 0.03 THEN NULL ELSE email END AS email,
  CASE WHEN null_seed < 0.05 THEN NULL ELSE phone END AS phone,
  CASE WHEN null_seed < 0.01 THEN NULL ELSE address END AS address
FROM (
  SELECT *, UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS null_seed
  FROM {source_table}
);
```

### 6. Anomaly Spike Injection

Date-targeted multipliers for known events (spikes, dips, outages).

```sql
SELECT
  *,
  value *
    CASE
      WHEN date_col BETWEEN '2024-11-25' AND '2024-11-28' THEN 3.0   -- Black Friday spike
      WHEN date_col BETWEEN '2024-07-01' AND '2024-07-05' THEN 0.4   -- July 4th dip
      WHEN date_col BETWEEN '2024-03-15' AND '2024-03-15' THEN 0.0   -- System outage
      ELSE 1.0
    END AS adjusted_value
FROM {fact_table};
```

### 7. Geographic Distribution

Weighted random region assignment using cumulative probability thresholds.

```sql
SELECT
  *,
  CASE
    WHEN region_seed < 0.40 THEN 'us_east'
    WHEN region_seed < 0.65 THEN 'us_west'
    WHEN region_seed < 0.80 THEN 'eu_west'
    WHEN region_seed < 0.90 THEN 'apac'
    WHEN region_seed < 0.96 THEN 'latam'
    ELSE 'mena'
  END AS region
FROM (
  SELECT *, UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS region_seed
  FROM TABLE(GENERATOR(ROWCOUNT => {n_rows}))
);
```

### 8. Referential Integrity Verification

Anti-join query returning orphan FK count (should return 0 for valid data).

```sql
SELECT
  '{fact_table}.{fk_column} -> {dim_table}.{pk_column}' AS relationship,
  COUNT(*) AS orphan_count
FROM {database}.{schema}.{fact_table} f
LEFT JOIN {database}.{schema}.{dim_table} d
  ON f.{fk_column} = d.{pk_column}
WHERE d.{pk_column} IS NULL;
```

### 9. Distribution Validation (Gini Coefficient)

Gini coefficient using ROW_NUMBER for correct results on skewed distributions. Expected range:
- **ZIPF-distributed** metrics (revenue, spend, cost): 0.20-0.85 (small scale); 0.45-0.85 (medium); 0.55-0.85 (large)
- **NORMAL-distributed** metrics (ARPU, scores, durations): 0.15-0.45
- **UNIFORM-distributed** metrics: ~0 (skip — meaningless for uniform data)

```sql
WITH ordered AS (
  SELECT
    {metric_column} AS val,
    ROW_NUMBER() OVER (ORDER BY {metric_column}) AS rn,
    COUNT(*) OVER () AS n
  FROM {database}.{schema}.{table}
  WHERE {metric_column} > 0
)
SELECT
  ROUND((2.0 * SUM(rn * val) / (MAX(n) * SUM(val))) - (MAX(n) + 1.0) / MAX(n), 4) AS gini_coefficient,
  APPROX_PERCENTILE(val, 0.5) AS p50,
  APPROX_PERCENTILE(val, 0.9) AS p90,
  APPROX_PERCENTILE(val, 0.99) AS p99
FROM ordered;
```

> **Why ROW_NUMBER, not PERCENT_RANK**: PERCENT_RANK starts at 0 (not 1/N), creating a systematic bias that returns negative Gini values on highly skewed distributions (e.g., ZIPF). The ROW_NUMBER formula `(2*SUM(rank*value))/(N*SUM(value)) - (N+1)/N` is the standard discrete Gini and always returns [0, 1).

> **Scale effect**: At `row_scale=small` (10K fact rows), ZIPF Gini typically falls to 0.20-0.55 due to reduced rank separation when N (the ZIPF parameter) is small. Use `fact_row_count` as N for metric columns to maximize rank spread. At medium+ scale (100K+ rows), expect 0.45+ consistently.

### 10. Holiday Multiplier Join (Compound)

Apply date-based seasonal multipliers to fact values via calendar join.
**CRITICAL**: Always pre-aggregate multipliers per date before joining to fact — overlapping holiday events cause row fanout that corrupts the fact table grain.

Uses **compound multiplication** (product of overlapping events) because real-world effects stack: if "Aguinaldo (+1.4x)" overlaps "Weekend (+1.1x)", the real effect is 1.4 * 1.1 = 1.54x, not MAX(1.4, 1.1) = 1.4x.

```sql
-- Compound multiplier: product of all overlapping event multipliers per date
-- Uses EXP(SUM(LN(x))) = PRODUCT(x) since Snowflake has no native PRODUCT aggregate
SELECT
  f.*,
  f.base_value * COALESCE(hm.compound_value_multiplier, 1.0) AS adjusted_value
FROM {database}.{schema}.{fact_table} f
LEFT JOIN (
  SELECT
    d2.date_key,
    EXP(SUM(LN(h2.value_multiplier))) AS compound_value_multiplier
  FROM {database}.{schema}.dim_date d2
  JOIN {database}.{schema}.holiday_calendar h2
    ON d2.full_date BETWEEN h2.start_date AND h2.end_date
  WHERE h2.value_multiplier > 0  -- guard against LN(0)
  GROUP BY d2.date_key
) hm ON f.date_id = hm.date_key;
```

> **NEVER join holiday_calendar directly to fact** — events like "Carnival + Tax Season" overlap on the same dates, creating duplicate rows per transaction. The GROUP BY + EXP(SUM(LN())) ensures exactly one compound multiplier per date.

> **Edge case**: If a holiday has `value_multiplier = 0` (complete outage), handle it separately with a CASE before the compound calc, because LN(0) is undefined.

### 11. Single-Seed Categorical Assignment

Correct multi-category CASE using one uniform seed per row with cumulative thresholds.

**WRONG** — different seeds per CASE branch produces independent Bernoulli trials, not a probability partition:
```sql
-- DON'T: each branch is an independent draw
CASE
  WHEN UNIFORM(0::FLOAT, 1::FLOAT, seed1) < 0.65 THEN 'A'
  WHEN UNIFORM(0::FLOAT, 1::FLOAT, seed2) < 0.80 THEN 'B'
  ELSE 'C'
END
```

**RIGHT** — single seed, cumulative thresholds forming a partition of [0, 1):
```sql
WITH base AS (
  SELECT *, UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS cat_seed
  FROM {source_or_generator}
)
SELECT
  CASE
    WHEN cat_seed < 0.65 THEN '{category_1}'       -- 65%
    WHEN cat_seed < 0.80 THEN '{category_2}'       -- 15%
    WHEN cat_seed < 0.90 THEN '{category_3}'       -- 10%
    ELSE '{category_4}'                             -- 10%
  END AS category
FROM base;
```

> **Key rule**: Thresholds must be cumulative (0.65, 0.80, 0.90) not individual probabilities (0.65, 0.15, 0.10). The same `cat_seed` column is referenced in every WHEN clause. Generate ONE seed per categorical column in the base CTE — never reuse the same seed for multiple independent categoricals.

### 12. Weighted Date Assignment

Distribute fact rows non-uniformly across dates using monthly_activity_index and holiday volume_multiplier. This produces more rows on busy days and fewer on slow days, reflecting real transaction patterns.

```sql
-- Step 1: Monthly activity index via CTE (never use SET for arrays)
WITH monthly_idx AS (
  SELECT ARRAY_CONSTRUCT({monthly_activity_csv}) AS idx  -- 12 floats from SYNTH_REGIONS
),
-- Step 2: Compute per-date weight (activity index * volume multiplier)
date_weights AS (
  SELECT
    d.date_key,
    d.full_date,
    mi.idx[MONTH(d.full_date) - 1]::FLOAT AS monthly_weight,
    COALESCE(hv.compound_volume_mult, 1.0) AS volume_mult
  FROM {database}.{schema}.dim_date d
  CROSS JOIN monthly_idx mi
  LEFT JOIN (
    SELECT d2.date_key, EXP(SUM(LN(h2.volume_multiplier))) AS compound_volume_mult
    FROM {database}.{schema}.dim_date d2
    JOIN {database}.{schema}.holiday_calendar h2
      ON d2.full_date BETWEEN h2.start_date AND h2.end_date
    WHERE h2.volume_multiplier > 0
    GROUP BY d2.date_key
  ) hv ON d.date_key = hv.date_key
),
-- Step 3: Build cumulative distribution (CDF) over dates
date_cdf AS (
  SELECT
    date_key,
    SUM(monthly_weight * volume_mult) OVER (ORDER BY date_key) /
      SUM(monthly_weight * volume_mult) OVER () AS cum_prob
  FROM date_weights
)
-- Step 4: Assign each fact row a date by sampling the CDF
-- In the fact CTAS, join each row's UNIFORM(0,1) date_seed to the CDF:
SELECT dc.date_key
FROM (SELECT UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS date_seed FROM raw_rows) r
JOIN date_cdf dc ON dc.cum_prob >= r.date_seed
QUALIFY ROW_NUMBER() OVER (PARTITION BY r.date_seed ORDER BY dc.cum_prob) = 1;
```

> **CRITICAL: QUALIFY and LEFT JOIN cannot coexist in the same subquery**. When combining Pattern #12 (date CDF with QUALIFY) and Pattern #10 (holiday value multiplier LEFT JOIN), you MUST separate them into two CTEs:
>
> ```sql
> -- CTE 1: Date assignment with QUALIFY (its own CTE)
> with_dates AS (
>   SELECT r.*, dc.date_key
>   FROM raw_rows r
>   JOIN date_cdf dc ON dc.cum_prob >= r.date_seed
>   QUALIFY ROW_NUMBER() OVER (PARTITION BY r.row_id ORDER BY dc.cum_prob) = 1
> ),
> -- CTE 2: Holiday value multiplier (separate LEFT JOIN)
> with_holidays AS (
>   SELECT wd.*, COALESCE(hvm.compound_value_multiplier, 1.0) AS holiday_mult
>   FROM with_dates wd
>   LEFT JOIN holiday_value_mults hvm ON wd.date_key = hvm.date_key
> )
> ```

> **Simpler alternative for moderate skew**: If the weight range is mild (0.7-1.5x), a rejection-sampling approach also works: generate UNIFORM dates, then randomly DELETE rows on low-weight dates at rate `1 - weight/max_weight`. This is simpler SQL but wastes generated rows.

> **Large-scale alternative (row_scale=large)**: For 1M+ fact rows, the CDF cross-join creates a large intermediate (~700M rows for 1M × 731 dates). Alternative: pre-bucket dates into N quantile bins (e.g., 1000), assign each row to a bucket via `FLOOR(date_seed * 1000)`, then UNIFORM within the bucket's date range. This trades precision for O(N) instead of O(N×D) intermediate rows.

### 13. Intraday Timestamp Generation

Generate realistic time-of-day patterns for fact tables that benefit from hourly granularity. Uses a bimodal distribution (lunch + evening peaks for retail, or custom peaks per vertical).

```sql
-- Retail bimodal: peaks at 12-14h and 18-20h
-- Method: weighted hour selection via cumulative thresholds
WITH hour_weights AS (
  -- 24 weights representing relative traffic per hour (0-23)
  -- Retail pattern: low overnight, lunch spike, evening spike
  SELECT COLUMN1 AS hour_val, COLUMN2 AS weight FROM VALUES
    (0, 0.02), (1, 0.01), (2, 0.01), (3, 0.01), (4, 0.01), (5, 0.02),
    (6, 0.03), (7, 0.04), (8, 0.06), (9, 0.07), (10, 0.08), (11, 0.09),
    (12, 0.10), (13, 0.09), (14, 0.07), (15, 0.05), (16, 0.04), (17, 0.05),
    (18, 0.08), (19, 0.09), (20, 0.07), (21, 0.05), (22, 0.03), (23, 0.02)
),
hour_cdf AS (
  SELECT hour_val, SUM(weight) OVER (ORDER BY hour_val) AS cum_prob
  FROM hour_weights
)
-- In fact generation, for each row:
SELECT
  DATEADD('hour', hc.hour_val,
    DATEADD('minute', UNIFORM(0, 59, RANDOM()),
      d.full_date::TIMESTAMP)) AS transaction_timestamp,
  hc.hour_val AS transaction_hour
FROM raw_fact_rows r
JOIN hour_cdf hc ON hc.cum_prob >= r.hour_seed
  QUALIFY ROW_NUMBER() OVER (PARTITION BY r.row_id ORDER BY hc.cum_prob) = 1;
```

**Vertical-specific hour patterns:**
| Vertical | Peak Hours | Pattern |
|----------|-----------|---------|
| Retail | 12-14, 18-20 | Bimodal (lunch + evening) |
| Telecom | 18-22 | Evening heavy |
| Media/Streaming | 19-23 | Evening/night |
| Financial Services | 9-16 | Business hours |
| Manufacturing | 6-14, 14-22 | Shift-based (flat within shift) |
| Healthcare | 8-17 | Business hours with ER flat-24h |

> **When to use**: Add intraday timestamps when the use case benefits from time-of-day analysis (retail POS, streaming sessions, network traffic). Skip for monthly-grain facts (insurance policy periods, credit applications).

#### Combining Patterns #12 + #13 (Date + Intraday)

When a fact table needs both weighted date assignment AND intraday timestamps, integrate them in one CTAS. Key: generate BOTH `date_seed` and `hour_seed` in the initial `raw_rows` CTE. Each CDF join uses its own QUALIFY in its own CTE (per BUG-3 fix above).

```sql
WITH monthly_idx AS (
  SELECT ARRAY_CONSTRUCT({monthly_activity_csv}) AS idx
),
date_weights AS (
  SELECT d.date_key, d.full_date,
    mi.idx[MONTH(d.full_date) - 1]::FLOAT * COALESCE(hv.compound_volume_mult, 1.0) AS weight
  FROM {database}.{schema}.dim_date d
  CROSS JOIN monthly_idx mi
  LEFT JOIN ({volume_mult_subquery}) hv ON d.date_key = hv.date_key
),
date_cdf AS (
  SELECT date_key, full_date,
    SUM(weight) OVER (ORDER BY date_key) / SUM(weight) OVER () AS cum_prob
  FROM date_weights
),
hour_weights AS (
  SELECT COLUMN1 AS hour_val, COLUMN2 AS weight FROM VALUES
    (0,0.02),(1,0.01),(2,0.01),(3,0.01),(4,0.01),(5,0.02),
    (6,0.03),(7,0.04),(8,0.06),(9,0.07),(10,0.08),(11,0.09),
    (12,0.10),(13,0.09),(14,0.07),(15,0.05),(16,0.04),(17,0.05),
    (18,0.08),(19,0.09),(20,0.07),(21,0.05),(22,0.03),(23,0.02)
),
hour_cdf AS (
  SELECT hour_val, SUM(weight) OVER (ORDER BY hour_val) AS cum_prob
  FROM hour_weights
),
raw_rows AS (
  SELECT
    ROW_NUMBER() OVER (ORDER BY SEQ4()) AS row_id,
    UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS date_seed,
    UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS hour_seed,
    -- other seeds...
  FROM TABLE(GENERATOR(ROWCOUNT => {n_rows}))
),
-- Pattern #12: Date assignment (QUALIFY in its own CTE)
with_dates AS (
  SELECT r.*, dc.date_key, dc.full_date
  FROM raw_rows r
  JOIN date_cdf dc ON dc.cum_prob >= r.date_seed
  QUALIFY ROW_NUMBER() OVER (PARTITION BY r.row_id ORDER BY dc.cum_prob) = 1
),
-- Pattern #13: Hour assignment (QUALIFY in its own CTE)
with_time AS (
  SELECT wd.*, hc.hour_val
  FROM with_dates wd
  JOIN hour_cdf hc ON hc.cum_prob >= wd.hour_seed
  QUALIFY ROW_NUMBER() OVER (PARTITION BY wd.row_id ORDER BY hc.cum_prob) = 1
)
SELECT
  wt.row_id,
  wt.date_key,
  DATEADD('hour', wt.hour_val,
    DATEADD('minute', UNIFORM(0, 59, RANDOM()),
      wt.full_date::TIMESTAMP)) AS transaction_timestamp,
  ...
FROM with_time wt;
```

---

### 14. Boolean Flag Generation

Generate boolean columns with a specified probability of TRUE. Uses a UNIFORM seed (same mechanism as Pattern #11 for a binary split).

```sql
-- In the base CTE, generate one seed per boolean column:
UNIFORM(0::FLOAT, 1::FLOAT, RANDOM()) AS flag_seed

-- In the final SELECT:
IFF(flag_seed < {probability}, TRUE, FALSE) AS {flag_column}
```

**Common probabilities by domain:**

| Flag | Typical Probability | Domain |
|------|-------------------|--------|
| fraud_flag | 0.008-0.04 | Financial, Insurance |
| anomaly_flag | 0.01-0.03 | Manufacturing, IoT |
| churn | 0.06-0.15 | SaaS, Telecom, Media |
| converted | 0.04-0.15 | Retail, SaaS |
| stp_flag | 0.55-0.65 | Insurance (straight-through processing) |
| on_time | 0.85-0.92 | Logistics |
| tamper_flag | 0.003-0.01 | Energy/Utilities |
| promo_flag | 0.10-0.20 | Retail |

> **Schema map notation**: `flag=UNIFORM(p)` means P(TRUE) = p. The pattern is identical to a two-category Pattern #11 with threshold at p. Generate one `UNIFORM(0::FLOAT, 1::FLOAT, RANDOM())` seed per boolean column in the base CTE — never reuse seeds across columns.
