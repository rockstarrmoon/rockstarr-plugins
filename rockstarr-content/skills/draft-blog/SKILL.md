---
name: draft-blog
description: "This skill should be used when the user asks to \"draft the blog\", \"write the researched blog\", \"draft the informational post\", \"turn this outline into a post\", or names an approved outline to turn into prose. It is the second step of the two-step researched-blog flow and refuses to run without an approved outline. Produces a researched, informational blog post in 03_drafts/content/ in the client's approved voice, with front-matter the approve skill reads. Cites first-party knowledge-base sources; third-party sources appear only in a References section, never as paraphrased voice."
---

# draft-blog

Turn an approved outline into a full, researched, informational
blog post in the client's approved voice. In v0.2 this is the
researched lane — longer, evidence-heavier, built to educate and
rank. Opinion-driven pieces belong in `draft-thought-leadership`
instead.

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
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
  If the style guide is missing or the version in the outline's
  front-matter does not match the current style guide version,
  warn the user and ask whether to proceed or re-outline.
- `/rockstarr-ai/01_knowledge_base/index.md` exists, and there is
  substantive first-party material on the topic.

If the first-party KB is thin on this topic (fewer than 2 owned
files with non-trivial content), stop and tell the user. Researched
blogs fabricated from a weak first-party base read as generic. The
fix is to gather more first-party material (founder calls, past
writing, internal memos) before drafting.

## Inputs

Read, in this exact order:

1. The approved outline — full, including the evidence map and
   open questions.
2. The style guide — full. Pay literal attention to:
   - **Brand Personality traits** and their Do / Do Not behaviors.
   - **Tone Definition** contrast pairs. Every sentence in the
     draft should pass these pairs.
   - **Style Rules** — sentence length, person, contractions,
     formatting preferences, banned phrases.
   - **Channel Adaptation — blog / long-form** — structural
     preferences for the researched lane.
3. Client profile — for offer names, audience phrasing, product
   proper nouns, authority posture.
4. First-party KB files cited in the outline's
   `kb_sources_used`. Read the actual files, not just the index.
   Pull concrete claims, frameworks, and stories.
5. Third-party KB files cited in
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
word_count: 1843
reading_time_minutes: 8
produced_by: "rockstarr-content/draft-blog@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
outline_source: "03_drafts/content/outline_[slug].md"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
  - "processed/offer-positioning.md"
third_party_references:
  - "processed/third-party/hbr-founder-led-growth.md"
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# [Final post title]

[Meta description candidate — 140 to 160 chars, not published in
body]

[Intro — 1 to 3 short paragraphs. Must hook with a specific
scene, claim, stat (first-party or attributed third-party), or
pointed question. No banned phrases. No LinkedIn-speak.]

## [H2 1]
[Body paragraphs for this section. Follow the outline bullets in
spirit, not line-by-line.]

## [H2 2]

### [Optional H3 for internal structure within a long section]

[...]

## [H2 3]

[...]

## [H2 4 — often the framework or the "so what" section]

[...]

## [Closing heading — can be "So what" or a section-specific
heading; not always "Conclusion"]
[Wrap-up, restating the POV in the client's voice. What changes
for the reader on Monday morning if they accept the argument?]

## [CTA heading — often "Next step" or the client's own language]
[The exact CTA from the outline. Do not invent a different CTA.]

## References

(Required if `third_party_references` is non-empty; optional if
empty.)
- [Title](link) — one-line attribution of what this source
  contributes.
- [Title](link) — ...

## About this piece

(Optional but recommended for longer pieces.)
One short paragraph or a byline block — who wrote it, what they
know, and why they can credibly make this argument. Pulled from
`client-profile.md`.
```

### Drafting rules

1. **Apply the style guide literally.** After drafting, re-read
   the Tone Definition contrast pairs. For each "X, not Y" pair,
   verify the draft sits on the X side. If any sentence falls on
   the Y side, rewrite.
2. **Banned language stays banned.** Do a search-and-replace
   pass for every banned phrase in the style guide before writing
   the file.
3. **Frameworks and claims, not hot takes.** Researched blogs are
   where the client builds authority. Name the framework, number
   the steps, give the reader something to cite. If the client
   has a named framework in the KB, use that exact name.
4. **Earn the length.** Target 1200 to 2500 words (up to 3000 for
   anchor pieces). Every H2 must carry weight. Cut anything that
   is not doing work.
5. **First-party stories and data do the heavy lifting.** A
   paragraph that has no first-party source and no line in the
   outline gets cut or marked `[CLIENT TO CONFIRM]` inline.
6. **Cite generously, paraphrase zero.** Third-party material may
   be quoted (short direct quote, with attribution) or linked in
   the References section. Never paraphrase third-party content
   into the client's voice. The footprint of someone else's
   analysis should be visible to the reader.
7. **No invented facts.** No case studies, named customers,
   revenue numbers, or dates the client has not made public.
   Use `[CLIENT TO CONFIRM]` for gaps.
8. **CTA fidelity.** Use the outline's exact CTA. If the offer
   language from `client-profile.md` has changed since the
   outline was written, flag it and ask the user before
   adjusting.
9. **Length honesty.** Hit the outline's `word_count_target`
   within ±20%. If the topic demands more, say so in a short
   summary at the end of the run and let the human decide.

### After writing

1. Print a summary in chat: title, slug, word count, H2 list,
   references count, any inline `[CLIENT TO CONFIRM]` markers,
   and any style-guide pairs the draft consciously leans toward.
2. End with:

   > Researched-blog draft landed at
   > `03_drafts/content/[slug].md`. This is a long piece —
   > expect a real review pass before approval. When ready, run
   > `rockstarr-infra:approve`.

3. Do not call `approve` yourself. Human review is the gate.

## What NOT to do

- Do not draft if the outline is not approved.
- Do not change the angle, pillar, or CTA from what the outline
  specified. Those decisions already passed a human gate.
- Do not write opinion-driven pieces here — route those to
  `draft-thought-leadership`.
- Do not paraphrase third-party content.
- Do not invent data, customers, or quotes.
- Do not strip the References section — researched blogs without
  sources break credibility.
- Do not move the file to `04_approved/`. That is `approve`'s
  job.
- Do not bulk-draft multiple blogs in one run. One per call.
