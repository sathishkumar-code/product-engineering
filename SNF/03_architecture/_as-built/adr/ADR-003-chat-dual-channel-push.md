# ADR-003 — Chat push: dual-channel split (FCM offline-only + Web Push to all) with in-memory presence

**Status:** Accepted (as-built, production HEAD `981704e9`) — with a scaling caveat. **Amended 2026-07-12** to record two admin-PWA decisions made downstream of this ADR (see "Amendment" below).
**Date:** 2026-06-17 (documenting an existing decision); amended 2026-07-12
**Area:** `senior_living_backend` — `services/chat/chatNotification.service.ts`, `services/webPush.service.ts`, `socket/`; `senior_living_admin` — `usePushNotifications`, `useGlobalChatSocket`, `public/sw.js`, `vite.config.ts`, `src/services/chatWindowMessages.ts`
**Related:** PRD `messaging-chat.md` MSG-FR-25/26, backend architecture "Messaging & real-time delivery", admin architecture "Messaging, real-time & web push"

## Context

Chat is real-time over the `/chat` Socket.io namespace. Recipients not holding a live socket need a push fallback across two device classes: mobile (FCM) and browser (no FCM — needs W3C Web Push/VAPID). Naively pushing to *everyone* double-buzzes online users; pushing only to "offline" users misses browsers that just closed, because Socket.io keeps a closed tab's socket in its room until the **~20 s `pingTimeout`** elapses.

## Decision

Split the two channels by recipient set:
- **FCM (mobile) → offline recipients only** (`offlineCNames`).
- **Web Push (browser) → all recipients** (`allCNames`), accepting redundant sends to cover the disconnect window.

Online detection is an **in-process in-memory** check of `/chat` room membership (`isUserOnline()`). The browser **service worker + in-page socket handler** enforce a three-layer dedupe (suppress if a tab is visible; bail on `document.hidden`; dismiss the `chat-<conversationId>`-tagged OS notification on focus, retried after 2 s for VAPID lag). Both payloads carry `conversationType` for tap-routing. Delivery is **at-least-once** with client-side suppression.

## Consequences

**Positive**
- No double-buzz for online mobile users; browsers reliably notified despite the presence-lag window.
- Client-side dedupe keeps the UX clean without a server-side delivery ledger.
- `WebPushSubscription` supports multi-device per user and self-prunes on HTTP 410-Gone.

**Negative / follow-ups**
- **Single-instance presence (the key caveat):** the in-memory Socket.io adapter means `isUserOnline()` only sees *this* node's sockets. On a multi-replica deployment a user connected to node A appears offline on node B → FCM double-notifies online mobile users. **Requires `@socket.io/redis-adapter` before horizontal scaling.**
- **VAPID secrets** (`VAPID_*`) are not in the `Senior_Assisted_Living` secret; web push silent-disables if unset. SW requires root-origin hosting for scope.
- **Subscriptions have no TTL** — abandoned ones persist until a send returns 410.
- **iOS reliability:** the explicit APNs override block was removed, so iOS chat push relies on FCM defaults (no custom sound/badge/`content-available`) — monitor delivery.
- **FCM not chunked** at 500 tokens in the chat path (not a practical risk at current scale, but a latent ceiling).

## Amendment (2026-07-12) — admin PWA scope and cross-window transport

`senior_living_admin` became an installable PWA in this cycle (`vite-plugin-pwa`, `injectManifest`, final app name "Shashi Messaging"). Two implementation decisions flow directly from this ADR's dual-channel model and are recorded here rather than as a new ADR, since neither changes the dual-channel decision itself — both only refine *how* the Web Push side is delivered inside the admin app:

- **Service worker scope is `/messages`, not `/` (root).** The alternative — registering the SW at root scope — was explicitly rejected in the code comments because it would make the entire admin panel falsely "installable" as a PWA, not just the chat surface. Consequence (tracked as a new Medium design gap in the admin per-repo doc): Web Push subscription and OS app-icon badging are unavailable to any staff/admin user who has never opened the `/messages` chat popup at least once — nothing in onboarding currently prompts that first open.
- **Cross-window transport (Messages↔Main) uses `BroadcastChannel` (`sal-chat-window-sync`), not `window.opener`.** `window.opener`-based messaging (`notifyOpener()`) broke for three real cases: an installed-PWA launch, a hand-typed `/messages` URL, and an independent second admin tab — in all three, `window.opener` is `null` or points at the wrong tab. `BroadcastChannel` is same-origin and doesn't depend on the window-opener relationship. Main→Messages traffic (`OPEN_CONVERSATION`, `PARENT_LOGGED_OUT`) remains a targeted `window.postMessage`, unchanged.

Both decisions are downstream refinements of the existing dual-channel/presence model, not a new architectural direction — recorded here so the SW-scope trade-off (root-scope rejected) and the BroadcastChannel rationale survive as a decision record rather than only as code comments. See [`review-senior_living_admin.md`](../../reviews/2026-07-12/review-senior_living_admin.md).
