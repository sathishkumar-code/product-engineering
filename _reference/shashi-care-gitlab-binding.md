# Binding: GitLab-Direct Authoring — Shashi Care

Filled instance of `skill-gitlab-promotion-template.md`. **Supersedes the
promotion model** (PRD/spec/TD/tech-spec drafted in `product-engineering`,
then mechanically copied to GitLab on approval). As of 2026-09-04, per
Sathish's request now that the Hermes `pm`/`sa`/`pjm`/`product-team` profiles
are live in WSL: PM, SA, and PjM author these documents **directly in each
product's GitLab `-docs` repo**, on the working tree of `main`, in their own
local checkout. There is no intermediate copy step and no separate
"promotion" event — the document's home from the first draft is its
GitLab repo. Applies uniformly to all three products (Shashi-Care-Core,
SAL, SNF) and all three repos (Shashi-Care-Core-docs, SAL-docs, SNF-docs).

`product-engineering` (the WSL working-doc root PM/SA/PjM used before this
change) is **frozen** — nothing in this system reads or writes to it going
forward, except the process-definition files described in "What still
copies to product-engineering" below. Its existing SNF/, SAL/, and
shashi-care-core/ document trees stay in place as a historical snapshot;
Sathish owns whatever happens to that folder from here, Process Architect
has no file access there to act on it.

## What authors where, and when it commits

| Document | Authored by | Destination | Commits when |
|---|---|---|---|
| intent.md | PM | `prd/{features,enhancements,bugs}/<slug>/intent.md` | On Sathish's go-ahead (see "Commit mechanics") |
| PRD / ER / BR | PM | `prd/{features,enhancements,bugs}/<slug>/prd-<slug>.md` (or `enhancement-request-<slug>.md` / `bug-report-<slug>.md`) | `status: approved` |
| spec.md | PM | `prd/{features,enhancements,bugs}/<slug>/spec.md` | Its own `status: approved` (only reachable once the source PRD is at least approved) |
| Technical Design | SA | `architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md` | Its own `status: approved` |
| SA-comments-<slug>.md | SA | `architecture/{features,enhancements,bugs}/<slug>/SA-comments-<slug>.md` | Each time SA finishes a review round (no status field — commits on SA reporting that round's write complete, same go-ahead rule) |
| tech-spec-<slug>.md | SA | `architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md` | Its own `status: approved` |
| Roadmap | PM | `00_roadmap/<product>-roadmap.xlsx` | On PM reporting the update complete — ongoing reference material, no approval-gate field (resolved 2026-09-04) |
| Release Plan | PjM / SA (whoever drafts it today) | `releases/<release-slug>.xlsx` | `status: approved` |
| Prototype (full export) | PM | `prototypes/<category>-<slug>/` | Its own `repo_status` in `prototype-meta.md` |
| _as-built PRD-side (personas-and-roles, prd-senior-living, prd-skilled-nursing, README, modules, codebase-analysis) | PM | `_as-built/prd/...` | On PM reporting the update complete — ongoing reference material, no approval-gate field |
| _as-built architecture-side (architecture-*, data-schema, technical-debt, ADRs) | SA | `_as-built/architecture/...` | Same as above, SA |
| Compliance register + source files | SA | `compliance/...` | Same as _as-built |
| PCC integration agreements + API contracts | SA | `integrations/pcc/...` | Same as _as-built |
| mapping-log.md | PjM | `tracker-sync/mapping-log.md` | Same transaction as the ClickUp item creation it logs |

Epics/Stories and test-scenarios still never live in GitLab — developers use
ClickUp directly (unchanged).

## Commit mechanics — replaces the old "product-team promotes" step

1. The specialist (PM or SA) writes file content into the local checkout's
   working tree. This is drafting, not a git operation — no add/commit/push
   happens yet, and it never happens automatically just because a draft
   looks finished.
2. Sathish reviews the content (in the working tree, or however Hermes
   surfaces the diff) and, for approval-gated documents, sets that
   document's own `status`/`Status` field to `Approved` — exactly like
   today. For ongoing reference material with no status field (SA-comments,
   as-built docs, compliance, integrations, mapping-log), Sathish's
   go-ahead is his explicit confirmation that a given update is ready to
   land — never inferred from the specialist simply finishing its output.
3. **`product-team`**, the Hermes orchestrator, is the only actor that runs
   the actual `git add` / `commit` / `push` to `main` — same
   single-mechanical-actor rule as before, just committing in place instead
   of copying to a new location. It verifies the approval signal (status
   field, or Sathish's explicit confirmation) before committing, and
   independently verifies afterward that the commit landed — via `git log`
   or an equivalent check — before treating the workflow step as advanced.
4. **No feature branch for PM/SA's own authoring.** They write and commit
   straight onto `main`'s working tree — there is no per-document branch or
   MR for this pathway. (Team-submitted Technical Designs are the
   exception — see below, unchanged.)
5. **Re-commits.** Same rule as before under the old promotion model: PRD,
   TD, and Release Plan re-commit on any edit after their first approval;
   `spec.md` and `tech-spec-<slug>.md` re-commit on any edit, or whenever
   their source document is revised.
6. A failed or ambiguous commit (working tree dirty in an unexpected way,
   merge conflict, unclear which revision is live) is a workflow blocker,
   escalated to Sathish directly rather than guessed through.

There is no `promotion-log.md` any more — each repo's own git history is
the record of what changed and when. (PjM's `mapping-log.md`, its own
ClickUp idempotency/creation log, is unaffected and unrelated to this.)

## Target structure — per repo

```
<GitLab repo>/
├── 00_roadmap/                   # NEW as of 2026-09-04 — split per product
│   └── <product>-roadmap.xlsx
├── releases/
│   └── <release-slug>.xlsx
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
├── architecture-submissions/     # team-submitted TDs awaiting review — unchanged, see below
│   └── <category>-<slug>/
├── prototypes/                   # full exports, retained permanently
│   │                              # (no deletion step, as of 2026-09-04)
│   └── <category>-<slug>/
├── _as-built/                    # NEW — ongoing reference, no approval gate
│   ├── prd/                      # PM-maintained (mirrors old 02_prd/_as-built/)
│   │   ├── README.md
│   │   ├── prd-senior-living.md
│   │   ├── prd-skilled-nursing.md
│   │   ├── personas-and-roles.md
│   │   ├── modules/
│   │   └── _codebase-analysis/
│   └── architecture/             # SA-maintained (mirrors old 03_architecture/_as-built/)
│       ├── architecture-<scope>.md   (one per system area, same set as today)
│       ├── data-schema.md
│       ├── technical-debt.md
│       └── adr/
├── compliance/                   # NEW
│   ├── <product>-compliance-register.md
│   └── (source docx/xlsx files)
├── integrations/                 # NEW
│   └── pcc/
│       ├── agreements/
│       └── api-contracts/
└── tracker-sync/                 # NEW — PjM's tracker bookkeeping
    └── mapping-log.md
```

`category` is `feature` / `enhancement` / `bug`. `prd/` and `architecture/`
nest by category and slug, matching the shape this system has always used.
`00_roadmap/`, `_as-built/`, `compliance/`, `integrations/`, and
`tracker-sync/` are new top-level folders — this material used to live only
in `product-engineering` (never promoted, since it isn't gated by document
approval); now that `product-engineering` is frozen, it needs a home, and
the GitLab repo is it.

## Repos

Unchanged: one per doc-tree folder, `-docs` suffix — **Shashi-Care-Core-docs**,
**SAL-docs**, **SNF-docs**.

## Team-submitted Technical Designs — unchanged

This pathway keeps its existing shape exactly, since the actor is the dev
team, not a Hermes persona:
- Dev team designs externally, submits via an MR into `architecture-submissions/
  <category>-<slug>/`, any format. Single permanent branch `main`; per-MR
  branches are short-lived, created and deleted per submission. Developers
  have write access (branches/MRs); merges to `main` are gated by Sathish or
  a team lead.
- SA reads the submission from that same local checkout (now read-write for
  SA's own authoring elsewhere, but SA never edits the submission itself or
  commits into its branch), writes the verdict to that slug's
  `SA-comments-<slug>.md` in `architecture/{features,enhancements,bugs}/<slug>/`.
  SA always asks Sathish whether to convert the submission to the standard
  template — never assumes.
- **Once approved and merged into `architecture-submissions/`**, SA (not
  `product-team`) writes the reviewed Technical Design directly onto `main`
  at `architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md` — an
  ordinary commit under the same "Commit mechanics" rules above
  (`product-team` verifies and runs it once Sathish confirms). This replaces
  the old "`product-team` promotes it back into `product-engineering`"
  language — there's no `product-engineering` copy any more, just the one
  authoritative `TD-<slug>.md` in `architecture/`.
- The team may still comment in their own Notion copy, ad hoc, alongside or
  instead of the GitLab MR.

## Mechanics — who touches git, and how

`product-team` is still the sole actor that runs `git add`/`commit`/`push`
for every PM/SA/PjM-authored document (see "Commit mechanics" above) —
never on a specialist's own say-so, always verified against Sathish's
approval signal first, and independently verified afterward. The one thing
that changed from the old promotion model is *what* the commit does: it
commits the file already sitting in the working tree, in place, rather than
copying it from `product-engineering` to a different repo.

Sathish's and the team lead's role throughout stays approval, never the git
operations themselves — the document's own status field (or, for
non-gated reference material, an explicit confirmation), or the MR merge
for a team-submitted TD. PM never commits into any checkout. SA's checkouts
are now read-write for its own authoring (`architecture/`, `_as-built/`,
`compliance/`, `integrations/`) but SA itself still never runs `git commit`
— that stays `product-team`'s job, same authorship boundary as before, just
narrower in scope (drafting vs. committing, not product-engineering vs.
GitLab).
