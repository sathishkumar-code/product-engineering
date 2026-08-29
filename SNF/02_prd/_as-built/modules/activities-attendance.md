# Module: Activities, Scheduling & Attendance

> Applies to: Both (Senior Living / Assisted Living and Skilled Nursing)
> FR prefix: ACT
> Sources: `_codebase-analysis/backend-wellness-dining-ops.md` (§0.4, §0.5, §5), `client-admin-web.md` (§2.3, dashboard notes), `client-resident-app-sn.md` (§3.4, §3.5, Brain Games), `client-resident-app-sl.md` (§3.10, §3.16), `client-tv-app.md` (§3.6). Code is the source of truth; line references below point into those analysis files' underlying repos.

---

## 1. Purpose & scope

This module covers the facility's **community activity/event program** and the **calendar spine** that makes resident engagements visible across the platform:

1. **Activity (event) management** — staff/admin define recurring or one-off activities (capacity, location, time, image, recurrence pattern) that residents can browse and join.
2. **Resident RSVP** — join/cancel-join with capacity, eligibility, and conflict rules.
3. **Attendance** — staff mark who actually showed up (PRESENT / ABSENT / NOT_MARKED), per activity per date, with bulk actions and manually added walk-ins.
4. **Attendance analytics** — per-resident and per-activity rates, streaks, and distributions for the admin report page.
5. **UnifiedSchedule** — the cross-module denormalized calendar (`models/UnifiedSchedule.model.ts`) that aggregates salon/massage/PT/care/rehab/transportation appointments *and* activity joins into one per-resident agenda; powers the resident "My Schedule" tab, the TV 7-day feed, conflict checking, the admin "recent activity" feed, and the reminder cron.
6. **Calendar PDF export** — printable monthly activity calendar from the admin panel.
7. **Brain games** — a curated, store-referral catalog of cognitive game apps (no in-app gameplay).

Out of scope here (own module docs): the underlying salon/massage/PT/care/rehab/transportation booking flows that *feed* UnifiedSchedule; dining meal windows (which appear as TV schedule rows); announcements.

---

## 2. Personas & surfaces

| Persona | Surface | What they do here |
|---|---|---|
| Admin | Admin web — `activities-schedule`, `activity-attendance`, `activity-attendance-report` (nav parent "Activities") | Full activity CRUD, calendar + list views, PDF export, attendance marking, analytics report, dashboard "Today's Activities" stat + recent-activity feed |
| Staff | Admin web (same pages, gated by `Activities` access permission); push reminders | Same as admin where granted; activity *list API* scopes staff-only callers to activities they own (`schedule.controller.ts`); reminded to "review attendance and readiness" before activities |
| Resident | SN resident app — Activities screen (RSVP) + MySchedule tab (unified agenda) + Brain Games (Health hub) | Browse day's activities, Join / Cancel join, view unified day agenda, cancel service appointments (not activity joins) from agenda, launch brain-game store links |
| Family member | Same backend routes via family login (auth middleware rewrites identity to the linked resident) | Join activities on behalf of the resident — join row flagged `createdByFamilyMember` |
| TV device (in-room) | TV app Schedule tab | View-only 7-day unified feed (activities + meals + the resident's own bookings); no join/cancel from TV |
| SL (assisted-living) resident app | Activities (RSVP) screen + MySchedule unified agenda | **Now live as of staging** — Activities screen joins/cancels via `/api/schedules/:id/{join,cancel}` and MySchedule renders the real unified agenda via `/api/unified-schedule`; the prior mock `TASKS` plan is gone — see §8 |

Personas/auth model and `x-facility-id` multi-tenancy are as defined platform-wide (backend analysis §0.1–0.2).

---

## 3. Functional requirements (as-built)

### Activity definition & management

- **ACT-FR-01 — Activity entity.** An activity (`Schedule` model, `models/schedule.model.ts`) has: name*, owning staff `cName`, **capacity* (required, min 1)**, description, location, image (S3 `imageKey` + signed `imageUrl`, optional save-to-gallery), `isActive` soft-visibility flag, startTime/endTime* ("HH:mm", end > start enforced in admin UI), and a recurrence definition (ACT-FR-02). Effective occurrence dates are **materialized server-side** into `effectiveDates[]` at create/update time (`utils/dateCalculations.calculateEffectiveDates`); `everyday` activities keep `effectiveDates` empty (unbounded).
- **ACT-FR-02 — Five recurrence patterns.** `repeatPattern ∈ everyday | one-time | weekly | multiple-dates | date-range`:
  - `everyday` — daily, no date bounds;
  - `one-time` — single `selectedDate`;
  - `weekly` — `days[]` (MONDAY…SUNDAY; backend also supports a single numeric `weeklyDay` 0–6 with multi-`days[]` fallback, `schedule.controller.ts:56-102`) within `[startDate, endDate]`;
  - `multiple-dates` — explicit `selectedDates[]`;
  - `date-range` — every day in `[startDate, endDate]`.
  This recurrence vocabulary is a **de-facto platform primitive** — the same shapes are reused by dining menu items, daily specials, and announcements (admin analysis §4.6) and should be specified once if formalized.
- **ACT-FR-03 — Admin CRUD & views.** Admin web (`MySchedule.tsx`, 1,955 ln) offers dual views: paginated list (10/page; search; sortable on 7 fields; day-of-week and active/inactive filters layered client-side) and a month calendar (max 4 events/day with "+N more", today highlighted). CRUD: `GET /schedules?page&limit&isAdminPanel=true&search&sortBy&sortOrder`, `POST /schedules`, `PUT/DELETE /schedules/{id}` (multipart, image under form key `activity`). Validation is submit-time only: 9-field error object, endDate ≥ startDate, per-pattern date requirements. Update recomputes `effectiveDates` and swaps the S3 image; delete removes all join rows for the activity.
- **ACT-FR-04 — Persona-scoped listing.** `GET /api/schedules` (auth): staff-only callers see **only activities they own**; residents/family see active activities, each item decorated with `joined` (a JOINED UnifiedSchedule row exists for that date) and `canJoin` (activity start instant still in the future — `lib/scheduleActivityJoinWindow`). Optional date filters apply recurrence-eligibility clauses. The SN resident app consumes this as a date-driven list (`Activities/index.tsx`, `GET /api/schedules?date=`).
- **ACT-FR-05 — TV activity catalog endpoint.** `GET /api/schedules/tv` (optional auth) returns an N-day grouped view (default 7); when called with an authenticated TV session it includes per-day `joined`/`canJoin` for the paired resident. *Note:* the current TV app does **not** consume this endpoint — it renders the unified-schedule feed instead (ACT-FR-17); the join-state decoration is built but unconsumed (§9.8).

### Join / RSVP

- **ACT-FR-06 — Resident join.** `POST /api/schedules/:id/join` (auth) runs this validation chain (`schedule.controller.ts:646-819`):
  1. activity exists, is active, and is eligible for the requested date (`isScheduleEligibleForDay`);
  2. join window open — the activity has not started yet;
  3. **cross-venue conflict** — the resident has no overlapping upcoming service appointment (SALON/MASSAGE/PT/CARE/REHAB UnifiedSchedule rows) via `hasUnifiedScheduleConflict`;
  4. **activity-vs-activity overlap** — no other JOINED activity row for that resident overlaps on that date;
  5. **capacity** — count of JOINED rows for (schedule, date) ≥ capacity → `409 "This activity is full"`;
  6. idempotency — already joined → `200` no-op;
  7. success → create a UnifiedSchedule row `{scheduleType: ACTIVITY, scheduledModel: Schedule, status: 'JOINED', createdBy, createdByFamilyMember?}`.
  **Known limitation:** the capacity check is read-then-write with no atomic guard — concurrent joins can exceed capacity (§9.2).
- **ACT-FR-07 — Cancel join.** `POST /api/schedules/:id/cancel` — allowed only while the join window is open (cannot cancel after the activity starts); deletes the join row (which fires the UnifiedSchedule delete socket event).
- **ACT-FR-08 — Family on-behalf join.** Family-member logins are rewritten to the linked resident by auth middleware; their joins are recorded against the resident with the `createdByFamilyMember` flag preserved for audit.
- **ACT-FR-09 — RSVP client UX (SN app).** Join/Cancel actions sit behind a confirmation dialog with an **optimistic `joined` toggle** (`Activities/index.tsx:275-315`). This "My Calendar / Activities" RSVP surface is distinct from the MySchedule read/cancel agenda (ACT-FR-16).

### Attendance

- **ACT-FR-10 — Attendance record.** One document per `(facility, scheduleId, scheduleDate)` (`models/ScheduleAttendance.model.ts`) with embedded `attendees[{cName, joined, status NOT_MARKED|PRESENT|ABSENT, markedAt, markedBy}]`. Attendance is **intentionally decoupled from the join lifecycle** — non-joined residents (walk-ins) can be added to the roster manually.
- **ACT-FR-11 — Marking flow.** Staff/Admin only (all three endpoints role-gated):
  - `GET /schedule-attendance/schedules?date=` — activities occurring that day (pick list; admin UI auto-selects the first);
  - `GET /schedule-attendance/{scheduleId}?date=` — roster = all JOINED residents seeded `NOT_MARKED`, merged with any manually-added non-joined residents, plus summary counts (totalResidents = joined count; present/absent/notMarked stat cards);
  - `POST /schedule-attendance/{scheduleId}/mark?date=` — upsert statuses; supports `markAllPresent` / `markAllAbsent` (**joined residents only**) or an explicit attendee list (may include non-joined residents).
  Admin UI (`ActivityAttendance.tsx`): tap a resident to cycle NOT_MARKED → PRESENT → ABSENT; bulk buttons; client-side resident search (debounced 300 ms).
- **ACT-FR-12 — Attendance analytics report.** Read-only admin page (`ActivityAttendanceReport.tsx`, backed by `activityAttendanceReport.controller.ts`):
  - per-resident roster: total activities, attended, attendance rate %, last activity (name + date);
  - per-activity list: attendance vs capacity, rate %;
  - resident drill-down: summary (rate, attended/total, distinct activities, date range, **current streak** `{days, startDate}`), daily bar chart color-banded by activity count (0–1 red, 2 yellow, 3+ green), activity-distribution pie (5-color palette, wraps beyond 5), raw record table with pagination/sorting.
  - **As of staging the report adds a PDF export** of the attendance calendar / resident detail via the shared HTML template (`buildActivityCalendarHtml`, opened in a new tab that auto-prints → "Save as PDF"; matches the Rehab/IDT export convention) — `ActivityAttendanceReport.tsx:21,31-58`.

### UnifiedSchedule (cross-module calendar spine)

- **ACT-FR-13 — Aggregation model.** One denormalized row per resident-facing engagement; `scheduleType ∈ {SALON, MASSAGE, PT, CARE, CARE_CONFERENCE, REHAB, TRANSPORTATION, ACTIVITY}` with `scheduledModel` pointing at the source document. Sync happens in Mongoose `post('save')` / `post('findOneAndUpdate')` / `post('findOneAndDelete')` hooks on each appointment model — a documented contract requiring controllers to use `{ new: true }` (`scheduleSync.service.ts:22-29`). Uniqueness via partial indexes: one row per appointment doc, except CareConference (row per attendee) and **ACTIVITY joins (row per resident + schedule + date)**.
- **ACT-FR-14 — Conflict-blocking role.** Types `SALON, MASSAGE, PT, CARE, REHAB` mutually block a resident's overlapping bookings (statuses counted as upcoming: `PENDING, CONFIRMED, Pending, Approved, REQUESTED` — `availableSlots.service.ts:43-51`). **ACTIVITY rows are deliberately excluded from blocking service bookings**; activity-vs-activity overlap is checked only inside the join flow. TRANSPORTATION rows are written but never block. Net rule: *services block activity joins, but activity joins do not block service bookings* — a one-way conflict (§9.3).
- **ACT-FR-15 — Query modes.** `GET /api/unified-schedule` supports: single day (`?date=`), `upcoming=true` (auth required — powers the SN app's Upcoming Appointments screen), and `noOfDays` range. **Unauthenticated callers receive a public payload limited to meals + activities** (`getPublicScheduleForDate/Range`) — used by the TV before pairing.
- **ACT-FR-16 — Resident agenda (SN app MySchedule).** `MyScheduleScreen` (1,461 ln) normalizes heterogeneous items (ACTIVITY, SALON, MASSAGE, PT, TRANSPORTATION, CARE, REHAB — rehab therapy `code` surfaced) into one card model with status. Detail bottom-sheet for `['SALON','MASSAGE','PT','TRANSPORTATION']`; type-routed **cancel** for salon/massage/private-training only — rehab, care, and activity items are not cancellable from the agenda (activity cancel lives on the Activities screen, ACT-FR-07). Tab re-press resets to today.
- **ACT-FR-17 — TV 7-day schedule feed.** TV app Schedule tab calls `GET unified-schedule?noOfDays=7` → 7 `DailyScheduleDto{date, day, schedules[]}` with item types `SCHEDULE` (facility activity), `BREAKFAST|LUNCH|DINNER` (meals), `PT`, `SALON`, `MASSAGE`, each carrying time range, description, image, status. UX: 7-day date strip → day list → focusable detail panel. **View-only** — no cancel/modify/join from TV; bookings made elsewhere appear on next tab entry (refetch on VM init). Anonymous (unpaired) TVs get the public meals+activities payload.
- **ACT-FR-18 — Recent-activity feed.** `GET /api/unified-schedule/recent-activity` returns a normalized cross-module event stream with humanized actions per type. Admin dashboard (ADMIN only) renders the latest 5 (Transport / Salon / Dining / Housekeeping / Massage Therapy / Rehab) with color-coded status badges. The dashboard also shows a "Today's Activities" stat from `GET /schedules?date=` meta.total.

### Calendar PDF export

- **ACT-FR-19 — Printable activity calendar.** Admin web builds an HTML month calendar (`utils/pdfTemplate.ts` → `buildActivityCalendarHtml`), previews it in an A4 iframe, and prints via the browser dialog — **no PDF library exists in the app**. Header carries the facility logo and the title-cased facility type ("Skilled Nursing" / "Assisted Living"). **As of staging the shared PDF template is theme-aware** — it reads the live `--primary` CSS variable via `getPdfPrimaryColor()` and tints PDF accents to the facility theme (falling back to legacy pink `#F41095`) (`pdfTemplate.ts:34-44`). The same template now also backs the Activity Attendance Report export (ACT-FR-12). (A standalone `CalendarPdfPreview.tsx` component exists with 0 importers — the live logic is inline in `MySchedule.tsx`; §9.7.)

### Brain games

- **ACT-FR-20 — Catalog.** `BrainGame` model: name, iconUrl, App Store / Play Store URLs, categories[], rating, isActive, sortOrder. List endpoint returns active games only, with text-search score ranking and pagination. **No `facilityId` field — the catalog is global across all facilities.** No server-side play/usage tracking.
- **ACT-FR-21 — Consumption.** **Pure store referral**: the SN resident app's Health hub Brain Games screen (`GET /api/brain-games`) renders icon, name, star rating, categories and deep-links out to the App Store (iOS) / Play Store (Android). **Consumed only by the SN resident app** — the SL resident app has no brain games at all (its analysis §3.16), and neither the admin web, TV app, nor staff app surfaces them. There is **no admin management UI** for the catalog despite CRUD endpoints existing (§9.4).

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| BR-1 | Capacity is required at activity creation, minimum 1; join is rejected with 409 when JOINED count for (schedule, date) reaches capacity | `schedule.model.ts`; join flow |
| BR-2 | Join window = activity start instant; both join and cancel-join are blocked once the activity has started | `lib/scheduleActivityJoinWindow` |
| BR-3 | Joining is idempotent — re-joining an already-joined activity returns 200 without creating a duplicate row (also enforced by the partial unique index on resident+schedule+date) | join flow; UnifiedSchedule indexes |
| BR-4 | One-way conflict: an upcoming service appointment (SALON/MASSAGE/PT/CARE/REHAB) blocks an overlapping activity join, but an activity join never blocks a service booking | `availableSlots.service.ts:43-51`; backend §5.2 note |
| BR-5 | A resident may not hold two overlapping JOINED activities on the same date | join flow step 4 |
| BR-6 | Attendance is per (facility, scheduleId, scheduleDate) and decoupled from joins: bulk mark actions apply to joined residents only, but explicit attendee lists may add non-joined residents | `scheduleAttendance.service.ts` |
| BR-7 | Attendance status set is exactly `NOT_MARKED | PRESENT | ABSENT`; every joined resident is seeded `NOT_MARKED` | `constants/attendance.ts`; roster endpoint |
| BR-8 | Staff-only API callers see and manage only activities they own (`cName`); residents/family see only `isActive` activities | list endpoint scoping |
| BR-9 | `everyday` activities have no date bounds and no materialized `effectiveDates`; all other patterns are resolved to explicit dates at write time | `calculateEffectiveDates` |
| BR-10 | Deleting an activity cascades: joined residents are notified, then all join rows are removed | `schedule.controller.ts` delete path |
| BR-11 | Unpaired/unauthenticated schedule consumers (TV) see only the public subset: meals + activities, never personal bookings | `getPublicScheduleForDate/Range` |
| BR-12 | Brain games are global (cross-facility) and listing shows active entries only | `BrainGame.model.ts` |

---

## 5. Notifications & real-time behavior

- **Activity reminders (push, cron-driven).** The per-minute notification cron scans UnifiedSchedule rows whose start falls within each configured offset window (`NotificationConfig.modules[].events[].scheduled.offsets` + env default `NOTIFICATION_LEAD_MINUTES` = 15). ACTIVITY rows map to module `ACTIVITIES`, event `ACTIVITY_REMINDER` (`notification.service.ts:24-44`). Residents + linked family receive the reminder; **staff holding Dashboard or Services page permissions are additionally notified** ("review attendance and readiness"). Duplicate sends are prevented by an atomic unique insert into `NotificationSentLog (scheduleId, offsetMinutes)`.
- **Activity changed / cancelled.** Admin update fires `notifyActivityChanged` to all joined residents; delete fires `notifyActivityCancelled` to joined residents before removing join rows.
- **Real-time calendar sync.** UnifiedSchedule sync hooks emit Socket.io `emitUnifiedScheduleUpserted` / `emitUnifiedScheduleDeleted` (`config/socket.ts`) — so a join/cancel or any appointment change can be reflected live by connected clients. (The TV app does **not** subscribe — its socket usage is pairing/auth only; schedule changes appear on next tab entry.)
- **Timezone caveat.** The reminder cron runs in `FACILITY_TIMEZONE` (a process-wide env var, default `America/Los_Angeles`) — not per-facility `Config.timeZone` (§9.6).
- All pushes go via Firebase FCM and are logged to `NotificationHistory` (recipientType, scheduleType, scheduleId, title, body).

---

## 6. Integrations

| Integration | Role in this module |
|---|---|
| AWS S3 + Gallery | Activity images stored as S3 keys with signed display URLs (send `imageKey`, never the signed URL); `isSavedToGallary: true` [sic] side-effects the image into the global `GalleryImage` library |
| Firebase FCM | Reminder, change, and cancellation pushes |
| Socket.io | UnifiedSchedule upsert/delete events for live calendar refresh |
| PMS switchboard | `Config.integratedModules` includes an `ACTIVITY` slot (OPERA / POINTCLICKCARE / YARDI / TELS / CUSTOM) but **no activity PMS provider is implemented today** — only HOUSEKEEPING/MAINTENANCE→TELS exists |
| App Store / Play Store | Brain-game deep links (outbound only) |
| Browser print | Calendar PDF export (HTML template + native print dialog; no PDF library) |

No Google Calendar sync applies to activities (that integration is scoped to salon/massage/PT staff calendars).

---

## 7. Permissions & access control

- **Admin web visibility**: nav parent "Activities" with sub-pages `activities-schedule`, `activity-attendance`, `activity-attendance-report`; gated by Filter 1 (facility-enabled pages) ∩ Filter 2 (staff `accessPermissions`, ADMIN bypass). The Activities sub-pages are among those not tracked in the access-pages API, so they ride the sub-item inheritance rule (`canAccessSubItem`). Page-level gating uses `usePageAccess("Activities")` with the universal read-only pattern (mutations disabled + toast).
- **Backend route auth (as-built):**

| Operation | Auth |
|---|---|
| Create activity (`POST /schedules`) | Authenticated, **any role** (no role gate) |
| List (`GET /schedules`, `GET /schedules/tv`) | Auth (TV variant optional-auth); persona-scoped per BR-8 |
| **Update / delete activity** (`PUT/DELETE /schedules/{id}`) | **No auth middleware at all** (`schedule.routes.ts:39-40`) — see §9.1 |
| Join / cancel join | Authenticated resident/family (family rewritten to resident) |
| Attendance (all 3 endpoints) | Role-gated **STAFF/ADMIN only** |
| Unified schedule | Auth for personal/upcoming modes; public meals+activities payload when unauthenticated |
| Brain games CRUD | **Auth only, no role gate** — any valid token (including a resident's) can mutate the **global** catalog; see §9.4 |

- Activities do **not** use the `resolveBookingContext` on-behalf policy that governs bookable services — family on-behalf joining is implicit via the auth-middleware identity rewrite, with no facility-level `bookingPermission` toggle for activities.

---

## 8. Product-split notes

- **One backend, per-app divergence.** Both resident apps and the TV hit the same endpoints; the split is per-repo/per-binary, not feature-flagged.
- **SN resident app = the live implementation**: Activities RSVP screen (join/cancel), MySchedule unified agenda (1,461 ln, all 7 schedule types), Upcoming Appointments (unified `upcoming=true`), Brain Games. Treat as the reference client.
- **SL resident app — now live for activities/agenda (staging):** the hardcoded `TASKS` mock day plan is gone. MySchedule (`src/screens/App/MyScheduleScreen`) renders the real unified agenda via `fetchResidentSchedule` → `/api/unified-schedule`, and a new **Activities RSVP screen** (`src/screens/App/Activities`, reached from the Home screen) joins/cancels via `joinScheduleEvent`/`cancelScheduleEvent` → `/api/schedules/:id/{join,cancel}` (`src/services/services/schedule/index.ts`). This is the activities/agenda parity the PRD previously called net-new — it has now shipped on the SL app. **Still absent on SL:** brain games (no brain-game screen or `/api/brain-games` call anywhere in the SL repo).
- **TV app is shared across both products**: unified 7-day feed, view-only; the per-resident `joined`/`canJoin` decoration on `GET /api/schedules/tv` is unused by the current TV client.
- **Admin web is one binary for both facility types**: no facility-type branching in the Activities pages; the PDF export header title-cases the facility type. Attendance and the report behave identically for AL and SNF.
- **Attendance & analytics are admin/staff-web-only** — no staff-app or resident-facing attendance surface exists.

---

## 9. Observations & candidate gaps (with evidence refs)

1. **Unauthenticated activity mutation (security).** `PUT /schedules/{id}` and `DELETE /schedules/{id}` have **no auth middleware** (`schedule.routes.ts:39-40`) — anyone who can reach the API can rewrite or delete activities (and trigger cancellation pushes to joined residents). Create, by contrast, requires auth but no role. Compounded by the dead facility-header check (`facilityMiddleware.ts:40` — missing `x-facility-id` is not actually rejected, so queries can run cross-tenant).
2. **Non-atomic capacity check.** Join capacity is read-then-write with no atomic guard (`schedule.controller.ts:646-819`); concurrent joins can oversubscribe an activity. Candidate fix: conditional upsert / counter document.
3. **One-way conflict rule.** Services block activity joins, but UnifiedSchedule deliberately excludes ACTIVITY rows from the service-booking conflict set (`availableSlots.service.ts:43-51`) — a resident can book a salon slot over an activity they joined, with no warning on either side. Decide whether this is policy (activities are soft commitments) or a gap; today it is undocumented behavior.
4. **Brain games governance gap.** Catalog is global (no `facilityId`), CRUD is auth-only with no role gate — a resident token could mutate the catalog every facility sees (backend §5.4, §8.3). Also: endpoints support full CRUD but **no admin UI manages brain games**; curation is presumably done by direct API calls.
5. **Attendance summary semantics.** `totalResidents` = joined count, while manually added non-joined attendees still receive statuses — so present+absent+notMarked can exceed `totalResidents`, and the report's denominators (rates, streaks) need a stated policy for walk-ins. Worth an explicit definition in the formal PRD.
6. **Timezone model.** Reminder cron and join-window math run on the process-wide `FACILITY_TIMEZONE` env var despite `Config.timeZone` existing per facility — multi-timezone deployments are not actually supported (backend §8.16).
7. **Dead/orphaned admin artifacts.** `ActivitiesEvents.tsx` (0 importers) and `CalendarPdfPreview.tsx` (0 importers; PDF logic lives inline in `MySchedule.tsx`); copy-paste React Query key bug — `use-delete-staff.ts` invalidates `['get-activities']` instead of `['get-staff']`, and MySchedule's delete invalidates `['get-activities']` where the fetch key is `['get-schedules']` (stale list after delete; admin analysis §4.5).
8. **TV ignores the purpose-built endpoint.** Backend `GET /api/schedules/tv` returns per-day `joined`/`canJoin` for the paired resident, but the TV app renders `GET unified-schedule?noOfDays=7` instead and offers no join UI — either retire the TV decoration or build TV-side RSVP against it.
9. **No edit-from-attendance or historical re-marking guard.** Attendance can be marked for any date the activity occurred on; nothing prevents (or audits beyond `markedBy/markedAt`) retroactive edits long after the event. Acceptable today; flag if attendance feeds billing or compliance later.
10. **Recurrence vocabulary divergence risk.** Activities use lowercase hyphenated patterns (`one-time`, `multiple-dates`, plus `everyday`); dining items use UPPER_SNAKE (`ONE_TIME`, `MULTIPLE_DATES`, no everyday). Same concept, two encodings — formalize one platform scheduling primitive (admin analysis §5.1).
11. **SL app over-credit risk — largely RESOLVED on staging.** The SL MySchedule mock (`TASKS` / "Haircut & Style — Approved" / "Movie Night") has been replaced with the real `/api/unified-schedule` agenda, and a live Activities RSVP screen now exists. The over-credit risk no longer applies to the activities/agenda surface. The one remaining SL gap relative to SN is **brain games**, which the SL app still does not implement at all.
