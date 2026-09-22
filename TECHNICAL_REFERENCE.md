# SAA/ISH Agent Observability — Technical Reference (concise)

High-level onboarding doc: what the pipeline is, how data flows, what each notebook and detector
does. No design history or change log — see `HANDOFF_LIVENESS_COMPLETENESS.md` for the latest
in-flight changes and `git log` for history.

## 1. Architecture

```
mq_gmdf_dp_prd.oil (prod, read-only)
  ptof_primary__ai_shift_outputs   ptof_ish_audit   ptof_etl_pipeline_audit
        |                                |                    |
        v                                v                    v
mq_gmdf_dev.oil_obs (dev, VIEWs — not copies, same history as prod)
  v_llm_bronze                    v_ish_bronze          v_etl_bronze
        |                                |                    |
        +---------------+----------------+--------------------+
                         v
        parallel detector notebooks -> 11 findings tables
                         v
        ptof_obs_alert.ipynb:
          - 5 scalar checks (Python)
          - MERGE 11 tables + 5 scalars into obs_incidents
          - auto-resolve, notify/digest, Teams post, raise-on-broken
                         v
                  obs_incidents (Delta table) -> Teams Adaptive Card
```

Reference/baseline data is produced by a separate nightly job, not the fresh-scan job:

```
ptof_obs_nightly_baseline.ipynb (nightly)
  v_llm_bronze -> response_field_baseline   (feeds schema_field_missing)
  v_etl_bronze -> etl_duration_baseline     (feeds etl_run_slow)

ptof_obs_setup_seed.ipynb (manual, run by hand)
  capability_registry, threshold_basis, obs_incidents DDL
```

Prod grants are SELECT/USE/BROWSE only. All writes target `mq_gmdf_dev.oil_obs`. The bronze
objects are views, so querying them is querying prod history directly.

## 2. Notebooks

| Notebook | Role |
|---|---|
| `ptof_obs_bronze_projection` | Builds the 3 bronze views (aliases + derived flags) from prod |
| `ptof_obs_liveness_detection` | 8 detectors: silence, staleness, slow-run, row-count checks |
| `ptof_obs_mal_output` | 2 detectors: blank outputs, schema-field drift |
| `ptof_obs_behavioral_correlation` | 2 detectors: handover delivery failure/rate |
| `ptof_obs_alert` | Persists all findings to `obs_incidents`, notifies Teams |
| `ptof_obs_nightly_baseline` | Nightly reference tables (field presence rate, ETL duration) |
| `ptof_obs_setup_seed` | One-time/manual: registry, evidence ledger, table DDL |

## 3. Bronze layer (what every detector reads)

- **`v_llm_bronze`** — `ptof_primary__ai_shift_outputs`. Renames `output_type`→`capability`,
  `generated_at`→`called_at`, `content`→`response_parsed`; adds `is_blank_output` (empty/null JSON,
  or for `summary` capability, blank `how_we_ran`).
- **`v_ish_bronze`** — `ptof_ish_audit`. Parses `before_json`/`after_json`, normalizes
  `shift_date`/`shift_type`/`batch_nbr`, adds `is_email_disabled_gate` (distinguishes "chose not to
  send" from "tried and failed").
- **`v_etl_bronze`** — `ptof_etl_pipeline_audit`, pass-through of `run_id`, `run_timestamp`,
  `table_or_view`, `task_name`, `status`, `rows_written`, `duration_seconds`, `error_message`.

## 4. Detectors

All 16 active detectors are CRITICAL. Two shapes: **table-backed** (a detector notebook writes a
findings table; alert notebook MERGEs it) and **scalar** (a Python `check()` in the alert notebook
itself yields one global yes/no).

### Table-backed (written in a detector notebook, MERGEd by `ptof_obs_alert` cell 5)

| Detector | Fires when | Notify |
|---|---|---|
| `handover_delivery` | A handover email `sent=false`, not gated by config | Per-incident |
| `handover_delivery_rate` | 7d failure rate >20% (n≥10) OR ≥3 absolute failures | Per-incident |
| `shift_context_missing` | Active capability has blank shift_type/batch_nbr/null shift_date | Per-incident |
| `blank_output` | ≥1 blank output for a capability/model_config in trailing 6h | Per-incident |
| `schema_field_missing` | A field present in ≥20% of 30d baseline is absent from last 24h | Per-incident |
| `capability_silence` | No call in 7d, or gap > `silence_grace_hours` | Per-incident |
| `capability_silence_ceiling` | Gap > `silence_ceiling_hours` (covers capabilities with no grace set) | Per-incident |
| `capability_liveness_unconfigured` | Active capability has both grace and ceiling NULL (config gap, not a live event) | **Not yet wired into `ptof_obs_alert`** |
| `etl_pipeline_failure` | Any ETL run with `status='failure'` in 24h | Per-incident |
| `etl_table_staleness` | A table's last run >60 min old | Digest (24h) |
| `etl_run_slow` | Run duration exceeds its own (table, task) MAD-based bound, ≥3x/hour | Digest (24h) |
| `etl_row_count_anomaly` | A table's zero/nonzero write pattern crosses its established regime | Digest (24h) |

### Scalar (Python `check()` in `ptof_obs_alert` cell 4)

| Detector | Fires when |
|---|---|
| `pipeline_heartbeat` | Zero rows in `v_llm_bronze` in trailing 25 minutes (total outage) |
| `etl_pipeline_staleness` | Fleet-wide `max(run_timestamp)` in `v_etl_bronze` >30 min old |
| `nightly_baseline_staleness` | `response_field_baseline` hasn't refreshed in 36h |
| `unacknowledged_critical` | Any unacknowledged, unresolved CRITICAL incident (excludes currently-digesting detectors, ≥1h old) |
| `long_running_incident` | Any unresolved, unacknowledged incident open ≥3 days |

## 5. Incident lifecycle

Each finding becomes one row in `obs_incidents`, keyed on `(detector, source_row_id)`:

1. **First match** → INSERT: `first_detected = last_detected = now`, `detection_count = 1`.
2. **Still matching** → UPDATE: `last_detected`, `detection_count += 1`, payload refreshed;
   `resolved_at`/`acknowledged_at`/`acknowledged_by` reset to NULL.
3. **Notify**: CRITICAL, not currently digest-routed, unacknowledged, and
   `notified_at` is NULL or >24h old → post to Teams, stamp `notified_at`.
4. **Digest** (for detectors currently fanning out — see below): bundled into one post per
   detector per 24h instead of paging per incident.
5. **Acknowledge**: a human runs the card's SQL, setting `acknowledged_by`/`acknowledged_at`. This
   suppresses the next Teams post but is cleared on the next re-match (step 2) — persistent
   suppression comes from the 24h `notified_at` cooldown, not the acknowledgement itself.
6. **Auto-resolve**: a detector that ran cleanly and didn't touch a row this cycle gets
   `resolved_at = now` on that row.

## 6. Fan-out digest routing

Three ETL detectors (`etl_run_slow`, `etl_table_staleness`, `etl_row_count_anomaly`) can generate
many simultaneous incidents from one real event. Routing to a once-per-24h digest (instead of
paging per incident) is dynamic: a detector is digest-routed for a given cycle if it has more than
5 open incidents whose `first_detected` values fall within a 15-minute window. Otherwise it
notifies per-incident like everything else. The backstop `unacknowledged_critical` check excludes
whatever is currently digest-routed, so a digested burst doesn't double-page.

## 7. Nightly baseline layer

- **`response_field_baseline`** — per capability/field, presence rate over a 30-day window
  (floor: ≥20 rows/capability). Feeds `schema_field_missing`.
- **`etl_duration_baseline`** — per (table_or_view, task_name), median + MAD of
  `duration_seconds` over 30 days (floor: ≥50 rows/stratum), with `upper_bound_s` floored at
  `median * 1.5` to prevent a near-zero-variance stratum from flagging trivial overages. Feeds
  `etl_run_slow`.

## 8. Supporting tables

- **`capability_registry`** — human-curated: which capabilities are active, their owner,
  `silence_grace_hours` / `silence_ceiling_hours` thresholds.
- **`threshold_basis`** — evidence ledger recording why each threshold was set. Documentation
  only; no detector reads it at runtime.
- **`obs_incidents`** — the single incident table every detector writes to and every Teams card
  reads from.
