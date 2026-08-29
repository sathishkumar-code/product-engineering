# SA Review: feature-director-operations-dashboard

| Field | Value |
|---|---|
| Source PRD | `SNF/02_prd/features/director-operations-dashboard/prd-director-operations-dashboard.md` — Round 1 against v1.3 (2026-08-18); Round 2 against v1.8 (2026-08-21) |
| Product | SNF |
| review_round | 2 (Round 1: initial PRD review, run alongside the first Technical Design draft. Round 2: PRD v1.3→v1.8 changeset, `feature-director-operations-dashboard_PRD-changes-for-SA.md`, confined to §5.4 Post Discharge Follow-up — TD updated in step, `SNF/03_architecture/features/feature-director-operations-dashboard_TD.md`) |
| Verdict | Approved-with-changes |

## PRD Review (Round 1)

**1. `Resident.dischargeDate` will have three unarbitrated writers once this ships, not two.** PRD §4.1/§7 frame "last write wins" as governing this dashboard, IDT Reports' "Discharge date set" chip, and (eventually) Discharge Planning — but the PCC webhook integration already writes the same field today, independently of all three. Per ADR-001 (`pcc-webhook-convergent-state`), every `discharge`/`cancelDischarge`/`readmit`/`transfer` event triggers a re-pull of PCC's authoritative record and a `$set` on `Resident.dischargeDate`, with no awareness of either dashboard's writes. Concretely: an Admin sets an anticipated discharge date on this dashboard; PCC fires an unrelated `updateResidentInfo` event for that same resident later that day; PCC's convergent-state handler re-pulls and silently overwrites the Admin's entry with whatever PCC currently reports (which may be nothing, if PCC's own discharge date field is empty until the actual discharge is recorded there). The PRD's "last write wins, always editable" framing (§4.1) is accurate as written, but understates how often "the last write" will be PCC's, not a human's. Recommend a short note in §4.1 acknowledging the PCC writer explicitly. The TD (§3.8/§5) adds source/timestamp tracking to make this feature's own writes attributable, per Sathish's decision — full resolution needs IDT Reports' write path updated too (see Recommended technical stories below).

**2. "PCC Facility Beds API" (§4.1) is net-new integration work, not a data source this feature can assume is already flowing.** I confirmed the endpoint exists in the contracted PCC REST API spec (`GET /public/preview1/orgs/{orgUuid}/facs/{facId}/beds`), but PCC's own API description states this endpoint's approval is separate from whatever Facility/Floors/Rooms/Units access the org already has — meaning occupancy/bed-inventory accuracy in the facility summary card (§2.1, §5.1) depends on an external approval this PRD doesn't call out as a dependency. Per Sathish's decision, the TD scopes the real integration (§3.3) with a graceful "pending" fallback rather than blocking on it — recommend the PRD's Sources/dependencies framing (currently only calling out Discharge Planning and IDT Reports as dependencies, §0 header table) be updated to list this too.

**3. Section 4.4's facility-local-time requirement and new business-hours cutover land on an unresolved platform question.** ADR-005 (facility timezone authority) is `Proposed`, not decided — only the staff app currently treats facility time as authoritative for computation; the admin app, where this dashboard lives, is explicitly unaudited in that ADR. Per Sathish's decision, the TD resolves this by computing facility-local date boundaries server-side for this feature's own endpoints (TD §3.4), rather than either replicating the staff app's client-side timezone library or waiting on the platform-wide decision. This is a real design choice worth the ADR-005 owner's attention (a server-side resolution may be a cheaper platform-wide answer than a client library per surface) but doesn't itself close ADR-005 — no PRD change needed, flagging for visibility only.

**4. No admin UI exists to edit any `Config` field today — §4.4's "Business hours must be added to facility settings" doesn't have a build target.** Every existing `Config` field, including `timeZone`, is set via direct DB/API access; there is no Settings-page affordance for any of them (confirmed in `dashboard-reporting.md` DSH-FR-22 / Gap G-6). This isn't a defect in this PRD specifically — it's a platform-wide gap this PRD happens to run into first for a field that actually needs to be right per-facility. Recommend Sathish/Product decide explicitly: ship with manual backend entry (consistent with existing practice, zero new UI work) or treat this as the trigger to build the platform's first Config-editing affordance. Either is defensible; leaving it implicit isn't.

**5. §5.2's default population rule ("short-stay residents by default... long-term residents added one at a time") doesn't map cleanly onto a field I could confirm in the as-built schema.** `Resident.careType` is the closest candidate (its enum includes `skilled_nursing` per schema review), but I could not confirm its full value set or whether it, rather than some combination of `careType` + admission type + payer, is actually what should drive default inclusion on this grid. Getting this wrong at launch means the primary surface (the Skilled Coverage grid the whole daily huddle is built around) silently shows the wrong resident population. This should be confirmed with whoever owns the Resident data model before Epic/Story creation for §5.2 — recommend as a spike (below).

**6. §9's 500-resident performance budget ("virtualisation required if runway can't meet them at that size") should be read as a certainty, not a conditional, given today's baseline.** No client-side virtualization pattern exists anywhere in the admin app — the rest of the app's answer to large lists is server-side pagination, which doesn't fit a continuously-scrollable, drag-editable coverage grid. Not a PRD defect — just flagging that "if runway can't meet them" will resolve to "yes, build it" as soon as anyone tests at facility scale, so Epic/Story estimation should assume this cost up front rather than discover it mid-sprint.

**7. Minor, non-blocking: the PRD names the same field two different things across sections.** §5.2/§7 call it "HMO auth"/"HMO auth column"/"HMO auth value"; §4.1 calls the same value "Plan authorisation end date." These are the same field (both Managed Care only, both clamped by R1–R3, both edited via "click the value to edit in place"). Recommend picking one term before Story-writing so stories don't accidentally spec two fields, or add a one-line note that they're synonyms.

### Recommended technical epics/stories/spikes

- **Technical Spike** — Confirm PCC's Facility Beds API approval status and timeline for this org with the PointClickCare partner team (Finding 2); determine whether the TD's "pending" fallback state needs to survive past initial launch.
- **Technical Spike** — Confirm the exact `Resident` field(s)/values distinguishing "short-stay" from "long-term/custodial" for §5.2's default population rule (Finding 5). Blocks writing the default-population query and the rollout backfill script correctly.
- **Technical Story** — Build the facility-local "today" resolution (business-hours cutover + `Config.timeZone`/new `businessHours` field) server-side, scoped to this feature's endpoints, per TD §3.4/§3.5. Doesn't touch other surfaces.
- **Technical Story** — Add `dischargeDateSource`/`dischargeDateUpdatedAt` to `Resident` and wire this feature's write path to set them (TD §3.8/§5); coordinate with whoever owns IDT Reports to backfill the same on their "Discharge date set" write path, since their own Technical Design already flagged this exact field as an unresolved risk before this feature existed (Finding 1).
- **Technical Task** — Build a reusable virtualized-grid primitive (`@tanstack/react-virtual`) for the admin app (Finding 6) — first consumer is this feature's maximised Skilled Coverage view, but written generically enough for the next large-list admin feature to reuse.
- **Technical Task** — Build a reusable inline-auto-save field pattern (PATCH-on-blur, optimistic update, inline-error rollback) for the admin app — no such pattern exists today; this feature needs it for five separate fields (available skilled days, discharge date, readmission risk, HMO auth date, Action Taken note).
- **Technical Task** — Extend `Config` with `businessHours {open, close}` (Finding 4) — contingent on the product decision about whether a settings UI is also needed at launch.
- **Technical Task** — ~~New nightly `facilityBeds.cron.ts` (PCC Facility Beds pull → `FacilityBedSnapshot`, TD §3.3)~~ — **superseded 2026-08-21.** Sathish revised the Facility Beds design to resident-derived occupancy (`Config.totalBeds` + active-resident count, no PCC polling); this task no longer exists. `postDischargeFollowUp.cron.ts` (day-5/12/19/26 push + daily reminder-until-response, TD §6) still stands, no existing job to extend.

## PRD Review (Round 2) — 2026-08-21, PRD v1.3 → v1.8

Scope: `feature-director-operations-dashboard_PRD-changes-for-SA.md` confirms this round is confined to §5.4 (Post Discharge Follow-up) and its supporting sections; none of Round 1's seven findings are affected. TD updated in step (§3.9 and ripple through §§1/4/5/6/7/8/9/10/11).

**1. §5.4's "AI-generated summary sentence" (and Open Question #3) should be reworded — the design behind it, once actually built, isn't AI generation.** I checked this platform's as-built docs for an existing text-summarization capability this panel could call into and found none — the only related precedent, Care Coordination's conference-transcript Lambda (`care-coordination.md` CARE-FR-23), is external, out-of-repo, and audio-specific, not reusable here. Separately, I confirmed the 20 (week, question) high-risk triggers in the PRD's own tables collapse to only 12 distinct feedback-task strings (several repeat verbatim across weeks — e.g. "Fall follow-up..." appears in all 4). That means every possible per-week summary is already a closed-form join over those 12 fixed strings — not the "hand-writing every combination as static text" the PRD correctly rules out (32 combinations), but a small rule-based generator over them, which is a different and much cheaper thing than real natural-language generation. **Decided with Sathish (2026-08-21): build it as a deterministic template join, not a real LLM call** — no new AI/LLM vendor, no PHI-to-third-party-vendor question for resident health-survey answers (which the PRD's existing HIPAA carve-out doesn't cover), no "summary pending" state, no sync-vs-async architecture question (TD §3.9). Recommend Product Manager reword §5.4's "AI-generated summary sentence" and close Open Question #3 to reflect this — the current wording would otherwise instruct engineering to scope a real AI capability that isn't going to be built.

**2. Confirmed: the new task-dedup key change (PRD change #2) is well-defined because of the same 12-string collapse above.** The changeset asks that the follow-up task record's natural key be "(resident, task-text label)" rather than "(resident, week, question)." I traced this against the actual 20 mapped rows and confirmed the 12 unique strings make that key concrete, not just directionally correct — TD §3.9/§5 restructures `flaggedTasks` from per-week to a resident-level `createdTasks` array keyed by a 12-entry `templateKey` table built directly from the PRD's binding tables. No PRD change needed; noting this for the record since it was flagged as "worth confirming/adjusting explicitly" in the changeset.

**3. Confirmed against `data-schema.md`: the Follow-up notes panel's destination dependency is correctly scoped as a distinct source from IDT Reports' `dischargePlanning.destination`.** PRD change #4 states this explicitly and I verified it against the as-built schema (IDTReport's `dischargePlanning.destination` is a Social-Worker-entered free-text field on the IDT report, a different value than Discharge Planning's structured Step-1 "Discharge to" field this panel will eventually read). No PRD change needed — informational confirmation only.

### Recommended technical epics/stories/spikes (Round 2 additions)

- **Technical Task** — Build the 12-entry `templateKey` lookup table (Finding 1/§3.9) mapping each of the PRD's 20 (week, question) rows to its collapsed task string, and the resident-level `createdTasks`/`additionalNotes` fields on `PostDischargeFollowUp` (TD §5) plus the three new/changed endpoints (`GET`/`PATCH .../notes`, `POST .../tasks`, TD §6). One story — the dedup key, the lookup table, and the two panels' "already added" consistency all depend on each other and should ship together, not be split across sprints.
- **Technical Task** — Unit-test the `templateKey` table against all 20 PRD rows verbatim (Finding 1) — this is the one place a hand-transcription slip would silently ship the wrong feedback-task wording, which PRD §13 classifies as a binding rule.

## Epics/Stories Review (Round 1/2)

Not yet run — `epics-stories.md` for this slug doesn't exist yet. This section will be completed once Product Manager generates it, confirming the Round 1 and Round 2 recommendations above were incorporated and reviewing the functional stories themselves.

## Technical Design Review

| Field | Value |
|---|---|
| Source | SA-authored |
| Submission path (if team-submitted) | N/A |
| Original format (if team-submitted) | N/A |
| Convert to standard template? | N/A |
| Verdict | In review |

### Findings

Not yet reviewed by anyone else — `feature-director-operations-dashboard_TD.md` was authored directly against `technical-design-template.md` in this same pass (2026-08-18), incorporating Sathish's three decisions on Facility Beds scope, facility-time authority, and discharge-date attribution. No independent findings recorded yet; this section is ready for Sathish's or Product Manager's pass.

**2026-08-21, revision 1:** Sathish revised the Facility Beds design (§3.3) from real PCC API integration to resident-derived occupancy — ripple updated through TD §§1/2.1/3.2/4/5/6/7/9/10/11. See Round 1 Finding 2 for the superseded original.

**2026-08-21, revision 2:** TD updated for PRD v1.8 (§5.4 Post Discharge Follow-up additions) — new §3.9 (task dedup redesign, resident-level Additional Notes, AI-summary decision), ripple through §§1/4/5/6/7/8/9/10/11. See PRD Review Round 2 above for the corresponding findings.

## Escalation

Round 2. Not applicable yet — escalation only triggers after 3 rounds without landing on `Approved-as-is`/`Approved-with-changes`. Both rounds have landed on `Approved-with-changes` without a round-trip: Round 1's seven findings didn't require a PRD rewrite (Findings 1, 3, 6 informational; 2, 4, 5 tracked as spikes/decisions, not blocking defects; 7 a one-line terminology fix). Round 2's three findings are the same shape — Findings 2 and 3 are confirmations, not defects; Finding 1 is a recommended wording fix (§5.4/Open Question #3) Product Manager can make directly, with the actual technical decision behind it already made and reflected in the TD.
