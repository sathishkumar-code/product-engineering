# Architecture: senior_living_skillednursing_resident

> **Doc status:** v3.1 re-verified against `origin/master` HEAD `f22dc60` (2026-08-28). Prior baseline was v3.0, `origin/master` HEAD `4734daf` (2026-08-05) — that baseline is folded into §1–§9 below and marked accordingly; the `staging` HEAD `026ea88` (2026-06-17) and `production` HEAD `bd01f4c` (2026-06-03) references are retained as deeper historical anchors. This pass reviews `4734daf..origin/master` (~30 commits, heavily merge-of-merge from one long-lived `kapil/expo_eas_setup` branch merged back three times — reviewed via net diff, not commit-by-commit) and covers two headline additions: an **Expo/EAS Update OTA pipeline** and a **device-level biometric App Lock**, both of which explicitly mirror equivalent work already shipped in `senior_living_staffapp` (cross-referenced below). Local `production` and local `master` checkouts on a dev machine may both be stale relative to `origin/master` — every claim below is verified via `git show`/`git diff`, never the working tree.
> Related: [../architecture-senior-living-product.md](../architecture-senior-living-product.md) | [./adr/](./adr/) | [./architecture-senior_living_staffapp.md](./architecture-senior_living_staffapp.md) (biometric App Lock / OTA precedent, Design Gap G14)

---

## 1. Purpose

`senior_living_skillednursing_resident` is the resident-facing React Native mobile application (iOS + Android) for the Shashi.AI Senior Living platform. It is the primary digital self-service touchpoint for skilled nursing residents in an assisted-living facility.

**Concrete production responsibilities (all code-verified against `origin/master`):**

- Authenticate residents via AWS Cognito — **now fully passwordless** (`USER_AUTH` flow, `EMAIL_OTP`/`SMS_OTP` challenge). Source: `src/services/Auth/cognitoUserAuth.service.ts`, `src/screens/Auth/SignInScreen/index.tsx`. See §4.2.
- Gate every API call and socket connection behind the resident's facility using the `x-facility-id` header, written to AsyncStorage by `HomeScreen` after the first successful `GET /api/config/residency-details` call. Source: `src/services/Api/index.tsx:87-90` (line numbers per the 026ea88 baseline; interceptor logic unchanged in shape at `origin/master`, see §4.3).
- Display a home dashboard (greeting, announcements preview, upcoming appointment strip, quick actions: calendar, dining, directory, concierge call). Source: `src/screens/App/HomeScreen/index.tsx`.
- Manage wellness services: book, reschedule, and cancel salon, massage therapy, and private training appointments; submit housekeeping/other service requests; book transportation rides with Google Places address input, **now with pickup-time selection, traffic-aware distance, and server-side scheduling-conflict handling** (§3.7e). Source: `src/screens/App/ServicesScreen/`.
- Access health records: view and book physical therapy and cognitive evaluation sessions; view outside agency services; browse medications; upload and view Advanced Care Directives; view lab results (PDF); browse and filter rehab appointment history; message the rehab team; view Care Conference summaries/history. Source: `src/screens/App/HealthScreen/`.
- Participate in real-time secure messaging with the care team via a Socket.io chat channel. Source: `src/services/ChatSocket/index.ts`, `src/hooks/useChatSocketLifecycle.ts`.
- Receive foreground push notifications (FCM + Notifee), with an **explicit OS notification-permission request helper** added (`src/services/Notifications/permissions.ts`, §3.7f). Background notifications are still NOT handled — see Design Gaps (BLOCKER, unresolved since the last review).
- Manage resident profile: edit personal details, manage family members, upload profile picture, upload TV gallery images, manage notification preferences, select display theme. **Change-password / forgot-password UI has been removed entirely** (obsoleted by passwordless sign-in — §4.2). Source: `src/screens/App/ProfileScreen/`.
- Pair the resident's mobile session with the facility TV app via QR code scanning. Source: `src/screens/App/ProfileScreen/LoginTvApp/QRScannerScreen.tsx`.
- Browse facility announcements (paginated, now with audience filtering — §3.7g), resident directory, and activity schedule. Source: `src/screens/App/HomeScreen/Announcements/`, `src/screens/App/Activities/`.
- **Sign consent forms** (new) — residents/family co-sign facility consent documents (PDF, per-signature regen) through the same PDF-viewer UI previously built for the acknowledgment gate. Source: `src/screens/App/HomeScreen/Acknowledgment/index.tsx` (dual-mode), see §3.7c.
- **React to an admin-revoked-access signal** (new) — any API response carrying `data.isRevoked` forces a client-side logout + reset-to-SignIn, independent of the normal 401 refresh path. Source: `src/services/Api/index.tsx`, see §4.4a.
- **Hidden developer environment switcher** (new, undocumented in-product) — 7 taps on the Profile screen logo + a hardcoded password reveals a screen to switch the app's backend target (staging/pre-production/production) at runtime. Source: `src/screens/App/ProfileScreen/ProfileScreen/index.tsx`, `src/screens/App/ChangeEnvironmentScreen/`, see §7.11 (new Design Gap).
- **Gate the whole authenticated app behind a device-level biometric App Lock** (new) — Face ID/Touch ID/device passcode, facility-configurable (`residentBiometricEnabled`, `residentLockScreenTimeout`), enforced both right after first OTP sign-in (`MFAVerifyScreen`) and on cold-start session resume (`SplashScreen`). Source: `src/services/Biometrics/biometrics.service.ts`, `src/components/App/AppLock/`, `src/utils/auth/enterApp.ts`. **Explicitly mirrors `senior_living_staffapp`'s same feature, including its device-scoped-not-account-scoped enrollment characteristic** — see §3.12, §4.2a, §7.12 (new Design Gap cross-referencing staffapp's G14).
- **Check for and apply JS-bundle OTA updates** (new) — Expo Updates (`expo-updates`) polls a backend-gated eligibility endpoint on every foreground transition; a user-confirmed reload applies the update without an app-store release. Source: `src/hooks/useOtaUpdate.ts`, `src/hooks/useOtaUpdateToast.ts`, `src/components/GlobalAlert/GlobalOtaBanner.tsx`, `scripts/configure-ota-channel.js`. See §3.13, §4.2b.

**New since `bd01f4c` and folded into `origin/master`** (all verified present at `origin/master`; this is the v2.1-documented delta, unchanged in substance):

- **Care Conference** module: residents view upcoming and historical care-conference summaries, drill into conference details, and join in-person/virtual conferences. Screen group `src/screens/App/HealthScreen/CareConferenceScreen/`. Service `src/services/App/index.ts`.
- **HIPAA inactivity auto-logoff**: `SessionGuard` wraps the authenticated `AppStack`, idle timer + Keychain-backed last-active timestamp. `IDLE_TIMEOUT` is still 7 days at `origin/master` (test values still commented out) — see Design Gaps (unchanged, still BLOCKER-adjacent).
- **Force/optional app-update gate**: Splash calls `checkForceUpdate()` before auth.
- **Resident-acknowledgment gate**: profile-driven PDF acknowledgment before app use — **now dual-mode with consent-form signing**, see above and §3.7c.
- **Resident-discharge flow**: discharged residents are logged out to a dedicated `DischargedScreen`.
- **Terms & Conditions + Privacy WebView**: in-app `WebViewScreen`/`TermsAndConditionsScreen`.
- **App-wide font-size control**: `getFontSize()` helper, `ModifyFontSizeScreen`.
- **Custom forgot-password OTP flow** — **superseded**. The `bd01f4c`→`026ea88` custom-OTP forgot-password path (`POST /api/auth/forgot-password` / `POST /api/auth/reset-password`, `ForgotPasswordScreen`/`ResetPasswordScreen`) was itself **deleted** at `origin/master` and replaced by the passwordless Cognito `USER_AUTH`/OTP flow described in §4.2. See §8 for the churn this represents.
- **Message bottom tab removed** (still commented out, joins the already-commented Profile tab, unchanged at `origin/master`).

**New at `origin/master` HEAD (since the 026ea88 baseline — 56 merges, ~13 distinct feature branches; this was the incremental review scope for the v3.0 pass):**

1. **Passwordless Cognito sign-in rewrite** (`kapil/KapilLoginFlow_Feature`, `kapil/PendingSignByFamily_Feature`) — full replacement of the phone+password Cognito login with an email-or-phone, OTP-challenge login using the AWS SDK directly. `ChangePasswordScreen`, `ForgotPasswordScreen`, `ResetPasswordScreen`, and `ChangeUserPassword` (Profile) screens and their tests were **deleted outright** (not deprecated). See §4.2, §3.1, §3.6.
2. **Consent-form co-signing** (`kapil/PendingSignByFamily_Feature`) — the resident-facing counterpart of the backend's new `ConsentForm` model/`/api/consent-forms` API (per `architecture-senior_living_backend.md` v2.5). Reuses the Acknowledgment PDF-viewer screen in a new "consent mode." See §3.7c.
3. **Admin-revoked-access handling** (`kapil/PendingSignByFamily_Feature`) — Axios interceptor now force-logs-out on any response carrying `isRevoked: true`. See §4.4a.
4. **Transportation pickup-time + conflict handling** (`PickupTimeTransportation`, `TransportationUIChanges`, `TransportationDetailsShow`, `sagar_development`) — booking now surfaces a resolved pickup time (traffic-aware distance/duration) and a `SchedulingConflictModal` for backend 409 conflict responses; a new `TransportationDetailsScreen` (route) was added. See §3.7e.
5. **Dining screen re-platformed onto Menu2U tray cards** (`sagar_development` era) — `DiningScreen` now calls `fetchDiningTrayMenu` (backend's new `DiningTrayCard`/`/api/dining-tray` feature) instead of `fetchMenuByDate`. Daily-specials, diet-plan cards, and the booking calendar are **commented out** in the same screen ("TEMP: … kept for restore") rather than removed — see §9 Tech Debt.
6. **Notification-permission request helper** (`NotificationPermissionIssue`) — explicit `notifee.requestPermission()` call added for Android 13+ `POST_NOTIFICATIONS`, since `messaging().requestPermission()` is a no-op on Android. See §3.7f.
7. **Hidden dev environment switcher** (`UIChnagesHomeScreen`/`sagar_development` era) — `ChangeEnvironmentScreen` + `DeveloperPasswordAlert`, gated by a hardcoded password in `ProfileScreen`. See §7.11.
8. **`trustAllCerts` fixed** on both PDF viewers (lab-report `TestResult/PDFViewerScreen` and the Acknowledgment/Consent screen) — both now `trustAllCerts={false}`. The DiningScreen's (commented-out) PDF usage still shows `trustAllCerts={false}` in the disabled block. This **resolves** the HIGH-severity MITM Design Gap from the prior review. See §7.5 (updated) and §8.
9. **CustomImage migrated to `@d11/react-native-fast-image`** with SVG-URL detection (`SvgUri`), replacing RN's built-in `Image`. `UserAvatar` shared component added (initials/color avatars via new `avatarColors.ts` util), adopted in `ConversationScreen`, `CareTeamScreen`, `ProfileScreen`.
10. **Hermes/AWS-SDK polyfill layer added to `index.js`** — `react-native-get-random-values`, `react-native-url-polyfill/auto`, `buffer`, `web-streams-polyfill`, `text-encoding`, plus manual `global.structuredClone`/`Blob.prototype.arrayBuffer` shims — required because the new `@aws-sdk/client-cognito-identity-provider` + `@smithy/*` client (item 1) does not run on Hermes without them. **No background FCM handler was added** — the BLOCKER from the prior review is unchanged. See §3.7f, §8.
11. **Announcements gained audience filtering + shared date-format helpers** (`kapil/AnnouncementUIChangeProfileUpdate` era, `date.ts` new utils).

**New at `origin/master` HEAD (since the `4734daf` baseline, 2026-08-05 → 2026-08-28 — ~30 commits/merges, dominated by one long-lived `kapil/expo_eas_setup` branch merged back into `master` three times; this is the incremental review scope for this v3.1 pass):**

1. **Expo/EAS Update OTA pipeline** (`kapil/expo_eas_setup`) — the bare RN native projects (Android/iOS) are now layered with Expo's build tooling (`babel-preset-expo`, `expo/metro-config`, `expo-updates`, `expo-asset`, `expo-modules-core`) to support JS-bundle-only over-the-air updates via EAS Update, **without** migrating to Expo's managed workflow (no `expo prebuild`; native project files remain hand-maintained and committed). See §2, §3.13, §4.2b.
2. **Biometric App Lock / enrollment** (`kapil/BiometricLogin_Feature`, merged in via `expo_eas_setup`) — device-level Face ID/Touch ID/passcode gate. Its own code comments state it "mirrors staffapp's `resolveGateTarget` exactly." See §3.12, §4.2a, §7.12.
3. **React Native 0.84.1 → 0.85.3 bump**, alongside `@react-native/*` toolchain packages, `react-native-reanimated` 4.3.0→4.3.1, `react-native-worklets` **downgraded** 0.10.2→0.8.3, `react-native-gesture-handler` →~2.31.1, TypeScript 5.8.3→~6.0.3. See §2, §9.
4. **Android release-signing now falls back to the debug keystore** when no release keystore is present (`hasReleaseKeystore` check in `android/app/build.gradle`) — a local/dev-convenience change (`npm run apk`/`npm run aab`); CI's `sign` job always supplies `KEYSTORE_FILE_TMP` so this does not affect the shipped signing path today. See §7.13.
5. **GitLab CI gained OTA-channel + `APP_ENVIRONMENT` branch wiring** in `build_android`/`build_ios` (branch-name `case` statements mapping `development`→staging/STAGING, `master`→preprod/PRE_PRODUCTION, `ota-release`→production/PRODUCTION); **the `production` branch itself is not an explicit case and falls through to the wildcard/no-op branch in both statements** — see §7.14 (new Design Gap).
6. **Package-manager supply-chain override** — `package.json` `resolutions`/`overrides` force `@agnoliaarisian7180/string-argv` (an odd, non-Shashi-owned scoped package name, almost certainly pulled in transitively via the new `eas-cli`/Expo tooling) to resolve to the legitimate `npm:string-argv@0.3.2` instead. Appears to be an intentional supply-chain mitigation already in place — do not remove this override without confirming why it was added.

**Backend:** `senior_living_backend` — Node/Express TypeScript, port 7000. Base URL per environment in `src/utils/local.constants.ts`. See [../architecture-senior_living_backend.md](../architecture-senior_living_backend.md).

**Out of scope (not present in production code):** hotel reservation/PMS flows, multi-facility switching (resident belongs to exactly one facility), staff operational views, facility admin operations.

---

## 2. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Framework | React Native (New Architecture enabled) | **0.85.3 (bumped this pass from 0.84.1 — see §1)** |
| UI runtime | React | 19.2.3 |
| Language | TypeScript | **~6.0.3 (bumped this pass from 5.8.3)** |
| State management | Redux Toolkit + react-redux | 2.11.2 / 9.2.0 |
| HTTP client | Axios | 1.14.0 |
| Auth (legacy — refresh/logout only) | amazon-cognito-identity-js | 6.3.16 |
| **Auth (new — passwordless sign-in)** | **@aws-sdk/client-cognito-identity-provider** | **^3.1091.0 (new)** |
| **Auth transport (new)** | **@smithy/fetch-http-handler** | **^5.6.8 (new — Hermes has no `node:https`, so the SDK's default `NodeHttpHandler` is swapped for a fetch-based one)** |
| Build-time config | react-native-config | 1.6.1 |
| Real-time | socket.io-client | 4.8.3 |
| Push (FCM) | @react-native-firebase/messaging | 23.8.3 |
| Push display | @notifee/react-native | 9.1.8 |
| Firebase (analytics, crashlytics, auth) | @react-native-firebase/\* | 23.8.3 |
| Navigation | @react-navigation/native + native-stack + stack + bottom-tabs | 7.2.2 / 7.14.10 / 7.8.9 / 7.15.9 |
| Token storage | @react-native-async-storage/async-storage | 2.2.0 |
| Credential storage | react-native-keychain | 10.0.0 — **now also backs biometric App Lock device enrollment** (§3.12, §5.2), in addition to remember-me identifiers and the HIPAA session timestamp |
| JWT decode | jwt-decode | 4.0.0 |
| Phone normalization | libphonenumber-js | 1.12.41 |
| Camera (QR scan) | react-native-camera-kit | 17.0.1 |
| PDF viewer | react-native-pdf | 7.0.4 |
| Document picker | @react-native-documents/picker | 12.0.1 |
| Image picker (crop) | react-native-image-crop-picker | 0.51.1 |
| Image picker (plain) | react-native-image-picker | 8.2.1 |
| **Image display (new)** | **@d11/react-native-fast-image** | **^8.13.0 (new — replaced RN `Image` in `CustomImage`)** |
| File transfer | react-native-blob-util | 0.24.7 |
| Google Places | react-native-google-places-autocomplete | 2.6.4 |
| OTP hint (Android) | react-native-otp-verify | 1.2.0 (iOS now uses a local no-op stub, `src/utils/react-native-otp-verify.ios.ts`, swapped in via a Metro `resolveRequest` — this package is Android-only) |
| Calendar UI | react-native-calendars | 1.1314.0 |
| Animation | react-native-reanimated | **4.3.1 (bumped this pass from 4.3.0)** |
| Worklets | react-native-worklets | **0.8.3 (downgraded this pass from 0.10.2 — likely an Expo/reanimated-4.3.1 compatibility pin; see §9)** |
| Gestures | react-native-gesture-handler | **~2.31.1 (bumped this pass from ^2.30.1)** |
| QR display | react-native-qrcode-svg | 6.3.21 (still a dead dependency — see §9) |
| Date utilities | date-fns + moment | 4.1.0 / 2.30.1 |
| PDF generation | jspdf | 4.2.1 |
| Tabs UI | react-native-tab-view | 4.3.0 |
| Pager | react-native-pager-view | 8.0.1 |
| Dropdown | react-native-element-dropdown | 2.12.4 |
| Clipboard | @react-native-clipboard/clipboard | 1.16.3 |
| Permissions | react-native-permissions | 5.5.1 |
| WebView (T&C / Privacy / Acknowledgment) | react-native-webview | 13.16.1 |
| Device / build info (force-update) | react-native-device-info | 15.0.2 |
| Android SMS OTP autofill | @eabdullazyanov/react-native-sms-user-consent | 1.3.0 |
| OTP code field | react-native-confirmation-code-field | 8.0.1 |
| **Dev-only app restart (new)** | **react-native-restart** | **^0.0.28 (new — powers the hidden `ChangeEnvironmentScreen`, §7.11)** |
| **Node polyfills (new)** | **buffer, stream-browserify, events, web-streams-polyfill, react-native-get-random-values, react-native-url-polyfill, text-encoding** | **new — required for the AWS SDK Cognito client to run under Hermes; `text-encoding` is imported directly by `index.js` but is undeclared in `package.json` (present only via `package-lock.json` as a transitive resolution) — see §9** |
| **Expo/EAS OTA tooling (new)** | **expo, expo-updates, expo-asset, expo-modules-core, babel-preset-expo, eas-cli** | **^56.0.11 / ~56.0.19 / ^56.0.17 / ^56.0.17 / ^56.0.15 / ^20.1.0 — bare-workflow OTA layer, not managed Expo; see §3.13** |
| **OTA build/publish scripting (new)** | **env-cmd, babel-plugin-transform-inline-environment-variables** | **^11.0.0 / ^0.4.4 — power the `npm run update:{staging,preprod,production}` EAS Update scripts, §3.13** |
| Node runtime (CI) | Node.js | `package.json` `engines` **relaxed this pass from `>= 22.11.0` to `>= 20.0.0`**, while GitLab CI still explicitly runs `nvm install 22` — the two are no longer in sync, see §9 |
| Android bundle ID | com.shashigroup.sl.resident | `android/app/build.gradle` |
| Android versionCode / versionName | **29 / "1.3.15"** (corrected this pass — the previously-documented "11 / 1.3.4" was already stale relative to the v3.0-pass codebase, which was actually at 20/"1.3.13") | `android/app/build.gradle` |
| iOS App Store ID (force-update deep-link) | 6768239464 | `src/utils/local.constants.ts` |
| CI/CD | GitLab CI + Fastlane | — |
| Security scanning | SonarQube, Semgrep, Snyk, Trivy, Gitleaks | `security-scan` job still `only: - master`, now also `allow_failure: true` — see §8 |

---

## 3. Key Components

### 3.1 Navigation Stacks

Same 4-navigator / 3-level shape as the prior review; the whole `AppStack` file was also re-indented (one extra nesting level) as part of this range, which inflates its diff without changing routes beyond the two below.

| Navigator | Type | File |
|---|---|---|
| RootStack | native-stack | `src/navigation/rootstack/index.tsx` |
| SplashStack | (nested in Root) | `src/navigation/splashstack/` |
| AuthStack | native-stack | `src/navigation/authstack/index.tsx` |
| AppStack | legacy `createStackNavigator` (JS-driven) | `src/navigation/appstack/index.tsx` |
| BottomTabNavigator | bottom-tabs | `src/navigation/appstack/BottomTabNavigator.tsx` |
| skilledNursingAppStack | constants/types only (no navigator) | `src/navigation/skilledNursingAppStack/` |

**AuthStack routes — 3 removed, param shape of MFA_VERIFY changed:** `CHANGE_PASSWORD`, `FORGOT_PASSWORD`, `RESET_PASSWORD` are **gone** from `AuthScreens`, the param list, and the `<Stack.Screen>` registrations (verified: zero remaining references to `CHANGE_PASSWORD`/`FORGOT_PASSWORD`/`RESET_PASSWORD` anywhere in `src/` at `origin/master` — clean removal, no orphaned navigation calls). `MFA_VERIFY` params changed from `{ cognitoUser, phoneNumber, password }` to `{ username, session, contact, challengeType: 'EMAIL_OTP' | 'SMS_OTP' }` — this is now an OTP-challenge screen, not a legacy TOTP/SMS-MFA screen. Remaining routes: `SIGNIN`, `MFA_VERIFY`, `ACCESS_DENIED`, `TERMS_AND_CONDITIONS`, `DISCHARGED`, **`BIOMETRIC_ENROLL` (new — §3.12)**.

**AppStack — net route changes:** `CHANGE_USER_PASSWORD` removed (its screen was deleted, §3.6); `TRANSPORTATION_DETAILS_SCREEN` and `CHANGE_ENVIRONMENT` added. `CHANGE_ENVIRONMENT` is reachable only via the hidden 7-tap gesture in `ProfileScreen` (§7.11), not from any visible menu entry.

**BottomTabNavigator — unchanged at `origin/master`:** still 4 active tabs (Home, Services, Health, MySchedule), Message and Profile tabs still commented out (`BottomTabNavigator.tsx`).

### 3.2 Redux Store — 6 Slices (was 4)

`user`, `dashboard`, `globalAlert`, `globalQuestionAlert` unchanged in shape from the prior review. **Two new slices this pass:** `appLock` (`src/store/features/appLock/appLock.slice.ts` — `status`, `biometryType`, `isPrompting`, `suppressLockUntil`, `isBiometricEnabled`; §3.12) and `otaBanner` (`src/store/features/otaBanner/otaBanner.slice.ts` — a single `visible` boolean driving `GlobalOtaBanner`; §3.13). `globalAlert.slice.ts`/`globalAlert.helper.ts` gained minor additions used by the `AccessRevoked` alert (§4.4a) and consent-form flow. Still no RTK Query.

### 3.3 API Service Modules

Base Axios wiring unchanged in shape (§3.6). Endpoint-level additions at `origin/master`:

| Module | File | New/changed endpoints |
|---|---|---|
| Home | `src/services/Home/index.tsx` | `fetchDiningTrayMenu` (new — Menu2U tray-card menu, replacing `fetchMenuByDate` in `DiningScreen`); `fetchConsentFormPdf` / `signConsentForm` (new — consent-form co-signing, reused by `AcknowledgmentScreen`) |
| Auth | `src/services/Auth/cognitoUserAuth.service.ts` (new) | Calls AWS Cognito `InitiateAuthCommand` (`USER_AUTH` flow) / `RespondToAuthChallengeCommand` directly — no backend REST call for sign-in; `resendOtp` used by `MFAVerifyScreen` |
| Transportation | `src/screens/App/ServicesScreen/Transportation/*` (via `Home`/`services` modules) | Booking response now includes `pickupTime` (UTC ISO) and `distance`; a 409 response returns `conflicts: TransportConflict[]` consumed by `SchedulingConflictModal` |
| **App (new this pass)** | `src/services/App/index.ts`, `src/services/App/type.ts` | **`fetchOtaEligibility` (`GET /api/ota/check`) / `logOtaDelivery` (`POST /api/ota/log`)** — OTA-update eligibility/delivery-logging, §3.13, §6 |

The `App`/Care-Conference and legacy custom-OTP `Auth` services described in the prior review (`checkTempPassword`, `sendForgotPasswordOtp`, `resetPasswordWithOtp`) are **removed** along with the screens that called them (§3.6).

### 3.4 Socket.io (Chat)

Unchanged in shape from the prior review — namespace, transport, auth headers, event set, and the FACILITY_ID race (§4.3, §8) are all still as previously documented. `useChatSocketLifecycle.ts` gained 3 lines (minor — no behavioral change to the connect-time facility-id read identified in the diff).

### 3.5 Cognito Auth Config — Split Across Two Services (new shape)

`src/services/Auth/cognito.config.ts` was extended (not replaced): the original synchronous `COGNITO_CONFIG` export backing the legacy `amazon-cognito-identity-js` path is still present, plus a new async `getActiveCognitoConfig()` that also resolves the AWS `Region` from the pool ID prefix and re-reads the runtime-selected environment from `AsyncStorage[CURRENTENVIRONMENT]` on every call (used by the new AWS-SDK Cognito client, §3.6a). Both configs read the same `react-native-config` env keys (`COGNITO_STAGING_*`, `COGNITO_PRE_PROD_*`, `COGNITO_PRODUCTION_*`).

### 3.6 Axios Singleton (`src/services/Api/index.tsx`)

Interceptor shape from the prior review is retained (baseURL resolution, Authorization/pushToken/x-fcm-token/x-facility-id headers, `__DEV__`-guarded logging with `maskHeaders()`, 401 refresh-without-queueing). **New at `origin/master`:**

- Every outbound request now carries `config.headers.isFromMobileRequest = true` — an explicit channel marker for the backend (plausibly consumed by the backend's new `POST /api/auth/login-mfa-channel` channel-pinning logic per `architecture-senior_living_backend.md` v2.5, though this repo does not itself call that endpoint — see §4.2 note).
- **Admin-revoked-access handling** — both the 2xx and non-2xx response interceptor branches now check `data?.data?.isRevoked` (or `error.response?.data?.data?.isRevoked`); on a match, `handleRevokedAccess()` calls `logout()` + `clearForLogout()`, dispatches `clearDashboard()`/`clearUser()`, resets navigation to `AUTH/SIGNIN` via the new `resetToSignIn()` util (`src/utils/auth/resetToSignIn.ts`), and shows an `AccessRevoked` alert. A module-level `isHandlingRevocation` flag deduplicates concurrent triggers. See §4.4a.

### 3.6a Passwordless Sign-In Service (new)

`src/services/Auth/cognitoUserAuth.service.ts` — a lazy-singleton `CognitoIdentityProviderClient` (`@aws-sdk/client-cognito-identity-provider`, `requestHandler: new FetchHttpHandler()` to avoid the SDK's default Node-only HTTP handler). `initiateUserAuth(username)`:

1. Sends `InitiateAuthCommand({ AuthFlow: 'USER_AUTH', AuthParameters: { USERNAME, PREFERRED_CHALLENGE } })`, where `PREFERRED_CHALLENGE` is derived client-side from whether `username` looks like an email (`EMAIL_OTP`) or not (`SMS_OTP`).
2. If Cognito responds `SELECT_CHALLENGE`, immediately responds with `RespondToAuthChallengeCommand` selecting the preferred challenge (or throws if unavailable).
3. On an `EMAIL_OTP`/`SMS_OTP` challenge, returns `{ status: 'OTP_CHALLENGE', username, session, challengeType }` for `SignInScreen` to hand off to `MFAVerifyScreen`.
4. On immediate success (no challenge), returns `{ status: 'SUCCESS' }` and tokens are saved via the shared `token.manager.saveTokens`.

This coexists with — does not replace — the legacy `cognito.service.ts` (`amazon-cognito-identity-js`), which is now reduced to `refreshSession()` and `logout()` only (the old `signIn()`/password/MFA logic was removed from that file). **Both Cognito client stacks are now bundled and initialized** — see §9 Tech Debt for the maintenance-burden implication.

### 3.7 Push Notification Wiring

Foreground FCM→Notifee wiring, chat-notification press-action, and the missing background handler are all **unchanged in substance** from the prior review — see §8, this remains an open BLOCKER at `origin/master`.

### 3.7a HIPAA Inactivity Auto-Logoff

Unchanged in shape and in the `IDLE_TIMEOUT = 7 days` misconfiguration (`src/components/SessionGuard/index.tsx` gained 19 lines of net change — appears to be a wiring/import cleanup around the new `resetToSignIn()` util rather than a behavior change to the timeout value itself; the 7-day production constant was not touched in this range).

### 3.7b Force / Optional Update Gate

Unchanged.

### 3.7c Resident Acknowledgment, Consent-Form Signing & Discharge Gates (extended)

`AcknowledgmentScreen` (`src/screens/App/HomeScreen/Acknowledgment/index.tsx`) is now a **dual-mode screen**, driven by whether `route.params.consentFormId` is present:

- **Acknowledgment mode** (`acknowledgmentUrl` param, unchanged from prior review): renders the facility acknowledgment PDF, submits via `PATCH /api/residents/acknowledge` (`submitAcknowledgment`).
- **Consent mode** (new — `consentFormId` param): on mount, fetches the consent-form PDF URL via `fetchConsentFormPdf(consentFormId)`; on submit, calls `signConsentForm(consentFormId)` and clears `requiresConsentSignature` in the Redux user-profile slice only if the response reports the form is now fully `Signed`. Both `HomeScreen`'s initial gate check and its post-`fetchResidentProfile` re-check now branch three ways: `Discharged` → `DischargedScreen`; `acknowledgement === false` → Acknowledgment mode; `requiresConsentSignature === true` → Consent mode (passing `consentFormId: currentProfile.id` — **note:** the field name is `consentFormId` but the value passed is the *resident's* `id`, not a distinct consent-form id; the backend endpoint presumably resolves the pending form by resident, but this is worth confirming against the backend's `/api/consent-forms` contract, since the naming implies a form-specific id). Source: `src/screens/App/HomeScreen/index.tsx` (gate logic around resident-profile handling), `src/screens/App/HomeScreen/Acknowledgment/index.tsx`.
- Both PDF renders now use `trustAllCerts={false}` (was `true` in the acknowledgment path at the 026ea88 baseline for `TestResult`'s lab-report viewer only — the Acknowledgment screen's own viewer is verified `false` at `origin/master`). See §7.5/§8 for the resolved MITM gap.

Discharge handling is unchanged. **Note:** consent/acknowledgment gate resolution now runs *after* the new biometric App Lock gate resolves (§3.12, §4.2a) — a resident who fails/cancels biometric unlock never reaches this gate at all in the same session.

### 3.7d App-wide Font Scaling

Unchanged.

### 3.7e Transportation Booking — Pickup Time, Traffic-Aware Distance, Conflict Handling (new)

`TransportationScreen` (`src/screens/App/ServicesScreen/Transportation/TransportationScreen/index.tsx`) gained:

- A resolved **pickup time** (`pickupTime`, a UTC ISO string returned by the backend booking response), parsed with an explicit UTC-aware helper (`parseUTC`) and rendered device-local — the same pattern used by the new `TransportationDetailsScreen`.
- **Traffic-aware distance**: `distanceMiles`/estimated travel time are recomputed at address-select time via the existing `distanceMatrix.ts` util (which itself changed — 34 lines — in this range) and retained across the pickup-time selection step rather than recomputed.
- **Scheduling-conflict handling**: a booking attempt that returns HTTP 409 with a `conflicts: TransportConflict[]` body now opens a new `SchedulingConflictModal` (`src/components/SchedulingConflictModal/`, 199 lines, new) instead of falling through to a generic failure alert. This is the client-side counterpart of the backend's transportation conflict-detection + `forceBook` override (`architecture-senior_living_backend.md` §1 item 9) — this app does not appear to expose a `forceBook` override itself from the modal's visible code path; confirm with backend/product whether force-booking is resident-facing or staff-only by design.
- A new leaf route, **`TransportationDetailsScreen`** (`src/screens/App/ServicesScreen/Transportation/TransportationDetailsScreen/`, 342 lines, new), shows a single ride's date/time/location/status/duration in read-only form.

### 3.7f Notification Permission Request Helper (new)

`src/services/Notifications/permissions.ts` — `requestNotificationPermission()` calls `notifee.requestPermission()` (not `messaging().requestPermission()`) because `messaging().requestPermission()` is a documented no-op on Android and never surfaces the Android 13+ `POST_NOTIFICATIONS` runtime dialog; `notifee.requestPermission()` triggers it. Returns `true` for `AUTHORIZED`/`PROVISIONAL` (iOS) or `AUTHORIZED` (Android). Consumers not fully traced in this pass — grep `requestNotificationPermission` to confirm call sites (likely Splash or first-run onboarding) before relying on this doc for exact wiring.

### 3.7g Announcements Audience Filtering (new)

`src/screens/App/HomeScreen/Announcements/index.tsx` gained an `AnnouncementAudience` type/filter and shared date-format helpers (`formatAnnouncementDateTime`, `formatAnnouncementSection` in the new `src/utils/date.ts`). Not deep-dived further in this pass.

### 3.8 Native Module Inventory

Unchanged from the prior review except: **`@d11/react-native-fast-image`** added (image display, replacing RN `Image` in `CustomImage`, §3.9); **`react-native-restart`** added (dev-only, powers `ChangeEnvironmentScreen`'s environment-switch reload); **`react-native-keychain`** gained a new consumer this pass (biometric App Lock device enrollment, §3.12) on top of its existing remember-me/session-timestamp use; Android manifest gained the **`USE_BIOMETRIC`** permission this pass. `react-native-qrcode-svg` remains installed with zero imports in `src/` — still a dead dependency (§9). `@react-native-firebase/auth` remains installed with no import — still a dead dependency (§9).

### 3.9 Shared UI Components

**New at `origin/master`:** `UserAvatar` (`src/components/UserAvatar/`, 90 lines — initials/color avatar, backed by new `src/utils/avatarColors.ts`; adopted in `ConversationScreen`, `CareTeamScreen`, `ProfileScreen`), `SchedulingConflictModal` (`src/components/SchedulingConflictModal/`, 199 lines, §3.7e), `DeveloperPasswordAlert` (`src/components/DeveloperPasswordAlert/`, 103 lines, §7.11). `CustomImage` (`src/components/CustomImage/index.tsx`) was rewritten (116 lines changed) to wrap `@d11/react-native-fast-image` with an `isSvgUrl()` branch that renders `SvgUri` for `.svg` URLs instead of `FastImage`. **New this pass:** the `AppLock` component family (`src/components/App/AppLock/` — `index.tsx`/`AppLockGate`, `EnrollPromptView`, `SessionLockedView`, `BiometricsUnavailableView`, `BiometricUnavailableDialog`; §3.12) and `GlobalOtaBanner` (`src/components/GlobalAlert/GlobalOtaBanner.tsx`, §3.13).

All previously-documented shared components (`AppButton`, `AppHeader`, `SessionGuard`, `TimeoutWarningModal`, `OTPInput`, `SegmentedControl`, etc.) remain present; `OTPInput` gained a small diff (37 lines) — not deep-dived, likely wiring to the new `MFAVerifyScreen`/`resendOtp` flow rather than a UI change.

### 3.10 Test Coverage

Jest 29.6.3 + @testing-library/react-native 13.3.3. **Test file count dropped from 56 to 52** at the v3.0-pass `origin/master` (`git ls-tree` count on `src/`). The net change reflects: 5 test files deleted alongside their deleted screens (`ChangePasswordScreen`, `ForgotPasswordScreen`, `ResetPasswordScreen`, `ChangeUserPasswordScreen`, plus the `SettingsScreen` test's `-13` line trim for the removed change-password menu entry), offset by rewrites (not net-new files) to `SignInScreen.test.tsx` and `MFAVerifyScreen.test.tsx` to match the new passwordless flow. **This v3.1-pass range (4734daf..origin/master) partially bucks that trend**: `biometrics.service.test.ts` (353 lines, new) and `appLock.slice.test.ts` (83 lines, new) shipped alongside the biometric App Lock feature — the first meaningfully-unit-tested new surface reviewed in this app's history. **Still untested in this range:** `enterApp.ts`/`resolveGateTarget()`, `AppLockGate` itself, `BiometricEnrollScreen`, the `EnrollPromptView`/`SessionLockedView`/`BiometricsUnavailableView`/`BiometricUnavailableDialog` view components, and the entire OTA layer (`useOtaUpdate`, `useOtaUpdateToast`, `GlobalOtaBanner`, `otaBanner.slice`/`otaBanner.helper`, `configure-ota-channel.js`) — contrast `senior_living_staffapp`, which shipped an `enterApp.test.ts` (364 lines) alongside its own biometrics/appLock tests. Combined with the still-untested areas carried over from prior reviews (HomeScreen, the entire chat flow, all service API modules, the Axios 401 interceptor, consent-form signing, admin-revoked-access handling, transportation pickup-time/conflict handling, the Menu2U dining rewrite), test coverage of security- and money/schedule-relevant flows remains a standing gap even where this pass made partial progress.

### 3.11 Provider Nesting Order (`App.tsx`)

Unchanged from the prior review.

### 3.12 Biometric App Lock (new)

`AppLockGate` (`src/components/App/AppLock/index.tsx`, 305 lines) wraps `AppStack` and renders `children` unconditionally, overlaying an opaque lock screen on top when `status === 'locked'` — navigation state and in-progress screen data survive a lock/unlock cycle rather than being unmounted. It arms two independent timers: a **background-duration** check (elapsed time since the app was last backgrounded, compared against a facility-configurable grace period) and a **foreground-inactivity** timer (reset on any touch via a non-consuming `PanResponder` capture handler). Both timers are disabled entirely unless `isBiometricEnabled` (Redux, `appLock` slice) **and** `facilityData.residentBiometricEnabled === true` — i.e. the feature is off by default and requires an explicit facility opt-in.

- **Facility-configurable grace period**: `facilityData.residentLockScreenTimeout` (seconds, from the API) overrides the hardcoded `APP_LOCK_GRACE_MS` (15s) fallback. `AppLockGate` also now refreshes `facilityData` on every foreground transition (new hook, `src/hooks/useOnAppForeground.ts`) — a deliberate fire-and-forget fetch so a facility toggling `residentBiometricEnabled`/`residentLockScreenTimeout` mid-session reaches an already-logged-in device without a fresh login.
- **Enrollment gate — resolved at both entry points**: `resolveGateTarget()` (`src/utils/auth/enterApp.ts`) is called from `MFAVerifyScreen` right after a successful OTP challenge, and from `SplashScreen` on cold-start session resume (which additionally attempts a *silent* biometric unlock directly over Splash, mirroring — per its own code comment — "staffapp's `enterAppThroughBiometricGate`", before falling back to `AppLockGate`'s own lock-screen overlay). It checks **device-level enrollment first** (`isBiometricEnrolled()`) — if the device has ever enrolled, the result is always `'unlock'`, **never** `'enroll'`, regardless of which resident/family account is currently authenticating. Only an unenrolled device consults `facilityData.residentBiometricEnabled`; when true the result is `'enroll'` (mandatory — `BiometricEnrollScreen` has no skip; both the header back arrow and Android hardware back sign the user out instead of bypassing enrollment).
- **Device-scoped, not account-scoped — the same characteristic already flagged for `senior_living_staffapp` (Design Gap G14).** `src/services/Biometrics/biometrics.service.ts:50-53,79-81,131-132` states this explicitly in its own comments: enrollment is a single fixed-label Keychain generic-password item (service `com.shashigroup.sal.resident.app_lock`, label `'device-biometric-lock'`) with no account/resident identifier stored in it at all — "a successful read means the device owner passed Face ID/Touch ID/passcode — it grants access regardless of which resident account is currently signed in." `src/utils/auth/enterApp.ts:13-24` explicitly states this "mirrors staffapp's `resolveGateTarget` exactly." A normal sign-out (`LogUserOut`, `src/utils/auth/LogUserOut.ts`) does **not** clear the Keychain enrollment — only the invalidated-unlock path (`signOutFromGate`, e.g. the device's biometric enrollment itself changed) does — so a family member or a different resident signing into a device a prior user enrolled inherits that device's unlock, exactly as staffapp's G14 describes for shared staff devices. See §7.12 (new Design Gap, cross-referencing staffapp G14) for the product/security decision this needs.
- **ACL upgrade-on-read**: the Keychain item's access-control (`BIOMETRY_ANY_OR_DEVICE_PASSCODE` vs `DEVICE_PASSCODE`-only) is frozen at creation and never re-evaluated by the OS on read; `unlockWithBiometrics()` detects "biometry became available after passcode-only enrollment" via a side-channel AsyncStorage flag (`BIOMETRIC_ENROLLED_WITH_BIOMETRY`) and silently re-enrolls to upgrade it.
- **Tested**: `biometrics.service.test.ts` (353 lines) and `appLock.slice.test.ts` (83 lines) shipped alongside this feature (§3.10). **Not tested**: `enterApp.ts`, `AppLockGate` itself, `BiometricEnrollScreen`, and the supporting view components — see §3.10, §8.
- Android manifest gained the `USE_BIOMETRIC` permission this pass; whether iOS's `NSFaceIDUsageDescription` (`Info.plist`) was added/updated was not independently re-confirmed in this pass — verify before relying on Face ID working on a fresh iOS install.

### 3.13 OTA Updates / Expo Integration (new)

The native Android/iOS projects (still built via Gradle/Fastlane and Xcode/Fastlane, not `eas build`) are now layered with Expo's bare-workflow tooling to support **JS-bundle-only** over-the-air updates via EAS Update — this is **not** a migration to Expo's managed workflow (no `expo prebuild`; native project files remain hand-maintained and committed, as they were before this pass).

- **Client flow** (`src/hooks/useOtaUpdate.ts`, invoked once from `RootNavigator` and on every foreground transition): calls `Updates.checkForUpdateAsync()`; if available, gates behind a backend eligibility check (`GET /api/ota/check` — facility/version-scoped via the existing `x-facility-id` header and JWT, §6), then logs delivery (`POST /api/ota/log`) before fetching the update and offering a user-confirmed reload (`showAppQuestionAlert` + a persistent `GlobalOtaBanner` that reopens the same dialog on tap). Every step fails closed — a failed eligibility check, failed delivery log, or failed download simply skips silently and retries on the next foreground/mount, never partially applies an update. `app.json`'s `updates.checkAutomatically: "NEVER"` ensures this hook is the *only* path that can ever trigger a check/download/apply.
- **Post-reload toast** (`src/hooks/useOtaUpdateToast.ts`): detects "this cold start followed our own OTA reload" via a one-shot AsyncStorage marker (`PENDING_OTA_RELOAD_UPDATE_ID`, written just before `Updates.reloadAsync()` and cleared immediately on next read, not after the toast shows — so a crash mid-flow can lose a toast but never gets stuck retrying it), shows a one-time success alert, and double-guards against re-showing it via `LAST_OTA_TOAST_SHOWN_UPDATE_ID`.
- **Native OTA channel is a build-time, not-auto-injected concern**: `scripts/configure-ota-channel.js` rewrites the channel baked into `AndroidManifest.xml` (`expo-channel-name` request header meta-data) and `ios/SkilledNursing/Supporting/Expo.plist` (`EXUpdatesChannel`) — required because native builds go through Gradle/Fastlane/Xcode directly, not `eas build`, so nothing auto-injects the channel the way it would for an EAS-built binary. The committed manifest/plist default to `"production"`. CI (`build_android`/`build_ios`) runs this script only when its branch-name `case` statement sets a non-empty `OTA_CHANNEL` (`development`→staging, `master`→preprod, `ota-release`→production); **the `production` branch itself is not an explicit case and falls through to the wildcard (`OTA_CHANNEL=""`, script skipped)** — this happens to still be correct today only because the committed manifest/plist default already say `"production"` and the JS-side `APP_ENVIRONMENT` fallback in `local.constants.ts` also defaults to `PRODUCTION` when the env var is unset/invalid. See §7.14 (new Design Gap) for the fragility this creates.
- **`eas.json`** defines `staging`/`preprod`/`production` EAS build profiles (internal-distribution APKs for the first two, an auto-incrementing production profile) and a `google-services-key.json`-backed Play Store submit config. `app.json` carries the EAS project id (`ad0c5b46-08a2-41d4-a2ec-50902f7e29ff`) and the `https://u.expo.dev/<projectId>` update URL.
- **Backend contract (client-visible only** — cross-check `senior_living_backend`'s own architecture doc for the server side): `OtaEligibilityResponse` returns `isNeedOTAUdate: boolean` plus the current/new `OtaVersionData` (facility-scoped, per-platform, includes `runtimeVersion`); `OtaLogParams`/`logOtaDelivery` records `otaVersion`/`previousOtaVersion`/`deliveredAt`. Both requests deliberately omit `cName`/`userLoginType`/`facilityId` params — the backend decodes identity from the JWT and reads facility from the existing `x-facility-id` header. Source: `src/services/App/index.ts`, `src/services/App/type.ts`.
- **Untested**: `useOtaUpdate`, `useOtaUpdateToast`, `GlobalOtaBanner`, `otaBanner.slice`/`otaBanner.helper`, and `configure-ota-channel.js` all ship with zero test files. See §3.10, §8.

---

## 4. Architecture Diagram and Key Flows

### 4.1 Overall Architecture

```mermaid
flowchart TD
    subgraph Device["Mobile Device (iOS / Android)"]
        IDX["index.js\nAppRegistry.registerComponent\n+ Hermes/AWS-SDK polyfill layer\nNO setBackgroundMessageHandler"]
        AT["App.tsx\nRedux Provider\nGestureHandlerRootView\nSafeAreaProvider\nThemeProvider\nLoaderProvider"]

        subgraph Nav["Navigation (3 stacks + 1 tab bar)"]
            ROOT["RootStack (native-stack)\nSPLASH | AUTH | APP\n+ useOtaUpdate/useOtaUpdateToast (new)"]
            AUTH_S["AuthStack (native-stack)\nSignIn | MFAVerify (OTP) | AccessDenied\nTermsAndConditions | Discharged\n+ BiometricEnroll (new)\n(ChangePassword/ForgotPassword/ResetPassword REMOVED)"]
            APP_S["AppStack (JS stack)\nwrapped in SessionGuard (HIPAA auto-logoff)\nwrapped in AppLockGate (biometric App Lock, new)\n+ ChangeEnvironment (hidden dev screen)\n+ TransportationDetails"]
            TABS["BottomTabNavigator\nHome | Services | Health | MySchedule\n(Message + Profile still commented out)"]
        end

        subgraph State["State Layer"]
            REDUX["Redux Store\nuser + dashboard + globalAlert + globalQuestionAlert\n+ appLock + otaBanner (new)"]
            AS["AsyncStorage\nACCESS_TOKEN ID_TOKEN REFRESH_TOKEN\nTOKEN_EXPIRY COGNITO_USERNAME\nFCM_TOKEN FACILITY_ID\nCurrentEnvironment APP_BG_PRESET\nPENDING_OTA_RELOAD_UPDATE_ID LAST_OTA_TOAST_SHOWN_UPDATE_ID (new)\nBIOMETRIC_ENROLLED_WITH_BIOMETRY (new)"]
            KC["react-native-keychain\nRemember Me (identifier only)\n+ device-level App Lock enrollment (new, DEVICE-SCOPED)"]
        end

        subgraph Services["Service Layer"]
            AX["AxiosInstance\nBearer auth + x-fcm-token + x-facility-id\n+ isFromMobileRequest header\n+ isRevoked handling -> forced logout\n401 refresh + global loader"]
            CS["cognito.service.ts (legacy)\nrefreshSession / logout ONLY"]
            CUA["cognitoUserAuth.service.ts\nAWS SDK USER_AUTH flow\nEMAIL_OTP / SMS_OTP passwordless sign-in"]
            TM["token.manager.ts\nAsyncStorage R/W"]
            SOCK["ChatSocketService\nio({baseUrl}/chat)"]
            HOOK["useChatSocketLifecycle\nconnect on AppStack mount"]
            FN["foregroundNotifications.ts\nmessaging().onMessage -> notifee"]
            CN["chatNotification.ts\ndisplayChatNotification -> notifee"]
            PERM["permissions.ts\nnotifee.requestPermission() -- Android 13+ POST_NOTIFICATIONS"]
            ALG["AppLockGate (new)\nbackground-duration + foreground-idle timers\nfacility-configurable grace period"]
            BIOM["biometrics.service.ts (new)\nreact-native-keychain\nDEVICE-SCOPED enrollment, not account-scoped"]
            OTAH["useOtaUpdate / useOtaUpdateToast (new)\nexpo-updates"]
        end

        subgraph Firebase["Firebase SDK"]
            FCM_LIB["@react-native-firebase/messaging\nFCM token harvest\nforeground listener only"]
            CRASH["Crashlytics"]
            ANAL["Analytics"]
        end

        NOTIFEE["@notifee/react-native\n2 channels:\ndefault_notification_channel_id\nchat_notifications"]
    end

    subgraph Backend["Senior Living Backend\nhttps://api.sal.shashitech.com (PRODUCTION)"]
        REST["/api/* REST endpoints\n+ /api/consent-forms\n+ /api/dining-tray\n+ /api/ota/check /api/ota/log (new)"]
        CHAT_NS["Socket.io /chat namespace"]
    end

    subgraph AWS["AWS Cognito (us-west-1)"]
        COG["Pool from react-native-config\nUSER_AUTH (passwordless) via AWS SDK\n+ legacy refresh/logout via amazon-cognito-identity-js"]
    end

    subgraph FirebaseCloud["Firebase Cloud"]
        FCM_GW["Firebase Cloud Messaging"]
    end

    subgraph ExpoCloud["Expo / EAS Update"]
        EAS_GW["u.expo.dev\nEAS Update manifest + JS bundle host"]
    end

    IDX --> AT
    AT --> Nav
    AT --> Services
    AT --> State

    AUTH_S --> CUA
    AUTH_S --> CS
    CUA --> COG
    CS --> COG
    CS --> TM
    TM --> AS

    APP_S --> AX
    APP_S --> ALG
    ALG --> BIOM
    BIOM --> KC
    AX --> REST
    AX --> TM
    AX --> AS

    ROOT --> OTAH
    OTAH -->|"checkForUpdateAsync / fetchUpdateAsync"| EAS_GW
    OTAH -->|"fetchOtaEligibility / logOtaDelivery"| REST

    HOOK --> SOCK
    SOCK -->|"io /chat\nauth + x-facility-id"| CHAT_NS
    CHAT_NS -->|"chat:new chat:status"| SOCK
    SOCK -->|"chat:delivered chat:read"| CHAT_NS

    FN --> FCM_LIB
    FCM_LIB -->|onMessage foreground| NOTIFEE
    SOCK -->|chat:new (not in convo)| CN --> NOTIFEE
    PERM --> NOTIFEE

    Backend -->|trigger push| FCM_GW
    FCM_GW -->|push delivery| FCM_LIB
```

### 4.2 Authentication Flow (rewritten — passwordless)

```
SplashScreen.mount
  ensurePushToken() — unchanged
  checkForceUpdate() — unchanged
  checkAuth() — unchanged shape (reads tokens, refreshes via legacy cognito.service.ts)
  enterAppMaybeGated() (new, valid-session cold-start path) — see §4.2a

SignInScreen (rewritten)
  Two tabs: Phone (E.164) or Email — NO password field.
  "Remember Me" now persists only the identifier (phone or email) via Keychain,
    NOT a password (contrast with the prior review's plaintext-password Keychain risk —
    see §7.9, now resolved for the sign-in path specifically).
  handleContinue():
    initiateUserAuth(username)  [cognitoUserAuth.service.ts, AWS SDK]
      InitiateAuthCommand(AuthFlow: USER_AUTH, PREFERRED_CHALLENGE: EMAIL_OTP|SMS_OTP)
        -> SELECT_CHALLENGE? -> RespondToAuthChallengeCommand(SELECT_CHALLENGE)
        -> EMAIL_OTP | SMS_OTP challenge + Session
      result.status === 'SUCCESS' -> saveTokens() -> navigate APP
      result.status === 'OTP_CHALLENGE' -> navigate AUTH/MFA_VERIFY
        { username, session, contact: displayContact, challengeType }

MFAVerifyScreen (rewritten — OTP entry, not legacy TOTP/SMS MFA)
  OTPInput -> RespondToAuthChallengeCommand(EMAIL_OTP|SMS_OTP, Session)
  resendOtp(username) available
  onSuccess -> proceedAfterAuth() (new — resolveGateTarget(), see §4.2a) -> navigate APP or BIOMETRIC_ENROLL
```

**Screens removed entirely (with their tests):** `ChangePasswordScreen`, `ForgotPasswordScreen`, `ResetPasswordScreen` (AuthStack), `ChangeUserPassword` (ProfileScreen). Password-based login, Cognito's `newPasswordRequired` first-login flow, and the `bd01f4c`→`026ea88` custom-OTP forgot-password backend calls (`POST /api/auth/forgot-password`, `POST /api/auth/reset-password`, `GET /api/residents/check-temp-password`) are **no longer called from this app** — verified via `git grep` at `origin/master` for `CHANGE_PASSWORD`/`FORGOT_PASSWORD`/`RESET_PASSWORD`, zero hits outside history. **Open question for product/architect handoff:** the backend architecture doc (`architecture-senior_living_backend.md` v2.5) still describes `POST /api/auth/login-mfa-channel` + Cognito `EMAIL_OTP`/`SELECT_MFA_TYPE` as a *staff/admin* login-hardening feature; this app's independent move to a *resident-facing* passwordless `USER_AUTH` flow appears to be the same underlying Cognito capability applied to residents. Confirm with the architect/backend whether these are one coordinated identity project or two independently-landed features that happen to use the same Cognito primitives — this doc does not have enough evidence to assert either way.

Token expiry check and refresh-session mechanics are otherwise unchanged from the prior review (`token.manager.ts`).

### 4.2a Biometric App Lock Flow (new)

```
Entry point 1 — MFAVerifyScreen.proceedAfterAuth() (right after fresh OTP success):
  target = resolveGateTarget(dispatch)   [src/utils/auth/enterApp.ts]
    facilityData.residentBiometricEnabled fetched/read from Redux (fetch if absent)
    enrolled = isBiometricEnrolled()      -- DEVICE-level Keychain check, not account-scoped
    enrolled === true  -> target = 'unlock'   (device already enrolled by ANYONE)
    enrolled === false && residentBiometricEnabled !== true -> target = 'skip'
    enrolled === false && residentBiometricEnabled === true -> target = 'enroll'  (mandatory)
  target === 'enroll' -> navigate AUTH/BIOMETRIC_ENROLL (no skip; back/hardware-back sign out)
  target === 'unlock' -> setBiometricEnabled(true); setAppLockStatus('unlocked'); navigate APP
    (this fresh OTP success itself counts as identity proof -- no re-prompt right now)
  target === 'skip'   -> navigate APP directly (facility opted out of biometrics)

Entry point 2 — SplashScreen.enterAppMaybeGated() (cold start, already-valid session token):
  target = resolveGateTarget(store.dispatch)   -- same device-level check as above
  target === 'enroll' -> navigate AUTH/BIOMETRIC_ENROLL
  target === 'unlock' -> attempt a SILENT unlockWithBiometrics() over Splash itself
    (native Face ID/fingerprint prompt appears directly -- mirrors staffapp's
     enterAppThroughBiometricGate; no "tap to unlock" screen shown first)
    ok -> setAppLockStatus('unlocked') -> navigate APP
    reason === 'invalidated' -> signOutFromGate(dispatch)  -- forces full re-sign-in
    else -> setAppLockStatus('locked') -> navigate APP (AppLockGate's own overlay takes over)
  target === 'skip' -> navigate APP directly

Once inside APP (AppLockGate wrapping AppStack):
  background-duration timer OR foreground-idle timer fires -> setAppLockStatus('locked')
  Lock screen overlay shown (SessionLockedView) -- user must tap "Tap to Unlock"
    -> attemptUnlock() -> unlockWithBiometrics()
       ok -> 'unlocked'
       reason === 'invalidated' -> signOutFromGate(dispatch)
       else -> show error reason (retry, or "Back to Login")
```

Enrollment is never cleared on a normal sign-out (`LogUserOut`) — only `signOutFromGate` (the invalidated-unlock path) clears the Keychain item. See §3.12, §7.12.

### 4.2b OTA Update Flow (new)

```
RootNavigator mount + every AppState -> 'active' transition:
  useOtaUpdate.checkForOtaUpdate()
    Updates.checkForUpdateAsync()  -- polls the EAS Update channel baked into the native build
    not available -> return (retry next foreground)
    available -> fetchOtaEligibility({ platform })   [GET /api/ota/check, facility/JWT-scoped]
      not eligible or request fails -> return (fail closed, retry next foreground)
      eligible -> logOtaDelivery({ otaVersion, previousOtaVersion, deliveredAt })  [POST /api/ota/log]
        fails -> return (fail closed, retry next foreground)
      Updates.fetchUpdateAsync()  -- downloads the JS bundle, does NOT apply it yet
        fails/nothing new -> return
      showOtaUpdateBanner(openReloadDialog) + openReloadDialog()  -- dialog + persistent banner
        user taps Reload -> setItem(PENDING_OTA_RELOAD_UPDATE_ID) -> Updates.reloadAsync()
        user taps Cancel -> dialog closes, banner stays up (tap to reopen dialog)

Next cold start after a reload:
  useOtaUpdateToast() reads PENDING_OTA_RELOAD_UPDATE_ID, clears it immediately,
  and shows a one-time "Updated" success alert if Updates.updateId matches and
  it hasn't already shown for this update id (LAST_OTA_TOAST_SHOWN_UPDATE_ID).
```

### 4.3 x-facility-id Injection and Socket FACILITY_ID Race

Unchanged from the prior review — the race condition described in Design Gaps (HIGH) is still present at `origin/master` (no changes to `useChatSocketLifecycle.ts`'s facility-id read ordering were found in this range beyond a 3-line diff that does not touch the race).

### 4.4 Token Refresh (401) Flow

Unchanged in shape from the prior review (still no request-queueing across concurrent 401s — HIGH design gap, unresolved).

### 4.4a Admin-Revoked-Access Flow (new)

```
Any API response (2xx or non-2xx):
  isRevokedResponse(data) := !!data?.data?.isRevoked
  if isRevokedResponse:
    if isHandlingRevocation: return (dedup)
    isHandlingRevocation = true
    logout() [cognito.service.ts] + clearForLogout() [AsyncStorage]
    store.dispatch(clearDashboard()); store.dispatch(clearUser())
    isHandlingRevocation = false
    resetToSignIn()  -- navigation reset to AUTH/SIGNIN via navigationRef
    showAppAlert(AccessRevoked title/message/button)
```

This is independent of the 401-refresh path — it fires on any status code as long as the response body shape matches, including a 2xx that happens to carry `isRevoked: true` (e.g. a facility admin revoking a resident's app access mid-session). Source: `src/services/Api/index.tsx`, `src/utils/auth/resetToSignIn.ts`.

### 4.5 TV Pairing Flow

Unchanged from the prior review, including the documented `qrToken: ''` bug for short QR values (§8, unresolved).

### 4.6 Push Notification Flow

Unchanged from the prior review's foreground/background behavior. The `requestNotificationPermission()` helper (§3.7f) sits alongside this flow but was not traced to a specific call site in this pass — see the open item in §3.7f.

---

## 5. Data and State

### 5.1 AsyncStorage — Persisted Across Sessions

Prior key set unchanged (`ACCESS_TOKEN`, `ID_TOKEN`, `REFRESH_TOKEN`, `TOKEN_EXPIRY`, `COGNITO_USERNAME`, `FCM_TOKEN`, `FACILITY_ID`, `CurrentEnvironment`, `APP_BG_PRESET`, `FONT_SCALE_STORAGE_KEY`). `CurrentEnvironment` still has a second, hidden write path via `ChangeEnvironmentScreen` (§7.11). **New this pass:** `PENDING_OTA_RELOAD_UPDATE_ID` / `LAST_OTA_TOAST_SHOWN_UPDATE_ID` (OTA reload/toast one-shot markers, §3.13/§4.2b) and `BIOMETRIC_ENROLLED_WITH_BIOMETRY` (records which Keychain ACL flavor the device's app-lock item was created with, §3.12).

### 5.2 Keychain (react-native-keychain)

**Behavior change on the auth service (`com.shashigroup.sal.resident.auth`):** the prior review documented this service storing `phone + password` in plaintext for "Remember Me" (§7.9/§9 CRITICAL-adjacent finding). At `origin/master`, `SignInScreen`'s remember-me path now calls `saveCredentials(activeTab, username)` — i.e. it persists only the **identifier** (phone or email), not a password, consistent with the passwordless rewrite. `src/utils/authStorage.ts` changed 41 lines in this range; confirm at the file level that no password-shaped value is still written anywhere in that module before fully closing out the §7.9/§9 finding — this doc treats it as resolved for the sign-in flow based on the call-site evidence above, but did not re-read `authStorage.ts` in full for this pass.

The Session Keychain service (`com.shashigroup.sl.resident.session`, HIPAA idle timestamp) is unchanged; the bundle-prefix mismatch (`sal` vs `sl`) documented previously is still present. **New this pass:** a third Keychain service, `com.shashigroup.sal.resident.app_lock` (biometric App Lock device enrollment, §3.12) — it uses the `sal` prefix, joining the auth service in the same naming inconsistency already flagged for the session service; see §9.

### 5.3 Redux (In-Memory, Session-Scoped)

`user.profile` now also carries `requiresConsentSignature` (boolean, driving §3.7c). **New this pass:** `appLock` slice (`status`, `biometryType`, `isPrompting`, `suppressLockUntil`, `isBiometricEnabled` — §3.2, §3.12) and `otaBanner` slice (`visible` — §3.2, §3.13). No persistence layer (redux-persist or similar) exists for any slice — all Redux state, including `isBiometricEnabled`, resets to its initial value on every cold start; the durable signal for "should this device be gated" is the Keychain enrollment check (`isBiometricEnrolled()`), not Redux.

### 5.4 Backend Data (Not Client-Owned)

Consent forms (via `/api/consent-forms`) and dining-tray menu data (via `/api/dining-tray`, Menu2U-sourced) are fetched on demand with no client persistence beyond render state. **New this pass:** OTA eligibility/version data (via `/api/ota/check`) and OTA delivery-log records (via `/api/ota/log`) — both backend-owned, facility/platform-scoped, fetched/posted on demand with no client persistence beyond the one-shot AsyncStorage reload markers in §5.1.

### 5.5 Environment URLs

Unchanged from the prior review (`PRODUCTION`/`STAGING`/`PRE_PRODUCTION`/`LOCAL` in `src/utils/local.constants.ts`). The set of environments a resident can be pointed at now has a second selection surface: the hidden `ChangeEnvironmentScreen` restricts choices to `STAGING`/`PRE_PRODUCTION`/`PRODUCTION` (no `LOCAL` option there), writing the same `CurrentEnvironment` key. `currentEnv`'s own fallback logic changed this pass: it now first tries `APP_ENVIRONMENT` (from `react-native-config`, baked in at build time — see `envConfig.ts`) and only falls back to `ENVIRONMENT.PRODUCTION` if that's unset/invalid, rather than a single hardcoded default — see §7.14 for how this interacts with CI's OTA-channel branch switch.

---

## 6. External Dependencies

Unchanged table from the prior review with these additions/notes:

| Dependency | Purpose | Auth / Notes |
|---|---|---|
| AWS Cognito (`us-west-1`) — **direct SDK path** | Passwordless sign-in (`USER_AUTH`/OTP) | `@aws-sdk/client-cognito-identity-provider` talking directly to Cognito's public API from the device (not proxied through the backend); pool/client id resolved via `getActiveCognitoConfig()` (§3.5), region derived from the pool id prefix |
| `https://api.sal.shashitech.com` | All backend REST + Socket.io | New endpoints consumed by this app: consent-form PDF fetch/sign (cross-check `/api/consent-forms` in the backend doc), dining-tray menu (cross-check `/api/dining-tray/menu`), transportation booking response shape (`pickupTime`/`distance`/409 `conflicts`), **and (new this pass) `GET /api/ota/check` / `POST /api/ota/log`** (OTA eligibility + delivery logging, §3.13, §6) |
| **`https://u.expo.dev/<projectId>` (new)** | **EAS Update manifest + JS-bundle host** | Polled by `expo-updates` (`Updates.checkForUpdateAsync`/`fetchUpdateAsync`) once the native build's baked-in channel (`configure-ota-channel.js`, §3.13) matches a published EAS Update branch. No Shashi-backend auth on this call — the eligibility gate that decides whether to *apply* what's fetched lives in `/api/ota/check`, not here. EAS project id `ad0c5b46-08a2-41d4-a2ec-50902f7e29ff`. |
| Google Places API | Address autocomplete for transportation booking | Still **hardcoded** in `src/utils/local.constants.ts` — unchanged, see §7.4 |

All other rows (Firebase Cloud Messaging, Crashlytics, Analytics, GitLab CI, SonarQube, Semgrep/Snyk/Trivy/Gitleaks, Figma, Google Play, TestFlight) are unchanged from the prior review, except: the `security-scan` CI job gained `allow_failure: true` at the v3.0-pass `origin/master` (previously blocking on `master`), and `build_android`/`build_ios` no longer trigger on the `staging` branch (only `master`/`development`/`ota-release`/`production` now, per this pass's `.gitlab-ci.yml` diff) — see §8.

---

## 7. Security and Multi-tenancy

### 7.1 Multi-tenancy

Unchanged — `x-facility-id` injection mechanics, conditional header, and lack of resident-side multi-facility switching are all as previously documented. The new OTA eligibility/log endpoints (§3.13, §6) ride the same `x-facility-id` header as every other request.

### 7.2 Auth Token Storage

Unchanged — access/ID/refresh tokens remain in unencrypted AsyncStorage; `react-native-keychain` is now used for remember-me identifiers, the HIPAA session timestamp, **and (new)** biometric App Lock device enrollment — still not for auth tokens themselves. Still below the security bar for a healthcare app — see §9.

### 7.3 CognitoStorage Adapter Bug (Broken Session Persistence)

Not re-verified in this pass (not touched by the diffstat for this range) — carried forward as previously documented, still believed present.

### 7.4 Hardcoded Google Places API Key

Unchanged — still hardcoded in `src/utils/local.constants.ts`.

### 7.5 `trustAllCerts` — RESOLVED for both PDF viewers

Both `TestResult/PDFViewerScreen` (lab reports) and the Acknowledgment/Consent screen now render with `trustAllCerts={false}` at `origin/master` (`git grep trustAllCerts origin/master -- src/` shows two live `false` sites and one commented-out `false` in the disabled DiningScreen calendar block). This **resolves** the HIGH-severity MITM Design Gap from the prior review (previously `true` on the lab-report viewer). No remaining `trustAllCerts={true}` site was found in `src/` at `origin/master`.

### 7.6 Security Scan Gap on `master`-only CI — unchanged, now also non-blocking

`security-scan` is still `only: - master` (not `production`, the deployed branch) — unchanged. The job gained `allow_failure: true` at the v3.0-pass `origin/master`, so even a `master`-branch run that finds Snyk/Semgrep/Trivy/Gitleaks issues no longer fails the pipeline. Net effect: the security-scan gate is weaker than the prior review, not stronger. See §8 (still BLOCKER-class).

### 7.7 No Background FCM Handler

Unchanged — confirmed absent at `origin/master` (`git show origin/master:index.js | grep setBackgroundMessageHandler` → no match). Still BLOCKER-class.

### 7.8 No FCM Token Refresh Listener

Not re-verified in this pass (not touched by the diffstat) — carried forward as previously documented.

### 7.9 Remember-Me Password Storage — resolved for sign-in, scope narrowed

The sign-in "Remember Me" path no longer stores a password (§5.2) — it stores only the phone/email identifier. This resolves the specific finding as it applied to `SignInScreen`. `src/utils/authStorage.ts` itself was not fully re-read in this pass to confirm no other call site still persists a password-shaped value; treat as resolved-pending-confirmation.

### 7.10 Production Logs in Non-DEV Paths

Not re-verified file-by-file in this pass. `cognito.service.ts` (previously flagged as the highest-risk unguarded-log site, 14+ sites logging `username`/auth-state with no `__DEV__` guard) lost 319 lines net in the v3.0-pass range as the password/MFA logic was stripped out to just `refreshSession`/`logout` — the specific line numbers previously cited are stale and most of that code no longer exists. This needs a fresh line-by-line pass in the next review rather than being carried forward as-is.

### 7.11 Hardcoded Developer-Environment-Switch Password (Design Gap)

`src/screens/App/ProfileScreen/ProfileScreen/index.tsx`: `const DEV_PASSWORD = '321@ShashiCare';`. Seven taps on the ProfileScreen logo within a 3-second window (`TAP_THRESHOLD = 7`, `TAP_WINDOW_MS = 3000`) opens a `DeveloperPasswordAlert` prompt; entering this hardcoded string navigates to `ChangeEnvironmentScreen`, which lets the user point the entire app (REST base URL + Cognito pool) at `STAGING`, `PRE_PRODUCTION`, or `PRODUCTION` and restarts the app (`react-native-restart`) to apply it. The password is extractable by static/binary inspection of any distributed APK/IPA — the same class of issue as the previously-documented hardcoded Google Places API key (§7.4), but with a materially higher-impact outcome (redirecting a resident's app, including its auth backend, to a different environment). See §8.

### 7.12 Biometric App Lock — Device-Scoped, Not Account-Scoped (new — cross-references staffapp G14)

The new biometric App Lock (§3.12) enrolls **the device**, not the signed-in account: `src/services/Biometrics/biometrics.service.ts:50-53,79-81,131-132` documents this in its own comments, and `src/utils/auth/enterApp.ts:13-24` confirms the implementation deliberately "mirrors staffapp's `resolveGateTarget` exactly." Concretely: once any resident or family member enrolls a device, `resolveGateTarget()` returns `'unlock'` (never `'enroll'`) for every subsequent sign-in on that device, regardless of account — and a normal sign-out does not clear the enrollment (only the invalidated-unlock path does, §3.12). This is the identical characteristic already tracked as **Design Gap G14** in `senior_living_staffapp`'s architecture doc. The risk profile differs somewhat from staffapp (this app's devices are typically resident- or family-owned rather than facility-shared staff devices, which lowers but does not eliminate the exposure — a shared family tablet or a resident's device handed to a new family member without a factory reset would still inherit a prior user's unlock). **This needs the same product/security decision as staffapp G14 — recommend tracking both as one cross-app decision, not two independent ones**, given the near-identical implementation. See §8 (new Design Gap row) for the tracked entry.

### 7.13 Release Signing Falls Back to Debug Keystore Locally (new)

`android/app/build.gradle` now computes `hasReleaseKeystore = System.getenv("KEYSTORE_FILE_TMP") || rootProject.file("keystore.properties").exists()` and sets `signingConfig hasReleaseKeystore ? signingConfigs.release : signingConfigs.debug` for the `release` build type. In CI, the `sign` job always decodes `KEYSTORE_FILE_TMP` before invoking Gradle, so the shipped signing path is unaffected. A local `npm run apk` / `npm run aab` (both new scripts this pass) run without either env var or `keystore.properties` present would silently produce a **debug-signed** "release" build instead of failing — see §8/§9.

### 7.14 CI OTA-Channel / `APP_ENVIRONMENT` Branch Switch Does Not Explicitly Handle `production` (new)

`.gitlab-ci.yml`'s `build_android`/`build_ios` jobs both contain an identical `case "$CI_COMMIT_REF_NAME"` block mapping `development`→(`OTA_CHANNEL=staging`, `APP_ENVIRONMENT=STAGING`), `master`→(`preprod`, `PRE_PRODUCTION`), `ota-release`→(`production`, `PRODUCTION`), and `staging`→(`OTA_CHANNEL=""`, `APP_ENVIRONMENT=PRODUCTION`) — but **`production` (the branch this repo is actually deployed from, per its own git-branch convention) is not one of the named cases** and falls into the wildcard (`OTA_CHANNEL=""`, `APP_ENVIRONMENT=""`, i.e. neither is configured for that build). This happens to still produce a correct production build today only because two independent fallbacks coincidentally agree: the committed native manifest/plist default the OTA channel to `"production"` (§3.13), and `src/utils/local.constants.ts`'s `currentEnv` falls back to `ENVIRONMENT.PRODUCTION` when `APP_ENVIRONMENT` is unset (§5.5). A future edit to either fallback, without a matching edit to the other or to this `case` statement, would silently break production channel/environment targeting with no CI failure to surface it. See §8.

---

## 8. Design Gaps

Carried forward from the prior review where unresolved, updated where this range changed the picture, and new items added. Evaluated the same way — functional gaps, correctness defects, or security holes, not general debt (see §9).

| Severity | Issue | Evidence (file:line) | Status | Recommended fix |
|---|---|---|---|---|
| **BLOCKER** | No background FCM message handler. | `index.js` (handler still absent at `origin/master`) | **Unresolved, unchanged.** | Register `messaging().setBackgroundMessageHandler(...)` in `index.js`. |
| **BLOCKER** | Security scan runs only on `master`, not `production` (the deployed branch); now `allow_failure: true` on top of that, so `master` runs no longer block either. | `.gitlab-ci.yml` (`security-scan` job: `only: - master`, `allow_failure: true`) | **Regressed** (v3.0 pass) — the branch-coverage gap is unchanged and the gate is now non-blocking even where it does run; unresolved this pass too. | Add `production` to `only:`, remove `allow_failure: true`, or otherwise ensure Gitleaks/Snyk/Semgrep/Trivy findings block a release. |
| **HIGH** | Socket FACILITY_ID race — chat socket may connect before `HomeScreen` has written `FACILITY_ID`. | `src/hooks/useChatSocketLifecycle.ts` | **Unresolved, unchanged.** | As previously recommended — delay connect or reconnect on facility-id arrival; verify backend rejects sockets without `x-facility-id`. |
| **HIGH** | TV pairing QR bug — short QR values send `{ qrToken: '' }`. | `src/screens/App/ProfileScreen/LoginTvApp/QRScannerScreen.tsx` | **Unresolved** (not touched in this range's diffstat). | As previously recommended. |
| ~~**HIGH**~~ **RESOLVED** | `trustAllCerts={true}` on lab-report PDF viewer. | `src/screens/App/HealthScreen/TestResult/PDFViewerScreen/index.tsx` | **Resolved** — now `false`, and the Acknowledgment/Consent viewer is also `false`. | None — verify no regression in future reviews. |
| **HIGH** | 401 refresh does not queue concurrent in-flight requests. | `src/services/Api/index.tsx` | **Unresolved, unchanged.** | As previously recommended (promise queue). |
| **HIGH** | HIPAA inactivity auto-logoff inert — `IDLE_TIMEOUT` still 7 days. | `src/utils/local.constants.ts` | **Unresolved, unchanged** (not touched in this range). | As previously recommended. |
| **HIGH** | Hardcoded developer-environment-switch password (`'321@ShashiCare'`) ships in the production binary and, once entered, redirects the whole app (REST + Cognito pool) to any of staging/pre-prod/production. | `src/screens/App/ProfileScreen/ProfileScreen/index.tsx` (`DEV_PASSWORD` constant) | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | Remove the hardcoded password; gate behind a build-time flag (strip from release builds) or a server-issued, rotatable secret. |
| **MEDIUM (new this pass)** | Biometric App Lock enrollment is device-level, not account-level (by explicit in-code design) — once any resident/family member enrolls a device, any subsequent signed-in user on that device unlocks the same way; a normal sign-out does not clear the enrollment. This is the same characteristic already tracked as `senior_living_staffapp`'s Design Gap **G14**. | `src/services/Biometrics/biometrics.service.ts:50-53,79-81,131-132`, `src/utils/auth/enterApp.ts:13-24` | **New at `origin/master`.** | Confirm with product/security whether this is acceptable for resident/family-owned devices (lower-risk than staffapp's shared facility devices, but not risk-free — e.g. a shared family tablet); track alongside staffapp G14 as one cross-app decision, not two independent ones. See §7.12. |
| **MEDIUM (new this pass)** | CI's OTA-channel/`APP_ENVIRONMENT` branch switch in `build_android`/`build_ios` has no explicit `production` case — it falls through to the no-op wildcard, relying on the native manifest's hardcoded `"production"` default and the JS `APP_ENVIRONMENT` fallback to coincidentally stay correct. | `.gitlab-ci.yml` (`build_android`/`build_ios` OTA-channel `case` blocks) | **New at `origin/master`.** | Add an explicit `production)` case mirroring the others (`OTA_CHANNEL=production`, `APP_ENVIRONMENT=PRODUCTION`), rather than relying on two independent defaults staying in sync. See §7.14. |
| **LOW (new this pass)** | Release Android builds fall back to debug signing when no release keystore is present (`hasReleaseKeystore` check). Not a risk in CI today (keystore always supplied by the `sign` job) but a local `npm run apk`/`aab` run without it could silently produce a debug-signed artifact rather than failing. | `android/app/build.gradle` (`hasReleaseKeystore`) | **New at `origin/master`.** | Consider failing loudly instead of silently falling back, at least for the `npm run apk`/`aab` scripts intended for release-shaped local testing. See §7.13. |
| **MEDIUM** | `AcknowledgmentScreen`'s consent-mode navigation passes the **resident's** `id` as `consentFormId` (`consentFormId: currentProfile.id` / `response.data.id`), not a distinct consent-form identifier — naming vs. actual value mismatch. | `src/screens/App/HomeScreen/index.tsx` (consent gate blocks) | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | Confirm against the backend `/api/consent-forms` contract whether this is intentional (resident id used to resolve "their" pending form) or a bug that happens to work because there's currently one pending form per resident. |
| **MEDIUM** | Message and Profile bottom tabs still commented out. | `src/navigation/appstack/BottomTabNavigator.tsx` | **Unresolved, unchanged.** | Confirm with product whether this is final. |
| **MEDIUM (expanded this pass)** | New feature areas ship without dedicated tests, including several security- or money/schedule-relevant flows: passwordless sign-in, consent-form signing, admin-revoked-access handling, transportation pickup-time/conflict handling, the Menu2U dining rewrite, the dev-environment switcher, **and (new this pass) most of the biometric App Lock UI layer (`AppLockGate`, `BiometricEnrollScreen`, `enterApp.ts`/`resolveGateTarget`) and the entire OTA layer (`useOtaUpdate`, `useOtaUpdateToast`, `GlobalOtaBanner`, `otaBanner` slice/helper, `configure-ota-channel.js`)** — though this pass did add real unit tests for the biometrics *service* and *slice* layers (`biometrics.service.test.ts`, `appLock.slice.test.ts`), unlike every other new surface to date. | Absence of `__tests__/` for the above; test file count dropped 56→52 net at the v3.0-pass baseline | **Expanded — carried-forward gap now covers more surface area, with partial improvement in one sub-area.** | Prioritize tests for `resolveGateTarget`/`AppLockGate` (security-relevant gate logic) and the OTA reload/toast flow (state-corruption risk on a failed reload) next, matching staffapp's `enterApp.test.ts` coverage. |
| **MEDIUM** | `DiningScreen` ships with daily-specials, diet-plan cards, and the booking calendar **commented out in place** ("TEMP: … kept for restore") rather than feature-flagged or removed, alongside the live Menu2U tray-card rewrite. | `src/screens/App/ServicesScreen/DiningScreen/index.tsx` | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | Confirm with product whether the reduced dining feature set is intentional/temporary; if temporary, track restoration; if permanent, delete the dead code rather than leaving it commented (see §9). |
| **LOW** | `@react-native-firebase/auth` installed, unused. | `package.json` | **Unresolved, unchanged.** | Remove. |

---

## 9. Technical Debt

Carried forward from the prior review where unresolved/not re-verified, updated where changed, and new items added.

| Severity | Issue | Evidence (file:line) | Status | Recommended fix |
|---|---|---|---|---|
| **CRITICAL** | `CognitoStorage.getItem()` always returns `null` (sync wrapper around an async call). | `src/services/Auth/cognito.storage.ts` | **Not re-verified this pass** — file not touched in the diffstat for this range; treat as still present. | As previously recommended. |
| **HIGH** | Auth tokens stored in unencrypted AsyncStorage, not Keychain. | `src/services/Auth/token.manager.ts` | **Unresolved, unchanged.** | Migrate to `react-native-keychain`. |
| ~~**HIGH**~~ **Narrowed** | Remember-me password storage — resolved for the sign-in flow (now identifier-only); `authStorage.ts` not fully re-audited for other call sites. | `src/utils/authStorage.ts` | **Improved, not fully closed.** | Confirm no remaining password-shaped writes anywhere in `authStorage.ts`. |
| **HIGH** | Unguarded `console.log` calls leaking auth state/PII in production builds. | Multiple files; `cognito.service.ts`'s specific line numbers are now stale (file shrank by 319 lines at the v3.0-pass baseline) | **Needs re-audit** — the previously-cited `cognito.service.ts` sites largely no longer exist post-rewrite; `SplashScreen`/`TransportationScreen` sites not re-verified this pass, nor is the new biometric/OTA code. | Re-audit for unguarded logs across the passwordless auth path, and the new biometric App Lock / OTA code specifically, since both are new and security- or reliability-relevant. |
| **HIGH** | `IDLE_TIMEOUT` = 7 days neutralizes HIPAA auto-logoff. | `src/utils/local.constants.ts` | **Unresolved, unchanged.** | As previously recommended. |
| **HIGH** | Two parallel Cognito client stacks now ship simultaneously: `amazon-cognito-identity-js` (legacy — `refreshSession`/`logout` only) and `@aws-sdk/client-cognito-identity-provider` + `@smithy/*` (new — passwordless sign-in), the latter requiring a substantial Hermes polyfill layer to function at all. This roughly doubles the auth-stack surface area and bundle weight, and ties correctness to polyfill behavior rather than native RN/Hermes APIs. | `src/services/Auth/cognito.service.ts` + `cognitoUserAuth.service.ts`; `index.js` polyfill block; `package.json` | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | Plan a follow-up to retire the legacy `amazon-cognito-identity-js` path (refresh/logout) onto the same AWS-SDK client used for sign-in, removing one of the two stacks once the new flow is proven in production. |
| **HIGH (new this pass)** | The app now layers **a third build/tooling stack** on top of the already-documented dual-Cognito-client debt above: the original bare-RN CLI toolchain, the Fastlane/GitLab CI native-build pipeline, and now an Expo bare-workflow layer added purely to enable OTA (`babel-preset-expo`, `expo/metro-config`, `expo-updates`, `expo-asset`, `expo-modules-core`). Each layer patches Metro/Babel independently — e.g. `metro.config.js` now manually calls `patchMetroSourceMapStringForPackedMaps()` because Xcode debug builds start Metro directly (`react-native-xcode.sh`), bypassing Expo CLI's own auto-patching for that same fix. | `metro.config.js`, `babel.config.js`, `package.json` | **New at `origin/master`.** | No action needed today — document the tooling-layer ordering (Expo config wraps RN config, not the reverse) so a future RN or Expo SDK bump doesn't silently break Metro/Babel resolution. |
| **MEDIUM (new this pass)** | `react-native-worklets` was **downgraded** 0.10.2→0.8.3 in the same range that bumped `react-native-reanimated` (4.3.0→4.3.1) and RN itself (0.84.1→0.85.3) — likely a compatibility pin for the Expo/reanimated combination, but reverses this doc's own prior "bumped from 0.8.1" note. | `package.json` | **New at `origin/master`.** | Confirm the 0.8.3 pin is an intentional Expo/reanimated compatibility requirement before any future "helpful" bump back toward 0.10.x. |
| **MEDIUM (new this pass)** | `package.json` `engines.node` was relaxed from `>= 22.11.0` to `>= 20.0.0`, while GitLab CI still explicitly installs and uses Node 22 (`nvm install 22`) — the two are no longer in sync, so a contributor building locally on Node 20 could pass local checks that CI's Node 22 environment might not reproduce, or vice versa. | `package.json` (`engines`), `.gitlab-ci.yml` (`nvm install 22`) | **New at `origin/master`.** | Confirm the intended minimum Node version and align both files. |
| **MEDIUM** | `text-encoding` is imported directly by `index.js` but is **not declared** in `package.json` dependencies — it resolves today only because it's present as a transitive dependency in `package-lock.json`. A future lockfile regeneration or upstream dependency change could silently remove it and break app boot on Hermes. | `index.js` (`import { TextEncoder, TextDecoder } from 'text-encoding'`); absent from `package.json`; present in `package-lock.json` | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | Add `text-encoding` as an explicit direct dependency in `package.json`. |
| **MEDIUM** | Keychain bundle-prefix mismatch (`sal` vs `sl`) — now spans **three** services, not two. | `src/utils/authStorage.ts`; `src/utils/auth/sessionTimestamp.ts`; **`src/services/Biometrics/biometrics.service.ts` (new — `com.shashigroup.sal.resident.app_lock`, §5.2)** | **Unresolved, and this pass's new biometric-enrollment Keychain service perpetuates the same naming inconsistency rather than fixing it.** | Unify on one bundle-id prefix (`sl`, matching the actual app bundle id `com.shashigroup.sl.resident`) across all Keychain service names, including the new App Lock one, before this compounds further. |
| **MEDIUM (expanded)** | New feature areas across this range ship without tests (see §8 for the full list, including this pass's OTA layer and most of the App Lock UI layer). | Absence of `__tests__/` | **Expanded.** | See §8. |
| **LOW (new this pass)** | Odd, non-Shashi-owned scoped package name (`@agnoliaarisian7180/string-argv`) appears in `package.json`'s `resolutions`/`overrides`, remapped to the legitimate `npm:string-argv@0.3.2` — almost certainly a supply-chain-mitigation override for a package pulled in transitively via the new `eas-cli`/Expo tooling. | `package.json` (`resolutions`, `overrides`) | **New at `origin/master`.** | Leave in place; document why (probable dependency-confusion/typosquat mitigation in the `eas-cli` dependency tree) so a future engineer doesn't remove it as "dead config." Periodically re-verify it still resolves to the intended package. |
| **LOW** | `react-native-otp-verify` + `@eabdullazyanov/react-native-sms-user-consent` both installed for SMS OTP. | `package.json` | **Unresolved, unchanged.** | Audit/consolidate. |
| **MEDIUM** | `moment` and `date-fns` both bundled. | `package.json` | **Unresolved, unchanged.** | Audit and remove `moment`. |
| **MEDIUM** | `jspdf` in dependencies with no obvious mobile use case. | `package.json` | **Not re-verified this pass.** | Audit usage. |
| **MEDIUM** | `serializableCheck: false` in Redux store. | `src/store/store.ts` | **Unresolved, unchanged.** | Enable and fix surfaced issues. |
| **MEDIUM** | No root `ErrorBoundary`. | Absent from `src/` tree | **Unresolved, unchanged.** | Add one. |
| **MEDIUM** | AppStack still on `@react-navigation/stack` (JS thread) vs native-stack elsewhere. | `src/navigation/appstack/index.tsx` | **Unresolved, unchanged this pass.** | Migrate post-launch. |
| **MEDIUM** | Axios interceptor reads `CURRENTENVIRONMENT` from AsyncStorage on every request. | `src/services/Api/index.tsx` | **Unresolved, unchanged in this specific mechanic.** | Cache in a module-level variable. |
| **MEDIUM** | No data-fetching cache layer (no RTK Query/React Query). | Absence in `package.json` | **Unresolved, unchanged.** | Introduce post-launch. |
| **MEDIUM** | `DiningScreen` carries large commented-out blocks (daily specials, diet-plan cards, booking calendar) alongside the live Menu2U rewrite rather than removing or flagging the dead code. | `src/screens/App/ServicesScreen/DiningScreen/index.tsx` | **Unresolved, unchanged this pass** (introduced at the v3.0-pass baseline). | See §8 — resolve product intent, then either restore behind a flag or delete. |
| **LOW** | Two image-picker libraries installed (`react-native-image-crop-picker` + `react-native-image-picker`). | `package.json` | **Unresolved, unchanged.** | Consolidate. |
| **HIGH** | Discord webhook URL with token hardcoded in `android/fastlane/Fastfile`. | `android/fastlane/Fastfile:10` | **Not re-verified this pass** (Fastfile not in this range's diffstat). | As previously recommended; note it is exactly the class of secret the now-`allow_failure: true` security scan (§8) would catch but still not fail on. |
| **MEDIUM** | `staging` branch previously ran full build/sign/deploy with no security gate. | `.gitlab-ci.yml` | **Partially improved** at the v3.0-pass baseline (`staging` removed from `build_android`/`build_ios`'s `only:` list); this pass's addition of `development`/`ota-release` to that same `only:` list (for the new OTA-publish step) does not reopen the gap, since `security-scan` remains `master`-only regardless. | Re-verify whether `production` build/deploy is still gated only by a `master`-only, `allow_failure: true` scan. |
| **LOW** | `react-native-qrcode-svg` installed, zero imports. | `package.json` | **Unresolved, unchanged.** | Remove. |
| **LOW** | Developer LAN IPs committed to `local.constants.ts`. | `src/utils/local.constants.ts` | **Not re-verified this pass.** | As previously recommended. |
| **LOW** | `clearForLogout()` preserves a never-written `'DEVICE_ID'` no-op key. | `src/utils/Localstorage/index.tsx` | **Not re-verified this pass.** | As previously recommended. |
| **LOW** | Duplicate `StatusBar` in `App.tsx`. | `App.tsx` | **Not re-verified this pass.** | As previously recommended. |
| **LOW** | No test coverage for HomeScreen, chat flow, service API modules, Axios 401 interceptor. | Absence of `__tests__/` | **Unresolved, unchanged** (still the largest coverage gap by runtime risk). | As previously recommended. |
