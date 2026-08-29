# Backend Functional Analysis — Wellness, Dining, Transportation, Housekeeping, Activities, Announcements/Gallery

> Reverse-engineered from `senior_living_backend` source (code is the source of truth).
> Repo root: `/home/sathish/projects/devicethread/shashi.ai/senior-living/senior_living_backend`
> All paths below are relative to `src/` unless noted. Verified against staging `62de4747` (2026-06-21).

---

## 0. Cross-cutting foundations (shared by every area in scope)

### 0.1 Personas / auth model (`middleware/authMiddleware.ts`, `contants/cognnito.types.ts`)

| Persona | Cognito group / mechanism | Notes |
|---|---|---|
| Resident | `RESIDENT` group, Cognito JWT access token | `req.user.username` = resident `cName` |
| Family member | `FAMILY_MEMBER` group | Auth middleware enriches the request: looks up `FamilyMember.cName` → linked `Resident` and **rewrites `req.user.username` to the resident's cName** (`authMiddleware.ts:95-145`). Original family identity kept as `familyMemberCName`. |
| Staff | `STAFF` group | Per-module read/write **access permissions** stored on `Staff.accessPermissions` (Dashboard, Salon, Housekeeping, Dining, Residents, Services, Transport, Maintenance, Massage Therapy, Rehab, Settings, Access Management). Plus a free-text `designation` (e.g. "Dietitian", "Case Manager") used by booking policy. |
| Admin / Super-admin | `ADMIN`, `SUPER_ADMIN` groups | Bypass staff designation checks (but each booking module must still be configured per facility). |
| TV device | Custom TV JWT + `istv: true` header (`authTv`) | Token verified against `TvAuthToken` collection (facility + cName + deviceId, revocable, expiring). TV requests carry the **resident's cName as username** (`isTv: true`). Issued via Socket.io QR-pairing flow. |

Role gates: `requireAnyRole` / `requireAllRoles` (`middleware/roleMiddleware.ts`); staff page permissions: `requireAnyStaffPermission` (read) / `requireAnyStaffWritePermission` (write) (`middleware/accessPermissionsMiddlewares.ts`). Important: the permission middlewares **no-op for non-STAFF users** — admins pass through on role alone.

### 0.2 Multi-tenancy

`facilityMiddleware` (mounted at `/api` in `app.ts:104`; `/api/fax` mounts before it at `app.ts:103`) reads `x-facility-id` / `facilityid` header into `req.facilityId`. Every query is scoped with `getFacilityFilter(req)`. NOTE (`middleware/facilityMiddleware.ts:40`): the missing-header check tests `facilityId === null || facilityId === ''` after a loop that leaves it `undefined` — so a missing header is **not actually rejected**; queries silently run unscoped (`getFacilityFilter` returns `{}`). All models carry `facilityId: { required: false }`.

### 0.3 On-behalf booking policy — `resolveBookingContext` (`middleware/bookingContextMiddleware.ts`)

Central authorization for "who may create/view a booking for which resident". Applies to: SalonAppointment, MassageAppointment, PrivateTrainingAppointment, TransportationRequest, FamilyMealRequest, Care, RehabAppointment, Housekeeping (keys in `models/config.model.ts` → `bookingPermission`).

Rules:
- **RESIDENT**: always allowed, self-service; any `residentCName` in the request is ignored (spoof guard).
- **FAMILY_MEMBER**: allowed only if facility config `bookingPermission[module].isFamilyMemberAllowed === true`; books for the linked resident.
- **STAFF**: must supply a resident identifier (`residentCName`, legacy `cName`, or `residentId`); allowed only if their `Staff.designation` is in `staffDesignationAllowed` (plus expanded `staffDesignationGroupAllowed` groups, e.g. `rehab`, `supervision` from `constants/designationGroup.ts`).
- **ADMIN/SUPER_ADMIN**: needs the resident identifier and the module to be configured at all; bypasses designation check.
- Output: `req.bookingContext = { createdByType: RESIDENT|FAMILY_MEMBER|STAFF, createdByCName, residentCName }` — persisted on every created document (`createdByType` / `createdByCName` fields).

A parallel, **separate** "case manager" route family (`/case-manager/resident/:residentId/...` on salon, transportation, housekeeping) lets any STAFF act for a resident by resident `_id` — these bypass `resolveBookingContext` and check only the STAFF role.

### 0.4 UnifiedSchedule — the cross-module calendar spine (`models/UnifiedSchedule.model.ts`, `services/scheduleSync.service.ts`)

One denormalized calendar row per resident-facing engagement. `scheduleType ∈ {SALON, MASSAGE, PT, CARE, CARE_CONFERENCE, REHAB, TRANSPORTATION, ACTIVITY}`; `scheduledModel` points to the underlying doc. Sync is done in Mongoose `post('save')` / `post('findOneAndUpdate')` / `post('findOneAndDelete')` hooks on each appointment model (controllers must use `{ new: true }` or the hook writes stale data — documented contract in `scheduleSync.service.ts:22-29`). Sync emits Socket.io events `emitUnifiedScheduleUpserted` / `emitUnifiedScheduleDeleted` (`config/socket.ts`).

Uniqueness (partial indexes): one row per appointment doc, except CareConference (row per attendee) and ACTIVITY joins (row per resident+schedule+date).

**Cross-venue conflict rule** (`services/availableSlots.service.ts:43-51`): types `SALON, MASSAGE, PT, CARE, REHAB` block each other for a resident; statuses counted as "upcoming" = `PENDING, CONFIRMED, Pending, Approved, REQUESTED`. ACTIVITY joins are deliberately excluded from blocking service bookings; activity-vs-activity overlap is checked separately in the join flow. TRANSPORTATION rows are *written* to UnifiedSchedule but not in the blocking set.

`GET /api/unified-schedule` (`controllers/unifiedSchedule.controller.ts`) supports three modes: single day, `upcoming=true` (auth required), and `noOfDays` range; unauthenticated callers (e.g. TV before pairing) get a public payload (meals + activities only via `getPublicScheduleForDate/Range`). `GET /api/unified-schedule/recent-activity` powers an admin "recent activity" feed with humanized actions per type.

### 0.5 Notifications infrastructure

- **Push**: Firebase FCM (`config/firebase.ts`); tokens on `Resident.pushToken`, `FamilyMember.pushToken`, `Staff.pushToken`. Every send is logged to `NotificationHistory` (recipientType RESIDENT/FAMILY/STAFF, scheduleType, scheduleId, title, body).
- **Creation fan-out** (`services/requestCreation.notification.service.ts`): notifies resident + linked family members + either (a) staff holding a page permission (`notifyCreationByPermission`, used by Dining/Housekeeping/Maintenance — `REQUEST_CREATION_PERMISSIONS`) or (b) specifically assigned staff (`notifyCreationForAssignedStaff`, used by salon/massage/PT — the service-owner staff).
- **Reminder cron** (`jobs/notification.cron.ts`, default every minute, TZ `FACILITY_TIMEZONE` env default `America/Los_Angeles`): collects all per-facility configured offsets from `NotificationConfig.modules[].events[].scheduled.offsets` (+ env default `NOTIFICATION_LEAD_MINUTES`, default 15) and for each offset scans `UnifiedSchedule` rows whose start falls in the window. Module mapping: SALON/MASSAGE/PT → module `SALON`, event `APPOINTMENT_REMINDER`; TRANSPORTATION → `TRANSPORT`/`RIDE_REMINDER`; ACTIVITY → `ACTIVITIES`/`ACTIVITY_REMINDER` (`services/notification.service.ts:24-44`). Duplicate-send protection via unique `NotificationSentLog (scheduleId, offsetMinutes)` insert (atomic claim, `notification.service.ts:414-427`). Staff are notified by permission map (SALON→Salon, MASSAGE→Services, PT→Rehab, TRANSPORTATION→Transport); ACTIVITY reminders additionally notify Dashboard + Services staff groups ("review attendance and readiness").
- **Auto-completion cron** (`jobs/appointmentCompletion.cron.ts`, default `*/15 * * * *`): `runAppointmentCompletion` (`services/appointmentCompletion.service.ts`) transitions overdue `CONFIRMED → COMPLETED` for Care, Salon, Massage, PT (and SCHEDULED→COMPLETED for Rehab/CareConference) once `endTime` passes in facility TZ; paged (500), idempotent, per-document `findOneAndUpdate` to keep UnifiedSchedule hooks firing.
- **In-app socket events** (`config/socket.ts`): `notification:new` (per-user room), `new-announcement` (broadcast), `mobile-{module}-request-{action}` via `emitAppRequestEvent` (used by family-meal and housekeeping create/update/delete), unified-schedule upsert/delete events, TV pairing events.

### 0.6 Google Calendar integration (`services/googleCalendarSync.service.ts`, `utils/staffAvailability.ts`)

Per-staff OAuth (Google refresh token on `Staff.googleRefreshToken`, `isGoogleLinked`). When a salon/massage/PT appointment is booked/rescheduled/cancelled, the system CREATE/UPDATE/DELETEs an event on the **service-owner staff member's** primary calendar (fire-and-forget async; result recorded on the appointment as `googleEventId` + `calendarSyncStatus: PENDING|SYNCED|FAILED`). Events carry extended properties (model type, appointment id, facility id) for webhook reconciliation. Conversely, the staff's calendar busy windows are cached on `Staff.cachedCalendarBusySlots` and **subtract slots** from availability (`filterSlotsByStaffBusy`). Webhook-driven refresh uses `GOOGLE_CALENDAR_WEBHOOK_BASE_URL`.

---

## 1. Wellness / personal services — Salon, Massage, Private Training

Three structurally identical modules ("bookable venue" pattern), with copy-paste divergence noted below.

### 1.1 Entities

| Entity | Salon | Massage | Private Training |
|---|---|---|---|
| Venue | `Salon` (name, address, location, contactNumber) | `Massage` (+`isActive`) | `PrivateTraining` (+`isActive`) |
| Service | `SalonService` (name, location, image, **duration min**, price, isActive, `isDeleted`, **`cName` = owning staff**) | `MassageService` (same; `isDeleted`) | `PrivateTrainingService` (same but flag is **`isDelete`** — naming drift) |
| Weekly hours | `SalonSchedule` — one row per (venue, DAY): openTime/closeTime/`isClosed`; unique `(salonId, day)` | `MassageSchedule` (massageId, day) | `PrivateTrainingSchedule` (privateTrainingId, day) |
| Appointment | `SalonAppointment` | `MassageAppointment` | `PrivateTrainingAppointment` (+`notes`) |

Appointment fields (all three): `facilityId, cName (resident), serviceId, venueId?, date (UTC start-of-day), startTime/endTime ("HH:mm")`, `status`, `specialRequest`, `isJoinedWaitList`, `googleEventId`/`calendarSyncStatus`, `createdByType/createdByCName`. Virtual `resident` populated by cName join. Post-save/update/delete hooks sync UnifiedSchedule.

**Status enum (all three): `PENDING | CONFIRMED | COMPLETED | WAITLIST | CANCELLED`** (default PENDING in schema, but booking flow writes CONFIRMED or WAITLIST directly).

Status transitions observed in code:
- Book → `CONFIRMED` (normal) or `WAITLIST` (joinWaitlist) — `salonAppointment.controller.ts:1067`.
- Salon: `WAITLIST → CONFIRMED` only via staff/admin `PATCH /appointments/:id/move` with new time (and optional new date); guarded "only waitlist appointments can be moved" (`salonAppointment.controller.ts:612-626`).
- Massage/PT: staff/admin `PATCH /appointments/:id/status` sets **any** status in the enum — no transition matrix enforced (`massageAppointment.controller.ts:481-525`, `privateTrainingAppointment.controller.ts:508-580`). Salon has no generic status endpoint.
- Cancel (resident/family/staff-on-behalf) → `CANCELLED`; only the owning resident context may cancel ("You can only cancel your own appointment").
- Reschedule allowed from `PENDING/CONFIRMED/WAITLIST` for salon (`salonAppointment.controller.ts:1217`) but only `CONFIRMED/PENDING` for massage/PT (`massageAppointment.controller.ts:909`) — divergence.
- Cron: `CONFIRMED → COMPLETED` once endTime passes (no grace) — §0.5.

### 1.2 Slot generation & booking rules (`services/availableSlots.service.ts` — shared engine)

For a given service + date (single-day or `noOfDays` multi-day):
1. Service must exist and be `isActive` (salon/massage also filter `isDeleted`; **PT lookup uses `findById` without facility or isDelete filter** — `availableSlots.service.ts:180`).
2. Venue weekly schedule for the weekday must exist, not `isClosed`, with open/close times → else "closed" day.
3. Slots generated back-to-back from openTime→closeTime at service `duration` minutes (`lib/slots.ts generateSlots`).
4. **Meal blackout**: slots overlapping facility breakfast/lunch/dinner windows (`Config.meals`) are removed.
5. **Same-service booked slots** removed (statuses PENDING/CONFIRMED, waitlist rows excluded). NOTE: massage & PT compute the booked range as `start + service.duration` (ignores stored endTime); salon uses stored `endTime`.
6. **Staff busy** windows from the owning staff's cached Google Calendar busy slots removed.
7. **Resident cross-venue conflicts** removed — overlap against the resident's UnifiedSchedule rows of types SALON/MASSAGE/PT/CARE/REHAB with upcoming statuses.
8. Same-day slots in the past (`startTime < nowTime()`) removed.
9. `canJoinWaitlist = (no remaining slots) && (baseSlots nonempty)` — i.e., the venue is open and the service would fit, but all times are taken. In the multi-day response, when a day has no available slots the API returns the **baseSlots** (pre-booking template) so the client can offer waitlist times.

Booking (`POST /book-appointment`, all three):
- Normal: requested (start,end) must be one of the currently available slots; then a **second explicit `hasUnifiedScheduleConflict` check** runs (defense in depth) → 400 "You already have another appointment at this time".
- Waitlist (`joinWaitlist: true`): allowed **only when `canJoinWaitlist`** (no free slots); requested time must match a baseSlot; conflict check is skipped; appointment stored with `status: WAITLIST, isJoinedWaitList: true`.
- Venue resolution via `lib/bookableResolution.resolveSalonId` (service's venue or facility default).
- Reschedule re-validates against fresh availability and conflicts, excluding the appointment's own UnifiedSchedule row.

### 1.3 Routes & permissions (representative — salon; massage/PT analogous)

| Operation | Route | Auth |
|---|---|---|
| Book / reschedule / cancel / my-appointments / history / available-slots | `/api/salon/book-appointment` etc. | `authMiddleware` + `resolveBookingContext('SalonAppointment')` — resident, family (if policy), staff/admin on-behalf |
| Staff queue (all appointments for visible services) | `GET /api/salon/appointments` | STAFF/ADMIN + Salon read permission. Staff-only callers see **only appointments for services they own** (`serviceVisibilityFilter.cName = userCName`); admins can filter by staff `cName` |
| Move waitlist → confirmed | `PATCH /appointments/:id/move` | STAFF/ADMIN + Salon **write** permission |
| Massage/PT status update | `PATCH /appointments/:id/status` | STAFF/ADMIN + Massage-Therapy / Rehab write permission |
| Staff "my assigned requests" | `GET /api/salon/salon-assigned-requests` | authMiddleware only (any logged-in user; scoped to services owned by caller) |
| Service CRUD | `POST /services` (STAFF/ADMIN/SUPER_ADMIN + write perm); **`PUT /services/:id`, `PATCH /services/:id/toggle` have NO auth middleware** (`salonService` routes; massage/PT toggle+update also unauthenticated); `DELETE` has authMiddleware only |
| Venue CRUD | `POST/GET/PUT/DELETE /api/salon` — **no auth at all** (`salon.routes.ts`); massage venue delete requires auth, others none |
| Weekly schedule upsert | `POST /schedule/update`, `/schedule/bulk`, `GET /schedule` — **no auth** (all three modules) |
| TV catalog | `GET /api/salon/tv/services`, `/api/massage/tv/services`, `/api/private-training/tv/services` — unauthenticated, used by the TV app |
| Case-manager booking/views (salon only) | `/case-manager/resident/:residentId/...` | STAFF role only (no designation/permission check) |

Permission mapping quirk: **Private Training is gated by the `Rehab` access permission** and massage by `Massage Therapy` (`privateTrainingAppointment.routes.ts:63`, notification map `notification.service.ts:56-67` maps MASSAGE staff notifications to the `Services` permission — three different permission keys across the same flow).

### 1.4 Notifications (salon richest; massage/PT lighter)

- Booking: assigned staff (service owner) push "new appointment/waitlist request"; resident + family get confirmation; admin-broadcast (`notifySalonAppointmentBooked` → admins). Distinct titles for waitlist vs normal (`constants/notificationConstants.ts`).
- Waitlist confirmed: resident push `SALON_WAITLIST_CONFIRMED` + admin notify.
- Cancel / reschedule: resident + service staff + admins.
- Reminder pushes via UnifiedSchedule cron (§0.5).
- Google Calendar create/update/delete for the owning staff (§0.6).

---

## 2. Dining — menu, items, categories, daily specials, menu library, diet plans, family meal requests

### 2.1 Menu composition (`controllers/menu.controller.ts`)

- `Category` (name, isActive, isDeleted, orderKey) → `Item` (categoryId, name, description, picture, isActive/isDeleted, orderKey, availability) — facility-scoped CRUD with reorder endpoints (`PATCH /items/reorder`, `/categories/reorder`).
- **Item availability model** (`isItemAvailable`, `menu.controller.ts:336-382`): `availabilityType ∈ EVERY_DAY | WEEKLY (effectiveDays as day names, optional date window) | MULTIPLE_DATES (effectiveDays as dates) | ONE_TIME (startDate) | DATE_RANGE (start..end)`, with per-date `dateOverrides[{date,isAvailable}]` taking highest priority.
- `GET /api/menu` (auth): single date (default today) or `startDate..endDate` range; assembles per-day menu = ordered categories → available items; picks **one** daily special per day by repeat-pattern priority `one-time(1) > multiple-dates(2) > weekly(3) > date-range(4)`; for residents also returns flattened active **diet plan entries** for the caller.
- `GET /api/menu/getMenuForAdmin` (**no auth**): full menu incl. effectiveDays/dateOverrides + all active specials, search + pagination.
- `GET /api/menu/price-and-time` (**no auth**): facility meal windows, per-meal price, maxGuest, concierge number, lat/lng — consumed by TV/mobile pre-login.

### 2.2 Daily specials & menu library (`controllers/dailySpecial.controller.ts`, `menuLibrary.controller.ts`)

- `MenuLibrary`: uploaded PDF/image menu files (fileName/fileType/fileUrl, usageCount, isActive, optional weekday tags). CRUD + `PATCH /:id/increment-usage`. **No auth on any menu-library, daily-special, item, or category route** (admin-panel endpoints relying on network trust).
- `DailySpecial`: a scheduled menu file with `repeatPattern ∈ one-time | weekly | multiple-dates | date-range`, computed `effectiveDates[]`, weekly `weeklyDay[]` (day names), `startDate/endDate`, `isActive`, unique `(facilityId, name)`.
- Create directly (file upload or `fileKey`, optional `saveToLibrary`/`isSavedToGallary` flags) or **from library** (`POST /from-library`): runs in a Mongo transaction, validates date config, and `resolveEffectiveDateConflicts` **strips overlapping dates from other active specials** (last-write-wins exclusivity per date), increments library usageCount (`dailySpecial.controller.ts:25-60,232-300`).
- `GET /week`: 7-day applicability view via `isSpecialApplicableToDate`.

### 2.3 Diet plans (`controllers/dietPlan.controller.ts`)

- `DietPlan`: resident ref + optional staff ref + entries `[{dietType, dietarySupplements, description}]` + notes + isActive. Created by STAFF/ADMIN (route-gated).
- Visibility rule (changed on staging): **any non-admin staff** see only plans where their cName appears in the resident's `assignedStaff[]` (`filter.resident = { $in: residents where assignedStaff: cName }`, `dietPlan.controller.ts:128-160`); **admins** see all facility plans. This replaces the old Dietitian-designation / `resident.dietitian` scoping (the legacy care-team field is gone — see backend-platform-identity §3.7). `/my-plans` = plans created by the calling staff. `/residents` lists residents (admin: all; staff: only those in their `assignedStaff`, `dietPlan.controller.ts:407`), searchable.
- Delete is ADMIN-only. Diet plan entries surface read-only in the resident menu (§2.1).

### 2.4 Family meal requests (`controllers/familyMealRequest.controller.ts`)

Purpose: resident (or family/staff on-behalf via booking policy) books guest meals (family dining) for a date range.

- Create (`POST /api/family-meal-requests`, bookingContext-gated): mealType BREAKFAST/LUNCH/DINNER; start/end date validated; **blackout dates** from `Config.blackoutDates` reject the whole range; `totalAmount = guests × pricePerPerson × days` (`:126`). `pricePerPerson` and `mealTime` are **trusted from the client** (config holds `mealRates`, `maxGuest`, meal windows — `maxGuest` is returned to clients via `/weekly-meals` and `/price-and-time` but **not enforced server-side**).
- Status model: enum `COMPLETED | PENDING | APPROVED | REJECTED` but **requests default to `COMPLETED`** — model comment says approval statuses are legacy/unused (`familyMealRequest.model.ts:29-31`). A `PATCH /:id/status` endpoint still exists (auth only, no role gate).
- Resident views: `/resident` (current/upcoming — endMealDate ≥ today with mealTime cutoff for today) and `/resident/history` (strictly past), paginated.
- Admin view `GET /` (auth only — resident/family-only callers are scoped to self, **any staff sees all**): returns requests + meal config; auto-creates a default Config (rates 20/25/30, standard meal windows) if missing.
- `GET /weekly-meals` (**no auth**): next-7-days bookable meal config skipping blackout dates.
- Notifications: creation fans out to Dining-permission staff + resident + family (`DINING_REQUEST_CREATED`); socket `mobile-family-meal-request-created/updated/deleted`.
- Update `PUT /:id` passes the raw body to `findOneAndUpdate` (no field whitelist, auth only).

---

## 3. Transportation

### 3.1 Entities & config

- `TransportationRequest` (`models/TransportationRequest.model.ts`): destinationType + `destinationRuleId` → `TransportationRule`; address; `appointmentStartTime` (UTC instant); `estimatedTravelTime` (min); optional `appointmentDuration`, `distance` (miles); computed `pickupTime` (canonical anchor); `startedAt/endedAt`; `travellingWith ∈ alone|with_family|with_caregiver`; `roundTrip`; `isComplimentary` (derived, not client input); `price`, `priceRemarks`; `driverCName` (assigned staff); `createdByType/CName`. Syncs to UnifiedSchedule (type TRANSPORTATION).
- `TransportationRule` (per facility, unique per `locationType`): `isComplimentary` + `complimentaryDistanceLimit` (miles), `isActive`. **Rule routes have no auth** (`transportationRule.routes.ts`). A schema-level validation for the complimentary/limit invariant is commented out (`transportationRule.model.ts:48-66`); the booking controller re-validates it instead.
- Facility config `Config.transportation`: `pricePerMiles`, `pickupBufferMin/Max`, `pickupBufferMultiplier`, `appointmentDurationOptions[]`, optional `MaxMilesForTransport`.

### 3.2 Status machine (`contants/transportation.ts`)

`Pending → Approved (driver assigned) → Started → Completed`, with `Arrived` settable via manage, and terminal `Rejected` / `Cancelled`. Enforced transitions:
- Assign (`POST /:requestId/assign`, ADMIN/STAFF): only from `Pending` or `Approved`; staff can only self-assign (cannot steal another driver's request); admin assigns any active staff by cName; **driver overlap check** — driver must not have another `Approved` request whose `[pickupTime, appointmentStartTime + estimatedTravelTime]` window overlaps (`transportationRequest.controller.ts:1376-1408`). Sets status `Approved`.
- Manage (`PATCH /:id/manage`, ADMIN/STAFF): combined driver + status + price + priceRemarks update. Staff assigning someone **other than themselves** requires their designation to be in `bookingPermission.TransportationRequest.staffDesignationAllowed` (`canStaffManageTransportationAssignments`). Driver assignment implies `Approved` unless an explicit status accompanies it. Status/price applied **as-is** (no transition matrix).
- Start ride (`POST /:id/start-ride`, STAFF): request must be `Approved`, caller must be the assigned driver, driver must not already have a `Started` ride, and start allowed only within **10 minutes before pickupTime** (`:1500-1510`). Sets `Started` + `startedAt`.
- End ride (`POST /:id/end-ride`, STAFF): from `Started` only, assigned driver only → `Completed` + `endedAt`.

### 3.3 Booking flow

`POST /api/resident-transportation/book-transportation` (bookingContext-gated):
1. Rule lookup (active, facility) → complimentary eligibility: `rule.isComplimentary && distance <= complimentaryDistanceLimit` → price 0; else `price = round(distance × pricePerMiles, 2¢)`.
2. `pickupTime = appointmentStartTime − (estimatedTravelTime + clamp(estimatedTravelTime × pickupBufferMultiplier, bufferMin, bufferMax))` (`services/transportationRequest.service.ts:96-117`).
3. Created `Pending`; staff with Transport permission notified.

Distance precheck: `GET /calculate-distance?lat&lng[&appointmentStartTime]` (auth) → **Google Distance Matrix API** road distance/duration from facility lat/lng (`lib/transportation.ts:58-66`); enforces `MaxMilesForTransport` with a user-facing message; optionally returns suggested pickupTime; `NoRouteFoundError` → 400. Note: the booking endpoint itself trusts the client-provided `distance`/`estimatedTravelTime` — the max-miles cap is only enforced in the precheck.

### 3.4 Views

- Staff main screen `GET /staff`: Pending (anyone) + Approved/Started assigned to caller; "current" window derived from `pickupTime + estimatedTravelTime` (ride-end aggregate expression `getTransportationRideEndExpression`). `GET /staff/history`: Completed + Rejected.
- Legacy `GET /` (ADMIN: all; STAFF-only: assigned to them).
- Resident: `/my-requests` (Pending/Approved/Started, future or in-progress), `/my-requests/history`, `/my-requests/list` (flexible filters: pickupDate, status list, upcoming) — all bookingContext-gated so family/staff-on-behalf follow facility policy.
- Case-manager trio by residentId (STAFF role only). `GET /destination-types` and `GET /:id` are **unauthenticated**.

### 3.5 Notifications (`services/transportRequest.notification.service.ts`, 1291 lines)

Event-rich: request created (Transport-permission staff), assignment (driver + resident + previous driver on reassignment), driver arrived, ride started, ride completed, ride cancelled/rejected, generic "request updated" (status/price changes). Resident + family + transport staff each get tailored bodies; `RIDE_REMINDER` offsets via cron (§0.5).

---

## 4. Housekeeping & Maintenance (single module, `requestType` discriminator)

### 4.1 Entity (`models/houseKeeping.model.ts` — model name `ServiceRequest`)

`requestType ∈ EXTRA_ROOM_CLEANING | EXTRA_LAUNDRY | MISC | MAINTENANCE`; `priority LOW/MEDIUM/HIGH (default MEDIUM)`; `requestCode` (unique, generated `R<last-4-of-epoch-ms>` — collision-prone, `housekeeping.controller.ts:942`); `unitNo`; `dateRequested` + `selectedDate`; `status PENDING → IN_PROGRESS → COMPLETED | REJECTED | CANCELLED` (no transition matrix enforced — update endpoint sets whatever is sent); `assignedTo` (Staff ref); image (S3, for MISC/MAINTENANCE); `categoryId/categoryName` + `scheduledTime` + `hasPermissionToEnter`; TELS back-links `telsWorkOrderId/telsJobId/telsScheduleId`; `createdByType/CName`. Housekeeping does **not** sync to UnifiedSchedule.

### 4.2 Flows

- Create (`POST /api/housekeeping`, bookingContext `Housekeeping`): validates type; **duplicate guard** — one EXTRA_ROOM_CLEANING / EXTRA_LAUNDRY per resident per selectedDate ("Can not add multiple request for same date", `:921-939`); MISC/MAINTENANCE accept an image.
- **TELS PMS sync** (`integrations/pms/workOrder.integration.ts` + `integrations/pms/tels/tels.service.ts`): if `Config.integratedModules.MAINTENANCE === 'TELS'`, MAINTENANCE requests create a TELS **work order**; if `integratedModules.HOUSEKEEPING === 'TELS'`, cleaning/laundry/misc create a TELS **housekeeping job**; ids stored back. Status mapping helpers (SAL priority ↔ TELS 1-3, status code mapping). Inbound `/webhooks/tels` (HMAC-auth middleware) updates request status from TELS. `GET /categories` returns TELS work-order categories for the facility.
- Update (`PUT /api/housekeeping`, auth only — **no role check**): status/assignedTo/remarks/rejectedReason/completedAt; if a STAFF user updates without `assignedTo`, they are **auto-assigned** (`:1076-1090`).
- Views: resident upcoming (`/resident` — selectedDate ≥ today, PENDING/IN_PROGRESS) and history (`/resident/history` — completed/rejected past + all cancelled), both bookingContext-gated (page-permission check deliberately commented out so Case Manager/Social Worker designations allowed by booking policy aren't blocked — route comment `housekeeping.routes.ts:48-53`); admin list `GET /housekeeping-admin` (**no auth**, paginated); staff queue `GET /housekeeping-staff` (auth; `isMaintenance` query splits MAINTENANCE vs non-MAINTENANCE views); case-manager trio (STAFF role).
- Notifications: separate maintenance vs housekeeping flavors for created / assigned / in-progress / completed (`services/serviceRequest.notification.service.ts` via controller imports); socket `mobile-housekeeping-request-*`. Recipients: permission-based staff (Housekeeping or Maintenance) + resident + family.

---

## 5. Activities / events scheduling (Schedule, UnifiedSchedule joins, attendance, brain games)

### 5.1 Activity templates (`models/schedule.model.ts`, `controllers/schedule.controller.ts`)

`Schedule` = recurring activity definition: name, owner staff `cName`, **capacity (required, min 1)**, `repeatPattern ∈ one-time | weekly | multiple-dates | date-range | everyday`, materialized `effectiveDates[]` (empty for everyday), `weeklyDay` (0-6) and/or `days[]` (MONDAY…), `isAllDays`, startDate/endDate window, startTime/endTime, location, description, image (S3 + optional gallery save), isActive.

- Create (auth; any role): computes effectiveDates via `utils/dateCalculations.calculateEffectiveDates`; weekly supports single `weeklyDay` or multi-`days[]` fallback (`schedule.controller.ts:56-102`).
- List `GET /` (auth): staff-only callers see **only their own** activities; residents/family see active ones; optional date filters by eligibility clauses; for residents each item gets `joined` (has a JOINED UnifiedSchedule row for that date) and `canJoin` (start instant still in future — `lib/scheduleActivityJoinWindow`).
- TV list `GET /tv` (optional auth): N-day grouped view (default 7); when a TV session is authenticated, includes per-day `joined`/`canJoin` for the paired resident.
- Update/Delete: **no auth middleware** (`schedule.routes.ts:39-40`). Update recomputes effectiveDates, swaps S3 image; fires `notifyActivityChanged` to joined residents. Delete fires `notifyActivityCancelled` to joined residents then removes all join rows.

### 5.2 Join / cancel (resident-facing state machine)

`POST /:id/join` (auth): validate active schedule + date eligibility (`isScheduleEligibleForDay`) → join-window open (activity not started) → **cross-venue conflict** check (`hasUnifiedScheduleConflict` — services block joining) → **activity-vs-activity overlap** check among the resident's JOINED rows that date → **capacity check** (count of JOINED rows for schedule+date ≥ capacity → 409 "This activity is full") → idempotent (already joined → 200) → create UnifiedSchedule row `{scheduleType: ACTIVITY, scheduledModel: Schedule, status: 'JOINED', createdByFamilyMember?, createdBy}` (`schedule.controller.ts:646-819`). Family members join on behalf of their resident (flagged).
`POST /:id/cancel`: same join-window rule (cannot cancel after start); deletes the join row.

Note asymmetry: activities consume service-appointment conflicts when joining, but service bookings ignore ACTIVITY rows (§0.4) — an activity join does not block a later salon booking at the same time.

### 5.3 Attendance (`models/ScheduleAttendance.model.ts`, `services/scheduleAttendance.service.ts`, routes `/api/schedule-attendance`)

Staff/Admin-only (role-gated all three endpoints). One doc per `(facility, scheduleId, scheduleDate)`; embedded `attendees[{cName, joined, status NOT_MARKED|PRESENT|ABSENT, markedAt, markedBy}]`.
- `GET /schedules?date=` — activities occurring that day (pick list).
- `GET /:scheduleId?date=` — roster = all JOINED residents (seeded NOT_MARKED) merged with any manually-added non-joined residents, with summary counts (totalResidents = joined count; present/absent/notMarked).
- `POST /:scheduleId/mark?date=` — upsert statuses; supports `markAllPresent` / `markAllAbsent` (joined residents only) or explicit attendee list (may include non-joined residents). Attendance is intentionally decoupled from the join lifecycle. Feeds the activity attendance report (`activityAttendanceReport.controller.ts`, out of scope here).

### 5.4 Brain games (`models/BrainGame.model.ts`, routes `/api/brain-games`)

Curated catalog of cognitive-game apps for residents: name, iconUrl, App Store / Play Store URLs, categories[], rating, isActive, sortOrder. CRUD (auth only, no role gate — any resident token could create); list = active only, text-search with score ranking, paginated. **No facilityId field — brain games are global across facilities.** No tracking of play/usage server-side; deep links are consumed by mobile/TV clients.

---

## 6. Announcements & Gallery

### 6.1 Announcements (`models/announcement.model.ts`, `controllers/announcement.controller.ts`, `jobs/announcement.cron.ts`)

- Fields: title, description, `iconType`, `createdBy` (Admin ref), **`type ∈ single | multiple | range`** (legacy null treated as single/range), **`audience ∈ family | resident | both`** (default both), startDate/endDate or `selectedDates[]`, **`startTime`/`endTime` time-of-day fields (staging)** driving a new 1-hour-before reminder, soft-delete `deletedAt`, notification bookkeeping (`notificationDates[]` per-day sent record incl. `${date}-reminder` keys, `notificationSentAt`, `notificationProcessingAt` claim lock).
- Routes (**no auth on any**): create/list/get/update/delete + `GET /past` (paginated archive). List supports "active today" vs "today + future" windows built from UTC-midnight day bounds keyed to the facility-TZ calendar date.
- Real-time: on create/update where the announcement is active "today" (today checked in **both Asia/Kolkata and America/Los_Angeles** — hardcoded dual-TZ heuristic, `announcement.controller.ts:56-66`), emits socket `new-announcement` broadcast.
- Push: cron now runs **`*/10 * * * *`** (was `0 8 * * *` daily; override `ANNOUNCEMENT_NOTIFICATION_CRON_SCHEDULE`, gated by its own `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON`) — `startAnnouncementNotificationCron` finds non-deleted announcements active today not yet notified for the date, atomically claims via `notificationProcessingAt`, collects resident and/or family push tokens per `audience`, multicasts in 500-token chunks, then appends the date to `notificationDates`; it now also sends **1-hour-before** and **1-day-before** reminders. A second per-minute `startAnnouncementReminderCron` (`* * * * *`) sends the "1 hour before" reminder for announcements carrying a `startTime` (dedup key `${date}-reminder`). The two functions live in `announcement.cron.ts` and produce 2 of the 6 cron starts wired in `server.ts`.

### 6.2 Gallery (`models/galleryImage.model.ts`, `services/gallery.service.ts`)

Lightweight shared image library: `GalleryImage{imageKey}` (**no facilityId — global**). Populated as a side effect: schedule images, daily-special files, housekeeping images pass `isSavedToGallary: true` (misspelled flag) → `saveToGalleryIfFlagged` stores the S3 key. `GET /api/gallery` (auth) lists images (optional `folder` prefix filter via regex) with signed URLs; handles legacy docs storing full `imageUrl`. No delete/update endpoints — write-only accumulation.

---

## 7. Product-split signals (Senior Living vs Skilled Nursing)

1. **Per-resident `careType`**, not per-facility: `Resident.careType ∈ assisted_living | memory_care | independent_living | skilled_nursing` (`models/resident.model.ts:9,115`; option list `contants/index.ts`). Surfaces in housekeeping list payloads (`housekeeping.controller.ts:333` etc.) and resident views — clients render mixed-acuity facilities from one backend.
2. **Rehab is the main split point** (adjacent to scope): `constants/rehab.ts:38-75` — assisted-living uses 4 fixed therapy types with per-type config in `Config.rehab`; skilled-nursing uses dynamic `RehabTherapy` rows via `therapyId` (`THERAPY_TYPES.OTHER`). PT (this scope) is the AL "private training" wellness flavor and is permission-keyed to `Rehab`.
3. **Per-facility module-to-PMS switchboard**: `Config.integratedModules` maps SALON/MASSAGE/PT/CARE/TRANSPORTATION/ACTIVITY/DINING/HOUSEKEEPING/MAINTENANCE → provider (`OPERA | POINTCLICKCARE | YARDI | TELS | CUSTOM`) with `Config.pms[]` credentials (`config.model.ts`). Only HOUSEKEEPING/MAINTENANCE→TELS is implemented today (`workOrder.integration.ts`); PCC webhooks serve the clinical side (medication/patient) — a strong skilled-nursing signal. `IntegrationAvailable` rows store PCC org/facility credentials.
4. **Per-facility feature shaping, not flags**: `Config.accessPages` (hidden/ranked nav incl. children), `bookingPermission` per module, `NotificationConfig.modules[].events[]` toggles, inactivity timeouts, designations list — the same binary is configured per facility rather than per product SKU.
5. **Client app fingerprints**: TV consumes unauthenticated `/tv/...` catalog routes (salon/massage/PT services, `/api/schedules/tv`, menu price-and-time) + TV-token flows; staff app consumes `/staff`, `/housekeeping-staff`, `salon-assigned-requests`, attendance, designation-driven case-manager routes; resident/family app consumes `my-*`/bookingContext routes; admin panel consumes the unauthenticated `*-admin`/CRUD routes (`getMenuForAdmin`, `housekeeping-admin`, announcements, rules). Skilled-nursing-only client surface (rehab/PCC/referrals/IDT) lives outside this scope but shares the same Express app.

---

## 8. Observations (gap-analysis seeds)

**Security / auth gaps (recurring pattern: admin-panel CRUD routes lack middleware):**
1. No auth: salon venue CRUD, salon/massage/PT weekly-schedule upsert (incl. bulk), salon/massage/PT service update + toggle, transportation-rule CRUD, announcements CRUD, menu/category/item/daily-special/menu-library CRUD, `GET /housekeeping-admin`, `GET /menu/getMenuForAdmin`, `GET /resident-transportation/:id`, `/destination-types`, `/family-meal-requests/weekly-meals`.
2. `facilityMiddleware` missing-header check is dead code (`undefined` never matches `null/''`) — unscoped cross-tenant queries possible (`facilityMiddleware.ts:31-46`); all schemas keep `facilityId` optional.
3. Brain game CRUD requires only a valid token (resident could mutate the global catalog); gallery + brain games have no facility scoping at all.
4. Family-meal `PUT /:id` applies the raw request body as a Mongo update (operator injection risk; no field whitelist); housekeeping `PUT` has no role gate.
5. Client-trusted pricing: family-meal `pricePerPerson` and transportation `distance`/`estimatedTravelTime` come from the client; `maxGuest` and `MaxMilesForTransport` are advertised but only the latter is enforced (and only in the precheck endpoint).

**Consistency drift (salon vs massage vs PT triplication):**
6. `availableSlots.service.ts` triplicates the engine with subtle differences: PT service lookup skips facility + soft-delete filters; massage/PT derive booked end from duration vs salon's stored endTime; PT flag `isDelete` vs `isDeleted` elsewhere.
7. Reschedule-allowed statuses differ (salon allows WAITLIST; massage/PT don't); only salon has a waitlist-promotion endpoint; massage/PT free-form status PATCH has no transition matrix; family-meal status enum is effectively dead (default COMPLETED).
8. Permission-key mismatch: PT gated by `Rehab`, massage queue by `Massage Therapy` but massage reminders go to `Services` staff.

**Dead/legacy code & TODOs:**
9. Commented-out schema validation in `transportationRule.model.ts`; commented-out `requireAnyStaffPermission` in housekeeping resident routes (intentional, documented); commented residentId/residentName fields in housekeeping model; duplicated `isFamilyMemberOnlyRequest` block pasted 3× in `familyMealRequest.controller.ts:205-225`; debug `console.log`s in meal-duration, transportation booking, TELS config, housekeeping create.
10. `appointmentCompletion.service.ts` carries two `TODO(for now)` notes about per-document updates vs bulk + reconciliation.
11. `notificationSentAt` on UnifiedSchedule appears superseded by `NotificationSentLog` claims.

**Functional quirks:**
12. Housekeeping `requestCode = 'R' + last 4 digits of Date.now()` — collisions on a unique index will throw raw 500s.
13. Salon `getAppointments` date filter applies `startTime >= nowTime()` to **every** future date, hiding earlier-time appointments on later days (`salonAppointment.controller.ts:269-273`).
14. Announcement "is today" socket check hardcodes IST + LA timezones (`announcement.controller.ts:56-66`) — developer-locale artifact.
15. Activity joins don't block service bookings (one-way conflict rule) — double-booking possible activity-side; capacity check is read-then-write (no atomic guard), so concurrent joins can exceed capacity.
16. `FACILITY_TIMEZONE` is a process-wide env var used in cron TZ and slot math — multi-facility deployments across timezones are not actually supported despite `Config.timeZone` existing.
17. Daily-special exclusivity silently edits sibling specials' effectiveDates (last-write-wins) — surprising for admins.
18. Family-meal default Config auto-creation embeds hardcoded rates (20/25/30) in a GET handler (side-effectful read).
