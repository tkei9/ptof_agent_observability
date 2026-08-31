# Notebook Diffs — agent_obs_behavioral_correlation.ipynb

## Current Cell Order
- cell-0: ish_entity_dim
- cell-1: rapid_human_correction
- cell-2: handover_delivery_failures
- cell-3: handover_delivery_rate

## Changes

### INSERT NEW CELL after cell-3 (handover_delivery_rate)

```sql
%sql
-- handover_delivery_rate_findings — single-row when rate exceeds threshold.
-- Distinct from handover_delivery (per-failure, keyed on ish_row_id) — this is the
-- DETERIORATION signal. The rate catches worsening; the per-failure check catches any
-- single unacknowledged recurrence. Both are needed (md §8.4).
-- Literal string key: one condition, one incident, ever. detection_count carries persistence.
-- Floor of 10 attempts because a 7-day window at ~2 attempts/day swings wildly on small
-- numbers (daily rates hit 33% and 100% on a single failure).
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.handover_delivery_rate_findings AS
SELECT
    failure_pct_7d, sent_ok, failed, last_attempt,
    'handover_delivery_rate_7d' AS finding_signature,
    current_timestamp() AS detected_at
FROM mq_gmdf_dev.oil_obs.handover_delivery_rate
WHERE failure_pct_7d > 20 AND (sent_ok + failed) >= 10;
```

## Final Cell Order (after changes)
- cell-0: ish_entity_dim
- cell-1: rapid_human_correction
- cell-2: handover_delivery_failures
- cell-3: handover_delivery_rate
- **cell-4: handover_delivery_rate_findings (NEW)**
