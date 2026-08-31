# Doc Tree — Shashi Care

Instantiated from `skill-doc-tree-template.md` (canonical side — Google Drive's
`shashi-care-docs/`, unchanged internal shape, three parallel copies;
`product-engineering/`, Product Manager/System Architect/Project Manager's
Hermes-side working root, is a git repo kept manually synced to this exact
same shape — see `PROCESS-WALKTHROUGH.md`'s "The agents" section) plus
`skill-gitlab-promotion-template.md` (GitLab side — as of the 2026-08-31
restructure, nested by category and slug to mirror the canonical shape
instead of flattened by document type; see `shashi-care-gitlab-binding.md`'s
"Target structure"). Shown together here because the two are related by
promotion — the only real shape divergence left is the repo-name `-docs`
suffix (see "GitLab — exact resulting structure" below).

## Canonical structure (Google Drive / `product-engineering`) — full working structure

`{DRIVE_ROOT}/shashi-care-docs/`, Google Drive–synced (Mirror mode) — read
directly by Process Architect (Cowork) and Developer/QA/DevOps (Hermes).
Product Manager, System Architect, and Project Manager (Hermes) instead work
out of `product-engineering/`, a git repo kept manually synced to this exact
same shape — see `PROCESS-WALKTHROUGH.md`'s "The agents" section.

```
shashi-care-docs/
├── 00_roadmap/
│   └── product-roadmap.xlsx        # ONE file, tabs: Shashi-Care-Core, SAL, SNF
├── templates/                       # shared reference, all personas, all products —
│   └── ...                          # every *-template.md / *-template.xlsx referenced
│                                     # throughout the skill/config files lives here.
│                                     # FILL-IN-THE-BLANK FORMATS ONLY — process/
│                                     # policy documents go in _reference/ instead,
│                                     # not here (see below) — keeps the two kinds
│                                     # of file from blurring together.
├── _reference/                      # process & structure docs, not templates —
│   ├── shashi-care-doc-tree.md      # this file
│   ├── shashi-care-clickup-binding.md
│   ├── shashi-care-gitlab-binding.md
│   ├── shashi-care-design-standards.md
│   ├── PROCESS-WALKTHROUGH.md
│   └── team-structure.md
├── shashi-care-core/
│   ├── 01_releases/                 # not yet in use — Core doesn't ship independent
│   │                                # releases today; add when/if it does
│   ├── 02_prd/
│   │   ├── _as-built/               # empty for now — see as-built note below
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   ├── 03_architecture/
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   ├── 04_handovers/
│   ├── 05_clickup-sync/
│   │   └── mapping-log.md
│   ├── 06_gitlab-sync/
│   │   └── promotion-log.md
│   ├── 07_build/
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   └── deferred-open-questions-register.md
├── SAL/
│   ├── 01_releases/
│   │   └── SAL-release-plan.xlsx    # tab per release, stacked sections
│   ├── 02_prd/
│   │   ├── _as-built/               # NOT populated yet — SAL is being rebuilt from
│   │   │                            # scratch; no current SAL-only ground truth
│   │   │                            # distinct from SNF's combined codebase
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   ├── 03_architecture/
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   ├── 04_handovers/
│   ├── 05_clickup-sync/
│   │   └── mapping-log.md
│   ├── 06_gitlab-sync/
│   │   └── promotion-log.md
│   ├── 07_build/
│   │   ├── features/
│   │   ├── enhancements/
│   │   └── bugs/
│   └── deferred-open-questions-register.md
└── SNF/
    ├── 01_releases/
    │   └── SNF-release-plan.xlsx
    ├── 02_prd/
    │   ├── _as-built/               # THE ground truth — the current combined
    │   │                            # codebase, technical debt and all, lives here
    │   │                            # under SNF until Core is properly separated out
    │   ├── features/
    │   ├── enhancements/
    │   └── bugs/
    ├── 03_architecture/
    │   ├── _as-built/               # existing architecture design docs for the
    │   │                            # current combined codebase — same temporary
    │   │                            # SNF placement as 02_prd/_as-built/ above
    │   ├── features/
    │   ├── enhancements/
    │   ├── bugs/
    │   ├── integrations/
    │   │   └── pcc/
    │   │       ├── api-contracts/   # PCC API Postman collections
    │   │       └── agreements/      # PCC partnership agreement — approved API
    │   │                            # list and terms; read-only reference
    │   └── compliance/
    │       └── hipaa-compliance-register.md   # converted from Excel 2026-08-28,
    │                                            # 39 entries, Sathish-only edits
    ├── 04_handovers/
    ├── 05_clickup-sync/
    │   └── mapping-log.md
    ├── 06_gitlab-sync/
    │   └── promotion-log.md
    ├── 07_build/
    │   ├── features/
    │   ├── enhancements/
    │   └── bugs/
    └── deferred-open-questions-register.md
```

## Per-slug folder shape (canonical / `product-engineering`) — not visible in the tree above
The ASCII tree above shows `features/`, `enhancements/`, `bugs/` as folders, but
abstracts away what's inside each `<slug>/` subfolder. As of the prototype-handling
update, that shape is:
```
02_prd/features/<slug>/
├── intent.md                     # precedes the PRD — see AI-Native SDLC note below
├── prd-<slug>.md                 # NOT a bare "PRD.md" — filename mirrors
│                                  # templates/prd-template.md, slug included
├── spec.md                       # developer-facing, derived from PRD — see below
├── epics-stories.md              # gated — see skill-pm-discipline.md
├── test-scenarios.md
├── test-cases.xlsx
└── prototype/                    # only if a full export was provided (Q2: yes,
    ├── prototype-meta.md         # full export, not just a link)
    └── ...                       # the actual exported HTML/assets bundle

02_prd/enhancements/<slug>/
└── enhancement-request-<slug>.md # mirrors templates/enhancement-request-template.md;
                                   # same intent.md / spec.md / epics-stories.md /
                                   # test-scenarios.md / test-cases.xlsx / prototype/
                                   # siblings as features, same rules

02_prd/bugs/<slug>/
└── bug-report-<slug>.md          # mirrors templates/bug-report-template.md
```
`prototype/` is retained permanently in `product-engineering` for now (Claude
Design isn't yet available to the whole team) — revisit once it is. Unlike
the GitLab copy, there's no scheduled deletion for this one — see "Prototype
retention and deletion policy" below for the GitLab-only rule.

**Filename rule, stated once so it doesn't drift again**: none of these top-level
documents is ever a bare `PRD.md`/`ER.md`/`BR.md` — the actual filename is always
`<template-basename>-<slug>.md`, i.e. the matching `templates/*-template.md` file's
name with `-template` dropped and the slug appended. This is what lets a
document be opened by constructing its path directly instead of searching for
it — see `skill-doc-tree-template.md`'s "Locating a document directly" section
for the general method any persona should follow.

## Per-slug folder shape (canonical / `product-engineering`, System Architect side) — mirrors 02_prd
As of the `03_architecture` restructure, this side splits by category and slug
the same way `02_prd` does, instead of the old flat `technical-designs/` +
`review-comments/` pair with category-prefixed filenames. Filenames are
slug-suffixed too (2026-08-29), same rationale as `02_prd/`'s
`prd-<slug>.md`/`enhancement-request-<slug>.md`/`bug-report-<slug>.md` — a bare
`TD.md` looks identical across every slug folder open at once, exactly the
readability problem the `02_prd` rename solved, and it applies here just as much:
```
03_architecture/features/<slug>/
├── TD-<slug>.md          # mirrors templates/technical-design-template.md
├── tech-spec-<slug>.md   # developer-facing, derived from TD once approved —
│                         # informally called the Impl Spec by the team; canonical
│                         # name stays tech-spec-<slug>.md
└── SA-comments-<slug>.md # mirrors templates/sa-review-comments-template.md — one
                          # running file per slug, covering both the PRD/ER/BR
                          # review and the TD review

03_architecture/enhancements/<slug>/
└── (same three files: TD-<slug>.md, tech-spec-<slug>.md, SA-comments-<slug>.md)

03_architecture/bugs/<slug>/
└── (same three files: TD-<slug>.md, tech-spec-<slug>.md, SA-comments-<slug>.md)
```
Same filename *rule* as `02_prd/` above — this is not "the folder already carries
category and slug, so the filename doesn't need to" (that reasoning would argue
for staying bare, and was this document's own error until 2026-08-29); it's the
opposite: the slug is deliberately repeated in the filename for readability across
open folders, exactly like `02_prd/`'s top-level document. `_as-built/`,
`integrations/`, `technical-debt-register.md` (and its per-item write-ups), and
`compliance/` sit outside this per-slug shape entirely — they're per-product or
platform-level, not
tied to any one slug, and are untouched by this restructure.

## Per-slug folder shape (canonical / `product-engineering`, Build stage)
As of the `07_build/` addition (2026-08-29), Developer and QA Engineer instances
(hosted in Hermes) write two per-slug artifacts here, mirroring the shape used by
`02_prd/` and `03_architecture/` above:
```
07_build/features/<slug>/
├── implementation-note-<slug>.md   # mirrors templates/implementation-note-template.md
└── qa-execution-report-<slug>.md   # mirrors templates/qa-execution-report-template.md

07_build/enhancements/<slug>/
└── (same two files)

07_build/bugs/<slug>/
└── (same two files)
```
Deployment records do NOT live here — a deployment can span multiple slugs, so
`deployment-record-<release-slug>.md` lives in `01_releases/` instead, alongside
the release plan workbook. See `templates/deployment-record-template.md`.

## GitLab — exact resulting structure (three repos)

Three separate repos, same names as the canonical product folders
(`shashi-care-core`, `SAL`, `SNF`) they draw from. As of the 2026-08-31
restructure (see `shashi-care-gitlab-binding.md`'s "Target structure"), each
repo is nested **by category and slug**, mirroring `product-engineering`'s
shape, instead of flattened by document type — a per-slug folder holding only
a PRD while its sibling `spec.md` was conspicuously absent looked broken the
old, flattened way. `releases/` and `prototypes/` stay flat (category-and-slug
in the file or folder name), since neither has a sibling document to nest
alongside. No folder below ever contains anything beyond what's listed; there
is no epics-stories, test-scenarios, or review-comments folder in GitLab at
all — those never promote.

```
Shashi-Care-Core-docs (repo)
├── releases/          # empty today — Core has no release plan yet
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
├── architecture-submissions/     # team-submitted TDs awaiting review — any format
│   └── <category>-<slug>/
└── prototypes/                   # full prototype exports, permanent until dev
    └── <category>-<slug>/        # complete + PjM deletes (see retention/deletion policy)

SAL-docs (repo)
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
├── architecture-submissions/
│   └── <category>-<slug>/
└── prototypes/
    └── <category>-<slug>/

SNF-docs (repo)
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
├── architecture-submissions/
│   └── <category>-<slug>/
└── prototypes/
    └── <category>-<slug>/
```

`category` here is `feature` / `enhancement` / `bug` — not to be confused with
the document types (PRD / Technical Design / Release Plan) that gave these
folders their names. `prd/` and `architecture/` now split by category and slug
the same way `product-engineering`'s `02_prd/` and `03_architecture/` do, so
`category` lives in the *path* on both sides — no more filename prefix needed
for the documents that nest (`prd-<slug>.md`, `TD-<slug>.md`, `spec.md`,
`tech-spec-<slug>.md`, all named the same way `product-engineering` already
names them). `releases/` and `prototypes/` stay flat, category-and-slug in the
file or folder name, since neither has a sibling document to nest alongside.

`spec.md` sits with the PRD and `tech-spec-<slug>.md` sits with the TD —
GitLab is the developer-facing side, and developers feeding Claude Code plan
mode want the pair together, not split into a separate folder by who wrote
each half.

`-docs` suffix: GitLab is the one place a code repo and a docs repo could share a
namespace, so the docs repo needs a name distinct from the existing code repo —
see `shashi-care-gitlab-binding.md`. Every other naming convention
(`product-engineering` folders, ClickUp Folders, filenames) stays exactly
as-is; this is a one-boundary exception, not a scheme-wide rename.

## GitLab access and branching
**Single permanent branch: `main`.** No `develop`, no environment branches. Per-MR
branches are created and deleted per submission — short-lived, never treated as a
second permanent branch. Developers have **write access** (can push branches, open
MRs) but merges to `main` are gated by Sathish or a team lead's approval — write
access is broad, what counts as "landed" stays narrow and reviewed.

## System Architect's Context (four folders, not one)
Unlike Product Manager and Project Manager, which each read one folder, System
Architect's Hermes session context includes `product-engineering/` — its
working doc root, manually synced with `shashi-care-docs/` —
**plus local checkouts of all three GitLab docs repos** (Shashi-Care-Core-docs,
SAL-docs, SNF-docs) — needed to review team-submitted Technical Designs landing in
`architecture-submissions/` across all three products. System Architect reads from
the checkouts but **never writes into them** — all output goes to that slug's
`SA-comments-<slug>.md` in `product-engineering/`. See `skill-sa-discipline.md`.

## Prototype retention and deletion policy
Deletion is never automatic, and the two copies are **not symmetric**:
- **`product-engineering`**: `prototype/` is retained permanently for now —
  Claude Design isn't yet available to the whole team; revisit deleting it
  once that changes (see "Per-slug folder shape" above). No scheduled
  deletion event today.
- **GitLab**: delete `prototypes/<category>-<slug>/` once development is
  actually complete (check tracker status, don't assume) — confirmed first
  by Project Manager persona, who always leaves a one-line note in
  `promotion-log.md` rather than deleting silently.

## `product-engineering` → GitLab mapping (what promotes, where it lands)

| `product-engineering` source | Trigger | GitLab target |
|---|---|---|
| `<folder>/02_prd/features/<slug>/prd-<slug>.md`, `enhancements/<slug>/enhancement-request-<slug>.md`, or `bugs/<slug>/bug-report-<slug>.md` | `status: approved` | `<repo>/prd/{features,enhancements,bugs}/<slug>/prd-<slug>.md` |
| `<folder>/03_architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md` | Its own `status: approved` | `<repo>/architecture/{features,enhancements,bugs}/<slug>/TD-<slug>.md` |
| `<folder>/01_releases/<Product>-release-plan.xlsx` (one release/tab) | `status: approved` | `<repo>/releases/<release-slug>.xlsx` |
| `<folder>/02_prd/.../<slug>/prototype/` (full export) | `prototype-meta.md`'s own `repo_status`, independent of the PRD's | `<repo>/prototypes/<category>-<slug>/` |
| `<folder>/02_prd/.../<slug>/spec.md` | Its own `status: approved` (only after the PRD is at least approved) | `<repo>/prd/{features,enhancements,bugs}/<slug>/spec.md` |
| `<folder>/03_architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md` | Its own `status: approved` (only after the TD is at least approved) | `<repo>/architecture/{features,enhancements,bugs}/<slug>/tech-spec-<slug>.md` |

Everything else in the `product-engineering` tree — epics-stories, test-scenarios,
SA-comments, handovers, the roadmap, both sync logs — stays
`product-engineering`-only. Epics/Stories are referenced live in ClickUp instead
(see the ID link-back rule in `skill-pjm-discipline.md`).

## Why the slug problem goes away (canonical / `product-engineering` side)
A feature/enhancement/bug slug only needs to be unique *within its product folder*,
not globally — `SAL/02_prd/features/resident-transport/` and a hypothetical
`SNF/02_prd/features/resident-transport/` can coexist without collision, and neither
needs the product name baked into the slug itself. Shared features go under
`shashi-care-core/02_prd/...` instead of picking one product folder arbitrarily.

## As-built note (current state, expected to change)
The codebase today is SAL's original code, pivoted to SNF, now combined — one
technical-debt-laden codebase, not two clean product codebases.
- `SNF/02_prd/_as-built/` is the real ground truth right now.
- `SAL/02_prd/_as-built/` stays empty until SAL development restarts from a clean
  base.
- `shashi-care-core/02_prd/_as-built/` stays empty until the actual Core-components
  separation happens, expected once SAL work begins.
Update this note when the separation happens rather than treating today's
arrangement as permanent.

## Architecture as-built, integrations, and compliance — same temporary placement
`03_architecture/_as-built/`, `integrations/pcc/`, and `compliance/` are genuinely
platform-level — the PCC partnership and HIPAA posture apply to the business as a
whole, not to SAL or SNF individually. They're placed under `SNF/` for now for the
same reason `02_prd/_as-built/` is: that's where current reality actually lives,
since there's no real Core separation yet. Move all three to `shashi-care-core/`
when that separation happens — don't leave them stranded under SNF once Core is a
real thing, since by then they'd be in the "wrong" folder for a reason that no
longer applies.

## Front-matter
Every document keeps a `product: Core | SAL | SNF` field even though folder
location already signals scope. A mismatch between the two is itself worth
flagging, not silently resolving. Promotable documents (PRD, Technical Design,
Release Plan) additionally carry `repo_status: not-promoted | promoted` and
`last_promoted_revision:` — see `shashi-care-gitlab-binding.md`.

## Intake note (unchanged)
- **Features**: `source: prototype-first`. `claude_design_link` is a single,
  project-level link. Each requirement/story additionally carries `prototype_page:`.
- **Enhancements/bugs**: `source: direct`.

## Open worklog items
1. Sprint-boundary status snapshots (a point-in-time capture of ClickUp status into
   sprint-plan/retro docs, replacing the dropped daily-sync idea) — not yet
   designed. Address later.

## Newer additions (not yet reflected in the ASCII tree above)
- **Technical debt register**: `<folder>/03_architecture/technical-debt-register.md`
  per product folder, plus `<folder>/03_architecture/technical-debt/<TD-ID>.md` for
  detailed write-ups of significant items. Uses
  `templates/technical-debt-register-template.md`.
- **Deferred Open Questions register**: `<folder>/deferred-open-questions-register.md`,
  one running file per product folder, sitting at the product-folder root
  (not nested under `03_architecture/` — it isn't architecture-specific).
  Fallback tracker only, for a Deferred Open Question disposition that's
  genuinely neither Technical Debt nor an Enhancement — see
  `PROCESS-WALKTHROUGH.md`'s "Open Question lifecycle and the
  development-readiness gate." Uses
  `templates/deferred-open-questions-register-template.md`.
- **Team structure**: `_reference/team-structure.md` — the real, filled roster and
  RACI for all 7 personas across both hosting systems (Cowork and Hermes), not
  duplicated per folder and not owned/edited by any single persona other than the
  Process Architect, who authors it on the team's behalf. Instantiated from
  `templates/team-structure-template.md`.
- **Sprint docs** (`sprint-plan.md`, `retro.md`, already in the tree above) now use
  `templates/sprint-plan-status-template.md` and `templates/sprint-retro-template.md`
  respectively.
- **Epics/Stories** (`epics-stories.md`, already in the tree above) now uses
  `templates/epics-stories-template.md`.
- **Test cases**: `<slug>/test-cases.xlsx`, sitting alongside `test-scenarios.md`
  in the same slug folder. Uses `templates/test-cases-template.xlsx`. This is the
  one document type that also crosses into ClickUp as a file attachment (on the
  `test_scenario`-tagged task), not just as an ID link-back — see
  `shashi-care-clickup-binding.md`.
- **Prototype metadata**: `<slug>/prototype/prototype-meta.md`, uses
  `templates/prototype-meta-template.md` — see "Per-slug folder shape" above.
- **SA-comments**: `03_architecture/{features,enhancements,bugs}/<slug>/SA-comments-<slug>.md`,
  using `templates/sa-review-comments-template.md` — the two-pass PRD/Epics-Stories
  review structure with `review_round` tracking and the 3-round escalation rule,
  plus the Technical Design Review section, all in one running file per slug. See
  `skill-sa-discipline.md`.
- **Build-stage artifacts**: `implementation-note-<slug>.md` and
  `qa-execution-report-<slug>.md`, both in
  `07_build/{features,enhancements,bugs}/<slug>/` — see "Per-slug folder shape
  (canonical / `product-engineering`, Build stage)" above. Uses `templates/implementation-note-template.md`
  and `templates/qa-execution-report-template.md` respectively.
- **Deployment record**: `<folder>/01_releases/deployment-record-<release-slug>.md`
  — one per deployment event, not per slug, since a deployment can span several.
  Uses `templates/deployment-record-template.md`.
- **Intent**: `<slug>/intent.md`, uses `templates/intent-template.md` — see the
  AI-Native SDLC alignment note below.

## AI-Native SDLC alignment (Anthropic's playbook)
Checked this process against Anthropic's "AI-Native SDLC playbook"
(claude.com/blog/the-ai-native-sdlc-playbook), which describes a six-stage
Plan/Design/Build/Test/Deploy/Maintain loop with version-controlled artifacts
(`intent.md` → `spec.md` → `plan.md` → tests/evals → PR review/deploy gates →
autonomous monitoring that writes a fresh `intent.md`).

**Adopted**: the `intent.md` concept — a fast, human-readable capture of the raw
idea, preceding the PRD/ER/BR. Adapted, not copied verbatim: ours sits per-slug
alongside the PRD it seeds, doesn't promote to GitLab, and is explicitly
disposable once superseded.

**Deliberately not adopted**: the playbook's `spec.md` merges requirements and
design into one artifact, produced in a single session. We keep PRD (Product
Manager) and Technical Design (System Architect) as two separate, separately
gated documents — this isn't an oversight, it's what the two-pass SA review, the
escalation threshold, and the PM/SA authorship boundary all depend on. Merging
them would remove the structure those were built to provide.

**The boundary, settled explicitly (updated 2026-08-29)**: this system originally
covered only Plan and Design — everything through Epics/Stories reaching `ready`
and the ClickUp handoff, with Build/Test/Deploy/Maintain left entirely to the dev
team. That boundary has since moved: three new personas — Developer, QA Engineer,
DevOps Engineer, all hosted in Hermes rather than Cowork — now extend this system
into Build (`skill-developer-discipline.md`), Test execution
(`skill-qa-discipline.md`), and Deploy (`skill-devops-discipline.md`), one instance
per code repo. Maintain (production monitoring feeding a fresh `intent.md`) is
partially covered — DevOps's monitoring return-path — but not fully autonomous yet.
This still isn't the playbook's `plan.md`/hooks/CI-eval loop verbatim: PRD and
Technical Design remain two separate, separately gated documents (see above), and
a code repo's own `CLAUDE.md` and Claude Code skills (including any code-side
HIPAA check) remain the dev team's own tooling, not authored here. A real return
path now exists where none did before: a Developer/QA/DevOps instance that
surfaces a scope, behavior, or design gap writes back to PM or SA per its own
discipline file's "Deviation/return path" section, rather than relying solely on
the informal dev-team feedback channel (chat, email, Notion) this note originally
described.
