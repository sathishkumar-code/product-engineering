# Doc Tree — Shashi Care

**GitLab is the sole document structure.** PRD, spec.md, Technical Design,
tech-spec, Release Plan, roadmap, prototype, as-built architecture docs,
compliance register, integration docs, tracker-sync material,
epics-stories/test material, and Developer's implementation notes, QA's
execution reports, and DevOps's deployment records are all authored directly
in each product's GitLab `-docs` repo — see `shashi-care-gitlab-binding.md`'s
"Target structure" for the shape (mirrored below) and its "Commit mechanics"
for how content lands on `main`. `product-engineering` holds none of this —
see `shashi-care-process-architect-config.md`'s "Hermes as primary host" for
what `product-engineering` actually contains (config/skill/reference only,
never product documents). This file, together with
`shashi-care-gitlab-binding.md`, is the current authority.

The governance layer this file's own folder lives in —
`_agent-instructions/`, `templates/`, `_reference/` in `shashi-care-docs/` —
is separate: the live source Process Architect maintains and Hermes personas
read via the `product-engineering` config mirror (see
`shashi-care-process-architect-config.md`'s "Hermes copy sync convention").
`shashi-care-docs` in that sense — the repo holding this governance layer —
is distinct from the GitLab product-docs repos described below.

Instantiated from `skill-doc-tree-template.md` and
`skill-gitlab-promotion-template.md` (both generic, reusable templates for
other projects). Shashi Care's actual structure: GitLab-direct authoring, no
promotion step, uniform across all three repos.

---

## The tree — identical across all three repos

Shashi-Care-Core-docs, SAL-docs, and SNF-docs share exactly the same
top-level shape. There is no per-repo variation — where a folder is empty
today (e.g. Core has no release plan yet), it's empty because there's no
content, not because the structure differs.

```
<GitLab repo>/          # Shashi-Care-Core-docs, SAL-docs, or SNF-docs
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
├── prototypes/                   # full exports, retained permanently — no deletion step
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

Epics/Stories and test material live in GitLab, in `readiness/` — the
*document*, not the tracker item. ClickUp holds the resulting Epic/Story/Task
items and tracks execution status — the two are different artifacts; see
`shashi-care-gitlab-binding.md`'s "What authors where" table for the full
distinction.

`category` is `feature` / `enhancement` / `bug`. `prd/`, `architecture/`,
`readiness/`, and `build/` all nest by category and slug — a slug only needs
to be unique
*within its own repo*, not globally; a shared cross-product feature goes
under Shashi-Care-Core-docs instead of picking one product arbitrarily.
`releases/` and `prototypes/` stay flat (category-and-slug in the file or
folder name), since neither has a sibling document to nest alongside.

## Per-slug folder shape

```
prd/{features,enhancements,bugs}/<slug>/
├── intent.md            # precedes the PRD/ER/BR — see "Intent" below
├── prd-<slug>.md         (or enhancement-request-<slug>.md / bug-report-<slug>.md)
└── spec.md                # developer-facing, derived from the PRD, own approval gate

architecture/{features,enhancements,bugs}/<slug>/
├── TD-<slug>.md
├── SA-comments-<slug>.md   # one running file per slug — PRD/Epics-Stories review
│                           # + Technical Design review, both passes
└── tech-spec-<slug>.md     # developer-facing, derived from the TD, own approval gate

readiness/{features,enhancements,bugs}/<slug>/
├── epics-stories.md        # PM drafts, SA adds Round 2 — one running file
├── test-scenarios.md       # PM drafts, SA adds technical scenarios
└── test-cases.xlsx         # PM-Test-Cases + SA-Technical-Test-Cases sheets,
                             # `qa_status` field gates QA execution starting

build/{features,enhancements,bugs}/<slug>/
├── implementation-note-<slug>.md
└── qa-execution-report-<slug>.md
```

**Filename rule**: none of these documents is ever a bare `PRD.md`/`TD.md`/
etc. — the filename is always `<template-basename>-<slug>.md`, the matching
`templates/*-template.md` file's name with `-template` dropped and the slug
appended. This is what lets a persona open a document by constructing its
path directly instead of searching for it — see
`skill-doc-tree-template.md`'s "Locating a document directly" section for the
general method.

Prototype export sits outside this per-slug shape, flat under `prototypes/
<category>-<slug>/` with its `prototype-meta.md` sidecar (per Q2, the full
export, not just a link) — no sibling document to nest alongside, same
reason `releases/` stays flat.

Epics/Stories and test material (`readiness/` above) get a `tracker_id`
written back into `epics-stories.md` by PjM once the corresponding ClickUp
item is created (the one narrow, additive exception to never editing
another persona's document) — referenced by ID link-back from there on, see
`shashi-care-clickup-binding.md`.

## Repos

One per doc-tree folder, `-docs` suffix — **Shashi-Care-Core-docs**,
**SAL-docs**, **SNF-docs**. `-docs` is a one-boundary naming exception (GitLab
needs the docs repo distinct from the code repo sharing the same product
name); every other naming convention stays as-is.

## GitLab access and branching

**Single permanent branch: `main`.** No `develop`, no environment branches.

**No feature branch for PM/SA/PjM/Developer/QA/DevOps's own authoring** — each
persona writes and commits straight onto `main`'s working tree; `product-team`
is the sole actor that runs the commit (see `shashi-care-gitlab-binding.md`'s
"Commit mechanics"). **Team-submitted Technical Designs are the one exception**:
the dev team submits via an MR into `architecture-submissions/<category>-<slug>/`,
short-lived per-MR branches created and deleted per submission. Developers
have write access there (can push branches, open MRs), but merges to `main`
are gated by Sathish or a team lead.

## Front-matter

Every document keeps a `product: Core | SAL | SNF` field even though folder
location already signals scope — a mismatch is itself worth flagging, not
silently resolving. Approval-gated documents (PRD/ER/BR, spec.md, Technical
Design, tech-spec, Release Plan) carry `status`/`Status: Approved` as their
commit trigger — see `shashi-care-gitlab-binding.md`'s "Commit mechanics."
The prototype export carries its own `repo_status` in `prototype-meta.md`
instead. No document carries a `repo_status: not-promoted | promoted` or
`last_promoted_revision` field.

## Prototype retention

**No deletion step.** The `prototypes/<category>-<slug>/` export is retained
permanently, same as every other committed artifact — no persona deletes it,
no stage checks for or triggers deletion. Any cleanup is Sathish's own manual
action, entirely outside this process.

## As-built ownership (current state, expected to change)

The codebase today is SAL's original code, pivoted to SNF, now combined —
one technical-debt-laden codebase, not two clean product codebases:
- `SNF-docs/_as-built/` is the real ground truth right now.
- `SAL-docs/_as-built/` stays empty until SAL development restarts from a
  clean base.
- `Shashi-Care-Core-docs/_as-built/` stays empty until the actual
  Core-components separation happens, expected once SAL work begins.

`compliance/` and `integrations/pcc/` are genuinely platform-level — the PCC
partnership and HIPAA posture apply to the business as a whole, not to SAL or
SNF individually — but sit under SNF-docs for the same reason: that's where
current reality lives, since there's no real Core separation yet. Move all
three (`_as-built/`, `compliance/`, `integrations/`) to Shashi-Care-Core-docs
once that separation happens — don't leave them stranded under SNF-docs for
a reason that no longer applies by then. Update this note when the
separation happens rather than treating today's arrangement as permanent.

## Intent

`intent.md` lives in the repo from the start, same as every other document —
`prd/{features,enhancements,bugs}/<slug>/intent.md`, using
`templates/intent-template.md`, preceding the PRD/ER/BR in the same slug
folder. Superseded (not deleted) once the PRD/ER/BR exists. A change-request
intent (`intent-change-<n>.md`, sequential per slug) files alongside the
already-superseded original, never overwriting it.

## Roadmap

`roadmap/<product>-roadmap.xlsx`, one per repo — theme-based
Now/Next/Later, kept updated in place by PM, no approval-gate field.

## Deployment record

`releases/deployment-record-<release-slug>.md`, one per deployment event
(not per slug, since a deployment can span several) — uses
`templates/deployment-record-template.md`, sits alongside the release plan
workbook. Authored by DevOps.

## Technical debt register and Deferred Open Questions register

- Technical debt register: `_as-built/architecture/technical-debt.md`, plus
  a detailed write-up per significant item — uses
  `templates/technical-debt-register-template.md`.
- Deferred Open Questions register: `deferred-open-questions-register.md` at
  the repo root — a fallback tracker only, for a disposition that's
  genuinely neither Technical Debt nor an Enhancement — see
  `PROCESS-WALKTHROUGH.md`'s "Open Question lifecycle and the
  development-readiness gate." Uses
  `templates/deferred-open-questions-register-template.md`.

## Team structure

`_reference/team-structure.md` — the real, filled roster and RACI for all
personas across both hosting systems (Cowork and Hermes), not duplicated per
repo and not owned/edited by any single persona other than Process
Architect, who authors it on the team's behalf. Instantiated from
`templates/team-structure-template.md`.

## AI-Native SDLC alignment (Anthropic's playbook)

Checked this process against Anthropic's "AI-Native SDLC playbook"
(claude.com/blog/the-ai-native-sdlc-playbook), which describes a six-stage
Plan/Design/Build/Test/Deploy/Maintain loop with version-controlled artifacts
(`intent.md` → `spec.md` → `plan.md` → tests/evals → PR review/deploy gates →
autonomous monitoring that writes a fresh `intent.md`).

**Adopted**: the `intent.md` concept — a fast, human-readable capture of the
raw idea, preceding the PRD/ER/BR. Adapted, not copied verbatim: ours sits
per-slug alongside the PRD it seeds and is explicitly disposable (superseded,
not deleted) once the PRD exists, and lives in the GitLab repo from the very
first draft, same as every other document.

**Deliberately not adopted**: the playbook's `spec.md` merges requirements
and design into one artifact, produced in a single session. We keep PRD
(Product Manager) and Technical Design (System Architect) as two separate,
separately gated documents — this isn't an oversight, it's what the two-pass
SA review, the escalation threshold, and the PM/SA authorship boundary all
depend on. Merging them would remove the structure those were built to
provide.

**The boundary**: this system covers Plan, Design, Build, Test, and Deploy.
PM and SA own Plan/Design — PRD through Epics/Stories reaching `ready` and
the ClickUp handoff. Developer, QA Engineer, and DevOps Engineer — all
hosted in Hermes rather than Cowork, one instance per code repo — own Build
(`skill-developer-discipline.md`), Test execution
(`skill-qa-discipline.md`), and Deploy (`skill-devops-discipline.md`). All
six personas' document output (PRD through deployment records) is authored
directly in the GitLab-direct model described above. Maintain (production
monitoring feeding a fresh `intent.md`) is partially covered — DevOps's
monitoring return-path — but not fully autonomous yet. This still isn't the
playbook's `plan.md`/hooks/CI-eval loop verbatim: PRD and Technical Design
remain two separate, separately gated documents (see above), and a code
repo's own `CLAUDE.md` and Claude Code skills (including any code-side
HIPAA check) remain the dev team's own tooling, not authored here. A
Developer/QA/DevOps instance that surfaces a scope, behavior, or design gap
writes back to PM or SA per its own discipline file's "Deviation/return
path" section, rather than relying solely on the informal dev-team feedback
channel (chat, email, Notion).

## Open worklog items

1. Sprint-boundary status snapshots (a point-in-time capture of ClickUp
   status into sprint-plan/retro docs, replacing the dropped daily-sync
   idea) — not yet designed. Address later.
