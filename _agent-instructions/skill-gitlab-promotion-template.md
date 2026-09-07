# Skill: GitLab Promotion Template (superseded — see templates/)

**This copy is no longer the canonical version.** Sathish confirmed
`templates/skill-gitlab-promotion-template.md` as the canonical, generic
reusable template (2026-09-04) — it had picked up two sections this copy
never received while the two drifted apart: a "Reverse flow: an
external-contributor inbox" pattern (for a team submitting work *into* a
repo, rather than the workspace promoting *out* to it), and a "Deleted (if
applicable)" line in the promotion log format. Read
`templates/skill-gitlab-promotion-template.md` instead of this file for the
actual procedure.

This file is kept as a stub, rather than deleted outright, because other
governance files still reference `skill-gitlab-promotion-template.md`
grouped with `_agent-instructions/`-only generic templates (e.g.
`skill-doc-tree-template.md`) — `shashi-care-doc-tree.md`'s "Instantiated
from" line and `shashi-care-process-architect-config.md`'s own file listing
both do this. Those references were not updated as part of this change
(`shashi-care-process-architect-config.md` is explicitly out of scope for
this pass) — reconciling them is a follow-up item.

## Shashi-Care-specific note (preserved from this copy — not copied into the canonical generic template, per instruction)
Shashi Care's own `shashi-care-gitlab-binding.md` doesn't instantiate this
template — it authors documents directly in the version-controlled repo from
the start (no separate drafting workspace, no promotion event), which this
template doesn't describe. The canonical template stays the reusable pattern
for a project that still wants the promote-on-approval shape; whether Shashi
Care's model is itself worth generalizing into a second reusable template
hasn't been decided — flagging that as an open question rather than doing it
unprompted, per the generic-skill-vs-project-config split this persona
maintains.
