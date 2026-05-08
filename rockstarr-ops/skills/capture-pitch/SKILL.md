---
name: capture-pitch
description: "This skill should be used at install time after capture-stack runs and the operator answers the ops-related questions, or any time the user says \"capture the pitch\", \"update the pitch\", \"refresh pitch.md\", \"the offer changed\", \"the pricing changed\", \"the positioning changed\". Interviews the operator on the current offer, current pricing, current positioning, who-it-is-for and who-it-is-not lists, and any landing-page phrases that must NOT appear in drafts. Writes the result to /rockstarr-ai/00_intake/pitch.md. This file is the single source of truth for current pricing in this Growth OS install — old call transcripts may quote stale pricing, and every prep doc, audit synthesis, recap-note draft, and reengagement message in rockstarr-ops trusts pitch.md over the transcript on conflict. Re-runnable any time the offer or pricing changes."
---

# capture-pitch

Intake interview that produces `00_intake/pitch.md` — the per-client
single source of truth for current offer, pricing, and positioning.
Re-runnable any time the offer changes.

This file is the answer to the most common embarrassing failure
mode in operator-driven sales ops: a reengagement message that
quotes a price the operator no longer offers because the call
transcript it was drafted from is six months old. Every prep doc,
audit synthesis, recap-note draft, and customer-facing message
this bot proposes consults this file BEFORE referencing the offer
or the price. On conflict between this file and a transcript,
**this file wins, every time**.

## When to run

- Install-time, immediately after `rockstarr-infra:capture-stack`
  finishes and the operator answers the ops-related questions
  (`ops_calendar`, `ops_meeting_recorder`, `ops_task_system`,
  `ops_email_outreach_tool`, `ops_email_platform`,
  `ops_deliverability_tool`, etc.).
- On demand, any time the operator says the offer or pricing
  changed. Re-running OVERWRITES the file.
- The `daily-call-prep` orchestrator and every prep / audit /
  reengagement / recap skill REFUSES to run when this file is
  missing.

## Preconditions

- `scaffold-client` has run (so `00_intake/` exists).
- `capture-stack` has run (so `stack.md` exists). This is a soft
  precondition — `capture-pitch` does not read stack.md directly,
  but the install-time chaining order is part of the contract.

## Behavior

Walk the interview one question at a time via `AskUserQuestion`.
The interview is short on purpose — eight questions, none deep.
Length and detail come from the operator's free-text answers, not
from the question count.

When `00_intake/pitch.md` already exists, read it first and
pre-fill the proposed answer for each question. Mark each pre-fill
HIGH / MEDIUM / LOW confidence. The operator confirms, amends, or
rejects per question.

### Q1 — Current offer

Free text. "In one sentence, what is the offer right now?"

Pre-fill from `00_intake/client-profile.md` positioning paragraph
when no `pitch.md` exists yet. Tag confidence MEDIUM.

### Q2 — Current pricing

Free text. "What does it cost? Include any setup fees,
recurring fees, tier names, payment cadence — whatever the
operator quotes a prospect on a sales call. Be specific.
'$5,000 to build, $500 a month to run' is ideal; '$X-Y range'
when there's a real range; 'tiered' with the tier names + prices
when the offer is tiered."

This is the field that most often goes stale. Be persistent
about getting the EXACT phrasing the operator uses on calls,
not a summary.

### Q3 — Positioning paragraph

Free text. "If you had to position the offer in 2-3 sentences
to a stranger who doesn't know what you do, what would you say?"

Pre-fill from `00_intake/client-profile.md` positioning when no
`pitch.md` exists. Tag confidence MEDIUM.

### Q4 — Who is this FOR

Free text or short list. "Who is this offer FOR? The right
prospect, in your words. Job title, company size, signal,
trigger — whatever you actually use to qualify."

Pre-fill from `00_intake/icp-qualifications.md` when it exists.
Tag confidence HIGH if pulled from icp-qualifications, MEDIUM
otherwise.

### Q5 — Who is this NOT for

Free text or short list. "Who is this offer NOT for? The wrong
prospect — disqualifiers, deal-breakers, polite-no patterns."

Pre-fill from `icp-qualifications.md`'s not-target rules when
they exist.

### Q6 — Banned landing-page phrases

Multi-select `AskUserQuestion`, plus free-text additions:

"Some phrases live on your landing page but feel weird in a
1:1 message. Which of these (if any) should NEVER appear in a
draft?"

- `Industry buzzwords from the landing page hero` (free text:
  list them)
- `Marketing-speak the founder doesn't actually say`
- `Awards / press logos`
- `Testimonial quotes verbatim`
- `Other (free text)`

The intent here is to keep ALSO-SAID-BY-MARKETING phrases out of
the bot's drafts. The landing page is a trust artifact; a 1:1
message is a peer artifact.

### Q7 — Common pitch-fit objections

Free text. "What's the most common objection you hear after
you've pitched? Two or three is plenty."

This shows up in `prep-call-2`'s objection handlers and in
`audit-lead` Play 2 (reframe).

### Q8 — Last-confirmed date

Date. "When did you last actually quote this exact offer to a
real prospect?"

Default: today. The skill stamps this into front-matter so
operators can spot stale `pitch.md` files at a glance.

## Write the file

Write `/rockstarr-ai/00_intake/pitch.md`. Overwrite if it exists
— the whole point of re-running is to refresh.

> **Template convention.** The fenced code block below shows
> `# ---` where YAML front-matter delimiters belong, only to keep
> Cowork's SKILL.md parser from misreading them as front-matter
> separators. **When writing the actual file, emit real `---`,
> not `# ---`.**

```markdown
# ---
captured_at: <ISO timestamp>
last_confirmed_at: <Q8 date>
schema_version: 1
# ---

# Pitch — current offer / pricing / positioning

> Single source of truth for what the offer is RIGHT NOW. Old
> recordings may quote stale pricing; every Rockstarr bot trusts
> this file over the transcript on conflict.

## Offer

<Q1 verbatim>

## Pricing

<Q2 verbatim, exact quoted phrasing the operator uses>

## Positioning

<Q3 verbatim>

## Who this is for

<Q4 verbatim, as a list if the operator gave a list>

## Who this is NOT for

<Q5 verbatim, as a list if the operator gave a list>

## Phrases that must NOT appear in drafts

<Q6 selections + free-text additions, as a bullet list>

## Common objections after the pitch

<Q7 verbatim>
```

## Outputs

- `/rockstarr-ai/00_intake/pitch.md` — overwritten on every run.
  Top-of-file `captured_at` and `last_confirmed_at` updated.

## What this skill does NOT do

- Does NOT touch `client-profile.md`, `icp-qualifications.md`,
  `style-guide.md`, or `stack.md`. Those are owned by their
  respective intake skills.
- Does NOT auto-pull pricing from any source. The whole point is
  the operator confirms the current number out loud. Auto-pulling
  from a website or invoicing tool would defeat the staleness
  check.
- Does NOT validate the pricing format. Free text is fine; the
  bot consumes the operator's verbatim quote on every reference.

## Failure modes

- **Operator answers Q2 vaguely** ("around $5K-ish", "depends
  on the deal"). Push back once: "What did you actually quote
  the last prospect?" Accept whatever they answer the second
  time. Capture vague pricing as-is — better to surface
  vagueness than to guess.
- **Operator skips Q5** ("everyone is a fit"). Accept it; tag
  the section `# operator declined to disqualify`. Most ICP
  filtering still happens through `icp-qualifications.md`.
- **Operator hasn't actually pitched recently** (Q8 date > 30
  days ago). Surface this in chat: "This offer hasn't been
  quoted in 30+ days — consider running a smoke-test pitch
  before the next sales call." Do not refuse to write the file.

## Re-running

Overwrite is the default. Diff is NOT preserved — the bot's job
is to keep ONE current version of the file, not a history of
versions. Operators who want history use git.
