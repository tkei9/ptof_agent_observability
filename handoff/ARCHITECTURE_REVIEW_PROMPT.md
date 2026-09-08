# Architecture review prompt — paste this into a fresh chat

---

You are acting as a senior data architect doing an independent, critical review of an existing
LLM/agent observability pipeline. This is a review task, not an implementation task — do not
edit any files unless I explicitly ask you to after seeing your findings.

## Where things live

- Repo: `/Users/L141230/Downloads/agent_obs` (git repo, branch `main`).
- Databricks workspace: `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo/`
  (CLI profile `"Tyler Kei"`, use `-p "Tyler Kei"`). SQL warehouse `cd6c1145a46bec44`
  (`databricks api post /api/2.0/sql/statements --json @file.json -p "Tyler Kei"`). All tables
  live in catalog/schema `mq_gmdf_dev.oil_obs` (dev only — no prod deployment yet). The two raw
  upstream tables this pipeline does NOT own are in `mq_gmdf_dev.oil`
  (`ptof_primary__ai_llm_audit_log`, `ptof_ish_audit`).
- Ignore any `.md` files in the repo other than this prompt — they are either work logs (stale
  by design once the work they describe is done) or empty placeholders. Treat the notebook
  source code itself as the only source of truth.

## Core architecture (stable — read the notebooks to confirm/expand this, don't take it purely on faith)

This is the **ISH agent observability pipeline**: it watches LLM calls made by a small set of
named "capabilities" (agent use-cases) plus a related human/ISH change-audit log, and raises
Teams-notified incidents when agent behavior looks wrong.

**Pipeline stages, in execution order (Databricks job `obs_fresh_scan`):**

1. `01_bronze_projections` — `ptof_obs_bronze_projection.ipynb`. Entry point. Projects the two
   raw oil-layer audit tables into two durable `CREATE OR REPLACE VIEW`s (not materialized
   tables, so they always reflect current base-table contents and stay safe to backtrack an
   incident against days later):
   - `v_llm_bronze` — one row per LLM call. Passes through call identity, shift/batch context,
     capability/model_config/transport, success, error_msg, latency_ms, called_at, ingestion_ts,
     prompts/response. Adds derived columns computed once here so every downstream detector
     agrees on their meaning: `write_lag_s` (ingestion_ts − called_at), `is_credential_fastfail`
     (specific known misconfigured-demo-model-config failure signature), `is_blank_output`
     (call succeeded transport-wise but returned an empty/null/empty-JSON response — the core
     signal for the CRITICAL blank-output detector).
   - `v_ish_bronze` — one row per ISH entity create/update/delete. Parses `before_json`/
     `after_json` once via `try_parse_json`, normalizes historically-inconsistent field names
     (`shift_date_norm`, `shift_type_norm`, `batch_id_norm`) so behavioral correlation can join
     ISH activity to `v_llm_bronze` calls without per-caller guesswork. Adds
     `is_email_disabled_gate` to distinguish "handover email deliberately not sent (feature flag
     off)" from "system tried and failed to send" (the latter is the real failure mode).
   - Every other notebook reads these two views, never the raw oil tables directly.

2. Four detector notebooks run in parallel, all reading `v_llm_bronze`/`v_ish_bronze` plus
   human-curated tables from setup/seed and nightly baseline:
   - `02_latency_detection` (`ptof_obs_latency_detection.ipynb`) — responsiveness regressions
     (call slower than the capability's own historical baseline) and sustained failure
     (capability/model_config failing hard for hours). Writes: `latency_anomalies` (raw),
     `latency_anomaly_findings` (aggregated, signature-keyed — what alert reads for CRITICAL
     `latency_anomaly`), `capability_error_rate_alert` (CRITICAL `capability_error_rate_sustained`
     reads this directly), `capability_outage_findings` (capability-level rollup, CRITICAL),
     `prompt_size_drift`. Also several WARN-tier / diagnostic-only tables not on the notify
     surface: `credential_fastfail_daily`, `write_lag_daily`, `latency_failures`,
     `capability_health`, `capability_silence`, `shift_context_missing`.
   - `03_malformed_output` (`ptof_obs_mal_output.ipynb`) — blank-output and transport-violation
     detection. Writes: `blank_output_incidents`, `blank_output_findings` (CRITICAL),
     `transport_violations`, `transport_violation_signatures` (checked against
     `runtime_allowlist`), `response_schema_drift`.
   - `04_hallucination_detection` (`ptof_obs_hallucination_detection.ipynb`) — faithfulness/
     grounding scoring. Uses an incremental watermark+MERGE pattern keyed off
     `_obs_watermark` (the only detector in the repo using this pattern; every other detector
     rescans a rolling window each run) so `faithfulness_scores` only scores new rows. Writes:
     `hallucination_verdicts`, `faithfulness_scores`, `hallucination_signal`.
   - `05_behavioral_correlation` (`ptof_obs_behavioral_correlation.ipynb`) — cross-signal
     correlation between LLM behavior and human/ISH activity. Writes: `ish_entity_dim`,
     `rapid_human_correction` (joins `capability_registry WHERE active = true`, keyed on
     `batch_nbr`, 72h window), `handover_delivery_failures`, `handover_delivery_rate`.

3. `06_alert` (`ptof_obs_alert.ipynb`) — the only notebook that evaluates detector conditions in
   Python (not just SQL) and decides what actually gets surfaced. Persists findings to
   `obs_incidents` (append-input, MERGE-deduped), posts an Adaptive Card to Teams, reports
   missing/stale upstream tables as `UNAVAILABLE` rather than crashing, and `raise`s **only** on
   `UNAVAILABLE` (so job task status means "is monitoring working", not "did it find something" —
   CRITICAL findings route to Teams/`obs_incidents` instead of failing the task). Notification
   policy: a CRITICAL incident notifies on first detection, then not again for 24h unless still
   unacknowledged (via `obs_incidents.notified_at`); WARN findings are never notified, only
   left visible in task output and `obs_incidents`.

**Human-curated reference tables** (all in `ptof_obs_setup_seed.ipynb`, no detection logic of
their own):
- `capability_registry` — one row per capability (`saa_insight`, `sev2_insight`, `summary`,
  `watchout_narratives` currently active; a few `dsa_*` legacy rows marked inactive/deprecated).
  Columns include `is_generative`, `is_gxp_relevant`, `expected_min_daily`, `required_fields`,
  `owner`, `active`, `silence_grace_hours`.
- `runtime_allowlist` — per (environment, capability): allowed transports/model_configs/
  scheduler_runs, checked by the transport-violation detector.
- `threshold_basis` — documents the threshold/basis/status for every detector's cutoff, so
  thresholds are self-documenting instead of magic numbers buried in SQL.
- `_obs_watermark` — incremental-processing bookmark, currently used only by
  `faithfulness_scores`.
- `obs_incidents` — the alert notebook's own persistence table (severity, first/last detected,
  detection_count, acknowledged_at, resolved_at, notified_at, signal_payload).
- `runtime_observed` — (confirm exact purpose/writer from the notebook; not fully traced above).

**Supporting notebooks, not part of the per-run detection path:**
- `ptof_obs_nightly_baseline.ipynb` — separate Databricks job, computes
  `capability_latency_baseline`, `response_schema_baseline`, `response_field_baseline` that the
  detectors above read.
- `ptof_obs_verification.ipynb` — post-run health-check notebook (one giant code cell with
  `# COMMAND ----------` markers), asserts things like "every alert detector's source table is
  reachable and non-stale," run separately from the two jobs above via a `run_config.json`
  multi-task submission.
- `ptof_obs_weekly_runtime_digest.ipynb` — weekly summary digest, separate cadence.

## What I want from you

**1. Full inventory.** For every detector/check/table-producing cell across all 10 notebooks,
give me: what it detects, what table(s) it reads/writes, its trigger condition and threshold,
its severity tier (CRITICAL / WARN / diagnostic-only), and whether it's actually wired into the
notify surface in `ptof_obs_alert.ipynb` or just console-visible. Organize this by pipeline
stage (per the ordering above), not by notebook, so I can see the full detection surface end to
end. Correct or fill in anything in the "Core architecture" section above that turns out to be
wrong or incomplete once you've read the actual code — that section is a starting map, not a
verified spec.

**2. End-to-end functional confirmation.** Trace data from `v_llm_bronze`/`v_ish_bronze` through
every detector to `obs_incidents`/Teams notification, and tell me plainly: does this pipeline
work end to end as built? Call out anything that's actually broken, silently disconnected (a
table nothing reads, or read by something other than what its own markdown claims), or
logically inconsistent — not stylistic nitpicks. Use `grep -rn <table_name>` across all
notebooks to verify claimed readers/writers rather than trusting a notebook's own markdown about
what reads what — this codebase has a documented history of markdown claiming a downstream
reader that, in fact, reads a different table (check for any current instances of that pattern).

**3. Simplification / over-engineering review.** This is the part I care most about. I'm a data
scientist, not a platform engineer, and I want this pipeline to be as interpretable as possible
without losing real detection value. Specifically:
   - Which tables, joins, or detectors exist but don't materially change what gets alerted or
     what an on-call person would do differently? Candidates to scrutinize: signature-keyed
     "findings" wrapper tables vs. the alert tables they're supposed to feed, WARN-tier
     diagnostic tables nothing acts on, dead/unused columns (e.g. check whether
     `expected_min_daily` in `capability_registry` is read anywhere), the incremental
     watermark+MERGE pattern in hallucination detection vs. a plain full rescan, and the
     idempotent-seed / `LEFT ANTI JOIN` patterns in `ptof_obs_setup_seed.ipynb`.
   - Where is complexity actually load-bearing (catches something a simpler version wouldn't) vs.
     defensive/speculative (guards against a scenario that can't currently happen given the
     4-active-capability, dev-only scope)?
   - Give me concrete "keep as-is" vs. "simplify, and here's the smaller version" calls, not just
     a list of concerns. If a simplification changes detection behavior (even for currently-empty
     dev data), flag that tradeoff explicitly rather than presenting it as free.

**4. Output format.** Structure your final answer as: (a) inventory table, (b) end-to-end
verdict with any concrete breakages found, (c) ranked simplification recommendations
(highest interpretability-gain-per-risk first), (d) anything you're unsure about or where you'd
want my input before recommending a change (e.g. whether `saa_insight`/`sev2_insight` are
heading to prod soon changes whether a currently-deferred prod-allowlist gap matters).

## Ground rules

- Read every notebook's actual cell source, not just markdown/doc cells.
- Query the live tables in `mq_gmdf_dev.oil_obs` via the SQL warehouse where it helps you judge
  whether a detector fires on real data or is dead weight (e.g. row counts, whether a table is
  ever non-empty).
- If you're unsure about intent behind a design choice, or need more context on how this
  pipeline is actually operated day-to-day (who watches Teams alerts, is there an on-call
  rotation, is this heading to prod), ask me before assuming — don't guess at operational
  context you can't derive from the code.
- Do not make any edits during this review. Recommendations only.
