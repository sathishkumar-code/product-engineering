# Module: Housekeeping & Maintenance

> Applies to: Both (Senior Living / Assisted Living **and** Skilled Nursing)
> FR prefix: HSK
> Sources: `_codebase-analysis/backend-wellness-dining-ops.md` (§0, §4, §7–8), `client-admin-web.md` (§2.8, §3–5), `client-resident-app-sl.md` (§3.6), `client-resident-app-sn.md` (§3.7), `client-staff-app.md` (§4.5, §5.1), `client-tv-app.md` (§3.7). Code is the source of truth; this document describes as-built behavior.

---

## 1. Purpose & scope

Housekeeping & Maintenance is the facility's resident-initiated service-request system. Residents (or family members, staff, and admins acting on their behalf) submit four kinds of requests — extra room cleaning, extra laundry, miscellaneous service, and maintenance — which flow into a staff work queue and an admin processing console, with optional outbound/inbound synchronization to the TELS facility-management PMS.

In the backend this is a **single module with a `requestType` discriminator**: one Mongoose model (`models/houseKeeping.model.ts`, model name `ServiceRequest`), one controller, one route family (`/api/housekeeping`). All four request types share the same lifecycle, queue, notification fan-out, and integration path; they differ only in submission-form fields, assignee designation, and TELS mapping.

**In scope:** request submission (resident app SL/SN, TV app, case-manager on-behalf routes), the PENDING → IN_PROGRESS → COMPLETED / REJECTED status workflow with approval and rejection rules, staff queue processing on the staff app, admin per-type request queues, TELS work-order / housekeeping-job sync via `Config.integratedModules`, notifications (FCM push + socket events).

**Out of scope (other modules):** dining/family meals, transportation, salon/wellness bookings, activities, the unified schedule (housekeeping deliberately does **not** sync to `UnifiedSchedule` — backend §4.1), rehab/clinical.

---

## 2. Personas & surfaces

| Persona | Surface(s) | Capabilities |
|---|---|---|
| Resident | Resident app (SL & SN variants), TV app | Submit any of the 4 request types; view own upcoming and historical requests (mobile only; TV has no history view) |
| Family member | Resident app (family login) | Submit/view for the linked resident, **only if** facility config `bookingPermission.Housekeeping.isFamilyMemberAllowed === true` (backend §0.3) |
| Housekeeping Staff (designation) | Staff app, admin web | Staff app: process non-maintenance queue (Accept → Complete). Admin web: process cleaning/laundry/misc queues if page permission granted |
| Maintenance Staff (designation) | Staff app, admin web | Same queue UI filtered to `MAINTENANCE` requests (`isMaintenance` param) |
| Case Manager / Social Worker (designations) | Admin web / API | Book and view on behalf of a resident via the `/case-manager/resident/:residentId/...` route family (STAFF role check only — bypasses booking-context policy, backend §0.3) |
| Facility admin (ADMIN / SUPER_ADMIN) | Admin web | Full per-type queues via `GET /housekeeping-admin`; all status transitions; staff assignment |
| TELS (external system) | Webhook `/webhooks/tels` | Pushes status updates back into requests it owns (HMAC-authenticated) |

Surface inventory:

- **Resident app — SL** (`OtherServices/*`): 4-type chooser → per-type request form → per-type request list. LIVE (API-backed).
- **Resident app — SN** (`OtherServices/*`): same component family with light drift; per-type list backed by `/api/housekeeping/resident[/history]`.
- **TV app** (`features/housekeeping/`): 4 hardcoded request cards → QR auth gate → calendar dialog → "charges may apply" confirmation → submit. No history/status view.
- **Staff app** (`DesignationViews/HousekeepingStaffView.tsx`): one shared queue component for Housekeeping Staff and Maintenance Staff designations.
- **Admin web**: four near-identical screens — `ExtraRoomCleaning.tsx`, `ExtraLaundry.tsx`, `MiscellaneousService.tsx`, `MaintenanceRequests.tsx` — over one backend, differing only by `type` param and assignee designation filter.

---

## 3. Functional requirements (as-built)

### Request model & types

- **HSK-FR-01 — Four request types on one entity.** A service request carries `requestType ∈ {EXTRA_ROOM_CLEANING, EXTRA_LAUNDRY, MISC, MAINTENANCE}` (backend enum; the admin web sends `MISCELLANEOUS_SERVICE` for the misc page — see §9 Observation O-2). Single model `ServiceRequest` with: `priority` (LOW/MEDIUM/HIGH, default MEDIUM), unique `requestCode`, `unitNo`, `dateRequested` + `selectedDate` (date the resident wants service), status, `assignedTo` (Staff ref), optional image (S3; MISC/MAINTENANCE only), `categoryId`/`categoryName`, `scheduledTime`, `hasPermissionToEnter`, TELS back-links (`telsWorkOrderId`, `telsJobId`, `telsScheduleId`), and creator provenance `createdByType`/`createdByCName` (backend §4.1).
- **HSK-FR-02 — Request code generation.** Each request receives a unique `requestCode` generated as `R` + last 4 digits of epoch milliseconds (`housekeeping.controller.ts:942`). Collision-prone on a unique index — see §9 gap G-1. (The admin-web analysis shows display examples like `HOUSE-20250612-001`, suggesting either a format drift or client-side display formatting — see §9 O-3.)

### Submission

- **HSK-FR-03 — Resident submission (mobile).** `POST /api/housekeeping` behind `resolveBookingContext` (module key `Housekeeping`). Form fields vary by type:
  - EXTRA_ROOM_CLEANING / EXTRA_LAUNDRY: date picker only; client validates the date is **strictly in the future** ("Please select a future date" — SL app §3.6).
  - MISC / MAINTENANCE: required free-text description (inline validation error) + optional single photo via camera/gallery with runtime permission handling (`AppImagePicker.tsx`).
  - Submission payload (SL, updated on staging): a `multipart/form-data` body with `residentId, residentName, unitNo, requestType, description, selectedDate?` and image files appended under the misspelled `maintanance` field as real React Native file parts (`{uri, type:'image/jpeg', name}`) — `createHousekeepingRequest(FormData)` posts `multipart/form-data` to `/api/housekeeping` (`OtherServiceRequest/index.tsx:233-275`). The prior "local device URI with no upload step" defect is fixed; the field name typo persists (see §9 G-2).
  - Success → per-type confirmation screen; **as of staging, failures also route to the confirmation screen with an error state** (`isSuccess: false` + message), not just a `console.warn` (§9 G-3).
- **HSK-FR-04 — TV submission.** TV app flow: request-type card → auth gate (QR pairing login if no token) → `HousekeepingCalendarDialog` date selection → confirmation dialog with "charges may apply" copy → `POST housekeeping` with `{residentId: "", residentName, selectedDate, unitNo, requestType, description}`. `residentId` is sent empty; the backend resolves resident identity from the TV token (TV requests carry the resident's cName as username, backend §0.1). MAINTENANCE on TV includes a free-text description area (a mic-icon flag exists but no voice input is implemented). The TV offers no request history or status view.
- **HSK-FR-05 — Duplicate guard for dated cleaning/laundry.** On create, the backend rejects a second EXTRA_ROOM_CLEANING or EXTRA_LAUNDRY request for the **same resident on the same `selectedDate`** with "Can not add multiple request for same date" (`housekeeping.controller.ts:921-939`). MISC and MAINTENANCE are not duplicate-guarded.
- **HSK-FR-06 — On-behalf submission policy.** Creation honors the central booking-context policy (backend §0.3): residents always self-serve (any supplied `residentCName` is ignored — spoof guard); family allowed per facility config; staff allowed only when their `designation` is in the facility's `staffDesignationAllowed` for Housekeeping; admins bypass designation checks but must name the resident. The resulting `createdByType`/`createdByCName` is persisted on the document.
- **HSK-FR-07 — Case-manager routes.** A parallel route trio `/case-manager/resident/:residentId/...` lets **any STAFF** create/view housekeeping requests for a resident by `_id`, bypassing `resolveBookingContext` (STAFF role check only) (backend §0.3, §4.2).

### Status workflow & processing

- **HSK-FR-08 — Status lifecycle.** `status ∈ PENDING → IN_PROGRESS → COMPLETED | REJECTED | CANCELLED`. **No transition matrix is enforced server-side** — the update endpoint sets whatever status is sent (backend §4.1; see §9 G-4). The intended workflow, as implemented by the admin client (admin §2.8):

  | From | Action | Required inputs | To |
  |---|---|---|---|
  | PENDING | Approve | assigned staff (≠ "Unassigned") **and** remarks | IN_PROGRESS |
  | IN_PROGRESS | Mark Completed | assigned staff **and** remarks; client sets `completedAt` (ISO) | COMPLETED |
  | PENDING | Reject | rejection reason | REJECTED |

- **HSK-FR-09 — Single update endpoint.** All transitions go through `PUT /api/housekeeping` with `{ id, status, assignedTo?, remarks?, rejectedReason?, completedAt? }`. The endpoint requires authentication but **no role or permission check** (backend §4.2; §9 G-5).
- **HSK-FR-10 — Staff auto-assignment.** If a STAFF user updates a request without supplying `assignedTo`, the backend auto-assigns the request to them (`housekeeping.controller.ts:1076-1090`). This is what makes the staff app's two-tap flow work without an explicit assignment step.
- **HSK-FR-11 — Staff queue (staff app).** `GET /api/housekeeping/housekeeping-staff?page&limit[&isMaintenance=true]` returns the processing queue; `isMaintenance=true` filters to MAINTENANCE, otherwise non-MAINTENANCE types. One shared view component serves both designations. Card shows request-type label, unit number, optional description, and raw `dateRequested`; the request `image` is never rendered (§9 G-6). Two-step processing via `PUT /api/housekeeping { id, status }`:
  - PENDING → **Accept Request** → IN_PROGRESS (card updates in place, button becomes green "Mark as Completed")
  - IN_PROGRESS → **Mark as Completed** → COMPLETED (card removed from list)
  - Pagination 10/page with pull-to-refresh; per-row updating lock (`isUpdatingId`); `hasMore` inferred from page-fullness because the endpoint returns no pagination meta (staff §4.5, §8.11). Note the staff app's lightweight flow sends **no remarks/assignee**, relying on auto-assignment (HSK-FR-10) — stricter input rules exist only in the admin client (HSK-FR-08).

### Views & queues

- **HSK-FR-12 — Resident views.** `GET /api/housekeeping/resident` — upcoming: `selectedDate ≥ today` with status PENDING/IN_PROGRESS. `GET /api/housekeeping/resident/history` — completed/rejected past requests + all cancelled. Both bookingContext-gated; the page-permission middleware on these routes is deliberately commented out so designations allowed by booking policy (Case Manager, Social Worker) are not blocked (route comment `housekeeping.routes.ts:48-53`). **As of staging the SL app reads via `/api/housekeeping/resident[/history]`** (`services/services/housekeeping/index.ts:19,56`), the same path as SN — the older `GET /api/housekeeping?type=` per-type read is no longer used, and the SL app now has a history view. Resident list cards render only title + requested date — status chips exist in the type model (`PENDING/COMPLETED/CANCELLED`) but are not rendered, and there is **no resident-initiated cancel** in any client (§9 G-7).
- **HSK-FR-13 — Admin queues (per type).** Four admin-web screens, one per request type, each a request-queue over the same backend:

  | Page (permission key) | `type` param sent | Assignee designation filter |
  |---|---|---|
  | `extra-room-cleaning` | `EXTRA_ROOM_CLEANING` | Housekeeping Staff |
  | `extra-laundry` | `EXTRA_LAUNDRY` | Housekeeping Staff |
  | `miscellaneous-service` | `MISCELLANEOUS_SERVICE` | Housekeeping Staff |
  | `maintenance-requests` | `MAINTENANCE` | Maintenance Staff |

  Role-dependent read endpoint: ADMIN reads `GET housekeeping/housekeeping-admin`, STAFF reads `GET housekeeping` — both with `type, status, search, date, page, limit`. Filters: search (resident / room / request ID), status, date; pagination default 30/page. Timezone-aware date display and querying via facility `timeZone` from config. The list payload includes resident population (unitNo, careType, photo) — per-resident `careType` lets one queue serve mixed-acuity facilities (backend §7.1). No bulk actions, no recurring schedules, no real-time updates, no staff-availability check at assignment (admin §2.8).
- **HSK-FR-14 — Dashboard surfacing.** Housekeeping pending counts contribute to the admin dashboard "Pending Requests" aggregate (component included only if the viewer passes `checkAccess()` for the module), and completed/approved requests appear in the ADMIN-only Recent Activity feed (`GET /unified-schedule/recent-activity`) with color-coded status badges (admin §3).

### Integration

- **HSK-FR-15 — TELS sync (outbound).** Governed by the per-facility module-to-PMS switchboard `Config.integratedModules` (backend §7.3):
  - `integratedModules.MAINTENANCE === 'TELS'` → MAINTENANCE requests create a TELS **work order**.
  - `integratedModules.HOUSEKEEPING === 'TELS'` → EXTRA_ROOM_CLEANING / EXTRA_LAUNDRY / MISC requests create a TELS **housekeeping job**.
  - Returned TELS ids are stored back on the request (`telsWorkOrderId` / `telsJobId` / `telsScheduleId`). Mapping helpers translate priority (platform LOW/MEDIUM/HIGH ↔ TELS 1–3) and status codes. Implementation: `integrations/pms/workOrder.integration.ts` + `integrations/pms/tels/tels.service.ts`.
- **HSK-FR-16 — TELS sync (inbound).** `POST /webhooks/tels`, protected by HMAC-auth middleware, updates request status from TELS-side changes (backend §4.2).
- **HSK-FR-17 — TELS categories.** `GET /api/housekeeping/categories` returns the facility's TELS work-order categories, used to populate `categoryId`/`categoryName` on requests (backend §4.2).

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| BR-1 | One EXTRA_ROOM_CLEANING and one EXTRA_LAUNDRY request per resident per `selectedDate`; duplicates rejected at create | backend §4.2 (`:921-939`) |
| BR-2 | Cleaning/laundry `selectedDate` must be strictly in the future (client-side validation only) | SL app §3.6 |
| BR-3 | MISC and MAINTENANCE require a description; photo optional. Cleaning/laundry require only a date | SL app §3.6, TV §3.7 |
| BR-4 | Approval requires an assigned staff member and remarks; completion requires staff + remarks + `completedAt`; rejection requires a reason — enforced in the **admin client only**, not the API | admin §2.8 |
| BR-5 | A STAFF updater who omits `assignedTo` becomes the assignee (auto-assignment) | backend §4.2 (`:1076-1090`) |
| BR-6 | Family submission allowed only when `bookingPermission.Housekeeping.isFamilyMemberAllowed` is true; staff on-behalf gated by designation allow-list; admin bypasses designation but must name the resident | backend §0.3 |
| BR-7 | Assignee candidate lists are designation-filtered: "Housekeeping Staff" for cleaning/laundry/misc, "Maintenance Staff" for maintenance | admin §1 (designations), §2.8 |
| BR-8 | Default priority MEDIUM; LOW/HIGH available; priority maps to TELS 1–3 when synced | backend §4.1, §4.2 |
| BR-9 | Housekeeping requests are **not** written to UnifiedSchedule and do not participate in cross-venue booking-conflict checks | backend §4.1, §0.4 |
| BR-10 | TV submissions display "charges may apply" before confirming; charging itself is not implemented anywhere in the flow | TV §3.7 (`AppConstants.kt:87-90`) |
| BR-11 | All queries are facility-scoped via the `x-facility-id` header (with the known missing-header bypass — see §9 G-9) | backend §0.2 |
| BR-12 | Status values are UPPER_SNAKE (`PENDING`, `IN_PROGRESS`) in this domain — one of three casing conventions across the platform | admin §5 |

---

## 5. Notifications & real-time behavior

**Push (FCM)** — `services/serviceRequest.notification.service.ts`, with separate **maintenance vs housekeeping flavors** for each event: created / assigned / in-progress / completed (backend §4.2). Recipients:

- **Creation fan-out** (`requestCreation.notification.service.ts` → `notifyCreationByPermission`): the resident + linked family members + all staff holding the relevant page permission (Housekeeping or Maintenance, via `REQUEST_CREATION_PERMISSIONS`) — permission-based fan-out, unlike salon/massage/PT which notify the specific service-owner staff (backend §0.5).
- **Lifecycle events** (assigned / in-progress / completed): resident + family + permission-based staff.
- Every send is logged to `NotificationHistory` (recipientType, scheduleType, scheduleId, title, body).
- Because housekeeping does not sync to UnifiedSchedule, it gets **no reminder-cron notifications and no auto-completion** — reminders and auto-complete are UnifiedSchedule-driven (backend §0.5; §9 O-5).

**Socket.io (in-app real-time)** — create/update/delete emit `mobile-housekeeping-request-upserted` / `mobile-housekeeping-request-deleted` via `emitAppRequestEvent` (backend §0.5).

- Staff app: Housekeeping Staff **and** Maintenance Staff designations both subscribe to the same two housekeeping channels; event payloads are ignored — any event bumps a `refreshTrigger` and the view silently re-fetches page 1 (staff §5.1). Consequence: maintenance staffers also receive refresh triggers for non-maintenance changes (over-refresh, benign — staff §8.16).
- Admin web: **no real-time updates** on the four queues — manual refresh/poll only (admin §2.8; §9 G-8).
- The socket is consume-only; no client→server emits.

---

## 6. Integrations (TELS)

TELS is the only PMS provider implemented for this module (the `integratedModules` switchboard enumerates `OPERA | POINTCLICKCARE | YARDI | TELS | CUSTOM`, but only HOUSEKEEPING/MAINTENANCE→TELS has an implementation today — backend §7.3).

| Aspect | Behavior |
|---|---|
| Activation | Per facility, per module: `Config.integratedModules.MAINTENANCE = 'TELS'` and/or `Config.integratedModules.HOUSEKEEPING = 'TELS'`; credentials in `Config.pms[]` |
| Outbound — maintenance | Request create → TELS **work order**; `telsWorkOrderId` stored on the request |
| Outbound — housekeeping | Cleaning / laundry / misc create → TELS **housekeeping job**; `telsJobId` / `telsScheduleId` stored |
| Field mapping | Priority LOW/MEDIUM/HIGH ↔ TELS 1–3; status-code mapping helpers in `tels.service.ts` |
| Inbound | `POST /webhooks/tels` (HMAC-authenticated middleware) updates request status from TELS |
| Categories | `GET /api/housekeeping/categories` proxies TELS work-order categories for the facility; selected category persisted as `categoryId`/`categoryName` |
| Code locations | `integrations/pms/workOrder.integration.ts`, `integrations/pms/tels/tels.service.ts` |

Known integration-adjacent quirks: debug `console.log`s in the TELS config path (backend §8.9); no documented retry/reconciliation behavior in the analysis if the TELS call fails at create time (§9 O-6).

---

## 7. Permissions & access control

| Operation | Endpoint | Guard (as-built) |
|---|---|---|
| Create | `POST /api/housekeeping` | Auth + `resolveBookingContext('Housekeeping')` (BR-6) |
| Update (all transitions) | `PUT /api/housekeeping` | **Auth only — no role/permission check** (G-5); STAFF auto-assignment side effect |
| Resident upcoming / history | `GET /api/housekeeping/resident[/history]` | Auth + bookingContext; page-permission check intentionally commented out (`housekeeping.routes.ts:48-53`) |
| Staff queue | `GET /api/housekeeping/housekeeping-staff` | Auth |
| Admin queue | `GET /api/housekeeping/housekeeping-admin` | **No auth** (G-10) |
| Case-manager trio | `/case-manager/resident/:residentId/...` | STAFF role only (bypasses bookingContext) |
| TELS webhook | `POST /webhooks/tels` | HMAC auth middleware |
| TELS categories | `GET /api/housekeeping/categories` | Per backend §4.2 (facility-scoped) |

Client-side gating:

- **Admin web**: four page-permission keys (`extra-room-cleaning`, `extra-laundry`, `miscellaneous-service`, `maintenance-requests`) drive nav visibility; the live permission tree comes from `GET /config/access-pages` with a hardcoded fallback list including `housekeeping` and `maintenance` pages (admin §1).
- **Staff app**: routing is by `designation` — `HOUSEKEEPING_STAFF` and `MAINTENANCE_STAFF` land on the queue view; all other designations see no housekeeping UI (staff §2).
- **Backend staff permissions**: `Staff.accessPermissions` includes distinct Housekeeping and Maintenance module entries used for notification fan-out and (where mounted) `requireAnyStaffPermission`; the permission middlewares no-op for non-STAFF users (backend §0.1).
- **TV**: submission requires a paired TV token (QR pairing); identity comes from the token, not the request body.

---

## 8. Product-split notes

- **Shared module, both products.** The backend module, status workflow, TELS integration, and staff/admin tooling are identical for Senior Living and Skilled Nursing. Admin-web housekeeping screens are facility-type agnostic in code; differentiation is purely which pages the facility-config API enables (admin §6).
- **Resident apps**: SL and SN variants ship near-identical `OtherServices` implementations ("shared-with-light-drift"). As of staging the read-path drift is gone — **both apps now hit `/api/housekeeping/resident[/history]`** (the SL app's older `GET /api/housekeeping?type=` read is retired), and the SL photo upload is now a real multipart upload (G-2). The SN app remains slightly newer RN; either can serve as the extraction baseline if components are unified.
- **Mixed-acuity ready**: `careType` is per-resident, not per-facility, and is included in queue payloads, so one facility's queues can mix assisted-living, memory-care, independent-living, and skilled-nursing residents (backend §7.1).
- **Staff app is the hospitality/operations half** — housekeeping/maintenance is fully functional there; clinical designations are stubs. In a product split, this module travels with the operations product on both SL and SN SKUs (staff §8.2).
- **TELS relevance** skews toward facilities with formal plant-operations programs (typically larger AL/SNF buildings); the switchboard already isolates it per facility, so no split work is needed.

---

## 9. Observations & candidate gaps (with evidence refs)

**Security / integrity gaps**

- **G-10 — Admin list endpoint is unauthenticated.** `GET /housekeeping-admin` (paginated, full request data incl. resident details) has no auth middleware (backend §4.2, §8.1).
- **G-5 — Update endpoint has no role gate.** Any authenticated user — including a resident — can `PUT /api/housekeeping` and set any status/assignee/remarks on any request; no transition matrix is enforced (backend §4.2, §8.4; HSK-FR-08/09).
- **G-9 — Facility-scoping bypass.** A missing `x-facility-id` header is not rejected (undefined slips past the null/empty check) and queries silently run unscoped — a platform-wide issue that includes this module (backend §0.2).
- **G-4 — Status machine is client-convention only.** The PENDING→IN_PROGRESS→COMPLETED/REJECTED workflow with required remarks/reason exists only in the admin client; the staff app sends bare `{id, status}` and the API accepts anything (admin §2.8 vs staff §4.5).

**Functional gaps**

- **G-1 — Request-code collisions.** `requestCode = 'R' + last 4 digits of Date.now()` on a unique index — only 10,000 possible values; collisions surface as raw 500s (backend §4.1, §8.12).
- **G-2 — SL photo upload FIXED on staging (typo persists).** The SL resident app now builds a `multipart/form-data` body and appends each photo as a real RN file part (`{uri, type, name}`) under the `maintanance` field; `createHousekeepingRequest` posts it as `multipart/form-data` (`OtherServiceRequest/index.tsx:233-275`, `services/services/housekeeping/index.ts`). Photos now resolve server-side. **Residual:** the field name is still the misspelled `maintanance` (O-4), and the client payload type is `FormData`/loose rather than the original typed shape.
- **G-3 — Silent submission errors RESOLVED on staging.** A failed SL submission now routes to the Confirmation screen with `isSuccess: false` and the error message (`OtherServiceRequest/index.tsx:280-291`); the resident sees a user-facing failure state (still also logged via `console.warn`).
- **G-7 — No resident cancel, no visible status.** Resident list cards render only title + date; status enums exist in the client models but are unrendered; there is no cancel action in any resident surface, despite CANCELLED existing in the lifecycle (SL §3.6, SN §3.7).
- **G-6 — Staff never see the request photo.** `HousekeepingStaffRequestData.image` is never rendered in the staff queue, defeating the purpose of MISC/MAINTENANCE photo capture (staff §8.4).
- **G-8 — No real-time on admin queues.** Sockets exist for mobile clients only; the admin console requires manual refresh (admin §2.8).
- **G-11 — TV has no history/status view.** A resident submitting from the TV gets a success dialog and no later visibility (TV §3.7).

**Consistency observations**

- **O-1 — Type-enum drift.** Backend enum is `MISC`; the admin web sends `MISCELLANEOUS_SERVICE` as the type param for the misc queue (admin §2.8 table vs backend §4.1). Mobile/TV clients map UI labels to `MISC`. Reconcile during any API formalization.
- **O-2 — Status casing.** This domain uses UPPER_SNAKE statuses while transport uses Title Case — a unified API needs a normalization story (admin §5).
- **O-3 — Request-code display drift.** Backend generates `R####`; the admin analysis cites a `HOUSE-YYYYMMDD-NNN` display example — verify whether this is client formatting or a second generator (admin §2.8 vs backend §4.1).
- **O-4 — Spelling/typos in contract surfaces.** `maintanance` payload field (SL §3.6), "Maintainance Request" label (staff §8.15), `isSavedToGallary` flag on the gallery side-effect path (backend §6) — all load-bearing strings.
- **O-5 — No reminders / auto-completion.** Because housekeeping skips UnifiedSchedule, the platform's reminder cron and auto-completion cron never touch these requests; an IN_PROGRESS request can sit forever (backend §0.4–0.5, §4.1).
- **O-6 — TELS failure handling unverified.** The analysis records the happy path (create → store ids) and inbound webhook, but no retry/reconciliation behavior on outbound failure — treat as an open question for the integration contract (backend §4.2).
- **O-7 — Dead/legacy code.** A static mock `Housekeeping.tsx` weekly-schedule screen exists in the admin repo but is never imported (admin §7); duplicate fetch hooks `use-fetch-housekeeping.tsx` (current) vs `.ts` (legacy) coexist (admin §7); commented-out `residentId`/`residentName` fields linger in the model (backend §8.9).
- **O-8 — Pagination meta absent.** The staff queue endpoint returns no pagination meta; the client infers `hasMore` from page-fullness (staff §8.11).
