---
name: draft-article
description: "This skill should be used when the user asks to \"draft a long-form article\", \"write the anchor piece\", \"draft the thought-leadership article\", or picks an article-format topic. It produces a long-form, research-heavy article in 03_drafts/content/ with dek, lede, body sections, optional pull-quotes, and a conclusion. Articles are longer and more formal than blog posts and can cite more third-party research as references (never as voice). Single-shot in v0.1 (no outline step)."
---

# draft-article

Draft a long-form, research-heavy article in the client's approved
voice. Articles are the anchor format — category-defining,
framework-introducing, or analytical pieces. Reserve for topics that
justify 2000 to 3000 words.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User picks an article-format topic from
  `02_inputs/content-topics_*.md`.
- User asks for an anchor piece, a cornerstone essay, or a long-form
  thought-leadership article.
- The topic clearly cannot fit inside a blog post (needs multiple
  frameworks, extensive evidence, or a narrative arc).

Articles are single-shot in v0.1. No outline gate. If the user wants
tighter structural control, steer them to `outline-blog + draft-blog`
for a shorter piece.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/01_knowledge_base/index.md` exists, and there is
  substantive first-party material on the topic.

If the first-party KB is thin on this topic (fewer than 2 owned
files with non-trivial content), stop and tell the user. Articles
fabricated from a weak first-party base read as generic. The fix
is to gather more first-party material (founder calls, past
writing, internal memos) before drafting.

## Inputs

Read in this order:

1. Style guide — full. The **Channel Adaptation — long-form /
   article** section (or the long-form subsection if there is no
   dedicated article section) drives tone. Articles still must pass
   the Tone Definition contrast pairs.
2. Client profile — positioning, authority posture, audience
   sophistication.
3. Source topic — from `02_inputs/content-topics_*.md` or free
   text. Preserve the angle exactly as stated.
4. First-party KB files on the topic (`kb_scope: owned`,
   `style_guide_eligible: true`). For articles, read the actual
   files, not just the index. Pull concrete claims, frameworks,
   and stories.
5. Third-party KB files on the topic (`kb_scope: third_party`).
   Articles can lean on more third-party material than blogs —
   but the same rule applies: references and attributions only,
   never voice.
6. Recent article entries in `05_published/_publish.log` — to
   avoid collision with a recent anchor piece.

## Output

Write to
`/rockstarr-ai/03_drafts/content/[slug].md` where `[slug]` comes
from the working title (kebab-cased, max 60 chars). If the file
exists, append `-v2`, `-v3`.

Required front-matter:

```yaml
# ---
channel: "article"
title: "Final article title"
slug: "kebab-cased-slug"
dek: "One-line standfirst that sits under the title"
pillar: "Pillar name"
audience: "One-line audience"
target_keyword: "primary keyword phrase"
word_count: 2413
reading_time_minutes: 10
produced_by: "rockstarr-content-bot/draft-article@0.1.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
  - "processed/offer-positioning.md"
third_party_references:
  - "processed/third-party/hbr-founder-led-growth.md"
  - "processed/third-party/gartner-b2b-buying.md"
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference"
# ---
```

Body structure:

```markdown
# [Article title]

*[Dek — one or two sentences that stand under the title and tell
the reader why the piece exists. Avoid "In this article, we will
explore…" framings.]*

[Lede — 2 to 4 paragraphs. Anchor on a scene, a claim, a striking
stat (first-party or attributed third-party), or a pointed
question. The lede earns the reader's next 10 minutes.]

## [H2 1 — usually "The problem" or a framed claim]

[Body. Full paragraphs, not bullet dumps.]

> [Optional pull-quote — a single sharp line lifted from the section
> that a reader skimming on mobile will remember.]

## [H2 2]

[...]

### [Optional H3 for internal structure within a long section]

[...]

## [H2 3]

[...]

## [H2 4 — often the framework or the "so what" section]

[...]

## Conclusion

[A real conclusion, not a restatement. What changes for the reader
if they accept the article's argument? What are they supposed to
do on Monday morning?]

## [CTA heading — often "Next step" or client-specific]

[The specific CTA. Articles can have a softer CTA than blogs or
newsletters, because the intent is authority-building — but it
still points somewhere real.]

## References

(Required if `third_party_references` is non-empty; optional if
empty.)
- [Title](link) — one-line attribution of what this source
  contributes.
- [Title](link) — ...

## About this piece

(Optional but recommended for articles.)
One short paragraph or a byline block — who wrote it, what they
know, and why they can credibly make this argument. Pulled from
`client-profile.md`.
```

### Drafting rules

1. **Earn the length.** Every H2 must carry weight. Articles
   that hit 2500 words because of padding fail — cut anything
   that is not doing work.
2. **Frameworks and claims, not hot takes.** Articles are where
   a client builds authority. Name the framework, number the
   steps, give the reader something to cite. If the client has a
   named framework in the KB, use that exact name.
3. **Cite generously, paraphrase zero.** Third-party material
   may be quoted (short direct quote, with attribution) or
   linked in the References section. Never paraphrase
   third-party content into the client's voice. The footprint
   of someone else's analysis should be visible to the reader.
4. **First-party stories and data do the heavy lifting.** If a
   section has no first-party backing, either replace it with
   something that does, or explicitly mark it `[CLIENT TO
   CONFIRM]`.
5. **Style guide is still canon.** Articles are longer and more
   formal, but the Tone Definition contrast pairs, Style Rules,
   and banned-language list all still apply. Re-read the style
   guide before the final pass.
6. **Lede and conclusion are the highest-leverage sentences.**
   Draft them, then re-draft them. They are what a skimming
   reader will actually read.
7. **No invented facts.** No customers, numbers, quotes, or
   dates the client has not made public. Use `[CLIENT TO
   CONFIRM]` for gaps.

### After writing

1. Print a summary: title, dek, word count, H2 list, references
   count, any `[CLIENT TO CONFIRM]` markers.
2. End with:

   > Article draft landed at
   > `03_drafts/content/[slug].md`. This is a long piece —
   > expect a real review pass before approval. When ready, run
   > `rockstarr-infra:approve`.

3. Do not call `approve` yourself.

## What NOT to do

- Do not draft articles shorter than ~1800 words. If the topic
  cannot sustain that length, write a blog post instead.
- Do not invent frameworks, studies, customer stories, or stats.
- Do not paraphrase third-party research as the client's
  original thinking.
- Do not strip the References section — articles without sources
  break credibility.
- Do not draft multiple articles in one run. One per call.
- Do not treat articles as long blog posts. They are a distinct
  format with different expectations for tone and evidence.
