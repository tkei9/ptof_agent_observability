# Phase 0 completion record + Phase 1 kickoff prompt

Written 2026-08-31 as a context handoff for continuing the remediation plan at
`/Users/L141230/.claude/plans/fuzzy-moseying-rivest.md` in a fresh chat.

## Phase 0 status: DONE, verified

All of Phase 0 ("Enabling infrastructure") is complete and verified against the
plan's own bar — real Repos, real secret, real end-to-end Teams POST confirmed.

### What's actually in place right now

1. **Git remote**: `https://github.com/tkei9/ptof_agent_observability.git`, branch `main`.
   Latest commit `21108f9` (rename to `ptof_` prefix). Prior commits: `cc5bfb3`
   (secrets-scope webhook migration), `a2ed3c5` (pre-remediation baseline snapshot).

2. **Databricks Repo (real, API-backed)**: created at
   `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo`, repo id
   `1666239020343210`, tracking `origin/main`. This is the copy every job now runs from.

   Notebook names inside it (post-rename):
   `ptof_obs_alert`, `ptof_obs_behavioral_correlation`, `ptof_obs_bronze_projection`,
   `ptof_obs_hallucination_detection`, `ptof_obs_latency_detection`, `ptof_obs_mal_output`,
   `ptof_obs_verification`, `ptof_obs_nightly_baseline`, `ptof_obs_setup_seed`,
   `ptof_obs_weekly_runtime_digest`, plus `ptof_observability.md`.

3. **Secrets scope**: `obs-alerting`, key `teams-webhook`. Both `ptof_obs_alert` and
   `ptof_obs_weekly_runtime_digest` read `WEBHOOK = dbutils.secrets.get(scope="obs-alerting",
   key="teams-webhook")`. The old `dbutils.widgets.text("alert_webhook", "")` plaintext
   widget is gone from both notebooks' code.

4. **Job definitions repointed** (user did this manually in the Databricks UI):
   - `obs_fresh_scan` (job id `585607309385820`) — all 6 tasks
     (`01_bronze_projections` … `06_alert`) now point at
     `.../ptof_agent_observability_repo/ptof_obs_*`. Confirmed via `databricks jobs get`.
   - `obs_weekly_runtime_digest` (job id `167611067157402`) — its task repointed the same way.
   - `obs_nightly_baseline` (job id `428356310089497`) — also repointed (confirmed
     2026-08-31, task `obs_nightly_baseline_` → `ptof_obs_nightly_baseline`).
   - The plaintext `alert_webhook` job parameter (which held the real Teams URL in
     cleartext) was stripped from both `obs_fresh_scan`'s and
     `obs_weekly_runtime_digest`'s `parameters` list via `databricks jobs reset`, and from
     the digest task's `base_parameters` override. Verified empty/absent afterward.

5. **Live end-to-end verification, done twice**:
   - `obs_weekly_runtime_digest` ran cleanly post-rewiring; `webhook_configured=True`,
     `runtime_observed` MERGE succeeded. (This run found nothing above the digest floor
     to send — not a bug, see note below — so it didn't exercise the actual POST.)
   - **Actual live POST confirmed** via the synthetic-incident procedure from
     `handoff/README.md` Step 5: inserted one `TEST-webhook-verify` CRITICAL row into
     `obs_incidents`, triggered the alert job, user confirmed "all the runs ran cleanly
     with the new github integration" (Teams card delivered). Synthetic row has been
     deleted (`DELETE FROM obs_incidents WHERE source_row_id LIKE 'TEST-%'` — confirmed
     0 rows remain as of this write).

### Loose ends NOT blocking Phase 0, but worth knowing about

- **Old, now-dead workspace folder**: `/Workspace/Users/tyler.kei@lilly.com/ish-agent-obs/`
  still exists with the pre-remediation code (plaintext `alert_webhook` widget, no git).
  No job points at it anymore, but it hasn't been deleted. Harmless dead weight; consider
  deleting once you're confident nothing references it, or leave it as an audit trail.
- **Orphaned bare-git folder**: `/Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability`
  (no trailing `_repo`) — a stray folder containing only a `.git` subdirectory with no
  checked-out files, `is_git_folder: false` in the Repos API, not a real Databricks Repo
  object. Predates this session's work, never connected to anything. Harmless; candidate
  for deletion but not urgent.
- **`obs_weekly_runtime_digest`'s 6-day suppression window**: every runtime-config
  violation above the digest floor already has a very recent `digest_reported_at`
  (set 2026-08-31), so the digest will legitimately report nothing new until that
  window lapses or a genuinely new violation appears. Not a defect — just don't be
  surprised if it stays quiet for a few days.

## Phase 1 — up next: close the silent detection gaps

Per the plan (`fuzzy-moseying-rivest.md`, lines 40–48), Phase 1 work is:

3. **`hallucination_verdicts` Layer 1** (`ptof_obs_hallucination_detection`, formerly
   `agent_obs_hallucination_detection.ipynb`): joins against a `verify_grounding`
   capability that doesn't exist, so the layer is always empty/NULL. Either label it
   explicitly `NOT YET ACTIVE` in `threshold_basis` (matching the existing
   `status`/`provisional`/`unvalidated` convention), or delete the dead layer and
   simplify `hallucination_signal`'s CASE logic to drop the `vg_verdict` branches.
   Decide (a) vs (b) with whoever owns the LLM-judge pipeline if possible; otherwise
   default to (a) since it's reversible and non-destructive.

4. **`rapid_human_correction`** (`ptof_obs_behavioral_correlation`): always 0 rows
   because `dsa_*` capabilities never populate `shift_date`/`shift_type`/`batch_nbr`.
   Add the same `threshold_basis` "parked" annotation. Add an assertion in
   `ptof_obs_verification` that fails loudly if this detector unexpectedly starts
   returning rows with a shape indicating the upstream fix hasn't landed, and a
   separate WARN-if-still-zero-after-some-date assertion with a revisit date noted
   in `threshold_basis.notes`.

5. **Registry drift**: `ptof_obs_verification` already shows 406 violations (dev
   backtest) vs. an expected 0, and 4 live capabilities missing from
   `capability_registry`: `dsa_copilot_step`, `dsa_session_summary`, `probe`,
   `saa_insight`. This needs real owner/transport/model_config values for these —
   check `handoff/README.md` for the existing "requires user input" flag on this,
   get the actual values from the user, apply via `ptof_obs_setup_seed`
   (mind Phase 3's idempotency rework hasn't happened yet — this notebook still
   has unconditional `%sql` cells as of now, so be careful about re-running it),
   then re-run verification and confirm 0 violations.

Remember the standing session rules for this engagement: explain what you're
about to do and why (pulling from the plan's own "What this adds" reasoning)
transparently as you go, not as an end-of-task summary; get explicit confirmation
before anything not directly doable yourself (Databricks UI actions, mutating
shared production tables/rows, anything the permission classifier flags); and stop
after Phase 1 is verified rather than auto-continuing into Phase 2.

---

## Prompt to paste into a fresh chat

```
Continue executing the approved remediation plan at
/Users/L141230/.claude/plans/fuzzy-moseying-rivest.md for the Databricks pipeline in
/Users/L141230/Downloads/agent_obs (git repo, remote
https://github.com/tkei9/ptof_agent_observability.git, connected to a real Databricks
Repo at /Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo, Databricks
CLI installed and authenticated to https://adb-6777798819429555.15.azuredatabricks.net).

Read /Users/L141230/Downloads/agent_obs/handoff/08_phase0_completion_and_next_steps.md
first — it's a complete record of what Phase 0 work was actually done (git/Repos setup,
obs-alerting secrets scope, webhook migration off the plaintext alert_webhook job
parameter, notebook rename to ptof_ prefix, job repointing, and a verified live Teams
POST via a synthetic obs_incidents row that has since been cleaned up). Phase 0 is
confirmed done and verified — do not redo it.

Start on Phase 1 ("Close the silent detection gaps") per the plan: the
hallucination_verdicts Layer 1 dead join on verify_grounding, the always-zero
rapid_human_correction detector, and the capability_registry drift (406 violations,
4 missing capabilities: dsa_copilot_step, dsa_session_summary, probe, saa_insight).

Standing rules for this engagement, unchanged: explain what you're about to do and why
(pulling from the plan's own "What this adds" reasoning for each phase) transparently as
you go, not as an end-of-task summary. Stop and get explicit user confirmation before
anything you can't do directly yourself (Databricks UI actions, sending real
notifications, mutating shared/production tables or rows) — a broad "yes, do it" does not
pre-authorize the specific mechanism (e.g. inserting a row into a shared table); confirm
the specific action too. Stop after Phase 1 is verified rather than auto-continuing into
Phase 2. Also: earlier in this engagement the handoff/*.md directory was unexpectedly
deleted from disk by an unexplained cause and had to be restored from git history — it
wasn't caused by an intentional command. Keep an eye out for recurrence, and don't assume
automated destructive-looking git/file output is safe to just re-run — verify state with
git status/ls before trusting any classifier-blocked or unexpected diff.
```
