# Binding: ClickUp — Shashi Care

Filled instance of `skill-clickup-binding-template.md`.

## Hierarchy mapping

| Local concept                      | ClickUp object                                            |
|--------------------------------------|-------------------------------------------------------------|
| Product folder (Core, SAL, SNF)      | Folder, inside the shared Space with other products         |
| Sprint                               | Subfolder under the relevant product Folder                  |
| Epic (feature/enhancement/bug-group) | List, inside the relevant Sprint subfolder                  |
| Story / Task / Bug                  | Task, inside the Epic's List, tagged `story` / `task` / `bug` |

Three ClickUp Folders now — **Shashi-Care-Core**, **SAL**, **SNF** — matching the
three doc-tree folders one-to-one. A shared feature filed under
`shashi-care-core/` in the docs gets its Epic created under the **Shashi-Care-Core**
ClickUp Folder, not duplicated into SAL's or SNF's.

## Statuses
**Backlog → Development → Review → QA → UAT → Done**, plus **Blocked** (used
alongside a stage, not instead of it — a blocked item still shows which stage it
was blocked in). New items default to **Backlog**.

## Tags
- Type tags: `story`, `bug`, `task`, `spike`, `tech_debt`, `test_scenario`. Every
  ClickUp task gets exactly one type tag — this is the workaround for Basic plan
  not supporting native custom Task Types.
- `test_scenario` is used for scenario-level tracking only, not per-case — the
  actual test cases live in the attached `test-cases.xlsx` workbook (see below),
  not as individual ClickUp tasks. Revisit this if per-case tracking in ClickUp
  itself becomes necessary later.
- `tech_debt` is used only once a Technical Debt Register item actually gets
  scheduled into a sprint — the register itself stays Drive-only until then.
- Additional: `SAL`, `SNF`, `core`, plus priority tags as needed. A Core-filed item
  doesn't need a `shared` tag anymore — its own Folder already says that; use `SAL`/
  `SNF` tags on a Core item only if it's useful to note which products currently
  consume it.

## Release tracking
ClickUp has no distinct Release object — the standard pattern (confirmed against
current ClickUp guidance, not assumed) is a **Custom Field** on Epics and Tasks,
e.g. `Release: 2026-Q4-SNF`, rather than an extra folder level nesting Sprints under
a Release. Verify Custom Fields are actually available on this Basic-tier
workspace before relying on this; if they're not, fall back to a
`release-<slug>` tag using the same pattern already proven to work for type tags.

## Test scenario attachment
A `test_scenario`-tagged task gets the corresponding `test-cases.xlsx` workbook
attached directly to it via the tracker's file-attachment tools — see
`skill-pjm-discipline.md`. Only re-attach when the workbook has actually changed.

## Cross-folder features
Since Core now has its own Folder, "shared SAL/SNF feature" no longer needs a
duplicate-vs-canonical decision the way it did with two folders — it simply lives in
Core. The remaining judgment call is narrower: if a Core feature's *implementation*
genuinely diverges between SAL and SNF (not just scope, but actual different
technical work), Sathish confirms whether that divergence gets its own Epic under
the relevant product Folder in addition to the Core Epic, rather than defaulting to
one or the other.

## Mapping log
One per folder: `<folder>/05_clickup-sync/mapping-log.md`. Format per the template.
