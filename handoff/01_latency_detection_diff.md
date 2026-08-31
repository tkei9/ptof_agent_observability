# Notebook Diffs — agent_obs_latency_detection.ipynb

## Current Cell Order
- cell-0: latency_anomalies
- cell-1: credential_fastfail_daily
- cell-2: write_lag_daily
- cell-3: latency_failures
- cell-4: capability_health
- cell-5: capability_silence
- cell-6: shift_context_missing
- cell-7: capability_error_rate_alert

## Changes

### INSERT NEW CELL after cell-0 (latency_anomalies)

```sql
%sql
-- latency_anomaly_findings — aggregated per (capability, model_config, verdict) for MERGE.
-- The original INCIDENT_SOURCES entry keyed on b.id (the per-call UUID), which made every
-- hourly poll from a slow capability a new incident. 529 incidents for one config.
-- Keying on the CONDITION (capability + model_config + verdict) means a persistently slow
-- capability is ONE incident whose detection_count climbs.
-- Widened from 'anomaly' only to include 'anomaly_fixed_ceiling': a >30s call with no
-- reliable baseline is worth an incident on its own (authorised simplification).
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.latency_anomaly_findings AS
SELECT
    capability, model_config, latency_verdict,
    max(latency_ms) AS worst_latency_ms,
    count(*) AS anomaly_count,
    max(p95_ms) AS baseline_p95_ms,
    max(anomaly_upper_bound_ms) AS anomaly_bound_ms,
    max(called_at) AS latest_called_at,
    sha2(concat_ws('|',
        coalesce(capability, '<null>'),
        coalesce(model_config, '<null>'),
        coalesce(latency_verdict, '<null>')
    ), 256) AS finding_signature,
    current_timestamp() AS detected_at
FROM mq_gmdf_dev.oil_obs.latency_anomalies
WHERE latency_verdict IN ('anomaly', 'anomaly_fixed_ceiling')
GROUP BY capability, model_config, latency_verdict;
```

### INSERT NEW CELL after cell-7 (capability_error_rate_alert)

```sql
%sql
-- capability_error_rate_findings — signature-keyed for obs_incidents MERGE.
-- Reuses capability_error_rate_alert which already applies the correct thresholds:
--   sum(total_calls) >= 5 AND error_rate_6h >= 0.8
-- aggregated over the 6h WINDOW total (not per-hour), which is what caught the sustained
-- dsa_optimize outage that the per-hour check missed.
-- Key on (capability, model_config) only — NOT the hour — so a sustained outage is one
-- incident whose detection_count climbs, not one incident per hour.
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.capability_error_rate_findings AS
SELECT
    capability, model_config,
    calls_6h, failures_6h, error_rate_6h,
    hours_with_data, hours_fully_failed, latest_hour,
    sha2(concat_ws('|',
        coalesce(capability, '<null>'),
        coalesce(model_config, '<null>')
    ), 256) AS finding_signature,
    current_timestamp() AS detected_at
FROM mq_gmdf_dev.oil_obs.capability_error_rate_alert;
```

## Final Cell Order (after changes)
- cell-0: latency_anomalies
- **cell-1: latency_anomaly_findings (NEW)**
- cell-2: credential_fastfail_daily (was cell-1)
- cell-3: write_lag_daily (was cell-2)
- cell-4: latency_failures (was cell-3)
- cell-5: capability_health (was cell-4)
- cell-6: capability_silence (was cell-5)
- cell-7: shift_context_missing (was cell-6)
- cell-8: capability_error_rate_alert (was cell-7)
- **cell-9: capability_error_rate_findings (NEW)**
