# Module: Care Coordination

> Applies to: Skilled Nursing
> FR prefix: CARE
> Sources: `_codebase-analysis/backend-clinical-care.md` (§0, §4b, §4c, §6, §9, §10), `_codebase-analysis/client-admin-web.md` (§2.10–§2.12, §3, §4.4–§4.5), `_codebase-analysis/client-resident-app-sn.md` (§3.3, §3.10, "Explicitly absent"). Code is source of truth; this document describes as-built behavior plus observed gaps.
> 2026-07-12: **Referrals & agencies extracted to its own module — [referrals.md](./referrals.md)** (REF prefix). This document now covers care conferences, IDT reports, and case-manager/doctor schedules. The staff-app "Pending Sign" doctor e-signature surface moved with referrals (REF-FR-05a).
> 2026-07-12 delta: `docs/reviews/2026-07-12/review-senior_living_staffapp.md` (dead staff-app Reports tab, O-10).
> 2026-08-28 delta: `assignedStaff[]` (BR-7) gained a second, automated writer — PCC practitioner sync — documented in full in `clinical-records.md` §3F (CLIN-FR-28 series), not re-derived here; see the BR-7 cross-reference below.
> 2026-08-28 delta: **new admin-web Care Conference Calendar view — new CARE-FR-26a.** The care-conference nav on `senior_living_admin` was restructured into a two-item "Care Conference" group (Schedule Care Conference + Care Conference Calendar); the calendar is a second, read/edit/delete view over the same `CareConference` records described in CARE-FR-20/21/25a, not a new scheduling model. Full admin-UI behavior in `client-admin-web.md` §2.11 (backfilled 2026-08-28 from the architecture doc's v2.2 delta, which first captured the component's addition against production HEAD `59d22ea`).

---

## 1. Purpose & scope

Care Coordination is the SNF clinical-workflow umbrella covering three interlocking capabilities by which a facility's interdisciplinary team plans, meets about, and documents a resident's care:

1. **Care conferences** — scheduled IDT/family meetings (virtual via Zoom or in-person with recorded audio + AI transcripts), with a status-machine-driven review pipeline, resident-shareable summaries, and (as of this pass) both a list/editor screen and a dedicated calendar view in the admin web app.
2. **IDT reports** — a multi-section interdisciplinary clinical snapshot collaboratively completed by case manager, rehab, and social worker, auto-submitted when all three disciplines have contributed, with PDF export and versioned history.
3. **Case-manager & doctor unified schedules** — persona-variant day-schedule aggregators across salon, transportation, rehab, and care-conference bookings.

> **Discharge referrals to Home Health Agencies** — including the dynamic discharge-order form, in-house physician e-signature, agency directory, and the staff-app "Pending Sign" mobile signing surface — are documented in their own module: **[referrals.md](./referrals.md)** (REF prefix). Referrals share this module's care-team designation / `assignedStaff[]` gating backbone (§7).

**In scope:** the workflows above as implemented in `senior_living_backend` (`/api/care-conference`, `/api/reports`, `/api/case-manager`) and surfaced in `senior_living_admin` (Care Conferences — Schedule + Calendar views, IDT Reports screens). Care-team designation gating (the role→resident scoping that governs all of the above and referrals) is documented here as the shared access-control backbone.

**Out of scope:** referrals & agencies (Referrals module — [referrals.md](./referrals.md)); rehab appointment scheduling and the therapy catalog (Rehab module); medications/labs and advance care directives (Clinical Records module — see CLIN-FR-19a/CLIN-GAP-09 for the ACD half of the doctor e-signature story, and CLIN-FR-25/26 for Consent Forms and the Resident Documents feed, both admin-web surfaces new this same interval and out of scope here); chat (Messaging module — though the care-team gating defined here governs who residents may chat with); the AL-side `/api/care` booking engine (Wellness module — referenced only for its security observations, §9). **Also out of scope:** how `assignedStaff[]` is populated by the PCC integration (automatic practitioner-name matching, ambiguous-match handling, unassignment on practitioner removal) — that is documented in clinical-records.md §3F (CLIN-FR-28 series); this module only documents how the array, however populated, is *consumed*.

---

## 2. Personas & surfaces

| Persona | Surface | Role in this module |
|---|---|---|
| Case Manager (staff) | Admin web, staff app | Fills IDT case-manager sections; hosts/attends care conferences; uses the case-manager day schedule ("what I booked / must attend today"). *(Also drafts referrals — [referrals.md](./referrals.md).)* |
| Doctor (staff) | Admin web, staff app | Referenced as attending MD on IDT reports; uses the doctor schedule variant (per-patient itinerary). *(Reviews and e-signs referrals via the admin web and the staff-app "Pending Sign" tab — [referrals.md](./referrals.md) REF-FR-05/05a; that tab replaces My Schedule for doctor-designation users.)* A doctor's presence on a resident's `assignedStaff[]` may now originate automatically from PCC practitioner sync rather than a manual admin pick — clinical-records.md §3F. |
| Social Worker / Rehab staff | Admin web, staff app | Complete their IDT report sections; participate in care conferences as care team |
| Admin | Admin web | Full visibility on conferences (not host-scoped); all IDT operations; can browse/edit/delete conferences via either the Schedule Care Conference list or the new Care Conference Calendar (CARE-FR-26a) — both are views over the same records |
| Resident | SN resident app | **Care conferences now live on staging:** the skilled-nursing resident app ships a Care Conference module (summaries list, detail, history-detail, in-person/virtual join) consuming `GET /api/care-conference/my-conferences` (+`/history`) via `fetchMyCareConferences` (`src/screens/App/HealthScreen/CareConferenceScreen/`, `src/services/App/index.ts`). **IDT (and referrals) remain absent** from the resident app. Case manager also exists as one of the chat care-team contacts (§3.3) |
| Family member | (no UI found) | Backend models family members as conference participants (`familyMemberCNames[]`) and notification recipients; family identity normalized to the linked resident at auth time, original preserved in `familyMemberCName` |

Surfaces: **admin web** is the primary operating surface for both remaining capabilities (`CareConferenceReports.tsx`, `CareConferenceCalendar.tsx` (new, CARE-FR-26a), `IDTReport.tsx`). The backend additionally serves staff-app and resident-app endpoints (my-conferences, case-manager schedule PDF) whose mobile consumers were, until staging, partially or wholly unbuilt.

---

## 3. Functional requirements (as-built)

### 3A. Care conferences

- **CARE-FR-20 — Conference entity & participants.** `CareConference`: host staff (`staffCName` — also the Zoom host via their linked OAuth Zoom account), `residentCNames[]` (multi-resident), `familyMemberCNames[]`, `careTeamCNames[]` (staff). Participants joined by cName, not `_id`. Meeting types (admin UI): `Care Conference` / `Family Meeting` / `Family Care Conference`. Scheduling form: residents* multi-select, care team* multi-select, family members **auto-derived from the selected residents' family lists** (no free entry), date* + time* (24h), duration 15/30/40/45/60/90 min (>40 min triggers a Zoom-limitation warning), where (`In Person` / `Virtual`), location (free text / Zoom), notes.

- **CARE-FR-21 — Status machine.** `SCHEDULED → IN_PROGRESS → IN_REVIEW → COMPLETED`, plus `CANCELLED` (`constants/careConference.ts`):
  - `SCHEDULED` on create.
  - **Enabler cron** (`careConferenceEnable.cron.ts`, every minute, env `ENABLE_CARE_CONFERENCE_ENABLE_CRON`) sets `isEnabled: true` + `IN_PROGRESS` within 5 minutes of start time (T-5) and pushes "starting" notifications to residents, host, and care team.
  - **SMS reminder cron** (`careConferenceSmsReminder.cron.ts`, env `ENABLE_CARE_CONFERENCE_SMS_REMINDER_CRON`): sends SMS reminders to conference attendees ahead of their scheduled conferences. Runs as a separate cron job independent of the enabler cron; fires alongside the FCM push channel.
  - Attaching audio recordings (in-person only) → `IN_REVIEW` (`careConference.service.ts:775`).
  - `POST /:id/complete`, or a summary review via `PUT /:id/update-summary` (allowed from IN_REVIEW or COMPLETED) → `COMPLETED`. Summary update stamps `summaryUpdatedBy/At` and back-fills every processed recording with `updatedSummary/updatedBy/updatedAt`.
  - `DELETE /:id` = **soft cancel** to `CANCELLED`, permitted only from `SCHEDULED`; removes calendar events and sends cancellation notifications.
  - The 15-minute appointment-completion cron also auto-completes overdue conferences (backend §0).

- **CARE-FR-22 — Virtual conferences (Zoom).** `where === 'Virtual'` → Zoom meeting provisioned **before** the DB write; Zoom failure aborts creation with 502. Stores `zoomMeetingId`/`joinUrl`/`startUrl`, plus (staging) `pstnPassword` and `dialInNumbers[]` (`ZoomDialInNumber`) for PAC phone-conference dial-in; scheduling-field updates PATCH the Zoom meeting. Response shaping (`shapeForCaller`): host receives `startUrl`, all other callers receive `joinUrl`. The admin UI exposes a **`JoinByPhoneSection`** component that surfaces the PSTN dial-in numbers and passcode for attendees without a Zoom client — reused verbatim by the Calendar view's event-detail popup (CARE-FR-26a). `transcriptText` may be fetched from Zoom after a `recording.completed` event.

- **CARE-FR-23 — In-person conferences (S3 audio + Lambda transcripts).** Mobile uploads audio via presigned S3 PUT (`POST /audio-presign`, supports multiple filenames; keys attachable atomically); `PATCH /:id/recording` persists keys (transitions to IN_REVIEW); an external **transcribe-processor Lambda** populates `recordings[]` with per-part transcript + AI summary (read-only in admin UI). `GET /:id/recording-urls` returns 1-hour presigned GETs.

- **CARE-FR-24 — Summary & share-with-resident.** Post-meeting summary edited via `PUT /:id/update-summary { summary, shareWithResident }` (permitted from `IN_REVIEW` or `COMPLETED`); it sets `status: COMPLETED`, stamps `summaryUpdatedBy/At`, and back-fills every recording's `updatedSummary/updatedBy/updatedAt`. The write uses `updateOne` **specifically to bypass the `findOneAndUpdate` post-hook**, so a summary edit does **not** itself trigger a UnifiedSchedule sync — summary review is a documentation-only action. The `shareWithResident` boolean controls whether the reviewed summary is exposed on resident-facing reads. (No SN resident app consumer exists today — §9, observation O-4.) (`careConference.service.ts:796-839`)

- **CARE-FR-25 — Calendar sync & UnifiedSchedule mirroring.** Google Calendar events are created for the host **and every care-team member** (per-staff event IDs stored in `careTeamGoogleEventIds`), with the Zoom link in the description; `calendarSyncStatus` aggregates sync state. Cancellation removes the calendar events. Conferences are mirrored into `UnifiedSchedule` per resident via `CareConference` model hooks (`post('save')`, `post('findOneAndUpdate')`, `post('findOneAndDelete')`), not by individual service writes (`CareConference.model.ts:155-184`). **Conference visibility in UnifiedSchedule list reads is restricted to `IN_PROGRESS` and `SCHEDULED`**: a DB-level `status: { $in: [IN_PROGRESS, SCHEDULED] }` branch for `CARE_CONFERENCE` rows, then an **authoritative post-population status check** that drops any row whose live `CareConference.status` is not visible (the mirrored `UnifiedSchedule.status` may be stale). A `skipCareConferenceStatusFilter` option lets callers fetch all rows and post-filter themselves. A completed conference therefore disappears from upcoming reads via this status filter — there is no explicit "remove from schedule on complete" write. (`unifiedSchedule.service.ts:139-169`) *(Note: this is the shared UnifiedSchedule mirror, distinct from the admin-web Care Conference Calendar's own `care-conference` list query — CARE-FR-26a — which queries `CareConference` directly with an explicit date range, not via UnifiedSchedule.)*

- **CARE-FR-26 — Listing & visibility.** Management routes require STAFF|ADMIN. Non-admin staff listing (`getCareConferences`) is scoped to conferences they **host** (`staffCName` filter); admins see all. Residents/family use `GET /my-conferences` (+`/history`) — any authenticated caller, scoped by a participant `$or` filter; family identity matched on the original `familyMemberCName`. Admin UI ("Schedule Care Conference" screen) tabs: Upcoming (SCHEDULED/IN_PROGRESS/IN_REVIEW) vs History (COMPLETED/CANCELLED); search (resident/meeting type/family names) and pagination are **client-side** over a `limit: 100` fetch.

- **CARE-FR-26a — Care Conference Calendar view (admin web, new).** A second admin-web surface over the same `CareConference` records, reached via a restructured top-level "Care Conference" nav group (`schedule-care-conference`, two sub-items: "Schedule Care Conference" → CARE-FR-26's screen, and "Care Conference Calendar" → this requirement) — not a new scheduling engine or a new backend endpoint set.
  - **Views:** day / week / month, default month; month view groups same-time conferences side-by-side with a "+N more" expand; week/day views render an hour-by-hour grid (7 AM–10 PM) with an overlap-safe layout algorithm shared with the Transportation calendar (client-admin-web.md §2.11/§4.6).
  - **Data fetch:** `GET /care-conference?limit=1000&status=SCHEDULED,IN_PROGRESS,IN_REVIEW,CANCELLED&startDate&endDate`, refetched per visible date range. **The calendar's own status filter omits `COMPLETED`** and its status-colour legend has no entry for it — a completed conference is therefore not visible on the calendar even though it remains visible in the sibling Schedule screen's History tab. This is a client-side query choice (not a backend restriction — `COMPLETED` conferences are not excluded by the API itself) and should be confirmed as intentional or fixed to match the History-tab behavior.
  - **Event detail popup:** resident identity + photo, status, date/time range, location (with `JoinByPhoneSection` when the location is a phone dial-in, CARE-FR-22), a combined care-team + auto-derived-family-member roster with the host's entry tagged "(Host)" rather than listed separately, agenda, and post-meeting notes/summary — all read from the same fields CARE-FR-20/21/24 already define.
  - **Edit / delete directly from the calendar**, using the same validated form and the same schedule-conflict detection as CARE-FR-25a below (shared component import, not a re-implementation); delete requires a confirmation dialog (a UX addition also backfilled onto the Schedule screen — CARE-FR-26).
  - **No "create new conference" affordance on the calendar** — scheduling remains exclusively the "Schedule Care Conference" screen's job; the calendar is a browse/edit/delete surface layered on the same data.
  - Timezone-aware: "today" highlighting and the default landing date use the facility's configured `timeZone`, not UTC (contrast the Rehab Calendar's deliberate UTC-date-key policy — these are two different timezone conventions coexisting in the app, `client-admin-web.md` §4.4).
  - No frontend permission gate (shares this gap with CARE-FR-26 and IDT — §7/§9 O-7).

### 3B. IDT reports

- **CARE-FR-40 — Report structure.** `IDTReport` sections: `basicInformation` (resident ref, `attendingMD` **string** — free-text physician name (changed from ObjectId ref, production 2026-06-22), DOB, room, admission date — **`birthDate` and `admissionDate` now surfaced as explicit editable fields in the admin IDT form**, patient phone + 2 family contacts), `medicalOverview` (codeStatus e.g. Full Code/DNR, weight, allergies, diet), `clinicalDetails` (changeOfCondition + linked upcoming `RehabAppointment` refs), `therapyDetails` (PT: bedMobility/transfers/gait/device; OT notes; Speech notes), `additionalNotes` (skinIssues), `dischargePlanning` (notes, destination Home/AL/Hospital…, DME needed, caregiver needed), `medications: string[]` (pill-style add/remove list), role fields `caseManager`/`rehabMembers`/`socialWorker` (staff cNames — IDT-report-local, distinct from the resident `assignedStaff` array), `doctor` ref, `isAgreed`, and a stored `pdfUrl` (CARE-FR-46). The admin IDT form surfaces case-manager / social-worker / rehab-members inputs (with diet auto-fill) as columns. `attendingMD` now also accepts an object shape (`{ name, cName }`) in addition to the legacy string.

- **CARE-FR-41 — Status machine with role-completeness auto-submit.** `DRAFT | PENDING | SUBMITTED` (default PENDING). On create/update: an explicit DRAFT is honored; otherwise the report **auto-promotes to SUBMITTED when all three role fields (caseManager + rehabMembers + socialWorker) are filled, or `isAgreed === true`** (`IDTReport.controller.ts:144-151`). Full Zod validation is enforced only at SUBMITTED; drafts skip validation (`stripEmpty` removes empty strings to avoid cast errors).

- **CARE-FR-42 — Role-derived section attribution.** The submitting staff member's designation determines which role field they populate — Case Manager → `caseManager`; Physical/Speech Therapist, Director of Rehab, Rehabilitation Specialist → `rehabMembers`; Social Worker → `socialWorker`. **Client-supplied values for these fields are stripped server-side** (`IDTReport.controller.ts:115,129-140`). The report is thus collaboratively completed by three disciplines, and the submission notification fires once the trio is complete.

- **CARE-FR-43 — Progressive upsert.** `POST /reports/addreport` doubles as an upsert: passing `_id` updates the existing report (this is how successive disciplines fill their sections). `PUT /reports/updatereport/{id}` and `DELETE /reports/deletereport/{id}` also exist; admin uses `draftId` to decide create vs update.

- **CARE-FR-44 — Admin form behavior.** Facility name read-only (title-cased from localStorage `facilityType` — "Skilled Nursing"); report date read-only (today); **patient selection locks the auto-filled fields** (attending MD from `doctorStaff`, DoB, room from `unitNo`, admission date, contacts from family members with "No contact on file" fallbacks) to prevent mid-form patient swaps. On resident selection, `GET /rehab/appointments/by-resident/{cName}` lists upcoming therapy appointments as checkboxes → `upcomingAppointmentIds`. A debounced auto-save to PENDING is fully written but commented out (`IDTReport.tsx:307-322`; `autoSaveStatus` set but never rendered). A **"Share to chat"** action (new) forwards the report PDF into a chat conversation via `ShareWithModal` (Messaging module).

- **CARE-FR-45 — Listing & "my residents" scope.** `GET /reports/getreport` with `status=PENDING` (includes DRAFT) vs `HISTORY` (SUBMITTED) + date range — **facility-wide, unscoped**. `GET /reports/getreport/resident` is the staff "my residents" view — scope = residents whose `assignedStaff[]` contains the caller's cName (`lib/myResidents.ts:27`, now designation-agnostic — replacing the prior case-manager/social-worker/doctor allowlist), with search, doctor filter, and a `type=reports|care-conference|both` switch that co-returns the residents' care conferences; also returns `doctorNamesFilter` (all staff with designation Doctor). The two endpoints implement **two different visibility models for the same data** (§9, observation O-3). Admin History tab is client-paginated, grouped per patient (room, report date, attending MD, admission date, total reports, latest report id) with per-version access.

- **CARE-FR-46 — PDF export & stored PDF.** Two paths on staging: `GET /reports/getreport/:id/pdf` renders the full report via `buildIdtReportHtml` (2-page print HTML: facility header, patient metadata, sectioned content, medication pills, appointment references) → Puppeteer **download**; and `POST /reports/getreport/:id/pdf` (`generateIDTReportPdf` → `idtReport.pdf.service.ts`) generates the PDF, uploads it to S3, persists the URL on `IDTReport.pdfUrl`, and returns `{ pdfUrl }` (`IDTReport.controller.ts:665-688`). The IDT report record now carries a `pdfUrl` field.

- **CARE-FR-47 — IDT notifications.** Created → the named role staff (per-role message wording) + all staff holding SERVICES + REHAB access permissions; submitted → SERVICES, DASHBOARD, REHAB permission groups; a reminder API targets staff with pending sections. All gated by per-facility `notificationConfig` (`HEALTH_CARE` category: `CREATE_IDT_REPORT`, `IDT_REPORT_SUBMISSION`, `IDT_REPORT_REMINDER`). FCM with notification-history persistence (`scheduleType: 'HEALTH_CARE'`).

### 3C. Case-manager & doctor schedules

- **CARE-FR-60 — Unified day-schedule aggregator.** `GET /api/case-manager/...` (`caseManagerSchedule.controller.ts`, 1,217 ln) returns a normalized day schedule with **two persona variants selected by the caller's designation** (controller:602-607):
  - **Case manager (default):** for a given date, all `SalonAppointment`s, `TransportationRequest`s, and `RehabAppointment`s **created by the caller** (`createdByCName`) plus `CareConference`s where the caller is **host or care team** — "what I booked / must attend today". Cancelled/rejected items excluded.
  - **Doctor:** requires a `residentCName` query param; aggregates that resident's salon/transport/rehab/care-conference items **from the day forward** — a per-patient itinerary view. **Note:** as of staging, doctor-designation users see this schedule's tab **replaced by Pending Sign** ([referrals.md](./referrals.md) REF-FR-05a) in the staff app's tab bar — the doctor schedule remains reachable via the admin web, but is no longer a staff-app tab for doctors specifically.

- **CARE-FR-61 — Card normalization.** Entries are normalized into typed cards (`SALON`/`TRANSPORTATION`/`CARE_CONFERENCE`/`REHAB`) with resident profile, createdBy resolution (resident vs staff creator), chronological sorting, and a day-name header.

- **CARE-FR-62 — Printable schedule.** `GET /api/case-manager/schedule-pdf` renders the same data as printable appointment cards (`buildUpcomingAppointmentsPdf` → Puppeteer).

### 3D. Assigned care-team notification fan-out

- **CARE-FR-63 — Assigned-staff notification channel (staging refactor).** A shared utility (`notifyAssignedStaff` / `getAssignedStaffCNames`, `utils/assignedStaff.ts` — **renamed from** the former `assignedCareTeam.ts`) resolves the affected resident(s)' **`assignedStaff[]`** array (any designation, not just Case Manager/Social Worker) and notifies *only* those individuals about that resident's events, via FCM (where a push token exists) plus a `NotificationHistory` row for every matched active staff member (so web-panel-only staff still get the record). As a delivery fallback, if an assigned staffer has no Staff push token, the utility also checks the `Admin` collection for the same `cName` (covering a manager who signs in with an admin account). **The care-team-designation distinction (and the `excludeCareTeamFilter` `$nin` exclusion of Case Manager/Social Worker from permission-group blasts) no longer exists** — the module comment notes "any staff designation can appear in `Resident.assignedStaff`"; assigned-staff notifications and permission-group broadcasts are now independent channels keyed on array membership vs. page permission. This channel is wired into every domain notification service: care conference, rehab appointment, salon, massage, private-training, transport, activities, service requests (maintenance/housekeeping), IDT report, referral, and the generic scheduled-reminder pipeline. (`utils/assignedStaff.ts:55-60`; `lib/assignedStaff.ts`; call sites incl. `careConference.notification.service.ts`, `transportRequest.notification.service.ts`, `idtReport.notification.service.ts`)

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| BR-1 | Zoom provisioning is a hard precondition for virtual conferences — Zoom failure aborts creation (502); no "create now, link later" fallback. | `careConference.service.ts` |
| BR-2 | Conference cancellation is allowed only from `SCHEDULED`; it is a soft state change that also tears down calendar events. | backend §4b |
| BR-3 | A conference becomes joinable (`isEnabled`) only via the T-5-minute cron; there is no manual "start now" override observed. | `careConferenceEnable.cron.ts` |
| BR-4 | Summary review may occur from `IN_REVIEW` or re-edit after `COMPLETED`; each edit re-stamps the editor and back-fills processed recordings. | `PUT /:id/update-summary` |
| BR-5 | IDT report submission is **completion-derived, not action-derived**: it auto-promotes when the three discipline fields are populated (or `isAgreed`), and each discipline's field is attributed server-side from the submitter's designation — clients cannot claim another role's section. | `IDTReport.controller.ts:115-151` |
| BR-6 | IDT drafts skip full validation; only SUBMITTED reports must pass the Zod schema. | backend §4c |
| BR-7 | Care-team assignment is the unified `Resident.assignedStaff: string[]` (any designation); the five legacy designation→field mappings (`DESIGNATION_TO_CARE_TEAM_FIELD`) were **removed** on staging. The admin SNF form still presents role-labelled dropdowns (Case Manager/Social Worker/Doctor/Dietitian) but persists every pick into the one array via `StaffMultiSelect`. **As of 2026-08-28, the array also has an automated writer**: PCC practitioner sync matches a resident's synced physicians against the Staff roster by name and `$addToSet`s/`$pull`s `assignedStaff` entries accordingly, skipping (and alerting internally, not auto-resolving) on an ambiguous name match — see clinical-records.md §3F (CLIN-FR-28a/28b), not re-derived here. | `resident.model.ts:33,94`; client-admin-web.md §1 residents form; clinical-records.md CLIN-FR-28a/28b |
| BR-8 | The case-manager vs doctor schedule variant is decided implicitly by designation — there is no explicit mode parameter. On staging, the staff app additionally routes doctor-designation users to the Pending Sign tab instead of the My Schedule/doctor-schedule tab ([referrals.md](./referrals.md) REF-FR-05a) — a client-side navigation decision layered on top of this backend variant selection, not a change to it. | `caseManagerSchedule.controller.ts:602-607`; REF-FR-05a |
| BR-9 | Multi-tenancy: every route is facility-scoped via the `x-facility-id` header (`facilityMiddleware`); all queries filter by facilityId. | backend §0 |
| BR-10 | Conferences, like other appointment types, are auto-completed by the 15-minute overdue-appointment cron (one-by-one updates to preserve Mongoose hooks). | `appointmentCompletion.service.ts` |
| BR-11 | Assigned staff (any designation in `resident.assignedStaff[]`) receive resident-specific events via the `notifyAssignedStaff` channel; this is now **independent** of the permission-group fan-out (the former `excludeCareTeamFilter` `$nin` exclusion was removed with the care-team-designation distinction). | `utils/assignedStaff.ts` (CARE-FR-63) |
| BR-12 | The admin-web Care Conference Calendar (CARE-FR-26a) is a second **view**, not a second **source of truth** — it reads/writes the same `CareConference` records via the same `/api/care-conference` endpoints as the Schedule Care Conference screen (CARE-FR-26), sharing that screen's form components and schedule-conflict detection (CARE-FR-25a) rather than re-implementing them. Its own list query, however, excludes `COMPLETED` conferences — a client-side choice that makes the calendar and the Schedule screen's History tab disagree on whether a completed conference is visible. | `client-admin-web.md` §2.11/§4.4 |

---

## 5. Notifications & real-time behavior

| Trigger | Channel | Recipients | Gate |
|---|---|---|---|
| Conference scheduled | FCM + **SMS** + history | Residents, family, host + care team | Per-facility `notificationConfig` immediate-enable flags; SMS reminders via `careConferenceSmsReminder.cron.ts` |
| Conference starting (T-5 min) | FCM + history | Residents, host, care team | Enabler cron `* * * * *`, env `ENABLE_CARE_CONFERENCE_ENABLE_CRON` |
| Conference completed / cancelled | FCM + history | Residents, family, named staff | `notificationConfig` |
| IDT created | FCM + history | Named role staff (per-role wording) + SERVICES + REHAB permission groups | `notificationConfig` HEALTH_CARE / `CREATE_IDT_REPORT` |
| IDT submitted | FCM + history | SERVICES, DASHBOARD, REHAB permission groups | `IDT_REPORT_SUBMISSION` |
| IDT reminder (explicit API) | FCM + history | Staff with pending sections | `IDT_REPORT_REMINDER` |
| Conference auto-IN_PROGRESS / appointment auto-COMPLETE | silent cron | — | env flags (`ENABLE_APPOINTMENT_COMPLETION_CRON`) |
| Any resident event (booking/clinical/transport/activity/service-request) | FCM + history | Every staff cName in the resident's `assignedStaff[]` (any designation) | `notifyAssignedStaff` (CARE-FR-63); now independent of permission-group blasts (the former `excludeCareTeamFilter` exclusion was removed) |

Notification fan-out is FCM-only with history persistence; no socket events exist for this module (sockets are used by chat and TV pairing elsewhere). The conference notification service is substantial (`careConference.notification.service.ts`, 505 ln) and per-facility configurable. Referral/ACD e-signature pending-queue items pushed via this channel are what populate the staff-app Pending Sign tab ([referrals.md](./referrals.md) REF-FR-05a) on next fetch — there is no dedicated real-time channel for the Pending Sign queue itself. The Care Conference Calendar (CARE-FR-26a) introduces no new notification triggers of its own — edits/deletes made there fire the same notification paths as the Schedule Care Conference screen because they hit the same endpoints.

**Resident-side caveat:** the SN resident app's notification center renders the FCM-backed feed but implements no mark-as-read and no tap-through navigation (client-resident-app-sn.md §3.10) — conference pushes land as inert feed entries on the resident device.

---

## 6. Integrations

| Integration | Used by | Behavior |
|---|---|---|
| **Zoom** (per-staff OAuth) | Care conferences | Meeting provisioned pre-create for Virtual conferences (failure = 502 abort); PATCH on reschedule; host `startUrl` vs participant `joinUrl`; transcript fetch after `recording.completed`. Host staff must have a linked Zoom account (`isZoomLinked` flag on staff profile; OAuth link via `GET /auth/zoom/url?cognitoUser={cName}` on the `authApi` base) |
| **Google Calendar** (per-staff OAuth) | Care conferences | Events created for host + each care-team member (`careTeamGoogleEventIds`); Zoom link in description; `calendarSyncStatus` aggregate; events removed on cancel. Linking via `GET /auth/google/url` |
| **S3 (presigned)** | Conferences | Conference audio: presigned PUT via `POST /audio-presign`, presigned 1-hour GETs via `/recording-urls`. *(Referral signature/PDF S3 usage → [referrals.md](./referrals.md) §6.)* |
| **Transcribe-processor Lambda** (out-of-repo) | In-person conferences | Asynchronously populates `recordings[]` with per-part transcript + AI summary; UI is read-only on these |
| **Puppeteer HTML→PDF** | IDT, case-manager schedule | `buildIdtReportHtml` (2-page), `buildUpcomingAppointmentsPdf`. *(Referral PDF templates → [referrals.md](./referrals.md) §6.)* Neither Care Conference view (Schedule or Calendar) offers a PDF export. |
| **FCM** | All notifications | Token harvested implicitly from resident-app request headers; per-facility config gating |
| **UnifiedSchedule sync** | Conferences | Post-save/update/delete hooks mirror conferences into the `UnifiedSchedule` collection per resident (resident master calendar). **`TRANSPORTATION` is now in `BLOCKING_SCHEDULE_TYPES`** for care conference conflict detection (production 2026-06-24) — active transport rides block new care conference creation. Other booking types (salon/massage/PT) still do not block conference creation. Conversely, transport booking now checks for `CARE_CONFERENCE` conflicts. |

*(Referral-specific integrations — in-house e-signature, AWS SES agency email, Documo fax, and the admin `authApi` split-backend quirk — are documented in [referrals.md](./referrals.md) §6. PCC practitioner sync's integration mechanics — a new PCC event pair, a second OAuth/mTLS module, and a new envelope-encryption key — are documented in clinical-records.md §3F/§6, not here.)*

---

## 7. Permissions & access control

**Assigned-staff membership is the module's spine (staging refactor).** The five legacy care-team designation→field mappings on the Resident document were **removed** and consolidated into a single indexed `Resident.assignedStaff: string[]` of Staff cNames (any designation; PLAT-FR-54). Resident-↔-staff scoping now keys on **array membership** rather than on a specific designation field. **As of 2026-08-28, that array can be populated by the PCC integration itself, not only by a manual admin pick — see BR-7 and clinical-records.md §3F.** This backbone is shared with the Referrals module and drives:
- the doctor referral queue — scoped by the referral's explicit `assignedPhysician` Staff ref ([referrals.md](./referrals.md) REF-FR-07), consumed by both admin web and the staff-app Pending Sign tab (REF-FR-05a),
- IDT role attribution (the submitter's designation still selects which IDT-report section field they fill — these are IDT-report-local fields, unaffected by the resident refactor) and the "my residents" report/staff view (`lib/myResidents.ts` `getMyResidentIds` now resolves residents whose `assignedStaff[]` contains the caller's cName — "all staff, regardless of designation, scoped to their assigned residents" — rather than the prior Case Manager/Social Worker/Doctor designation allowlist),
- the case-manager vs doctor schedule variant (still keyed on the caller's designation),
- resident↔staff chat eligibility (residents may only initiate chat with staff in their own `assignedStaff[]` — Messaging §4.1),
- assigned-staff notification targeting (`notifyAssignedStaff`, CARE-FR-63).

**Route-level enforcement as-built:**

| Surface / route group | Gate |
|---|---|
| Care-conference management routes | `requireAnyRole STAFF\|ADMIN`; non-admin staff list scoped to hosted conferences |
| `GET /care-conference/my-conferences` (+history) | Any authenticated caller, participant-scoped `$or` filter |
| IDT `/api/reports/*` | `authMiddleware` only — **no STAFF/ADMIN role gate** (single mount on staging; double-mount resolved — §9, O-1) |
| Case-manager schedule | Authenticated; behavior keyed off designation |
| Admin web frontend | IDT Report and **both** Care Conference views (Schedule Care Conference and Care Conference Calendar, CARE-FR-26a) have **no `usePageAccess` gate** — they rely on nav-level filtering + backend enforcement only; direct state navigation renders them regardless of page permission (client-admin-web.md §2.11/§4.4). *(The Referrals screen has the same gap — [referrals.md](./referrals.md) §7/§9.)* |

*(Referral and agency route gates — both `authMiddleware`-only with no role check — are documented in [referrals.md](./referrals.md) §7.)*

Multi-tenant isolation (facilityId filter from `x-facility-id`) **is** consistently enforced across the module even where role gates are missing.

---

## 8. Product-split notes

- The entire module is **SNF-flavored**: IDT reports are an SNF/CMS construct; case-manager/doctor schedules and the rehab-linked IDT sections all serve skilled-nursing workflows (backend §8.3). Exposure in the admin web is governed by the facility-pages API (Filter 1), not code conditionals — `idt-report` ships in the SNF nav set alongside the SNF rehab suite (`AppContent.tsx:109-222`), and the new `schedule-care-conference` nav group (both Schedule + Calendar sub-items, CARE-FR-26a) is likewise gated only by Filter 1/Filter 2, with **no facility-type branching in the calendar component's own code** — consistent with the rest of this module. The staff-app Pending Sign tab ([referrals.md](./referrals.md) REF-FR-05a) is likewise only reachable by doctor-designation staff, who exist predominantly in SNF facilities under the current designation model.
- Code-level SNF conditionals touching this module are minimal: IDT's facility-name field is title-cased from the two-value `facilityType` enum (`IDTReport.tsx:159, 331, 402, 530-532`); the SNF residents table previously surfaced the Case Manager/Social Worker/Doctor care-team columns this module depends on — as of the 2026-08-08→2026-08-27 admin-web Residents rework, that table's SNF/AL column split was **retired** in favor of one unified column set (client-admin-web.md §3), so this module's care-team data (`assignedStaff[]`) is now surfaced the same way for every facility type at the list-view level; the underlying `assignedStaff[]` array and its role semantics (BR-7) are unaffected.
- The SN **resident app has no care-coordination surface** for IDT or discharge planning ("Explicitly absent", client-resident-app-sn.md §3) — care conferences are the one exception, now live on staging (CARE-FR-26). The case manager exists for residents only as one of the four fixed chat care-team roles (Case Manager, Social Worker, Doctor, Dietitian — `CareTeamRole`, `src/services/Chat/type.ts:38-42`). Backend resident-facing endpoints (`/my-conferences`, `shareWithResident` summaries) are built — a future-product split decision.
- One backend serves both products; care level is resident-granular (`CARE_TYPE_VALUES`, four values) while the admin facilityType enum has two values — Memory Care / Independent Living exist only as resident care types (client-admin-web.md §3).

---

## 9. Observations & candidate gaps

Evidence-backed defects, mismatches, and half-built areas a forward PRD must resolve. *(Referral- and agency-specific observations moved to [referrals.md](./referrals.md) §9.)*

- **O-1 — IDT missing role gate (double-mount resolved).** The `/api/reports` duplicate mount was **removed** on staging — now a single mount (`app.ts:209`, T-3 resolved). The router still carries bare `authMiddleware` only — any authenticated facility user (including a resident identity) can create, update, or delete IDT reports (backend §4c, §10.4).
- **O-2 — Unauthenticated `/api/care/:id` routes (adjacent module, shared `/care` engine).** `GET /api/care/:id` and `DELETE /api/care/:id` are deliberately commented "Open — no auth required" (`care.routes.ts:75,87`) — an unauthenticated caller with a facility header can read or **hard-delete** any care record. Owned by the Wellness module but listed here because the admin outside-agency and AL therapy flows ride this engine (backend §10.1).
- **O-3 — Two IDT visibility models.** `GET /getreport` is facility-wide while `GET /getreport/resident` scopes by the caller's designation-mapped residents — same data, inconsistent exposure (backend §10.17).
- **O-4 — Resident-facing conference surface now built (staging).** `GET /my-conferences` (+`/history`) are now consumed by the skilled-nursing resident app's Care Conference module (summaries, detail, history-detail, join). *Residual: verify `shareWithResident`-gated summary rendering and whether conference "starting" notifications now deep-link into the new module (notification tap-through was previously absent — client-resident-app-sn.md §3.10).*
- **O-5 — IDT auto-save dead code.** A debounced auto-save to PENDING is fully written but commented out (`IDTReport.tsx:307-322`); `autoSaveStatus` is set but never rendered — long IDT forms risk data loss on navigation.
- **O-6 — Client-side pagination over bulk fetches.** The Schedule Care Conference screen and IDT history paginate client-side over `limit: 100`-style fetches — a scale ceiling for high-volume facilities (client-admin-web.md §4.4). The Care Conference Calendar (CARE-FR-26a) instead fetches a `limit: 1000` window scoped to the visible date range, which sidesteps this specific ceiling for the calendar view but introduces its own scale question for a facility with a very dense single day/week/month.
- **O-7 — No frontend permission gates** on IDT Report or either Care Conference view (Schedule or Calendar) in the admin web (`usePageAccess` absent); combined with O-1's missing backend role gate, IDT access control is effectively "any authenticated user in the facility" (client-admin-web.md §2.11/§4.4). *(The Referrals screen shares this gap — [referrals.md](./referrals.md) §9 O-7.)*
- **O-8 — Zoom >40-minute limitation** is surfaced only as a frontend warning on the duration picker; no backend constraint or paid-account detection (client-admin-web.md §2.11).
- **O-9 — No manual conference start.** `IN_PROGRESS` is reachable only via the per-minute cron (T-5); if the cron is disabled (`ENABLE_CARE_CONFERENCE_ENABLE_CRON`), conferences never become joinable (backend §4b).
- **O-10 — Staff-app Reports tab is dead code (2026-07-12).** `SkilledNursingBottomTabNavigator`'s `<Tab.Screen name={SkilledNursingTabScreens.REPORTS} .../>` registration is entirely commented out (`src/navigation/skilledNursingAppStack/index.tsx:257-283`) on the staff app, while the import, route constant, and `ReportsScreen` component file all still exist. The tab is unreachable for **any** user today, even though care-conference/IDT/rehab reports are otherwise live features in this module and Therapy & Rehab. This is unrelated to the Pending Sign tab ([referrals.md](./referrals.md) REF-FR-05a), which replaces **My Schedule**, not Reports, in the doctor tab set. Needs a product decision: intentional interim state (Reports superseded by something else and simply not yet re-scoped) or an accidental regression. *(`docs/reviews/2026-07-12/review-senior_living_staffapp.md`, Design Gap G11.)*
- **O-11 — New: the Care Conference Calendar's own list query silently excludes `COMPLETED` conferences (2026-08-28).** See CARE-FR-26a and BR-12 — the calendar's status filter (`SCHEDULED,IN_PROGRESS,IN_REVIEW,CANCELLED`) and colour legend have no `COMPLETED` entry, while the sibling Schedule Care Conference screen's History tab does show completed conferences. The two admin-web views of the same `CareConference` collection therefore disagree on what "all conferences in this date range" means. Needs a product decision: add `COMPLETED` to the calendar's fetch + legend, or confirm the calendar is intentionally an "active conferences only" view distinct from the History-bearing list screen.
