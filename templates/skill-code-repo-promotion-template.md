# Skill: Code-Repo Promotion Template

A generic pattern for promoting an approved document out of a fluid drafting
workspace (Drive, a wiki, etc.) into a version-controlled repo developers actually
work in. Copy this template per project and fill in the placeholders.

## Trigger
Promote a document the moment its `status` field reaches `<PROMOTION_STATUS>`
(typically the fully-approved state, not draft or in-review). Nothing promotes
before that regardless of how finished it looks.

## What promotes
`<DOCUMENT_TYPES>` — name exactly which document types cross over. Don't promote
everything in a slug's folder by default; developers referencing a promoted
document may not need every intake artifact that led to it.

## Target structure: flatten by type, don't mirror the source
The source workspace typically groups documents **by slug** (a feature's PRD, its
epics/stories, its test scenarios, all together) because that's convenient for
drafting. Resist carrying that same per-slug folder into the promotion target if
only some of those document types are meant to be promoted — a per-slug folder in
the target that's missing its non-promoted siblings looks incomplete, and an
incomplete-looking folder invites the exact question the promotion boundary was
meant to avoid ("is something missing, or is this intentional?").

Instead, group the promotion target **by document type**:
```
<target repo>/
├── <type-a>/
│   └── <slug>-<type-a>.md
├── <type-b>/
│   └── <slug>-<type-b>.md
```
Every folder in the target only ever contains what was always meant to be there —
nothing is conspicuously absent, because nothing not meant to be promoted ever had
a folder waiting for it.

## Re-promotion on revision
A promoted document is not a frozen snapshot if the source workspace keeps changing
it. If `<REVISION_POLICY>` says revisions after promotion should flow through: any
edit to an already-promoted document triggers a new promotion (new commit), not a
one-time copy. Track this with a field on the document itself (e.g.
`repo_status: not-promoted | promoted`, plus a promotion log entry) so it's always
clear whether the repo's copy matches the workspace's current version.

## Mechanics
`<MECHANISM>` — e.g., manual (a person runs a copy script, reviews the diff, commits
and pushes themselves) versus automated (a persona or job does this unattended).
Start manual; only automate once the trigger and cadence have been observed to be
reliable — see the project config for current phase.

## Idempotency / promotion log
Before promoting: check the promotion log for this slug's last-promoted version.
Only promote if the source has changed since. Log immediately after: slug, version/
revision promoted, commit reference, date, who ran it.

## Reverse flow: an external-contributor inbox (optional)
Some content flows the other direction — a team submits work *into* the repo for
review, rather than the workspace promoting *out* to it (e.g. an externally-drafted
design document a reviewing agent needs to pick up). If this applies:
- Give it its own folder, separate from the already-approved target (e.g.
  `<type>-submissions/`, sibling to `<type>/`), so incoming and approved never sit
  in the same place.
- Folder naming stays `<slug>` regardless of the submitted file's format — that's
  what lets a reviewing agent locate the right submission even when the format
  varies submission to submission.
- **Single permanent branch, short-lived branches per submission**: contributors
  get write access via merge requests, not direct pushes to the permanent branch;
  an approver gates the merge. This keeps write access broad (anyone can submit)
  while keeping what's actually "approved" narrow and reviewed.
- A reviewing agent with access to this folder should default to **read-only**
  unless explicitly told otherwise — reviewing and writing a verdict elsewhere is a
  different action than committing into the inbox itself.

## Promotion log format
```
## <slug>
- Document: <type, e.g. PRD>
- Promoted revision: <version or last-modified timestamp of the source>
- Commit: <sha or link>
- Promoted on: <date>
- Promoted by: <person>
- Deleted (if applicable): <date> — <one-line reason, e.g. "prototype removed,
  development complete">
```
