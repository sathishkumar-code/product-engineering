# Skill: Finalize Document Discipline

Generic, reusable procedure for turning a PRD, Enhancement Request (ER), Technical
Design (TD), or Spec (`spec.md`) that has accumulated a PM↔SA (and Sathish)
revision/deliberation trail into a clean, first-read-ready version of the same
document — without losing anything that document type's own template already
treats as permanent record. Pair with `shashi-care-finalize-config.md` for the
paths and companion-file names this project uses; this file itself should not
need per-project edits.

**Scope note**: `spec.md` is in scope, confirmed against a real sample that
already carries its own accumulated revision-noise patterns (see below) even
though `framework/templates/spec-template.md` doesn't formally define a Revision History
section — real documents keep one anyway, and this pass treats it the same as
PRD/TD's. `tech-spec-<slug>.md` — System Architect's structurally-parallel
derived document — is deliberately NOT yet in scope; extend it only once a real
sample confirms the same patterns apply there too, don't infer it from the
`spec.md` parallel alone.

## Mission
A document that's been through several review rounds accumulates its own working
history *inside* its prose — narrated decisions, references to which review round
settled something, superseded text nobody deleted. That history is valuable while the
document is still moving, and becomes noise once it's settled: someone opening the
document for the first time shouldn't need to have sat through the PM/SA conversation
that produced it. Finalize produces that "first-time reader" version, in place,
without touching the parts of the document whose entire purpose is to be a permanent
record.

## When this runs
On request, whenever revision noise has visibly accumulated in a document — not gated
to `status: approved`, not tied to promotion, and not a numbered pipeline stage. A
document can be finalized mid-review if its narrated sections are getting hard to
read, then finalized again later after more rounds add more noise. Whoever owns the
document type runs it on that type: Product Manager finalizes PRDs, Enhancement
Requests, and the `spec.md` derived from either — all of which it authors; System
Architect finalizes Technical Designs it authors (not yet its own
`tech-spec-<slug>.md` — see the scope note above). Neither finalizes the other's
document type — same authorship boundary as everywhere else in this system.

## What counts as revision noise (rewrite, don't preserve verbatim)
Concrete patterns to look for, drawn from how this project's documents actually
accumulate history:
- Narrated decisions: "**Decided with Sathish (2026-08-21):** build it as X, not Y —
  because Z" → becomes a plain declarative statement of the settled design, in the
  relevant section, with no date-stamped narration.
- Review-provenance references: "per Round 2 Finding 1", "confirmed in SA's Round 1
  review", "PRD change #2" → remove the citation, keep only the substance it was
  citing.
- Investigative narration: "I checked the as-built schema and found...", "Confirmed:
  the dedup key is well-defined because..." → becomes the plain fact itself ("The
  dedup key is (resident, task-text label), per §X"), not the account of how it was
  checked.
- `~~strikethrough~~` superseded markers → resolve fully: delete the superseded
  content if nothing of it survives, or replace with the current design if part of it
  does. Don't leave a strikethrough in a document meant to read as settled.
- Meta-commentary about the review process itself ("Section 13 now classifies this as
  a binding rule, not example data") → fold the *substance* (it IS a binding rule)
  into the relevant section as a plain statement; drop the commentary about the fact
  that it changed.

Rewrite for a reader who has no idea any of this deliberation happened — the test is:
does this sentence require having read the conversation to make sense of it? If yes,
it isn't finalized yet.

## What must never be touched
- **Open Questions** (PRD/ER §11 or §6, TD §11; `spec.md`'s own `## Open questions`
  section, lowercase heading, carried forward from its source per
  `framework/templates/spec-template.md`) — every row still genuinely unresolved stays,
  verbatim, including its Priority. Only remove a row when the question itself was
  actually answered during review (in which case fold the answer into the relevant
  section using the rewrite rule above, and remove the now-answered row) — never
  remove a row just because it's inconvenient to the "clean" read.
- **Explicit deferrals** — anything already recorded as intentionally not being built
  now (PRD/ER's Out of scope, PRD's §10 Next phase/explicitly deferred; `spec.md`'s
  own `### Out of scope` under its `## Scope` heading) stays visible. A finalize pass
  tidies language; it never quietly erases a documented decision not to build
  something.
- **Front-matter** (status, repo_status, last_promoted_revision, product,
  review_round, etc.) — informational fields this pass has no business changing.
  `spec.md` has no YAML front-matter; its markdown header table (Source document,
  Status, repo_status, last_promoted_revision) serves the identical role and gets
  the identical protection — including the "re-derive this spec if the source
  document's Revision History gets a new entry after this date" instruction
  embedded in its Source document row. That's a separate, standing re-derivation
  trigger this pass never edits or acts on — finalize cleans the document's current
  prose, it doesn't decide whether a fresh re-derivation from a newer source
  revision is needed.
- **Companion review files** — `SA-comments-<slug>.md`, any
  `_PRD-changes-for-SA.md`-style changeset file, and the source document's own
  Revision History table's *reason for existing*. These are the permanent audit trail
  this project deliberately keeps separate from git history (see
  `shashi-care-doc-tree.md`) — finalize cleans the *document*, it doesn't delete the
  record of why it changed. (See the Revision History rule below for the one place
  this pass does touch that record, and how to do it without erasing the "why.")
  `spec.md` has no dedicated companion review file of its own — when a `spec.md`
  revision traces back to an SA finding rather than a source-document change or
  direct Sathish input, `SA-comments-<slug>.md` above is the relevant read-only
  record to check; there's no second, spec-specific file.

## Revision History: condense, never delete outright
Applies equally to `spec.md`'s own `## Revision history` table — real `spec.md`
documents keep one (informally, since `framework/templates/spec-template.md` doesn't yet
define the section) and it accumulates the same narration patterns as PRD/TD's,
so treat it the same way here rather than leaving it untouched for lack of a
template definition. A run of granular Revision History rows whose individual
"what changed" is now fully reflected in the clean prose can be consolidated into
one row — but the row that
survives must still carry the *reason* the change happened (what question or session
triggered it), because that's specifically what this table exists to preserve that
git history doesn't. Drop a row only when it is truly redundant with a sibling row
(adds no information beyond what's already recorded) — never because collapsing it
makes the table shorter. When in doubt, consolidate rather than delete, and say in
your summary to Sathish which rows you merged and why.

## Detecting a resolution that's actually an enhancement

This is one specific trigger into the general "Change requests to an in-flight
(not yet released) feature" rule (see PROCESS-WALKTHROUGH.md and
`skill-pm-discipline.md`) — a review-round resolution surfacing the same kind of
scope question a customer input or a Developer-noticed deviation would. The
decision procedure below is that same rule, applied at this discovery point.

For every resolved item you're about to fold into the document's prose, check it
against that document's own recorded scope boundary — PRD/ER's §2/§3 In scope vs. Out
of scope, TD's §2 Goals vs. Non-goals, or `spec.md`'s own `## Scope` → In
scope/Out of scope (same shape as its source PRD/ER). This check applies even when the resolution
is about the *same* feature/slug being finalized, not only when it's about something
else entirely — a decision can expand what this very document commits to beyond what
it originally scoped, and that's still an enhancement candidate, not just a detail to
fold in.

If a resolution reads as adding capability beyond that recorded boundary, **do not
decide alone** whether it's a genuine scope expansion or an enhancement that belongs in
its own document. Stop and ask Sathish directly, describing the specific resolution
and its text, and let him choose:
1. **Spin it into a new (or updated) Enhancement Request** — draft via the standard PM
   intake pathway (`framework/templates/enhancement-intake-questions.md` →
   `framework/templates/enhancement-request-template.md`), filed at
   `02_prd/enhancements/<new-slug>/` per `shashi-care-doc-tree.md`'s per-slug shape.
2. **Accept it as an explicit scope expansion of this same document** — update its own
   §2/§3 Scope (TD §2 Goals, or `spec.md`'s own `## Scope`) to say so plainly, and add
   a Revision History row recording the expansion (this is the one case where a *new*
   Revision History row is added by a finalize pass, not just consolidated).
3. Something else Sathish specifies.

Never silently fold an out-of-scope-boundary resolution into the clean prose as if it
had always been in scope, and never silently drop it either — both are "categorizing
or skipping on your own," which is exactly what to avoid here.

**Authorship boundary carries through.** If System Architect is finalizing a Technical
Design and finds a candidate enhancement this way, System Architect does not draft the
Enhancement Request itself — SA doesn't invent product requirements, full stop, same
as every other SA rule. Write a short handover note to Product Manager describing the
candidate (same shape as SA's existing handover protocol) and let Product Manager run
the intake pathway once Sathish has made the call above. When Product Manager is
finalizing its own PRD/ER/`spec.md` and finds a candidate this way, Product Manager
may draft the new intent.md/Enhancement Request itself once Sathish has confirmed
that's the right move — Product Manager already owns that document type.

## Procedure
1. Read the full document being finalized, plus whatever companion files fed changes
   into it (the `SA-comments-<slug>.md` file, any `_PRD-changes-for-SA.md`-style changeset)
   — you need to know which parts of the current text are settled-but-still-narrated
   versus still genuinely open before touching anything.
2. Rewrite every passage matching the revision-noise patterns above into plain,
   declarative, settled language.
3. Run the enhancement check above on every resolution you fold in; escalate per that
   section rather than deciding alone.
4. Consolidate Revision History per the rule above; leave Open Questions,
   front-matter, and companion files untouched except where explicitly directed.
5. Re-read the whole document as if seeing it for the first time. If any sentence
   still requires knowing the review conversation to parse, it isn't done.
6. Edit the canonical file in place — there is no separate "finalized" file type in
   this project's doc tree; the same PRD/ER/TD/`spec.md` document that promotes to
   GitLab is the one this pass cleans up, whatever its instantiated filename is per
   the project's own doc tree (this project: `prd-<slug>.md` / `enhancement-request-
   <slug>.md` / `TD-<slug>.md` / `spec.md`, the last sitting alongside its source
   PRD/ER in the same `02_prd/<slug>/` folder, not a new type of its own).
7. Summarize to Sathish what was rewritten, which Revision History rows were
   consolidated (and why), and list every enhancement-candidate escalation raised
   during the pass, even ones already resolved by his answer in this same
   conversation.

## What this pass does NOT do
- Doesn't touch `status`, `repo_status`, or any other front-matter field (or
  `spec.md`'s equivalent header-table fields).
- Doesn't edit or archive the companion review-comments/changeset files themselves.
- Doesn't remove or reword a still-unresolved Open Question.
- Doesn't decide, on its own, that a resolution is or isn't enhancement-scale — always
  escalates per the rule above.
- Doesn't run on a different persona's document type (PM never finalizes a TD; SA
  never finalizes a PRD/ER).
- Doesn't run on `tech-spec-<slug>.md` yet — deliberately out of scope until a real
  sample confirms the same noise pattern applies there too (see the scope note at
  the top of this file); don't extend it from the `spec.md` parallel alone.
