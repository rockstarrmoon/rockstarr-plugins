---
name: draft-article
description: "DEPRECATED back-compat alias. This skill should be used when the user asks to \"draft an article\" or \"draft a long-form article\". In v0.2 it routes to draft-blog (researched, outline-first) or draft-thought-leadership (opinion, single-shot). Scheduled for removal in v0.3."
---

# draft-article — DEPRECATED

**This skill is a back-compat alias.** In v0.1 of rockstarr-content,
`draft-article` meant "long-form research-heavy piece". In v0.2 the
long-form lane split into two skills with clearer names:

- `draft-blog` — researched, informational, outline-first. The
  old draft-article workflow lives here now.
- `draft-thought-leadership` — shorter, opinion-driven,
  single-shot. The lane for pieces that argue a point.

## When this skill fires

- User types "draft an article", "write the article", or similar.
- A saved workflow still references `draft-article`.

## What this skill does

1. Explain the rename in chat — one short paragraph so the user
   understands the lane split, not a wall of text.
2. Ask via `AskUserQuestion` which lane the user meant:
   - **Researched blog** (educational, outline-first) → route to
     `outline-blog` (if no outline exists) or `draft-blog` (if
     an approved outline exists).
   - **Thought leadership** (opinion, single-shot) → route to
     `draft-thought-leadership`.
3. Hand off to the chosen skill with the user's topic preserved.
   Do not draft anything in this skill.

## Removal timeline

- v0.2: this shim ships; every call routes through the disambiguation
  question above.
- v0.3: this skill is deleted. Any remaining reference to
  `draft-article` returns a "not found" error.

## What NOT to do

- Do not draft any content inside this skill. Its only job is
  the disambiguation prompt + handoff.
- Do not pick a lane silently for the user. Always ask.
- Do not suppress the deprecation notice. The user should know
  the shim is going away in v0.3.
