---
name: draft-newsletter
description: "This skill should be used when the user asks to \"draft a newsletter\", \"write this week's newsletter\", \"draft the email newsletter\", or names a topic they want turned into a newsletter. Produces a single-shot email newsletter draft in 03_drafts/content/ with subject line options, a preheader, a personal intro, one to three short sections, and a CTA that links to the month's approved blog and thought-leadership pieces on the client's website. Newsletters are shorter and more conversational than blog posts and ship without an outline-first gate."
---

# draft-newsletter

Draft an email newsletter in the client's approved voice. Newsletters
are the most personal channel Rockstarr drafts for — the voice should
sound like the client on their best day talking to one reader, not a
corporate update.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User asks for this week's or this month's newsletter.
- User picks an email-newsletter-format topic from
  `02_inputs/content-topics_YYYY-MM.md`.
- User hands over a free-text topic and says "newsletter".
- A calendar-scheduled newsletter slot fires.

Newsletters are single-shot. No outline step. If the user wants a
longer, more structured piece, steer them to the researched blog
lane (`outline-blog` + `draft-blog`) or the opinion lane
(`draft-thought-leadership`).

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- Either a topic (title + angle) from the user, OR a pointer to a
  line item in `02_inputs/content-topics_YYYY-MM.md`.
- `/rockstarr-ai/00_intake/stack.md` — so the CTA points to a real
  destination (the client's website base URL for linking the
  month's blog + TL pieces, plus the client's booking link, form,
  or reply-to address as the conversion CTA).

If the style guide is missing or unapproved, refuse and point the
user at `rockstarr-infra:generate-style-guide`.

## Inputs

Read in this order:

1. Style guide — full. The **Channel Adaptation — email / newsletter**
   section drives format. The Tone Definition contrast pairs drive
   every sentence.
2. Client profile — for the "from" voice (who is this newsletter
   from — the founder, the team, a named sender?), the audience,
   and the offer the CTA will point to.
3. Stack — to confirm where the CTA leads (website base URL,
   calendar link, offer page, reply-to address).
4. `/rockstarr-ai/04_approved/content/` — scan for the current
   month's approved researched-blog and thought-leadership pieces.
   Their titles + website URLs become the "what I published this
   month" links in the newsletter body.
5. First-party KB files (`kb_scope: owned`) relevant to the topic.
6. Third-party KB files (`kb_scope: third_party`) — allowed as
   reference / link-worthy external material, never as voice.
7. Recent newsletter entries in `05_published/_publish.log` — to
   avoid repeating last week's subject line or intro framing.

## Output

Write to
`/rockstarr-ai/03_drafts/content/YYYY-MM-DD_newsletter_[slug].md`.
If the file exists, append `-2`, `-3`.

Required front-matter:

```yaml
# ---
channel: "newsletter"
title: "Working headline for the issue"
slug: "kebab-cased-slug"
send_date_target: "YYYY-MM-DD"
subject_line_options:
  - "Option A — the primary pick"
  - "Option B — alternate"
  - "Option C — contrarian alternate"
preheader: "30 to 90 chars, completes or complicates the subject line"
from_sender: "Name or alias from client-profile.md"
audience: "One-line audience — usually list segment"
produced_by: "rockstarr-content/draft-newsletter@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
word_count: 742
kb_sources_used:
  - "processed/founder-call-2026-03.md"
third_party_references: []
monthly_pieces_linked:
  - title: "How we think about positioning"
    slug: "how-we-think-about-positioning"
    lane: "blog"
    url: "https://client.example.com/blog/how-we-think-about-positioning"
  - title: "The pipeline myth most founders fall for"
    slug: "pipeline-myth-most-founders-fall-for"
    lane: "thought-leadership"
    url: "https://client.example.com/blog/pipeline-myth-most-founders-fall-for"
cta_text: "exact primary CTA line in the body"
cta_destination: "URL or action reference from stack.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# [Working headline — used internally, not necessarily sent]

## Subject line options
1. [Option A]
2. [Option B]
3. [Option C — contrarian]

## Preheader
[30 to 90 chars]

## Body

[Opening line — personal, specific, not "Hope you had a great week".
Can reference timing, a recent conversation, a piece of news, or
a direct question.]

[Intro paragraph — 2 to 4 sentences. Sets up the beat of the
newsletter.]

### [Optional section header — only if the newsletter has multiple
beats; single-beat newsletters skip this and stay as flowing prose]

[Section body — 2 to 5 short paragraphs. Short sentences. Reader
should be able to skim and still get the point.]

### [Second section header, optional]

[...]

[Close — one short paragraph that leaves the reader with one idea
to carry into their week, in the client's voice.]

## What I published this month

(Auto-populated from `04_approved/content/`. Include ONLY the
current month's approved blog + thought-leadership pieces. Format
matches the client's newsletter style — can be a short list, a
mini-section, or a PS line. Use the exact titles from the
approved pieces and the website URLs reconstructed from
`stack.md.website_base_url + /blog/<slug>` (or the client's
actual pattern). Skip this section entirely if no approved pieces
exist yet for the month.)

- [Title of blog piece](url)
- [Title of TL piece](url)

## CTA
[The primary CTA line — a single specific next action. Format it
the way it will appear in the email (button-style line, link, or
reply prompt). This is the conversion CTA, separate from the
"what I published this month" links above.]

Destination: [URL or action reference from stack.md]

## Sign-off
[The client's sign-off. Do not invent warmth that is not in the
style guide.]
```

### Drafting rules

1. **Personal, not corporate.** Newsletters fail when they read
   like press releases. Use first person ("I", "we") per the
   style guide. If the guide prefers a team voice, use that.
2. **One idea per issue.** If the topic has multiple beats, the
   newsletter can have 2 to 3 short sections, but they all
   reinforce one idea. If you need 4+ beats, flag that the topic
   is too big for a newsletter and belongs in a researched blog
   or thought-leadership piece.
3. **Short sentences, short paragraphs.** Respect the style
   guide's Style Rules on length. Newsletters are skimmed on
   phones.
4. **Subject line options matter.** Give 3 subject lines: primary,
   alternate, contrarian. No clickbait unless the style guide
   explicitly permits. No ALL CAPS. No emojis unless the guide
   allows them.
5. **CTA must be real.** Pull the destination from `stack.md` and
   the client profile. Do not invent URLs. Do not soften the CTA
   into "let me know your thoughts" unless that is a real goal.
   The primary CTA is separate from the "what I published this
   month" links — those are content teasers, not the conversion
   ask.

8. **Monthly-pieces links must be real.** Only list blog and
   thought-leadership pieces from `04_approved/content/` that were
   approved in the current calendar month. Skip the section if
   none exist yet. Reconstruct URLs from
   `stack.md.website_base_url` (or the pattern the client
   actually uses — ask if missing). Do not link to drafts or
   invent URLs.
6. **First-party voice only.** Same as every drafting skill in
   this plugin. Third-party material may appear as an attributed
   link, never as paraphrased voice.
7. **No invented facts.** No made-up stats, customer stories, or
   dates. Use `[CLIENT TO CONFIRM]` for gaps.

### After writing

1. Print a summary: subject line picks, word count, CTA
   destination, any `[CLIENT TO CONFIRM]` markers.
2. End with:

   > Newsletter draft landed at
   > `03_drafts/content/YYYY-MM-DD_newsletter_[slug].md`. Review,
   > pick a subject line, edit, then run
   > `rockstarr-infra:approve` when it is ready.

3. Do not call `approve` yourself.

## What NOT to do

- Do not draft a newsletter longer than ~1000 words. If the idea
  demands it, route to the researched blog or thought-leadership
  lane instead.
- Do not invent the "from" sender. Pull from `client-profile.md`.
- Do not invent the CTA destination. Pull from `stack.md`.
- Do not use generic openers ("Hope you had a great week", "In
  today's fast-paced world", "Quick update for you").
- Do not paraphrase third-party content as the client's take.
- Do not bulk-draft multiple weeks of newsletters in one run. One
  issue per call.
