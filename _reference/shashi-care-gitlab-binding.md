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
`SA-comments-<slug>.md` files also stay `product-engineering`-only; only the
Technical Design itself promotes.

**Prototype retention differs from the other three**: it isn't kept indefinitely.
Project Manager persona deletes the GitLab copy once development is actually
complete (confirmed, logged — see `_reference/shashi-care-doc-tree.md`'s deletion policy),
unlike PRD/TD/Release Plan which stay permanently once promoted.

## Target structure — nested by category and slug, mirroring product-engineering

`product-engineering` groups by slug (a feature's PRD and, once promoted, its
`spec.md`, living together under `02_prd/features/<slug>/`; its TD and
`tech-spec-<slug>.md` together under `03_architecture/features/<slug>/`).
GitLab now mirrors that shape instead of flattening it: a per-slug folder holding only a PRD while its
sibling `spec.md` is conspicuously absent looked broken the old, flattened way,
and splitting the developer-facing pair into a separate folder pulled it away
from the requirements/design pair it belongs next to. Instead:

```
<GitLab repo>/
├── releases/
│   └── <release-slug>.xlsx
├── prd/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── prd-<slug>.md
│           └── spec.md
├── architecture/
│   └── {features,enhancements,bugs}/
│       └── <slug>/
│           ├── TD-<slug>.md
│           └── tech-spec-<slug>.md
├── architecture-submissions/     # team-submitted TDs awaiting review — see below
│   └── <category>-<slug>/
└── prototypes/                   # full exports, deleted post-development
    └── <category>-<slug>/
```

`category` here is `feature` / `enhancement` / `bug` — not to be confused with the
document types (PRD / Technical Design / Release Plan) that gave these folders
their names. `prd/` and `architecture/` now split by category and slug the same
way `product-engineering`'s `02_prd/` and `03_architecture/` do, so `category`
lives in the *path* on both sides — no more filename prefix needed for the
documents that nest (`prd-<slug>.md`, `TD-<slug>.md`, `spec.md`,
`tech-spec-<slug>.md`, all named the same way `product-engineering` already
names them — see `_reference/shashi-care-doc-tree.md`). `releases/` and `prototypes/` stay flat,
category-and-slug in the file or folder name, since neither has a sibling
document to nest alongside.

Every folder here only ever contains what was always meant to be there.

## Repos
One per doc-tree folder, named with a `-docs` suffix: **Shashi-Care-Core-docs**,
**SAL-docs**, **SNF-docs** — not the bare product names. This is the one place the
names diverge from the "same names throughout" rule used everywhere else
(`product-engineering` folders, ClickUp Folders, filenames): GitLab is the one system where a code repo
and a docs repo plausibly sit in the same namespace, and a code repo named `SAL`
already exists — the docs repo needs a name that doesn't collide with it. A
promoted document from `product-engineering/shashi-care-core/...` goes to the
Shashi-Care-Core-docs repo's matching folder, and so on.

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
- Once approved and merged into `architecture/` in GitLab, **`product-team`
  promotes the merged TD back into `product-engineering`'s
  `03_architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md`** — the
  same mechanical, orchestrator-owned promotion described below, run in
  reverse. Sathish's or the team lead's role is the MR-merge approval, not
  the copy.

## Mechanics — `product-team` performs every promotion mechanically
`product-team`, the Hermes orchestrator profile, is the sole actor that runs the
copy step for every promotion in this system: `product-engineering`→GitLab
(PRD, `spec.md`, Technical Design, `tech-spec-<slug>.md`, Release Plan,
Prototype) and, for team-submitted TDs, the reverse GitLab→`product-engineering`
direction. It never promotes on a
specialist's own say-so — only once the document's own `status`/`Status` field
(or, for the prototype, `prototype-meta.md`'s `repo_status`) reads approved
does it run the copy, then independently verifies the artifact landed
correctly before treating the workflow as advanced — see
`_reference/PROCESS-WALKTHROUGH.md`'s "Document promotion" and "Orchestration
and verification" sections for the full mechanics and the escalation path when
a promotion fails or is ambiguous.

Sathish's and the team lead's role throughout is approval — the document's own
status field, or the MR merge for a team-submitted TD — never the Git
operations themselves. Neither PM nor SA ever commits into a GitLab checkout;
System Architect's own checkouts stay explicitly read-only (see
`shashi-care-sa-config.md`).

## Promotion log
`<folder>/06_gitlab-sync/promotion-log.md`, one per product folder, covering
every promotable type (add a `Document:` field per entry to distinguish PRD /
Technical Design / Release Plan / Prototype / Spec / Tech-spec — the log
format in the generic template already has this field), plus a `Deleted:`
field for the prototype's eventual removal. `product-team` writes this log as
part of running the promotion, not as a separate manual step.
