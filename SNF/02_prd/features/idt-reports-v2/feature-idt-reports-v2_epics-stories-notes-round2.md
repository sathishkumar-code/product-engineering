# PM Notes: Epics / Stories / Tasks — Round 2

**For:** PM / PRD owner
**From:** System Architect review of `idt-reports-handoff-round2.md` against `feature-idt-reports-v2_TD.md`
**Purpose:** The handoff document's own "Suggested ClickUp tasks" section was a first draft, explicitly
flagged as "my read, not confirmed engineering." This corrects that list against the finished technical
design so it's ready to create tickets from — some items turned out to need no backend work at all,
and two new coordination items surfaced. These are additions to the round-1 candidate epic list already
in `feature-idt-reports-v2_SA-comments.md`, not a replacement for it.

---

## Backend / Data — revised

- [ ] **Nurse-persona permission check.** No code change expected — this design's Clinical Staff matrix
  never included a "Nurse" designation. One-line confirmation only; folded into the pre-implementation
  Spike (see below), not its own ticket.
- [ ] **`reportDate`/`createdAt` sequencing fix — confirmed by Sathish, ready.** The handoff's
  "implement report-date update on Refresh + Save" item turned out to need more than a field update:
  making `reportDate` mutable breaks its old role as the ordering/anchor key for carry-forward, the
  rehab progression trend, the general lookup index, and retention. Resolution, now confirmed:
  `reportDate` keeps its new PRD-defined job of capturing the report's date as revised by a Refresh +
  Save (display/filing only); every actual ordering/retention use switches to the already-existing
  `createdAt` field, with a preceding Refresh detected via a server-side diff at Save time. Fully
  specified in TD §3.3/§3.4/§3.6/§3.8. A single well-scoped ticket: update the four call sites, rename
  the two indexes, add the Save-time diff, add the retention/ordering regression test described in
  TD §8. Note for whoever writes the story: reference the TD directly, since the PRD text alone only
  says `reportDate` moves and doesn't mention `createdAt`.
- [ ] Add `status` field to `faxLog` (TD §3.4) — unblocked, no dependency on the vendor discussion below
  to add the field itself; the *values* it can hold depend on that discussion.
- [ ] ~~Build/confirm query to list a report's fax history~~ — **not needed.** `faxLog` already returns
  with every report fetch; the new Fax History section is frontend-only work over existing data
  (TD §3.4, §6).
- [ ] Expose `signature.state` in list-row responses for the Completed tab's Signature Status column —
  no new index needed, `{facilityId, signature.state}` already exists (TD §6).
- [ ] Expose `signature.editedAfterSignAt` in the report-get response for the form sub-title indicator —
  no new field needed, it already exists and is already set/cleared correctly (TD §3.3, §3.4, §6).
- [ ] One-time engineering discussion: fax vendor integration and send-status values (already flagged in
  PRD §12) — folded into the pre-implementation Spike, not its own ticket.

## Frontend — unchanged from the handoff's draft, with one dependency note

- [ ] Standardize all date rendering to `mm/dd/yy` (report, list, summary, PDF) — display-only,
  no backend dependency.
- [ ] Remove document-status filter from list Filters & Sort — no backend dependency; this design never
  modeled a separate status-filter parameter.
- [ ] Fax popover: remove saved-contacts list, free-text-only number field with US phone format
  validation — needs matching server-side validation on the fax-send endpoint (paired backend item,
  not listed separately above since it's the same endpoint change).
- [ ] Build collapsed-by-default Fax History section on the report summary — **no backend ticket
  blocks this** once `faxLog.status` exists; it's a straightforward render over the existing report-get
  response.
- [ ] Remove Status chip from both list tabs; add Signature Status column to Completed tab only — can
  start once the backend item above ships `signature.state` in the list response.
- [ ] Add "Edited after signature [date]" as the last item in the report form's sub-title — can start
  once the backend item above exposes `editedAfterSignAt` on report-get.

## Verification / no-op

- [ ] Confirm the Create-picker exclusion (open-Draft residents) and its partial unique index are
  unaffected by this round — reconfirmed already in this review, no code change expected; a sanity
  check only, not a ticket that needs engineering time beyond a quick look.

## New coordination items (fold into the existing pre-implementation Spike)

The Spike task in `feature-idt-reports-v2_SA-comments.md` ("Spike — pre-implementation confirmations
and coordination") now covers two more items from this round, alongside its original five:

6. **Fax vendor integration and send-status values** — gates only the Fax History section's exact
   status display; the fax send/write itself is unblocked.
7. **Nurse-persona permission sanity check** — expected to be a no-op; gates nothing.

No new ticket needed for either — they're additions to the existing Spike's scope.

---

## Net effect on scope

Six of the eight round-2 changes are smaller than the handoff's draft made them look — three have zero
backend work, and two others turn out to need only field exposure rather than new endpoints. The one
item that grew in scope, the `reportDate`/`createdAt` fix, is now confirmed by Sathish and ready for
story creation like everything else in this list.
