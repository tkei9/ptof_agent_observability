# Handoff: capability-liveness completeness gap closed (2026-09-22)

Branch: `fix-unacknowledged-critical-regression`. This document exists so the reasoning and exact
code for this change can be merged into the separate technical reference document (not tracked in
this repo) without re-deriving it. This supersedes `HANDOFF_FANOUT_AUTOROUTING.md` as the current
pending handoff (that work already landed in `ptof_obs_alert.ipynb` and should already be in the
technical document by now; this handoff is unrelated in topic — it covers §6.6/§6.7's completeness
follow-up in `ptof_obs_liveness_detection.ipynb`, done the same day).

## Notebooks attached

The current, most up-to-date state of both notebooks referenced throughout this handoff — as of
this commit, uncommitted working-tree changes included:

- `ptof_obs_liveness_detection.ipynb` (10 cells: 0 markdown, 1-9 code) — cell indices below are
  from this exact file, not the technical document's existing §6.x cell references, which predate
  Changes 1-2 and are now stale (see validation check below).
- `ptof_obs_alert.ipynb` (14 cells) — `INCIDENT_SOURCES` (cell 5), `BACKTRACK` (cell 7),
  `DETECTOR_META` (cell 9) all unchanged by this handoff's edits; cited below only to confirm what
  they do *not* yet contain.

Current cell map for `ptof_obs_liveness_detection.ipynb` (for anyone updating the technical
document's "Defined in ... cell N" lines):

| cell | detector / content |
|---|---|
| 0 | markdown overview |
| 1 | `capability_silence` (§6.6 — unchanged position) |
| 2 | `capability_silence_ceiling` (§6.7 — unchanged position, TODO comment added, Change 3) |
| 3 | `capability_liveness_unconfigured` (new, Change 1 — no §6.x section exists yet) |
| 4 | gap-verification diagnostic (new, Change 2 — not part of any §6.x numbering; diagnostic only) |
| 5 | `shift_context_missing` (§6.4 — was cell 3, now shifted +2) |
| 6 | `etl_pipeline_health` (§6.5 — was cell 4, now shifted +2) |
| 7 | `etl_table_staleness` (§6.9 — was cell 5, now shifted +2) |
| 8 | `etl_run_slow` (§6.10 — was cell 6, now shifted +2) |
| 9 | `etl_row_count_anomaly` (§6.11 — was cell 7, now shifted +2) |

## Validation check — does the technical document reflect the current state of the observability layer?

Checked directly against the two notebooks' current content (not against memory of a prior read):

1. **Cell-number drift, confirmed.** Every "Defined in `ptof_obs_liveness_detection.ipynb` cell N"
   line for §6.4, §6.5, §6.9, §6.10, §6.11 is now off by +2 (see table above), caused directly by
   this handoff's two new cell insertions (Changes 1-2). §6.6/§6.7's cell references (1/2) are
   still correct — unaffected, since both new cells were inserted *after* cell 2. This is
   purely a citation-accuracy issue, not a behavior change.

2. **A real detector is entirely undocumented.** `capability_liveness_unconfigured` (cell 3) has
   no corresponding §6.x section anywhere in the technical document — confirmed by re-checking
   every pasted section against this handoff; none describes it. The document currently describes
   only 10 liveness/ETL detectors where the notebook now defines 11.

3. **§6.10's internal "5 vs 4" discrepancy — now resolved (Change 6 below), was live at time of
   writing.** Re-running the detector's own `(task_name, window_start)` grouping query scoped to
   2026-09-14..21 against live prod data returned exactly 5 distinct qualifying bins:
   `09-14T07`, `09-15T19`, `09-15T20`, `09-20T22`, `09-21T22`. "5" (raw bin count) is correct and
   matches the majority of existing citations, including `ptof_obs_alert.ipynb` cell 5's
   `INCIDENT_SOURCES` comment, which already said "5" and needed no change. "4" comes from
   treating `09-15T19` and `09-15T20` — adjacent hours, same tasks — as one continuous real-world
   event split across an hour boundary rather than two separate bins; both numbers are defensible
   under different definitions, but "5" is now the one to standardize on. `etl_run_slow`'s cell
   (cell 8 pre-insert / cell 9 post-insert, see Change 5) now states this explicitly instead of
   silently picking one.

4. **`capability_liveness_unconfigured` is not wired into `ptof_obs_alert.ipynb` — confirmed by
   direct inspection, not assumption.** `INCIDENT_SOURCES` (cell 5) lists 10 table-backed
   detectors; `capability_liveness_unconfigured` is not among them. `BACKTRACK` (cell 7) and
   `DETECTOR_META` (cell 9) likewise have no entry for it. This means: even once a §6.x section is
   written for it, that section cannot yet say "Persisted by `ptof_obs_alert.ipynb` cell 5,
   `INCIDENT_SOURCES`" the way every other liveness detector's section does — as of right now it
   computes a table and nothing downstream reads it. Any documentation added for it must say so
   explicitly rather than following the established §6.x template on autopilot.

5. **Previously-known drift, now fixed (Change 7 below).** `ptof_obs_nightly_baseline.ipynb`
   cell 2's comment said "Feeds ... etl_run_slow (WARN, provisional)" — stale since the
   2026-09-21 CRITICAL promotion. Now reads "CRITICAL, promoted 2026-09-21 -- see
   threshold_basis." Cell 0's matching line was already correct and needed no change.

**Net assessment:** at the time this validation check was first run, the technical document was
out of date in five identifiable ways. Items 3 and 5 are now resolved by this same handoff's
Changes 5-7 (below) rather than left open. Items 1, 2, and 4 remain: cell-number drift (item 1)
and the unwired new detector (items 2, 4) are structural and require either updating the
document's cell references or wiring `capability_liveness_unconfigured` into
`ptof_obs_alert.ipynb` — neither was in scope for this second round of edits. None of the five
were behavior bugs in the pipeline itself — all are documentation-vs-code drift, the same class
of issue the fan-out-routing handoff's regression (commit `1369b54`) grew out of when left
uncorrected. Recommend resolving item 4 (wiring the new detector in) before writing its §6.x
section, so the section describes the detector's actual persisted state rather than a state that
doesn't exist yet.

## Problem this fixes

§6.6 `capability_silence` and §6.7 `capability_silence_ceiling` are meant to jointly partition
every active `capability_registry` row with no gap: §6.6 covers `silence_grace_hours IS NOT NULL`,
§6.7 covers `grace IS NULL AND silence_ceiling_hours IS NOT NULL`. That partition only holds if
every active row actually has one of the two fields set. A third case was previously unhandled:
**active AND both fields NULL** — a capability with no liveness detector watching it at all, not
because it's healthy, but because nobody configured a threshold for it.

This is not hypothetical: `sev2-insights` was in exactly this state before §6.7 existed (silent
64h+, undetected until a human noticed during an audit — see the now-closed 2026-09-21 finding in
`prod-checkup-notification-gap.md`). §6.7 closed the gap for that one capability by giving it a
threshold. It did not close the *structural* gap — a future capability added to the registry
without either field set would fall through both detectors the same way, silently.

A second, unrelated issue was raised in review of §6.7's own evidence: the worked stats cite "989
gaps" for sev2-insights while a separate caveat cites "~572-row history" — those shouldn't both be
right for one capability's history (gaps ≈ rows − 1), so the 120h ceiling's ~17% margin claim
needed a data-verification step before being trusted further.

Three changes close these, all in `ptof_obs_liveness_detection.ipynb`.

---

## Change 1 — `capability_liveness_unconfigured` (new cell 3, CRITICAL)

**Reasoning:** the third, complementary bucket to §6.6/§6.7's two WHERE clauses. Not a silence
check itself — it doesn't read `v_llm_bronze`/call history at all, only the registry's own
configuration — so a fired incident should read as "fix this capability's registry row," not
"this capability just went quiet." Ships CRITICAL from day one rather than staged WARN-first,
by the same reasoning §6.9 `etl_table_staleness` did: this reuses the already-proven simple
boolean-config pattern, not a new statistical model, and the failure mode it exists to catch — a
monitored-in-name-only capability going dark with zero backstop — is the same one that already
happened once undetected.

```sql
CREATE OR REPLACE TABLE mq_gmdf_dev.oil_obs.capability_liveness_unconfigured AS
SELECT
    r.capability, r.owner,
    sha2(r.capability, 256) AS finding_signature,
    current_timestamp()     AS detected_at
FROM mq_gmdf_dev.oil_obs.capability_registry r
WHERE r.active = true
  AND r.silence_grace_hours IS NULL
  AND r.silence_ceiling_hours IS NULL;
```

Key: constant `sha2(capability, 256)` — same lifecycle pattern as §6.6/§6.7/§6.4
(`shift_context_missing`): an ongoing config gap is one incident whose `detection_count` climbs
and whose `resolved_at` auto-clears the moment a human sets either field on that capability's
registry row.

**Currently fires on:** nothing observed yet as of this handoff — every active capability in the
registry has one of the two fields set (`sev2-insights` got its ceiling in §6.7; the other three
have grace). This is a forward-looking completeness guard, not a response to a second live gap.

---

## Change 2 — gap-verification diagnostic (new cell 4, read-only, not persisted)

**Reasoning:** resolve the 989-gaps-vs-572-rows discrepancy in §6.7's recorded evidence before
leaning further on either number. DIAGNOSTIC ONLY — not part of the automated liveness run, no
output persisted anywhere; run by hand.

```sql
WITH ordered AS (
    SELECT called_at,
           lag(called_at) OVER (ORDER BY called_at) AS prev_called_at
    FROM mq_gmdf_dev.oil_obs.v_llm_bronze
    WHERE capability = 'sev2-insights'
)
SELECT
    count(*)                                            AS total_rows,
    count(prev_called_at)                               AS total_gaps,
    round(max((unix_timestamp(called_at) - unix_timestamp(prev_called_at)) / 3600.0), 1)
                                                         AS worst_gap_hours_recomputed
FROM ordered;
```

**Not yet run against live prod data as of this handoff** — the query is in place; running it and
reconciling whichever of "989 gaps" / "~572 rows" is wrong (and re-checking the 120h margin against
the recomputed worst gap) is the immediate next step, not something this handoff can close on its
own.

---

## Change 3 — periodic-recheck TODO on §6.7 (cell 2, comment only)

**Reasoning:** §6.7's own caveat already flags cadence drift as a risk distinct from
"hasn't-fired-yet" risk — a heavy-tailed, irregular-cadence capability is exactly the type whose
behavior can shift over time, and the 120h ceiling was set from one fixed historical snapshot, not
something recomputed on a schedule. Recorded as a standing TODO rather than built now, since there
is no evidence yet that the distribution has actually drifted — matches this pipeline's
evidence-before-action standard.

```sql
-- TODO(next audit cycle, added 2026-09-22): re-pull sev2-insights' full gap distribution and
-- recheck 120h still clears the worst gap. Cadence drift is an explicit caveat above, not just
-- unfired-event risk -- a heavy-tailed, irregular-cadence capability is exactly the type whose
-- behavior can shift over time, and this threshold was set from one fixed historical snapshot.
```

---

## Change 4 — `ptof_obs_liveness_detection.ipynb` cell-0 documentation

The "In plain terms" list, "What this notebook does" list, "Tables/views touched" writes list, and
a new "Capability liveness unconfigured" section were all updated in lockstep with Changes 1-3 so
the written overview doesn't drift from what the notebook actually computes — matches the existing
per-detector documentation pattern (every other detector, e.g. §6.7/§6.9, has its own prose
section).

---

## Net effect — why this improves completeness

§6.6 and §6.7 together were previously documented as a complete partition of the active registry,
but that completeness claim depended on an assumption (every active row has grace or ceiling set)
that nothing in the code actually enforced. This closes that gap structurally: any future active
capability that ships without a liveness threshold now gets flagged immediately as a config
problem, rather than silently having zero detection coverage until a human notices during an
audit — which is exactly how the original `sev2-insights` gap was found in the first place.

---

## Change 5 — MAD-floor diagnostic (new cell, read-only, inserted after `etl_run_slow` in
`ptof_obs_liveness_detection.ipynb`)

**Reasoning:** §6.10 `etl_run_slow`'s `upper_bound_s` had no floor — a stratum with
`mad_duration_s` at or near 0 collapses `upper_bound_s` toward `median_duration_s`, the same
degenerate-MAD shape that got `write_lag_anomalies` retired entirely. Before fixing it, checked
whether it's already live.

```sql
SELECT table_or_view, task_name, n, median_duration_s, mad_duration_s, upper_bound_s
FROM mq_gmdf_dev.oil_obs.etl_duration_baseline
WHERE mad_duration_s = 0
   OR mad_duration_s < 1
ORDER BY mad_duration_s, median_duration_s;
```

**Run 2026-09-22 against prod: 0 rows returned.** No degenerate stratum exists today — the fix
below is precautionary, not closing an active false-positive.

## Change 6 — `upper_bound_s` floor + event-count reconciliation (`ptof_obs_nightly_baseline.ipynb`
cell 2; `ptof_obs_liveness_detection.ipynb` `etl_run_slow` cell)

**Floor:** `upper_bound_s` changed from `median + 5*1.4826*MAD` to
`greatest(median + 5*1.4826*MAD, median * 1.5)`, so a `MAD=0` stratum floors at 1.5x its own
median instead of collapsing to the median itself.

**Reconciliation:** re-ran `etl_run_slow`'s own `(task_name, window_start)` grouping query scoped
to 2026-09-14..21 against live prod data. Result: exactly 5 distinct bins clearing the `>=3` floor
(`09-14T07`, `09-15T19`, `09-15T20`, `09-20T22`, `09-21T22`). "5" is correct and is what
`ptof_obs_alert.ipynb` cell 5's `INCIDENT_SOURCES` comment already said — no change needed there.
"4" comes from treating `09-15T19`/`09-15T20` (adjacent hours, same tasks) as one continuous event
rather than two bins. Both are defensible under different definitions; the detector's own cell now
states this explicitly instead of silently picking one.

## Change 7 — stale `nightly_baseline.ipynb` comment fix (cell 2)

`"Feeds ... etl_run_slow (WARN, provisional)"` → `"Feeds ... etl_run_slow (CRITICAL, promoted
2026-09-21 -- see threshold_basis)"`. Cell 0's markdown line was already correct.

## Change 8 — §6.11 disclosure note (`etl_row_count_anomaly` cell)

Added one sentence: `irregular`-regime tables and any table below the `n>=50` floor have zero
row-count coverage from this or any other detector — not a defect, but previously undisclosed.

---

## Outstanding before this can be considered fully closed

- Run the Change 2 diagnostic against prod and reconcile the 989-gaps/572-rows discrepancy; update
  §6.7's recorded evidence with whichever number is correct.
- `capability_liveness_unconfigured` is defined in `ptof_obs_liveness_detection.ipynb` but **not
  yet wired into `ptof_obs_alert.ipynb`'s `INCIDENT_SOURCES`/`BACKTRACK`/`DETECTOR_META`** — as
  defined today it computes a table but nothing reads it, so it cannot yet reach
  `obs_incidents`/Teams. This is the same "computed but not persisted" bug class §6.1's original
  `pipeline_heartbeat`/`etl_pipeline_staleness` gap was (fixed 2026-09-15) — wiring it in is the
  natural next step, not yet done or explicitly requested as of this handoff.
