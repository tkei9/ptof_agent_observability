# Handoff: Production-readiness hardening (post v1.1 simplification)

## State right now
All 7 phases of the pipeline-simplification plan (`/Users/L141230/.claude/plans/sleepy-gathering-flask.md`)
are committed on `main` and verified end-to-end (full `obs_fresh_scan` job run succeeded on all 6
tasks after the changes). Nothing is uncommitted in the working tree except this file and the
`handoff/` scratch directory (old backups from the simplification pass — safe to ignore/delete,
not needed anymore since everything they backed up is committed and verified).

Databricks: CLI profile `"Tyler Kei"`, catalog `mq_gmdf_dev.oil_obs`, SQL warehouse
`cd6c1145a46bec44`, workspace notebooks live at
`/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`. Job `obs_fresh_scan` =
job id `585607309385820` (6 tasks: 01_bronze_projections, 02_latency_detection,
03_malformed_output, 04_hallucination_detection, 05_behavioral_correlation, 06_alert).
`ptof_obs_setup_seed.ipynb` and `ptof_obs_nightly_baseline.ipynb` are NOT in that job's DAG —
run manually. **Running jobs / notebooks against the live warehouse requires asking the user
directly first** (auto-mode classifier treats this as a shared-resource action) — either ask via
AskUserQuestion naming the exact action (e.g. "this will DROP table X"), or give the user the CLI
command to run themselves.

## What just happened: fresh architecture review

Ran a full production-readiness review (senior-data-architect persona, general-purpose agent,
fresh context — not colored by the implementation work) across all 7 notebooks plus live
schema/data in `mq_gmdf_dev.oil_obs`. Full findings below, ranked by severity. **Verdict: not
ready for production data as-is.** Two findings are live, confirmed conditions in the dev catalog
right now, not hypotheticals.

### Minimum bar before cutover (do these first, in this order)

**1. `capability_registry` inner-join blind spot (Architecture — most severe, confirmed live)**
Every detection query across `ptof_obs_latency_detection.ipynb` (cells 1,3,5,6,7,8,9,11),
`ptof_obs_mal_output.ipynb` (cells 2,3,4), `ptof_obs_hallucination_detection.ipynb`
(cells 2,3,6), `ptof_obs_nightly_baseline.ipynb` (cells 1,2) does
`JOIN capability_registry r ON r.capability = b.capability AND r.active = true`. Live
`v_llm_bronze` has **7 capabilities with real recent traffic not in `capability_registry` at
all**: `dsa_optimizer_step` (1,953 calls since 8/29), `dsa_optimizer_narrate` (825),
`dsa_copilot_narrate` (246), `dsa_canvas_director` (230), `downtime_rationale` (25, first seen
9/8), `alarm_rationale` (3, first seen 9/9), `dsa_ask` (2). `capability_registry` currently has
exactly 11 rows. `alarm_rationale`/`downtime_rationale` look like new GxP-relevant SAA/ISH
capabilities — exactly what this pipeline exists to catch.

The one thing that looks like it should catch this — `transport_violation_signatures`'s
`unknown_capability` violation_type in `ptof_obs_mal_output.ipynb` cell-3 — can't fire either,
because its own `flagged` CTE *also* inner-joins `capability_registry r ... AND r.active = true`
before the `LEFT JOIN runtime_allowlist`. So a new capability is invisible everywhere, including
to the one check meant to catch "is this even a known capability."

**Fix direction:** add a new, cheap heartbeat-style check — `v_llm_bronze` capabilities NOT IN
`capability_registry`, unconditioned on `active` — that pages loudly on its own (belongs in
`ptof_obs_alert.ipynb` as a new `check()`, or as a new INCIDENT_SOURCES entry if it should persist
to `obs_incidents`). Separately, fix `mal_output.ipynb` cell-3's `unknown_capability` CTE so it
doesn't require registry membership to evaluate "is this registered."

**2. Incident lifecycle is broken (Architecture/Query — confirmed live)**
`ptof_obs_alert.ipynb` cell-4's MERGE into `obs_incidents`:
`WHEN MATCHED THEN UPDATE SET t.last_detected=..., t.detection_count=..., t.signal_payload=...`
— **`severity` is not in that list.** `runtime_violation` was demoted CRITICAL→WARN on
2026-08-31 (per the comment in `INCIDENT_SOURCES`), but live data shows **11 `runtime_violation`
incidents still sitting at `severity='CRITICAL'`** in `obs_incidents`, unacknowledged, last
detected 8/31 — they'll keep re-notifying at the old severity forever because nothing corrects
`t.severity` on re-detection.

More broadly: there is no auto-resolve. `resolved_at`/`acknowledged_at` are "set by hand" only.
Right now there are **235 open, unacknowledged CRITICAL incidents** across 8 detectors
(`hallucination_high`=139 since 9/2, `handover_delivery`=77 since 8/31, etc.) with zero
acknowledgments recorded anywhere. This is the exact failure mode a real on-call rotation hits
within weeks of production traffic: an ever-growing backlog, some permanently mis-severitized,
none auto-expiring.

**Fix direction:** (a) add `t.severity = s.severity` to the MERGE's `UPDATE SET` in
`ptof_obs_alert.ipynb` cell-4. (b) add a staleness-based auto-resolve path — e.g. a new cell/check
that sets `resolved_at = current_timestamp()` for incidents whose `last_detected` hasn't advanced
in N runs (meaning the underlying condition stopped recurring), so the backlog doesn't grow
unbounded pending human review of every single row.

**3. `threshold_basis` coverage gaps on paging checks (Schema)**
Checked live `threshold_basis` (20 rows) against every hardcoded threshold in
`ptof_obs_alert.ipynb` and `ptof_obs_mal_output.ipynb`. Missing entirely, most importantly
**`blank_output`'s three-part threshold** (`>3 blank AND >=10 total AND >2% rate`,
`ptof_obs_mal_output.ipynb` cell-2 — feeds a CRITICAL Teams alert with zero documented basis).
Also missing: `hallucination_unverified_rate`, `schema_field_missing`'s presence-rate floor,
`latency_fixed_ceiling`, `nightly_baseline_staleness`, `pipeline_heartbeat`, `credential_outage`,
`latency_baseline_missing`, `latency_anomaly_unreliable_baseline`, `runtime_allowlist_populated`,
`unacknowledged_critical`, `handover_delivery_new_failure`.

**Fix direction:** add `threshold_basis` rows for every check above (`ptof_obs_setup_seed.ipynb`'s
threshold_basis INSERT cell, following the existing anti-join pattern), starting with
`blank_output` since it's the only one with zero documented rationale on a paging CRITICAL.
Consider adding a verification-notebook assertion (in `ptof_obs_verification.ipynb`) that every
`check()`/`INCIDENT_SOURCES` name with a numeric literal has a corresponding `threshold_basis`
row, so this can't silently drift again.

### Not cutover-blocking, but real — schedule for the next pass after the minimum bar

4. **`hallucination_signal` window is watermark-anchored, not a stable lookback.**
   `ptof_obs_hallucination_detection.ipynb` cell-6: `WHERE b.called_at >= GREATEST(watermark, now() - 7 DAYS)`,
   watermark advances to `now()` every run, table is `CREATE OR REPLACE`'d fresh each run — so
   between runs it holds only "since last run" (minutes), not 7 days. Live: table currently has
   **0 rows** despite 139 open `hallucination_high` incidents. Anyone querying this table outside
   the post-run instant will misread "empty" as "healthy." `ptof_obs_alert.ipynb` cell-16's
   `hallucination_unverified_rate` check queries this table with its own independent 24h filter —
   same under-counting risk. Fix: decouple "scored since watermark" bookkeeping from "available
   for detection" (should be a stable rolling window unconditioned on watermark position).

5. **Unescaped f-string interpolation of live data into SQL** in `ptof_obs_alert.ipynb`'s
   `BACKTRACK` dict (cell-4) — the `where` lambdas for `handover_delivery_rate`,
   `capability_error_rate_sustained`, `blank_output`, `schema_field_missing`, `latency_anomaly`,
   `prompt_size_drift` interpolate `capability`/`model_config`/`row_id` (sourced from
   `obs_incidents.signal_payload`, itself from external LLM-audit data) with no quote-escaping —
   unlike `_mask_emails`'s deliberate defense-in-depth elsewhere in the same file. Fix: escape
   (`.replace("'","''")`) the same way `mal_output.ipynb`'s `:env` binding already does correctly.

6. **`required_fields` on `capability_registry` is confirmed fully dead** — no detector reads it,
   only referenced in `setup_seed.ipynb`'s own comment saying it's no longer seeded. Fix:
   `ALTER TABLE ... DROP COLUMN required_fields` (confirm Delta column-mapping is enabled first).

7. **Orphaned tables from the "already cleaned up" pass are still physically present (empty) in
   THIS live catalog.** `SHOW TABLES` still lists `blank_output_incidents`, `transport_violations`,
   `capability_outage_findings` (all 0 rows) — the Phase-6/7 cleanup cell exists in code and WAS
   run once during Phase 7 verification (see prior session), but double-check current state before
   assuming it's still applied — re-run `ptof_obs_setup_seed.ipynb`'s cleanup cell if these
   reappear.

8. **`obs_incidents.signal_payload` as untyped JSON string** will get harder to operate on as
   incident volume grows (every consumer does `get_json_object`/`json.loads` rather than querying
   structured columns). Not urgent at today's volume (944 rows); revisit if ad hoc SQL against
   `signal_payload` becomes common at real prod volume.

## Progress update (2026-09-09)

**Item 1 — SKIPPED, by explicit user decision, not a bug.** The user clarified that the
`capability_registry` inner-join scoping across every detector is intentional (see the SAA/ISH
rescope, commit `8ebdbf1`: 4 active capabilities, not everything in `v_llm_bronze`). The 7
"unregistered" capabilities this review found live are out-of-scope traffic by design, not a
blind spot. The user will specify which capabilities should be actively tracked once the
pipeline integrates with prod data — **do not** add capability-registry-membership alerting
(new `check()`, new `INCIDENT_SOURCES` entry, or loosening the `unknown_capability` CTE join in
`ptof_obs_mal_output.ipynb` cell-3) until that list is provided. See memory
`agent-obs-capability-scoping` for full context if picking this up in a new session.

**Item 2 — DONE, both halves.** (a) commit `a2c96e2`: added `t.severity = s.severity` to the
`WHEN MATCHED` branch of `ptof_obs_alert.ipynb` cell-4's MERGE into `obs_incidents` — the 11
mis-severitized `runtime_violation` incidents will self-correct next run. (b) commit `e072c42`:
added a staleness-based auto-resolve cell immediately after the MERGE loop — any incident whose
detector ran cleanly this run but didn't re-detect it now gets `resolved_at` stamped, excluding
detectors that errored this run (tracked via `_skipped_detectors`) so a broken query can't be
misread as "resolved." No auto-resolve for `acknowledged_at` — that remains hand-set by design.

**Item 3 — DONE**, commit `eca9524`. Added `threshold_basis` rows (documentation only, no
threshold values or detector logic changed) for all 12 checks flagged as missing: `blank_output`,
`credential_outage`, `hallucination_unverified_rate`, `handover_delivery_new_failure`,
`latency_anomaly_unreliable_baseline`, `latency_baseline_missing`, `latency_fixed_ceiling`,
`nightly_baseline_staleness`, `pipeline_heartbeat`, `runtime_allowlist_populated`,
`schema_field_missing`, `unacknowledged_critical`.

**Item 4 — DONE**, commit `22e28e5`. `hallucination_signal`'s detection window
(`ptof_obs_hallucination_detection.ipynb` cell-6) changed from `GREATEST(watermark, now()-7d)`
to a stable `now() - INTERVAL 7 DAYS`, decoupled from the `_obs_watermark` bookkeeping that
`faithfulness_scores`' incremental scoring uses. That watermark still governs only Layer-2
scoring, not what's in scope for detection.

**Item 5 — DONE**, commit `88410f7`. Added a `_sqlq()` escaping helper in
`ptof_obs_alert.ipynb`'s `BACKTRACK` dict (cell-4) and applied it to every interpolated
`capability`/`model_config`/`row_id` value across all 6 affected lambdas.

**Items 6 and 7 — NOT YET DONE, both require a live warehouse action** (an `ALTER TABLE ...
DROP COLUMN` and a set of `DROP TABLE IF EXISTS`, respectively) — code-only work is exhausted for
these; see their original writeups above for exact commands. Ask the user before running either.

**Item 8 — deliberately deferred**, per the original review itself ("not urgent at today's
volume... revisit if ad hoc SQL against signal_payload becomes common at real prod volume").

## Next steps for the fresh session

Everything code-only from this review is committed (`a2c96e2`, `eca9524`, `22e28e5`, `88410f7`,
`e072c42`, plus this file's updates). Nothing has been run against the live warehouse yet by this
pass. Before any live action, ask the user directly (per the standing rule at the top of this
file):

1. Run `ptof_obs_setup_seed.ipynb`'s `capability_registry`/`threshold_basis` cells to apply the
   item-3 documentation rows (and any other pending seed-table changes).
2. Run/verify `ptof_obs_alert.ipynb` (via `obs_fresh_scan` job or interactively) and confirm: the
   11 `runtime_violation` incidents now read `severity='WARN'` (item 2a); the incident count
   trends down rather than growing unbounded on subsequent runs (item 2b); `hallucination_signal`
   now holds a real 7-day window of rows, not 0 (item 4).
3. Decide on items 6 and 7 (both live DDL/DML, both low-risk/low-urgency) — do them opportunistically
   once you're already in the warehouse for step 1/2, or skip for now.
4. Separately, confirm whether/when the user wants to provide the prod capability tracking list
   referenced in item 1's skip decision (see memory `agent-obs-capability-scoping`).
