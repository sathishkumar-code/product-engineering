# Prototype Metadata Template

Sidecar file for a promoted prototype export, tracked independently of the PRD's
own `repo_status`/`last_promoted_revision` — a prototype can be re-exported and
re-promoted without the PRD changing, and vice versa, so one shared field would
conflate two different things.

```markdown
# Prototype: <slug>

| Field | Value |
|---|---|
| claude_design_link | <project-level link — provenance only> |
| repo_status | not-promoted / promoted |
| last_promoted_revision | <timestamp/version last pushed to the GitLab prototypes/ folder> |
| retention | Drive copy: deleted once ClickUp Epic/Story items are created (Project Manager persona, with confirmation). GitLab copy: deleted once development is complete (same persona, same confirmation rule). |
```

Deletion is never automatic — see `skill-pjm-discipline.md`. When deletion happens,
a one-line note is left in the relevant log (`mapping-log.md` for the Drive-side
deletion, `promotion-log.md` for the GitLab-side deletion) rather than removed
without a trace.
