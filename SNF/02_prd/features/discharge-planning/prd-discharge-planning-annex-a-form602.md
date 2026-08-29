# Annex A — LIC 602 A Field Specification

Complete field inventory for the Physician's Report for Community Care Facilities (State of California CDSS form LIC 602 A), the state form required when a resident is discharged to an ALF. Companion to §5.8 of the Discharge Planning PRD (`prd-discharge-planning.md`).

| Item | Value |
|---|---|
| Status | Draft for Product Management review — v0.3, resynced against the main PRD (now Draft v2.6) |
| Relationship | Annex to *Safe Discharge Plan for SNF Residents — Product Requirements*. §5.8 of that document owns the modal's behaviour (open, print, fax, draft, signature); this annex owns the field list. |
| Sources | SNF Discharge Demo prototype (structure and prefill of record) and the CDSS form LIC 602 A. Prefill values marked *sample* are demo fixtures, not requirements. |
| Regulatory | HIPAA. The completed form is PHI in full. |
| Cross-check | v0.2: the field inventory in §3 was verified field-by-field against the `form602Blocks` structure in the current prototype source and confirmed accurate. The field-count summary in §4 was recounted directly against the code (every `tf(...)` call site) and corrected — see the note there. v0.3: §1's OQ-20 note was stale relative to the main PRD, which resolved it in an earlier PM round — corrected to match (§1, §5). No other content changed. |

## 1. Scope and trigger

The form is reachable from one place: the **Form 602 card on Step 1 (Discharge Order)**, which renders only when **Discharge to = ALF**. It opens as a full-screen modal over the plan. Board & Care does not surface the card, and — per the main PRD, updated since this annex was last synced — Step 5's "Form 602 Signed" checklist item is now confirmed **hidden entirely** when the destination is not ALF (the opposite of "shown regardless of destination," which was this annex's stale assumption). Both are resolved, not open — see the main PRD §5.3 and §5.7 (**OQ-20 closed**).

The modal header carries two pills: "Auto-populated · editable" and, when any field is blank, "*n* field(s) need input". The count is live and covers every text field below; it does **not** count unanswered Yes/No rows or unselected choice groups.

## 2. Field conventions

| Control type | Behaviour |
|---|---|
| Text | Single-line, editable inline. Filled fields render borderless; empty fields render on an amber background with the placeholder "Not available — enter" and count towards the header pill. |
| Text (multi) | Same, auto-sizing and vertically resizable. Used for diagnoses, treatment lines and comments. |
| Choice | Pill radio group, single-select, **nothing selected by default**. No option clears a selection once made. |
| Yes/No row | Two buttons (Yes green, No red when active), single-select, unanswered by default. Each row also has a free-text *Explain* field; §13 rows additionally have an *Assistive Device* field. |
| Checkbox | Independent toggle, unchecked by default (§9 only). |

> **OQ-A1 · Required fields** No field is currently required, and nothing blocks print, fax, draft or discharge completion when fields are blank. The state form expects every question to be answered. Which fields must be enforced, and at which moment (fax, sign, or discharge completion)?

## 3. Field inventory

Numbering follows the state form. "Prefill" names the source the product should use; *sample* marks a value that exists only as a demo fixture and needs a real source decision (**OQ-51**). This section was re-verified field-by-field against the current prototype source for v0.2; no changes were needed.

### I. Facility Information — completed by licensee / designee

| Ref | Field | Control | Prefill |
|---|---|---|---|
| 1 | Name of Facility | Text | First segment of the Step 1 destination address string |
| 2 | Telephone | Text | None — always blank |
| 3 | Address | Text | Second segment of the destination address |
| 3 | City | Text | Third segment of the destination address |
| 3 | Zip Code | Text | First five-digit run found in the destination address |
| 4 | Licensee's Name | Text | None |
| 5 | Telephone | Text | None |
| 6 | Facility License Number | Text | None |

> **OQ-A2 · Facility directory** Facility name, address, telephone, licensee and licence number are parsed from a free-text address string today, which is fragile and leaves four fields permanently blank. Should ALF destinations come from a maintained facility directory (structured record with licence number and phone) instead of the address autocomplete?

### II. Resident / Patient Information — completed by resident or responsible person

| Ref | Field | Control | Prefill |
|---|---|---|---|
| 1 | Name | Text | Plan resident (PCC) |
| 2 | Birth Date | Text | PCC date of birth. *Sample* today, and it disagrees with the Step 1 value (**OQ-15**). |
| 3 | Age | Text | Should be derived from date of birth at render time; *sample* static value today. |

### III. Authorization for Release of Medical Information

Printed statement: the resident authorises release of the medical information in this report to the facility named above.

| Ref | Field | Control | Prefill |
|---|---|---|---|
| 1 | Signature of Resident and/or Resident's Legal Representative | Text | None — plain text field, not an e-signature (**OQ-A3**) |
| 2 | Address | Text | None |
| 3 | Date | Text | None |

> **OQ-A3 · Resident authorisation** §III is a HIPAA release signed by the resident or their legal representative, but the product captures it as free text typed by staff. Is a real resident-facing signature step needed (in person, e-sign link, or wet signature on the printed form), and may the form be faxed before it is obtained?

### IV. Patient's Diagnosis — Examination (completed by the physician)

Note to physician printed on the form: the person named is a resident or prospective resident of a licensed RCFE; these facilities provide primarily non-medical care and supervision and do not provide skilled nursing care, so all questions must be answered.

| Ref | Field | Control | Prefill |
|---|---|---|---|
| 1 | Date of Exam | Text | None |
| 2 | Sex | Text | PCC. *Sample* today; should this be a controlled value list? |
| 3 | Height | Text | Candidate for PCC vitals |
| 4 | Weight | Text | Candidate for PCC vitals |
| 5 | Blood Pressure | Text | Candidate for PCC vitals |

### 6. Tuberculosis (TB) Test

| Ref | Field | Control | Prefill / options |
|---|---|---|---|
| 6a | Date TB Test Given | Text | PCC immunisation / lab record. *Sample* today. |
| 6b | Date TB Test Read | Text | Same. *Sample* today. |
| 6c | Type of TB Test | Text | Same. *Sample* today. |
| 6d | TB test is: | Choice | Negative · Positive |
| 6e | Results (mm) | Text | *Sample* today. |
| 6f | Action Taken (if positive) | Text | None. Not conditionally shown — always visible even when 6d is Negative. |
| 6g | Chest X-ray Results | Text (full width) | None |
| 6h | Check one of the following: | Choice | Active TB Disease · Latent TB Infection · No Evidence of TB Infection or Disease |

### 7–12. Diagnosis and condition blocks

Sections 7, 8, 10, 11 and 12 share one four-part pattern. Substitute the section number for *n*:

| Ref | Field | Control | Notes |
|---|---|---|---|
| *n* | The condition itself | Text (multi) | Prefill from PCC diagnoses / allergy list where one exists |
| *n*a | Treatment / medication (type and dosage) / equipment | Text (multi) | Prefill from PCC medication list where one exists |
| *n*b | Can patient manage own treatment / medication / equipment? | Choice | Yes · No |
| *n*c | If not, what type of medical supervision is needed? | Text (multi) | Always visible; not conditional on *n*b = No |

| Section | Title | Prefill source |
|---|---|---|
| 7 | Primary Diagnosis | PCC primary diagnosis and its treatment. *Sample* today. |
| 8 | Secondary Diagnosis(es) | PCC secondary diagnoses, one per line. *Sample* today. |
| 10 | Contagious / Infectious Disease | PCC infection control record. *Sample* "None" today. |
| 11 | Allergies | PCC allergy list. *Sample* today; allergies are not on the confirmed PCC field list. |
| 12 | Other Conditions | None — manual |

### 9. Check if Applicable to 7 or 8 Above

| Checkbox | Printed definition |
|---|---|
| Mild Cognitive Impairment | Cognitive abilities in a conditional state between normal aging and dementia. |
| Dementia | Loss of intellectual function sufficient to interfere with activities of daily living or social / occupational activities. |

### 13–16. Yes/No assessment grids

Each row is answered Yes or No and carries a free-text *Explain* field. §13 rows additionally carry an *Assistive Device* field. No row is answered by default and none is enforced.

| Section | Rows |
|---|---|
| 13. Physical Health Status (13 rows · with Assistive Device) | a Auditory Impairment · b Visual Impairment · c Wears Dentures · d Wears Prosthesis · e Special Diet · f Substance Abuse Problem · g Use of Alcohol · h Use of Cigarettes · i Bowel Impairment · j Bladder Impairment · k Motor Impairment / Paralysis · l Requires Continuous Bed Care · m History of Skin Condition or Breakdown |
| 14. Mental Condition (11 rows) | a Confused / Disoriented · b Inappropriate Behavior · c Aggressive Behavior · d Wandering Behavior · e Sundowning Behavior · f Able to Follow Instructions · g Depressed · h Suicidal / Self-Abuse · i Able to Communicate Needs · j At Risk if Allowed Direct Access to Personal Grooming and Hygiene Items · k Able to Leave Facility Unassisted |
| 15. Capacity for Self-Care (5 rows) | a Able to Bathe Self · b Able to Dress / Groom Self · c Able to Feed Self · d Able to Care for Own Toileting Needs · e Able to Manage Own Cash Resources |
| 16. Medication Management (6 rows) | a Able to Administer Own Prescription Medications · b Able to Administer Own Injections · c Able to Perform Own Glucose Testing · d Able to Administer Own PRN Medications · e Able to Administer Own Oxygen · f Able to Store Own Medications |

> **OQ-A4 · Assessment prefill** Sections 13–16 are 35 clinical judgements the physician must make. SNFs hold much of this in the MDS assessment. Should any of it prefill from PCC, and if so which rows and with what "last assessed" provenance shown to the physician?

### 17. Ambulatory Status

Printed definitions: *nonambulatory* — unable to leave a building unassisted under emergency conditions, including a person who depends on mechanical aids such as crutches, walkers and wheelchairs. *Bedridden* — requires assistance with turning or repositioning in bed. An illness or recovery is temporary if it will last 14 days or less.

| Ref | Field | Control | Options / notes |
|---|---|---|---|
| 17a1 | Able to independently transfer to and from bed | Choice | Yes · No |
| 17a2 | For purposes of a fire clearance, this person is considered | Choice | Ambulatory · Nonambulatory · Bedridden |
| 17b | If nonambulatory, this status is based upon | Choice | Physical Condition · Mental Condition · Both |
| 17c | If bedridden — Illness | Text (full width) | Always visible; not conditional on 17a2 |
| 17c | If bedridden — Recovery from Surgery | Text | — |
| 17c | If bedridden — Other | Text | — |
| 17d1 | Bedridden status expected to persist (number of days) | Text | Numeric in practice; not validated |
| 17d2 | Estimated date illness or recovery is expected to end | Text | Date in practice; no picker, no validation |
| 17d3 | If illness or recovery is permanent, please explain | Text (multi) | — |
| 17e | Is resident receiving hospice care? | Choice | No · Yes |
| 17e | If receiving hospice care, specify the terminal illness | Text (full width) | Always visible |

### 18–19. Overall status and comments

| Ref | Field | Control | Options / prefill |
|---|---|---|---|
| 18 | Overall physical health status | Choice | Good · Fair · Poor |
| 19 | Comments | Text (multi, full width) | Prefilled with a standard sentence referencing the attached discharge plan and medication list. *Sample* — confirm whether a standard sentence should ship, and whether it is editable. |

### 20–24. Physician

| Ref | Field | Control | Prefill |
|---|---|---|---|
| 20 | Physician's Name and Address (print) | Text | Attending physician from the plan. Name only — the address is not held anywhere. |
| 21 | Telephone | Text | None — candidate for a physician directory |
| 22 | Length of Time Resident Has Been Your Patient | Text | None |
| 23 | Physician's Signature | Text | None. Plain text, **not** routed through the mobile Pending Sign queue (**OQ-54**). |
| 24 | Date | Text | None |

## 4. Field count summary

**Corrected for v0.2.** The previous draft's totals (47 text fields; 16 prefilled, of which 12 samples) were stale relative to the current prototype code. Recounted directly by enumerating every `tf(...)` call site in the source (53 call sites total, confirmed by direct search) and classifying each by its prefill argument:

| Control type | Count | Prefilled today |
|---|---|---|
| Text fields (counted by the header pill) | **53** | **21**, of which **6** are mapped from a real source (facility name/address/city/zip parsed from the Step 1 destination address; resident name; physician name) and **15** are demo-fixture sample values with no real source yet |
| Choice groups | 12 | None — all start unselected |
| Yes/No rows (§13–16) | 35 | None |
| Explain fields (one per Yes/No row) | 35 | None |
| Assistive Device fields (§13 only) | 13 | None |
| Checkboxes (§9) | 2 | None |

Roughly 150 inputs, of which about a seventh carry any value today (real or sample). This remains the single heaviest data-entry surface in the workflow and the strongest candidate for PCC-driven prefill. The choice-group, Yes/No-row, explain-field, assistive-device and checkbox counts were re-verified against the same source and are unchanged from the prior draft.

## 5. Open questions raised here

| ID | Question | Priority | Answer |
|---|---|---|---|
| OQ-A1 | Which fields are mandatory, and at which moment are they enforced? | Blocker |  |
| OQ-A2 | Should ALF destinations come from a facility directory rather than a parsed address string? | High |  |
| OQ-A3 | How is the §III resident authorisation actually obtained and recorded? | Blocker |  |
| OQ-A4 | Should §13–16 prefill from MDS / PCC, and how is provenance shown? | High |  |
| OQ-A5 | Should conditional fields (6f, *n*c, 17c–e) hide until their trigger answer is given? | Medium |  |
| OQ-A6 | Who may complete which parts? The form separates licensee (§I), resident (§II–III) and physician (§IV onwards), but the product lets one planner type everything. | Blocker |  |

Carried from the main PRD and still open here: **OQ-51** (auto-populate vs manual), **OQ-52** (fax destination and tracking), **OQ-53** (draft persistence and lock-after-signature), **OQ-54** (signature path). **OQ-20** (which destinations require the form) is resolved in the main PRD — see §1 above.
