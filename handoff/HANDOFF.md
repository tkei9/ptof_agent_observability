# Handoff — ISH Agent Observability: Verification Gate

**Date:** 2026-09-04
**Supersedes:** `HANDOFF_ARCH_HARDENING.md`, `HANDOFF_ARCH_HARDENING_VERIFICATION.md`, prior `HANDOFF.md`

---

## TL;DR

The SAA/ISH rescope is committed (`8ebdbf1`). Architecture hardening (Phases 1–6) plus a
scope-alignment smoke test are implemented in the working tree across 7 notebooks. All
notebooks are modified but **not committed**. The nightly baseline job was kicked off with the
hardened code. What remains: sync the latest local edits, run the sev2_insight allowlist seed,
run the fresh scan, run verification, commit.

---

## Completed Work

### Rescope (commit `8ebdbf1`, pushed)

Narrowed from ~11 capabilities to **4 active SAA/ISH capabilities**: `saa_insight`,
`sev2_insight`, `summary`, `watchout_narratives`. All `dsa_*` and `probe` set `active=false`.
Added `is_groundable` column to `capability_registry`. All detectors filter via
`JOIN capability_registry WHERE active = true`.

### Architecture Hardening (Phases 1–6, in working tree)

| Phase | Notebook | Key changes |
|-------|----------|-------------|
| 1 | `hallucination_detection` | Deleted duplicate MERGE cell, similarity cross-check on Rule 3 (`< 0.80`), time-bounded all scans to 7d, `_scoring_ts` watermark gap fix, `GREATEST(watermark, 7d)` window |
| 2 | `latency_detection` | `capability_registry` JOIN on `latency_anomalies`, `HAVING count(*) >= 2` on findings, GxP error rate threshold `0.8 → 0.5`, 30d bound on `credential_fastfail_daily`, NEW `capability_outage_findings`, NEW `prompt_size_drift` |
| 3 | `mal_output` | 7d bound on `blank_output_incidents`, replaced inline 30d baseline with `response_field_baseline` read |
| 4 | `alert` | `nightly_baseline_staleness` WARN check, `blank_output` aligned to `blank_output_findings`, `capability_outage` full wiring |
| 5 | `setup_seed` | Per-row `LEFT ANTI JOIN` idempotency fix (cells 11/13/14), `capability_error_rate_sustained` threshold provenance |
| 6 | `nightly_baseline` | NEW `response_field_baseline` table, `p95_prompt_chars`/`p95_response_chars` on latency baseline, `computed_at` on schema baseline |

### Scope-Alignment Improvements (2026-09-04 session, in working tree)

| # | Change | Notebook(s) | Why |
|---|--------|-------------|-----|
| 1 | `sev2_insight` added to `runtime_allowlist` | `setup_seed` | Gap in rescope: every sev2_insight call was flagged as a transport violation |
| 2 | `capability_registry` JOIN added to 4 unscoped cells | `hallucination` (cell 4), `latency` (cells 3, 5, 12) | Defense-in-depth: stopped regex/queries running on inactive dsa_* traffic |
| 3 | `prompt_size_drift` promoted WARN → CRITICAL with full alerting | `latency` (cell 12), `alert` (cells 4, 6, 19) | Added `finding_signature` to table, wired INCIDENT_SOURCES + BACKTRACK + DETECTOR_META + check upgraded to `gt0` |
| 4 | 3 stale-assertion FAILs fixed + `finding_signature` check | `verification` | Allowlist: hardcoded count → active-capability coverage. Auth: `==22` → `>=22`. DSA uncovered: split in-scope FAIL vs out-of-scope WARN |

---

## What Remains (in order)

### Step 1: Wait for nightly baseline job

Job `428356310089497` was kicked off 2026-09-04. It must complete before the fresh scan.
Creates `response_field_baseline`, adds `p95_prompt_chars`/`p95_response_chars` and
`computed_at` columns.

Check status: `databricks jobs get-run <run_id>`

### Step 2: Sync ALL modified notebooks to Databricks

5 notebooks were modified after the previous sync. ALL must be re-synced:

```bash
for nb in ptof_obs_setup_seed ptof_obs_hallucination_detection ptof_obs_latency_detection ptof_obs_alert ptof_obs_verification; do
  databricks workspace import --file ${nb}.ipynb --format JUPYTER --overwrite \
    /Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/${nb}
done
```

The `nightly_baseline` and `mal_output` notebooks were synced in the prior session and have NOT
been modified since — no re-sync needed for those two.

### Step 3: Run the `sev2_insight` allowlist cell

Open `ptof_obs_setup_seed` in Databricks and run the **last cell** (the new
`sev2_insight` runtime_allowlist INSERT). This must happen BEFORE the fresh scan, or every
`sev2_insight` call shows as a transport violation.

This is an **idempotent** INSERT (LEFT ANTI JOIN guard). Safe to re-run.

### Step 4: Run fresh scan job

```bash
databricks jobs run-now 585607309385820
```

This exercises all 6 detection notebooks with the hardened + scope-aligned code.

### Step 5: Run verification

```bash
databricks api post /api/2.0/jobs/runs/submit --json @run_config.json
```

`run_config.json` is in the repo root with the correct cluster spec (`is_single_node: true`,
`kind: CLASSIC_PREVIEW`). Do NOT use `num_workers: 0` without those flags.

**Note:** Verification output is via `print()`, not `dbutils.notebook.exit()`. The
`notebook_output` field in `get-run-output` will be empty. View results in the Databricks run
page UI, or paste the Summary cell output.

### Step 6: Review results

The 3 previously-known stale FAILs should now pass. Watch for:
- Any new FAILs from the scope-filter or prompt_size_drift changes
- The `P3` structural completeness checks should pass (prompt_size_drift now has DETECTOR_META +
  BACKTRACK entries)
- `H prompt_size_drift exists with expected columns` now expects `finding_signature`

### Step 7: Commit (get explicit go-ahead first)

Single commit for all hardening + scope-alignment work:
```
Architecture hardening: similarity cross-check, unbounded scan fixes, GxP error threshold,
prompt_size_drift alerting, sev2_insight allowlist, scope-filter defense-in-depth

Co-Authored-By: Lilly Code <lillycode@lilly.com>
```

Files to commit (all modified):
- `ptof_obs_hallucination_detection.ipynb`
- `ptof_obs_latency_detection.ipynb`
- `ptof_obs_mal_output.ipynb`
- `ptof_obs_nightly_baseline.ipynb`
- `ptof_obs_setup_seed.ipynb`
- `ptof_obs_alert.ipynb`
- `ptof_obs_verification.ipynb`
- `run_config.json` (verification job spec — keep or .gitignore, user's choice)

Clean up old handoff files after commit:
- `handoff/HANDOFF_ARCH_HARDENING.md` → delete (merged here)
- `handoff/HANDOFF_ARCH_HARDENING_VERIFICATION.md` → delete (merged here)

---

## Key Lessons

1. **Databricks workspace sync is manual** — no git-folder integration. Every local `.ipynb`
   edit must be pushed with `databricks workspace import` before running a job.
2. **Cluster spec:** Use `is_single_node: true` + `kind: CLASSIC_PREVIEW`, NOT `num_workers: 0`
   alone — the latter hangs at 0/1 forever.
3. **NotebookEdit is NOT safe for large .ipynb edits** — use the Python json.load → anchor
   search → splice → json.dump approach for targeted changes.
4. **Notebook cell convention:** All code in the verification notebook lives in a single massive
   code cell with `# COMMAND ----------` markers as virtual separators. New code must be spliced
   inline, not added as trailing cells.
5. **Verification output not in API:** `print()` output is not captured by `get-run-output`.
   View in the Databricks run page UI.

---

## v1.1 Candidates (post-commit)

| Detector | Effort | Why deferred |
|----------|--------|--------------|
| `runtime_violation` promotion to CRITICAL | Small-Medium | Reseed allowlist from observed production traffic for all 4 capabilities |
| `rapid_human_correction` | Medium | Un-park: DSA null-identity reason is gone, SAA/ISH capabilities have shift/batch identity |
| `output_delivery_gap` | Medium | Capability-to-output_type mapping is 4 rows; detects "generated but never delivered" |
| `latency_failures` findings table | Medium | Bimodal failure data + 4 capabilities makes a volume-floor threshold tractable |
| Stale comment cleanup | Small | ~25 dsa_* references in comments across all notebooks; cosmetic only |
| `silence_grace_hours` calibration | Small | Currently NULL for all 4 active capabilities |
| `response_schema_baseline` deprecation review | Small | May be vestigial — `response_schema_drift` reads `response_field_baseline`, not this table |
| Per-capability baseline coverage assertions | Small | Verification only checks `sev2_insight` baseline coverage, not the other 3 |
| Threshold re-calibration | Medium | Hallucination thresholds (0.75, 0.80) calibrated against old broader capability set |
