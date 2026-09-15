# Handoff: obs_incidents auto-resolve fix — Read This First

> **Start a fresh context window, paste this file's contents as your first message.**
> Today's date is 2026-09-15. This continues directly from HANDOFF_PROD_CHECKUP.md (prod
> migration complete 2026-09-14, checkup done 2026-09-15). That file's original gap is now
> fixed and deployed. A second, deeper gap was found while verifying the fix and is also
> fixed and deployed. **One related design question is still open — see "Open question" below.**

## Your role
Operator of the SAA/ISH agent observability pipeline on Databricks (`ptof_obs_alert.ipynb` +
friends). Monitors an LLM agent's outputs/delivery, persists findings to `obs_incidents`,
posts triage Adaptive Cards to Teams.

## What was done this session (in order)

### 1. Original gap (from HANDOFF_PROD_CHECKUP.md) — FIXED
`pipeline_heartbeat` and `etl_pipeline_staleness` (both CRITICAL) computed a result in
`ptof_obs_alert.ipynb` but were never in `INCIDENT_SOURCES`, so they could never reach
`obs_incidents` or Teams — only the job log. Chose approach (A) from that handoff (smallest
change, avoids duplicating threshold logic in two notebooks):
- Moved both `check()` calls earlier (now cell 4, before persistence).
- Added `SCALAR_INCIDENT_SOURCES` list + a MERGE loop (in cell 5, after the existing
  `INCIDENT_SOURCES` loop) that persists a synthetic constant-id row
  (`pipeline_heartbeat_global`, `etl_staleness_global`) only when that run's check came back
  CRITICAL — skipping the MERGE otherwise has the same effect as "no matching rows" for a
  table-backed detector, so existing auto-resolve logic clears it for free.
- Extended auto-resolve's `_active_detectors` (cell 6) to include the two scalar detectors.
- Added `DETECTOR_META` (cell 9) and `BACKTRACK` (cell 5) entries for both, so their Teams
  cards render a real label/explanation/triage steps instead of the bare detector name.

### 2. Verification of #1 — surfaced a second, bigger bug
- Deployed to workspace, confirmed byte-identical cell source to local.
- Triggered a live `obs_fresh_scan` run (job `585607309385820`) — all tasks SUCCESS.
- Both real conditions are currently healthy (47 rows in `v_llm_bronze` last 2h; ETL run 7 min
  old), so 0 rows in `obs_incidents` for these two detectors — correct, not a false negative.
- Inserted a synthetic CRITICAL row to test lifecycle → it got silently auto-resolved before
  ever reaching notify, because it contradicted live (healthy) data. This is the auto-resolve
  safety net working as designed — cleaned up the test row.
- Ran a scratch notebook (uploaded, executed, deleted — never touched `obs_incidents`) that
  calls the real `post_teams()` with fabricated incidents for both new detectors. Teams
  returned `202`. **User has not yet confirmed the `[SMOKE TEST -- IGNORE]` card visually
  arrived/read correctly in Teams — worth asking if not already confirmed.**

### 3. Root-cause bug found: `resolved_at` never cleared on re-detection — FIXED
User asked whether auto-resolve could ever silently quiet a real CRITICAL. Tracing the MERGE
in `ptof_obs_alert.ipynb` cell 5: `WHEN MATCHED` updated `severity`/`last_detected`/
`detection_count`/`signal_payload` but **never touched `resolved_at`**. Auto-resolve (cell 6)
only sets `resolved_at`, never clears it. Net effect: for any detector whose `source_row_id`
is **stable/time-invariant** (not unique per occurrence), the first time an incident is
auto-resolved and the same condition recurs later, the MERGE hits `WHEN MATCHED` on the same
`(detector, source_row_id)`, refreshes the row, but `resolved_at` stays frozen from the earlier
resolution — permanently. The notify query, `unacknowledged_critical`, and auto-resolve itself
all filter `resolved_at IS NULL`, so that detector goes dark forever with no error.

Audited all 7 persisted CRITICAL detectors by how their key is built (grepped the `CREATE OR
REPLACE TABLE` definitions in `ptof_obs_mal_output.ipynb`, `ptof_obs_liveness_detection.ipynb`,
`ptof_obs_behavioral_correlation.ipynb`):

| Detector | Key | Affected? |
|---|---|---|
| `handover_delivery` | `ish_row_id` (per specific audit row) | Safe — new failure = new id |
| `blank_output` | `sha2(capability \| model_config)`, no time component | **Affected** |
| `schema_field_missing` | `sha2(capability \| field_name)`, no time component | **Affected** |
| `handover_delivery_rate` | literal constant `'handover_delivery_rate_7d'` | **Affected** (worst case) |
| `etl_pipeline_failure` | `sha2(table_or_view \| run_id)`, `run_id` unique per run | Safe — new run = new id |
| `pipeline_heartbeat` | constant `'pipeline_heartbeat_global'` | **Affected** (this session's addition) |
| `etl_pipeline_staleness` | constant `'etl_staleness_global'` | **Affected** (this session's addition) |

**5 of 7 CRITICAL detectors were exposed** — not an edge case. Checked live data:
`handover_delivery_rate` has zero rows in `obs_incidents` ever (never fired since migration),
so this hasn't bitten anyone yet, but it was a live landmine.

**Fix applied** (one change, fixes all 5 at once since they share the MERGE template): added
`t.resolved_at = NULL` to `WHEN MATCHED` in **both** MERGE loops in cell 5 (the
`INCIDENT_SOURCES` table-backed loop and the `SCALAR_INCIDENT_SOURCES` loop) — a match means
"detected again this run," which should never coexist with a non-null `resolved_at`. Updated
cell 3's markdown to document why.

**Deployed and verified**: workspace copy re-synced, confirmed byte-identical to local, and
confirmed the string `resolved_at     = NULL` appears exactly twice in the deployed notebook
(once per MERGE loop).

**NOT yet done**: have not re-triggered a live `obs_fresh_scan` job run since this fix (the
last live run, `29938232059412`, predates it). Should do a sanity run to confirm no regression,
though the change is additive/low-risk (adding one column to an UPDATE SET clause) and matches
the existing proven MERGE pattern exactly.

## Open question — not yet resolved, needs user decision
There is a **structurally identical, adjacent gap** in `acknowledged_at`: it is documented as
"hand-set only" (cell 3 markdown, deliberate design choice) and is never cleared automatically
anywhere. So: if a human acknowledges an incident, it later resolves, and the *same*
stable-key condition recurs afterward, the notify query
(`WHERE ... AND acknowledged_at IS NULL AND resolved_at IS NULL ...`) will still exclude it
forever — same silent-dark-forever failure mode as the `resolved_at` bug, just gated on
`acknowledged_at` instead. This was flagged as a related finding but **intentionally not
changed**, since "hand-set only" was called out as an explicit design decision in the
notebook's own docs, and unilaterally reversing that wasn't requested. **Ask the user whether
they want `acknowledged_at` cleared on re-detection too** (mirroring the `resolved_at` fix), or
whether "once acknowledged, always suppressed until a human re-clears it" is intended
behavior even across a full resolve→recur cycle.

## Immediate next steps for the next session
1. Ask user to confirm the `[SMOKE TEST -- IGNORE]` Teams card from step 2 rendered correctly
   (labels/triage text for `pipeline_heartbeat`/`etl_pipeline_staleness`), if not already done.
2. Optionally re-trigger `obs_fresh_scan` (job `585607309385820`) once to confirm SUCCESS
   post-`resolved_at`-fix (low risk, but cheap to check).
3. Resolve the open question above (`acknowledged_at`) with the user — implement if they want
   symmetry with the `resolved_at` fix.
4. Otherwise: keep watching. No other known gaps as of this handoff.

## Environment
- **Dev workspace profile:** `"Tyler Kei"` — use `-p "Tyler Kei"` on every CLI command.
  Never auto-select a profile.
- **Catalog/schema (all writes):** `mq_gmdf_dev.oil_obs`
- **Prod source (read-only):** `mq_gmdf_dp_prd.oil` (via views `v_llm_bronze`, `v_ish_bronze`,
  `v_etl_bronze`)
- **SQL warehouse (dev):** `cd6c1145a46bec44`
- **Secret scope:** `obs-alerting`, key `teams-webhook`
- **obs_fresh_scan job:** `585607309385820`. Tasks:
  `01_bronze_projections` → (`02_liveness_detection` ∥ `03_malformed_output` ∥
  `05_behavioral_correlation`) → `06_alert`. Runs every ~5-10 min.
- **obs_nightly_baseline job:** `428356310089497` — 02:00 America/Indianapolis daily.
- **Workspace notebook path:** `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`
  — local repo root mirrors these names (`ptof_obs_alert.ipynb` etc). **Always re-sync after
  local edits** — the job runs the workspace copy, not local disk.
- **SQL from CLI:** no `databricks sql` command in this CLI (v1.14.1). Use
  `printf '%s' "<SQL>" | python3 /tmp/sql_exec.py` (Statement Execution API wrapper, warehouse
  `cd6c1145a46bec44`, profile `Tyler Kei`). Recreate if missing — see
  HANDOFF_PROD_CHECKUP.md for the exact spec.
- **Serverless one-off Python (e.g. for scratch smoke tests):** upload a `# Databricks
  notebook source`-prefixed `.py` file under
  `/Workspace/Users/tyler.kei@lilly.com/.ai_dev_kit/`, then
  `databricks jobs submit -p "Tyler Kei" --json @submit.json` (see
  `databricks-execution-compute` skill, "Serverless Job" section). Delete the scratch notebook
  after.

## Critical design decisions (do NOT revisit without user sign-off)
- capability_registry inner-join scoping is intentional — only registered capabilities
  monitored. `pipeline_heartbeat`/`etl_pipeline_staleness` are deliberately **unscoped** by
  capability (global backstops) — this is correct and independent of the 4-capability
  rescoping.
- `raise` fires ONLY on UNAVAILABLE (broken detector). CRITICALs route to Teams + obs_incidents,
  never fail the task.
- WARN findings are NOT notified (shift_context_missing, capability_silence are standing gaps).
- All writes stay in `mq_gmdf_dev.oil_obs`; prod catalog `mq_gmdf_dp_prd` is read-only.
- `acknowledged_by`/`acknowledged_at` are hand-set only by explicit prior design — see "Open
  question" above before changing this.

## Notebook files (repo root, /Users/L141230/Downloads/agent_obs)
`ptof_obs_setup_seed.ipynb`, `ptof_obs_bronze_projection.ipynb`, `ptof_obs_liveness_detection.ipynb`,
`ptof_obs_mal_output.ipynb`, `ptof_obs_behavioral_correlation.ipynb`, `ptof_obs_nightly_baseline.ipynb`,
`ptof_obs_alert.ipynb` (the one edited this session, twice).
