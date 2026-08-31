# Execution Checklist

Apply changes in this order. Each step is independent of the next unless marked otherwise.

## Pre-flight (Step 0)
- [ ] Run dedup verification query (README.md Step 0)
- [ ] Confirm `violation_signature` AND `violation_tier` exist in `transport_violations`
- [ ] STOP if any stop condition met

## Pre-existing fixes (Step 0b)
- [ ] Run owner backfill MERGE (0b-1)
- [ ] Verify all 12 rows have non-null owner in signal_payload
- [ ] Identify orphaned digest rows with dispositioned counterparts (0b-2)
- [ ] Acknowledge only those specific rows
- [ ] Report proposal for weekly digest SQL addition → user decides

## Registry reconciliation (Step 1) — BLOCKS Step 3b
- [ ] Run unregistered capabilities query
- [ ] Run phantom capabilities query
- [ ] Report both lists to user
- [ ] **WAIT** for user to provide registry values
- [ ] Add INSERTs to obs_setup_seed
- [ ] Run obs_setup_seed INSERT cells
- [ ] Fix obs_verify grace hours assertion (26, not 18)

## Notebook changes — apply in task-graph order

### Task 02: agent_obs_latency_detection
- [ ] Print current cell-0 contents (record)
- [ ] Insert `latency_anomaly_findings` cell after cell-0
- [ ] Detach/reattach cluster
- [ ] Run new cell — confirm 0 rows (expected: traffic silence)
- [ ] Print current cell-7 contents (record)
- [ ] Insert `capability_error_rate_findings` cell after cell-7
- [ ] Detach/reattach cluster
- [ ] Run new cell — confirm 0 rows (expected: traffic silence)

### Task 03: agent_obs_mal_output
- [ ] Print current cell-1 contents (record)
- [ ] Insert `blank_output_findings` cell after cell-1
- [ ] Detach/reattach cluster
- [ ] Run new cell — confirm 0 rows (expected: genuinely 0)
- [ ] Print current cell-4 contents (record)
- [ ] Modify cell-4 (response_schema_drift) to add `finding_signature`
- [ ] Detach/reattach cluster
- [ ] Run modified cell — verify `finding_signature` column exists: `DESCRIBE response_schema_drift`

### Task 05: agent_obs_behavioral_correlation
- [ ] Print current cell-3 contents (record)
- [ ] Insert `handover_delivery_rate_findings` cell after cell-3
- [ ] Detach/reattach cluster
- [ ] Run new cell — confirm 0 rows (rate is currently 0%, below threshold)

### Task 06: agent_obs_alert
- [ ] Print current INCIDENT_SOURCES (record)
- [ ] Replace INCIDENT_SOURCES with new version
- [ ] Add 3 new DETECTOR_META entries
- [ ] Update `latency_anomaly` check to read from `latency_anomaly_findings`
- [ ] Update `capability_error_rate_sustained` check to read from findings table
- [ ] Detach/reattach cluster
- [ ] Run alert notebook interactively

### obs_setup_seed (manual, never scheduled)
- [ ] Add threshold_basis INSERT cell
- [ ] Run threshold_basis INSERT
- [ ] Add capability_registry INSERTs (when user provides values)

### agent_obs_verification (manual, never scheduled)
- [ ] Add new assertions cell
- [ ] Fix grace hours assertion (26 not 18)
- [ ] Run full verification
- [ ] Confirm no new FAILs introduced by these changes

## Post-deploy verification (Step 4 + 5)
- [ ] Run the summary query from README Step 4
- [ ] Confirm: keys ≈ rows for every detector
- [ ] Confirm: no detector has hundreds of rows for one capability
- [ ] Insert synthetic TEST- incidents (Step 5)
- [ ] Point webhook at test channel
- [ ] Call post_teams with synthetic incidents
- [ ] Inspect card renders for all new detectors
- [ ] Delete synthetic rows
- [ ] Confirm table count returned to prior state

## Items awaiting user input
- [ ] capability_registry values for unregistered capabilities
- [ ] Whether to add digest acknowledgement to weekly digest SQL
- [ ] Which capabilities get is_gxp_relevant = true (for hallucination_high)
