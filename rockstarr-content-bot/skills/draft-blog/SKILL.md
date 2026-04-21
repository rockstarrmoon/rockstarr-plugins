---
name: draft-blog
description: "This skill should be used when the user asks to \"draft the blog post\", \"write the blog\", \"turn this outline into a post\", or names an approved outline to turn into prose. It is the second step of the two-step blog flow and refuses to run without an approved outline. It produces a full blog post in 03_drafts/content/ in the client's approved voice, with front-matter the approve skill reads, and cites first-party knowledge-base sources. Third-party sources appear only in a References section, never as paraphrased voice."
---

# draft-blog

Turn an approved outline into a full blog post in the client's
approved voice. This skill is voice-fidelity-first — structural
decisions already happened in `outline-blog`.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User asks to draft the blog from a specific outline file.
- User approves an outline in the current session and says
  "go ahead and draft it".

## Preconditions

- Target outline file exists under
  `/rockstarr-ai/03_drafts/content/outline_*.md`.
- Outline front-matter has `approval_status: "approved"`. If not,
  refuse to run and tell the user to approve the outline first
  (via `outline-blog`'s approval step).
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
  If the style guide is missing or the version in the outline's
  front-matter does not match the current style guide version,
  warn the user and ask whether to proceed or re-outline.

## Inputs

Read, in this exact order:

1. The approved outline — full, including the evidence map and
   open questions.
2. The style guide — full. Pay literal attention to:
   - **Brand Personality traits** and their Do / Do Not behaviors.
   - **Tone Definition** contrast pairs ("X, not Y"). Every
     sentence in the draft should pass these pairs.
   - **Style Rules** — sentence length, person, contractions,
     formatting preferences, banned phrases.
   - **Channel Adaptation — blog / long-form** — structural
     preferences specific to blog.
3. Client profile — for offer names, audience phrasing, product
   proper nouns.
4. First-party KB files cited in the outline's
   `kb_sources_used`. Read the actual files, not just the index.
5. Third-party KB files cited in
   `third_party_references` — references only. Never paraphrase
   them. Do not absorb their voice.

## Output

Write to
`/rockstarr-ai/03_drafts/content/[slug].md` where `[slug]` comes
from the outline's `slug` field. If the file exists, append
`-v2`, `-v3`, etc. Never overwrite a previous draft without
archiving it first to `99_archive/`.

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
word_count: 1187           # actual word count of the body
produced_by: "rockstarr-content-bot/draft-blog@0.1.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
outline_source: "03_drafts/content/outline_[slug].md"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
third_party_references:
  - "processed/third-party/hbr-founder-led-growth.md"
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference"
# ---
```

Body structure:

```markdown
# [Final post title]

[Meta description candidate — 140 to 160 chars, not published in body]

[Intro — 1 to 3 short paragraphs. Must hook with a specific
scene, claim, or question. No "In today's fast-paced world"
openers. No banned phrases. No LinkedIn-speak.]

## [H2 1]
[Body paragraphs for this section. Follow the outline bullets
in spirit, not line-by-line.]

## [H2 2]
...

## [H2 3]
...

## [Closing heading — can be "So what" or a section-specific
heading; not always "Conclusion"]
[Wrap-up, restating the POV in the client's voice.]

## [CTA heading — often "Next step" or the client's own language]
[The exact CTA from the outline. Do not invent a different CTA.]

## References
(Only if third_party_references is non-empty.)
- [Title](link) — [one-line attribution]
```

### Drafting rules

1. **Apply the style guide literally.** After drafting, re-read
   the Tone Definition contrast pairs. For each "X, not Y" pair,
   verify the draft sits on the X side. If any sentence falls on
   the Y side, rewrite.
2. **Banned language stays banned.** Do a search-and-replace
   pass for every banned phrase in the style guide before writing
   the file.
3. **First-party voice only.** Paraphrases, stories, and specific
   claims must trace to a first-party KB source listed in
   `kb_sources_used`. If a paragraph has no first-party source
   and no line in the outline, cut it or mark it
   `[CLIENT TO CONFIRM]` inline.
4. **Third-party is a reference, not a voice.** Third-party
   material may be quoted with attribution (block quote + source)
   or summarized in a References section. It may never be
   paraphrased as if the client said it.
5. **No invented facts.** No case studies, named customers,
   revenue numbers, or dates the client has not made public.
   Use `[CLIENT TO CONFIRM]` for gaps.
6. **CTA fidelity.** Use the outline's exact CTA. If the offer
   language from client-profile.md has changed since the outline
   was written, flag it and ask the user before adjusting.
7. **Length honesty.** Hit the outline's `word_count_target`
   within ±15%. If the topic demands more, say so in a short
   summary at the end of the run and let the human decide.

### After writing

1. Print a summary in chat: title, slug, word count, any inline
   `[CLIENT TO CONFIRM]` markers, and any style-guide pairs the
   draft consciously leans toward.
2. End with:

   > Draft landed at `03_drafts/content/[slug].md`. Review and
   > either edit in place, ask me for a revised pass, or run
   > `rockstarr-infra:approve` when it is ready.

3. Do not call `approve` yourself. Human review is the gate.

## What NOT to do

- Do not draft if the outline is not approved.
- Do not change the angle, pillar, or CTA from what the outline
  specified. Those decisions already passed a human gate.
- Do not write in a voice the style guide does not support. If
  the outline's angle cannot be expressed in the approved voice,
  stop and tell the user.
- Do not paraphrase third-party content.
- Do not invent data, customers, or quotes.
- Do not move the file to `04_approved/`. That is `approve`'s
  job.
