# Skilled Nursing Resident App — Codebase Analysis (PRD reverse-engineering)

**Repo:** `senior_living_skillednursing_resident` (React Native 0.84, TypeScript)
**App identity:** `app.json` → `SkilledNursing`; `package.json` → `com.shashigroup.sl.resident`; branding string `AppStrings.appName = 'Skilled Nursing'`, footer brand `'Shashi.ai'` (`src/utils/AppStrings.ts:2-3`).
**Backend:** Same Senior Living backend as the sibling resident app — `https://api.sal.shashitech.com` (staging/preprod variants), port 7000 locally (`src/utils/local.constants.ts`). Multi-tenancy via `x-facility-id` header; the product split is **per-repo/per-app**, not a runtime feature flag.
**Analysis date:** verified against `origin/master` HEAD `f22dc60` (2026-08-28). Source of truth: code only (no architecture doc in repo; repo `CLAUDE.md` covers conventions only). Prior checkpoint: staging `026ea88` (2026-06-21); this pass reviews `4734daf..f22dc60` (~30 commits, `kapil/expo_eas_setup` branch merged to `master`) — the platform-foundation.md PLAT-FR-72a passwordless-login pass in between (`4734daf`) is already reflected in §2 below and is not re-derived here.
**Staging additions (read first):** Care Conference module, HIPAA inactivity auto-logoff (SessionGuard — but `IDLE_TIMEOUT` is set to 7 DAYS, neutralising it), force/optional app-update gate, resident acknowledgment gate, resident discharge flow, Terms & Conditions / Privacy WebView, app-wide font scaling, custom-OTP forgot-password (off Cognito), medications endpoint changed to `/api/medications/resident`, and the Message bottom tab is now commented out (4 active tabs).
**This pass's additions (read first):** a mandatory, device-level **Biometric App Lock** gating the whole app (§2), and an **Expo/EAS JS-bundle OTA update pipeline** with a backend eligibility gate and a persistent reload-reminder banner (§8, new). Both are net-new since the passwordless-login pass; neither touches the Cognito auth flow itself.

---

## 1. Navigation map

Root: `src/navigation/rootstack/index.tsx` — `NativeStack` with 3 children + global alert modals (`GlobalAlertModal`, `GlobalQuestionAlertModal` mounted beside the container).

```
RootStack (Splash | Auth | App)
├── SplashStack (src/navigation/splashstack/index.tsx)
│   └── SplashScreen                       src/screens/SplashScreen/index.tsx
├── AuthStack (src/navigation/authstack/index.tsx)
│   ├── SignInScreen                       src/screens/Auth/SignInScreen/index.tsx
│   ├── ChangePasswordScreen (force-new-pw) src/screens/Auth/ChangePasswordScreen/index.tsx
│   ├── ForgotPasswordScreen               src/screens/Auth/ForgotPasswordScreen/index.tsx
│   ├── ResetPasswordScreen                src/screens/Auth/ResetPasswordScreen/index.tsx
│   ├── MFAVerifyScreen                    src/screens/Auth/MFAVerifyScreen/index.tsx
│   ├── AccessDeniedScreen (gesture-locked) src/screens/Auth/AccessDeniedScreen/index.tsx
│   ├── TermsAndConditionsScreen (NEW)     src/screens/Auth/TermsAndConditionsScreen/index.tsx
│   └── DischargedScreen (NEW, gesture-locked) src/screens/Auth/DischargedScreen/index.tsx
└── AppStack (src/navigation/appstack/index.tsx — ~68 registered screens; whole navigator now wrapped in <SessionGuard>)
    ├── RootTabs → BottomTabNavigator      src/navigation/appstack/BottomTabNavigator.tsx
    │   ├── HomeTab      → HomeScreen      src/screens/App/HomeScreen/index.tsx
    │   ├── ServicesTab  → ServicesScreen  src/screens/App/ServicesScreen/index.tsx
    │   ├── (MessegeTab — NOW COMMENTED OUT on staging; Message reached via other paths)
    │   ├── HealthTab    → HealthScreen    src/screens/App/HealthScreen/index.tsx
    │   ├── MyScheduleTab→ MyScheduleScreen src/screens/App/MyScheduleScreen/index.tsx
    │   └── (ProfileTab — COMMENTED OUT; Profile reached via Home header avatar)
    │   → 4 active tabs (Home, Services, Health, MySchedule)
    ├── Health area
    │   ├── MedicationList                 …/HealthScreen/Medication/index.tsx
    │   ├── TestResultsScreen              …/HealthScreen/TestResult/index.tsx
    │   ├── PDFViewerScreen                …/HealthScreen/TestResult/PDFViewerScreen/index.tsx
    │   ├── AdvancedCareDirectiveScreen    …/HealthScreen/AdvancedCareDirective/index.tsx
    │   ├── AddScanDocumentScreen          …/AdvancedCareDirective/AddScanDocumentScreen/index.tsx
    │   ├── CareConferenceScreen (NEW)     …/HealthScreen/CareConferenceScreen/index.tsx
    │   ├── CareConferenceDetailScreen (NEW) …/CareConferenceScreen/CareConferenceDetailScreen
    │   ├── CareConferenceHistoryDetailScreen (NEW) …/CareConferenceScreen/CareConferenceHistoryDetailScreen
    │   ├── RehabScreen                    …/HealthScreen/Rehab/index.tsx
    │   ├── RehabHistoryScreen             …/Rehab/RehabHistory/index.tsx
    │   ├── RehabReportScreen              …/Rehab/RehabHistory/RehabReport/index.tsx
    │   ├── MessageRehabTeamScreen         …/Rehab/MessageRehabTeam/index.tsx
    │   ├── BrainGamesScreen               …/HealthScreen/BrainGames/index.tsx
    │   ├── PhysicalTherapy* (5 screens)   …/HealthScreen/PhysicalTherapy/…  ← ORPHANED, see §7
    │   ├── CognitiveEvaluation* (4)       …/HealthScreen/CognitiveEvaluation/… ← ORPHANED
    │   └── OutsideAgency* (3)             …/HealthScreen/OutsideAgency/…   ← ORPHANED
    ├── Messaging
    │   ├── MessageScreen (care-team list) …/HomeScreen/MessageScreen/index.tsx
    │   └── ConversationScreen             …/MessageScreen/ConversationScreen/index.tsx
    ├── Home satellites
    │   ├── NotificationsScreen            …/HomeScreen/Notification/index.tsx
    │   ├── AnnouncementsScreen            …/HomeScreen/Announcements/index.tsx
    │   ├── UpcomingAppointmentsScreen     …/HomeScreen/UpcomingAppointment/index.tsx
    │   ├── ActivitiesScreen               src/screens/App/Activities/index.tsx
    │   └── ResidentDirectoryScreen        …/HomeScreen/ResidentDirectoryScreen/index.tsx ← DEAD (no entry point)
    ├── Services
    │   ├── DiningScreen / RequestFamilyMealScreen / RequestedMealListScreen
    │   ├── TransportationListScreen / TransportationScreen (booking)
    │   ├── OtherServicesListScreen / OtherServiceDetailScreen / ServiceRequestListScreen / ConfirmationScreen (housekeeping)
    │   ├── Salon: SalonAppointmentScreen / SalonAppointmentTypesScreen / SalonBookAppointmentScreen
    │   ├── Massage Therapy (3 screens) ← hidden from Services hub (commented), reachable only via reschedule
    │   ├── Private Training (4 screens) ← hidden from Services hub (commented), reachable only via reschedule
    │   └── BookSessionConfirmScreen / BookSessionConfirmDetailsScreen (shared success/failure screens)
    ├── Home satellites (NEW gates)
    │   ├── AcknowledgmentScreen (NEW, gesture-locked) …/HomeScreen/Acknowledgment/index.tsx
    └── Profile cluster (pushed from Home header)
        ├── ProfileScreen, EditProfileScreen, ManageAccountScreen, AddEditMemberScreen
        ├── CareTeamScreen, SettingsScreen, ChangeUserPassword, DisplayScreen, ThemeSelectionScreen
        ├── ModifyFontSizeScreen (NEW), WebViewScreen (NEW — T&C / Privacy / Acknowledgment PDF)
        ├── NotificationsSettingScreen ← registered, entry commented out in ProfileScreen
        ├── QRScannerScreen (Login TV App), UploadImagesScreen (TV photo frames)
```

\* Tab key is literally `MessegeTab` (typo) — `BottomTabNavigator.tsx:25`.

---

## 2. Personas, auth, onboarding, profile

**Persona: one login type — the resident (patient).** There is no separate family-member login. Family members are *data* managed by the resident (max 4, `ManageAccountScreen/index.tsx:84`), and `ResidentProfile.isFamilyMember` exists (`src/services/Home/type.ts:28`) suggesting the backend can mark a family-member-operated account, but the app renders one persona. The roster persona ("staff") lives in the separate staff app.

**Auth method (AWS Cognito, `amazon-cognito-identity-js`):**
- **Phone-number + password login** (E.164, `libphonenumber-js`, country picker, Android phone-hint autofill via `react-native-otp-verify` `requestHint()`) — `src/screens/Auth/SignInScreen/index.tsx`. This is a key divergence from the senior-living sibling, which logs in with **email**.
- Sign-in outcomes handled (`src/services/Auth/cognito.service.ts`): `SUCCESS` → tokens saved (access/id/refresh + username in AsyncStorage, `token.manager.ts`); `NEW_PASSWORD_REQUIRED` → ChangePasswordScreen (FORCE_CHANGE_PASSWORD challenge); `mfaRequired`/`totpRequired` → MFAVerifyScreen (SMS code entry with `react-native-confirmation-code-field`, Android auto-read via `getHash`/`startOtpListener`).
- **No MFA *setup* screens** — sibling has `StartMFASetupScreen`/`MFASetupScreen` (TOTP enrollment); this app only verifies.
- **Forgot password — now custom-OTP (staging), off Cognito**: new `src/services/Auth/auth.service.ts` calls backend `POST /api/auth/forgot-password` and `POST /api/auth/reset-password`, plus a `check-temp-password` pre-check (`services/App/index.ts`). Adds a new `OTPInput` component and Android SMS-consent autofill via `@eabdullazyanov/react-native-sms-user-consent`. (SignIn now also requires T&C agreement — see below.)
- "Remember Me" persists credentials (incl. password) via `utils/authStorage.ts`.
- **Terms & Conditions / Privacy WebView (NEW)**: `react-native-webview` powers `TermsAndConditionsScreen` (AuthStack) and `WebViewScreen` (AppStack); SignIn requires T&C agreement before proceeding.
- **HIPAA inactivity auto-logoff (NEW, effectively inert)**: `SessionGuard` wraps the AppStack; `useInactivityTimeout` runs a foreground `setTimeout` + AppState background-idle check; last-active epoch is stored in the encrypted Keychain (`sessionTimestamp.ts`); idle is re-checked on Splash across kill+relaunch, with a `TimeoutWarningModal`. **WARNING:** `IDLE_TIMEOUT` is set to **7 DAYS** (`local.constants.ts:11`) with realistic 15s/60s test values commented out — the control is effectively neutralised (flagged HIGH in §7).
- **Force/optional app-update gate (NEW)**: `useForceUpdate` runs on Splash before auth, calling `GET /api/app-version/:bundleId` (via `react-native-device-info` build number); force-update shows a non-dismissable alert that opens the store (iOS App Store ID 6768239464 / Android `market://com.shashigroup.sl.resident`) and blocks routing.
- Session: Splash checks refresh token → access-token expiry → silent `refreshSession()` → App or SignIn (`src/screens/SplashScreen/index.tsx`); now also runs the force-update + idle bootstrap. Axios layer auto-refreshes on 401 once, hard-logout on failure.
- **AccessDeniedScreen**: shown when `/api/config/residency-details` 404s on Home focus — i.e. the Cognito user is valid but no longer a resident of any facility → forced logout into a gesture-locked dead-end.
- **Resident acknowledgment gate (NEW)**: residents with `profile.acknowledgement === false` are routed to the new `Acknowledgment` screen (PDF from `facilityData.acknowledgementPdf` + agree checkbox); `PATCH /api/residents/acknowledge` on submit. Gate in HomeScreen (~lines 860-911).
- **Resident discharge flow (NEW)**: `fetchResidentProfile` returning `status === 'Discharged'` triggers logout + Redux clear + nav reset to the new `DischargedScreen` (discharge date, facility name, concierge call) — reads `facilityData.facilityName`/`conciergeNo`. Gate in HomeScreen (~lines 877-903).
- **Biometric App Lock (NEW, mandatory, device-scoped)**: `AppLockGate` (`src/components/App/AppLock/index.tsx`) wraps the *entire* app (mounted around `RootNavigator` in `App.tsx`, i.e. before Auth/App routing, not just inside AppStack) and renders an opaque lock-screen overlay whenever Redux `appLock.status === 'locked'`. Gate resolution is `resolveGateTarget()` (`src/utils/auth/enterApp.ts:14-73`), called from `MFAVerifyScreen.proceedAfterAuth()` right after first OTP success and from `SplashScreen.enterAppMaybeGated()` (`SplashScreen/index.tsx:94-140`) at every cold start: **device Keychain enrollment is checked first and always wins** (`isBiometricEnrolled()`, Keychain service `com.shashigroup.sal.resident.app_lock`) — only an *unenrolled* device consults the new `FacilityData.residentBiometricEnabled` flag, and when that's `true` the result is always `'enroll'`, never `'skip'`; on any error the gate fails closed to `'enroll'`. There is **no opt-out**: `BiometricEnrollScreen` (`src/screens/Auth/BiometricEnrollScreen/index.tsx`) has no dismiss/skip action — its header back arrow and Android hardware back both sign the resident out (`signOutFromGate`) rather than admitting an unenrolled device. `src/services/Biometrics/biometrics.service.ts` implements enroll/unlock/clear against `react-native-keychain` (biometry-or-device-passcode ACL, `AES_GCM` storage, write-then-read-back to force an OS prompt on both platforms). **A code comment states this is a "device-level gate, not tied to any resident account"** and that `resolveGateTarget()` "Mirrors staffapp's resolveGateTarget exactly" (`enterApp.ts:14-20`) — i.e. the SN team explicitly ported `senior_living_staffapp`'s device-scoped design rather than building an account-scoped equivalent. Re-lock triggers, mirroring staffapp: background duration past `FacilityData.residentLockScreenTimeout` seconds (new `Home/type.ts:513-514` field; falls back to `APP_LOCK_GRACE_MS = 15_000` = 15 s, `local.constants.ts`), or foreground touch-inactivity via a `PanResponder` capture layer that resets the idle timer on every touch without consuming it. `SplashScreen.enterAppMaybeGated()` attempts a **silent** unlock directly over Splash for an already-enrolled device (the native Face ID/fingerprint sheet appears with no intermediate "tap to unlock" screen) before falling back to the lock-screen overlay on failure. **Voluntary Sign-Out does not clear the device's enrollment**: `ProfileScreen`'s Sign Out action calls plain `LogUserOut` (`ProfileScreen/index.tsx:80,119,132,171`); only `signOutFromGate` (`src/utils/auth/LogUserOut.ts`) — invoked from the lock screen's "Back to Login", an `invalidated`-reason unlock failure, or `BiometricEnrollScreen`'s own back action — calls `clearBiometricEnrollment()`. **Net effect: same device-scoped, not account-scoped, characteristic already flagged for the staff app** — once any resident enrolls a device, a different resident who later signs in on that same device inherits the unlock, and an ordinary sign-out does not reset this. See platform-foundation.md PLAT-FR-77 (cross-referencing PLAT-FR-75) and §7 item 19 below. Redux slice: `src/store/features/appLock/appLock.slice.ts` (`status`, `isBiometricEnabled`, `suppressLockUntil`, not persisted — recomputed by `resolveGateTarget()`/`SplashScreen` each entry point). Test coverage: `biometrics.service.test.ts` (353 lines) and `appLock.slice.test.ts` are thorough; `AppLockGate` itself (`components/App/AppLock/index.tsx`, the ~300-line component owning all the timer/AppState logic) ships with **no dedicated test**.

**Bootstrap sequence (Home, on focus):** facility data (`/api/config/residency-details` → persists `FACILITY_ID`, fed into the `x-facility-id` header for all later calls) → resident profile (`/api/residents/profile`) → dashboard (`/api/residents/dashboard`, 5s TTL throttle) — `HomeScreen/index.tsx:786-888`.

**Profile (resident):** name, phone, dob, unitNo, profile picture, careType, assignedNurse, familyMembers, location (`ResidentProfile`, `src/services/Home/type.ts:11-38`). Profile menu (`ProfileScreen/ProfileScreen/index.tsx`): Edit Profile, Manage Account (family members), **Login to TV App** (QR pair), Care Team, Display (theme), Settings (change password), Sign Out (`POST /api/residents/logout`), **Delete Account** (`DELETE /api/residents/self-delete`).

**Family member management:** ManageAccountScreen lists `profile.familyMembers` of `type === 'Family'`; add/edit via AddEditMemberScreen (first/last name, relation — options come from `facilityData.familyRelations` with already-used single relations removed — phone with country detection, email); remove via `POST /api/residents/remove-family-member`. Add/edit endpoints: `/api/residents/add-family-member`, `/api/residents/edit-family-member` (`src/services/Profile/staff/index.tsx`).

---

## 3. Feature areas (functional spec)

### 3.1 Home dashboard (`src/screens/App/HomeScreen/index.tsx`)
- Header: avatar (→ ProfileScreen), welcome + name, bell (→ NotificationsScreen).
- "Upcoming appointments" count card → UpcomingAppointmentsScreen.
- 4 quick-action tiles (hard-coded): My Calendar → ActivitiesScreen; Dining → DiningScreen; **Rehab → RehabScreen**; **Care Team → CareTeamScreen** (`onCardPress`, lines 891-923). Comments show tile 1 previously routed to MySchedule tab and tile 4 previously dialed a concierge.
- Family/contacts strip from `dashboard.contacts` — tap = phone call (`tel:` Linking).
- Latest announcement preview → AnnouncementsScreen (paginated list, `/api/announcements`).
- A floating "message" FAB is commented out (lines 1161-1171) — superseded by the Message tab.

### 3.2 Health hub (`src/screens/App/HealthScreen/index.tsx`)
Hard-coded card list (staging): **Medication List, Test Results, Advanced Care Directive, Care Conference Summaries**. The **Rehab and Brain Games cards are now commented out** of the hub (the Rehab/BrainGames screens remain registered/reachable from other paths). (The sibling SL app's hub instead lists Medication, ACD, Physical Therapy Evaluation, Cognitive Evaluation, Outside Agency.)

#### Medications (`Medication/index.tsx`, `GET /api/medications/resident`)
- **Endpoint changed (staging):** `GET /api/medications` → **`GET /api/medications/resident`**, now taking `cName` + an optional `status` filter (`health/index.tsx:529`; new `MedicationStatus` type). Medication screen heavily reworked (~909 lines). Read-only paginated list; each item: medicationName, strength • route • frequency summary, expandable detail (strength, route, directions, prescribing doctor, start date). No actions (no refill, no adherence logging).

#### Test Results / Lab Reports (`TestResult/index.tsx`, `GET /api/lab-reports`)
- Paginated list keyed by `reportS3Key`; fields: testName, patientName, reportDate, S3 `signedUrl`.
- Debounced (1s) **server-side search**; infinite scroll by `meta.totalPages`.
- Tap → PDFViewerScreen (`react-native-pdf` viewer, title = test name). View-only; no download/share/sign.

#### Advance Care Directives (`AdvancedCareDirective/index.tsx`, `GET/POST /api/advance-care-directives`)
- Paginated list of the resident's directive documents (title, fileName, createdAt) → tap opens PDFViewerScreen.
- "+ Add" → AddScanDocumentScreen: two paths — (a) pick an existing **PDF** via `@react-native-documents/picker` (PDF type only); (b) **scan**: capture images via `react-native-image-crop-picker`, client-side conversion to a single PDF using **jsPDF + react-native-blob-util** (`AddScanDocumentScreen/index.tsx:123-135`). Required: a title and a selected document (submit disabled otherwise). Upload = multipart `POST /api/advance-care-directives`; success/failure routes to the shared confirmation screen. **No e-signature flow** — upload-only.

#### Rehab (`Rehab/index.tsx`) — the clinical centerpiece
- Two segments (TabView, swipe disabled): **My Rehab Schedule** and **Rehab Team**.
- Schedule: collapsible date calendar (today → +1 month), paginated `GET /api/rehab/appointments/my-appointments?date=` ; cards show therapy name + code (PT/OT/ST style codes), start time + duration, location. Read-only — residents cannot book or cancel rehab; scheduling is staff-driven.
- "Rehab History" → RehabHistoryScreen (paginated `…/my-appointments/history`); tap → RehabReportScreen: resident name, therapy (name/code/duration), date/time/location, "Message Note" section showing therapy description + appointment notes. (Type has `summary` field; report renders description/notes.)
- Team: `GET /api/staff?designationGroup=rehab` — list of rehab staff (photo, name, designation). Footer: "Message the Rehab Team" → MessageRehabTeamScreen — **topic + message** free-text form, `POST /api/rehab/rehab-message`, submit disabled until both non-empty, success/failure → confirmation screen with "Responses may take up to 24 hours" hint on the team tab.

#### Care Conference (NEW — `…/HealthScreen/CareConferenceScreen/`)
- Residents view upcoming + historical care-conference summaries, drill into detail / history-detail, and join in-person or virtual conferences. Screens: `CareConferenceScreen` (642 ln), `CareConferenceDetailScreen` (454 ln), `CareConferenceHistoryDetailScreen`, `CareConferenceSkeleton`.
- Service `src/services/App/index.ts`: `fetchMyCareConferences` (`GET /api/care-conference/my-conferences`), `fetchMyCareConferenceHistory` (`GET /api/care-conference/my-conferences/history`).
- Integrated into the Health hub, MySchedule, and UpcomingAppointment surfaces. New types under `src/services/App/type.ts` (CareConference*). Ships without dedicated tests.

#### Brain Games (`BrainGames/index.tsx`, `GET /api/brain-games`)
- Curated catalog (icon, name, star rating, categories); tap deep-links out to the App Store (iOS) / Play Store (Android) URL per game. Pure referral — no in-app games.

#### Physical Therapy / Cognitive Evaluation / Outside Agency (orphaned but complete)
12 screens + full service layer exist (`src/services/health/index.tsx`): upcoming list + history (`GET /api/care/upcoming|history?type=PHYSICAL_THERAPY|COGNITIVE_EVALUATION|OUTSIDE_AGENCY`), available slots (`GET /api/care/available-slots?type&date`), booking (`POST /api/care`). Flow: request screen (reason/notes) → slot screen (calendar + slot picker, slots debounced per date) → booking → confirm-details screen with date/time/location. Status model: `REQUESTED | APPROVED | CANCELLED` (`src/services/health/type.ts:22`). Outside-agency adds `serviceType` (PT or cognitive) + `agencyName`. **None of these screens are navigated to from any live screen** — they are registered in the AppStack but unreachable (carried over from the senior-living app where they form the Health hub). Treat as latent/disabled features, not shipped UX.

### 3.3 Messaging / Case-manager interactions (Message tab — NOW COMMENTED OUT)
> **Staging:** the `MessegeTab` bottom-tab entry is commented out (`BottomTabNavigator.tsx:154-182`) in addition to the already-commented Profile tab — only 4 active tabs remain. The chat screens/services below still exist in the repo and are reachable via deep-link from chat notifications, but the tab entry point is gone.
- **MessageScreen** (`HomeScreen/MessageScreen/index.tsx`): fixed care-team contact list from `GET /api/chat/care-team-contacts` with exactly four roles — **Case Manager, Social Worker, Doctor, Dietitian** (`CareTeamRole`, `src/services/Chat/type.ts:38-42`). Each row: member name/photo, role label, last message preview, unread badge. Tap → ConversationScreen (existing `conversationId` or fresh).
- **ConversationScreen** (778 lines): cursor-paginated history (`GET /api/chat/conversations/:id/messages`), optimistic send, two send paths — existing conversation (`{conversationId, text}`) vs first-message (`{to:{cName, role}, text}` → conversation created server-side), both `POST /api/chat/messages`.
- **Real-time** via Socket.io namespace `/chat` (`src/services/ChatSocket/index.ts`), auth Bearer token + `x-facility-id` header. Events: `chat:new`, `chat:status`; emits `chat:delivered`, `chat:read`. Message status ladder SENT → DELIVERED → READ with per-message tick icons; read receipts emitted on open/focus.
- Socket lifecycle is app-wide (`src/hooks/useChatSocketLifecycle.ts`, mounted in AppStack): messages for non-active conversations raise a **local chat notification** (notifee, channel `chat_notifications`) whose tap deep-navigates to the conversation (§4).
- Messaging is resident↔staff only (roles `STAFF|RESIDENT|ADMIN`); no resident↔resident or group creation UI (GROUP type exists in types only).

### 3.4 My Schedule tab (`MyScheduleScreen/index.tsx`, 1,461 lines)
- Unified day agenda from `GET /api/unified-schedule?date=` (`src/services/services/schedule/index.ts`), normalizing heterogeneous items: ACTIVITY, SALON, MASSAGE, PT (private training), TRANSPORTATION, CARE, REHAB (therapy `code` surfaced for rehab) into one card model with status (`ACTIVE/INACTIVE`, request statuses, transportation status).
- Tab re-press resets to today (`BottomTabNavigator.tsx:217-223`). Detail bottom-sheet for `DETAILED_TYPES = ['SALON','MASSAGE','PT','TRANSPORTATION']`.
- Cancel: confirmation dialog then type-routed cancel — salon `POST /api/salon/cancel-appointment`, massage `/api/massage/cancel-appointment`, private training `/api/private-training/...` (`appointmentActions.ts`). Rehab/care/activity items are not cancellable here.

### 3.5 Activities / My Calendar (`Activities/index.tsx`, 1,350 lines)
- Date-driven list from `GET /api/schedules?date=` with **Join / Cancel join** actions: `POST /api/schedules/:id/join` and `/cancel` behind a confirmation dialog, with optimistic `joined` toggle (lines 275-315). This is community-event RSVP, distinct from MySchedule's read/cancel agenda.

### 3.6 Upcoming Appointments (`HomeScreen/UpcomingAppointment/index.tsx`)
- All-upcoming view of the unified schedule (no date param), filtered client-side; supports **Reschedule** routing per type: SALON → SalonBookAppointmentScreen (isReschedule), MASSAGE → MassageTherapyBookAppointmentScreen, PT → PrivateTrainingBookAppointmentScreen (lines 344-377). This is the only reachable path into the massage/private-training booking screens.

### 3.7 Services tab (`ServicesScreen/index.tsx`)
Visible cards: **Dining, Transportation, Housekeeping, Salon Services**. Massage Therapy and Private Training Sessions cards are **commented out** (lines 71-82).

- **Dining** (`DiningScreen/index.tsx`): date-pull calendar; menu by date (`GET /api/menu?date=`) grouped by category, daily specials, resident **diet plan cards** (`DietInfoCard` — `dietType`; SN-only component); meal price/time from `/api/menu/price-and-time`. "Request Family Meal" → guest count (dropdown), date+time, creates `POST /api/family-meal-requests`; "Requested meals" list + history (`/api/family-meal-requests`, `/resident/history`).
- **Transportation** (`Transportation/…`): list screen with Upcoming/History segments (`/api/resident-transportation/my-requests[/history]`) and status badges; booking screen — destination type from facility **transportation rules** (`GET /api/transportation-rules`), Google Places address autocomplete, road-distance from facility via Google Distance Matrix (`utils/distanceMatrix.ts`), complimentary-vs-chargeable determination against `complimentaryDistanceLimit`, books via `POST /api/resident-transportation/book-transportation`.
- **Housekeeping / Other Services** (`OtherServices/…`): 4 request types — Extra Room Cleaning, Extra Laundry, Miscellaneous, Maintenance (`services/services/housekeeping/serviceConfig.ts`); request detail w/ date + optional image (ImagePicker), `POST /api/housekeeping`; per-type request list with status `PENDING|COMPLETED|CANCELLED` (`/api/housekeeping/resident[/history]`); shared ConfirmationScreen.
- **Salon** (`SalonAppointment/…`): services catalog (`/api/salon/services`), multi-service selection w/ detail modal, slot picker (`/api/salon/:serviceId/available-slots`, `/api/salon/slots`), book/reschedule/cancel (`/api/salon/book-appointment`, `/reschedule-appointment`, `/cancel-appointment`), my-appointments + history.
- **Massage / Private Training** (hidden): full service layers exist (`/api/massage/*`, `/api/private-training/*` incl. reschedule/cancel/history) — functional but de-merchandised.

### 3.8 TV-app companion features (Profile)
- **Login to TV App**: QR scan (`react-native-camera-kit`) of the TV's pairing QR → `POST /api/tv/pairing/authorize` with `{sessionId}` (QR payload >20 chars) — authorizes the in-room TV session against the resident account (`LoginTvApp/QRScannerScreen.tsx`, `services/Profile/tvPairing/tvPairingApi.ts`).
- **Upload Pictures**: manage the TV photo-frame gallery — `GET/POST/DELETE /api/residents/pictures` (`UploadPictures/UploadImagesScreen.tsx`, `services/Profile/uploadPictures/tvImages.ts`).

### 3.9 Care Team directory (`ProfileScreen/CareTeamScreen/index.tsx`)
- `GET /api/staff` list (photo, name, designation) with **Call** action (`tel:`); message action commented out (lines ~165-170). Distinct from the chat care-team (which is the fixed 4-role set).

### 3.10 Notifications center (`HomeScreen/Notification/index.tsx`)
- Paginated `GET /api/notifications` (title, messageBody, isRead), grouped by date sections, pull-to-refresh. **No mark-as-read call and no tap-through navigation** — purely a feed.

### Explicitly absent (searched, not found)
No IDT report module, no referrals, no vitals, no billing/insurance. **Care-conference summaries are now present** (resident-facing, §3.2) and a **resident discharge flow** exists (§2). "Case manager" exists only as a chat role. If the PRD expects IDT/referrals/billing for skilled nursing, they are net-new.

---

## 4. Push notifications & deep links

- **FCM token**: requested on Splash (`registerDeviceForRemoteMessages` + `requestPermission` + `getToken`), cached in AsyncStorage `FCM_TOKEN` (`SplashScreen/index.tsx:23-40`), then sent on **every API request** as both `pushToken` and `x-fcm-token` headers (`services/Api/index.tsx:79-84`) — backend registers/refreshes it implicitly; there is no explicit register-device endpoint call.
- **Foreground FCM** (`services/Notifications/foregroundNotifications.ts`): generic handler displays any incoming message via notifee (Android channel `default_notification_channel_id`, data-payload `title|body|message` fallback). **No type switch, no navigation** on these.
- **Chat local notifications** (`services/Notifications/chatNotification.ts`): generated by the socket lifecycle hook (not FCM) for messages outside the active conversation; channel `chat_notifications`; tap handler (`notifee.onForegroundEvent` PRESS) deep-navigates `Root → App → ConversationScreen{conversationId, contactName, contactAvatar}`. **Caveat:** `setupChatNotificationPressHandler` is a foreground-event handler only; no background/quit-state `onBackgroundEvent` or `getInitialNotification` routing exists, and no URL-scheme/universal-link config was found — cold-start deep linking is not implemented.

---

## 5. Diff vs `senior_living_reactnative` (senior-living resident app)

Both apps share the same backend (`*.sal.shashitech.com`), the same architecture (RootStack→Auth/App, Redux Toolkit slices user/dashboard, Axios singleton w/ facility header, Cognito), and a large copy-pasted component library. This is a **fork, not a shared library** — every "shared" file has drifted (see counts below).

### Modules unique to THIS app (skilled nursing)
| Module | Files |
|---|---|
| Care-team chat (REST + Socket.io + chat notifications) | `src/services/Chat/*`, `src/services/ChatSocket/*`, `src/hooks/useChatSocketLifecycle.ts`, `…/MessageScreen/*`, `services/Notifications/chatNotification.ts` |
| Rehab (schedule, history, report, message team) | `…/HealthScreen/Rehab/**`, rehab endpoints in `src/services/health/index.tsx` |
| Test Results / lab-report PDFs | `…/HealthScreen/TestResult/**` |
| Brain Games referral catalog | `…/HealthScreen/BrainGames/index.tsx` |
| In-app Notifications center service | `src/services/Home/notification/*`, `Notification/NotificationsSkeleton.tsx` |
| Unified schedule + Activities RSVP | `src/services/services/schedule/*`, `src/screens/App/Activities/*`, `src/services/Home/UpcomingAppointments/*` |
| Profile suite: EditProfile, AddEditMember, ChangeUserPassword, ThemeSelection, UploadPictures (TV), QR TV pairing | `…/ProfileScreen/**`, `src/services/Profile/tvPairing/*`, `src/services/Profile/uploadPictures/*`, `src/services/Profile/staff/*` |
| Phone-based auth & AccessDenied | `src/utils/phoneAuth.ts`, `Auth/AccessDeniedScreen` |
| Global alert/question-alert Redux modals | `src/store/features/globalAlert/*`, `globalQuestionAlert/*`, `components/GlobalAlert/*` |
| Massage-therapy + private-training *service layers* split out, session/slot UI kit | `src/services/services/massageTherapy/*`, `PrivateTraining/*`, `components/App/Services/{SessionList,SlotOptions,ServiceTypeList,ServiceSummaryBottomSheet}` |

### Modules in the sibling app, absent here
- **MFA TOTP enrollment**: `StartMFASetupScreen`, `MFASetupScreen` (SN verifies SMS/TOTP but never enrolls).
- **NetworkWrapper** offline-state component.
- Health-hub **PT/Cognitive/OutsideAgency request flows as live UX** (`PhysicalTherapy/PhysicalTherapyForm|EvaluationScreen|CognitiveEvaluation`, `OutsideAgencyRequest`) — in SN these were rebuilt as the (currently orphaned) `/api/care` screens.
- Static data files `DiningData.ts`, `SalonData.ts` (SN is fully API-driven).
- Legacy top-level `src/screens/ProfileScreen/*` tree (SN consolidated under `App/ProfileScreen`).
- A **Profile bottom tab** (SN dropped it for the Message tab).

### Copy-pasted/shared-but-diverged (refactor candidates)
File-identical paths with heavy drift (diff line counts, SN vs SL):
`HomeScreen/index.tsx` 2,006 (1,177/829) • `MyScheduleScreen/index.tsx` 1,862 (1,461/401) • `TransportationScreen` 1,733 (929/804) • `DiningScreen` 1,006 (633/373) • `services/Home/index.tsx` 813 • `Medication` 761 • `AdvancedCareDirective` 620 • `cognito.service.ts` 616.
Effectively shared-with-light-drift: theme, scale, Skeleton, CustomImage/CustomBackground, AppButton/AppHeader, GlobalLoader, Localstorage, authStorage, ServiceCard, calendars/date pickers, salon + housekeeping services, store hooks. Chat types are explicitly annotated as **ported from `senior_living_staffapp`** (`src/services/Chat/type.ts:1-2`) — a third copy of the chat domain now exists across staff + SN resident apps. Any shared-component extraction should treat the SN app as the most current implementation (newer RN 0.84, larger feature surface).

---

## 6. Product-split signals & conditionals

- **No runtime facility-type/feature-flag gating found.** No `careType`-based branching, no remote-config flags; `grep` for skilled/facility-type conditionals only hits data fields. The skilled-nursing vs senior-living split is achieved by shipping different app binaries against the same backend tenant model (`x-facility-id`).
- Residual senior-living identity: bundle name `com.shashigroup.sl.resident` (note `sl`), `CareType = 'assisted_living' | string` (`src/services/Home/type.ts:7`), `MealRequest.careType: 'assisted_living' | 'memory_care'` (`type.ts:387`) — vocabulary not yet updated for skilled nursing.
- Feature *merchandising* is hard-coded per app: Health hub array (`HealthScreen/index.tsx:60-99`), Services array with commented-out massage/PT (`ServicesScreen/index.tsx:45-85`), Home quick actions (`HomeScreen/index.tsx:727-763`). These arrays are the de-facto product configuration surface.
- Cognito pools per environment come from `react-native-config` env vars (`cognito.config.ts`) — likely separate user pool from the SL app (phone-alias sign-in), unverifiable from code.

---

## 7. Observations (TODOs, dead code, drift, risks)

1. **Orphaned clinical flows**: all 12 PhysicalTherapy/CognitiveEvaluation/OutsideAgency screens are registered in `appstack/index.tsx` but unreachable (no `navigate()` call targets their list screens). Decide: delete or re-merchandise into Health hub.
2. **Dead screens**: `ResidentDirectoryScreen` registered + unit-tested but no entry point; `NotificationsSettingScreen` registered, Profile row commented out (`ProfileScreen/index.tsx:297-304`); Profile bottom tab commented out; Home message FAB commented out.
3. **Hidden-but-live booking**: Massage Therapy and Private Training are commented out of the Services hub yet their *reschedule* paths remain reachable from UpcomingAppointments/MySchedule — inconsistent UX if a legacy appointment exists.
4. **Hardcoded Google Places API key** in source: `src/utils/local.constants.ts:6` (`GOOGLE_PLACES_API_KEY = 'AIza…'`). Also developer LAN IPs/names (Anish, Ronak, aakash, Durgesh) in the LOCAL env block.
5. `currentEnv = PRODUCTION` is the committed default (`local.constants.ts:18`) — dev builds hit prod unless the AsyncStorage override is set (sibling defaults to STAGING).
6. **Sibling app's prod URLs look wrong**: `senior_living_reactnative/src/utils/local.constants.ts` PRODUCTION/PRE_PRODUCTION point at `api.hospitality.andmv.com` / `reservationapp.andmv.com` (Shashi *Hotels* backends) — copy-paste leftover; SN fixed this to `*.sal.shashitech.com`.
7. **Typos/debris**: tab key `MessegeTab`; `console.log('Chirag App' + error)` in QRScanner (`QRScannerScreen.tsx:177`); fallback alert text `'Appdemo'`; verbose auth `console.log`s including usernames (`cognito.service.ts`), full request/response logging gated by `__DEV__` with header masking (good) but body unmasked.
8. **"Remember Me" stores the raw password** on-device (`utils/authStorage.ts` via SignIn) — flag for security review.
9. Notifications center has **no mark-as-read or deep-link handling**; `isRead` is fetched and rendered but never mutated. Generic FCM taps go nowhere; only chat local notifications navigate, and only in foreground.
10. Zero `TODO/FIXME` comments repo-wide — drift is expressed through commented-out blocks instead (QA-review style comments like "As per QA review, removed elevation…" appear in styles).
11. Test hygiene is better than the sibling, but corrected to **56 test files** on staging (was claimed 58); all new staging feature areas (Care Conference, SessionGuard/inactivity, force-update, acknowledgment, discharge, WebView, ModifyFontSize/font-scale, custom-OTP auth.service) **ship without tests** — several are security-relevant (NEW MEDIUM).
12. Copy-paste lineage is three-way: chat ported from the staff app, most UX forked from the senior-living resident app — a shared RN package would collapse ~70% of this repo.

**Staging additions / new flags**
13. **NEW (HIGH): `IDLE_TIMEOUT = 7 DAYS`** (`local.constants.ts:11`) neutralises the new HIPAA inactivity auto-logoff; the realistic 15s/60s test values are commented out — the SessionGuard control is effectively inert.
14. **NEW (MEDIUM): Keychain bundle-prefix mismatch** — the auth service uses `com.shashigroup.SAL.resident.auth` while the new session service uses `com.shashigroup.SL.resident.session` (`sl` matches the actual `BUNDLE_ID`).
15. **App-wide font scaling (NEW)**: a module-level `appFontScale` clamped 0.8–1.5 (`scale.ts:27-41`) drives a new `getFontSize()` helper that replaced `moderateScale()` at most typography call sites; new `ModifyFontSizeScreen`; persisted under `APP_FONT_SCALE_V1`.
16. **New deps**: `react-native-webview` 13.16.1 (T&C/Privacy/Acknowledgment PDF), `react-native-device-info` 15.0.2 (force-update build check), `@eabdullazyanov/react-native-sms-user-consent` 1.3.0 (Android OTP autofill). Two SMS-consent native modules now coexist (legacy `react-native-otp-verify` + the new one) — NEW LOW.
17. **Version bump**: Android `versionCode 6 → 11`, `versionName 1.3 → 1.3.4`.
18. **Still dead (grep-verified)**: `react-native-qrcode-svg` and `@react-native-firebase/auth` confirmed unused on staging; CognitoStorage.getItem always-null, AsyncStorage token storage, hardcoded Google Places key, trustAllCerts PDF, no background FCM handler, no onTokenRefresh, hardcoded Discord webhook all persist.

**This pass's additions (`4734daf..f22dc60`)**
19. **NEW (HIGH-adjacent, product/security): Biometric App Lock is device-scoped, not account-scoped, and the same gap already flagged on the staff app.** §2's Biometric App Lock write-up above; cross-referenced in `../modules/platform-foundation.md` PLAT-FR-77 (which itself cross-references PLAT-FR-75) and business rule 32/§9 item 40 of that doc. Not a code defect — the `enterApp.ts` comment states the device-level design is intentional and modeled directly on staffapp — but an unconfirmed product/security posture if this app is ever deployed on facility-owned/shared resident tablets rather than personal devices, and it additionally survives an ordinary voluntary sign-out (only the lock-screen's "Back to Login"/`invalidated`-unlock path clears the Keychain enrollment, not Profile → Sign Out).
20. **NEW (LOW): a third Keychain-service naming scheme.** Item 14 above already flagged `SAL` vs `SL` case drift between the auth and session-timestamp Keychain services; the new App Lock item adds a third variant, all-lowercase `com.shashigroup.sal.resident.app_lock` (`local.constants.ts` `APP_LOCK_KEYCHAIN_SERVICE`) — matching neither the `BUNDLE_ID`-aligned `sl` casing nor either prior scheme. Functionally harmless (each service string only needs to be internally consistent with itself), but a third independent naming convention for what is conceptually the same kind of on-device secret.
21. **NEW (LOW): `AppLockGate` — this repo's largest untested new component.** `biometrics.service.test.ts` (353 lines) and `appLock.slice.test.ts` cover the Keychain/Redux layers well, but `components/App/AppLock/index.tsx` itself — the ~300-line component owning the background/foreground timers, `AppState` listener, and `PanResponder` idle-reset — ships with no test file. Same pattern already noted in item 11 for this pass's other new surfaces (Care Conference, SessionGuard, force-update, acknowledgment, discharge, WebView, font-scale, custom-OTP auth service): security/access-control-relevant code is consistently the least-tested addition in this repo.
22. **NEW (LOW): "Shashi Care" is a previously-unseen product name.** The new `AppStrings.AppLock.*` and `AppStrings.OtaUpdate.*` copy blocks both refer to the product as **"Shashi Care"** in user-facing strings (e.g. `enableFaceIdDescription`, `biometricsUnavailableBodyFaceId`) — a third branding string alongside the header's `AppStrings.appName = 'Skilled Nursing'` and footer `'Shashi.ai'` (see file header, line 4). Not confirmed whether this is an intentional rebrand in progress or a copy-paste from another app's string file; worth a product check before the next release.
23. **NEW (LOW, build/CI): release-signing can silently fall back to the debug keystore.** `android/app/build.gradle`'s `release` build type now resolves `hasReleaseKeystore ? signingConfigs.release : signingConfigs.debug` — if neither `KEYSTORE_FILE_TMP` nor a committed `keystore.properties` is present at build time, a "release" build is silently signed with the debug key rather than failing the build. Paired with the version bump this pass (`versionCode 20 → 29`, `versionName 1.3.13 → 1.3.15`) and the new `scripts/configure-ota-channel.js` OTA-channel patcher (§8) — both native-build-adjacent, both outside this repo's own test suite. Native/CI build-pipeline risk detail beyond this observation is architecture-doc territory, not re-derived here.

---

## 8. OTA / Expo Update Pipeline (NEW)

The app was migrated onto the Expo/EAS Update runtime this pass while keeping its existing native Gradle/Fastlane
build path (i.e. **not** a full Expo-managed migration — native projects remain committed and hand-built; only
the JS bundle is Expo-Updates-aware). `app.json`: `expo.updates = { enabled: true, checkAutomatically: "NEVER",
fallbackToCacheTimeout: 0, url: "https://u.expo.dev/ad0c5b46-08a2-41d4-a2ec-50902f7e29ff" }`, `runtimeVersion:
"1.0.0"`. `checkAutomatically: "NEVER"` is deliberate — it disables `expo-updates`' own automatic check so that
`useOtaUpdate()` (below) is the **only** path that ever downloads or applies an update. `eas.json` defines three
build profiles/channels: `staging` (dev client, internal APK), `preprod` (internal APK), `production`
(auto-incrementing, submits to Play internal track via `google-services-key.json`).

**Native-channel wiring is manual, not EAS-managed.** Because native builds go through Gradle/Fastlane directly
rather than `eas build`, nothing auto-injects the channel into the compiled binary the way an EAS-built APK/IPA
would get it. `scripts/configure-ota-channel.js` (new) must be run before each native build —
`OTA_CHANNEL=staging|preprod|production node scripts/configure-ota-channel.js` — and patches two otherwise-static,
committed native files by regex: `android/app/src/main/AndroidManifest.xml`'s
`expo.modules.updates.UPDATES_CONFIGURATION_REQUEST_HEADERS_KEY` meta-data value, and
`ios/SkilledNursing/Supporting/Expo.plist`'s `EXUpdatesChannel` + `expo-channel-name` keys. The script fails loudly
if `OTA_CHANNEL` is unset/invalid or if the expected patterns aren't found (defensive against the native files
changing shape), but there is **no CI step wiring it in** in this repo's own files — whoever builds (locally or a
pipeline) must remember to run it first, or the binary silently keeps polling whatever channel it was last built
with.

**Client-side check/apply flow (`src/hooks/useOtaUpdate.ts`):** mounted once in `RootNavigator`
(`src/navigation/rootstack/index.tsx`), fires on mount and on every foreground `AppState` transition to `active`
(re-entrant-guarded via an in-flight ref). Sequence: (1) `Updates.isEnabled` short-circuit (no-op in Expo
Go/dev); (2) `Updates.checkForUpdateAsync()` — a plain `expo-updates` manifest check against the channel baked
into the binary; (3) **only if that's available**, a server-side eligibility gate —
`fetchOtaEligibility({ platform })` → `GET /api/ota/check` (`src/services/App/index.ts`) — `cName`/facility are
inferred backend-side from the JWT/`x-facility-id` header, not sent by the client (`OtaEligibilityParams` is
deliberately just `{ platform }`); a `false` `isNeedOTAUdate` [sic] or a request failure both fail closed (skip
this cycle, retry next foreground); (4) a delivery-audit call, `logOtaDelivery()` → `POST /api/ota/log`
(`otaVersion`/`previousOtaVersion` = backend `OtaVersions` ObjectIds from the eligibility response) — also fails
closed; (5) `Updates.fetchUpdateAsync()` downloads the bundle *before* asking the resident anything, so tapping
Reload is instant; (6) an explicit Reload/Cancel question-alert (`showAppQuestionAlert`,
`AppStrings.OtaUpdate.title/subtitle`) — **the update is never applied without this confirmation** — plus a
persistent reminder banner (`showOtaUpdateBanner`) that stays up across a Cancel (only a successful reload or a
fresh check cycle clears it) and re-opens the same dialog on tap rather than reloading directly
(`otaBanner.helper.ts`). Tapping Reload persists `PENDING_OTA_RELOAD_UPDATE_ID` (the fetched update's manifest id)
to `AsyncStorage` before calling `Updates.reloadAsync()` — this is the sole signal `useOtaUpdateToast()` uses on
the next boot to detect "this cold start followed our own gated OTA reload" (compares `Updates.updateId` against
the pending id, clears the pending key immediately on read to bound a crash mid-toast to at most one missed
toast, and separately tracks `LAST_OTA_TOAST_SHOWN_UPDATE_ID` so a repeat cold start on the same update never
re-shows the toast) and fires a one-time "You're on the latest version" alert
(`AppStrings.OtaUpdate.updatedTitle/updatedToast`).

**UI surface:** `GlobalOtaBanner` (`src/components/GlobalAlert/GlobalOtaBanner.tsx`) is not mounted globally —
it's rendered individually inside `SignInScreen` (with an explicit `statusBarColor`, since that screen has no
shared header to inherit color from) and inside `HomeScreen` (`insetTop={false}`, since `AppHeader` already
clears the status bar). It is **not** rendered on any other of the ~68 registered app-stack screens — a resident
mid-flow on, say, a booking screen gets no visual reminder until they return to Home.

**Backend surface consumed (new, not independently verified against the backend repo this pass — flagged for
cross-check, consistent with this doc's "code only" scope):** `GET /api/ota/check` (`OtaEligibilityResponse`:
`isNeedOTAUdate`, `currentOTAVersion`/`newOTAVersion` — each an `OtaVersions` document with `facilityId`,
`otaVersion`, `runtimeVersion`, `platform`, `publishedAt`, `status`) and `POST /api/ota/log`
(`{ device, otaVersion, previousOtaVersion, deliveredAt }` → `{ success }`) — both under `src/services/App/{index,type}.ts`.
This implies a facility-scoped, backend-managed OTA-version registry with per-platform targeting exists
server-side; no admin-web surface for authoring/targeting these versions was found by grepping this repo (as
expected — that would live in `senior_living_admin`, out of this doc's scope).

**Product framing:** distinct from `platform-foundation.md` PLAT-FR-70 (the native force-update floor, which
blocks on an installed **binary** version) — this is a second, independent update channel that can patch JS-only
behavior into already-installed apps **without any app-store review**, gated per-facility/platform by the new
backend eligibility check rather than being a blanket rollout. No rollback runbook, no admin UI, and no
automated-test coverage for any of `useOtaUpdate`/`useOtaUpdateToast`/`otaBanner.*`/`configure-ota-channel.js`
were found in this repo. New dependencies: `expo` ^56.0.11, `expo-updates` ~56.0.19, `expo-asset`/`expo-modules-core`
^56.0.17, `babel-preset-expo` ^56.0.15 (`package.json`). (`app.json`, `eas.json`, `scripts/configure-ota-channel.js`,
`src/hooks/useOtaUpdate.ts`, `src/hooks/useOtaUpdateToast.ts`, `src/store/features/otaBanner/*`,
`src/components/GlobalAlert/GlobalOtaBanner.tsx`, `src/services/App/index.ts`, `src/services/App/type.ts`)
