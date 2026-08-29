# Design Standards — Shashi Care Prototypes

Instantiated from `skill-prototype-authoring-standards.md`. Apply when starting any
new Claude Design project intended to become a Shashi Care feature, enhancement, or
bug PRD.

## How to apply — one honest gap to check yourself
I can't confirm from current Claude Design documentation whether a project carries
persistent custom instructions the way a Cowork project's Instructions field does —
the documented mechanism is attaching context (a design system, screenshots, a
codebase) rather than a separate standing-instructions field. Until you've verified
which applies on your plan: paste this file's content (or the generic skill's) as
the first message / attached context when starting each new prototype project. If
your org's Claude Design admin settings do support baking standards in at the
design-system level (Team/Enterprise), that would remove the need to re-paste this
per project — worth checking directly rather than assuming either way.

## Healthcare-specific edge case categories to always seed
- **Status/state model** — one seeded example per state (Draft / Submitted / Under
  Review / Approved / etc., whichever model this feature uses), not just the common
  ones.
- **Age/date boundaries** — cutoffs relevant to eligibility, billing, or care-plan
  timing (exact boundary ages, date-of-service edge cases).
- **Role/permission edge cases** — Nurse vs. Administrator vs. Facility Admin views,
  whichever roles this feature involves — seed at least one example per role's
  distinct view/permission.
- **Cross-product overlap** — if this feature might apply to both SAL and SNF, seed
  examples that would actually surface a difference between the two, not just one
  product's data.

## Tie-in to downstream conventions
- `// BUSINESS RULE:` comments become the primary source the Product Manager
  persona checks during its mandatory cross-check step — see the updated step in
  `skill-pm-discipline.md`.
- `// PROTOTYPE ONLY:` comments map directly onto the PRD's "Known prototype
  artifacts" section — PM should be able to pull that section almost verbatim from
  these comments rather than inferring what's a shortcut.
- Stable page names support the `prototype_page` tagging convention already used in
  the PRD and Epics/Stories templates.

## Data locality
Already your practice — formalized here as a team standard rather than a personal
habit: all prototype data stays synthetic and local, no live external calls.
