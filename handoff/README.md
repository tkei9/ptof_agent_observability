# ISH Agent Observability — Wire All Detectors to obs_incidents

## Implementation Handoff

**Date:** 2026-08-28  
**Task:** Make every ISH observability detector that can produce a real finding route into obs_incidents with correct dedup, severity, and Teams rendering.  
**Status:** Plan approved. Local .ipynb files edited where possible (new cells inserted in latency_detection, mal_output, behavioral_correlation). The alert notebook changes (INCIDENT_SOURCES + DETECTOR_META) require manual application in Databricks per `04_alert_diff.md` because the notebook is a single mega-cell.

### What's been applied locally (in the .ipynb files)
- `agent_obs_latency_detection.ipynb` — 2 new cells inserted (latency_anomaly_findings, capability_error_rate_findings)
- `agent_obs_mal_output.ipynb` — 1 new cell inserted (blank_output_findings), 1 cell modified (response_schema_drift + finding_signature)
- `agent_obs_behavioral_correlation.ipynb` — 1 new cell inserted (handover_delivery_rate_findings)

### What needs manual application in Databricks
- `agent_obs_alert.ipynb` — INCIDENT_SOURCES replacement + DETECTOR_META additions (see `04_alert_diff.md`)
- `obs_setup_seed.ipynb` — threshold_basis INSERTs + capability_registry INSERTs (see `05_setup_seed_diff.md`)
- `agent_obs_verification.ipynb` — new assertion cell + grace hours fix (see `06_verification_diff.md`)

---

## Continuation Prompt

If context runs out, start a fresh session with:

> Read `/Users/L141230/Downloads/agent_obs/handoff/README.md` and all files in that directory. Continue implementing the ISH observability wiring plan. The task spec is in the user's original message — re-read it. The plan is at `/Users/L141230/.claude/plans/twinkling-percolating-firefly.md`. Apply changes one notebook at a time, showing diffs before editing. Key constraint: this is a Databricks workspace with no Git; all tables are in `mq_gmdf_dev.oil_obs`; upstream traffic has been silent for 17+ hours so empty results are expected.

---

## Step 0 — Dedup Verification (run in Databricks)

```sql
-- Run this FIRST before any changes
SELECT detector, length(source_row_id) AS key_len, count(*) AS rows_,
       count_if(resolved_at IS NULL) AS still_open,
       min(first_detected) AS oldest, max(first_detected) AS newest
FROM mq_gmdf_dev.oil_obs.obs_incidents
WHERE detector LIKE 'runtime_violation%'
GROUP BY detector, length(source_row_id)
ORDER BY detector, key_len;

-- Expected (verified 2026-08-28):
-- runtime_violation         36  675   still_open=0   newest 2026-08-27T16:53:05
-- runtime_violation         64    8   still_open=8   oldest 2026-08-27T16:58:00
-- runtime_violation_digest  64    4   still_open=4   oldest 2026-08-27T16:58:02

-- ALSO confirm tiering is deployed:
DESCRIBE mq_gmdf_dev.oil_obs.transport_violations;
-- Must have BOTH violation_signature AND violation_tier
```

**STOP conditions:**
- Any `key_len = 36` group with `still_open > 0`
- Any `key_len = 36` newest after earliest `key_len = 64` oldest
- `violation_tier` missing from `transport_violations`

---

## Step 0b-1 — Backfill Owner on Open Incidents

```sql
MERGE INTO mq_gmdf_dev.oil_obs.obs_incidents t
USING (
  SELECT i.detector, i.source_row_id, coalesce(r.owner, 'unassigned') AS owner_now
  FROM mq_gmdf_dev.oil_obs.obs_incidents i
  LEFT JOIN mq_gmdf_dev.oil_obs.capability_registry r ON r.capability = i.capability
  WHERE i.detector IN ('runtime_violation', 'runtime_violation_digest')
    AND i.resolved_at IS NULL
) s
ON t.detector = s.detector AND t.source_row_id = s.source_row_id
WHEN MATCHED THEN UPDATE SET t.signal_payload = to_json(map(
    'model_config',    get_json_object(t.signal_payload, '$.model_config'),
    'transport',       get_json_object(t.signal_payload, '$.transport'),
    'scheduler_run',   get_json_object(t.signal_payload, '$.scheduler_run'),
    'violation_type',  get_json_object(t.signal_payload, '$.violation_type'),
    'violation_tier',  get_json_object(t.signal_payload, '$.violation_tier'),
    'calls_in_window', get_json_object(t.signal_payload, '$.calls_in_window'),
    'owner',           s.owner_now
));

-- Verify:
SELECT detector, source_row_id, get_json_object(signal_payload, '$.owner') AS owner
FROM mq_gmdf_dev.oil_obs.obs_incidents
WHERE detector IN ('runtime_violation', 'runtime_violation_digest')
  AND resolved_at IS NULL;
-- All 12 rows should have non-null owner
```

---

## Step 0b-2 — Close Orphaned Digest WARN Rows

```sql
-- First: identify which digest incidents have a dispositioned runtime_observed counterpart
SELECT 
    i.source_row_id,
    i.capability,
    get_json_object(i.signal_payload, '$.model_config') AS model_config,
    ro.disposition,
    ro.dispositioned_by,
    ro.dispositioned_at
FROM mq_gmdf_dev.oil_obs.obs_incidents i
JOIN mq_gmdf_dev.oil_obs.runtime_observed ro
  ON ro.capability = i.capability
  AND ro.violation_signature = i.source_row_id
WHERE i.detector = 'runtime_violation_digest'
  AND i.acknowledged_at IS NULL
  AND ro.disposition IS NOT NULL;

-- Then acknowledge ONLY those:
UPDATE mq_gmdf_dev.oil_obs.obs_incidents
SET acknowledged_by = 'auto:digest_disposition_sync',
    acknowledged_at = current_timestamp()
WHERE detector = 'runtime_violation_digest'
  AND acknowledged_at IS NULL
  AND source_row_id IN (
    SELECT ro.violation_signature
    FROM mq_gmdf_dev.oil_obs.runtime_observed ro
    WHERE ro.disposition IS NOT NULL
  );
```

**Proposed addition to weekly digest** (not yet implemented — user decides):
After the generated `UPDATE runtime_observed SET disposition='sanctioned'...` SQL, also generate:
```sql
UPDATE mq_gmdf_dev.oil_obs.obs_incidents 
SET acknowledged_by = '<user>', acknowledged_at = current_timestamp()
WHERE detector = 'runtime_violation_digest' 
  AND source_row_id = '<violation_signature>'
  AND acknowledged_at IS NULL;
```

---

## Step 1 — Reconcile capability_registry

```sql
-- 1a: Unregistered capabilities (no time bound!)
SELECT b.capability, count(*) AS calls, max(b.called_at) AS last_call
FROM mq_gmdf_dev.oil_obs.v_llm_bronze b
LEFT JOIN mq_gmdf_dev.oil_obs.capability_registry r ON r.capability = b.capability
WHERE r.capability IS NULL
GROUP BY b.capability ORDER BY calls DESC;

-- 1b: Phantoms
SELECT r.capability FROM mq_gmdf_dev.oil_obs.capability_registry r
LEFT JOIN mq_gmdf_dev.oil_obs.v_llm_bronze b ON b.capability = r.capability
WHERE b.capability IS NULL AND r.active = true;
```

**WAIT for user to provide:** is_generative, is_gxp_relevant, owner, silence_grace_hours for each unregistered capability.

**Then add to obs_setup_seed** (after existing INSERT):
```sql
INSERT INTO mq_gmdf_dev.oil_obs.capability_registry VALUES
-- User provides these values:
-- ('dsa_copilot_step', <is_generative>, <is_gxp_relevant>, <expected_min_daily>, 
--  <required_fields>, '<owner>', true, '<notes>', <silence_grace_hours>),
-- ('dsa_session_summary', ...), ('probe', ...), ('saa_insight', ...), ...
```

---

## Step 2 — New Findings Tables & INCIDENT_SOURCES Entries

### File: agent_obs_latency_detection.ipynb

**ADD NEW CELL** after cell-7 (capability_error_rate_alert):

```sql
-- capability_error_rate_findings — signature-keyed for obs_incidents MERGE.
-- Reuses capability_error_rate_alert which already applies the correct thresholds:
-- sum(total_calls) >= 5 AND error_rate_6h >= 0.8, aggregated over the 6h WINDOW total.
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

**ADD NEW CELL** after cell-0 (latency_anomalies):

```sql
-- latency_anomaly_findings — aggregated per (capability, model_config, verdict) for MERGE.
-- Keying on the call (id) made every poll a new incident; keying on the condition means a
-- persistently slow capability is ONE incident whose detection_count climbs.
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

### File: agent_obs_mal_output.ipynb

**MODIFY cell-4** (response_schema_drift) — add `finding_signature` to the final SELECT:

Add this column to the end of the SELECT list in the response_schema_drift CREATE:
```sql
    sha2(concat_ws('|',
        coalesce(coalesce(bk.capability, ck.capability), '<null>'),
        coalesce(coalesce(bk.field_name, ck.field_name), '<null>')
    ), 256) AS finding_signature,
```

**ADD NEW CELL** after cell-1 (blank_output_incidents):

```sql
-- blank_output_findings — aggregated over 6h window for MERGE dedup.
-- Key on (capability, model_config), NOT the hour — a sustained blank-output condition
-- is one incident. Currently verified as genuinely 0 rows; wired so it works when it fires.
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

### File: agent_obs_behavioral_correlation.ipynb

**ADD NEW CELL** after handover_delivery_rate:

```sql
-- handover_delivery_rate_findings — single-row when rate exceeds threshold.
-- Literal string key: one condition, one incident. detection_count carries persistence.
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.handover_delivery_rate_findings AS
SELECT
    failure_pct_7d, sent_ok, failed, last_attempt,
    'handover_delivery_rate_7d' AS finding_signature,
    current_timestamp() AS detected_at
FROM mq_gmdf_dev.oil_obs.handover_delivery_rate
WHERE failure_pct_7d > 20 AND (sent_ok + failed) >= 10;
```

---

## Step 2 continued — INCIDENT_SOURCES Changes in agent_obs_alert.ipynb

**Replace the entire INCIDENT_SOURCES list** in cell-3 with:

```python
INCIDENT_SOURCES = [
    # (detector, source_table, id_column, capability_column, severity, payload_columns, extra_WHERE)
    #
    # --- Latency ---
    # Widened to include anomaly_fixed_ceiling: a >30s call with no reliable baseline is still
    # worth an incident. Keyed on (capability, model_config, verdict) so a persistently slow
    # capability is one incident, not one per call.
    ("latency_anomaly", "latency_anomaly_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "latency_verdict", "worst_latency_ms", "baseline_p95_ms", "anomaly_count"],
     ""),

    # --- Capability error rate ---
    # HIGHEST PRIORITY new source. dsa_optimize failed 100% of 121 calls over 4 days and nothing
    # persisted it. Reuses capability_error_rate_alert's 6h window aggregation.
    ("capability_error_rate", "capability_error_rate_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "error_rate_6h", "calls_6h", "failures_6h", "hours_fully_failed"],
     ""),

    # --- Runtime configuration ---
    # Keyed on violation_signature (sha2 of the configuration tuple), not the call.
    ("runtime_violation", "transport_violation_signatures", "violation_signature",
     "capability", "CRITICAL",
     ["model_config", "transport", "scheduler_run", "violation_type", "violation_tier",
      "calls_in_window", "owner"], "WHERE violation_tier = 'immediate'"),
    ("runtime_violation_digest", "transport_violation_signatures", "violation_signature",
     "capability", "WARN",
     ["model_config", "transport", "scheduler_run", "violation_type", "violation_tier",
      "calls_in_window", "owner"], "WHERE violation_tier = 'digest'"),

    # --- Malformed output ---
    # blank_output: currently 0 rows and verified genuinely 0. Wired so it works when it fires.
    ("blank_output", "blank_output_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "blank_rate_window", "blank_count_window", "total_calls_window"],
     ""),
    # schema_field_missing: a field present in baseline that disappeared. field_added stays WARN.
    ("schema_field_missing", "response_schema_drift", "finding_signature",
     "capability", "CRITICAL",
     ["field_name", "drift_type", "baseline_presence_rate", "current_present"],
     "WHERE drift_type = 'field_missing'"),

    # --- Behavioral / delivery ---
    ("handover_delivery", "handover_delivery_failures", "ish_row_id",
     None, "CRITICAL",
     ["failure_reason", "shift_date", "batch_nbr"], ""),
    # Rate deterioration — distinct from per-failure check. See md §8.4.
    ("handover_delivery_rate", "handover_delivery_rate_findings", "finding_signature",
     None, "CRITICAL",
     ["failure_pct_7d", "sent_ok", "failed", "last_attempt"],
     ""),

    # --- Hallucination ---
    ("hallucination_high", "hallucination_signal", "row_id",
     "verified_capability", "CRITICAL",
     ["hallucination_risk", "ungrounded_token_count", "resp_vs_prompt_similarity"],
     "WHERE hallucination_risk = 'high'"),
]
```

---

## Step 2 continued — DETECTOR_META Additions

Add these entries to the DETECTOR_META dict in agent_obs_alert.ipynb:

```python
    "blank_output": {
        "label": "Capability returning empty or structurally void responses",
        "what": "More than 2% of successful calls returned a response that is null, empty, "
                "'{}', '[]', or missing the required how_we_ran field for summary capabilities.",
        "triage": [
            "Check is_blank_output definition in v_llm_bronze — the fourth test catches "
            "semantically blank JSON with missing how_we_ran",
            "Compare model_config: a config change is the usual trigger for shape changes",
            "Currently verified as genuinely 0 — any hit is a real regression, not noise",
        ],
    },
    "schema_field_missing": {
        "label": "Expected response field has disappeared",
        "what": "A JSON field present in ≥20% of 30-day baseline responses is now entirely "
                "absent from the last 24 hours. The downstream UI may show empty panels.",
        "triage": [
            "Check which capability lost the field — a model config change is the usual cause",
            "Compare response_parsed in v_llm_bronze for recent rows against the baseline",
            "field_added (WARN) is usually a deliberate rollout; field_missing is not",
        ],
    },
    "handover_delivery_rate": {
        "label": "Handover delivery failure rate is deteriorating",
        "what": "More than 20% of real send attempts (excluding config-gated rows) failed over "
                "7 days. Baseline is 9.3% since Jun 19 — this means it is getting worse.",
        "triage": [
            "Check handover_delivery_failures for the most recent failure reason",
            "getaddrinfo failure = DNS resolution to the mail host, not an app bug",
            "Compare against the per-failure incidents (handover_delivery) to see if "
            "acknowledged — the rate catches worsening, the per-failure catches recurrence",
        ],
    },
```

---

## Step 3 — threshold_basis Inserts (obs_setup_seed)

```sql
INSERT INTO mq_gmdf_dev.oil_obs.threshold_basis VALUES
('capability_error_rate', '>=0.8 error rate over 6h window with >=5 total calls',
 'dsa_optimize failed 153/153 over 4 days undetected; per-hour floor hid it because no single hour had >=5 calls. Window aggregation fixes this.',
 '2026-08-28', 'provisional'),
('latency_anomaly_fixed_ceiling', '>30000ms with is_reliable=false (no reliable baseline)',
 'Trades per-capability IQR precision for a working detection path under low volume. anomaly_fixed_ceiling means a successful call took >30s regardless of baseline quality.',
 '2026-08-28', 'provisional'),
('blank_output', '>2% blank rate AND >3 blank AND >=10 total calls over 6h',
 'Verified 0 rows historical as of Aug 28 — every unparseable row is success=false not blank. Any occurrence is novel.',
 '2026-08-28', 'not yet active'),
('schema_field_missing', 'field present in >=20% of 30d baseline AND 0 in current 24h window (>=10 rows)',
 'Frequency guard prevents single-row baseline presence from triggering drift. Current window requires >=10 rows.',
 '2026-08-28', 'provisional'),
('handover_delivery_rate', '>20% failure rate over 7d with >=10 real send attempts',
 'Baseline 9.3% since Jun 19. 20% threshold is approximately 2x baseline. Floor of 10 attempts prevents noise from ~2 attempts/day swinging the rate.',
 '2026-08-28', 'provisional');
```

---

## Step 4 — New Verification Assertions (agent_obs_verification.ipynb)

Add before the Summary cell:

```python
# --- New assertions for incident source completeness ---

try:
    # Every detector in INCIDENT_SOURCES has rows OR is documented as not-yet-active
    incident_detectors = {r["detector"] for r in 
        q(f"SELECT DISTINCT detector FROM {CAT}.obs_incidents")}
    not_yet_active = {r["check_name"] for r in
        q(f"SELECT check_name FROM {CAT}.threshold_basis WHERE status = 'not yet active'")}
    expected_sources = {'latency_anomaly', 'capability_error_rate', 'runtime_violation',
                        'runtime_violation_digest', 'blank_output', 'schema_field_missing',
                        'handover_delivery', 'handover_delivery_rate', 'hallucination_high'}
    missing = expected_sources - incident_detectors - not_yet_active
    record("incident sources: every detector has fired or is documented not-yet-active",
           len(missing) == 0, f"missing={sorted(missing)}" if missing else "all accounted for")
except Exception as e:
    record("incident source completeness", False, str(e)[:150])

try:
    # No per-call keying: all unresolved rows have length(source_row_id) = 64
    # Exceptions: handover_delivery (ish_row_id UUID=36), handover_delivery_rate (literal string)
    bad_keys = q(f"""
        SELECT detector, length(source_row_id) AS key_len, count(*) AS n
        FROM {CAT}.obs_incidents
        WHERE resolved_at IS NULL
          AND detector NOT IN ('handover_delivery', 'handover_delivery_rate')
          AND length(source_row_id) <> 64
        GROUP BY 1, 2
    """)
    record("incident keys: no per-call id (length=64 for sha2 detectors)",
           len(bad_keys) == 0,
           "; ".join(f"{r['detector']} has {r['n']} rows with key_len={r['key_len']}"
                     for r in bad_keys) if bad_keys else "all correct")
except Exception as e:
    record("incident key lengths", False, str(e)[:150])

try:
    dupes = q(f"""
        SELECT environment, capability, count(*) AS n
        FROM {CAT}.runtime_allowlist
        GROUP BY 1, 2 HAVING count(*) > 1
    """)
    record("runtime_allowlist: no duplicate (environment, capability)",
           len(dupes) == 0,
           "; ".join(f"{r['environment']}/{r['capability']}={r['n']}" for r in dupes)
           if dupes else "no duplicates")
except Exception as e:
    record("runtime_allowlist duplicates", False, str(e)[:150])

try:
    bad_disposition = q(f"""
        SELECT count(*) AS n
        FROM {CAT}.runtime_observed
        WHERE disposition = 'sanctioned' AND dispositioned_by IS NULL
    """)[0]["n"]
    record("runtime_observed: sanctioned rows have dispositioned_by",
           (bad_disposition or 0) == 0,
           f"{bad_disposition} row(s) sanctioned with no author")
except Exception as e:
    record("runtime_observed disposition integrity", False, str(e)[:150])

try:
    notified_warns = q(f"""
        SELECT count_if(notified_at IS NOT NULL) AS n
        FROM {CAT}.obs_incidents
        WHERE detector = 'runtime_violation_digest'
    """)[0]["n"]
    record("runtime_violation_digest: WARN never notified",
           (notified_warns or 0) == 0,
           f"{notified_warns} digest rows have notified_at set — WARN must never notify")
except Exception as e:
    record("digest notification integrity", False, str(e)[:150])
```

---

## Step 5 — Synthetic Teams Test

Run interactively AFTER all cells are deployed:

```python
# Insert synthetic incidents for each new detector
import json

CAT = "mq_gmdf_dev.oil_obs"
TEST_DETECTORS = [
    ("capability_error_rate", "dsa_optimize", 
     json.dumps({"model_config": "test-bot-claude-v1", "error_rate_6h": "1.0", 
                 "calls_6h": "12", "failures_6h": "12", "hours_fully_failed": "6"})),
    ("blank_output", "dsa_copilot",
     json.dumps({"model_config": "test-bot-claude-v1", "blank_rate_window": "0.15",
                 "blank_count_window": "5", "total_calls_window": "33"})),
    ("schema_field_missing", "summary",
     json.dumps({"field_name": "how_we_ran", "drift_type": "field_missing",
                 "baseline_presence_rate": "0.95", "current_present": "0"})),
    ("handover_delivery_rate", None,
     json.dumps({"failure_pct_7d": "25.0", "sent_ok": "9", "failed": "3",
                 "last_attempt": "2026-08-28T10:00:00"})),
    ("latency_anomaly", "dsa_compare",
     json.dumps({"model_config": "vertex26", "latency_verdict": "anomaly_fixed_ceiling",
                 "worst_latency_ms": "45000", "baseline_p95_ms": "null", "anomaly_count": "3"})),
]

for detector, cap, payload in TEST_DETECTORS:
    cap_val = f"'{cap}'" if cap else "NULL"
    spark.sql(f"""
        INSERT INTO {CAT}.obs_incidents
        (detector, source_row_id, capability, severity, first_detected, last_detected, 
         detection_count, signal_payload)
        VALUES ('{detector}', 'TEST-{detector}', {cap_val}, 'CRITICAL',
                current_timestamp(), current_timestamp(), 1, '{payload}')
    """)

# Point WEBHOOK at test channel before running:
# post_teams(to_notify_test, [])

# Cleanup:
# spark.sql(f"DELETE FROM {CAT}.obs_incidents WHERE source_row_id LIKE 'TEST-%'")
```

---

## Summary of All New Tables

| Table | Notebook | Key Pattern | Severity |
|-------|----------|-------------|----------|
| `capability_error_rate_findings` | latency_detection | sha2(capability\|model_config) | CRITICAL |
| `latency_anomaly_findings` | latency_detection | sha2(capability\|model_config\|verdict) | CRITICAL |
| `blank_output_findings` | mal_output | sha2(capability\|model_config) | CRITICAL |
| `handover_delivery_rate_findings` | behavioral_correlation | literal string key | CRITICAL |

Existing table modified:
| Table | Change |
|-------|--------|
| `response_schema_drift` | Added `finding_signature` column |

---

## What Requires User Input Before Proceeding

1. **Step 1c:** capability_registry values for unregistered capabilities (dsa_copilot_step, dsa_session_summary, probe, saa_insight, possibly dsa_ask, sev2_insight)
2. **Step 0b-2:** Whether to add the digest acknowledgement SQL to the weekly digest (proposal only)
3. **Step 3b:** Which capabilities get `is_gxp_relevant = true` (needed for hallucination_high to fire)
