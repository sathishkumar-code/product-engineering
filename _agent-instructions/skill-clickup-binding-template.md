# Skill: ClickUp Binding Template

A tracker-specific binding for the Project Manager persona skill. Copy this template
per team/project and fill in the placeholders — don't edit this file itself once
you're customizing for a real project; make a copy.

## Hierarchy mapping

| Local concept                      | ClickUp object                                  |
|-------------------------------------|--------------------------------------------------|
| Product/Team                        | Folder, inside `<SPACE_NAME>`                    |
| Sprint                               | Subfolder under the product/team Folder          |
| Epic (feature/enhancement/bug-group) | List, inside the relevant Sprint subfolder       |
| Story / Task / Bug                  | Task, inside the Epic's List, tagged `story` / `task` / `bug` |

## Statuses
`<STATUS_LIST>` — new items default to `<DEFAULT_STATUS>` (typically Backlog). A
common minimum shape worth starting from: Backlog → Development → Review → QA →
UAT → Done, plus Blocked used alongside a stage rather than replacing it (a
blocked item should still show which stage it was blocked in). Don't set a
further-along status unless work has genuinely started.

## Tags
- Type tags: `story`, `bug`, `task`, `spike`, `tech_debt` — every Task gets exactly
  one. If test scenarios are tracked in the tracker at all, `test_scenario` for the
  scenario level, with actual test cases living in an attached spreadsheet rather
  than one tracker item per case (revisit if per-case tracking becomes necessary).
- Additional tags: `<TAG_LIST>` — check existing tags in the List before inventing
  a new one.

## Release tracking
Most trackers, ClickUp included, don't have a distinct "Release" object separate
from Epics/Lists/Tasks. The standard pattern is a **Custom Field** on Epics and
Tasks (e.g. `Release: 2026-Q4`) rather than an extra hierarchy level nesting Sprints
under a Release folder — verify the field type is actually available on the
project's specific plan tier before relying on it; fall back to a `release-<slug>`
tag using the same pattern as the type tags if it isn't.

## Shared-scope features
If a feature spans multiple products/teams: `<SHARED_SCOPE_POLICY>` — e.g. one
canonical List with cross-tags, vs. duplicated Lists for independently trackable
timelines. Default to canonical + tag; only duplicate on explicit instruction.

## Idempotency
Before creating anything: check the mapping log for the slug, confirm it still
exists in ClickUp (logs can drift if something was deleted manually), only create if
genuinely absent, log immediately after creation.

## Mapping log format
```
## <slug>
- Scope: <product/team tag(s)>
- Folder: <name>
- Sprint subfolder: <name>
- Epic (List): <name> — <clickup list id/link>
- Tasks:
  - <name> — <type tag> — <clickup task id/link>
- Last synced: <date>
- Deleted (if applicable): <date> — <one-line reason, e.g. "prototype removed,
  superseded by Epic/Story creation above">
```
