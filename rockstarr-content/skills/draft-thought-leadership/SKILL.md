---
name: draft-thought-leadership
description: "This skill should be used when the user asks to \"draft a thought-leadership piece\", \"write an opinion post\", \"push back on X\", \"make the argument for Y\", or picks a thought-leadership-format topic. It produces a shorter, hard-hitting, opinion-driven piece in 03_drafts/content/ that argues a point, pushes back on a current view, or stakes a stance. Single-shot: no outline gate. Weight comes from the stance, not the research. This is the lane where the client sounds like themselves with a point to make."
---

# draft-thought-leadership

Draft an opinion-driven piece in the client's approved voice.
Thought leadership is the lane where the client takes a stance:
pushes back on a prevailing view, names a pattern, reframes a
problem, or says something that deserves disagreement. Weight
comes from the POV, not the evidence pile. Shorter and sharper
than a researched blog.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User asks to draft a thought-leadership piece, an opinion post,
  or a point-of-view piece.
- User picks a thought-leadership-format topic from the month's
  `02_inputs/content-topics_YYYY-MM.md`.
- User names a stance, a provocation, or an argument they want on
  the page.

Single-shot in v0.2. No outline gate. Thought leadership earns its
weight from the stance — adding an outline step slows down the
kind of piece that works best when it goes from conviction to page
fast.

If the user asks for something clearly researched and informational
(explain how something works, step-by-step how-to, roundups of
tactics), steer them to the researched blog lane (`outline-blog`
plus `draft-blog`) instead.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- Either a specific stance or topic from the user, OR a pointer to
  a thought-leadership line item in
  `02_inputs/content-topics_YYYY-MM.md`.
- `/rockstarr-ai/00_intake/stack.md` — so the CTA points to a real
  destination.

If the style guide is missing or unapproved, refuse and point the
user at `rockstarr-infra:generate-style-guide`.

## Inputs

Read in this order:

1. Style guide — full. The **Brand Personality**, **Tone Definition
   contrast pairs**, and **Style Rules** drive voice. The **Channel
   Adaptation — blog / long-form** section (or a dedicated
   thought-leadership subsection if present) drives format.
2. Client profile — positioning, the recurring arguments or
   unpopular truths the client makes, the audience, and the offer
   the CTA will point to.
3. The stance — either the user-supplied POV or the topic-file
   entry. Preserve the angle exactly; do not soften it.
4. First-party KB files (`kb_scope: owned`) that back the stance —
   founder calls, internal memos, past writing. This is where the
   credibility comes from.
5. Third-party KB files (`kb_scope: third_party`) — allowed only
   as a contrast point the client is pushing back on, or as a
   cited reference. Never paraphrased as if the client said it.
6. Recent thought-leadership entries in `05_published/_publish.log`
   — to avoid recycling last month's argument.

## Output

Write to
`/rockstarr-ai/03_drafts/content/[slug].md` where `[slug]` comes
from the working title (kebab-cased, max 60 chars). If the file
exists, append `-v2`, `-v3`.

Required front-matter:

```yaml
# ---
channel: "thought-leadership"
title: "Working headline"
slug: "kebab-cased-slug"
pillar: "Pillar name"
audience: "One-line audience"
stance: "One-line statement of the POV being argued"
target_keyword: "primary keyword phrase (optional — TL is stance-led, not SEO-led)"
word_count: 812
reading_time_minutes: 4
produced_by: "rockstarr-content/draft-thought-leadership@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
third_party_references: []
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference from stack.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
stop_slop_score: 41   # five-dimension score after the mandatory final pass
# ---
```

Body structure:

```markdown
# [Working headline]

[Hook — 1 to 3 short paragraphs. Open on a concrete scene, a claim
that invites disagreement, or a pointed question. No generic
openers. No LinkedIn-speak. The hook should telegraph the stance.]

[Optional: one line of stance — the argument in a sentence. Some
pieces need this explicit; others make it implicit in the hook.
Match the style guide.]

## [H2 1 — usually the argument's core claim or the counter-view
being pushed back on]

[Body — 2 to 4 short paragraphs. Lean on first-party stories and
direct observation. One sharp idea per section.]

## [H2 2 — usually the reframe or the "what I think instead"]

[Body — 2 to 4 short paragraphs. This is where the POV gets
concrete. If the client has a framework or a named rule, use the
exact name from the KB.]

## [H2 3 — optional; often the "so what" or "what changes"]

[Body — 2 to 4 short paragraphs.]

## [Closing heading — can be "So what now" or a stance-specific
heading; not always "Conclusion"]

[Close — one or two short paragraphs. Land the stance. What is
the reader supposed to do or believe after reading this?]

## [CTA heading — often "Next step" or client-specific]

[The specific CTA. Thought-leadership CTAs can lean toward
conversation or toward offer — pick per style guide and the offer
from client-profile.md. Do not invent URLs.]

## References
(Only if third_party_references is non-empty. Usually empty for
thought leadership — when present, these are the views being
pushed back on, clearly attributed, never paraphrased.)
- [Title](link) — one-line attribution.
```

### Drafting rules

1. **Stance first, evidence second.** Thought leadership reads
   flat when it is organized like a blog post. Open with the
   stance. Use first-party stories and pattern-matching as
   evidence. The reader should know by paragraph two what the
   client thinks.
2. **Apply the style guide literally.** Re-read the Tone Definition
   contrast pairs after drafting. Every sentence should sit on
   the X side of each "X, not Y" pair.
3. **Banned language stays banned.** Scan for every banned phrase
   in the style guide before writing the file.
4. **First-party voice only.** Paraphrases, stories, and specific
   claims must trace to a first-party KB source listed in
   `kb_sources_used`. A paragraph with no first-party backing
   gets cut or marked `[CLIENT TO CONFIRM]`.
5. **Third-party is a foil, not a voice.** If the piece quotes or
   pushes back on a third-party view, attribute it clearly.
   Never paraphrase third-party analysis into the client's voice.
6. **No invented facts.** No customers, numbers, quotes, or dates
   the client has not made public. Use `[CLIENT TO CONFIRM]` for
   gaps.
7. **Length honesty.** Target 600 to 1200 words. If the argument
   needs more, it is probably a researched blog instead — flag
   that and ask the user whether to switch lanes.
8. **Don't soften the stance.** If the argument is spicy, ship it
   spicy (inside the style guide's limits). The whole point of
   the lane is the client sounding like themselves with a point
   to make.

### Final pass — stop-slop (mandatory)

Before writing the file, run the shared stop-slop skill at
`rockstarr-infra/skills/_shared/stop-slop/SKILL.md` on the full
draft body (hook, stance line, each H2 body, closing, CTA copy).
Order is fixed: the style guide shapes the draft first; stop-slop
then strips the AI tells that slip through even on-voice writing.
Running them the other way round would scrub voice-specific turns
of phrase the style guide wants in.

Thought-leadership pieces are especially prone to "LinkedIn voice"
drift: throat-clearing openers, binary contrasts, em-dash-heavy
rhythm. stop-slop is the gate that catches those before the file
lands in `03_drafts/`.

What stop-slop catches on this lane:

- "Here's the thing" / "What's interesting is" / "The truth is"
  openers.
- "Not X, it's Y" framings where the stance deserves a direct
  statement.
- Em dashes — swap for periods, commas, or a rewrite.
- Vague declaratives standing in for a concrete claim.
- Quotable-sounding sentences that read as a machine-written pull
  quote.

After the pass, record a `stop_slop_score` in the front-matter
and surface it in the chat summary. Scores below 35/50 get
flagged for the reviewer.

### After writing

1. Print a summary in chat: title, stance in one line, word
   count, CTA destination, any `[CLIENT TO CONFIRM]` markers,
   and the stop-slop score.
2. End with:

   > Thought-leadership draft landed at
   > `03_drafts/content/[slug].md`. Review, edit in place, ask
   > for a revised pass, or run `rockstarr-infra:approve` when
   > it is ready.

3. Do not call `approve` yourself. Human review is the gate.

### Pairing with LinkedIn newsletters

Thought-leadership pieces are the source material for LinkedIn
newsletter sends via `publish-linkedin-newsletter` (DEFER in v0.2)
when `stack.md.linkedin_newsletters_per_month >= 1`. No extra work
in this skill — the downstream skill reads the approved TL piece
from `04_approved/content/` and republishes it.

## What NOT to do

- Do not add an outline step. Thought leadership is single-shot.
  If you are drafting one and feel the urge to outline, stop and
  ask whether this belongs in the researched blog lane instead.
- Do not over-research. Thought leadership works from conviction
  and pattern, not a link pile.
- Do not paraphrase third-party content as the client's take.
- Do not invent data, customers, or quotes.
- Do not move the file to `04_approved/`. That is `approve`'s
  job.
- Do not bulk-draft multiple TL pieces in one run. One per call.
- Do not skip the stop-slop pass. TL that reads as AI-written
  undercuts the stance — stop-slop is the difference between
  "the client said this" and "a model said this".
