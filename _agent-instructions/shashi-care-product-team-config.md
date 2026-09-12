# Config: Product Team — Shashi Care (Core + SAL + SNF)

Pairs with `SOUL.md` (the `product-team` Hermes orchestrator profile), the
same way `shashi-care-pm-config.md` pairs with `skill-pm-discipline.md` and
`shashi-care-sa-config.md` pairs with `skill-sa-discipline.md`. SOUL.md
defines what an orchestrator does in general; this file supplies the
Shashi-Care-specific facts and bindings SOUL.md deliberately leaves generic —
project name, repository bindings, approver identity, artifact locations,
and naming conventions. **When uncertain about a repository binding,
artifact location, or process detail not spelled out here, check
`_reference/` in the doc root** (`shashi-care-doc-tree.md`,
`shashi-care-gitlab-binding.md`, `shashi-care-clickup-binding.md`,
`PROCESS-WALKTHROUGH.md`) before guessing or defaulting to the simplest
interpretation.

## Purpose and pairing

This is `product-team`'s own project config, filling the same role for the
orchestrator that `shashi-care-pm-config.md` and `shashi-care-sa-config.md`
fill for the Product Manager and System Architect personas. It exists so
that SOUL.md — the generic orchestration behavior — never has to hard-code a
product name, a filesystem path, an approver's identity, or a document
naming convention. Where SOUL.md says "determine the appropriate artifact
repository and location from the active project's configuration and
repository bindings," this file, together with the `_reference/` bindings it
points to, is that configuration for Shashi Care. This file does not
restate SOUL.md's orchestration behavior, PROCESS-WALKTHROUGH.md's stage
detail, or either specialist's discipline file — it binds `product-team` to
the concrete facts those generic documents need to operate on this project.

## Project identity

The active project is **Shashi Care**. `product-team`, when operating under
this configuration, is the orchestration agent for the Shashi Care
product-engineering team specifically — not a generic, project-less
instance.

## Products / documentation repositories

Shashi Care has three parallel products/scopes, each a full instance of the
same document tree (see `_reference/shashi-care-doc-tree.md`):

- **Shashi Care Core** (`shashi-care-core/`) — features/enhancements/bugs
  shared across SAL and SNF.
- **SAL**
- **SNF**

Each product has its own GitLab documentation repository:
**Shashi-Care-Core-docs**, **SAL-docs**, **SNF-docs**. A shared,
cross-product feature is filed under Shashi-Care-Core-docs rather than
arbitrarily picking SAL or SNF — see `shashi-care-doc-tree.md`'s "Repos" and
"Cross-folder features" (`shashi-care-clickup-binding.md`) for the same
rule applied on the ClickUp side.

`product-team` does not maintain its own copy of the products/folders list —
it is stated here only so the orchestrator has an authoritative,
project-scoped anchor; `shashi-care-doc-tree.md` remains the source of
truth for the tree shape itself, and `shashi-care-pm-config.md` /
`shashi-care-sa-config.md` remain the source of truth for how each
specialist works within it.

## Authoritative project references

For this project, "the active project's configuration and repository
bindings" (SOUL.md's phrase) resolves to the following files, all under
this repository's `_reference/` and `_agent-instructions/`:

- `_reference/shashi-care-doc-tree.md` — the GitLab folder structure: full
  tree, per-slug shape, front-matter conventions, access/branching rules.
  Authoritative for every concrete document path; `product-team` never
  invents or guesses a path outside what this file (or
  `shashi-care-gitlab-binding.md`'s matching "Target structure") defines.
- `_reference/shashi-care-gitlab-binding.md` — GitLab-direct authoring
  rules, the three docs repos, access model, and commit mechanics.
  Authoritative for how and when `product-team` commits.
- `_reference/shashi-care-clickup-binding.md` — ClickUp hierarchy, tags,
  statuses. Relevant once Project Manager is activated; not required for
  the current PM/SA-only scope, but kept as the binding `product-team` will
  need when that stage is reached.
- `_reference/PROCESS-WALKTHROUGH.md` — the authoritative stage-by-stage
  process, gates, Open Question lifecycle, and commit discipline. This
  config does not restate its content.
- `_agent-instructions/shashi-care-pm-config.md` and
  `_agent-instructions/shashi-care-sa-config.md` — the specialist-side
  project bindings; `product-team` reads these to understand what each
  specialist is configured to author and where, without reproducing that
  detail here.

## Artifact repository and authoring model

Product and technical artifacts for Shashi Care are authored directly in
the working tree of the appropriate product's GitLab `-docs` repository
(Shashi-Care-Core-docs, SAL-docs, or SNF-docs), on `main`, in a local
checkout — never in a separate draft location and never promoted through an
intermediate stage. See `_reference/shashi-care-gitlab-binding.md` for the
full authoring and commit model.

This repository, `product-engineering`, is the framework/governance/
configuration repository for this system: `_agent-instructions/`,
`templates/`, `_reference/`. It holds no product or technical document
content for Shashi Care, and is not a staging area or working mirror for
that content. `product-team` must never treat a path under
`product-engineering` as the destination or intermediate home for a PRD,
spec, Technical Design, tech-spec, or any other product artifact — the
destination is always the relevant GitLab `-docs` repo checkout, per
`shashi-care-doc-tree.md` and `shashi-care-gitlab-binding.md`.

Concrete repository selection, artifact paths, and document naming
(e.g. which slug folder, which of `prd-<slug>.md` /
`enhancement-request-<slug>.md` / `bug-report-<slug>.md`, `spec.md`,
`TD-<slug>.md`, `tech-spec-<slug>.md`) are determined by reading
`shashi-care-doc-tree.md` and `shashi-care-gitlab-binding.md` directly for
the specific document type in question — never guessed, never hard-coded
into `product-team`'s own behavior, and not reproduced in this file.

## Human approval authority

**Sathish** is the designated human approval authority for Shashi Care.
Every reference in SOUL.md's generic orchestration behavior to "the human"
or "the project's designated approver" resolves, for this project, to
Sathish.

Approval must never be inferred from a specialist's report, a completed
Kanban task, chat discussion, or the mere existence of an artifact.
Approval is established only by the mechanism `shashi-care-gitlab-binding.md`
and `PROCESS-WALKTHROUGH.md` already define: the relevant document's own
`status`/`Status` field reading `Approved`/`approved` (for gated documents),
or Sathish's explicit confirmation (for non-gated reference material with no
status field). `product-team` verifies that signal directly before treating
a stage as cleared or before committing anything on Sathish's behalf.

## Active specialists and current scope

For Shashi Care, the currently active specialist personas are:

- **Product Manager (`pm`)** — configured by `shashi-care-pm-config.md`.
- **System Architect (`sa`)** — configured by `shashi-care-sa-config.md`.

These are the only specialists `product-team` orchestrates today for this
project. Project Manager, Developer, QA Engineer, and DevOps Engineer
configurations exist in this repository's file index (see
`PROCESS-WALKTHROUGH.md`'s "File index" —
`shashi-care-pjm-config.md`, `shashi-care-developer-config.md`,
`shashi-care-qa-config.md`, `shashi-care-devops-config.md`) but are out of
scope for `product-team` until explicitly activated. Their presence in the
repository is not itself authorization to invoke, assign Kanban work to, or
orchestrate around those personas.

## Specialist profile bindings

`product-team` creates Kanban tasks against the `pm` and `sa` specialist
profiles. Each profile's own behavior is defined by its discipline file
(`skill-pm-discipline.md`, `skill-sa-discipline.md`) plus its project config
(`shashi-care-pm-config.md`, `shashi-care-sa-config.md`) — `product-team`
does not reproduce or override that behavior; it only supplies the task
objective, the authoritative source artifact(s), the expected output
artifact, and the Kanban dependency structure, per SOUL.md's generic
Execution rules.

Which document each specialist authors and where is defined by
`shashi-care-pm-config.md` / `shashi-care-sa-config.md`'s own "Storage
paths" sections and by `shashi-care-doc-tree.md` — `product-team` reads
those when constructing a task's expected-artifact path rather than
maintaining a second copy of that mapping here.

## Kanban and workflow bindings

Kanban tasks for Shashi Care PM/SA work reference source and expected
artifacts by their concrete GitLab `-docs` checkout path (per
`shashi-care-doc-tree.md`), never by a path under `product-engineering`.
Kanban parent/dependency relationships enforce the stage ordering defined
in `PROCESS-WALKTHROUGH.md` (e.g. a Technical Design task depends on the
PRD/ER reaching the Stage 2 approval gate; a tech-spec task depends on the
TD reaching its own approval gate) — `product-team` does not invent an
ordering not already implied by that process document.

The orchestration shape described generically in SOUL.md (approved
requirement → PM-authored spec → SA-authored Technical Design → human
approval → SA-authored tech-spec/Impl-Spec → human approval →
development-ready) maps, for Shashi Care specifically, onto the literal
artifacts `shashi-care-doc-tree.md` defines: the PRD/ER/BR, `spec.md`,
`TD-<slug>.md`, and `tech-spec-<slug>.md`, each in the per-slug folder that
file describes. `product-team` resolves the generic shape to these literal
names by reading `shashi-care-doc-tree.md`, not by this config restating
the tree.

## Verification and commit orchestration

`product-team` is the sole actor that runs `git add`/`commit`/`push` for
Shashi Care's PM- and SA-authored documents, exactly as
`shashi-care-gitlab-binding.md`'s "Commit mechanics" and
`PROCESS-WALKTHROUGH.md`'s "Document commit" section define. For this
project, verifying a workflow gate or a commit concretely means:

- Reading the document directly from its GitLab `-docs` checkout path (per
  `shashi-care-doc-tree.md`), not from any location under
  `product-engineering`.
- Checking that document's own `status`/`Status` field, or, for non-gated
  reference material, confirming Sathish's explicit go-ahead — never
  inferring either from a specialist's self-report.
- Confirming a commit actually landed via `git log` (or an equivalent
  check) against the relevant `-docs` repo's `main` branch, per
  `shashi-care-gitlab-binding.md`'s "Commit mechanics" — never treating the
  commit step as complete merely because it ran without error.

The mechanics of the commit itself (branching model, which document types
commit on their own approval vs. re-commit on revision, the team-submitted
Technical Design pathway) are fully defined in
`shashi-care-gitlab-binding.md` and are not restated here; `product-team`
follows that binding directly.

## Open Question disposition destinations

SOUL.md defines the generic disposition categories (Resolved in current
scope, Accepted as-is, Enhancement, Technical Debt, Deferred, or another
Sathish-approved disposition) and requires that an Enhancement or Technical
Debt disposition reference an actual tracked item, not just close the
question. For Shashi Care, those tracked items live at:

- **Enhancement** → a new Enhancement Request drafted by Product Manager,
  at `prd/enhancements/<slug>/enhancement-request-<slug>.md` in the
  relevant product's GitLab `-docs` repo (see `shashi-care-doc-tree.md`).
- **Technical Debt** → an entry in that product's
  `_as-built/architecture/technical-debt.md`, with a detailed per-item
  write-up where warranted (see `shashi-care-doc-tree.md`'s "Technical
  debt register and Deferred Open Questions register").
- **Deferred**, when it doesn't fit either of the above (fallback only) →
  an entry in `deferred-open-questions-register.md` at that product's
  repo root.

`product-team` confirms the referenced item actually exists at one of these
locations before treating an Open Question row as satisfying the
development-readiness gate — a bare disposition label with no recorded
reference does not satisfy it, per `PROCESS-WALKTHROUGH.md`'s "Open
Question lifecycle" section.

## Access

`product-team`'s own read/write/commit access to the three GitLab `-docs`
checkouts (Shashi-Care-Core-docs, SAL-docs, SNF-docs) has been confirmed.
Verification covered:

- each checkout is a valid local Git repository on branch `main`, tracking
  `origin/main`;
- SSH authentication to the GitLab remote succeeded (`git@gitlab.com`, as
  `@sathish55`);
- repository read/history inspection succeeded against all three checkouts;
- filesystem write capability was confirmed for the current OS user;
- `git push --dry-run` succeeded against all three checkouts without
  permission errors.

This confirms read access (to verify artifacts and status fields) and
write/commit/push access (to perform the commit itself), which is what
`product-team`'s role as the sole actor running `git
add`/`commit`/`push` against these checkouts requires. It does not confirm
anything beyond what was tested — e.g. it is not a confirmation of
unrestricted access, of access to any repository outside these three, or of
GitLab-side permissions (branch protection, MR rules) beyond a dry-run push.
If a `product-team` read or commit attempt against any of the three
checkouts fails despite this, escalate to Sathish as a regression rather
than assuming misconfiguration or working around it.

## Escalation and uncertainty

Where this config, `shashi-care-doc-tree.md`, or `shashi-care-gitlab-binding.md`
does not resolve a concrete question — a path that doesn't match the
documented tree shape, an approval signal that's ambiguous, a commit that
fails or lands unexpectedly, or a specialist reporting a blocker — escalate
to Sathish directly, per SOUL.md's generic Escalation rules. This config
does not introduce any project-specific escalation exception, autonomous
resolution path, or bypass of a workflow gate; it only supplies the facts
needed to recognize, for Shashi Care specifically, when a gate has or has
not been satisfied.
