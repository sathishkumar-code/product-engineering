# Binding: GitLab Promotion — Shashi Care

Filled instance of `skill-gitlab-promotion-template.md`.

## What promotes, and its trigger

| Document | Promotes when | Re-promotes on |
|---|---|---|
| PRD | `status: approved` | Any edit after approval |
| Technical Design | Its own `status: approved` (SA's sign-off) — not tied to the PRD's approval, since a TD can be added or revised after the PRD already promoted | Any edit after approval |
| Release Plan | `status: approved` — same flow as the PRD, not the old Planning/In-progress/Shipped gate | Any edit after approval |
| Prototype (full export) | `prototype-meta.md`'s own `repo_status`, independent of the PRD's — a prototype can be re-promoted without the PRD changing | Any re-export |
| Spec | Its own `status: approved`, only draftable once the source PRD is at least approved | Any edit, or whenever the source PRD is revised (re-derive first, then re-promote) |
| Tech-spec | Its own `status: approved`, only draftable once the source TD is at least approved | Any edit, or whenever the source TD is revised (re-derive first, then re-promote) |

Epics/Stories and test-scenarios never promote — developers use ClickUp directly
for those (see the ID link-back in `skill-pjm-discipline.md`). SA's
`SA-comments-<slug>.md` files also stay Drive-only; only the Technical Design itself
promotes.

**Prototype retention differs from the other three**: it isn't kept indefinitely.
Project Manager persona deletes the GitLab copy once development is actually
complete (confirmed, logged — see `_reference/shashi-care-doc-tree.md`'s deletion policy),
unlike PRD/TD/Release Plan which stay permanently once promoted.

## Target structure — flattened by document, not mirrored from Drive

Drive groups by slug (a feature's PRD, epics-stories, tests, and later its TD, all
living together under `02_prd/features/<slug>/` and `03_architecture/`). GitLab
does **not** mirror that shape — a per-slug folder in GitLab holding only a PRD
while its sibling epics-stories/tests are conspicuously absent looks broken. Instead:

```
<GitLab repo>/
├── releases/
│   └── <release-slug>.xlsx
├── prd/
│   └── <category>-<slug>-PRD.md
├── architecture/
│   └── <category>-<slug>-TD.md
├── specs/                         # developer-facing pair, see below
│   ├── <category>-<slug>-spec.md
│   └── <category>-<slug>-tech-spec.md
├── architecture-submissions/     # team-submitted TDs awaiting review — see below
│   └── <category>-<slug>/
└── prototypes/                   # full exports, deleted post-development
    └── <category>-<slug>/
```

`category` here is `feature` / `enhancement` / `bug` — not to be confused with the
document types (PRD / Technical Design / Release Plan) that gave these folders
their names. Both `prd/` and `architecture/` are flat across categories — unlike Drive's
`03_architecture/`, which now splits by category and slug the same way `02_prd/`
does — so category goes in the filename here specifically because GitLab stays
flat; see `_reference/shashi-care-doc-tree.md` for the full Drive-side convention.

Every folder here only ever contains what was always meant to be there.

## Repos
One per doc-tree folder, named with a `-docs` suffix: **Shashi-Care-Core-docs**,
**SAL-docs**, **SNF-docs** — not the bare product names. This is the one place the
names diverge from the "same names throughout" rule used everywhere else (Drive
folders, ClickUp Folders, filenames): GitLab is the one system where a code repo
and a docs repo plausibly sit in the same namespace, and a code repo named `SAL`
already exists — the docs repo needs a name that doesn't collide with it. A
promoted document from `shashi-care-core/...` goes to the Shashi-Care-Core-docs
repo's matching folder, and so on.

## Team-submitted Technical Designs
A second, separate pathway alongside Sathish working directly with the SA agent
(see `skill-sa-discipline.md` for both):
- Dev team designs externally, submits via an MR into `architecture-submissions/
  <category>-<slug>/`, any format.
- **Access**: developers have write access (can push branches, open MRs) to these
  repos; merges to `main` are gated by Sathish or a team lead's approval.
  **Single permanent branch — `main`.** No `develop`, no environment branches;
  per-MR branches are short-lived, created and deleted per submission.
- SA agent's Context includes **read-only local checkouts of all three repos**
  specifically to review these submissions — it never commits into the checkout,
  and always asks Sathish whether to convert the submission to the standard
  template rather than assuming.
- Once approved and merged into `architecture/` in GitLab, **Sathish manually
  copies the merged TD back into Drive's
  `03_architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md`** —
  matching every other GitLab↔Drive step in this workflow, all currently manual
  by choice, not automated.

## Mechanics — current phase: manual
Sathish runs the copy step from Drive into the relevant GitLab checkout for
Drive→GitLab promotion (PRD, TD, Release Plan, Prototype), reviews the diff,
commits and pushes directly. The reverse direction — team-submitted TDs — moves via
MR instead, gated by approval rather than a direct push, but is equally manual in
the sense that nothing here is automated end-to-end yet.

## Promotion log
`<folder>/06_gitlab-sync/promotion-log.md`, one per product folder, now covering
all four promotable types (add a `Document:` field per entry to distinguish PRD /
Technical Design / Release Plan / Prototype — the log format in the generic
template already has this field), plus a `Deleted:` field for the prototype's
eventual removal.
