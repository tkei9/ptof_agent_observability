# Notebook Diffs — agent_obs_mal_output.ipynb

## Current Cell Order
- cell-0: `dbutils.widgets.text("env", "dev")`
- cell-1: blank_output_incidents
- cell-2: transport_violations
- cell-3: transport_violation_signatures
- cell-4: response_schema_drift

## Changes

### INSERT NEW CELL after cell-1 (blank_output_incidents)

```sql
%sql
-- blank_output_findings — aggregated over 6h window for MERGE dedup.
-- Key on (capability, model_config), NOT the hour — a sustained blank-output condition
-- is one incident. The alert check thresholds (>2% rate, >3 blank, >=10 total) are
-- applied here rather than in INCIDENT_SOURCES WHERE clause so the findings table
-- contains only actionable rows.
-- Currently verified as genuinely 0 rows (every unparseable row in this data is a
-- success=false row with NULL response_parsed, not a blank output). Wired so it works
-- when it fires.
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.blank_output_findings AS
SELECT
    capability, model_config,
    sum(blank_output_count) AS blank_count_window,
    sum(total_successful_calls) AS total_calls_window,
    round(sum(blank_output_count) * 1.0 / nullif(sum(total_successful_calls), 0), 4) AS blank_rate_window,
    max(hour) AS latest_hour,
    sha2(concat_ws('|',
        coalesce(capability, '<null>'),
        coalesce(model_config, '<null>')
    ), 256) AS finding_signature,
    current_timestamp() AS detected_at
FROM mq_gmdf_dev.oil_obs.blank_output_incidents
WHERE hour >= date_trunc('HOUR', current_timestamp() - INTERVAL 6 HOURS)
GROUP BY capability, model_config
HAVING sum(blank_output_count) > 3
   AND sum(total_successful_calls) >= 10
   AND sum(blank_output_count) * 1.0 / nullif(sum(total_successful_calls), 0) > 0.02;
```

### MODIFY cell-4 (response_schema_drift)

Add `finding_signature` column to the final SELECT. The full replacement cell:

```sql
%sql
-- response_schema_drift — top-level key comparison via explode.
-- Two earlier approaches failed on this data:
--   1. DDL string equality: nullability variation alone changes the string.
--   2. Regex 'name:' extraction from DDL: matches at every nesting depth, so
--      proposal.op_fields.insert_after_batch was hoisted and read as top-level drift.
-- Exploding a map<string,string> cast yields the top-level key set at the correct depth.
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.response_schema_drift AS
WITH baseline_keys AS (
  SELECT b.capability, k.key AS field_name, count(*) AS baseline_present
  FROM mq_gmdf_dev.oil_obs.v_llm_bronze b
  JOIN mq_gmdf_dev.oil_obs.capability_registry r
    ON r.capability = b.capability AND r.active = true
  LATERAL VIEW explode(from_json(cast(b.response_parsed AS STRING), 'map<string,string>')) k AS key, value
  WHERE b.success = true AND b.is_blank_output = false
    AND b.is_credential_fastfail = false
    AND b.called_at >= current_timestamp() - INTERVAL 30 DAYS
  GROUP BY b.capability, k.key
),
baseline_rows AS (
  SELECT b.capability, count(*) AS n_rows
  FROM mq_gmdf_dev.oil_obs.v_llm_bronze b
  WHERE b.success = true AND b.is_blank_output = false
    AND b.is_credential_fastfail = false
    AND b.called_at >= current_timestamp() - INTERVAL 30 DAYS
  GROUP BY b.capability
),
current_keys AS (
  SELECT b.capability, k.key AS field_name, count(*) AS current_present
  FROM mq_gmdf_dev.oil_obs.v_llm_bronze b
  JOIN mq_gmdf_dev.oil_obs.capability_registry r
    ON r.capability = b.capability AND r.active = true
  LATERAL VIEW explode(from_json(cast(b.response_parsed AS STRING), 'map<string,string>')) k AS key, value
  WHERE b.success = true AND b.is_blank_output = false
    AND b.is_credential_fastfail = false
    AND b.called_at >= current_timestamp() - INTERVAL 24 HOURS
  GROUP BY b.capability, k.key
),
current_rows AS (
  SELECT b.capability, count(*) AS n_rows
  FROM mq_gmdf_dev.oil_obs.v_llm_bronze b
  WHERE b.success = true AND b.is_blank_output = false
    AND b.is_credential_fastfail = false
    AND b.called_at >= current_timestamp() - INTERVAL 24 HOURS
  GROUP BY b.capability
)
SELECT
    coalesce(bk.capability, ck.capability)          AS capability,
    coalesce(bk.field_name, ck.field_name)          AS field_name,
    bk.baseline_present,
    br.n_rows                                       AS baseline_rows,
    round(bk.baseline_present * 1.0 / br.n_rows, 3) AS baseline_presence_rate,
    coalesce(ck.current_present, 0)                 AS current_present,
    cr.n_rows                                       AS current_rows,
    CASE WHEN bk.field_name IS NULL THEN 'field_added'
         ELSE 'field_missing' END                   AS drift_type,
    true                                            AS schema_changed,
    sha2(concat_ws('|',
        coalesce(coalesce(bk.capability, ck.capability), '<null>'),
        coalesce(coalesce(bk.field_name, ck.field_name), '<null>')
    ), 256) AS finding_signature,
    current_timestamp()                             AS detected_at
FROM      baseline_keys bk
FULL JOIN current_keys  ck ON ck.capability = bk.capability AND ck.field_name = bk.field_name
LEFT JOIN baseline_rows br ON br.capability = coalesce(bk.capability, ck.capability)
LEFT JOIN current_rows  cr ON cr.capability = coalesce(bk.capability, ck.capability)
WHERE cr.n_rows >= 10
  AND (bk.field_name IS NULL
       OR (coalesce(ck.current_present, 0) = 0
           AND bk.baseline_present * 1.0 / br.n_rows >= 0.2));
```

## Final Cell Order (after changes)
- cell-0: widgets
- cell-1: blank_output_incidents
- **cell-2: blank_output_findings (NEW)**
- cell-3: transport_violations (was cell-2)
- cell-4: transport_violation_signatures (was cell-3)
- cell-5: response_schema_drift WITH finding_signature (MODIFIED, was cell-4)
