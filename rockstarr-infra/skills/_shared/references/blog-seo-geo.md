---
title: "Blog SEO + GEO reference"
purpose: "Canonical SEO and GEO (Generative Engine Optimization) standard for Rockstarr researched-blog content. Defines what makes a blog rank on Google AND get cited by AI search systems (ChatGPT, Perplexity, Google AI Overviews)."
applies_to: "researched blog lane only — informational pieces drafted via outline-blog → draft-blog. NOT applied to thought leadership (different lane, different rubric), newsletters, case studies, or repurposed derivatives."
read_by:
  - "rockstarr-content/outline-blog (research phase, FAQ outline, keyword plan, internal linking plan, meta drafts)"
  - "rockstarr-content/draft-blog (FAQ in body, keyword density, direct-answer pattern, structured definitions, inline sources, quality checklist)"
do_not_fork: true
---

# Blog SEO + GEO reference

Use this when planning or writing a researched blog post for a
Rockstarr client. The goal is content that serves two audiences
at once: human readers searching Google, and AI systems
(ChatGPT, Perplexity, Google AI Overviews) that retrieve and
cite content. There is no real tradeoff — the same techniques
that earn a Google ranking also earn an LLM citation.

## Why GEO matters

Traditional SEO gets a blog in front of people searching Google.
GEO (Generative Engine Optimization) gets the same content cited
when people ask AI assistants the same questions. AI systems
select content to cite based on five signals:

1. **Specificity.** Content with exact numbers, dates, and proper
   nouns gets cited over vague content.
2. **Authority signals.** Named sources, referenced studies, and
   expert attribution.
3. **Structured answers.** Clear definitions, step-by-step
   processes, and FAQ formats.
4. **Freshness.** Current data with dates attached.
5. **Consistency.** Using the same terminology the searcher
   uses.

Every rule below targets at least one of those five signals.

## The GEO patterns

These are the techniques that move a blog from "professional
content" to "content AI systems cite." Apply all of them.

### 1. Cited statistics with named sources

Every major claim has a source. Instead of "commercial real
estate is growing," write "Houston's industrial market absorbed
6.2 million square feet in Q3 2024, according to CBRE." The
specific number plus the named source is what AI systems verify
and cite.

The minimum: at least 5 cited statistics or facts from named
sources per blog. More is fine.

### 2. Structured definitions

When introducing a concept, define it in 1-2 sentences. AI
systems extract these definitions directly into answers.

Example: "A 1031 exchange lets investors defer capital gains
taxes by reinvesting sale proceeds into a similar property
within 180 days."

That sentence will surface in an AI answer to "what is a 1031
exchange." A vaguer definition will not.

### 3. Direct-answer-first

Start FAQ answers and key paragraphs with the direct answer,
then elaborate. AI systems extract the FIRST sentence of an
answer more than any other.

Bad: "There are several factors to consider when buying
commercial property in Houston, including..."

Good: "Buying commercial property in Houston typically takes 60
to 90 days from offer to closing, plus another 30 days of
pre-offer due diligence. Here's how each phase breaks down..."

The good version answers the search intent in the first
sentence. The bad version makes the AI system keep reading
and possibly cite a different page.

### 4. Named sources, not "research shows"

"According to the National Association of Realtors" is a
verifiable signal AI systems can trust. "Research shows" is
not. Always name the organization, report, or expert. Date
the data when possible ("a 2024 NAR survey found...").

### 5. Consistent terminology

Don't alternate between five synonyms for the same concept.
Pick one term and use it. AI systems match queries to content
by terminology consistency — drift dilutes the match.

If the target keyword is "industrial real estate," use
"industrial real estate" throughout. Don't randomly swap to
"warehouse property," "industrial CRE," or "logistics space"
mid-piece. Pick one.

### 6. Specific numbers, dates, proper nouns

Vague content doesn't get cited. Specific content does.

Vague: "Houston has a strong industrial market."

Specific: "Houston ranks as the third-largest industrial real
estate market in the U.S. by total inventory, with 750+ million
square feet as of 2024."

## The FAQ section — required, not optional

Every researched blog has an FAQ section as one of its H2s
(typically labeled "Frequently Asked Questions"). 3-5 H3
questions. Each answer is 2-4 sentences, direct-answer-first.

Why required: the FAQ section is where AI extraction is
densest. It's also where Google's "People Also Ask" boxes pull
from. A blog without an FAQ section forfeits both surfaces.

How to source FAQ questions:

- Check Google's "People Also Ask" box for the target keyword.
- Check competitor blogs ranking for the keyword — what
  questions do they answer?
- Pull from the client's first-party KB — what does the
  founder hear repeatedly from prospects?
- Use plain language the searcher would actually type, not
  marketing-speak rewrites.

Each FAQ answer follows the direct-answer-first rule. The first
sentence answers the question; the next 1-3 sentences elaborate.

## Keyword placement and density

Place the main keyword (specified in the topic / outline) in
these locations:

1. **Title (H1)** — naturally, not stuffed.
2. **First 100 words** of the intro.
3. **1-2 H2 subheadings** — only where it reads naturally.
4. **Final paragraph** of the conclusion.
5. **Naturally throughout body copy.**

Density target: 0.5-1% (so a 1,500-word blog has the keyword or
close variants 8-15 times total). This is a guidance band, not
a hard enforcement — surface the count to the human reviewer if
it falls outside the band, but do not refuse to write or
sacrifice readability to hit it.

If a sentence reads worse with the keyword forced in, leave it
out of that sentence.

## Internal linking

Place 3-5 internal links inline within the body copy, right
where the linked topic comes up naturally. A link earns its
place when the surrounding sentence already discusses the thing
being linked.

Rules:

- Link to the client's website (the URL from
  `stack.md.website_base_url` and any sub-paths the client has
  shared).
- Use descriptive anchor text that tells the reader (and Google)
  what the linked page is about. Never "click here" or "learn
  more."
- Spread links across different sections. Don't cluster three
  links in one paragraph and leave the rest empty.
- At least one link goes to the client's primary service page
  related to the blog topic.
- When mentioning a service the client offers, hyperlink those
  words to the service page.
- When referencing a related topic the client has covered in a
  previous post, link to that post mid-sentence.
- Never collect internal links into a "Resources" list at the
  bottom. Inline placement is the rule.

If `stack.md.website_base_url` is not set, the skill cannot
internal-link credibly. Surface this to the user as a blocker
during the outline pass — collecting the website URL is part
of `capture-stack`, not part of the blog flow.

## Inline source URL convention

Every external statistic, data point, or factual claim that
came from outside research has its source URL placed inline,
immediately after the claim it supports. Format:

```
...absorbed 6.2 million square feet in Q3 2024 [Source: https://www.cbre.com/insights/figures/houston-industrial-figures-q3-2024].
```

Rules:

- Plain text, NOT a hyperlink. The source URL stays as text in
  the markdown.
- Immediately after the specific claim. Not buried mid-paragraph
  or floated to the end.
- Every external fact gets one — numbers, percentages, dollar
  figures, regulations, named studies, expert quotes.
- If the same source is cited multiple times, the URL appears
  inline each time.
- Internal links (to the client's own site) are hyperlinked
  normally — this convention applies only to external research
  sources.

A References section also appears at the bottom of the body for
the human reviewer's audit. The two coexist: inline `[Source: URL]`
serves AI extraction; the References section serves the human.

## Meta title and meta description

Both are required outputs of the blog flow. They live in the
draft's front-matter.

**Meta title:**
- ≤ 60 characters.
- Keyword near the front.
- Compelling enough to click in search results.
- Specific and actionable, not vague.
- Example: "Steps to Buy Commercial Real Estate in Houston | Guide"

**Meta description:**
- ≤ 155 characters.
- Includes the keyword.
- Summarizes what the reader will learn.
- Soft CTA or value prop at the end.
- Example: "Learn the 8 steps to buying commercial property in Houston, from market research to closing. Expert guide for investors and business owners."

## Quality checklist

Run this checklist BEFORE stop-slop, AFTER style-guide
application. Order is fixed: style guide first, this checklist
second, stop-slop last. Style guide shapes voice, this
checklist enforces SEO/GEO discipline, stop-slop strips AI
tells.

Every item must pass before the file is written. A failed item
returns the draft for a targeted fix; the skill regenerates
with the failed item as the correction prompt.

1. Word count ≥ 1200 (no hard ceiling — thoroughness over
   brevity).
2. Main keyword appears in: title, first 100 words, 1-2 H2s,
   final paragraph of conclusion.
3. Keyword density 0.5-1%. (Warn the reviewer if outside; do
   not refuse.)
4. 3-5 internal links, inline, descriptive anchor text, spread
   across sections.
5. At least one internal link goes to the client's primary
   service page related to the blog topic.
6. FAQ section present with 3-5 H3 questions, each answered
   direct-answer-first.
7. ≥ 5 cited statistics or facts from named sources.
8. Every external fact carries an inline `[Source: URL]` next
   to the claim it supports.
9. Meta title ≤ 60 characters and includes the keyword near
   the front.
10. Meta description ≤ 155 characters and includes the keyword.
11. Active voice with a named subject in every sentence.
12. Tone matches the client's style guide.
13. References section at the bottom lists all external sources
    cited (in addition to inline `[Source: URL]` placements).

If items 6, 7, or 8 fail, the issue is upstream — the outline
didn't capture enough research or didn't plan for the FAQ.
Surface to the user and recommend re-running the outline rather
than burning compute on a regeneration that will hit the same
wall.

## What this reference does NOT do

- Does not address opinion-driven content. Thought leadership
  uses the canonical TL rubric at `tl-rubric.md`. Different
  lane, different rules.
- Does not enforce voice. The client's style guide remains the
  voice authority. This reference layers SEO/GEO discipline on
  top of voice; it does not replace it.
- Does not enforce a time SLA on the writer. Drafting speed is
  a separate concern from output quality.
- Does not paraphrase external content into the client's voice.
  External research provides cited facts and references; the
  voice stays first-party-only. This is the rockstarr-content
  posture, not negotiable.

## Note on the dual roles

In Rockstarr-AI workflows, "the writer" can be the founder
themselves OR the drafting bot.

- When the bot is the author (the common case for this lane):
  the GEO patterns are applied during prose generation; the
  quality checklist runs as an automated pass before stop-slop;
  failures surface to the human with the failed items named so
  they can decide whether to regenerate, re-outline, or
  override.
- When the founder is the author: the same checklist serves as
  a self-review before they hand the draft to approve. Skills
  using this reference can offer to run the checklist over a
  human-authored draft and surface failures the same way.

In both modes, the reference is descriptive, not prescriptive
of the writing process — what matters is the output meets the
checklist before it ships.
