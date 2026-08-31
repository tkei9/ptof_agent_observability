# Notebook Diffs — agent_obs_alert.ipynb

This is the largest change. The notebook is a single mega-cell in the .ipynb export.
Changes affect specific sections within that cell.

## Change 1: Replace INCIDENT_SOURCES (in cell-0, after the `check()` function definitions)

### BEFORE (lines starting with `INCIDENT_SOURCES = [`):

```python
INCIDENT_SOURCES = [
    # detector, source table, id column, capability column, severity, payload columns, extra WHERE
    # latency: only 'anomaly' is CRITICAL. baseline_unreliable and no_baseline are WARN-level
    # ...
    ("latency_anomaly",    "latency_anomalies",          "id",
     "capability",          "CRITICAL",
     ["latency_ms", "latency_verdict", "p95_ms"], "WHERE latency_verdict = 'anomaly'"),
    ...
    ("hallucination_high", "hallucination_signal",       "row_id",
     "verified_capability", "CRITICAL",
     ["hallucination_risk", "ungrounded_token_count", "resp_vs_prompt_similarity"],
     "WHERE hallucination_risk = 'high'"),
]
```

### AFTER:

```python
INCIDENT_SOURCES = [
    # (detector, source_table, id_column, capability_column, severity, payload_columns, extra_WHERE)
    #
    # --- Latency ---
    # Widened to include anomaly_fixed_ceiling: a >30s call with no reliable baseline is still
    # worth an incident. Keyed on (capability, model_config, verdict) via finding_signature so
    # a persistently slow capability is one incident, not one per call.
    ("latency_anomaly", "latency_anomaly_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "latency_verdict", "worst_latency_ms", "baseline_p95_ms", "anomaly_count"],
     ""),

    # --- Capability error rate ---
    # HIGHEST PRIORITY new source. dsa_optimize failed 100% of 121 calls over 4 days and nothing
    # persisted it. Reuses capability_error_rate_alert's 6h window aggregation with the volume
    # floor applied to the window total (not per-hour), which is what fixed the suppression.
    ("capability_error_rate", "capability_error_rate_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "error_rate_6h", "calls_6h", "failures_6h", "hours_fully_failed"],
     ""),

    # --- Runtime configuration ---
    # Keyed on violation_signature (sha2 of the config tuple), not b.id.
    # Source is transport_violation_signatures (one row per distinct config) because MERGE
    # requires unique source keys.
    ("runtime_violation", "transport_violation_signatures", "violation_signature",
     "capability", "CRITICAL",
     ["model_config", "transport", "scheduler_run", "violation_type", "violation_tier",
      "calls_in_window", "owner"], "WHERE violation_tier = 'immediate'"),
    ("runtime_violation_digest", "transport_violation_signatures", "violation_signature",
     "capability", "WARN",
     ["model_config", "transport", "scheduler_run", "violation_type", "violation_tier",
      "calls_in_window", "owner"], "WHERE violation_tier = 'digest'"),
    # WARN persists to obs_incidents but is never notified (§9.4 filters severity='CRITICAL'),
    # which is exactly the digest queue. obs_weekly_runtime_digest reads it.

    # --- Malformed output ---
    # blank_output: currently 0 rows and verified genuinely 0. Wired so it works when it fires.
    ("blank_output", "blank_output_findings", "finding_signature",
     "capability", "CRITICAL",
     ["model_config", "blank_rate_window", "blank_count_window", "total_calls_window"],
     ""),
    # schema_field_missing: a field present in >=20% of baseline that disappeared entirely.
    # field_added stays WARN and does NOT become an incident source.
    ("schema_field_missing", "response_schema_drift", "finding_signature",
     "capability", "CRITICAL",
     ["field_name", "drift_type", "baseline_presence_rate", "current_present"],
     "WHERE drift_type = 'field_missing'"),

    # --- Behavioral / delivery ---
    ("handover_delivery", "handover_delivery_failures", "ish_row_id",
     None, "CRITICAL",
     ["failure_reason", "shift_date", "batch_nbr"], ""),
    # Rate deterioration signal — distinct from the per-failure check. The rate catches
    # worsening; handover_delivery catches any single unacknowledged recurrence. Both needed.
    ("handover_delivery_rate", "handover_delivery_rate_findings", "finding_signature",
     None, "CRITICAL",
     ["failure_pct_7d", "sent_ok", "failed", "last_attempt"],
     ""),

    # --- Hallucination ---
    # Cannot fire until a summarisation capability with is_gxp_relevant=true has active traffic.
    # Currently blocked on registry reconciliation (Step 1) + user decision on GxP flags.
    ("hallucination_high", "hallucination_signal", "row_id",
     "verified_capability", "CRITICAL",
     ["hallucination_risk", "ungrounded_token_count", "resp_vs_prompt_similarity"],
     "WHERE hallucination_risk = 'high'"),
]
```

## Change 2: Add new DETECTOR_META entries

### INSERT these entries into the DETECTOR_META dict (after the existing `"capability_error_rate_sustained"` entry):

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

## Change 3: Update the existing latency_anomaly CHECK

The `check("latency_anomaly", ...)` call currently only fires on `is_reliable = true` and the existing INCIDENT_SOURCES version. Since we changed the INCIDENT_SOURCES to use the findings table, the CHECK should reflect what incidents will actually appear:

### BEFORE:
```python
check(
    "latency_anomaly",
    f"""
    SELECT count(*) AS n, max(la.latency_ms) AS worst_ms,
           concat_ws(',', collect_set(la.capability)) AS capabilities
    FROM {CAT}.latency_anomalies la
    JOIN {CAT}.capability_latency_baseline base USING (capability)
    WHERE la.latency_verdict = 'anomaly'
      AND base.is_reliable = true
      AND la.called_at >= current_timestamp() - INTERVAL {LOOKBACK} MINUTES
    """,
    gt0,
)
```

### AFTER:
```python
# Reliable-baseline anomalies AND fixed-ceiling anomalies (>30s with no reliable baseline).
# The incident source uses latency_anomaly_findings which includes both verdicts.
check(
    "latency_anomaly",
    f"""
    SELECT count(*) AS n, max(la.worst_latency_ms) AS worst_ms,
           concat_ws(',', collect_set(la.capability)) AS capabilities
    FROM {CAT}.latency_anomaly_findings la
    """,
    gt0,
)
```

**Keep** the existing `latency_anomaly_unreliable_baseline` check as-is (it's informational WARN).

## Change 4: Remove the now-redundant capability_error_rate_sustained check

Since `capability_error_rate_findings` reuses `capability_error_rate_alert` and is now an incident source, the separate `capability_error_rate_sustained` check becomes the same signal. 

### OPTION A (recommended): Merge them — keep one check that reports both signals:

Replace:
```python
check(
    "capability_error_rate_sustained",
    f"""
    SELECT count(*) AS n,
           concat_ws(', ', collect_set(concat(capability,'/',model_config,'=',
                      cast(error_rate_6h AS STRING),' over ',
                      cast(calls_6h AS STRING),' calls'))) AS offenders
    FROM {CAT}.capability_error_rate_alert
    """,
    gt0,
)
```

With:
```python
# Now an incident source (capability_error_rate). The check stays for task-output visibility
# but no longer needs a separate "sustained" name — both the per-hour and the 6h-window
# signal feed the same incident via capability_error_rate_findings.
check(
    "capability_error_rate_sustained",
    f"""
    SELECT count(*) AS n,
           concat_ws(', ', collect_set(concat(capability,'/',model_config,'=',
                      cast(error_rate_6h AS STRING),' over ',
                      cast(calls_6h AS STRING),' calls'))) AS offenders
    FROM {CAT}.capability_error_rate_findings
    """,
    gt0,
)
```

This way the check name stays stable but reads from the findings table (same data, just confirms the findings table populated).
