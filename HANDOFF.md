# Handoff: SAA/ISH Agent Observability Pipeline — Read This First

> **Start a fresh context window, paste this file's contents as your first message.**
> Today's date is 2026-09-21. This file replaces `HANDOFF_PROD_CHECKUP.md` and all prior
> plan-mode scratch files (see "Superseded" at the bottom) — it is now the single source of
> truth for this pipeline's architecture and open items.

## Your role
Operator of the SAA/ISH agent observability pipeline on Databricks. Monitors an LLM agent's
outputs/delivery and upstream ETL health, persists findings to `obs_incidents`, posts triage
Adaptive Cards to Teams.

## Why this pipeline reads prod but writes dev — permission-verified, not a workaround
Confirmed via `SHOW GRANTS ON CATALOG mq_gmdf_dp_prd`: the operating identity has only
`SELECT` / `USE` / `BROWSE` on the prod catalog — **no `CREATE`**. There is no grant path to
write into prod. Reading `mq_gmdf_dp_prd.oil` and writing all detector output to
`mq_gmdf_dev.oil_obs` is therefore the only permission-compliant architecture, not a
stepping-stone to something else. Do not revisit "move everything into prod" — it was
investigated on 2026-09-15/16 and closed for this reason.

## Full architecture — bronze projection downstream

```
mq_gmdf_dp_prd.oil (READ-ONLY)                    mq_gmdf_dev.oil_obs (ALL WRITES)
─────────────────────────────                     ──────────────────────────────
ptof_primary__ai_shift_outputs ──┐
ptof_ish_audit ──────────────────┼──► ptof_obs_bronze_projection.ipynb
ptof_etl_pipeline_audit ─────────┘         │
                                            ▼
                          v_llm_bronze, v_ish_bronze, v_etl_bronze  (pass-through views,
                          alias prod column names to the names every downstream notebook uses:
                          output_type→capability, generated_at→called_at, etc.)
                                            │
                ┌───────────────┬───────────┼───────────────┬─────────────────────┐
                ▼               ▼           ▼               ▼                     ▼
  ptof_obs_liveness_   ptof_obs_mal_   ptof_obs_behavioral_  ptof_obs_nightly_
  detection.ipynb       output.ipynb    correlation.ipynb     baseline.ipynb (separate
  (job task              (job task       (job task             job, runs nightly,
  02_latency_detection)  03_malformed_   05_behavioral_        NOT in obs_fresh_scan)
       │                 output)         correlation)               │
       ▼                     ▼                ▼                     ▼
  capability_silence    blank_output_    handover_delivery_    response_field_baseline
  shift_context_missing findings         failures               write_lag_baseline
  etl_pipeline_health   response_        handover_delivery_     etl_duration_baseline
  write_lag_anomalies   schema_drift     rate[_findings]      (read by liveness_detection,
  etl_run_slow                                                 mal_output for baselines)
                └───────────────┴────────────────┴──────────────────────┘
                                            │
                                            ▼
                              ptof_obs_alert.ipynb (job task 06_alert)
                              1. check() each condition in Python
                              2. MERGE CRITICAL findings into obs_incidents (dedup on
                                 (detector, source_row_id); resolved_at/acknowledged_at
                                 cleared on re-detection)
                              3. auto-resolve incidents whose detector ran clean this run
                              4. notify: CRITICAL, unacknowledged, unresolved, not notified
                                 in 24h → Teams Adaptive Card. WARN is NEVER notified.
                              5. raise() ONLY on UNAVAILABLE (broken detector — not "found
                                 something"), so job status means "is monitoring working"
                                            │
                                            ▼
                                    Teams (via obs-alerting/teams-webhook)
```

`ptof_obs_setup_seed.ipynb` is off to the side — not in any job DAG, run by hand. It seeds
`capability_registry` (which capability/table this pipeline is scoped to and its thresholds)
and `threshold_basis` (append-only documentation of every detector's threshold, including
retired ones marked `status='not_applicable_prod'`). Every other notebook reads from these
two tables; nothing else writes to them.

## Active detectors (13)

| # | Detector | Severity | Notebook | Notified to Teams? |
|---|---|---|---|---|
| 1 | `handover_delivery` | CRITICAL | behavioral_correlation | Yes |
| 2 | `handover_delivery_rate` | CRITICAL | behavioral_correlation | Yes |
| 3 | `blank_output` | CRITICAL | mal_output | Yes |
| 4 | `schema_field_missing` | CRITICAL | mal_output | Yes |
| 5 | `pipeline_heartbeat` | CRITICAL | alert (scalar) | Yes |
| 6 | `etl_pipeline_failure` (aka `etl_pipeline_health`) | CRITICAL | liveness_detection | Yes |
| 7 | `etl_pipeline_staleness` | CRITICAL | alert (scalar, **global**) | Yes |
| 8 | `etl_table_staleness` (added 2026-09-21) | CRITICAL | liveness_detection | Yes |
| 9 | `etl_run_slow` (added 2026-09-18 WARN, promoted CRITICAL 2026-09-21) | CRITICAL | liveness_detection | Yes |
| 10 | `capability_silence_ceiling` (added 2026-09-21 WARN/168h, promoted CRITICAL and tightened to 120h same day) | CRITICAL | liveness_detection | Yes |
| 11 | `capability_silence` | WARN | liveness_detection | No — log only |
| 12 | `shift_context_missing` | WARN | liveness_detection | No — log only |
| 13 | `write_lag_anomalies` (added 2026-09-18, 300s floor added 2026-09-21) | WARN | liveness_detection | No — log only |

WARN findings are computed and printed to the job log by `ptof_obs_alert.ipynb`, but are
**structurally incapable** of reaching `obs_incidents` or Teams (they're never in
`INCIDENT_SOURCES`/`SCALAR_INCIDENT_SOURCES`, and cell 11's notify query filters
`severity = 'CRITICAL'`). This is deliberate anti-noise design, not an oversight — items 11-13
ship WARN because their thresholds are new/unvalidated (13: MAD-based on a degenerate
zero-variance baseline, see `threshold_basis`, `status='provisional'`; on hold pending the data
owner, see "FP/FN bias review follow-up" below) or a coarse backstop on a thin sample (11).
`etl_run_slow` (item #9) and `capability_silence_ceiling` (item #10) were both promoted out of
WARN to CRITICAL 2026-09-21 — see "FP/FN bias review follow-up" below. `capability_silence_ceiling`
has not yet fired on a real occurrence, unlike `etl_run_slow` before its promotion — its
promotion is a judgment call from historical-gap analysis, not accumulated live-fire evidence.

## ✅ FIXED 2026-09-21 — per-table ETL staleness blind spot

**Status: built and signed off 2026-09-21.** New CRITICAL detector `etl_table_staleness`
(`ptof_obs_liveness_detection.ipynb`), wired into `ptof_obs_alert.ipynb`'s `INCIDENT_SOURCES`
and `DETECTOR_META`, threshold documented in `threshold_basis`. Answers to the three open
questions below, as decided:
1. **Grace threshold:** uniform `minutes_since_last_run > 60` across all 19 tables (not
   per-table) — real cadence is consistently ~13-16 min fleet-wide, no evidence of a
   genuinely-slower subset that would need its own threshold.
2. **Runs alongside the existing global `etl_pipeline_staleness`, not replacing it** — the
   global scalar is the cheap, unambiguous "is the ETL scheduler alive at all" signal for a
   total outage (one incident instead of up to 19 near-duplicates), and its dead-simple
   single-aggregate logic is independent insurance against a bug in this more complex
   per-table query ever creating a total blind spot.
3. **Ships CRITICAL from day one**, not staged as provisional/WARN — it reuses the
   already-proven simple-interval pattern (same shape as `pipeline_heartbeat`/
   `etl_pipeline_staleness`), not a new unvalidated statistical model like the 2026-09-18 MAD
   detectors.

Keyed on a **constant** `sha2(table_or_view, 256)` (not table + window), so an ongoing stall is
one incident whose `detection_count` climbs and whose `resolved_at` auto-clears the moment the
table resumes — same lifecycle as `pipeline_heartbeat`/`etl_pipeline_staleness`. Grouped
directly off `v_etl_bronze` (no `capability_registry`-style seed list), so coverage can't drift
if a table is added or removed upstream.

**Not yet done:** deploy to the Databricks workspace (`databricks workspace import` for
`ptof_obs_liveness_detection.ipynb`, `ptof_obs_alert.ipynb`, `ptof_obs_setup_seed.ipynb`) and
run the seed notebook's `threshold_basis` insert cell once against prod. Local files are
updated; the scheduled job still runs the old workspace copies until synced.

## Original writeup (kept for history) — per-table ETL staleness blind spot

**Incident that exposed it:** 14 of the 19 ETL source tables (`ptof_primary__b100_alarms`,
`b100_interventions`, `campstatus`, `mtc_status`, `mtc_status_previous`,
`pfs3_combined_cycle_status`, `pfs3_pmx_completed_steps`, `pfs3_pmx_process_values`,
`pfs3_pmx_steps`, `pfs3_traksys_events`, `sanit_status`, `seeq_primary`, `seeq_signals`,
`tbb_config`) went silent for **~24 hours over the weekend, 2026-09-19 ~12:00 UTC through
2026-09-20 ~13:00 UTC**, then resumed normally on their own. Verified directly against
`mq_gmdf_dp_prd.oil.ptof_etl_pipeline_audit`: zero `status='failure'` rows during the gap —
the tables simply stopped producing runs, no error was ever logged.

**Confirmed nothing detected or notified this:** `obs_incidents` has zero rows for
`etl_pipeline_failure` or `etl_pipeline_staleness` in the last 5 days.

**Root cause:** `etl_pipeline_staleness` (`ptof_obs_alert.ipynb` cell 4) checks
`max(run_timestamp)` across **all 19 tables combined** — a single global scalar. During the
outage the other 5 tables kept running every hour on schedule, which kept the global max fresh
and completely masked the 14-table stall. `etl_pipeline_failure` only fires on an explicit
failure row, which never existed here — the tables didn't fail, they just stopped being
scheduled/written.

**Neither of the two detectors added 2026-09-18 (`write_lag_anomalies`, `etl_run_slow`) would
have caught this either** — both only evaluate rows that *did* get written; they have no
concept of a table that stopped writing altogether.

### Proposed fix (historical — see "FIXED 2026-09-21" above for what was actually built)
Add a **per-`table_or_view` staleness check**, replacing or supplementing the current global
one:
- New table `etl_table_staleness` (or extend `etl_pipeline_health`'s notebook) —
  `GROUP BY table_or_view`, compute `minutes_since_last_run = now() - max(run_timestamp)` per
  table, compare against a per-table or fleet-wide grace threshold above the normal ~10-15 min
  refresh cadence (HANDOFF's existing docs cite this cadence). Something like
  `minutes_since_last_run > 60` (4-6x normal cadence, avoids paging on a single slow cycle)
  would have caught this incident within ~45-50 minutes of it starting, instead of never.
- Severity: CRITICAL — recommend persisting to `obs_incidents` via the existing
  `INCIDENT_SOURCES` table-backed pattern (like `etl_pipeline_failure`), keyed on
  `sha2(table_or_view || window_start)` so each stalled table is its own incident and
  auto-resolves independently once it resumes.
- Open questions for you to decide before I build anything:
  1. Grace threshold — is 60 minutes the right number, or should it vary per table (some of
     the 19 tables may have a slower natural cadence than others — worth checking before
     picking one number for all 19)?
  2. Should this **replace** the existing global `etl_pipeline_staleness`, or run **alongside**
     it? The global check is still a useful "is the ETL scheduler itself alive at all" signal
     even with per-table coverage added.
  3. CRITICAL from day one, or provisional/WARN first (like `write_lag_anomalies`/`etl_run_slow`)
     until proven not to be noisy on the other 5 tables' normal cadence variance?

(Answered and built 2026-09-21 — see "FIXED" section above.)

## ✅ FIXED 2026-09-21 — items #2-#5 from the detector value audit

**Status: all built and signed off 2026-09-21**, same session as item #1 (`etl_table_staleness`
above). Local files only — **not yet deployed to the workspace** (see item #1's note; all four
share the same pending deploy).

- **Item #2 — `capability_silence_ceiling` (new WARN detector).** Backstop for
  `sev2-insights`, the one capability with `silence_grace_hours IS NULL` (irregular cadence)
  and therefore structurally excluded from `capability_silence` — before this, a permanent
  silent failure of `sev2-insights` had no detection path at any length. New
  `silence_ceiling_hours` column on `capability_registry` (NULL for every capability except
  `sev2-insights`, set to **168h / 7 days**). New table `capability_silence_ceiling`
  (`ptof_obs_liveness_detection.ipynb`) + `check()` in `ptof_obs_alert.ipynb`. WARN, log-only —
  same as `capability_silence`. 168h was chosen with ~65h/1.6x margin over the worst historical
  gap ever observed (102.5h) and also lands on a clean calendar week; decided by the user, not
  derivable from the thin ~572-row sample alone.
- **Item #3 — `write_lag_anomalies` 300s floor.** The audit found `write_lag_baseline`'s
  median and MAD are both exactly 0 for every capability×scheduler_run (write_lag_s is 0 for
  almost all prod rows), so the "MAD-based" threshold had degenerated to `write_lag_s > 0`.
  Fixed to `write_lag_s > greatest(upper_bound_s, 300)` — a 5-minute floor so only genuinely
  extreme lag can trip this regardless of the still-degenerate baseline. 300s chosen (over an
  initial 30s placeholder) specifically because the user wants this to only catch *extreme* lag,
  not "more than trivial."
- **Item #4 — `etl_run_slow` grain fix.** Rolled up from
  `(table_or_view, task_name, window_start)` to `(task_name, window_start)`, with
  `affected_tables`/`affected_table_count` preserved in the payload. All tables under one
  `task_name` move together — the 4 real slowdown events in the audit's 7-day window each hit
  14-19 of 19 tables simultaneously, so the old grain produced 14-19 near-duplicate rows per
  real event. Stays WARN after the fix; observe longer before considering promotion to
  CRITICAL, per the audit's recommendation.
- **Item #5 — `handover_delivery_rate`'s stale `threshold_basis` text.** The audit flagged that
  its documented basis (`"all-time baseline 9.3%... currently 12.5% over 7d"`) was pre-migration
  dev data, not prod reality (actual: 0% failure, 0/49 all-time, 0/14 last 7d). Corrected via an
  explicit `UPDATE` in `ptof_obs_setup_seed.ipynb` (the anti-join seed INSERT won't overwrite an
  already-existing row). The user asked why this doesn't just get replaced by per-occurrence
  detection: it already exists, separately — `handover_delivery` (detector #1) already fires on
  every single failed send, keyed on `ish_row_id`. `handover_delivery_rate` is a deliberate
  second layer answering a different question (is the underlying failure mechanism getting
  systemically worse, a trend/escalation signal with its own independent incident lifecycle),
  not a duplicate of per-event detection — this is now recorded directly in the corrected basis
  text.

## Environment
- **Dev workspace profile:** `"Tyler Kei"` — use `-p "Tyler Kei"` on every CLI command. Never
  auto-select a profile.
- **Catalog/schema (all writes):** `mq_gmdf_dev.oil_obs`
- **Prod source (read-only, no CREATE grant):** `mq_gmdf_dp_prd.oil` (via views `v_llm_bronze`,
  `v_ish_bronze`, `v_etl_bronze`)
- **SQL warehouse (dev):** `cd6c1145a46bec44`
- **Secret scope:** `obs-alerting`, key `teams-webhook`
- **obs_fresh_scan job:** `585607309385820`. Tasks: `01_bronze_projections` →
  (`02_latency_detection` ∥ `03_malformed_output` ∥ `05_behavioral_correlation`) →
  `06_alert`. Runs every ~5-10 min.
- **obs_nightly_baseline job:** `428356310089497` — 02:00 America/Indianapolis daily. Now
  computes `response_field_baseline`, `write_lag_baseline`, `etl_duration_baseline`. Confirmed
  ran clean 2026-09-21 06:00 UTC with the new baselines (7 and 19 rows respectively).
- **Workspace notebook path:** `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`
  — local repo root mirrors these names. **Always re-sync after local edits** (`databricks
  workspace import <path> --file <local.ipynb> --format JUPYTER --overwrite -p "Tyler Kei"`) —
  the job runs the workspace copy, not local disk. Verify with `workspace export` + diff before
  trusting a deploy.
- **SQL from CLI:** no `databricks sql` command in this CLI (v1.14.1). Use
  `printf '%s' "<SQL>" | python3 /tmp/sql_exec.py` (Statement Execution API wrapper, warehouse
  `cd6c1145a46bec44`, profile `Tyler Kei`). **This file does not persist across sessions** —
  recreate it if missing (read SQL from stdin, POST to `/api/2.0/sql/statements/`, poll
  `/api/2.0/sql/statements/{id}` until `SUCCEEDED`, print `manifest.schema.columns` + rows).

## Critical design decisions (do NOT revisit without user sign-off)
- capability_registry inner-join scoping is intentional — only registered capabilities
  monitored.
- `pipeline_heartbeat`/`etl_pipeline_staleness` are deliberately **unscoped** by capability
  (global backstops). `etl_table_staleness` (added 2026-09-21) is their per-table complement —
  runs alongside, not instead of, `etl_pipeline_staleness`; see "FIXED 2026-09-21" above for
  the full supplement-vs-replace reasoning.
- `raise` fires ONLY on UNAVAILABLE (broken detector). CRITICALs route to Teams + obs_incidents,
  never fail the task.
- WARN findings are NOT notified (`shift_context_missing`, `capability_silence`,
  `write_lag_anomalies` are standing/provisional gaps by design).
  `etl_run_slow` and `capability_silence_ceiling` were both promoted out of this list to CRITICAL
  2026-09-21 — see "FP/FN bias review follow-up" below.
- All writes stay in `mq_gmdf_dev.oil_obs`; prod catalog `mq_gmdf_dp_prd` is read-only (no
  CREATE grant — verified, not assumed).
- `acknowledged_by`/`acknowledged_at` are hand-set only by explicit prior design; cleared
  automatically only on re-detection (see the resolved_at/acknowledged_at fix below).
- `is_groundable=false` for all prod capabilities — hallucination detection dropped (4.9% FP
  rate on computed numbers being flagged as ungrounded). A redesigned claim-based approach was
  researched 2026-09-17/18 but **not approved** — see "Hallucination rebuild" below.
- `write_lag_anomalies`/`etl_run_slow` measure pipeline write-lag / ETL-run-duration, not model
  inference latency — true per-call latency (`latency_ms` on
  `ptof_primary__ai_llm_audit_log`) is blocked: that table exists in prod but has **0 rows**.

## Prior fixes already deployed (context, don't redo)
- `resolved_at`/`acknowledged_at` are cleared in the MERGE `WHEN MATCHED` clause on
  re-detection (fixed 2026-09-15) — without this, any detector with a stable/time-invariant key
  (`blank_output`, `schema_field_missing`, `handover_delivery_rate`, `pipeline_heartbeat`,
  `etl_pipeline_staleness`) would go permanently dark the first time it resolved and later
  recurred.
- `pipeline_heartbeat`/`etl_pipeline_staleness` wired into `SCALAR_INCIDENT_SOURCES` so they
  can actually reach `obs_incidents`/Teams (fixed 2026-09-15) — previously computed but never
  persisted or notified.
- Phase 1 latency/ETL-duration detectors (`write_lag_anomalies`, `etl_run_slow` +
  their baselines) added 2026-09-18, deployed and verified byte-identical to workspace
  2026-09-21, confirmed producing rows via the 2026-09-21 06:00 UTC nightly baseline run.
- `etl_table_staleness` (CRITICAL) added 2026-09-21 — per-table companion to the global
  `etl_pipeline_staleness` scalar, closing the confirmed masking gap from the 2026-09-19/20
  weekend incident (documented above). Built in `ptof_obs_liveness_detection.ipynb`, wired into
  `ptof_obs_alert.ipynb`'s `INCIDENT_SOURCES`/`DETECTOR_META`, threshold recorded in
  `threshold_basis`. **Local files only as of 2026-09-21 — not yet deployed to the workspace.**

## Hallucination rebuild — researched, not approved, do not build
A research brief (2026-09-17) proposed claim-based verification to replace the dropped
regex+similarity hallucination detector. Three pieces would need individual sign-off before any
code is written: (1) `capability_registry.groundability_mode` column, (2) a
`numeric_recomputation_rules` table, (3) a claim extraction/verification pipeline (recommended
scoped to `summary` capability only, shadow mode first). None of the three has been approved.
True per-call inference-latency detection is separately blocked on
`ptof_primary__ai_llm_audit_log` being populated (0 rows today) — flag to the owning team if/when
you want to pursue it.

## Notebook files (repo root, /Users/L141230/Downloads/agent_obs)
`ptof_obs_setup_seed.ipynb`, `ptof_obs_bronze_projection.ipynb`, `ptof_obs_liveness_detection.ipynb`,
`ptof_obs_mal_output.ipynb`, `ptof_obs_behavioral_correlation.ipynb`, `ptof_obs_nightly_baseline.ipynb`,
`ptof_obs_alert.ipynb`.

## Architecture review prompt — evaluate detector value against prod behavior

> Not yet run. Paste this whole section as the first message of a fresh session when you
> want this review done. It is a review/documentation task only — **no code or detector
> changes**, no writes beyond appending findings to this file, consistent with the standing
> "don't add anything without my permission and complete transparency" rule.

```
You are acting as a senior data architect doing a value/necessity audit of this
observability pipeline. Read HANDOFF.md in this repo first for full architecture context —
don't re-derive it. Your job: for each of the 11 detectors in the "Active detectors" table,
decide whether it is EARNING its place given how prod has actually behaved recently, not
given how it was designed to behave in theory.

For each detector, do this:
1. State its stated purpose and threshold (pull from threshold_basis where present).
2. Query the real prod-derived data it watches (v_llm_bronze / v_ish_bronze / v_etl_bronze,
   and capability_registry for scoping) over a recent window (start with the last 14-30 days;
   widen if the detector's cadence is sparser than that). Characterize actual behavior:
   how often would this detector fire, on what, how close to its threshold does real data
   sit, is there a capability/table it never covers because of registry scoping.
3. Cross-reference against obs_incidents history for that detector: how many times has it
   actually fired, were those firings true positives (real operational problems) or noise,
   how many times was it acknowledged vs auto-resolved vs re-notified.
4. Render a verdict: KEEP AS-IS / RETUNE (threshold too loose or too tight — say which
   direction and why, with the numbers that justify it) / DOWNGRADE-UPGRADE SEVERITY /
   RETIRE (no longer catches anything real, or superseded by another detector) / GAP (does
   NOT cover something it looks like it should — the per-table etl_pipeline_staleness gap
   documented above is exactly this pattern: check whether ANY other detector has a similar
   blind spot, e.g. does capability_silence's per-capability scoping have the same
   masking problem etl_pipeline_staleness had, does handover_delivery_rate's ratio hide a
   single-capability collapse the way the global ETL max hid a 14-table stall).

Do this same treatment for the two provisional WARN detectors (write_lag_anomalies,
etl_run_slow) specifically: they were added 2026-09-18 with unvalidated MAD thresholds —
you now have several days of real prod data against them. Are their thresholds firing at a
sane rate, or are they silently useless (never fire) or would-be-noisy-if-promoted (fire
constantly)? That's the evidence needed before anyone decides whether to promote them out of
WARN.

Also assess the two scalar CRITICAL backstops (pipeline_heartbeat, etl_pipeline_staleness)
against the weekend incident: pipeline_heartbeat is even coarser than etl_pipeline_staleness
— confirm whether it would catch the same 14-table stall or is equally blind, and say so
explicitly.

Output: a markdown table (detector | verdict | one-line evidence) plus a short paragraph per
detector that got anything other than KEEP AS-IS, with the specific query results backing the
call. Append this as a new "## Detector value audit (run <date>)" section at the bottom of
HANDOFF.md — do not edit or delete any existing section. Do not change any detector code,
threshold, or seed data as part of this — flag proposed changes as open questions the same
way the per-table staleness gap above does, and wait for sign-off.
```

## Detector value audit (run 2026-09-21)

> Review/documentation only — no detector code, threshold, or seed data changed. All numbers
> below are from live queries against `v_llm_bronze`/`v_ish_bronze`/`v_etl_bronze`,
> `capability_registry`, `obs_incidents`, and the detector output tables, run 2026-09-21.
> **`obs_incidents` has 0 rows total** — every CRITICAL detector has been live since 2026-09-15
> (or 2026-09-18 for the two WARN additions) and has literally never persisted a finding. That
> single fact underlies most of the "KEEP AS-IS, unproven-not-noisy" verdicts below.

| # | Detector | Verdict | One-line evidence |
|---|---|---|---|
| 1 | `handover_delivery` | KEEP AS-IS | 0 of 49 prod handover attempts (all-time, 2026-08-27→) have failed; `handover_delivery_failures` has 0 rows; correctly designed, just untested by a real failure yet. |
| 2 | `handover_delivery_rate` | KEEP AS-IS (doc stale) | Actual prod rate is 0% (0/49 all-time, 0/14 last 7d) vs. a 20% trip threshold — huge headroom; `threshold_basis`'s "9.3% baseline / 12.5% current" numbers are pre-migration dev data and don't describe prod reality. |
| 3 | `blank_output` | KEEP AS-IS | blank_rate = 0.0000 for all 4 capabilities over 30 days (0 blank outputs ever recorded); insurance detector, correctly structured, simply nothing to catch yet. |
| 4 | `schema_field_missing` | KEEP AS-IS | `response_schema_drift` has 0 rows; baselines exist for all 4 active capabilities (3-16 fields each); no drift observed in 24h windows checked. |
| 5 | `pipeline_heartbeat` | KEEP AS-IS + confirmed blind to the weekend gap | Max gap between any two `v_llm_bronze` rows in 30 days is 16.3 min (threshold 2h) — never close to firing; **confirmed equally blind to the 14-table ETL stall** (see below) since LLM output never stopped. |
| 6 | `etl_pipeline_failure` (`etl_pipeline_health`) | KEEP AS-IS | Exactly 1 `status='failure'` row in all of `v_etl_bronze` history, dated 2026-09-03 — before this detector went live (2026-09-15); 0 obs_incidents rows; correctly scoped to explicit failures, structurally unable to see "stopped scheduling" (that's the documented staleness gap, not this detector's job). |
| 7 | `etl_pipeline_staleness` | GAP (confirmed) | During the 2026-09-19–20 outage, `v_etl_bronze` held steady at exactly 5 tables × ~12 runs/hr the entire ~25h window — the global max never went stale, confirming the masking independently of the incident writeup. Only other 30d gap >30min (155.7 min) was 2026-09-02, pre-dating this detector's 2026-09-15 deploy. |
| 8 | `capability_silence` | GAP (new, same shape as #7) | `sev2-insights` (`silence_grace_hours = NULL`) has produced **zero rows since 2026-09-18 22:23 UTC — 64+ hours as of this audit** — and is structurally excluded from `capability_silence`'s query (`WHERE silence_grace_hours IS NOT NULL`), so nothing is watching it at all. Its historical gap distribution (p95 0.2h, max 102.5h) means a 64h gap isn't yet abnormal for this specific capability, but there is **no backstop at any length** — a permanent silent failure of `sev2-insights` would never be detected. |
| 9 | `shift_context_missing` | KEEP AS-IS | 0 rows currently for all active capabilities — shift context fields are fully populated in the last 7 days; standing-gap-by-design but not presently triggering. |
| 10 | `write_lag_anomalies` (WARN, provisional) | RETUNE / GAP in its statistical basis | `write_lag_baseline` shows `median_write_lag_s = 0` and `mad_write_lag_s = 0` for **every** capability×scheduler_run combo → `upper_bound_s = 0.0000`. `write_lag_s` is in fact always exactly 0 in the last 24h (p50/p95/max all 0). The MAD formula degenerates to "> 0" with zero statistical tolerance — this isn't a validated 5×MAD bound, it's a bare nonzero-value tripwire that happens to never fire because the underlying metric has zero variance. If write_lag ever becomes nonzero even briefly, it will trip on effectively any occurrence (moderated only by the ≥3/hour rollup), not a genuine anomaly threshold. Currently silently useless; would-be-noisy-if-tripped is unproven either way since it's never seen a real nonzero value. |
| 11 | `etl_run_slow` (WARN, provisional) | Real, repeatable signal — evidence favors eventual promotion, not yet | Fired 4 distinct times in the last 7 days (2026-09-15 13:00, 19:00, 20:00 UTC; 2026-09-20 22:00 UTC), each time correlated across 14-19 of the 19 tables simultaneously, durations 440-878s vs. ~440-462s baselines (roughly 1.1-2x median). This is not noise — it looks like a real periodic fleet-wide ETL slowdown (shared upstream bottleneck, since tables under the same `task_name` always move together) recurring multiple times a week. |

### `etl_pipeline_staleness` vs `pipeline_heartbeat` — both confirmed blind to the weekend incident
Direct query against `v_etl_bronze` for 2026-09-19 00:00 through 2026-09-20 12:00 UTC shows
`count(distinct table_or_view)` pinned at exactly 5 every single hour (60 rows/hr) — the other 14
tables produced zero rows, yet the global `max(run_timestamp)` never crossed the 30-minute
staleness threshold because those 5 tables never stopped. `pipeline_heartbeat` is coarser still:
it watches `v_llm_bronze` (agent output), which kept flowing normally throughout — that detector
was never in a position to see an ETL-only outage regardless of how many tables stalled. Neither
is a design flaw in itself; the documented per-table staleness gap above is the fix for #7, and
#6/#5 are working exactly as scoped.

### `capability_silence`'s new gap (#8) vs. the documented `etl_pipeline_staleness` gap (#7)
Same failure shape, different mechanism: #7 is masked by *aggregation* (14 silent tables hidden
behind 5 noisy ones in a single global scalar). #8 is masked by *exclusion* — `sev2-insights` is
simply never evaluated because its `silence_grace_hours` is `NULL` by design (irregular cadence,
documented in `capability_registry`). The result is the same: a capability-level outage with no
detection path. Whether a 64h-and-counting gap on `sev2-insights` right now is business-as-usual
or an actual problem is a question for the capability owner, not something this audit can
resolve from the data alone — flagging as an open question, no code changed:
- Should `sev2-insights` get *some* backstop (e.g., a much longer fixed ceiling like 7 days,
  or a "hasn't produced ANY output in N days" check independent of the per-shift grace model),
  or is "no ceiling, ever" the intentional design given its natural cadence?

### `handover_delivery_rate`'s ratio — checked for the same single-stream-collapse pattern
`handover_delivery_rate` is not scoped per-capability (it watches one ISH email stream, not the
4 LLM capabilities), so there's no analogous "one bad capability hidden in a global average"
risk the way `etl_pipeline_staleness`'s per-table gap works. No masking pattern found here.

### `write_lag_anomalies` / `etl_run_slow` promotion readiness
- **`write_lag_anomalies`**: not ready to evaluate, let alone promote — the baseline it computes
  against is degenerate (0/0/0.0000 for every capability×scheduler_run), so "provisional MAD
  threshold" is currently indistinguishable from "any nonzero value at all." Recommend leaving
  in WARN and revisiting once `write_lag_s` has actually been observed nonzero in prod, rather
  than promoting a threshold that has never been tested against real variance.
- **`etl_run_slow`**: has fired 4 times in 7 days on what looks like a real, recurring,
  fleet-wide slowdown — the strongest evidence of any provisional detector that it's watching a
  real condition, not noise. Two open questions before promoting, not decided here: (1) is 4x/wk
  an acceptable CRITICAL/Teams cadence, or should it be collapsed from per-table to per-`task_name`
  first (all tables under one task always move together, so 14-19 near-duplicate rows per event
  is likely to read as alert noise even though the underlying signal is real); (2) is a
  fleet-wide slowdown actually actionable/CRITICAL, or is it better suited to staying WARN
  long-term as a "watch, don't page" seasonality signal.

## False-positive/false-negative bias review prompt (drafted 2026-09-21)

> Run this whenever you want a fresh pass at whether the 13 active detectors are erring the
> right direction. The user's explicit standing preference: **accept a chance of false
> positives; do not accept any chance of false negatives.** Review/documentation only — no
> detector code, threshold, or seed-data changes. Any proposed change is an open question that
> waits for the user's item-by-item sign-off, same as every other change in this pipeline's
> history. Append findings as a new "## FP/FN bias review (run <date>)" section at the bottom
> of HANDOFF.md — do not edit or delete any existing section.

```
You are acting as a senior observability engineer auditing this pipeline's 13 active
detectors (see HANDOFF.md "Active detectors" table) for false-positive vs. false-negative
risk. Read HANDOFF.md first for full architecture context, including the 2026-09-21
"Detector value audit" section — don't re-derive what's already documented there, build on it.

The user's stated bias, which should drive every verdict below: a detector that occasionally
fires on something benign (false positive) is acceptable and correctable via triage; a
detector that stays silent through a real problem (false negative) is not. When a threshold or
scoping choice trades one risk for the other, prefer the choice that leans toward more false
positives / fewer false negatives, and say so explicitly if today's design leans the other way.

For each of the 13 detectors, answer:
1. **False-negative risk**: under what real, plausible prod condition would this detector
   fail to fire even though something is actually wrong? Consider: registry/scoping exclusions
   (anything like the capability_silence NULL-grace gap or the etl_pipeline_staleness global-max
   masking already found in the 2026-09-21 audit — check every other detector for the same two
   shapes: exclusion-based blind spots and aggregation-based masking), thresholds set so loose
   they'd never trip on a real-magnitude problem, and detectors that only see one failure mode
   of a capability while blind to others (e.g. schema_field_missing only sees missing fields,
   not wrong-but-present values).
2. **False-positive risk**: under what real, plausible prod condition would this detector fire
   on something that is actually fine? Query recent v_llm_bronze/v_ish_bronze/v_etl_bronze data
   to check how close normal variation sits to each threshold — a threshold sitting right on top
   of normal noise is a false-positive risk even if it hasn't fired yet.
3. **Net verdict given the stated bias**: TOO LOOSE (real false-negative risk found — tighten,
   propose a specific new threshold/scoping fix with numbers) / TOO TIGHT (no false-negative
   risk, but also minimal false-positive risk — meaning there's slack to tighten further and
   still be safe, not a problem on its own but worth noting) / WELL-CALIBRATED (leans toward
   false positives as intended, or is a zero-tolerance backstop that should stay maximally
   sensitive) / CANNOT ASSESS (sample too thin to say either way — say what data would be
   needed).

Additionally, using actual prod data trends (not hypothetical design), answer:
4. Is there any **failure mode currently uncovered by any of the 13 detectors** — something
   that could go wrong operationally in this pipeline that no existing check, even an
   imperfect one, would ever catch? Look for gaps the same way the 2026-09-21 audit found the
   etl_pipeline_staleness/capability_silence pair: something structurally invisible, not just
   statistically rare.
5. For each detector already flagged WARN/provisional (write_lag_anomalies, etl_run_slow,
   capability_silence_ceiling), given the stated false-negative-averse bias, is WARN (never
   notified to Teams) itself a false-negative risk in disguise — i.e., is there a real problem
   this detector would correctly identify today that is going completely unseen because WARN
   findings never reach a human? Say explicitly, per detector, whether "log-only" is
   appropriate caution (unvalidated threshold) or is itself the false negative the user wants
   eliminated.
6. Are there any **new detectors** the prod data suggests are worth adding — a real failure
   mode visible in the data that nothing above covers? Propose with a specific query-backed
   threshold, not just a concept.
7. For every detector already firing or with headroom data available, is there a **specific
   retune** (tighter threshold, broader scoping, finer grain) that would reduce false-negative
   exposure without producing an unreasonable volume of false positives? Give the number and
   the data that justifies it.

Output: a markdown table (detector | false-negative risk (one line) | false-positive risk (one
line) | verdict) covering all 13 detectors, followed by a short paragraph for every detector
that is not WELL-CALIBRATED, with the specific query results backing the call, followed by a
"Candidate new detectors" subsection and a "Candidate retunes" subsection (each item numbered,
each with the query evidence and a specific proposed number). Every proposed change is an open
question awaiting sign-off — do not implement any of them.
```

## FP/FN bias review (run 2026-09-21)

> Review/documentation only — no detector code, threshold, or seed data changed. Numbers below
> are from live queries against `v_llm_bronze`/`v_ish_bronze`/`v_etl_bronze`, `capability_registry`,
> `obs_incidents`, `threshold_basis`, and detector output tables, run 2026-09-21. Confirmed
> **items #1 (`etl_table_staleness`), #2 (`capability_silence_ceiling`), and #4's
> `etl_run_slow` grain fix are still local-files-only** — `SHOW TABLES` on `mq_gmdf_dev.oil_obs`
> and `DESCRIBE capability_registry` both confirm the deployed workspace copy predates all of
> this session's earlier fixes, so several of the risks below are live in prod right now, not
> just historical.

| # | Detector | False-negative risk | False-positive risk | Verdict |
|---|---|---|---|---|
| 1 | `handover_delivery` | None found — fires on every row in `handover_delivery_failures` (0 rows currently); risk would live upstream in whatever populates that table, not in this check. | Minimal — only fires on an actual failure row. | WELL-CALIBRATED |
| 2 | `handover_delivery_rate` | Real volume is only 14 attempts/7d (0 failed); the `attempts >= 10` floor means a burst of e.g. 3-4 failures out of 5 attempts in a slow week (n<10) would never trip despite an 60-80% real failure rate. | None observed (0% actual). | TOO LOOSE (sample floor, not the 20% threshold itself) |
| 3 | `blank_output` | `summary` capability averages ~3.2 calls/day (97 calls/30d) — the `>=10 total_calls` 6h-window floor can structurally never be met for this capability even if every recent output were blank. | None observed (blank_rate = 0.0000 for all 4 capabilities/30d). | TOO LOOSE for low-volume capabilities (`summary`, and any future low-traffic one) |
| 4 | `schema_field_missing` | Scoping matches: `response_field_baseline` has rows for exactly the 4 `active=true` capabilities (saa-display 5 fields, situational-awareness 16, summary 3, sev2-insights 5) — no exclusion gap. | None observed (0 rows). | WELL-CALIBRATED (CANNOT ASSESS FP fully — never fired) |
| 5 | `pipeline_heartbeat` | Real max gap in `v_llm_bronze` over 30d is 16.4 min; threshold is 2h (7.3x normal) — a genuinely stuck pipeline could run silent for up to ~1h44m past 3x normal cadence before this fires. Confirmed still structurally blind to an ETL-only outage (watches LLM output, which never stopped during the 09-19/20 incident). | None — headroom is large, i.e. very unlikely to false-positive at 2h. | TOO LOOSE (large FN window within its own domain) |
| 6 | `etl_pipeline_failure` | Only sees explicit `status='failure'` rows (1 ever, pre-dates this detector); by design does not see "stopped scheduling" — that's `etl_table_staleness`'s job, not a gap in this detector's scope. | None observed. | WELL-CALIBRATED (narrow, correctly scoped) |
| 7 | `etl_pipeline_staleness` | **Confirmed aggregation-masking**: during the 09-19/20 outage, `v_etl_bronze` held at exactly 5/19 tables running the whole ~25h window, so the global max never went stale. This blind spot is only closed once `etl_table_staleness` (item #1) is actually deployed — right now it is not (`SHOW TABLES` confirms the table doesn't exist in the deployed workspace), so **this exposure is live today, not historical**. | None — 30-min threshold vs. real ~12-16 min cadence has comfortable margin. | TOO LOOSE until item #1 ships; WELL-CALIBRATED as a companion layer once it does |
| 8 | `etl_table_staleness` | Would close #7's gap at a 60-min per-table grace vs. real ~12.4 min cadence (4.8x margin) — sound design. **Not yet deployed**, so today it provides zero actual coverage; the false-negative risk isn't in the design, it's in the pending deploy. | Low — 4.8x margin over real cadence. | WELL-CALIBRATED once deployed; **currently non-existent in prod** |
| 9 | `capability_silence` | Structurally excludes `sev2-insights` (`silence_grace_hours IS NULL`) by design — confirmed again: `sev2-insights` is 65.6h silent right now (last call 2026-09-18 22:23 UTC) and climbing, with zero coverage from this detector. | None observed (0 rows for the 3 capabilities it does cover). | GAP (known, backstopped by #13 once deployed) |
| 10 | `shift_context_missing` | 0 rows currently; standing WARN/log-only gap-by-design, no evidence either way yet. | None observed. | CANNOT ASSESS (thin sample) |
| 11 | `write_lag_anomalies` | **`write_lag_s` is exactly 0 for all 10,679 rows in the last 30 days (0 nonzero, p99=0, max=0)** — not "rarely nonzero," *literally never* nonzero. No threshold, 300s floor or otherwise, can catch real write lag if the upstream field never reflects it. This reads as a possible instrumentation gap, not (only) a threshold problem. | None possible — can't false-positive on a field that's always 0. | CANNOT ASSESS / likely data-integrity gap upstream of this detector |
| 12 | `etl_run_slow` | Confirmed a **5th** correlated slowdown event since the prior audit (2026-09-20 22:00 UTC, 12 tables, ~532s vs ~440-462s baseline, 1.15-1.2x) — a real, repeating signal that is WARN/log-only and has never reached a human despite 5 occurrences in ~1 week. | Low — each firing has been a genuine correlated multi-table event, not noise. | TOO LOOSE **on the notification path**, not the threshold — see below |
| 13 | `capability_silence_ceiling` | 168h ceiling vs. current 65.6h-and-climbing `sev2-insights` gap means this won't fire for another ~102h even if the silence is a genuine outage; and even when it does fire, it's WARN — never reaches a human. | Low — 168h has 1.6x margin over the worst historical gap (102.5h), by design. | TOO LOOSE **on the notification path** — see below |

### Items #7/#8 — the etl_pipeline_staleness masking gap is live in prod today, not just historical
`SHOW TABLES IN mq_gmdf_dev.oil_obs` does not list `etl_table_staleness`, and
`DESCRIBE capability_registry` has no `silence_ceiling_hours` column — both confirm the
deployed workspace still predates every fix from this session (items #1-#5). Until
`ptof_obs_liveness_detection.ipynb`/`ptof_obs_alert.ipynb`/`ptof_obs_setup_seed.ipynb` are
redeployed, a repeat of the 09-19/20 masking incident would go undetected exactly as before.
**Recommend prioritizing that deploy** over any further tuning below — it's a already-signed-off
fix sitting un-shipped, not a new open question.

### Items #12/#13 — WARN-forever is itself a false negative for a proven-real signal
The bias review's question 5 asks directly: is "log-only" hiding a real problem from a human?
For `etl_run_slow`, yes by the data — 5 real, correlated, multi-table slowdown events in about
a week with zero human visibility. For `capability_silence_ceiling`, the honest answer is
"can't tell yet" — `sev2-insights` hasn't crossed its own 168h ceiling, so there's no evidence
of a real silent failure being hidden, only the structural fact that if one occurred, WARN would
still hide it from a human. Flagging both as open questions, no change made:
- Should `etl_run_slow` get *some* notification path (even a lower-severity Teams post or a
  daily digest, short of full CRITICAL/page) given it's now proven to be a real recurring signal
  and not noise?
- Should `capability_silence_ceiling` have a staged design — WARN for validation now, with a
  planned promotion to notified once it's been observed not to fire spuriously — rather than an
  open-ended WARN?

### Item #11 — write_lag_s being always exactly 0 looks like an instrumentation question, not a threshold question
Before tuning `write_lag_anomalies`'s floor any further, worth checking with whoever owns the
upstream write path whether `write_lag_s` is actually wired up to measure real lag, or whether
it's a placeholder/always-zero field in the current data-generation process. If lag is a real
operational concern, no amount of detector-threshold tuning fixes a metric that never varies.

### Candidate new detectors
1. **`write_lag_instrumentation_check`** — WARN if `write_lag_s` has been exactly 0 for 100% of
   rows over the last N days (e.g. 7) for any capability with `expected_min_daily > 0`. This
   distinguishes "no lag" from "lag is never measured," which the current anomaly detector can't
   tell apart. Query basis: 0/10,679 nonzero rows over 30 days today.
2. **Low-volume floor bypass for `blank_output`** — for capabilities where
   `expected_min_daily` (from `capability_registry`) implies fewer than ~10 calls per 6h window
   (e.g. `summary`, `expected_min_daily=1`), either lower the `total_calls` floor proportionally
   or widen the window so a real blank-output failure isn't masked by insufficient sample size.
   Query basis: `summary` = 97 calls/30d ≈ 3.2/day, well under the 10-calls-per-6h floor.

### Candidate retunes
1. **`pipeline_heartbeat`**: tighten from 2h to ~45 min. Real max gap over 30 days is 16.4 min;
   45 min is still 2.7x that, comfortable margin against false positives, but cuts the
   false-negative exposure window by ~63% (105 min → 45 min) versus the current 2h.
2. **`handover_delivery_rate`**: lower the `attempts >= 10` floor, or add an absolute-count
   secondary trigger (e.g. `OR failed >= 3` within the window) so a real failure cluster during
   a low-volume week (current pace: 14/7d) isn't invisible purely because it didn't clear the
   sample-size floor.
3. **`blank_output`**: see "Candidate new detectors" #2 above — same underlying floor issue,
   framed as a retune of the existing threshold rather than a new detector, whichever the user
   prefers.

Every item above is an open question awaiting sign-off — no detector code, threshold, or seed
data was changed as part of this review.

## Detector tuning review prompt — risk-tolerance-driven retune pass (drafted 2026-09-21)

> Not yet run. Paste this whole section as the first message of a fresh session when you want
> this review done. It is a review/documentation task only — **no detector code, threshold, or
> seed-data changes**, no writes beyond appending findings to this file, same standing rule as
> every other review prompt above. This supersedes the general "accept FP, reject FN" framing of
> the FP/FN bias review prompt above with a more specific risk-tolerance statement the user gave
> directly this session — use this prompt's framing, not a re-derivation of the older one.

```
You are acting as a senior observability engineer doing a tuning-correctness pass over every
active detector in this pipeline (see HANDOFF.md "Active detectors" table — 13 detectors as of
2026-09-21, including capability_silence_ceiling now CRITICAL/120h). Read HANDOFF.md first for
full architecture context, including the 2026-09-21 "Detector value audit" and "FP/FN bias
review" sections and their follow-up fixes — don't re-derive what's already documented there,
build on it and note anywhere its findings are now stale (e.g. because a fix shipped since).

The user's exact stated risk tolerance for this pass, more specific than "prefer false positives
over false negatives" in general: "I am okay with a slight increase in false positive rate if
that means the detectors never miss anything that could provide valuable insight... I am okay
with trading a slight increase in false positive rate for a minimal chance of false negatives."
Read this precisely: a SLIGHT false-positive increase is the acceptable price, in exchange for a
MINIMAL (not necessarily zero — zero is not achievable, as established this session) false-
negative chance. A retune that would produce a large or unbounded false-positive increase is NOT
what this bias asks for, even in service of shrinking false negatives further — flag that
tradeoff explicitly if you find one, rather than recommending it by default.

**Data source constraint — prod data only, no exceptions:** every empirical claim in this review
must come from data traceable to the prod catalog `mq_gmdf_dp_prd.oil`, i.e. `v_llm_bronze`,
`v_ish_bronze`, `v_etl_bronze` (the pass-through views over prod), and `capability_registry`
filtered to `active = true` (the 4 live prod capabilities: `saa-display`,
`situational-awareness`, `sev2-insights`, `summary`). Explicitly exclude from every query and
every finding:
- the historical dev capabilities in `capability_registry` (`active = false`: `saa_insight`,
  `sev2_insight`, `watchout_narratives`, `dsa_*`, `probe`, etc.) — these are dev-only rows kept
  for reference, not prod signal, and must never be counted toward an empirical rate.
- any `not_applicable_prod` rows in `threshold_basis` — retired dev-era detectors, out of scope
  for this review entirely (don't re-assess them).
- any pre-migration dev-era basis text still sitting in `threshold_basis` that hasn't been
  corrected yet — if you find one, flag it as stale documentation (same as the 2026-09-21
  `handover_delivery_rate` fix) rather than treating it as a valid prod baseline.
If a query result looks like it might include a dev-era row (e.g. an unexpectedly large row
count, a capability name you don't recognize from the 4 active ones), stop and check
`capability_registry`/`active` scoping before reporting the number. If you are ever unsure
whether a table, column, or row is prod-sourced or dev-only, stop and ask rather than guessing.

For each of the 13 detectors, do this:
1. Pull its current threshold/scoping from `threshold_basis` and the detector's own SQL.
2. Query the real prod data it watches (per the data source constraint above) over a recent
   window (last 14-30 days; widen if its cadence is sparser, e.g. sev2-insights). Compute: how
   close does normal variation sit to the current threshold, how many historical occurrences
   would have crossed it (empirical false-positive count/rate), and whether there is a labeled or
   observed real failure this detector should have caught (empirical false-negative evidence,
   where any exists at all).
3. Explicitly separate what CAN be empirically measured (false-positive rate, from historical
   data that stayed under threshold) from what CANNOT (false-negative rate, absent labeled
   failures) — say which case each detector is in, don't imply a threshold is "safe" just because
   it's never fired.
4. Render a verdict against the stated bias specifically:
   - RETUNE-TIGHTER: propose a specific new number, backed by the query results, where tightening
     costs zero or near-zero additional empirical false positives (same standard used for the
     capability_silence_ceiling 168h→120h decision this session: identical empirical FP rate at
     both candidate values, but a materially smaller false-negative exposure window at the
     tighter one).
   - ALREADY AT THE FLOOR: the threshold sits at or near the tightest value the historical data
     supports without crossing into real, repeated false positives — say so and stop, don't
     manufacture a change.
   - STRUCTURAL GAP, NOT A THRESHOLD PROBLEM: cases like write_lag_anomalies (paused,
     instrumentation question) or a detector whose scoping excludes a whole capability/table —
     no threshold number fixes this; say what does.
   - NOTIFICATION-PATH GAP: threshold and scoping are fine, but the detector is WARN/log-only and
     therefore invisible to a human even when it correctly fires (this session's
     capability_silence_ceiling/etl_run_slow question, now partly resolved for etl_run_slow and
     capability_silence_ceiling — check whether any of the 3 remaining WARN detectors
     (capability_silence, shift_context_missing, write_lag_anomalies) fit this shape).
   - WELL-CALIBRATED: no change indicated under this bias.

Also explicitly revisit the two known standing gaps from this session under this specific bias
framing (do they still apply, has anything shipped that changes the answer):
- `capability_silence`'s exclusion of `sev2-insights` (mitigated by `capability_silence_ceiling`,
  now CRITICAL/120h — is that mitigation itself now well-calibrated under this bias, or does a
  120h floor still leave an unacceptably large minimal-but-nonzero false-negative window given
  how the user framed "minimal"?).
- Whether `write_lag_anomalies` should stay paused, or whether the "slight FP increase for
  minimal FN" bias changes the calculus on shipping some interim floor while the SME question is
  still open (the user's standing pause instruction takes precedence if it conflicts with this
  bias — flag the tension explicitly rather than silently overriding the pause).

Output: a markdown table (detector | current threshold | empirical FP evidence | empirical/known
FN evidence | verdict) covering all 13 detectors, followed by a short paragraph for every
detector that is not WELL-CALIBRATED or ALREADY AT THE FLOOR, with the specific query results and
a specific proposed number where a retune is recommended. Every number in the table and every
paragraph must cite a query against prod-sourced data per the data source constraint above — if
a finding cannot be backed by prod data (e.g. no prod occurrences exist yet to measure), say so
explicitly (CANNOT ASSESS) rather than filling the gap with a dev-era number or an assumption.
Append this as a new "## Detector tuning review (run <date>)" section at the bottom of
HANDOFF.md — do not edit or delete any existing section. Do not change any detector code,
threshold, or seed data as part of this — flag every proposed change as an open question
awaiting the user's item-by-item sign-off.
```

## Superseded (deleted this session — content folded into this file)
- `HANDOFF_PROD_CHECKUP.md` (repo) — prod migration checkup, one gap found, now fixed (see
  "Prior fixes already deployed" above).
- `~/.claude/plans/dapper-enchanting-clarke.md` and 10 other stale plan-mode scratch files
  covering already-completed work (prod source migration, incident-lifecycle fixes,
  architecture hardening, notebook documentation passes, SAA/ISH rescope). All represented
  completed, deployed work with no open items beyond what's captured above.

## ✅ FIXED 2026-09-21 — FP/FN bias review priorities 2, 4, 5

**Status: built and signed off 2026-09-21, same session as the FP/FN bias review above.** Local
files only — **not yet deployed to the workspace** (shares the same pending deploy as every other
change in this session; see "Prior fixes already deployed").

- **Priority 2 — `etl_run_slow` promoted WARN → CRITICAL.** Signed off directly: "priority 2: yes
  just promote it ocritical." Rationale already in the bias review above (item #12) — 5 real,
  correlated, multi-table slowdown events observed in ~1 week while sitting in WARN/log-only,
  meaning a proven-real signal was structurally invisible to a human. Now wired into
  `INCIDENT_SOURCES`/`DETECTOR_META` in `ptof_obs_alert.ipynb`, documented in `threshold_basis`.
- **Priority 4 — absolute-count OR-triggers for `handover_delivery_rate` and `blank_output`.**
  Closes the sample-size-floor false-negative gap flagged in the bias review (items #2, #3):
  `handover_delivery_rate` now trips on `(failure_pct_7d > 20 AND attempts >= 10) OR failed >= 3`;
  `blank_output` now also trips on a low-volume bypass (2-9 total_calls with 100% blank), on top
  of its existing `>3 blank AND >=10 total AND >2% rate` path. Applied consistently across
  `ptof_obs_alert.ipynb` (`INCIDENT_SOURCES`), `ptof_obs_behavioral_correlation.ipynb`
  (`handover_delivery_rate_findings`), `ptof_obs_mal_output.ipynb` (`blank_output_findings`), and
  `threshold_basis`.
- **Priority 5 — `pipeline_heartbeat` tightened 2h → 45min.** Matches the bias review's
  candidate retune #1 exactly (16.4 min real max gap, 45 min is still 2.7x that — comfortable
  margin, cuts the false-negative exposure window by ~63%). Updated in `ptof_obs_alert.ipynb`
  (`DETECTOR_META`, payload field renamed `rows_last_2h`→`rows_last_45m`) and `threshold_basis`.

### Still open from the FP/FN bias review — not decided or deferred
- **Priority 3 — `write_lag_anomalies` instrumentation gap.** `write_lag_s` is exactly 0 for
  every row in the last 30 days (10,679/10,679) — looks like an unwired/placeholder field, not a
  "no lag" signal. **Paused explicitly by the user**: "lets pause on write_lag_s until i get a
  subject matter expert to address this." No further threshold or code work on this detector
  until revisited with SME input. `threshold_basis` carries a note recording the hold.
- **Documentation lag (this section + `threshold_basis` corrections) — approved and applied**
  2026-09-21 ("yes i approve item 4 to be fixed"): `threshold_basis` rows for `blank_output`,
  `handover_delivery_rate`, `pipeline_heartbeat`, and `etl_run_slow` corrected via idempotent
  `UPDATE`s in `ptof_obs_setup_seed.ipynb` (mirrors the pre-existing item-#5 pattern), and this
  HANDOFF.md file brought back in sync (Active detectors table, WARN-notified list, Critical
  design decisions list, this section).

## ✅ FIXED 2026-09-21 — `capability_silence_ceiling` promoted WARN → CRITICAL, tightened 168h→120h

**Status: built and signed off 2026-09-21, same session as the priorities above.** Local files
only — shares the same pending deploy as every other change this session.

Signed off directly: "ok lets do it, after review if everything from the fp/fn bias is
addresse/implemented." Decision process: the user asked a sequence of analytical questions about
whether promoting this detector could guarantee zero false negatives (no — no threshold can), and
how likely the detector was to miss something given a low false-positive rate (unquantifiable
without labeled failure history — false-positive rate is empirically measurable from historical
gaps, false-negative rate is not). Given the user's explicit refined bias — "I am okay with a
slight increase in false positive changes if that means the detectors never misses anything that
could provide valuable insight... trading a slight increase in false positive rate for a minimal
change of false negatives" — the recommendation was: (1) promote WARN → CRITICAL (WARN-forever was
itself judged a false-negative risk for a permanent-dark event of `sev2-insights`, the only
capability this detector covers), and (2) tighten 168h → 120h, since both values have the
identical 0/989 empirical historical false-positive rate (max gap ever observed 102.5h) but 120h
cuts the false-negative exposure window by ~29%.

Implemented: `capability_registry.silence_ceiling_hours` for `sev2-insights` changed 168→120
(`ptof_obs_setup_seed.ipynb`, idempotent seed + UPDATE); `capability_silence_ceiling` table gained
`finding_signature`/`detected_at` columns to fit the standard `INCIDENT_SOURCES` MERGE pattern
(`ptof_obs_liveness_detection.ipynb`); wired into `INCIDENT_SOURCES`/`BACKTRACK`/`DETECTOR_META`
in `ptof_obs_alert.ipynb`, with the old standalone WARN `check()` call removed; `threshold_basis`
row updated via idempotent `UPDATE` for already-seeded environments.

**Caveat carried forward, not resolved by this promotion**: unlike `etl_run_slow` before its
promotion, this detector has never fired on a real occurrence — the promotion is a judgment call
from historical-gap analysis (989 gaps, max 102.5h), not accumulated live-fire evidence. Revisit
once it has real firing history.

## Detector tuning review (run 2026-09-21)

> Review/documentation only — no detector code, threshold, or seed data changed. "Current
> threshold" for every detector below is read from the LOCAL `.ipynb` cells on disk (source of
> truth for what the pipeline is *intended* to do), not from the deployed `threshold_basis`/
> `capability_registry` rows in `mq_gmdf_dev.oil_obs`, which are confirmed stale. All empirical
> query results are live against `v_llm_bronze`/`v_ish_bronze`/`v_etl_bronze`, scoped to
> `capability_registry.active = true` (`saa-display`, `situational-awareness`, `sev2-insights`,
> `summary` only), run 2026-09-21.

### ⚠️ Two findings that must be read before the table below

**1. Every "FIXED 2026-09-21" item in this file is still undeployed, confirmed again this run —
and one of them ("`etl_table_staleness`, built and signed off") is not actually built.**
`DESCRIBE mq_gmdf_dev.oil_obs.capability_registry` still has no `silence_ceiling_hours` column,
and `SHOW TABLES IN mq_gmdf_dev.oil_obs` still does not list `etl_table_staleness` or
`capability_silence_ceiling` — the deployed workspace is running pre-2026-09-21 logic for
`pipeline_heartbeat` (2h, not 45min), `handover_delivery_rate`/`blank_output` (no absolute-count
floors), `etl_run_slow` (still WARN, per-table grain), and has **zero** coverage from
`etl_table_staleness`/`capability_silence_ceiling`, which do not exist there at all. This was
already flagged in the FP/FN bias review ("recommend prioritizing that deploy") — it is still
true today, not a new discovery, but it means every verdict below describes intended local-code
behavior, not what the live pipeline is actually running right now.

**2. New this run: `etl_table_staleness` has no implementation — the CREATE TABLE cell is
missing from `ptof_obs_liveness_detection.ipynb`.** `grep`-ing the notebook for
`CREATE OR REPLACE TABLE` finds `capability_silence`, `capability_silence_ceiling`,
`shift_context_missing`, `etl_pipeline_health`, `write_lag_anomalies`, and `etl_run_slow` (twice —
see below) but **no `etl_table_staleness` cell at all**, despite extensive markdown in the same
notebook and full wiring in `ptof_obs_alert.ipynb` (`INCIDENT_SOURCES`, `DETECTOR_META`,
`check()`) describing it as built. This is not "not yet deployed" (item #1's gap) — it is a
missing implementation that would make the wired-up `check("etl_table_staleness", ...)` call in
`ptof_obs_alert.ipynb` fail with a table-not-found error the moment it ran, regardless of
deployment. Likely lost in the merge referenced in this repo's git log
("Merge origin/main: keep local's notebook (superset of remote's dead-cell-only edit)") — that
same merge left a **dead duplicate `etl_run_slow` cell** in the notebook (the old
`(table_or_view, task_name, window_start)`-grain version, cell after the current
`(task_name, window_start)`-grain one) that is harmless only because a later cell always
overwrites the same table with `CREATE OR REPLACE`. Recommend restoring the
`etl_table_staleness` CREATE TABLE cell (the SQL is fully specified in this file's own "FIXED
2026-09-21" and "Original writeup" sections above — `GROUP BY table_or_view`,
`minutes_since_last_run > 60`, keyed on `sha2(table_or_view, 256)`) and deleting the dead
`etl_run_slow` cell, before any further work on this notebook. This is a build/hygiene gap, not
a threshold question — flagged here as an open item, no code changed as part of this review.

| # | Detector | Current threshold (local code) | Empirical FP evidence | Empirical/known FN evidence | Verdict |
|---|---|---|---|---|---|
| 1 | `handover_delivery` | Any row in `handover_delivery_failures` (per-occurrence, no rate/count floor) | N/A — zero-tolerance backstop, nothing to tune. 0/49 all-time real failures (v_ish_bronze, `HandoverEmail`, since 2026-08-27). | None found — fires on every failure row that exists; a miss would be an upstream instrumentation gap, not this detector. CANNOT ASSESS beyond that (no real failure has ever occurred to test against). | WELL-CALIBRATED |
| 2 | `handover_delivery_rate` | `(failure_pct_7d > 20 AND (sent_ok+failed) >= 10) OR failed >= 3` (OR-floor shipped 2026-09-21, local only) | 0/49 all-time failures → both the rate path and the `failed>=3` floor have **never fired**, 0 empirical FP at either. Weekly attempts range 7-14 (v_ish_bronze, `HandoverEmail`), so the `>=10` rate-path floor is itself sometimes unmet — exactly the gap the `failed>=3` OR-floor was added to close. | Previously TOO LOOSE gap (rate floor unmeetable in a low-volume week) is now closed by the OR-floor — but CANNOT ASSESS whether `failed>=3` itself is right, since it has never fired (0 failures ever, all-time). | ALREADY AT THE FLOOR (given available data) — but **local-only, not deployed** |
| 3 | `blank_output` | `(blank_count>3 AND total>=10 AND rate>0.02) OR (2<=total<10 AND blank_count=total)` (low-volume bypass shipped 2026-09-21, local only) | 0 blank outputs recorded for any of the 4 active capabilities over 30 days — 0 empirical FP on both paths. `summary` (lowest-volume capability, 97 calls/30d, ~3.2/day) reaches up to 12 calls in a rolling 6h window (checked directly) and as few as 2 — so the low-volume bypass is NOT structurally unreachable for `summary` as it would have been pre-fix. | Residual gap: the bypass needs `total>=2` to compute a rate; a rolling-6h window with exactly 1 call for a low-traffic capability still can't be judged (no denominator to call a rate). Not verified whether such a 1-call window actually occurs for `summary` in practice (observed windows cluster in pairs, ~12h apart) — CANNOT FULLY RULE OUT. | RETUNE-TIGHTER (see below) |
| 4 | `schema_field_missing` | `current_present = 0 AND baseline_presence_rate >= 0.2` on `current_rows >= 10` | 0 rows in `response_schema_drift` with `drift_type='field_missing'` today — 0 empirical FP. Scoping confirmed exact: baselines exist for all 4 `active=true` capabilities, no exclusion. | None found in scoping; real detection performance CANNOT ASSESS — no real schema drift has occurred yet to test against. | WELL-CALIBRATED |
| 5 | `pipeline_heartbeat` | `0 rows in v_llm_bronze, trailing 45 minutes` (tightened from 2h 2026-09-21, local only) | Real max gap across ALL capabilities, `v_llm_bronze`, 30 days = **16.27 min**. 0 gaps over 20 min, 0 over 25 min, 0 over 30 min — the 45min threshold has very wide slack. | Genuinely stuck pipeline could run silent up to 45 min before firing — still a real, if smaller, FN exposure window. Confirmed still structurally blind to an ETL-only outage (watches `v_llm_bronze`, not ETL). | RETUNE-TIGHTER (see below) |
| 6 | `etl_pipeline_failure` | Any `status='failure'` row in `v_etl_bronze`, trailing 24h | N/A — zero-tolerance backstop. 1 failure row in 30 days (2026-09-03, pre-dates this detector's 2026-09-15 deploy). | By design does not see "stopped scheduling" — that is `etl_table_staleness`'s (missing, see above) job, not a gap in this detector's own scope. | WELL-CALIBRATED |
| 7 | `etl_pipeline_staleness` | `max(run_timestamp) < now() - 30 minutes`, global scalar | Since this detector's 2026-09-15 deploy: max gap 26.03 min, p99 19.48 min, **0 gaps over 30 min**. (9 gaps over 30 min exist in the full 30-day window, but all pre-date 2026-09-15 — 0 empirical FP in the detector's actual live period.) | Confirmed masked the 2026-09-19/20 weekend outage (documented above) — that gap is closed only once `etl_table_staleness` actually exists (it doesn't, see finding #2 above), so this exposure is live today. | RETUNE-TIGHTER, modest (see below); underlying gap still STRUCTURAL, blocked on finding #2 |
| 8 | `etl_table_staleness` | `minutes_since_last_run > 60` per table (documented, not implemented) | CANNOT ASSESS — no code exists to query. | CANNOT ASSESS as a live detector — it provides literally zero coverage today, worse than "not yet deployed": there is no CREATE TABLE cell to deploy. | STRUCTURAL GAP, NOT A THRESHOLD PROBLEM |
| 9 | `etl_run_slow` | MAD-based bound on `duration_seconds`, rolled up to `(task_name, window_start)`, `>=3` occurrences/hour, **CRITICAL** (promoted 2026-09-21, local only) | Fired far more often than the FP/FN bias review's "5 times in a week" characterization: **11 distinct `(task_name, window_start)` correlated events in the last 14 days** (2026-09-08, 09-10 ×4, 09-14, 09-15 ×3, 09-20), i.e. roughly every 1.3 days, not weekly. Each event is genuinely correlated (7/7 or more tables under the same task moving together) — not noise, but a materially higher CRITICAL/Teams cadence than previously assessed. | None found — was WARN/log-only and invisible before promotion; now CRITICAL in local code, but **still WARN in the deployed pipeline** since the promotion isn't deployed (finding #1). | NOTIFICATION-PATH GAP (in the deployed pipeline, right now) / open volume question once redeployed (see below) |
| 10 | `capability_silence_ceiling` | `sev2-insights` only, `hours_since_last_call > 120` (tightened from 168h 2026-09-21, local only) | 0/990 gaps ever exceeded 120h for `sev2-insights` (full history); worst gap ever 102.5h — identical 0-FP rate as 168h, confirmed again this run. `sev2-insights` is **currently 66.9h silent and climbing** (last call 2026-09-18 22:23 UTC), still well under 120h. | Would not fire on a repeat of the worst historical gap (102.5h) with ~17h/17% margin to spare — by design. Does not exist in the deployed environment at all (no `silence_ceiling_hours` column, no table) — 100% FN exposure live today regardless of the local threshold's soundness. | ALREADY AT THE FLOOR (threshold itself) / STRUCTURAL-GAP-VIA-NONDEPLOYMENT (live impact) |
| 11 | `capability_silence` | Per-capability grace: `saa-display` 2h, `situational-awareness` 2h, `summary` 36h; `sev2-insights` excluded (`silence_grace_hours IS NULL`) | 0 rows currently for the 3 capabilities it covers (`saa-display`/`situational-awareness` last call 0.0h ago, `summary` 7.0h ago vs 36h grace) — comfortable margin, 0 empirical FP. | Confirmed structural exclusion of `sev2-insights` stands; see "standing gaps revisited" below — its only mitigation (`capability_silence_ceiling`) is not deployed. | GAP (mitigation exists in code, not in prod — see below) |
| 12 | `shift_context_missing` | Any blank `shift_type`/`batch_nbr`/null `shift_date` among active capabilities, trailing 7d | 0 blank/null fields across all 4 active capabilities, 7d (1370/1370/28/306 rows checked, all fully populated) — 0 empirical FP, but also 0 evidence of value: this has never fired. | CANNOT ASSESS — unlike `etl_run_slow`'s proven-real signal before its promotion, this WARN has no track record of catching anything; promoting it on the same "WARN-forever is itself a FN" logic isn't supported by evidence the way `etl_run_slow`'s was. | NOTIFICATION-PATH GAP in shape, but weak evidence for it — open question, not a recommendation (see below) |
| 13 | `write_lag_anomalies` | `write_lag_s > greatest(MAD-based bound, 300s)`, WARN, **paused** | `write_lag_s` is exactly 0 for all 10,701 rows across all 4 active capabilities, 30 days — unchanged since the prior audit. No threshold change can produce a false positive OR catch a true one against an always-zero field. | Cannot fire on real lag by construction if the field is never wired up — this is the known instrumentation question, still open, still paused. | STRUCTURAL GAP, NOT A THRESHOLD PROBLEM — user's pause takes precedence (see below) |

### #3 `blank_output` — candidate to close the residual 1-call-window edge case
Query basis: `summary`'s 30-day call history clusters in pairs roughly every 12 hours (checked via
rolling-6h-window counts, max 12 calls/window, but the underlying hourly pattern is bursts of 2).
The low-volume bypass requires `total_calls >= 2` in the 6h window to have any denominator to
judge a 100%-blank rate against — a window with exactly 1 call for a low-traffic capability still
can't be judged by the current logic. Since `blank_count` has been 0 for every capability, every
window, all 30 days, **lowering the bypass floor to `total_calls >= 1 AND blank_count = 1` costs
zero additional empirical false positives** (identical 0-observed-blank rate at either floor) while
closing the single-call case entirely — the same standard used for the `capability_silence_ceiling`
168h→120h decision. Proposed: extend the low-volume bypass to `1 <= total < 10 AND blank_count =
total` (currently `2 <= total < 10`). Open question awaiting sign-off — no code changed.

### #5 `pipeline_heartbeat` — retune candidate given wide real-world margin
Query basis: max gap across all `v_llm_bronze` capabilities over 30 days is 16.27 min; 0 gaps
exceed 20, 25, or 30 minutes. The 45min threshold set 2026-09-21 already reflected a real fix, but
leaves ~1.75x the slack the 20-minute figure would (45min ÷ 16.27min ≈ 2.8x vs. 20min ÷ 16.27min
≈ 1.2x). Proposed: tighten to **25 minutes** — still 0/30d empirical false positives (identical to
45min), roughly 1.5x the real max gap for comfortable margin against normal jitter, but cuts the
false-negative exposure window by ~44% (45min → 25min) versus the already-improved 45min. Open
question awaiting sign-off — no code changed.

### #7 `etl_pipeline_staleness` — modest retune, smaller headroom than other candidates
Query basis: since this detector's 2026-09-15 deploy, max real gap is 26.03 min (p99 19.48 min),
vs. the current 30min threshold — only ~13% margin, materially tighter than `pipeline_heartbeat`'s
~2.8x. A candidate tighten to **28 minutes** would still show 0/empirical false positives (26.03min
< 28min) and shave a modest ~7% off the false-negative exposure window, but the margin at 28min
(7%) is thin enough that a single slightly-slower-than-usual ETL cycle could tip it into a real
false positive that 30min currently absorbs comfortably. Given the bias explicitly rules out
trading a large/uncertain FP increase for a smaller FN win, this is presented as a marginal,
optional tighten rather than a clear recommendation — flagging the tradeoff explicitly per the
task's own instruction, not defaulting to it. Open question awaiting sign-off — no code changed.

### #9 `etl_run_slow` — real firing rate is higher than previously documented, and it's promoted-but-undeployed
Query basis: re-running the `(task_name, window_start)` rollup over the last 14 days (rather than
the FP/FN bias review's 7-day window) finds **11 distinct correlated events**, not the "5 times in
a week" the prior review found — this is the same real, correlated, fleet-wide signal, just more
frequent than initially characterized. At CRITICAL/Teams cadence, ~11 events per 14 days is roughly
one page every 1.3 days for what has consistently been the same recurring ETL-fleet slowdown
pattern (all-tables-under-one-task moving together). This is not itself a false-positive — every
firing has been a genuine correlated multi-table event — but it is worth surfacing as a volume
question distinct from the "should this be CRITICAL at all" question already answered ("yes,
promote it") 2026-09-21: **is ~1 CRITICAL Teams card every 1.3 days for a recurring-but-real
condition the intended paging cadence**, or does this belong on a lower-friction channel (a daily
digest) despite being CRITICAL-worthy in severity? This is exactly the "large FP increase" caution
the task asked to flag rather than default into — the fix already shipped (promotion to CRITICAL)
is not in question, only its notification cadence once deployed. More urgently: this promotion,
like every other 2026-09-21 fix, **is not deployed** — the live pipeline right now still treats
these 11 events as WARN/invisible-to-Teams, which is a live, ongoing false-negative-by-omission of
a proven-real signal. Open questions awaiting sign-off — no code changed.

### #10/#11 Standing gap revisited: `capability_silence`'s `sev2-insights` exclusion, given `capability_silence_ceiling`'s real deployment status
The mitigation (`capability_silence_ceiling`, 120h) is sound as a *threshold* — 0/990 empirical
false positives against `sev2-insights`'s full gap history, tightened to the evidence floor with a
deliberate ~17h margin over the single worst-ever gap (102.5h) so as not to trip on a benign repeat
of that exact outlier. Under the user's precise bias ("minimal, not necessarily zero, false-negative
chance"), 120h is a reasonable point on that curve — going tighter (e.g. ~105-110h) would leave
almost no margin against a recurrence of the one historical outlier, converting "slight FP increase"
into "any repeat of past-normal behavior trips it," which the task's own framing says is NOT what
this bias asks for. **The live answer to "does this gap still apply" is yes, unchanged, and for a
different reason than the threshold**: `capability_silence_ceiling` does not exist in the deployed
environment (no `silence_ceiling_hours` column, no table) — `sev2-insights`, currently 66.9h silent
and climbing, has **zero actual detection coverage in production today**, identical to the state
documented in the original 2026-09-21 audit before any of this session's fixes were built. No
threshold number changes this; only deploying the already-signed-off local code does. Open question
awaiting sign-off (the deploy itself, not a new threshold) — no code changed.

### #13 Standing gap revisited: `write_lag_anomalies` pause vs. the "slight FP for minimal FN" bias
Query basis: `write_lag_s` remains exactly 0 for all 10,701 active-capability rows in the last 30
days — unchanged from the FP/FN bias review. The bias's general push toward more sensitivity does
not change the calculus here, because there is no threshold to loosen: the field itself carries no
signal (0 variance, 0 nonzero observations ever), so no floor from 300s down to 1s would produce a
single additional true positive — only additional false positives if the field is later found to be
wired up incorrectly (e.g. a unit or timezone bug that makes benign values look nonzero). Shipping
any interim floor now would be tightening against noise with no evidence it improves recall, which
is the specific "unbounded/unjustified FP increase" the task asks to flag rather than recommend.
**The user's standing pause instruction takes precedence and is not in tension with this bias** —
pausing an instrumentation question is not the same as accepting a false negative; it's declining to
tune a detector whose input signal cannot currently distinguish "no lag" from "lag is unmeasured."
The already-proposed `write_lag_instrumentation_check` (FP/FN bias review, "Candidate new
detectors" #1) remains the correct next step once the SME is available, not a threshold change to
this detector. No action taken, consistent with the pause.
