# Module: Messaging / Chat

> Applies to: Both (platform capability) — currently consumed by Skilled Nursing surfaces, plus the admin web and (since staging) the staff app for every designation
> FR prefix: MSG
> Sources: _codebase-analysis files (code is source of truth)
> — `backend-clinical-care.md` §0, §7, §9 (primary — backend `Conversation`/`Message`, `/api/chat`, socket `/chat`)
> — `client-admin-web.md` §2.14 (admin web Messages)
> — `client-resident-app-sn.md` §3.3, §4, §5 (SN resident app chat)
> — `client-staff-app.md`, `client-resident-app-sl.md` (verified absences)
> — 2026-07-12 delta: `docs/reviews/2026-07-12/review-senior_living_backend.md` (chat production hardening, `feature/chat-production` merged 2026-07-10), `review-senior_living_admin.md` (installable Chat PWA, cache/socket reconciler rewrite), `review-senior_living_staffapp.md` (staff-app chat surface correction, production hardening)

---

## 1. Purpose & scope

Internal, facility-scoped chat between residents (and family members acting as the
resident), care-team staff, and admins. Supports 1:1 (DIRECT) and staff/admin-managed
GROUP conversations with rich message features (attachments, @mentions, reactions,
replies, soft-delete tombstones), WhatsApp-style delivery receipts, per-conversation
encryption at rest, and real-time fan-out over Socket.io with FCM push fallback.

**In scope (as-built):**
- Conversation lifecycle (DIRECT dedup, implicit DM creation, GROUP CRUD + membership + admin roles)
- Message send/receive: text, attachments (image/video/audio/document), mentions, reactions, replies, sender soft-delete, system messages for group events
- Delivery semantics: SENT → DELIVERED → READ, per-member receipts (now with a monotonic status ratchet and membership-aware group quorum — §3 "Delivery semantics & unread"), unread counts, bulk mark-read
- Initiation access policy (care-team gating, resident↔resident ban, per-facility config flags)
- Encryption-at-rest (per-conversation KMS envelope key)
- Real-time socket event contract + reconnect unread snapshot
- User directory search and resident care-team contact list
- An installable admin-web chat PWA ("Shashi Messaging") with Web Push + OS app-icon badging (§3 "Push channels", §6)

**Out of scope (separate modules):**
- Rehab Messages (`/api/rehab/rehab-message`) — a status-tracked resident→rehab-department request queue, *not* chat (see Rehab/Therapy module; backend-clinical-care.md §5c)
- Announcements (broadcast, no replies — admin web §2.13)
- In-app notification feeds and generic FCM notification handling (Notifications module)

**Backend footprint (staging — chat module decomposed):** `Conversation.model.ts`,
`Message.model.ts`, `/api/chat` (`chat.routes.ts`). The former monolithic
`controllers/chat.controller.ts` (~970 LOC) and flat `services/chat/*.service.ts`
files were split into a `controllers/chat/` directory (10 files) and
`services/chat/{conversation,message}/` sub-packages (`chatAccessPolicy.ts`,
`conversation/careTeam.service.ts`, `message/send.service.ts`, etc.);
`src/constants/chat.ts`; socket handler for the `/chat` namespace. Chat routes are
imported via the legacy `contants/` (sic) typo directory (`chat.routes.ts`).
(backend-clinical-care.md §7.)

**Production-hardening pass (`feature/chat-production`, merged 2026-07-10, backend HEAD `ca628bc8`):**
a follow-up hardening effort rewrote delivery/read-receipt status computation, unread-badge
aggregation, and the chat notification service, and added a `Message.recipients: string[]`
field (+2 new indexes) to make group delivery/read quorum membership-aware. See §3
"Delivery semantics & unread" and §5 "Push & local notifications" for the resulting
functional changes. This was accompanied by an admin-web rewrite of the client-side
message cache/socket-reconciler layer and the admin Chat module becoming an installable
PWA (§3 "Push channels", §6) — both client-side changes with no new backend contract.

**Chat "v2" hardening wave (`feature/chat-v2-staging`, promoted to `pre-production` 2026-07-28/29):**
despite the "v2" branch name this is a **hardening pass, not a rewrite** — two backend
correctness fixes plus the admin Message-Info panel. (a) **KMS key-cache concurrency fix**
(`cacheOrReuseConversationKey`, `chatKeyCache.ts`): two concurrent cache-miss callers could
previously zero each other's in-use AES key buffer mid-decrypt; the new path reuses a single
buffered key so a concurrent write no longer corrupts an in-flight message (MSG-FR-24).
(b) **Receipt integrity against the per-user clear floor**: mark-read / mark-delivered / the
reconnect sweep now exclude any message at or below the caller's `ConversationMemberState`
clear cursor, so acknowledging new activity can never retroactively re-stamp a cleared
message (MSG-FR-29). (c) Admin web: the **Message Info** per-recipient delivered/read panel
(MSG-FR-30) reached pre-production. **v1.5 additions (admin web):** (i) message **URL
linkification** — sent-message URLs render as blue `target=_blank` anchors (new
`urlPattern.ts`/`linkify.tsx`) and the composer auto-links live via a Lexical
`AutoLinkPlugin` (requires the new `@lexical/link` dep); (ii) a chat **build-version +
PWA update system** — `chat.version.json` (hand-bumped) is watermarked in the installed
PWA and reported to `/{staff|residents}/profile/activity`, and `registerType:'prompt'` +
a 20-min update poll surface a dismissible "New version available / Refresh" toast that
never auto-reloads (protecting composer drafts); (iii) **residents were temporarily
removed from the @mention candidate list** (`residentCandidates = []`, marked TEMP) —
group @mention currently resolves staff/admin only. (`MessageComposer.tsx`,
`ConversationList.tsx`, `useChatVersionReport.ts`, `ChatToast.tsx`) ⚠ **Deployment flag:** on the pre-production admin branch
the `/chat` socket base URL comes from a **`VITE_SOCKET_PREPROD_URL`** var that neither the
production socket path (`VITE_SOCKET_PROD_URL`) nor the repo CLAUDE.md documents — likely
branch-specific; verify before promoting past pre-production (`src/services/chatSocket.ts`).

**Chat "v2.7" (`feature/chat-v2-pre-production`, merged 2026-08-10, plus two cherry-picks
2026-08-13/17) — admin web only:** message editing, forwarding, per-user pinning, drafts, and UI
polish. (a) **Message editing** — sender-only, in-place text edit (MSG-FR-31), closing half of
the module's long-standing "no edit, no forward" limitation (§4.2, §9 item 7). (b) **Message
forwarding** — one or more messages to one or more targets in a single action, with per-target
optimistic UI and rollback (`POST /chat/messages/forward`, MSG-FR-32). (c) **Per-user conversation
pinning** — a personal, non-shared display preference; pinned threads sort above unpinned ones
(MSG-FR-33). (d) **localStorage message drafts**, keyed per conversation, surviving a full
tab/browser close but deliberately not live-synced across windows (MSG-FR-34). (e) UI polish: a
**Copy Text** message-options action and a **Jump to Present** floating scroll-to-latest control
(MSG-FR-35). A same-window follow-up cherry-pick (2026-08-13) **disabled the sent-message
notification sound** — `playSendSound()` calls commented out, not deleted, with a code comment
"disabled for now; will be made configurable per user" (§5.3). **All of this is admin-web only** —
the SN resident app and staff app chat surfaces are unaffected, a new client-parity gap (§8, §9).

**Backend chat v2 (`feature/chat-v2-pre-production`, merged 2026-08-10, backend HEAD `4612b2be`) — the
actual server-side edit/forward/retention work behind "v2.7," plus a new automated-messaging identity.**
(a) **Configurable edit window** — `Config.chat.maxEditWindowMinutes` (default 15, mirroring WhatsApp's
own edit-window convention) is enforced server-side in `edit.service.ts` (`FORBIDDEN` once elapsed); the
admin UI hiding the Edit action past the window is a convenience only, not the authority — backs MSG-FR-31.
(b) **Retention policy reversed to fully non-destructive — corrects MSG-FR-12 and the §4.2 "Zero-member
group auto-destruction" rule below, both of which previously documented eager hard-deletion as current
behavior.** Two commits (2026-07-31, 2026-08-03; code comments cite HIPAA / California's 7-year retention
requirement) removed all physical destruction from both delete paths: `softDeleteMessageRecord` now only
sets `deletedAt`/`deletedBy` — message `content`, `attachments`, and `reactions` are never cleared and the
S3 object is never deleted, only hidden behind the existing `deletedAt`-gated tombstone read paths.
`destroyGroupData` (renamed `softDeleteGroupData`) likewise no longer hard-deletes messages, the
`Conversation` document, the group picture, or any S3 attachment for a destroyed group (creator-initiated
delete or automatic last-member-leaves cleanup) — it sets `deletedAt`/`deletedBy` on the `Conversation`,
and every read path that loads or lists conversations now filters `deletedAt: null`, reproducing the old
"vanishes" behavior without destroying data. No API/socket contract change — every client-facing read path
already gated on `deletedAt` rather than inferring "deleted" from empty fields. A pre-existing bug
(predating this change) was fixed as a side effect: a destroyed group's `ConversationMemberState.unreadCount`
is now cleared, where it previously stayed permanently inflated. (c) **`ChatSystemUser` — a new non-human,
per-facility chat identity** that lets a backend module post automated cards into a facility-configured
chat group via `sendModuleMessage()` (same `sendMessage()` core as a human send — encryption, mention
validation, persistence, receipts, notification fan-out — with no separate contract). A module is bound to
one or more `(conversation, identity)` pairs via `Config.chat.moduleMessageBindings`, with a per-binding,
per-event `messageNotifyPreference` (opt-out, not opt-in — an event absent from the map is enabled).
**Transportation is the only consumer so far** (`transportationRequest.controller.ts` →
`transportationChatTemplates.ts`, one shared card template per ride-lifecycle event, mention-enabled — see
transportation.md). `sendModuleMessage` never throws; a chat-automation failure never affects the calling
module's own API response. **Provisioning is manual only** — there is no creation API for a
`ChatSystemUser`; facilities are onboarded onto module chat automation by direct document write (§9 new
observation, MSG-FR-36).

**Message pinning (`feature/chat-v2-pre-production`, merged 2026-08-25, backend HEAD `e6469276`,
admin web HEAD `f5b461c6`) — the next wave on the same branch as "v2.7"/"backend chat v2" above,
admin web only.** Adds pin/unpin for individual messages to a **conversation-wide, shared** tray —
distinct from the per-user, view-only conversation pinning of MSG-FR-33 — backed by a new
`ConversationPinState` collection (one document per conversation) that makes a **dual cap** (how
many messages one participant may pin, and how many the conversation holds in total across every
pinner) enforceable atomically without a transaction. Pins can expire on a facility-configured
duration, including a "forever" option. See MSG-FR-43–43d below. **Known rough edge, not yet
resolved:** a same-window, in-progress fix for the message list's general scroll-to-bottom
reliability (mount position, rapid-send follow-through, native scroll-anchoring fighting the
virtualizer) landed as an explicit checkpoint commit — the author's own commit message states the
underlying issue is "not yet fully resolved in all cases." Treat scroll-to-bottom behavior as still
occasionally unreliable, not a shipped guarantee, until a follow-up closes it out (§9 item 16).

---

## 2. Personas & surfaces

### 2.1 Chat identities

Chat roles on documents are `RESIDENT | STAFF | ADMIN` (backend §7). Two notes:

- **Family members chat as the resident.** Auth middleware rewrites
  `req.user.username` to the linked resident's cName (`authMiddleware.ts:94-145`);
  the original identity survives only in `familyMemberCName`. There is no distinct
  FAMILY chat role on conversation/message documents, though FAMILY_MEMBER tokens may
  call the HTTP routes (backend §0, §7).
- **cName (Cognito username) is the person join key** across residents, staff, and
  chat (client-admin-web.md §"observations" item 7). Identity redesigns must preserve it.

### 2.2 Surface matrix (verified from client analyses; staff-app row corrected 2026-07-12)

| Surface | Chat present? | Capability |
|---|---|---|
| **SN resident app** (`senior_living_skillednursing_resident`) | **Yes** — dedicated bottom tab | 1:1 only, against the resident's care-team contact list from `GET /chat/care-team-contacts`. On staging the backend returns the resident's **full `assignedStaff[]`** roster (any designation; MSG-FR-23); the client's `CareTeamRole` enum still names the four legacy roles (stale type). Text-first UI; no group creation UI (GROUP exists in types only). Socket.io real-time + local chat notifications. |
| **Admin web** (`senior_living_admin`) | **Yes** — full Messages module (~5k lines, 16 files), and (since production HEAD `324840a`, 2026-07-10) an **installable PWA** ("Shashi Messaging", scoped to `/messages`) | DIRECT + GROUP, attachments, mentions, reactions, replies, delete, group management, receipts, unread badges (client-admin-web.md §2.14). Message cache is now a React Query `useInfiniteQuery` + single-reconciler design (Zod-validated inbound socket events) rather than hand-rolled buffering. Web Push + OS app-icon badge via a `/messages`-scoped service worker (§3 "Push channels"). No frontend permission gate — all authenticated portal users can chat; restrictions are server-side. |
| **SL resident app** (`senior_living_reactnative`) | **No** | "No brain games, no chat/messaging…" — explicitly absent (client-resident-app-sl.md, §"explicitly absent"). |
| **Staff app** (`senior_living_staffapp`) | **Yes — corrected 2026-07-12.** Chat is live for every designation and is under active production hardening as of `master@4aa3849` (2026-07-11). | 1:1 + GROUP conversations. Delivery/read receipts are emitted with reconnect-safe re-flush (`pendingDeliveredIds`/`pendingReadIds` queued while disconnected, flushed on connect plus a delayed second pass; `forceNew: true` socket reconnects tear down stale sockets). Message delete renders a distinct sent/received tombstone state (`chat:deleted`, `MessageDeletedIcon`). Group-membership/system events (`chat:group`) drive in-app system messages on the Group Conversation/Details screens. A dedicated Android notification channel (`chat_messages_v2`) plays a custom sound (`chat_notification.mp3`), with separate in-app sent/received cues (`message_sent.mp3`/`message_receive.mp3`) via `react-native-sound`. The conversation list (`MessageListPage`) is now real-server-side-paginated rather than fetch-all. Optimistic sends are preserved across background resyncs. Chat media supports pinch-zoom (`ZoomableImage`). *This corrects the module's prior "no chat module found" reading — see §9 observation 1.* **2026-08-21 addition:** message forward (to multiple destination conversations, with attachments), PHI-aware Keychain-backed per-conversation drafts, a long-press conversation menu (pin/unpin — cross-device synced, capped by a facility-config limit — mark-as-read, leave-group), a per-recipient **Message Info** sheet, and a rewritten video player with orientation support — see MSG-FR-40/41/42. **Offline message sync was built on a separate branch but never merged into `master` — chat remains fully online-only in the shipped app.** |
| **TV app** | **No** | No chat referenced in TV analysis scope. |

**Net (revised 2026-07-12):** residents on SN can message their care team; staff/admins
can now answer from **either** the admin web portal **or** the staff app — chat is a live
mobile surface for every staff designation, not admin-web-only. The residual gap is
narrower than previously documented: SN resident-side feature parity (attachments,
reactions, replies, mentions, groups) still lags the admin web and staff app clients
(§9 observation 2), and cold-start/background chat deep links are still missing on
the SN resident app (§9 observation 3).

---

## 3. Functional requirements (as-built)

### Conversations

- **MSG-FR-01 — Direct conversations.** The system supports 1:1 DIRECT conversations
  between two participants identified by cName + chat role. (backend §7)
- **MSG-FR-02 — Direct dedup.** Concurrent first-sends cannot create duplicate DIRECT
  threads: a sparse-unique `directPairKey` (sorted cName hash, per facility) enforces
  one thread per pair. (backend §7 "Conversations & messages")
- **MSG-FR-03 — Implicit DM creation.** A first message sent with
  `to: { cName, role }` (instead of `conversationId`) creates the conversation
  server-side; `conversationId` and `to` are mutually exclusive in the send payload.
  (admin web §2.14 send payload; SN app §3.3 "two send paths")
- **MSG-FR-04 — Group conversations.** GROUP conversations carry metadata: name,
  picture, createdBy, admins[]. Creation/update/deletion and membership/admin
  management are restricted to STAFF | ADMIN | SUPER_ADMIN roles (route-level).
  At the service layer, group **creation** is allowlisted to STAFF and ADMIN; an
  ADMIN creator is additionally gated by the facility's `chat.isAdminAllowed` flag
  (creation rejected when admin chat is disabled). The creator becomes the sole
  initial admin. **`config.chat.maxGroupMembers` is now enforced client-side** in `GroupCreationModal` (cap at `maxGroupMembers - 1`; creator counts as 1 member) and `AddGroupMembersModal` (cap at `maxGroupMembers - existing members`); selection past the limit is blocked with a toast (production 2026-06-30). Falls back to no limit when config is unavailable. (backend §7; `group.service.ts:69-81`)
- **MSG-FR-05 — Conversation list.** Users retrieve their conversations
  (`GET /chat/conversations?limit=50`) with per-conversation `unreadCount`,
  `lastMessage` preview, and (DIRECT) peer / (GROUP) participant previews for the
  avatar stack. (admin web §2.14)
- **MSG-FR-06 — Conversation info & attachment panels.** Per-conversation endpoints
  expose `/info` and `/attachments` (separate media vs file cursors); attachment
  counts are denormalized on the conversation (`mediaAttachmentCount`,
  `fileAttachmentCount`) for O(1) info panels. (backend §7; admin web §2.14 endpoints)

### Messages

- **MSG-FR-07 — Text messages.** Plain-text body; admin composer uses Enter = send,
  Shift+Enter = newline, with an emoji picker. (admin web §2.14)
- **MSG-FR-08 — Attachments.** Messages may carry attachments of type
  `image | video | audio | document`, subject to **config-driven per-type size and
  per-message count limits**. Backend defaults: max 5 images, 2 videos, 10 attachments
  total per message; image 5 MB, video 30 MB (config defaults); MIME allowlists
  enforced. Admin composer enforces the limits it reads from `config.chat` (observed
      values there: image 5 MB ×10, video 50 MB ×3, audio 10 MB, document 20 MB — see
  §9 observation 4 on the discrepancy). Video attachments are paired by index with
  client-extracted first-frame poster thumbnails. (backend §7; admin web §2.14)
- **MSG-FR-09 — @Mentions (groups only).** GROUP messages support up to 20 mentions,
  serialized as sentinel-wrapped tokens (STX + `@cName` + ETX) in the text plus a
  `mentions: [{cName, role}]` array; server-validated against conversation
  participants; receivers get a `mentionsMe` flag; mentions are denormalized onto
  `lastMessage.mentions` for inbox badges. Admin web provides a Lexical rich composer
  with typeahead. (backend §7; admin web §2.14)
- **MSG-FR-09a — Mention-candidate directory.** `GET /chat/mention-residents`
  returns the resident candidates for group @mention typeahead, scoped server-side by
  caller role: **STAFF** → residents whose `assignedStaff[]` contains the caller's
  cName; **ADMIN/SUPER_ADMIN** → all active facility residents; **RESIDENT/FAMILY** →
  empty (no DB query). Profile pictures are returned as signed URLs. The admin web
  consumes this via `useMentionResidents` (showing all candidates on an empty query).
  (`controllers/chat/mention.controller.ts:20-60`)

- **MSG-FR-10 — Reactions.** One emoji reaction per user per message with
  ADD / REMOVE (toggle) / REPLACE semantics
  (`POST /chat/messages/{id}/reactions`); responses expose emoji, count, reactor
  list, and `iReacted`; admin web applies optimistic local computation. (backend §7;
  admin web §2.14)
- **MSG-FR-11 — Replies.** A message may quote another via `replyToMessageId`; the
  server enriches responses with a `replyTo` context (sender, role, text,
  hasAttachment, mentions). (admin web §2.14)
- **MSG-FR-12 — Delete (sender-only, soft).** Only the sender can delete a message;
  deletion produces a tombstone (`deleted: true`, content cleared **to the caller only — see
  the retention-policy correction below**) rather than removal. **Correction (2026-08-21): the prior
  "eager S3 deletion at soft-delete time" text here was accurate through 2026-07-30 and is now stale —
  retention policy was reversed to fully non-destructive 2026-07-31/08-03 (HIPAA/CA 7-year requirement).
  Content, attachments, and reactions are retained in the document indefinitely; only `deletedAt`/`deletedBy`
  are set, and every read path renders the tombstone by gating on `deletedAt`, not by inferring deletion
  from empty fields.** **When the deleted message was the conversation's `lastMessage`, the inbox preview now retains it as a tombstone (`deleted: true`, text and mentions cleared) rather than going blank** — clients render "message was deleted". A self-heal path (`recomputeDeletedLastMessage`) rebuilds the tombstone from the latest message when `lastMessage` is absent on legacy data (production 2026-06-30). **No edit, no forward.** The staff app renders the same tombstone state with a dedicated icon and distinct sent/received styling (`MessageDeletedIcon`, staging). (backend §7; admin web §2.14; staff app)
- **MSG-FR-13 — System messages.** Group lifecycle events emit SYSTEM messages
  rendered as centered activity rows: GROUP_CREATED, MEMBER_ADDED, MEMBER_REMOVED,
  MEMBER_LEFT, GROUP_NAME_CHANGED, GROUP_PICTURE_CHANGED, ADMIN_PROMOTED,
  ADMIN_DEMOTED. As of staging the staff app also consumes the `chat:group` socket
  event to render these system messages on its Group Conversation/Details screens
  (previously an admin-web-only rendering; see §5.1). (backend §7; admin web §2.14; staff app)
- **MSG-FR-14 — History pagination.** Message history is cursor-paginated
  (`GET /chat/conversations/{id}/messages?limit&cursor`, 30/page in admin web,
  newest-first reversed for display; infinite scroll). The staff app's conversation
  **list** (`MessageListPage`) is likewise now server-side paginated as of staging,
  replacing a prior fetch-all pattern. (admin web §2.14; SN app §3.3; staff app)

### Delivery semantics & unread

- **MSG-FR-15 — Status ladder.** Every message carries an aggregate status
  `SENT → DELIVERED → READ` plus per-member `deliveredTo[] {cName, deliveredAt}` and
  `readBy[] {cName, readAt}` receipts. In GROUP conversations the aggregate flips only
  when **all non-senders** have acknowledged that level. Clients apply status events
  **upgrade-only** (rank SENT < DELIVERED < READ). (backend §7; admin web §2.14)
- **MSG-FR-15a — Monotonic status ratchet (production hardening, 2026-07-10).**
  A guard (`earlierMessageStatuses()`, `src/constants/chat.ts`) prevents concurrent
  delivery/read acknowledgements — e.g. a reconnect-sweep receipt racing a live socket
  event — from ever regressing a message's recorded status. This closed a class of
  bug where out-of-order acks could flip a message backward from READ to DELIVERED.
  (`services/chat/message/status.service.ts`, backend HEAD `ca628bc8`)
- **MSG-FR-15b — Membership-aware group delivery/read quorum (production hardening, 2026-07-10).**
  The aggregate DELIVERED/READ status for a GROUP message is now computed against
  the intersection of `Message.recipients[]` (a new field capturing who the message
  was actually addressed to at send time, +2 supporting indexes) **and the
  conversation's current participant list**, via `recomputeConversationAggregates`.
  Consequence: a member who has since **left the group** no longer blocks the
  aggregate status of messages sent before they left — the group status ladder now
  reflects only currently-present members. (`models/Message.model.ts`,
  `services/chat/message/status.service.ts`; backend HEAD `ca628bc8`)
- **MSG-FR-16 — Receipt UI.** Tick rendering: ✓ sent, ✓✓ muted delivered, ✓✓ colored
  read — implemented on admin web, the SN resident app, and (staging) the staff app.
  (admin web §2.14; SN app §3.3; staff app)
- **MSG-FR-17 — Unread counts.** Bulk mark-read per conversation via
  `PUT /chat/conversations/{id}/read`. Admin sidebar badge counts **distinct
  conversations** with `unreadCount > 0`, not message totals. **As of staging the
  per-user counts moved off the `Conversation.unreadCounts` map into a dedicated
  per-member `ConversationMemberState` collection** (MSG-FR-29) — `unreadCount`,
  `mediaCount`, and `fileCount` are now maintained per `{conversationId, cName}`
  row alongside that member's clear-floor, and the old embedded `unreadCounts` map
  was retired. (backend §7; admin web §2.14)
- **MSG-FR-17a — Batched unread-badge aggregation (production hardening, 2026-07-10).**
  Unread-badge computation for push/notification purposes is now batched per
  recipient (`getChatUnreadSummaryForUsers`), replacing prior per-event badge
  computation; the same batched summary feeds both the FCM push payload and the
  Web Push payload so mobile and browser badge counts derive from one computation.
  (`services/chat/conversation/unread.service.ts`, `chatNotification.service.ts`
  rewritten to 329 lines; backend HEAD `ca628bc8`)
- **MSG-FR-18 — Read emission.** Clients emit `chat:delivered` on receipt and
  `chat:read` when the conversation is opened/focused. As of staging these emits
  are additionally **queued while the socket is disconnected** on the staff app
  (`pendingDeliveredIds`/`pendingReadIds`) and flushed on reconnect plus a ~1.5 s
  delayed second pass, to catch receipts that would otherwise be dropped at the
  very start of a connection. (SN app §3.3; admin web §2.14; staff app
  `ChatSocket/index.ts`)

### Group administration

- **MSG-FR-19 — Group CRUD.** Create (`POST /chat/groups`, multipart with optional
  picture), update name/picture (`PUT /chat/groups/{id}`), delete (**creator only**).
  Groups are also destroyed automatically when their last member leaves (same cleanup
  path as delete; see §4.2). (admin web §2.14; backend §7)
- **MSG-FR-20 — Membership.** Add/remove members
  (`POST | DELETE /chat/groups/{id}/members`); each change emits the matching SYSTEM
  message. (admin web §2.14; staff app renders the resulting system message via `chat:group`)
- **MSG-FR-21 — Admin roles & invariants.** Promote/demote admins
  (`POST /chat/groups/{id}/admins`, `DELETE …/admins/{cName}`). Invariants enforced:
  a group always has ≥1 admin; the sole admin cannot leave without first promoting
  another. (backend §7)

### Directory & search

- **MSG-FR-22 — User search.** `GET /chat/search?q&page&limit` returns a role-filterable, **page-based paginated** user directory (default page size 30, `CHAT_SEARCH_DEFAULT_PAGE_SIZE`). Because results merge three independently-queried collections (Resident, Staff, Admin) with no shared cursor key, the service uses an over-fetch-and-slice strategy: each sub-query fetches `page * limit + 1` deterministically-sorted rows (`{name, _id}`) and the concatenated set is sliced to the requested window; the +1 peek yields `hasNextPage` without a separate count query (production 2026-06-30). Access policy applied: residents only see their care team. **Role-keyword queries** (`"staff"`, `"admin"`, `"resident"`) return all users of that role. The admin web wires this via `useInfiniteQuery` with an `IntersectionObserver`-based sentinel for infinite scroll across `NewConversationModal`, `GroupCreationModal`, `AddGroupMembersModal`, and `ShareWithModal`.
  (backend §7; `search.service.ts:386-404`, `:235-242`)
- **MSG-FR-23 — Care-team contacts.** `GET /chat/care-team-contacts` (RESIDENT role
  only) returns the resident's chatable care-team directory — on staging this is the
  resident's **full `assignedStaff[]`** roster (each entry carries its `designation`),
  not a fixed four-role set: `careTeam.service.ts` reads `Resident.assignedStaff`,
  loads those Staff (`cName name designation profilePicture`), and returns them with
  per-contact conversation state (`careTeam.service.ts:116-141`). The SN resident
  app's client `CareTeamRole` type still enumerates the four legacy roles
  (`caseManager|socialWorker|doctor|dietitian`) — a **client-side stale artifact**:
  the backend now returns any assigned designation. (backend §7; SN app §3.3 —
  `services/Chat/type.ts:38-42`)

### Encryption

- **MSG-FR-24 — Encryption at rest.** Each conversation holds a KMS-wrapped data key
  (`dataKey.wrappedKeyB64`, keyVersion). Message text and the inbox `lastMessage`
  preview are stored AES-256-GCM encrypted under that key; keys are cached in-process
  for 5 minutes. Mentions, reactions, and system events are **deliberately stored
  unencrypted** so notification fan-out and badge queries work without KMS
  round-trips. (backend §7)

### Push channels

- **MSG-FR-25 — Browser Web Push (VAPID).** In addition to FCM mobile push, the
  system delivers chat new-message notifications to browsers via the W3C Web Push
  protocol (VAPID). Authenticated users register a browser subscription through
  `POST /api/push/subscribe` (`{ endpoint, keys: { p256dh, auth } }`, upserted on
  `endpoint`) and remove it via `DELETE /api/push/subscribe`; one user may hold
  multiple subscriptions across browsers/devices. Subscriptions are facility-scoped
  (`WebPushSubscription`). VAPID is configured from `VAPID_SUBJECT` /
  `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY`; when absent, web push is silently
  disabled. Stale subscriptions (HTTP 410 Gone) are auto-pruned on delivery failure.
  The web-push payload carries `{ title, body, icon, conversationId, conversationType, messageId }`
  (`messageId` added production 2026-07-10 so the receiving service worker can
  de-duplicate a push notification against a socket event for the same message —
  see MSG-FR-27).
  (`webPush.service.ts:32-50,71-119`; `WebPushSubscription.model.ts:23-43`;
  `webPush.controller.ts:31-72`; `webPush.routes.ts:34-53`; admin registration
  `usePushNotifications.ts:62-117`)
- **MSG-FR-26 — Push fan-out split (FCM offline-only, web push all recipients).**
  FCM mobile push is sent **only to offline recipients** (online users already have
  the socket event; a redundant mobile push is disruptive). Web push is sent to
  **all** recipients regardless of online status, to cover Socket.io's ~20 s presence
  lag after a browser closes (the ping-timeout window), during which a
  recently-disconnected browser would otherwise miss the notification. The browser
  service worker deduplicates: when the socket event arrives it closes the OS
  notification immediately. Both payloads carry `conversationType` (`DIRECT | GROUP`)
  so the client can route the tap without a follow-up fetch. The FCM/APNs payload
  additionally carries **`mutableContent: true`** (production 2026-07-02, closing a
  prior gap where iOS rich-notification processing — a required precondition for any
  future notification-service-extension work — was not requested).
  (`chatNotification.service.ts:188-279`; `chat.controller.ts:230,255-264`; SW dedup
  `useGlobalChatSocket.tsx:308-336`)
- **MSG-FR-27 — Admin web installable Chat PWA ("Shashi Messaging", staging, admin production HEAD `324840a`, 2026-07-10).**
  The admin web's Messages window is now an installable Progressive Web App, built
  via `vite-plugin-pwa` (`injectManifest` strategy, workbox) and **deliberately
  scoped to `/messages`** (not the app root) so the rest of the admin panel does not
  falsely present as "installable". `public/sw.js` precaches the static app shell
  only (JS/CSS/HTML/images — no PHI or message content is cached), and implements:
  - `install`/`activate` lifecycle handlers (`skipWaiting`/`clients.claim`);
  - a `message`-relay handler that forwards `CHAT_BADGE_UPDATE` events to
    `navigator.setAppBadge()` / `clearAppBadge()` for the OS app-icon badge;
  - a `push` handler with **`messageId`-keyed deduplication** (60 s TTL, MSG-FR-25)
    and **OS-focus-based suppression** (the notification is suppressed if a client
    reports `focused === true`, not merely "visible" — closing false-positive
    suppression when a window is open but not foregrounded);
  - a `notificationclick` handler with three branches: an open chat popup is
    focused and routed; an open main window is focused, routed, and shown a toast;
    if nothing is open, `clients.openWindow('/messages?convId=...')` opens a new
    window directly at the conversation.
  `public/manifest.json` (`start_url`/`scope` both `/messages`) carries the final
  app name **"Shashi Messaging"** (the manifest churned through an intermediate
  "SAL Chat" naming during rollout — verify `public/manifest.json` directly rather
  than trusting commit-message text if re-checking this). **Icon rebrand (pre-production):**
  the `/messages` PWA icons + favicon/apple-touch-icon were switched from the old
  `messages-icon-*` assets to **Shashi Care** branding (`public/shashi_care_logo_*`,
  injected only on `/messages`) — the installable app now presents as Shashi Care visually
  even though the manifest `name` string remains "Shashi Messaging". `index.html` injects the
  manifest `<link>` only on `/messages` paths and registers a `beforeinstallprompt`
  suppressor before the main bundle parses. **Design gap (open):** because the
  service worker registers — and the Web Push subscription is created — only inside
  `MessagesWindowShell.tsx` (i.e. only once a user opens the Messages window at
  least once), a staff/admin user who has never opened Messages gets **no** Web
  Push subscription and **no** OS app-icon badge, ever; nothing in onboarding
  prompts a first open. See §9 observation 12.
  (`vite.config.ts`, `public/sw.js`, `public/manifest.json`, `index.html`,
  `src/hooks/chat/usePushNotifications.ts`, `src/hooks/chat/useAppBadge.ts`)
- **MSG-FR-28 — Cross-window sync via BroadcastChannel (staging, admin production HEAD `324840a`).**
  The admin web's Main-window ↔ Messages-window state sync (unread counts, active
  conversation, focus state) moved from a `window.opener` reference to a same-origin
  `BroadcastChannel` (`sal-chat-window-sync`) for the Messages→Main direction
  (`CONVERSATION_READ`, `CONVERSATION_UNREAD_BUMP`, `CONVERSATIONS_INVALIDATE`,
  `CHAT_WINDOW_FOCUS_CHANGED`, `USER_ACTIVITY`, and a new `ACTIVE_CONVERSATION_CHANGED`).
  The Main→Messages direction (`OPEN_CONVERSATION`, `PARENT_LOGGED_OUT`) remains a
  targeted `window.postMessage`. This fixes three concrete failure modes where
  `window.opener` was `null` or wrong: installed-PWA launches (no opener at all),
  hand-typed `/messages` URLs, and a second independently opened admin tab. A
  separate, previously undocumented **third** channel — `src/services/swMessages.ts`
  — carries the service-worker↔page contract (`CHAT_NOTIFICATION_CLICK`) over
  `navigator.serviceWorker`'s message channel, distinct from the window `message`
  event. (`src/services/chatWindowMessages.ts`, `src/services/swMessages.ts`)

### Per-user conversation clearing

- **MSG-FR-29 — Per-user Clear Chat / Delete Conversation (staging + pre-production).**
  A single endpoint `PUT /chat/conversations/:id/clear` (any authenticated participant;
  `clearConversationHandler` → `chat/conversation.controller.ts`) lets a user wipe a
  conversation **from their own view only**. The client labels it **"Delete
  Conversation"** on DIRECT threads and **"Clear Chat"** on GROUP threads; the backend
  behaviour is identical — permanent for that user, no undo, and **invisible to the
  other participants** (their history is untouched). This is distinct from the
  sender-only, everyone-visible message *tombstone* of MSG-FR-12.
  - **Model.** A new per-member collection **`ConversationMemberState`** (one row per
    `{conversationId, cName}`, unique index) holds the clear state as a **read-floor +
    soft cursor**, not a hard delete of message documents: `clearedAt`,
    `clearedThroughMessageId` (messages with `_id ≤` this cursor are hidden **from this
    member only**), plus the per-user `unreadCount` / `mediaCount` / `fileCount`
    (reset to 0 on clear, then maintained forward — MSG-FR-17). Core logic:
    `services/chat/conversation/memberState.service.ts` → `clearConversationForUser`.
  - **Read consistency.** Message listing, the attachments gallery, the conversation
    info panel, and unread computation all filter against the caller's floor, so a
    cleared thread reappears empty and rebuilds only from messages sent after the
    clear. A **receipt-integrity fix** ensures `markConversationRead` /
    `markConversationDelivered` / the reconnect sweep never retroactively mark
    messages below a user's clear floor.
  - **Multi-device sync.** A new socket event `chat:cleared` (`CONVERSATION_CLEARED`,
    `emitConversationCleared`) is emitted to the **clearing user's own device room**
    so their other logged-in devices drop the thread too. No event reaches the other
    participants.
  (`senior_living_backend` `chat.routes.ts`, `ConversationMemberState.model.ts`,
  `memberState.service.ts`, `chatNotification.service.ts`; `senior_living_admin`
  conversation-list Clear/Delete action)

- **MSG-FR-30 — Admin message-info (delivery/read) panel (pre-production).** The admin web
  gained a per-message **Message Info** panel exposing the delivery ladder detail that
  the tick UI (MSG-FR-16) only summarizes: per-recipient **Delivered / Read** lists
  with timestamps, surfaced through tabbed views and an **"All"** tab that renders a
  **flat, alphabetical recipient list** (rather than grouping) for group messages. The
  panel resolves `@mentions` to display names in its message preview. This is an
  admin-web presentation surface over the existing `deliveredTo[]` / `readBy[]`
  per-member arrays (MSG-FR-15) and the unified message-options menu — no new backend
  contract. (`senior_living_admin` chat message-info panel / message-options menu)

### Message editing, forwarding, pinning & drafts (v1.6, admin web only)

- **MSG-FR-31 — Message editing (sender-only, admin web).** The sender of a text message can edit
  it in place from the message-options menu; the server returns the updated `text`/`mentions`
  and an `editedAt` timestamp (`useEditMessage`, `PUT` — response shape `EditMessageResponse`).
  Edited messages are visually distinguished from the sender-only soft-delete tombstone of
  MSG-FR-12 — editing changes content in place, deletion clears it. No edit history/audit trail is
  surfaced to other participants beyond the current text + `editedAt`.
- **MSG-FR-32 — Message forwarding (multi-message, multi-target, admin web).** One or more
  selected messages can be forwarded to one or more targets — existing conversations or new
  `{cName, role}` recipients — in a single action (`ForwardTargetPicker.tsx`,
  `POST /chat/messages/forward { messageIds[], targets[] }`). Forwarded attachments are referenced
  server-side, never re-uploaded from the client. The response reports a per-target result
  (`OK | ERROR`, plus the created/updated conversation and the forwarded messages) so a caller can
  tell which targets succeeded. **Optimistic UI with scoped rollback:** each target gets its own
  optimistic placeholder message(s) and, for a brand-new DM target, a placeholder conversation-list
  entry; a per-target `ERROR` result rolls back only that target's optimistic entries, while a
  total request failure rolls back everything (`useForwardFlow.ts` `rollbackAll`). This is the
  first forward capability in the module — previously "no forward" was a documented ceiling
  (§4.2, §9 item 7 — now resolved for admin web).
- **MSG-FR-33 — Per-user conversation pinning (admin web).** A user can pin/unpin a conversation
  for **their own view only** (never visible to or shared with other participants) — a personal
  display preference, distinct from any conversation-level state. Pinned conversations sort above
  every unpinned one (`usePinConversation.ts`, `applyConversationPinChanged`). Optimistic: the list
  re-sorts immediately on click; on request failure only that conversation's `pinned` flag is
  rolled back (not a full list snapshot), so an unrelated real-time update that lands mid-request
  is never silently reverted.
- **MSG-FR-34 — LocalStorage message drafts (admin web).** Unsent composer text (+ mentions) is
  persisted to `localStorage` per conversation (`SAL_CHAT_DRAFTS` key, `chatDrafts.ts`), saved on
  conversation switch and on `beforeunload`, so a draft survives a full browser tab/window close
  and reopen. A draft's `savedAt` timestamp is compared against the conversation's real
  `lastMessage.sentAt` to decide whether the draft or the actual last message wins the
  conversation-list preview. Deliberately **not** synced live across open browser windows/tabs
  (unlike other cross-window chat state, MSG-FR-28) — a draft saved in one window becomes visible
  in another only on that window's next conversations fetch.
- **MSG-FR-35 — Copy Text & Jump to Present (admin web, UI polish).** The message-options menu
  gained a **Copy Text** action, available on any message with text content (parses the same
  mention-sentinel format as reply quotes, MSG-FR-11). A floating **Jump to Present** button
  appears over the message list when the user has scrolled up, and scrolls to the latest message
  on click — the message list itself is deliberately **not** auto-scrolled on new-message arrival
  while scrolled up; the button's badge count is derived independently (`ConversationView.tsx`).

### Message pinning (backend + admin web only, merged 2026-08-25)

- **MSG-FR-43 — Pin/unpin a message to a conversation-wide shared tray.** Any active participant
  (resident, staff, or admin) can pin or unpin any message in a conversation they belong to, from
  the message-options menu (`PUT`/`DELETE /chat/messages/:messageId/pin`); pinning an
  already-pinned message, or unpinning an already-unpinned one, is a no-op success (idempotent).
  This is a **different capability from MSG-FR-33's per-user conversation pinning** — a pinned
  **message** is visible to, and un-pinnable by, every participant, not just the person who pinned
  it. A message is automatically unpinned if it is later deleted (MSG-FR-12). Listing the tray
  (`GET /chat/conversations/:id/pinned-messages`) requires **active** participation — an ex-member
  is denied, stricter than ordinary message history (which stays readable up to an ex-member's exit
  boundary). (`pin.controller.ts`, `pin.service.ts`, `ConversationPinState.model.ts`)
- **MSG-FR-43a — Two independent, facility-configurable pin limits, enforced together.** A
  conversation's pinned-message tray is bounded by two caps, checked atomically in the same
  request: how many messages **one participant** may have pinned at once in that conversation
  (`maxPinnedMessagesPerUser`, default 5), and how many the conversation may hold **in total across
  every pinner** (`maxPinnedMessagesPerConversation`, default 5, facility-configurable up to 100).
  Hitting either cap rejects the pin attempt with a distinct, deliberately number-free message —
  "You've pinned the maximum number of messages here. Unpin one of yours first" (own cap) vs. "This
  conversation has the maximum number of pinned messages. Unpin one first" (conversation cap),
  since the two failures call for different remedies. The admin web pre-checks both caps
  client-side and shows the same wording — dimmed but still clickable — before the user even
  attempts the action, kept word-for-word in sync with the backend's own rejection copy.
- **MSG-FR-43b — Configurable pin duration, including "pins forever."** When pinning, the user
  picks a duration from a facility-configured list (`pinMessageDurationOptions`; defaults 1 hour /
  7 hours / 24 hours / 7 days / **forever**); omitting a duration pins forever if that option is
  offered. An expired pin immediately stops counting against both caps and stops appearing in the
  tray on the next read of, or interaction with, that conversation (an unpin, or opening the
  pinned-messages panel); a daily background sweep is a safety net for conversations nobody
  interacts with again, so a fully dormant conversation's expired pin is cleared within ~24h at the
  latest. No push/socket event announces an expiry on its own — the admin web additionally hides
  already-expired entries client-side within a few seconds via a locally-ticking clock, so the
  banner, panel, and bubble indicator don't keep showing an expired pin as active until the next
  network refetch.
- **MSG-FR-43c — Pinned-message banner and full pinned-messages panel (admin web).** A sticky
  banner below the conversation header shows the single most-recently-pinned, not-yet-expired
  message; clicking it jumps to that message, and repeated clicks advance through the remaining
  pins. A "show all pinned" panel lists every currently-pinned message (who pinned it, a preview,
  and a direct unpin action), rendered as a side panel alongside the conversation — the same
  pattern as the existing Message Info / Group Info panels, not an overlay. A pinned message also
  gets a compact "Pinned" row on its own bubble, matching the existing "Forwarded" row treatment.
- **MSG-FR-43d — Jump-to-message (new, general-purpose, admin web).** Clicking the pin banner or a
  panel entry scrolls the conversation to that specific message, fetching it first if it isn't
  already loaded — a new capability distinct from MSG-FR-35's "Jump to Present," which only ever
  targets the newest message. Built for message pinning but written as a reusable engine, not
  pin-specific.
- **A SYSTEM message announces a manual pin or unpin** ("{name} pinned a message" / "{name}
  unpinned a message") — but **only** for a genuine, manually-triggered state change; an expiry
  sweep or a delete-triggered auto-unpin stays silent, matching the module's existing system-event
  convention (MSG-FR-13) of narrating only actor-driven changes.
- **Real-time sync**: a new socket event, `chat:message-pin-changed`, broadcasts to **every**
  participant on pin, unpin, or expiry (contrast MSG-FR-33's conversation-pin event, which reaches
  only the acting user's own devices) — no push notification, matching the existing delete/edit
  precedent that there is nothing actionable for an already-offline user.
- **Admin web only — not yet at parity with per-user conversation pinning.** Like message editing
  (MSG-FR-31), message pinning ships on admin web only in this pass; the SN resident app and staff
  app have no message-pinning surface at all (contrast the staff app's own, differently-mechanised
  **conversation** pin, MSG-FR-41). A parity gap, not a defect — see also §9 item 16 for the
  related, still-unresolved scroll-to-bottom reliability caveat that landed in the same commit
  range.

### Staff-app chat expansion (2026-08-21, `senior_living_staffapp` `origin/master` HEAD `35b3cc82`)

Built independently of, and largely in parallel with, the admin-web "v2.7" work above. First formal
PRD-lineage pass on this repo's chat surface beyond the 2026-07-12 correction.

- **MSG-FR-40 — Message forward (staff app).** `ForwardMessageScreen` + `ForwardActionBar` let a
  user select 1+ messages and forward them (with attachments) to 1+ destination conversations,
  grouped into "Recent Chats" and "People" pickers. **This merged into `master` between 2026-08-04
  and 08-08 — chronologically ahead of the admin-web forward feature (MSG-FR-32, merged
  2026-08-10).** Whether the two clients call the same `POST /chat/messages/forward` backend
  contract documented under MSG-FR-32, or a separate/earlier mechanism, was **not confirmed this
  pass** — flag for a backend-contract cross-check before assuming shared behavior (e.g. the
  per-target optimistic-rollback semantics MSG-FR-32 describes for admin web). This corrects §4.2
  and §9 item 7's "no forward … not available on … the staff app" framing — it is, and arguably
  shipped first.
- **MSG-FR-41 — Conversation actions: pin/unpin, mark-as-read, leave-group (staff app).** A
  long-press `ConversationActionMenu` adds **pin/unpin** (synced across a user's own devices, capped
  by a facility-config max-pins value — a different mechanism from the admin web's local-only,
  non-shared pin preference in MSG-FR-33; whether the two share a backend field was **not confirmed
  this pass**), **mark-as-read**, and **leave-group** (self-service; existing group-membership
  removal was previously admin/creator-only from the client's perspective — see MSG-FR-19/20/21) on
  top of the pre-existing sender-only delete (MSG-FR-12).
- **MSG-FR-41a — Per-conversation drafts, Keychain-backed (staff app).** Unsent composer text is
  persisted to the device Keychain (not AsyncStorage) because a draft can contain resident PHI via
  @-mentions — one Keychain entry holds a JSON map of all per-conversation drafts, debounced 2 s
  while typing, flushed immediately on screen-blur/app-background. **Client-only — no server
  endpoint backs staff-app drafts**, unlike the admin web's `localStorage` equivalent (MSG-FR-34),
  which is likewise client-only but browser-scoped rather than Keychain-scoped.
- **MSG-FR-42 — Per-recipient Message Info sheet (staff app).** A `MessageInfoSheet` shows a
  per-recipient read/delivered/sent breakdown with ALL/READ/DELIVERED/SENT tabs — the staff app's
  own presentation surface over the same `deliveredTo[]`/`readBy[]` per-member arrays (MSG-FR-15)
  that back the admin web's Message Info panel (MSG-FR-30); no new backend contract.
- **Video player rewrite (staff app, no FR number — presentation only).** A new `VideoPlayer`
  component adds orientation support (`react-native-orientation-locker`) and swipe-to-dismiss to
  chat video attachments, with an inline download-progress ring on `ChatBubble`.
- **⚠ Offline message sync did NOT ship.** A `feat/offline-message-sync` branch (Realm-backed local
  chat cache + a send outbox) was built but its commits are **not ancestors of `origin/master`** —
  verified via `git merge-base --is-ancestor`. **Chat on the staff app remains fully online-only.**
  If any planning artifact assumed offline chat shipped on this client, this is the correction.

### Automated system messages (v1.6, backend)

- **MSG-FR-36 — Module-authored automated messages (`ChatSystemUser`).** A backend module can post an
  automated message into a facility's chat group under a dedicated non-human identity
  (`ChatSystemUser` — facility-scoped, `cName`/`name`/`profilePicture`, no auth capability, no
  operational fields) via `sendModuleMessage()`. Routing is config-driven per facility:
  `Config.chat.moduleMessageBindings` maps a `ModuleKey` (e.g. `'transportation'`) to one or more
  `{ conversationId, chatSystemUserId, messageNotifyPreference? }` bindings — one module can post into
  several groups, or share one identity across modules. `messageNotifyPreference` is a per-event
  **opt-out** map (an event key absent from it is enabled by default). The send reuses the human-send
  core (`sendMessage()`: encryption, mention validation, persistence, receipt scaffolding, real-time +
  push fan-out) — messages authored this way are indistinguishable in the delivery pipeline from a
  human message, flagged internally via `isSystemUser`/`skipParticipantCheck` only. `sendModuleMessage`
  never throws; a chat-automation failure is caught and logged, never surfaced to or blocking the
  calling module's own API response. **Transportation is the first and (as of this pass) only consumer**
  — one shared card template renders a full-state summary per ride-lifecycle event (create/assign/
  start/arrive/etc.), with its own `resolveMentions` per template (see transportation.md). **No creation
  API exists for `ChatSystemUser` documents** — provisioning a facility onto module chat automation is a
  manual direct-write operation today (§9).

---

## 4. Business rules & policies

### 4.1 Initiation access policy (`chatAccessPolicy.ts` — single source of truth)

Gating applies to **conversation initiation only**. Existing participants keep access
even if the policy later tightens — intentional, no retroactive revocation; tightening
is enforced going forward via search/directory visibility. (backend §7)

| # | Rule |
|---|---|
| 1 | Self-chat denied. |
| 2 | **RESIDENT ↔ RESIDENT always denied** (hard rule, not configurable). |
| 3 | Per-facility config flags `chat.isResidentAllowed`, `chat.isAdminAllowed` (defaults permissive; config cached 30 s). |
| 4 | **RESIDENT → STAFF**: the target staff cName must appear in the resident's **`assignedStaff[]`** array (any designation — the five fixed `nurse/caseManager/socialWorker/doctor/dietitian` fields were removed on staging) **and** the target's designation must pass `chat.staffDesignationAllowed` (empty list / `"*"` = unrestricted). (`chatAccessPolicy.ts:305-320`) |
| 5 | **STAFF → RESIDENT**: initiator's designation must pass the allowlist **and** the initiator's cName must appear in the target resident's `assignedStaff[]`; there is no longer a designation→field branch (any assigned staff may reach their assigned resident). (`chatAccessPolicy.ts:324-345`) |
| 6 | **STAFF → STAFF**: both designations must pass the allowlist. |
| 7 | **ADMIN reachability** (`isAdminAllowed`, default true): when true, admins appear in search results for **STAFF and ADMIN** callers, and any **non-RESIDENT** user may initiate a conversation with an admin (and vice versa); both parties must be active. When false, admins are hidden from all search and no new admin-involving conversation may be initiated (existing conversations unaffected). STAFF callers requesting admins still pass the `staffDesignationAllowed` caller-gate. *(Previously only ADMIN callers could find other admins; STAFF could not reach admins — `chatAccessPolicy.ts:102-115`, `search.service.ts:334-336,357-368`.)* |

### 4.2 Other rules

- **Groups are staff/admin-only constructs** — residents cannot create, manage, or
  administer groups; route-level requireAnyRole STAFF | ADMIN | SUPER_ADMIN on all
  group management endpoints. Residents can be group *members*. (backend §7)
- **Zero-member group auto-destruction — corrected 2026-08-21, no longer physically destructive.**
  When the last participant leaves a group (`leaveGroup` reduces participants to 0), the system
  fire-and-forget soft-destroys the group (`softDeleteGroupData`, renamed from the prior
  `destroyGroupData`): it sets `deletedAt`/`deletedBy` on the `Conversation` document only. As of
  2026-08-03 (retention-policy reversal, HIPAA/CA 7-year requirement — see the chat v2 note in §1),
  **no message, no `Conversation` document, no group picture, and no S3 attachment is ever removed**
  — every read path that loads or lists conversations filters `deletedAt: null`, reproducing the old
  "vanishes" observable behavior without destroying data. This is the same soft-destruction path used
  by creator-initiated `deleteGroup`. A pre-existing bug fixed as part of the same change: a destroyed
  group's `ConversationMemberState.unreadCount` is now cleared (previously stayed permanently inflated
  for the departed member). (`group.service.ts`)
- **Family-as-resident**: a family member's messages are attributed to the resident
  identity (username normalization); audit granularity below the resident level
  depends on `familyMemberCName` capture upstream. (backend §0)
- **Multi-tenancy**: all chat routes pass `facilityMiddleware`; conversations are
  facility-scoped (`x-facility-id` header; directPairKey is per-facility). (backend §0, §7)
- **No full-text message search** (search is conversation-level only: name/last-message).
  Message **edit exists on admin web only** (MSG-FR-31, v1.6); **forward exists on both
  admin web (MSG-FR-32, v1.6) and the staff app (MSG-FR-40, independently built, merged
  chronologically first)** — see §9 item 7 for the residual edit-parity gap. (admin web §2.14)
- Attachment count denormalization and eager S3 cleanup on delete are part of the
  storage contract (MSG-FR-06, MSG-FR-12).
- **Status ratchet is upgrade-only everywhere** (MSG-FR-15a) — no client or server
  path may write a status regression once a higher status has been recorded.

---

## 5. Notifications & real-time behavior

### 5.1 Socket event catalog (Socket.io namespace `/chat`)

Auth: Bearer token + `x-facility-id` header on the socket handshake (SN app §3.3).

| Event | Direction | Meaning |
|---|---|---|
| `chat:new` | server → client | New message in a conversation the user participates in |
| `chat:delivered` | client → server | Delivery ack for received message(s) |
| `chat:read` | client → server | Read ack (conversation open/focus) |
| `chat:status` | server → client | Status upgrade fan-out (DELIVERED/READ) to other participants |
| `chat:group` | server → client | Group lifecycle change (meta/membership/admin) — now also consumed by the staff app (§2.2) |
| `chat:deleted` | server → client | Message tombstoned — now also consumed by the staff app (§2.2) |
| `chat:reaction` | server → client | Reaction added/removed/replaced |
| `chat:unread` | server → client | **Authoritative total-unread snapshot pushed on connect/reconnect** |
| `chat:cleared` | server → client | Conversation cleared by this user — emitted only to the **clearing user's own device room** for multi-device sync (MSG-FR-29); other participants receive nothing |
| `chat:message-pin-changed` | server → client | A message's pin state changed conversation-wide (pin/unpin/expiry) — broadcast to every participant, not just the acting user (contrast the conversation-level `chat:pin-changed` implied by MSG-FR-33, which reaches only the acting user's own devices); MSG-FR-43 |
| `chat:error` | server → client | Error channel |

(backend §7; admin web §2.14 — admin web consumes all of the above; SN app consumes
`chat:new` + `chat:status` and emits `chat:delivered` / `chat:read`; the staff app now
also consumes `chat:group` and `chat:deleted` as of `master@4aa3849`.)

### 5.2 Reconnect behavior

- Admin web maintains a **singleton** socket with dynamic auth-token injection
  (token rotates on reconnect), exponential backoff 1 s → 10 s, and proactive Cognito
  token refresh on `connect_error`. The `chat:unread` snapshot on connect/reconnect is
  authoritative — clients reconcile badges from it rather than from replayed events.
  Status events for messages not yet loaded are buffered (`pendingStatusRef`) and
  applied after load. (admin web §2.14)
- SN app mounts the socket lifecycle app-wide (`useChatSocketLifecycle`). (SN app §3.3)
- The staff app now uses `forceNew: true` on socket (re)connection and explicitly
  tears down stale sockets, queuing `chat:delivered`/`chat:read` emits while
  disconnected and flushing them (twice — immediately on connect, then again ~1.5 s
  later) to catch receipts otherwise dropped at the start of a connection (MSG-FR-18,
  staging).

### 5.3 Push & local notifications

- **FCM push (mobile)** for chat new-messages via `chatNotification.service.ts`, sent
  to **offline** recipients only (MSG-FR-26). Payload: `notification: { title, body }`
  + `data: { type: "CHAT_MESSAGE", conversationId, messageId, conversationType }`. The Android FCM channel ID is **`chat_messages_v2`** (updated production 2026-07-02 to ensure ring/alert behavior on Android; the previous channel ID was `chat_messages`). Mobile push notifications are configured to **ring** for new chat messages (production 2026-06-30). The staff app additionally plays a **custom notification sound** (`chat_notification.mp3`) on its own `chat_messages_v2` channel, plus distinct in-app sent/received audio cues (`message_sent.mp3`/`message_receive.mp3`) via `react-native-sound` (staging). (`chatNotification.service.ts:229-254`)
- **Web push (browser)** to all recipients via VAPID (MSG-FR-25); payload
  `{ title, body, icon: "/logo.png", conversationId, conversationType, messageId }`. Delivered through the admin's installable-PWA service worker with `messageId` dedup and focus-based suppression (MSG-FR-27).
- **Standardized notification content** across both channels: title = group name for
  GROUP, sender name for DIRECT; body = `"{Sender} sent a message"` /
  `"{Sender} sent an attachment"` for GROUP, `"Sent you a message"` /
  `"Sent an attachment"` for DIRECT. (`chatNotification.service.ts:229-237`)
- **SN app**: messages arriving via socket for a *non-active* conversation raise a
  **local** notification (notifee, channel `chat_notifications`); tapping
  deep-navigates to the conversation — **foreground only**. No background/quit-state
  handler (`onBackgroundEvent` / `getInitialNotification`) and no URL-scheme config —
  cold-start chat deep links do not work. (SN app §4)
- **Staff app**: local chat notifications now render a deleted-message state and a
  custom sound as above; delivery/read acks are reconnect-resilient (§5.2). Deep-link
  / cold-start tap-routing behavior for the staff app's chat push was not
  independently re-verified in the 2026-07-12 pass — treat as unconfirmed rather than
  assuming parity with the SN app's foreground-only limitation.
- **Admin web**: global socket callbacks drive sidebar badges and toasts;
  the in-app NotificationPanel is a separate, non-persistent feed. The Messages
  window's own push/badge wiring is now scoped to the installable PWA (MSG-FR-27),
  distinct from the admin shell's general notification bell. (admin web §2.14, §2.14b)

---

## 6. Integrations

| Integration | Use |
|---|---|
| **AWS KMS** | Per-conversation envelope encryption: wrapped data key on the conversation, AES-256-GCM for message text + inbox preview, 5-min key cache. One of three distinct KMS envelope schemes in the backend (others: PCC medications per-resident, RehabMessage per-field) — schemes are not unified. (backend §7, §9) |
| **AWS S3** | Attachment storage; signed URLs on read; group pictures (multipart upload). **No object deletion anywhere in this module as of the 2026-07-31/08-03 retention-policy reversal** (see §1, §4.2) — message/group soft-delete no longer removes attachments, and replacing a group picture no longer deletes the previous one either. (backend §7; admin web §2.14) |
| **FCM (Firebase)** | Mobile push to conversation participants (`chatNotification.service.ts`); **offline-only** for chat (MSG-FR-26). SN app sends its FCM token implicitly on every API request (`pushToken` / `x-fcm-token` headers) — no explicit device-registration endpoint. (backend §9; SN app §4) |
| **Web Push (VAPID)** | Browser push channel via the W3C Web Push protocol, mounted at `/api/push` (`webPush.service.ts`, `WebPushSubscription`). Fires to **all** recipients to cover the socket-disconnect presence-lag window (MSG-FR-25/26). Configured from `VAPID_*` env vars; silently disabled when absent; 410-Gone subscriptions auto-pruned. Admin web registers via a `/messages`-scoped installable-PWA service worker (`usePushNotifications.ts`, MSG-FR-27). |
| **Socket.io** | `/chat` namespace for all real-time events (§5.1). |
| **Facility config service** | `config.chat` supplies initiation flags, designation allowlist, and attachment limits; 30 s server-side cache; admin web caches `AppConfig` in localStorage with background refetch. (backend §7; admin web §"config") |
| **Chat designation-allowlist management API** | New config endpoints manage `chat.staffDesignationAllowed` per facility: `GET /api/config/chat/staff-designations`, `PUT /api/config/chat/staff-designations`, `DELETE /api/config/chat/staff-designations/{designation}` (authenticated; admin web can update which staff designations may initiate chat without a full config save). Designation aliases also inherit chat access via the `staffDirectoryRoles` parent designation (production 2026-06-24). |
| **`vite-plugin-pwa` / workbox** (admin web build) | Generates the `/messages`-scoped service worker + precache manifest for the installable Chat PWA (MSG-FR-27); `injectManifest` strategy, `maximumFileSizeToCacheInBytes` raised to 6 MB to fit the unsplit main bundle. |

---

## 7. Permissions & access control

- **Initiation policy** is the primary control surface — see §4.1. Enforcement is
  server-side in `chatAccessPolicy.ts`; clients mirror it only through directory
  visibility (residents see only chatable care team).
- **Care-team gating** is bidirectional and now keyed on the unified
  **`Resident.assignedStaff: string[]`** array (staging refactor; the five fixed
  `nurse/caseManager/socialWorker/doctor/dietitian` fields and the
  `DESIGNATION_TO_CARE_TEAM_FIELD` mapping were removed — PLAT-FR-54). Membership in
  that array defines both who a resident can initiate chat with and which residents a
  staff member can reach — for **every** designation (the prior unmapped-designation
  "reach any resident" bypass is gone). (`chatAccessPolicy.ts:305-355`)
- **Group management routes**: requireAnyRole STAFF | ADMIN | SUPER_ADMIN; group
  delete is creator-only; admin invariants per MSG-FR-21.
- **Admin web has no frontend permission gate** on the Messages module — nav-level
  filtering plus backend enforcement only; direct state navigation renders the module
  regardless of page permission (client-admin-web.md observations, "Modules with no
  frontend permission gate"). Acceptable only because the backend policy is the
  real gate.
- **Tenant isolation**: facility middleware on every route; conversation documents and
  pair keys are facility-scoped.
- **No retroactive revocation**: removing a staff member from a care team (or
  tightening the designation allowlist) does not close existing conversations
  (deliberate; §4.1). Product should confirm this is the desired policy for
  offboarding/termination scenarios (§9).

---

## 8. Product-split notes

- **Chat is a platform capability, not an SN-only feature — it now also has an
  active client on the operational (Senior Living-shared) staff app.** The backend
  has no facility-type branching in chat; the SN resident app has the Message tab,
  the SL resident app has none, and the staff app (shared across both products) now
  has a live chat surface for every designation regardless of which product the
  facility runs — client-resident-app-sn.md §6; client-resident-app-sl.md; staff-app
  review 2026-07-12.
- **Admin web is facility-type agnostic** for chat: "everything else (… Chat …) is
  facility-type agnostic in code and differentiated purely by which pages the
  facility-config API enables" (client-admin-web.md §"product-split").
- The SN app's four-role care-team labelling is now purely a **client-side
  artifact**: the backend care-team model is the unified `Resident.assignedStaff[]`
  array (any designation; the five-field `RESIDENT_CARE_TEAM_FIELDS` model was removed
  on staging — PLAT-FR-54). The contacts endpoint already returns every assigned
  designation; an SL/AL adoption of chat needs no backend change to surface other
  roles.
- If chat is extended to the SL product, the per-facility config flags
  (`isResidentAllowed`, `isAdminAllowed`, `staffDesignationAllowed`) are the intended
  rollout levers — no code split required server-side.
- **Chat client implementations now number at least three actively maintained
  surfaces** (admin web, SN resident app, staff app) rather than "admin web + SN app,
  plus stale ported types in the staff app" — the staff app's chat types were
  originally ported from an earlier iteration and the app has since grown a fully
  live implementation on top of them. Any shared-package extraction should treat all
  three as live sources, not assume the staff app is dormant.
- **Feature parity is diverging again, in the other direction (v1.6/v1.8) — and it is not purely
  admin-web-led.** Where previous gaps were "mobile app X lacks a capability admin web has always
  had" (attachments, reactions, mentions on the SN app — §9 item 2), the picture for editing/
  forwarding/pinning/drafts is now three-way and asymmetric: **editing** is admin-web-only
  (MSG-FR-31); **forwarding** exists independently on **both** admin web (MSG-FR-32) and the staff
  app (MSG-FR-40, which merged first); **pinning** and **drafts** exist independently on both
  surfaces too, but via **different mechanisms** (admin: local/non-shared pin + browser
  `localStorage` drafts; staff app: cross-device-synced pin + Keychain drafts) — whether either
  pair shares a backend contract was not confirmed this pass. The SN resident app has none of
  this. Any shared-package extraction plan should map capability-by-capability, not assume a
  single "web is ahead" or "mobile is ahead" direction.

- **Message pinning (new, 2026-08-25) sharpens this further.** Admin web is now the only surface
  with a **shared, conversation-wide** pinned-message tray (MSG-FR-43). Per-user conversation
  pinning already existed independently on both admin web and the staff app (MSG-FR-33/41); message
  pinning, by contrast, has no staff-app or SN-app counterpart at all in this pass.

---

## 9. Observations & candidate gaps (with evidence refs)

1. **No staff mobile chat surface — RESOLVED, correction dated 2026-07-12.** This
   module previously stated "the staff app has no chat" (client-staff-app.md obs. 2,
   "clinical half is vapor"), treating it as the single most consequential gap. As of
   `senior_living_staffapp` `master@4aa3849` (2026-07-11), this is **no longer
   accurate**: the staff app has a live 1:1 + GROUP chat surface for every
   designation, under active production hardening (delivery/read-receipt
   reliability, delete, group events, custom notification sound, pagination — §2.2,
   §5). The original observation may still hold for *other* clinical modules in the
   staff app (medications/labs/etc. — see Clinical Records and Care Coordination
   module docs), but not for chat specifically. Residual: parity of attachments/
   reactions/mentions/replies between the staff app and the admin web composer was
   not independently re-verified this pass.
2. **SN app chat is text-only in practice.** Backend and admin web support
   attachments, reactions, replies, mentions, and groups; the SN resident
   conversation UI implements text + status ticks only (client-resident-app-sn.md
   §3.3 — GROUP "exists in types only"; no attachment/reaction UI described).
   Resident-side feature parity is a product decision, not a backend gap.
3. **Cold-start chat deep links missing on SN app.** Chat local-notification tap
   navigation is foreground-only; no `onBackgroundEvent`, no `getInitialNotification`,
   no URL scheme (client-resident-app-sn.md §4). A resident tapping a chat push with
   the app killed lands on Home. The staff app's equivalent behavior was not
   independently re-verified this pass — do not assume parity either way.
4. **Attachment-limit defaults disagree across analyses.** Backend defaults state
   5 images / 2 videos / 10 total, image 5 MB, video 30 MB (backend §7); admin web
   reads image 5 MB ×10, video 50 MB ×3, audio 10 MB, document 20 MB from
   `config.chat` (client-admin-web.md §2.14). Values are config-driven so both can be
   "true", but the divergence between server defaults and deployed config should be
   reconciled and documented as the canonical product limit.
5. **Mentions/reactions/system events are plaintext by design** (backend §7). This is
   a deliberate encryption carve-out for fan-out/badging — but mention tokens embed
   cNames and group system messages reveal membership history. Confirm this meets the
   product's PHI posture, since chat between a resident and their Doctor/Dietitian is
   plausibly health-adjacent.
6. **No retroactive access revocation** (backend §7 — initiation-only gating).
   Off-boarded staff or care-team reassignment leaves prior conversations readable/
   writable. Candidate: conversation freeze/archival on care-team unassignment or
   staff deactivation.
7. **No message edit, no full-text search; forward now exists independently on two clients —
   corrected 2026-08-21.** Message editing (MSG-FR-31) shipped admin-web-only (2026-08-10) and
   remains a genuine gap on the SN resident app and staff app. **Forwarding is not a gap**: it
   shipped on the staff app first (MSG-FR-40, merged 2026-08-04→08) and independently on admin web
   (MSG-FR-32, merged 2026-08-10) — the prior wording of this item (and the v1.6 changelog entry
   that produced it) incorrectly stated forward was "admin-web only." Residual: forward is still
   absent on the SN resident app, and whether the two existing forward implementations share a
   backend contract is unconfirmed (see the new "Staff-app chat expansion" subsection above).
   Full-text search remains unbuilt everywhere and is additionally constrained by
   encryption-at-rest (would require a searchable-index design decision).
8. **Admin web Messages module has no frontend permission gate** (client-admin-web.md
   observations). Backend policy is the real control; acceptable, but the permission
   matrix UI implies page-level gating that does not exist for Messages.
9. **Family attribution is collapsed into the resident.** Because family members chat
   as the resident (backend §0), staff cannot distinguish "resident typed this" from
   "spouse typed this" in the thread. If SN families gain app access, sender
   sub-attribution (e.g., surface `familyMemberCName`) is a likely requirement.
10. **`MessegeTab` typo** is the SN app's chat tab key (client-resident-app-sn.md §1)
    — cosmetic, but it leaks into navigation analytics.
11. **Unread model differs by surface**: admin sidebar counts unread *conversations*;
    `chat:unread` socket snapshot carries total unread; SN contact list shows
    per-contact badges (admin web §2.14; SN app §3.3). The new batched unread-badge
    aggregation (MSG-FR-17a) unifies the *computation* feeding push payloads but does
    not yet unify the *display* semantics across surfaces. Define the canonical badge
    semantics before adding more surfaces.
12. **New — Admin Messages PWA push/badge blind spot (2026-07-12).** Because the
    service worker and Web Push subscription for the installable Chat PWA (MSG-FR-27)
    register only inside `MessagesWindowShell.tsx`, a staff/admin user who has never
    opened the Messages window gets no push subscription and no OS app-icon badge —
    ever — and nothing in onboarding prompts a first open. The root-scope alternative
    was explicitly rejected in code comments (it would have made the entire admin
    panel falsely "installable"). Recommend a product decision: prompt chat-eligible
    roles to open Messages once on first login, or explicitly accept and document the
    gap. *(admin-web review 2026-07-12, `MessagesWindowShell.tsx`)*
13. **New — PWA naming churn (2026-07-12, informational).** The admin Chat PWA's
    manifest name flipped between "SAL Chat" and "Shashi Messaging" several times
    during the rollout window; the final, currently-shipping name is **"Shashi
    Messaging"** (verified directly against `public/manifest.json` at admin
    production HEAD `324840a`). If documenting this again, re-verify the manifest
    file directly rather than trusting commit-message text, since several commit
    subjects describe a rename in the opposite direction from what the diff actually
    did.
14. **New — `ChatSystemUser` has no provisioning UI or API (2026-08-21).** Onboarding a facility onto
    module-authored automated chat messages (MSG-FR-36) requires a manual database write to create the
    `ChatSystemUser` document and hand-populate `Config.chat.moduleMessageBindings` — there is no admin
    surface or self-service endpoint. Fine for a single-module (Transportation) pilot; a candidate gap if
    more modules adopt this pattern before an admin-facing configuration UI exists.
15. **New — retention-policy reversal has no data-lifecycle counterpart yet (2026-08-21).** The
    2026-07-31/08-03 change makes chat content, attachments, and destroyed-group data permanent by
    default (HIPAA/CA 7-year retention). There is no corresponding purge/archival job for content once
    the 7-year window *does* elapse — "never delete" today, not "delete after 7 years." If the 7-year
    figure is a hard compliance ceiling rather than a floor, a future archival/purge mechanism is a
    candidate follow-up, not yet built.
16. **New — General message-list scroll-to-bottom reliability is a known, in-progress rough edge
    (2026-08-24, admin web).** Independent of the pin/jump-to-message feature itself (MSG-FR-43d),
    a same-window commit landed a partial fix for three previously reported scroll issues: a
    conversation not opening exactly at the bottom (occasionally with visible jitter), a
    just-sent message settling short of the true bottom, and rapid successive sends/arrivals not
    reliably following to the tail. Per the commit's own message, the underlying issue is **"not
    yet fully resolved in all cases"** — a checkpoint, not a completed fix, with further
    investigation continuing in a follow-up. Do not describe scroll-to-bottom as reliable, and do
    not treat this as closing the gap, until a follow-up confirms it.
