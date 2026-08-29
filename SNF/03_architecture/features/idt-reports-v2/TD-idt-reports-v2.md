# Technical Design: feature-idt-reports-v2

| Field | Value |
|---|---|
| Source PRD | SNF/02_prd/features/prd-idt-reports-v2.md (v1.0, 2026-08-12; updated through the round-3 handoff, `idt-reports-handoff-round3.md`) |
| Author | Dev team (schema and lifecycle design, marked **[Team-submitted]** below) + System Architect persona (everything else, marked **[SA-authored]**, plus a small number of schema amendments marked **[SA amendment]**) |
| Status | Draft — converted from team-submitted PDF (`architecture-submissions/feature-idt-reports-v2/td_idt_reports_v2.pdf`) and extended into the standard template |
| Reviewers | System Architect persona (in review — see `feature-idt-reports-v2_SA-comments.md`) |
| Product | SNF |

> **Provenance note.** The team's original submission (schema tables and lifecycle interactions)
> is reproduced faithfully below, marked **[Team-submitted]**. The sections their PDF didn't cover
> at all — context/goals, alternatives, API impact, non-functional considerations, testing, rollout,
> and risks — are marked **[SA-authored]**. A small number of concrete schema fixes for gaps that
> were always engineering decisions (not product ambiguities) are marked **[SA amendment]**.
> Decisions Sathish has since made directly are marked **[Confirmed by Sathish]**. Everything still
> genuinely unresolved is called out in §11.

---

## 1. Context and problem statement *(SA-authored)*

The PRD requires replacing the existing IDT Reports feature end-to-end: a new three-surface UI
(list, form, summary), a two-axis status model (document status + signature state, replacing the
current three-value `Draft`/`Pending`/`Submitted` status), server-computed per-discipline completion,
and full version/audit history strong enough to reproduce exactly what a physician signed. The
existing IDT Reports collection has never been used in production and holds only stale test data, so
this is a clean schema rewrite with a data cleanup step, not a migration — but it's still a full
rewrite of the persistence layer for a document that becomes part of the clinical record, so it
warrants a design doc rather than being scoped ad hoc during implementation.

## 2. Goals and non-goals *(SA-authored)*

### 2.1 Goals
- Replace `idtreports`' schema and status model per PRD §6/A7, reusing the collection in place —
  no migration is required, since the existing data is all pre-production test records (PRD §6).
- Support the full report lifecycle server-side: Draft → Complete (one-way), and an independent
  signature state that can cycle (Not sent → Awaiting → Signed → edited-while-signed → re-sent →
  re-signed) without limit, per PRD §6.
- Make every signed version of a report exactly reproducible later, per PRD §9's version-integrity
  requirement.
- Correctly represent that a resident can have multiple assigned Case Managers and Social Workers
  **[Confirmed by Sathish — by design]** — the list's Case Manager/Social Worker filters and columns
  must not silently drop staff.
- Keep the list's hot-path query (tab split, search, filter, sort — including Attending MD) fast at
  the stated scale (500 residents/facility; 52 reports/resident/year) — **[Confirmed by PM, per PRD
  update]** PRD §9 now states explicitly that this 52/year figure is a planning-ceiling placeholder,
  not derived from any cadence, retained as a round conservative number pending real deployment data.
  No index or query design changes as a result — see §11 item 15 (resolved).
- Reuse the existing PDF/print and fax infrastructure rather than rebuilding it, per PRD §5.6, and
  ship fax as part of this release **[Confirmed by Sathish]**.
- Join the existing staff mobile app signature-notification workflow as a third document type
  (alongside the Health Referral Order Summary and Medication List), rather than building new
  notification infrastructure, per PRD §2.1/§5.4/§7 — see §3.11.
- **[Confirmed by Sathish, per PRD update]** Support reports created on no fixed cadence — a resident
  may get a new report whenever staff need one, not on a weekly (or any other calendar) rule, per PRD
  §1/A1. The app neither assumes nor enforces a cadence, and no facility-configurable cadence setting
  is introduced — staff-driven timing was explicitly chosen over both of those alternatives.

### 2.2 Non-goals
- A system-decided or facility-configurable report cadence **[Confirmed by Sathish, per PRD update]**
  — considered and rejected in favor of purely staff-driven timing (PRD §1/A1); see §4 for the
  rejected alternative and §3.6/§11 for what replaces the old calendar-based duplicate-report
  safeguard this decision removes.
- Editing clinical source data (medications, diet, code status, weights, demographics) — these stay
  PCC-sourced and read-only, per PRD §2.2.
- Pushing signed reports to PCC as a document/Progress Note — blocked on PRD §11 item 1 (PCC
  document-API capability question), out of scope until that's answered.
- Structured wound documentation, Speech Therapy "Import Report," dictation, autosave, per-discipline
  explicit sign-off, and a physician queue view — all explicitly deferred per PRD §10.
- Visit-frequency tracking for PT/OT/SLP — that belongs to the therapy plan of care, not this report
  (PRD §3, §5.2). A structured cognitive score (e.g. BIMS) — cognition is a free-text narrative only
  (PRD §3, §5.2).
- Assigning an IDT report to a specific staff member for completion tracking **[Confirmed by
  Sathish]** — out of scope; per-discipline completion (§3.4) already covers who's done what, and no
  "assigned to" field is added to the report itself.
- Redesigning the platform's authentication or role model — this design assumes the existing staff/
  role directory and only specifies which existing roles gate which action (PRD §8).

## 3. Proposed design

### 3.1 Architecture summary *(SA-authored)*

No new services are introduced. This is a data-model and business-logic change inside the existing
`senior_living_backend`, touching: the `idtreports` collection (rewritten schema, reused in place),
two new collections (`idtReportVersions`, `idtReportDeletionLogs`), the existing `residents`
collection's existing `dischargeDate` field (reused directly — no new field, per §3.4), the existing
IDT report controller/service layer, and the resident picker used by the Create action (§6/§3.9 —
multi-select, scoped to the requesting staff member's assigned residents, alphabetically sorted and
searchable). The existing PDF-generation service, the existing fax infrastructure, and the existing
staff mobile app notification workflow (§3.11 — new to this feature, but not a new service) are
integration touchpoints, not new components — see §6.

### 3.2 Collections overview *(Team-submitted, with one correction)*

| Collection | Purpose | Volume expectation |
|---|---|---|
| `idtreports` | The report itself — current state only | ~26,000/year per facility, per PRD §9's 500 residents × 52/year — **[Confirmed by PM]** an explicit planning-ceiling placeholder, not cadence-derived; grows until the 10-year retention cutoff (§3.8) |
| `idtReportVersions` | Full content snapshot on every Save/Submit — the version/audit history | Largest-growing collection, since edits are unlimited for the life of a report; bounded in practice by the same 10-year retention cutoff as its parent report (§3.8) |
| `idtReportDeletionLogs` | Minimal record of a hard-deleted draft | Small, rare writes |
| `residents` (existing) | **No schema change.** The existing `dischargeDate` field is reused directly (§3.4) — the submission's originally proposed new `anticipatedDischargeDate` field is dropped per Sathish's decision. | No change |

`idtReports` holds only the current version of a report and is read constantly (every list load,
every time a report is opened), so it needs to stay small and fast. Full history lives in
`idtReportVersions` instead of being embedded, so the hot document doesn't grow unbounded.

### 3.3 Lifecycle interactions *(Team-submitted, with amendments noted)*

**Create (single resident, and once per resident within bulk-create, §3.9) — [SA-authored addition;
this step was previously undocumented.]** Referenced elsewhere in this design (§3.9, §8) as if already
specified here; it wasn't. Filling that gap:
1. Resident(s) come from the eligible-residents picker (§3.9), already scoped and filtered to exclude
   anyone who already has an open Draft report — replacing what used to be a calendar-week exclusion,
   now that PRD §1/A1 confirms there is no fixed cadence (§3.6).
2. For this `residentId`, query the most recent `idtreports` document of **any** status (Draft or
   Complete), ordered by **`createdAt` descending — [SA amendment, per PRD round-2 update, see the
   `reportDate` mutability note below]**, not `reportDate`. Per PRD §4.2, an abandoned draft still
   counts as the most recent report for carry-forward purposes.
3. Auto-populated data — `patientInformationSnapshot`, `weightTrendSnapshot`, `medicationsSnapshot` —
   is always freshly resolved from current backend data at this moment, regardless of what the prior
   report (if any) contained. Per PRD §4.1/§4.2, these fields are **never** carried forward. This same
   resolution logic is reused later by **Refresh** (see below), which lets the user re-run it on an
   already-open report instead of only at Create time.
4. Team-entered content — `clinicalInformation`, `rehab`, `dischargePlanning`, `ordersDiscussion` — is
   copied from the prior report found in step 2, if one exists; otherwise every field starts at its
   schema default (empty). **[Confirmed by Sathish]** `rehab.markedComplete`
   is the one exception: it does **not** carry forward with the rest of `rehab` and always starts
   `false` on a new report. A completion checkbox is a signal about *this particular* assessment, not
   carried-forward content — this reading of PRD §4.2 is confirmed correct.
5. `disciplineCompletion` is computed fresh from whatever content resulted from steps 3–4, using the
   normal content-derived rule (§3.4) — it is a derived value, not itself something to carry forward.
6. Create the `idtreports` document immediately: `status = 'DRAFT'`, `reportDate = now` — no
   `weekOf` or other period field is set, since none exists on this document anymore (§3.4, §3.6). The
   Draft is persisted at Create time, not deferred until the user's first Save — this is what makes
   bulk-create's "N reports in one shot" (§3.9) possible in the first place.
7. Append the first `idtReportVersions` row, exactly as any other save does (step 7 below).

**Start blank (PRD §4.2) — [Confirmed by Sathish].** Modeled here as a
client-side form reset only: it clears the just-created Draft's carried-forward fields back to empty
in the open form and dismisses the banner, with no dedicated backend call. Nothing about the
already-persisted Draft from step 6 above changes until the user's next ordinary Save, which then
simply saves whatever the form currently holds — cleared or not — via the normal Save flow below. The
lighter client-side-only reset is confirmed sufficient; Start blank does not need to immediately
overwrite the already-persisted Draft.

**Refresh auto-populated data — [SA-authored addition, per Sathish's decision on §11 item 3].**
Answers the staleness question for a report left open a long time as a Draft or while
Awaiting-signature: rather than a passive staleness indicator, or leaving the data frozen with no way
to update it, the report gains an explicit **Refresh** action the user can trigger on demand.
1. A **Refresh** control sits with the auto-populated sections of an open report (Patient Information,
   Medications Details, and the weight trend row) — one action refreshes all three together, since
   they're resolved by the same step-3 logic above and always change as a set.
2. Refresh re-runs the exact same resolution used at Create step 3 — `patientInformationSnapshot`,
   `weightTrendSnapshot`, `medicationsSnapshot` are freshly re-derived from current backend/PCC-synced
   data as of the moment Refresh is pressed. Nothing else on the report (team-entered content, status,
   signature state) is touched.
3. This is a **read, not a write** — a lightweight endpoint (§6) resolves and returns the current
   values; nothing is persisted to `idtreports` yet. The open form's auto-populated display updates to
   show the refreshed values, exactly as if the user had just reopened a brand-new report.
4. Persisting the refreshed values is not a special case — it happens via the ordinary **Save** flow
   below, the next time the user saves. **[SA amendment, design review — corrects an earlier statement]**
   That save's `idtReportVersions` row now also captures the refreshed
   `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` alongside whatever
   team-entered content is current (§3.4) — an earlier pass at this design said "no new version-row
   shape is needed," which was true only because those three fields used to be invariant across every
   version; now that Refresh can change them, the version row needs to capture them too, or a
   post-signature refresh would leave no record of what the auto data looked like at signing time. **[PRD
   round-2 update]** That same save also moves `reportDate` to the save date — see the dedicated note
   below, since this has ripple effects across several other parts of this design.
5. Refresh is available whenever the report is open for editing, which per PRD §6 rule 1 is at every
   point in a report's life, including after signature — refreshing a signed report's auto data and
   then saving is just a "correction after signature" (§6 rule 4): recorded in the audit trail, no
   stamp or banner. No new UI treatment is introduced for this case.
6. **Not in scope for this addition:** an automatic/background refresh, a staleness indicator or "as
   of" timestamp on the auto sections, or a diff view showing what changed since the last resolution —
   Sathish's direction was a manual refresh button specifically, not these alternatives; they remain
   available as future enhancements if a staleness indicator turns out to still be wanted alongside the
   button.

**`reportDate` mutation on Refresh + Save — [Confirmed by Sathish, PRD round-2 update, resolving the
ripple effects the PRD's own handoff note flagged for review — see §11 item 16].** PRD §4.1/§4.3 now state that
saving after a Refresh moves the report's `reportDate` to that save date, "since the data now pertains
to that day." This is a genuine, deliberate exception to `reportDate` being otherwise immutable — but
`reportDate` was also this design's sole ordering key for "most recent report" (carry-forward, Create
step 2), "two most recent prior reports" (rehab progression trend), the general by-resident lookup
index (§3.6), and the retention-cleanup anchor date (§3.8). If `reportDate` can move on an existing
document, an older Complete report that gets reopened, refreshed, and saved could jump *ahead* of a
genuinely newer report in `reportDate` order — corrupting carry-forward's "most recent," the rehab
trend's report sequence, and (most seriously) resetting a report's retention clock indefinitely just by
refreshing it periodically.

**Resolution:** decouple the two concepts that `reportDate` was doing double duty for. `reportDate`
keeps its new PRD-defined meaning — "the date this report is filed under," mutable via Refresh + Save —
and is used only for display (the list's Report Date column, the report/summary/PDF header) and for the
weight-trend anchor (§3.4, which re-derives alongside `reportDate` on the same Refresh anyway, so the
two stay consistent with each other). Every place that actually needs **creation order** — "most recent
report," "two most recent prior reports," the general by-resident lookup index, and the retention
anchor — switches to the existing, genuinely-immutable **`createdAt`** field (§3.4, "Fax, PDF, and
record-keeping") instead. `createdAt` and `reportDate` hold the same value for the overwhelming majority
of reports (any report never refreshed-and-resaved); they only diverge once a report is refreshed, which
is exactly the case this fix needs to handle correctly. **[Confirmed by Sathish]** — `reportDate`'s job
is specifically to capture the report's date as revised by a Refresh + Save, not to double as the
ordering/retention key; the alternative of leaving `reportDate` as the sole ordering key, accepting the
sequencing/retention risk above, is recorded in §4 as rejected.

Mechanism for detecting "this save follows a refresh," since Refresh itself is a stateless read (step 3
above) with nothing persisted in between: **at Save time, compare the incoming
`patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` against the currently persisted
values.** If any differ, this save carries refreshed data — set `reportDate = now` as part of the same
write. If they're identical (the ordinary case — no Refresh happened since the last save), `reportDate`
is left untouched. This is server-authoritative (no client-supplied flag to trust) and needs no new
request field; the diff is cheap given the small size of the three snapshot objects.

**Save (including Submit):**
1. User edits `clinicalInformation`, `rehab`, `dischargePlanning`, and/or `ordersDiscussion`.
2. Recompute `disciplineCompletion` — content-derived for Case Manager/Social Worker, checkbox-derived
   for Rehab, **unless** this save is a Submit, in which case all three are forced `COMPLETE` (and
   `rehab.markedComplete` is force-persisted `true`).
3. If this is the first transition to Complete: set `status = COMPLETE`, `completedAt`, `completedBy`.
4. If `ordersDiscussion.dischargeDateSet` + a date are present: propagate to `residents.dischargeDate`
   (the existing field — see §3.4).
5. If `signature.state === 'SIGNED'` at this moment: set `signature.editedAfterSignAt = now`.
6. **[PRD round-2 update]** If the incoming `patientInformationSnapshot`/`weightTrendSnapshot`/
   `medicationsSnapshot` differ from what's currently persisted (i.e., this save follows a Refresh, per
   the note above): set `reportDate = now`. Otherwise `reportDate` is untouched.
7. Overwrite the live `idtreports` document with the above.
8. Append a new `idtReportVersions` row (`isAsSignedVersion: false`).

**Send for signature — [SA-authored addition; this step was previously undocumented, folded only
into schema notes and the signer-resolution note below.]**
1. Verify `status === 'COMPLETE'` — PRD §5.4 requires Submit before Send is enabled.
2. Resolve the eligible signer pool live (see the signer-resolution note below).
3. The user selects **exactly one** recipient from that pool — even when the pool holds more than one
   eligible signer — matching the single-select `sentTo` field (§3.4); set
   `signature.state → 'AWAITING'`, `signature.sentTo`, `signature.sentAt = now`, and append one entry
   to `signature.submissions`.
4. Dispatch a signature-request push notification to `signature.sentTo` via the existing staff
   mobile app notification workflow (§3.11) — new per PRD §2.1/§5.4/§7.

**Sign:**
1. The physician or PA/NP named in `signature.sentTo` signs (enforced: caller must equal `sentTo`;
   exactly one signer per send, per §3.4).
2. Update `signature`: `state → 'SIGNED'`, `signedAt`, `signedBy`, `editedAfterSignAt → null`.
3. No new `idtReportVersions` row — content is unchanged.
4. Clear `isAsSignedVersion` on whichever row previously held it (if any), and set it `true` on the
   most recent `idtReportVersions` row for this report.
5. Dispatch a signed push notification to every staff member in this report's `residents.assignedStaff`
   via the same existing notification workflow (§3.11) — new per PRD §2.1/§5.4/§7.

**Edit after signing:** not a separate mechanism — it's the Save flow above running while
`signature.state` already equals `'SIGNED'`. The only differentiator is the Save flow's step 5 (set
`signature.editedAfterSignAt = now`). **[SA amendment, PRD round-2 update, resolves handoff round 2
item 8]** Previously nothing was surfaced in the UI for this; PRD §5.2/§6 rule 4 now add an "Edited
after signature [date]" indicator as the last item in the report form's sub-title, once
`signature.editedAfterSignAt` is non-null. **[Confirmed by Sathish, design review]** The indicator's
wording is being extended to also show the *current* signature status alongside the date, not just the
edited-after-signature fact and date on their own — for example, something like "Signed — edited after
signature on 03/14/26" rather than just "Edited after signature on 03/14/26" — so a user reading the
form sub-title doesn't need to separately check the Completed tab's Signature Status column (§6) to
know where the report currently stands. This still needs no new field: `signature.state` (§3.4) and
`editedAfterSignAt` are both already available and can be exposed together on the report-get response
(§6). The exact wording is a PRD content decision — see `feature-idt-reports-v2_SA-comments.md` for
the suggested text handed to the PM. `editedAfterSignAt` clears back to `null` on the next successful
sign, as already designed above. The summary and list remain unchanged (PRD §5.2); this is form-only.
The drift is otherwise still detectable via `editedAfterSignAt`, or by comparing current content
against the version flagged `isAsSignedVersion: true`.

**Delete (Draft only):**
1. Verify `status === 'DRAFT'` — reject otherwise.
2. Delete the `idtreports` document.
3. Delete all associated `idtReportVersions` rows.
4. Write one row to `idtReportDeletionLogs`.

**Bulk delete — the same flow above, run independently per report.** **[Confirmed by Sathish — now
in scope, PRD §2.1/§5.1]** There is no separate bulk-delete mechanism at the data layer: steps 1–4
above run once per report `_id` in the selected set, exactly as a single delete would. See §3.10 for
the endpoint shape, the count-based confirmation, and per-report failure handling.

**Rehab progression trend (§3.8 of the submission) — not stored, computed at read time. [SA amendment
— relabeled from "week-over-week" per the PRD's removal of any fixed report cadence.]** The summary's
report-over-report PT/OT trend table is computed on read by querying the two most recent prior
`idtreports` documents for the same `residentId` (by `createdAt`, descending — **[SA amendment, PRD
round-2 update]** not `reportDate`, which can now move on Refresh + Save (see the dedicated block in
§3.3 above); and not `weekOf`, which no longer exists on this document, §3.4/§3.6) and pulling their
`rehab.pt`/`rehab.ot` values directly
(including `dmeUsed`, per PRD §5.4's confirmed field list). If fewer than two prior reports exist, the
corresponding column renders N/A. This matches PRD §5.4: "2 Reports Ago"/"Last Report" mean the two
literal most recent prior reports in report sequence, regardless of how far apart in time they
actually fall — cadence between IDT reports is staff-driven, not fixed (PRD §1/A1). This is
deliberately a different rule from the weight trend's calendar-week slots (§3.4 below), which stay
calendar-based on purpose — weight monitoring in a SNF typically follows its own regulatory weighing
schedule, independent of when an IDT report happens to be filed.

**Signer resolution (§3.9 of the submission) — not stored, resolved live.** The eligible-signer pool
is not a snapshotted field. It's resolved live, at the moment "Send for signature" is opened, so a
long-`AWAITING` report doesn't offer a stale, no-longer-assigned signer. **The pool itself can contain
more than one person** — the attending MD plus one or more PA/NPs — **but the user selects exactly one
of them to send to.** This mirrors the Create picker's pattern of presenting a list (§3.9) but is
deliberately single-select, not multi — see the schema note under `signature.sentTo` below.

> **[Resolved, PRD round 3 item 7 — RESOLVED, no facility setting needed]** The submitted query was
> `Staff.find({ facilityId, cName: { $in: residents.assignedStaff }, designation: 'Physician' })`,
> which excludes PA/NP entirely. An intermediate design (this round's prior pass) amended this to read
> a new per-facility `facilities.idtReportSettings.eligibleSignerDesignations` list, since the PA/NP
> designation label was believed to vary by facility with no single fixed value. **After consulting
> Engineering, the PM confirms this is unnecessary**: the `staff` object already carries a
> `mobile_access` field, valued `'Doctor'` for every signer-eligible user (the attending MD and their
> PA/NP alike). The signer-resolution query is now:
> `Staff.find({ facilityId, cName: { $in: residents.assignedStaff }, mobile_access: 'Doctor' })`
> — no facility-configurable designation list, no per-facility rollout step, and no remaining
> engineering discussion needed for this item. This is now fully settled, not merely amended pending
> discussion. See §3.7 and §9 for the corresponding removal of the facility-setting and rollout step
> this replaces.

### 3.4 Auto-populated and team-entered field tables *(Team-submitted, with amendments noted)*

**`idtreports` — identity & scoping**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `_id` | ObjectId | Yes | auto | Primary key |
| `facilityId` | String | Yes | — | Multi-tenancy scope |
| `residentId` | ObjectId → `residents` | Yes | — | Single canonical link |
| `reportDate` | Date | Yes | — | Set to `now` at creation; **[Confirmed by Sathish, PRD round-2 update — supersedes the "immutable" note below]** no longer immutable — moves to the save date when a Save follows a Refresh (see the dedicated "`reportDate` mutation" block in §3.3). Not the meeting date. Display/filing date only — **not** the ordering/anchor key for carry-forward, rehab trend, the general lookup index, or retention (those all use `createdAt`, which stays genuinely immutable; see §3.4's "Fax, PDF, and record-keeping" table below, §3.6, §3.8). `weekOf` (Monday 00:00, facility-local time zone) is removed entirely, since PRD §1/A1 confirms there's no fixed cadence for it to represent. |

**Status**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `status` | String (enum: `DRAFT`, `COMPLETE`) | Yes | `DRAFT` | One-way transition, enforced in application logic |
| `completedAt` | Date | No | — | Set once, on first transition to Complete — never overwritten by later re-submits |
| `completedBy` | String | No | — | Same — first-completion attribution only |

**Signature (embedded object)**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `signature.state` | String (enum: `NOT_SENT`, `AWAITING`, `SIGNED`) | Yes | `NOT_SENT` | Independent of `status` |
| `signature.sentTo` | String (staff identifier) | No | — | Single recipient, chosen from the amended signer pool at send time (§3.3) |
| `signature.sentAt` | Date | No | — | |
| `signature.signedAt` | Date | No | — | |
| `signature.signedBy` | String (staff identifier) | No | — | Must equal `signature.sentTo`, enforced in application logic |
| `signature.editedAfterSignAt` | Date | No | `null` | Set on any save while `state = SIGNED`; cleared back to `null` on the next successful sign. **[SA amendment, PRD round-2 update]** Now surfaced in the UI — the report form's sub-title shows an "Edited after signature [date]" indicator when this is non-null (PRD §5.2/§6 rule 4, handoff round 2 item 8). **[Confirmed by Sathish, design review]** The indicator now also shows the current `signature.state` alongside the date, not the date alone — both fields must be included in the report-get response (§6) |
| `signature.submissions` | Array of `{date, to, by}` | Yes | `[]` | Append-only send history — one recipient per entry |

> **[Confirmed by Sathish]** Signature recipient is single-select, not multi — **even though the
> eligible-signer pool it's chosen from can itself hold more than one person** (the attending MD plus
> one or more PA/NPs, §3.3/§3.7). `sentTo`/`signedBy` stay a single `String` — no schema change
> needed; this closes the submission's own open question.

**Auto-populated data (frozen at creation, only re-read on an explicit user-triggered Refresh — §3.3) —
`patientInformationSnapshot`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `patientInformationSnapshot.residentName` | String | Yes | — | |
| `patientInformationSnapshot.room` | String | Yes | — | |
| `patientInformationSnapshot.dob` | Date | Yes | — | |
| `patientInformationSnapshot.attendingMD` | Array of `{name, practitionerId?}` | No | `[]` | |
| `patientInformationSnapshot.admissionDate` | Date | Yes | — | |
| `patientInformationSnapshot.diagnosis` | String | No | — | **[Renamed, PRD round 3 item 6]** Was `chiefComplaint` — the PRD field is renamed "Chief Complaint" → "Diagnosis," a display-layer rename with no change in meaning or source. The PM confirms this field is available from PCC, resolving the Diagnosis half of former §11 item 1 (Code Status remains open, §11 item 2). |
| `patientInformationSnapshot.diet` | String | Yes | `'N/A'` | |
| `patientInformationSnapshot.codeStatus` | String | No | — | |

> **[Diagnosis half resolved, PRD round 3 item 6]** Diagnosis (formerly Chief Complaint) is now
> confirmed sourced from PCC by the PM. **Code Status remains open** — still deliberately left open
> for now (§11 item 2), not overlooked.
>
> **[SA amendment]** The PRD §5.1 Attending MD filter/sort had no supporting index in the submission.
> Rather than adding a separate resolved-name field (the same over-engineering Sathish flagged for
> Case Manager/Social Worker, below), this is fixed by indexing the existing
> `patientInformationSnapshot.attendingMD.name` array directly (§3.6) — no new field needed, since
> Attending MD is already snapshotted here for clinical/print reasons (PRD §4.1: a signed report must
> render years later exactly as signed).

**`weightTrendSnapshot`** — **[Confirmed by Sathish, resolves §11 item 25]** the recent-3-readings
ring buffer, with **no calendar-week cadence or grouping at all** — this simplifies an earlier design
that anchored 3 slots on calendar weeks relative to the report's `reportDate`. PRD §4.1 is explicit:
"a rolling set of 3 — a new reading replaces the oldest once a fourth exists — with no calendar-week
grouping or bucketing." `weightTrendSnapshot` now directly mirrors `residentObservationTrends.readings`
(below) at the moment of Create or Refresh — no anchor date, no week math, nothing left to derive.

**[SA amendment, design review — upstream dependency, resolved, see §11's closing note on former item
19]** This section describes how
`weightTrendSnapshot` behaves once populated, but the underlying resident-level weight-trend data it
snapshots from doesn't exist as a built capability yet. The real `observations` collection has now been
reviewed, which lets this be designed concretely rather than left as a placeholder.

**What exists today:** `observations` is a shared, cross-feature, append-only event log — one document
per PCC observation event (not per resident), field-level encrypted (each clinical field individually
AES-256-GCM encrypted with its own IV/auth tag), scoped by `residentId`/`pcc_patientId`, distinguished
by `type` (`"weight"` is one of presumably several observation types this same collection carries).
Written once per event via the PCC webhook; nothing in it is currently bucketed by calendar week or
capped to a recent window — every event since PCC integration began is presumably retained.

**Design — [Confirmed by Sathish] a new, separate `residentObservationTrends` collection, not a change
to `observations` itself.** `observations` keeps its full per-event history exactly as it does today,
untouched; this new collection is a small, fast-read derived cache holding only the 3 most recent
readings per resident per type — **[Confirmed by Sathish]** older readings are simply not retained
here (they remain fully available in `observations` if ever needed). Update rule: **the latest reading
replaces the oldest one on every update** — a fixed-size ring buffer, not a per-calendar-week bucket.
**[Confirmed by Sathish]** Vitals (including weight) are PHI, and this data stays field-level encrypted
exactly like `observations` — see the note below on the one plaintext field this still needs.

```
{
  _id: ObjectId,
  facilityId: "S101",
  facId: "12",
  residentId: ObjectId,           // FK to residents
  pcc_patientId: 6209,
  cName: "...",
  type: "weight",                  // one document per (residentId, type)
  readings: [                      // fixed-size ring buffer, at most 3 entries, oldest → newest
    {
      sourceObservationId: ObjectId,  // reference to the source `observations` document (its `_id`),
                                       // for traceability back to the full encrypted event record
      recordedAt: Date,               // plaintext shadow of the encrypted recordedDate — the only
                                       // plaintext field in this collection beyond identity/scoping;
                                       // needed to order readings and evict the oldest one
      observation_data: {             // same field-level encryption pattern as `observations`, limited
                                       // to what's actually needed for trend display
        value:    { alg: "AES-256-GCM", ciphertextB64, ivB64, authTagB64 },
        unit:     { alg: "AES-256-GCM", ciphertextB64, ivB64, authTagB64 },
        unitCode: { alg: "AES-256-GCM", ciphertextB64, ivB64, authTagB64 }
      }
    }
    // ... up to 2 more entries
  ],
  createdAt: Date,
  updatedAt: Date
}
```

**Webhook update logic:** on each new observation event of this type for this resident — (1) write the
full event to `observations` exactly as today, unchanged; (2) upsert `residentObservationTrends` for
`{residentId, type}`, pushing the new `{sourceObservationId, recordedAt, observation_data}` entry and
evicting the oldest by `recordedAt` once more than 3 remain (a single atomic operation — MongoDB's
`$push` with `$each`/`$sort`/`$slice: -3` does this in one call). No per-week deduplication logic: if
PCC sends more than one reading in the same week, all of them cycle through the buffer in order, same
as any other reading — **[Confirmed by Sathish, resolves §11 item 25]** this is no longer a concern to
flag, since `weightTrendSnapshot` doesn't assume distinct calendar weeks either, now that both this
collection and the report-level snapshot are the same plain 3-reading ring buffer end to end.

**`weightTrendSnapshot` mirrors this collection directly — no calendar-week computation anywhere.**
At Create or Refresh, `weightTrendSnapshot` is populated straight from `residentObservationTrends
.readings` for this resident's `"weight"` type: the same 0–3 entries, in the same oldest→newest order,
each carrying its `date` (from `recordedAt`) and `lb` (from `observation_data.value`). No `weekOf`, no
anchor date, no derivation logic of any kind — this is a plain copy at the moment of Create/Refresh, not
a computation.

**Encryption — [Confirmed by Sathish]: vitals are PHI, encryption stays.** Weight (and vitals generally)
tied to an identified resident are unambiguously PHI under HIPAA's broad definition of individually
identifiable health information. On whether HIPAA strictly *mandates* encryption: under the Security
Rule currently in effect, encryption of ePHI at rest and in transit is an "addressable," not a flatly
"required," implementation specification — a covered entity must assess whether it's reasonable and
appropriate, and if it chooses not to encrypt, must document why and apply an equivalent alternative
safeguard ([Kiteworks](https://www.kiteworks.com/hipaa-compliance/hipaa-encryption-requirements-safe-harbor-guide/)).
In practice this has never meant optional: encrypted PHI gets safe-harbor treatment under the Breach
Notification Rule, so encryption is the de facto industry-standard control. HHS has proposed (a January
2025 NPRM) eliminating the "addressable" category entirely and making encryption a hard requirement,
but that rule is not yet finalized — OMB currently targets mid-2027 for final action
([Medcurity](https://medcurity.com/hipaa-encryption-requirements/)). Net effect either way: this data
should stay encrypted, which is what this design already does — nothing changes here except confirming
the reasoning. (Not legal advice — worth a confirmation from actual compliance/legal counsel given a
real regulatory change is in motion.)

**Only `recordedAt` is plaintext — [Confirmed by Sathish]**, for the reason above (ordering/eviction
without decrypting on every webhook event and every read) — a bare date carries meaningfully less risk
than the value it's attached to, and `createdAt`/`updatedAt`/`type` are already plaintext on the
existing `observations` document. The proposed object shape and approach are confirmed as-is; no
further change is needed here.

**No window-advance job is needed** — the `readings[]` array only changes when a new webhook event
arrives; "how stale is this" is a **read-time** calculation (compare the newest `recordedAt` against
today's date), the same approach `weightTrendSnapshot` already uses for N/A-rendering gaps. A resident
who hasn't been weighed recently just keeps showing their last 3 known readings until a fresh one
arrives — no scheduled job has to "roll" anything forward. See §12 for a related, explicitly deferred
idea: alerting when a resident's data has gone stale for too long.

This is a cross-feature dependency, not part of the IDT Reports codebase — likely owned by whoever owns
the Resident Info/patient-record feature and the PCC integration layer, with IDT Reports (and the
Resident Info UI) as downstream consumers reading `residentObservationTrends` once it exists.

`weightTrendSnapshot` itself is an array of **0 to 3** entries (no fixed length, no padding with
nulls) — **[Confirmed by Sathish, resolves §11 item 25]** whatever `residentObservationTrends
.readings` currently holds for this resident's weight, copied as-is, oldest→newest. The UI pads to 3
display slots and renders N/A for any that don't exist (PRD §4.1), rather than this array storing
placeholder entries itself.

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `weightTrendSnapshot[].date` | Date | Yes | — | The reading's actual date, copied from `residentObservationTrends.readings[].recordedAt`. Per §3.4a's confirmed encryption scope, this field is field-level encrypted on `idtreports` (same `{alg, ciphertextB64, ivB64, authTagB64}` shape as the rest of this document's clinical content) even though its source field in `residentObservationTrends` is plaintext — the two collections don't need matching encryption treatment field-for-field. |
| `weightTrendSnapshot[].lb` | Number | Yes | — | Copied from `residentObservationTrends.readings[].observation_data.value` (decrypted at copy time). Field-level encrypted on `idtreports`, per §3.4a. |

**`medicationsSnapshot`**

| Field | Type | Required | Default |
|---|---|---|---|
| `medicationsSnapshot[].name` | String | Yes | — |
| `medicationsSnapshot[].dose` | String | No | — |
| `medicationsSnapshot[].route` | String | No | — |
| `medicationsSnapshot[].frequency` | String | No | — |

**Team-entered content — `clinicalInformation`** (exclusively staff-entered; weight is deliberately
kept out of this object even though the PRD's UI groups it visually in the same card — see the
alternative in §4)

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `clinicalInformation.changeInCondition` | String | No | — | |
| `clinicalInformation.skinWoundStatus` | String | No | — | Free text; structured wound measurement is a future phase |

**`rehab`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `rehab.priorLevelOfFunction` | String | No | — | Free text |
| `rehab.weightBearingStatus` | String (enum §3.5 — Weight-Bearing Status Scale) | No | — | |
| `rehab.pt.active` | Boolean | Yes | `false` | |
| `rehab.pt.bedMobility` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.pt.supSitTransfers` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.pt.sitStandTransfers` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.pt.gait` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.pt.gaitDistanceFeet` | Number | No | — | Minimum 0 |
| `rehab.pt.dmeUsed` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.pt.goal` | String | No | — | Free text |
| `rehab.ot.active` | Boolean | Yes | `false` | |
| `rehab.ot.upperBodyDressing` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.ot.lowerBodyDressing` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.ot.toiletTransfers` | String (enum §3.5 — Standard Assist-Level Scale) | No | — | |
| `rehab.ot.goal` | String | No | — | Free text |
| `rehab.slp.active` | Boolean | Yes | `false` | |
| `rehab.slp.notes` | String | No | — | Free text (swallowing, communication, diet tolerance). No "imported report" field — Import Report is deferred to a later phase, no storage need defined yet |
| `rehab.cognition` | String | No | — | Free text narrative — PRD §3/§5.2 confirm no cognitive score or BIMS entry |
| `rehab.markedComplete` | Boolean | Yes | `false` | Forced `true` and persisted on Submit, regardless of prior state. **[PRD round 3 item 3]** Once `status = 'COMPLETE'`, the checkbox this field backs becomes disabled/read-only in the UI (still shown ticked) — document status only moves forward, so there's nothing further for the checkbox to control at that point. Frontend-only state change; no new field, no backend enforcement needed beyond the existing one-way `DRAFT → COMPLETE` rule already in place (§3.3, §6 rule 2). |

**Rehab discipline clear-on-deactivate — [Confirmed by Sathish, PRD round 3 item 2].** Turning off
`rehab.pt.active` or `rehab.ot.active` (the two disciplines with chip-rated fields) after any of that
discipline's rated fields have been entered now requires a client-side confirmation dialog (already
part of this design) that warns the user before the toggle actually turns off; declining leaves the
toggle on and nothing is cleared. On confirm: clear **all** of that discipline's fields — the
chip-rated assist-scale fields (`rehab.pt.bedMobility`/`supSitTransfers`/`sitStandTransfers`/`gait`/
`dmeUsed`, or `rehab.ot.upperBodyDressing`/`lowerBodyDressing`/`toiletTransfers`), for PT also
`rehab.pt.gaitDistanceFeet`, and **[Confirmed by Sathish]** the free-text `rehab.pt.goal`/`rehab.ot.goal`
field too — then set `active = false`. No confirmation is needed, and nothing needs clearing, if none
of that discipline's rated fields were ever entered. Speech Therapy (`rehab.slp`) has no rated fields,
so this behavior doesn't apply to it. This is primarily a client-side interaction (confirmation dialog
+ clearing the in-progress form state before the next Save persists it) — no new endpoint, since it
rides the ordinary Save flow (§3.3) like any other edit. **[Confirmed by Sathish, closes §11 item 23]**
Turning the discipline back on afterward **starts fresh** — it does not restore the just-cleared
values, exactly like a brand-new, never-activated discipline. This closes the last open point on this
feature; nothing further is undecided here.

**`dischargePlanning`**

| Field | Type | Required | Default |
|---|---|---|---|
| `dischargePlanning.destination` | String | No | — |
| `dischargePlanning.dmeNeededOwned` | String | No | — |
| `dischargePlanning.caregiverNeeded` | String | No | — |
| `dischargePlanning.notes` | String | No | — |

**`ordersDiscussion`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `ordersDiscussion.continueSkilledCare` | Boolean | Yes | `false` | Independently selectable alongside the field below — confirmed non-exclusive |
| `ordersDiscussion.dischargeDateSet` | Boolean | Yes | `false` | |
| `ordersDiscussion.dischargeDate` | Date | No | `null` | **[Confirmed by Sathish]** Valid range is **today or any later date** — only strictly-past dates are rejected, to accommodate late entries. Represents an *anticipated* date, not a final/confirmed discharge |
| `ordersDiscussion.notes` | String | No | — | |

**Side effect, not a stored field on this document:** when `dischargeDateSet` and `dischargeDate`
are saved, the value is propagated to `residents.dischargeDate` — **[Confirmed by Sathish]** the
existing field, not a new one — so other modules can consume it. See the residual note under
"Modified collection: `residents`" below.

**Derived / denormalized (recomputed on every save) — `disciplineCompletion`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `disciplineCompletion.caseManager` | String (enum §3.5 — Discipline Completion Status) | Yes | `PENDING` | Content-derived while Draft; forced `COMPLETE` on Submit, unconditionally |
| `disciplineCompletion.socialWorker` | String (enum §3.5) | Yes | `PENDING` | Same rule |
| `disciplineCompletion.rehabTeam` | String (enum §3.5) | Yes | `PENDING` | Driven by `rehab.markedComplete`; also forced `COMPLETE` on Submit |

> **[SA amendment, superseding an earlier draft of this section]** An earlier version of this design
> proposed denormalizing `assignedCaseManagerCNames`/`assignedSocialWorkerCNames`/
> `assignedAttendingMDName` onto `idtreports`. Per Sathish: **no dedicated designation fields are
> needed at all** — the existing `residents.assignedStaff` field is the single source of truth, and
> nothing about Case Manager/Social Worker assignment needs to be resolved or duplicated onto the
> report. Case Manager and Social Worker are dropped from this table entirely. Where the list needs
> to display or filter by Case Manager/Social Worker, it resolves this **live, at read time**, the
> same way the signer pool is resolved live rather than snapshotted (§3.3):
> `idtreports.residentId` → `residents.assignedStaff` → cross-referenced against `Staff.designation`.
> Nothing is stored on the report itself for this purpose. See §3.6 for the supporting index and §6
> for how the list query performs this join. (Attending MD is handled differently — see the note
> under `patientInformationSnapshot` above and §3.6 — since it's already a frozen per-report snapshot
> for clinical/print reasons, not something that needs a live join.)
>
> This also confirms the underlying permission model needs no change: **[Confirmed by Sathish]** any
> Case Manager or Social Worker can start a new IDT report or complete one already in progress — this
> isn't gated by whether that specific person is "the" Case Manager assigned to that specific
> resident. See the resident-picker note in §6 for the one place `assignedStaff` does matter (which
> residents are *offered* by default when creating a report), which is a UX default, not an
> authorization check.

**Fax, PDF, and record-keeping**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `faxLog` | Array of `{number, actor, sentAt, status}` | Yes | `[]` | **[Confirmed by Sathish]** Fax is in scope for this release — this field is actively written on every fax send (PRD §7), not reserved/deferred. **[SA amendment, PRD round-2 update, resolves handoff round 2 items 5–6]** `number` is now validated as a US phone number (10 digits, formatted/masked as typed) both client- and server-side — the popover no longer offers a saved-contacts shortlist, since the product only stores fax numbers for Home Health Agencies, which IDT reports never target. New `status` field records the send outcome per entry; the exact enum values depend on what the fax-transmission service actually reports back (PRD §12, tracked as the pre-existing fax-vendor engineering discussion in the Spike, SA-comments) — provisionally `PENDING` / `SENT` / `FAILED` until that discussion settles it. The new PRD §5.4 **Fax History** section (collapsed by default, listing every fax's number/sent date-time/actor/status) needs **no new endpoint** — `faxLog` already lives on the `idtreports` document and is already returned whenever a report is fetched, so the frontend renders Fax History directly from the existing report-get response. |
| `pdfUrl` | String | No | — | **[Confirmed by Sathish, design review]** Populated by the existing PDF render pipeline (`buildIdtReportHtml`, `idtReport.pdf.service.ts` — §6), which needs its data access updated to the new schema shape; its output contract (this field) doesn't change |
| `createdBy` | String | Yes | — | |
| `updatedBy` | String | No | — | |
| `createdAt` / `updatedAt` | Date | Yes | auto | Standard timestamps. **[Confirmed by Sathish, PRD round-2 update]** `createdAt` is now this document's sole ordering/anchor key — carry-forward's "most recent report" (§3.3), the rehab progression trend's "two most recent prior reports" (§3.3), the general by-resident lookup index (§3.6), and the retention-cleanup anchor date (§3.8) all key off `createdAt`, not `reportDate`, since `reportDate` can now move on Refresh + Save. |

**[SA amendment, resolves former §11 item 18]** `createdByType` and `updatedByType` are removed
entirely — confirmed by Sathish to be remnants of the previous IDT report structure, not required in
this design. Every other actor-type distinction this design needs is already handled by existing,
purpose-built mechanisms: `deletedBy`'s free-form actor-type strings (e.g. `'system:stale-draft-cleanup'`,
§3.7/§3.8) for scheduled-job deletes, and the plain `createdBy`/`updatedBy`/`savedBy` fields (here and
on `idtReportVersions`) for ordinary attribution — neither needs a companion `*Type` field.

**`idtReportVersions`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `_id` | ObjectId | Yes | auto | |
| `reportId` | ObjectId → `idtreports` | Yes | — | |
| `facilityId` | String | Yes | — | Denormalized for facility-scoped access |
| `residentId` | ObjectId → `residents` | Yes | — | **[SA amendment, design review — new field]** Denormalized for the same reason `facilityId` is: allows resident-scoped audit queries (e.g. "all version history for this resident across all their reports") directly against this collection, without a join back through `idtreports` — which is symmetric with how `facilityId` is already justified above. No prior reason surfaced for the asymmetry; this closes it. |
| `content` | Object (freeform) | Yes | — | Full copy of `clinicalInformation`/`rehab`/`dischargePlanning`/`ordersDiscussion` at save time. **[SA amendment, design review — corrects a bug]** `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` are **no longer invariant** across every version now that Refresh + Save can change them (§3.3's "`reportDate` mutation" block) — the "storage waste" reasoning that justified excluding them was only ever true before that change. As designed here, if a signed report is refreshed and saved again, the version history would have no record of what the auto-populated data looked like at the moment it was signed, breaking the PRD's own "a signed report must render years later exactly as it was signed" requirement (PRD §4.1). Fix: `content` now also captures a copy of `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` on every save, not only when they change — simpler and safer than conditionally including them only on refresh-driven saves. |
| `status` | String (enum: `DRAFT`, `COMPLETE`) | Yes | — | Document status at the moment of this save |
| `isAsSignedVersion` | Boolean | Yes | `false` | Exactly one row per report carries `true` at any time — the version that was current when the report was last signed |
| `savedBy` | String | Yes | — | |
| `savedAt` | Date | Yes | auto | |

A new row is written on every Save and every Submit. Signing does not create a new row.

**`idtReportDeletionLogs`**

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `_id` | ObjectId | Yes | auto | |
| `facilityId` | String | Yes | — | |
| `residentId` | ObjectId → `residents` | Yes | — | |
| `reportDate` | Date | Yes | — | Preserved identity of the deleted report — the sole identity date now, since `weekOf` no longer exists on `idtreports` (§3.4/§3.6) and is dropped from this table too |
| `deletedBy` | String | Yes | — | Set to `'system:stale-draft-cleanup'` for scheduled cleanup deletes (§3.7) so they're distinguishable from user-initiated deletes without a schema change |
| `deletedAt` | Date | Yes | auto | |

Written only when a Draft is hard-deleted. The report's `idtReportVersions` rows are purged along
with it — this one row is what survives, matching "only the audit entry retained."

**Modified collection: `residents`**

**[Confirmed by Sathish]** No new field. `ordersDiscussion`'s discharge-date write (above) propagates
directly to the existing `residents.dischargeDate` field, rather than a separate
`anticipatedDischargeDate` field as originally proposed.

> **Residual note:** the existing `dischargeDate` field is also used by the Referral workflow and by
> the daily-summary report's confirmed-discharge query (`{status:'Discharged', dischargeDate:{$gte,
> $lte}}`). That query is gated on `status:'Discharged'`, which the IDT report never sets, so writing
> an anticipated (non-final) date shouldn't cause it to mismatch. The Referral workflow's own use of
> `dischargeDate` wasn't visible in this review — worth a quick confirmation with whoever owns that
> workflow that it tolerates an anticipated date being present well before an actual discharge is
> processed, so an IDT-set anticipated date doesn't get misread as a confirmed one elsewhere.

### 3.4a Field-level encryption for `idtreports`/`idtReportVersions` — [Confirmed by Sathish, PRD round 3 item 4, resolves §11 item 24]

**The gap.** PRD §9/§12 now require all IDT report data — both PCC-sourced auto-populated snapshots
(`patientInformationSnapshot`, `weightTrendSnapshot`, `medicationsSnapshot`) and staff-entered content
(`clinicalInformation`, `rehab`, `dischargePlanning`, `ordersDiscussion`) — to be field-level encrypted
at rest, applying to `idtreports` and `idtReportVersions` alike, "consistent with how other PHI on the
platform is handled." Nothing in this design up to this point specifies field-level encryption for
either collection; the only encryption work designed so far in this document is for the unrelated
`observations`/`residentObservationTrends` collections (§3.4's weight-trend section). **[Confirmed by
Sathish]** The recommendation below, including the carry-forward decrypt/re-encrypt mechanism it
requires, is now confirmed as the design to build.

**Recommended approach — reuse the existing platform pattern.** The same per-field AES-256-GCM scheme
already used for `observations` (each field individually encrypted as its own
`{alg, ciphertextB64, ivB64, authTagB64}`) is recommended here too, for exactly the consistency PRD §9
asks for, rather than introducing a second encryption pattern for this one collection.

**Recommended scope — encrypt content, leave scoping/control-flow fields plaintext:**

| Field group | Encrypt? | Why |
|---|---|---|
| `patientInformationSnapshot.*`, `weightTrendSnapshot[]`, `medicationsSnapshot[]` | **Yes** | PRD §9 names these explicitly as in scope — PCC-sourced clinical content. |
| `clinicalInformation.*`, `rehab.*` (excluding `rehab.markedComplete`), `dischargePlanning.*`, `ordersDiscussion.notes` | **Yes** | Staff-entered clinical content, explicitly in scope. |
| `idtReportVersions.content` (the full copy of the above at save time, §3.4) | **Yes** | Same content, same requirement — this field needs the identical per-field treatment as the live document, not a separate scheme. |
| `_id`, `facilityId`, `residentId`, `reportDate`, `createdAt`/`updatedAt`, `status`, `completedAt`/`completedBy`, `createdBy`/`updatedBy`/`savedBy` | **No** | Needed in plaintext for indexing, filtering, sorting, and permission checks (§3.6); none of these are themselves clinical content. |
| `signature.*` (state, sentTo, sentAt, signedAt, signedBy, editedAfterSignAt, submissions) | **No** | Staff identifiers and workflow timestamps, not clinical content; needed plaintext to drive the signature state machine, notification routing (§3.11), and the list's Signature Status column (§6). |
| `disciplineCompletion.*` | **No** | Derived enum values (`PENDING`/`IN_PROGRESS`/`COMPLETE`) needed plaintext for the list's discipline columns (§5.3); not clinical content in itself. |
| `ordersDiscussion.continueSkilledCare`, `ordersDiscussion.dischargeDateSet`, `ordersDiscussion.dischargeDate` | **Open — flagging, not deciding.** | These are structured decision/date fields, not free text, and `dischargeDate` is also written to `residents.dischargeDate` (plaintext, per the existing design there) — encrypting it here while it's plaintext on `residents` would be an inconsistency worth Sathish's attention rather than something this design resolves by itself. |
| `faxLog[]`, `pdfUrl` | **No** | Operational metadata (a fax number and send status, a storage URL) rather than clinical content; PRD §9's scope statement doesn't name these. |

**Design wrinkles this scope introduces, confirmed as implementation detail to build correctly:**
- **Carry-forward (§3.3 step 4) copies team-entered content from the prior report into a new one.** If
  each field's ciphertext is tied to a unique IV, carry-forward can't just copy ciphertext byte-for-byte
  onto the new document — reusing an IV across two different documents is a real cryptographic
  anti-pattern, not just a style concern. Carry-forward decrypts the prior report's content, then
  re-encrypts it fresh (new IV) onto the new report — an extra decrypt/encrypt round trip on every
  Create, not just on every Save.
- **The rehab progression trend (§3.3) reads `rehab.pt`/`rehab.ot` from the two most recent prior
  reports on every summary view** — each of those reads now needs a decrypt step, on a request path
  that's read-heavy (every time a Complete report's summary is opened, not just on save).
- **Refresh (§3.3) re-derives `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot`
  from current backend data and compares them against the currently-persisted values to detect "does
  this save follow a Refresh"** (the `reportDate`-mutation mechanism, §3.3) — that comparison needs to
  happen on decrypted values, not ciphertext, since two independently-encrypted copies of the same
  plaintext won't compare equal (different IVs). This is a correctness-affecting detail, not just a
  performance one, if missed.
- None of these are reasons not to encrypt — they're implementation details the actual build needs to
  get right.

**[Confirmed by Sathish]** — is the carry-forward decrypt/re-encrypt cycle avoidable? Sathish asked
whether a better alternative exists before accepting it as designed. Alternatives considered and
rejected:
- **Copy the prior report's ciphertext byte-for-byte instead of decrypting and re-encrypting**, since
  the content is identical at the moment of Create. This is not cryptographically broken on its own —
  reusing an IV is only catastrophic when the *same* (key, IV) pair encrypts two *different* plaintexts,
  and at the instant of copy the plaintext is identical — but it only holds until the very first edit,
  which carry-forward content is specifically meant to receive; the saving would only ever cover the
  narrow window between Create and the user's first keystroke. It would also require a special-cased
  "copy raw ciphertext" code path alongside the normal "always encrypt fresh on write" path, which is
  exactly the kind of exception that risks a future bug reusing an IV across genuinely different
  plaintext elsewhere. Rejected: the risk of a fragile special case outweighs a saving that evaporates
  on first edit anyway.
- **Deterministic encryption (e.g. AES-SIV)**, so identical plaintext always produces identical
  ciphertext and copying needs no decrypt step. Rejected: this trades away semantic security — an
  observer with ciphertext access could tell two fields hold identical content without decrypting
  either — a weaker guarantee than the platform's existing random-IV AES-256-GCM pattern (already used
  for `observations`), and introducing a second encryption scheme for one collection isn't proportionate
  to this one optimization.
- **Per-document data keys (envelope encryption)** — doesn't avoid the problem at all; carry-forward
  would still need to decrypt under the source document's key and re-encrypt under the new document's
  key. Only changes blast radius on key compromise, unrelated to this question.
- **Store a reference to the source document's encrypted value instead of copying it** — rejected
  because it would break `idtReportVersions`' already-fixed invariant that each version row captures a
  full, independent copy of report content as of that save (§3.4's `content` field notes) — reintroducing
  by reference exactly the kind of gap that fix closed.

**Conclusion: no alternative avoids the cycle without either introducing new cryptographic risk or
breaking an already-established design invariant.** Decrypt-then-re-encrypt with a fresh random IV on
every write, including carry-forward's copy step, is confirmed as the design to build — consistent with
standard practice for content that is copied and then made independently editable, and consistent with
the platform's existing `observations` pattern. This closes §11 item 24.

### 3.5 Controlled vocabularies *(Team-submitted, unified per the PRD update)*

**Standard Assist-Level Scale.** Applies to: Bed Mobility, Sup-Sit Transfers, Sit-Stand Transfers,
Gait, and DME Used (PT); Upper Body Dressing, Lower Body Dressing, and Toilet Transfers (OT).

`Mod Ind` · `SBA` · `CGA` · `Min Assist` · `Mod Assist` · `Max Assist` · `TD`

Meanings: `Mod Ind` = Modified Independent · `SBA` = Standby Assist · `CGA` = Contact Guard Assist ·
`Min Assist` = Minimal Assist · `Mod Assist` = Moderate Assist · `Max Assist` = Maximum Assist ·
`TD` = Total Dependence.

**Weight-Bearing Status Scale.** Applies to: `rehab.weightBearingStatus`.

`As Tolerated` · `Non Weight Bearing` · `Partial Weight Bearing` · `TTWB`

(`TTWB` = Touch-Down Weight Bearing.)

**Discipline Completion Status.** Applies to: Case Manager, Social Worker, Rehab Team.

`PENDING` = Pending · `IN_PROGRESS` = In Progress · `COMPLETE` = Complete

### 3.6 Indexes *(Team-submitted, with amendments)*

| Collection | Index | Purpose |
|---|---|---|
| `idtreports` | `{facilityId, status, reportDate}` | **[Confirmed by Sathish, design review — field switched from `createdAt` to `reportDate`]** List tab split (In-progress/Completed) + default newest/oldest sort — the most common query. PRD §5.1 names the visible list column "Report Date" (line 119) and defines the sort as newest/oldest against it; this index needs to match the column the grid actually displays, which is `reportDate`, not the internal `createdAt` used for carry-forward/rehab-trend/retention purposes (§3.3/§3.4/§3.8). Before `reportDate` became mutable this distinction didn't matter — the two fields held identical values — but now a report refreshed today would show today's date in the grid while potentially sorting mid-list if this index stayed on `createdAt`. |
| `idtreports` | `{facilityId, residentId}` — **unique, partial** (`partialFilterExpression: {status: 'DRAFT'}`) | **[Confirmed by PM, per PRD §6 rule 5]** Replaces the old `{facilityId, residentId, weekOf}` unique index. PRD §6 rule 5 now states this exact invariant directly: **a resident has at most one open Draft at a time**; any number of Complete reports may accumulate over time, created whenever staff need one. Because it's a *partial* index (MongoDB-supported), it only constrains documents matching `status: 'DRAFT'`; Complete reports are invisible to it and unconstrained. This was originally proposed as an SA recommendation (§11 item 14) and is now formally specified in the PRD, so the recommendation is resolved — no design change needed, the index as designed already matches; the rejected alternative of no constraint at all is in §4. |
| `idtreports` | `{facilityId, residentId, createdAt}` | **[SA-authored — new index, replacing the old unique index's secondary role; field renamed from `reportDate` to `createdAt` — SA amendment, PRD round-2 update.]** The old unique index also happened to serve every plain by-`residentId` lookup (carry-forward's "most recent report," §3.3; the rehab progression trend's "two most recent prior reports," §3.3; the Case Manager/Social Worker live-join's step (2) below) — a partial index scoped to Draft-only can't do that anymore, since it doesn't cover Complete reports. This general, non-unique, non-partial index takes over that role, sorted by `createdAt` for "most recent" queries — **not** `reportDate`, which can now move on Refresh + Save (see §3.3's dedicated block) and would otherwise corrupt "most recent" ordering. |
| `idtreports` | `{facilityId, signature.state}` | Future physician-queue view; useful now for internal "awaiting signature" lookups |
| `idtreports` | `{facilityId, "patientInformationSnapshot.attendingMD.name"}` | **[SA amendment — new index, replaces an earlier proposed `assignedAttendingMDName` field.]** Multikey index on the existing frozen snapshot array. Backs the PRD §5.1 Attending MD filter and A–Z sort directly, with no new field. |
| `idtreports` | `{facilityId, "patientInformationSnapshot.residentName"}` | **[SA-authored — new index, filling a gap.]** Backs the PRD §5.1 resident A–Z sort, which two other places in this design referred to as already accounted for — it wasn't; this closes that gap the same way the Attending MD index above does. |
| `idtReportVersions` | `{reportId, savedAt}` | Per-report history, newest first |
| `idtreports` | `{facilityId, createdAt}` | **[SA-authored — new index; field renamed from `reportDate` to `createdAt` — SA amendment, PRD round-2 update.]** Backs the nightly retention-cleanup job's per-facility age-cutoff scan (§3.8), anchored on the immutable `createdAt` rather than the now-mutable `reportDate` — see §3.8's "Anchor date" note. |
| `residents` | `{facilityId, assignedStaff}` | **[SA amendment — new index, or confirm one already exists.]** Backs the live Case Manager/Social Worker resolution (§3.4) and the Create-action resident-picker scoping (§6) — both look up residents by whether a given staff member appears in `assignedStaff`. If an equivalent index already exists elsewhere in the codebase for other assignment-based features, this doesn't need duplicating; needs confirming either way. |

List search (resident name / room) runs as a plain regex match against
`patientInformationSnapshot.residentName`/`.room` directly on `idtreports` — no join to `residents`
needed. Tradeoff: since these fields are frozen at creation, a resident name correction made later in
PCC won't retroactively update older reports' searchability by the new name.

**Case Manager / Social Worker list filter — the live-join query, and why it's not a performance
concern at this scale.** Since neither is denormalized onto `idtreports` (§3.4), filtering or
displaying by Case Manager/Social Worker involves two collections instead of one field: (1) query
`residents` with `{facilityId, assignedStaff: <staffId>}` (using the index above — a single, direct,
indexed lookup, nothing unusual) to get matching `residentId`s, cross-referencing `Staff.designation`
to distinguish Case Manager from Social Worker; (2) query `idtreports` with
`{facilityId, status, residentId: {$in: [...]}, ...}` — the general `{facilityId, residentId,
createdAt}` index above already covers `residentId` lookups across every status, so no dedicated new
index is needed on the `idtreports` side just for this join. Both steps can run as one database call
via a `$lookup` aggregation rather than two separate round-trips, if preferred.

At the PRD's stated scale (500 residents/facility), step (1) returns at most 500 IDs — worst case,
if a filter somehow matched every resident — and an `$in` with up to 500 indexed values in step (2)
is a fast, ordinary query, not a bottleneck. The only way this actually underperforms is if
`residents.assignedStaff` isn't indexed at all (hence the index above); once that's in place, this
is a standard query shape, not a risk to design around. (An earlier pass flagged this in the risk
table below as a performance concern — that was overstated; corrected there.)

### 3.7 Facility-level IDT report settings *(SA-authored)*

Two per-facility configuration values, added to the existing facility-settings surface as
`facilities.idtReportSettings` rather than new standalone collections:

- **`staleDraftCleanupDays`** (Number, default `15`, admin-editable) — PRD §9 requires clearing
  drafts untouched for a facility-configurable number of days, and PRD §12 delegates the "where the
  setting lives / hard delete vs. archive" decision to the Architect. A nightly scheduled job,
  evaluated once per facility in that facility's local time zone (PRD §4.3), finds `idtreports` where
  `status = 'DRAFT'` and `updatedAt` is older than `now - staleDraftCleanupDays`. Cleanup reuses the
  exact Delete flow in §3.3 (hard delete, purge `idtReportVersions`, write one
  `idtReportDeletionLogs` row with `deletedBy = 'system:stale-draft-cleanup'`). Recommend hard delete
  over an archive tier, for consistency with manual Delete, unless Sathish or compliance flags a
  reason a stale draft needs to be recoverable that a manually-deleted draft doesn't.
- ~~`eligibleSignerDesignations`~~ — **removed, PRD round 3 item 7.** No facility-configurable
  designation list is needed after all. Engineering confirmed the `staff` object already carries a
  `mobile_access` field, valued `'Doctor'` for every signer-eligible user (the attending MD and their
  PA/NP alike) — the signer-resolution query (§3.3) reads this field directly instead. This removes a
  facility setting, a rollout step (§9), and a risk row (§10) that this design previously carried;
  nothing here needs building.
- **`versionRetentionYears`** (Number, default `10`) — **[Confirmed by Sathish]** a facility-level
  configuration, not a fixed platform-wide constant, since medical-record retention requirements vary
  by state. Governs how long a report and its full version history are kept before being purged.
  **[Confirmed by Sathish, resolves §11 item 5]** The default applies at the application level when
  the field is absent — no rollout backfill/migration is needed to seed every facility's settings
  document with an explicit `10` before go-live (§9). See §3.8 for the anchor date, purge mechanism,
  and safety recommendations.

### 3.8 Version and record retention *(SA-authored)*

**The retention window is a facility configuration, defaulting to 10 years.** **[Confirmed by
Sathish]** — not a fixed platform-wide constant. Configured as
`facilities.idtReportSettings.versionRetentionYears` (Number, default `10`), on the same
facility-settings surface as §3.7's other two values, since medical-record retention requirements
vary by state and this needs to be overridable per facility without a deploy. The 10-year default
matches ordinary SNF medical-record retention practice and is what a facility gets out of the box
until an admin overrides it.

**Anchor date:** `idtreports.createdAt` — **[Confirmed by Sathish, PRD round-2 update; supersedes the
original `reportDate`-anchored design]** the immutable, system-set creation timestamp already on every
report (§3.4, "Fax, PDF, and record-keeping"). `reportDate` was the original anchor choice, but PRD
round-2 made `reportDate` mutable (it now moves to the save date when a Save follows a Refresh, §3.3) —
an anchor that can move forward is unsafe for a retention cutoff, since a report could reset its own
retention clock indefinitely just by being refreshed and saved periodically. `createdAt` is the one
date field on the document that's always present and never edited, which keeps the retention cutoff a
simple, unambiguous calculation rather than one that drifts every time a long-lived report is re-saved
(`updatedAt` would move forward on every edit, and — as of this round — so can `reportDate`, whose job
is now confirmed as capturing the report's revised/filing date, not anchoring retention).

**Scope — applies to the whole report, not just its version history.** *(This extends beyond the
original question — worth a quick confirm.)* A report's audit trail only makes sense as a unit:
retaining `idtReportVersions` for the full window while purging the current-state `idtreports`
document earlier (or the reverse) would leave an inconsistent, harder-to-reconstruct record.
Recommendation: once a report's `createdAt` passes the facility's `versionRetentionYears` cutoff,
purge the `idtreports` document itself and every one of its `idtReportVersions` rows together, in one
operation — the same bundle already deleted together in the Draft-delete flow (§3.3), just gated on
age instead of status.

**Mechanism — a second scheduled job, parallel to §3.7's stale-draft cleanup:**
1. Nightly, per facility, evaluated in that facility's local time zone (same pattern as §3.7).
2. Find `idtreports` documents where `createdAt < now - facility.idtReportSettings.versionRetentionYears`
   (regardless of `status` — a Draft this old would already have been cleared by stale-draft cleanup
   long before reaching 10 years, so in practice this job only ever touches Complete/signed reports).
3. Delete all `idtReportVersions` rows for each matching report, then the `idtreports` document itself.
4. Write one row to `idtReportDeletionLogs` per purged report, with
   `deletedBy = 'system:retention-cleanup'` — a second distinguishable system-actor string alongside
   `'system:stale-draft-cleanup'` (§3.7); no schema change needed, since `deletedBy` is already a
   free-form String.

**Safety recommendation, given the compliance stakes of getting this wrong:** run the job in a
report-only mode ("would delete N reports for facility X") for at least one full cycle before
enabling actual deletion, and keep a short-term backup/snapshot of anything the job deletes for a
grace period after each run. Purging a record that turns out to still be inside the required
retention window is a compliance problem; purging it a few weeks later than the exact cutoff is not.

**Deliberately out of scope for this pass — storage tiering within the 10-year window.** Version
history is essentially never read once a report is old and settled, which makes it a reasonable
candidate for moving to cheaper storage (a cold collection, or object storage) well before the
10-year mark, purely as a cost optimization, not a retention change. **[Confirmed by Sathish, resolves
§11 item 6]** Tracked as a deferred technical item, not an open design question — see §12.

### 3.9 Bulk report creation (multi-resident Create) *(SA-authored)*

**The shape of the change.** **[Confirmed by Sathish]** The Create action's resident picker moves
from single-select to multi-select: a user selects one or more residents and the system generates a
Draft report for each, in one action — "one shot," not one Create click per resident.

**Picker data source and scoping — extends the existing design (§6), doesn't replace it.** The
eligible-residents-for-Create query keeps its existing scoping rule (residents in the requester's own
`assignedStaff` for Clinical Staff; the full facility roster for a Director — §6), and adds one new
filter: it excludes any resident who already has an open Draft `idtreports` document. **[SA amendment
— per PRD update]** This was originally a "current `weekOf`" exclusion; since PRD §1/A1 confirms
there's no fixed cadence, that no longer means anything, and this design replaces it with the
open-Draft exclusion instead (matching the new partial unique index, §3.6). The reasoning for having
some exclusion at all is unchanged: offering a resident who can't be created for right now (the
partial unique index, §3.6) just produces an avoidable failure once bulk-create is in play, and this
way most of those get filtered out before they're ever offered. A resident with only Complete reports,
however recent, is never excluded — any number of Complete reports may accumulate over time.

**Ordering.** **[Confirmed by Sathish]** The picker list is sorted alphabetically by resident name —
the same name representation the main list's existing "resident A–Z" sort already uses (PRD §5.1),
for visual consistency between the two A–Z sorts a user will see in this feature.

**Display-name field — [Confirmed by Sathish, resolves former §11 item 10]** The Create picker queries
the **`residents`** collection directly (it must list residents who may have *no* `idtreports` document
yet — a brand-new admission, for instance), so it cannot literally read
`idtreports.patientInformationSnapshot.residentName` the way the main list does. Now that the actual
`residents` schema has been reviewed, the field is **`residents.name`** — a pre-composed, already-flat
display name (e.g. `"Shariin Abreuu"`), distinct from the separate `firstName`/`lastName` fields on the
same document. This should be the single source for the picker's name, its sort/search match, and
`patientInformationSnapshot.residentName` at Create time (§3.3 step 3), so all three read from one
consistent value rather than each deriving its own representation. This closes the item that was
previously left open pending a look at the real schema.

**Search — client-side filtering of an already-fetched list, not a query per keystroke.**
**[Confirmed by Sathish]** As the user types, the list filters to matching residents. The picker
fetches its scoped, sorted candidate list once when it opens, and the search box filters that
in-memory list reactively on every keystroke, rather than issuing a fresh request each time. Even in
the largest realistic case — a Director's picker, scoped to the full ~500-resident roster — filtering
that many already-fetched items client-side is effectively instant; a server round-trip per keystroke
would only add latency here for no real benefit.

**Bulk-create endpoint.** A new endpoint accepts a list of `residentId`s in place of the existing
single-resident Create call. For each `residentId` in the request, the server independently runs the
existing single-resident Create logic unchanged — carry-forward, auto-population and snapshotting
(`patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot`), and creation of a fresh
`DRAFT` `idtreports` document (§3.3, §3.4) — no `weekOf` or other period value to set, since none
exists anymore. Nothing about how one report is built changes; only the entry point now accepts many
resident IDs instead of one.

**Failure handling is per-resident, not all-or-nothing.** If creating a report for one resident in
the batch fails — most likely a race where a Draft for that resident was created by someone else
between the picker loading and the bulk-create call landing, tripping the new partial unique index
(§3.6) — that one resident is skipped and every other resident in the batch is still created. This
matches the platform's existing "last write wins, no cross-document transaction" concurrency posture
(PRD §8); a bulk-create endpoint that rolled every resident back because one collided would be a
worse experience than creating the ones that succeeded. The response reports a per-resident outcome
(created vs. skipped, with a reason) so the UI can tell the user plainly.

**Partial-failure UX — [Confirmed by Sathish, resolves §11 items 8 and 9]: a toast/alert naming the
skipped residents, with an individual-retry path.** After a bulk-create call returns, the UI shows a
toast/alert (not a blocking modal — the successful creates already happened) listing exactly which
residents were skipped and why, so the user can retry them one at a time from the list's existing
single-resident Create action if they choose to. Suggested copy, following the same shape already
proposed for the "N of M" outcome:

> "**{N} of {M} reports created.** Skipped {K}: {Resident A}, {Resident B} — an open Draft already
> exists for each. You can create a report for a skipped resident individually once their existing
> draft is completed."

For a longer skip list, name the first 3 and summarize the rest, so the toast doesn't grow unbounded:
`"{Resident A}, {Resident B}, {Resident C} +2 more"`. Since bulk-create's only realistic skip reason
is the partial-unique-Draft collision (§3.6), one shared reason clause is enough — there's no need to
vary the reason text per resident.

**Note for the frontend/engineering team — [Confirmed by Sathish, design review]:** the toast's "+N
more" truncation means a genuinely large failed batch has no way to see the full list of skipped
residents and reasons beyond the transient toast itself. Recommend a "view details" affordance (e.g. a
link in the toast opening a modal or expandable panel) listing every skipped item and its reason in
full. No new API is needed for this — the bulk-create endpoint already returns the complete per-item
failure data behind the toast copy above; this is purely a more durable UI presentation of data the
frontend already has.

**No cap on bulk-create selection. [Confirmed by Sathish, resolves §11 item 9]** The picker's existing
exclusion filter (above) already keeps a resident with an open Draft from being offered in the first
place, so the vast majority of would-be collisions never reach the failure path at all; the residual
race is rare enough, and self-correcting enough (the toast above tells the user exactly what to do
about it), that a selection cap adds friction without addressing a real problem. This is unchanged from
the design's original position — no cap — now confirmed rather than merely proposed.

**Landing behavior after bulk-create. [Confirmed by Sathish, resolves §11 item 7]** Return to the list
(In-progress tab). The list must refresh (re-fetch or re-run its current query) rather than rely on
stale client-side state, so the newly created Drafts — and any partial-failure toast above — are both
visible immediately without a manual reload.

### 3.10 Bulk report deletion (multi-select Delete) *(SA-authored, per PRD update)*

**Scope, per the PRD update.** **[Confirmed by Sathish]** Bulk delete of Draft reports is now in
scope (PRD §2.1); bulk submit and bulk send-for-signature remain explicitly out of scope and stay
single-report actions, since a bulk resident selection can span more than one attending physician
(PRD §2.2) — nothing about Submit or Send changes as a result of this update.

**List UI, per PRD §5.1.** Each Draft row gets a select checkbox (Complete rows don't — they're not
deletable, so they take no part in bulk selection, consistent with how bulk-create's picker already
excludes ineligible residents rather than offering and then rejecting them, §3.9). Selecting one or
more rows surfaces a bulk action bar with the selected count and a single **Delete selected** action.
Confirming opens a modal naming the count (not each resident/date individually, unlike the existing
single-delete modal) and hard-deletes every selected report on confirm.

**No new query or index needed.** Unlike bulk-create's picker (§3.9), which has to *discover* an
eligible set of residents, bulk delete acts on report `_id`s the user already has in hand from the
already-rendered, already-filtered list — there's nothing to look up. The bulk-delete endpoint simply
accepts a list of `idtreports._id`s and processes each by primary key.

**Bulk-delete endpoint — per-report independent processing, same posture as bulk-create (§3.9).** For
each `_id` in the request: re-verify `status === 'DRAFT'` **at delete time**, not trusting that it was
still a Draft when the list was rendered or when the checkbox was ticked — the existing single-delete
guard (§3.3, step 1), just re-run once per report. A report that fails this check (most plausibly: it
was Submitted to Complete by someone else in the moments between page load and confirm) is skipped
rather than aborting the whole batch, matching the platform's existing no-cross-document-transaction
concurrency posture (PRD §8) and the same reasoning already applied to bulk-create's partial failures
(§3.9). Every report that does pass the check is deleted via the exact single-delete flow (§3.3):
`idtreports` document removed, its `idtReportVersions` rows purged, one `idtReportDeletionLogs` row
written — per PRD §7's "a bulk delete writes one audit entry per report deleted, identical to a
single delete," which requires no schema change, since `idtReportDeletionLogs` was already designed
as one row per deleted report regardless of what triggered the deletion.

**Partial-failure UX — [Confirmed by Sathish, resolves §11 item 8] — same toast/alert pattern as
bulk-create (§3.9), for consistency between the two bulk actions.** PRD §5.1 describes the
confirmation and success toast ("shows a toast with the count") but not the partial-failure case. This
design resolves it the same way as bulk-create: a toast/alert naming the reports/residents that
couldn't be deleted, so the user can act on them individually if needed (in this case, by opening the
report to see its current — now Complete — state, since a report that failed this race is no longer a
Draft at all). Suggested copy, mirroring §3.9's format:

> "**{N} of {M} reports deleted.** Skipped {K}: {Resident A}, {Resident B} — already marked Complete by
> someone else. Completed reports can't be deleted."

As with bulk-create, name the first 3 skipped and summarize the rest for a longer list. Bulk-delete's
skip reason is always the same (the report was Submitted to Complete by someone else between page
load and confirm), so one shared reason clause is sufficient here too.

**Note for the frontend/engineering team — [Confirmed by Sathish, design review]:** same recommendation
as §3.9 — add a "view details" affordance so the full list of skipped reports and reasons is visible
beyond the toast's "+N more" truncation, using the per-item failure data the bulk-delete endpoint
already returns.

**Permissions — no change.** Bulk delete is gated by the exact same role check as single delete (PRD
§8: Clinical Staff and Director, not Physician), applied per report inside the loop above — there's
no separate "bulk delete" permission to define.

### 3.11 Staff mobile app notifications (signature workflow) *(SA-authored, per PRD update)*

**Scope, per the PRD update.** **[Confirmed by Sathish]** Two push notifications join the existing
staff mobile app signature-notification workflow, which already serves two other document types
(the Health Referral Order Summary and the Medication List) — the IDT report becomes a third. This
is an existing shared service, not a new one: delivery mechanism, tap-to-open behavior, and
notification-center listing all follow that workflow's existing pattern (PRD §5.4). This design
covers only the two new trigger points, their recipients, and the document-identifying content —
exactly the scope the PRD itself draws the line at.

**Triggers and recipients:**
1. **On Send for signature** (§3.3, new step 4): one push notification to `signature.sentTo` — the
   single recipient just chosen from the eligible-signer pool. No new resolution logic; the recipient
   is already known at this point.
2. **On Sign** (§3.3, new step 5): one push notification to *every* staff member currently in this
   report's `residents.assignedStaff` — not just whoever sent it or signed it, per PRD §5.4 ("all
   staff assigned to the resident"). Resolved live at signing time, the same pattern already
   established for the signer pool (§3.3) and the Case Manager/Social Worker list (§3.4/§3.6) — no
   snapshot, no new field, just a read of the existing array at the moment of dispatch.

**No new schema needed.** The underlying facts these notifications announce are already captured by
existing fields: `signature.submissions` already records the send-for-signature event's date and
recipient, and `signature.signedAt`/`signature.signedBy` already record the signing event. The
notification dispatch is a side effect of an already-audited write, not a new fact requiring its own
storage — consistent with this design's recurring principle of not inventing a field where the data
already exists elsewhere (§3.4, §3.9). PRD §9's audit requirement ("every write in Section 7 is
logged") is satisfied by the existing fields; no `notificationLog` array is needed alongside `faxLog`.

**Content.** The notification must identify the document type as "IDT report," reading distinctly
from a Health Referral Order Summary or Medication List notification (PRD §5.4). **[Open — needs
confirming with whoever owns the shared notification workflow, see §11.]** The exact registration
mechanism for a new document type in that existing service — a config entry, a template ID, an
enum value in that service's own code — wasn't visible in this review, since it lives outside the
IDT reports codebase.

**Failure handling — best-effort, non-blocking.** A failed push dispatch (the mobile push service is
down, the recipient has no registered device, etc.) must not block or roll back the underlying
Send-for-signature or Sign write itself — those are the clinically meaningful state transitions;
notification delivery is secondary to them. This matches the platform's existing non-transactional
posture elsewhere in this design (§3.9, §3.10). Recommend logging a dispatch failure for
observability, separately from the report's own audit trail, so it doesn't silently disappear.

**Explicitly out of scope for this design, per the PRD's own framing:** delivery mechanism, retry
behavior, tap-to-open deep-linking, and notification-center listing — all inherited from the existing
shared workflow and already built for the other two document types.

## 4. Alternatives considered *(SA-authored)*

| Alternative | Why rejected |
|---|---|
| Embed version history as a growing array inside the `idtreports` document itself, instead of a separate `idtReportVersions` collection. | `idtreports` is read on every list load and every report open — embedding an unbounded, ever-growing history array would slow down that hot path for history that's rarely read. |
| Migrate the existing `idtreports` collection's data into the new schema, instead of dropping it. | PRD §6/A7 confirm the existing collection has never been used in production and holds only stale test records with no clinical value. Migrating throwaway test data would add real complexity for zero benefit — a clean drop before rollout (§9) is simpler and lower-risk. |
| Add a new `residents.anticipatedDischargeDate` field, kept separate from the existing `dischargeDate`. | This was the submission's original proposal. Sathish decided to reuse the existing `dischargeDate` field directly instead — simpler, avoids introducing a second discharge-date field for other modules to reconcile, at the cost of the residual interaction noted in §3.4 that's worth a quick check with the Referral workflow's owner. |
| Hard-code the eligible-signer designation values (e.g. `['Physician', 'PA', 'NP']`) directly in the query. | Rejected once Sathish confirmed the PA/NP designation label varies by facility — a hard-coded list would work for some facilities and silently exclude valid signers at others. A facility-level setting (§3.7) is the mechanism that survives that variation, even though the exact values still need one round of discussion with engineering. |
| Denormalize `assignedCaseManagerCNames`/`assignedSocialWorkerCNames`/`assignedAttendingMDName` onto `idtreports`, resolved and stored at save time (an earlier draft of this design). | Rejected per Sathish: `residents.assignedStaff` is already the source of truth for staff assignment, and duplicating it onto every report adds resolution logic and drift risk for no real benefit at this scale. Resolving live at query/read time (§3.4, §3.6) keeps the report schema focused on report content, consistent with how the signer pool is already handled. |
| Snapshot the eligible-signer pool onto the report at creation time (matching how `patientInformationSnapshot` is frozen), instead of resolving it live at send time. | A report can sit `AWAITING` for a long time, and a snapshotted pool risks offering a signer who's no longer assigned to the resident. |
| Store `weightTrendSnapshot` inside `clinicalInformation` (matching how the PRD's UI visually groups weight with Change in Condition and Skin & Wound). | Weight is auto-populated/frozen while the rest of `clinicalInformation` is team-entered and versioned every save — mixing them would force redundant re-copying of frozen data into every version row. This is a presentation-layer concern the UI/API can reassemble, not a storage one. |
| No retention policy — let `idtreports` and `idtReportVersions` grow indefinitely (the original design, before this pass). | Rejected once raised: unbounded growth with no compliance basis for infinite retention is a storage-cost and cleanup problem waiting to happen. A 10-year, facility-configurable window (§3.8) matches ordinary SNF medical-record retention practice instead. |
| Delete `idtReportVersions` rows immediately once superseded by a later save, keeping only the current and as-signed states. | Rejected — the full version history is itself the audit trail required by PRD §9's version-integrity requirement (§7); collapsing it early would defeat the purpose of versioning every save, not just optimize storage. |
| Repeat the existing single-resident Create flow N times from the client (one request per resident) instead of a dedicated bulk-create endpoint (§3.9). | Rejected — defeats the point of "one shot": N round-trips instead of one, and no single place to assemble a clean per-resident success/failure summary for the user. |
| Re-query the eligible-residents list from the server on every keystroke in the picker's search box (§3.9), instead of filtering client-side. | Rejected — the scoped candidate list is already small (bounded by a staff member's caseload, or at most the ~500-resident roster for a Director) and fetched once when the picker opens; filtering it client-side is effectively instant and a request-per-keystroke would add latency for no real benefit. |
| Make bulk delete (§3.10) a single atomic all-or-nothing operation, instead of per-report independent processing. | Rejected — a legitimate race (another user Submits one of the selected drafts moments before the bulk delete confirms) shouldn't block deleting the rest of the batch; matches the same per-item independent-failure posture already established for bulk-create (§3.9) and the platform's existing no-cross-document-transaction concurrency model (PRD §8). |
| Add a dedicated `notificationLog` field on `idtreports` (mirroring `faxLog`) to record each push dispatch (§3.11). | Rejected — the underlying facts are already captured by `signature.submissions` and `signature.signedAt`/`signedBy`; a separate log would just duplicate data already satisfying PRD §9's audit requirement, for no benefit. |
| Block Send-for-signature or Sign on push-notification delivery succeeding, instead of best-effort dispatch (§3.11). | Rejected — notification delivery is secondary to the underlying state transition; failing a clinically meaningful write because a push service is briefly down would be a worse outcome than the notification simply not arriving. |
| Drop the duplicate-report constraint entirely (no replacement), instead of the new partial-unique-Draft index (§3.6), once the old weekly constraint was removed. | Rejected. This was the SA's recommendation pending confirmation as of the last pass; **[Confirmed by PM, per PRD §6 rule 5]** the PRD now states this invariant explicitly — a resident has at most one open Draft at a time — settling the narrower question the original cadence update hadn't addressed. |
| Keep some form of calendar-based grouping (e.g., "one report per resident per day," or per some other fixed window) rather than a status-based constraint. | Rejected — any calendar-based rule reintroduces exactly the fixed-cadence assumption PRD §1/A1 just removed, just at a different granularity. A status-based constraint (at most one open Draft) doesn't assume anything about timing at all. |
| **[SA amendment, PRD round-2 update]** Leave `reportDate` as the sole ordering/anchor key for carry-forward, the rehab trend, the general lookup index, and retention (§3.3/§3.4/§3.6/§3.8), now that PRD round-2 makes `reportDate` mutable on Refresh + Save. | Rejected — accepting this would let a report's position in "most recent"-style queries change after the fact, and would let a report reset its own retention clock indefinitely just by being refreshed and re-saved periodically. Switching those roles to the already-existing, genuinely-immutable `createdAt` field avoids both failure modes at no cost, since `createdAt` is already captured on every report. **[Confirmed by Sathish]** Recorded here as the rejected alternative per §3.3's dedicated block. |

See §3.4–§3.6 above for the complete schema, controlled vocabularies, and indexes; §3.7 for the
facility-level settings; §3.8 for the version/record retention policy; §3.9 for the multi-resident
Create/bulk-create design; §3.10 for bulk delete; and §3.11 for staff mobile app notifications. No
migration is required (§4, §9) — this is a schema rewrite of a collection holding only pre-production
test data. No new field is added to `residents`, and no schema change is needed anywhere for bulk
delete or for notifications — the latter's underlying facts are already captured by existing
signature fields (§3.11).

## 6. API / interface changes *(SA-authored)*

- **IDT report CRUD endpoints** (list, create, get, save, submit, delete) — internal handlers need
  rewriting to read/write the new schema shape and status model. No externally-visible contract
  change is implied by the PRD, but this needs confirming against the actual current route/controller
  layer, which wasn't in scope of this review.
- **Report Date visible on the Edit and Summary pages.** **[Confirmed by Sathish, design review]**
  Today the PRD only shows Report Date in the list row (§5.1); the form and summary headers don't
  display it at all, even though it's the date that tells staff which day the auto-populated data
  pertains to — more important now that Refresh + Save can move it (§3.3). No schema or API change is
  needed — `reportDate` is already a core field on every report-get response — this is purely a UI
  addition to the form header/sub-title and the summary header. Flagged as a suggested PRD content
  addition; see `feature-idt-reports-v2_SA-comments.md` for the suggested wording.
- **Unsaved-changes confirmation on navigating away.** **[Confirmed by Sathish, design review]** Already
  committed in PRD §10 ("this release ships manual Save with an unsaved-changes guard"), but this
  design never captured it. Frontend-only behavior, no schema impact: navigating away from an open
  report (Draft or Complete) with unsaved edits should prompt a confirmation dialog before discarding
  them. Add an explicit test case in §8.
- **Rehab discipline clear-on-deactivate.** **[Confirmed by Sathish, PRD round 3 item 2 — fully
  resolved]** Frontend confirmation dialog plus a clear operation on **all** of that discipline's
  fields, including the Goal field, when a PT/OT toggle is turned off after ratings exist — no new
  endpoint, rides the ordinary Save flow. Re-enabling the toggle afterward starts fresh, not restoring
  the cleared values — closes former §11 item 23. Add an explicit test case in §8.
- **Rehab mark-complete checkbox disabled once Complete.** **[New, PRD round 3 item 3]** Frontend-only
  state change — no schema or API impact. Full detail in §3.4's `rehab.markedComplete` field notes.
- **Refresh auto-populated data endpoint.** **[Confirmed by Sathish, resolves §11 item 3]** New,
  lightweight, read-only endpoint that re-runs the existing Create-time resolution logic (§3.3 step 3)
  for `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` and returns the current
  values — it does not write to `idtreports`. Backs a new Refresh action in the open report's
  auto-populated sections, addressing staleness during a long-lived Draft or Awaiting-signature period
  without a passive staleness indicator; the refreshed values are persisted only via the user's next
  ordinary Save (§3.3).
- **Resident picker for the Create action.** **[Confirmed by Sathish]** The list of residents offered
  when creating a new report must be scoped to residents the requesting staff member is assigned to
  (`residents.assignedStaff` contains the requester's identifier, any designation) rather than the
  full facility roster. Confirmed against the real `residents` schema: `assignedStaff` is a flat array
  of staff identifiers with no per-designation structure on the resident document itself, so
  distinguishing "is this staff member the Case Manager, the Social Worker, or Rehab for this resident"
  still requires cross-referencing each ID's `Staff.designation`, exactly as this design already
  assumed for the CM/SW live-join (§3.6). A Director, who has facility-wide oversight rather than a
  specific resident assignment, sees the full roster instead of a scoped one, consistent with their
  existing "full oversight plus delete" role. **[SA amendment, design review — reverses an earlier
  confirmed decision]** This is now a genuine **backend authorization rule, not just a UX default**: a
  Case Manager or Social Worker may only Create, Edit, or Complete a report for a resident whose
  `assignedStaff` includes them — the earlier position ("any Case Manager or Social Worker can act on
  any report they can reach, regardless of assignment") has been reversed. Implement this as a
  server-side check on Create, Save, and Submit, not merely as a picker filter: the picker's scoping
  and the backend enforcement must now agree, where before they were deliberately allowed to differ.
  **[Confirmed, PRD round 3 §8]** The first of two open points from the prior round is now settled:
  the same resident-assignment requirement **does** extend to the Rehab Team, the same way it applies
  to Case Manager/Social Worker. **The second is not settled, and is now a formally tracked PRD open
  question (PRD §11 item 2, medium priority) rather than an assumption this design can carry
  forward as confirmed:** whether the Director's facility-wide access remains exempt from the
  assignment requirement, or whether Directors should also be scoped to assigned residents. This
  design does **not** assume Directors keep their bypass — treat this as genuinely open until Sathish
  or the PM settles it; see §11's new item on this.

  **Separately, and more narrowly: *how* the system determines which staff count as "Rehab Team" for
  this authorization check is explicitly not a design decision this document — or the PM, or the SA
  agent — gets to make (PRD §8, §11 item 3).** Unlike the signer-resolution question (§3.3, resolved
  this round via `staff.mobile_access`), the PM was explicit that the analogous Rehab Team question
  should be answered by Engineering on its own; "a similar approach may apply" is only a possibility,
  not a design instruction. **This design deliberately does not propose a field, a hard-coded
  designation list, or a facility-configurable setting for Rehab Team membership** — building against
  any assumed mechanism here would need to be redone once Engineering answers. The Permissions epic
  should not be treated as ready for story-writing until that answer comes back; see §11.
  The picker is now multi-select, alphabetically sorted by resident name, and filters live as the user
  types (client-side, against the already-fetched scoped list) — see §3.9 for the full design,
  including the new exclusion of residents who already have an open Draft (replacing what used to be
  a current-week exclusion, now that there's no fixed cadence per PRD §1/A1).
- **Bulk-create endpoint.** New — accepts a list of `residentId`s and creates one Draft report per
  resident in a single call, each built via the existing unchanged single-resident Create logic
  (§3.9). Per-resident failures (most likely a race against the new partial-unique-Draft index, §3.6)
  are reported back individually rather than failing the whole batch.
- **Bulk-delete endpoint.** New, per PRD §2.1/§5.1 — accepts a list of `idtreports._id`s (drawn
  directly from the list's multi-select, no lookup needed) and hard-deletes each via the existing
  unchanged single-delete flow (§3.3/§3.10), re-verifying `status === 'DRAFT'` per report at delete
  time rather than trusting the client. Writes one `idtReportDeletionLogs` row per deleted report,
  per PRD §7 — same shape as a single delete, no schema change. Bulk submit and bulk
  send-for-signature are explicitly not introduced by this endpoint or any other — both remain
  single-report actions per PRD §2.2's differing-attending-physician rationale.
- **Case Manager / Social Worker list filter and column** — resolved live via the two-step join
  described in §3.6, not read from a stored field. The PRD §5.1 list currently describes these as
  single-value columns/filters; since a resident can have several of each (confirmed by Sathish, by
  design), the column needs to render multiple names (e.g. comma-separated, or "+N more") and the
  filter needs an array-contains match. This is a UI/PRD-side follow-up worth flagging when
  Epics/Stories are drafted, not a schema gap.
- **Send-for-signature endpoint** — calls the signer-resolution query in §3.3, now reading
  `staff.mobile_access == 'Doctor'` (PRD round 3 item 7, resolved — no facility setting), appends to
  `signature.submissions`, and sets `AWAITING`.
- **Sign endpoint** — enforces caller identity equals `signature.sentTo`; sets `SIGNED`.
- **Staff mobile app notification integration** — new calls into the existing shared
  signature-notification workflow at Send-for-signature and Sign (§3.11), not a new notification
  service. The exact mechanism for registering "IDT report" as a new document type in that existing
  service (config, template, enum) wasn't visible in this review and needs confirming with whoever
  owns it.
- **PDF endpoints** — PRD §5.6 requires both existing endpoints (direct-download render, and the
  generate-and-store variant exposing `pdfUrl`) to carry forward unchanged at the contract level.
  The underlying render service (`buildIdtReportHtml`, `idtReport.pdf.service.ts`) reads from the
  *old* schema shape today; its data access needs updating to the new field names even though its
  external contract doesn't change.
- **Fax send endpoint** — confirmed in scope for this release (§3.4). Writes to `faxLog` per send,
  now including a `status` field (§3.4). **[SA amendment, PRD round-2 update, resolves handoff round 2
  item 5]** Accepts a single free-text fax number only — no saved-contacts lookup — validated to US
  phone format (10 digits) both client- and server-side; reject malformed numbers before attempting to
  transmit. The exact integration with whatever underlying mechanism actually transmits the fax (an
  existing internal service vs. a third-party gateway), and the resulting `status` enum's exact values,
  wasn't visible in this review and should be confirmed with engineering — already flagged as the
  one-time fax-vendor discussion in the Spike (SA-comments), but the scope question itself is settled.
- **Fax History** — **[SA amendment, PRD round-2 update, resolves handoff round 2 item 6]** No new
  endpoint needed. The new PRD §5.4 Fax History section renders directly from `faxLog`, which is
  already part of the standard report-get response — the frontend just needs to build the
  collapsed-by-default UI over data it already receives.
- **List endpoint — Signature Status column.** **[SA amendment, PRD round-2 update, resolves handoff
  round 2 item 7]** PRD §5.1 replaces the per-row Status chip (Draft/Complete — redundant with the
  tab split) with a Signature Status column on the Completed tab only (Not sent / Awaiting signature /
  Signed). `signature.state` is already a field on the report document (§3.4); it just wasn't
  previously surfaced in the list response. Recommendation: always include `signature.state` in list
  rows regardless of tab, and let the frontend decide whether to render the column — simpler than a
  tab-conditional API response, and no new index is needed since `{facilityId, signature.state}`
  (§3.6) already exists. If the list query previously accepted a separate document-status filter
  parameter distinct from the tab split, it can be dropped or deprecated (PRD §5.1) — this design never
  modeled one, since the In-progress/Completed tabs already partition by `status` server-side.
- **`residents.dischargeDate` write** — a cross-feature interface, since the Referral workflow and
  the daily-summary report already read/write this field (§3.4's residual note). No new field is
  introduced; the write just needs to land on the existing one.
- **PCC-sourced fields (Diagnosis, Code Status).** **[PRD round 3 item 6]** Diagnosis (renamed from
  Chief Complaint) is now resolved — the PM confirms it's available from PCC, no new integration work
  implied. **Code Status remains open** (§11 item 2): it's still unconfirmed whether it currently syncs
  from PCC into the backend at all. If it doesn't sync today, this implies new PCC integration work not
  scoped anywhere in this document.
- **Date rename: Diagnosis field.** **[PRD round 3 item 6]** Rename `patientInformationSnapshot.chiefComplaint`
  to `patientInformationSnapshot.diagnosis` (§3.4) — display-layer rename only, same source and
  meaning, no migration needed since no production data exists yet (§9).
- **Date display format.** **[SA amendment, PRD round-2 update, resolves handoff round 2 item 3]**
  PRD §4.3 standardizes every rendered date (report, list, summary, PDF) to a single `mm/dd/yy` format,
  replacing the prior split (`MMM D, YYYY` on the report, `MMM D` in list columns). Display-layer only
  — no field, no schema, no API contract change. Any shared date-formatting utility/component used
  across report/list/summary/PDF rendering should be updated to this one format; no backend action
  needed beyond confirming no endpoint currently formats dates server-side into the old split styles.
  **[Flagged, PRD round 3 item 9, §11 item 22]** PRD §4.3 now describes `mm/dd/yy` as a *default*, with
  whether it should instead become a facility-configurable setting raised for consideration but not
  decided (low priority, non-blocking). Build this round's fix as above for now — don't build the
  hard-coded version as the final answer if a facility setting is later confirmed, since that would
  mean adding one more field to `facilities.idtReportSettings` (§3.7) plus a lookup in the shared
  formatting utility, not a redesign.

## 7. Non-functional considerations *(SA-authored)*

- **Performance.** Sized to 500 residents/facility and PRD §9's 52 reports/resident/year (~26,000/year)
  — **[Confirmed by PM, per PRD update]** PRD §9 now states explicitly that this figure is a
  planning-ceiling placeholder, not derived from cadence, retained as a round conservative number
  pending real deployment data (§11, resolved). The list's tab split, default sort, and Attending MD
  filter are backed by
  direct indexes (§3.6). Case Manager/Social Worker filtering is a live join against `residents`
  (§3.6) rather than a direct index on `idtreports` — at this scale (at most 500 residents matched,
  then an indexed `$in` lookup) this is an ordinary query, not a performance concern, as long as
  `residents.assignedStaff` is indexed (§3.6). No denormalized field is needed, per Sathish's
  direction; there's nothing here to revisit unless a load test surfaces a genuine surprise. Removing
  the old weekly duplicate-report constraint in favor of the new partial-unique-Draft index (§3.6)
  doesn't add a performance concern of its own — a partial index is typically *smaller* than an
  equivalent full index, since it only covers Draft documents.
- **Security / Authorization.** PRD §8 requires every action enforced server-side regardless of what
  the client sends, and now names the roles explicitly: **Clinical Staff** = Case Manager, Social
  Worker, and Rehab Team; **[PRD round 3 item 1]** Rehab Team is no longer scoped to a single
  "Director of Rehab" designation — a facility's rehab team may include several designations (e.g.
  PT, OT, SLP, and/or Director of Rehab), and **which staff count as Rehab Team for this purpose is an
  open question reserved for Engineering (PRD §11 item 3) — this design does not assume a mechanism**,
  see §6's Resident-picker bullet. **Physician** (the attending MD or their PA/NP, resolved via
  `staff.mobile_access == 'Doctor'` — §3.3, §3.7, PRD round 3 item 7) is separate and sign-only;
  **Director** has full oversight plus delete. **[SA amendment, design review — reverses an earlier
  confirmed decision]** This is now role-**and**-assignment-based, not role-only: a Case Manager,
  Social Worker, or Rehab Team member may Create, Edit, or Complete a report only for a resident whose
  `assignedStaff` includes them — the earlier position that "any Case Manager or Social Worker can act
  on any report they can reach" has been reversed, and **[PRD round 3 §8, confirmed]** the same
  requirement now explicitly extends to the Rehab Team too. Implement this as a role-**and**-assignment
  check in the existing controller/middleware layer, keyed off the existing staff/role directory
  (`residents.assignedStaff`, confirmed as a flat array of staff identifiers, cross-referenced with
  `Staff.designation` for Case Manager/Social Worker — Rehab Team's own cross-reference mechanism is
  pending Engineering's answer above) rather than any new field. **Whether Directors keep their
  facility-wide bypass from this assignment requirement is explicitly not settled** — PRD §11 item 2,
  medium priority, raised in design review and still open. Do not build the Director bypass as
  confirmed; the existing controller/middleware layer should make this an easy toggle once the
  question is answered rather than something requiring rework either way.
  **[SA amendment, PRD round-2 update]** PRD §3's persona list narrows "Case Manager / Nurse" to
  "Case Manager" only — this design's Clinical Staff matrix above never included a "Nurse" designation
  to begin with, so no change is needed here; §11 item 17 tracks a one-line confirmation with the
  permissions-model owner that no code path grants creation rights to "Nurse" specifically.
- **Accessibility.** Standard platform conformance applies (PRD §9); no schema-level impact.
- **HIPAA / Compliance.** Standard platform requirements apply; this feature introduces no new
  data-handling category (PRD §9). The version/deletion-log model satisfies the audit requirement in
  PRD §7/§9 by construction. A facility-configurable retention window (§3.7/§3.8, default 10 years)
  now governs how long a report and its full version history are kept before being purged — matching
  ordinary SNF medical-record retention practice out of the box, while letting a facility subject to
  a stricter state requirement override the default without a deploy. **[Confirmed by Sathish, PRD
  round 3 item 4, resolves §11 item 24]** PRD §9 now also requires field-level encryption of all IDT
  report data at rest, applying to `idtreports` and `idtReportVersions` alike — a gap this design
  didn't previously specify. The scope and pattern in §3.4a, including its carry-forward decrypt/
  re-encrypt mechanism, are now confirmed as the design to build.
- **Notifications.** Two push notifications (§3.11) join the existing staff mobile app
  signature-notification workflow. Dispatch is best-effort and non-blocking — a failed push must not
  block or roll back the underlying Send-for-signature or Sign write. PRD §9's audit requirement is
  satisfied by the existing `signature.submissions`/`signature.signedAt` fields; no new stored log is
  needed for this.
- **Version integrity.** Well addressed by `idtReportVersions`/`isAsSignedVersion` (§3.4) — every
  save is versioned, and signed content is reproducible. `isAsSignedVersion` only flags the most
  recent signing event; across multiple sign→edit→re-sign cycles, earlier signed states remain
  recoverable but only indirectly (cross-referencing `signature.submissions` timestamps against
  version history). Worth confirming this indirect reconstruction is sufficient for audit purposes.

## 8. Testing strategy *(SA-authored)*

- **Unit:** discipline-completion derivation (content-derived vs. checkbox-derived vs.
  forced-on-Submit, all three disciplines); the signature state machine (`NOT_SENT → AWAITING →
  SIGNED`, re-send, re-sign, `editedAfterSignAt` set/clear behavior); the one-way `DRAFT → COMPLETE`
  transition; the rehab progression trend against 0/1/2+ prior reports; **[Updated, PRD round 3 item
  7]** the signer query against `staff.mobile_access == 'Doctor'`, confirming both the attending MD and
  a PA/NP with that same field value both resolve into the pool; the
  discharge-date boundary (today accepted, yesterday rejected, tomorrow accepted); **[New, PRD round 3
  item 2]** the rehab clear-on-deactivate logic — toggling a PT/OT discipline off with ratings present
  requires confirmation and clears **all** of the discipline's fields on confirm (chip ratings, PT's
  gait distance, **and** the Goal field), does not clear anything and needs no confirmation when nothing
  was entered, does not apply to Speech Therapy, and re-enabling the toggle afterward starts fresh
  rather than restoring the cleared values.
- **Integration:** list query correctness for each tab/filter/sort combination, including Attending
  MD (now indexed directly on the snapshot field) and the Case Manager/Social Worker live-join filter
  (a resident with 2+ Case Managers must surface under each one's filter, resolved via `residents` →
  `idtreports`, not a stored field); the partial unique `{facilityId, residentId}` Draft-only
  constraint (§3.6) — a second Draft for the same resident is rejected, but a new Draft is freely
  allowed once the first is Complete, no matter how recently; the PDF render/generate-and-store
  endpoints against the new field shape; server-side permission checks
  against the concrete role matrix in §7, including **[SA amendment, design review — test direction
  reversed]** a negative test that a Case Manager or Social Worker *cannot* start, edit, or complete a
  report for a resident *not* in their own `assignedStaff` (confirming the assignment check is
  genuinely enforced server-side, not just reflected in what the picker shows), a positive test that
  they *can* act on a resident who is in their `assignedStaff`, and a positive test that a Director can
  act regardless of assignment; and a negative test that a Physician cannot create/edit/delete and
  Clinical Staff cannot sign; the resident-picker scoping for Create itself (a Case Manager's picker
  defaults to their assigned residents; a Director's shows the full roster) now needs to agree with the
  backend check rather than differ from it.
- **End-to-end:** full report lifecycle — create with carry-forward, edit, submit, send for
  signature, sign (as both a Physician and a PA/NP-equivalent), edit-after-signing (**[SA amendment,
  round-2 update]** confirming the form sub-title's "Edited after signature [date]" indicator appears,
  showing current signature status alongside the date — not the earlier "no UI stamp" behavior), re-send,
  re-sign, fax (including US phone validation and Fax History), print, delete (Draft only, rejected on
  Complete). **[Confirmed by Sathish, design review]** Also: navigating away from an open report with
  unsaved changes prompts a confirmation dialog rather than silently discarding them (PRD §10).
  **[New, PRD round 3 item 3]** Once a report reaches Complete, the Rehab mark-complete checkbox
  renders disabled and ticked, and stays that way through any later re-open/edit/re-submit cycle.
  **[Confirmed, PRD round 3 item 4, §3.4a]** A
  report saved with encrypted fields round-trips correctly on read (get, list, PDF render, and the
  rehab-trend/carry-forward reads that pull content from a *different* report than the one being
  viewed), and a version row written before vs. after this change both decrypt correctly. Carry-forward
  specifically: creating a new report from a prior one decrypts the prior report's encrypted content
  and re-encrypts it fresh on the new document with a distinct IV per field (confirm the new document's
  ciphertext/IV pairs are not byte-identical to the source's, even where the plaintext is), and the
  Refresh-follows-Save diff (§3.3) correctly compares decrypted values rather than ciphertext.
- **Pre-rollout cleanup (see §9):** verify in the target environment that the existing `idtreports`
  collection contains no real (non-test) completed or signed report before it's cleared, and that
  the cleared collection is empty and ready for the new schema before the new code deploys.
- **Stale-draft cleanup:** a scheduled-job test confirming facility-time-zone evaluation, the
  configurable-days boundary, and that cleanup produces an audit log row identical in shape to a
  manual delete's.
- **Retention cleanup (§3.8):** a scheduled-job test confirming facility-time-zone evaluation, the
  exact 10-year boundary (a report one day inside the window survives, one day past it is purged), a
  facility-configured override to a non-default `versionRetentionYears` value, that both the
  `idtreports` document and every one of its `idtReportVersions` rows are removed together, and that
  the resulting `idtReportDeletionLogs` row is written with `deletedBy = 'system:retention-cleanup'`.
- **Bulk report creation (§3.9):** the picker's eligible-residents query excludes a resident who
  already has an open Draft (not, as before, one tied to a "current week"); alphabetical ordering and client-side search
  filtering against a scoped list of residents (name substring match, case-insensitive); selecting a
  mix of residents and confirming creates exactly one Draft per resident, each independently
  carrying forward and snapshotting correctly per §3.3/§3.4; a forced race (create a competing report
  for one resident mid-batch) confirms that resident is skipped with a clear reason while the rest of
  the batch still succeeds; the bulk-create response's per-resident outcome shape; the confirmed
  partial-failure toast names the skipped resident(s) and reason (§3.9), and truncates to "+N more" for
  a longer skip list; after bulk-create the list refreshes and shows the newly created Drafts (§3.9);
  no selection cap is enforced even at a large selection size.
- **Bulk report deletion (§3.10):** selecting a mix of Draft rows and confirming hard-deletes every
  selected report, purges each one's `idtReportVersions`, and writes one `idtReportDeletionLogs` row
  per report; a Complete row never carries a select checkbox and can't be included in the request
  body directly either (server-side re-check, not just a missing UI control); a forced race (mark one
  selected Draft Complete via a concurrent Submit right before the bulk delete confirms) is skipped
  rather than aborting the rest of the batch, and the confirmed partial-failure toast names the
  skipped report(s)/resident(s) and reason (§3.10); permission checks match single delete exactly
  (Clinical Staff and Director, not Physician).
- **Refresh auto-populated data (§3.3, §6):** pressing Refresh on an open Draft returns current
  backend/PCC-synced values for all three auto-populated fields without writing anything; the
  refreshed values display in the form but the persisted `idtreports` document is unchanged until the
  next Save; a Save after Refresh writes the refreshed values, moves `reportDate` to that save date
  (server-side diff of incoming vs. persisted snapshot fields — §3.3's "`reportDate` mutation" block),
  and appends a normal `idtReportVersions` row; **[SA amendment, PRD round-2 update]** verify a Save
  that does *not* follow a Refresh leaves `reportDate` untouched even though `createdAt` never changes
  either way; verify carry-forward, the rehab trend, and the retention-cleanup anchor all still reflect
  true creation order (`createdAt`) after an older report is refreshed-and-saved out of order (§3.3,
  §3.6, §3.8, §10). Refresh on a Signed report followed by a Save is recorded as a correction
  (`editedAfterSignAt` set) that is now surfaced via the form sub-title's "Edited after signature
  [date]" indicator (handoff round 2 item 8), consistent with any other edit-after-signing.
- **Create and carry-forward (§3.3):** a resident with a prior report (Draft or Complete) gets its
  team-entered content pre-filled, and its auto-populated data freshly resolved rather than copied; a
  resident with no prior report starts fully empty; `rehab.markedComplete` specifically resets to
  `false` on the new report regardless of the prior report's value; a Draft is persisted immediately
  at Create time (needed for bulk-create, §3.9), not deferred to first Save.
- **Staff mobile app notifications (§3.11):** Send-for-signature dispatches exactly one notification
  to `signature.sentTo`; Sign dispatches one notification per entry in `residents.assignedStaff`
  (including a resident with several assigned staff); a simulated dispatch failure doesn't block or
  roll back the underlying Send-for-signature/Sign write; the payload identifies the document type as
  "IDT report" distinctly from the other two document types on the same shared workflow.

## 9. Rollout and data cleanup plan *(SA-authored)*

PRD §6/A7 confirm no migration is needed: the existing `idtreports` collection has never been used in
production and holds only stale test records, which are cleaned up on rollout rather than carried
forward.

1. **Verify before clearing.** Query the existing `idtreports` collection in the target environment
   to confirm it genuinely contains no real production report — a quick sanity check before deleting
   anything, rather than taking "never used in production" purely on faith.
2. **Clear the collection** (drop and recreate, or delete all documents) before the new application
   code is deployed, so the old three-status documents can't be misread by code that only understands
   the new schema.
3. **Deploy the new application code.** No dual-read/dual-write phase is needed — there's no data to
   bridge between the old and new shapes.
4. **Contingency.** Since no transform is applied to real data, there's no migration to roll back.
   Standard code rollback (redeploy the previous version) is sufficient if something is wrong
   post-deploy; as a low-cost safety net, keep a snapshot of the pre-clear collection for a short
   retention window in case step 1's "no real data" check turns out to have missed something.

**Facility-configuration rollout steps — [Confirmed by Sathish, resolves §11 item 5]**, distinct
from the data-cleanup steps above since these are about seeding `facilities.idtReportSettings` (§3.7)
correctly before go-live, not about the `idtreports` collection itself:

5. ~~`eligibleSignerDesignations` must be populated per facility~~ — **removed, PRD round 3 item 7.**
   No facility-configurable rollout step is needed here anymore: the signer query reads
   `staff.mobile_access == 'Doctor'` directly (§3.3, §3.7), a field that already exists per-user, not
   a setting to seed per facility before go-live.
6. **`versionRetentionYears` needs no rollout backfill.** **[Confirmed by Sathish]** If the field is
   absent on a facility's `idtReportSettings` document, the application defaults it to `10` at read
   time — no migration or explicit per-facility seeding step is required before rollout. A facility
   that needs a different value (per §11 item 5's original framing — a stricter state requirement)
   sets it explicitly whenever that's identified; until then, every facility gets the same 10-year
   default with no extra rollout work.

## 10. Risks and mitigations *(SA-authored)*

| Risk | Likelihood / Impact | Mitigation |
|---|---|---|
| ~~The facility-configurable `eligibleSignerDesignations` is misconfigured for a facility~~ — **retired, PRD round 3 item 7.** This risk no longer applies now that the signer query reads `staff.mobile_access == 'Doctor'` directly — there's no facility setting left to misconfigure. | — | — |
| Reusing `residents.dischargeDate` for the IDT report's anticipated date interacts unexpectedly with the Referral workflow, which wasn't visible in this review. | Low / Medium — could cause the Referral workflow to treat an anticipated date as more final than intended. | Confirm with whoever owns the Referral workflow before implementation (§3.4 residual note). |
| Step 1 of the cleanup plan (§9) turns up real (non-test) data in the existing collection. | Low / High — would mean PRD §6's "no migration needed" premise is wrong for at least some records. | Halt the clear-and-rewrite plan and escalate to Sathish/PM immediately if any real data is found; do not proceed with §9 as written. |
| List UI for Case Manager/Social Worker (now multi-valued) isn't redesigned before Epics/Stories are drafted, and stories get written against the old single-value assumption. | Low / Medium — rework later if missed. | Flagged directly in §3.4, §6, and here; make sure whoever drafts Epics/Stories sees this note. |
| `residents.assignedStaff` isn't indexed when the Case Manager/Social Worker live-join query (§3.6) ships. | Low / Low — at 500 residents/facility this is the only realistic way the live join underperforms, and it's a one-line fix. | Confirm the index in §3.6 exists (or add it) before implementation. Not expected to require a denormalized fallback — see §3.6 for why this is a non-issue at the stated scale once the index is in place. |
| **[Superseded, design review]** The Create resident-picker's "assigned residents" default is implemented as only a UX filter, with no matching backend check — the risk this row originally warned about (treating a UX default as a hard gate) is now the *opposite* of the confirmed design. | Medium / High — since assignment is now a real authorization requirement (§6, §7), skipping the server-side check would let a Case Manager or Social Worker create/edit/complete reports for residents they aren't assigned to, which is now explicitly disallowed. | Server-side role-**and**-assignment check on Create, Save, and Submit is required, not optional (§6, §7); explicit test in §8; call this out clearly in the story/ticket, since it reverses what earlier tickets or documentation may have said. |
| **[New, PRD round 3]** The Rehab Team authorization check gets built against an assumed field or hard-coded designation list before Engineering answers PRD §11 item 3, requiring rework once the real mechanism is known. | Medium / Medium — wasted build effort, and a possible security gap if the assumed mechanism under- or over-includes staff as "Rehab Team." | This design deliberately proposes no mechanism (§6, §7, §11 item 21) — flag this explicitly in the Permissions epic's ticket so it isn't picked up and built against a guess before the answer comes back. |
| ~~The encryption scope for `idtreports`/`idtReportVersions` is left undecided into implementation~~ — **resolved, §3.4a confirmed by Sathish.** The recommended scope, pattern, and carry-forward decrypt/re-encrypt mechanism are now the confirmed design (§11 item 24 closed) — the residual risk is ordinary implementation risk, not an undecided scope. | Low / Medium — implementation must actually follow §3.4a's confirmed scope table and decrypt/re-encrypt mechanism correctly; the three design wrinkles there (carry-forward, rehab-trend reads, Refresh-diff comparison) are the specific places a build could get it wrong. | Cover each of the three wrinkles with an explicit test (§8); no further design sign-off is needed before the Report data model & lifecycle epic is scoped. |
| ~~`weightTrendSnapshot`'s calendar-week-slot design ships despite PRD §4.1's "no calendar-week grouping or bucketing" wording~~ — **resolved, confirmed by Sathish.** `weightTrendSnapshot` is simplified to directly mirror `residentObservationTrends.readings` (§3.4) — no calendar-week logic anywhere. §11 item 25 closed. | — | — |
| The retention-cleanup job (§3.8) purges a report before its facility's actual configured retention window has elapsed, due to a timezone or off-by-one bug. | Low / High — the failure mode is irreversible loss of a compliance-relevant clinical record, not just an ordinary bug. | Run the job in report-only (dry-run) mode for at least one full cycle before enabling actual deletion, and keep a short-term backup/snapshot of anything deleted for a grace period after each run, per §3.8. |
| Bulk-create's or bulk-delete's partial-failure toast (§3.9, §3.10) isn't implemented, and a user assumes every selected resident/report was processed when one or more were silently skipped. | Low / Medium — a Case Manager could believe a resident has a new report, or a report was deleted, when it wasn't. | The per-item outcome and suggested toast copy are now specified for both bulk actions (§3.9, §3.10); cover with the explicit test in §8. |
| The eligible-residents query's "exclude residents with an open Draft" filter (§3.9) has a stale read against a concurrent bulk-create from another user, offering a resident who's actually just been given a Draft. | Low / Low — self-corrects: the per-resident failure handling in §3.9 catches this at create time and reports it, rather than silently corrupting data. | No dedicated mitigation needed beyond the per-resident failure handling already designed in §3.9. |
| The MongoDB driver/ODM layer in use doesn't support partial indexes as assumed for the new `{facilityId, residentId}` Draft-only unique index (§3.6). | Low / Medium — the invariant itself is now confirmed (PRD §6 rule 5); the residual risk is purely a tooling-support gap, not a wrong design. | Confirm partial-index support in the actual driver/ODM/migration tooling in use before implementation. |
| Bulk delete (§3.10) trusts the client's belief that a selected row is still a Draft, instead of re-checking server-side, and hard-deletes a report that was just marked Complete by someone else. | Low / High — would delete part of the clinical record that's supposed to be undeletable once Complete. | Server-side re-verification of `status === 'DRAFT'` per report at delete time is already designed into §3.10, not left to the client; cover with the explicit test in §8. |
| A push-notification failure (§3.11) is implemented as blocking, holding up or failing the Send-for-signature or Sign write itself. | Low / Medium — would make a clinically meaningful action fail because of an unrelated mobile-push outage. | Best-effort, non-blocking dispatch is explicitly designed into §3.11; cover with the explicit test in §8. |
| The "IDT report" document type isn't correctly registered in the existing shared notification workflow (§3.11), so notifications either don't fire or read as the wrong document type. | Low / Medium — confusing or missing notifications, not a data-integrity issue. | Confirm the registration mechanism with whoever owns the shared workflow before implementation (§11). |
| The Send-for-signature picker gets built as multi-select, by analogy to the Create picker (§3.9) — both are described as "opening a picker" over a list of people/residents, but only one of the two is multi-select. | Low / Medium — would contradict the single-select `sentTo` field and the "exactly one recipient" rule (§3.3/§3.4). | The explicit single-select language now in §3.3/§3.4 is meant to preempt this; call it out specifically in the story/ticket for whoever builds the Send-for-signature picker. |
| **[SA amendment, PRD round-2 update]** Implementation switches carry-forward, the rehab trend, the general lookup index, or the retention job (§3.3/§3.4/§3.6/§3.8) to key off `createdAt` in some places but leaves a stray `reportDate` reference in others, given how many call sites this touches. | Medium / High — a missed spot would silently reintroduce the exact sequencing/retention bug this change exists to prevent (e.g., a refreshed-and-resaved report resetting its own retention clock). | Every affected call site is now enumerated in §3.3's dedicated "`reportDate` mutation" block and cross-referenced from §3.4/§3.6/§3.8; use that block as the implementation checklist and cover with an explicit test in §8 (create two reports, refresh + save the older one, confirm ordering/retention still reflect true creation order). |
| Engineering builds against the PRD's literal text alone (which only specifies `reportDate` moving, and says nothing about `createdAt`), missing the confirmed `createdAt`-anchored resolution (§3.3, §11 item 16). | Low / Medium — this is now a confirmed decision, not just an SA recommendation, but it's easy to miss since the PRD itself doesn't state it. | **[Confirmed by Sathish]** Use the TD, not the PRD alone, as the source of truth for this item when drafting stories — reference TD §3.3's dedicated "`reportDate` mutation" block directly in the story/ticket. |
| **[SA amendment, design review]** `idtReportVersions.content` isn't updated to also capture `patientInformationSnapshot`/`weightTrendSnapshot`/`medicationsSnapshot` on every save, leaving version history unable to reconstruct what the auto-populated data looked like at signing time if the report is later refreshed and saved again. | Medium / High — would silently break the PRD's "a signed report must render years later exactly as it was signed" requirement (PRD §4.1) for any report refreshed after signature. | Fixed in this round's design (§3.4) — every version row now captures all three snapshots, not only the previously-versioned team-entered content; cover with an explicit test in §8 (sign a report, refresh + save it, confirm the resulting version row's snapshots match what was actually saved, not the pre-refresh values). |

## 11. Open questions

**Still genuinely open** *(mix of Team-submitted and SA-authored)*:

| # | Question | Current position |
|---|---|---|
| 2 | Code Status — confirmed the source is PCC, but nothing syncs it into the backend today. Which PCC API/resource should this come from? | Open — intentionally left open per Sathish, not blocking. **[PRD round 3 item 6]** The Diagnosis half of this item (formerly Chief Complaint) is now resolved — renamed, and the PM confirms it's available from PCC. Code Status remains open. |
| 13 | Exact mechanism for registering "IDT report" as a new document type in the existing shared staff-mobile-app notification workflow (§3.11) — config, template, or code enum. | Open — not visible in this review, since it lives outside the IDT reports codebase; needs confirming with whoever owns that workflow. |
| 17 | Confirm no code path currently grants IDT-report creation rights to a "Nurse" designation specifically, now that PRD §3 narrows the creating persona to Case Manager only (handoff round 2, item 1). | Open — not visible in this review; needs a one-line confirmation from whoever owns the permissions model, per the handoff document's own framing. This design's Clinical Staff definition (Case Manager, Social Worker, Rehab Team) never granted creation rights to a "Nurse" label to begin with, so no design change is expected either way. |
| 20 | **[New, PRD round 3 item 2 — matches PRD §11 item 2]** Does the Director's facility-wide access remain exempt from the new resident-assignment gating (§6, §7), or should Directors also be scoped to assigned residents? | Open, medium priority — raised in design review, not yet settled by the PM or Sathish. This design deliberately does **not** assume Directors keep their bypass; treat the Director's authorization path as unconfirmed until this is answered. |
| 21 | **[New, PRD round 3 item 8 — matches PRD §11 item 3]** How does the system determine which staff count as "Rehab Team" for authorization purposes (§6, §7, PRD §8), now that Rehab Team is no longer a single "Director of Rehab" designation? | Open, medium priority, **explicitly reserved for Engineering** — not a product or SA-design decision. A similar approach to the now-resolved signer-resolution mechanism (`staff.mobile_access`, item 7) may end up applying, but the PM was explicit this isn't to be assumed by the PM, the AI, or the SA agent. This design proposes no field, list, or setting for it. The Permissions epic isn't ready for story-writing until this comes back from Engineering. |
| 22 | **[New, PRD round 3 item 9 — matches PRD §11 item 4]** Should the `mm/dd/yy` date format (PRD §4.3) be a fixed platform-wide value, or a facility-configurable setting like `staleDraftCleanupDays`/`versionRetentionYears`? | Open, low priority, not yet decided. This design's date-display work (round 2, §6) was built assuming a single fixed format — if this becomes a facility setting, that work would need a small follow-up (a new `facilities.idtReportSettings` field plus a formatting-utility change), not a redesign. Non-blocking either way. |

**Resolved by Sathish this round** (kept here briefly for traceability, not as open items): signer
picker is single-select; discharge date accepts today or later; fax ships in this release;
`residents.dischargeDate` (existing field) is reused rather than adding `anticipatedDischargeDate`;
report-to-staff assignment is out of scope. On Case Manager/Social Worker specifically: they're
confirmed multi-valued by design, no dedicated designation fields are added to `idtreports` at all
(resolved live off the existing `residents.assignedStaff` instead, §3.4/§3.6), any Case Manager or
Social Worker can start or complete any report they can reach, and the one place `assignedStaff`
actually matters is narrowing the Create resident picker's default view (§6) — a UX default, not a
permission gate. A default 10-year, facility-configurable retention window (§3.8) now governs how
long a report and its full version history are kept, closing the gap that `idtReportVersions`
previously had no lifecycle policy at all. The Create picker now supports selecting multiple
residents at once and generating a report for each in one action (§3.9), alphabetically sorted and
live-searchable. Bulk delete of Draft reports is now in scope per a PRD update (§3.10) — multi-select
on the list, one audit entry per report, same permissions and same hard-delete semantics as single
delete; bulk submit and bulk send-for-signature remain explicitly out of scope and single-report-only,
since a bulk selection can span more than one attending physician. Most recently, following a
PRD-vs-TD consistency review: the Create/carry-forward flow and the Send-for-signature step, both
previously undocumented in this design despite being referenced elsewhere in it, are now written up
explicitly (§3.3); the two staff mobile app push notifications added to scope now have a full design
(§3.11); and a missing index for the list's resident A–Z sort has been added (§3.6). Two other items
from that same consistency review — the discharge-date valid-range wording (PRD §5.2) and the
signer-picker's "attending MD and their PA/NP" wording (PRD §5.4) — are being resolved by updating the
PRD to match this design, which Sathish confirmed is correct as written; no TD change was needed for
either. Most recently, the PM removed the fixed weekly cadence assumption entirely (PRD §1/A1, and
throughout Sections 2, 4, 5, 10): reports are now created whenever staff need one, not on any
calendar rule. This cascades through the whole document — `weekOf` is removed from `idtreports` and
`idtReportDeletionLogs` entirely (§3.4), the old unique `{facilityId, residentId, weekOf}` index is
replaced with a partial unique index enforcing "at most one open Draft per resident" plus a general
lookup index (§3.6), the rehab progression trend is relabeled report-over-report to match the PRD
(§3.3), carry-forward and the bulk-create/bulk-delete exclusion filters now key off "has an open
Draft" instead of "current week" (§3.3/§3.9), and the weight-trend snapshot's calendar-week anchor is
re-derived from `reportDate` instead of the removed field (§3.4) — its own calendar-week *behavior* is
intentionally unchanged, since it reflects a real, separate PCC weighing cadence. The two points raised
as SA recommendations from that pass are now both resolved by direct PRD updates: PRD §6 rule 5 states
outright that a resident has at most one open Draft at a time (confirming the Draft-only partial unique
index, §3.6, as designed — no change needed) and that a resident with an open Draft is excluded from
the Create picker until it's Complete (confirming §3.9's exclusion filter as designed); and PRD §9 now
states explicitly that the "52 reports/resident/year" figure is a planning-ceiling placeholder, not
derived from cadence, to be revisited once real deployment data exists — also requiring no design
change, just the framing already carried in §2.1/§3.2/§7.

Most recently, Sathish resolved nine of eleven remaining open items in one pass (former items 3–9,
11–12; item 10 stays open, see below): auto-populated data staleness gets a user-triggered **Refresh**
action rather than a passive indicator, re-running the Create-time resolution logic on demand and
persisting only via the report's normal Save (§3.3, §6, §8); `eligibleSignerDesignations` taxonomy and
`versionRetentionYears` defaulting are both now explicit rollout/deployment steps (§9) rather than open
design questions, one requiring per-facility population before go-live and the other requiring no
rollout work at all since it defaults to 10 at the application level when absent; storage tiering for
`idtReportVersions` moves from an open question to a tracked deferred technical item, a candidate for
its own future epic (§12); bulk-create's post-creation landing behavior is confirmed as returning to
the list, which must refresh to show the new Drafts (§3.9); both bulk actions' partial-failure UX is
confirmed as a toast/alert naming the skipped residents/reports with suggested copy, and no cap is
placed on bulk-create's selection size (§3.9, §3.10); and both remaining Create/carry-forward
assumptions are confirmed exactly as this design proposed — `rehab.markedComplete` resets to `false` on
a new report, and "Start blank"'s lighter client-side-only reset is sufficient (§3.3). **Item 10 (the
Create picker's exact resident display-name field) was deliberately left open** — Sathish asked that
this specific one be left for engineering to confirm against the real `residents` schema rather than
decided here; this design's recommendation (use whichever field feeds
`patientInformationSnapshot.residentName`, §3.9) stands as a recommendation only, not a confirmed
decision — see §11 item 10. **Item 10 has since been resolved**, once the actual `residents` schema was
reviewed during this round's design-review pass: the field is `residents.name`, a pre-composed display
name distinct from `firstName`/`lastName` (§3.9).

**Item 18 (`createdByType`/`updatedByType`) has also been resolved, and removed.** Sathish confirmed
these are remnants of the previous IDT report structure, not required in this design — they carried no
notes or explanation in the original team submission because there was nothing left for them to do.
Both fields are dropped from `idtreports` (§3.4); every actor-type distinction this design actually
needs is already covered elsewhere (`deletedBy`'s free-form actor strings, plain `createdBy`/
`updatedBy`/`savedBy` attribution).

**Item 19 (weight-trend dependency) is now fully resolved — removed from the open-questions table.**
With the real `observations` collection reviewed, **[Confirmed by Sathish]** the retention-scope
question is settled: `residentObservationTrends` keeps only the 3 most recent readings per resident per
type as a fixed-size ring buffer (the latest reading replaces the oldest on every update, no
calendar-week bucketing) — `observations` itself is untouched and remains the full per-event
audit-of-record, exactly as this design recommended. **[Confirmed by Sathish]** the encryption question
is also settled: vitals (including weight) are PHI, so `residentObservationTrends` stays field-level
encrypted exactly like `observations`, with no relaxation. **[Confirmed by Sathish]** the one remaining
point — `recordedAt` staying plaintext in the new collection, the same way `observations` already
carries plaintext `createdAt`/`updatedAt` — is confirmed acceptable; the proposed object shape and
approach are confirmed as-is with no further changes needed. Full design in §3.4.

**Most recently, PRD round 2 (`idt-reports-handoff-round2.md`) made eight further changes, each
reviewed against this design:** the "Case Manager / Nurse" persona is now "Case Manager" only — this
design's Clinical Staff matrix never included a Nurse designation, so no change was needed beyond a
one-line permissions confirmation (§7, §11 item 17); dates render everywhere in a single `mm/dd/yy`
format, a display-only change with no TD impact (§6); the list's document-status filter is removed,
which this design never modeled separately from the tab split, so no TD action was needed (§6); the
Fax popover drops its saved-contacts shortlist in favor of a single free-text number validated to US
phone format, added to `faxLog`'s field notes and the Fax send endpoint (§3.4, §6); a new Fax History
section needs a `status` field added to `faxLog` but **no new endpoint**, since `faxLog` already
returns with every report fetch (§3.4, §6); the Status chip is replaced by a Signature Status column
on the Completed tab, resolved by exposing the already-existing `signature.state` field in list
responses with no new index (§6); and the "Edited after signature" fact — previously deliberately
silent everywhere — is now surfaced on the report form's sub-title, resolved by exposing the
already-existing `signature.editedAfterSignAt` field rather than adding a new one (§3.3, §3.4, §6).
The Create-picker open-Draft exclusion was separately reconfirmed as-is, with no technical design
action needed. **The one substantive, non-cosmetic change in this round — Refresh-then-Save now moves
`reportDate` — has ripple effects the PRD's own handoff note flagged for review:** `reportDate` was
this design's sole ordering/anchor key for carry-forward, the rehab trend, the general lookup index,
and retention, and a mutable `reportDate` would let those all drift or reset unsafely. This design's
resolution is to decouple the two: `reportDate` keeps its new PRD-defined meaning for display/filing
only, while every actual sequencing/retention use switches to the already-existing, genuinely-immutable
`createdAt` field, detected via a server-side diff at Save time rather than a client-supplied flag
(§3.3's dedicated block, §3.4, §3.6, §3.8). **Confirmed by Sathish** — even though the PRD text itself
only specifies `reportDate` moving and is silent on `createdAt`, this design's resolution is now the
confirmed source of truth: `reportDate`'s job is specifically to capture the report's date as revised
by a Refresh + Save, while `createdAt` (already existing, genuinely immutable) is the one and only key
for carry-forward, the rehab trend, the general lookup index, and retention. This closes former open
item 16 — no PRD change is needed, since the PRD's silence on `createdAt` isn't a contradiction, just a
level of detail the PRD doesn't need to carry.

**This `createdAt`-anchored resolution was revisited once more during design review** — the question
raised was why not sort carry-forward and the rehab trend by `reportDate` alone, since a Refresh
naturally makes it the latest date on the report. **Reviewed and reconfirmed: no change.** Refresh only
re-pulls the three auto-populated snapshots (Patient Information, Weight trend, Active medications) —
it never touches `rehab.pt`/`rehab.ot` or any other team-entered content — so refreshing-and-saving an
old Complete report doesn't make its actual clinical assessment content any more current; only its
`reportDate` label would move. Sorting the rehab trend or carry-forward by `reportDate` would let that
untouched-but-refreshed report jump ahead of a genuinely newer one, displaying stale rehab data as "Last
Report" or carrying forward outdated content into a new Draft. `createdAt` (true encounter order) stays
correct for those two uses; `reportDate` remains correct for the grid's display/sort (§3.6) and for
retention stays anchored on `createdAt` as designed (§3.8).

**PRD round 3 (`idt-reports-handoff-round3.md`) brought nine further changes, reviewed against this
design:** the Refresh control was already designed as a single consolidated action across all three
auto-populated sections (§3.3) — PRD round 3 item 5 confirms this matches, no design change needed.
Chief Complaint's rename to Diagnosis and its confirmed PCC source (item 6) close the Diagnosis half of
former item 1 above; Code Status (item 2) stays open. The signer-designation-taxonomy mechanism is
resolved more simply than previously designed (item 7): `staff.mobile_access == 'Doctor'` replaces the
proposed `eligibleSignerDesignations` facility setting entirely — removing a facility setting, a
rollout step, and a risk row this design previously carried (§3.3, §3.7, §9, §10). Two new,
well-scoped frontend features are added and fully specified: the Rehab discipline clear-on-deactivate
confirmation (item 2, §3.4) — **[Confirmed by Sathish]** now fully resolved: clearing extends to the
Goal field along with the ratings, and re-enabling the toggle afterward starts fresh rather than
restoring the cleared values, closing former item 23 — and the Rehab mark-complete checkbox becoming
disabled once Complete (item 3, §3.4). Rehab Team is confirmed no longer scoped to a single "Director of
Rehab" designation (item 1), and the resident-assignment requirement is confirmed to extend to the
Rehab Team the same way it applies to Case Manager/Social Worker (§6, §7) — but **the mechanism for
determining Rehab Team membership is explicitly reserved for Engineering** (item 8, §11 item 21), not
a decision this design, the PM, or the SA agent gets to make, unlike the now-resolved signer question
it otherwise parallels. The Director-bypass question, previously carried in this document as an
unstated assumption, is corrected to match its real status: a formally open PRD question (§11 item 20,
PRD §11 item 2), not something this design should treat as settled either way. Date format is flagged,
not decided, as a possible future facility setting (item 9, §11 item 22) — non-blocking. PRD §9/§12
introduce a genuinely new requirement this design hadn't addressed: field-level encryption of all
`idtreports`/`idtReportVersions` data (item 4) — the recommended scope and pattern proposed in the new
§3.4a, including the carry-forward decrypt/re-encrypt mechanism it requires, are **[Confirmed by
Sathish]**, closing former open item 24; no materially better alternative to the decrypt/re-encrypt
cycle exists without introducing new cryptographic risk or breaking an already-established design
invariant (§3.4a has the full alternatives analysis). **One additional inconsistency surfaced
independently during this review, not itself a round 3 handoff item, is also now resolved:** PRD §4.1's
wording for the weight trend ("a rolling set of 3... with no calendar-week grouping or bucketing") is
**[Confirmed by Sathish]** the correct behavior — `weightTrendSnapshot` is simplified to directly
mirror `residentObservationTrends.readings` (§3.4), with no calendar-week logic anywhere in either
collection. This closes former open item 25. Round 3 leaves only two genuinely open items: the
Rehab Team authorization mechanism (item 21, reserved for Engineering) and the Director bypass
question (item 20) — plus the low-priority, non-blocking date-format question (item 22).

## 12. Deferred technical items (candidates for future epics) *(SA-authored)*

Distinct from PRD §10's product-level next-phase list (already reflected in this design's non-goals,
§2.2): these are technical/engineering-only deferrals that don't change any user-facing behavior, so
they don't belong in the PRD, but are worth tracking here so they aren't lost between this design and
whenever Epics/Stories get drafted.

- **Storage tiering for `idtReportVersions` within the 10-year retention window (§3.8).** **[Confirmed
  by Sathish]** Version history is essentially never read once a report is old and settled, making it
  a reasonable candidate for moving to cheaper storage (a cold collection, or object storage) well
  before the 10-year retention cutoff — purely a cost optimization, not a retention-policy change (the
  data is still retained for the full window either way). Not needed for this release; worth spinning
  into its own engineering spike/epic once real production volume data exists to justify the cost/effort
  tradeoff. No design decision is needed now — the retention mechanism in §3.8 works identically
  whether or not tiering is ever added on top of it.
- **Stale resident-data detection and remediation (§3.4, resolved former §11 item 19).** **[Confirmed by Sathish — later
  phase]** Once `residentObservationTrends` (or an equivalent capability) exists, add a scheduled check
  for residents whose data (weight, and potentially other observation types this same mechanism covers
  later) hasn't been refreshed in more than a week. On detection, two actions in sequence: first, the
  system attempts an on-demand PCC API pull for that resident (distinct from the passive webhook —
  actively asking PCC for current data, in case a webhook was missed or delayed); second, if the entry
  is still stale after that attempt, raise a notification to the resident's assigned staff
  (`residents.assignedStaff`), likely via the same staff mobile app notification workflow already built
  for the signature flow (§3.11), rather than a new channel. Not part of this release — this is a
  future-phase capability layered on top of the weight-trend dependency above, not something IDT
  Reports itself needs to build, but worth tracking now so it isn't lost.
