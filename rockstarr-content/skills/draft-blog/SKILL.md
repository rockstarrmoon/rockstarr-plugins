---
name: draft-blog
description: "This skill should be used when the user asks to \"draft the blog\", \"write the researched blog\", \"draft the informational post\", \"turn this outline into a post\", or names an approved outline to turn into prose. It is the second step of the two-step researched-blog flow and refuses to run without an approved outline. As of v0.4 it enforces the canonical blog SEO/GEO reference: a required FAQ section (3-5 H3 questions, direct-answer-first), inline [Source: URL] placement next to every external fact, structured definitions when introducing concepts, keyword density tracking, named sources over generic phrasing, and meta title + meta description in front-matter. Runs the SEO/GEO quality checklist as Pass 1 BEFORE stop-slop (Pass 2). Produces a draft in 03_drafts/content/ in the client's approved voice, citing first-party knowledge-base sources for voice and external sources for stats; third-party voice never paraphrased."
---

# draft-blog

Turn an approved outline into a full, researched, informational
blog post in the client's approved voice. In v0.4 this is the
researched lane — longer, evidence-heavier, built to rank on
Google AND get cited by AI search systems (ChatGPT, Perplexity,
Google AI Overviews). Opinion-driven pieces belong in
`draft-thought-leadership` instead.

**v0.4 change.** The draft now reads the canonical blog SEO/GEO
reference at `rockstarr-infra/skills/_shared/references/blog-seo-geo.md`
and applies it as a Pass 1 gate before stop-slop. The body
structure now requires an FAQ section (3-5 H3 questions). External
facts carry inline `[Source: URL]` placements, alongside the
existing References section at the bottom. Meta title and meta
description live in front-matter. Keyword density is tracked and
surfaced to the reviewer.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User asks to draft a researched, informational blog from a
  specific outline file.
- User approves an outline in the current session and says
  "go ahead and draft it".
- User invokes the back-compat alias `draft-article` (routes here).

## Preconditions

- Target outline file exists under
  `/rockstarr-ai/03_drafts/content/outline_*.md` (produced by
  `outline-blog`).
- Outline front-matter has `approval_status: "approved"`. If not,
  refuse to run and tell the user to approve the outline first
  (via `outline-blog`'s approval step).
- Outline front-matter carries the v0.4 fields:
  `meta_title_draft`, `meta_description_draft`, `faq_questions`
  (≥3), `keyword_placement_plan`, `internal_links_planned`
  (≥3), `external_sources_planned` (≥5). If any of these are
  missing, the outline was produced by an older outline-blog and
  doesn't have the SEO/GEO scaffolding this skill needs. Refuse
  and tell the user to re-run `outline-blog` (the upgrade is
  cheap — same approved angle, fresh research and FAQ).
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
  If the style guide is missing or the version in the outline's
  front-matter does not match the current style guide version,
  warn the user and ask whether to proceed or re-outline.
- The shared blog-seo-geo reference at
  `rockstarr-infra/skills/_shared/references/blog-seo-geo.md`
  exists. Ships in **rockstarr-infra v0.8.2+**. If missing,
  refuse — the SEO/GEO Pass 1 needs it. Tell the user to update
  rockstarr-infra (additive, one new file).
- `/rockstarr-ai/01_knowledge_base/index.md` exists, and there is
  substantive first-party material on the topic.

If the first-party KB is thin on this topic (fewer than 2 owned
files with non-trivial content), stop and tell the user. Researched
blogs fabricated from a weak first-party base read as generic. The
fix is to gather more first-party material (founder calls, past
writing, internal memos) before drafting.

## Inputs

Read, in this exact order:

1. The approved outline — full. Specifically:
   - The angle, audience, why-now (the spine).
   - The H2 list and the FAQ questions (the structure).
   - The keyword placement plan (where the keyword lands).
   - The internal links planned (anchor + target URL + which H2
     each lives in).
   - The external sources planned (every cited fact + URL +
     date).
   - The meta title draft and meta description draft.
   - The evidence map (which first-party KB files back which
     section).
   - The open questions (anything the outline couldn't resolve).
2. The shared blog-seo-geo reference — full. Read § "GEO
   patterns", § "FAQ section", § "Inline source URL convention",
   and § "Quality checklist" before drafting. The Pass 1 gate
   runs against this reference's checklist.
3. The style guide — full. Pay literal attention to:
   - **Brand Personality traits** and their Do / Do Not behaviors.
   - **Tone Definition** contrast pairs. Every sentence in the
     draft should pass these pairs.
   - **Style Rules** — sentence length, person, contractions,
     formatting preferences, banned phrases.
   - **Channel Adaptation — blog / long-form** — structural
     preferences for the researched lane.
4. Client profile — for offer names, audience phrasing, product
   proper nouns, authority posture.
5. First-party KB files cited in the outline's
   `kb_sources_used`. Read the actual files, not just the index.
   Pull concrete claims, frameworks, and stories.
6. Third-party KB files cited in
   `third_party_references`. Researched blogs may lean on more
   third-party material than thought leadership — but the rule
   still applies: references and attributions only, never voice.

## Output

Write to
`/rockstarr-ai/03_drafts/content/[slug].md` where `[slug]` comes
from the outline's `slug` field. If the file exists, append
`-v2`, `-v3`. Never overwrite a previous draft without archiving
it first to `99_archive/`.

Required front-matter:

```yaml
# ---
channel: "blog"
stage: "draft"
title: "Final post title (may differ from working title)"
slug: "kebab-cased-slug"
pillar: "Pillar name"
audience: "One-line audience"
target_keyword: "primary keyword phrase"
meta_title: "≤60 chars, keyword near front"
meta_description: "≤155 chars, includes keyword + value prop"
word_count: 1843
reading_time_minutes: 8
keyword_density: "0.7%"   # actual count / word_count, surfaced even when in band
keyword_count: 13         # raw count of keyword + close variants
faq_question_count: 4     # must be 3-5
external_sources_cited: 7  # raw count of inline [Source: URL] placements; must be ≥5
internal_link_count: 4     # must be 3-5
produced_by: "rockstarr-content/draft-blog@0.4.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
seo_geo_reference_version: "matched from blog-seo-geo.md front-matter"
outline_source: "03_drafts/content/outline_[slug].md"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
  - "processed/offer-positioning.md"
third_party_references:
  - "processed/third-party/hbr-founder-led-growth.md"
external_sources:
  - { url: "https://www.cbre.com/insights/...", date: "2024-10-15", citation_count: 2 }
  - { url: "https://www.nar.realtor/...", date: "2024-09-01", citation_count: 1 }
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
seo_geo_pass_status: "13/13 pass"   # see "Final passes" — Pass 1 of 2
stop_slop_score: 42   # five-dimension score after Pass 2 (min 35 to proceed without a flag)
# ---
```

Body structure (in this order):

```markdown
# [Final post title — keyword carried, ≤70 chars]

[Intro — 1 to 3 short paragraphs. Must hook with a specific
scene, claim, stat (first-party or external with inline
[Source: URL]), or pointed question. The target keyword lands
in this intro within the first 100 words. No banned phrases.
No LinkedIn-speak.]

## [H2 1 — clear, descriptive heading]
[Body paragraphs. Follow the outline bullets in spirit, not
line-by-line. When introducing a concept, define it in 1-2
sentences (structured definition pattern). When citing an
external fact, place the inline `[Source: URL]` immediately
after the claim.]

## [H2 2]

### [Optional H3 for internal structure within a long section]

[...]

## [H2 3 — often the framework or the named-process section]

[Use the client's exact framework name from the KB if one
exists.]

## [H2 4]

[...]

## Frequently Asked Questions

(REQUIRED — 3 to 5 H3 questions. Each answer is 2-4 sentences,
direct-answer-first per the blog-seo-geo reference.)

### [Q1 — question in the user's plain language]
[Direct answer in the first sentence. 2-3 sentences of
elaboration. Inline `[Source: URL]` if the answer cites an
external fact.]

### [Q2]
[...]

### [Q3]
[...]

## [Closing heading — can be "So what now", "Next step", or a
section-specific heading]

[Wrap-up. Restate the practical takeaway in the client's voice.
What changes for the reader on Monday morning if they accept
the argument? Confirm: the target keyword appears in this
final paragraph.]

## [CTA heading — often "Talk to us" or the client's own
language]

[The exact CTA from the outline. Do not invent a different
CTA. Internal link to the client's primary service page if
the CTA copy supports it naturally.]

## References

(Required — every external source cited inline gets an entry
here for the human reviewer's audit.)

- [Source title or organization name](URL) — one-line
  attribution of what this source contributes.
- [Source title](URL) — ...

This section coexists with the inline `[Source: URL]`
placements throughout the body. Inline serves AI extraction;
this section serves the human reviewer. Both are required.

## About this piece

(Optional but recommended for longer pieces.)
One short paragraph or a byline block — who wrote it, what
they know, and why they can credibly make this argument.
Pulled from `client-profile.md`.
```

### Drafting rules

1. **Outline is the spine.** The angle, audience, H2 list, FAQ
   questions, keyword placement plan, internal links, and
   external sources are non-negotiable. The drafting pass writes
   prose; it does not redesign the structure.
2. **Apply the style guide literally.** After drafting, re-read
   the Tone Definition contrast pairs. For each "X, not Y" pair,
   verify the draft sits on the X side. If any sentence falls on
   the Y side, rewrite.
3. **Banned language stays banned.** Do a search-and-replace
   pass for every banned phrase in the style guide before
   running the SEO/GEO pass.
4. **Frameworks and claims, not hot takes.** Researched blogs
   are where the client builds authority. Name the framework,
   number the steps, give the reader something to cite. If the
   client has a named framework in the KB, use that exact name.
5. **Direct-answer-first** (per blog-seo-geo § "GEO patterns").
   Every FAQ answer and every key paragraph leads with the
   direct answer in the first sentence. Elaboration follows.
6. **Structured definitions** (per blog-seo-geo § "GEO
   patterns"). When introducing a concept, define it in 1-2
   sentences. AI systems extract these directly into answers.
7. **Named sources, not "research shows".** Every external fact
   carries the named source. "According to CBRE" beats "research
   shows" every time.
8. **Consistent terminology.** Pick one term for each concept
   and use it. Don't alternate between five synonyms.
9. **Inline `[Source: URL]` for every external fact** (per
   blog-seo-geo § "Inline source URL convention"). Plain text,
   immediately after the claim. The same source is listed in
   the References section at the bottom; both are required.
10. **Earn the length.** Target 1200 words minimum, no hard
    ceiling. Every H2 carries weight. Cut anything not doing
    work — but do not cut substance to hit a cap.
11. **First-party stories and data do the heavy lifting on
    voice.** Paragraphs that carry voice trace back to the
    first-party KB. External sources carry stats only.
12. **Cite generously, paraphrase zero.** Third-party material
    is quoted (short direct quote with attribution) or linked
    in the References section. Never paraphrase third-party
    analysis into the client's voice. The footprint of someone
    else's analysis stays visible to the reader.
13. **No invented facts.** No case studies, named customers,
    revenue numbers, or dates the client has not made public.
    Use `[CLIENT TO CONFIRM]` for gaps.
14. **CTA fidelity.** Use the outline's exact CTA. If the offer
    language from `client-profile.md` has changed since the
    outline was written, flag it and ask the user before
    adjusting.
15. **Internal linking discipline** (per blog-seo-geo §
    "Internal linking"). 3-5 inline links, descriptive anchor
    text, spread across sections, at least one to the client's
    primary service page. No "click here." No bottom Resources
    list.
16. **Keyword placement** (per blog-seo-geo § "Keyword placement
    and density"). Keyword in title, first 100 words, 1-2 H2s,
    final paragraph. Density target 0.5-1% — track and surface,
    do not refuse to write if outside.

### Final passes — SEO/GEO checklist, then stop-slop (both mandatory)

Two passes run after drafting, in this fixed order:

1. **Pass 1 — SEO/GEO quality checklist** (this section).
   Argument and structure quality.
2. **Pass 2 — stop-slop** (next section). Prose hygiene.

Order is fixed because they catch different things at different
layers. SEO/GEO catches missing FAQ sections, missing inline
sources, off-target keyword placement, undefined concepts —
structural problems that no amount of prose polishing will fix.
Stop-slop catches AI tells in the prose itself. Running stop-slop
first would polish prose that's structurally broken.

#### Pass 1 — SEO/GEO checklist

Read the canonical reference at
`rockstarr-infra/skills/_shared/references/blog-seo-geo.md` §
"Quality checklist" and run all 13 tests against the draft.
Each test yields one of: `pass`, `needs-work`, `fail`. Capture
one-line evidence per result.

The 13 tests (verbatim from the reference):

1. Word count ≥ 1200.
2. Main keyword in title, first 100 words, 1-2 H2s, final
   paragraph.
3. Keyword density 0.5-1% (warn the reviewer if outside; do not
   refuse).
4. 3-5 internal links, inline, descriptive anchor text, spread
   across sections.
5. At least one internal link goes to the client's primary
   service page related to the blog topic.
6. FAQ section present with 3-5 H3 questions, each answered
   direct-answer-first.
7. ≥ 5 cited statistics or facts from named sources.
8. Every external fact carries an inline `[Source: URL]` next
   to the claim.
9. Meta title ≤ 60 chars and includes the keyword near the
   front.
10. Meta description ≤ 155 chars and includes the keyword.
11. Active voice with named subject in every sentence.
12. Tone matches the style guide.
13. References section at the bottom lists all external sources
    cited.

Compute `seo_geo_pass_status` as `<passes>/13` (e.g.,
`13/13 pass`, `11/13`).

**Decision matrix:**

- 13 passes → proceed to Pass 2 (stop-slop).
- 11 to 12 passes (1 to 2 `needs-work`, 0 `fail`) → proceed to
  Pass 2, but surface the issues in the chat summary so the
  reviewer sees them.
- Any `fail` → do NOT write the file. Surface the failures
  with the failed test name and a one-line "what's missing"
  per failure. The user decides:
    a. Re-run drafting with the failures as correction prompts
       (skill regenerates once with targeted fixes).
    b. Re-open the outline if the failure is upstream — items
       6, 7, 8 (FAQ, sources, inline citations) usually mean
       the outline didn't capture enough research.
    c. Override and write anyway (escape hatch — logged in the
       front-matter as `seo_geo_pass_status: "X/13 (override)"`
       so the human reviewer sees the intentional gap).

The skill performs at most ONE regeneration attempt. If the
second draft still fails, surface to the human and stop. Don't
burn compute against a structural problem.

#### Pass 2 — stop-slop

Only runs if Pass 1 cleared (11+ passes, no `fail`, or the user
explicitly overrode). Run the shared stop-slop skill at
`rockstarr-infra/skills/_shared/stop-slop/SKILL.md` on the full
draft body (every prose section — title, intro, H2 sections,
FAQ answers, closing, CTA copy, About block).

Style guide shapes voice first, SEO/GEO checklist enforces
structure second, stop-slop strips AI tells last. Running
stop-slop earlier would scrub voice-specific turns of phrase
the style guide wants in.

What stop-slop catches on this lane:

- Filler openers ("Here's the thing", "What's interesting is").
- "Not X, it's Y" binary contrasts and negative listings.
- Passive voice and inanimate-subject phrasing ("the decision
  emerges", "the framework delivers").
- Em dashes — replace with periods, commas, or a restructured
  sentence.
- Three-in-a-row same-length sentences (metronome rhythm).
- Vague declaratives ("The implications are significant") —
  name the specific implication.
- Meta-joiners ("The rest of this post will show…") — let the
  writing move without narrating itself.

After the pass, record `stop_slop_score` in front-matter and
surface it in the chat summary. Scores below 35/50 get flagged
for the reviewer.

### After writing

1. Print a summary in chat: title, slug, meta title (verbatim),
   meta description (verbatim), word count, keyword count and
   density, H2 list, FAQ question count, internal link count,
   external sources cited count, any inline `[CLIENT TO
   CONFIRM]` markers, the `seo_geo_pass_status` (X/13), and
   the `stop_slop_score`.
2. If Pass 1 produced any `needs-work` results, list them
   under a "Reviewer should notice" sub-block — they didn't
   fail hard but the reviewer should see them.
3. If keyword density is outside the 0.5-1% band, call it out
   explicitly: "Keyword density: 1.4% — above the 1% target
   band. Consider trimming one or two non-essential
   placements before publish."
4. End with:

   > Researched-blog draft landed at
   > `03_drafts/content/[slug].md`. This is a long piece —
   > expect a real review pass before approval. When ready,
   > run `rockstarr-infra:approve`.

5. Do not call `approve` yourself. Human review is the gate.

## What NOT to do

- Do NOT draft if the outline is not approved.
- Do NOT draft if the outline is missing the v0.4 SEO/GEO
  scaffolding (`meta_title_draft`, `meta_description_draft`,
  `faq_questions`, `keyword_placement_plan`,
  `internal_links_planned`, `external_sources_planned`).
  Refuse and route the user to re-run `outline-blog`.
- Do NOT change the angle, pillar, or CTA from what the
  outline specified. Those decisions already passed a human
  gate.
- Do NOT write opinion-driven pieces here — route those to
  `draft-thought-leadership`.
- Do NOT paraphrase third-party content into the client's
  voice. External sources are CITED inline with `[Source: URL]`
  and listed in References. Their phrasing stays visible as
  someone else's analysis.
- Do NOT invent data, customers, or quotes.
- Do NOT strip the References section. Inline `[Source: URL]`
  serves AI extraction; the bottom References section serves
  the human reviewer's audit. Both required.
- Do NOT skip the FAQ section. The blog-seo-geo reference
  treats it as required because it's where AI extraction is
  densest. A researched blog without an FAQ section ships
  broken.
- Do NOT skip the SEO/GEO checklist (Pass 1) because the
  prose looks good. Polishing structural failures into smoother
  prose makes the failure harder to spot.
- Do NOT skip the stop-slop pass (Pass 2). Order is fixed:
  SEO/GEO first, stop-slop last.
- Do NOT regenerate more than once per drafting call. Two
  consecutive failures means the issue is upstream — surface
  to the human and stop. Continuing to regenerate against an
  outline that didn't capture enough research just burns
  compute.
- Do NOT move the file to `04_approved/`. That is `approve`'s
  job.
- Do NOT bulk-draft multiple blogs in one run. One per call.
