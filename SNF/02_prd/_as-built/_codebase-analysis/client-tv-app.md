# Codebase Analysis — `senior_living_tvapp` (Resident In-Room Android TV App)

> Reverse-engineered functional requirements. Source of truth: code at
> `shashi.ai/senior-living/senior_living_tvapp` (Kotlin, Jetpack Compose, Hilt, Retrofit,
> Socket.IO, ExoPlayer). Secondary input `docs/architecture/architecture-senior_living_tvapp.md`
> was consulted; where they differ, code wins. All paths below are relative to the repo root;
> Kotlin paths are relative to `app/src/main/java/com/shashigroup/sal/tvapp/` unless prefixed.

---

## 0. Top-level shape

- **Single-activity Compose app, server-driven navigation.** There is no Compose `NavHost` route
  graph. `MainActivity` hosts one composable (`HomeScreen`) and the *backend* defines the menu:
  `GET android-tv/get-android-categories/{facilityId}` returns `categories` + `subCategories`
  with a `viewType` string per item; `HomeScreen` switches on `viewType` to render the matching
  feature composable (`features/home/HomeScreen.kt:297-374`, view-type constants at
  `core/data/annotation/AppConstants.kt:44-56`).
- **Two activities** (`app/src/main/AndroidManifest.xml:38-60`):
  - `features/registration/RegistrationActivity` — **the launcher** (also `category.HOME`),
    leanback GuidedStep onboarding wizard.
  - `MainActivity` — main resident UI, `singleInstance`.
- **One foreground service**: `core/service/SocketService.kt` (Socket.IO,
  `foregroundServiceType="remoteMessaging"`).
- **One ContentProvider**: `provider/MyContentProvider.kt` — an *external onboarding backdoor*:
  another (system/installer) app can call `call("onboardUnitMethod", json)` with
  `{facilityId, unitNo}` to provision the TV without the on-screen wizard, and
  `OPEN_PLAY_STORE_METHOD` to deep-link the Play Store.
- **Modules**: `app/` (everything), `QRCodeHelper/` (vendored AwesomeQR fork used to render the
  pairing QR with a logo, `QRCodeHelper/src/main/java/com/github/sumimakito/awesomeqr/`),
  and `apsdk/` — **vestigial**: contains only `build/intermediates` leftovers and is *not* in
  `settings.gradle.kts` (only `:app` and `:QRCodeHelper` are included). There is no apsdk source;
  any prior "apsdk" responsibility (per the hotel TV heritage) is gone from this repo.

---

## 1. Screen / section map

The bottom **category bar** (`features/home/CategoryItem.kt:91-180`) and **sub-category bar**
(`features/home/SubCategoryItem.kt`) render whatever the backend sends; the screens the app can
actually render are fixed by the `viewType` switch in `HomeScreen.kt:297-374`:

| viewType (server string) | Screen | Files |
|---|---|---|
| *(fallback / none)* | Home ambient slider (photo slideshow + welcome + weather + pairing QR) | `features/home/SliderScreen.kt` |
| `Alert` | Announcements / notifications list | `features/home/AlertContent.kt`, `AlertViewModel.kt`, `AlertPagingSource.kt` |
| `liveTv` | Live TV channel browser + preview + full-screen player | `features/channels/LiveTvFragment.kt`, `ChannelBrowseFragment.kt`, `VideoStreamingForLiveTvFragment.kt`, `LiveTVNewViewModel.kt` (leanback Fragments embedded via `AndroidView`) |
| `Photos` / `photoFrameWithoutText` | Family photo gallery / photo frame | `features/photos/PhotosContent.kt`, `PhotosViewModel.kt` |
| `cardWithDateTimeSelection` | Service reservation (salon / massage / private training) | `features/services/ServiceReservation.kt`, `ServiceCard.kt`, `ServicesViewModel.kt`, `features/dialogs/timeselection/TimeSelectionDialog.kt` |
| `housekeeping` | Housekeeping & maintenance requests | `features/housekeeping/HousekeepingContent.kt`, `HousekeepingViewModel.kt`, `features/dialogs/housekeeping/HousekeepingCalendarDialog.kt` |
| `singleCenterImage` | Today's dining specials (single poster image) | `features/dining/TodaySpecials.kt`, `TodaysSpecialsViewModel.kt` |
| `timeWithRecyclerView` | Dining menu browser + family meal request | `features/dining/DinningContent.kt`, `DinningContentViewModel.kt`, `DinningSidebar.kt`, `MealCategorySection.kt`, `features/dialogs/familymeal/RequestFamilyMealDialog.kt` |
| `My_Schedules` | Resident personal schedule (7 days) | `features/schedule/MyScheduleScreen.kt`, `MyScheduleViewModel.kt`, `ScheduleComponents.kt`, `ScheduleDetailPanel.kt` |
| `calls` | Calls screen (concierge + contacts) — **mock data** | `features/calls/CallContent.kt` |
| `apps` | Installed-apps launcher grid | `features/apps/AppsContent.kt` |
| `music` | Music player (server-curated playlists) | `features/music/MusicContent.kt`, `MusicPlaybackSection.kt`, `MusicProgressBar.kt`, `core/utils/MusicCacheManager.kt` |

Plus non-tab surfaces:

- **Registration / pairing wizard**: `features/registration/RegistrationActivity.kt`,
  `RegistrationFragment.kt` (facility picker), `UnitEntryFragment.kt` (unit number entry) —
  leanback `GuidedStepSupportFragment`s, *not* Compose.
- **QR sign-in dialog** (per-action auth): `core/ui/LoginDialog.kt` + `LoginItem.kt`.
- **Developer settings dialog** (hidden, 7× D-pad-up on the first category within 10 s):
  `features/dialogs/developer/DeveloperSettingsDialog.kt` via `CategoryItem.kt:281-301`.
- **Sleep-timer popup** from the power icon: `CategoryItem.kt:436-560`.
- **Full-screen video overlay**: `HomeScreen.kt:105-150` (`FullScreenPlayerOverlay` hosting
  `VideoStreamingForLiveTvFragment`).

There is **no settings screen** for residents (no language, font-size, or accessibility settings).
The "ADA" icon in the category bar invokes a `logout` lambda whose body is commented out
(`HomeScreen.kt:273-276`) — it does nothing.

---

## 2. Device provisioning & auth

### 2.1 Device identity

- Device ID = Base64 of the **Widevine DRM unique device ID**
  (`core/utils/Utils.kt:76-92`, `MediaDrm.PROPERTY_DEVICE_UNIQUE_ID`), not `ANDROID_ID`
  (an `ANDROID_ID` helper exists in `core/data/models/CommonUtils.kt:43-45` but registration uses
  Widevine — `core/ui/base/BaseActivity.kt:83-90`).

### 2.2 Onboarding (facility + unit binding)

`RegistrationActivity.onCreate` (`features/registration/RegistrationActivity.kt:80-90`) routes by
stored state:

1. **No facility** → `GET config/getAllfacility` (no auth, `TvHomeApiService.kt:111-112`), then
   requests location permission and **geofilters** facilities: a facility is offered only if the
   TV's GPS fix is within `radius` meters (default 500) of one of the facility's
   `tvSetupLocations` lat/lng points (`RegistrationViewModel.filterFacilitiesByLocation`,
   `RegistrationViewModel.kt:70-93`; models `core/data/remote/model/FacilityModels.kt`). If
   permission is denied, the unfiltered list is shown (`RegistrationActivity.kt:189-199`).
   Resident picks the facility (`RegistrationFragment.kt`) → `preferenceManager.facilityId`.
2. **No unit** → `UnitEntryFragment`: numeric unit number entered twice (enter + confirm, must
   match), saved to `preferenceManager.unitNo`, and the **system device name is set to the unit**
   (`Utils.setTvName`, used for Cast/BT identification).
3. **No deviceId** → `POST tv/register` with header `facilityId` and body `{deviceId}`
   (`TvHomeApiService.kt:98-102`); on success the deviceId is persisted
   (`SeniorRepositoryImpl.kt:214-229`) and the app restarts into `MainActivity`.
4. All present → straight to `MainActivity`.

Alternative path: the **ContentProvider backdoor** (`provider/MyContentProvider.kt:67-78`) or the
`ONBOARD_UNIT_METHOD` broadcast (`MainActivity.kt:70-82`, `RegistrationActivity.kt:52-69`) sets
facility/unit programmatically and triggers `registerDevice()` — i.e. bulk provisioning by an
installer tool is a supported flow.

`clearAllData()` (`core/data/local/PreferenceManager.kt:38-46`) deliberately **preserves**
`unitNo`, `facilityId`, `deviceId` — logout is resident-level, never device-level.

### 2.3 QR pairing → resident TV token (Socket.IO)

`SocketService` (`core/service/SocketService.kt`) is started by `MainActivity` as a foreground
service; it exits immediately if no `deviceId` is registered (`onStartCommand:71-82`). It opens a
Socket.IO websocket to `BuildConfig.BASE_URL_SOCKET` (`…/tv` path) with headers
`facilityId`, `unitNo`, `deviceId` injected on every transport request (`connectSocket:101-104`).

Event protocol (names in `core/data/models/CommonUtils.kt:15-20`):

1. On connect, **if no `accessToken` stored**, TV emits `pairing:create` `{facilityId, deviceId}`.
2. Server replies `pairing:created` → `{sessionId, qrToken, expiresIn}`
   (`core/data/remote/model/PairingModels.kt`). Service broadcasts these to the UI
   (`BroadcastActions.SERIAL_NUMBER`).
3. UI renders a QR code **encoding the `sessionId`** (note: `qrToken` is carried around but the
   QR content is the sessionId — `SliderScreen.kt:130-133`, `LoginDialog.kt:90-94`,
   `core/utils/generateQrCodeFromLib.kt`). QR shows on the home slider (when signed out) and in
   the contextual `LoginDialog`.
4. Resident scans with the **SAL mobile app** (dialog instructions: open SAL app → Profile →
   "Login TV App" → point camera at QR; `LoginDialog.kt:225-264`). The mobile app authorizes the
   session server-side.
5. Server emits `pairing:authorized` → TV emits `auth:exchange` `{sessionId}`.
6. Server emits `auth:tokens` → `{accessToken, refreshToken, expiresIn}`; service persists both
   and broadcasts `AUTH_TOKENS_SAVED`; `MainActivity` silently reloads home data
   (`MainActivity.kt:103-106`) and any open `LoginDialog` auto-dismisses and continues the
   gated action (`LoginDialog.kt:104-123`).
7. `pairing:expired` → TV re-emits `pairing:create` (rotating QR).

### 2.4 Token usage, refresh, expiry

- Every REST call sends headers `facilityId`, `isTv: true`, and `Authorization: Bearer <token>`
  when a token exists (`SeniorRepositoryImpl.getHeaders`, `:267-281`). Commented-out
  `roomNo`/`hotelCode`/`hotelId` headers betray the hotel-app lineage.
- **401 handling**: `core/data/remote/TokenAuthenticator.kt` (OkHttp `Authenticator`) posts
  `{refreshToken}` to `tv/auth/refresh` with `facilityId` header, stores the new `accessToken`,
  and replays the request; double-refresh is guarded with a `synchronized` token-changed check.
  If refresh fails it returns `null` (request fails); there is **no forced logout / re-pair UX** —
  the resident just sees errors until tokens are cleared.
- **Anonymous vs signed-in**: browse features (menu, schedule, specials, live TV, photos via
  device identity, alerts) work device-scoped; *transactional* features (family-meal request,
  service booking, housekeeping request) check `preferenceManager.accessToken` and pop the
  QR `LoginDialog` when empty (`DinningContent.kt:210-216`, `ServiceReservation.kt:151`,
  `HousekeepingContent.kt:297-299`).

---

## 3. Per-section functional spec

### 3.1 Home / ambient slider (`features/home/SliderScreen.kt`)

- Full-screen crossfading photo slideshow (30 s per image, 2 s fade) sourced from
  `userpreferencesData.familyPhotoFrame_Default_without_sample_text` in the home payload
  (`HomeViewModel.kt:159`).
- Overlays: personalized greeting "Welcome, {guestName}" (or generic), current temperature from
  home payload (`userPreferencesData.currentTemperature`), Shashi.AI brand line, and — when not
  paired — the **pairing QR** (§2.3).
- Top logo is a placeholder `AsyncImage(model = "")` (`SliderScreen.kt:84-90`) — never loads.
- `VideoPlayerSlider` (video ambient mode) is fully commented out (`SliderScreen.kt:184-212`).

### 3.2 Announcements / alerts (`features/home/AlertContent.kt`)

- Paged list from `GET notifications?page&limit` (10/page, Jetpack Paging 3 —
  `AlertPagingSource.kt`), refreshed every time the tab is entered (`AlertContent.kt:88-90`).
- View-only cards (title/description/date, shimmer placeholders). No mark-as-read, no
  acknowledgement, no deep-link. **No push/socket channel — announcements are pull-only**;
  a resident sees new announcements only when opening the Alert tab.

### 3.3 Live TV (`features/channels/`)

- **Channel source**: `GET v1/livetv?isTVos=true` (`ApiEndPoints.GET_LIVE_TV`,
  `LiveTVNewViewModel.loadLiveTV`) returns `live-tv.categories[]` → `Category{name, channels[]}`,
  `Channel{channelId, number, name, logo, type, streaming{protocol, url, multicastGroup, port},
  encryption{isEncrypted, decryptionKey, drmSystem}, epg[], tuner}`
  (`core/ui/theme/Channel.kt`). So channel lineup, grouping into categories, EPG and stream
  endpoints are all facility-configured server-side.
- **Offline cache**: categories persisted to Room (`LiveTVDatabase`, `CategoryDao`); cache is
  authoritative for 24 h (`LiveTVNewViewModel.kt:39-59`), API result silently refreshes it.
- **Browse UX**: leanback `ChannelBrowseFragment` rows per category with channel cards
  (`CustomChannelPresenter`); focusing a channel starts a **muted inline preview** after a debounce
  (`LiveTvFragment.kt:131,231-241`, mutex-guarded), showing channel logo + current/next EPG
  programme; clicking opens the full-screen player overlay (`HomeViewModel.openFullScreenVideo` →
  `FullScreenPlayerOverlay` → `VideoStreamingForLiveTvFragment`).
- **Playback**: `CommonExoPlayer` (`core/utils/player/`) supports HLS/DASH/RTMP/HTTP and
  **UDP multicast** (`udp://@{multicastGroup}:{port}`, `VideoStreamingForLiveTvFragment.kt:137`)
  with a local piping layer (`PipedUdpStream`, `PipedHttpServer/Stream` via NanoHTTPD) feeding
  a **native Pro:Idiom/AES decryptor** (`KotlinDemo` JNI, `app/src/main/cpp/`,
  `DecryptedDataSourceWithNewBuffer.kt`) — i.e. encrypted on-prem hotel-grade multicast streams
  are supported in addition to cloud OTT URLs.
- **EPG**: expired entries pruned client-side (`Category.cleanUpList`), active programme computed
  per channel (`Channel.getSelectedCh`).
- **Refresh**: 5-minute timer re-emits the local channel list (EPG "now playing" recompute), not a
  network refetch (`LiveTVNewViewModel.startAutoRefresh:117-133`).
- **Telemetry**: bandwidth/error reports stored in Room (`ChannelBandwidthDatabase`); the upload
  call is commented out (`MainActivity.callAPIFor:221-233`) — dead path. Analytics events
  `live_tv_preview` / `live_tv_watch` fire to Firebase.

### 3.4 Dining — menu browser + family meal (`features/dining/`)

- **Menu**: `GET menu?date=YYYY-MM-DD`. Left sidebar = next 7 days (Today/Tomorrow/then dates,
  generated client-side, `DinningContent.kt:86-106`); focusing a date fetches that day's menu.
  Content = meal categories (e.g. Breakfast/Lunch/Dinner) with item cards (name + picture)
  (`MealCategorySection.kt`). **Browse-only**: no per-item ordering, no dietary filters.
- **Today's specials** (`singleCenterImage` tab): `GET menu` → `dailySpecials[0].fileUrl`
  rendered as a single full-bleed poster (`TodaySpecials.kt`, `TodaysSpecialsViewModel.kt`) —
  an admin-uploaded image, with `repeatPattern`/`effectiveDates` resolved server-side.
- **Family meal request** (the one dining transaction): "Request Meal" button → auth gate
  (QR login if signed out) → `GET family-meal-requests/weekly-meals` →
  `RequestFamilyMealDialog`: pick a weekly meal, guest count, meal type/time, price-per-person →
  `POST family-meal-requests` `{numberOfGuests, mealType, pricePerPerson, startMealDate,
  endMealDate, mealTime}` (`DinningContentViewModel.requestFamilyMeal:121-187`,
  `FamilyMealRequestModels.kt`) → confirmation dialog. Revenue event logged via
  `logMyStay(value = guests × pricePerPerson)`.

### 3.5 Services — salon / massage / private training (`features/services/`)

- The sub-category id selects the catalog: `Salon` → `GET salon/tv/services`,
  `Massage_Therapy` → `GET massage/tv/services`, `Private_Training_Session` →
  `GET private-training/tv/services` (`ServicesViewModel.fetchServices:30-44`; unknown categories
  fall back to salon). Cards show image, name, description, duration (min), price ($).
- "Book" on a card → auth gate (QR login) → `POST {category}/{serviceId}/available-slots`
  `{noOfDays: 7}` → `TimeSelectionDialog` shows 7 days of slots (closed days filtered;
  `canJoinWaitlist` flag carried per day but waitlist UI is **not** implemented —
  `ServiceReservation.kt:112-115`) → `POST {category}/book-appointment`
  `{date, startTime, endTime, salonServiceId|massageServiceId|privateTrainingServiceId}`
  (`ServicesViewModel.bookAppointment:142-249`) → success/failure dialog + toast.
- `ServiceCard.kt:170` has an "Add" button with `/* TODO: Add to reservation */` — a dead
  add-to-cart affordance.

### 3.6 Schedule (`features/schedule/`)

- `GET unified-schedule?noOfDays=7` returns 7 `DailyScheduleDto{date, day, schedules[]}`;
  items are typed `SCHEDULE` (facility activity), `BREAKFAST|LUNCH|DINNER` (meals), `PT`,
  `SALON`, `MASSAGE` (the resident's own service bookings), each with time range, description,
  image, `status` (`DailyScheduleDto.kt`; type-driven title/image/add-on logic at `:30-75`).
- UX: date strip (7 days) → list of that day's items → focusable detail panel
  (`ScheduleDetailPanel.kt`). View-only — no cancel/modify from the TV.
- This is the "unified calendar": activities + meals + booked appointments in one feed; bookings
  made in §3.5 surface here on next visit (screen refetches on entry,
  `MyScheduleViewModel.init`).

### 3.7 Housekeeping (`features/housekeeping/`)

- Four request types, **hardcoded client-side** (`HousekeepingContent.kt:104-114`,
  constants `AppConstants.kt:92-102`): Extra Room Cleaning (`EXTRA_ROOM_CLEANING`),
  Extra Laundry (`EXTRA_LAUNDRY`), Miscellaneous Service (`MISC`), Report Maintenance
  (`MAINTENANCE`, with free-text description area; a mic icon flag exists but no voice input is
  implemented).
- Flow: pick card → auth gate (QR login) → `HousekeepingCalendarDialog` (date selection) →
  confirmation ("charges may apply" copy, `AppConstants.kt:87-90`) →
  `POST housekeeping` `{residentId: "", residentName, selectedDate, unitNo, requestType,
  description}` (`HousekeepingViewModel.kt:34-120`; note `residentId` sent empty — backend
  resolves identity from the TV token) → success dialog. No request history/status view on TV.

### 3.8 Photos (`features/photos/`)

- `GET residents/pictures` → list of URLs (`PhotosViewModel`, refetched on each tab entry).
  These are resident-specific family photos (uploaded by family/admin elsewhere in the platform).
- UX: main image + thumbnail strip; OK on a photo opens full-screen; KenBurns-style fade/scale
  animation; empty state "No photos found" (`PhotosContent.kt`). View-only.
- Separately, `photoFrameWithoutText` viewType reuses `PhotosContent` for the generic
  photo-frame slideshow images from the home payload.

### 3.9 Music (`features/music/`)

- Tracks come from the **home payload**, not a dedicated API: the sub-category's
  `musicSettings.subData[]` (`MusicItemDto{id, name, genre, thumbnail, url}`,
  `TvHomeDto.kt:139-155`). Playback list = that curated set.
- All tracks are **pre-downloaded in the background** when home data loads
  (`HomeViewModel.downloadMusicTracks:226-236`, `core/utils/MusicCacheManager.kt`); playback
  prefers the cached file, falling back to streaming URL (`HomeViewModel.playMusic:260-281`).
- Controls: play/pause, next/previous (wraps around), seek bar with 1 s progress updates —
  a single shared ExoPlayer owned by `HomeViewModel`, so music keeps playing while browsing
  other tabs (released only when the VM clears).

### 3.10 Apps launcher (`features/apps/AppsContent.kt`)

- Grid grouped by `apps[]{category, list[]{name, pkg}}` from the home payload; icons resolved
  from the local PackageManager (`AppItemDto.getAppsIcon`); click launches the package's
  leanback/launch intent (`AppsContent.kt:208-215`). Apps not installed simply show name text;
  there is **no install/update flow** in this app (Play-Store deep-link exists only via the
  ContentProvider backdoor).
- `ClearCredentialsCard` (`AppsContent.kt:275-336`): a "clear app data" tile whose
  `Surface(onClick = { })` is **empty** — built UI, no behavior (the intended
  `preferenceManager.clearAllData()` path is the commented-out logout in `HomeScreen.kt:273-276`).

### 3.11 Calls (`features/calls/CallContent.kt`)

- "Who would you like to call?" with a Concierge card + 3 contacts — **all hardcoded mock data
  with randomuser.me portraits** (`CallContent.kt:79-82`). Call/video buttons render but no
  calling backend (no WebRTC/SIP/Twilio anywhere in the repo). This is a façade/placeholder
  section.

### 3.12 Hidden / system surfaces

- **Developer settings** (7× UP on first category, `CategoryItem.kt:281-301`): renders a password
  field with show/hide toggle — **no submit action and no settings behind it**
  (`DeveloperSettingsDialog.kt`). Half-built.
- **Sleep timer** (power icon → `SleepTimerPopup`, options from home payload `turnOffTimer` or
  default 15/30/45/60 min): selecting an option only fires a debug
  `Toast("Selected $selectedItem")` (`CategoryItem.kt:453-455`) — **no actual screen-off/timer
  logic**. Screen ON/OFF transitions are *observed* and logged (`MainActivity.kt:84-92`,
  `HomeViewModel.onTvTurnOff`) but never initiated.
- **Logging/analytics**: `core/utils/TVLogger.kt` — Firebase Analytics events + screen views,
  Crashlytics, and a local JSON "portal log" file (`app_logs.txt` in filesDir, append-as-JSON-array
  via RandomAccessFile). **No upload path for the portal log file exists in this repo** —
  zip-folder constants and a password (`CommonUtils.kt:10-12`) survive from the hotel app but
  nothing ships the file.

---

## 4. Real-time & refresh behavior

| Mechanism | What it does |
|---|---|
| Socket.IO (`SocketService`) | **Pairing/auth only**: `pairing:create/created/expired/authorized`, `auth:exchange`, `auth:tokens`. No announcement, menu, schedule, or command events are consumed. |
| Broadcasts (in-app) | `SERIAL_NUMBER` (new QR session), `AUTH_TOKENS_SAVED` (silent home reload + dialog continuation), `ONBOARD_UNIT_METHOD` (external provisioning), `ACTION_SCREEN_ON/OFF` (logging), `BROADCAST_BANDWIDTH_SETTINGS` (dead — report API commented out). |
| On-tab-entry refetch | Dining menu (`refresh()` on composition), Alerts (`resetAndRefresh()`), Photos (`fetchPhotos()`), Schedule (on VM init), Specials, Services. |
| Timers | Live TV: 5-min local EPG recompute (`startAutoRefresh`); home slider: 30 s image rotation; clock: 1 s tick. |
| Caching | Home payload → SharedPreferences (fallback when API fails, `SeniorRepositoryImpl.kt:40-57`); live TV categories → Room with 24 h validity; music → file cache. |

Net: **there is no real-time content push**. An "announcement broadcast" reaches the resident only
on next visit to the Alert tab; menu/schedule changes appear on next tab entry.

---

## 5. Build flavors (functional impact only)

`app/build.gradle.kts:77-103` — `staging` / `preproduction` / `production`, differing only in
applicationId suffix, display name, `BASE_URL`, `BASE_URL_SOCKET`. `IS_TESTING` is `false` in all
three and unused in code. Functional notes:

- **staging** and **production** socket URLs are plain `http://` (preproduction is `https://`);
  manifest sets `usesCleartextTraffic="true"` — required for UDP/local streaming but also leaves
  the production websocket unencrypted.
- All flavors share the permissive `MyTrustManager` (accept-all SSL + hostname verifier
  `{ true }`, `core/di/NetworkModule.kt:61-99`) — certificate validation is bypassed in
  production REST traffic too.

---

## 6. Product-split signals (skilled-nursing vs senior-living)

**There are no skilled-nursing / facility-type conditionals in this codebase.** A repo-wide search
for `skilled`, `nursing`, `facilityType`, `assisted`, `memory care`, `independent living` returns
nothing in `app/src/main`. The split levers that *do* exist:

1. **Server-driven menu is the gate.** Because every tab is a `viewType` row in the
   `get-android-categories/{facilityId}` response, a skilled-nursing facility vs a senior-living
   facility can ship entirely different TV experiences *without any client conditional* —
   the product split lives in the backend's per-facility category configuration.
2. **`FacilityData.designations`** (`FacilityModels.kt:38`) — a `List<String>` per facility that
   the TV app receives during registration but **never reads**. This is the only client-visible
   facility-classification field; today it is staff-app domain vocabulary.
3. Per-facility knobs already flowing through: `conciergeNo`, `maxFutureBookingDays`,
   `tvSetupLocations`/`radius` (geofence), `turnOffTimer` options, photo-frame assets,
   music playlists, apps list.
4. Hard-coded assumptions that would need parameterization for a different care setting:
   housekeeping's four request types (§3.7), the salon/massage/private-training service trio
   (§3.5 — category slugs are compiled in), 7-day windows for menu/schedule/slots, and the
   "charges will be added to your account" billing copy.

---

## 7. Observations (TODOs, dead code, half-built, risks)

**Half-built / façade features**
1. **Calls tab is mock** — hardcoded contacts with randomuser.me avatars; no calling stack
   (`CallContent.kt:79-82`).
2. **Sleep timer does nothing** but toast the selection (`CategoryItem.kt:453-455`).
3. **Developer settings dialog** has a password field and no submit/verification or content
   (`DeveloperSettingsDialog.kt`).
4. **ClearCredentialsCard** in Apps has an empty `onClick` (`AppsContent.kt:296`); the matching
   logout body in `HomeScreen.kt:273-276` is commented out — there is **no working resident
   sign-out on the TV**.
5. **Waitlist**: `canJoinWaitlist` is parsed and mapped (`ServiceReservation.kt:112-116`) but no
   waitlist UI/action exists.
6. `ServiceCard.kt:170` — `/* TODO: Add to reservation */` dead button.
7. `MainActivity` stubs: `pausePlayer`, `openVideoPlayer`, `isVideoPlayerIsOpen`,
   `showTopToolBarView`, `startScreenSaver`, `closePlayer` (`MainActivity.kt:179-261`) — leanback
   leftovers; no screensaver exists despite the stub.

**Dead / commented-out code**
8. Channel bandwidth/error report upload commented out (`MainActivity.callAPIFor:222-232`);
   Room table still fills.
9. `VideoPlayerSlider` (ambient video) fully commented (`SliderScreen.kt:184-212`).
10. `LiveTVNewViewModel.sendErrorTvMail` fully commented (`:161-188`) — references hotel-era
    endpoints (`SEND_ERROR_TV_MAIL`, `getHotelRoom()`).
11. Hotel heritage throughout: commented `roomNo/hotelCode/hotelId` headers
    (`SeniorRepositoryImpl.kt:271-273`), `hotelroomtvapp` broadcast actions
    (`BroadcastType.kt:12-13`), `HOTEL_CODE/HOTEL_ID` preference keys, hotel-named log folders
    (`CommonUtils.kt:10-11`), "Dear Guest" salutation constant (`AppConstants.kt:7`).
12. Portal log file (`app_logs.txt`) is written forever but never uploaded or rotated —
    unbounded growth risk on device (`TVLogger.kt:169-208`).

**Hardcoded content**
13. Housekeeping service catalog (4 types) and all registration/housekeeping copy are compiled-in
    English strings in `AppConstants.kt` — no localization, no server config.
14. Home top logo is `AsyncImage(model = "")` (`SliderScreen.kt:84-90`).
15. `CommonUtils.TRACK_THUMBNAIL` default is an Unsplash URL; zip password literal
    `321@Shashigroupzip` in source (`CommonUtils.kt:12`).

**Security-relevant (flagged, per scope)**
16. **Signing keystore `senior_living_jks.jks` committed at repo root** (not opened, per
    constraints) and referenced in `app/build.gradle.kts:32-38`; oddly the keystore *password
    string* is used as the Gradle **property name** (`project.findProperty("321@SeniorLiving")`),
    and the **release build type signs with the debug config**
    (`signingConfig = signingConfigs.getByName("debug")`, `build.gradle.kts:63`) — the declared
    release signing config is unused.
17. Accept-all TLS trust manager + hostname verifier in all flavors (`NetworkModule.kt:74-99`);
    cleartext socket endpoints for staging/production (§5).
18. `WRITE_SECURE_SETTINGS` permission (manifest:10) used to set device name — requires
    privileged grant, signaling managed/sideloaded fleet deployment rather than Play-only.
19. Access/refresh tokens stored in plain SharedPreferences; refresh token logged at error level
    (`SocketService.kt:169`).

**Architecture-doc drift**
20. `docs/architecture/architecture-senior_living_tvapp.md` claims SocketService handles
    "event streaming" beyond pairing and lists a "Home/Channels/Music/... tab" nav as if fixed —
    code shows pairing-only sockets and a fully server-driven tab set. It also omits `apsdk`'s
    vestigial state and the ContentProvider provisioning backdoor.
