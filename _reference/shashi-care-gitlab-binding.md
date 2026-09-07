# Binding: GitLab-Direct Authoring — Shashi Care

Filled instance of `framework/_agent-instructions/skill-code-repo-promotion-template.md`. PM, SA, and PjM
author PRD/spec/TD/tech-spec/release-plan/roadmap/prototype/as-built/
compliance/integrations/tracker-sync/readiness/build content **directly in
each product's GitLab `-docs` repo**, on the working tree of `main`, in a
local checkout. There is no intermediate draft location and no promotion
step — a document's home from its first draft is its GitLab repo. Applies
uniformly to all three products (Shashi-Care-Core, SAL, SNF) and all three
repos (Shashi-Care-Core-docs, SAL-docs, SNF-docs).

PM, SA, and PjM (one shared Hermes instance) work from **one checkout per
product**. Developer, QA, and DevOps (one Hermes instance per code repo) each
work from **their own separate checkout** of the relevant product's `-docs`
repo — not shared with PM/SA/PjM's checkout, and not shared with each other.
`product-engineering` holds none of this — see
`shashi-care-process-architect-config.md`'s "Hermes as primary host" for what
`product-engineering` actually contains (config/skill/reference only, never
product documents).

## What authors where, and when it commits

| Document                                                                                                            | Authored by                                                         | Destination                                                                                                                 | Commits when                                                                                              |
| ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| intent.md                                                                                                           | PM                                                                  | `prd/{features,enhancements,bugs}/<slug>/intent.md`                                                                       | On Sathish's go-ahead (see "Commit mechanics")                                                            |
| PRD / ER / BR                                                                                                       | PM                                                                  | `prd/{features,enhancements,bugs}/<slug>/prd-<slug>.md` (or `enhancement-request-<slug>.md` / `bug-report-<slug>.md`) | `status: approved`                                                                                      |
| spec.md                                                                                                             | PM                                                                  | `prd/{features,enhancements,bugs}/<slug>/spec.md`                                                                         | Its own`status: approved` (only reachable once the source PRD is at least approved)                     |
| Technical Design                                                                                                    | SA                                                                  | `architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md`                                                           | Its own`status: approved`                                                                               |
| SA-comments-<slug></slug>.md                                                                                        | SA                                                                  | `architecture/{features,enhancements,bugs}/<slug>/SA-comments-<slug>.md`                                                  | Each time SA finishes a review round (no status field)                                                    |
| tech-spec-<slug></slug>.md                                                                                          | SA                                                                  | `architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md`                                                    | Its own`status: approved`                                                                               |
| epics-stories.md                                                                                                    | PM (drafts) + SA (Round 2 review, same file)                        | `readiness/{features,enhancements,bugs}/<slug>/epics-stories.md`                                                          | Each time PM or SA finishes a round on it (no status field)                                               |
| test-scenarios.md, test-cases.xlsx                                                                                  | PM (drafts) + SA (adds technical scenarios/cases to the same files) | `readiness/{features,enhancements,bugs}/<slug>/test-scenarios.md` and `test-cases.xlsx`                                 | Each time PM or SA finishes a round; QA's`qa_status` field gates execution start, not the commit itself |
| implementation-note-<slug></slug>.md                                                                                | Developer                                                           | `build/{features,enhancements,bugs}/<slug>/implementation-note-<slug>.md`                                                 | On Developer reporting the note complete (no status field)                                                |
| qa-execution-report-<slug></slug>.md                                                                                | QA                                                                  | `build/{features,enhancements,bugs}/<slug>/qa-execution-report-<slug>.md`                                                 | On QA reporting the report complete (no status field)                                                     |
| Roadmap                                                                                                             | PM                                                                  | `roadmap/<product>-roadmap.xlsx`                                                                                       | On PM reporting the update complete (no status field)                                                     |
| Release Plan                                                                                                        | PjM / SA (whoever drafts it)                                        | `releases/<product>-release-plan.xlsx` (one workbook per repo, one tab per release)                                       | `status: approved`                                                                                      |
| deployment-record-<release-slug></release>.md                                                                       | DevOps                                                              | `releases/deployment-record-<release-slug>.md`                                                                            | On DevOps reporting the record complete — one per deployment event, not per slug                         |
| Prototype (full export)                                                                                             | PM                                                                  | `prototypes/<category>-<slug>/`                                                                                           | Its own`repo_status` in `prototype-meta.md`                                                           |
| _as-built PRD-side (personas-and-roles, prd-senior-living, prd-skilled-nursing, README, modules, codebase-analysis) | PM                                                                  | `_as-built/prd/...`                                                                                                       | On PM reporting the update complete (no status field)                                                     |
| _as-built architecture-side (architecture-*, data-schema, technical-debt, ADRs)                                     | SA                                                                  | `_as-built/architecture/...`                                                                                              | Same as above, SA                                                                                         |
| Compliance register + source files                                                                                  | SA                                                                  | `compliance/...`                                                                                                          | Same as _as-built                                                                                         |
| PCC integration agreements + API contracts                                                                          | SA                                                                  | `integrations/pcc/...`                                                                                                    | Same as _as-built                                                                                         |
| mapping-log.md                                                                                                      | PjM                                                                 | `tracker-sync/mapping-log.md`                                                                                             | Same transaction as the ClickUp item creation it logs                                                     |

Epics/Stories and test material live in GitLab as *documents*, in
`readiness/`. ClickUp separately holds the resulting Epic/Story/Task tracker
items and tracks execution status (Backlog → Development → Review → QA →
UAT → Done) — the document and the tracker item are different artifacts.
PjM creates the ClickUp items (see `shashi-care-clickup-binding.md`) and
writes `tracker_id` back into `epics-stories.md` — that write is itself an
edit needing its own re-commit, same as any other edit to the file.

## Commit mechanics

1. The specialist (PM, SA, Developer, QA, or DevOps) writes file content
   into its checkout's working tree. This is drafting, not a git operation —
   no add/commit/push happens, and it never happens automatically just
   because a draft looks finished.
2. Sathish reviews the content (in the working tree, or however Hermes
   surfaces the diff) and, for approval-gated documents, sets that
   document's own `status`/`Status` field to `Approved`. For ongoing
   reference material with no status field (SA-comments, as-built docs,
   compliance, integrations, mapping-log, readiness/build content), Sathish's
   go-ahead is his explicit confirmation that a given update is ready to
   land — never inferred from the specialist simply finishing its output.
3. **`product-team`**, the Hermes orchestrator, is the only actor that runs
   the actual `git add` / `commit` / `push` to `main`. It verifies the
   approval signal (status field, or Sathish's explicit confirmation) before
   committing, and independently verifies afterward that the commit landed —
   via `git log` or an equivalent check — before treating the workflow step
   as advanced.
4. **No feature branch for any persona's own authoring.** Every persona
   writes and commits straight onto `main`'s working tree — there is no
   per-document branch or MR for this pathway. (Team-submitted Technical
   Designs are the exception — see below.)
5. **Re-commits.** PRD, TD, and Release Plan re-commit on any edit after
   their first approval; `spec.md` and `tech-spec-<slug>.md` re-commit on
   any edit, or whenever their source document is revised.
6. A failed or ambiguous commit (working tree dirty in an unexpected way,
   merge conflict, unclear which revision is live) is a workflow blocker,
   escalated to Sathish directly rather than guessed through.

Each repo's own git history is the record of what changed and when. (PjM's
`mapping-log.md`, its own ClickUp idempotency/creation log, is unrelated to
this.)

## Target structure — per repo

```
<GitLab repo>/
├── roadmap/
│   └── <product>-roadmap.xlsx
├── releases/
│   ├── <product>-release-plan.xlsx   # one workbook per repo, one tab per release
│   └── deployment-record-<release-slug>.md
├── prd/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── intent.md
│           ├── prd-<slug>.md          (or enhancement-request-<slug>.md / bug-report-<slug>.md)
│           └── spec.md
├── architecture/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── TD-<slug>.md
│           ├── SA-comments-<slug>.md
│           └── tech-spec-<slug>.md
├── readiness/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── epics-stories.md
│           ├── test-scenarios.md
│           └── test-cases.xlsx
├── build/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── implementation-note-<slug>.md
│           └── qa-execution-report-<slug>.md
├── architecture-submissions/     # team-submitted TDs awaiting review — see below
│   └── <category>-<slug>/
├── prototypes/                   # full exports, retained permanently, no deletion step
│   └── <category>-<slug>/
├── _as-built/                    # ongoing reference, no approval gate
│   ├── prd/                      # PM-maintained
│   │   ├── README.md
│   │   ├── prd-senior-living.md
│   │   ├── prd-skilled-nursing.md
│   │   ├── personas-and-roles.md
│   │   ├── modules/
│   │   └── _codebase-analysis/
│   └── architecture/             # SA-maintained
│       ├── architecture-<scope>.md   (one per system area)
│       ├── data-schema.md
│       ├── technical-debt.md
│       └── adr/
├── compliance/
│   ├── <product>-compliance-register.md
│   └── (source docx/xlsx files)
├── integrations/
│   └── pcc/
│       ├── agreements/
│       └── api-contracts/
└── tracker-sync/
    └── mapping-log.md
```

`category` is `feature` / `enhancement` / `bug`. `prd/`, `architecture/`,
`readiness/`, and `build/` all nest by category and slug — a slug only needs
to be unique *within its own repo*, not globally; a shared cross-product
feature goes under Shashi-Care-Core-docs instead of picking one product
arbitrarily. `releases/` and `prototypes/` stay flat (category-and-slug in
the file or folder name), since neither has a sibling document to nest
alongside.

## Repos

One per product, `-docs` suffix — **Shashi-Care-Core-docs**, **SAL-docs**,
**SNF-docs**. The suffix distinguishes each docs repo from the code repo
sharing the same product name; every other naming convention stays as-is.

## Team-submitted Technical Designs

This pathway is the one place the actor is the dev team, not a Hermes
persona:

- Dev team designs externally, submits via an MR into `architecture-submissions/ <category>-<slug>/`, any format. Single permanent branch `main`; per-MR
  branches are short-lived, created and deleted per submission. Developers
  have write access (branches/MRs); merges to `main` are gated by Sathish or
  a team lead.
- SA reads the submission from its own checkout (read-write for SA's own
  authoring elsewhere, but SA never edits the submission itself or commits
  into its branch), writes the verdict to that slug's `SA-comments-<slug>.md`
  in `architecture/{features,enhancements,bugs}/<slug>/`. SA always asks
  Sathish whether to convert the submission to the standard template — never
  assumes.
- **Once approved and merged into `architecture-submissions/`**, SA (not
  `product-team`) writes the reviewed Technical Design directly onto `main`
  at `architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md` — an
  ordinary commit under the same "Commit mechanics" above (`product-team`
  verifies and runs it once Sathish confirms).
- The team may still comment in their own Notion copy, ad hoc, alongside or
  instead of the GitLab MR.

## Mechanics — who touches git, and how

`product-team` is the sole actor that runs `git add`/`commit`/`push` for
every PM/SA/PjM/Developer/QA/DevOps-authored document (see "Commit
mechanics" above) — never on a specialist's own say-so, always verified
against Sathish's approval signal first, and independently verified
afterward.

Sathish's and the team lead's role throughout stays approval, never the git
operations themselves — the document's own status field (or, for non-gated
reference material, an explicit confirmation), or the MR merge for a
team-submitted TD. No persona ever runs `git commit` itself — SA's, PM's,
PjM's, Developer's, QA's, and DevOps's checkouts are all read-write for
their own authoring, but committing is `product-team`'s job alone, in every
case.

GitLab checkout access is not yet confirmed for any persona specifically —
each config's "Access (Hermes)" section flags this; escalate to Sathish
rather than assuming access exists.
