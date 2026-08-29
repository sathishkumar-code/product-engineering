# Architecture: senior_living_staffapp

> **Doc status:** Re-verified against **`origin/master` HEAD `35b3cc82`**, 2026-08-21 — the branch this repo actually deploys from (unchanged branch note, see below). Previous baseline was `origin/master` HEAD `4aa3849` (2026-07-11). This pass covers **165 commits / 46 merges** (`4aa3849..origin/master`, `218 files changed, +24501/-6237`). Worked thematically by feature branch, not commit-by-commit; every merge-branch name in the range is accounted for below except pure CI/version-bump/Xcode-project-file noise. Sections are verified against `origin/master` file:line unless noted; several items are flagged **not fully traced this pass** given the volume of change — treat those as open follow-ups, not confirmed facts.
>
> **Headline changes this pass:** an **Expo + EAS Update integration** (bare-workflow OTA JS-bundle pushes layered onto the existing native Android/iOS projects — **not** a migrated-to-managed-Expo rewrite); a **device-level biometric App Lock** (Face ID/Touch ID/passcode gate, backend-configurable per facility); **sign-in now supports phone OR email** (segmented toggle) plus a **Cognito user-pool migration fallback** (`migratorSignIn`) and **3-channel MFA** (TOTP/SMS/Email OTP); a large **chat module expansion** (message forward, PHI-aware Keychain-backed drafts, conversation pin/mark-read/leave-group, per-recipient message-info sheet, a rewritten video player with orientation support); a **new Transport tab** and a **reworked/re-gated Scan Documents tab** on the Skilled Nursing bottom-tab bar (previously just Home/Messages + MySchedule-XOR-PendingSign); a **new "Secure Call" summary/transcript/approval workflow**; and **React Query + NetInfo** newly introduced as a second data-fetching layer alongside the existing hand-rolled service pattern. A parallel **offline chat sync (local cache + outbox queue, Realm-backed)** was built on `feat/offline-message-sync` but **never merged into `master`** — verified via `git merge-base --is-ancestor`; treat chat as still fully online-only in the shipped app (see §3.8).
>
> **Branch note (unchanged):** this repo's CI/CD deploys from `master`, not `production`. The local checkout's `production` branch (HEAD `6307ac0`) remains stale relative to `master` by hundreds of commits — do not read `production` as current.
>
> **Delta — 2026-08-27, against `origin/master` HEAD `d4d0bc5c`** (6 commits / 0 merges since the `35b3cc82` checkpoint above; **`.gitlab-ci.yml` + `package.json` only — no application source changed**). This is not pure CI/version-bump noise: it operationalizes the previously-documented-but-not-yet-wired Expo + EAS Update OTA pipeline (§3.1b) into the actual per-branch CI build/deploy flow. **Branch model:** `development` and `ota-release` now participate in CI job gates (`designqa_check`, `lint_reactnative`, `quality_check`, `sonar_reactnative`, `security-scan`, `build_android`, `build_ios`) — both branches already existed on `origin` (traced back to the repo's `2026-02-23` initial commit via `git merge-base --is-ancestor` / `git rev-list --count`, ~960 commits each) but were previously unreferenced by any CI job; this diff wires them in, it does not create them. **Per-branch OTA-channel → environment mapping** (new `case "$CI_COMMIT_REF_NAME"` block added to both `build_android` and `build_ios`): `development` → OTA channel `staging` / `APP_ENVIRONMENT=STAGING`; `master` → `preprod` / `PRE_PRODUCTION`; `ota-release` → `production` / `PRODUCTION`; `staging` → no OTA channel / `APP_ENVIRONMENT=PRODUCTION` (env resolved, no OTA publish); any other branch → neither. **OTA publish now happens automatically, post-native-build**: after `bundle exec fastlane build` succeeds, the job installs `eas-cli` and runs `npm run update:staging|preprod|production` (update message from `$CI_MERGE_REQUEST_TITLE`, falling back to `$CI_COMMIT_MESSAGE`, now passed through via a new `--message` flag on the `package.json` scripts) — previously a manual/scripts-only capability (§2, §3.1b prior pass), now a real automatic pipeline step, which raises the stakes on **Design Gap G13** (no OTA-rollback runbook, §8) without resolving it. Unrelated in this range: iOS `pod install --repo-update` was split into its own `pod repo update` step run before `pod install` (in both `build_ios` and `deploy_ios`), inside the existing 3-attempt retry loop — a CI reliability tweak, not a behavior change.
> Related: [../architecture-senior-living-product.md](../architecture-senior-living-product.md) | [./adr/](./adr/)

---

## 1. Purpose

`senior_living_staffapp` is a React Native mobile application delivering role-specific operational task management to authenticated staff members at senior-living facilities. It is the staff-facing operational surface for the Senior Living platform; the resident-facing equivalents are `senior_living_reactnative` (assisted living) and `senior_living_skillednursing_resident` (skilled nursing).

The app still resolves each staff member to one of **three app flows** (`src/utils/featureAccess.ts:resolveAppFlowFromProfile()`, unchanged mapping logic):

- **LEGACY** — the original designation task queue (Transport, Housekeeping/Maintenance, Salon, Massage, Private Trainer, Activities Director), hosted inside a 2-tab bottom navigator (Home + Messages).
- **MIGRATED** — the **Skilled Nursing** experience: a bottom-tab navigator that has grown from 2–3 tabs to **up to 5**: Home, Messages, **Transport** (new — role-gated, real-time via Socket.io), **Documents/Scan** (new — feature-flagged + role-gated, the reworked digital-document-scan flow), and either **My Schedule** or **Pending Sign** depending on whether the user is in the `Doctor` role group — plus the same deep stack of resident-care screens as before (now also including a **Secure Call summary/transcript/approval** workflow — see §3.2e).
- **CHAT_HOME** — a chat-only home (no bottom tabs) for designations that only have messaging access.

**Concrete `master` responsibilities, updated this pass (all code-verified):**

- Authenticate staff via AWS Cognito, now accepting **either phone number (E.164) or email**, chosen via a segmented control on the sign-in screen (`isEmailLogin` state), each with independent Remember-Me credential storage. `signIn()` takes an explicit `identifierType: 'phone' | 'email'` (`src/services/Auth/cognito.service.ts:127-131`). Source: `src/screens/Auth/SignInScreen/index.tsx:294-405`.
- Fall back to a **backend Cognito user-pool migrator** (`migratorSignIn()`) when a normal Cognito password check fails — POSTs to `resolveUserpoolMigratorUrl()`, and either returns fresh tokens directly (non-MFA users) or a `MIGRATED_NO_TOKENS` result that routes into a dedicated post-migration MFA screen. Source: `src/services/Auth/cognito.service.ts:26-69,157-230` (see §3.7, §4.1).
- Support **3 MFA delivery channels** — `SOFTWARE_TOKEN_MFA` (TOTP), `SMS_MFA`, and a new `EMAIL_OTP` — with an explicit `loginMfaChannel()` pin call before `InitiateAuth` so Cognito picks the right channel on the first attempt. Source: `cognito.service.ts:95-116`.
- Gate the whole authenticated app behind a **device-level biometric App Lock** (Face ID / Touch ID / device passcode) once a device has enrolled — a facility-configurable feature (`residencyConfig.biometricEnabled`, `residencyConfig.lockScreenTimeout`), re-locking on backgrounding and on foreground inactivity. This is new and sits **before** any PHI screen mounts (enroll/unlock resolved during the Splash → app-entry gate, not lazily). Source: `src/services/Biometrics/biometrics.service.ts`, `src/components/App/AppLock/`, `src/utils/auth/enterApp.ts:29-70` (see §3.1a).
- Resolve the app flow and gate access from the staff profile's `staffDirectoryRoles`; unchanged mechanism, now joined by two additional gated surfaces on the Skilled Nursing tab bar (Transport, Documents) driven by `TRANSPORT_TAB_ROLE_GROUPS` / `DOCUMENT_UPLOAD_ROLE_GROUPS` (`featureAccess.ts:92-102`, both currently `['Case Manager', 'Social Worker', 'Director of Nursing']`).
- Gate every API call and socket connection behind a selected facility using the `x-facility-id` header — **unchanged this pass** (confirmed: `src/services/Auth/cognito.storage.ts` — the file backing multi-tenancy-adjacent auth storage — was not touched in this diff range at all).
- Display a real-time, designation-specific task queue driven by Socket.io events (LEGACY flow) — unchanged. The new Skilled Nursing **Transport tab** gets its own real-time Socket.io updates (`9dbc8c62` "Implement real-time updates in TransportScreen using socket.io") — see §3.2c.
- Provide real-time 1:1 and group chat, substantially expanded this pass: **message forwarding** (to multiple chats, with attachments), **PHI-aware Keychain-backed drafts** (debounced per-conversation), **conversation pin/mark-as-read/leave-group** (long-press action menu, pin state synced across devices and capped via facility config), a **per-recipient message-info sheet** (read/delivered/sent breakdown), and a **rewritten video player** (orientation support via `react-native-orientation-locker`, swipe-to-dismiss, inline attachment-download progress in `ChatBubble`). See §3.8, §3.13.
- Render the whole app's date/time UI in the facility's configured IANA timezone — **unchanged this pass**, see §3.2d (kept from the prior pass, still accurate).
- Place outbound resident phone calls via Twilio Programmable Voice — unchanged mechanism, now feeding into the new Secure Call summary/approval workflow when the call is recorded (§3.2e).
- Record/upload care-conference audio — unchanged.
- Present a **Pending Sign** tab for `DOCTOR`-role users — extended this pass with local PDF caching and facility-date support, and now integrated with the reworked **Scan Documents** flow (a scanned document can be routed to a selected physician for signature). Source: `72ce28e3`, `60ea4245`, `cd077557`.
- Receive FCM push, including two new push types this pass: pending-signature notifications (`pendingSignatureNotification.ts`) and resident-document-signed notifications (`residentDocumentSignedNotification.ts`).
- Embed a Freshworks customer-support widget — unchanged.
- Let a staff member switch the active backend environment at runtime — extended with a new **`LocalConnectScreen`** (LAN IP / team-preset picker for local dev, `src/screens/App/LocalConnectScreen/index.tsx`), alongside the existing `ChangeEnvironmentScreen`.
- Enforce a 24-hour inactivity timeout — unchanged, now layered *underneath* the new biometric App Lock (two independent, differently-scoped timeout mechanisms — see §3.1a).
- **New: Expo + EAS Update OTA** — the app can receive JS-bundle-only updates without an app-store review cycle, gated to per-environment channels (staging/preprod/production), surfaced via an in-app banner/progress UI. See §3.1b.
- **New: Secure Call summary/transcript/approval** — a staff member can review an AI-transcribed summary of a recorded resident call, edit it, and approve/sign off on it before it's shared. See §3.2e.

**Out of scope (not present in code):** TV control, admin CRUD, address autocomplete (Google Places key/library present but no autocomplete UI wired — unchanged, see Technical Debt). **Offline chat message caching/sending** was built but did not land in `master` this pass — see §3.8.

---

## 2. Tech Stack

| Layer | Technology | Version |
|---|---|---|
| Framework | React Native | **0.85.3** (was 0.84.0) |
| UI Runtime | React | 19.2.3 (unchanged) |
| Language | TypeScript | **6.0.3** (was 5.8.3) |
| **Native-module host (new)** | **Expo SDK (bare-workflow modules only, not managed workflow)** | **`expo` ~56.0.12** + `expo-asset`, `expo-build-properties`, `expo-constants`, `expo-updates` |
| **OTA updates (new)** | **EAS Update** (`eas-cli`, channel-per-environment: staging/preprod/production) | see `eas.json`, `app.json` |
| Navigation | React Navigation native-stack + bottom-tabs | 7.x (unchanged) |
| State management | useReducer + split Context (hand-rolled) | — (unchanged) |
| **Server-state cache (new, coexists with the above)** | **`@tanstack/react-query`** + `@react-native-community/netinfo` (online-manager) | 5.101.4 / 12.0.1 |
| HTTP client | Axios | 1.13.2 (unchanged) |
| Authentication | amazon-cognito-identity-js | 6.3.16 (unchanged) |
| Config / secrets injection | react-native-config | 1.6.1 (unchanged) |
| Phone number parsing/UI | react-native-international-phone-number + libphonenumber-js | unchanged |
| Real-time | socket.io-client | 4.8.3 (unchanged) |
| Voice calling | @twilio/voice-react-native-sdk | 1.7.0 (unchanged) |
| Audio recording | react-native-nitro-sound (+ nitro-modules) + native Android service / iOS module | unchanged |
| Push notifications | @react-native-firebase/app + /messaging | 24.0.0 (unchanged) |
| Notification display | @notifee/react-native | 9.1.8 (unchanged) |
| Notification / UI sound (in-app) | react-native-sound | 0.13.0 (unchanged) |
| SMS OTP autofill | @eabdullazyanov/react-native-sms-user-consent, react-native-otp-verify | unchanged |
| PDF view / print / generate / sign | react-native-pdf, react-native-print, react-native-blob-util, jspdf, pdf-lib | unchanged |
| Media | react-native-image-picker, react-native-image-crop-picker, @react-native-documents/picker, react-native-video, react-native-create-thumbnail, @d11/react-native-fast-image | unchanged |
| **List virtualization (new)** | **@shopify/flash-list** | 2.3.2 — adoption breadth not fully traced this pass |
| **Network state (new)** | **@react-native-community/netinfo** | 12.0.1 — backs React Query's online-manager (`src/services/queryClient.ts`); **not** wired into any chat-screen offline UI (that lives only on the unmerged offline-sync branch, see §3.8) |
| In-app browser / webview | react-native-inappbrowser-reborn, react-native-webview | unchanged |
| Calendar / date | react-native-calendars, @react-native-community/datetimepicker, date-fns 4.x, @date-fns/tz | unchanged |
| Timezone / Intl | react-native-localize, @formatjs/intl-* (iOS-only Hermes polyfill) | unchanged |
| **Video orientation (new)** | **react-native-orientation-locker** | 1.7.0 — used by the rewritten chat `VideoPlayer` |
| **Icon set (new)** | **react-native-vector-icons** | 10.3.0 — coexists with the pre-existing hand-authored SVG icon set (`AppIcons.ts`) |
| **Text encoding polyfill (new)** | **text-encoding** | 0.7.0 — `src/polyfills/textEncoding.ts`, likely an Expo/Hermes compatibility shim (not fully traced this pass) |
| Emoji | rn-emoji-keyboard | unchanged |
| Permissions | react-native-permissions | unchanged |
| Device / update / restart | react-native-device-info, react-native-restart | unchanged |
| Credential / token storage | react-native-keychain | 10.0.0 — now also backs biometric App Lock enrollment (device-level) and PHI-bearing chat drafts (per-conversation), in addition to the existing auth-token and Remember-Me use |
| Key-value storage | @react-native-async-storage/async-storage | 2.2.0 (unchanged) |
| JWT decode | jwt-decode | 4.0.0 (unchanged) |
| Dropdown | react-native-element-dropdown | unchanged |
| Gestures | react-native-gesture-handler | unchanged |
| SVG rendering | react-native-svg (+ svg-transformer) | 15.15.4 (was 15.15.3) |
| Safe area | react-native-safe-area-context | ~5.7.0 (was 5.5.2) |
| Screen optimization | react-native-screens | unchanged |
| Keyboard avoidance | react-native-avoid-softinput | unchanged |
| Clipboard | @react-native-clipboard/clipboard | unchanged |
| Testing | Jest 29 + react-test-renderer | unchanged — but see §3.15, real unit-test coverage appeared for the first time this pass (Biometrics, AppLock, `enterApp`) |
| Linting | ESLint 8 + typescript-eslint | unchanged |
| Build / CI | GitLab CI + Fastlane to Google Play internal / TestFlight; **`eas update` OTA scripts are now invoked automatically in CI per-branch** (2026-08-27 delta — see §3.1b/§3.14: `development`→staging, `master`→preprod, `ota-release`→production), not just available as manual npm scripts | — |
| App identity | Android `applicationId` / iOS bundle `com.shashigroup.sl.staffapp`; **versionName 1.3.29 / versionCode 34** (was 1.3.21 / 26) | `android/app/build.gradle`, `ios/SALStaff.xcodeproj/project.pbxproj` |
| Backend (prod) | `https://api.sal.shashitech.com` | unchanged |
| Cognito pool | injected per-environment via `react-native-config` | unchanged mechanism |

> **Expo posture — read carefully.** This is **not** a migration to Expo's managed workflow. `android/` and `ios/` native project directories are still present and still hand-maintained (native modules: `AudioRecorderService.kt`, `AppBadgeModule.kt`, `PendingRecordingModule.swift`, `ChatDeliveryNotifier.swift` all still exist and were touched this pass for other reasons). What landed is: (1) a handful of Expo library modules (`expo`, `expo-asset`, `expo-build-properties`, `expo-constants`) added as native-module dependencies inside the existing bare app, largely to satisfy `expo-updates`' runtime requirements; (2) **`expo-updates` + EAS Update** for OTA JS-bundle pushes, configured via `app.json` (`runtimeVersion`, `updates.url` pointing at `u.expo.dev/<projectId>`, `checkAutomatically: "NEVER"` — i.e. **not** silent/automatic, the app must explicitly check) and `eas.json` (per-environment build/update channels). `docs/ota-channel-setup.md` documents channel setup; **no OTA-rollback runbook entry exists yet** — see Design Gap G13.

> **Build/run note (updated):** native modules are still patched on install (`postinstall` → `node scripts/apply-patches.js`, now **401 lines**, up from a much smaller file — grew substantially, presumably to reconcile Expo-module patches with the existing native customizations; not diffed line-for-line this pass). `react-native-config` still requires Metro `--reset-cache` after `.env` changes. Node >= 22.11.0 (unchanged).

> **`acorn-typescript` / `acorn-walk` / `write-good`** remain in `dependencies` rather than `devDependencies` — unchanged debt, see TD13.

---

## 3. Key Components

### 3.1 Navigation Stacks (5 stacks, unchanged count; tab content inside them grew)

Flow selection (LEGACY / MIGRATED / CHAT_HOME) is still resolved at sign-in/splash via `appRoute.helper.ts:appFlowRoute()` — mechanism unchanged. What changed is what sits **in front of** flow resolution (the biometric gate, §3.1a) and what the MIGRATED flow's tab bar now contains.

| Stack | File | Screens |
|---|---|---|
| RootStack | `src/navigation/rootstack/index.tsx` | Splash (nested), Auth (nested), App (nested) |
| SplashStack | `src/navigation/splashstack/index.tsx` | SplashScreen |
| AuthStack | `src/navigation/authstack/index.tsx` | SIGNIN, CHANGE_PASSWORD, FORGOT_PASSWORD, RESET_PASSWORD, MFA_VERIFY, ACCESS_DENIED, TERMS, **BIOMETRIC_ENROLL** (new), **BIOMETRIC_UNLOCK** (new) |
| AppStack (LEGACY entry) | `src/navigation/appstack/index.tsx` | Unchanged screen set (HOME 2-tab nav, CHAT_HOME, SELECT_FACILITY, SKILLED_NURSING_HOME, SETTINGS, etc. + shared chat sub-graph), now also wrapped by the AppLock gate at the App-tree root (see §3.10) |
| SkilledNursingAppStack (MIGRATED entry) | `src/navigation/skilledNursingAppStack/index.tsx` | `ROOT_TABS` (bottom-tab navigator — **now up to 5 tabs**, see below) plus the existing ~50 stack screens **plus** `LOCAL_CONNECT` (new), `CALL_SUMMARY` / `EDIT_CALL_SUMMARY` (new), a reworked `ADD_SCAN_DOCUMENT` flow, `FORWARD_MESSAGE` (new), `IN_APP_WEBVIEW` (new, generic), and `TRANSPORT` (new stack screens under the new Transport tab) |

**`ROOT_TABS` tab set — grew from 3 effective tabs to up to 5.** `SkilledNursingBottomTabNavigator` (`skilledNursingAppStack/index.tsx:265-475`) renders, in order:

1. **Home**, **Messages** — always shown (unchanged).
2. **Transport** (new) — shown only when `canAccessTransportTab` (`isInRoleGroups(profile, TRANSPORT_TAB_ROLE_GROUPS)`, currently `Case Manager` / `Social Worker` / `Director of Nursing` — `featureAccess.ts:92-102`). Renders `TransportScreen`, a new file (`skilledNursingAppStack/index.tsx:328-356`).
3. **Documents / "Scan"** (new) — shown only when **both** `SHOW_SCAN_DOCUMENTS_TAB` (a hardcoded `true` constant, `local.constants.ts:43` — not a real remote feature flag, see TD15) **and** `canAccessDocumentsTab` (`DOCUMENT_UPLOAD_ROLE_GROUPS`, same three role groups). Renders the reworked `AddScanDocumentScreen` (`skilledNursingAppStack/index.tsx:358-386`).
4. `REPORTS` — **still commented out** (`skilledNursingAppStack/index.tsx:391-419`, unchanged from the prior pass) — dead code, see Design Gap G11 (still open).
5. **My Schedule** XOR **Pending Sign** — same doctor-group gating as before (`skilledNursingAppStack/index.tsx:421-475`).

So the effective tab bar for a Case Manager / Social Worker / Director of Nursing (non-doctor) user is now **Home / Messages / Transport / Documents / My Schedule** (5 tabs) — a meaningful UI-density increase over the prior pass's 3-tab bar. A doctor-group user without transport/document access still sees **Home / Messages / Pending Sign** as before.

The AppStack still wraps its navigator in `<SessionGuard>` and calls `useChatSocketLifecycle()` — unchanged. New this pass: the App-tree root (§3.10) now also mounts `AppLockGate`, which renders `SessionLockedView` / `BiometricsUnavailableView` in place of the app content whenever the device-level lock is engaged, **before** any nested navigator's screens are reachable.

### 3.1a Biometric App Lock (new module)

A device-level (not per-account) app-lock layer sitting between "signed in" and "app content visible," backed by `src/services/Biometrics/biometrics.service.ts` and `src/components/App/AppLock/` (`index.tsx` — the gate component, `BiometricPromptView`, `BiometricUnavailableDialog`, `BiometricsUnavailableView`, `EnrollPromptView`, `SessionLockedView`), plus a Redux-pattern slice `src/store/features/appLock/appLock.slice.ts`.

- **Enrollment is device-level, not account-level** — `isBiometricEnrolled()` checks for a Keychain-stored device lock (service `com.shashigroup.sl.staffapp.app_lock`, a fixed label `device-biometric-lock` + random nonce, not tied to who is signed in). The code's own comments state this explicitly: once *any* staff member enrolls the device, subsequent sign-ins on that device skip straight to "unlock" rather than "enroll," regardless of account. **This is an intentional design choice per the in-code documentation, but it is a PHI-exposure consideration on shared/handed-off devices worth confirming with product/security** — flagged as Design Gap G14, not asserted as a bug.
- **Facility-configurable** — `residencyConfig.biometricEnabled` (default-safe: treated as enabled/locking whenever this hasn't loaded yet) and `residencyConfig.lockScreenTimeout` (seconds; falls back to a hardcoded `APP_LOCK_GRACE_MS` if the facility hasn't set one) gate whether the lock applies at all and how long a foreground-idle window is tolerated before re-locking. Source: `src/components/App/AppLock/index.tsx:44-60`.
- **Gate resolution** happens in `src/utils/auth/enterApp.ts:resolveGateTarget()`, called from the Splash cold-start path: already-enrolled devices resolve to `'unlock'` with **zero network calls** (no latency added for returning users); not-yet-enrolled devices fetch the profile + residency config to decide `'enroll'` vs `'skip'` (facility opted out), failing safe to `'enroll'` if the config fetch errors.
- **Two independent re-lock triggers**: (a) app backgrounded for longer than the grace period, tracked via `AppState`; (b) foreground inactivity — a touch-driven `PanResponder` resets an idle timer, paused while the keyboard is visible (typing generates no touch events on the underlying view tree). Both funnel into the same `'locked'` Redux status. Care is taken to distinguish "backgrounded because our own Face ID system sheet popped up" from genuine backgrounding (`wasPromptingWhenBackgroundedRef`), avoiding a spurious re-lock immediately after a successful unlock.
- New auth-adjacent screens: `BiometricEnrollScreen`, `BiometricUnlockScreen` (`src/screens/Auth/`).
- **This is the first meaningfully unit-tested new surface in the app** — `biometrics.service.test.ts` (363 lines), `appLock.slice.test.ts` (63 lines), and `enterApp.test.ts` (364 lines) all shipped alongside the feature. See §3.15.

### 3.1b Expo + EAS Update / OTA (new module)

Covered in detail in §2's Expo posture callout. UI surface: `GlobalOtaBanner` (prompts the user that an update is available) and `GlobalOtaUpdateProgress` (download/apply progress), driven by `useOtaUpdate.ts` / `useOtaUpdateToast.ts`. `checkAutomatically: "NEVER"` in `app.json` means the app must explicitly call into `expo-updates` to check — this is **not** a silent forced-update mechanism; the exact trigger point (app foreground? a specific screen mount?) was **not fully traced this pass** — flag for follow-up. `scripts/configure-ota-channel.js` and `docs/ota-channel-setup.md` document the per-environment channel setup (`staging`/`preprod`/`production`, matching `eas.json`'s build profiles). **No OTA-rollback runbook entry exists** — see Design Gap G13.

**CI wiring (2026-08-27 delta — see header):** the OTA publish step above is now invoked automatically by `.gitlab-ci.yml`'s `build_android`/`build_ios` jobs immediately after `bundle exec fastlane build`, for three branches — `development` (channel `staging`), `master` (channel `preprod`), and `ota-release` (channel `production`) — via `npm run update:{staging,preprod,production}` after installing `eas-cli`. The `staging` branch resolves `APP_ENVIRONMENT=PRODUCTION` but publishes no OTA channel. This closes the gap between the OTA mechanism existing (prior pass) and it actually firing in CI; **it does not add a rollback path** — G13 remains open and is now more load-bearing since a push to `ota-release` reaches the `production` OTA channel automatically.

### 3.2 Auth & Shared App Screens

| Screen | Path | Function |
|---|---|---|
| SplashScreen | `src/screens/SplashScreen/index.tsx` | Force-update gate → hydrate Cognito memory storage → FCM token request → inactivity-expiry check → **biometric gate resolution (new)** → session resume/refresh → flow routing |
| SignInScreen | `src/screens/Auth/SignInScreen/index.tsx` | **Phone (E.164) OR email** sign-in, chosen via a `SegmentedControl` toggle (new); Remember-Me now keyed per identifier type; routes to CHANGE_PASSWORD / MFA_VERIFY / app |
| ChangePasswordScreen | `src/screens/Auth/ChangePasswordScreen/index.tsx` | Unchanged (first-login forced password change) |
| ForgotPasswordScreen | `src/screens/Auth/ForgotPasswordScreen/index.tsx` | Unchanged (OTP reset + resend-credentials path); file grew (243 lines changed) — not line-diffed this pass beyond confirming the two existing flows still exist |
| ResetPasswordScreen | `src/screens/Auth/ResetPasswordScreen/index.tsx` | Unchanged |
| MFAVerifyScreen | `src/screens/Auth/MFAVerifyScreen/index.tsx` | Now handles **3 MFA channels** (TOTP / SMS / Email OTP) via `resolveMfaTypeFromChallenge()`, and is reused for the post-migration MFA step (see §3.7, §4.1) |
| TermsWebViewScreen | `src/screens/Auth/TermsWebViewScreen/index.tsx` | Unchanged |
| **BiometricEnrollScreen** (new) | `src/screens/Auth/BiometricEnrollScreen/index.tsx` | First-time device biometric enrollment UI — see §3.1a |
| **BiometricUnlockScreen** (new) | `src/screens/Auth/BiometricUnlockScreen/index.tsx` | Re-auth prompt for an already-enrolled device — see §3.1a |
| AccessDeniedScreen | `src/screens/Auth/AccessDeniedScreen/index.tsx` | Unchanged |
| HomeScreen (LEGACY) | `src/screens/App/HomeScreen/index.tsx` | Unchanged (LEGACY designation dispatcher + socket host) |
| ChatHomeScreen | `src/screens/App/ChatHomeScreen/index.tsx` | Unchanged |
| CallScreen | `src/screens/App/CallScreen/index.tsx` | Twilio call UI, extended to feed a completed/recorded call into the new Secure Call summary flow (§3.2e) |
| SelectFacilityScreen | `src/screens/App/SelectFacilityScreen/index.tsx` | Unchanged |
| SettingsScreen | `src/screens/App/SettingsScreen/index.tsx` | Minor addition (7 lines) — not traced further |
| **LocalConnectScreen** (new) | `src/screens/App/LocalConnectScreen/index.tsx` | Dev-only LAN-IP / team-preset environment picker, pairs with `ChangeEnvironmentScreen`; persists to the same `CurrentEnvironment` AsyncStorage key and restarts via `react-native-restart` |
| ChangeStaffPasswordScreen / DisplayScreen / ThemeSelectionScreen / RescheduleSalonAppointmentScreen | — | Unchanged |

### 3.2b Skilled Nursing Screens (updated — new Transport, Documents/Scan rework, Secure Call)

All under `src/screens/SkilledNursing/App/`. Bold = new or materially reworked since the `4aa3849` baseline:

- **Tabs:** HomeScreen (extended with physician + admission-type resident filters, see below), MessagesScreen, **TransportScreen** (new tab), MyScheduleScreen, PendingSignScreen (extended — local PDF caching, facility-date support), **AddScanDocumentScreen** (reworked — 735-line diff, now supports selecting a physician to route a scanned document to for signature, tying into Pending Sign). `ReportsScreen` remains a dead/unreachable tab (G11, unchanged).
- **Resident filtering (new):** `HomeScreen` (274-line diff) gained a **physician filter** and an **admission-type filter** for case-manager-type roles (`60d2287d`, `19b9513d`) — not fully traced for exact UI/API shape this pass.
- **Residents & care team:** unchanged screen set; `ResidentDetailsScreen` (164-line diff) and `AdvancedCareDirectiveScreen` (289-line diff) both changed — not fully re-verified line-for-line this pass.
- **Care conferences:** unchanged.
- **Digital signature module:** unchanged screen set (PendingSignScreen / PendingSignDetailScreen / SignedPdfPreviewScreen), extended with local caching + facility-date support (see above) and integration with the Scan Documents rework.
- **Secure Call (new module):** **CallSummaryScreen** (new, 368 lines) and **EditCallSummaryScreen** (new, 122 lines) — review, edit, and approve/sign-off a transcribed summary of a recorded resident call before it's shared. Backed by `callSummary.slice.ts` (new, 210 lines) — see §3.2e.
- **Environment switching:** ChangeEnvironmentScreen unchanged; joined by the new LocalConnectScreen (§3.2 above).
- **Services:** unchanged screen set.
- **Transportation:** `TransportationRequestScreen` (1023-line diff — the largest single-file change this pass besides `ConversationScreen`) continues to support create + edit; `TransportationScreen` (189-line diff) and the new `TransportScreen`/Transport tab stack coexist — the exact relationship between the legacy `TransportationScreen` (existing, case-manager-facing) and the new tab-level `TransportScreen` was **not fully disambiguated this pass** — flag as an open question for the next pass (possible duplication or a deliberate list-vs-detail split).
- **Chat (shared with LEGACY AppStack):** the existing screens plus **ForwardMessageScreen** (new, 454 lines — pick 1+ destination conversations/attachments to forward a message to, grouped into "Recent Chats" and "People" sections). `ConversationScreen` (1392-line diff) and `GroupConversationScreen` (986-line diff) are the two largest touched files in the whole range — see §3.8 for the behavioral summary; not line-diffed exhaustively.
- **Generic webview (new):** `InAppWebViewScreen` (101 lines) — a generic in-app webview screen; relationship to the existing Freshworks-embedded webview and Terms webview not traced (may be a shared replacement or a separate use case).
- **Calls:** CallScreen (Twilio) unchanged, now also the entry point into Secure Call review.
- **Customer support:** unchanged (Freshworks widget in ProfileScreen).

### 3.2c Transportation — now two surfaces (LEGACY driver view + new Skilled Nursing tab)

The LEGACY `TransportDriverView` (driver-facing task queue) is unchanged in structure. Separately, the **new Skilled Nursing Transport tab** (`TransportScreen`, role-gated to Case Manager / Social Worker / Director of Nursing) gives non-driver staff a case-manager-facing view of transportation requests with **live Socket.io updates** (`9dbc8c62`). `TransportationRequestScreen` continues to handle create/edit with validation for pickup time, address coordinates, and 12-hour formatting (unchanged from the prior pass's description); `TransportRideCard` (new, 243-line component) and `BottomSheetSelect` enhancements (`a14c878b`) support both surfaces. Both flows remain timezone-aware via `src/utils/timezone.ts` (unchanged, §3.2d).

### 3.2d Timezone Layer (unchanged this pass)

No material change verified in this range to `src/utils/timezone.ts` or the iOS Hermes `Intl` polyfill approach (`src/polyfills/intl.ios.ts` — 7-line diff, not re-verified line-for-line but the file's core design — documented in the prior pass — was not superseded). Full detail retained from the prior pass: facility `timeZone` from `GET /api/config/residency-details`, applied via `applyAppTimeZoneFromConfig()`, formatted through `@date-fns/tz`; absolute-instant math must not route through these helpers.

### 3.2e Secure Call Summary / Transcript Review & Approval (new module)

A workflow for staff to review an (externally-produced) transcribed summary of a recorded resident call, edit it, and approve/sign off before it is shared with the resident or family. State lives in `src/store/features/callSummary/callSummary.slice.ts` (new): holds the selected call's PHI (recordings, callee name, initiating staff `cName`, an editable summary draft, a `shareWithResident` flag, who last edited vs. who approved, and two status fields — a transcription-pipeline `status` (`PENDING`/`IN_REVIEW`/`SUMMARIZED`/`COMPLETED`, used only as a legacy-call fallback) and a staff-facing `approvalStatus` (`PENDING`/`APPROVED`) with `approvedBy`/`approvedAt`. Screens: `CallSummaryScreen` (review/approve), `EditCallSummaryScreen` (edit the summary text). **The transcription/summarization pipeline itself is external to this app** — no client-side transcription code was found; the app only displays and approves pipeline output. **The upstream transcription service/vendor was not identified this pass** — flag as an external-dependency gap for the architecture/PM follow-up (see §6, External Dependencies).

### 3.3 LEGACY Designation Views (unchanged this pass)

No structural change verified in this range beyond what the prior pass already noted (UI/text tweaks to `ActivitiesDirectorView` and `TransportDriverView`). Table retained from the prior pass:

| View | File | Roles served |
|---|---|---|
| TransportDriverView | `DesignationViews/TransportDriverView.tsx` | TRANSPORT_DRIVER |
| HousekeepingStaffView | `DesignationViews/HousekeepingStaffView.tsx` | HOUSEKEEPING_STAFF, MAINTENANCE_STAFF (via `isMaintenance` prop) |
| ActivitiesDirectorView | `DesignationViews/ActivitiesDirectorView.tsx` | ACTIVITIES_DIRECTOR |
| SalonStylistView | `DesignationViews/SalonStylistView.tsx` | SALON_STYLIST |
| MassageTherapistView | `DesignationViews/MassageTherapistView.tsx` | MASSAGE_THERAPIST |
| PrivateTrainerView | `DesignationViews/PrivateTrainerView.tsx` | PRIVATE_TRAINER |
| DefaultDesignationView | `DesignationViews/DefaultDesignationView.tsx` | Any LEGACY-flow designation not matched above |

`DietitianView.tsx` still exists but is not wired in (unchanged, TD12). `isDoctorDesignation()` / `DOCTOR = 'Physician'` label mapping unchanged from the prior pass.

### 3.4 Store / State Management

Still the hand-rolled `useReducer` + split Context pattern (`src/store/index.tsx`) — **no migration to Redux, but React Query (§2, §3.5) is now a second, coexisting state layer for server data.** New slices this pass:

| Feature slice | File | State shape |
|---|---|---|
| **appLock** (new) | `src/store/features/appLock/appLock.slice.ts` | Biometric App Lock status (`'locked'`/`'unlocked'`/etc.) + `suppressLockUntil` — see §3.1a |
| **callSummary** (new) | `src/store/features/callSummary/callSummary.slice.ts` | Secure Call summary review/approval state — see §3.2e |
| **otaBanner** (new) | `src/store/features/otaBanner/otaBanner.slice.ts` + `.helper.ts` | OTA update-available banner visibility — see §3.1b |
| **pendingSignature** (new) | `src/store/features/pendingSignature/pendingSignature.slice.ts` | Backs the Pending Sign tab's unread-style badge (`usePendingSignatureBadge` hook) |
| **residentDocumentSignal** (new) | `src/store/features/residentDocumentSignal/residentDocumentSignal.slice.ts` + `.helper.ts` | Cross-screen signal for resident-document-signed events (feeds `residentDocumentSignedNotification.ts`) |
| globalQuestionAlert | `src/store/features/globalQuestionAlert/globalQuestionAlert.slice.ts` | Minor addition (a new helper file, 13-line diff) — confirmation dialog, unchanged shape |

Unchanged slices from the prior pass: `facility`, `user`, `globalAlert`, `config` (still carries `timeZone`; now presumably also `biometricEnabled`/`lockScreenTimeout` per §3.1a, not independently re-verified against the slice file since `config.slice.ts` itself wasn't in the diff — the new fields likely arrive via the existing `ResidencyConfigData` type, which **was** touched, see `type.ts` 272-line diff), `unread`.

### 3.4a App-Flow Resolution (`src/utils/featureAccess.ts`)

Core mapping logic unchanged (`ROLE_GROUP_FLOW_MAP`, `isDesignationAllowedFromProfile()`, `isDoctorDesignation()`). New this pass: `DOCUMENT_UPLOAD_ROLE_GROUPS` and `TRANSPORT_TAB_ROLE_GROUPS` (both `['Case Manager', 'Social Worker', 'Director of Nursing']`, `featureAccess.ts:92-102`), consumed by the new Skilled Nursing tab gates (§3.1).

### 3.5 API Service Functions (~85+ functions, plus a new React Query layer)

Still centered on `src/services/App/index.tsx` (327-line diff this pass — grew further, exact new function count not re-tallied) plus the existing domain service files, all routing through the singleton `AxiosInstance`. **New this pass:** `src/utils/queries.ts` (new, 115 lines) — a **generic React Query method toolkit** (`useApiQuery`, `useApiInfiniteQuery`, `useApiMutation`, `useInvalidateQuery`, `useReadQueryData`) with an explicit doc comment stating feature-specific hooks that call real service functions live elsewhere (referenced as `src/hooks/useQueries.ts` in the file's own comment — **existence/location not independently confirmed this pass**, flag for follow-up) — and `src/services/queryClient.ts` (new — the `QueryClient` instance + NetInfo-backed online-manager + AppState-backed focus-manager, §2). **Open question for the next pass:** how broadly React Query has actually been adopted vs. the existing hand-rolled `App/index.tsx` pattern — both now coexist, and the extent of migration was not traced.

Every outbound request now carries a new header: `isFromMobileRequest: true` (`src/services/Api/index.tsx` request interceptor) — a new client-identification signal for the backend; worth confirming with the backend team what (if anything) currently branches on it.

Updated endpoint-adjacent additions this pass (representative, not exhaustive): forward-message send (backing `ForwardMessageScreen`), draft-adjacent endpoints are **not** server-backed (drafts are Keychain-only, §3.8), conversation pin/mark-read/leave-group actions, Secure Call summary fetch/edit/approve, Scan Document + physician-selection endpoints, Transport-tab read/real-time endpoints.

### 3.6 Axios Singleton (`src/services/Api/index.tsx`)

Materially refactored this pass:
- `baseURL` resolution moved out of the interceptor and into a new shared helper, `resolveApiBaseUrl()` (`src/utils/apiEnvironment.ts`, new file) — same effective behavior (AsyncStorage `CurrentEnvironment` override) but centralized, presumably so `LocalConnectScreen`/socket connections can reuse the same resolution logic.
- 401-handling no longer calls the old `logout()` directly — it now calls `forceLogoutFromInterceptor()` (new, `src/utils/auth/LogUserOut.ts`), part of a broader logout-flow centralization this pass (`LogUserOut.ts` — 113 new lines, `logoutGuard.ts` — 24 new lines guarding against duplicate/racing logout calls, `appDispatchRegistry.ts` / `currentUserRegistry.ts` — 12 lines each, imperative registries so non-React code — the interceptor, the biometric gate — can dispatch/read without prop-drilling).
- Everything else (loader counter, dev-log masking, `x-facility-id`/Authorization/pushToken headers) — unchanged, not re-verified line-for-line beyond the diff shown above (`src/services/Api/index.tsx`, 22-line diff, fully reviewed).

### 3.7 Auth Services

| File | Responsibility |
|---|---|
| `src/services/Auth/cognito.config.ts` | Unchanged (27-line diff — not re-verified beyond confirming no hardcoded credentials reappeared) |
| `src/services/Auth/cognito.storage.ts` | **Not touched this pass at all** — SA-2 Blocker (sync return before async in `getItem()`) is unchanged, still open |
| `src/services/Auth/cognito.service.ts` | **Materially reworked** (334-line diff). `signIn(username, password, identifierType)` now takes an explicit `'phone'|'email'` identifier type; pins the MFA delivery channel via `loginMfaChannel()` before `authenticateUser()`; on a Cognito auth failure, falls back once to `migratorSignIn()` (POSTs to a backend "userpool migrator" URL resolved via `resolveUserpoolMigratorUrl()`) — either resolves with tokens directly or retries natively once more after a `MIGRATED_NO_TOKENS` result, routing into MFA if the migrated user requires it. `MfaType` now includes `'EMAIL_OTP'` alongside `'SOFTWARE_TOKEN_MFA'`/`'SMS_MFA'`. **This implies an in-flight backend-side Cognito user-pool migration project** — worth an architecture/ADR-level look on the `senior_living_backend` side; out of this repo's scope but flagged for the architect/PM follow-up. |
| `src/services/Auth/auth.service.ts` | Unchanged (custom OTP reset + `sendCredentials` resend, both from the prior pass) |
| `src/services/Auth/token.manager.ts` | Unchanged |
| `src/utils/apiEnvironment.ts` (new) | `resolveApiBaseUrl()`, `resolveUserpoolMigratorUrl()` — centralizes per-environment URL resolution, consumed by both the Axios interceptor and the Cognito migrator fallback |
| `src/utils/auth/enterApp.ts` (new) | Post-gate app-entry orchestration: `resolveGateTarget()` (biometric enroll/unlock/skip decision) + `enterApp()` (the shared "resolve route → dispatch profile/facility → reportLogin → replace(APP)" block, now deduplicated across SignInScreen/MFAVerifyScreen/SplashScreen — previously triplicated) |
| `src/utils/auth/LogUserOut.ts`, `logoutGuard.ts`, `appDispatchRegistry.ts`, `currentUserRegistry.ts` (all new) | Centralized logout flow + imperative dispatch/user registries for use outside the React tree |
| `src/utils/profileActivity.ts`, `profileActivityQueue.ts` (both new) | Login/logout telemetry reporting (`reportLogin`), with an offline-tolerant retry queue (`enqueuePendingEvent`/`flushPendingEvents`, capped at 20 entries, per-staff-member scoped so a handed-off device doesn't leak the wrong person's telemetry) — this is the closest thing to "offline sync" that actually landed in `master`; it is telemetry-only, not chat messages (see §3.8) |
| `src/utils/phoneAuth.ts` | Unchanged |

### 3.8 Socket.io Usage (two independent connections — unchanged count, chat namespace behavior extended)

**(a) LEGACY designation socket** — unchanged from the prior pass.

**(b) Chat socket (`/chat` namespace)** — still the process-wide singleton `chatSocketService` (`src/services/ChatSocket/index.ts`, 57-line diff this pass — additive, not re-verified event-by-event beyond what's below), driven by `useChatSocketLifecycle()`. Connection/reconnect/`forceNew` behavior and the `chat:new/unread/status/reaction/group/deleted` event set are **unchanged from the prior pass**. What's new is entirely client-side UX built on top of the existing event/REST surface:

- **Message forward** — `ForwardMessageScreen` + `ForwardActionBar`, send a message to multiple destination conversations, with attachments, grouped into Recent Chats / People pickers.
- **Message drafts** — **Keychain-backed**, not AsyncStorage, because drafts can contain resident PHI via @-mentions (`src/utils/draftStorage.ts`: one Keychain entry holds a JSON map of all per-conversation drafts, with an in-memory write-through cache to avoid an async read/write race between leaving a screen and the list re-reading). `useMessageDraft.ts` debounces saves (2s) while typing and flushes immediately on screen-blur or app-background.
- **Conversation actions** — `ConversationActionMenu` (long-press): delete, **pin/unpin** (state synced across a user's devices, capped via a facility-config max-pins value — `00bfd96b`), **mark-as-read**, **leave-group**.
- **Message info** — `MessageInfoSheet` (new, 538 lines): per-recipient read/delivered/sent breakdown with tabs (ALL/READ/DELIVERED/SENT), handling both timestamped and legacy plain-cName `deliveredTo`/`readBy` shapes from the API/socket.
- **Video player rewrite** — a new `VideoPlayer` component (610 lines) replacing the prior in-`ChatMediaViewerScreen` video handling, adding orientation support (`react-native-orientation-locker`) and swipe-to-dismiss; `ChatBubble` gained inline video-attachment download-progress UI (a circular progress ring) and `MessageListItem` gained a typing indicator.
- **Optimistic send** — unchanged mechanism from the prior pass (`pendingOptimisticIdsRef`).

**What did NOT land: offline message sync.** A `feat/offline-message-sync` branch (Realm-backed local chat cache in `src/services/ChatDb/`, an offline send outbox in `src/services/ChatOutbox/`, an `OfflineBanner`/`OfflineHeaderIndicator` UI, a `useChatOutboxLifecycle` hook) was built (commits `d2d4310c`, `286fee2d`, `574a181f`, merged into the `pin-conversations-message-info` branch at `5c30ea95`) but **none of these commits are ancestors of `origin/master`** (verified via `git merge-base --is-ancestor <commit> origin/master` for all four — all report `NOT ancestor`), and none of `ChatDb/`, `ChatOutbox/`, `OfflineBanner/`, or a `realm` dependency exist in the `origin/master` tree (`git ls-tree` confirms). **Chat remains fully online-only in the shipped app.** If product/QA believe offline messaging shipped, this is the correction to make — it exists only on an unmerged branch.

### 3.9 Push Notification Wiring

Foreground/background/quit-launch handling, FCM token registration, and the native Android app-icon badge — **unchanged this pass**. New notification-type handlers:

| Layer | File | Behavior |
|---|---|---|
| **Pending-signature notification (new)** | `src/services/Notifications/pendingSignatureNotification.ts` (95 lines) | Push for a new/updated document awaiting a `DOCTOR`-group user's signature; likely feeds the `pendingSignature` slice's tab badge (§3.4) |
| **Resident-document-signed notification (new)** | `src/services/Notifications/residentDocumentSignedNotification.ts` (7 lines — thin) | Push when a routed document has been signed; feeds `residentDocumentSignal` slice |
| Foreground handler | `src/services/Notifications/foregroundNotifications.ts` (60-line diff) | Not re-verified line-for-line; core `notifee.displayNotification()` mechanism assumed unchanged |
| Everything else (token registration, badge module, quit-launch capture) | — | Unchanged from the prior pass |

### 3.9a Twilio Voice (unchanged mechanism, new downstream consumer)

`src/services/TwilioService.js` (59-line diff, not fully re-verified) — core `makeCall`/`mute`/`setSpeaker`/`disconnect` API assumed unchanged; a completed/recorded call now feeds the new Secure Call summary/approval flow (§3.2e) as a downstream consumer of the recording.

### 3.9b Audio Recording (unchanged)

No material change verified in this range.

### 3.10 Provider Nesting Order (`App.tsx`)

Updated — a new `AppLockGate` sits inside the authenticated App tree (exact nesting position — whether it wraps the whole `StoreProvider` tree or sits inside it — was **not precisely re-diffed** this pass; `App.tsx` had a 49-line diff). Structurally:

```
GestureHandlerRootView
  SafeAreaProvider
    ThemeProvider (initialMode="light" — dark mode still unreachable, see Design Gap G8)
      StoreProvider
        LoaderProvider
          [React Query provider — new, exact placement not confirmed]
          AppBody (StatusBar + RootNavigator, now gated by AppLockGate for the authenticated app tree — see §3.1a)
```

### 3.11 Pagination Pattern (unchanged mechanism, new list-virtualization dependency)

The five LEGACY designation views' pagination pattern (`usePaginatedList`, `PAGE_LIMIT = 10`) — unchanged. The chat Messages tab's paginated conversation list — unchanged from the prior pass's migration onto real server-side pagination. **New:** `@shopify/flash-list` was added as a dependency; adoption breadth (which lists were actually migrated onto it vs. still using `FlatList`) was **not traced this pass**.

### 3.12 Native Modules

| Module | Purpose |
|---|---|
| react-native-gesture-handler, react-native-screens, react-native-safe-area-context | Unchanged |
| react-native-keychain | Auth tokens, Remember-Me, inactivity timestamp — **plus (new)** biometric App Lock device enrollment and per-conversation chat drafts |
| react-native-config | Unchanged |
| **`expo` / `expo-asset` / `expo-build-properties` / `expo-constants` / `expo-updates` (new)** | Native-module host + OTA update runtime — see §2, §3.1b |
| react-native-svg (+ svg-transformer) | Unchanged |
| react-native-element-dropdown | Unchanged |
| react-native-international-phone-number / libphonenumber-js | Unchanged |
| react-native-otp-verify / @eabdullazyanov/react-native-sms-user-consent | Unchanged |
| @twilio/voice-react-native-sdk | Unchanged (now also a Secure Call input) |
| react-native-nitro-sound + native Android/iOS audio modules | Unchanged |
| react-native-pdf / react-native-print / react-native-blob-util / jspdf / pdf-lib | Unchanged |
| react-native-image-picker / react-native-image-crop-picker / @react-native-documents/picker / react-native-video / react-native-create-thumbnail / @d11/react-native-fast-image | Unchanged |
| react-native-webview | Unchanged (Terms, Freshworks) — **plus (new)** a generic `InAppWebViewScreen` |
| react-native-calendars / @react-native-community/datetimepicker / date-fns / @date-fns/tz | Unchanged |
| react-native-localize / @formatjs/intl-* | Unchanged |
| **react-native-orientation-locker (new)** | Chat `VideoPlayer` orientation support |
| **react-native-vector-icons (new)** | Icon set added alongside the existing hand-authored SVG set |
| **text-encoding (new)** | Polyfill, `src/polyfills/textEncoding.ts` — likely an Expo/Hermes compatibility shim, not fully traced |
| rn-emoji-keyboard | Unchanged |
| react-native-permissions | Unchanged — **plus (new)** `NSFaceIDUsageDescription` (iOS biometrics) |
| react-native-device-info / react-native-inappbrowser-reborn | Unchanged |
| react-native-restart | Unchanged — now also used by `LocalConnectScreen` |
| react-native-avoid-softinput | Unchanged |
| react-native-sound | Unchanged |
| **@react-native-community/netinfo (new)** | Backs React Query's online-manager only — not chat offline UI (§3.8) |
| **@shopify/flash-list (new)** | List virtualization — adoption breadth not traced |
| **@tanstack/react-query (new)** | Server-state cache layer, coexists with the hand-rolled service pattern — see §3.5 |
| @react-native-async-storage/async-storage | Unchanged |
| @react-native-firebase/app + /messaging | Unchanged |
| @react-native-clipboard/clipboard | Unchanged |
| @notifee/react-native | Unchanged |
| `AppBadgeModule.kt` / `AppBadgePackage.kt` (first-party native) | Unchanged |

### 3.13 Shared Components

New/changed components this pass (beyond what's already covered above in context):

| Component | File | Notes |
|---|---|---|
| **AppLock UI (new)** | `src/components/App/AppLock/*` | See §3.1a |
| **ConversationActionMenu (new)** | `src/components/App/ConversationActionMenu/index.tsx` (339 lines) | Pin/unpin, mark-as-read, leave-group, delete — long-press menu |
| **ForwardActionBar (new)** | `src/components/App/ForwardActionBar/index.tsx` (94 lines) | Selection-mode bottom bar for forwarding |
| **MessageInfoSheet (new)** | `src/components/App/MessageInfoSheet/index.tsx` (538 lines) | Per-recipient read/delivered/sent — see §3.8 |
| **MessageInputScrollHint (new)** | `src/components/App/MessageInputScrollHint/index.tsx` (140 lines) | Chat input UX polish |
| **VideoPlayer (new)** | `src/components/App/VideoPlayer/index.tsx` (610 lines) | Rewritten chat video player — see §3.8 |
| **TransportRideCard (new)** | `src/components/App/TransportRideCard/index.tsx` (243 lines) | New Transport-tab ride card |
| **List, InputText, Checkbox (new, generic)** | `src/components/App/List/`, `src/components/InputText/`, `src/components/Checkbox/` | Generic reusable primitives — likely backing the new Documents/Transport/CallSummary screens; usage breadth not traced |
| **NotificationDot (new)** | `src/components/App/NotificationDot/index.tsx` (33 lines) | Badge-dot primitive — `8363c821` "Notification Badge Dot on (HomeScreen, ResidentialScreen, Document List screen)" |
| **GlobalOtaBanner / GlobalOtaUpdateProgress (new)** | `src/components/GlobalOtaBanner/`, `src/components/GlobalOtaUpdateProgress/` | OTA update UI — see §3.1b |
| ChatBubble | `src/components/App/ChatBubble/index.tsx` (169-line diff) | Deleted-message state (prior pass) **plus (new)** video-download progress ring |
| MessageListPage | `src/components/App/MessageListPage/index.tsx` (431-line diff) | Paginated (prior pass); further changed this pass, not fully re-diffed |
| ReactionPicker, SegmentedControl, SessionGuard, AppHeader, BottomSheetSelect, GlobalToast | Various-sized diffs | Not individually re-verified beyond confirming no structural removal |
| Digital signature UI, Scheduling UI, Audio UI, Misc (AppHeader, NotificationBell, ConfirmationModal, ComingSoon, TransportDriverDropdown) | — | Unchanged from the prior pass unless noted above |

### 3.14 Build / Quality Scripts

`quality-report.js` unchanged (still analyses the source tree; `acorn-typescript`/`acorn-walk`/`write-good` still misplaced in `dependencies`, TD13). `postinstall` still runs `scripts/apply-patches.js`, now **401 lines** (grew substantially — not diffed line-for-line, presumably reconciling Expo-module patches with existing native customizations). **New:** `scripts/configure-ota-channel.js` (120 lines, EAS Update channel setup).

**CI pipeline detail (2026-08-27 delta):** `.gitlab-ci.yml`'s `build_android`/`build_ios` jobs resolve `OTA_CHANNEL`/`APP_ENVIRONMENT` from `$CI_COMMIT_REF_NAME` via a `case` statement (`development`→staging, `master`→preprod, `ota-release`→production, `staging`→prod-env/no-OTA, else→none), write `APP_ENVIRONMENT` into the build-time config file (loaded via `react-native-config`), and — when an OTA channel resolved — call `node scripts/configure-ota-channel.js` before the native build. After `bundle exec fastlane build`, both jobs install `eas-cli` and invoke `npm run update:staging|preprod|production` (§2's OTA npm scripts, now passing `--message "$OTA_UPDATE_MESSAGE"` sourced from `$CI_MERGE_REQUEST_TITLE`/`$CI_COMMIT_MESSAGE`). The `development` and `ota-release` branches — pre-existing on `origin` but previously outside every job's `only`/`rules` gate — are now included in `designqa_check`, `lint_reactnative`, `quality_check`, `sonar_reactnative`, `security-scan`, `build_android`, and `build_ios`. Separately, `build_ios`/`deploy_ios` now run `pod repo update` as its own step ahead of `pod install` inside the existing 3-attempt retry loop (previously one combined `pod install --repo-update` call) — a reliability change, not a behavior change.

### 3.15 Test Coverage

**Grew for the first time in a meaningful way this pass**, though still small relative to the app's surface: `biometrics.service.test.ts` (363 lines), `appLock.slice.test.ts` (63 lines), `enterApp.test.ts` (364 lines) — all covering the new biometric App Lock / gate-resolution logic. The pre-existing `__tests__/App.test.tsx` and `screens/Auth/*` tests are presumably still present (not confirmed removed). **Still untested:** the entire chat expansion (forward, drafts, pin, message-info, video player), the Skilled Nursing Transport/Documents/Call-Summary modules, the Expo/OTA update flow, and the Cognito migrator/multi-channel-MFA auth rework — all net-new, all currently at zero test coverage. See TD5 (updated).

---

## 4. Architecture Diagram and Key Flows

The high-level flow diagram from prior passes (Splash → Auth/App routing → Axios/Cognito/Socket services → Backend/AWS/Firebase) is still structurally accurate; this pass's additions (biometric gate, Expo/OTA, chat expansion, Transport/Documents/CallSummary modules) are service/screen-level and don't change the top-level shape, **except** that a new mandatory gate (biometric lock) now sits between "authenticated" and "app content visible." Flows below are updated in place; unlisted flows (4.2 Session Startup, 4.3 Real-time Event Propagation, 4.5 401 Token Refresh in its broad shape) are unchanged from the prior pass.

### Flow 4.1: Authentication (phone-or-email, + Cognito migrator + biometric gate)

```
SplashScreen.mount
  checkForceUpdate(): if BLOCK → stay on Splash
  hydrateCognitoStorage()
  ensurePushToken()
  getRefreshToken() from Keychain
    null  → navigate AUTH/SIGNIN
    present:
      handleInactivityExpiry(): 24h idle → server logout → AUTH/SIGNIN
      isTokenExpired() (2-min buffer)
        false → resolveGateTarget() [NEW]
                  'unlock' → AUTH/BIOMETRIC_UNLOCK → on success → enterApp() → flow route
                  'enroll' → AUTH/BIOMETRIC_ENROLL → on success/skip → enterApp() → flow route
                  'skip'   → enterApp() → flow route directly (facility opted out of biometrics)
        true  → refreshSession() → saveTokens() → resolveGateTarget() → (as above)
                AccessDeniedError → goToAccessDenied(); refresh failure → clearTokens() → AUTH/SIGNIN

SignInScreen → choose Phone or Email via SegmentedControl [NEW]
  signIn(identifier, password, identifierType)
    loginMfaChannel(identifier, channel) — pins MFA delivery channel [NEW, best-effort]
    CognitoUser.authenticateUser(USER_PASSWORD_AUTH)
      onSuccess               → saveTokens() → resolveGateTarget() → enterApp()
      NEW_PASSWORD_REQUIRED    → AUTH/CHANGE_PASSWORD
      MFA_VERIFY (TOTP/SMS/EMAIL_OTP) [3rd channel NEW] → AUTH/MFA_VERIFY
      onFailure (allowMigratorFallback) [NEW]
        → migratorSignIn(identifier, password, clientId) — backend userpool-migrator fallback
            'TOKENS'             → saveTokens() → resolveGateTarget() → enterApp()
            'MIGRATED_NO_TOKENS' → retry authenticateUser() natively once (fallback disabled)
                                     → typically resolves to MFA_VERIFY → AUTH/MFA_VERIFY
            migrator call itself fails → reject with the ORIGINAL Cognito error, not the migrator's

Forgot password: unchanged from the prior pass (OTP-reset + resend-credentials paths)
```

The Cognito migrator path (`migratorSignIn`) implies a live, in-flight Cognito user-pool migration on the backend — this app is the client-side half of that migration. Recommend confirming the backend-side migration plan/status with the architecture/backend team; out of this repo's scope to verify further.

### Flow 4.6: Environment Switching — unchanged core mechanism, now with a second entry point

`ChangeEnvironmentScreen` (Skilled Nursing Profile) and the new **`LocalConnectScreen`** (dev-only LAN/team-preset picker) both write the same `CurrentEnvironment` AsyncStorage key, log out, and call `RNRestart.restart()`. Base-URL resolution for both Axios and the two Socket.io connections now goes through the shared `resolveApiBaseUrl()` helper (§3.6) instead of being duplicated per call-site.

### Flow 4.8: OTA Update Check (new)

```
(Trigger point not fully traced this pass — likely app-foreground or a specific screen mount)
useOtaUpdate() → expo-updates checkForUpdateAsync()
  update available → GlobalOtaBanner prompts the user
    user accepts → fetchUpdateAsync() → GlobalOtaUpdateProgress shows download/apply progress → reload
  channel is resolved from the EAS build profile the binary was built with (staging/preprod/production, eas.json)
```

---

## 5. Data and State

### 5.1 Client-Owned (persisted) — additions this pass

| Data | Storage | Key / service | Notes |
|---|---|---|---|
| **Biometric App Lock device enrollment (new)** | react-native-keychain | service `com.shashigroup.sl.staffapp.app_lock` | Device-level, not account-level — see §3.1a |
| **Per-conversation chat drafts (new)** | react-native-keychain | service `com.shashigroup.sl.staffapp.drafts` | PHI-bearing (resident @-mentions); one JSON-map entry for all drafts, `ACCESSIBLE.ALWAYS` — see §3.8 |
| **Pending profile-activity telemetry queue (new)** | AsyncStorage | `PENDING_PROFILE_ACTIVITY_EVENTS` | Capped at 20 entries, per-`cName`-scoped, flushed on next successful profile fetch — see §3.7 |
| Everything else (Cognito tokens, username, token expiry, inactivity timestamp, FCM token, app flow, selected facility, active environment override, background preset, theme cache, pending audio-recording state, facility timezone override, Remember-Me credentials) | — | — | **Unchanged** from the prior pass |

### 5.2 / 5.3 / 5.4 — unchanged this pass

No material change verified to in-memory-only state, the "read from backend, not owned" posture (still no offline domain-data cache for anything except the new profile-activity telemetry queue — chat itself remains online-only, §3.8), or the logout/session-clear sequence beyond the new centralized `LogUserOut.ts`/`logoutGuard.ts` (§3.7) — behaviorally the same clear set, differently organized.

---

## 6. External Dependencies

| System | Kind | Direction | Auth method | Notes |
|---|---|---|---|---|
| Senior Living Backend | REST API | Outbound | Bearer + `x-facility-id` | Unchanged; every request now also carries `isFromMobileRequest: true` (new, §3.5) |
| Senior Living Backend (Socket.io, default ns / `/chat` ns) | WebSocket | Bidirectional | Unchanged | Chat event set unchanged this pass (§3.8) |
| AWS Cognito | HTTPS SDK | Outbound | USER_PASSWORD_AUTH (phone **or email**, new), TOTP/SMS/**Email OTP** (new) MFA | See §3.7, §4.1 |
| **Backend Cognito user-pool migrator (new)** | HTTPS (custom endpoint, not Cognito) | Outbound | Resolved per-env via `resolveUserpoolMigratorUrl()` | Fallback sign-in path when native Cognito auth fails — see §3.7 |
| Twilio Programmable Voice | SDK + token endpoint | Bidirectional (voice) | Unchanged | Now also the input to the new Secure Call summary flow |
| **Call-transcription/summarization service (new, unidentified)** | Unknown | Unknown | Unknown | The Secure Call summary screens consume transcribed text but this app contains no transcription code — the upstream service was **not identified this pass**, flag for architecture/PM follow-up |
| Firebase Cloud Messaging | Native module | Inbound (push) | Google Services config files, still tracked by git (unchanged — see G1) | Two new push types (pending-signature, resident-document-signed) |
| Notifee | Native module | N/A | N/A | Unchanged |
| Freshworks Widget | Webview-hosted JS SDK | Outbound | Unchanged | — |
| **EAS Update (`u.expo.dev`) (new)** | HTTPS (Expo-hosted) | Inbound (OTA JS bundle) | Project ID + per-environment channel (`eas.json`) | See §3.1b — bypasses app-store review; no documented rollback runbook yet (G13) |
| Google Play / App Store + OS settings | OS deep link / intent | Outbound | N/A | Unchanged |

No direct AWS S3, SES, or Secrets Manager calls from the app (unchanged). Google Places API key/dependency still present, still unused (unchanged, TD/G7).

---

## 7. Security and Multi-tenancy

### 7.1 Authentication — updated

Cognito Bearer auth from Keychain, 2-minute expiry buffer, reactive 401 refresh — unchanged mechanism. **Sign-in identifier is now phone OR email** (was phone-only) — broadens the credential surface; Remember-Me now stores per-identifier-type. MFA now supports 3 channels (TOTP/SMS/Email OTP) with an explicit channel-pin call before `InitiateAuth`. **New:** a backend Cognito user-pool migrator fallback path (§3.7, §4.1) — this is a meaningful auth-architecture change; recommend the backend-side migration plan get an explicit security review if it hasn't had one (out of this repo's scope to assess). **New:** a device-level biometric App Lock sits in front of the authenticated app tree (§3.1a) — note its device-level (not account-level) enrollment scope as a discussion point for shared-device deployments (Design Gap G14). Terms & Conditions acceptance-persistence semantics remain untraced (G12, unchanged, still open).

### 7.2 Multi-tenancy (Facility Scoping) — unchanged

`x-facility-id` header injection unchanged; `cognito.storage.ts` (adjacent to this posture) untouched this pass. Facility isolation remains backend-enforced.

### 7.3 Committed Secrets — unchanged, one new low-risk item

| Secret | Location | Risk |
|---|---|---|
| Google Places API Key | `src/utils/local.constants.ts:1` | HIGH — unchanged |
| Developer LAN IPs / ngrok URLs | `src/utils/local.constants.ts` | LOW — unchanged |
| Firebase Android/iOS config | `android/app/google-services.json`, `ios/GoogleService-Info.plist` | HIGH — still tracked by git (confirmed via `git ls-tree` on `origin/master`) |
| `.env` secrets (Cognito IDs, Twilio URL, Freshworks widget ID/URL) | repo `.env` | Source of injected config — unchanged |
| Freshworks widget hardcoded fallback | `src/services/App/FreshworksWidgetService.ts:4-6` | LOW — unchanged, TD14 |
| **EAS/Expo project ID (new)** | `app.json` (`extra.eas.projectId`) | LOW — a project identifier, not a credential, but worth noting as a new externally-resolvable identifier in the committed config |

### 7.4 Credential Storage — extended

Tokens + Remember-Me in react-native-keychain — unchanged (Remember-Me still plaintext phone/email + password, TD3, unchanged). **New:** biometric App Lock device-enrollment nonce and per-conversation chat drafts also now live in Keychain (§5.1) — both are PHI-adjacent-or-PHI-bearing and Keychain is the correct choice, but note drafts use `ACCESSIBLE.ALWAYS` (readable even when the device is locked) — worth confirming this is the intended tradeoff given drafts can contain resident PHI via @-mentions.

### 7.5 Client-Side Authorization — extended

App-flow/feature-access still backend-authoritative via `staffDirectoryRoles`. **New this pass:** `TRANSPORT_TAB_ROLE_GROUPS` / `DOCUMENT_UPLOAD_ROLE_GROUPS` extend the same client-side-derived-from-backend-data pattern already flagged in the prior pass (§7.5 there) — same posture, same caveat (worth backend-side enforcement-parity checks on the Transport/Documents endpoints, not independently verified this pass).

### 7.6 Sensitive Data in Logs — not re-verified this pass

Prior-pass findings (`cognito.service.ts`, `foregroundNotifications.ts`, `index.js`) not re-checked given the volume of unrelated churn in `cognito.service.ts` specifically (334-line diff this pass) — this file in particular is now overdue for a dedicated logging-hygiene pass given how much new `__DEV__`-gated (and possibly ungated) logging was added around the migrator/MFA-channel flow (`console.log('[Auth][signIn] ...')` calls observed throughout, all appear `__DEV__`-gated in what was sampled — not exhaustively confirmed).

---

## 8. Design Gaps

Re-evaluated against **`origin/master` HEAD `35b3cc82`**. G1–G12 re-confirmed unless noted; **G13 and G14 are new this pass.**

| ID | Severity | Issue | Evidence (file:line) | Decision needed before promote? | Recommended fix |
|---|---|---|---|---|---|
| G1 | **HIGH** | Firebase config files committed and tracked by git. | `android/app/google-services.json`, `ios/GoogleService-Info.plist` — confirmed via `git ls-tree` on `origin/master` | YES | Gitignore both paths; inject via CI; rotate keys. |
| G2 | ~~BLOCKER~~ **RESOLVED** | `CognitoStorage` functional. | — | NO | — |
| G3 | **MEDIUM** | FCM background data-only messages logged but not displayed. | `index.js` | YES | Display via Notifee. |
| G4 | **HIGH** | App-flow gating entirely client-side off `staffDirectoryRoles`; now joined by two more client-derived role-group gates (Transport, Documents tabs) on the same pattern. `DietitianView` still unreachable. | `src/utils/featureAccess.ts:16-28,92-102` | YES | Drive from backend config; wire or remove DietitianView. |
| G5 | **MEDIUM** | `moveSalonAppointment` still silently falls back PATCH→POST on 404/405. | `src/services/App/index.tsx` (not re-line-numbered this pass) | YES | Standardize the method. |
| G6 | **MEDIUM** | `messaging().onTokenRefresh()` still unregistered. | Not found in `src/` | YES | Register it. |
| G7 | **MEDIUM** | Google Places key + autocomplete dependency present, unused. | `src/utils/local.constants.ts:1` | YES | Restrict/revoke. |
| G8 | **HIGH** | Dark mode still locked to light. | `App.tsx`; `src/theme/index.tsx` | NO unless required | Wire a toggle or document deferral. |
| G9 | **MEDIUM** | LEGACY designation socket still tied to HomeScreen mount/unmount. | `HomeScreen/index.tsx` | NO | Lift to an AppStack-level manager like the chat one. |
| G10 | **MEDIUM** | Twilio token endpoint trust unverifiable from the client. | `src/services/TwilioService.js` | YES | Require an authenticated call to mint Voice tokens. |
| G11 | **MEDIUM** | MIGRATED-flow `REPORTS` tab still registered but commented out — dead, unreachable. Unchanged this pass despite the tab bar otherwise growing (Transport, Documents both added). | `src/navigation/skilledNursingAppStack/index.tsx:391-419` | YES — same open question as before | Re-enable behind the correct role gate or delete. |
| G12 | **LOW** | Terms & Conditions acceptance-persistence semantics still not fully traced. | `src/screens/Auth/TermsWebViewScreen/index.tsx` | YES | Dedicated review. |
| **G13** | **MEDIUM** | **New.** Expo/EAS OTA update mechanism (`expo-updates`) can push JS-bundle changes directly to production users bypassing app-store review, but no rollback runbook or kill-switch procedure was found — `docs/ota-channel-setup.md` covers channel *setup* only. **(2026-08-27: OTA publish is now wired into CI and fires automatically per-branch — see §3.1b/§3.14 delta — so this gap is no longer theoretical.)** | `app.json`, `eas.json`, `docs/ota-channel-setup.md`, `.gitlab-ci.yml` | YES — before relying on this for a production hotfix path | Write an OTA-rollback runbook entry (revert-to-previous-update procedure, who can trigger it, how staging validates first). |
| **G14** | **MEDIUM** | **New.** Biometric App Lock enrollment is device-level, not account-level (explicitly by design per in-code comments) — once any staff member enrolls a device, any subsequent user of that device unlocks the same way, regardless of which account is signed in. On a shared/handed-off facility device this could let one staff member's biometric unlock a session that isn't theirs (though the underlying Cognito session is still per-account and still expires/requires re-auth independently). | `src/services/Biometrics/biometrics.service.ts` (comments throughout) | YES — confirm with product/security whether this is the intended posture for shared devices | Confirm intent; if not intended for shared devices, consider account-scoped enrollment or a device-sharing warning. |

---

## 9. Technical Debt

Re-verified against `origin/master` `35b3cc82`. TD1–TD14 re-confirmed unless noted; **TD15 and TD16 are new.**

| ID | Severity | Issue | Evidence (file:line) | Recommended fix |
|---|---|---|---|---|
| TD1 | **HIGH** | Unguarded `console.log` in production builds (not re-verified line-for-line this pass — see §7.6; likely grew given `cognito.service.ts`'s size increase). | as previously listed | Wrap in `__DEV__` or strip in release. |
| TD2 | ~~HIGH~~ **RESOLVED** | Auth tokens in Keychain. | — | — |
| TD3 | **HIGH** | "Remember Me" still stores plaintext phone/email + password in Keychain. | `src/utils/authStorage.ts` | Store only the refresh token (or nothing). |
| TD4 | ~~HIGH~~ **RESOLVED** | Cognito pool/client IDs injected per-env. | — | — |
| TD5 | **HIGH** | Test coverage grew (biometrics/appLock/enterApp — first real unit tests in the app, §3.15) but the surface grew far faster: the entire chat expansion, Transport/Documents/Call-Summary modules, the Expo/OTA flow, and the Cognito migrator/MFA rework are all untested. | `__tests__/` | Prioritize: chat draft/pin/forward logic, the migrator sign-in fallback branch, the biometric-gate ↔ inactivity-timeout interaction. |
| TD6 | **MEDIUM** | God-file API service: `src/services/App/index.tsx`, grew further this pass (327-line diff), exact new size not re-tallied. | `src/services/App/index.tsx` | Split by domain; consider routing new domains through the new React Query layer instead of appending here. |
| TD7 | **MEDIUM** | LEGACY designation socket lifecycle still tied to HomeScreen mount/unmount. | `HomeScreen/index.tsx` | Lift to an AppStack-level manager. |
| TD8 | **MEDIUM** | Axios interceptor still resolves the environment override per-request (now via `resolveApiBaseUrl()`, same cost, relocated). | `src/utils/apiEnvironment.ts` | Cache in a module variable. |
| TD9 | **MEDIUM** | No top-level error boundary. | Not found in `src/` | Wrap `RootNavigator`. |
| TD10 | **MEDIUM** | `MassageTherapistView` lacks pull-to-refresh. | `DesignationViews/MassageTherapistView.tsx` | Add `RefreshControl`. |
| TD11 | **MEDIUM** | Cross-flow chat-screen sharing by string-name aliasing. | `src/navigation/appstack/index.tsx` | Extract the chat sub-graph into a shared navigator. |
| TD12 | **LOW** | Dead/unwired code: `DietitianView.tsx`; commented-out `REPORTS` tab (G11). | as listed | Remove or wire. |
| TD13 | **LOW** | `acorn-typescript`, `acorn-walk`, `write-good` still in `dependencies`. | `package.json` | Move to `devDependencies`. |
| TD14 | **LOW** | `FreshworksWidgetService.ts` hardcoded fallback widget URL/ID. | `src/services/App/FreshworksWidgetService.ts:4-6` | Remove the fallback; fail loudly if config is absent. |
| **TD15** | **MEDIUM** | **New.** `SHOW_SCAN_DOCUMENTS_TAB` is a hardcoded `true` constant in source, not a real remote/server-driven feature flag — shipping a "flag" that can only be toggled by a new app release defeats the usual purpose of gating a still-maturing feature. | `src/utils/local.constants.ts:43` | If the Documents tab needs a kill-switch independent of app releases, move this to `residencyConfig` (same pattern as `biometricEnabled`) or a remote-config service. |
| **TD16** | **MEDIUM** | **New.** React Query (`@tanstack/react-query`) was introduced as a second data-fetching pattern alongside the existing hand-rolled `src/services/App/index.tsx` + Context/useReducer approach, with no migration plan documented in-repo (no ADR, no README note on which is canonical going forward). Two coexisting patterns for the same concern raises long-term maintainability risk if adoption stalls halfway. | `src/services/queryClient.ts`, `src/utils/queries.ts` | Document the intended end-state (full migration vs. deliberate coexistence for specific use cases) so new code has a clear default. |

---

## 10. Open Questions for Next Pass (new section)

Carried forward explicitly so they aren't lost in prose above:

1. Relationship between the legacy `TransportationScreen` (existing) and the new tab-level `TransportScreen` — duplication, or deliberate list/detail split? (§3.2b)
2. Exact trigger point for the OTA update check (`useOtaUpdate.ts`) — app-foreground, a specific screen, or a manual action? (§3.1b)
3. Adoption breadth of `@tanstack/react-query` and `@shopify/flash-list` — which screens actually use them vs. the existing patterns? (§3.5, §3.11, TD16)
4. Identity of the external call-transcription/summarization service feeding the Secure Call module — not identified this pass. (§3.2e, §6)
5. Exact provider-tree placement of the React Query provider and `AppLockGate` relative to `StoreProvider` — `App.tsx`'s 49-line diff was not fully re-diffed. (§3.10)
6. Whether `residencyConfig`'s new fields (`biometricEnabled`, `lockScreenTimeout`) are documented on the backend side / have a corresponding admin-UI control — worth cross-checking against `senior_living_backend` and `senior_living_admin`.
