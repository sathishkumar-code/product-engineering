# Module: In-Room TV Experience

> Applies to: Both
> FR prefix: TV
> Sources: `_codebase-analysis/client-tv-app.md` (primary), `_codebase-analysis/backend-platform-identity.md` §6 (TV platform), `_codebase-analysis/client-resident-app-sn.md` §3.8 (TV companion features). Code is source of truth.

---

## 1. Purpose & scope

The in-room TV experience turns the resident's television into a facility-managed
ambient display and self-service portal. A single Android TV app
(`senior_living_tvapp`, Kotlin/Jetpack Compose) is provisioned per unit and serves
two operating modes:

- **Device-scoped (unpaired)**: ambient photo slideshow with greeting and weather,
  live TV, dining menus and specials, the 7-day schedule, announcements, photos,
  music, and an installed-apps launcher — available to anyone in the room with no
  sign-in.
- **Resident-scoped (paired)**: transactional actions — family meal requests,
  service bookings (salon / massage / private training), and housekeeping
  requests — gated behind a QR pairing flow in which the resident authorizes the
  TV from their mobile app. After pairing, the TV acts *as* that resident.

In scope for this module: device provisioning (wizard + installer backdoor), the
QR pairing and token model, server-driven section configuration, every TV section
as built, refresh/caching behavior, and the device-vs-resident permission model.

Out of scope: the mobile-app features themselves (covered by the resident-app
modules), admin authoring of menus/services/announcements (admin-web module), and
stream head-end infrastructure.

---

## 2. Personas & surfaces

| Persona | Surface | Role in this module |
|---|---|---|
| **Resident (and visiting family)** | TV in the unit, D-pad remote | Primary consumer. Browses device-scoped content; completes transactions after QR pairing. No resident-managed settings exist on the TV (no language, font size, accessibility, or sign-out). |
| **Resident (as authenticator)** | Resident mobile app → Profile → "Login TV App" | Scans the TV's pairing QR with the phone camera; the phone's Cognito session authorizes the TV session. As of staging this companion flow exists in **both** resident apps — SN (`client-resident-app-sn.md` §3.8) and the SL app (`senior_living_reactnative`: `ProfileScreen/LoginTvApp/QRScannerScreen` via `react-native-camera-kit` → `authorizeTvPairing` → `POST /api/tv/pairing/authorize`). Both also manage the TV photo-frame gallery via "Upload Pictures" (`/api/residents/pictures`). |
| **Installer / facilities technician** | TV registration wizard (leanback GuidedStep) or external installer app via ContentProvider | Binds the TV to a facility and unit number; registers the device. Geofenced facility pick reduces mis-binding. |
| **Facility admin (indirect)** | Admin web / backend configuration | Defines what the TV shows: the `AndroidCategories` section tree per facility, channel lineups/EPG, menus and specials, service catalogs, photo-frame assets, music playlists, apps list, geofence points, concierge number, sleep-timer options. No admin UI exists on the TV itself. |

Hidden/system surfaces: developer settings dialog (7× D-pad-up on the first
category — non-functional), sleep-timer popup (non-functional), full-screen video
overlay, ContentProvider provisioning backdoor.

---

## 3. Functional requirements (as-built)

### 3.1 Device identity & provisioning

- **TV-FR-01 — Device identity from DRM.** The device ID is the Base64 of the
  Widevine DRM unique device ID (`MediaDrm.PROPERTY_DEVICE_UNIQUE_ID`), not
  `ANDROID_ID` (`Utils.kt:76-92`, `BaseActivity.kt:83-90`).
- **TV-FR-02 — Launcher registration wizard.** `RegistrationActivity` is the
  Android launcher activity. On start it routes by stored state: no facility →
  facility picker; no unit → unit entry; no deviceId → register; all present →
  main UI (`RegistrationActivity.kt:80-90`).
- **TV-FR-03 — Geofenced facility pick.** The wizard fetches all facilities via
  unauthenticated `GET config/getAllfacility`, requests location permission, and
  offers only facilities whose `tvSetupLocations` lat/lng points are within
  `radius` meters (default 500) of the TV's GPS fix. If location permission is
  denied, the full unfiltered list is shown
  (`RegistrationViewModel.filterFacilitiesByLocation`).
- **TV-FR-04 — Unit binding with confirmation.** The unit number is entered twice
  (enter + confirm, must match) and persisted; the **system device name is set to
  the unit number** (for Cast/Bluetooth identification — requires
  `WRITE_SECURE_SETTINGS`, implying managed/sideloaded fleet deployment).
- **TV-FR-05 — Device registration.** `POST tv/register` with `facilityId` header
  and `{deviceId}` body; backend upserts `TvDevice {facilityId, deviceId (unique
  per facility), lastSeenAt}`. Registration is **open** — no device
  authentication (`backend-platform-identity.md` §6.2.1). On success the app
  restarts into `MainActivity`.
- **TV-FR-06 — Installer (ContentProvider) provisioning.** An external
  system/installer app can provision the TV without the on-screen wizard by
  calling `MyContentProvider.call("onboardUnitMethod", {facilityId, unitNo})` or
  sending the `ONBOARD_UNIT_METHOD` broadcast; this sets facility/unit and
  triggers device registration. The provider also exposes
  `OPEN_PLAY_STORE_METHOD` to deep-link the Play Store — bulk fleet provisioning
  is a supported flow (`MyContentProvider.kt:67-78`).
- **TV-FR-07 — Device binding survives data clears.** `clearAllData()`
  deliberately preserves `unitNo`, `facilityId`, `deviceId` — logout (if it
  existed; see §9) is resident-level, never device-level
  (`PreferenceManager.kt:38-46`).

### 3.2 QR pairing & resident authorization

- **TV-FR-08 — Pairing session creation.** When the TV has no stored access
  token, on socket connect it emits `pairing:create {facilityId, deviceId}` over
  a Socket.IO foreground service (`/tv` namespace, headers `facilityId`,
  `unitNo`, `deviceId` on every transport request). The server reuses an existing
  PENDING unexpired session or creates one, returning
  `{sessionId (uuid), qrToken (6-char uppercase), expiresIn}`. An HTTP
  equivalent exists (`POST /api/tv/pairing/create`); the socket path makes
  `unitNo` **mandatory** and rotates stale sessions bound to another room.
- **TV-FR-09 — QR rendering.** The TV renders a logo-branded QR (vendored
  AwesomeQR fork) **encoding the `sessionId`** (the `qrToken` is carried but not
  the QR content). The QR appears on the home ambient slider when signed out and
  inside the contextual `LoginDialog` for gated actions (`SliderScreen.kt:130-133`,
  `LoginDialog.kt:90-94`). The backend also returns the 6-char code for display.
- **TV-FR-10 — Pairing TTL & rotation.** Pairing sessions expire after 120 s
  (env-overridable). On `pairing:expired` the TV re-emits `pairing:create`,
  rotating the QR indefinitely (`tvAuth.ts:27-39`; `SocketService` protocol).
- **TV-FR-11 — Resident authorization with unit match.** The resident scans the
  QR with the resident mobile app (Profile → "Login TV App"; available on **both**
  the SN and, as of staging, the SL app via `react-native-camera-kit` →
  `authorizeTvPairing`); the app calls
  `POST /api/tv/pairing/authorize {sessionId | qrToken}` under its Cognito
  session. Server rules: session PENDING and unexpired; caller has a Resident
  profile; **resident.unitNo must equal session.unitNo** — sessions without a
  unitNo are rejected outright, preventing cross-room pairing
  (`tvAuth.controller.ts:214-239`). On success the session stores the resident's
  `cName` and Cognito `groups`, and `pairing:authorized` is emitted to the TV's
  socket room.
- **TV-FR-12 — Token exchange.** On `pairing:authorized` the TV emits
  `auth:exchange {sessionId}`; the server validates AUTHORIZED + deviceId match,
  **revokes all prior active TvAuthTokens for (cName, deviceId)** (single active
  session per resident-device), issues an access JWT (HS256, `TV_JWT_SECRET`,
  payload `{cName, device:'tv', deviceId}`, 30-min TTL) plus a 64-hex refresh
  token (stored SHA-256-hashed, 30-day TTL), and marks the session USED. Tokens
  arrive via `auth:tokens`; the TV persists both, silently reloads home data, and
  any open `LoginDialog` auto-dismisses and **continues the gated action**.
- **TV-FR-13 — Token refresh.** On REST 401, an OkHttp `Authenticator` posts
  `{refreshToken}` to `tv/auth/refresh`, stores the new access token, and replays
  the request (double-refresh guarded). The backend validates hash, facility,
  revocation, and expiry. **No refresh-token rotation.** If refresh fails the
  request simply fails — there is no forced logout / re-pair UX (see §9).
- **TV-FR-14 — Contextual login QR for transactions.** Browse features work
  unauthenticated; transactional features (family meal request, service booking,
  housekeeping request) check for a stored access token and pop the QR
  `LoginDialog` when absent, with on-screen instructions for the mobile-app scan
  flow (`DinningContent.kt:210-216`, `ServiceReservation.kt:151`,
  `HousekeepingContent.kt:297-299`).
- **TV-FR-15 — TV request authentication.** Authenticated TV requests send
  `facilityId`, `isTv: true`, and `Authorization: Bearer <accessToken>`. The
  backend's `authTv` middleware verifies the JWT **and** requires a live
  `TvAuthToken` record (not revoked, not expired) — server-side revocation is
  effective within the JWT window. The resulting request user is
  `{username: cName, groups, isTv: true}` — **the TV impersonates the
  authorizing resident** (`authMiddleware.ts:154-204`).

### 3.3 Server-driven section configuration

- **TV-FR-16 — Backend-defined navigation.** There is no client route graph. The
  TV calls `GET android-tv/get-android-categories/{facilityId}` (optional auth)
  and renders whatever `categories` + `subCategories` rows arrive; each row's
  `viewType` string selects the matching feature composable
  (`HomeScreen.kt:297-374`). Supported `viewType` values: *(none/fallback)* →
  ambient slider; `Alert`; `liveTv`; `Photos` / `photoFrameWithoutText`;
  `cardWithDateTimeSelection` (services); `housekeeping`; `singleCenterImage`
  (today's specials); `timeWithRecyclerView` (dining menu); `My_Schedules`;
  `calls`; `apps`; `music`.
- **TV-FR-17 — Per-facility experience without client conditionals.** Because the
  tab set is a per-facility backend document (`AndroidCategories`), two facility
  types can ship entirely different TV experiences with the same APK. The home
  payload also carries per-facility knobs: photo-frame assets, music playlists,
  apps list, `turnOffTimer` options, `conciergeNo`, current temperature.
  (Note: the admin write endpoint `POST /android-categories` is a stub — see §9.)

### 3.4 Sections — as built

- **TV-FR-18 — Home / ambient slider.** Full-screen crossfading slideshow (30 s
  per image, 2 s fade) from the home payload's photo-frame assets; overlays:
  "Welcome, {guestName}" (or generic) greeting, current temperature, brand line,
  and — when unpaired — the pairing QR. (Top logo never loads; ambient video mode
  is commented out — see §9.)
- **TV-FR-19 — Live TV.** Channel lineup, category grouping, EPG, and stream
  endpoints are all facility-configured server-side
  (`GET v1/livetv?isTVos=true`). Browse UX: leanback rows per category; focusing
  a channel starts a debounced **muted inline preview** with logo and
  current/next EPG programme; click opens the full-screen player. Playback
  supports HLS/DASH/RTMP/HTTP and **UDP multicast**, with a local piping layer
  feeding a native Pro:Idiom/AES decryptor (JNI) — encrypted on-prem hotel-grade
  multicast streams are supported alongside cloud OTT URLs (summary level; detail
  in `client-tv-app.md` §3.3). Channel categories are cached in Room and
  authoritative for **24 h**, silently refreshed by API. EPG "now playing" is
  recomputed locally every 5 minutes (no network refetch); expired EPG entries
  are pruned client-side.
- **TV-FR-20 — Dining menu browser.** `GET menu?date=YYYY-MM-DD`; left sidebar
  generates the next 7 days client-side (Today / Tomorrow / dates); focusing a
  date fetches that day's meal categories and item cards (name + picture).
  **Browse-only** — no per-item ordering, no dietary filters.
- **TV-FR-21 — Today's dining specials.** `singleCenterImage` tab renders
  `dailySpecials[0].fileUrl` from the menu payload as a single full-bleed,
  admin-uploaded poster (recurrence/`effectiveDates` resolved server-side).
- **TV-FR-22 — Family meal request (dining transaction).** "Request Meal" → auth
  gate (TV-FR-14) → `GET family-meal-requests/weekly-meals` → dialog: pick weekly
  meal, guest count, meal type/time, price-per-person →
  `POST family-meal-requests {numberOfGuests, mealType, pricePerPerson,
  startMealDate, endMealDate, mealTime}` → confirmation. A revenue analytics
  event logs `guests × pricePerPerson`.
- **TV-FR-23 — Services booking.** Sub-category id selects the catalog: `Salon` →
  `GET salon/tv/services`; `Massage_Therapy` → `GET massage/tv/services`;
  `Private_Training_Session` → `GET private-training/tv/services` (unknown
  categories fall back to salon; slugs are compiled into the client). Cards show
  image, name, description, duration, price. "Book" → auth gate →
  `POST {category}/{serviceId}/available-slots {noOfDays: 7}` → 7-day slot picker
  (closed days filtered; `canJoinWaitlist` parsed but no waitlist UI) →
  `POST {category}/book-appointment {date, startTime, endTime, …ServiceId}` →
  success/failure dialog.
- **TV-FR-24 — Housekeeping & maintenance requests.** Four request types
  hardcoded client-side: Extra Room Cleaning, Extra Laundry, Miscellaneous
  Service, Report Maintenance (free-text description; a mic-icon flag exists but
  no voice input). Flow: pick card → auth gate → calendar date pick →
  "charges may apply" confirmation → `POST housekeeping {residentId: "",
  residentName, selectedDate, unitNo, requestType, description}` — `residentId`
  is sent empty; the backend resolves identity from the TV token. No request
  history or status view on the TV.
- **TV-FR-25 — 7-day unified schedule.** `GET unified-schedule?noOfDays=7`
  returns 7 daily buckets typed `SCHEDULE` (facility activity),
  `BREAKFAST|LUNCH|DINNER` (meals), `PT`, `SALON`, `MASSAGE` (the resident's own
  bookings), each with time range, description, image, status. UX: 7-day date
  strip → day's items → focusable detail panel. **View-only** — no cancel/modify
  from the TV. Bookings made via TV-FR-23 surface here on next visit (refetch on
  entry).
- **TV-FR-26 — Announcements / alerts.** Paged list from
  `GET notifications?page&limit` (10/page, Paging 3), refreshed on every tab
  entry. View-only cards (title/description/date). No mark-as-read,
  acknowledgement, or deep-link; **pull-only** — no push or socket channel.
- **TV-FR-27 — Photos (family gallery).** `GET residents/pictures` → resident
  family photos uploaded by family/admin elsewhere on the platform (the resident
  mobile app manages this gallery via `GET/POST/DELETE /api/residents/pictures` —
  "Upload Pictures"; available in both the SN app (`client-resident-app-sn.md` §3.8)
  and, as of staging, the SL app (`ProfileScreen/UploadPictures` → `services/Profile/uploadPictures/tvImages.ts`)).
  UX: main image +
  thumbnail strip, full-screen view, Ken Burns animation, empty state. View-only.
  The `photoFrameWithoutText` viewType reuses the same screen for the generic
  facility photo-frame slideshow assets.
- **TV-FR-28 — Music.** Tracks come from the home payload's curated
  `musicSettings.subData[]` (no dedicated API). All tracks are pre-downloaded in
  the background when home data loads; playback prefers the cached file, falling
  back to the streaming URL. Controls: play/pause, next/previous (wrap-around),
  seek bar. A single shared ExoPlayer keeps music playing while browsing other
  tabs.
- **TV-FR-29 — Apps launcher.** Grid grouped by the home payload's
  `apps[]{category, list[]{name, pkg}}`; icons resolved from the local
  PackageManager; click launches the package's leanback/launch intent.
  Not-installed apps show name text only; no install/update flow in-app
  (Play-Store deep-link exists only via the ContentProvider backdoor).
- **TV-FR-30 — Calls (mock).** "Who would you like to call?" renders a Concierge
  card + 3 contacts — **all hardcoded mock data with randomuser.me portraits**;
  call/video buttons have no calling backend (no WebRTC/SIP/Twilio in the repo).
  Façade/placeholder section (see §9).

### 3.5 Refresh & caching model

- **TV-FR-31 — Pull-on-tab-entry.** Dining menu, alerts, photos, schedule,
  specials, and services all refetch when their tab is entered. There is no
  real-time content push of any kind (§5).
- **TV-FR-32 — Caching.** Home payload cached to SharedPreferences (used as
  fallback when the API fails); live TV categories cached in Room with 24 h
  validity; music cached to files. Timers: 5-min local EPG recompute, 30 s
  slider rotation, 1 s clock tick.

---

## 4. Business rules & policies

| # | Rule | Source |
|---|---|---|
| BR-1 | One TV device binds to exactly one (facility, unit); `deviceId` is unique per facility. Binding survives any in-app data clear. | TV-FR-05/07 |
| BR-2 | Pairing QR is valid 120 s, then auto-rotates. Access token 30 min; refresh token 30 days; all env-overridable. | TV-FR-10/12/13 |
| BR-3 | A resident may authorize a TV **only for their own unit**: `resident.unitNo == session.unitNo`; unit-less sessions are rejected. | TV-FR-11 |
| BR-4 | One active TV session per (resident, device): token exchange revokes all prior active tokens for that pair. | TV-FR-12 |
| BR-5 | Browse is device-scoped and anonymous; transactions (meal request, service booking, housekeeping) require a paired resident token. | TV-FR-14 |
| BR-6 | Once paired, the TV acts with the resident's identity (and Cognito groups, transport-dependent — see §9 #11) on every authenticated call. | TV-FR-15 |
| BR-7 | Facility pick at setup is geofenced to `tvSetupLocations` within `radius` m (default 500); permission denial falls back to the full list. | TV-FR-03 |
| BR-8 | 7-day forward windows everywhere: dining menu dates, service slot search (`noOfDays: 7`), unified schedule. Compiled into the client. | TV-FR-20/23/25 |
| BR-9 | Housekeeping and family-meal requests may incur charges; the TV shows "charges may apply / charges will be added to your account" copy before submission. | TV-FR-22/24 |
| BR-10 | Channel lineup cache is authoritative for 24 h; content correctness on other tabs depends on tab re-entry. | TV-FR-19/31 |
| BR-11 | Service catalogs are limited to the compiled-in trio salon / massage / private training; unknown sub-categories silently fall back to salon. | TV-FR-23 |

---

## 5. Notifications & real-time behavior

- **Socket.IO is pairing-only.** The foreground `SocketService` consumes exactly:
  `pairing:create/created/expired/authorized`, `auth:exchange`, `auth:tokens`
  (plus error events server-side). No announcement, menu, schedule, or
  device-command events exist on the wire.
- **No content push.** A facility announcement reaches the resident only on the
  next visit to the Alert tab; menu/schedule changes appear on next tab entry.
  Server-driven category changes apply on next home load.
- **Server→TV delivery** uses socket room `tv-pairing:<facilityId>:<sessionId>`;
  pairing expiry is driven by a one-shot **in-memory** server timer (process
  restart drops pending expiry emits).
- **In-app broadcasts** glue the service to the UI: new QR session
  (`SERIAL_NUMBER`), tokens saved (`AUTH_TOKENS_SAVED` → silent home reload +
  gated-action continuation), external provisioning (`ONBOARD_UNIT_METHOD`),
  screen on/off (logging only).

---

## 6. Integrations

| Integration | Direction | Notes |
|---|---|---|
| Senior-living backend REST (`/api/*`) | TV → backend | All content and transactions; headers `facilityId`, `isTv: true`, optional Bearer token. Key endpoints: `tv/register`, `tv/pairing/*`, `tv/auth/*`, `android-tv/get-android-categories/{facilityId}`, `config/getHotelForAndroidTV`, `v1/livetv`, `menu`, `family-meal-requests*`, `{salon|massage|private-training}/tv/services` + slots/booking, `housekeeping`, `unified-schedule`, `notifications`, `residents/pictures`. |
| Socket.IO `/tv` namespace | Bidirectional | Pairing/auth lifecycle only (§5). |
| Resident mobile app (SN **and** SL as of staging) | Companion | QR scan → `POST /api/tv/pairing/authorize`; TV photo-frame gallery management (`/api/residents/pictures`). Both resident apps now ship the "Login TV App" (camera-kit QR scan) and "Upload Pictures" flows. |
| OpenWeather (via backend proxy) | Backend → OpenWeather | `GET /api/config/currentTemperature` for the home overlay; API key hardcoded in backend source (flagged in backend analysis). |
| Stream head-ends | TV → streams | Cloud OTT URLs (HLS/DASH/RTMP/HTTP) and on-prem UDP multicast with native Pro:Idiom/AES decryption (summary level). |
| Firebase Analytics / Crashlytics | TV → Firebase | Screen views, `live_tv_preview` / `live_tv_watch`, revenue (`logMyStay`) events; crash reporting. |
| Local PackageManager / Play Store | On-device | Apps-launcher icons + launch intents; Play-Store deep-link via ContentProvider only. |
| Installer tooling | Installer app → TV | ContentProvider/broadcast provisioning (TV-FR-06). |

---

## 7. Permissions & access control (device vs resident scope)

| Capability | Scope required | Enforcement |
|---|---|---|
| Register device, create pairing session | None (open) | `tv/register` and pairing creation have no device authentication — facility header only. |
| Browse: home, live TV, menu, specials, schedule, alerts, photos, music, apps | **Device** (facility + unit headers; no token) | Client sends headers; content endpoints accept optional/no auth. Note: family photos are served by device identity even when unpaired. |
| Family meal request, service booking, housekeeping request | **Resident** (paired TV token) | Client-side auth gate (QR `LoginDialog`) + backend `authTv` (JWT + live token record). Backend resolves resident identity from the token (housekeeping sends `residentId` empty). |
| Authorize a pairing session | Resident Cognito session on mobile, same `unitNo` | Backend rule TV-FR-11. |
| Revoke a TV session | Server-side | Live-token check means revocation takes effect within the 30-min JWT window; new pairing by the same resident on the same device revokes prior tokens. **No resident-facing revoke/sign-out UI exists on the TV** (§9). |
| Provision / re-bind device | Installer (physical access) | Wizard or ContentProvider; no authentication on the provisioning surface beyond device possession. |

The TV's authenticated identity is the *authorizing resident* — there is no
separate "device account". Cognito `groups` ride on the token (socket-exchange
path only; see §9 #11).

---

## 8. Product-split notes

- **No client-side split exists.** A repo-wide search for skilled-nursing /
  facility-type conditionals (`skilled`, `nursing`, `facilityType`, `assisted`,
  `memory care`, `independent living`) returns nothing in the TV app
  (`client-tv-app.md` §6). The split lever is entirely server-side: the
  per-facility `AndroidCategories` document decides which tabs exist, so SN and
  SL facilities can ship different TV experiences from one APK.
- **`FacilityData.designations`** is delivered to the TV during registration but
  never read — the only client-visible facility-classification field is unused.
- **Parameterization gaps if SN diverges**: housekeeping's four compiled-in
  request types; the salon/massage/private-training service trio (slugs compiled
  in); 7-day windows for menu/schedule/slots; "charges will be added to your
  account" billing copy (likely wrong for SN payer models); compiled-in English
  strings (no localization, no server copy config).
- **Shared-platform implication**: a product split should be expressed as two
  `AndroidCategories` templates plus backend catalog config, not as a client
  fork — but the gaps above require client work before SN-specific sections
  (e.g., care-oriented tabs) can exist.

---

## 9. Observations & candidate gaps

Evidence references are to `client-tv-app.md` (TV) and
`backend-platform-identity.md` (BE) unless noted.

**Façade / half-built features**
1. **Calls tab is a mock** — hardcoded contacts with randomuser.me portraits; no
   calling stack anywhere in the repo (TV §3.11, `CallContent.kt:79-82`).
   Decision needed: build (WebRTC/concierge dial-out) or remove from facility
   category configs.
2. **No working resident sign-out on the TV.** The logout lambda is commented out
   (`HomeScreen.kt:273-276`) and the `ClearCredentialsCard` tile has an empty
   `onClick` (`AppsContent.kt:296`). Combined with no forced re-pair on refresh
   failure (TV-FR-13), a paired resident identity persists on the device for up
   to the 30-day refresh window with no in-room way to end it — a privacy gap on
   unit turnover unless server-side revocation is operationally exercised.
3. **Sleep timer does nothing** — selection only fires a debug toast
   (`CategoryItem.kt:453-455`); screen on/off is observed/logged but never
   initiated.
4. **Developer settings dialog** renders a password field with no submit action
   and nothing behind it (`DeveloperSettingsDialog.kt`).
5. **Waitlist** flag (`canJoinWaitlist`) is parsed per day but no waitlist
   UI/action exists (`ServiceReservation.kt:112-116`); dead "Add" button with
   `/* TODO: Add to reservation */` (`ServiceCard.kt:170`).
6. **Admin write path for the section tree is a stub** —
   `POST /android-categories` returns 201 unconditionally with its insert logic
   commented out (BE §6.4) — i.e., the server-driven menu is read-only in
   practice; configuration presumably happens directly in Mongo.

**Dead / commented-out code & heritage**
7. **Hotel-app heritage throughout**: commented `roomNo/hotelCode/hotelId`
   headers, `hotelroomtvapp` broadcast actions, hotel-named log folders, "Dear
   Guest" salutation constant (TV §7 #11); backend endpoint named
   `getHotelForAndroidTV` and `isShashiId`/`emailId` fields in
   `AndroidCategories` (BE §6.4). Cosmetic now, but it obscures intent and
   invites misuse of vestigial fields.
8. Channel bandwidth/error telemetry is collected into Room but the upload call
   is commented out (`MainActivity.callAPIFor:222-232`) — silent storage growth,
   zero observability value. Likewise the local JSON "portal log" file is written
   forever with **no upload or rotation** — unbounded device storage growth
   (`TVLogger.kt:169-208`).
9. Ambient video slider and the live-TV error-mail path are fully commented out
   (TV §7 #9-10).

**Security observations (flagged at summary level; details in source analyses)**
10. **Transport security**: accept-all TLS trust manager + hostname verifier in
    all build flavors (`NetworkModule.kt:74-99`); staging and production socket
    URLs are plain `http://` with `usesCleartextTraffic="true"` (TV §5) — the
    pairing/auth channel that carries tokens is unencrypted in production.
11. **Groups inconsistency by transport**: socket `auth:exchange` persists the
    resident's Cognito `groups` on the TvAuthToken; the HTTP variant does not —
    TVs paired over HTTP get empty groups, so any role-gated behavior differs by
    transport (BE §6.2.4, observation #17).
12. **Open registration**: `tv/register` and pairing-session creation require no
    device credential; any caller knowing a facilityId can mint devices/sessions.
    Mitigated by the unitNo-match rule for authorization, but enumeration/squat
    risk remains (BE §6.2.1).
13. Tokens stored in plain SharedPreferences; refresh token logged at error level
    (`SocketService.kt:169`). Signing keystore committed at repo root; release
    build signs with the **debug** config (`build.gradle.kts:63`).
    `WRITE_SECURE_SETTINGS` requires privileged grant (fleet deployment signal).
14. Backend pairing expiry uses an in-memory timer — pending `pairing:expired`
    emits are lost on process restart (BE §6.2.2); TTL indexes still expire data,
    but the TV-side QR rotation then relies on its own failure handling.

**UX / product gaps**
15. No resident settings on the TV: no language, font size, or accessibility
    options; the "ADA" icon is a no-op (TV §1). Notable for a senior-living
    audience.
16. No request history or status anywhere: housekeeping, family-meal, and service
    bookings give a one-shot confirmation; only service bookings surface later
    (in the schedule). No cancel/modify for anything from the TV.
17. Announcements are pull-only with no acknowledgement — unsuitable as-is for
    urgent facility communications (TV §3.2, §4).
18. Architecture doc drift: `architecture-senior_living_tvapp.md` claims
    event-streaming sockets and a fixed tab set; code shows pairing-only sockets
    and a server-driven menu (TV §7 #20). Code wins; doc needs correction.
