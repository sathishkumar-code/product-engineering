# Architecture: senior_living_reactnative

> Doc-status: Re-verified v3.0 against staging HEAD `97f75c4`, 2026-06-21. Source of truth is now the `staging` branch (pending promotion to production; previously verified against production HEAD `4faca64`, 2026-06-17).
> Delta-patched 2026-07-12 against `origin/master` HEAD `3af3c3e` (2026-04-25) — already folded into staging via `97f75c4` (same-day merge). Corrected the Outside Agency cancel endpoint path and documented Physical Therapy / Cognitive Evaluation cancel + reschedule, plus the shared-cancel naming issue (TD-17).
> Related docs: [../architecture-senior-living-product.md](../architecture-senior-living-product.md) | [./adr/](./adr/)

---

## 1. Purpose

`senior_living_reactnative` is the resident-facing mobile application for the Shashi.AI Senior Living platform. It runs on iOS and Android and is the primary digital touchpoint for residents in a senior living facility. The app provides:

- Authenticated resident session management (AWS Cognito, TOTP/MFA) — also supports family-member logins (`profile.isFamilyMember`)
- Home dashboard: greeting, concierge call shortcut, family contacts, announcements preview, upcoming appointments — facility-aware (loads facility config before profile)
- Dining: date-scoped menu browsing and family meal request workflow (with request history)
- Transportation: ride request with Google Places address picker, complimentary-ride distance check (facility coordinates from Redux), past-ride history
- Housekeeping / Other Services: room cleaning, extra laundry, miscellaneous and maintenance request creation; service history list
- Salon: service browsing, slot selection, and **live appointment booking / reschedule / cancel** (now wired to the backend — DG-03 resolved on staging)
- Massage Therapy and Private Training: service browsing, slot selection, live booking / reschedule / cancel (new API-backed flows on staging)
- Profile management: profile view (Redux-backed), edit profile, manage account, care team, notification preferences, theme/display, settings, change password, family-member add/edit/remove, profile-picture upload, **TV-app QR pairing** and resident-picture upload to TV
- Health section: Medication, Advanced Care Directive, and three new API-backed care flows — **Physical Therapy**, **Cognitive Evaluation**, **Outside Agency** — each with list / details / request / slot-selection screens (all routed through the unified `/api/care` endpoints), with **cancel and reschedule now uniformly supported across all three** (`PUT /api/care/:id` reschedule, `DELETE /api/care/cancel/:id` cancel — see §3.3, §9 TD-17)
- Activities: API-backed activity schedule with join/cancel (new on staging)
- MySchedule: API-backed unified schedule (now live — DG-05 resolved)
- Announcements and Notifications feeds (both paginated, API-backed; Notifications now wired to `/api/notifications`)
- Resident Directory (API-backed via `/api/residents` with favourite toggle)
- Foreground push notifications rendered locally via `@notifee/react-native` (new on staging — partial DG-04 closure)

**Backend:** Node.js/Express TypeScript service, port 7000. Base URLs configured per environment in `src/utils/local.constants.ts`. See [../architecture-senior_living_backend.md](../architecture-senior_living_backend.md) for backend architecture.

**Multi-tenancy note — RESOLVED on staging:** The backend requires an `x-facility-id` header on every request for tenant isolation (`senior-living/CLAUDE.md` line 21). On staging the Axios request interceptor now injects `x-facility-id` from the `FACILITY_ID` AsyncStorage key (`src/services/Api/index.tsx:87-90`), which is populated from the facility-config API on the Home screen (`HomeScreen/index.tsx:765-766`). See Design Gap DG-01 (resolved).

---

## 2. Tech Stack

| Concern | Library / Version | Source |
|---|---|---|
| Framework | React Native 0.82.1, React 19.1.1 | `package.json` |
| Language | TypeScript 5.8 | `package.json` |
| State management | Redux Toolkit 2.x, react-redux 9.x | `package.json` |
| Navigation | `@react-navigation/native-stack` (root + auth stacks) + `@react-navigation/stack` legacy (app screens) + `@react-navigation/bottom-tabs` (5 tabs) | `package.json`, `src/navigation/appstack/index.tsx` |
| Auth | `amazon-cognito-identity-js` 6.3.16 | `package.json` |
| HTTP client | Axios 1.13.2 | `package.json` |
| Firebase | `@react-native-firebase/app` + `/analytics` + `/crashlytics` + `/messaging` + `/auth` all v23.8.3 | `package.json` |
| Local notifications | `@notifee/react-native` 9.1.8 — renders foreground FCM messages as local notifications | `package.json`, `src/services/Notifications/foregroundNotifications.ts` |
| Local storage | `@react-native-async-storage/async-storage` 2.2.0 | `package.json` |
| Secure storage | `react-native-keychain` 10.0.0 | `package.json` |
| Google Places | `react-native-google-places-autocomplete` | `package.json` |
| Google Distance Matrix | Native `fetch` REST call | `src/utils/distanceMatrix.ts` |
| Date utilities (moment) | `moment` — TransportationScreen, SalonBookAppointment, CognitiveEvaluation | `package.json` |
| Date utilities (date-fns) | `date-fns` — TransportationListScreen | `package.json` |
| Date utilities (dayjs) | `dayjs` — `OtherServiceRequest/index.tsx:23`, `DiningScreen/RequestedMealList/index.tsx:31` — still **NOT in package.json** at staging HEAD | file-level imports only |
| Responsive scaling | Custom `scale.ts` (baseline 375x812) | `src/utils/scale.ts` |
| Theme | Custom `ThemeProvider` / `useThemeProvider` context, light/dark | `src/theme/index.tsx` |
| Gesture handling | `react-native-reanimated`, `react-native-gesture-handler` | `package.json` |
| Calendar | `react-native-calendars` | `package.json` |
| Image picker | `react-native-image-picker` | `package.json` |
| Date/time picker | `@react-native-community/datetimepicker` | `package.json` |
| Network state | `@react-native-community/netinfo` — installed; the `NetworkWrapper` component was **deleted on staging** (no current consumer) | `package.json` |
| Dropdown | `react-native-element-dropdown` | `package.json` |
| Fast image | `@d11/react-native-fast-image` (`CustomImage` component) — **not** the original `react-native-fast-image` DylanVann package | `src/components/CustomImage/index.tsx:6` |
| JWT decode | `jwt-decode` 4.0.0 — decodes access token to check expiry | `src/services/Auth/token.manager.ts:2,20` |
| OTP/code input | `react-native-confirmation-code-field` 8.0.1 — MFA and reset-password OTP entry fields | `ResetPasswordScreen/index.tsx:25`, `MFASetupScreen/index.tsx:30`, `MFAVerifyScreen/index.tsx:23` |
| Permissions | `react-native-permissions` 5.4.4 — runtime camera/photo-library permissions for image picker | `src/components/App/Services/ImagePicker/AppImagePicker.tsx:23,30` |
| QR scanner | `react-native-camera-kit` 17.0.1 — camera view for the TV-app QR pairing flow | `src/screens/App/ProfileScreen/LoginTvApp/QRScannerScreen.tsx:35` |
| QR code | `react-native-qrcode-svg` 6.3.21 — in `package.json`; still no screen import found at staging HEAD | `package.json` |
| Firebase Auth | `@react-native-firebase/auth` 23.8.3 — in `package.json`; still no import found at staging HEAD (installed, unused) | `package.json:26` |
| Testing | Jest 29, `react-test-renderer` | `package.json` |
| Linting / quality | ESLint 8 flat config, SonarQube (`sonar-project.properties`) | `package.json` |
| CI/CD | GitLab CI + Fastlane (iOS: TestFlight; Android: Play internal track) | `senior-living/CLAUDE.md` |

---

## 3. Key Components

### 3.1 Navigation Structure

```
RootNavigator (NativeStack)
  SPLASH       -> SplashScreen
  AUTH         -> AuthStack (NativeStack)
                    SIGN_IN           -> SignInScreen
                    CHANGE_PASSWORD   -> ChangePasswordScreen
                    FORGOT_PASSWORD   -> ForgotPasswordScreen
                    RESET_PASSWORD    -> ResetPasswordScreen
                    MFA_SETUP         -> MFASetupScreen
                    START_MFA_SETUP   -> StartMFASetupScreen
                    MFA_VERIFY        -> MFAVerifyScreen
  APP          -> AppStack (LEGACY createStackNavigator)
                    ROOT_TABS         -> BottomTabNavigator (HomeTab / ServicesTab / HealthTab / MyScheduleTab / ProfileTab)
                                           HomeTab       -> HomeScreen
                                           ServicesTab   -> ServicesScreen
                                           HealthTab     -> HealthScreen
                                           MyScheduleTab -> MyScheduleScreen (resets to today on tab press via listeners)
                                           ProfileTab    -> ProfileScreen (src/screens/App/ProfileScreen/ProfileScreen/index.tsx — Redux-backed, LIVE)
                    [~50 other screens registered as push routes in AppStack]
GlobalAlertModal + GlobalQuestionAlertModal are mounted once at the RootNavigator level (rootstack/index.tsx:47-48), outside the NavigationContainer.
```

All non-tab screens are registered in a single flat `AppStack` (legacy `createStackNavigator`). Route constants are defined in the `AppScreens` object at `src/navigation/appstack/index.tsx:85-144` (the inventory roughly **doubled on staging** — see §3.2). The typed `AppStackParamList` (`appstack/index.tsx:146-277`) now carries real param shapes for the new care, salon, massage, and private-training flows (e.g. `isReschedule` / `appointmentData` on the request/slot screens, `selectedService` on the booking screens, and a richer `CONFIRMATION_SCREEN` param with `isSuccess` / `onDoneScreen` / `onDonePopCount`).

**The two formerly-conflicting Profile screens have been consolidated.** On staging the old `src/screens/ProfileScreen/**` tree (and the unreachable `src/screens/App/ProfileScreen/index.tsx` stub with hardcoded "Andrew Smith") were **deleted**. There is now one Profile screen tree under `src/screens/App/ProfileScreen/`. `AppScreens.PROFILE` still exists as a registered route constant but the Profile tab renders `App/ProfileScreen/ProfileScreen/index.tsx` directly via the tab navigator. DG-21 is resolved.

**Navigator type mismatch (unchanged):** Root/Auth use `@react-navigation/native-stack` (hardware-accelerated native transitions). App screens use the legacy JS-driven `@react-navigation/stack` (`createStackNavigator`, `appstack/index.tsx:279`; commented-out `createNativeStackNavigator` import at `appstack/index.tsx:83`). This inconsistency persists. See Technical Debt TD-03.

### 3.2 Screen Inventory

> Status legend: **LIVE** = wired to a real backend endpoint; **MOCK** = hardcoded data, no API; **STATIC** = navigation/launcher grid with no data fetch by design; **STUB** = present but action not implemented. Paths are relative to `src/screens/`.

| Screen | Path | Data source | Runtime status |
|---|---|---|---|
| SplashScreen | `SplashScreen/index.tsx` | AsyncStorage token check + `refreshSession()` | LIVE |
| SignInScreen | `Auth/SignInScreen/index.tsx` | Cognito `signIn()` | LIVE |
| ForgotPasswordScreen | `Auth/ForgotPasswordScreen/index.tsx` | Cognito `forgotPassword()` | LIVE |
| ResetPasswordScreen | `Auth/ResetPasswordScreen/index.tsx` | Cognito `confirmPassword()` | LIVE |
| ChangePasswordScreen | `Auth/ChangePasswordScreen/index.tsx` | Cognito `completeNewPassword()` | LIVE |
| MFASetupScreen / StartMFASetupScreen / MFAVerifyScreen | `Auth/MFASetupScreen`, `Auth/StartMFASetupScreen`, `Auth/MFAVerifyScreen` | Cognito TOTP setup / trigger / `verifySoftwareToken` | LIVE |
| HomeScreen | `App/HomeScreen/index.tsx` | Three sequential `useFocusEffect` fetches: `GET /api/config/residency-details` (facility, sets `x-facility-id`), then `GET /api/residents/profile`, then `GET /api/residents/dashboard` | LIVE |
| AnnouncementsScreen | `App/HomeScreen/Announcements/index.tsx` | `GET /api/announcements?page=&limit=` (paginated, skeleton loader) | LIVE |
| NotificationsScreen | `App/HomeScreen/Notification/index.tsx` | `GET /api/notifications?page=&limit=` (paginated, skeleton loader) | LIVE |
| ResidentDirectoryScreen | `App/HomeScreen/ResidentDirectoryScreen/index.tsx` | `GET /api/residents/getContact` via `fetchResidentContacts`; favourite toggle via `PUT` | LIVE |
| UpcomingAppointmentsScreen | `App/HomeScreen/UpcomingAppointment/index.tsx` | `GET /api/unified-schedule` via `fetchUpcomingUnifiedSchedule` (skeleton loader); care-card title now prefers `care.agencyName` over `care.name`/`care.careType` (delta 2026-07-12) | LIVE |
| MyScheduleScreen | `App/MyScheduleScreen/index.tsx` | `GET /api/unified-schedule?date=` via `fetchResidentSchedule`; cancel via `appointmentActions.ts` | LIVE |
| ActivitiesScreen | `App/Activities/index.tsx` | `GET /api/schedules` via `fetchResidentActivities`; join/cancel via `/api/schedules/:id/join`, `/cancel` | LIVE |
| ServicesScreen | `App/ServicesScreen/index.tsx` | 6-card launcher grid → Dining, OtherServices, Salon list, Transportation list, Massage Therapy, Private Training (`ServicesScreen/index.tsx:102-112`) | STATIC |
| DiningScreen | `App/ServicesScreen/DiningScreen/index.tsx` | `GET /api/menu?date=` | LIVE |
| RequestedMealListScreen | `App/ServicesScreen/DiningScreen/RequestedMealList/index.tsx` | `GET /api/family-meal-requests/resident` (+ `/history`) | LIVE (dayjs import — see TD-01) |
| RequestFamilyMealScreen | `App/ServicesScreen/DiningScreen/RequestFamilyMeal/index.tsx` | `GET /api/menu/price-and-time`, `POST /api/family-meal-requests` | LIVE |
| TransportationScreen | `App/ServicesScreen/Transportation/TransportationScreen/index.tsx` | `GET /api/transportation-rules`, `POST /api/resident-transportation/book-transportation`, Google Distance Matrix (facility coords from Redux `facilityData`) | LIVE |
| TransportationListScreen | `App/ServicesScreen/Transportation/TransportationListScreen/index.tsx` | `GET /api/resident-transportation/my-requests` (+ `/history`) | LIVE |
| OtherServicesListScreen | `App/ServicesScreen/OtherServices/OtherServicesList/index.tsx` | Static service-type list → `ServiceRequestListScreen` — see §4.6 | STATIC |
| ServiceRequestListScreen | `App/ServicesScreen/OtherServices/ServiceRequestListScreen/index.tsx` | `GET /api/housekeeping/resident?type=` (+ `/history`) | LIVE |
| OtherServiceRequestScreen | `App/ServicesScreen/OtherServices/OtherServiceRequest/index.tsx` | `POST /api/housekeeping` — reached from `ServiceRequestListScreen` "New Request" | LIVE (dayjs import — see TD-01) |
| ConfirmationScreen | `App/ServicesScreen/OtherServices/ConfirmationScreen/index.tsx` | Generic success/failure display (params-driven, now `isSuccess` + `onDone*` callbacks) | STATIC |
| SalonAppointmentScreen | `App/ServicesScreen/SalonAppointment/SalonAppointmentScreen/index.tsx` | `GET /api/salon/services` | LIVE |
| SalonAppointmentScreenlist | `App/ServicesScreen/SalonAppointment/SalonAppointmentScreenlist/index.tsx` | `GET /api/salon/my-appointments` (+ `/history`) | LIVE |
| SalonBookAppointmentScreen | `App/ServicesScreen/SalonAppointment/SalonBookAppointmentScreen/index.tsx` | `POST /api/salon/:serviceId/available-slots`, `POST /api/salon/book-appointment`, `PATCH /api/salon/reschedule-appointment` | LIVE (DG-03 resolved) |
| MassageTherapyScreen | `App/ServicesScreen/MassageTherapy/MassageTherapyScreen/index.tsx` | `GET /api/massage/my-appointments` (list) | LIVE |
| MassageTherapyTypesScreen | `App/ServicesScreen/MassageTherapy/MassageTherapyTypesScreen/index.tsx` | `GET /api/massage/services` | LIVE |
| MassageTherapyBookAppointmentScreen | `App/ServicesScreen/MassageTherapy/MassageTherapyBookAppointmentScreen/index.tsx` | `POST /api/massage/:serviceId/available-slots`, `POST /api/massage/book-appointment`, `PATCH .../reschedule-appointment` — now registered in AppStack | LIVE |
| PrivateTrainingSessionsScreen | `App/ServicesScreen/PrivateTrainingSessions/PrivateTrainingSessionsScreen/index.tsx` | `GET /api/private-training/my-appointments` (list) | LIVE |
| PrivateTrainingServiceListScreen | `App/ServicesScreen/PrivateTrainingSessions/PrivateTrainingServiceListScreen/index.tsx` | `GET /api/private-training/services` | LIVE |
| PrivateTrainingBookAppointmentScreen | `App/ServicesScreen/PrivateTrainingSessions/PrivateTrainingBookAppointmentScreen/index.tsx` | `POST /api/private-training/:serviceId/available-slots`, `POST .../book-appointment` — now registered in AppStack | LIVE |
| BookSessionConfirmScreen | `App/ServicesScreen/PrivateTrainingSessions/BookSessionConfirmScreen/index.tsx` | Params-driven success display | STATIC |
| HealthScreen | `App/HealthScreen/index.tsx` | 5-card launcher grid → Medication, Advanced Care Directive, Physical Therapy, Cognitive Evaluation, Outside Agency (all five cards now navigate; DG-11 resolved) | STATIC |
| MedicationScreen | `App/HealthScreen/Medication/index.tsx` | Hardcoded `MED_DATA`; Print/Share buttons no `onPress` | MOCK |
| AdvancedCareDirectiveScreen | `App/HealthScreen/AdvancedCareDirective/index.tsx` | Hardcoded `DOCUMENTS` | MOCK |
| AddScanDocumentScreen | `App/HealthScreen/AdvancedCareDirective/AddScanDocumentScreen/index.tsx` | Upload PDF / Scan Document — no upload API | STUB |
| PhysicalTherapyListScreen | `App/HealthScreen/PhysicalTherapy/PhysicalTherapyList/index.tsx` | `GET /api/care/upcoming` + `/api/care/history` (`type=physical_therapy`); cancel via `DELETE /api/care/cancel/:id` (`CancelOutsideAgencyService` — shared helper, see §3.3 / TD-17) | LIVE |
| RequestPhysicalTherapyEvaluation / RequestPhysicalTherapySlot | `App/HealthScreen/PhysicalTherapy/RequestPhysicalTherapyEvaluation`, `.../RequestPhysicalTherapySlot` | `GET /api/care/available-slots`, `POST /api/care`, `PUT /api/care/:id` (reschedule via `UpdatePhysicalTherapySlot`; screens pre-fill from `route.params.appointmentData` when `isReschedule`) | LIVE |
| PhysicalTherapyEvaluationDetailsScreen | `App/HealthScreen/PhysicalTherapy/PhysicalTherapyEvaluationDetails/index.tsx` | Params-driven detail display | STATIC |
| BookSessionConfirmDetailsScreen | `App/HealthScreen/PhysicalTherapy/BookSessionConfirmDetails/index.tsx` | Params-driven success display | STATIC |
| CognitiveEvaluationListScreen | `App/HealthScreen/CognitiveEvaluation/CognitiveEvaluationList/index.tsx` | `GET /api/care/upcoming` + `/api/care/history` (`type=cognitive_evaluation`); cancel via `DELETE /api/care/cancel/:id` (`CancelOutsideAgencyService` — shared helper, see §3.3 / TD-17) | LIVE |
| RequestCognitiveEvaluation / RequestCognitiveEvaluationSlot | `App/HealthScreen/CognitiveEvaluation/RequestCognitiveEvaluation`, `.../RequestCognitiveEvaluationSlot` | `GET /api/care/available-slots`, `POST /api/care`, `PUT /api/care/:id` (reschedule via `UpdateCognitiveEvaluationSlot`; screens pre-fill from `route.params.appointmentData` when `isReschedule`) | LIVE |
| CognitiveEvaluationDetailsScreen | `App/HealthScreen/CognitiveEvaluation/CognitiveEvaluationDetails/index.tsx` | Params-driven detail display | STATIC |
| OutsideAgencyServiceListScreen | `App/HealthScreen/OutsideAgency/OutsideAgencyServiceList/index.tsx` | `GET /api/care/upcoming` + `/api/care/history` (`type=outside_agency`); cancel via `DELETE /api/care/cancel/:id` (`CancelOutsideAgencyService`) | LIVE |
| RequestOutsideAgencyServiceScreen | `App/HealthScreen/OutsideAgency/RequestOutsideAgencyService/index.tsx` | `POST /api/care`, `PUT /api/care/:id` (reschedule via `UpdateOutsideAgencyService`; pre-fills from `route.params.appointmentData` when `isReschedule`). **Correction:** cancel is not initiated here — it lives on `OutsideAgencyServiceListScreen` (row above) via `DELETE /api/care/cancel/:id` | LIVE |
| OutsideAgencyServiceDetailsScreen | `App/HealthScreen/OutsideAgency/OutsideAgencyServiceDetails/index.tsx` | Params-driven detail display | STATIC |
| ProfileScreen (tab) | `App/ProfileScreen/ProfileScreen/index.tsx` | Redux `selectUserProfile`; branches on `profile.isFamilyMember`; logout via `logoutResident()` + `LogUserOut()` | LIVE |
| EditProfileScreen | `App/ProfileScreen/EditProfile/index.tsx` | `PUT /api/residents/...` via `updateResidentProfile` | LIVE |
| AddEditMemberScreen | `App/ProfileScreen/AddEditMemberScreen/index.tsx` | `POST /api/residents/add-family-member`, `PUT .../edit-family-member`, `POST .../remove-family-member` | LIVE |
| ManageAccountScreen | `App/ProfileScreen/ManageAccountScreen/index.tsx` | Account menu (Redux profile) | LIVE |
| CareTeamScreen | `App/ProfileScreen/CareTeamScreen/index.tsx` | `GET /api/staff` via `fetchStaff` | LIVE |
| ChangeUserPasswordScreen | `App/ProfileScreen/ChangeUserPassword/index.tsx` | Cognito `changePassword` | LIVE |
| NotificationsSettingScreen | `App/ProfileScreen/NotificationsSettingScreen/index.tsx` | Local-state toggles — no persistence API | MOCK |
| DisplayScreen / ThemeSelectionScreen | `App/ProfileScreen/DisplayScreen`, `.../ThemeSelectionScreen` | Theme selection via `ThemeProvider` | LIVE (theme), STUB (other rows) |
| SettingsScreen | `App/ProfileScreen/SettingsScreen/index.tsx` | Settings menu | STUB |
| QRScannerScreen (LoginTvApp) | `App/ProfileScreen/LoginTvApp/QRScannerScreen.tsx` | Camera scan → `POST /api/tv/pairing/authorize` | LIVE |
| UploadImagesScreen | `App/ProfileScreen/UploadPictures/UploadImagesScreen.tsx` | `GET/POST/DELETE /api/residents/pictures` (TV-displayed photos) | LIVE |

### 3.3 Service Layer

| File | Purpose | Key endpoints called |
|---|---|---|
| `src/services/Api/index.tsx` | Singleton Axios instance; interceptors: env-override, multipart detection, Bearer auth, FCM token (`x-fcm-token` + `pushToken`), **`x-facility-id` injection** (`:87-90`), global loader, single-flight 401 refresh, `__DEV__`-guarded logging with `maskHeaders()` Authorization redaction | — |
| `src/services/Api/api.helper.ts` | Typed wrappers `apiGet<T>` and `apiPost<T,P>` around Axios instance | — |
| `src/services/Auth/cognito.service.ts` | `signIn`, `refreshSession`, `logout`, `forgotPassword`, `completeNewPassword`, `verifyTOTP` | AWS Cognito SDK |
| `src/services/Auth/cognito.config.ts` | UserPool ID and ClientID — hardcoded constants (see Security S-03) | — |
| `src/services/Auth/cognito.storage.ts` | Custom `ICognitoStorage` adapter — **still broken async/sync mismatch** on staging (see Security S-04) | AsyncStorage |
| `src/services/Auth/token.manager.ts` | `saveTokens`, `getTokens`, `clearTokens`, `isTokenExpired` | AsyncStorage |
| `src/utils/auth/LogUserOut.ts` | **New** common logout flow — `logout()` (Cognito) + `clearForLogout()` (AsyncStorage) + `clearDashboard`/`clearUser` Redux dispatch | — |
| `src/services/Home/index.tsx` | Resident, facility, dashboard, announcement, menu, transportation (book + history), family-meal (list + history), residents-list API calls | `/api/residents/*`, `/api/config/residency-details`, `/api/announcements`, `/api/menu/*`, `/api/resident-transportation/*`, `/api/family-meal-requests/*` |
| `src/services/Home/notification/index.ts` | `fetchNotifications` | `/api/notifications` |
| `src/services/Home/resident/index.ts` | `fetchResidentContacts`, `toggleResidentFavorite` | `/api/residents/getContact`, `PUT` favourite |
| `src/services/Home/UpcomingAppointments/index.ts` | `fetchUpcomingUnifiedSchedule` | `/api/unified-schedule` |
| `src/services/health/index.tsx` | Unified care service — Physical Therapy / Cognitive Evaluation / Outside Agency upcoming, history, slots, book (`POST /api/care`). **Reschedule and cancel added 2026-07-12** (`master`, folded into staging): `UpdatePhysicalTherapySlot`, `UpdateCognitiveEvaluationSlot`, `UpdateOutsideAgencyService` — all `PUT /api/care/:id`; **`CancelOutsideAgencyService`** (`DELETE /api/care/cancel/:id`) is the single cancel function reused by all three care-type list screens despite its Outside-Agency-specific name — see TD-17 | `/api/care`, `/api/care/upcoming`, `/api/care/history`, `/api/care/available-slots`, `/api/care/:id`, `/api/care/cancel/:id` |
| `src/services/services/salon/index.tsx` | `fetchSalonServices`, `fetchSalonSlots`, `fetchSalonAppointments` (+ upcoming/history), `fetchAvailableSalonSlots`, **`bookSalonAppointment`**, **`rescheduleSalonAppointment`**, **`cancelSalonAppointment`** | `/api/salon/*` |
| `src/services/services/massageTherapy/index.tsx` | **New** — services, slots, book, my-appointments (+ history), reschedule, cancel | `/api/massage/*` |
| `src/services/services/PrivateTraining/index.tsx` | **New** — services, slots, book, my-appointments (+ history), cancel, reschedule | `/api/private-training/*` |
| `src/services/services/schedule/index.ts` | **New** — `fetchResidentSchedule`, `fetchResidentActivities`, `joinScheduleEvent`, `cancelScheduleEvent` | `/api/unified-schedule`, `/api/schedules`, `/api/schedules/:id/join`, `/cancel` |
| `src/services/services/housekeeping/index.ts` | `fetchHousekeepingRequests` (+ history), `createHousekeepingRequest` | `/api/housekeeping`, `/api/housekeeping/resident` (+ `/history`) |
| `src/services/Profile/staff/index.tsx` | **New** — `logoutResident`, `fetchStaff`, `uploadProfilePicture`, `updateResidentProfile`, family-member add/edit/remove | `/api/staff`, `/api/residents/logout`, `/api/residents/profile-picture`, `/api/residents/*-family-member` |
| `src/services/Profile/tvPairing/tvPairingApi.ts` | **New** — `authorizeTvPairing` (TV QR pairing) | `/api/tv/pairing/authorize` |
| `src/services/Profile/uploadPictures/tvImages.ts` | **New** — `fetchTvImages`, `uploadTvImage`, `deleteTvImage` | `/api/residents/pictures` |
| `src/services/Notifications/foregroundNotifications.ts` | **New** — `setupForegroundNotificationListener` (FCM `onMessage` → `@notifee` local notification) | — |

### 3.4 Redux Store

| Slice | State shape | Actions |
|---|---|---|
| `user` | `{ profile: ResidentProfile \| null, facilityData: FacilityData \| null }` | `setUserProfile`, `setFacilityData`, `updateUserProfile`, `clearUser` |
| `dashboard` | `{ data: DashboardResponseData \| null, loading: boolean }` | `setDashboardLoading`, `setDashboardData`, `clearDashboard` |
| `globalAlert` (**new**) | `{ visible, title, subtitle, buttonText }` | `showGlobalAlert`, `hideGlobalAlert` |
| `globalQuestionAlert` (**new**) | `{ visible, title, subtitle, leftButtonText, rightButtonText }` | `showGlobalQuestionAlert`, `hideGlobalQuestionAlert` |

The store (`store.ts`) now registers four reducers (`user`, `dashboard`, `globalAlert`, `globalQuestionAlert`). The `user` slice gained `facilityData` (and `setFacilityData` / `selectFacilityData`) so the transportation flow and `x-facility-id` header can derive from the facility-config response. The two global-alert slices back app-wide modal dialogs rendered once at the root (`GlobalAlertModal` / `GlobalQuestionAlertModal`, `rootstack/index.tsx:47-48`) and driven imperatively via `globalAlert.helper.ts` (`showAppAlert`) / `globalQuestionAlert.helper.ts`, with a Jest-env fallback to the native `Alert`.

`serializableCheck: false` is still set in `store.ts` (`store.ts:16`). See Technical Debt TD-06. All other screen-level data lives in local `useState` and is re-fetched on every screen mount or focus via `useFocusEffect`. There is no caching layer beyond the dashboard slice.

### 3.5 Utilities

| File | Purpose |
|---|---|
| `src/utils/local.constants.ts` | AsyncStorage key names, `ENVIRONMENT` enum, `envVariables` map per environment, `GOOGLE_PLACES_API_KEY` |
| `src/utils/scale.ts` | `scale()`, `verticalScale()`, `moderateScale()` — baseline 375x812 |
| `src/utils/authStorage.ts` | Keychain `saveCredentials` / `getCredentials` / `clearCredentials` (service name: `com.yourapp.auth`, `WHEN_UNLOCKED`) |
| `src/utils/distanceMatrix.ts` | `getRoadDistanceFromFacility()` — Google Distance Matrix REST call |
| `src/utils/date.ts` | `formatSectionTitle`, `formatTimeAgo` |
| `src/utils/Validation.ts` | `isValidEmail`, `validatePassword` |
| `src/utils/ValidationMessages.ts` | Validation error message string constants — imported by `TransportationScreen` and other screens | `TransportationScreen/index.tsx:30` |
| `src/utils/AppStrings.ts` | Centralized string constants |
| `src/utils/AppIcons.ts` | SVG icon exports |
| `src/utils/AppImages.ts` | Image asset exports |
| `src/utils/dateFormats.ts` | **New** — `DATE_FORMATS` constants and `formatDobLabel()` |
| `src/utils/CommonDialog.ts` | **New** — `showTwoButtonDialog()` native-Alert helper |
| `src/utils/Localstorage/` | AsyncStorage `getItem` / `setItem` / `removeItem` wrapper; **new `clearForLogout()`** clears tokens, FCM, facility id |

### 3.6 Shared Components

`AppHeader`, `AppButton`, `CustomBackground` (see note below), `CustomImage` (`@d11/react-native-fast-image` wrapper), `Skeleton`, `ZoomableImage`, `GlobalLoader` + `loader.helper.ts`, `AppCheckbox`, `ToggleRow`, `ToggleButton` (**new** — `src/components/App/Profile/ToggleButton.tsx`), `ProfileRow`, `ResidentRow`, `ContactActionSheet`, `PullCalendarComponent`, `ExpandableCalendar`, `BackButton`, `GradientText`, `AppDatePickerInput`, `AppImagePicker` + `ImagePickerActionSheet` (requires `react-native-permissions`), `HorizontalCalendar`, `HorizontalDateSelection` (**new** — `src/components/App/Services/PrivateTraining/`), `ServiceCard`, `ServiceItem` (**new** — `src/components/App/Services/OtherServices/`), `ServiceDetailModal` (**new** — `src/components/App/Services/SalonAppointment/`), `ServiceSummaryBottomSheet`, `ServiceTypeList`, `SessionList`, `SlotOptions` (**new** — under `src/components/App/Services/`), `AppointmentListElements` (`src/components/App/Health/` — gained `onCancel` / `onReschedule` action buttons + a `showActions` shimmer state on the 2026-07-12 delta; shared by all three Health list screens), `GlobalAlertModal` + `GlobalQuestionAlertModal` (**new** — `src/components/GlobalAlert/`), dining-specific skeleton loaders.

> **Removed on staging:** `NetworkWrapper` (`src/components/NetworkWrapper/NetworkWrapper.tsx`) was deleted. `@react-native-community/netinfo` remains in `package.json` but has no current consumer.

**`CustomBackground` background-preset system:** `src/components/CustomBackground/index.tsx` implements a pub/sub background-preset system (`setBackgroundPreset`, `subscribe`, `notifyAll`) persisted to AsyncStorage under `BG_PRESET_KEY = 'APP_BG_PRESET'` (`local.constants.ts:5`). The component self-loads the saved preset on mount and responds to live `setBackgroundPreset` calls via the subscriber. Theme/background selection now flows through the dedicated `DisplayScreen` / `ThemeSelectionScreen` and `ThemeProvider`; verify whether `setBackgroundPreset` is wired to a user action before relying on this path. See Design Gap DG-22.

---

## 4. Architecture Diagram and Key Flows

### 4.1 Overall Architecture

```mermaid
flowchart TD
    subgraph Device["Mobile Device (iOS / Android)"]
        App["App.tsx\nRedux Provider\nThemeProvider\nGestureHandlerRootView"]
        subgraph Nav["Navigation"]
            Root["RootNavigator\nnative-stack"]
            Auth["AuthStack\nnative-stack"]
            AppStack["AppStack\nlegacy createStackNavigator"]
            Tabs["BottomTabNavigator\n5 tabs"]
        end
        subgraph StateLayer["State"]
            Redux["Redux Store\nuser + dashboard slices"]
            LocalState["Screen useState\nall other data"]
            AsyncStore["AsyncStorage\ntokens, FCM, env override"]
            Keychain["Keychain\nRemember Me credentials"]
        end
        subgraph Services["Service Layer"]
            AxiosInst["Axios Instance\nBearer auth + x-fcm-token\n401 refresh + loader"]
            CognitoSvc["cognito.service.ts"]
            HomeSvc["Home Service"]
            SalonSvc["Salon Service"]
            HkSvc["Housekeeping Service"]
        end
        subgraph FirebaseSDK["Firebase SDK"]
            Analytics["Analytics"]
            Crashlytics["Crashlytics"]
            Messaging["Messaging\ntoken harvest +\nforeground onMessage\n→ notifee local notif"]
        end
    end

    subgraph External["External Services"]
        Cognito["AWS Cognito\nus-west-1"]
        Backend["Senior Living Backend\nExpress / port 7000"]
        GooglePlaces["Google Places API"]
        GoogleDist["Google Distance Matrix API"]
        FirebaseCloud["Firebase Cloud"]
    end

    App --> Nav
    App --> StateLayer
    Nav --> Services
    AxiosInst -->|"Bearer token, x-fcm-token,\nx-facility-id (from FACILITY_ID)"| Backend
    CognitoSvc --> Cognito
    HomeSvc --> AxiosInst
    SalonSvc --> AxiosInst
    HkSvc --> AxiosInst
    AsyncStore -.->|token read| AxiosInst
    AsyncStore -.->|token read/write| CognitoSvc
    FirebaseSDK --> FirebaseCloud
```

### 4.2 Authentication Flow

```mermaid
flowchart TD
    Splash["SplashScreen onMount"] --> CheckRefresh{"getItem(REFRESH_TOKEN)"}
    CheckRefresh -- null --> NavAuth["navigate AUTH"]
    CheckRefresh -- present --> CheckExpiry{"isTokenExpired(ACCESS_TOKEN)?"}
    CheckExpiry -- false --> NavApp["navigate APP"]
    CheckExpiry -- true --> Refresh["refreshSession()\nreads REFRESH_TOKEN + COGNITO_USERNAME\ndirectly from AsyncStorage\ncognito.service.ts:164-165"]
    Refresh -- success --> SaveNew["saveTokens(new tokens)"] --> NavApp
    Refresh -- fail --> NavAuth

    NavAuth --> SignIn["SignInScreen\ncognito.signIn(email, password)"]
    SignIn -- SUCCESS --> SaveTokens["saveTokens()"] --> NavApp
    SignIn -- NEW_PASSWORD_REQUIRED --> ChangePassword["ChangePasswordScreen"]
    SignIn -- MFA_SETUP --> MFASetup["MFASetupScreen\nTOTP QR code display"]
    SignIn -- MFA_VERIFY --> MFAVerify["MFAVerifyScreen\nverifySoftwareToken"]
    MFAVerify -- success --> SaveTokens

    RememberMe["Remember Me checkbox"] -.->|"Keychain saveCredentials"| SignIn
    SignIn -.->|"Keychain getCredentials on mount\nauto-fills email + password"| RememberMe
```

### 4.3 Token Refresh (401 Handler)

The Axios response interceptor at `src/services/Api/index.tsx:130-162` handles expired access tokens:

1. On the first 401, sets `isRefreshing = true`, stores the refresh promise in `refreshPromise`.
2. Concurrent 401s from other in-flight requests `await` the same `refreshPromise` (deduplication via shared reference, lines 19-21 — no thundering herd).
3. On success: stores new tokens via `saveTokens()`, updates `AxiosInstance.defaults.headers.common.Authorization`, retries the original request with the new token.
4. On failure: calls `logout()` which calls `clearTokens()`. The interceptor does not navigate — the screen layer is responsible for routing to AUTH after receiving the rejection.

**Critical note:** `refreshSession()` reads `REFRESH_TOKEN` and `COGNITO_USERNAME` directly from AsyncStorage (`cognito.service.ts:164-165`), bypassing the broken `CognitoStorage` adapter. This is the workaround that makes refresh function at runtime.

### 4.4 Home Dashboard Flow

On staging the Home screen runs **three** chained `useFocusEffect` hooks, now **sequentially gated**: facility config must load before profile, and facility must be ready before dashboard (`HomeScreen/index.tsx:753-797+`). The facility step is what populates the `FACILITY_ID` AsyncStorage key that the Axios interceptor reads for `x-facility-id`.

```mermaid
flowchart TD
    HomeMount["HomeScreen onFocus"]

    HomeMount --> HookFac["useFocusEffect #1 — Facility\n(index.tsx:754)\nguard: facilityData present?"]
    HookFac -- present --> SkipFac["skip"]
    HookFac -- absent --> FetchFac["FetchFacilityDataApi()\nGET /api/config/residency-details"]
    FetchFac --> DispatchFac["dispatch(setFacilityData)\nsetItem(FACILITY_ID, facilityId)"]

    DispatchFac --> HookProfile["useFocusEffect #2 — Profile\n(index.tsx:778)\nguard: facilityData && !profile"]
    SkipFac --> HookProfile
    HookProfile -- ok --> FetchProfile["fetchResidentProfile()\nGET /api/residents/profile"]
    FetchProfile --> DispatchProfile["dispatch(setUserProfile)"]

    DispatchFac --> HookDash["useFocusEffect #3 — Dashboard\n(index.tsx:798+)\nguard: facilityData ready"]
    SkipFac --> HookDash
    HookDash --> FetchDash["fetchResidentDashboard()\nGET /api/residents/dashboard"]
    FetchDash --> DispatchDash["dispatch(setDashboardData)"]

    DispatchProfile --> Render["Render HomeScreen"]
    DispatchDash --> Render

    Render --> Actions["Quick Actions"]
    Actions --> |"My Calendar"| TabNav["navigate MySchedule tab"]
    Actions --> |"Dining"| DiningNav["navigate DINING_SCREEN"]
    Actions --> |"Resident Directory"| DirNav["navigate RESIDENT_DIRECTORY (LIVE)"]
    Actions --> |"Call Concierge"| Tel["makeCall(dashboard.concierge)\nLinking.openURL('tel:...')"]

    Render --> FamilyStrip["Family member strip\n(profile.familyMembers)"]
    FamilyStrip --> |"tap avatar"| CallFamily["makeCall(member.phone)"]
```

**Note on the concierge tile:** The "Call Concierge" quick-action dials `dashboard?.concierge` (the facility concierge number from the dashboard API). Family-member calls are a separate flow (tapping an avatar in the family strip). The tile label may still mislead QA into expecting a family-call flow.

### 4.5 Transportation Request Flow

```mermaid
flowchart TD
    TScreen["TransportationScreen onMount"] --> FetchRules["fetchTransportationRules()\nGET /api/transportation-rules"]
    FetchRules --> Dropdown["Populate destination dropdown"]
    User --> Places["Google Places Autocomplete\ncountry: IN (India)"]
    Places --> DistCalc["getRoadDistanceFromFacility()\nGoogle Distance Matrix REST\nfacility = facilityData.lat/lng (Redux) ?? '40.712776,-74.005974'"]
    DistCalc --> CompCheck{"distanceMiles <= rule.maxDistanceMiles?"}
    CompCheck -- yes --> Comp["isComplimentary = true"]
    CompCheck -- no --> NotComp["isComplimentary = false"]
    User --> DTP["DateTimePicker\niOS: Modal spinner\nAndroid: inline"]
    DTP --> Submit["createResidentTransportation(payload)\nPOST /api/resident-transportation/book-transportation"]
    Submit --> NavList["navigate TRANSPORTATION_LIST_SCREEN"]
```

**Resolved on staging:** facility coordinates now come from the Redux `facilityData` slice (`facility?.lat`/`facility?.lng`, `TransportationScreen/index.tsx:443`), populated from the facility-config API. The hardcoded fallback changed from Vadodara, India to NYC (`40.712776, -74.005974`) and is only used when facility data is missing. See Technical Debt TD-05 (largely resolved).

### 4.6 Housekeeping Request Flow

The flow is two steps, not one — `OtherServicesListScreen` does **not** navigate directly to `OtherServiceRequestScreen`:

1. User taps a service type in `OtherServicesListScreen` (static type list). Navigates to `ServiceRequestListScreen` with `{ serviceId, requestType, title, emptyText }` params.
2. `ServiceRequestListScreen` fetches existing requests (`GET /api/housekeeping/resident?type=`, plus `/history`) and renders them. A "New Request" action navigates to `OtherServiceRequestScreen` (`OTHER_SERVICE_DETAIL_SCREEN`) with the same params.
3. In `OtherServiceRequestScreen`: for `RoomCleaning` / `ExtraLaundry` — shows date picker; future date required. Validation uses `dayjs` (not in `package.json` — see TD-01).
4. For `Miscellaneous` / `Maintenance`: shows text description field (required). `Maintenance` additionally shows image picker (`react-native-image-picker` + `react-native-permissions`).
5. On submit: `createHousekeepingRequest(payload)` → `POST /api/housekeeping` → navigate `CONFIRMATION_SCREEN`.

Payload shape: `{ residentId, residentName, unitNo, requestType, [selectedDate], [remarks], [maintanance] }`. Note the `maintanance` key is a typo in the active service function (`src/services/services/housekeeping/index.ts`).

### 4.7 Salon Flow (now fully wired)

```mermaid
flowchart TD
    SalonScreen["SalonAppointmentScreen onMount"] --> FetchSvc["fetchSalonServices()\nGET /api/salon/services"]
    FetchSvc --> Cards["Render service cards"]
    User --> SelectSvc["Select services"]
    SelectSvc --> NavBook["navigate SALON_DETAIL_SCREEN\n{ selectedServices | isReschedule + appointmentData }"]
    NavBook --> BookScreen["SalonBookAppointmentScreen"]
    BookScreen --> FetchSlots["fetchAvailableSalonSlots({serviceId, date})\nPOST /api/salon/:serviceId/available-slots\nreads res.data.slots ?? [] (no typo workaround)"]
    BookScreen --> Decide{"isReschedule?"}
    Decide -- no --> Book["bookSalonAppointment(payload)\nPOST /api/salon/book-appointment"]
    Decide -- yes --> Resched["rescheduleSalonAppointment(req)\nPATCH /api/salon/reschedule-appointment"]
    Book --> NavConfirm["navigate CONFIRMATION_SCREEN (isSuccess)"]
    Resched --> NavConfirm
```

The salon service layer now has `bookSalonAppointment`, `rescheduleSalonAppointment`, and `cancelSalonAppointment`. Bookings persist to the backend (DG-03 resolved). The `solts` backend-typo client workaround is gone (DG-20 resolved), and the redundant post-navigation `Alert.alert` was removed (DG-15 resolved). Cancellation is routed through `cancelSalonAppointment` (`POST /api/salon/cancel-appointment`), replacing the malformed `updateSalonAppointment` body (DG-08 resolved). The same book/reschedule/cancel + `:serviceId/available-slots` pattern applies to **Massage Therapy** (`/api/massage/*`) and **Private Training** (`/api/private-training/*`).

### 4.8 Announcements / Notifications Flow

`AnnouncementsScreen` (`GET /api/announcements?page=&limit=`) and `NotificationsScreen` (`GET /api/notifications?page=&limit=`) both render a `FlatList` with a dedicated skeleton loader and pull-to-refresh; subsequent pages load via `onEndReached` by incrementing the `page` state counter. Announcement items are date-grouped using `formatSectionTitle` from `src/utils/date.ts`.

### 4.9 Firebase / Push Notification Architecture

`@react-native-firebase/messaging` is installed. At app launch:
1. `messaging().getToken()` retrieves the FCM device token.
2. The token is stored to AsyncStorage under key `FCM_TOKEN`.
3. The Axios request interceptor injects it as `x-fcm-token` (and `pushToken`) on every API call.

**Now wired on staging:** the **foreground** handler is registered. `App.tsx` (`useEffect`, `App.tsx:21-24`) calls `setupForegroundNotificationListener()` (`src/services/Notifications/foregroundNotifications.ts`), which subscribes to `messaging().onMessage()` and renders the incoming payload as a local notification via `@notifee/react-native` (creating an Android channel `default_notification_channel_id` and using iOS foreground presentation options). The `NotificationsScreen` is now API-backed (`/api/notifications`).

**Still NOT wired:** `messaging().onNotificationOpenedApp()` (background tap handler) and `messaging().getInitialNotification()` (terminated-state tap handler) are still absent — deep-link/tap routing from a notification into a specific screen is not implemented. See Design Gap DG-04 (partially resolved).

---

## 5. Data and State

### 5.1 Data Ownership

| Data domain | Owner | Persistence | Re-fetch trigger |
|---|---|---|---|
| Auth tokens (ACCESS, REFRESH, ID, EXPIRY) | `token.manager.ts` | AsyncStorage | On 401 via interceptor |
| Cognito username | `token.manager.ts` | AsyncStorage key `COGNITO_USERNAME` | Written on signIn |
| FCM token | App root | AsyncStorage key `FCM_TOKEN` | Written on app launch |
| Facility id | HomeScreen facility effect | AsyncStorage key `FACILITY_ID` | Written after `GET /api/config/residency-details`; read per-request by interceptor for `x-facility-id` |
| Environment override | Axios request interceptor | AsyncStorage key `CurrentEnvironment` | Read per-request |
| Background preset | `CustomBackground` component | AsyncStorage key `BG_PRESET_KEY` = `'APP_BG_PRESET'` (`local.constants.ts:5`) | Written by `setBackgroundPreset()` — see DG-22 |
| Remember Me credentials | `authStorage.ts` | Keychain (`WHEN_UNLOCKED`) | Read on SignInScreen mount |
| Facility config | Redux `user.facilityData` | In-memory only | HomeScreen focus when `null` |
| Resident profile | Redux `user.profile` | In-memory only | HomeScreen focus when `null` (after facility ready) |
| Dashboard data | Redux `dashboard` slice | In-memory only | HomeScreen focus |
| Global alert / question-alert | Redux `globalAlert`, `globalQuestionAlert` slices | In-memory only | Imperative via `showAppAlert` helpers |
| All other screen data | Local `useState` | In-memory only | Every screen mount or focus |

There is no offline persistence for service data. Every list screen re-fetches from the backend on each focus event.

### 5.2 Cross-reference

Backend data schema and API contract: [../architecture-senior_living_backend.md](../architecture-senior_living_backend.md).

---

## 6. External Dependencies

| Service | Kind | Direction | Notes |
|---|---|---|---|
| AWS Cognito (`us-west-1_b4O3UxhMA`) | Managed auth | Outbound | UserPool ID and ClientID hardcoded in `src/services/Auth/cognito.config.ts`. Single pool for all environments — no per-environment switching. |
| Senior Living Backend API | REST HTTP | Outbound | Base URLs per environment in `src/utils/local.constants.ts`. Default env now = **PRE_PRODUCTION** (`preproduction-api.sal.shashitech.com`). STAGING and PRE_PRODUCTION both point at correct Senior Living backends; **PRODUCTION still points at the Shashi Hotels backend** (`api.hospitality.andmv.com`) — see Security S-05 / DG-02. |
| Firebase Analytics | SDK | Outbound | Event tracking. Requires `google-services.json` / `GoogleService-Info.plist` (not committed to repo). |
| Firebase Crashlytics | SDK | Outbound | Crash and ANR reporting. Same config files. |
| Firebase Cloud Messaging | SDK | Bidirectional | FCM token sent to backend via `x-fcm-token` / `pushToken` header. Foreground receive now wired via `@notifee` (see §4.9); background/terminated tap routing still not implemented. |
| @notifee/react-native | SDK | Local | Renders foreground FCM payloads as local notifications. Creates Android channel `default_notification_channel_id`. |
| Google Places Autocomplete | REST | Outbound | API key hardcoded in source (`src/utils/local.constants.ts:6`). Country filter: `IN` (India only). |
| Google Distance Matrix | REST | Outbound | Same hardcoded API key. Called from `src/utils/distanceMatrix.ts`. Used for complimentary-ride threshold check. |
| react-native-camera-kit | Native module | Local | Camera view for TV-app QR pairing (`LoginTvApp/QRScannerScreen.tsx`). |

> **Removed on staging:** the placeholder CDNs `pravatar.cc`, `images.unsplash.com`, and `picsum.photos` no longer appear anywhere in `src/` — the mock data arrays that referenced them were removed when the screens were wired to real APIs (TD-13 resolved).

---

## 7. Security and Multi-tenancy

### S-01 RESOLVED on staging: `x-facility-id` header now injected

The backend enforces multi-tenant isolation via `x-facility-id` on every request (`senior-living/CLAUDE.md:21`). On staging the Axios request interceptor reads the `FACILITY_ID` AsyncStorage key and sets `x-facility-id` when present (`src/services/Api/index.tsx:87-90`). The key is populated on the Home screen after the facility-config call (`HomeScreen/index.tsx:765-766`). **Residual risk:** API calls made before the Home screen's facility effect completes (or on screens reached without passing through Home) will send no `x-facility-id`; the header is best-effort, not enforced at the interceptor. Verify backend behaviour for the no-header case.

### S-02 CRITICAL: Google API key hardcoded in source

`GOOGLE_PLACES_API_KEY = 'AIzaSyCWsDqQfmtUlTOPFcWlXOkkAs7Q5lbntFc'` at `src/utils/local.constants.ts:6` (unchanged on staging). The key is live and used for both Places Autocomplete and Distance Matrix REST calls. If the key is not restricted by iOS bundle ID / Android package name in Google Cloud Console, it is extractable from the distributed binary.

### S-03 HIGH: Cognito pool credentials hardcoded in source

`UserPoolId: 'us-west-1_b4O3UxhMA'`, `ClientId: '2lc4usnui28vd9kn1gcaqq5tt3'` at `src/services/Auth/cognito.config.ts`. Commented-out prior IDs visible in lines 2-3. Single pool shared across all environments.

### S-04 HIGH: Broken CognitoStorage adapter

`src/services/Auth/cognito.storage.ts` implements `getItem()` as a synchronous function that internally calls `AsyncStorage.getItem()` (async). The `.then()` callback fires after the function has already returned `undefined`. The Cognito SDK cannot restore sessions through this adapter. The app functions only because `refreshSession()` reads `REFRESH_TOKEN` and `COGNITO_USERNAME` directly from AsyncStorage (`cognito.service.ts:164-165`), bypassing the adapter. Any Cognito SDK path that calls `CognitoStorage.getItem()` — such as `CognitoUserPool.getCurrentUser()` — will silently return `null`.

### S-05 MEDIUM (downgraded): only PRODUCTION env URL still points to Shashi Hotels backend

`src/utils/local.constants.ts` (staging):
- `STAGING.API_BASE_URL = 'https://staging-api.sal.shashitech.com'` — correct Senior Living.
- `PRE_PRODUCTION.API_BASE_URL = 'https://preproduction-api.sal.shashitech.com'` — **now correct** Senior Living (was Shashi Hotels at production HEAD).
- `PRODUCTION.API_BASE_URL = 'https://api.hospitality.andmv.com'` — **still the Shashi Hotels hospitality backend**, not Senior Living. PRODUCTION builds will call the wrong service.
- `LOCAL.API_BASE_URL` — developer LAN IP / port 7000.

The `API_BASE_URL_SECOND` field was **removed** from all environments (was dead config). PRE_PRODUCTION is now the default env (see S-06). Only PRODUCTION remains misconfigured. See DG-02.

### S-06 MEDIUM: Default environment is PRE_PRODUCTION with no build-time override

`src/utils/local.constants.ts:18`: `currentEnv: EnvironmentKey = ENVIRONMENT.PRE_PRODUCTION` (changed from STAGING). No build-time injection mechanism (no `react-native-config` or equivalent) is present. Switching environment still requires a source-code constant change (or the `CurrentEnvironment` AsyncStorage override read by the interceptor).

### S-07 MEDIUM: `console.log` in non-DEV-guarded paths (reduced but not eliminated)

The Axios interceptor's request/response logging is now wrapped in `if (__DEV__)` **with `Authorization` redaction via `maskHeaders()`** (`Api/index.tsx:203-219`). Several unguarded `console.log` calls remain in app code (verified at staging HEAD):
- `src/screens/SplashScreen/index.tsx:47,53,57,68,74,78,81,89` — logs auth-flow decisions including refresh-token presence and refresh errors (prefixed `[Splash]`) — **highest risk**: leaks auth state on every app launch
- `src/screens/App/ServicesScreen/Transportation/TransportationScreen/index.tsx:436,447` — logs selected address with lat/lng and full Distance Matrix result
- `src/services/Home/*` and other service modules — `console.warn(...)` on failure (acceptable but not env-guarded)

Note: many of the previously-cited unguarded `console.log` sites (the deleted `App/ProfileScreen/index.tsx:313`, the old `OutsideAgencyRequest`, `CognitiveEvaluation` mock, etc.) no longer exist because those screens were removed or rewritten.

### S-08 LOW: Keychain `WHEN_UNLOCKED` — credentials unavailable on locked device launch

`src/utils/authStorage.ts`: accessibility is `WHEN_UNLOCKED` (unchanged). If the app is launched while the device is locked, `getCredentials()` returns `null` and Remember Me auto-fill silently fails.

### S-09 RESOLVED: `API_BASE_URL_SECOND` removed

The dead `API_BASE_URL_SECOND` config field was removed from all environments on staging.

---

## 8. Design Gaps

Status as re-verified against staging HEAD `97f75c4`. Many gaps from the production baseline are now **RESOLVED**; remaining and new items are noted below.

| ID | Severity | Status | Issue / current state | Evidence (file:line) |
|---|---|---|---|---|
| DG-01 | CRITICAL | **RESOLVED** | `x-facility-id` now injected from `FACILITY_ID` AsyncStorage key (set from facility-config API). Residual: header is best-effort, not enforced before Home loads — see S-01. | `Api/index.tsx:87-90`, `HomeScreen/index.tsx:765-766` |
| DG-02 | HIGH | **PARTIALLY RESOLVED** | PRE_PRODUCTION now points at the correct Senior Living backend; STAGING correct. **PRODUCTION still points at the Shashi Hotels backend** (`api.hospitality.andmv.com`). `react-native-config` build-time injection still absent. | `src/utils/local.constants.ts:29-31` |
| DG-03 | HIGH | **RESOLVED** | Salon booking now POSTs via `bookSalonAppointment` / `rescheduleSalonAppointment`; reschedule + cancel implemented. | `salon/index.tsx:220-296`, `SalonBookAppointmentScreen/index.tsx:343-352` |
| DG-04 | HIGH | **PARTIALLY RESOLVED** | Foreground FCM handler wired (`onMessage` → `@notifee` local notification). **Still missing:** background (`onNotificationOpenedApp`) and terminated (`getInitialNotification`) tap routing. | `foregroundNotifications.ts`, `App.tsx:21-24` |
| DG-05 | HIGH | **RESOLVED** | MySchedule now fetches `GET /api/unified-schedule?date=` and supports cancel. | `MyScheduleScreen/index.tsx:45,191`, `MyScheduleScreen/appointmentActions.ts` |
| DG-06 | HIGH | **RESOLVED** | Physical Therapy flow rebuilt — list/request/slot screens registered; navigation targets exist. | `appstack/index.tsx:102-114`, `PhysicalTherapyList/index.tsx` |
| DG-07 | HIGH | **RESOLVED** | Private Training flow rebuilt — sessions/service-list/book/confirm screens registered. | `appstack/index.tsx:133-137`, PrivateTrainingSessions/* |
| DG-08 | MEDIUM | **RESOLVED** | Cancellation now uses `cancelSalonAppointment` (`POST /api/salon/cancel-appointment`); the malformed `updateSalonAppointment({headers})` call is no longer used by the booking flow. | `salon/index.tsx:271-296` |
| DG-09 | MEDIUM | **RESOLVED** | Resident Directory now calls `fetchResidentContacts` (`GET /api/residents/getContact`) with favourite toggle. | `ResidentDirectoryScreen/index.tsx:17,75`, `services/Home/resident/index.ts` |
| DG-10 | MEDIUM | **RESOLVED** | Upcoming Appointments now calls `fetchUpcomingUnifiedSchedule` (`GET /api/unified-schedule`). | `UpcomingAppointment/index.tsx:48,103` |
| DG-11 | MEDIUM | **RESOLVED** | Cognitive Evaluation HealthScreen card now navigates to `COGNITIVE_EVALUATION_LIST_SCREEN`. | `HealthScreen/index.tsx:88,112` |
| DG-12 | MEDIUM | **MOSTLY RESOLVED** | Physical Therapy, Cognitive Evaluation, Outside Agency are now API-backed (`/api/care`). **Medication** and **Advanced Care Directive** remain mock (hardcoded `MED_DATA` / `DOCUMENTS`). | `HealthScreen/Medication/index.tsx:66`, `AdvancedCareDirective/index.tsx` |
| DG-13 | MEDIUM | **RESOLVED** | Profile tab now reads Redux `selectUserProfile` and branches on `isFamilyMember`. Care Team reads `/api/staff`. | `ProfileScreen/ProfileScreen/index.tsx:27,49,51` |
| DG-14 | MEDIUM | **OPEN** | Notification preferences toggles still local-state only — no persistence API. | `App/ProfileScreen/NotificationsSettingScreen/index.tsx` |
| DG-15 | MEDIUM | **RESOLVED** | Redundant post-navigation `Alert.alert` removed from salon booking. | `SalonBookAppointmentScreen/index.tsx:306-352` |
| DG-16 | LOW | **PARTIALLY RESOLVED** | Display/Settings reworked: `DisplayScreen` + `ThemeSelectionScreen` drive theme via `ThemeProvider`. Some settings rows remain stubs. | `App/ProfileScreen/DisplayScreen`, `.../SettingsScreen` |
| DG-17 | LOW | **RESOLVED** | Outside Agency now has distinct list / details / request screens, not a single shared form. | `appstack/index.tsx:104,116,111` |
| DG-18 | LOW | **RESOLVED** | `MassageTherapyBookAppointmentScreen` and `PrivateTrainingBookAppointmentScreen` are now registered in AppStack. | `appstack/index.tsx:512-516,493-498` |
| DG-19 | LOW | **RESOLVED** | Massage Therapy booking is now a real flow (`/api/massage/book-appointment`). | `massageTherapy/index.tsx:78-110` |
| DG-20 | LOW | **RESOLVED** | Salon slots now read `res.data.slots ?? []` — the `solts` typo workaround was removed. | `SalonBookAppointmentScreen/index.tsx:226` |
| DG-21 | HIGH | **RESOLVED** | The unreachable hardcoded `App/ProfileScreen/index.tsx` and the old `src/screens/ProfileScreen/**` tree were deleted; one consolidated Profile tree under `App/ProfileScreen/`. | `appstack/index.tsx` (no `App/ProfileScreen/index.tsx` import) |
| DG-22 | MEDIUM | **OPEN (re-scoped)** | `CustomBackground` pub/sub preset system still present; theme selection now flows through `DisplayScreen`/`ThemeSelectionScreen`/`ThemeProvider`. Confirm whether `setBackgroundPreset` is invoked by any user action or is dead. | `components/CustomBackground/index.tsx`, `App/ProfileScreen/ThemeSelectionScreen` |
| DG-24 (new) | MEDIUM | **OPEN** | `dayjs` is imported by `RequestedMealList` and `OtherServiceRequest` but is not declared in `package.json` (carried over from TD-01; relocated files). | `RequestedMealList/index.tsx:31`, `OtherServiceRequest/index.tsx:23` |

---

## 9. Technical Debt

| ID | Severity | Status | Issue | Evidence (file:line) |
|---|---|---|---|---|
| TD-01 | HIGH | **OPEN** | `dayjs` imported but not declared in `package.json` (relocated files; tracked as DG-24). | `RequestedMealList/index.tsx:31`, `OtherServiceRequest/index.tsx:23` |
| TD-02 | HIGH | **OPEN** | Three date libraries still in use (`moment`, `date-fns`, `dayjs`) plus new `dateFormats.ts` helpers. | `package.json`; per-file imports |
| TD-03 | MEDIUM | **OPEN** | Mixed navigator types persist: root/auth native-stack, AppStack legacy `createStackNavigator`. | `appstack/index.tsx:279`; commented import `:83` |
| TD-04 | MEDIUM | **OPEN** | `createHousekeepingRequest` payload still typed loosely (typed version commented out). | `services/services/housekeeping/index.ts:86-99` |
| TD-05 | MEDIUM | **RESOLVED** | Facility coordinates now sourced from Redux `facilityData` (facility-config API); fallback changed to NYC and only used when facility missing. | `TransportationScreen/index.tsx:443` |
| TD-06 | MEDIUM | **OPEN** | `serializableCheck: false` still set; now also covers the global-alert slices. | `store.ts:16` |
| TD-07 | MEDIUM | **OPEN** | No `ErrorBoundary` in the component tree (verify). | Absence throughout codebase |
| TD-08 | MEDIUM | **OPEN** | Default env is a source constant with no build-time injection; now defaults to PRE_PRODUCTION. | `src/utils/local.constants.ts:18` |
| TD-09 | LOW | **OPEN** | `StatusBar` still hardcoded `barStyle="dark-content"` (two instances). | `App.tsx:32,37` |
| TD-10 | LOW | **RESOLVED** | `API_BASE_URL_SECOND` removed from all environments. | `src/utils/local.constants.ts` |
| TD-11 | LOW | **RESOLVED** | Old hardcoded `*Screenlist` mock screens deleted; replaced by API-backed sessions/list screens with working cancel/reschedule. | `MassageTherapy/*`, `PrivateTrainingSessions/*` |
| TD-12 | LOW | **MOSTLY RESOLVED** | Interceptor logs now `__DEV__`-guarded with header masking. A few unguarded `console.log` remain (SplashScreen, TransportationScreen). See S-07. | `SplashScreen/index.tsx:47-89`, `TransportationScreen/index.tsx:436,447` |
| TD-13 | LOW | **RESOLVED** | `pravatar.cc`, `images.unsplash.com`, `picsum.photos` no longer referenced anywhere in `src/`. | (no matches in `src/`) |
| TD-14 | LOW | **RESOLVED** | The hardcoded PhysicalTherapyForm screen was deleted in the Physical Therapy rebuild. | (file removed) |
| TD-15 | LOW | **RESOLVED** | Old `src/screens/ProfileScreen/NotificationsSettingScreen` (with the dead `setSalonAlert;`) deleted; new screen under `App/ProfileScreen/`. | (file removed) |
| TD-16 (new) | INFO | **IMPROVED** | Test coverage went from ~1 file to ~60 `*.test.ts(x)` files across screens/store/utils, with `jest.setup.js` added. Real regression coverage now exists. | `find src -name '*.test.ts*'` → 60 |
| TD-17 (new) | LOW | **OPEN** | `CancelOutsideAgencyService` (`src/services/health/index.tsx:525-551`) is the generic cancel function for **all three** care types (`DELETE /api/care/cancel/:id`), called from `PhysicalTherapyListScreen`, `CognitiveEvaluationListScreen`, and `OutsideAgencyServiceListScreen`. The name implies Outside-Agency-only scope and is misleading for future maintenance — a PT/CE-specific bug fix could be missed if a reviewer assumes the function is OA-scoped. Rename to a generic name (e.g. `CancelCareAppointment`). | `services/health/index.tsx:525-551`; call sites in `PhysicalTherapyList/index.tsx`, `CognitiveEvaluationList/index.tsx`, `OutsideAgencyServiceList/index.tsx` |
