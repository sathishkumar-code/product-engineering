# Codebase Analysis — Senior Living Resident Mobile App (`senior_living_reactnative`)

> **Source of truth:** code at `shashi.ai/senior-living/senior_living_reactnative` (React Native 0.82, TypeScript, Redux Toolkit).
> **Scope:** Senior Living RESIDENT app only ("Shashi Care", `com.shashigroup.sal.resident`). The skilled-nursing resident app (`senior_living_skillednursing_resident`) is a separate repo and out of scope.
> **Method:** full read of navigation, services, store, and every screen. The architecture doc (`docs/architecture/architecture-senior_living_reactnative.md`) was used as secondary input only; where it disagrees with code, code wins.
> **Date:** verified against staging `97f75c4` (2026-06-21).
>
> **MAJOR STAGING OVERHAUL — read first.** This analysis was originally written against the production HEAD where most feature areas were mock/hardcoded. Staging substantially rewired the app: most previously-MOCK areas are now **LIVE** (API-backed), `x-facility-id` is now sent on every request, the Profile subsystem was rebuilt, several new Health/booking flows were added, and foreground push is wired. Section-by-section status flags below have been updated; resolved gaps are marked.

---

## 1. Navigation map

### 1.1 Root structure (`src/navigation/rootstack/index.tsx`)

```
RootNavigator (NavigationContainer)
├── Splash  → SplashStack  (src/navigation/splashstack/index.tsx)
│     └── SplashScreen                      src/screens/SplashScreen/index.tsx
├── Auth    → AuthStack    (src/navigation/authstack/index.tsx)
│     ├── SignInScreen                      src/screens/Auth/SignInScreen/index.tsx
│     ├── ChangePasswordScreen (force-new)  src/screens/Auth/ChangePasswordScreen/index.tsx
│     ├── ForgotPasswordScreen              src/screens/Auth/ForgotPasswordScreen/index.tsx
│     ├── ResetPasswordScreen (code+new pw) src/screens/Auth/ResetPasswordScreen/index.tsx
│     ├── StartMFASetupScreen (intro)       src/screens/Auth/StartMFASetupScreen/index.tsx
│     ├── MFASetupScreen (TOTP secret+code) src/screens/Auth/MFASetupScreen/index.tsx
│     └── MFAVerifyScreen (TOTP at login)   src/screens/Auth/MFAVerifyScreen/index.tsx
└── App     → AppStack     (src/navigation/appstack/index.tsx, createStackNavigator)
      ├── RootTabs → BottomTabNavigator (src/navigation/appstack/BottomTabNavigator.tsx)
      │     ├── HomeTab       → HomeScreen        src/screens/App/HomeScreen/index.tsx
      │     ├── ServicesTab   → ServicesScreen    src/screens/App/ServicesScreen/index.tsx
      │     ├── HealthTab     → HealthScreen      src/screens/App/HealthScreen/index.tsx
      │     ├── MyScheduleTab → MyScheduleScreen  (resets-to-today via tabPress listener)
      │     └── ProfileTab    → App/ProfileScreen/ProfileScreen  (Redux-backed — see §2.4)
      └── pushed screens (staging rewired AppStack to ~100 route/component entries). Notable groups:
            Dining: DiningScreen, RequestFamilyMealScreen, RequestedMealListScreen
            Other Services: OtherServicesListScreen, ServiceRequestListScreen, OtherServiceDetailScreen, ConfirmationScreen
            Salon: SalonAppointmentScreenlist, SalonAppointmentScreen, SalonDetailScreen
            Transportation: TransportationListScreen, TransportationScreen
            Massage Therapy (NEW live): MASSAGE_THERAPY_TYPES_SCREEN, MASSAGE_THERAPY_BOOK_APPOINTMENT_SCREEN
            Private Training (NEW live): PRIVATE_TRAINING_SERVICE_LIST_SCREEN, PRIVATE_TRAINING_BOOK_APPOINTMENT_SCREEN, BOOK_SESSION_CONFIRM_SCREEN
            Activities (NEW): ACTIVITIES_SCREEN
            Health — Cognitive Evaluation (NEW live): COGNITIVE_EVALUATION_LIST_SCREEN, REQUEST_COGNITIVE_EVALUATION_SCREEN, REQUEST_COGNITIVE_EVALUATION_SLOT_SCREEN, COGNITIVE_EVALUATION_DETAILS_SCREEN
            Health — Outside Agency (NEW live): OUTSIDE_AGENCY_SERVICE_LIST_SCREEN, REQUEST_OUTSIDE_AGENCY_SERVICE_SCREEN, OUTSIDE_AGENCY_SERVICE_DETAILS_SCREEN
            Health — Physical Therapy (NEW live): REQUEST_PHYSICAL_THERAPY_EVALUATION_SCREEN, REQUEST_PHYSICAL_THERAPY_SLOT_SCREEN, PHYSICAL_THERAPY_EVALUATION_DETAILS_SCREEN, BOOK_SESSION_CONFIRM_DETAILS_SCREEN
            Health (other): MedicationList, AdvancedCareDirectiveScreen
            Misc: AnnouncementsScreen, UpcomingAppointmentsScreen, NotificationsScreen, ResidentDirectoryScreen, SERVICE_REQUEST_LIST_SCREEN
            Profile (rebuilt under src/screens/App/ProfileScreen/): EDIT_PROFILE, ADDEDIT_MEMBER, MANAGE_ACCOUNT, CARE_TEAM, CHANGE_USER_PASSWORD, DisplayScreen, THEME_SELECTION, Settings, LOGIN_TVApp (QRScannerScreen), UPLOAD_PICTURES
```

**Removed/replaced screens (staging):** the legacy `src/screens/ProfileScreen/**` tree and the unreachable `src/screens/App/ProfileScreen/index.tsx` stub were deleted; the old mock Health screens (`HealthScreen/OutsideAgencyRequest`, `PhysicalTherapy/CognitiveEvaluation`, `PhysicalTherapy/EvaluationScreen`, `PhysicalTherapy/PhysicalTherapyForm`), `MassageTherapy/MassageTherapyScreenlist`, and `PrivateTrainingSessions/{PrivateTrainingScreen,PrivateTrainingSessionsScreenlist}` were removed and replaced by the API-backed flows above. New global components `GlobalAlert/GlobalAlertModal` and `GlobalAlert/GlobalQuestionAlertModal` are mounted in the root stack.

### 1.2 Broken / unregistered navigation targets

**Resolved on staging.** The previously dead-ending private-training, physical-therapy, and massage booking targets were removed when those flows were rebuilt as API-backed screens with proper navigator registration (the 0-byte `PrivateTrainingBookAppointmentScreen` and orphan `MassageTherapyBookAppointmentScreen` no longer exist in their old form). See §3.8 / §3.9 / §3.11.

---

## 2. Personas, auth & onboarding

### 2.1 Persona: Resident (the only login persona)

- Login is **AWS Cognito** (`amazon-cognito-identity-js`), user pool `us-west-1_b4O3UxhMA`, client `2lc4usnui28vd9kn1gcaqq5tt3` (`src/services/Auth/cognito.config.ts`; an older pool id is commented out).
- `signIn()` returns a discriminated union `SUCCESS | NEW_PASSWORD_REQUIRED | MFA_SETUP | MFA_VERIFY` (`src/services/Auth/cognito.service.ts:25-29`); SignInScreen switches on it.
- The backend identifies the resident from the bearer token: profile is fetched from `GET /api/residents/profile` (`src/services/Home/index.tsx:23`) and stored in the Redux `user` slice. Profile fields include `careType`, `unitNo`, `assignedNurse`, `cName`, `familyMembers[]`, facility `location {lat,lng}` (`src/services/Home/type.ts:11-30`).

### 2.2 Family members — data, not a persona

- Family members appear **only as contact data** on the resident profile (`FamilyMember` with `type: 'Family' | 'Emergency'`, `relation`, `phone`, and a `hasPortalAccess` boolean — `src/services/Home/type.ts:32-48`). The app renders them as a horizontal tap-to-call strip on Home (`HomeScreen/index.tsx:756-789`).
- `hasPortalAccess` implies a family-portal concept elsewhere in the platform, but **there is no family-member login or family mode in this app**.

### 2.3 First-run / session flow

1. **Splash** (`src/screens/SplashScreen/index.tsx`): no refresh token → Auth/SignIn; access token valid (with 2-min early-expiry buffer, `token.manager.ts:42-48`) → App; expired → `refreshSession()`, on failure clear tokens → SignIn.
2. **Sign-in** (`SignInScreen/index.tsx:353-423`): email + password with inline validation (email regex; password ≥8 chars, upper, lower, digit, special — `src/utils/Validation.ts`), show/hide password, **Remember Me** stores email+password in the device Keychain (`src/utils/authStorage.ts`) and auto-fills on next launch (`SignInScreen:186-195`).
3. **Forced password change**: `NEW_PASSWORD_REQUIRED` → ChangePasswordScreen → `completeNewPassword()`; may chain into MFA setup/verify (`cognito.service.ts:104-157`).
4. **MFA (TOTP)**: StartMFASetupScreen (explainer) → MFASetupScreen (`associateSoftwareToken` shows the setup key with copy-to-clipboard; user enters 6-digit code; `verifySoftwareToken(code,'MyDevice')` — `MFASetupScreen:63-123`). Returning users with TOTP: MFAVerifyScreen → `sendMFACode(code, …, 'SOFTWARE_TOKEN_MFA')`, then resets nav to App (`MFAVerifyScreen:66-99`).
5. **Forgot password**: ForgotPasswordScreen (email) → Cognito `forgotPassword` sends code → ResetPasswordScreen (6-digit code + new password, same password policy) → back to SignIn.
6. **Token plumbing**: tokens in AsyncStorage (`ACCESS_TOKEN/ID_TOKEN/REFRESH_TOKEN/TOKEN_EXPIRY/COGNITO_USERNAME` — `token.manager.ts`); Axios injects `Authorization: Bearer` + `x-fcm-token` headers, single-flight 401 refresh-and-retry, `logout()` on hard refresh failure. **Multi-tenancy resolved (staging):** the request interceptor now also injects **`x-facility-id`** from the `FACILITY_ID` AsyncStorage key (`Api/index.tsx:87-90`), which is populated on the Home screen after `GET /api/config/residency-details` (`HomeScreen/index.tsx:763-766`). The old critical "x-facility-id never sent" gap (DG-01/S-01) is closed; residual: the header is best-effort, set only after Home loads. Interceptor logging is now `__DEV__`-guarded with Authorization masking (`maskHeaders`).

### 2.4 Profile management — REBUILT (staging)

The entire Profile subsystem was consolidated under `src/screens/App/ProfileScreen/` and is now Redux-backed; the old hardcoded mock profile screens (`src/screens/ProfileScreen/**` and the unreachable `App/ProfileScreen/index.tsx` stub) were deleted (resolves DG-13/DG-21).

- **Profile tab** (`App/ProfileScreen/ProfileScreen`): reads the real profile from Redux (`selectUserProfile`) and branches on `isFamilyMember`. Menu navigates to the rebuilt sub-screens.
- **EditProfile** — edits the resident profile.
- **AddEditMember** — family-member CRUD (`POST /api/residents/{add|edit|remove}-family-member`).
- **ManageAccount** — account management.
- **CareTeam** — now API-backed via `GET /api/staff` (was hardcoded `CARE_TEAM_DATA`).
- **ChangeUserPassword** — real Cognito change-password.
- **Display / ThemeSelection** — theme selection screen (the `DisplayScreen`/theme stubs are largely replaced).
- **Settings** — settings screen.
- **LoginTvApp** (QRScannerScreen) — **NEW**: TV-app QR pairing via `react-native-camera-kit` → `POST /api/tv/pairing/authorize`.
- **UploadPictures** — **NEW**: resident TV image upload (`GET/POST/DELETE /api/residents/pictures`).
- Logout now calls `POST /api/residents/logout` and clears Redux + storage (`clearForLogout`).

---

## 3. Feature areas (functional-spec depth)

Legend: **LIVE** = wired to backend API; **MOCK** = hardcoded/dummy data or no API call; **PARTIAL** = mixed.

### 3.1 Home / Dashboard — LIVE (`src/screens/App/HomeScreen/index.tsx`)

- Header: avatar + static "Good morning" + profile name (Redux); bell icon → NotificationsScreen.
- On focus: fetches profile once (if absent) and dashboard with a 5-second TTL throttle (`:519-589`). Skeleton loaders for every section.
- **Upcoming appointments card**: shows `dashboard.upcomingAppointments` count from `GET /api/residents/dashboard`; tap → UpcomingAppointmentsScreen (which is itself MOCK, see 3.10).
- **Quick actions grid** (4 tiles, `:482-508`): My Calendar (→ MySchedule tab), Dining (→ DiningScreen), Resident Directory (→ ResidentDirectoryScreen), Call Family — tile id '4' dials `dashboard.concierge` via `tel:` (mislabeled "Call Family"; comments call it "caretaker").
- **Family strip**: horizontal scroll of `profile.familyMembers`; tap = phone call (`Linking.openURL('tel:…')`, `:638-655`). No call fallback other than alerts.
- **Announcement preview**: latest announcement from dashboard with icon by `iconType` (weather/event/menu/broadcast default); "View More" → AnnouncementsScreen.

### 3.2 Announcements — LIVE (`src/screens/App/HomeScreen/Announcements/index.tsx`)

- `GET /api/announcements?page&limit` (PAGE_LIMIT 10), infinite scroll via `onEndReached` + `page < totalPages`, pull-to-refresh, skeletons. Read-only (no detail screen, no read/unread state).

### 3.3 Services hub — (`src/screens/App/ServicesScreen/index.tsx`)

Six cards: Dining, Transportation, Housekeeping, Salon Services, Private Training Sessions, Massage Therapy → respective list screens (`:104-123`).

### 3.4 Dining / menus / family meals — LIVE

- **DiningScreen** (`DiningScreen/index.tsx`): two tabs — "All Day Menu" (accordion of categories/items with images from `GET /api/menu?date=YYYY-MM-DD`) and "Specials" (zoomable image of `dailySpecials[0].fileUrl`). Date picked from a pull-down calendar component; menu reloads per date. On fetch error both lists silently empty. There is a leftover hardcoded `DiningData.ts` (unused mock).
- **Request Family Meal** (`RequestFamilyMeal/index.tsx`): config-driven form from `GET /api/menu/price-and-time` — enabled meals (breakfast/lunch/dinner) with per-meal price, serving-time `from` (used as fixed meal time; **no time picker for the user**), and `maxGuest` constraining the guest-count dropdown (default 1–10). Total = guests × price. Submit → `POST /api/family-meal-requests` with `{numberOfGuests, mealType, pricePerPerson, mealDate, mealTime}` → ConfirmationScreen ("Family Meal Requested… chef will prepare…", pops 2). Failure path only `console.warn` — **no user-facing error**.
- **Requested Meal List** (`RequestedMealList/index.tsx`): `GET /api/family-meal-requests` on focus; cards show meal type, $price/person, "dddd MMM DD at hh:mm A". Status enum exists (`PENDING/APPROVED/CANCELLED` in types) but the card **does not render status**, and there is no cancel action.

### 3.5 Salon appointments — LIVE (staging — full booking now persisted)

The salon flow is now fully API-backed via `services/services/salon/index.tsx`:
- **List**: `GET /api/salon/my-appointments` (+ `/my-appointments/history`).
- **Service catalog**: `GET /api/salon/services` (filters `isActive`).
- **Slots**: `GET /api/salon/:serviceId/available-slots`.
- **Book**: `POST /api/salon/book-appointment` — booking now persists (resolves the prior CRITICAL "Book Appointment performs no API call" gap, DG-03/DG-08).
- **Reschedule**: `POST /api/salon/reschedule-appointment` (resolves DG-15/DG-18).
- **Cancel**: `POST /api/salon/cancel-appointment` (replaces the old empty-body `PUT` abuse; resolves DG-19/DG-20).

### 3.6 Housekeeping / "Other Services" — LIVE (`OtherServices/*`)

- **OtherServicesListScreen**: 4 request types — Extra Room Cleaning, Extra Laundry, Miscellaneous Service, Report Maintenance → ServiceRequestListScreen (per-type history).
- **ServiceRequestListScreen**: maps UI type → API enum (`EXTRA_ROOM_CLEANING / EXTRA_LAUNDRY / MISC / MAINTENANCE`, `serviceConfig.ts`), fetches `GET /api/housekeeping?type=` on focus; cards show title + requested `selectedDate`. Statuses exist in the model (`PENDING/COMPLETED/CANCELLED`, priority, assignedTo — `housekeeping/types.ts:28-41`) but the **card renders only title + date** — no status chip, no cancel.
- **OtherServiceDetailScreen** (request form): field set varies by type —
  - RoomCleaning/ExtraLaundry: date picker; validation: date must be **strictly future** ("Please select a future date").
  - Miscellaneous/Maintenance: required description (+ inline error) and optional single photo via camera/gallery with runtime permission handling (`components/App/Services/ImagePicker/AppImagePicker.tsx`).
  - Submit → `POST /api/housekeeping` with `{residentId, residentName: profile.cName, unitNo, requestType, selectedDate?, remarks?, maintanance: [imageUri]}` (note misspelled `maintanance` field; the local image URI is sent, no upload step) → per-type success ConfirmationScreen (pops 2). Errors only `console.warn`.

### 3.7 Transportation — LIVE (richest form in the app)

- **List** (`TransportationListScreen/index.tsx`): `GET /api/resident-transportation/my-requests` on focus; card = destination type, status badge (**Approved** green / **Pending** yellow / **Canceled** red / other gray), address, "EEEE, MMM d at hh:mm a", duration, travelling-with (hidden when Canceled). No cancel/edit actions. Footer "Request Ride" → request form.
- **Request form** (`TransportationScreen/index.tsx`):
  - Destination type dropdown from `GET /api/transportation-rules?isActive=true` (each rule: `locationType`, `isComplimentary`, `complimentaryDistanceLimit`).
  - Address via **Google Places Autocomplete** (hardcoded key in `local.constants.ts:5`; biased to facility lat/lng from profile — fallback coords `22.294640,73.137946` are in **Vadodara, India**, and `components: 'country:in'` restricts results to India `:395-401` — dev leftovers).
  - On address pick, road distance from facility computed via Google **Distance Matrix** (`src/utils/distanceMatrix.ts`); ride is complimentary when `distanceMiles <= rule.complimentaryDistanceLimit` (`:134-138`) — shows "complimentary up to X miles" note; otherwise requires a paid-ride agreement checkbox + extra-charge disclaimer.
  - Appointment date-time picker bounded **tomorrow … +2 months** (`:105-116`); duration dropdown (10/15/30/45/60 min); optional special request; "Travelling with" dropdown from API `travellingWithOptions`; Round Trip checkbox.
  - Validation: destination, geocoded address, future date-time, duration, travelling-with, agreement (if paid) — all with inline error strings from `ValidationMessages`.
  - Submit → `POST /api/resident-transportation` with `status:'Pending'`; success and failure both route to ConfirmationScreen (success pops 2; failure shows "Request failed / unable to save").

### 3.8 Massage Therapy — LIVE (staging)

Now a fully API-backed booking flow (`services/services/massageTherapy/`): services list, `:serviceId/available-slots`, book / reschedule / cancel via `/api/massage/*`. New screens `MASSAGE_THERAPY_TYPES_SCREEN`, `MASSAGE_THERAPY_BOOK_APPOINTMENT_SCREEN` replace the old mock list/catalog and the orphaned `MassageTherapyBookAppointmentScreen`.

### 3.9 Private Training Sessions — LIVE (staging)

Now a fully API-backed booking flow (`services/services/PrivateTraining/`): services, `:serviceId/available-slots`, book / reschedule / cancel via `/api/private-training/*`. New screens `PRIVATE_TRAINING_SERVICE_LIST_SCREEN`, `PRIVATE_TRAINING_BOOK_APPOINTMENT_SCREEN`, `BOOK_SESSION_CONFIRM_SCREEN` replace the old hardcoded catalog and the removed 0-byte booking file (resolves the dead-end `SessionDetailScreen` nav).

### 3.10 My Schedule / Activities — LIVE (staging)

- **MySchedule** is now wired to `GET /api/unified-schedule` (via `services/services/schedule/index.ts`); the tab resets to today on tab press. Resolves the prior schedule-crash gaps (DG-05/DG-06/DG-07).
- **Activities** (new `src/screens/App/Activities/`) is live against `GET /api/schedules` with join/cancel (`POST /api/schedules/:id/join`, `/:id/cancel`).

### 3.11 Health — LIVE for PT / Cognitive / Outside Agency (staging)

Three care flows are now API-backed via the unified `/api/care` endpoints (new `src/services/health/index.tsx`), each with list / details / request / slot-selection screens:
- **Physical Therapy** — `REQUEST_PHYSICAL_THERAPY_EVALUATION_SCREEN`, `REQUEST_PHYSICAL_THERAPY_SLOT_SCREEN`, `PHYSICAL_THERAPY_EVALUATION_DETAILS_SCREEN`, `BOOK_SESSION_CONFIRM_DETAILS_SCREEN`.
- **Cognitive Evaluation** — `COGNITIVE_EVALUATION_LIST_SCREEN`, `REQUEST_COGNITIVE_EVALUATION_SCREEN`, `REQUEST_COGNITIVE_EVALUATION_SLOT_SCREEN`, `COGNITIVE_EVALUATION_DETAILS_SCREEN`.
- **Outside Agency** — `OUTSIDE_AGENCY_SERVICE_LIST_SCREEN`, `REQUEST_OUTSIDE_AGENCY_SERVICE_SCREEN`, `OUTSIDE_AGENCY_SERVICE_DETAILS_SCREEN`.
- `/api/care` supports `/upcoming`, `/history`, `/available-slots`, `/:id`.
- **Still mock (open):** Medication List and Advanced Care Directive remain hardcoded (DG-12). The medication-list backend exists but is not yet consumed here.

### 3.12 Resident Directory — LIVE (staging)

Now API-backed (`HomeScreen/ResidentDirectoryScreen`): `GET /api/residents` (paginated) + `GET /api/residents/getContact`, with a favourite toggle (`PUT residents` favourite). Resolves the prior unused-client gap; rows still offer Call/Message contact actions.

### 3.13 Notifications (in-app) — LIVE (staging)

Now backed by `GET /api/notifications` (`HomeScreen/Notification`). Reached from the bell icons. Notification preferences persistence is still incomplete (DG-14).

### 3.14 Upcoming Appointments — LIVE (staging)

Now backed by `GET /api/unified-schedule` (`HomeScreen/UpcomingAppointment`) rather than the old hardcoded list.

### 3.15 Care Team — LIVE (staging)

Now API-backed (`App/ProfileScreen/CareTeam`) via `GET /api/staff` (was hardcoded `CARE_TEAM_DATA`).

### 3.16 Not present at all

- No brain games, no chat/messaging, no photo gallery, no TV-control/TV-pairing features, no wellness check-ins, no events RSVP, no payments/wallet. (TV is a separate `senior_living_tvapp`; staff flows are in `senior_living_staffapp`.)

---

## 4. Push notifications — foreground wired (staging)

- **Foreground push now works:** `App.tsx` registers `setupForegroundNotificationListener` (`services/Notifications/foregroundNotifications.ts`) which renders FCM `onMessage` payloads as local notifications via the **new dependency `@notifee/react-native`** (creates Android channel `default_notification_channel_id`). FCM token harvesting feeds the `x-fcm-token` header.
- **Still missing (DG-04 partial):** background / terminated notification-tap routing is not wired — only the foreground path renders. `@react-native-firebase/auth` is listed installed-but-unused; `react-native-qrcode-svg` also installed-but-unused.

---

## 5. Product-split signals (vs skilled nursing)

1. **Separate repos**: `senior_living_skillednursing_resident` exists alongside this repo (folder present at `shashi.ai/senior-living/`); this app has **zero references** to "skilled nursing" (grep over `src` confirms).
2. **`careType` on the data model**: `ResidentProfile.careType: 'assisted_living' | string` (`services/Home/type.ts:7,18`) and `Resident.careType: 'assisted_living' | 'memory_care'` (`type.ts:339`), plus `careType` inside `SalonResident` (`services/services/salon/type.ts:76`). **No screen branches on careType** — the split is per-app/per-repo, not per-flag.
3. **No feature-flag system**: no remote config usage, no facility-type conditionals, no per-facility feature toggles anywhere in `src`. Multi-tenancy is now resolved: `x-facility-id` **is sent** on every request (from the `FACILITY_ID` AsyncStorage key, populated after `GET /api/config/residency-details` on Home — §2.3).
4. **Branding**: app name "Shashi Care" (`AppStrings.ts:2`), bundle `com.shashigroup.sal.resident`. **Default env is now `PRE_PRODUCTION`** (`currentEnv = ENVIRONMENT.PRE_PRODUCTION`), with PRE_PRODUCTION corrected to `preproduction-api.sal.shashitech.com` and STAGING to `staging-api.sal.shashitech.com`. `API_BASE_URL_SECOND` was removed (resolves S-09/TD-10).
5. **Hotel platform bleed-through (partially open):** the **PRODUCTION** env URL still wrongly points at the Shashi Hotels backend `api.hospitality.andmv.com` (`local.constants.ts`) — DG-02/S-05 remains open for production builds; staging/pre-prod are correct.

---

## 6. Observations (TODOs, dead code, half-built UI, inconsistencies)

### 6.1 Mock-data screens — mostly resolved (staging)
The earlier large mock surface has been substantially closed. Now API-backed: MySchedule, Activities, Resident Directory, Notifications, Upcoming Appointments, Care Team, the rebuilt Profile, Massage Therapy, Private Training, and the PT/Cognitive/Outside-Agency Health flows — in addition to the previously-live auth, home dashboard, announcements, dining/menu, family meals, salon, housekeeping, transportation, resident profile. **Still mock (open):** Medication List and Advance Care Directive (DG-12); some Display/Settings stubs and a possibly-dead `CustomBackground` preset (DG-16/DG-22).

### 6.2 Salon booking — resolved (staging)
Salon now persists through dedicated endpoints (`book-appointment` / `reschedule-appointment` / `cancel-appointment` / `:serviceId/available-slots` / `my-appointments`) — see §3.5. The old empty-body `PUT` cancel abuse and the no-POST booking gap are gone.

### 6.3 Dead/orphaned artifacts (mostly cleaned up on staging)
- The empty `PrivateTrainingBookAppointmentScreen`, orphan `MassageTherapyBookAppointmentScreen`, and the unregistered `'SessionDetailScreen'`/`'ScheduleScreen'` targets were removed when those flows were rebuilt (§3.8/§3.9/§3.11).
- `NetworkWrapper` (NetInfo offline banner) was **removed** — still no offline handling.
- `fetchResidents`/`fetchResidentContacts` are now consumed by the live Resident Directory.
- Still open: `dayjs` is imported (RequestedMealList, OtherServiceRequest) but **absent from `package.json`** (DG-24/TD-01); 3 date libraries in use (TD-02).

### 6.4 Auth/security notes
- "Remember Me" stores the **plaintext password** in Keychain (acceptable-ish but notable), and `SplashScreen`/interceptors `console.log` auth-flow state; `__DEV__` API logging masks only the Authorization header — request bodies (passwords at sign-in are Cognito-direct, but housekeeping/transport PII bodies) are logged in dev.
- Google Places/Distance Matrix API key hardcoded in source (`local.constants.ts:5`).
- No logout affordance (see 2.4); account stays signed in until refresh fails.
- Old Cognito pool credentials left commented in `cognito.config.ts`.

### 6.5 Consistency gaps
- Two different "Profile" screens (tab vs stack) with different hardcoded identities.
- Status enums defined but unrendered (family meals, housekeeping); salon shows status but with CONFIRMED/PENDING styling commented out.
- Transportation uses `'Canceled'` (one L), salon/housekeeping use `'CANCELLED'`.
- India-biased Places autocomplete + Vadodara fallback coordinates vs Arizona-flavored mock data ("Medical Plaza, AZ").
- Error handling is mostly `console.warn` + silent empty states; only transportation and salon-cancel surface failures to the user.
- Greeting is hardcoded "Good morning" regardless of time (`HomeScreen/index.tsx:688`).
- Header comment in `App.tsx` still says "Sample React Native App".
- Tests: **real coverage now exists** — ~60 `*.test.ts(x)` files plus `jest.setup.js` (was ~1 file). (TD-16)
- **Redux store grew from 2 → 4 slices** (staging): added `globalAlert` and `globalQuestionAlert` slices (driving the new global modals), and the `user` slice gained `facilityData: FacilityData | null` with `setFacilityData`/`selectFacilityData`. New `FacilityData`/`FetchFacilityData` types; `ResidentProfile` gained `isFamilyMember`, `location {lat,lng}`, `familyMembers[]`.
- Transportation now derives facility coordinates from Redux `facilityData` (facility-config API) — the Vadodara hardcoded fallback was replaced with an NYC fallback used only when the facility is missing (resolves TD-05). Transportation create endpoint renamed to `/api/resident-transportation/book-transportation`.

### 6.6 Backend endpoints consumed (inventory)

```
GET  /api/config/residency-details     GET  /api/residents/profile          GET  /api/residents/dashboard
GET  /api/residents (directory)        GET  /api/residents/getContact       PUT  residents favourite toggle
GET/POST/DELETE /api/residents/pictures
POST /api/residents/{add|edit|remove}-family-member   PUT residents/profile-picture   POST /api/residents/logout
GET  /api/staff (care team)            POST /api/tv/pairing/authorize
GET  /api/announcements?page&limit     GET  /api/notifications
GET  /api/menu?date                    GET  /api/menu/price-and-time
POST /api/family-meal-requests         GET  /api/family-meal-requests (+/history)
GET  /api/unified-schedule             GET  /api/schedules  (+ /:id/join, /:id/cancel)
/api/salon/{services, :id/available-slots, book-appointment, reschedule-appointment, cancel-appointment, my-appointments(+/history)}
/api/massage/*                         /api/private-training/*
/api/care (+ /upcoming /history /available-slots /:id)
GET  /api/housekeeping/resident (+/history)   POST /api/housekeeping
GET  /api/transportation-rules?isActive
POST /api/resident-transportation/book-transportation   GET /api/resident-transportation/my-requests/history
```
Plus AWS Cognito (auth), Google Places/Distance Matrix (transportation, now using facility coords), and FCM foreground push via Notifee.
