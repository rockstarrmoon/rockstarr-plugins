---
name: outline-blog
description: "This skill should be used when the user asks to \"outline a blog post\", \"plan the researched blog\", \"structure the post before drafting\", or names a researched informational topic they want to blog about. It is the first step of the two-step researched-blog flow and must complete before draft-blog runs. As of v0.4 it runs an external WebSearch research phase on top of the first-party KB read, and produces an outline that includes SEO/GEO targets, a required FAQ section (3-5 questions), a keyword placement plan, an internal linking plan with anchors and target URLs, an external sources table, meta title and meta description drafts, and the standard angle/audience/H2 structure. The outline must be explicitly approved by the user before any drafting begins. Opinion pieces do not outline — route those to outline-thought-leadership instead."
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
- `/rockstarr-ai/00_intake/stack.md` exists. Specifically,
  `website_base_url` must be set — the internal-linking plan
  cannot be built without it. If missing, refuse and route the
  user back at `rockstarr-infra:capture-stack`.
- `/rockstarr-ai/01_knowledge_base/index.md` exists.
- The shared blog-seo-geo reference at
  `rockstarr-infra/skills/_shared/references/blog-seo-geo.md`
  exists. This file ships in **rockstarr-infra v0.8.2+**. If
  missing, refuse with this message:

  > Cannot run: the shared blog-seo-geo reference is missing.
  > Update rockstarr-infra to v0.8.2 or later — the reference
  > ships at `skills/_shared/references/blog-seo-geo.md`. The
  > upgrade is pure additive (one new file), so there's nothing
  > else to coordinate.

- Either a specific topic (title + angle) from the user, OR a
  pointer to a line item in `02_inputs/content-topics_*.md`.

If the style guide is missing or unapproved, refuse and point the
user at `rockstarr-infra:generate-style-guide`.

## Inputs

Read in this order:

1. The shared blog-seo-geo reference — full. The GEO patterns,
   FAQ requirement, keyword placement rules, internal linking
   rules, inline source URL convention, and meta title /
   description specs apply across the whole outline pass. Do
   not reinvent — the reference IS the contract.
2. Style guide — full. The Channel Adaptation section for blog /
   long-form drives the outline shape. The Tone Definition and
   Style Rules drive how the outline is written.
3. Client profile — positioning, audience sophistication, offers
   (so the CTA points somewhere real).
4. `stack.md` — pull `website_base_url` for the internal-linking
   plan. Pull any other linkable client URLs noted in stack
   (service pages, primary blog index).
5. The source topic — either the user-supplied topic or the
   researched-blog line item from the month's
   `02_inputs/content-topics_YYYY-MM.md`. Preserve the angle
   from the topic file; do not drift it in the outline.
6. First-party KB files (`kb_scope: owned`,
   `style_guide_eligible: true`) that are relevant to the topic.
   Cite them by filename in the outline.
7. Third-party KB files (`kb_scope: third_party`) that are
   relevant — allowed as references only, clearly marked.

## External research phase (additive, on top of KB read)

Researched blogs need current data the founder hasn't yet written
about themselves. The first-party KB carries voice and proprietary
claims; the external research carries market stats, regulations,
expert citations, and local details that earn AI citation. Both
matter. Run this phase AFTER reading the KB so you know what the
KB already covers and don't duplicate.

Use `WebSearch` to gather:

1. **Top-ranking articles** for the target keyword. Read the top
   3-5 results to understand the existing competitive set and
   identify angles already covered (so the outline can pick a
   different one or a sharper one).
2. **Specific data points** — statistics, dollar figures,
   timelines, regulations, market data. The more specific and
   current, the better. Aim for 5-10 candidate stats.
3. **Expert sources** — government agencies, industry
   associations, research firms, named studies. AI systems
   prefer content that references authoritative sources (per
   the blog-seo-geo reference, signal #2).
4. **Local or niche details** — for geo-targeted blogs (e.g.
   "Houston commercial real estate"), find city-specific
   regulations, market stats, neighborhood data, local
   resources.
5. **"People Also Ask" questions** for the target keyword.
   These are the canonical FAQ questions for the outline (per
   the blog-seo-geo reference, FAQ section).

Capture each source with: a one-line summary of what it
contributes, the verbatim stat or fact, the source URL, and
the publication date if available.

The first-party voice integrity rule still applies. External
sources are CITED and LINKED, never paraphrased into the
client's voice. The footprint of someone else's analysis stays
visible to the reader.

If `WebSearch` returns nothing useful (rare but possible for
hyper-niche topics), surface this to the user before proceeding.
A blog with no external citations can still ship, but it loses
GEO leverage and the human should know that's the trade.

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
meta_title_draft: "Up to 60 chars, keyword near front"
meta_description_draft: "Up to 155 chars, includes keyword + value prop"
faq_questions:
  - "Direct-language question 1?"
  - "Direct-language question 2?"
  - "Direct-language question 3?"
keyword_placement_plan:
  title: true
  first_100_words: true
  h2_indexes_carrying_keyword: [1, 3]
  final_paragraph: true
internal_links_planned:
  - { anchor: "commercial real estate broker", target: "https://example.com/services" }
  - { anchor: "previous post on zoning", target: "https://example.com/blog/zoning-101" }
external_sources_planned:
  - { fact: "Houston absorbed 6.2M sqft in Q3 2024", source_url: "https://www.cbre.com/insights/...", date: "2024-10-15" }
produced_by: "rockstarr-content/outline-blog@0.4.0"
produced_at: "ISO timestamp"
style_guide_version: "from style-guide.md front-matter"
seo_geo_reference_version: "matched from blog-seo-geo.md front-matter"
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

## SEO / GEO targets
- **Target keyword:** [primary keyword from topic file]
- **Estimated search intent volume:** [if known from topic file]
- **Top 3 ranking competitors:** [titles + URLs from research]
- **Differentiation angle:** [one line — how this piece earns
  the click vs. existing rankers]

## Outline

### H1 (post title — may differ from working title)
A candidate title plus one alternate for the reviewer to choose
from. Title carries the keyword. Each option ≤ 70 chars.

### Intro (roughly 100 words)
What the reader will believe by the end. One line.
Confirm: the target keyword lands in the first 100 words.

### H2 1 — [Section heading]
- Bullet substance (3 to 5 bullets). Each bullet names a claim, a
  story, or a stat the section will cover.
- Cite first-party KB sources next to claim-style bullets:
  `[source: processed/founder-call-2026-03.md]`.
- Cite external sources next to stat-style bullets with the
  exact format the draft will carry inline:
  `[Source: https://www.cbre.com/insights/...]`.

### H2 2 — [Section heading]
- ...

### H2 3 — [Section heading]
- ...

(Researched blogs usually land in 4 to 7 H2s including the FAQ
section. More than 7 and the piece is really an anchor essay —
the outline still feeds draft-blog, but flag the length and ask
the user whether to cap it or split into two pieces.)

### H2 [N] — Frequently Asked Questions
**Required section** per the blog-seo-geo reference. List the 3
to 5 H3 questions here. Pull from the WebSearch "People Also
Ask" gathering plus any first-party recurring-question
material.

- **Q:** [Direct-language question 1]
  - One-line gloss of what the answer will say
  - Source if external: `[Source: ...]`
- **Q:** [Direct-language question 2]
  - ...
- **Q:** [Direct-language question 3]
  - ...

Each answer in the draft will be 2-4 sentences,
direct-answer-first (per the blog-seo-geo reference).

### Closing
- What to leave the reader with. One line.
- Confirm: the target keyword appears in the final paragraph.

## CTA
The specific next action. Pull the offer from client-profile.md.
Do not invent products or URLs.

Examples of good CTAs:
- "Book a 20-minute positioning review — link."
- "Reply to this post with your current first-line."

Examples of bad CTAs:
- "Follow for more content" (generic).
- "DM me to learn more" (not specific enough).

## Keyword placement plan
For each H2, note whether it carries the target keyword in its
heading. Aim for 1-2 H2s carrying it, naturally. List the H2
indexes that will and a one-line justification.

## Internal linking plan
3-5 internal links inline in the body. Pull `website_base_url`
from `stack.md` for the base. Each entry: anchor text, target
URL, the H2 section it lives in, and a one-line justification
(why the surrounding sentence already discusses what's being
linked).

| Anchor text | Target URL | H2 | Why |
|---|---|---|---|
| ... | ... | ... | ... |

At least one entry must point to the client's primary service
page related to the blog topic. If the topic doesn't have an
obvious service-page match, flag it as an open question.

### Cluster-aware defaults (v0.5)

If the source topic carries `from_backlog: true` and a
`cluster` + `cluster_role` (set by `ideate-topics` when it
draws from `02_inputs/seo/backlog.md`), default the internal
linking plan as follows:

- **If `cluster_role: supporting`** — at least one entry
  links to the parent pillar page. Use `parent_pillar_slug`
  from the topic and resolve to the published URL via
  `05_published/_publish.log` (search the log for that slug
  and pull the URL). If the pillar has not shipped yet,
  flag this as an open question — the support shouldn't
  ship before its pillar (per content-calendar's
  cluster-ordering rule), but if the user wants to proceed,
  use a placeholder URL noted as `[CLIENT TO CONFIRM:
  pillar URL when shipped]`.
- **If `cluster_role: supporting`** — also include 1-2
  links to peer supporting posts in the same cluster (read
  the backlog to find them). Use the same publish-log
  lookup; placeholder if not yet shipped.
- **If `cluster_role: pillar`** — at least one entry per
  H2 section links FROM the pillar TO a supporting post in
  the cluster (inverse direction). Pillars carry the most
  outbound cluster links because they're the cluster's
  navigation hub. Aim for one link per support (5-7
  internal links total when the pillar has 5-7 supports).
  Pillars frequently exceed the standard 3-5 link guidance —
  that's expected. Note in the front-matter that this is a
  pillar page (`cluster_role: pillar`) so the SEO/GEO
  checklist's "3-5 links" check is interpreted as "≥3"
  rather than "exactly 3-5."

These defaults are recommendations the human reviewer can
adjust. The pillar↔support linking is what makes a topic
cluster work as a topical-authority signal — skipping it
sacrifices the strategic value of the cluster structure.

## External sources planned
Every external statistic, regulation, or expert citation the
draft will use. Each row carries the verbatim fact or stat,
the source URL, and the publication date if known.

| Fact / claim | Source URL | Date |
|---|---|---|
| ... | ... | ... |

Aim for ≥ 5 entries (per the blog-seo-geo reference). Fewer
than 5 means the research phase didn't gather enough; surface
to the user.

## Evidence map (first-party)
Short table mapping each H2 to the first-party KB file(s) that
back it up. If any H2 has no first-party source, flag it — the
draft will need to rely on the author's own claim, which is
fine but must be visible at outline time.

## Meta title draft
Up to 60 characters. Keyword near the front. Specific and
actionable. Carry into the draft's front-matter.

## Meta description draft
Up to 155 characters. Includes the keyword. Summarizes what the
reader will learn + a soft CTA. Carry into the draft's
front-matter.

## Open questions
Anything the outline cannot resolve without the client. List
them explicitly so the human reviewer can answer before
approval.

## References (third-party, for the draft's References section)
Attributed list with titles and links. These feed both the
inline `[Source: URL]` placements in the draft AND the
References section at the bottom of the draft body. Never
paraphrased into the client's voice.
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
- Do not paraphrase external research findings into the client's
  voice. External sources are CITED with `[Source: URL]` and
  carried verbatim into the draft. The footprint of someone
  else's analysis stays visible.
- Do not skip the FAQ section. The blog-seo-geo reference
  treats it as required. An outline without 3-5 FAQ questions
  is incomplete.
- Do not skip the external research phase because the
  first-party KB feels rich. The KB carries voice; external
  research carries the cited stats AI systems verify. Both
  matter.
- Do not skip the keyword placement plan or the internal
  linking plan. These are the artifacts that let `draft-blog`
  do its job downstream — without them it has to guess.
- Do not use third-party KB content as a claim in the client's
  voice. Third-party only appears as cited references.
- Do not update `approval_status` to "approved" without an
  explicit user approve signal.
- Do not proceed to `draft-blog` from inside this skill.
  Approving the outline and starting the draft are two separate
  events.
