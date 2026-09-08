# Handoff — ISH Agent Observability: v1.1 Hardening

**Date:** 2026-09-08
**Supersedes:** prior HANDOFF.md (Phase A/B item write-ups — Phase A+B now fully done)

---

## TL;DR

Phase A, Phase B, and Phase C of v1.1 hardening are **all fully implemented and live** (code +
Databricks workspace + SQL warehouse). C.2 and C.9 — the last two open items — were finished
this session (see "Phase C — status" below for exact detail). All 5 modified notebooks
(`ptof_obs_setup_seed`, `ptof_obs_alert`, `ptof_obs_verification`,
`ptof_obs_behavioral_correlation`, `ptof_obs_latency_detection`) are synced to the Databricks
workspace as of this session (confirmed via individual `databricks workspace import` exit codes,
not just the batch loop). **Nothing has been committed to git yet.**

**What's left, for a fresh session:** run the fresh-scan job, run the verification job, spot-check
key outcomes via direct SQL, then get the user's explicit go-ahead before the single final commit.
See "Immediate next steps" below — steps 1–3 are done, resume at step 4.

**Working style for this effort (carry forward):** every item has a Keep/Remove/Defer decision.
Present the tradeoffs, wait for the user's call, THEN implement. The user has been doing ALL
implementation across phases first, then wants ONE final sync → fresh scan → verification →
commit at the end, rather than per-phase commits.

---

## Phase A — DONE (committed in `fdbb499`/`8ebdbf1`, verified)
A.1–A.5 all implemented, synced, and previously verified via a Databricks fresh-scan +
verification run (all SUCCESS) plus spot-check SQL. See prior commits for detail; not re-derived
here since git history is authoritative.

## Phase B — DONE (code + live DB, all 5 notebooks re-synced this session)
- **B.1**: `nightly_baseline_staleness` promoted to CRITICAL in `ptof_obs_alert.ipynb`.
- **B.2**: findings-table liveness checks added to `ptof_obs_verification.ipynb` for
  `latency_anomaly_findings`, `transport_violation_signatures`, `blank_output_findings`,
  `handover_delivery_rate`, `capability_error_rate_findings` (that last one has since been
  replaced by `capability_error_rate_alert` — see C.2 follow-on below).
- **B.3**: `rapid_human_correction` redesigned (join on `batch_nbr` alone, 72h window) and wired
  to CRITICAL alerting. Live-tested against the warehouse: correctly returns 0 rows given
  current dev-data staleness.
- **B.4**: 6 `threshold_basis` documentation rows added (prompt_size_drift 2x multiplier,
  latency baseline n_samples>=30/span_days>=6, response baseline min rows>=20, latency anomaly
  findings min count>=2, hallucination similarity cross-check <0.80). Confirmed live in DB.
- **B.5**: `silence_grace_hours`/`expected_min_daily` calibrated for all 4 active capabilities
  from real `v_llm_bronze` cadence: `saa_insight` (144h/5), `sev2_insight` (144h/1) — these two
  share the same 107h quiet period 2026-08-31→09-04, confirmed a real shared gap not
  per-capability noise — `summary` (36h/1), `watchout_narratives` (NULL/0 — only 1 call ever,
  deliberately left NULL to skip silence monitoring rather than invent a number from n=1).
  Confirmed live via SELECT.

---

## Databricks references (unchanged)

| Resource | ID / Path |
|----------|-----------|
| Fresh scan job | `585607309385820` |
| Nightly baseline job | `428356310089497` |
| Workspace root | `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/` |
| Verification spec | `run_config.json` (repo root) |
| Databricks profile | `"Tyler Kei"` (use `-p "Tyler Kei"` with CLI) |
| SQL warehouse | `cd6c1145a46bec44` (use `databricks api post /api/2.0/sql/statements --json @file.json -p "Tyler Kei"`) |

### Sync command (all notebooks)
```bash
for nb in ptof_obs_setup_seed ptof_obs_hallucination_detection ptof_obs_latency_detection \
          ptof_obs_mal_output ptof_obs_alert ptof_obs_verification ptof_obs_nightly_baseline \
          ptof_obs_behavioral_correlation ptof_obs_bronze_projection ptof_obs_weekly_runtime_digest; do
  databricks workspace import --file ${nb}.ipynb --format JUPYTER --overwrite \
    /Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/${nb} -p "Tyler Kei"
done
```

### Run jobs
```bash
databricks jobs run-now 428356310089497 -p "Tyler Kei" --no-wait   # nightly baseline (if baseline schema changed)
databricks jobs run-now 585607309385820 -p "Tyler Kei" --no-wait   # fresh scan
databricks api post /api/2.0/jobs/runs/submit --json @run_config.json -p "Tyler Kei"   # verification
```

---

## Phase C — status (all 10 original items + 1 follow-on discovered mid-work)

| # | Item | Decision | Status |
|---|------|----------|--------|
| C.1 | Add `capability_registry` JOIN to `behavioral_correlation` | Keep | **Moot / already satisfied** — `rapid_human_correction`'s `ai_publish` CTE already joins `capability_registry WHERE active = true` (added in B.3). The other two write cells (`handover_delivery_failures`, `handover_delivery_rate`) query `v_ish_bronze`, which has no `capability` column — nothing to scope. No code change made. |
| C.2 | Add verification checks for 9 unverified alert detectors | Keep | **DONE.** Re-derived the missing-checks list from scratch (grepped `INCIDENT_SOURCES` in `ptof_obs_alert.ipynb` against every `record(...)` call in `ptof_obs_verification.ipynb`) rather than trusting the prior session's candidate list. Confirmed only 2 of the 14 detectors genuinely lacked coverage: `pipeline_heartbeat` and `nightly_baseline_staleness` (the candidates `hallucination_high`, `schema_field_missing`, `rapid_human_correction` were already covered via P1.5/P1.6, P1.7, P1.8 respectively). Added two new WARN-level "H ..." checks to `ptof_obs_verification.ipynb`'s Architecture-hardening section, following the existing pattern: (1) `v_llm_bronze` has rows in the last 2h, (2) `capability_latency_baseline.computed_at` is within 36h for every capability. WARN not FAIL, matching the section's existing tolerance for dev-data sparseness. |
| C.3 | Make bronze `is_blank_output`/`is_credential_fastfail` dynamic | **Defer** | No action — deferred per user, 4-capability scope is stable. |
| C.4 | Make verification counts dynamic (not hardcoded 11/22) | Keep | **DONE.** `P1.2 registry has 11 rows` (hardcoded) replaced with `P1.2 registry has no duplicate capability rows` (`len(rows) == len(caps)`, self-adjusting). The `auth >= 22` check was intentionally left alone — it's a monotonic non-decreasing floor for regression detection, not a rescope-friction hardcode; making it dynamic would defeat its purpose. |
| C.5 | Remove orphaned `handover_delivery_rate_findings` table | Remove | **DONE.** Cell removed from `ptof_obs_behavioral_correlation.ipynb`, markdown updated, `DROP TABLE IF EXISTS` added to `ptof_obs_setup_seed.ipynb` and run live (confirmed dropped). |
| C.6 | Wire `behavioral_event_clusters` to alerting or document as diagnostic | Document as diagnostic | **Moot** — `behavioral_event_clusters` does not exist anywhere in the codebase or git history. It was apparently only ever planned, never built. No action needed; this row is stale in the original handoff and should not resurface as a to-do (this table now documents that explicitly). |
| C.7 | Add BACKTRACK for `runtime_violation` | **Defer** | No action — only needed when/if `runtime_violation` is promoted to CRITICAL (currently WARN). |
| C.8 | Add prod allowlist rows for `saa_insight`/`sev2_insight` | **Defer** | No action — until these capabilities actually deploy to prod. |
| C.9 | Clean ~25 stale `dsa_*` comment references | Keep | **DONE.** Audited every `dsa_*` occurrence across all `.ipynb` files (grep, then filtered out SQL literals/data values which are legitimately still needed for deactivation logic — those were left alone). Found and fixed 5 genuinely stale spots, all following the same pattern: they described `rapid_human_correction` as permanently "parked" due to `dsa_*`'s missing shift/batch identity, or described `shift_context_missing` as blocked "until dsa_* populates" its fields — both stale because (a) B.3 redesigned and un-parked `rapid_human_correction` (confirmed 2 real matches for `summary`), and (b) `shift_context_missing`'s query already joins `capability_registry WHERE active = true`, so deactivated `dsa_*` rows never reach it at all — a hit today would mean an active SAA/ISH capability regressed, not the old dsa_* gap. Fixed in: `ptof_obs_alert.ipynb` (markdown "Known-expected states" + the `shift_context_missing` check() comment), `ptof_obs_latency_detection.ipynb` (the `shift_context_missing` table's build comment), `ptof_obs_verification.ipynb` (P1.8 comment block + the final "Expected WARNs" markdown bullet). Everything else referencing `dsa_*` (calibration history, SQL literals for deactivation, threshold rationale) was left untouched — genuinely still accurate. |
| C.10 | Mark 2 stale `threshold_basis` entries deprecated | Mark deprecated | **DONE.** `capability_silence dsa_copilot` and `capability_silence dsa_optimize` rows updated to `status = 'deprecated'` via new cell in `ptof_obs_setup_seed.ipynb`, run live and confirmed. |
| **C.11 (new, found mid-work)** | `capability_error_rate_findings` orphaned the same way as C.5's table | Remove (same as C.5) | **DONE.** Discovered while investigating C.2: `ptof_obs_alert.ipynb`'s `capability_error_rate_sustained` detector has always read `capability_error_rate_alert` directly, never the signature-keyed `capability_error_rate_findings` wrapper built in `ptof_obs_latency_detection.ipynb`. Removed the build cell, updated markdown, fixed the 2 verification checks in `ptof_obs_verification.ipynb` to check `capability_error_rate_alert` instead, added `DROP TABLE IF EXISTS` to `ptof_obs_setup_seed.ipynb`, ran live and confirmed dropped. |

---

## Immediate next steps for a fresh session

1. **Sync is confirmed current** (done this session) — all 5 modified notebooks
   (`ptof_obs_setup_seed`, `ptof_obs_alert`, `ptof_obs_verification`,
   `ptof_obs_behavioral_correlation`, `ptof_obs_latency_detection`) were re-synced individually
   (not just via the batch loop) with confirmed exit code 0 each. `git status` will show these
   5 files modified plus this HANDOFF.md — nothing has been committed yet.
2. ~~Finish C.2~~ — **done this session**, see table above.
3. ~~Do C.9~~ — **done this session**, see table above.
4. **Final phase-C completion** — resume here. **Fresh scan job is already running as of this
   session's end** — run_id `982452007261806` (started 2026-09-08, was `RUNNING` at handoff
   time). Poll with `databricks jobs get-run 982452007261806 -p "Tyler Kei"` until
   `state.life_cycle_state` is `TERMINATED` and check `state.result_state` is `SUCCESS`. Then:
   run verification (`databricks api post /api/2.0/jobs/runs/submit --json
   @run_config.json -p "Tyler Kei"` — no-wait not needed, this one's fast), spot-check key
   outcomes via direct SQL (verification `print()` output is not readable via API — see Key
   Lessons below). Specifically worth spot-checking given this session's changes: the two new
   `H pipeline_heartbeat` / `H nightly_baseline_staleness` checks actually PASS/WARN as expected
   (not FAIL), and that `shift_context_missing` is still empty or WARN-only for the 4 active
   SAA/ISH capabilities (confirms the C.9 comment fix's claim about the `active = true` join is
   accurate).
5. **Commit everything** — get explicit go-ahead from the user before committing/pushing, per
   this project's working style. Suggested commit message shape:
   ```
   v1.1 hardening: Phase B complete (staleness alerting, liveness checks, rapid_human_correction
   redesign, threshold documentation, silence calibration), Phase C cleanup (dynamic verification
   counts, 2 orphaned tables removed, deprecated stale thresholds)
   ```

---

## Key Lessons (carried forward, unchanged from prior handoff)

1. **Databricks workspace sync is manual** — no git-folder integration. Every local `.ipynb`
   edit must be pushed with `databricks workspace import` before running a job.
2. **Cluster spec:** Use `is_single_node: true` + `kind: CLASSIC_PREVIEW`, NOT `num_workers: 0`
   alone — the latter hangs at 0/1 forever.
3. **NotebookEdit is risky for large .ipynb cells** — for the giant single-cell notebooks
   (`ptof_obs_verification.ipynb`), this session used a Python json.load → string splice →
   json.dump approach instead of NotebookEdit for targeted mid-cell edits. NotebookEdit itself
   worked fine for whole-cell replace/insert/delete on smaller, multi-cell notebooks.
4. **Notebook cell convention:** Verification notebook has a single massive code cell with
   `# COMMAND ----------` markers. New code must be spliced inline via string replace, not
   NotebookEdit, when editing mid-cell (not replacing the whole cell).
5. **Verification output not in API:** `print()` output is not captured by `get-run-output`.
   View in the Databricks run page UI, or run the same SQL directly via the warehouse and
   spot-check manually.
6. **Delta schema evolution:** `CREATE TABLE IF NOT EXISTS` + `INSERT OVERWRITE` does NOT
   handle new columns — use `CREATE OR REPLACE TABLE ... AS SELECT` instead.
7. **Qualify all column refs in multi-table JOINs** — Spark's `AMBIGUOUS_REFERENCE` error
   rejects unqualified columns.
8. **`databricks jobs run-now` without `--no-wait`** blocks until the run completes. Use
   `--no-wait` and poll with `databricks jobs get-run <run_id>` for long jobs.
9. **Multi-row `threshold_basis` inserts must use `LEFT ANTI JOIN`, not a correlated
   `WHERE NOT EXISTS (... = t.check_name)` over a multi-row `VALUES`** — the latter doesn't
   resolve per-row against `SELECT *`. Cell 11's `LEFT ANTI JOIN` pattern is the one to copy for
   any future multi-row idempotent insert into a table keyed on a natural key.
10. **Before removing/trusting a table as "orphaned," `grep -rn` the exact table name across
    every `.ipynb` in the repo** — this session found two real orphaned tables
    (`handover_delivery_rate_findings`, `capability_error_rate_findings`) this way, both cases
    where a notebook's own markdown/comments claimed a downstream reader existed that, in fact,
    read a *different* table instead. Don't trust in-notebook documentation about what reads
    what without verifying via grep.
