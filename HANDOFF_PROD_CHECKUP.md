# Handoff: Production Checkup — Read This First

> **Start a fresh context window, paste this file's contents as your first message.**
> Today's date is 2026-09-15. The prod source migration (9 steps) is **complete and deployed**.
> This handoff is about a **production health checkup** and one design gap found during review.

## Your role
You are the operator of an SAA/ISH agent observability pipeline on Databricks. The pipeline
monitors an LLM agent's outputs and delivery, persists findings to `obs_incidents`, and posts
triage Adaptive Cards to a Teams webhook. The prod source migration finished on 2026-09-14.
A checkup on 2026-09-15 confirmed the pipeline is running and healthy. **Your job: fix the one
real gap found during the checkup (below), then keep watch.**

## The user's original question (already answered — for context)
> "I haven't seen any new alerts pushed to Teams. Is it working as expected?"

**Answer: yes, it is working correctly. No alerts = healthy prod data.** Verified on 2026-09-15:

| Check | Result |
|---|---|
| `obs_fresh_scan` job runs | Every ~5–10 min, all **SUCCESS** (2916 runs; one RUNNING at checkup time) |
| Workspace notebook version | **Prod-migrated** (`mq_gmdf_dp_prd`, `etl_pipeline_failure`, `etl_pipeline_staleness` present; all 5 dropped dev detectors absent) |
| Prod data freshness | `v_llm_bronze`: 48 rows in last 2h, latest 15:17 UTC, all 4 capabilities generating |
| `blank_output_findings` | **0** (prod content is 100% valid JSON, zero blanks) |
| `response_schema_drift` | **0** (stable schemas) |
| `handover_delivery_failures` | **0** (all 28 emails `sent=true`) |
| `handover_delivery_rate` | **1 row**: failure_pct_7d=0.0, sent_ok=13, failed=0 (threshold is >20% & ≥10 attempts) |
| `etl_pipeline_health` | **0** failures in 24h window |
| `capability_silence` | **0** (all within grace windows) |
| `shift_context_missing` | **0** |
| `obs_incidents` | **0 rows** (empty — no incidents persisted because no detector found anything) |
| `response_field_baseline` | 29 rows (nightly baseline OK across 4 capabilities) |
| `capability_registry` | 14 rows (4 active prod + deactivated dev) |
| Teams webhook | `teams-webhook` secret in scope `obs-alerting` — valid Azure Logic Apps URL, readable |
| Broken (UNAVAILABLE) detectors | **None** (that's why all runs are SUCCESS, not just the data being clean) |

The notification logic (cell 10 of `ptof_obs_alert.ipynb`) only posts to Teams when there is a
CRITICAL incident in `obs_incidents` OR a broken (UNAVAILABLE) detector. Neither exists right now,
so the pipeline is correctly **silent**. This is the intended "silent when healthy" design —
NOT a broken webhook or dead job.

## ⚠️ THE ONE REAL GAP TO FIX (this is your actual task)

**Two CRITICAL detectors can detect a problem but cannot alert on it.** They are not wired into
the incident-persistence path, so even when they fire they only print to the job log — no Teams
card, no `obs_incidents` row, no task failure.

In `ptof_obs_alert.ipynb`, incidents are persisted by the `INCIDENT_SOURCES` list (cell 4) which
MERGEs findings into `obs_incidents`, then notified in cell 10 which posts for anything in
`obs_incidents` (unacknowledged, unresolved, not-notified-in-24h) plus any `UNAVAILABLE` detector.
The `check()` calls in cell 6 produce a `criticals` list, but **`criticals` is only used for a
final print statement — it is never passed to `post_teams`** and never raises.

`INCIDENT_SOURCES` persists exactly 5 detectors: `handover_delivery`, `blank_output`,
`schema_field_missing`, `handover_delivery_rate`, `etl_pipeline_failure`.

But these CRITICAL `check()`s are **NOT** in `INCIDENT_SOURCES` → never persisted → never notified:

1. **`pipeline_heartbeat`** (CRITICAL) — "zero outputs from any capability in 2h." This is the
   backstop for a **total upstream outage**. If it fires, nothing alerts. The job stays SUCCESS.
2. **`etl_pipeline_staleness`** (CRITICAL) — "no ETL run completed in 30 min, agent on stale data."
   Only detector that catches a silent stale-data condition (outputs still arrive, so
   `capability_silence`/`pipeline_heartbeat` won't fire; `etl_pipeline_failure` needs an actual
   failed run, not "no runs"). If it fires, nothing alerts.

(`handover_delivery_new_failure` and `unacknowledged_critical` are also non-persisted CRITICALs,
but they are rollups — the underlying `handover_delivery` failures ARE persisted and notified, so
those two are fine as log-only backstops. Only heartbeat + staleness are true blind spots.)

### Why it matters
These two detectors are the worst-case backstops — total outage and stale data. They were
explicitly built to catch the conditions "every other detector implicitly assumes cannot happen."
Right now they can catch them in the job log but **cannot page anyone**. That defeats the purpose.

### Recommended fix (present options to the user, do not assume)
The `INCIDENT_SOURCES` mechanism needs `FROM {CAT}.{table} {extra}` — a findings table with an
id column. Heartbeat and staleness are scalar checks with no findings table. Two approaches:

- **(A) Smallest change — persist scalars directly in the alert notebook.** After the
  `INCIDENT_SOURCES` loop (cell 4), add a small block that MERGEs two synthetic rows into
  `obs_incidents` using constant `source_row_id` values (e.g. `'pipeline_heartbeat_global'`,
  `'etl_staleness_global'`) when the cell-6 checks come back CRITICAL. Reuse the existing
  `notified_at`/`acknowledged_at`/`resolved_at` lifecycle so the 24h re-notify and auto-resolve
  still apply. This keeps the "silent when healthy" behavior and routes these through Teams.
- **(B) Consistent with architecture — materialize findings tables.** In
  `ptof_obs_liveness_detection.ipynb`, add `pipeline_heartbeat_findings` and
  `etl_staleness_findings` tables (CREATE OR REPLACE, one row when the condition holds), then add
  both to `INCIDENT_SOURCES` like `handover_delivery_rate` uses a constant id column. More moving
  parts, but the alert notebook stays uniform.

Either way, the auto-resolve in cell 5 and the notify/re-notify in cell 10 then work for free
because the incidents are real rows in `obs_incidents`.

**Do NOT** fix this by making heartbeat/staleness `raise` — that contradicts the deliberate
severity semantics ("raise fires ONLY on UNAVAILABLE; task status means *is monitoring working*,
not *did it find something*"). CRITICALs are meant to route to Teams, not fail the task.

**Before editing:** re-read `ptof_obs_alert.ipynb` cells 4, 6, 10, 11 yourself — confirm the
`criticals`-not-passed-to-`post_teams` claim is still true against the current notebook (the
local repo copy matches the deployed workspace copy as of 2026-09-15, but verify before editing).

## End-to-end webhook test (optional, to fully close the user's concern)
To prove the webhook path end-to-end (not just that the secret is readable), insert one synthetic
CRITICAL incident and run the alert notebook once, then acknowledge+resolve it:
```sql
INSERT INTO mq_gmdf_dev.oil_obs.obs_incidents
  (detector, source_row_id, capability, severity, first_detected, last_detected, detection_count, signal_payload)
VALUES
  ('webhook_smoke_test', 'smoke_2026_09_15', NULL, 'CRITICAL',
   current_timestamp(), current_timestamp(), 1,
   to_json(map('note', 'end-to-end Teams webhook smoke test')));
-- run the alert notebook (06_alert task) once -> expect a Teams card
-- then clean up:
-- DELETE FROM mq_gmdf_dev.oil_obs.obs_incidents WHERE detector='webhook_smoke_test';
```
Ask the user to confirm the card arrived in Teams before deleting the test row.

## Environment
- **Dev workspace profile:** `"Tyler Kei"` — use `-p "Tyler Kei"` on every CLI command
- **Catalog/schema (all writes):** `mq_gmdf_dev.oil_obs`
- **Prod source (read-only):** `mq_gmdf_dp_prd.oil` (read via views `v_llm_bronze`, `v_ish_bronze`, `v_etl_bronze`)
- **SQL warehouse (dev):** `cd6c1145a46bec44`
- **Secret scope:** `obs-alerting`, key `teams-webhook` (Azure Logic Apps / Teams Workflows URL)
- **Never auto-select a profile. Always pass `--profile` / `-p`.**

## How to run SQL from the CLI (this session's method — reuse it)
There is no `databricks sql` command in this CLI (v1.14.1). Use the Statement Execution API via
the `databricks api` passthrough. A working helper lives at `/tmp/sql_exec.py`:
```bash
printf '%s' "SELECT ... " | python3 /tmp/sql_exec.py
```
It POSTs to `/api/2.0/sql/statements/` with `wait_timeout=50s`, `format=JSON_ARRAY`,
`disposition=INLINE`, warehouse `cd6c1145a46bec44`, profile `Tyler Kei`, and prints a TSV. If
`/tmp/sql_exec.py` is gone, recreate it: read SQL from stdin, build the JSON body, shell out to
`databricks api post /api/2.0/sql/statements/ --json <body> -p "Tyler Kei" -o json`, poll/inline
the result, print columns + `data_array` rows. (Note: `databricks jobs list-runs` uses `-o json`,
not `--output JSON`; `get-run` takes the run_id positionally; secret values come back base64-encoded.)

## The two jobs
- **`obs_fresh_scan`** (job `585607309385820`) — the alerting pipeline. Tasks in order:
  `01_bronze_projections` → (`02_liveness_detection` ∥ `03_malformed_output` ∥ `05_behavioral_correlation`) → `06_alert`.
  Runs every ~5–10 min (trigger is set on the job though `settings.schedule` shows NONE — check
  `settings.trigger` if you need the exact cadence). All tasks SUCCESS = healthy.
- **`obs_nightly_baseline`** (job `428356310089497`) — runs `ptof_obs_nightly_baseline` at 02:00
  America/Indianapolis daily. Computes `response_field_baseline` (29 fields).

## Architecture (condensed — replaces the old migration handoff)
```
mq_gmdf_dp_prd.oil (READ)                mq_gmdf_dev.oil_obs (WRITE)
ai_shift_outputs ─► v_llm_bronze ─┬─► blank_output_findings
  (4 output_types)                ├─► response_schema_drift
                                  ├─► capability_silence
                                  └─► shift_context_missing
ptof_ish_audit ────► v_ish_bronze ┬─► handover_delivery_failures
                                  └─► handover_delivery_rate[_findings]
ptof_etl_pipeline_audit ► v_etl_bronze ─► etl_pipeline_health
                                            ▼
                              obs_incidents ─► Teams Adaptive Card (06_alert)
```
**8 active detectors:** handover delivery failures (CRIT), handover delivery rate (CRIT),
shift context missing (WARN), blank output (CRIT), schema drift/field missing (CRIT),
capability silence (WARN), pipeline heartbeat (CRIT), ETL pipeline health (CRIT).
**4 prod capabilities:** saa-display (2h grace), situational-awareness (2h grace),
sev2-insights (NULL grace — irregular cadence), summary (36h grace). All `is_groundable=false`.
**7 dropped detectors** (won't come back): latency, error rate, hallucination, transport
violations, prompt size drift, credential fastfail, rapid human correction.

## Critical design decisions (do NOT revisit)
- capability_registry inner-join scoping is intentional — only registered capabilities monitored.
- `is_groundable=false` for all prod capabilities — hallucination detection dropped (4.9% FP rate).
- sev2-insights gets `silence_grace_hours=NULL` — natural 100h+ gaps make silence meaningless.
- `raise` fires ONLY on UNAVAILABLE (broken detector). CRITICALs route to Teams + obs_incidents.
- WARN findings are NOT notified (shift_context_missing, capability_silence are standing gaps).
- All writes stay in `mq_gmdf_dev.oil_obs`; prod catalog `mq_gmdf_dp_prd` is read-only.

## Notebook files (repo root, /Users/L141230/Downloads/agent_obs)
`ptof_obs_setup_seed.ipynb`, `ptof_obs_bronze_projection.ipynb`, `ptof_obs_liveness_detection.ipynb`,
`ptof_obs_mal_output.ipynb`, `ptof_obs_behavioral_correlation.ipynb`, `ptof_obs_nightly_baseline.ipynb`,
`ptof_obs_alert.ipynb`. Deployed copies live in the workspace at
`/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`. **After editing locally,
sync to the workspace** (the job runs the workspace copy, not local disk — verify they match).

## Constraints
- Do NOT modify the prod catalog (read-only).
- Do NOT add unregistered-capability alerting (scope decision — see [[agent-obs-capability-scoping]]).
- No retired/unused code in notebooks — delete cells, don't comment them out.
- The user offered to run long queries manually and paste output if CLI is too slow — the
  Statement Execution API path above has been fast (seconds) so far, so it likely isn't needed,
  but take them up on it if a query times out.
