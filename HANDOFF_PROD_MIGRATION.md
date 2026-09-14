# Handoff: Prod Source Migration — Execute This Plan

## Your role
You are implementing a prod source migration for an SAA/ISH agent observability pipeline on
Databricks. The analysis and design are complete. This document tells you exactly what to change,
in what order, and why.

**Current status (2026-09-14):** All 9 steps are complete. Notebooks rewritten and verified;
retired files deleted; `ptof_obs_latency_detection.ipynb` renamed to
`ptof_obs_liveness_detection.ipynb` on disk (content already used the new name). Ready for
post-migration verification (see that section below).

## What you're doing
Switching the pipeline's source reads from dev (`mq_gmdf_dev.oil.*`) to prod
(`mq_gmdf_dp_prd.oil.*`). All writes stay in `mq_gmdf_dev.oil_obs.*`. The pipeline runs in
the dev workspace. Only the source data changes.

## Environment
- **Dev workspace:** profile `"Tyler Kei"`, catalog `mq_gmdf_dev`, schema `oil_obs`
- **Prod catalog:** `mq_gmdf_dp_prd`, schema `oil` (read-only)
- **SQL warehouse (dev):** `cd6c1145a46bec44`
- Always use `--profile "Tyler Kei"` for CLI commands

## The 4 tracked prod capabilities (output_type values)
`saa-display`, `sev2-insights`, `situational-awareness`, `summary`

## Architecture after migration
```
mq_gmdf_dp_prd.oil (READ-ONLY)         mq_gmdf_dev.oil_obs (WRITE)
───────────────────                     ───────────────────────────
ai_shift_outputs ──────► v_llm_bronze ──┬─► blank_output_findings
  (4 output_types)                      ├─► response_schema_drift
                                        ├─► capability_silence
                                        ├─► shift_context_missing
                                        └─► (heartbeat check in alert)
ptof_ish_audit ────────► v_ish_bronze ──┬─► handover_delivery_failures
                                        ├─► handover_delivery_rate[_findings]
ptof_etl_pipeline_audit► v_etl_bronze ──► etl_pipeline_health (NEW)
                                        ▼
                              obs_incidents → Teams Adaptive Card
```

## 8 active detectors
1. **Handover delivery failures** (CRITICAL) — email send failures from ISH audit
2. **Handover delivery rate** (CRITICAL) — 7-day rolling failure rate >20%
3. **Shift context missing** (WARN) — blank shift_date/shift_type/batch_nbr in outputs
4. **Blank output** (CRITICAL) — content is NULL/empty/trivial on an output record
5. **Schema drift / field missing** (CRITICAL) — JSON key appeared or disappeared vs nightly baseline
6. **Capability silence** (WARN) — output_type hasn't generated in >grace_hours
7. **Pipeline heartbeat** (CRITICAL) — zero outputs from any capability in 2 hours
8. **ETL pipeline health** (CRITICAL, NEW) — ETL source data refresh failed or stale >30min

## 7 dropped detectors (and why)
- **Latency anomalies** — no `latency_ms` in prod `ai_shift_outputs`
- **Error rate** — no `success`/`error_msg` in prod (all rows are successful outputs)
- **Hallucination** — 4.9% false positive rate; flagged computed numbers as hallucinated
- **Transport violations** — single transport (`cortex`), zero violations ever recorded
- **Prompt size drift** — stable templated prompts, zero findings in entire history
- **Credential fastfail** — `demo-claude-sonnet-4-6-pwc-omi` doesn't exist in prod
- **Rapid human correction** — 6 rows total, dormant, research-quality signal

## Prod source schemas

**`mq_gmdf_dp_prd.oil.ptof_primary__ai_shift_outputs`:**
```
id, shift_date, shift_type, batch_nbr, output_type, scheduler_run,
content (JSON string), model_config, generated_at, ingestion_ts
```
Volume: ~5,900 rows. output_types: saa-display (2,633), situational-awareness (2,633),
sev2-insights (572), summary (53). Content is 100% valid JSON. Zero blanks.

**`mq_gmdf_dp_prd.oil.ptof_ish_audit`:** (identical schema to dev)
```
id, action, entity_type, entity_id, user_id, user_email, ts, before_json, after_json, change_summary
```
335 rows. 28 HandoverEmail SEND events, all `sent=true`.

**`mq_gmdf_dp_prd.oil.ptof_etl_pipeline_audit`:**
```
run_id, run_timestamp, table_or_view, task_name, operation, status,
rows_written, total_rows, error_message, duration_seconds, ingestion_ts
```
25,089 successes, 1 failure. Runs every ~10-15 min. 19 tables per run.

## Execution plan — read /Users/L141230/.claude/plans/curious-painting-sun.md

The full step-by-step plan is in that file. It covers 9 steps in strict sequential order.

## PROGRESS — where this plan stands (resume from here)

Steps 1–7 are **complete** (notebooks rewritten, verified cell-by-cell). Step 8 (file
deletions) is the only remaining work. Step 9 (this handoff) is already written.

### ✅ Step 1 — ptof_obs_bronze_projection.ipynb (DONE)
Rewrote `v_llm_bronze` to read from prod `mq_gmdf_dp_prd.oil.ptof_primary__ai_shift_outputs`
with aliases `output_type→capability`, `generated_at→called_at`, `content→response_parsed`.
Added `write_lag_s`, `is_blank_output` (adapted for `content`), `response_chars`, `resp_v`.
Removed all dev-only columns (`success`, `error_msg`, `transport`, `latency_ms`, prompts,
`is_credential_fastfail`). Swapped `v_ish_bronze` FROM clause to prod `ptof_ish_audit` (schema
identical, all derived columns unchanged). Added new `v_etl_bronze` pass-through view over
`ptof_etl_pipeline_audit`. Notebook is 4 cells: markdown + 3 SQL view cells.

### ✅ Step 2 — ptof_obs_setup_seed.ipynb (DONE)
Added 4 prod capabilities to `capability_registry` (saa-display 2h grace, situational-awareness
2h grace, sev2-insights NULL grace, summary 36h grace — all `is_groundable=false`). Deactivated
old dev capabilities (saa_insight, sev2_insight, watchout_narratives) with migration notes.
Updated `summary` row to `is_groundable=false`. Removed `runtime_allowlist` cell, `_obs_watermark`
cell, and `runtime_observed` cell entirely. Rewrote `threshold_basis` with ETL pipeline entries
and marked all retired detector thresholds `status='not_applicable_prod'`. Expanded cleanup
drops to include all retired detector output tables + `v_ungrounded_tokens` view. Notebook is
4 cells: capability_registry, obs_incidents, threshold_basis, cleanup drops.

### ✅ Step 3 — ptof_obs_latency_detection.ipynb (DONE)
Renamed to `ptof_obs_liveness_detection`. Deleted 9 cells (latency_anomalies,
latency_anomaly_findings, credential_fastfail_daily, write_lag_daily, latency_failures,
capability_health, capability_error_rate_alert, capability_error_rate_findings,
prompt_size_drift). Kept `capability_silence` and `shift_context_missing` (comment updates
only — work through view aliases). Added `etl_pipeline_health` cell (CRITICAL, reads
`v_etl_bronze`, filters `status='failure'` in 24h window). Notebook is 4 cells: markdown +
3 SQL cells.

### ✅ Step 4 — ptof_obs_mal_output.ipynb (DONE)
Deleted env widget cell and `transport_violation_signatures` cell. Adapted
`blank_output_findings`: removed `WHERE b.success = true` and `b.is_credential_fastfail = false`,
removed `transport` from GROUP BY. Adapted `response_schema_drift`: removed same two filters
from both `current_keys` and `current_rows` CTEs. Notebooks is 3 cells: markdown +
blank_output_findings + response_schema_drift.

### ✅ Step 5 — ptof_obs_behavioral_correlation.ipynb (DONE)
Deleted `ish_entity_dim` and `rapid_human_correction` cells. Kept 3 handover delivery cells
(`handover_delivery_failures`, `handover_delivery_rate`, `handover_delivery_rate_findings`)
with comment-only updates — they read `v_ish_bronze` only, which now points at prod. ISH
schema is identical, no code change needed. Notebook is 4 cells: markdown + 3 SQL cells.

### ✅ Step 6 — ptof_obs_nightly_baseline.ipynb (DONE)
Deleted `capability_latency_baseline` cell. Adapted `response_field_baseline`: removed
`AND b.success = true` and `AND b.is_credential_fastfail = false` from the `eligible` CTE.
Notebook is 2 cells: markdown + response_field_baseline.

### ✅ Step 7 — ptof_obs_alert.ipynb (DONE)
Rewrote `INCIDENT_SOURCES` to 5 entries (removed latency_anomaly, runtime_violation,
runtime_violation_digest, hallucination_high, capability_error_rate_sustained,
prompt_size_drift; added etl_pipeline_failure). Rewrote `BACKTRACK` to 5 entries with prod
lineage strings (removed capability_error_rate_sustained, latency_anomaly, hallucination_high,
prompt_size_drift; added etl_pipeline_failure). Deleted all check() calls for dropped detectors
(latency, error rate, credential outage, write_lag, runtime_violation, hallucination,
prompt_size_drift). Kept check() calls for capability_silence, pipeline_heartbeat,
shift_context_missing, blank_output, schema_drift, schema_field_missing,
handover_delivery_rate, handover_delivery_new_failure, unacknowledged_critical,
long_running_incident. Added etl_pipeline_staleness and etl_pipeline_failure checks. Rewrote
`DETECTOR_META` to 5 entries (removed 6 dropped, added etl_pipeline_failure). Removed the
`runtime_violation` special-case grouping block from `post_teams`. Cells 1-2 (widgets,
check framework), cell 5 (auto-resolve), notify cell, and raise cell unchanged.

### ✅ Step 8 — Delete retired files (DONE)
Deleted `ptof_obs_hallucination_detection.ipynb`, `ptof_obs_weekly_runtime_digest.ipynb`,
`ptof_obs_verification.ipynb` (`HANDOFF.md`, `HANDOFF_PROD_INTEGRATION.md`,
`HANDOFF_PROD_REVIEW.md`, `PROD_MIGRATION_GAP_ANALYSIS.md` were already gone). Also renamed
`ptof_obs_latency_detection.ipynb` → `ptof_obs_liveness_detection.ipynb` on disk (the notebook's
own content, job task references, and cross-notebook comments already called it
`liveness_detection` — only the filename hadn't caught up) and fixed 3 stale
`ptof_obs_latency_detection` string references in `ptof_obs_bronze_projection.ipynb`,
`ptof_obs_alert.ipynb`, and `ptof_obs_setup_seed.ipynb`.

### ✅ Step 9 — Write HANDOFF_PROD_MIGRATION.md (DONE — this file)

## Critical design decisions (do NOT revisit)
- **capability_registry inner-join scoping is intentional** — only registered capabilities are monitored. Do NOT add unregistered-capability alerting.
- **`is_groundable = false` for all prod capabilities** — hallucination detection is dropped because the detector's regex+similarity approach flags computed numbers as hallucinated (4.9% false positive rate on `saa_insight`).
- **No dev enrichment LEFT JOIN in v1** — the v_llm_bronze view reads prod only. Dev enrichment (latency, errors, prompts) is Phase 2 after validating the dev-prod join is stable.
- **sev2-insights gets `silence_grace_hours = NULL`** — irregular cadence with natural 100+ hour gaps makes silence detection meaningless for this capability.

## Constraints
- **Do NOT modify prod catalog** — read-only access to `mq_gmdf_dp_prd`
- **All writes stay in `mq_gmdf_dev.oil_obs`**
- **Teams webhook stays in dev workspace** (`obs-alerting` secrets scope)
- **Profile `"Tyler Kei"` for all CLI commands**
- **No retired/unused code in notebooks** — delete cells, don't comment them out

## Post-migration verification
After all steps complete, run each notebook in order and verify:
1. `v_llm_bronze` returns rows with 4 output_types as `capability`
2. `capability_registry` has 4 active prod rows
3. `capability_silence` and `shift_context_missing` run clean
4. `blank_output_findings` and `response_schema_drift` run clean
5. Handover delivery detectors return prod ISH data
6. `response_field_baseline` computes 29 distinct fields across 4 output_types
7. Alert notebook runs with no UNAVAILABLE

## Phase 2 roadmap (future, not this migration)
- Dev enrichment LEFT JOIN for latency/error detection
- Cross-output numeric grounding (saa-display stats vs situational-awareness grounded_facts)
- Output cadence gap detection (inter-generated_at timing anomalies)
- Content signature staleness detection
