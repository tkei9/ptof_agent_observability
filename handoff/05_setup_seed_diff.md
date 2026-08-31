# Notebook Diffs — obs_setup_seed.ipynb

## Changes

### ADD NEW CELL (or append to existing threshold_basis cell)

```sql
%sql
-- New threshold_basis rows for newly-wired incident sources
INSERT INTO mq_gmdf_dev.oil_obs.threshold_basis VALUES
('capability_error_rate', '>=0.8 error rate over 6h window with >=5 total calls',
 'dsa_optimize failed 153/153 over 4 days undetected; per-hour floor hid it because no single hour reached the volume floor. Window aggregation over 6h fixes this. §5.7 documents the failure.',
 '2026-08-28', 'provisional'),

('latency_anomaly_fixed_ceiling', '>30000ms with is_reliable=false (no reliable baseline)',
 'Trades per-capability IQR precision for a working detection path under low volume. anomaly_fixed_ceiling means a successful call took >30s regardless of baseline quality. Authorised simplification to give latency a working incident path today.',
 '2026-08-28', 'provisional'),

('blank_output', '>2% blank rate AND >3 blank AND >=10 total calls over 6h window',
 'Verified 0 rows historical as of Aug 28 — every unparseable row in this data is a success=false row with NULL response_parsed, not a blank output. Any occurrence is novel and warrants investigation.',
 '2026-08-28', 'not yet active'),

('schema_field_missing', 'field present in >=20% of 30d baseline rows AND entirely absent in current 24h window (>=10 current rows)',
 'Frequency guard: a field must appear in 20%+ of baseline to count as expected. Current window requires >=10 rows to prevent low-activity hours from triggering. Only field_missing is CRITICAL; field_added stays WARN.',
 '2026-08-28', 'provisional'),

('handover_delivery_rate', '>20% failure rate over 7d with >=10 real send attempts',
 'Baseline 9.3% since Jun 19 (15 of 162 attempts). Threshold at approximately 2x baseline filters noise. Floor of 10 attempts because daily volume is ~2 attempts and daily rates hit 33%/100% on a single failure.',
 '2026-08-28', 'provisional');
```

### PLACEHOLDER for capability_registry inserts (BLOCKED on user input)

```sql
%sql
-- Unregistered capabilities identified by Step 1 reconciliation.
-- WAITING for user to provide: is_generative, is_gxp_relevant, owner, silence_grace_hours
-- for each capability below.
--
-- Expected unregistered (from obs_verify FAIL and observed traffic):
--   dsa_copilot_step, dsa_session_summary, probe, saa_insight
--   Possibly also: dsa_ask, sev2_insight
--
-- Flag: watchout_narratives appears in v_llm_bronze traffic but has no documentation
-- anywhere. Already registered but requires confirmation of owner.
--
-- INSERT INTO mq_gmdf_dev.oil_obs.capability_registry VALUES
-- ('dsa_copilot_step',    <is_generative>, <is_gxp_relevant>, <expected_min_daily>,
--  <required_fields_array>, '<owner>', true, '<notes>', <silence_grace_hours>),
-- ('dsa_session_summary', ...),
-- ('probe',               ...),
-- ('saa_insight',         ...);
```
