# Module: Announcements, Gallery & Notification Platform

> Applies to: Both
> FR prefix: ANN
> Sources: `_codebase-analysis/backend-platform-identity.md` (§5 Notifications platform, §5.3 cron), `_codebase-analysis/backend-wellness-dining-ops.md` (§0.5, §6 Announcements & Gallery), `_codebase-analysis/client-admin-web.md` (§2.13, §2.14b, §2.17), `_codebase-analysis/client-resident-app-sl.md` (§3.2, §3.13, §4), `_codebase-analysis/client-resident-app-sn.md` (§3.8, §3.10, §4), `_codebase-analysis/client-tv-app.md` (§3.1, §3.2, §3.8). Code is source of truth.

---

## 1. Purpose & scope

This module covers three intertwined platform capabilities shared by every other module:

1. **Announcements** — facility-wide, time-windowed broadcasts authored by admins and delivered to residents and/or family members via mobile feeds, the TV Alert tab, in-app socket events, and a daily push-notification cron.
2. **Photo gallery** — two distinct image stores that share branding under "gallery":
   - the **facility media library** (`GalleryImage`) — a write-only accumulation of S3 keys reused across admin modules (menu items, daily specials, activities, announcements, rehab services) via the shared image picker;
   - **resident TV photos** — per-resident family photos uploaded from the SN resident app and displayed on the in-room TV photo frame.
3. **Notification platform** — the cross-module delivery engine: three channels (FCM push, Socket.io in-app, persistent `NotificationHistory` feed), a per-facility `NotificationConfig` event catalog (~40 events across 7 modules), a scheduled-reminder pipeline with offsets and atomic dedupe, and 4 cron jobs.

This module **is** the notifications module — domain modules (dining, transport, salon, rehab, etc.) define *when* events fire; this module defines *how* they are configured, deduplicated, fanned out, and consumed by each client.

Out of scope: chat messaging and its local notifications (Messaging module), the per-domain event trigger logic (owning modules), TV pairing (TV Platform module).

---

## 2. Personas & surfaces

| Persona | Surface | Capabilities in this module |
|---|---|---|
| Facility admin | Admin web — Announcements page | Create/edit/soft-delete announcements; choose display type, audience, icon-or-image identity; view past announcements |
| Facility admin | Admin web — NotificationPanel (bell) | Combined feed: today's announcements + Redux-stored in-session notifications; local-only read state |
| Facility admin | Admin web — Settings → Notification Preferences | 5 toggles — **decorative, localStorage only, no backend sync** |
| Facility admin | Backend `/api/notification-config` | CRUD per-facility event/offset configuration (no dedicated admin UI observed) |
| Resident (SL app, "Shashi Care") | Announcements screen | Paginated read-only announcement feed (LIVE) |
| Resident (SL app) | Notifications screen | NotificationsScreen still **MOCK** (single hardcoded item), but FCM is now wired for **foreground** local-notification display + token harvest on staging (ANN-FR-24); background/quit routing absent (§9) |
| Resident (SN app, "SkilledNursing") | Announcements screen | Paginated announcement feed from Home preview |
| Resident (SN app) | Notifications center | Paginated `NotificationHistory` feed, date-grouped; `isRead` rendered but never mutated; no tap-through |
| Resident (SN app) | Upload Pictures (Profile) | Manage TV photo-frame gallery: list / upload / delete resident pictures |
| Resident / family (FCM) | Push notifications | Reminders, event notifications, daily announcement broadcast |
| Family member | FCM + history | Receives family-audience announcements and family-specific reminder copy via the resident's linked `FamilyMember` records |
| Resident (TV app) | Alert tab | Pull-only paginated feed (10/page); view-only cards |
| Resident (TV app) | Photos tab / ambient slider | Resident pictures gallery; generic photo-frame slideshow from facility config |
| Staff | FCM + socket + history | Event notifications filtered by access-permission; web-panel-only staff (no push token) still get history rows + socket emits |

---

## 3. Functional requirements (as-built)

### Announcements

- **ANN-FR-01 — Create/manage announcements.** Admin web provides CRUD on announcements with: `title*`, `description*`, **displayType** `single | multiple | range`, **audience** `resident | family | both` (default both), optional time-of-day **`startTime`/`endTime`** (new on staging; `startTime` drives the 1-hour-before reminder cron, ANN-FR-06), and a visual identity. Soft delete via `deletedAt`. Endpoints: `GET /announcements?page&limit&startDate&endDate`, `GET /announcements/past`, `POST /announcements`, `PUT /{id}` (multipart), `DELETE /{id}` (soft). The list query's `startDate`/`endDate` accept **either** `YYYY-MM-DD` **or** a full ISO datetime (e.g. `2026-06-15T00:00:00.000Z`): a value containing `T` is normalized to its `YYYY-MM-DD` date portion before windowing; invalid values are rejected ("date must be YYYY-MM-DD or a valid ISO datetime"). [admin §2.13; backend-wellness §6.1; `validation/announcement.schema.ts:54-71`]
- **ANN-FR-02 — Display-type date semantics.**
  - `single`: one date ≥ today (UI copies it to endDate on submit);
  - `multiple`: ≥1 discrete `selectedDates[]` chosen on a multi-date calendar, serialized `YYYY-MM-DD`;
  - `range`: startDate + endDate, both ≥ today, end ≥ start. Legacy rows with `type: null` are treated as single/range. [admin §2.13; backend-wellness §6.1]
- **ANN-FR-03 — Icon-vs-image identity.** An announcement's visual is **either** one of 16 preset icons (`{name, lucide icon, hex color, public image path}` — on save the icon is fetched and converted to a File via `createIconFile()`) **or** an uploaded/gallery image following the platform `imageKey`/signed-URL contract; plus a `color` hex. Validation: must have a file OR imageKey OR icon. [admin §2.13]
- **ANN-FR-04 — Immediate publish, no draft state.** Creating an announcement publishes it; "active" = today within the date window. Expired announcements move to the **Past** archive (`GET /announcements/past`, paginated). List sorted by `updatedAt` desc; description truncates at 150 chars in list view. [admin §2.13]
- **ANN-FR-05 — Real-time socket emit on create/update.** When a created/updated announcement is active "today", the backend broadcasts socket event `new-announcement` to **all connected clients** on the default namespace. "Today" is checked in **both Asia/Kolkata and America/Los_Angeles** (hardcoded dual-TZ heuristic). The admin web listens; resident apps and TV do **not** consume this event. [backend-wellness §6.1; backend-platform §8.4; tv §4]
- **ANN-FR-06 — Announcement broadcast cron (overhauled on staging).** `announcement.cron` now runs on a **`*/10 * * * *`** default schedule (overridable via `ANNOUNCEMENT_NOTIFICATION_CRON_SCHEDULE`) under its own gate `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` (defaults on; `!== 'false'`) — separate from the master `ENABLE_NOTIFICATION_CRON`. It finds non-deleted announcements active today and not yet notified for the date, **atomically claims** each via `notificationProcessingAt` lock, collects resident and/or family push tokens per `audience`, sends FCM multicast in **500-token chunks** with payload `{type: 'ANNOUNCEMENT', announcementId}`, then appends the date to `notificationDates[]` — the per-date send ledger that dedupes across ticks and overlapping workers. Beyond the same-day broadcast it now also sends a **1-day-before** reminder (keyed `{tomorrow}-dayBefore` in `notificationDates[]`) and, via the new per-minute `startAnnouncementReminderCron`, a **1-hour-before** reminder for announcements carrying a `startTime`. (`jobs/announcement.cron.ts:25-122`)
- **ANN-FR-07 — Resident app announcement feeds.** Both resident apps: Home shows a latest-announcement preview (icon by `iconType`: weather/event/menu/broadcast default) → AnnouncementsScreen with `GET /api/announcements?page&limit` (10/page), infinite scroll, pull-to-refresh. Read-only: no detail screen, no read/unread state. [sl §3.1–3.2; sn §3.1]
- **ANN-FR-08 — TV Alert tab.** Pull-only paged list (`GET notifications?page&limit`, 10/page, Jetpack Paging 3), refetched on every tab entry. View-only cards (title/description/date); no mark-as-read, no acknowledgement, no deep-link, no push/socket channel — a resident sees new items only when opening the tab. [tv §3.2, §4]
- **ANN-FR-09 — Admin NotificationPanel announcement section.** The bell panel queries today's announcements (today's date range) live and shows them as a collapsible section alongside the in-session notification feed. [admin §2.13, §2.14b]

### Gallery & photos

- **ANN-FR-10 — Facility media library (`GalleryImage`).** Global image store of S3 `imageKey`s (no facilityId — **global across facilities**). Populated only as a side effect: activity images, daily-special files, menu items, announcements, and housekeeping images carry `isSavedToGallary: true` (misspelling is the wire contract) → `saveToGalleryIfFlagged` persists the key. `GET /api/gallery?folder=` (auth) lists with signed URLs and an optional folder-prefix regex filter; legacy docs storing a full `imageUrl` are handled. **No delete/update endpoints — write-only accumulation.** [backend-wellness §6.2]
- **ANN-FR-11 — Shared image picker (admin).** `ImageSelectionModal` is the single picker used by Dining, Salon/wellness, Rehab Service, and Announcements: "Upload from Computer" (drag-drop, optional **Save to Gallery** checkbox) or "Select from Gallery" (grid via `GET /gallery?folder=`, per-module folders e.g. `items`, `daily-specials`). Platform contract: clients send the S3 `imageKey`, render only the signed `imageUrl`, never send the signed URL back. [admin §2.4]
- **ANN-FR-12 — Resident TV photos (SN app).** Profile → Upload Pictures manages the in-room TV photo frame: `GET / POST / DELETE /api/residents/pictures` (S3-backed, per-resident). [sn §3.8; backend-platform §4]
- **ANN-FR-13 — TV photo display.** TV Photos tab fetches `GET residents/pictures` for the paired resident (refetched on each tab entry): main image + thumbnail strip, full-screen view with Ken Burns animation, empty state "No photos found"; view-only. The ambient home slider separately cross-fades generic photo-frame assets from facility config (`familyPhotoFrame_Default_without_sample_text`, 30 s/image, 2 s fade); `photoFrameWithoutText` viewType reuses the Photos UI for those assets. [tv §3.1, §3.8]

### Notification platform

- **ANN-FR-14 — Three delivery channels.**
  1. **FCM push** via firebase-admin (initialized from `FIREBASE_*` secrets; **silently disabled if missing**);
  2. **In-app socket** — namespace `/notifications`, Cognito-authenticated, per-user room `user:<cName>`, event `notification:new`;
  3. **Persistent history** — `NotificationHistory` rows (recipientType `RESIDENT|STAFF|FAMILY`, recipientCName, scheduleType/scheduleId, title, body, isRead/isDeleted) read via `GET /api/notifications` (auth, paginated). Every send writes history; staff without a push token (web-panel-only) still get history rows + socket emits. [backend-platform §5.1]
- **ANN-FR-15 — Per-facility NotificationConfig.** One `NotificationConfig` doc per facility: modules → events → `{immediate: bool, scheduled: {enabled, offsets[]}}`. Lazily seeded from `DEFAULT_NOTIFICATION_MODULES` on first read. Admin CRUD via `/api/notification-config`. Default reminder offsets: **20 min / 1 h / 1 day**. [backend-platform §5.2]
- **ANN-FR-16 — Event catalog (~40 events, 7 modules).** Summary of the default catalogue (\* = scheduled reminder enabled by default):

  | Module | Events |
  |---|---|
  | TRANSPORT | RIDE_REQUESTED, RIDE_ASSIGNED, RIDE_REMINDER\*, DRIVER_ARRIVED, TRIP_STARTED, TRIP_COMPLETED, RIDE_CANCELLED |
  | MAINTENANCE | REQUEST_CREATED, REQUEST_ASSIGNED, WORK_IN_PROGRESS, REQUEST_COMPLETED |
  | HOUSEKEEPING | CLEANING_REQUEST_RAISED, REQUEST_ASSIGNED, CLEANING_IN_PROGRESS, COMPLETED |
  | ACTIVITIES | ACTIVITY_SCHEDULED, ACTIVITY_REMINDER\*, ACTIVITY_CHANGED, ACTIVITY_CANCELLED, ACTIVITY_COMPLETED |
  | SALON | APPOINTMENT_BOOKED, APPOINTMENT_CONFIRMED, WAITLIST_TO_CONFIRMED, APPOINTMENT_REMINDER\*, APPOINTMENT_COMPLETED, APPOINTMENT_CANCELLED |
  | HEALTH_CARE | CREATE_REHAB_APPOINTMENTS, REHAB_APPOINTMENT_REMINDER\*, REHAB_APPOINTMENT_COMPLETED/CANCELLED, CARE_CONFERENCE_SCHEDULE/REMINDER\*/COMPLETED/CANCELLED, CREATE_IDT_REPORT, IDT_REPORT_REMINDER (1d only), IDT_REPORT_SUBMISSION |
  | DINING | MEAL_ORDER_PLACED, MEAL_ORDER_CONFIRMED, MEAL_READY, MEAL_DELIVERED, ORDER_CANCELLED |

  ~13 dedicated `*.notification.service.ts` files implement immediate sends per domain (salon, massage, PT, rehab, transport, care conference, IDT report, referral, service requests, chat, announcements, activity). [backend-platform §5.2]
- **ANN-FR-17 — Scheduled reminder pipeline.** Source of truth is `UnifiedSchedule` (mirror of all bookable items: SALON, MASSAGE, PT, CARE, CARE_CONFERENCE, REHAB, TRANSPORTATION, ACTIVITY). Per offset, a window query selects rows whose `scheduleDate`+`startTime` falls in `[now+offset, now+offset+NOTIFICATION_WINDOW_MINUTES (1)]`, excluding CANCELLED, capped at `NOTIFICATION_MAX_PER_RUN` (200) per tick. The offset must be enabled in that facility's config for the mapped module/event (SALON/MASSAGE/PT → SALON/APPOINTMENT_REMINDER; TRANSPORTATION → TRANSPORT/RIDE_REMINDER; ACTIVITY → ACTIVITIES/ACTIVITY_REMINDER; etc.). [backend-platform §5.4]
- **ANN-FR-18 — Send dedupe (`NotificationSentLog`).** Unique `{scheduleId, offsetMinutes}` index; the insert acts as an **atomic claim** across overlapping cron workers — exactly-once per appointment per offset. [backend-platform §5.4]
- **ANN-FR-19 — Recipient fan-out.** Per reminder: resident (FCM + history) → that resident's family members (multicast with family-specific copy) → staff filtered by access permission per service (`SERVICE_PERMISSION_MAP`: SALON→Salon, MASSAGE/CARE/CARE_CONFERENCE→Services, PT→Rehab, TRANSPORTATION→Transport) → secondary staff groups with bespoke copy (e.g. Dashboard+Services for ACTIVITY). Staff matching uses `$elemMatch` on `accessPermissions`. Creation events fan out either by page permission (Dining/Housekeeping/Maintenance) or to specifically assigned staff (salon/massage/PT service owners). [backend-platform §5.4; backend-wellness §0.5]
- **ANN-FR-20 — Push token lifecycle.** FCM tokens are upserted opportunistically from the `x-fcm-token` header on profile/config fetches (`Resident.pushToken`, `FamilyMember.pushToken`, `Staff.pushToken`). The header-driven upsert (`upsertPushTokenForUserFromHeader`) now branches on **ADMIN/SUPER_ADMIN** in addition to STAFF/RESIDENT/FAMILY: an admin caller's token is written to the `Admin` document **and** to a matching `Staff` document with the same `cName`, so staff-targeted sends — notably assigned-care-team notifications (CARE-FR-63) — reach a care manager logged in via an admin account; previously admin-group users had no branch and their tokens were never stored. **Logout = clearing the stored pushToken** (`/api/residents/logout`, `/api/staff/logout`). The SN app requests permission + token at Splash and sends it as `pushToken` and `x-fcm-token` headers on **every** request — no explicit register-device endpoint. [backend-platform §3.3, §5.1; sn §4; `lib/pushToken.ts:48-82`]
- **ANN-FR-21 — User notification preferences.** Per-user toggles limited to keys `DINING, SALON, TRANSPORT, HOUSE_KEEPING, REHAB`; a missing key = ON. Residents and staff have GET/PUT endpoints; staff additionally carry a global `notificationStatus` boolean. (No client currently persists these — see §9.) [backend-platform §4]

### Cron jobs

- **ANN-FR-22 — Cron suite (staging: 6 cron starts from 5 job files).** Crons start at bootstrap (gated by the master `ENABLE_NOTIFICATION_CRON` and each job's own `ENABLE_*` kill-switch); each has a `*_CRON_SCHEDULE` override; timezone from `FACILITY_TIMEZONE` env (default America/Los_Angeles). `announcement.cron.ts` now exports **two** start functions, and a care-conference **review** cron was added:

  | Job (start fn) | Default schedule | Gate | Function |
  |---|---|---|---|
  | `notification.cron` | `* * * * *` | `ENABLE_NOTIFICATION_CRON` | Collects all distinct scheduled offsets across every facility's NotificationConfig (+ env default `NOTIFICATION_LEAD_MINUTES`, 15) and runs `sendScheduledNotificationsForOffset(offset)` per offset |
  | `appointmentCompletion.cron` | `*/15 * * * *` | `ENABLE_APPOINTMENT_COMPLETION_CRON` | Auto-completes overdue appointments: `CONFIRMED → COMPLETED` for Care/Salon/Massage/PT, `SCHEDULED → COMPLETED` for Rehab/CareConference once endTime passes in facility TZ; paged (500), idempotent, per-document updates so UnifiedSchedule hooks fire |
  | `announcement.cron` → `startAnnouncementNotificationCron` | `*/10 * * * *` | `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` | Announcement broadcast + 1-day-before reminder (ANN-FR-06) |
  | `announcement.cron` → `startAnnouncementReminderCron` | `* * * * *` | `ENABLE_ANNOUNCEMENT_NOTIFICATION_CRON` | Per-minute 1-hour-before reminder for announcements with a `startTime` (ANN-FR-06) |
  | `careConferenceEnable.cron` | `* * * * *` | `ENABLE_CARE_CONFERENCE_ENABLE_CRON` | Enables care conferences starting within 5 min (`isEnabled=true`, status → IN_PROGRESS) and notifies residents + care team |
  | `careConferenceReview.cron` | `* * * * *` | `ENABLE_CARE_CONFERENCE_REVIEW_CRON` | Care-conference review-pipeline cron (per-minute) |

  (`server.ts:77-82`; `jobs/announcement.cron.ts`; `jobs/careConferenceReview.cron.ts`) [backend-platform §5.3; backend-wellness §0.5]

### Client-side notification handling (as-built state)

- **ANN-FR-23 — SN app handling.** Foreground FCM messages are displayed generically via notifee (channel `default_notification_channel_id`) — **no type switch, no navigation**. The Notifications center (`GET /api/notifications`) renders `isRead` but never mutates it; no tap-through. Chat local notifications (socket-driven, not FCM) are the only notifications that deep-navigate, and only in foreground — no background/quit-state routing, no URL scheme. [sn §3.10, §4, §7.9]
- **ANN-FR-24 — SL app handling (staging: push now wired).** The SL resident app (`senior_living_reactnative`) now requests notification permission and fetches the FCM token at Splash (`messaging().requestPermission()` + `getToken()`), and its Axios interceptor attaches both `pushToken` and `x-fcm-token` headers on every request (`services/Api/index.tsx:82-83`) — so the backend's header-driven token upsert (ANN-FR-20) now reaches SL-app users. Foreground FCM messages are rendered as local notifications via `@notifee/react-native` (`setupForegroundNotificationListener`, channel `default_notification_channel_id`, mounted in `App.tsx`). **Still absent:** background/quit-state tap routing, type-switched navigation, and deep links; NotificationsScreen remains a hardcoded mock and per-category toggles are local-only placeholders. The interceptor also now injects `x-facility-id` from the `FACILITY_ID` AsyncStorage key (multi-tenancy fix). [sl §3.13, §4; `services/Notifications/foregroundNotifications.ts`; `services/Api/index.tsx:82-89`]
- **ANN-FR-25 — Admin web handling.** Bell panel merges socket/in-app events (Redux) with today's announcements; unread badge counts unread Redux items; mark-read / mark-all-read are **local-only** (no API); clicking does not navigate; the feed is **lost on page reload** (no fetch-on-login seeding from `NotificationHistory`). [admin §2.14b]

---

## 4. Business rules & policies

1. **Audience targeting** is announcement-level: `resident`, `family`, or `both` selects which push-token populations the daily cron multicasts to. In-app feeds do not filter by audience (clients fetch the same list endpoint).
2. **One push per announcement per active date.** `notificationDates[]` is the per-day ledger; `notificationProcessingAt` is the atomic claim lock preventing double-send across concurrent workers.
3. **One reminder per appointment per offset.** `NotificationSentLog` unique-insert claim (ANN-FR-18). The older `notificationSentAt` field on UnifiedSchedule appears superseded by this mechanism.
4. **Offsets are facility-configurable** per module/event; defaults 20 min / 1 h / 1 day; the env default `NOTIFICATION_LEAD_MINUTES` (15) is always added to the offset sweep.
5. **Missing preference = opted in.** User notification preference keys default ON when absent; only 5 module keys are recognized.
6. **Date windows are facility-calendar-based**: announcement "active today" windows are built from UTC-midnight day bounds keyed to the facility-TZ calendar date; but cron TZ and slot math use the single process-wide `FACILITY_TIMEZONE` env — multi-timezone facility fleets are not actually supported despite `Config.timeZone` existing per facility.
7. **Multicast chunking**: FCM sends batch at 500 tokens per chunk.
8. **No approval/draft workflow** for announcements — publish-on-create; correction path is edit or soft delete.
9. **Gallery persistence is opt-in at upload time** (`isSavedToGallary`) and irreversible from the UI (no delete endpoint).
10. **Push degradation**: if Firebase credentials are absent, FCM is silently disabled — socket + history channels still operate.
11. **Assigned-staff channel (staging refactor).** Every staff cName in a resident's unified `assignedStaff[]` array (any designation) receives that resident's events via the `notifyAssignedStaff` channel. This is now **independent** of the permission-group fan-out (ANN-FR-19): the former care-team-designation distinction and the `excludeCareTeamFilter` `$nin` exclusion of Case Manager/Social Worker were removed when the five fixed care-team fields were consolidated into `assignedStaff[]` (PLAT-FR-54). See CARE-FR-63 for the cross-module wiring. [`utils/assignedStaff.ts`]

---

## 5. Notifications & real-time behavior

This module is the notifications platform; the operative summary:

- **Channels**: FCM push (mobile), `notification:new` socket (per-user room, consumed today only by admin web), `NotificationHistory` REST feed (consumed by SN app + TV Alert tab). `new-announcement` socket broadcast goes to all default-namespace clients (admin web only consumer).
- **Cron jobs**: 6 starts from 5 files (ANN-FR-22) — reminder sweep (every minute), appointment auto-complete (15 min), announcement broadcast + 1-day reminder (`*/10`), announcement 1-hour reminder (every minute), care-conference enabler (every minute), care-conference review (every minute).
- **Event catalog**: ~40 events across TRANSPORT / MAINTENANCE / HOUSEKEEPING / ACTIVITIES / SALON / HEALTH_CARE / DINING (ANN-FR-16), each independently toggleable per facility for immediate and scheduled sends.
- **Real-time gaps**: resident apps and the TV consume **no** real-time channel for announcements or notifications — both are pull-on-open. The unified-schedule socket events (`mobile-<Model>-request-upserted/deleted`) broadcast globally with **no facility scoping and no auth** on the default namespace (cross-tenant leak — see §9).

---

## 6. Integrations (FCM, Socket.io)

| Integration | Role | Notes |
|---|---|---|
| Firebase Cloud Messaging (firebase-admin) | Push delivery to resident/family/staff devices | Initialized from `FIREBASE_PROJECT_ID/CLIENT_EMAIL/PRIVATE_KEY` (AWS Secrets Manager); silently disabled when missing; multicast in 500-token chunks |
| Firebase RN SDK (`@react-native-firebase/messaging`) + notifee | Client receipt/display | Wired in SN app and (staging) the SL app for **foreground** generic display + token harvest; background/quit handlers absent in both |
| Socket.io `/notifications` namespace | In-app real-time | Cognito JWT + facility auth; room `user:<cName>`; event `notification:new` |
| Socket.io default namespace | `new-announcement` broadcast | **No auth on default namespace**; broadcast to all connected clients |
| AWS S3 (+ CloudFront read URLs) | Gallery images, resident TV photos, announcement images | `imageKey` written / signed (CloudFront public) URL read |
| node-cron | All 6 cron starts (5 job files) | In-process scheduler; `ENABLE_NOTIFICATION_CRON` master gate implies single-instance deployment assumption (claims mitigate overlap) |

---

## 7. Permissions & access control

- **Admin web**: Announcements page gated by permission key `"Announcements"` (Filter 1 facility pages ∩ Filter 2 staff permissions; ADMIN bypasses Filter 2). Read-only staff see the page with mutating actions disabled.
- **Backend announcement routes have NO auth middleware** — create/list/get/update/delete/past rely only on the (broken) facility-header requirement. Anyone who can reach the API can author facility-wide broadcasts. [backend-wellness §6.1, §8.1]
- **Notification feed** (`GET /api/notifications`) requires Cognito auth; rows are scoped per recipientCName.
- **Notification config** (`/api/notification-config`): admin CRUD (no client UI observed; effectively API-managed).
- **Gallery**: `GET /api/gallery` requires auth but has **no facility scoping** (global collection); resident TV pictures are scoped to the calling resident.
- **Staff reminder targeting doubles as authorization**: staff only receive event notifications for modules where they hold the mapped access permission (ANN-FR-19).
- **TV**: Alert and Photos tabs operate under the TV token impersonating the paired resident; Alert browsing is listed among device-scoped (anonymous-capable) features on the TV.

---

## 8. Product-split notes

- **Shared platform module — applies to both SKUs unchanged.** Announcement model, notification engine, cron suite, and gallery are facility-agnostic; per-facility `NotificationConfig` and `Config.accessPages` do the shaping. The HEALTH_CARE event group (rehab, care conference, IDT report) is the SNF-leaning slice of the catalog; lifestyle groups (SALON, DINING, ACTIVITIES) lean Senior Living — but both ship in the same default catalogue for every facility.
- **Client maturity split is narrowing (staging)**: the SN resident app has a live notifications center, FCM receipt, and TV photo upload; the SL resident app now has **foreground** FCM receipt + token harvest (ANN-FR-24) but still a mock notifications center, no background/quit routing, and no TV photo upload. Any product split spec must decide whether SL inherits the rest of the SN implementations or the residual gap is intentional.
- **Audience `family` only has delivery reach where family push tokens exist** — family members are provisioned phone-first with portal access; there is no family-specific announcement feed UI (family members see the resident feed when operating the resident account).
- **TV Alert tab** exists in the (shared) TV app; it reads the notifications feed for the paired resident regardless of SKU.

---

## 9. Observations & candidate gaps (with evidence refs)

**Delivery-chain gaps (highest product risk)**
1. **SL resident app push partly wired (staging).** The SL app now requests permission, harvests the FCM token at Splash, sends `pushToken`/`x-fcm-token` headers, and renders **foreground** FCM via notifee (ANN-FR-24) — the prior "cannot receive push at all" gap is closed for foreground. *Residual:* no background/quit-state handler or tap-routing, and NotificationsScreen is still a mock. [sl §4; `foregroundNotifications.ts`, `Api/index.tsx:82-89`]
2. **SN notifications feed is a dead end** — `isRead` never mutated (no mark-as-read API call), no deep links; generic FCM taps go nowhere; cold-start/background notification routing absent. [sn §3.10, §4, §7.9]
3. **Admin in-app notifications are Redux-local only** — lost on reload, never seeded from `NotificationHistory`, mark-read is cosmetic. [admin §2.14b, §4.5]
4. **No real-time announcement channel for residents/TV** — `new-announcement` socket is consumed only by admin web; TV is pull-only per tab entry. An "urgent broadcast" capability does not exist as-built. [tv §3.2, §4; backend-platform §8.4]

**Security**
5. **Announcement CRUD routes unauthenticated** (backend); combined with the dead facility-header check, cross-tenant announcement reads/writes are possible. [backend-wellness §6.1, §8.1-2]
6. **Default-namespace socket has no auth and `io.emit` broadcasts unified-schedule events globally** — cross-facility data leakage to any connected client. [backend-platform §8.4, §10.16]
7. **Gallery is global (no facilityId)** — facility A's saved images are listable by facility B's authenticated users. [backend-wellness §6.2, §8.3]

**Correctness / consistency**
8. **Dual-timezone "is today" hack** in the announcement socket check (Asia/Kolkata + America/Los_Angeles hardcoded) — developer-locale artifact. [backend-wellness §8.14]
9. **Single process-wide `FACILITY_TIMEZONE`** governs all cron TZ math despite per-facility `Config.timeZone` — multi-timezone deployments unsupported. [backend-wellness §8.16]
10. **`notificationSentAt` on UnifiedSchedule superseded** by `NotificationSentLog` — dead field. [backend-wellness §8.11]
11. **Notification preference endpoints exist but no client persists them** — SL toggles are local placeholders; SN's settings row is commented out; admin's 5 toggles are localStorage-only and use different labels than the backend's 5 keys. [backend-platform §4; sl §2.4; sn §7.2; admin §2.17]
12. **Gallery write-only accumulation** — no delete/update; `usageCount` exists only on the separate menu-library collection, not GalleryImage. [backend-wellness §6.2]
13. **Icon-as-file round trip** — preset icons are converted to uploaded Files on save rather than stored as an enum, conflating identity types in storage. [admin §2.13]
14. **NotificationConfig has no admin UI** — the per-facility event/offset matrix (the module's central policy surface) is API-only today; candidate for an admin Settings page. [backend-platform §5.2; admin §2.17]
