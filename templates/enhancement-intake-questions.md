# Enhancement Intake — Conversation Questions

Ask these before drafting `enhancement-request-template.md`. Don't skip straight to
drafting from a one-line request — an enhancement described in a sentence usually
hides at least one of these answers.

1. Which existing feature or PRD does this touch? (name or path — if unsure, this
   persona should search the existing PRDs before asking the user to recall it)
2. What does the system do today, in your own words? (Then verify against the actual
   as-built doc — don't take the user's recollection as the "current behavior"
   section without checking.)
3. What's the desired behavior instead?
4. What's driving this now — a customer request, a gap noticed during use, something
   adjacent to a bug?
5. Any constraints — must ship with a specific release, must not break a specific
   other flow, must stay backward-compatible with existing data?
6. Who requested it / who should be listed as the requester?

If the answers reveal this is actually large enough to need its own scope/personas/
NFRs treatment, say so and propose switching to the PRD template instead of forcing
it into the lighter enhancement shape.
