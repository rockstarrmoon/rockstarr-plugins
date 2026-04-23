---
name: draft-case-study
description: "This skill should be used when the user asks to \"draft a case study\", \"write a case study on [client name]\", \"document the [customer] win\", \"start the case-study interview\", or when a quarterly case-study reminder fires. Runs an interview-driven flow using AskUserQuestion to walk the client through the Rockstarr custom-GPT case-study prompt one question at a time. Writes a running transcript to 02_inputs/content/case-study-interview-[slug].md as it goes. Produces the polished case-study draft to 03_drafts/content/case-study-[slug].md only after every required question has an answer. Does NOT accept pre-written case-study documents — this lane is interview-first by design."
---

# draft-case-study

Interview-driven case-study drafting. The case study is proof, not
promotion — and that quality only comes from interrogating the
client hard enough to get concrete numbers and concrete stories.
This skill runs the Rockstarr custom-GPT case-study interview
one question at a time via `AskUserQuestion`.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- Quarterly reminder fires for a case study.
- User says "we had a big win with [customer], let's document it".
- User asks to start the case-study interview.

Case studies sit outside the monthly content calendar. They do not
slot alongside blogs or TL pieces — they fire on their own cadence
driven by `stack.md.case_studies_per_quarter` (default 1).

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/00_intake/stack.md` exists.
- Shared reference prompt exists at
  `rockstarr-infra/skills/_shared/references/case-study-prompt.md`.
  If missing, refuse and tell the user to update rockstarr-infra.
- The user has named a specific customer win to document (either
  by company name, a slug, or a short description).

## Inputs

Read in this order:

1. The shared case-study prompt —
   `rockstarr-infra/skills/_shared/references/case-study-prompt.md`.
   This is the interview script and the output structure. Do not
   reinvent either.
2. Style guide — full. Tone and Style Rules govern how the
   interview transcript is turned into prose.
3. Client profile — for the consultant's company, positioning,
   audience, and the offer the CTA will point to.
4. Any prior case studies in `05_published/content/` — to avoid
   repeating structure, opening lines, or positioning beats.

## The interview

Walk the client through the questions in the shared prompt one at
a time via `AskUserQuestion`. The required questions (from the
prompt) are, in order:

1. **Consultant's company** — name of the Rockstarr client being
   represented in the case study.
2. **Ideal client audience** — who this case study is aimed at
   reaching. (Pulled from client-profile.md; confirm with the
   user.)
3. **Client company** — the customer whose story is being told.
4. **Anonymity preference** — named customer, partial
   anonymization (industry/size only), or fully anonymous.
5. **Client snapshot** — customer's industry, size, business
   model, and what they do in one sentence.
6. **Trigger** — what prompted outreach or engagement. The
   moment the customer realized they needed help.
7. **Challenges before engagement** — the specific problems they
   were facing. Push back on vague answers. Ask for numbers or
   specific situations.
8. **The consultant's approach** — not deliverables, but the
   thinking, leadership, strategy, systems, and execution the
   consultant brought. What was the mental model?
9. **Concrete results with numbers** — push back hard on vague
   answers ("it was better" is not an answer). Ask for before /
   after numbers, time-to-value, revenue impact, anything
   quantifiable. If the client cannot produce numbers, mark
   `[CLIENT TO CONFIRM]` and flag it prominently.
10. **Before vs. after** — one clear side-by-side comparison.
11. **Ongoing impact** — what is different about the customer's
    business 6-12 months later.
12. **Optional quote** — a direct quote from the customer, if
    available.
13. **Video-testimonial headline** — a short headline the
    customer could say on camera that summarizes the whole
    transformation.

### Interview mechanics

- **One question per `AskUserQuestion` round.** Never batch
  multiple required fields into one prompt. The interview pattern
  is what surfaces quality.
- **After each answer, append it to the running transcript** at
  `/rockstarr-ai/02_inputs/content/case-study-interview-[slug].md`
  with the question, the answer, and a timestamp.
- **Push back on vague answers.** If the user answers "we
  delivered results" for question 9, follow up: "what numbers?
  what percentage change? what timeline?" Use a second
  `AskUserQuestion` round. Vague case studies do not convert.
- **Allow skip / defer only for optional questions.** Questions
  12 and 13 are optional. Questions 1-11 are required — if the
  user wants to skip a required question, the draft does not
  generate until they answer (or explicitly mark the field
  `[CLIENT TO CONFIRM]` with a note).
- **Resume-safe.** If the interview is interrupted, the transcript
  captures progress. On re-invoke, read the transcript and pick up
  at the first unanswered required question. Don't re-ask
  questions the user already answered.

## Transcript file

Write to
`/rockstarr-ai/02_inputs/content/case-study-interview-[slug].md`
as the interview progresses.

Required front-matter:

```yaml
# ---
type: "case-study-interview"
slug: "kebab-cased-slug"
customer: "Customer company (or 'anonymized')"
interview_started_at: "ISO timestamp"
interview_completed_at: "ISO timestamp or null"
interviewer: "Rockstarr client name (the consultant)"
required_questions_answered: 11  # out of 11 required
optional_questions_answered: 2   # out of 2 optional
produced_by: "rockstarr-content/draft-case-study@0.2.0"
# ---
```

Body structure:

```markdown
# Case-study interview — [Customer or slug]

(Live transcript. Do not edit by hand while the interview is in
progress.)

## 1. Consultant's company
**Q:** [question text from the shared prompt]
**A:** [user's answer]
**Asked at:** [ISO timestamp]

## 2. Ideal client audience
**Q:** ...
**A:** ...
**Asked at:** ...

...

## Status
- Required questions answered: X / 11
- Optional questions answered: X / 2
- Outstanding: [list of unanswered required question numbers]
```

## Draft generation — the gate

Do NOT produce the polished case study until:

- All 11 required questions have non-null answers (including
  `[CLIENT TO CONFIRM]` markers the user explicitly chose to
  defer).
- The user has confirmed, via a final `AskUserQuestion`, that the
  interview is complete and they're ready to see the draft.

If those conditions are not met, print the current status from
the transcript and tell the user which question comes next.

## Case-study draft output

Once the gate is cleared, write to
`/rockstarr-ai/03_drafts/content/case-study-[slug].md`. If the
file exists, append `-v2`, `-v3`.

Required front-matter:

```yaml
# ---
channel: "case-study"
title: "Final case-study title"
slug: "kebab-cased-slug"
customer: "Customer company (or 'anonymized')"
customer_anonymity: "named | industry-only | fully-anonymous"
pillar: "Pillar name"
audience: "One-line audience from the interview"
word_count: 1120
produced_by: "rockstarr-content/draft-case-study@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
interview_source: "02_inputs/content/case-study-interview-[slug].md"
kb_sources_used: []
third_party_references: []
cta_text: "exact CTA line in the body"
cta_destination: "URL or action reference from stack.md"
video_testimonial_headline: "short headline from question 13, if provided"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure — follow the shared prompt's mandated structure:

```markdown
# [Transformation headline — the big change, in one line]

**[Descriptive headline — a subhead that frames the piece]**

[One-sentence hook — the single most compelling line of the case
study. Often a before/after in one sentence.]

## Overview

[2 to 4 sentences. Who the customer is, what they do, and the
shape of the transformation.]

## Impact / Key Metrics

[3 to 5 concrete metrics. Before → after. Timeline. Numbers only —
if a field is `[CLIENT TO CONFIRM]`, flag it inline.]

## Challenge

[Narrative, not bullet points. Tell the story of where the
customer was stuck before engagement. Use the client's voice
describing the customer's state.]

## Solution

[The consultant's approach, from question 8. Thinking, leadership,
strategy, systems, execution. Not a deliverables list. The reader
should understand how the consultant thinks, not just what they
did.]

## Results — Before and After

(Two-column side-by-side, or a narrative "before this / after
this" structure — pick based on how the transcript reads.)

### Before
- [specific state, with numbers]

### After
- [specific state, with numbers]

## Conclusion

[Ongoing impact from question 11. What has the customer's
business looked like 6-12 months after the engagement?]

## [Optional: quote block]

> [Direct quote from question 12, if provided. Attributed with
> customer name + title + company, respecting the anonymity
> choice.]

## [Optional: Video testimonial headline]

[Short headline from question 13 — the single line the customer
could say on camera.]

## [CTA heading — from client-profile.md]

[The specific CTA. Calm, not pushy. Case studies convert on
credibility, not urgency.]

Destination: [URL or action reference from stack.md]
```

## After writing

1. Print a summary in chat: customer, anonymity choice, key
   metrics, any `[CLIENT TO CONFIRM]` markers, CTA destination.
2. End with:

   > Case-study draft landed at
   > `03_drafts/content/case-study-[slug].md`. Review, edit in
   > place, and run `rockstarr-infra:approve` when ready. The
   > interview transcript at
   > `02_inputs/content/case-study-interview-[slug].md` stays
   > so you can audit which answer drove which paragraph.

3. Do not call `approve` yourself.

## What NOT to do

- Do not batch interview questions. One per `AskUserQuestion`
  round. The whole point of the lane is the slowness.
- Do not accept vague answers on the required questions without
  pushing back at least once.
- Do not generate the polished draft before the interview is
  complete.
- Do not invent metrics, quotes, or customer details. If the
  customer did not provide a stat, use `[CLIENT TO CONFIRM]` and
  mention it in the chat summary.
- Do not modify the shared prompt's mandated output structure.
  If the structure needs to evolve, update the prompt in
  rockstarr-infra, not here.
- Do not accept a pre-written case-study document and "polish"
  it. This lane is interview-first by design. Pre-written drafts
  belong in the researched blog lane instead.
- Do not move the file to `04_approved/`. That is `approve`'s
  job.
