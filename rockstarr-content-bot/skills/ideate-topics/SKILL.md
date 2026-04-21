---
name: ideate-topics
description: "This skill should be used when the user asks to \"ideate content topics\", \"brainstorm blog topics\", \"suggest content angles\", \"give me topic ideas\", or \"what should we write about this month\". It reads the client's profile, approved style guide, first-party knowledge base, and recent publish log, then proposes 8 to 12 ranked topic angles with working titles, pillars, audience, evidence, and a suggested format (blog, newsletter, or article). Output is written to 02_inputs/content-topics_YYYY-MM-DD.md for the user to pick from before drafting."
---

# ideate-topics

Propose a batch of content angles grounded in the client's actual
profile, voice, and first-party material — not generic thought-
leadership prompts. The user picks which angles to draft from the
list this skill produces.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- Kicking off a new month's content plan.
- User asks for blog / newsletter / article topic ideas.
- Publish log is getting thin and you need to refill the pipeline.
- User has supplied new first-party KB material and wants to turn it
  into content angles.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved
  (front-matter should not flag LOW CONFIDENCE on positioning).
- `/rockstarr-ai/01_knowledge_base/index.md` exists with at least
  one first-party processed file.

If the style guide is missing or unapproved, stop and tell the user
to run `rockstarr-infra:generate-style-guide` first. Topic ideation
without a locked voice produces generic drift.

## Inputs

Read in this order:

1. `/rockstarr-ai/00_intake/client-profile.md` — positioning,
   audience, offers.
2. `/rockstarr-ai/00_intake/style-guide.md` — voice, tone, pillars,
   banned language.
3. `/rockstarr-ai/01_knowledge_base/index.md` — summary of every KB
   file with `kb_scope` tags.
4. Every first-party processed KB file (`kb_scope: owned`) — pull
   concrete claims, frameworks, numbers, customer stories, and
   phrases the client has already used. These become evidence
   pointers for topics.
5. Third-party processed KB files (`kb_scope: third_party`) —
   **reference only**. They can inform what competitors / the
   category are saying, but never act as voice signal and never get
   paraphrased into a topic angle.
6. `/rockstarr-ai/05_published/_publish.log` — the last 90 days of
   shipped pieces. Use this to avoid repeating recent topics within
   90 days, and to spot pillars that are under-served.

## Pillar selection

Derive 3 to 5 content pillars from the profile and style guide
before listing topics. Pillars are the buckets topics fit into — for
a B2B founder-led coach, pillars might be "Positioning",
"Pipeline systems", "Founder-led selling", "The Growth Amplifier in
practice". Every topic you propose must sit under one pillar. If
you can only justify 2 pillars from the inputs, say so — do not
invent pillars to hit a number.

## Output

Write to
`/rockstarr-ai/02_inputs/content-topics_YYYY-MM-DD.md` (ISO date in
the filename; if a file for today already exists, append `-2`).

File structure:

```markdown
# ---
client_id: [from client.toml]
generated_at: [ISO timestamp]
generate_skill_version: "0.1.0"
pillars: ["Pillar A", "Pillar B", ...]
topic_count: [int]
recent_publish_window_days: 90
kb_sources_scanned: [int]
# ---

# Content topics — [Client name] — [Month Year]

## Pillars

- **Pillar A** — one-line description
- **Pillar B** — one-line description
- ...

## Topic list

### 1. [Working title]

- **Pillar:** Pillar A
- **Format:** blog | newsletter | article
- **Audience:** [specific audience from the profile]
- **Angle:** [one sentence — the contrarian / specific POV, not a summary]
- **Evidence (first-party):** [KB file or client-profile.md section]
- **References (third-party, optional):** [KB file + link]
- **Why now:** [one line — topical, seasonal, gap in publish log, customer question]
- **Rough length:** [500-word blog | 1500-word blog | 800-word newsletter | 2500-word article]

### 2. ...
```

Aim for 8 to 12 topics. Rank them by strength of first-party
evidence, not by how "viral" they feel. Angles grounded in a real
claim the client has made are more useful than clever frames with
no evidence.

## Format suggestions — how to decide

- **Blog (500–1500 words):** Single-focus angle, SEO-friendly,
  answers a question or makes one argument. Most topics default to
  blog.
- **Newsletter (600–900 words):** Multi-beat, conversational, often
  stitches 2 or 3 related short thoughts. Good for topical
  commentary, quick takes, or shorter observations that would feel
  thin as a blog post.
- **Article (2000–3000 words):** Research-heavy, framework-
  introducing, or category-defining. Reserve for anchor pieces —
  one per month at most.

If a topic could work as any of the three, default to blog and note
the alternative in a one-line comment under the topic.

## Interview step

After writing the file:

1. Summarize the top 3 topics in chat (title, pillar, format).
2. Ask the user, via `AskUserQuestion`, which ones they want to
   start drafting, or whether they want to reorder / kill any. Use
   at most 4 options per question; if there are more than 4 topics
   worth presenting, ask across multiple rounds.
3. Do not proceed to drafting inside this skill. Hand off to
   `outline-blog` (for blog topics), `draft-newsletter`, or
   `draft-article` as the user directs.

## What NOT to do

- Do not use third-party KB content as a topic angle or an evidence
  pointer. Third-party material may only appear in the optional
  references line and must remain clearly attributed.
- Do not propose topics that contradict the style guide's Do Not
  list or the banned-language list.
- Do not propose topics that repeat a headline from
  `_publish.log` in the last 90 days, unless the user explicitly
  asks for a follow-up or refresh.
- Do not invent customer stories, numbers, or case studies. If a
  proposed topic needs a stat or story the client has not already
  made public, flag it as a blocker in the "Evidence" line (e.g.,
  "Needs: a named customer story — ask client before drafting").
- Do not draft the actual content here. Ideation is the whole job.
