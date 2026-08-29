# Skill: Shared Doc Tree Template

Generic folder shape for a three-persona (PM / SA / PjM) Cowork setup. Copy and
instantiate per project — fill in product/team names, don't edit this file directly.

```
<project>-docs/
├── 00_roadmap/
│   ├── product-roadmap.md
│   └── release-calendar.md
├── 01_releases/
│   └── <release-slug>/
│       ├── release-plan.md
│       └── sprints/
│           └── <sprint-slug>/
│               ├── sprint-plan.md
│               └── retro.md
├── 02_prd/
│   ├── _as-built/                 # ground truth, code-derived, quarantined
│   │   └── <product-or-team>/
│   ├── features/
│   │   └── <slug>/
│   │       ├── PRD-<slug>.md       # front-matter carries provenance:
│   │       │                       #   source: direct | prototype-first
│   │       │                       #   design_link: <if prototype-first — one
│   │       │                       #     project-level link, not per page>
│   │       │                       #   handoff_link: <e.g. Figma, added later>
│   │       │                       #   signed_off_by / signed_off_date
│   │       │                       # each requirement/story additionally tags
│   │       │                       # which prototype page it corresponds to,
│   │       │                       # when the prototype has multiple pages
│   │       ├── epics-stories.md
│   │       └── test-scenarios.md
│   ├── enhancements/
│   │   └── <slug>/                 # same shape, source: direct by default
│   └── bugs/
│       └── <slug>/
│           ├── BR-<slug>.md
│           └── notes-to-sa.md
├── 03_architecture/
│   ├── _as-built/                   # architecture-level ground truth, same role
│   │                                # as 02_prd/_as-built/ — code-derived,
│   │                                # quarantined, extended not overwritten
│   ├── features/                    # Option B (nested — see rationale below):
│   │   └── <slug>/                  # mirrors 02_prd/'s shape. Filenames shown
│   │       ├── TD-<slug>.md         # slug-suffixed (can stay bare instead —
│   │       ├── tech-spec-<slug>.md  # TD.md/tech-spec.md/SA-comments.md — see
│   │       └── SA-comments-<slug>.md # rationale below for the tradeoff)
│   ├── enhancements/
│   │   └── <slug>/                  # same three files
│   ├── bugs/
│   │   └── <slug>/                  # same three files
│   │
│   │   # Option A (flat — the other valid shape, not shown nested above):
│   │   # technical-designs/<category>-<slug>_TD.md,
│   │   # technical-designs/<category>-<slug>_tech-spec.md,
│   │   # review-comments/<category>-<slug>_SA-comments.md — category goes in the
│   │   # filename instead of a subfolder. Pick this when a slug typically has
│   │   # just these three files and nesting would be one folder for very little.
│   ├── integrations/
│   │   └── <partner>/               # one subfolder per external partner/system
│   │       ├── api-contracts/       # e.g. Postman collections — the contract as
│   │       │                        # actually implemented, ground truth like
│   │       │                        # _as-built
│   │       └── agreements/          # partnership terms, approved API list — read
│   │                                # only reference; check before designing
│   │                                # against an API that isn't actually approved
│   └── compliance/
│       └── <framework>-compliance-register.md   # same register pattern as
│                                                  # technical debt
├── 04_handovers/
│   └── <date>_<from>-to-<to>_<topic>.md
└── 05_tracker-sync/
    └── mapping-log.md
```

## Design rationale (why the shape works)
- `_as-built` stays separate from proposed/in-flight work so there's always an
  unambiguous "what does the system do today."
- Feature / enhancement / bug are peers, not nested — a bug doesn't need to pretend
  to be a feature to get a home.
- Provenance lives in front-matter, not folder location — a feature that started as
  a design prototype and one that started as a direct chat end up in the same place,
  distinguished by metadata, not by where they're filed.
- A multi-page prototype gets **one project-level link**, with page-level
  granularity carried per requirement/story instead — this avoids link sprawl while
  still letting a downstream reviewer or design-handoff step find the exact page a
  requirement came from.
- Handovers are pointers, not copies — they name what's ready and where, not restate
  its content.
- The mapping log is what makes the tracker-creation persona idempotent.
- **Category lives in the path where the folder splits by category, and in the
  filename where it doesn't.** `02_prd/` always splits by category
  (`features/`, `enhancements/`, `bugs/`), so slugs stay bare there.
  `03_architecture/` is a choice between two equally valid shapes — pick per
  project, don't mix both within one project:
  - **Option A — flat, category-prefixed filenames.** `technical-designs/` and
    `review-comments/` stay flat, with `<category>-<slug>_TD.md` etc. This fits
    when a slug typically has just a TD and a comments file — not enough siblings
    to justify a folder of its own, and a flat folder full of category-prefixed
    filenames stays scannable as volume grows.
  - **Option B — nested by category and slug, mirroring `02_prd/`.** Each slug
    gets its own `03_architecture/{features,enhancements,bugs}/<slug>/` folder
    holding a TD, a tech-spec, and an SA-comments file. Filenames can stay bare
    (`TD.md`, `tech-spec.md`, `SA-comments.md` — the folder already carries the
    category and slug) or be slug-suffixed too (`TD-<slug>.md`,
    `tech-spec-<slug>.md`, `SA-comments-<slug>.md`), same rationale as
    slug-suffixing a top-level PRD/ER/BR even though `02_prd/` also splits by
    category: a bare filename still looks identical across every slug folder
    open at once, which matters more the more folders tend to be open
    simultaneously. Fits either way when a project wants both sides of the doc
    tree to look the same at a glance, or expects a slug's architecture side to
    accumulate more than just these three files over time.

## Locating a document directly (no searching)
This template's whole point is that the shape is knowable in advance — so once a
project's doc tree exists, an agent that already knows the product/team, the
document type, and the slug should never need to search for a file. Recursive
listing, glob, or grep-for-a-filename to "find" a document you can already name
burns tool calls and tokens on something the tree shape already answers.

**The method:**
1. Identify the three coordinates you already have: which product/team folder,
   which document type (PRD, ER, BR, TD, spec, epics-stories, etc.), and the slug.
2. Compose the path directly from the shape above — or, in an instantiated
   project, from that project's own doc-tree reference, which is the authority on
   the *exact* filename (see below).
3. Read that exact path. Don't list the containing folder first "just to check" —
   that's the search behavior this method replaces.
4. If the read fails (file doesn't exist yet, wrong guess, or the slug turns out
   not to exist), do exactly **one** bounded listing — of that slug's own folder
   only, never the wider tree — to see what's actually there. Either the file
   exists under a name close to the guess (open it), or the slug genuinely
   doesn't have that document yet (say so; don't keep guessing paths or escalate
   to a broader search).
5. Reserve open-ended search for genuinely open-ended requests — "what
   enhancements exist for X," where the slug itself isn't known yet. That's a
   different task from opening a document you can already name.

**Filenames are project-specific, not generic.** This file only teaches the
folder shape and the read-then-bounded-listing protocol above — it deliberately
doesn't fix exact filenames, because those vary per project. The exact filename
prefix for each document type (e.g. does a PRD live at `PRD.md` or
`prd-<slug>.md`?) belongs in the project's own instantiated `_reference/*-doc-
tree.md`, typically matching the corresponding `templates/*-template.md` file's
name with `-template` dropped and the slug appended. Keep that project file
current as filenames evolve — a direct-path mechanism is only as good as the
reference it's built from.
