# Test Scenarios Template

Scenario-level document — happy path, edge cases, negative cases — tied to specific
stories by ID. Product Manager authors the functional scenarios; System Architect
adds technical scenarios in a clearly separated section, the same pattern used for
epics/stories. QA lead's role is review and approval, not authorship — the goal is
speed: PM/SA draft, QA verifies, rather than QA starting from a blank sheet.

```markdown
# Test Scenarios: <slug>

| Field | Value |
|---|---|
| Source Epic/Stories | <path to epics-stories.md> |
| Product | Core / SAL / SNF |
| qa_status | Pending QA review / Approved / Changes requested |
| Test cases workbook | `test-cases.xlsx` (same folder — see the Excel template) |

## Functional scenarios (Product Manager)

### Scenario <ID>: <name>
- **Type**: Happy path / Edge case / Negative case
- **Linked story**: <story ID>
- **Description**: what this scenario verifies, in plain terms
- **Covered in**: `test-cases.xlsx`, PM-Test-Cases sheet, case IDs <...>

(repeat per scenario)

## Technical scenarios (System Architect)
Kept visually separate from the functional scenarios above — not blended in.
Performance, security, data-integrity, and other non-user-facing scenarios the PRD
wouldn't surface but the technical design implies.

### Scenario <ID>: <name>
- **Type**: Performance / Security / Data integrity / Other
- **Linked technical design**: <TD path>
- **Description**:
- **Covered in**: `test-cases.xlsx`, SA-Technical-Test-Cases sheet, case IDs <...>

(repeat per scenario)

## QA review notes
QA lead's comments go here, not scattered across the case workbook — one place to
see the verdict and any requested changes before `qa_status` moves to Approved.
```
