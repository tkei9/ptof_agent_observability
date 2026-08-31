# v1 POC: agent I/O quality observability, registry-independent scope

Written 2026-08-31 as a context handoff for executing the finalized v1 scope decision in a
fresh chat. This supersedes nothing in `fuzzy-moseying-rivest.md` — it's the concrete
execution plan for the three amendments already recorded there (runtime_violation silencing,
POC v1 scope narrowing, and this document's detector-level detail).

## What "v1" means

The first version of this pipeline to run against real ISH production data, scoped to:
1. Real ISH agent-backed capabilities (`dsa_optimize`, `dsa_copilot`, `dsa_batch_summary`,
   `dsa_session_summary`, `handover_delivery`, etc. — whatever `v_llm_bronze` actually carries).
2. Only detectors that judge the agent's own input/output behavior or delivery — not
   detectors whose correctness depends on `capability_registry`/`runtime_allowlist` being
   populated with trustworthy production values, because today they aren't (406 violations
   in the dev backtest, 4 live capabilities unregistered: `dsa_copilot_step`,
   `dsa_session_summary`, `probe`, `saa_insight`).

Real data pulled 2026-08-31 (see "Evidence" under each detector below) informed every
decision here — this was not a policy call made in the abstract.

## Final detector disposition

### IN — will notify to Teams in v1 (7 detectors, 3 already wired, 3 new/fixed here)

| Detector | Status | Source table | Registry dependency |
|---|---|---|---|
| `handover_delivery` | already wired | `handover_delivery_failures` | none |
| `handover_delivery_rate` | already wired | `handover_delivery_rate` | none |
| `capability_error_rate_sustained` | already wired | `capability_error_rate_alert` | none |
| `blank_output` | **new this pass** | `blank_output_findings` | none |
| `schema_field_missing` | **new this pass** | `response_schema_drift` | soft (own query joins `capability_registry r.active=true`) |
| `latency_anomaly` | **fix this pass** | `latency_anomaly_findings` (was: raw `latency_anomalies`) | none |
| `hallucination_high` | already wired | `hallucination_signal` | soft (upstream `capability_health`-style join) |

**Why each is in, with the actual evidence behind it:**

- **`handover_delivery` / `handover_delivery_rate`** — a shift handover is a safety-critical
  communication; ISH's whole point is the next shift knowing what happened on the last one.
  Already caught a real incident (64% 7-day failure rate, 16 failed sends, DNS `getaddrinfo
  failure`) before it self-resolved. No other detector sees mail-delivery failure. Zero
  registry dependency confirmed in both tables' own defining queries.
- **`capability_error_rate_sustained`** — the "is the agent actually working" pager, at the
  capability/model_config level, sustained over 6h so a single quiet hour can't hide an
  outage. Real evidence found this session: `spe-claude-sonnet-46` had a 100% auth-failure
  burst across `saa_insight`/`sev2_insight`/`dsa_optimize`/`summary` on 2026-08-27 — this
  detector is the mechanism that would have caught it had it persisted past 6h. It didn't
  this time, but the next one might. **Known caveat**: its only current offender
  (`dsa_optimize`/`test-bot-claude-v1`, 17/17 failed) is the notebook's own documented
  expected condition (client-side ~75s timeout on a test harness) — see Step 5 below for the
  pre-ack needed so day-one isn't a known-non-issue card.
- **`blank_output`** — catches "the call formally succeeded but returned nothing," a failure
  class invisible to every error-rate/success-flag-based detector in the system. For ISH, a
  blank handover summary silently delivered is worse than an error because nothing else
  flags it. Currently 0 rows system-wide — zero day-one noise, pure tripwire, confirmed zero
  registry dependency in its own query (reads only `blank_output_incidents`).
- **`schema_field_missing`** — catches output-shape drift (an expected field silently
  disappearing from parsed responses), which downstream automation could misinterpret as
  "no risk" instead of "unknown." Currently 0 rows. Confirmed soft registry dependency
  (`capability_registry r.active=true` join in its own query) — blind to the 4 unregistered
  capabilities, documented, not a blocker.
- **`latency_anomaly`** — per-capability regression against that capability's *own* history,
  not a fixed SLA. `dsa_copilot` crossed into `is_reliable=true` this week (149 samples,
  13-day span) — this detector goes from theoretical to actually-live for a real capability
  today. Currently 0 anomalies (genuinely quiet, not data-starved). Requires the dedup-key
  fix below or it repeats a documented spam bug ("529 cards for one config").
- **`hallucination_high`** — answers "did the agent tell the truth about what it was given,"
  the core of the "agent input/output relationship" framing. Highest compliance/audit
  exposure in the list (GxP-adjacent content). **Honest caveat**: only 2 scored rows exist
  system-wide right now, both `low` risk — armed and correct, but data-starved. Its value
  grows with volume; nothing to point to yet.

### OUT — excluded or WARN-only, not notified in v1

| Detector | Disposition | Why |
|---|---|---|
| `runtime_violation` / `runtime_violation_digest` | silenced (WARN) | Real offenders (`dsa_copilot_step`, `dsa_optimize`, `sev2_insight`, `summary`, `saa_insight`) line up with test-bot/unregistered-config traffic exploring new configs, not production drift. Confirmed noise, not signal, until registry drift (Phase 1 item 5) is fixed. |
| `runtime_allowlist_populated`, `schema_baseline_coverage` | excluded | Both query the reference tables as their entire subject — "is our governance table complete," not "did the agent behave." Wrong question for v1. |
| `pipeline_heartbeat`, `credential_outage`, `write_lag`, `capability_error_rate` (per-hour), `capability_silence`, `shift_context_missing` | WARN-only | Pipeline/infra health, not agent-output quality. `shift_context_missing` in particular is a known standing gap (383-441 `dsa_optimize` calls never populate shift/batch identity) — real, but not actionable via alerting, and not what v1 is telling a story about. |
| `latency_failures` | excluded (not ready) | Raw per-hour `error_class` breakdown with no threshold/findings table built on top — needs real design work, not a wiring fix. |
| `rapid_human_correction` | excluded | Still 0 rows (upstream `dsa_*` capabilities never emit shift/batch identity to correlate against) AND registry-dependent. Double-disqualified. |

## Step-by-step execution plan

### Step 0 — confirm starting state

There is already an **uncommitted, local-only** change sitting in
`ptof_obs_alert.ipynb` from this session: `runtime_violation` downgraded from `CRITICAL` to
`WARN` in both its `INCIDENT_SOURCES` tuple and its `check()` call. Run `git status` /
`git diff ptof_obs_alert.ipynb` first to confirm this is still there (8 insertions, 3
deletions) before starting Step 1, so it gets bundled into the same commit rather than lost
or double-applied.

### Step 1 — `ptof_obs_alert.ipynb`: add `blank_output` to `INCIDENT_SOURCES`

Insert into the `INCIDENT_SOURCES` list (after the `hallucination_high` tuple, or anywhere
in the list — order doesn't matter to the MERGE loop):

```python
("blank_output", "blank_output_findings", "finding_signature",
 "capability", "CRITICAL",
 ["model_config", "blank_rate_window", "blank_count_window", "total_calls_window",
  "latest_hour"], ""),
```

`finding_signature` is `blank_output_findings`' own sha2 key over `(capability,
model_config)` — already unique and dedup-safe, no signature engineering needed.

Add to `DETECTOR_META`:

```python
"blank_output": {
    "label": "Agent returned a blank response",
    "what": "A capability's calls formally succeeded (no error, no exception) but the "
            "response content was empty often enough to exceed the blank-rate floor. "
            "Invisible to every error-rate-based detector in this system.",
    "triage": [
        "Check whether this is a prompt/template regression or an upstream input problem",
        "blank_rate_window / blank_count_window / total_calls_window show how bad and how big",
        "A blank handover or summary reaching a real person is worse than an error — nobody "
        "else is watching for this",
    ],
},
```

Add to `_facts_from_payload`'s `pretty` dict:

```python
"blank_rate_window": "Blank rate", "blank_count_window": "Blank count",
"total_calls_window": "Total calls (window)", "latest_hour": "Latest hour",
```

**Reasoning:** `blank_output_findings` already exists (built in `ptof_obs_mal_output.ipynb`,
task `03_malformed_output`, which already runs upstream of `06_alert` in the `obs_fresh_scan`
job DAG) with a proper sha2-keyed dedup signature and a sane threshold (>3 blanks, >2% rate,
≥10 calls, 6h window) already baked in. This is purely a wiring fix — no new SQL, no new
notebook logic, just pointing the alert notebook's persistence layer at a findings table that
was already computed and already correct.

### Step 2 — `ptof_obs_alert.ipynb`: add `schema_field_missing` to `INCIDENT_SOURCES`

```python
("schema_field_missing", "response_schema_drift", "finding_signature",
 "capability", "CRITICAL",
 ["field_name", "drift_type", "baseline_presence_rate", "current_present", "current_rows"],
 "WHERE drift_type = 'field_missing'"),
```

`finding_signature` here is `response_schema_drift`'s own sha2 key over `(capability,
field_name)` — a real per-drift-event unique key, already dedup-safe.

Add to `DETECTOR_META`:

```python
"schema_field_missing": {
    "label": "Expected output field went missing",
    "what": "A field that was reliably present in this capability's response schema has "
            "disappeared. Downstream automation reading a missing field as null/false "
            "instead of unknown is the real risk here.",
    "triage": [
        "Check for a recent prompt-template edit or model swap on this capability",
        "baseline_presence_rate tells you how reliably the field used to appear",
        "Known gap: this detector can't see dsa_copilot_step / dsa_session_summary / "
        "probe / saa_insight yet -- they aren't in capability_registry",
    ],
},
```

Add to `_facts_from_payload`'s `pretty` dict:

```python
"field_name": "Field", "drift_type": "Drift type",
"baseline_presence_rate": "Baseline presence rate", "current_present": "Currently present",
"current_rows": "Current rows",
```

**Reasoning:** same shape as Step 1 — `response_schema_drift` already exists
(`ptof_obs_mal_output.ipynb`), already has a real unique key, this is purely a wiring fix.
The one thing worth being explicit about (and the triage text says so) is that this table's
own query joins `capability_registry r.active=true`, so it inherits the registry-blindness
caveat — documented in the card itself so whoever's on call sees the limitation without
having to go read the SQL.

### Step 3 — `ptof_obs_alert.ipynb`: fix `latency_anomaly`'s dedup key (spam-bug fix)

Replace the existing `INCIDENT_SOURCES` tuple:

```python
# OLD — remove this:
("latency_anomaly",    "latency_anomalies",          "id",
 "capability",          "CRITICAL",
 ["latency_ms", "latency_verdict", "p95_ms"], "WHERE latency_verdict = 'anomaly'"),

# NEW — replace with:
("latency_anomaly",    "latency_anomaly_findings",   "finding_signature",
 "capability",          "CRITICAL",
 ["model_config", "latency_verdict", "worst_latency_ms", "baseline_p95_ms",
  "anomaly_bound_ms", "anomaly_count", "latest_called_at"],
 "WHERE latency_verdict = 'anomaly'"),
```

Update `_facts_from_payload`'s `pretty` dict — the old `latency_ms`/`p95_ms` keys can stay
(harmless, just unused now) or be removed; add the new ones either way:

```python
"worst_latency_ms": "Worst latency (ms)", "baseline_p95_ms": "Baseline p95 (ms)",
"anomaly_bound_ms": "Anomaly bound (ms)", "anomaly_count": "Anomaly count",
"latest_called_at": "Latest call",
```

No `DETECTOR_META` change needed — the `latency_anomaly` entry already exists and its
label/triage text doesn't reference the old column names directly.

**Reasoning:** `latency_anomaly_findings` (`ptof_obs_latency_detection.ipynb`) already exists
specifically to fix a documented incident — the current `INCIDENT_SOURCES` entry keys on the
raw per-call `id` from `latency_anomalies`, which the notebook's own comment says "made every
hourly poll from a slow capability a new incident... 529 cards for one config." The fixed
table groups by `(capability, model_config, latency_verdict)` with a real sha2 signature.
This is not optional polish — shipping `latency_anomaly` into v1's notify path without this
fix risks immediately repeating that exact spam incident, and `dsa_copilot` just became the
first capability with a reliable-enough baseline for this detector to actually fire.

### Step 4 — commit

Bundle Steps 0–3 (the pending `runtime_violation` silencing + the three changes above) into
one commit. Suggested message: something like "Wire blank_output/schema_field_missing into
alerting, fix latency_anomaly dedup key, silence runtime_violation for v1 POC scope" — since
all four changes together are what makes the v1 scope decision actually true in the running
system, not just documented in a plan file.

### Step 5 — push, confirm Databricks Repo sync

`git push origin main`, then `databricks repos get
"/Users/tyler.kei@lilly.com/ptof_agent_observability_repo" -o json` and confirm
`head_commit_id` matches the new commit hash (it auto-synced last time; verify it again
rather than assuming).

### Step 6 — run and observe

Run `ptof_obs_alert.ipynb` with `env=dev` (interactively, or trigger job `obs_fresh_scan`,
id `585607309385820`, task `06_alert` — full job run also re-computes all upstream findings
tables, which is the more realistic test). Confirm:
- `emitted: blank_output`, `emitted: schema_field_missing`, `emitted: latency_anomaly` all
  appear with no `SKIPPED` lines.
- Console CRITICAL section still shows `capability_error_rate_sustained` for
  `dsa_optimize`/`test-bot-claude-v1` (expected, addressed in Step 7) and `runtime_violation`
  no longer appears as CRITICAL anywhere (it's WARN now).
- `Teams webhook status=202` if anything new/unacknowledged is eligible to notify.

### Step 7 — pre-acknowledge the known test-bot incident

Once `capability_error_rate_sustained` has persisted at least once (it already has, from
prior runs this session), run:

```sql
UPDATE mq_gmdf_dev.oil_obs.obs_incidents
SET acknowledged_by = '<your actual email>', acknowledged_at = current_timestamp()
WHERE detector = 'capability_error_rate_sustained'
  AND capability = 'dsa_optimize'
  AND acknowledged_at IS NULL;
```

**Reasoning:** this detector's only current offender is a documented, expected condition
(the notebook's own "Known-expected-states" note: `dsa_optimize` on `test-bot-claude-v1`
succeeds only under a ~75s client timeout, 1-in-22 recently). Without this pre-ack, the
notify query re-fires this exact known-non-issue every 24h forever (that's the designed
behavior for genuinely-unacknowledged CRITICALs — it's not a bug, it's just the wrong target
for it here). This is a data action, not a code change — confirm with whoever owns this
engagement before running an `UPDATE` against a shared production-adjacent table, same as
every other data-mutation step in this plan.

### Step 8 — final check

Re-run once more within the same day and confirm: no repeat card for `capability_error_rate_sustained`/`dsa_optimize` (pre-ack held), and any of the three new detectors that do fire render a card with real triage text (not a bare fallback label), confirming the `DETECTOR_META` additions actually took.

## Prompt to paste into a fresh chat

```
Continue executing the approved v1 POC integration plan for the Databricks pipeline in
/Users/L141230/Downloads/agent_obs (git repo, remote
https://github.com/tkei9/ptof_agent_observability.git, connected to a real Databricks Repo
at /Workspace/Users/tyler.kei@lilly.com/ptof_agent_observability_repo, Databricks CLI
installed and authenticated to https://adb-6777798819429555.15.azuredatabricks.net, SQL
warehouse id cd6c1145a46bec44 available for direct data queries via
`databricks api post /api/2.0/sql/statements`).

Read /Users/L141230/Downloads/agent_obs/handoff/09_v1_poc_scope_and_integration_plan.md
first -- it is the complete, approved plan: which detectors are in v1 and why (with real
production-data evidence backing each decision), which are out and why, and a step-by-step
list of the exact ptof_obs_alert.ipynb changes needed (INCIDENT_SOURCES entries,
DETECTOR_META entries, pretty-label additions) plus the one data-mutation step (pre-acking
the known dsa_optimize/test-bot-claude-v1 incident). Execute Steps 0 through 8 in that
document, in order.

Standing rules for this engagement: explain what you're about to do and why, transparently,
before doing it -- not as an end-of-task summary. Get explicit confirmation before any
action that mutates a shared/production-adjacent table or row (Step 7's UPDATE especially),
before committing/pushing to git, and before triggering a real Databricks job run that will
post a live Teams card. A broad "yes, do it" does not pre-authorize the specific mechanism --
confirm the specific action too. This notebook (ptof_obs_alert.ipynb) is a single mega-cell
Databricks notebook stored as .ipynb -- do NOT use NotebookEdit with a partial new_source
against it, as that replaces the entire cell's content and has already caused one accidental
918-line deletion this engagement (recovered via git checkout). Edit it by loading the
notebook JSON in Python, doing exact string replacement on the joined cell source, and
writing the JSON back out -- the pattern already used successfully for every prior edit to
this file this session. Verify the diff with `git diff` after every edit before moving on.
```
