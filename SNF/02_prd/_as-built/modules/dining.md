# Module: Dining

> Applies to: **Both** (Senior Living | Skilled Nursing)
> FR prefix: **DIN**
> Sources: `_codebase-analysis/backend-wellness-dining-ops.md` (§2, §0, §7, §8), `client-admin-web.md` (§2.4, §4), `client-resident-app-sl.md` (§3.4), `client-resident-app-sn.md` (§3.7), `client-tv-app.md` (§3.4). Code is the source of truth.
> **2026-08-21 (v1.7) delta — SN resident app first pass.** `DiningScreen` (`senior_living_skillednursing_resident`, `origin/master` HEAD `4734daf`) now sources its menu from the resident-facing tray-card route `GET /api/dining-tray/menu`, **correcting** this doc's prior §3.H claim that `DiningTrayCard`/Menu2U data is "not surfaced in the resident mobile app or TV." New DIN-FR-38; §3.H's business-rule bullet corrected in place.

---

## 1. Purpose & scope

The Dining module lets a facility publish what its kitchen serves and lets residents (and their families) see it everywhere they live — mobile app, in-room TV, and the public daily schedule. It covers five capability clusters:

1. **Menu management** — admin-maintained category → item hierarchy with rich per-item availability scheduling (everyday / one-time / weekly / multiple-dates / date-range, plus per-date overrides) and active/inactive toggles.
2. **Menu library** — a reusable facility gallery of uploaded menu files (images and PDFs) with usage tracking, shared with the platform-wide image-picker pattern.
3. **Daily specials** — scheduled promotional menu files (one poster per date) with the same recurrence vocabulary, a week-at-a-glance admin view, and schedule-from-library reuse.
4. **Diet plans** — per-resident dietary records (diet type, supplements, notes) maintained by staff/dietitians and surfaced read-only in the resident's menu view.
5. **Family meal requests** — the module's only transaction: a resident (or family/staff on their behalf) books guest meals for a date range at configured per-meal rates, subject to blackout dates.

Out of scope here: facility meal time windows as they affect *other* modules (wellness slot blackouts — see §6), the shared gallery's non-dining consumers, and per-item food ordering (does not exist anywhere in the system — all menu surfaces are browse-only).

---

## 2. Personas & surfaces

| Persona | Admin web | Resident mobile (SL) | Resident mobile (SN) | TV app | Staff app |
|---|---|---|---|---|---|
| **Admin / Super-admin** | Full Dining cluster: All Day Menu CRUD, Specials, Menu Library, Family Meal Requests (view + pricing/blackout config), Diet plans (full, incl. delete) | — | — | — | — |
| **Staff (Dining page permission)** | Same Dining pages, gated by per-staff read/write access permissions; receives push on new family-meal requests | — | — | — | No dining surface in staff app analysis |
| **Staff (designation "Dietitian")** | Diet plans scoped to residents assigned via `resident.dietitian`; "my plans" view | — | — | — | — |
| **Resident** | — | Browse menu by date, view daily special, request family meal, view requested meals | Same as SL **plus** own diet-plan cards and meal-request history | Browse 7-day menu, view today's special poster, request family meal (after pairing) | — |
| **Family member** | — | Books on behalf of the linked resident (if facility booking policy allows); receives request notifications | Same | — (TV is resident-paired) | — |
| **Anonymous / unpaired TV** | — | — | — | Browse menu, specials, meal price/time (pre-login catalog routes); transaction blocked behind QR pairing | — |

Capability summary: admins configure and observe; residents/families browse and book; staff with the Dining permission are the notified fulfilment audience; dietitians maintain a clinical-adjacent record that feeds the resident's menu view.

---

## 3. Functional requirements (as-built)

### 3.A Menu management — categories & items

**DIN-FR-01 — Category hierarchy.** The facility menu is a flat list of categories (name, active flag, soft-delete flag, explicit `orderKey` ordering), each containing menu items. Categories and items are facility-scoped and support full CRUD plus dedicated reorder operations (`PATCH /items/reorder`, `/categories/reorder`). *(backend §2.1)*

- Business rule: ordering is explicit via `orderKey`, not insertion order.
- Note: the admin UI's drag-and-drop reordering is commented out — the reorder mutations exist client-side but are never invoked (see §9-G7).

**DIN-FR-02 — Menu items.** An item belongs to one category and carries name (required), description, picture (image **or PDF** — PDFs render as a file icon + "View PDF" in admin), active/inactive state, soft-delete flag, order key, and an availability schedule (DIN-FR-03). Items are created/updated via multipart form when a file is attached; images follow the platform image contract (client sends an S3 `imageKey`, server returns a signed `imageUrl` for display only). *(backend §2.1; admin §2.4)*

**DIN-FR-03 — Item availability model.** Each item declares one `availabilityType`:

| Pattern | Inputs | Item appears on… |
|---|---|---|
| `EVERY_DAY` | none | every date |
| `ONE_TIME` | startDate (required) | exactly that date |
| `WEEKLY` | ≥1 weekday name, plus start+end date window (admin UI requires both) | matching weekdays within the window |
| `MULTIPLE_DATES` | ≥1 explicit date | each listed date |
| `DATE_RANGE` | start + end date | every date in the range |

On top of any pattern, per-date `dateOverrides[{date, isAvailable}]` take **highest priority** — an override can force an item in or out on a specific day regardless of its pattern. *(backend §2.1, `isItemAvailable`)*

- This recurrence vocabulary (one-time / weekly / multiple-dates / date-range / everyday) is a de-facto platform scheduling primitive shared with Specials, Activities, and Announcements. *(admin §6)*

**DIN-FR-04 — Per-item active toggle.** Admin can switch an item active/inactive independently of its availability schedule (`PATCH /items/{id}/toggle`); inactive items never appear in any resident-facing menu. *(admin §2.4)*

**DIN-FR-05 — Date-resolved menu assembly (resident read).** `GET /api/menu` (authenticated) returns, for a single date (default today) or a `startDate..endDate` range, the per-day menu assembled as: ordered categories → only the items whose availability resolves true for that date → **plus exactly one daily special per day** (DIN-FR-12) → **plus**, when the caller is a resident, the flattened entries of their active diet plan (DIN-FR-15). *(backend §2.1)*

**DIN-FR-06 — Admin menu read.** A separate admin read (`GET /menu/getMenuForAdmin`) returns the *unresolved* menu — full availability metadata (`effectiveDays`, `dateOverrides`) for every item plus all active specials — with search, pagination (10/page default in admin UI), and an optional "view menu for date" filter that previews date resolution. **This route has no authentication** (see §9-G3). *(backend §2.1; admin §2.4)*

**DIN-FR-07 — Pre-login meal facts.** `GET /menu/price-and-time` (unauthenticated by design — consumed by TV and mobile before login) exposes the facility's meal time windows, per-meal price, max guest count, concierge phone number, and facility coordinates. *(backend §2.1)*

### 3.B Menu library (reusable file gallery)

**DIN-FR-08 — Library entries.** The menu library is a facility-wide list of uploaded menu files: file name, type (`pdf | image`), S3 key + signed URL, upload date, **usage count**, active flag, and optional weekday tags. CRUD plus an explicit `increment-usage` operation. No route in the library carries authentication (see §9-G3). *(backend §2.2)*

**DIN-FR-09 — Shared picker & save-to-gallery.** Admin file selection uses the platform-wide `ImageSelectionModal`: either upload from computer (with a "Save to Gallery" checkbox, persisted as the misspelled `isSavedToGallary` flag) or pick from the existing gallery (per-module folders, e.g. `items`, `daily-specials`). Saved files land in the global `GalleryImage` store as a side effect. PDFs get inline previews. *(admin §2.4; backend §6.2)*

- Business rule: the gallery is **write-only** (no delete/update endpoints) and **global** — `GalleryImage` has no `facilityId` (see §9-G13).

**DIN-FR-10 — Usage accounting.** Scheduling a special from the library increments that file's `usageCount` (inside the same transaction as the special's creation); the admin library list displays usage stats to inform reuse/deletion decisions. *(backend §2.2; admin §2.4)*

### 3.C Daily specials

**DIN-FR-11 — Special definition.** A daily special is a scheduled menu *file* (PDF or image poster), not an item list: name (unique per facility), file reference, `repeatPattern ∈ one-time | weekly | multiple-dates | date-range`, computed `effectiveDates[]`, weekly day names, start/end dates, and an active flag. There is **no meal-period dimension** — one special applies to a whole date. *(backend §2.2; admin §2.4)*

**DIN-FR-12 — One special per date, by pattern priority.** When the resident menu is assembled, exactly one special is chosen per day using repeat-pattern priority: `one-time (1) > multiple-dates (2) > weekly (3) > date-range (4)` — the more specific the schedule, the more it wins. *(backend §2.1)*

**DIN-FR-13 — Creation flows & date-conflict exclusivity.** Two creation paths:

1. **Direct** — upload a file (or reference an already-uploaded `fileKey`), optionally saving it to the library/gallery, then create the special with its recurrence config.
2. **Schedule from library** (`/from-library`) — pick an existing library file; runs in a Mongo transaction that validates the date configuration and increments library usage.

Both paths run `resolveEffectiveDateConflicts`: any date the new special claims is **silently stripped from other active specials' effectiveDates** — last-write-wins exclusivity per date, with no warning to the admin (see §9-G5). *(backend §2.2)*

**DIN-FR-14 — Week view.** Admins see a "This Week" board of 7 weekday cards showing each day's applicable special (or "No special menu"); weekly-recurring specials carry a "Weekly" badge. Backed by a 7-day applicability read (`GET /daily-specials/week`). Update supports optional file re-upload; delete takes a date qualifier. *(admin §2.4; backend §2.2)*

### 3.D Diet plans

**DIN-FR-15 — Diet plan record.** One diet plan per resident (admin UI treats it as one record per resident): resident reference (immutable after creation in the admin UI), optional authoring-staff reference, 1..n entries `{dietType (required), dietarySupplements, description}`, free-text notes, active flag. Entries can be added/removed dynamically ("Add Another Diet Plan"). Plan entries surface **read-only** inside the resident's own menu response (DIN-FR-05). *(backend §2.3; admin §2.4)*

- Business rules: created by STAFF/ADMIN only (route-gated); **delete is ADMIN-only**; no client-side prevention of duplicate diet types within a plan.

**DIN-FR-16 — Dietitian-scoped visibility.** Staff whose free-text designation is exactly **"Dietitian"** see only plans for residents assigned to them via `resident.dietitian`; admins and staff of any other designation see all facility plans. A "/my-plans" view returns plans the calling staff member created. A searchable resident list backs the picker (admins see all residents; staff see only residents they hold plans for). *(backend §2.3)*

**DIN-FR-17 — Admin diet table.** The Diet Management page lists Resident/Room, Diet Type, Supplements, Description with search + pagination — but renders **only the first plan entry** per resident in the table; the edit modal shows all entries (see §9-G9). *(admin §2.4)*

### 3.E Family meal requests

**DIN-FR-18 — Request creation (the dining transaction).** A booking-policy-gated create (`POST /family-meal-requests`) records: resident, guest count, `mealType ∈ BREAKFAST | LUNCH | DINNER`, meal time, and a start/end meal-date range (multi-day bookings supported). Flow:

1. Caller context resolved by the central booking policy (DIN-FR-23).
2. Start/end dates validated.
3. **Blackout check**: if any date in the range is in `Config.blackoutDates`, the whole range is rejected.
4. `totalAmount = guests × pricePerPerson × days` computed server-side — but `pricePerPerson` and `mealTime` are **taken from the client request** rather than re-read from config (see §9-G2).
5. Request persisted with `createdByType/createdByCName` provenance; notifications fan out (§5).

*(backend §2.4)*

- Business rule (advertised, not enforced): `maxGuest` from config is returned to all clients via `/weekly-meals` and `/price-and-time` and constrains client dropdowns, but the server never enforces it (see §9-G2).

**DIN-FR-19 — Status model (effectively single-state).** The schema enum is `COMPLETED | PENDING | APPROVED | REJECTED`, but **requests default to `COMPLETED` on creation** — a model comment marks the approval statuses as legacy/unused. A `PATCH /:id/status` endpoint survives (any authenticated user; no role gate). The admin UI renders status badges but offers **no approve/reject actions**; the SL resident app defines a status type but never renders it. Net: there is no functioning approval workflow today (see §9-G1). *(backend §2.4; admin §2.4)*

State machine as-built:

```
create ──────────────────────► COMPLETED   (default; terminal in practice)
PATCH /:id/status ───────────► any enum value (ungated, unused by any UI)
```

**DIN-FR-20 — Resident views.** Residents (and family on their behalf) see:
- **Current/upcoming**: requests whose `endMealDate ≥ today`, with a meal-time cutoff applied for today's date (a meal whose time has passed drops out).
- **History**: strictly past requests, paginated. (Consumed by the SN app and — **as of staging — by the SL app too**, which now has its own Upcoming/History segments via `/api/family-meal-requests/resident[/history]` — see §8.)
*(backend §2.4; client-sl §3.4; client-sn §3.7)*

**DIN-FR-21 — Admin view (grouped by date).** Admin sees requests **grouped by date** in collapsible cards — per-date request count, total guests, total revenue, and a blackout flag — each row showing resident avatar/room, guest count, meal-type badge, meal time, and total price. Multi-day bookings are **expanded client-side into one row per day** (composite id `{_id}-{dateStr}`). The read is a flat array with no server pagination. Authorization quirk: the list route is auth-only — residents/family are scoped to self but **any staff member sees all requests** regardless of Dining permission. *(admin §2.4; backend §2.4)*

**DIN-FR-22 — Meal configuration (admin levers).**
- **Pricing**: per-meal-type rates (breakfast/lunch/dinner), each required > 0, saved via the meal-config update.
- **Blackout dates**: a multi-date calendar of days on which **no new requests are accepted**; existing requests on a blacked-out date remain visible, labelled "Blackout Date" with destructive styling.
- **Meal time windows** (breakfast 07:00–10:00, lunch 11:30–14:00, dinner 17:00–20:00 defaults) exist in config but are **not editable in any UI** (see §9-G12).
- If a facility has no meal config, the first admin list read **auto-creates a default** (rates $20/$25/$30, standard windows) — a side-effectful GET with hardcoded business numbers (see §9-G6).
*(admin §2.4; backend §2.4)*

**DIN-FR-23 — On-behalf booking policy.** Family meal requests participate in the platform booking-context policy:
- Residents always book for themselves (any supplied resident identifier is ignored — spoof guard).
- Family members may book only if facility config `bookingPermission.FamilyMealRequest.isFamilyMemberAllowed` is true; they book for their linked resident (auth middleware rewrites their identity to the resident's).
- Staff must name a resident and hold an allowed designation; admins bypass the designation check but the module must be configured.
*(backend §0.3)*

**DIN-FR-24 — Bookable week preview.** `GET /weekly-meals` (unauthenticated) returns the next 7 days' bookable meal configuration, skipping blackout dates — the data source for the TV's request dialog. *(backend §2.4; client-tv §3.4)*

### 3.F Resident mobile — menu browsing & meal requests

**DIN-FR-25 — Dining screen (SL).** Two tabs: **All Day Menu** — an accordion of categories/items with images for a selected date (pull-down calendar; menu reloads per date) — and **Specials** — a zoomable image of the day's first special file. Fetch errors silently empty both lists (no error UI — §9-G8). *(client-sl §3.4)*

**DIN-FR-26 — Request Family Meal (SL, updated on staging).** A config-driven form built from the pre-login meal facts (DIN-FR-07): enabled meals with per-meal price; `maxGuest` constrains the guest-count dropdown (default 1–10). The form **now offers a meal-time picker and a start/end meal-date range** (`RequestFamilyMeal/index.tsx` — `time`/`setTime`, `startDate`/`endDate`, default start = tomorrow), submitting `startMealDate`/`endMealDate`/`mealTime` — the prior "fixed serving time, no picker, single-day" behavior is gone (closing most of the SL↔SN dining gap). Total displayed = guests × price. Submit lands on a confirmation screen; **submission failure now routes to the confirmation screen with an error state** (`isSuccess: false`), no longer console-only (§9-G8). *(client-sl `RequestFamilyMeal`)*

**DIN-FR-27 — Requested meal list (SL, updated on staging).** Now an **Upcoming/History segmented view** (`RequestedMealList/index.tsx` — `upcomingMeals` via `GET /api/family-meal-requests/resident`, `historyMeals` via `/resident/history`, paginated). Cards show meal type, price/person, formatted date-time, **and a rendered status badge** (`getMealStatusStyle` / `formatStatus`). The prior "status not rendered / no history" gaps are closed. There is still **no resident cancel action** — once requested, a resident cannot withdraw from the app. *(client-sl `RequestedMealList`)*

**DIN-FR-28 — Dining screen (SN) — re-platformed onto Menu2U tray cards (2026-08-21, corrects the description below).** As of `origin/master` HEAD `4734daf`, the menu source is `GET /api/dining-tray/menu` (DIN-FR-38), not the category/item catalog — specials, the diet-plan calendar, and the date picker are commented out (§9-G17), leaving a **Today/Tomorrow** toggle over the tray-card menu plus the resident **diet-info card** (Diet Type, Texture Level, Liquid Consistency — now sourced from Menu2U tray data, not the `DietPlan` record described in DIN-FR-15), meal price/time facts, "Request Family Meal" with guest count and date+time selection, and a requested-meals list with history. *Prior description (still accurate for the family-meal-request half; superseded for the menu-browsing half):* the app previously showed the full category/item catalog plus daily specials via `DietInfoCard`-adjacent components. *(client-sn §3.7; `senior_living_skillednursing_resident` `DiningScreen/index.tsx`)*

### 3.G TV app — dining surfaces

**DIN-FR-29 — 7-day menu browser.** The TV dining tab shows a left sidebar of the next 7 days (Today / Tomorrow / dates, generated client-side); focusing a date fetches that day's resolved menu, rendered as meal categories with item cards (name + picture). **Browse-only** — no per-item ordering, no dietary filters. Content refetches on every tab entry (no push/socket channel). *(client-tv §3.4, §4)*

**DIN-FR-30 — Today's specials poster.** A dedicated tab renders the day's first special file as a single full-bleed poster image; recurrence is resolved server-side. *(client-tv §3.4)*

**DIN-FR-31 — TV family-meal request.** The one TV dining transaction: "Request Meal" → auth gate (QR pairing login if signed out) → bookable-week read (DIN-FR-24) → dialog to pick a weekly meal, guest count, meal type/time, price-per-person → request posted with a start/end meal-date pair → confirmation dialog. A revenue analytics event (`guests × pricePerPerson`) is logged client-side. *(client-tv §3.4)*

**DIN-FR-32 — Meals on the unified schedule.** Facility meal windows surface as `BREAKFAST | LUNCH | DINNER` typed entries in the TV's daily schedule feed and in the **public** (pre-pairing) unified-schedule payload (meals + activities only). Family meal *requests* do not write to the unified schedule. *(client-tv §3.8-equivalent; backend §0.4)*


### 3.H Dining tray-card system — Menu2U integration (production 2026-06-23)

A new `DiningTrayCard` subsystem maps external menu/dietary data to facility residents and maintains a per-resident tray-card record used by kitchen and dining staff.

**DIN-FR-33 — Tray-card entity.** `DiningTrayCard` records bind: facility, resident ref, `roomNo`, `bedNo`, `residentName`, menu-item data imported from Menu2U (dietary preferences, portion sizes, special requirements), `isResolved` flag, soft-delete flag, and timestamps. Room + bed mapping is used to match incoming Menu2U data to the correct resident. (`diningTrayCard.model.ts`)

**DIN-FR-34 — Menu2U / Menu2Uplus integration.** An external scraper (`src/integrations/menu2plus/menu2plus.scraper.ts`, 648 lines) fetches menu and dietary data from the Menu2Uplus service. A cron job (`diningTrayCard.cron.ts`) runs on a schedule to sync the scraped data into `DiningTrayCard` documents with resident mapping. The scraper resolves residents by matching on `roomNo` + `bedNo` (primary) or resident name (fallback). (`diningTrayCard.cron.ts`, `menu2plus.scraper.ts`)

**DIN-FR-35 — Backfill on resident creation.** When a new resident is created, the system **backfills** tray-card entries: it scans for any existing unresolved `DiningTrayCard` rows whose `roomNo`/`bedNo` or `residentName` match the newly admitted resident, then links them to the resident record. Enhanced backfill (production 2026-06-29) updates **all** unresolved tray-card entries (not just the first match) and supports both room-and-name matching. (`diningTrayCard.service.ts:backfillResidentInTrayCards`)

**DIN-FR-36 — Tray-card CRUD and role-based reads.** Controller endpoints:
- `GET /dining-tray-cards` — paginated list (search by resident name, room, bed); admin sees all, staff see their assigned residents' cards.
- `GET /dining-tray-cards/by-role` — role-scoped fetch (endpoint `35de4b41`, production 2026-06-26): returns tray-card data scoped to the caller's designation and assigned residents.
- `GET /dining-tray-cards/residents` — searchable resident list for the tray-card picker.
- `POST /dining-tray-cards`, `PUT /:id`, `DELETE /:id` — full CRUD. (`diningTrayCard.controller.ts`, `diningTrayCard.routes.ts`)

**DIN-FR-37 — Room/bed mapping logic.** The tray-card service resolves `roomNo` and `bedNo` from the raw Menu2U data (production 2026-06-30 update improved parsing). When a match is not found by exact room + bed, the service falls back to resident name matching. Unresolved entries (`isResolved: false`) accumulate until a resident with a matching room/bed is admitted or a manual assignment is made. (`diningTrayCard.service.ts`)

**DIN-FR-38 — Resident-facing tray-card menu (NEW, `senior_living_skillednursing_resident`, `origin/master` HEAD `4734daf`).** `GET /api/dining-tray/menu` (`?date=YYYY-MM-DD`, default today) is a **dedicated resident/family mobile endpoint** distinct from the internal `/dining-tray-cards` CRUD (DIN-FR-36) — it resolves the calling resident (by Cognito `cName`) or, when `DINING_TRAY_TEST_BYPASS=true` and a `residentId` query param is supplied, by that id with **no authentication at all**, and returns their `DiningTrayCard`-sourced menu for the date via `getResidentMenu()`. The SN resident app's `DiningScreen` now sources its menu from this route (`fetchDiningTrayMenu`), not the category/item catalog `GET /api/menu` (DIN-FR-05) that the SL app and TV app still use — **this corrects the business-rule note under DIN-FR-33–37 below**, which documented tray cards as staff-only. A second route, `GET /dining-tray/residents-menu` (DIN-FR-36's sibling), serves the admin/staff role-scoped multi-resident read. (`diningTrayCard.routes.ts`, `diningTrayCard.controller.ts: getMyMenuController`)
*As-built gap:* `DINING_TRAY_TEST_BYPASS=true` skips auth entirely on `GET /dining-tray/menu` whenever a `residentId` query param is present, with no `NODE_ENV=production` fail-closed check — the same defect pattern as `FAX_LOCAL_BYPASS` (platform-foundation.md §9 item 33). See §9-G17.

*Business rules:*
- One tray card per resident per menu-sync cycle; duplicates within a cycle are de-duped.
- Backfill is triggered at resident creation and runs against all unresolved entries.
- **Corrected 2026-08-21 (was: "not surfaced in the resident mobile app or TV").** Tray-card data **is** now resident-facing on the SN resident app via the dedicated `GET /dining-tray/menu` read (DIN-FR-38) — the internal `/dining-tray-cards` CRUD (DIN-FR-36) remains staff/kitchen-only, but the underlying `DiningTrayCard` data feeds both. Not confirmed on the SL app or TV app — no evidence either client calls `/dining-tray/menu` as of this pass.
- The Menu2U scraper is configurable (credentials, facility mapping) via environment/config.

*Observations:*
- **O-tray-1 — Auth not yet verified**: tray-card CRUD routes' authentication posture was not confirmed at the time of writing; verify that `authMiddleware` is applied to write endpoints.
- **O-tray-2 — Menu2U credentials**: the scraper requires external credentials; their storage location (Secrets Manager vs env var) should be confirmed per the platform secrets policy.

---

## 4. Business rules & policies (consolidated)

| # | Rule | Source |
|---|---|---|
| BR-1 | Item availability resolves pattern → date; per-date `dateOverrides` always win | backend §2.1 |
| BR-2 | Exactly **one** daily special per date; selection priority one-time > multiple-dates > weekly > date-range | backend §2.1 |
| BR-3 | Special names unique per facility | backend §2.2 |
| BR-4 | A new special's dates are exclusive: overlapping dates are stripped from other active specials (last-write-wins, silent) | backend §2.2 |
| BR-5 | Scheduling from library increments the file's usage count (transactional) | backend §2.2 |
| BR-6 | Diet plan delete is ADMIN-only; create/update is STAFF/ADMIN | backend §2.3 |
| BR-7 | "Dietitian"-designated staff see only their assigned residents' plans; everyone else with access sees all | backend §2.3 |
| BR-8 | Diet plan's resident is immutable after creation (admin UI) | admin §2.4 |
| BR-9 | Family-meal total = guests × pricePerPerson × days | backend §2.4 |
| BR-10 | Any blackout date inside the requested range rejects the **entire** request | backend §2.4 |
| BR-11 | Meal rates must each be > 0 when configured | admin §2.4 |
| BR-12 | `maxGuest` constrains client UIs only — not enforced server-side | backend §2.4 |
| BR-13 | Family booking allowed only when facility policy `isFamilyMemberAllowed` is on; staff need an allowed designation; residents always self-serve | backend §0.3 |
| BR-14 | Requests default to status `COMPLETED`; approval states are legacy | backend §2.4 |
| BR-15 | Resident "upcoming" list applies a same-day meal-time cutoff | backend §2.4 |
| BR-16 | Missing meal config self-heals with defaults ($20/$25/$30) on first admin read | backend §2.4 |
| BR-17 | Image contract platform-wide: send S3 `imageKey`, never the signed URL | admin §2.4 |
| BR-18 | All data facility-scoped via `x-facility-id` header (with a known dead missing-header check) | backend §0.2 |

---

## 5. Notifications & real-time behavior

- **Creation fan-out**: a new family-meal request notifies (a) the resident, (b) linked family members, and (c) **all staff holding the Dining page permission** (`DINING_REQUEST_CREATED` via the permission-based creation-notification service). Dining uses the *permission-audience* variant, not the assigned-staff variant used by salon/massage/PT. *(backend §0.5, §2.4)*
- **Socket events**: `mobile-family-meal-request-created/updated/deleted` are emitted for app-side live refresh. *(backend §0.5)*
- **Push logging**: every send is recorded in `NotificationHistory`. *(backend §0.5)*
- **No reminder cron coverage**: the reminder cron maps SALON/TRANSPORT/ACTIVITIES modules only — there are no scheduled meal reminders. Menu/special changes have **no push or socket channel**; the TV and apps see changes on next fetch/tab entry. *(backend §0.5; client-tv §4)*
- **Admin dashboard**: Dining events appear in the admin "Recent Activity" cross-module feed (ADMIN only). The Settings "Family Meal Requests" notification-preference toggle is **decorative** (localStorage only, no backend sync). *(admin §3, §4)*

---

## 6. Integrations

- **PMS switchboard (declared, unimplemented)**: `Config.integratedModules` can map DINING → `OPERA | POINTCLICKCARE | YARDI | TELS | CUSTOM`, but no dining provider integration exists in code today (only Housekeeping/Maintenance→TELS is implemented). Treat dining-PMS sync as roadmap, not as-built. *(backend §7.3)*
- **Menu2U / Menu2Uplus** (production 2026-06-23): external menu-data scraper and cron that imports dietary/menu data per resident into the `DiningTrayCard` subsystem (DIN-FR-33–37). This is the as-built dining integration path for facilities using the Menu2Uplus platform; it does not write to the Menu/Category/Item catalog used by residents' menu views — it populates the separate tray-card collection used by kitchen staff.
- **S3 + signed URLs**: all menu/special/library files are S3 objects served via signed URLs (platform image contract). *(admin §2.4)*
- **Meal windows → wellness modules**: `Config.meals` blackout windows remove overlapping slots from salon/massage/PT availability generation — dining config has cross-module side effects. *(backend §1.2)*
- **Payment history**: family meals surface as `MEAL`-typed line items in the resident payment-history view (location/provider columns hidden for meals). *(admin §1)*
- **Analytics**: TV logs a client-side revenue event per meal request. *(client-tv §3.4)*

---

## 7. Permissions & access control

| Operation | As-built gate |
|---|---|
| Admin Dining pages (menu, specials, library, requests, diet) | Staff per-module **Dining** read/write access permissions; admins pass on role alone (permission middleware no-ops for non-STAFF) |
| Menu / category / item / special / library CRUD endpoints | **No authentication middleware** — admin-panel routes rely on network trust (§9-G3) |
| `getMenuForAdmin`, `price-and-time`, `weekly-meals` | **Unauthenticated** (the latter two intentionally pre-login) |
| Resident menu read (`GET /menu`) | Authenticated; residents additionally receive their diet entries |
| Family-meal create / resident views | Auth + central booking-context policy (DIN-FR-23) |
| Family-meal admin list | Auth only — **any staff sees all**, no Dining-permission check |
| Family-meal status PATCH / generic PUT | Auth only — no role gate; PUT applies the raw body (§9-G4) |
| Diet plan create/update | STAFF/ADMIN roles (route-gated); Dietitian visibility scoping per DIN-FR-16 |
| Diet plan delete | ADMIN only |
| TV browse routes | Device-scoped (unauthenticated catalog); TV transaction requires resident pairing token |

Multi-tenancy: every query scopes by `facilityId` from the request header — but the missing-header rejection is dead code, so an absent header silently yields unscoped queries (platform-wide issue, inherited here). *(backend §0.2)*

---

## 8. Product-split notes (Senior Living vs Skilled Nursing)

- **The backend dining module is product-agnostic** — one codebase, one schema, differentiated only by per-facility configuration (`accessPages`, booking permissions, meal config). The admin web Dining cluster is identical for both facility types. *(admin §5; backend §7)*
- **SN resident app is the richer client (gap narrowed on staging)**: it adds resident-facing **diet plan cards** (`DietInfoCard`, SN-only component). The SL app has since gained a meal-request **Upcoming/History** view and **date-range + time selection** in the request flow (DIN-FR-26/27), so the remaining SL omission relative to SN is the **resident diet-plan card display** (no `DietInfoCard`/`dietType` anywhere in the SL DiningScreen). *(client-sn §3.7; client-sl `DiningScreen`/`RequestFamilyMeal`/`RequestedMealList`)* **2026-08-21 update:** the SN app's menu-browsing half diverged further — it now reads from the Menu2U tray-card pipeline (DIN-FR-38) instead of the shared category/item catalog the SL app and TV app still use. "Richer client" is no longer the whole story: the two resident apps' dining screens now draw from **two different backend data sources** for the browse-menu experience, not just different UI depth over the same data. A product decision is needed on whether Menu2U/tray-cards or the admin-maintained catalog is the intended long-term source of truth for resident-facing menus.
- **Vocabulary lag in SN**: the SN app's `MealRequest.careType` is typed `'assisted_living' | 'memory_care'` — senior-living vocabulary not yet updated for skilled nursing (cosmetic, but a signal the dining flow was ported, not redesigned). *(client-sn §6)*
- **Care type is per-resident, not per-facility** (`assisted_living | memory_care | independent_living | skilled_nursing`), so a mixed-acuity facility serves both products from one menu. Dining requirements should therefore stay in the shared core; only client-side presentation (diet cards, history) differs. *(backend §7.1)*
- **Diet plans straddle the split**: in SL they read as a hospitality preference; in SN they sit adjacent to clinical nutrition. The current model (free-text dietType, no link to medical records/allergies) is the hospitality flavor — an SN clinical-nutrition requirement would be net-new.

---

## 9. Observations & candidate gaps

Each item cites its evidence source; "G" numbers are referenced from earlier sections.

**G1 — Family-meal approval workflow is half-built (highest product priority).** The status enum (`PENDING|APPROVED|REJECTED|COMPLETED`) exists, but creation defaults to `COMPLETED` (model comment marks approval as legacy), the admin UI has no approve/reject actions, the status PATCH endpoint is role-ungated and unused, and the SL resident app never renders status. Decide: either ship the approval loop end-to-end (default PENDING, admin actions, resident status display + notifications) or delete the enum and the orphan endpoint. *(backend §2.4 / `familyMealRequest.model.ts:29-31`; admin §4 "no approve/reject UI"; client-sl §3.4)*

**G2 — Client-trusted pricing and limits.** `pricePerPerson` and `mealTime` are taken from the request body, not re-read from config; `maxGuest` is advertised to clients but never enforced server-side. A modified client can book unlimited guests at $0. Server must price from config and validate guest count/meal time. *(backend §2.4, §8.5)*

**G3 — Unauthenticated admin CRUD surface.** No auth middleware on menu, category, item, daily-special, or menu-library routes, nor on `getMenuForAdmin` — anyone who can reach the API can rewrite the facility's menu. Part of a platform-wide admin-route pattern, but dining is among the most exposed clusters. *(backend §8.1)*

**G4 — Raw-body update injection.** Family-meal `PUT /:id` passes the request body directly into a Mongo update (no field whitelist; operator-injection risk), gated by auth only. *(backend §2.4, §8.4)*

**G5 — Silent special-date exclusivity.** Scheduling a special silently strips overlapping dates from sibling specials (last-write-wins). Admins get no warning that an existing special lost dates — surfaced in the backend analysis as "surprising for admins". Needs a conflict-confirmation UX or at least an audit trail. *(backend §2.2, §8.17)*

**G6 — Side-effectful default config.** The admin family-meal list GET auto-creates a default meal config with hardcoded rates (20/25/30) — business pricing embedded in a read handler. Move to explicit facility-setup. *(backend §2.4, §8.18)*

**G7 — Admin dead code (AllDayMenu/Specials).** Drag-drop item reordering commented out (reorder mutations exist, unused); category-reorder mutation never called; item-description edit UI commented out while the description still renders in the view modal; Specials "Schedule Menu" pending-disable check commented out. Indicates abandoned mid-feature work to either finish or remove. *(admin §4: AllDayMenu lines 84-90/365-575/911-931; Specials.tsx:1875)*

**G8 — SL resident app error/UX debt (partially RESOLVED on staging).** Resolved: meal-request submit failure now routes to a Confirmation screen with `isSuccess: false` + message (`RequestFamilyMeal/index.tsx:349-356`), and requested-meal cards now render a status badge across Upcoming/History segments (DIN-FR-27). Still open: menu fetch errors silently fall back to an empty state (`DiningScreen/index.tsx:262-263` — `console.warn` + empty list, no distinct error UI); there is still no resident cancel action on a requested meal. *(client-sl `DiningScreen`, `RequestFamilyMeal`, `RequestedMealList`)*

**G9 — Diet management display & validation gaps.** Admin table shows only the first diet entry per resident (multi-entry plans look single-entry at a glance); no duplicate-diet-type prevention; resident reassignment impossible after creation (by design, but undocumented). *(admin §2.4, §4.4)*

**G10 — Multi-day requests expanded client-side.** Admin builds one row per day with composite ids `{_id}-{dateStr}` from a flat, unpaginated list response — fine at current volume, but grouping/pagination/totals belong server-side before scale. *(admin §2.4)*

**G11 — Copy-paste block in controller.** The `isFamilyMemberOnlyRequest` block is pasted three times in the family-meal controller — refactor target. *(backend §8.9 / `familyMealRequest.controller.ts:205-225`)*

**G12 — Meal time windows have no edit UI.** Breakfast/lunch/dinner windows drive resident-facing display, the SL fixed serving time, the upcoming-list cutoff, and wellness slot blackouts — yet no surface can edit them. *(admin §2.4)*

**G13 — Gallery is global and write-only.** `GalleryImage` has no facility scoping and no delete endpoint; menu files saved-to-gallery accumulate forever and are visible cross-tenant via the picker. *(backend §6.2, §8.3)*

**G14 — Decorative notification preference.** The admin Settings "Family Meal Requests" toggle persists to localStorage only — it controls nothing. *(admin §3)*

**G15 — Specials lack a meal-period dimension.** One poster per date, globally. If the product wants per-meal specials (breakfast special vs dinner special), the model needs a meal-period axis — note this also interacts with the one-special-per-day priority rule (BR-2). *(admin §2.4; backend §2.1)*

**G16 — No dining PMS integration despite the switchboard.** `Config.integratedModules.DINING` accepts a provider but nothing consumes it. Either build (PCC nutrition orders would be the SN-shaped candidate) or hide the option. *(backend §7.3)*

**G17 — SN resident-app dining screen: built-then-disabled surfaces, and a test-bypass auth gap (new, 2026-08-21).** Two distinct items from the DIN-FR-38 re-platform: (a) daily-specials, the diet-plan-calendar view, and the date picker are commented out in `DiningScreen` ("TEMP … kept for restore"), not removed — same "built but disabled" pattern flagged elsewhere in the platform (dining §9-G7's dead reorder code, prd-skilled-nursing.md §7 item 4's orphaned booking screens); decide restore vs. delete rather than leaving it commented indefinitely. The resident-facing diet-info card itself is **not** disabled — it now reads Texture Level / Liquid Consistency fields from the Menu2U tray-card data rather than the admin-authored Diet Plan record (DIN-FR-15), a system-of-record fork worth a product decision. (b) `GET /dining-tray/menu`'s `DINING_TRAY_TEST_BYPASS=true` env flag skips auth entirely when a `residentId` query param is present, with no production fail-closed guard — see DIN-FR-38. *(`senior_living_skillednursing_resident` `DiningScreen/index.tsx`; `senior_living_backend` `diningTrayCard.routes.ts`)*

---

*End of module document. Cross-references: booking policy and notification infrastructure are specified once in the platform foundations and consumed here; the recurrence vocabulary (BR-1/DIN-FR-03) should graduate to a shared platform-scheduling spec used by Activities and Announcements as well.*
