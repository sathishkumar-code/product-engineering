# PRD changes for SA — Director Operations Dashboard (v1.3 → v1.8)

| Field | Value |
|---|---|
| Source PRD | `SNF/02_prd/features/director-operations-dashboard/prd-director-operations-dashboard.md` |
| Covers | v1.3 (2026-08-18, the version SA's Round 1 review — `feature-director-operations-dashboard_SA-comments.md` — was run against) through v1.8 (2026-08-21, current) |
| Purpose | Input for updating `feature-director-operations-dashboard_TD.md`. None of SA's Round 1 findings (discharge-date writers, PCC Facility Beds API, facility-time, Config UI, short-stay population field, virtualization, HMO-auth naming) are affected by anything below — these changes are additive, confined to Section 5.4 (Post Discharge Follow-up) and its supporting sections. |

All changes this round came from Sathish reviewing Section 5.4 against a direct extraction of the prototype's source (`post-discharge-survey-reference.md`) and then adding new requirements for the follow-up panels. Section/line references below are to the current PRD.

## 1. Per-question feedback-task mapping corrected (Section 5.4, Section 13)

Each of the 20 (week, question) high-risk flags now has one specific, verbatim feedback-task string, taken directly from the prototype's `wk1Questions(week)` source — added as a "Feedback task" column to the Week 1–4 tables. Previously the PRD described this mechanism with two generic, invented example strings that didn't match the actual prototype code, and Section 13 had incorrectly signed those off as "approved and final." No data-shape change (still a label + timestamp per task), but **the TD/implementation should use the exact strings from the PRD's tables, not a paraphrase or a shared generic template pool** — Section 13 now classifies this text as a binding rule, not example data.

## 2. New Follow-up notes panel (Section 5.4)

A previously one-line stub ("opens a panel with the same task-template mechanism plus a free-text field") is now a fully specified, **resident-level** panel (rolls up all 4 weeks at once, not per-week), reached via the same note icon next to the week segments:

- **Summary of responses** — one row per week: "Not yet responded," "All OK" (responded, no flag), or — for any High-risk week, even a single flag — an AI-generated one-line summary of the flagged issue(s). **New technical dependency: an AI/LLM summarization capability.** Whether it's generated synchronously when the Admin opens the panel, or precomputed asynchronously when that week's response is submitted (the same shape as Care Coordination's existing transcript-processor Lambda for conference recordings, `care-coordination.md` CARE-FR-23), is explicitly unresolved — logged as **new Open Question #3** (Section 11), flagged for an Architect decision before Story estimation for this panel.
- **Tasks** — a consolidated list of every flagged task across all 4 weeks, **deduplicated by task text** (a task flagged in more than one week — e.g. a fall in both Week 1 and Week 3 — shows once, not once per week). Creating a task here writes the same follow-up task metadata record as the existing per-week "Create task"/"Create all tasks" in the Week check-in panel (Section 7 updated to reflect both entry points). **For the TD:** this means the record's natural key should be (resident, task-text label) rather than (resident, week, question) — worth confirming/adjusting explicitly so dedup and cross-panel "already added" status stay correct without extra query-side logic.
- **Additional notes** — **new write**: a single free-text field for the resident's whole follow-up record, capped at 5,000 characters (the existing app-wide text-limit convention), distinct from the existing per-week "Action Taken" note. New row in Section 7.
- **Badge** — the note icon now shows a count badge = number of follow-up tasks created for that resident, across all weeks and either entry point. Plain count, no colour-coding.

## 3. New popup subtitles (Section 5.4)

Both the Week check-in panel and the Follow-up notes panel now open with a subtitle: resident name · Discharged on `mm/dd/yy` · a third segment — but the third segment differs by popup:

- **Week check-in panel:** "Week N check-in due `mm/dd/yy`" — a pure derived field, `discharge date + 7×N` (end of that week's 7-day window, the same boundary where Section 5.4's reminders already stop). No new data dependency.
- **Follow-up notes panel:** the resident's discharge **destination** — see item 4, hidden until available.

## 4. New dependency: Discharge destination (Section 2.3, Section 4.1)

The Follow-up notes panel's subtitle needs a destination value this PRD didn't previously source anywhere. Added as a new row in the existing Discharge-Planning dependency table (Section 2.3), following the same pattern as the other Phase-2-gated elements already there (Discharge planning progress, Stage list, etc.): **hidden this release**, subtitle shows resident name + discharge date only; **shown once Discharge Planning ships**, sourced live from its Step 1 "Discharge to" field (Home / ALF / Board & Care / Other, plus address). Also added to Section 4.1's data-sources table for consistency.

Important for the TD: this is confirmed to be a **distinct source from IDT Reports v2's own destination field** (`IDTReport.dischargePlanning.destination`, Social-Worker-entered, per `data-schema.md` §2.50). Both this dashboard and IDT Reports depend on Discharge Planning as the eventual single source for destination — they do not read it from each other.

## 5. Section 7 (Writes performed) — updated/new rows

| Write | What changed |
|---|---|
| Follow-up task metadata | Entry points expanded: "Create task"/"Create all tasks" in the Week check-in panel **and** the equivalent Add button in the Follow-up notes panel's consolidated list — both now explicitly write the same record. |
| Follow-up action-taken note | Unchanged behavior; now explicitly labeled "per-week" to distinguish it from the new field below. |
| Follow-up additional notes | **New.** Free text, up to 5,000 characters, one per resident's whole follow-up record — not per week. |

## 6. Section 11 (Open questions) — new item

| # | Area | Question | Priority |
|---|---|---|---|
| 3 | Follow-up notes panel — AI summary generation timing | Synchronous (on panel-open) vs. asynchronous/precomputed (on response submission, per the Care Coordination transcript-summary Lambda precedent) — affects latency, cost, and whether a "summary pending" UI state is needed. | Medium — needed before Story estimation for this panel. |

---

**Net new technical asks for the TD, at a glance:**
1. An AI/LLM summarization capability for weekly follow-up responses — architecture (sync vs. async) still open, Care Coordination's existing transcript-summary Lambda is the closest precedent to evaluate for reuse.
2. Confirm the follow-up task metadata record's uniqueness key supports resident-level dedup by task text (not per-week).
3. New resident-level "Additional notes" field/write, 5,000-char cap.
4. A new dependency edge on Discharge Planning for "destination" (no work needed until Discharge Planning ships; add to the TD's dependency inventory alongside the existing ones).
5. Purely presentational: two new popup subtitles, one of which (check-in due date) is a simple derived value with no data-model impact.
