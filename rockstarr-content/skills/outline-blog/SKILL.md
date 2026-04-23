---
name: outline-blog
description: "This skill should be used when the user asks to \"outline a blog post\", \"plan the researched blog\", \"structure the post before drafting\", or names a researched informational topic they want to blog about. It is the first step of the two-step researched-blog flow and must complete before draft-blog runs. Produces an outline file in 03_drafts/content/ with working title, angle, audience, H2 structure, CTA, target keyword, rough word-count, and cited first-party KB sources. The outline must be explicitly approved by the user before any drafting begins. Opinion pieces do not outline — route those to draft-thought-leadership instead."
---

# outline-blog

The outline gate for the **researched, informational** blog lane.
Researched-blog drafts are expensive to fix after the fact — a bad
angle, a missing H2, or a CTA that does not fit the offer can
force a full rewrite. This skill catches those problems cheaply,
before any prose is written.

In v0.2 the blog lane split. Researched, educational pieces are
outlined here and drafted with `draft-blog`. Opinion, stance-led
pieces skip the outline and run single-shot via
`draft-thought-leadership`. If the user brings a topic that is
clearly opinion-first, steer them to the TL lane instead of
outlining.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User picks a researched blog-format topic from the month's
  `content-topics_YYYY-MM.md` file.
- User asks to outline a researched post on a fresh topic.
- A previous researched-blog draft was rejected for structural
  reasons and the reviewer wants to re-outline before the next
  attempt.
- User asks to "outline an article" — this skill handles both,
  since the research-heavy article lane merged into draft-blog
  in v0.2.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/01_knowledge_base/index.md` exists.
- Either a specific topic (title + angle) from the user, OR a
  pointer to a line item in `02_inputs/content-topics_*.md`.

If the style guide is missing or unapproved, refuse and point the
user at `rockstarr-infra:generate-style-guide`.

## Inputs

Read in this order:

1. Style guide — full. The Channel Adaptation section for blog /
   long-form drives the outline shape. The Tone Definition and
   Style Rules drive how the outline is written.
2. Client profile — positioning, audience sophistication, offers
   (so the CTA points somewhere real).
3. The source topic — either the user-supplied topic or the
   researched-blog line item from the month's
   `02_inputs/content-topics_YYYY-MM.md`. Preserve the angle
   from the topic file; do not drift it in the outline.
4. First-party KB files (`kb_scope: owned`,
   `style_guide_eligible: true`) that are relevant to the topic.
   Cite them by filename in the outline.
5. Third-party KB files (`kb_scope: third_party`) that are
   relevant — allowed as references only, clearly marked.

## Output

Write to
`/rockstarr-ai/03_drafts/content/outline_[slug].md` where `[slug]`
is the kebab-cased working title, max 60 chars. If the file exists,
append `-2`, `-3`, etc. Never overwrite an existing outline without
archiving it first to `99_archive/`.

Required front-matter:

```yaml
# ---
channel: "blog"
stage: "outline"
title: "Working title — human-readable"
slug: "kebab-cased-slug"
pillar: "Pillar name"
audience: "One-line audience"
target_keyword: "primary keyword phrase"
word_count_target: 1500
produced_by: "rockstarr-content/outline-blog@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "from style-guide.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
  - "processed/offer-positioning.md"
third_party_references:
  - "processed/third-party/hbr-founder-led-growth.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure (in this exact order):

```markdown
# [Working title]

## Angle
One sentence: the specific POV, not a summary. If this reads like a
topic label, go back and sharpen it.

## Who this is for
One to two sentences. Pull the audience language from the style
guide's Audience Definition section, not from generic SaaS-speak.

## Why now
One line. What makes this post worth reading this week?

## Outline

### H1 (post title — may differ from working title)
A candidate title plus one alternate for the reviewer to choose from.

### Intro (roughly 100 words)
What the reader will believe by the end. One line.

### H2 1 — [Section heading]
- Bullet substance (3 to 5 bullets). Each bullet names a claim, a
  story, or a stat the section will cover.
- Cite the first-party KB source next to each claim, e.g.,
  `[source: processed/founder-call-2026-03.md]`.

### H2 2 — [Section heading]
- ...

### H2 3 — [Section heading]
- ...

(Researched blogs usually land in 3 to 5 H2s. More than 6 and
the piece is really an anchor essay — the outline still feeds
draft-blog, but flag the length and ask the user whether to cap
it or split into two pieces.)

### Closing
- What to leave the reader with. One line.

## CTA
The specific next action. Pull the offer from client-profile.md.
Do not invent products or URLs.

Examples of good CTAs:
- "Book a 20-minute positioning review — link."
- "Reply to this post with your current first-line."

Examples of bad CTAs:
- "Follow for more content" (generic).
- "DM me to learn more" (not specific enough).

## Evidence map
Short table mapping each H2 to the first-party KB file(s) that
back it up. If any H2 has no first-party source, flag it — the
draft will need to rely on the author's own claim, which is fine
but must be visible at outline time.

## Open questions
Anything the outline cannot resolve without the client. List them
explicitly so the human reviewer can answer before approval.

## References (third-party, optional)
Attributed list with titles and links. These are for the draft's
References section, not for paraphrasing.
```

## Approval gate

After writing the file:

1. Summarize the outline in chat: title, angle, H2 headings, CTA,
   word-count target, open questions.
2. Ask the user via `AskUserQuestion`:
   - Approve outline as-is
   - Amend outline (free-text direction)
   - Reject and restart
3. On approve, update the file's front-matter
   `approval_status: "approved"` and add `approved_at` and
   `approved_by`. This is the signal `draft-blog` reads to decide
   whether it is allowed to draft.
4. On amend, apply the edits and re-present.
5. On reject, archive the outline to `99_archive/` with a timestamp
   and stop. Do not recurse inside this skill.

`draft-blog` refuses to run if `approval_status != "approved"`.

## What NOT to do

- Do not draft paragraphs, hooks, or full intros in the outline.
  Bullets only.
- Do not invent customer stories, stats, or numbers. If the outline
  needs one the client has not made public, put it in "Open
  questions".
- Do not use third-party KB content as a claim. Third-party only
  appears in References.
- Do not update `approval_status` to "approved" without an explicit
  user approve signal.
- Do not proceed to `draft-blog` from inside this skill. Approving
  the outline and starting the draft are two separate events.
