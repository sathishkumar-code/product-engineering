# Module: Therapy & Rehab

> Applies to: **Both products — with an explicit product fork** (Assisted Living fixed-types path vs Skilled Nursing dynamic-catalog path)
> FR prefix: `REHAB`
> Sources: `_codebase-analysis/backend-clinical-care.md` (§0, §5, §6, §8, §10), `_codebase-analysis/client-admin-web.md` (§1.1, §2.6, §2.9, §2.10, §2.12, §4.3–4.5), `_codebase-analysis/client-resident-app-sn.md` (Rehab + orphaned health screens), `_codebase-analysis/client-resident-app-sl.md` (§3.11 mock health screens), `_codebase-analysis/backend-wellness-dining-ops.md` (§0.4–0.6 UnifiedSchedule / slot engine / calendar). Code is the source of truth; this document describes **as-built** behavior.
> 2026-07-12 delta: `docs/reviews/2026-07-12/review-senior_living_reactnative.md` (AL-path reschedule for Physical Therapy / Cognitive Evaluation / Outside Agency, and a corrected/shared cancel endpoint — REHAB-FR-11a).

---

## 1. Purpose & scope (the fork, up front)

Therapy & Rehab covers all therapy scheduling, therapist resourcing, therapy cataloging, and resident↔rehab-department communication. It is the **single clearest product-split point in the platform** (`constants/rehab.ts:38-101`, `RehabAppointment.model.ts:19-26`): one backend serves two divergent operating models, and the admin web app exposes two disjoint navigation sets switched per facility by the facility-pages API (Filter 1) — *not* by frontend branching (`AppContent.tsx:104-107`).

**(a) Assisted Living path — "fixed therapy types" (Senior Living product).**
Therapy is a small, fixed menu of service types: `PHYSICAL_THERAPY`, `COGNITIVE_EVALUATION`, `REHAB_EVALUATION`, `OUTSIDE_AGENCY`. Appointments ride the generic **`/api/care` engine** (`Care` model). Slot durations come from per-facility `Config.rehab.*` (legacy key spellings preserved in code). Admin staff schedule PT and cognitive sessions, book outside-agency visits (agency captured as **free-text** `agencyName`), and monitor everything through a **Therapy Evaluations** aggregate calendar. There is no therapist-resource model — availability is computed per **resident** per date, and PT/cognitive entries sync to the facility's *Physical Therapist's* Google Calendar.

**(b) Skilled Nursing path — "dynamic catalog + therapist resourcing".**
Therapy is a facility-managed **RehabTherapy catalog** (name / uppercase code / duration / image). Appointments live in the dedicated **`/api/rehab` suite** (`RehabAppointment` model, `therapyType: "OTHER"` + `therapyId`), are booked against a specific **rehab staff member** ("Rehabilitation Specialist" designation), and the slot engine subtracts meals, the staff member's existing bookings, their cached Google Calendar busy windows, and the resident's cross-venue conflicts. The SN side adds a rehab calendar (monthly/daily + print/PDF), rehab team management with real-time availability, weekly availability self-service for therapists, and an encrypted resident→rehab-team message queue. The SN resident app exposes a read-only rehab schedule/history/report plus the messaging form.

A facility sees exactly one of these navigation sets; the backend keeps both code paths alive in the same Express app, and the `RehabAppointment` model itself can carry either AL fixed types or SN `OTHER`+catalog entries (see §8 and §9 for the resulting dual-backend ambiguity).

**Out of scope for this module:** Massage Therapy and Private Training (wellness cluster — separate module; note Private Training is *permission-gated by the Rehab key*, §7), IDT Reports (clinical reporting module; consumes rehab appointments), Referrals/Agency directory (clinical module; the Agency entity is referenced here only for the free-text inconsistency in §8), Care Conferences.

---

## 2. Personas & surfaces

| Persona | Surface | Therapy & Rehab capabilities |
|---|---|---|
| **Admin** (Cognito `ADMIN`) | Admin web | AL: schedule PT/cognitive/outside-agency sessions, Therapy Evaluations view. SN: full rehab appointment CRUD, therapy catalog CRUD, rehab team CRUD, edit any therapist's availability, triage rehab messages, rehab calendar + PDF. |
| **Staff** (Cognito `STAFF`, page permissions + designation) | Admin web | Same screens as Admin, filtered by Filter 1 (facility pages) + Filter 2 (per-staff `accessPermissions`); booking additionally gated by designation via booking policy. "My Availability" auto-loads self and is self-edit-only. |
| **Rehabilitation Specialist / Director of Rehab / Rehab Therapists** (designations) | Admin web | Assignable to rehab appointments (`REHAB_STAFF_DESIGNATIONS`); manage own weekly availability; appear in Rehab Team and in the resident app's "Rehab Team" tab. |
| **Resident** (Cognito `RESIDENT`) | SN resident app | Read-only rehab schedule, history, per-appointment report; send messages to the rehab team. (AL self-booking flows exist as orphaned screens — §9.) |
| | SL resident app | **As of staging: API-backed AL self-booking, including reschedule.** The Health hub merchandises Physical Therapy, Cognitive Evaluation, and Outside Agency Services as reachable tiles, each with live list / details / request / slot-selection screens against `/api/care/*` (`src/services/health/index.tsx`). The prior "mock end-to-end" state is gone — the SL app is now the live AL resident self-booking surface, and as of the most recent delta (`master@3af3c3e`) all three types also support **reschedule** in addition to book/cancel (REHAB-FR-11a; §9). |
| **Family member** (`FAMILY_MEMBER`) | Resident apps | Acts as the linked resident (auth-time username normalization); may book via booking policy where `isFamilyMemberAllowed` is set; receives rehab appointment notifications. |
| **TV app** | — | No therapy/rehab surface. |

Backend route prefixes: `/api/care` (AL path), `/api/rehab` (SN path: `rehab/therapy`, `rehab/appointments`, `rehab/available-slots`, `rehab/rehab-message`), `/staff` (team + availability).

---

## 3. Functional requirements (as-built)

### 3A. Assisted Living path — fixed therapy types via `/care`

- **REHAB-FR-01 — Fixed therapy type menu.** The system SHALL support exactly four AL therapy types on care appointments: `PHYSICAL_THERAPY`, `COGNITIVE_EVALUATION`, `REHAB_EVALUATION`, `OUTSIDE_AGENCY` (`Care.model.ts:12`, `constants/rehab.ts:40-101`). Slot durations per type come from facility `Config.rehab.*` (legacy key spellings preserved). `OUTSIDE_AGENCY` additionally requires `agencyName` (free text) and `serviceType` (`PHYSICAL_THERAPY | COGNITIVE_EVALUATION`); it reuses the PT duration.

- **REHAB-FR-02 — Care appointment entity.** A care appointment SHALL carry: resident `cName`, `date` + `startTime`/`endTime` (24h `HH:mm` strings), `location` (default `"Therapy Room"`), `reason`, post-completion `summary`, staff `notes`, booking provenance (`createdByType`/`createdByCName`), and Google Calendar sync fields (`Care.model.ts`).

- **REHAB-FR-03 — Care status machine.** Status SHALL follow `REQUESTED → CONFIRMED → COMPLETED | CANCELLED` (default `REQUESTED`). "Upcoming" lists = REQUESTED/CONFIRMED; "history" = COMPLETED/CANCELLED. Overdue CONFIRMED entries are auto-completed by cron (REHAB-FR-31).

- **REHAB-FR-04 — Policy-gated booking.** `POST /care` SHALL run `resolveBookingContext('Care')`: residents always self-book; family members book for their linked resident iff `Config.bookingPermission.Care.isFamilyMemberAllowed`; staff must supply a resident identifier and have a designation in the policy allowlist (or an allowed designation group, e.g. `rehab`, `supervision`); admins bypass the designation check but the per-facility policy must exist. Resident-supplied `residentCName` is ignored for RESIDENT callers (spoof guard).

- **REHAB-FR-05 — Per-resident slot availability.** The system SHALL expose care available-slots per resident/date (`GET /care/available-slots?date&residentCName`, single-day and multi-day via `noOfDays`) and SHALL re-validate the exact requested slot at booking time for PT / cognitive / outside-agency types (`assertCareSlotAvailable`, `careAvailability.service.ts`). Conflicts include the resident's blocking UnifiedSchedule rows (REHAB-FR-29).

- **REHAB-FR-06 — Admin PT & Cognitive session screens.** The admin web SHALL provide structurally identical "Physical Therapy" and "Cognitive Sessions" pages (differing only in `type`) with: a weekly 7-day calendar grid + daily list from `GET /care/staff/upcoming?type&startDate&endDate`, prev/next-week + "This Week" navigation, and a creation modal (resident → date → slots from the care slots API → location/reason/notes → `POST /care`). These pages are **view + create only** — no edit, delete, or reschedule UI (client-admin-web §2.6).

- **REHAB-FR-07 — Outside Agency Services screen.** The admin web SHALL schedule outside-agency appointments with required resident, `serviceType` (PT | cognitive), **free-typed** `agencyName`, location, date (past dates blocked), start/end time, optional notes → `POST /care { type: "OUTSIDE_AGENCY" }`. Weekly grid + daily card views, stat cards (total, today), and "Outside Agency" warning badges on cards. The agency *directory* is managed elsewhere (Referrals screen) and is **not referenced** by this form (§8).

- **REHAB-FR-08 — Therapy Evaluations aggregate view.** The admin web SHALL provide an AL-side aggregator across `PHYSICAL_THERAPY`, `COGNITIVE_EVALUATION`, `REHAB_EVALUATION`, `OUTSIDE_AGENCY` **plus** `PRIVATE_SESSION` (private training), with monthly + daily calendar views, per-type session labels (e.g. "Outside Agency - Physical Therapy"), and a creation modal posting to `/care`. View + create only; listed sessions cannot be edited or deleted from this page (client-admin-web §2.6, flagged §4.3).

- **REHAB-FR-09 — Google Calendar sync (AL).** PT and COGNITIVE_EVALUATION care entries SHALL sync to the facility **Physical Therapist's** Google Calendar (`CARE_TYPES_SYNCED_TO_CALENDAR`, `careCalendar.ts`); create/update/cancel/delete propagate; per-record `calendarSyncStatus` ∈ PENDING/SYNCED/FAILED. REHAB_EVALUATION and OUTSIDE_AGENCY are not calendar-synced.

- **REHAB-FR-10 — Staff panel reads.** `GET /care/staff/upcoming` and `GET /care/staff/history` SHALL be facility-wide and restricted to ADMIN|STAFF roles.

- **REHAB-FR-11 — Resident self-service flows.** **Status diverges by app as of staging.** In the **SL resident app** this flow is now **live and reachable**: the Health hub merchandises Physical Therapy, Cognitive Evaluation, and Outside Agency tiles, each wired to `/api/care/*` — upcoming/history lists (`GET /api/care/upcoming`, `/api/care/history`), per-date slot fetch (`GET /api/care/available-slots`), booking (`POST /api/care`), detail (`GET /api/care/:id`), reschedule (`PUT /api/care/:id`, all three types — REHAB-FR-11a), and cancel (`DELETE /api/care/cancel/:id`, shared across all three types — REHAB-FR-11a) (`src/services/health/index.tsx`, `src/screens/App/HealthScreen/index.tsx`). The same screen family in the **SN app** remains **orphaned** (registered in the navigator with no inbound navigation — §9.1). Net: AL resident self-booking (including reschedule) is now live on the SL app; the SN app continues to drive AL therapy admin/staff-side only.

- **REHAB-FR-11a — Reschedule and unified cancel for PT / Cognitive Evaluation / Outside Agency (SL app, `master@3af3c3e`, 2026-07-12 delta).**
  - **Reschedule**, previously absent, is now supported for all three AL care flows: `UpdatePhysicalTherapySlot` and `UpdateCognitiveEvaluationSlot` (both `PUT /api/care/:id`) were added to `src/services/health/index.tsx`, and `RequestPhysicalTherapyEvaluation`/`RequestPhysicalTherapySlot`, `RequestCognitiveEvaluation`/`RequestCognitiveEvaluationSlot` now accept `route.params.{isReschedule, appointmentData}` and pre-fill the booking form, calling Update instead of Book when in reschedule mode. `UpdateOutsideAgencyService` (also `PUT /api/care/:id`) does the equivalent for `RequestOutsideAgencyServiceScreen`.
  - **Cancel** for all three types is served by **one shared function**, `CancelOutsideAgencyService` (`DELETE /api/care/cancel/:id`), invoked from `PhysicalTherapyListScreen`, `CognitiveEvaluationListScreen`, and `OutsideAgencyServiceListScreen`'s `onCancelAppointment` handlers. **The function's name implies Outside-Agency-only scope but it is in fact the shared cancel endpoint for all three care types** — logged as Technical Debt TD-17 in the app's own architecture doc; a future PT/CE-only bug fix could be missed if a reviewer assumes the function is Outside-Agency-scoped. Recommend renaming to something generic (e.g. `CancelCareAppointment`) under its own small ticket.
  - `AppointmentListElements` (shared component, +254 lines) gained `onCancel`/`onReschedule` action-button rendering and a `showActions` shimmer-loading variant, used by all three Health list screens.
  - Agency-name display was also improved in the same delta: `ServiceDetailModal`'s title now wraps to 2 lines, and `UpcomingAppointmentsScreen` (dashboard card + detail modal) prefers `care.agencyName` over `care.name`/`care.careType` when present (`UnifiedService` type gained an optional `agencyName` field).
  - **Corrects a prior documentation error in this module**: REHAB-FR-11 (above) previously and incorrectly stated cancel as `POST /api/care/cancel/:id`; the verified endpoint is `DELETE /api/care/cancel/:id`, and it is explicitly the shared cancel path for PT, Cognitive Evaluation, *and* Outside Agency, not an outside-agency-specific one.
  - **Not independently verified this pass**: whether the backend's `DELETE /api/care/cancel/:id` requires (or infers) a `careType` discriminator server-side — the client sends only `appointmentId`. Flag for the backend architecture doc's next refresh.

**AL path — endpoint reference:**

| Operation | Method & path | Caller |
|---|---|---|
| Book therapy session | `POST /care { residentCName, type, date, startTime, endTime, location?, reason?, notes?, serviceType?, agencyName? }` | Admin web (PT / Cognitive / Therapy Evals / Outside Agency modals); SL resident app |
| Reschedule therapy session | `PUT /care/:id` (`UpdatePhysicalTherapySlot` / `UpdateCognitiveEvaluationSlot` / `UpdateOutsideAgencyService`) | SL resident app only (REHAB-FR-11a) — no admin-web reschedule UI exists (REHAB-FR-06/07 remain view+create only) |
| Cancel therapy session | `DELETE /care/cancel/:id` — shared across PT/Cognitive/Outside Agency via one client function (`CancelOutsideAgencyService`, TD-17 naming) | SL resident app (REHAB-FR-11a) |
| Staff weekly/daily lists | `GET /care/staff/upcoming?type&startDate&endDate`, `GET /care/staff/history` | Admin web (ADMIN\|STAFF) |
| Resident lists (latent) | `GET /care/upcoming?type=…`, `GET /care/history?type=…`, `GET /care/my-*` equivalents | Orphaned SN-app screens |
| Available slots | `GET /care/available-slots?date&residentCName` (+ `noOfDays` multi-day) | Admin modals; SL resident app; orphaned SN resident slot picker |
| Read / delete by id | `GET /care/:id`, `DELETE /care/:id` — **unauthenticated** ⚠ | (see §7, §9.4) |

**AL path — `/care` create payload (key data shape):**

```typescript
{ residentCName, type: 'PHYSICAL_THERAPY'|'COGNITIVE_EVALUATION'|'REHAB_EVALUATION'|'OUTSIDE_AGENCY',
  date: 'YYYY-MM-DD', startTime, endTime,          // "HH:mm" 24h
  location?,                                        // default "Therapy Room"
  reason?, notes?,
  serviceType?, agencyName? }                       // OUTSIDE_AGENCY only; agencyName is free text
```

### 3B. Skilled Nursing path — dynamic catalog via `/rehab`

**Catalog**

- **REHAB-FR-12 — RehabTherapy catalog.** The system SHALL maintain a facility-scoped therapy catalog: `name`, **code** (uppercase-normalized, 2–3 letters in UI e.g. PT/OT/ST, unique per facility via a *partial* index that excludes soft-deleted rows so codes can be reused), `duration` (minutes — drives slot length), `description`, `image` (S3 key, signed on read), creator `staffCName`. Soft delete (`isDeleted`/`deletedAt`). CRUD restricted to STAFF|ADMIN; list/get open to any authenticated role (`RehabTherapy.model.ts`, `rehabTherapy.service.ts`).

- **REHAB-FR-13 — Catalog admin UI.** Admin web "Rehab Service" page (permission key `"rehab-list"`): responsive card grid (image, name, code badge, duration, 80-char-truncated description), create/edit/delete via `rehab/therapy` endpoints (FormData). Image is **required on create, optional on edit**; uses the shared gallery picker.

**Appointments**

- **REHAB-FR-14 — Rehab appointment entity & typing pivot.** A rehab appointment SHALL carry resident `cName`, `therapyType`, `therapyId`, optional `agencyName`/`serviceType`, `date` (ISO), `startTime`/`endTime` (`HH:mm`), `location`, assigned `staffCName`, `notes`, `status`, booking provenance. When `therapyType` is `OTHER` (the SN path — and the admin client **always** sends `"OTHER"`, with the real selection in `therapyId`) or omitted, `therapyId` is mandatory (`isRehabTherapyIdRequired`) and duration resolves from the catalog row. The same model also accepts the four AL fixed types "when the appointment belongs to an ASSISTED_LIVING facility" with Config-driven durations (`RehabAppointment.model.ts:19-26`, `rehab.ts:40-101`) — see §9.3.

- **REHAB-FR-15 — Rehab status machine.** `SCHEDULED → COMPLETED | CANCELLED` (default SCHEDULED; legacy records missing status are treated as SCHEDULED; `cancelledAt` timestamped). Upcoming endpoints = SCHEDULED; history = COMPLETED/CANCELLED; cron auto-completes overdue SCHEDULED entries (REHAB-FR-31).

- **REHAB-FR-16 — Staff assignment rule.** The assigned `staffCName` MUST belong to a staff member whose designation ∈ `REHAB_STAFF_DESIGNATIONS` (`assertRehabStaffAssignable`, `rehabAppointment.service.ts:256`).

- **REHAB-FR-17 — Slot engine (therapist-resource availability).** `GET rehab/available-slots?date&staffCName&cName&therapyId` SHALL compute available slots for a **staff member** on a day as: base grid (from Config, slot length = therapy duration) minus (a) facility meal windows (`Config.meals`), (b) the staff member's existing bookings, (c) the staff member's **cached Google Calendar busy slots**, and (d) the **resident's** blocking UnifiedSchedule entries (cross-venue: SALON/MASSAGE/PT/CARE/REHAB — REHAB-FR-29). Booking SHALL re-validate the exact requested slot server-side and reject with CONFLICT otherwise (`rehabAvailability.service.ts`).

- **REHAB-FR-18 — Appointment admin UI.** Admin web "Rehab Appointments": full CRUD with Upcoming/History tabs (`GET rehab/appointments` / `…/history`; scope/page/limit/status/search/staffCName params), 400 ms debounced search (resident name/email/unit, location), status filter (ALL/SCHEDULED/COMPLETED/CANCELLED), server pagination (10/25/50/100). The booking form requires resident, therapy (catalog), rehab staff (`GET /staff?designationGroup=rehab`), date, and a start/end pair chosen from available slots — slots fetch only once resident+therapy+staff+date are all selected; picking a start auto-fills its paired end (and vice versa); editing restores the original pair when the original start is re-selected. Location is required free text. `POST/PUT/DELETE rehab/appointments[/{id}]`.

- **REHAB-FR-19 — Per-resident upcoming feed.** `GET /rehab/appointments/by-resident/{cName}` SHALL list a resident's upcoming rehab appointments (consumed by the IDT Report form as checkable `upcomingAppointmentIds`).

**Calendar**

- **REHAB-FR-20 — Rehab calendar.** Admin web SHALL render a monthly view (7-column grid; per-day up to 3 appointment badges + "+N more"; badge = 12-hour time, truncated resident name, therapy code; today ring-highlighted) that switches to a daily stacked-card view on day click (therapy label, resident + unit#, location, booked-by staff, time; prev/next-day + "Today"). Per-therapy stat cards on top (PT, ST, OT first, then alphabetical). Therapy color theming is hash-based on code (`hash(code) % 5` over 5 color pairs) for stable colors. Date keys deliberately extracted in **UTC** (date-only policy) to prevent day-shift in negative UTC offsets.

- **REHAB-FR-21 — Calendar print/PDF.** The calendar SHALL export a print/PDF document via a shared HTML template: legend (codes + names) + appointment table (resident, room, date, time, therapy, location, booked by), facility display name from `facilityType`, facility logo.

**Team & availability**

- **REHAB-FR-22 — Rehab team management.** Admin web "Rehab Team": staff CRUD scoped to designation **"Rehabilitation Specialist"** (`REHAB_STAFF_DESIGNATION`), riding the shared `/staff` endpoints with `filterDesignation: "rehab"`. Cards show photo, name, designation, phone, email, and a **real-time "Available Now" badge** recomputed every 60 s (`active` AND today `isAvailable` AND now within a `{start,end}` slot). Required fields: name (≥2 chars), rehab-filtered designation, E.164 phone (≥7 digits), speciality (mapped from therapy codes); optional validated email, photo, active toggle.

- **REHAB-FR-23 — Weekly availability self-service.** "My Availability": a Monday–Sunday editor (per day: "Closed" toggle + start/end `HH:mm` inputs). ADMIN selects any rehab staffer from a dropdown; STAFF auto-loads and can only edit self. Edits **auto-save on a 450 ms debounce** (no Save button) → `PUT staff/availabilty/{staffCName}` *(endpoint path typo is real)* with `{ weekly: { MONDAY..SUNDAY: { isAvailable, slots: [{start,end}] } } }`; per-save toast. This availability feeds the Available-Now badge and (with bookings/busy/meals) the slot engine.

**Messages**

- **REHAB-FR-24 — Rehab messages (resident → rehab team).** Residents (and only residents — `requireAnyRole RESIDENT`) SHALL create messages with `topic` + `message` via `POST rehab/rehab-message`. Both fields SHALL be **KMS envelope-encrypted at rest** (AES-256-GCM, per-field wrapped data key; encryption context binds facilityId+cName+model+field to prevent ciphertext swapping — `rehabMessage.service.ts:55-66`). Plaintext is never persisted; server decrypts on every read.

- **REHAB-FR-25 — Message triage workflow.** Status machine `NEW → IN_PROGRESS → CLOSED`. Residents read their own (`GET rehab/rehab-message/my-message`); staff/admin read all for the facility (`GET rehab/rehab-message`) and update via `PATCH rehab/rehab-message/{id} { status, replyBy? }` — `replyBy` (cName) + `replyByRole` (STAFF|ADMIN) record who actioned it. **No threaded replies** — it is a status-tracked request queue with responder identity surfaced. Admin UI: table (Resident/Room, Topic, Message, Reply By, Status) + detail modal with status dropdown.

**Resident app (SN)**

- **REHAB-FR-26 — Resident rehab schedule & team.** The SN resident app "Rehab" screen SHALL show two tabs: **My Rehab Schedule** (collapsible date calendar today→+1 month; paginated `GET /api/rehab/appointments/my-appointments?date=`; cards: therapy name + code, start time + duration, location) and **Rehab Team** (`GET /api/staff?designationGroup=rehab`: photo, name, designation). Scheduling is **read-only for residents** — no book/cancel UI; rehab booking is staff-driven.

- **REHAB-FR-27 — Resident rehab history & report.** "Rehab History" SHALL page `…/my-appointments/history`; tapping an item opens a **Rehab Report**: resident name, therapy (name/code/duration), date/time/location, and a "Message Note" section rendering the therapy description + appointment notes. (The type carries a `summary` field; the report renders description/notes — §9.8.)

- **REHAB-FR-28 — Resident messaging UX.** "Message the Rehab Team" SHALL present a topic + message free-text form (`POST /api/rehab/rehab-message`), with submit disabled until both fields are non-empty, a success/failure confirmation screen, and a "Responses may take up to 24 hours" hint on the team tab.

**SN path — endpoint reference:**

| Operation | Method & path | Caller |
|---|---|---|
| Therapy catalog | `GET/POST rehab/therapy`, `PUT/DELETE rehab/therapy/{id}` (FormData for image) | Admin web (Rehab Service) |
| Appointments (staff) | `GET rehab/appointments` / `rehab/appointments/history` (scope, page, limit, status, search, staffCName), `POST rehab/appointments`, `PUT/DELETE rehab/appointments/{id}` | Admin web (Rehab Appointments, Rehab Calendar) |
| Appointments (resident) | `GET /api/rehab/appointments/my-appointments?date=`, `…/my-appointments/history` | SN resident app |
| Per-resident upcoming | `GET /rehab/appointments/by-resident/{cName}` | IDT Report form |
| Available slots | `GET rehab/available-slots?date&staffCName&cName&therapyId` | Admin booking form |
| Messages | `POST rehab/rehab-message` (RESIDENT), `GET rehab/rehab-message` (STAFF\|ADMIN), `GET rehab/rehab-message/my-message`, `PATCH rehab/rehab-message/{id} { status, replyBy? }` | SN app + admin web |
| Team | `GET/POST /staff` (+ `filterDesignation: "rehab"` / `designationGroup: "rehab"`), `PUT/DELETE /staff/{id}` | Admin web (Rehab Team), SN app (team tab) |
| Weekly availability | `PUT staff/availabilty/{staffCName}` *(path typo is real)* | Admin web (My Availability) |

**SN path — key data shapes:**

```typescript
RehabTherapy {
  _id, name,
  code,                  // uppercase 2–3 letters, e.g. PT|OT|ST; unique per facility (partial index)
  duration,              // minutes — drives slot length
  description?, imageKey?, imageUrl? }

RehabAppointment {
  _id, cName,                                   // resident
  status: 'SCHEDULED'|'COMPLETED'|'CANCELLED',  // + cancelledAt when cancelled
  therapyType,                                  // always "OTHER" from the admin client (legacy)
  therapyId -> therapy?,                        // catalog row; mandatory when therapyType OTHER/omitted
  agencyName?, serviceType?,                    // outside-agency tolerance
  date /*ISO*/, startTime, endTime /*HH:mm*/,
  location?, staffCName?, notes?,
  createdByType?, createdByCName?,              // booking provenance
  staff?, resident? /* populated */ }

Availability {
  weekly: { MONDAY..SUNDAY: { isAvailable, slots: [{ start:"HH:mm", end:"HH:mm" }] } } }

RehabMessage {
  _id, cName,                                   // resident author
  topic, message,                               // KMS envelope-encrypted at rest
  status: 'NEW'|'IN_PROGRESS'|'CLOSED',
  replyBy?, replyByRole? /* STAFF|ADMIN */ }
```

### 3C. Shared mechanics (both paths)

- **REHAB-FR-29 — UnifiedSchedule mirroring & cross-venue blocking.** `Care` and `RehabAppointment` documents SHALL mirror into the `UnifiedSchedule` collection via Mongoose post-save/post-update/post-delete hooks (`scheduleSync.service`; controllers must use `{ new: true }` or the hook writes stale data). UnifiedSchedule powers the resident master calendar and the cross-venue conflict rule: `scheduleType ∈ {SALON, MASSAGE, PT, CARE, REHAB}` rows **block each other** for a resident; "upcoming" statuses = `PENDING, CONFIRMED, Pending, Approved, REQUESTED` (`availableSlots.service.ts:43-51`, `rehabAvailability.service.ts:57`). ACTIVITY and TRANSPORTATION rows are written but non-blocking. Sync emits Socket.io upsert/delete events.

- **REHAB-FR-30 — Booking provenance.** Both appointment types persist `createdByType` / `createdByCName` / `residentCName` from `req.bookingContext`, enabling "booked by" displays (rehab calendar daily cards) and the case-manager "what I booked today" aggregator (which includes RehabAppointments created by the caller — backend-clinical-care §6b).

- **REHAB-FR-31 — Auto-completion cron.** A cron (default `*/15 * * * *`, gated by `ENABLE_APPOINTMENT_COMPLETION_CRON`) SHALL transition overdue appointments once `endTime` passes in facility TZ: Care `CONFIRMED → COMPLETED`, RehabAppointment `SCHEDULED → COMPLETED`. Paged (500), idempotent, per-document `findOneAndUpdate` to keep UnifiedSchedule hooks firing (known throughput limitation — TODOs at `appointmentCompletion.service.ts:126,209`).

- **REHAB-FR-32 — Unified day agenda.** The SN resident app's "My Schedule" SHALL render CARE and REHAB items (therapy `code` surfaced for rehab) inside the unified day agenda from `GET /api/unified-schedule?date=`. Rehab/care items are **not cancellable** from the agenda (unlike salon/massage/PT wellness items).

- **REHAB-FR-33 — Case-manager / doctor day-schedule inclusion.** Rehab appointments SHALL appear in the case-manager unified day aggregator (`/api/case-manager`): for case managers, RehabAppointments **created by the caller** (`createdByCName`) on the chosen date; for doctors, the named resident's rehab items from the day forward. Entries are normalized into typed REHAB cards (resident profile, creator resolution, chronological sort) and included in the printable schedule PDF (`buildUpcomingAppointmentsPdf`) (backend-clinical-care §6b).

- **REHAB-FR-34 — Search & shaping conventions.** Rehab appointment search SHALL span resident name/email/unit and location (`rehabAppointment.service.ts`); admin lists resolve populated `staff`/`resident` objects; UI conventions shared with the wellness cluster apply (durations stored as minutes and displayed "N Min", times `HH:mm` on the wire vs `h:mm AM/PM` in UI, uppercase day names with DAY_MAP translation, resident display-name fallback chain `name → firstName+lastName → "Unknown Resident"`).

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| BR-1 | The product variant a facility gets (AL nav set: `therapy-evaluations`, `physical-therapy`, `cognitive-sessions`, `private-training`, `outside-agency-services` vs SN nav set: `rehab-calendar`, `rehab-appointments`, `rehab-service`, `rehab-message`, `rehab-team`, `rehab-my-availability`, `idt-report`) is decided **entirely by the facility-pages API (Filter 1)**; no admin bypass exists. | client-admin-web §1.1, §2.9 |
| BR-2 | Care level is **resident-granular**, not facility-level: `careType ∈ {assisted_living, memory_care, independent_living, skilled_nursing}` on the Resident document. The rehab typing comment keys off "the appointment belongs to an ASSISTED_LIVING facility" — the facility/resident granularity mismatch is unresolved in code. | backend-clinical-care §8.1–8.2 |
| BR-3 | Booking-for-whom is centrally policy-driven per module key (`Config.bookingPermission.Care` / `.RehabAppointment`): resident self-service always; family by flag; staff by designation allowlist (+ groups `rehab`, `supervision`); admin bypasses designation but policy must exist. | backend-clinical-care §0 |
| BR-4 | SN rehab appointments MUST be assigned to staff with a rehab designation (`Director of Rehab`, `Rehab Therapists`, Rehabilitation Specialist group). | `rehab.ts:13`, `rehabAppointment.service.ts:256` |
| BR-5 | A booked slot must survive a server-side re-validation at write time (defense in depth on both paths); resident cross-venue double-booking is blocked across SALON/MASSAGE/PT/CARE/REHAB. | REHAB-FR-05/17/29 |
| BR-6 | Therapy catalog codes are unique per facility but **reusable after soft-delete** (partial index). | REHAB-FR-12 |
| BR-7 | OUTSIDE_AGENCY (AL) requires `agencyName` + `serviceType`; duration inherits PT. The SN `RehabAppointment` payload also tolerates optional `agencyName`/`serviceType` for outside-agency entries. | `rehab.ts`, client-admin-web §2.9 |
| BR-8 | Appointments past their end time auto-complete — there is no no-show/missed status on either path; "completed" therefore means "time elapsed", not "verified delivered". | REHAB-FR-31 |
| BR-9 | Rehab messages are a one-shot triage queue (NEW/IN_PROGRESS/CLOSED), not a conversation; resident-only creation; staff identity recorded on action. | REHAB-FR-24/25 |
| BR-10 | Resident-facing rehab scheduling is read-only on the SN path (staff book on behalf); on the AL path resident self-booking (book, reschedule, and cancel) is policy-supported in the backend and is now **live on the SL resident app** (Health hub → PT / Cognitive / Outside Agency via `/api/care/*`). The SN app still carries the same flows as orphaned (unreachable) screens. | REHAB-FR-11/11a/26 |
| BR-11 | Slot pairs are atomic: choosing a start time auto-fills its paired end (and vice versa) — staff cannot construct arbitrary start/end combinations; slot length always equals the therapy duration. | REHAB-FR-18 |
| BR-12 | Therapy catalog images are mandatory at creation but optional on edit — every catalog entry has a visual identity from day one. | REHAB-FR-13 |
| BR-13 | A therapist is "Available Now" iff their staff record is `active`, today's weekly availability has `isAvailable: true`, and the current time falls inside one of today's `{start,end}` slots — evaluated client-side every 60 s. | REHAB-FR-22 |
| BR-14 | Calendar day-bucketing treats backend ISO datetimes as **date-only in UTC** — a deliberate policy so appointments do not shift days for facilities in negative UTC offsets (e.g. Pacific). | REHAB-FR-20 |
| BR-15 | Outside-agency bookings cannot be backdated (admin form blocks past dates); no such guard is documented for the other AL types' modals. | REHAB-FR-07 |
| BR-16 | The AL path's cancel operation is a **single shared endpoint/function** across PT, Cognitive Evaluation, and Outside Agency (`DELETE /care/cancel/:id`) — despite the client function's misleading Outside-Agency-specific name (`CancelOutsideAgencyService`, TD-17). Reschedule is likewise implemented per-type but follows the same `PUT /care/:id` pattern for all three. | REHAB-FR-11a |

---

## 5. Notifications & real-time behavior

| Trigger | Channel | Recipients | Gate |
|---|---|---|---|
| Rehab appointment created / cancelled / completed | FCM push + `NotificationHistory` persistence | Resident, linked family members, assigned rehab staff | Per-facility `notificationConfig`, `HEALTH_CARE` category (e.g. `CREATE_REHAB_APPOINTMENTS`) — `rehabAppointment.notification.service.ts` |
| Care (AL therapy) creation | FCM via the shared `notifyCreationForAssignedStaff` pipeline | Resident, family, service-owner staff | Creation fan-out service (backend-wellness-dining-ops §0.5) |
| Appointment reminder | FCM, reminder cron (every minute, facility TZ) | Resident + family; staff by permission map — **PT reminders route to staff holding the `Rehab` permission** | `NotificationConfig.modules[].events[].scheduled.offsets` (+ default 15-min lead); duplicate-send guarded by unique `NotificationSentLog(scheduleId, offsetMinutes)` |
| UnifiedSchedule row upsert/delete (incl. Care/Rehab) | Socket.io `emitUnifiedScheduleUpserted` / `emitUnifiedScheduleDeleted` | Connected clients (resident calendar refresh) | Always on sync |
| Auto-completion | Silent (cron) | — | `ENABLE_APPOINTMENT_COMPLETION_CRON` |
| Rehab messages | **None observed** — no FCM/socket emission on create or status change; staff discover via the admin queue, residents via polling their list | — | — |

The Rehab Team "Available Now" badge is client-side real-time (60 s interval recompute), not socket-driven.

**Reminder-mapping nuance:** the reminder cron's module map sends SALON/MASSAGE/PT reminders under the `SALON` module / `APPOINTMENT_REMINDER` event, with staff fan-out keyed by page permission — PT → `Rehab` permission holders (`notification.service.ts:24-44`). No dedicated CARE or REHAB reminder mapping is documented in the analyzed sources; whether AL care sessions and SN rehab appointments receive pre-session reminders should be verified against `NotificationConfig` module definitions before committing reminder requirements. No reminder or notification event has been added for the new reschedule action (REHAB-FR-11a); reschedule appears to be silent, mirroring the module's general lack of a reschedule-notification pattern elsewhere in the platform.

All FCM sends persist a `NotificationHistory` row (recipientType RESIDENT/FAMILY/STAFF, scheduleType, scheduleId, title, body), giving an auditable notification trail for therapy events.

---

## 6. Integrations

| Integration | Role in this module |
|---|---|
| **Google Calendar (per-staff OAuth)** | Two directions: (1) AL — PT/COGNITIVE_EVALUATION care entries create/update/delete events on the facility **Physical Therapist's** calendar (`careCalendar.ts`; `calendarSyncStatus` PENDING/SYNCED/FAILED, fire-and-forget). (2) SN — each rehab staffer's calendar **busy windows are cached** on `Staff.cachedCalendarBusySlots` (webhook-driven refresh via `GOOGLE_CALENDAR_WEBHOOK_BASE_URL`) and **subtract** from rehab slot availability. |
| **AWS KMS** | Per-field envelope encryption of RehabMessage `topic`/`message` (AES-256-GCM, wrapped data key per field, encryption context binds facilityId+cName+model+field). One of three distinct KMS schemes in the backend (others: PCC medications, chat). |
| **AWS S3 (+ signed URLs)** | Therapy catalog images (upload via FormData; key signed on read through the shared gallery contract). |
| **Firebase FCM** | All push notifications above; every send logged to `NotificationHistory`. |
| **Print/PDF** | Rehab calendar export is a **client-side shared HTML template** (browser print), not backend Puppeteer. (Backend Puppeteer PDF exists in adjacent modules — IDT, medication list, referrals, case-manager schedule.) |
| **Socket.io** | UnifiedSchedule upsert/delete events feeding resident calendars. |

No PMS/EHR (PCC) integration touches rehab directly; rehab appointments are referenced *by* IDT reports (`upcomingAppointmentIds`).

---

## 7. Permissions & access control

**Admin web (two-filter model):** Filter 1 (facility pages API — selects the AL or SN page set, no admin bypass) → Filter 2 (per-staff `accessPermissions` tree; admins bypass). Page keys observed in the rehab suite: `"rehab-list"` (Rehab Service), `"rehab-message"`, `"rehab-team"`, and the bare `"Rehab"` key for My Availability. Page-name matching is normalized (lowercase, alphanumerics only).

**Backend route guards (as-built):**

| Route group | Guard |
|---|---|
| `POST /care` | auth + `resolveBookingContext('Care')` (policy/designation enforcement) |
| `PUT /care/:id`, `DELETE /care/cancel/:id` | auth (booking-context-equivalent enforcement was not independently re-verified for reschedule/cancel this pass — treat as consistent with `POST /care` pending confirmation) |
| `GET /care/staff/upcoming`, `/staff/history` | requireAnyRole ADMIN\|STAFF |
| `GET /care/:id`, `DELETE /care/:id` | **None — intentionally unauthenticated** (comments "Open — no auth required", `care.routes.ts:75,87`). Any caller with a facility header can read or hard-delete a care record. ⚠ |
| `rehab/therapy` CRUD | STAFF\|ADMIN (list/get: any authenticated role) |
| `rehab/appointments` create / list / history / by-resident / available-slots | STAFF\|ADMIN; create additionally passes `resolveBookingContext('RehabAppointment')` |
| `rehab/appointments` my-appointments (+history) | Resident/family via booking context |
| `rehab/appointments/{id}` get / update / delete | **auth only** — no role middleware (any authenticated identity) ⚠ |
| `rehab/rehab-message` create | requireAnyRole RESIDENT |
| `rehab/rehab-message` list (facility) / PATCH status | STAFF\|ADMIN |
| `rehab/rehab-message/my-message` | auth (own messages) |
| `PUT staff/availabilty/{staffCName}` | ADMIN any staff; STAFF self-only (UI-enforced role behavior) |

**Permission-key quirks (cross-module):** Private Training (an AL wellness offering) is gated by the **`Rehab`** access permission, and PT appointment reminders notify staff holding the `Rehab` permission — the Rehab permission key therefore spans both true rehab and private-training/PT wellness surfaces (backend-wellness-dining-ops §1.3 note).

**Tenant isolation:** all routes pass `facilityMiddleware` and scope by `x-facility-id`.

---

## 8. Product-split notes (the fork IS the split)

### 8.1 Divergence table

| Dimension | (a) Assisted Living path | (b) Skilled Nursing path |
|---|---|---|
| Therapy definition | Fixed 4-type enum (`PHYSICAL_THERAPY`, `COGNITIVE_EVALUATION`, `REHAB_EVALUATION`, `OUTSIDE_AGENCY`) | Dynamic facility-managed `RehabTherapy` catalog (name/code/duration/image) |
| Backend engine | `/api/care` (`Care` model) | `/api/rehab` (`RehabAppointment`, `therapyType:"OTHER"` + `therapyId`) |
| Slot duration source | `Config.rehab.*` per type (legacy keys) | Catalog row `duration` |
| Resource being scheduled | The **resident's** time (per-resident/date slots; no therapist assignment) | A **therapist's** time (per-staff slots; designation-validated assignment) |
| Availability inputs | Resident cross-venue conflicts + care config | Meals + staff bookings + staff Google busy cache + resident cross-venue conflicts + weekly self-availability |
| Status machine | `REQUESTED → CONFIRMED → COMPLETED \| CANCELLED` | `SCHEDULED → COMPLETED \| CANCELLED` |
| Admin surfaces | physical-therapy, cognitive-sessions, therapy-evaluations, outside-agency-services (+ private-training under the Rehab permission) | rehab-calendar, rehab-appointments, rehab-service, rehab-message, rehab-team, rehab-my-availability (+ idt-report) |
| Admin edit capability | View + create only (no edit/delete/reschedule UI) — even though the resident app itself now has reschedule (REHAB-FR-11a) | Full CRUD |
| Calendar | Therapy Evaluations monthly/daily aggregate (incl. PRIVATE_SESSION) | Rehab Calendar monthly/daily + print/PDF |
| Google Calendar | Outbound sync of PT/cognitive entries to the facility PT's calendar | Inbound busy-slot subtraction per rehab staffer |
| Resident messaging | — (none) | Encrypted rehab-message triage queue |
| Resident app surface | **Live self-booking + reschedule + cancel on the SL app** (Health hub → PT/Cognitive/Outside Agency, `/api/care/*`); same flows orphaned (unreachable) on the SN app | Live read-only schedule/history/report + messaging |
| External agency capture | **Free-text `agencyName`** on the care record | Optional `agencyName`/`serviceType` tolerated on RehabAppointment |
| Switch mechanism | Facility-pages API (Filter 1) — same backend, same admin bundle | same |

### 8.2 The agencyName inconsistency (flag for product decision)

The platform has a **managed Agency entity** (`Agency.model.ts`, `/api/agencies`: name, email, phone, address, specialties, Active/Inactive) used by the Referrals module (`selectedHHA` foreign key), CRUD-managed inside the Referrals screen — and served from the **second backend** (`authApi`). The AL Outside Agency scheduling flow ignores it entirely: `agencyName` is a free-typed string on the care record with **no referential link** to the directory (client-admin-web §2.6, §4.4). Consequences: no agency-level rollups (visits per agency), typo-fragmented data, no contact info attached to a scheduled outside visit, and two sources of truth for "agencies". The SN `RehabAppointment` carries the same free-text fields. Any future "agency performance" or "outside provider portal" feature needs this unified.

### 8.3 Other split seams

- **`PRIVATE_SESSION` leaks into Therapy Evaluations** — the AL aggregator includes private training (a wellness module riding salon slot endpoints), blurring the wellness/clinical boundary in the one screen meant to summarize therapy.
- **Permission-key sprawl**: `Rehab` gates private training and PT reminders; the SN suite uses four different page keys (`rehab-list`, `rehab-message`, `rehab-team`, `Rehab`).
- **One model, two products**: `RehabAppointment` accepts both AL fixed types and SN catalog entries; the admin AL UI nevertheless books via `/care`. Nothing in code prevents an AL-typed RehabAppointment and a Care record coexisting for the same engagement (see §9.3).

---

## 9. Observations & candidate gaps

1. **Orphaned SN-app clinical booking flows.** All 12 PhysicalTherapy/CognitiveEvaluation/OutsideAgency screens + a full `/api/care` service layer (upcoming/history/slots/booking) exist and are registered in the **SN** app's AppStack, but **no `navigate()` call reaches them**. Note: this same flow has now been **merchandised and made reachable on the SL app** (Health hub, §9.2), including reschedule (REHAB-FR-11a) — so the live precedent for re-merchandising already exists. Product decision for the SN app: delete, or wire the Health hub the same way the SL app now does.

2. **SL-app therapy screens — RESOLVED on staging, now including reschedule.** Physical Therapy, Cognitive Evaluation, and Outside Agency are now live, reachable Health-hub tiles wired to `/api/care/*` (list / details / request / slot-selection / book / **reschedule** / cancel) via `src/services/health/index.tsx` and `src/screens/App/HealthScreen/index.tsx`. The prior mock data ("Mary Johnson / Room 302"), dead tiles, and unregistered `'ScheduleScreen'` dead-end are gone. **Residual risk — status-model drift (see §9.6):** the SL care-flow type still declares `REQUESTED | APPROVED | CANCELLED` against a backend Care enum of `REQUESTED | CONFIRMED | COMPLETED | CANCELLED`, so CONFIRMED/COMPLETED appointments may mis-render in the now-live SL UI.

3. **Dual `/care` vs `rehab/*` backends for AL therapy.** The `RehabAppointment` model explicitly supports the four AL fixed types with Config durations (`rehab.ts:40-101`, `RehabAppointment.model.ts:19-26`), yet every live AL surface books through `/care`. Two parallel engines can represent the same business event ("resident has PT Tuesday 10:00") with different status machines (REQUESTED-first vs SCHEDULED-first), different slot logic, and different notification wiring. Candidate consolidation: deprecate AL fixed types on RehabAppointment, or migrate AL onto the rehab engine with a fixed catalog.

4. **Unauthenticated read + hard-delete of care records.** `GET /api/care/:id` and `DELETE /api/care/:id` are deliberately open (`care.routes.ts:75,87`) — any caller with a facility header can read or destroy AL therapy records (PHI + data-loss exposure). Related dead code: `cancelCare` is exported but never routed (`care.controller.ts:622`), so AL cancellation happens via generic update rather than a guarded cancel flow. *(Note: the SL app's cancel action, per REHAB-FR-11a, hits `DELETE /care/cancel/:id` — a distinct, dedicated cancel route, not `DELETE /care/:id`; whether that route carries proper auth was not independently re-verified this pass — see the new §7 guard-table row flagged "not independently re-verified.")*

5. **Rehab appointment by-id routes are role-unguarded.** `GET/PUT/DELETE rehab/appointments/{id}` require auth only — a resident identity could in principle mutate any appointment in their facility (backend-clinical-care §5b "get/update/delete by id require auth only").

6. **Status-model drift, app vs backend — now live-impacting.** The care-flow type declares `REQUESTED | APPROVED | CANCELLED` (`src/services/health/type.ts:23-25` in both resident apps) while the backend Care enum is `REQUESTED | CONFIRMED | COMPLETED | CANCELLED`. Because the SL app's `/care` flows are now **live** (§9.2), this drift now affects a shipped surface (not just the SN-orphaned screens): CONFIRMED and COMPLETED care appointments fall outside the app's status union and may mis-render. Reconcile the client type before relying on status display.

7. **No no-show / missed-session concept.** Auto-completion flips everything past endTime to COMPLETED on both paths (REHAB-FR-31) — therapy compliance reporting (a likely SNF need, given IDT/discharge context) cannot distinguish delivered vs missed sessions.

8. **Rehab report under-uses its data.** The resident-facing Rehab Report renders therapy description + appointment notes; the appointment type carries a `summary` field that is not rendered (client-resident-app-sn §"Rehab History"). Post-session outcome capture (the `summary` on Care; nothing structured on RehabAppointment) is effectively unused product surface.

9. **Rehab messages lack notification fan-out.** Unlike every other request queue in the platform, no FCM/socket emission was found on rehab-message create or status change — staff must poll the admin queue; residents are told "up to 24 hours" but nothing pushes a reply-status update. Candidate: wire into the creation fan-out + notificationConfig like rehab appointments.

10. **AL admin screens are create-only, even after the resident app gained reschedule.** PT, Cognitive Sessions, Therapy Evaluations, and Outside Agency pages have no edit/delete/reschedule UI (client-admin-web §2.6, §4.3) — corrections require backend access or generic update tooling. SN Rehab Appointments has full CRUD; the asymmetry is unexplained, and is now sharper: a resident can reschedule their own AL therapy appointment from the SL app (REHAB-FR-11a), but staff cannot do the same from the admin web.

11. **Endpoint and code hygiene.** `PUT staff/availabilty/{cName}` path typo is load-bearing; `therapyType:"OTHER"` is a legacy constant the admin always sends; Rehab Team's 60 s availability interval has no observed cleanup on unmount (memory-leak risk, client-admin-web §4.5); therapy calendar colors are hash-derived (collisions possible across codes; no facility control over color identity). **New:** `CancelOutsideAgencyService` (the SL app's shared cancel function for PT/CE/Outside-Agency, REHAB-FR-11a) is itself a naming-hygiene issue — logged as staff-app-analogue TD-17 in the reactnative architecture doc; recommend renaming to `CancelCareAppointment` or similar in its own small ticket.

12. **Cross-venue blocking asymmetries inherited from the spine.** Activities consume rehab/care conflicts when residents join, but rehab/care bookings ignore ACTIVITY rows — a rehab session can be double-booked over an activity the resident already joined (backend-wellness-dining-ops §0.4 note). TRANSPORTATION is also non-blocking, so a rehab slot can collide with a scheduled ride.

13. **Facility-vs-resident care-type granularity.** The fork keys off "the facility's" product variant (Filter 1, rehab.ts comments) while `careType` is stored per resident (BR-2). A mixed-acuity community (AL + SNF wings) cannot run both rehab models under one facilityId today.

14. **AL therapist resourcing is implicit and singular.** The AL path assumes exactly one "Physical Therapist" per facility (the Google Calendar sync target) and never assigns a named therapist to a session — multi-therapist AL facilities have no way to express who delivers a session, and the calendar sync target selection logic is a single-staff lookup.

15. **Slot-engine input asymmetry between paths.** The SN engine subtracts meal windows, staff bookings, staff Google busy, and resident conflicts; the AL care engine validates resident conflicts and care config only — meal blackouts and (multi-)therapist load are not inputs to AL availability. If AL adopts therapist assignment later, the SN engine is the reusable asset.

16. **New — no reschedule/cancel notification fan-out (2026-07-12).** Neither the new SL-app reschedule action nor its cancel action (REHAB-FR-11a) appears to trigger any FCM push, socket event, or `NotificationHistory` row — unlike booking creation (which does fan out per REHAB-FR notification table). Assigned staff and family members currently have no way to learn a resident rescheduled or cancelled their own AL therapy appointment except by re-checking the admin queue. *(`docs/reviews/2026-07-12/review-senior_living_reactnative.md`.)*

### Open questions for product

| # | Question | Blocking |
|---|---|---|
| Q-1 | The SL app has already merchandised AL resident `/care` self-booking, including reschedule (§9.2, REHAB-FR-11a). Should the SN app match it (wire the orphaned flows, including reschedule) or delete them? Either way, fix the status-model drift (§9.6) — it now affects the live SL surface. | AL resident self-booking roadmap |
| Q-2 | Unify `agencyName` free text with the managed Agency directory (§8.2)? Requires deciding which backend owns Agency (it currently lives behind `authApi`, a second service). | Outside-agency reporting, provider portal |
| Q-3 | Consolidate AL therapy onto one engine (`/care` vs AL-typed `RehabAppointment`, §9.3)? | Any cross-path therapy reporting |
| Q-4 | Add a missed/no-show status before auto-completion (§9.7)? | Therapy compliance & billing accuracy |
| Q-5 | Close the unauthenticated `GET/DELETE /care/:id` and role-unguarded `rehab/appointments/{id}` routes (§9.4–9.5)? | Security/PHI — should precede any external launch |
| Q-6 | Should Therapy Evaluations include PRIVATE_SESSION (wellness) or stay clinical-only (§8.3)? | AL navigation IA |
| Q-7 | Should reschedule/cancel of an AL therapy appointment notify assigned staff and family (§9 item 16), matching the notification pattern already used for booking creation? | Care-team awareness of self-service changes |
| Q-8 | Rename `CancelOutsideAgencyService` to reflect its actual shared scope (§9 item 11), and add the equivalent admin-web reschedule/edit UI (§9 item 10) to close the staff/resident capability gap? | Engineering hygiene; admin-side parity |
