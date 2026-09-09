# Handoff: Prod Integration (2026-09-09)

## What this pipeline is
An SAA/ISH agent observability pipeline running on Databricks. 10 notebooks, 6 scheduled as
job `obs_fresh_scan` (job id `585607309385820`), 2 manual (`setup_seed`, `nightly_baseline`),
1 weekly digest, 1 verification. Detects latency anomalies, hallucinated content, blank
outputs, schema drift, error rates, handover delivery failures, and transport violations
across LLM agent capabilities. Findings persist to `obs_incidents` with acknowledgement
lifecycle and route CRITICAL findings to a Teams Adaptive Card.

## Current environment (dev)
- **Databricks CLI:** profile `"Tyler Kei"`, workspace
  `https://adb-6777798819429555.15.azuredatabricks.net`
- **Catalog/schema:** `mq_gmdf_dev.oil_obs` (hardcoded as `CAT` in every notebook)
- **SQL warehouse:** `cd6c1145a46bec44`
- **Workspace path:** `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`
- **Teams webhook:** `obs-alerting` secrets scope, key `teams-webhook` (same webhook for prod)
- **Source bronze tables:** `mq_gmdf_dev.oil.ptof_primary__ai_llm_audit_log` (LLM calls),
  `mq_gmdf_dev.oil.ptof_primary__ish_audit_log` (ISH entities)

## What's ready
All code-only hardening from the v1.1 architecture review is done, committed on `main`, synced
to the Databricks workspace, and verified against live dev data (two successful `obs_fresh_scan`
job runs post-changes). Specifically:

1. **Incident lifecycle works end-to-end** — severity updates on re-detection, staleness-based
   auto-resolve for incidents that stop recurring (went from 235 open unacked CRITICAL → 104
   genuinely active), Teams notify with 24h re-suppress, broken detectors raise the task.
2. **Hallucination detection has a stable 7-day window** — 1,550 rows, 73 high-risk (was 0
   rows due to a watermark-coupling bug, now fixed).
3. **SQL injection defense** — all f-string interpolations in Teams-card backtracking queries
   are escaped via `_sqlq()`.
4. **32 threshold_basis rows** — every hardcoded threshold in the pipeline is documented with
   rationale and validation status.
5. **Runtime violation simplified to transport-only** — model_config and scheduler_run removed
   from allowlist evaluation (−278 lines). model_config remains as attribution in error rate,
   blank output, and hallucination detectors where per-config granularity matters.

## What the user needs to provide for prod

### 1. Capability list (blocking — everything else is gated on this)
The pipeline only monitors capabilities in `capability_registry` with `active = true`. Currently
4 active: `saa_insight`, `sev2_insight`, `summary`, `watchout_narratives`. The user said they
will provide the prod capability list before integration. Each capability needs:
- `capability` (string, the name as it appears in the audit log)
- `is_gxp_relevant` (boolean — does the output need grounding checks?)
- `is_groundable` (boolean — can the output meaningfully be grounded against input data?)
- `owner` (string — team or person responsible)
- `silence_grace_hours` (int, nullable — how long before "no calls" is a concern)
- Which `transport` values are valid for it (for `runtime_allowlist`)

**Do NOT add unregistered-capability alerting.** The `capability_registry` inner-join scoping
is intentional — the user explicitly decided this (see memory `agent-obs-capability-scoping`).

### 2. Prod catalog/schema
Replace `mq_gmdf_dev.oil_obs` → prod equivalent everywhere. It's set as `CAT` in cell-1 of
every notebook except `ptof_obs_mal_output.ipynb` (which hardcodes it in `%sql` cells). The
user confirmed the prod catalog is similar but has some schema differences — need to inspect
the prod bronze tables and adjust `v_llm_bronze`/`v_ish_bronze` projections in
`ptof_obs_bronze_projection.ipynb` if column names or types differ.

### 3. Prod workspace
Prod is in a **different Databricks workspace**. Need:
- Workspace URL and CLI profile for the prod workspace
- Import the notebooks there
- Create the `obs_fresh_scan` job (6 tasks, same DAG structure)
- Create the `obs-alerting` secrets scope with the Teams webhook (same webhook URL)
- Run `ptof_obs_setup_seed.ipynb` to create all tables and seed reference data

### 4. Credential fastfail hardcode
`ptof_obs_bronze_projection.ipynb` cell-1 has:
```sql
CASE WHEN model_config = 'demo-claude-sonnet-4-6-pwc-omi'
      AND error_msg = 'Unable to locate credentials' THEN true ELSE false
END AS is_credential_fastfail
```
This model_config won't exist in prod. Either parameterize it, replace with the prod
equivalent, or broaden the pattern to match on `error_msg` alone.

## Remaining low-priority cleanup (not blocking prod)

### Items 6/7 — DDL the auto-mode classifier won't run
Run these yourself against the **dev** warehouse if you want to clean up:
```sql
-- Item 6: drop dead column (enable column mapping first)
ALTER TABLE mq_gmdf_dev.oil_obs.capability_registry
  SET TBLPROPERTIES ('delta.columnMapping.mode' = 'name');
ALTER TABLE mq_gmdf_dev.oil_obs.capability_registry DROP COLUMN required_fields;

-- Item 7: drop orphaned empty tables
DROP TABLE IF EXISTS mq_gmdf_dev.oil_obs.blank_output_incidents;
DROP TABLE IF EXISTS mq_gmdf_dev.oil_obs.transport_violations;
DROP TABLE IF EXISTS mq_gmdf_dev.oil_obs.capability_outage_findings;
```

### Threshold recalibration
32 thresholds, most marked `provisional` or `unvalidated`. They're anchored to dev-environment
observed ranges. Once prod data flows, review whether they fire too much (noisy) or not enough
(blind spots): `SELECT * FROM threshold_basis WHERE status LIKE 'unvalidated%'`.

### Item 8 — signal_payload as untyped JSON
Not urgent at current volume (~1,000 rows). Revisit if ad hoc SQL against `signal_payload`
becomes a pain at real prod volume.

## Architecture quick reference

### Job DAG (`obs_fresh_scan`)
```
01_bronze_projections
  ├── 02_latency_detection
  ├── 03_malformed_output
  ├── 04_hallucination_detection
  └── 05_behavioral_correlation
        └── 06_alert  (depends on all of 02-05)
```

### Key tables
| Table | What | Written by |
|-------|------|-----------|
| `v_llm_bronze` | Bronze projection (VIEW) | `01_bronze_projections` |
| `v_ish_bronze` | ISH bronze projection (VIEW) | `01_bronze_projections` |
| `capability_registry` | Human-curated: which capabilities to monitor | `setup_seed` (manual) |
| `runtime_allowlist` | Human-curated: sanctioned transports per capability | `setup_seed` (manual) |
| `obs_incidents` | Append-only finding record, MERGE-deduped | `06_alert` |
| `threshold_basis` | Why each threshold is set where it is | `setup_seed` (manual) |
| `hallucination_signal` | Combined hallucination verdict per call | `04_hallucination_detection` |
| `capability_latency_baseline` | Nightly p50/p95/p99 per capability | `nightly_baseline` (manual) |
| `transport_violation_signatures` | Transport-only violation detection | `03_malformed_output` |
| `runtime_observed` | What's been seen but not yet sanctioned | `weekly_runtime_digest` |

### Commits this session (on `main`)
| Commit | What |
|--------|------|
| `a2c96e2` | Fix severity not updating on incident re-detection |
| `eca9524` | Document 12 missing threshold_basis rows |
| `22e28e5` | Decouple hallucination_signal from watermark (0→1,550 rows) |
| `88410f7` | SQL-escape all BACKTRACK f-string interpolations |
| `e072c42` | Auto-resolve for incidents that stop recurring (235→104 open) |
| `04defa7` | Simplify runtime violation to transport-only (−278 lines) |

### Git state
Branch `main`, 15 commits ahead of `origin/main` (not pushed). `handoff/` directory is
untracked scratch from prior sessions — safe to ignore or delete.
