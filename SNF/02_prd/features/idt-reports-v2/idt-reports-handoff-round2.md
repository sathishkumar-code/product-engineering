# IDT Reports — Change Summary for SA / Technical Design Update

**Purpose:** everything below changed in the PRD since the last technical design sync. For each item: what changed, where it lives in the PRD, and what it likely means for `feature-idt-reports-v2_TD.md`. Technical-design impact is my read, not confirmed engineering — the SA agent should verify against the actual schema/design before treating any of it as settled.

---

## 1. Personas — Nurse removed (PRD §3)

**Change:** The "Case Manager / Nurse" persona is now "Case Manager" only. Case Manager is the sole role that creates the IDT report.

**TD impact:** Confirm no code path grants report-creation rights to a "Nurse" designation specifically. Should already align with §8's Clinical Staff definition (Case Manager, Social Worker, Rehab Team/Director of Rehab) — likely no schema change, just a permissions-model sanity check.

---

## 2. Refresh now moves the report date on Save (PRD §4.1, §4.3)

**Change:** Using the Refresh control (re-pulls live Patient Information / Weight trend / Active medications) and then saving now **moves the report's date to that save date**, since the data pertains to that day. This is a new, explicit exception to "report date is set at creation and not editable."

**TD impact — flagging for careful review, this one has ripple effects:**
- The rehab progression trend (TD §3.3/§3.4) orders by `reportDate` descending. Need to confirm what happens to a report's position in that sequence if its `reportDate` moves after other reports already reference it.
- Carry-forward's "most recent report" lookup and the `{facilityId, residentId, reportDate}` general index (added to replace the old `weekOf` index) both key off `reportDate` — confirm a date mutation on an existing document is handled cleanly by both.
- The partial unique index enforcing "one open Draft per resident" is keyed on `{facilityId, residentId}` filtered to `status: DRAFT`, not on `reportDate`, so this change shouldn't interact with that index — worth a one-line confirmation from the SA agent that this is still true.

---

## 3. Date format standardized to `mm/dd/yy` everywhere (PRD §4.3)

**Change:** Replaces the old split format (`MMM D, YYYY` on the report, `MMM D` in list columns) with a single `mm/dd/yy` format used everywhere a date renders — report, list, summary, PDF.

**TD impact:** Display-layer only, no schema change expected. Any shared date-formatting utility/component should be updated to this one format product-wide.

---

## 4. Document-status filter removed from list Filters & Sort (PRD §5.1)

**Change:** The status filter (All · Draft · Complete) is removed — it was already a no-op inside a status tab, since the In-progress/Completed tabs already partition by status.

**TD impact:** If the list query API accepts a status-filter parameter, it can be dropped or deprecated; the tabs' own status partitioning is unaffected.

---

## 5. Fax — free text only, no saved contacts, US phone format validation (PRD §5.4)

**Change:** Removed the "short list of saved contacts" (attending MD, resident's primary care practice) from the Fax popover — the product only stores fax numbers for Home Health Agencies, which IDT reports never target. The popover is now a single free-text fax number field, validated to US phone number format (10 digits, formatted/masked as typed).

**TD impact:** Drop any planned saved-contacts lookup for this popover. Add client- and server-side US phone number format validation — confirm the validation library/regex with engineering.

---

## 6. New Fax History section, collapsed by default (PRD §5.4, §7, §12)

**Change:** A new **Fax History** section on the report summary, collapsed by default, lists every fax sent for the report — number, sent date/time, actor, and send status. The Fax write (§7) now also records a status field, not just number/actor/timestamp.

**TD impact:**
- Add a `status` field to whatever write model currently backs the fax audit log, if not already present.
- New query/endpoint needed to list a report's fax history (or this may already be servable from the existing audit log with a filter).
- **Open per PRD §12:** the exact status values available (a simple "Sent" vs. a real delivery/failure status) depend on which service actually transmits the fax and what it reports back — this needs the one-time engineering discussion already flagged for the fax vendor integration.

---

## 7. Status chip removed from both tabs; new Signature Status column (Completed tab only) (PRD §5.1, §6)

**Change:** The old per-row Status chip (Draft/Complete) is removed from both tabs — the tabs already convey that. In its place, the **Completed** tab only gets a new **Signature Status** column showing Not sent / Awaiting signature / Signed. The In-progress tab has no equivalent column (every row there is a Draft, which can never have a signature state beyond "Not sent").

**TD impact:** List query/response shape changes — drop the per-row document-status field if it was a separate API field from the tab/status split, and add signature-state exposure for Completed-tab rows (likely already modeled as a field on the report document, just not previously surfaced in the list response).

---

## 8. Edited-after-signature indicator now shown on the form (PRD §5.2, §5.4, §6 rule 4)

**Change:** Previously this fact was deliberately silent everywhere (report, summary, list) — visible only in the audit trail. Now the **report form's sub-title** shows an "Edited after signature [date]" indicator as its last item, once the report has been saved after being signed. The summary and list remain unchanged — this is form-only.

**TD impact:** Needs a data source for "was this report edited after its most recent signature, and when." Likely derivable from the existing edit-after-signing audit trail (§7) rather than a new field — confirm whether a computed/query-time derivation is sufficient or whether a dedicated field (e.g., a `lastEditedAfterSignatureAt` timestamp) is cleaner for the frontend to consume directly.

---

## No change — reconfirmed as-is

**Create picker / carry-forward exclusion for residents with an open Draft.** I'd raised a question about whether "Create" should redirect to a resident's existing open Draft instead of excluding them from the picker. This was withdrawn — the existing design stands exactly as already confirmed with the SA agent: the Create picker excludes residents who already have an open Draft, the partial unique index enforces it server-side, and the bulk-create partial-failure toast copy is unchanged. **No technical design action needed here** — flagging only so this doesn't get re-opened by mistake.

---

# Suggested ClickUp tasks

Draft list, grouped by area — refine titles/estimates as needed before creating:

**Backend / Data**
- [ ] Confirm no report-creation permission path exists for a "Nurse" designation (Case Manager-only)
- [ ] Implement report-date update on Refresh + Save; verify rehab progression ordering, carry-forward lookup, and the `{facilityId, residentId, reportDate}` index all handle a `reportDate` mutation correctly
- [ ] Add `status` field to the fax audit log write (if not already present)
- [ ] Build/confirm query to list a report's fax history for the new Fax History UI section
- [ ] Expose signature state as a list-row field for the Completed tab (Signature Status column)
- [ ] Expose "edited after signature" fact + date for the form sub-title (derive from audit trail or add a dedicated field — engineering call)
- [ ] One-time engineering discussion: fax vendor integration and what send-status values it actually returns (already flagged in PRD §12)

**Frontend**
- [ ] Standardize all date rendering to `mm/dd/yy` (report, list, summary, PDF)
- [ ] Remove document-status filter from list Filters & Sort
- [ ] Fax popover: remove saved-contacts list, free-text-only number field with US phone format validation
- [ ] Build collapsed-by-default Fax History section on the report summary
- [ ] Remove Status chip from both list tabs; add Signature Status column to Completed tab only
- [ ] Add "Edited after signature [date]" as the last item in the report form's sub-title

**Verification / no-op**
- [ ] Confirm the Create-picker exclusion (open-Draft residents) and its partial unique index are unaffected by this round — no change expected, just a sanity check

---

*Source: `prd-idt-reports-v2.md`, current version after this review round.*
