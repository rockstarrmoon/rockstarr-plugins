---
name: draft-thought-leadership
description: "This skill should be used when the user asks to \"draft a thought-leadership piece\", \"write an opinion post\", \"turn this outline into a TL post\", \"push back on X\", \"make the argument for Y\", or names an approved TL outline to turn into prose. It is the second step of the two-step thought-leadership flow added in v0.3 and refuses to run without an approved outline from outline-thought-leadership. Produces a shorter, hard-hitting, opinion-driven piece in 03_drafts/content/ that argues the thesis from the outline, opens on the named scene, and crystallizes the take in the named quotable line. Runs the canonical TL rubric as a post-draft pass before stop-slop. Weight comes from the stance, not the research."
---

# draft-thought-leadership

Draft an opinion-driven piece in the client's approved voice.
Thought leadership is the lane where the client takes a stance:
pushes back on a prevailing view, names a pattern, reframes a
problem, or says something that deserves disagreement. Weight
comes from the POV, not the evidence pile. Shorter and sharper
than a researched blog.

**v0.3 change.** This skill now requires an approved outline from
`outline-thought-leadership`. The v0.2 single-shot flow was fast
but flat — too many pieces shipped without a real argument and
needed multiple regeneration rounds to fix the underlying
fuzziness. The outline gate forces the writer (founder or bot) to
commit to thesis, counter-argument, opening scene, quotable line,
and proprietary-term burial BEFORE prose runs. If you want a
single-shot opinion piece without the gate, you don't want a
thought-leadership piece — you probably want a social post.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User names an approved TL outline and asks to turn it into a
  draft.
- User asks to "draft the thought-leadership piece on [slug]" and
  the matching outline at
  `03_drafts/content/outline-tl_[slug].md` has
  `approval_status: "approved"`.
- A previous TL draft was rejected on rubric grounds and the
  outline is still good — re-run drafting against the same
  approved outline.

If the user asks to draft a TL piece without an approved outline
existing, refuse and route them to `outline-thought-leadership`
first. State the reason explicitly:

> Thought leadership requires an approved outline before drafting
> — the outline locks in the thesis, the counter-argument, the
> opening scene, and the quotable line. Run
> `outline-thought-leadership` first; come back here when it's
> approved.

If the user asks for something clearly researched and informational
(explain how something works, step-by-step how-to, roundups of
tactics), steer them to the researched blog lane (`outline-blog`
plus `draft-blog`) instead.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- An approved outline at
  `03_drafts/content/outline-tl_[slug].md` with
  `approval_status: "approved"` and the five required fields
  populated (`thesis`, `counter_argument`, `opening_scene`,
  `quotable_line`, `proprietary_term_buried`). Refuse to run
  without all five.
- `/rockstarr-ai/00_intake/stack.md` — so the CTA points to a real
  destination.
- The shared TL rubric at
  `rockstarr-infra/skills/_shared/references/tl-rubric.md` exists.
  This file ships in **rockstarr-infra v0.8.1+**. If missing,
  refuse — the post-draft rubric pass needs it. Tell the user
  to update rockstarr-infra to v0.8.1 or later (the upgrade is
  pure additive — one new file under
  `skills/_shared/references/` — so it's cheap to run).

If the style guide is missing or unapproved, refuse and point the
user at `rockstarr-infra:generate-style-guide`.

## Inputs

Read in this order:

1. The approved outline at
   `03_drafts/content/outline-tl_[slug].md` — full. The five
   required fields are the spine of the draft. **Do not deviate.**
   If the draft drifts from the outline's thesis or counter, the
   outline gate's value is lost.
2. The shared TL rubric at
   `rockstarr-infra/skills/_shared/references/tl-rubric.md` —
   full. Read § "Structural rewrite checklist" and § "Patterns to
   cut on sight" before drafting. Re-read § "Quick critique frame"
   for the post-draft pass.
3. Style guide — full. The **Brand Personality**, **Tone Definition
   contrast pairs**, and **Style Rules** drive voice. The **Channel
   Adaptation — blog / long-form** section (or a dedicated
   thought-leadership subsection if present) drives format.
4. Client profile — positioning, the recurring arguments or
   unpopular truths the client makes, the audience, and the offer
   the CTA will point to.
5. First-party KB files (`kb_scope: owned`) that back the thesis —
   founder calls, internal memos, past writing. This is where the
   credibility comes from. The outline names which sources to use
   in its evidence map; honor that.
6. Third-party KB files (`kb_scope: third_party`) — allowed only
   as a contrast point the client is pushing back on, or as a
   cited reference. Never paraphrased as if the client said it.
7. Recent thought-leadership entries in `05_published/_publish.log`
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
thesis: "One-sentence argument — copied from outline"
counter_argument: "Smart competitor's rebuttal — copied from outline"
quotable_line: "The dinner-table sentence — copied from outline"
proprietary_term_buried: "Term name, or null"
enemy: "What this argues against, or null"
outline_source: "03_drafts/content/outline-tl_[slug].md"
target_keyword: "primary keyword phrase (optional — TL is stance-led, not SEO-led)"
word_count: 812
reading_time_minutes: 4
produced_by: "rockstarr-content/draft-thought-leadership@0.3.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
rubric_version: "matched from tl-rubric.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
third_party_references: []
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference from stack.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
tl_rubric_score: "6/6 pass"   # see "Final pass — TL rubric" below
stop_slop_score: 41           # five-dimension score after stop-slop
# ---
```

Front-matter cross-check before writing: `thesis`,
`counter_argument`, and `quotable_line` MUST match the outline
verbatim. The drafting pass is allowed to choose H2 headings,
phrasing, and supporting evidence — but not to drift the
argument. If the draft's thesis no longer matches the outline,
that's a sign the outline needs to be amended in
`outline-thought-leadership`, not silently rewritten here.

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

1. **Outline is the spine.** The outline's thesis, counter, opening
   scene, and quotable line are non-negotiable. Use them
   verbatim where they're meant to appear verbatim (opening
   scene, quotable line) and as guardrails where they're not
   (thesis stays the thesis; counter shapes which arguments need
   addressing).
2. **Stance first, evidence second.** Thought leadership reads
   flat when it is organized like a blog post. Open on the
   outline's named scene. State the thesis as one sentence in the
   first third (per the rubric's structural checklist). Use
   first-party stories and pattern-matching as evidence.
3. **Place the quotable line high.** Per rubric § "Test 3" and the
   structural checklist: the quotable line goes in the first
   third of the article, not buried at the end. Bold it inline so
   the reviewer can find it at a glance.
4. **Bury the proprietary term.** If the outline named one, it
   does not appear before the second half of the piece. The
   reader cares about the IDEA before they care about what the
   author calls the thing.
5. **One story, told end-to-end.** Cut stat lists. Cut multi-client
   name-drops. Pick the strongest first-party story and walk it
   through — what was the person doing on a Tuesday before, what
   broke, what changed. (Rubric § "Test 2.")
6. **Apply the style guide literally.** Re-read the Tone Definition
   contrast pairs after drafting. Every sentence should sit on
   the X side of each "X, not Y" pair.
7. **Banned language stays banned.** Scan for every banned phrase
   in the style guide AND every pattern in rubric § "Patterns to
   cut on sight" before writing the file.
8. **First-party voice only.** Paraphrases, stories, and specific
   claims must trace to a first-party KB source listed in
   `kb_sources_used`. A paragraph with no first-party backing
   gets cut or marked `[CLIENT TO CONFIRM]`.
9. **Third-party is a foil, not a voice.** If the piece quotes or
   pushes back on a third-party view, attribute it clearly.
   Never paraphrase third-party analysis into the client's voice.
10. **No invented facts.** No customers, numbers, quotes, or dates
    the client has not made public. Use `[CLIENT TO CONFIRM]` for
    gaps.
11. **Length honesty.** Target 600 to 1200 words (per the
    outline's `word_count_target`). If the argument needs more,
    it is probably a researched blog instead — flag that and ask
    the user whether to switch lanes.
12. **Don't soften the stance.** If the argument is spicy, ship it
    spicy (inside the style guide's limits). The whole point of
    the lane is the client sounding like themselves with a point
    to make.
13. **One reveal max.** Per rubric: "Here's the thing nobody tells
    you" is fine once. Twice is a tic. Same for any other
    rhetorical reveal device.

### Final passes — TL rubric, then stop-slop (both mandatory)

Two passes run after drafting and before the file is written.
**Order is fixed.** Argument quality first (does this piece have a
real thesis the reader could disagree with?), prose hygiene
second (does it read like a human wrote it?). Reversing the order
polishes a slogan into smoother prose, which is worse than a
rough piece with a real argument.

#### Pass 1 — TL rubric (argument quality)

Read the canonical rubric at
`rockstarr-infra/skills/_shared/references/tl-rubric.md` § "Quick
critique frame" and run all seven tests against the draft. Each
test yields one of: `pass`, `needs-work`, `fail`. Capture
one-line evidence per result.

The seven tests, in order:

1. **Single argument.** Can you state the article's argument as
   one sentence? Cross-check against the outline's `thesis` —
   they must match. If the draft no longer matches the thesis,
   that's a `fail`.
2. **Smart competitor could disagree.** Could a credible expert
   write a counter-piece? The outline already named the
   counter — verify the draft actually engages with it,
   doesn't dodge it.
3. **One specific story told in detail.** Is there one client
   or scenario walked through end-to-end? List of stats and
   testimonials → `fail`.
4. **Quotable line lands and is placed high.** Verify the
   outline's `quotable_line` is in the first third, set off
   visually (bold), and reads as a sharp line on its own.
5. **Proprietary term burial.** If the outline named a
   `proprietary_term_buried`, verify it does not appear in the
   first half. Search the draft body for the term and check the
   character offset against the half-mark.
6. **No rhetorical-tic overlap with recent series.** Compare
   draft openers and reveal phrasings against the last 3
   thought-leadership entries in `05_published/_publish.log`.
   If the same opener pattern repeats across pieces in the
   same monthly cadence, that's a `needs-work` at minimum.
7. **Closing crystallizes the take.** The closing paragraph
   ends with a take-home line above the CTA — not the CTA
   alone.

Compute `tl_rubric_score` as `<passes>/7` (e.g., `7/7 pass`,
`5/7`).

**Decision matrix:**

- 7 passes → proceed to stop-slop.
- 5 to 6 passes (1 to 2 `needs-work`, 0 `fail`) → proceed to
  stop-slop, but surface the issues in the chat summary so the
  reviewer sees them.
- Any `fail`, OR fewer than 5 passes → do NOT write the file.
  Surface the failures to the user with the rubric's "Handing
  this to a writer" question framing (one short question per
  failed test). User decides whether to:
    a. Re-run drafting with the failures as correction prompts
       (skill regenerates once with the targeted fix).
    b. Re-open the outline (if the failure is at the argument
       level — fuzzy thesis, no real counter — the fix is
       upstream in `outline-thought-leadership`).
    c. Stop. Sometimes the issue is that the founder hasn't
       decided what they think yet, and the right answer is
       a 20-minute conversation, not another regeneration.

The skill performs at most ONE regeneration attempt. If the second
draft still fails, surface to the human and stop. Per the rubric:
"This rubric helps a writer who has something to say but is
buried under template language. It does NOT rescue a writer who
is genuinely fuzzy on what they think."

#### Pass 2 — stop-slop (prose hygiene)

Only runs if Pass 1 cleared (5+ passes, no `fail`). Run the
shared stop-slop skill at
`rockstarr-infra/skills/_shared/stop-slop/SKILL.md` on the full
draft body (opening scene, thesis line, each H2 body, closing,
CTA copy).

Style guide shapes voice first; rubric checks argument; stop-slop
strips AI tells last. Running stop-slop earlier would scrub
voice-specific turns of phrase the style guide wants in.

What stop-slop catches on this lane:

- "Here's the thing" / "What's interesting is" / "The truth is"
  openers.
- "Not X, it's Y" framings where the stance deserves a direct
  statement.
- Em dashes — swap for periods, commas, or a rewrite.
- Vague declaratives standing in for a concrete claim.
- Quotable-sounding sentences that read as a machine-written pull
  quote.

After the pass, record `stop_slop_score` in the front-matter and
surface it in the chat summary. Scores below 35/50 get flagged
for the reviewer.

### After writing

1. Print a summary in chat: title, thesis (one line), counter-
   argument (one line), the quotable line (verbatim), word count,
   CTA destination, any `[CLIENT TO CONFIRM]` markers, the
   `tl_rubric_score`, and the `stop_slop_score`.
2. If the rubric pass produced any `needs-work` results, list them
   under a "Reviewer should notice" sub-block — they didn't fail
   hard but the reviewer should see them before approving.
3. End with:

   > Thought-leadership draft landed at
   > `03_drafts/content/[slug].md`. Review, edit in place, ask
   > for a revised pass, or run `rockstarr-infra:approve` when
   > it is ready.

4. Do not call `approve` yourself. Human review is the gate.

### Pairing with LinkedIn newsletters

Thought-leadership pieces are the source material for LinkedIn
newsletter sends via `publish-linkedin-newsletter` (DEFER in v0.2)
when `stack.md.linkedin_newsletters_per_month >= 1`. No extra work
in this skill — the downstream skill reads the approved TL piece
from `04_approved/content/` and republishes it.

## What NOT to do

- Do NOT run without an approved outline. The v0.2 single-shot
  flow is gone in v0.3 — it produced too many flat pieces. If
  there's no outline, route the user to
  `outline-thought-leadership` and stop.
- Do NOT drift the thesis from the outline silently. If you find
  yourself writing a different argument than the outline's
  thesis, that's a sign the outline needs amending — go back
  there, don't paper over it here.
- Do NOT over-research. Thought leadership works from conviction
  and pattern, not a link pile.
- Do NOT paraphrase third-party content as the client's take.
- Do NOT invent data, customers, or quotes.
- Do NOT move the file to `04_approved/`. That is `approve`'s
  job.
- Do NOT bulk-draft multiple TL pieces in one run. One per call.
- Do NOT skip the rubric pass. The rubric catches argument
  failures that stop-slop can't see — a slogan polished by
  stop-slop is still a slogan.
- Do NOT skip the stop-slop pass. TL that reads as AI-written
  undercuts the stance — stop-slop is the difference between
  "the client said this" and "a model said this".
- Do NOT regenerate more than once per drafting call. Two
  consecutive failures means the issue is upstream — surface to
  the human and stop. Continuing to regenerate against a fuzzy
  thesis just burns compute.
