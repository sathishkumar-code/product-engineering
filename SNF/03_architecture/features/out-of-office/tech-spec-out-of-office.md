
# impl-spec.md — Out of Office (OOO)

## Overview

Staff can configure Work Hours and mark themselves Out of Office (OOO) either automatically (outside Work Hours) or manually (vacation/leave). A badge and optional delegate assignment are displayed across all apps. Admins can set OOO for staff. Delegates are notified when assigned.

---

## Data Model

### Staff.availability

```json
{
  "workHours": {
    "MONDAY": { "isWorkDay": true, "start": "09:00", "end": "17:00" },
    ...
    "SUNDAY": { "isWorkDay": false }
  },
  "therapistBookingSlots": { ... }, // unchanged rehab slots
  "manualOOO": {
    "startAt": "Date",
    "endAt": "Date",
    "badge": "String (enum)",
    "statusMessage": "String",
    "delegateCName": "String",
    "createdBy": "String",
    "updatedBy": "String",
    "updatedAt": "Date"
  },
  "workHoursUpdatedAt": "Date"
}
```

### Indexes

- `{facilityId, manualOOO.endAt}` (sparse)
- `{facilityId, manualOOO.delegateCName}` (sparse)

---

## Endpoints

- `PUT /api/staff/{staffCName}/work-hours` → set weekly Work Hours
- `PUT /api/staff/{staffCName}/ooo` → set/cancel manual OOO (badge, status, delegate)
- `GET /api/staff/{staffCName}/ooo-status` → resolved OOO state for one staff
- `GET /api/facilities/{facilityId}/staff/ooo-status?staffIds=...` → bulk resolved OOO states
- `GET /api/facilities/{facilityId}/ooo-staff` → admin list of currently OOO staff
- `GET /api/staff/{staffCName}/covering-for` → delegate’s reverse list

---

## Badge Evaluation

```typescript
function resolveOOOState(staff, facilityNow) {
  if (manualOOO.active) return manualOOO.badge/status/delegate;
  else if (outsideWorkHours) return defaultBadge;
  else return available;
}
```

- Manual OOO takes precedence over automatic.
- Cache key: `ooo:{facilityId}:{staffCName}`
- TTL: min(1h, time until next workHours boundary).
- Invalidate cache on any OOO/Work Hours/delegate write.

---

## Notifications

- **Delegate assignment** → fires on delegate set/change.
- **OOO start (manual only)** → cron runs per minute, checks if today’s startAt == workHours start.
- Delivery: FCM + socket.
- No `NotificationHistory` entry.

---

## UI Components

- **OOOBadge** → shows badge everywhere staff name/avatar appears.
- **OOOProfilePopup** → hover/long‑press popup with name, designation, Work Hours, OOO status, delegate.
- **Resident variant** → omits staff contact info, adds facility phone button.
- **WorkHoursEditor** → weekly recurring editor.
- **DelegatePicker** → facility staff directory, single select.

---

## Business Rules

- BR1: Manual OOO overrides automatic.
- BR2: Manual OOO requires fixed start/end.
- BR3: Delegate assignment is informational only.
- BR4: Delegate eligibility = any staff.
- BR5: No delegate chain resolution.
- BR6: Badge/status changes do not trigger notifications (except delegate + OOO start).

---

## Testing Checklist

- **Unit**: resolveOOOState precedence, cache invalidation, cron dedupe.
- **Integration**: Save → badge updates everywhere; delegate notified once.
- **Performance**: Bulk endpoint under facility‑scale staff list.
- **E2E**: Staff sets OOO → badge appears across apps → delegate notified → admin clears OOO → badge disappears.

---
