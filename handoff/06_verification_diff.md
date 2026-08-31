# Notebook Diffs — agent_obs_verification.ipynb

## Changes

### INSERT NEW CELL before the Summary cell (before the `order = {"FAIL": 0, ...}` line)

Add as a new `%md` + code section:

```python
# %md ## Incident source completeness (new assertions)

# COMMAND ----------

# --- Incident source completeness ---
# Every detector in INCIDENT_SOURCES should have fired, or be documented as not-yet-active.
# Prevents silently dead incident sources from recurring unnoticed.

try:
    incident_detectors = {r["detector"] for r in
        q(f"SELECT DISTINCT detector FROM {CAT}.obs_incidents")}
    not_yet_active = {r["check_name"] for r in
        q(f"SELECT check_name FROM {CAT}.threshold_basis WHERE status = 'not yet active'")}
    expected_sources = {'latency_anomaly', 'capability_error_rate', 'runtime_violation',
                        'runtime_violation_digest', 'blank_output', 'schema_field_missing',
                        'handover_delivery', 'handover_delivery_rate', 'hallucination_high'}
    missing = expected_sources - incident_detectors - not_yet_active
    record("incident sources: every detector has fired or is documented not-yet-active",
           len(missing) == 0,
           f"missing={sorted(missing)}" if missing else "all accounted for")
except Exception as e:
    record("incident source completeness", False, str(e)[:150])

try:
    # No per-call keying remaining: all unresolved rows use sha2 signatures (length=64).
    # Exceptions: handover_delivery (ish_row_id = UUID, length 36),
    #             handover_delivery_rate (literal string 'handover_delivery_rate_7d', length 25)
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
                     for r in bad_keys) if bad_keys else "all sha2-keyed")
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

## Also update the existing assertion for registry grace hours

The current assertion expects `graced.get("dsa_copilot") == 18` but the actual value is 26 (raised after two false breaches). Update:

### BEFORE:
```python
    record("P1.3 grace hours on the two monitored capabilities",
           graced.get("dsa_copilot") == 18 and graced.get("dsa_optimize") == 2, f"graced={graced}")
```

### AFTER:
```python
    record("P1.3 grace hours on the two monitored capabilities",
           graced.get("dsa_copilot") == 26 and graced.get("dsa_optimize") == 2, f"graced={graced}")
```

(The grace was raised 18→26 after two false breaches at 19.5h and 22.7h with zero true ones — documented in §5.8 and the existing notebook markdown.)
