# Codebase Analysis — Senior Living Staff App (`senior_living_staffapp`)

> Reverse-engineered functional requirements for PRD work. **Source of truth: code only.**
> Repo: `/home/sathish/projects/devicethread/shashi.ai/senior-living/senior_living_staffapp`
> Stack: React Native 0.84 / React 19 / TypeScript, React Navigation 7, hand-rolled Context+useReducer store, Axios, AWS Cognito (`amazon-cognito-identity-js`), Socket.io client 4.x, FCM + Notifee, AsyncStorage + Keychain.
> Secondary input cross-checked: `docs/architecture/architecture-senior_living_staffapp.md` (consistent with code; code wins throughout).
> **Verified against staging `d4f8169` (2026-06-21).**
>
> **MAJOR STAGING OVERHAUL — read first.** The staff app was substantially expanded. New on staging: a **three-way app-flow model** (MIGRATED Skilled-Nursing / LEGACY designation-task-queue / CHAT_HOME) resolved from backend `staffDirectoryRoles`; a full **Skilled Nursing module** (~45 screens) with care conferences, reports, chat, audio recording, services; **real-time chat** on the Socket.io `/chat` namespace; **Twilio Programmable Voice** calling; **native audio recording** for care conferences; **phone-number + password auth** (MFA *enrollment* removed); security hardening (Cognito IDs via react-native-config, tokens in Keychain); **24h inactivity logout**; production host changed to `api.sal.shashitech.com`; package renamed `com.shashigroup.sal.staff → com.shashigroup.sl.staffapp`. The original (LEGACY) designation-task-queue behaviour documented below still exists for non-migrated roles.

---

## 1. Designation / role matrix (CENTRAL TO PRD)

### 1.1 The full designation enum

`src/services/App/type.ts` — `FacilityDesignation` now defines **20 designations** (staging added `CASE_MANAGER = 'Case Manager'`, `DOCTOR = 'Doctor'`, `DIETITIAN = 'Dietitian'` — the Skilled-Nursing clinical roles). The original 17:

| # | Enum value | Wire value (string) |
|---|---|---|
| 1 | `EXECUTIVE_DIRECTOR` | `Executive Director` |
| 2 | `DIRECTOR_OF_NURSING` | `Director of Nursing` |
| 3 | `CARE_COORDINATOR` | `Care Coordinator` |
| 4 | `ACTIVITIES_DIRECTOR` | `Activities Director` |
| 5 | `CAREGIVER_AM` | `Caregiver AM` |
| 6 | `CAREGIVER_PM` | `Caregiver PM` |
| 7 | `SALON_STYLIST` | `Salon Stylist` |
| 8 | `TRANSPORT_DRIVER` | `Transport Driver` |
| 9 | `HOUSEKEEPING_STAFF` | `Housekeeping Staff` |
| 10 | `DINING_STAFF` | `Dining Staff` |
| 11 | `MAINTENANCE_STAFF` | `Maintenance Staff` |
| 12 | `RECEPTIONIST` | `Receptionist` |
| 13 | `PHYSICAL_THERAPIST` | `Physical Therapist` |
| 14 | `OCCUPATIONAL_THERAPIST` | `Occupational Therapist` |
| 15 | `SOCIAL_WORKER` | `Social Worker` |
| 16 | `PRIVATE_TRAINER` | `Private Trainer` |
| 17 | `MASSAGE_THERAPIST` | `Massage Therapist` |

### 1.1b Three-way app-flow model (NEW — staging)

`src/utils/featureAccess.ts` introduces an `AppFlow` enum — `MIGRATED` (Skilled Nursing), `LEGACY` (the designation task-queue documented in §1.2), `CHAT_HOME`. A staff profile's designation is resolved to a flow via the backend-supplied `staffDirectoryRoles` (`Record<string, string[]>`) against a hardcoded `ROLE_GROUP_FLOW_MAP` (e.g. `Case Manager`/`Doctor`/`Director of Nursing`/`Social Worker` → MIGRATED; `Salon Stylist` etc. → LEGACY; any unlisted key defaults to LEGACY). `appRoute.helper.ts` is now the **central post-auth router**: it persists `APP_FLOW`, routes to the Skilled-Nursing root, the LEGACY Home, or a chat home accordingly, and enforces an **access-denied gate** — a designation in no role group forces logout into `AccessDeniedScreen`. This off-`ROLE_GROUP_FLOW_MAP` client-side gating is flagged in §8 (G4/TD12), and `DietitianView` is unreachable.

### 1.2 What each designation gets in the LEGACY task-queue UI

> Applies to the **LEGACY** flow only. MIGRATED (Skilled Nursing) designations get the SN module (§4b), not this view.

The single switch point is `HomeScreen.renderDesignationView()`:

| Designation | View component | Functional capability | Real-time |
|---|---|---|---|
| Transport Driver | `TransportDriverView` | Transportation request queue with Accept → Start → End lifecycle | yes |
| Housekeeping Staff | `HousekeepingStaffView` | Housekeeping request queue, Accept → Complete | yes |
| Maintenance Staff | `HousekeepingStaffView` with `isMaintenance` prop (`index.tsx:281-285`) | Same queue UI, backend filtered with `isMaintenance: true` query param | yes |
| Salon Stylist | `SalonStylistView` | Today's confirmed appointments + waitlist tab + reschedule ("Move to Slot") flow | yes |
| Private Trainer | `PrivateTrainerView` | Read-only "Today's Sessions" list | yes |
| Massage Therapist | `MassageTherapistView` | Read-only "Today's Sessions" list (CONFIRMED only) | yes |
| **All other 11 designations** (Executive Director, Director of Nursing, Care Coordinator, Activities Director, Caregiver AM/PM, Dining Staff, Receptionist, Physical Therapist, Occupational Therapist, Social Worker) | `DefaultDesignationView` (`DesignationViews/DefaultDesignationView.tsx`) | **Nothing** — static "No data Available" text | no (empty socket-event list) |

### 1.3 Display-name mapping

`getDisplayDesignation()` — `HomeScreen/index.tsx:86-105` — maps designation to the subtitle under the greeting:
Transport Driver → "Driver", Housekeeping Staff → "Housekeeping", Maintenance Staff → "Maintenance", **Dining Staff → "Dining"** (label exists but Dining has no view — falls to default), Salon Stylist → "Salon Specialist", Private Trainer → "Private Training", Massage Therapist → "Massage". Any other designation shows its raw enum string; missing designation shows fallback `"Staff"` (`AppStrings.Home.designationFallback`).

### 1.4 Permissions / profile model (expanded — staging)

`StaffProfileData` was extended on staging with `staffDirectoryRoles` (`Record<string,string[]>` — now the **app-flow driver**, §1.1b), `accessPermissions`, `cName`, `isGoogleLinked`/`isZoomLinked`/`zoomAuthRequired`, `profilePicture`, `dob`, etc. The legacy "neither field is read" note is no longer true: `staffDirectoryRoles` drives flow routing. Fine-grained `accessPermissions` UI gating remains largely unused; flow gating is off the hardcoded `ROLE_GROUP_FLOW_MAP`, not the backend permission tree.

---

## 2. Navigation map

Three-level native-stack tree, all headers hidden, root in `src/navigation/rootstack/index.tsx`:

```
RootStack (rootstack/index.tsx)
├── Splash → SplashStack (splashstack/index.tsx)
│   └── SplashScreen                src/screens/SplashScreen/index.tsx
├── Auth → AuthStack (authstack/index.tsx)
│   ├── SignInScreen                (NOW phone-number + password)
│   ├── ChangePasswordScreen        (FORCE_CHANGE_PASSWORD; param now {username})
│   ├── ForgotPasswordScreen        / ResetPasswordScreen (custom-OTP; param {phoneE164, maskedPhone?})
│   ├── MFAVerifyScreen             (param now {username, phoneNumber, password})
│   └── AccessDeniedScreen          (NEW — designation in no role group → forced logout)
│       (REMOVED on staging: StartMFASetupScreen, MFASetupScreen — MFA *enrollment* dropped)
└── App → AppStack (appstack/index.tsx)
    ├── HomeScreen (LEGACY)         ← now a 2-tab bottom navigator (Home + Messages); designation switch + socket
    ├── CHAT_HOME / ChatHomeScreen, NOTIFICATIONS, ACKNOWLEDGMENT, ROOT_TABS alias
    ├── shared chat sub-graph (Conversation, GroupConversation, GroupDetails, GroupMediaAndFiles,
    │     EditGroup, AddGroupMembers, CreateNewChat, SelectMemberNewGroup, CreateNewGroup, Success, ChatMediaViewer)
    ├── CallScreen                  (Twilio outbound call)
    ├── SelectFacilityScreen, SettingsScreen, ChangeStaffPasswordScreen, DisplayScreen, ThemeSelectionScreen
    └── RescheduleSalonAppointmentScreen

SkilledNursingAppStack (NEW — skilledNursingAppStack/index.tsx + SkilledNursingScreens.ts):
    ├── ROOT_TABS — 4-tab navigator (Home / Messages / Reports / MySchedule; Reports & MySchedule hidden for DOCTOR)
    └── ~45 stack screens: HomeScreen, MessagesScreen, ReportsScreen, MyScheduleScreen,
        ResidentDetailsScreen, ResidentReport, CareTeamScreen, FamilyPartyScreen, ProfileScreen,
        CareConferenceScreen(+Detail/HistoryDetail/Skeleton), CareConferenceScheduleScreen,
        CareConferenceReportScreen(+Detail), AudioRecorderScreen, AudioUploadScreen, AcknowledgmentScreen,
        InterdisciplinaryReportsScreen(+Detail), RehabReportsScreen(+Detail), AdvancedCareDirectiveScreen,
        MedicationListScreen, TestResultsScreen, PDFViewerScreen, ServicesScreen, HousekeepingScreen,
        ExtraLaundryScreen(+Request), MiscServiceScreen(+Request), ReportMaintenanceScreen, MaintenanceRequestScreen,
        SalonServicesScreen/SalonBookingScreen/SalonAppointmentsScreen, TransportationScreen/TransportationRequestScreen,
        UpcomingAppointmentsScreen, + the shared chat screens (route-name aliased between LEGACY and SN stacks)
```

**Per-designation navigation reach:** only Salon Stylist navigates beyond Home + Settings (Home → RescheduleSalonAppointment, `SalonStylistView.tsx:478-481`). Every designation can open Settings from the Home header (`HomeScreen/index.tsx:329-336`) and Reschedule screen header (`RescheduleSalonAppointmentScreen/index.tsx:159-165`). There is **no tab bar and no drawer** — one home surface per role.

---

## 3. Auth & onboarding

### 3.1 Cognito configuration (hardened — staging)

- **Cognito pool/client IDs are no longer hardcoded** — injected per-environment via `react-native-config` (`cognito.config.ts` reads `Config.COGNITO_{STAGING,PRE_PROD,PROD}_{USER_POOL_ID,CLIENT_ID}`). Resolves the prior HIGH (TD4/I4).
- `cognito.storage.ts` is now a **functional synchronous in-memory cache hydrated at splash** (resolves the prior BLOCKER G2 where the adapter always returned null).
- **Tokens (access/id/refresh) + username moved from AsyncStorage to `react-native-keychain`** (`token.manager.ts` via `setGenericPassword`, separate services) — resolves the prior HIGH (TD2/I1).
- CI now adds gitleaks (`.gitleaks.toml`) + Semgrep/Snyk.

### 3.2 Sign-in flow (SignInScreen → cognito.service)

`signIn()` in `src/services/Auth/cognito.service.ts:76-179` returns a discriminated `SignInResult`:

**Login identity changed: phone-number (E.164) + password** (`libphonenumber-js`, `utils/phoneAuth.ts`), not email. **MFA *enrollment* was removed** — `MFASetupScreen`/`StartMFASetupScreen` are deleted; the app now only **verifies** an MFA challenge when the pool issues one, using a new in-repo `OTPInput` component (`react-native-confirmation-code-field` removed).

1. **`SUCCESS`** → save tokens (Keychain), optional "Remember me" (still stores plaintext phone+password — TD3).
2. **`NEW_PASSWORD_REQUIRED`** → `ChangePasswordScreen` (param `{username}`).
3. **`MFA_VERIFY`** (challenge only) → `MFAVerifyScreen` (param `{username, phoneNumber, password}`).
4. **Forgot password — now custom-OTP**: `src/services/Auth/auth.service.ts` → `POST /api/auth/forgot-password` + `POST /api/auth/reset-password`, alongside Cognito `forgotPassword`.
5. **AccessDenied**: a designation in no role group routes to `AccessDeniedScreen` + forced logout (§1.1b).

Validation: phone + full password policy in `src/utils/Validation.ts`.

### 3.3 Session persistence & routing (SplashScreen)

`src/screens/SplashScreen/index.tsx:49-91`: registers FCM token first (`ensurePushToken`, lines 29-46, stored under `FCM_TOKEN`), then: no refresh token → SignIn; token valid → App; expired → `refreshSession()` (`cognito.service.ts:292-328`) then App; refresh failure → clear tokens → SignIn.

### 3.4 Facility binding (multi-facility)

- **Route decision:** `getAuthenticatedAppScreen()` (`src/navigation/appRoute.helper.ts:5-8`) — if AsyncStorage `FACILITY_ID` exists → Home, else → SelectFacilityScreen.
- **SelectFacilityScreen** (`src/screens/App/SelectFacilityScreen/index.tsx`): `GET /api/config/getAllfacility` → dropdown of `FacilityData` (`facilityId`, `facilityName`, lat/lng, `conciergeNo?`, `maxFutureBookingDays?`, `designations?`) → on submit persists `FACILITY_ID` and dispatches `setSelectedFacility`, then `navigation.replace(HOME)`.
- **Home fallback:** if no facility in store, Home calls `GET /api/config/get-facility-data` and **auto-binds the first facility** (`HomeScreen/index.tsx:139-166`) — i.e. single-facility staff never see the picker.
- **Tenancy enforcement:** every API request carries `x-facility-id` plus `Authorization: Bearer` and FCM token headers via the Axios request interceptor (`src/services/Api/index.tsx:64-81`); the socket handshake carries the same (`HomeScreen/index.tsx:217-225`). There is **no in-app facility switcher** after binding — switching requires logout (logout clears `FACILITY_ID`, `src/utils/Localstorage/index.tsx:36-54`).

### 3.5 Profile / designation resolution

`GET /api/staff/profile` (`src/services/App/index.tsx:262-284`) returns `StaffProfileData` including `designation` — the designation comes from the **backend staff record, not from Cognito claims**. Stored in the `user` slice (`src/store/features/user/user.slice.ts`); Home renders greeting "Hello, {name}" + designation label and picks the view.

### 3.6 Token refresh & logout

- Axios response interceptor (`src/services/Api/index.tsx:160-186`): on 401, single-flight `refreshSession()` (shared promise), retry original request once; on refresh failure → `logout()`.
- Logout (Settings): calls `POST /api/staff/logout` first, then `LogUserOut` → Cognito `signOut`, clear tokens, wipe AsyncStorage **except** `FCM_TOKEN`/`CurrentEnvironment`/`DEVICE_ID`, null out store slices, reset nav to SignIn. If the logout API fails, the user stays logged in (alert only).
- **24h inactivity logout (NEW)**: `SessionGuard` (`src/components/App/SessionGuard`) + `useInactivityTimeout` enforce an idle timeout, with a relaunch idle check in SplashScreen and a `TimeoutWarningModal`; the inactivity session timestamp is stored in the Keychain. There is also a `self-delete` flow (`/api/staff/self-delete`).

---

## 4. Functional spec per designation/feature area

### 4.1 Transport Driver — `DesignationViews/TransportDriverView.tsx`

**Data:** `GET /api/resident-transportation?page&limit` (`services/App/index.tsx:122-150`), paginated (10/page), infinite scroll + pull-to-refresh. No date filter — the backend decides the visible set.

**Card contents** (lines 268-327): Activity Request (= `destinationType`), Address, Booking Date (formatted from `appointmentStartTime`), Pickup Time (from `pickupTime`). The `TransportRequestData` model (`type.ts:323-350`) carries much more (round trip, complimentary flag, price, special request, travellingWith, distance, estimated duration, resident & driver objects) — **not displayed**.

**Status machine** (`TransportRequestStatus`, `type.ts:294-300`): `Pending → Approved → Started → Completed` (+ `Rejected`, never shown with an action).

| Card status | Button | API | UI effect |
|---|---|---|---|
| `Pending` | **Accept** (primary color) | `POST /api/resident-transportation/{id}/assign` | merge response data into card in place (`applyActionResponse`, lines 196-225) — i.e. driver self-assigns |
| `Approved` | **Start** (success/green) | `POST /api/resident-transportation/{id}/start-ride` | card updates in place to Started |
| `Started` | **End** (cancel/red) | `POST /api/resident-transportation/{id}/end-ride` | **full list reload** from page 1 with loader (`afterEndRideRefresh`, lines 244-249) — completed ride drops off |
| `Completed` / `Rejected` | none | — | read-only |

Per-row action lock (`actionLoadingId`); failures show global alert "Action Failed". Empty state: "No Request Available".

### 4.2 Salon Stylist — `DesignationViews/SalonStylistView.tsx` + reschedule screen

**Two-segment UI** (lines 58-69): "Today's" (status `CONFIRMED`) and "Waitlist" (status `WAITLIST`), implemented as a swipeable horizontal pager with synced segment buttons. Each segment is lazily loaded once (`hasLoadedRef`, lines 217-220, 336-344), separately paginated (10/page), pull-to-refresh per tab.

**Data:** `GET /api/salon/salon-assigned-requests?page&limit&status&date` with `date = today (YYYY-MM-DD, local)` (`services/App/index.tsx:385-421`; date helper `SalonStylistView.tsx:81-87`). Client re-filters returned rows by status defensively (`getSegmentFilteredData`, lines 100-109).

**Card** (lines 126-201): service name + price ($), Customer Name, Unit No, Booking Date, Time. Waitlist cards add a **"Move to Slot"** outline button → `RescheduleSalonAppointmentScreen` with the appointment as param.

**Reschedule flow** (`src/screens/App/RescheduleSalonAppointmentScreen/index.tsx`):
1. On mount: `POST /api/salon/{serviceId}/available-slots` with `{ date }` = the appointment's own date (lines 67-96) — slots are **same-day only**; no date picker.
2. 2-column grid of `startTime to endTime` slot chips; single select; empty state shows backend message or "No slots available".
3. Submit "Reschedule Appointment" → `PATCH /api/salon/appointments/{id}/move` `{ startTime, endTime }`; on 404/405 the client **falls back to POST** on the same path (env API drift workaround, `services/App/index.tsx:478-504`).
4. Success → `goBack()` + success alert. Validation: must select a slot (alert otherwise); double-submit guarded.

Statuses in the salon model (`SalonAppointmentStatus`, `type.ts:102-107`): `PENDING | CONFIRMED | WAITLIST | COMPLETED | CANCELLED` — the app only surfaces CONFIRMED and WAITLIST; **no complete/cancel action exists for stylists** in this client.

### 4.3 Massage Therapist — `DesignationViews/MassageTherapistView.tsx`

**Read-only** day sheet: `GET /api/massage/appointments?page&limit&status=CONFIRMED&date=today` (`services/App/index.tsx:83-118`; constant `CONFIRMED_STATUS` line 15 of view). Section "Today's Sessions". Card: service name + price, Customer Name, Unit No, conditional Special Request block (lines 201-210), Booking Date, Time (24h → 12h formatter lines 30-45). Infinite scroll; **no pull-to-refresh** (socket trigger reloads with full skeleton, line 149); **no actions** — no complete/cancel/reschedule. Empty: "No Sessions Booked".

### 4.4 Private Trainer — `DesignationViews/PrivateTrainerView.tsx`

Identical pattern to Massage but: `GET /api/private-training/appointments?page&limit&date=today` — **no status filter sent** (`services/App/index.tsx:46-80`), and pull-to-refresh **is** wired (lines 309-314). Card fields identical (service/price, customer, unit, optional special request, date, time). Read-only — no actions. (`PrivateTrainingAppointmentStatus` mirrors massage statuses; unused for filtering.)

### 4.5 Housekeeping Staff & Maintenance Staff — `DesignationViews/HousekeepingStaffView.tsx`

One shared component; Maintenance passes `isMaintenance` which only changes the fetch query param.

**Data:** `GET /api/housekeeping/housekeeping-staff?page&limit[&isMaintenance=true]` (`services/App/index.tsx:312-346`).

**Request types** (`ServiceRequestType`, `type.ts:70-74`): `EXTRA_ROOM_CLEANING`, `EXTRA_LAUNDRY`, `MISC`, `MAINTENANCE` — labels mapped at view lines 57-70 (note typo "Maintainance Request" in `AppStrings.Home.maintenanceRequest`).

**Card:** request type label, Unit No (`residentUnitNo`), optional Description, Requested Date (raw `dateRequested` string, unformatted). The `image?` field on the model is **never rendered**.

**Status machine** (`HousekeepingRequestStatus`): `PENDING → IN_PROGRESS → COMPLETED` via `PUT /api/housekeeping` `{ id, status }` (`services/App/index.tsx:349-381`):

| Status | Button | Next status | UI effect (lines 179-185) |
|---|---|---|---|
| `PENDING` | **Accept Request** | `IN_PROGRESS` | card updates in place (button turns green "Mark as Completed") |
| `IN_PROGRESS` | **Mark as Completed** (green) | `COMPLETED` | card removed from list |
| `COMPLETED` | none | — | (not normally in list) |

Pagination 10/page, pull-to-refresh, silent socket refresh. Per-row updating lock (`isUpdatingId`).

### 4.6 Default (everyone else) — `DesignationViews/DefaultDesignationView.tsx`

Static centered "No data Available". No fetches, no socket events, no actions. This is what the non-MIGRATED/unhandled LEGACY designations get. **Staging additions:** `ActivitiesDirectorView` is now wired into `renderDesignationView`; a `DietitianView` exists but is **NOT wired** (unreachable — §8 G4/TD12).

### 4b. Skilled Nursing module (NEW — MIGRATED flow, ~45 screens)

MIGRATED designations (Case Manager, Doctor, Director of Nursing, Social Worker, …) land on the `SkilledNursingAppStack` 4-tab navigator (Home / Messages / Reports / MySchedule; Reports & MySchedule hidden for DOCTOR). Capabilities:
- **Care conferences**: schedule / detail / history-detail / reports, with **native audio recording** (see audio below).
- **Clinical reports**: interdisciplinary & rehab reports (list + detail), medication lists (`/api/medications/resident`), test results, advanced care directives, resident directory/details, family party (`/api/staff/my-residents`, `/api/residents/{search,acknowledge}`, `/api/reports/getreport/resident`).
- **Service requests** on behalf of residents: transportation, housekeeping, laundry, misc, maintenance, salon booking (`/api/salon/{book-appointment,my-appointments,services}`, `/api/housekeeping/resident`, etc.), upcoming appointments, unified schedule (`/api/schedules/all`, `/api/case-manager/unified-schedule`).
- **Acknowledgment** gate and residency config (`/api/config/residency-details`, `/api/transportation-rules`).
The API service surface grew from ~14 to **~80 functions** across `src/services/App/index.tsx` (~2,300 lines — god-file, TD6), `App/UpcomingAppointments/`, and `Auth/auth.service.ts`.

### 4c. Real-time chat (NEW — `/chat` namespace)

A process-wide `chatSocketService` singleton (`src/services/ChatSocket/index.ts`) on the Socket.io `/chat` namespace, driven by `useChatSocketLifecycle` (mounted in AppStack). Handles `chat:new`/`unread`/`status`/`reaction`, emits `chat:delivered`/`read`, drives a new `unread` store slice and FCM/Notifee chat notifications (channel `chat_notifications`) with deep-link tap routing. **Chat screens are physically shared between the LEGACY and Skilled-Nursing stacks via route-name aliasing** (fragile coupling — TD11). New endpoints: `/api/chat/{conversations,messages,groups,search}`. This runs alongside the existing default-namespace per-designation events (§5.1). A large `src/components/App/*` chat UI library (ChatBubble, MessageListItem/Page, ReactionPicker, SystemEventBubble, etc.) was added.

### 4d. Twilio Programmable Voice (NEW)

`src/services/TwilioService.js` + `CallScreen` provide outbound resident calling (`@twilio/voice-react-native-sdk`). The access token is minted from `Config.TWILIO_FUNCTIONS_URL/token.js`; CallerID = the facility's `conciergeNo` (from Redux facility state). **Trust gap (G10):** confirm the mint function requires backend auth.

### 4e. Native audio recording for care conferences (NEW)

Android **foreground service** (`com.shashigroup.sl.staffapp.AudioRecorderService`) + iOS `PendingRecordingModule`, `react-native-nitro-sound`, a `useAudioUpload` hook, and AsyncStorage recording-state maps (`RECORDING_COUNT_MAP`, `PENDING_RECORDINGS_MAP`). New Android permissions: `RECORD_AUDIO`, `MODIFY_AUDIO_SETTINGS`, `FOREGROUND_SERVICE(_MICROPHONE)`, `CAMERA`, `WAKE_LOCK`, `ACCESS_NETWORK_STATE`.

### 4.7 Settings cluster

- **SettingsScreen** (`src/screens/App/SettingsScreen/index.tsx`): rows Display, Change Password, Logout (red).
- **ChangeStaffPasswordScreen**: current + new + confirm with full strength validation (lines 132-196), `changeCurrentUserPassword()` (`cognito.service.ts:387-444` — pre-refreshes session, reconstructs a `CognitoUserSession` from stored tokens, calls `changePassword`).
- **DisplayScreen**: Theme Selection row + "Font Size" row that deep-links to OS text-size settings (Android `ACTION_TEXT_SETTINGS` intent / `Linking.openSettings()`, lines 42-68).
- **ThemeSelectionScreen**: picks one of 8 background images (`background`…`background7`, lines 45-52) persisted as `APP_BG_PRESET` via the pub/sub helper inside `src/components/CustomBackground.tsx` (subscriber model so all mounted backgrounds update live). Note: despite ThemeProvider supporting light/dark (`src/theme/index.tsx`, `App.tsx` hardcodes `initialMode="light"`), the "theme selection" feature is **background wallpaper choice, not dark mode** — there is no dark-mode toggle in the UI.

---

## 5. Real-time behavior

### 5.1 Socket.io (single connection, Home-scoped)

Created in `HomeScreen` only (`index.tsx:197-265`), torn down on unmount/designation change. Connection: base URL resolved per environment override (`resolveApiBaseUrl`, lines 78-84), `transports: ['websocket']`, `reconnection: true`, auth via `auth.token = Bearer …` **and** `extraHeaders` `Authorization` + `x-facility-id` (lines 217-225).

**Event subscriptions per designation** (`getSocketEventsByDesignation`, lines 43-76):

| Designation | Events |
|---|---|
| Transport Driver | `mobile-TransportationRequest-request-upserted`, `mobile-TransportationRequest-request-deleted` |
| Salon Stylist | `mobile-SalonAppointment-request-upserted`, `-deleted` |
| Massage Therapist | `mobile-MassageAppointment-request-upserted`, `-deleted` |
| Private Trainer | `mobile-PrivateTrainingAppointment-request-upserted`, `-deleted` |
| Housekeeping **and** Maintenance | `mobile-housekeeping-request-upserted`, `mobile-housekeeping-request-deleted` |
| All others | `[]` → socket never connects (guard at line 204) |

**Reaction model:** payloads are ignored entirely — any event increments a `refreshTrigger` counter (line 233) passed as a prop into the active view; each view responds by re-fetching page 1:
- Transport: visible "refresh" mode w/ scroll-to-top (`TransportDriverView.tsx:172-177`)
- Salon: **silent** re-fetch of the active segment only (`SalonStylistView.tsx:352-358`)
- Massage: full reload **with skeleton** (`MassageTherapistView.tsx:145-150` reuses `'initial'` mode — visual flash)
- Private Trainer: refresh-control mode (`PrivateTrainerView.tsx:151-156`)
- Housekeeping: silent (`HousekeepingStaffView.tsx:140-145`)

No client→server emits anywhere; the socket is consume-only.

### 5.2 Push notifications

- FCM token acquired at Splash (§3.3); sent on **every HTTP request** as both `pushToken` and `x-fcm-token` headers (`Api/index.tsx:71-75`) — the backend learns/refreshes the device binding implicitly; token survives logout.
- **Foreground**: `setupForegroundNotificationListener` (`src/services/Notifications/foregroundNotifications.ts`) displays every FCM message via Notifee (Android channel `default_notification_channel_id`, HIGH importance; iOS alert/badge/sound). Title/body resolved from `notification` block or `data.title`/`data.body`/`data.message`.
- **Staging:** FCM bumped to v24; new Notifee `chat_notifications` channel with **chat / care-conference tap-routing** (deep-link via `usePendingChatNavigation`). The FCM **background handler is no longer a strict no-op (now logs)**, and `index.js` captures `getInitialNotification` for quit-launch deep links. Data-only background pushes are still not surfaced (G3, downgraded HIGH→MEDIUM); `onTokenRefresh` remains unregistered (G6).

---

## 6. Schedule / availability management

> **Staging:** the Skilled-Nursing (MIGRATED) flow now includes a **MySchedule tab** and unified-schedule views (`/api/schedules/all`, `/api/case-manager/unified-schedule`, `/api/unified-schedule`) plus care-conference scheduling. The note below applies to the **LEGACY** designation views only.

**For LEGACY designations there is none.**
- No screen or API call exists for setting working hours, availability windows, blocking time, or syncing a calendar.
- `StaffProfileData.isGoogleLinked` (`type.ts:57`) implies the platform's Google-Calendar link (the backend has Google Calendar integration per `senior-living/CLAUDE.md`), but the staff app never reads it and offers no link/unlink flow — availability/calendar linkage must be managed in the admin web app.
- The only availability-adjacent feature is **read** access to salon slot availability (`POST /api/salon/{serviceId}/available-slots`) used solely for waitlist rescheduling (§4.2).
- Appointment lists are hard-pinned to "today" computed on-device (salon/massage/PT) — staff cannot look ahead or back. `FacilityData.maxFutureBookingDays` exists on the model but is unused here (resident-app concern).

---

## 7. Product-split signals (skilled-nursing vs senior-living)

1. **No facility-type conditional exists.** Grep for `skilled|nursing|facilityType|SNF|assisted` finds only the `DIRECTOR_OF_NURSING` enum value (`type.ts:4`). There is no per-facility feature gating beyond the unused `FacilityData.designations?` array (`type.ts:31`) — which *looks* designed to declare which designations a facility supports, but is never consumed.
2. **Clinical roles are now built (staging).** The MIGRATED flow (Case Manager, Doctor, Director of Nursing, Social Worker, …) delivers the Skilled-Nursing module (§4b) — care conferences, clinical reports, chat, resident management. The earlier "clinical half is vapor" observation is **superseded**; the split is now a real per-designation app-flow fork, not an enum-only placeholder. Remaining LEGACY-only designations still land on the operations queue or DefaultDesignationView.
3. **Naming / host (staging)**: app display name "Shashi Care" + "Staff App"; package renamed `com.shashigroup.sal.staff → com.shashigroup.sl.staffapp`; **production API base changed `api.hospitality.andmv.com → https://api.sal.shashitech.com`** (`local.constants.ts`); compile-time `currentEnv` default is `PRODUCTION`; versionName 1.3.6 / versionCode 10.
4. **Per-facility tenancy is header-based** (`x-facility-id`) and the staff profile is facility-scoped (`StaffProfileData.facilityId`), so a facility-type flag on `FacilityData` would be the natural split point — none exists today.

---

## 8. Observations (TODOs, dead code, half-built, inconsistencies)

**Resolved on staging**
0a. Auth tokens moved AsyncStorage → **Keychain** (prior HIGH TD2/I1). Cognito pool/client IDs no longer hardcoded — **react-native-config** injection (prior HIGH TD4/I4). `CognitoStorage` adapter is now functional (synchronous in-memory cache hydrated at splash) — **prior BLOCKER G2 resolved**. FCM background handler no longer a strict no-op (logs) — G3 downgraded HIGH→MEDIUM.

**New on staging**
0b. **God-file** `src/services/App/index.tsx` ~2,300 lines / ~80 functions (TD6). **Cross-flow chat-screen sharing by route-name aliasing** — fragile coupling (TD11). `DietitianView` exists but is **unreachable** (not wired into `renderDesignationView`) and client-side app-flow gating is off the hardcoded `ROLE_GROUP_FLOW_MAP` (G4/TD12). **Twilio token-endpoint trust gap** — confirm the mint function requires backend auth (G10). Unguarded `console.log` expanded (cognito forgotPassword logs print phone + pool/client IDs; foreground FCM payload; index.js boot/background payloads) (TD1). Near-zero test coverage against a much larger surface; MFASetup/StartMFASetup tests were deleted (TD5). `acorn`/`write-good` still in dependencies (TD13).

**Half-built / dead surface (LEGACY)**
1. Several LEGACY designations still land on DefaultDesignationView (operations roles without a built queue).
2. Fine-grained `accessPermissions` still largely unused for UI gating (flow gating is via `staffDirectoryRoles`/`ROLE_GROUP_FLOW_MAP`, §1.4).
3. `FacilityData.designations`, `conciergeNo`, `maxFutureBookingDays`, lat/lng are fetched but unused in this app.
4. `HousekeepingStaffRequestData.image` never rendered; rich transport fields (price, roundTrip, specialRequest, travellingWith, resident contact) never rendered (§4.1).
5. `GOOGLE_PLACES_API_KEY` is defined (`local.constants.ts:6`) but **never used anywhere** — copied from the resident app (which does address autocomplete); it's a hardcoded key in source (security/ops concern: rotate/remove).
6. Unused string `Splash.pushTokenInitFailedLogPrefix`; `TOKEN` storage key cleared on logout but never written.
7. `cognito.config.ts:2-3`: previous Cognito pool/client IDs left commented out.
8. `local.constants.ts:36-40`: four developers' LAN IPs left commented in LOCAL env (Anish/Ronak/aakash/Durgesh) — team-process artifact.

**Inconsistencies / bugs**
9. **No-MFA path after forced password change skips facility routing**: `ChangePasswordScreen` `SUCCESS` branch does `navigation.replace(RootStackScreens.APP)` with **no `screen` param** (line ~209) — always lands on Home (stack initial route) even when `FACILITY_ID` is unset, bypassing `getAuthenticatedAppScreen()` used everywhere else (Home's auto-bind fallback masks this).
10. `moveSalonAppointment` PATCH→POST fallback on 404/405 documents backend API drift between environments (`services/App/index.tsx:478-489`).
11. Pagination meta shapes differ per domain (`meta.totalPages` for transport/massage/PT, `meta.pageCount` for salon, **no meta** for housekeeping which infers `hasMore` from page-fullness — `HousekeepingStaffView.tsx:117`).
12. Massage socket refresh reuses `'initial'` mode → skeleton flash on every real-time event, unlike the silent refresh of salon/housekeeping (§5.1).
13. SignIn password-validation early-return (`SignInScreen/index.tsx:434-442`) forgets `setIsSubmitting(false)` is handled but the error path before `showLoader` returns without resetting `isSubmitting` — actually it returns without `setIsSubmitting(false)` on the `passwordError` branch (line 441 returns; `finally` not yet reached since try hasn't begun) → **login button can soft-lock after one invalid-password attempt** until remount.
14. Salon "Move to Slot" exists only on the Waitlist tab; there is no action to confirm/complete/cancel any salon appointment, and massage/PT have no actions at all — completion bookkeeping must happen elsewhere (admin).
15. Settings title string is "Setting" (singular, `AppStrings.Settings.title`); "Maintainance Request" typo (`AppStrings.Home.maintenanceRequest`).
16. Designation-based socket events for Maintenance reuse the housekeeping channel — a maintenance staffer receives refresh triggers for non-maintenance housekeeping changes too (over-refresh, benign).
17. ThemeProvider supports dark mode but app pins `initialMode="light"` (`App.tsx:37`); StatusBar logic for dark exists but is unreachable. "Theme Selection" actually selects background wallpaper.
18. No `TODO|FIXME|HACK` comments anywhere in `src/` (grep clean) — hygiene is good; gaps are structural, not annotated.

**API surface consumed (staging — ~66 new endpoints; service grew ~14 → ~80 functions across `src/services/App/index.tsx` (~2,300 ln), `App/UpcomingAppointments/`, `Auth/auth.service.ts`)**
LEGACY (existing): `GET /api/config/{getAllfacility,get-facility-data}` · `GET /api/staff/profile` · `POST /api/staff/logout` · `GET/POST /api/resident-transportation` (+`/{id}/assign|start-ride|end-ride`) · `GET /api/housekeeping/housekeeping-staff` · `PUT /api/housekeeping` · `GET /api/salon/salon-assigned-requests` · `POST /api/salon/{serviceId}/available-slots` · `PATCH|POST /api/salon/appointments/{id}/move` · `GET /api/massage/appointments` · `GET /api/private-training/appointments`
NEW (staging): `/api/config/residency-details` · `/api/staff/{my-residents,self-delete}` · `/api/care-conference[/my-conferences[/history]]` · `/api/chat/{conversations,messages,groups,search}` · `/api/residents/{search,acknowledge}` · `/api/salon/{book-appointment,my-appointments[/history],services}` · `/api/housekeeping/resident[/history]` · `/api/medications/resident` · `/api/notifications` · `/api/reports/getreport/resident` · `/api/schedules/all` · `/api/case-manager/unified-schedule` · `/api/auth/{forgot,reset}-password` · `/api/transportation-rules`
Plus Twilio token mint (`Config.TWILIO_FUNCTIONS_URL/token.js`), the Socket.io `/chat` namespace, and Cognito.
