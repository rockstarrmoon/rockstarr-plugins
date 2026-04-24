---
name: capture-icp-qualifications
description: "This skill should be used at install time after capture-interceptly-personas, or when the user says \"capture ICP qualifications\", \"interview me on who counts as a target\", \"update icp-qualifications.md\", or \"my ICP rules changed\". Interviews the client on their target / not-target / ambiguous rules and writes /00_intake/icp-qualifications.md. This file is the source-of-truth for qualify-lead on every reply — the bot has no baked-in opinion on who is or isn't a target beyond what the client writes here."
---

# capture-icp-qualifications

Shared intake interview. Canonical source lives at
`rockstarr-infra/skills/_shared/capture-icp-qualifications/` once
the shared tree exists; for V0.1 it ships inline in
`rockstarr-outreach-interceptly`. If you edit this file, edit the
matching copy in any sibling plugin that ships its own inline copy
in the same PR.

This is where a client's own ICP judgement gets encoded into a file
`qualify-lead` reads on every single reply. Thin answers here mean
the bot will punt to "ambiguous" more often — which is fine, that's
the design. Empty answers mean the bot refuses to qualify anyone,
which blocks the daily loop, which is also the design.

## When to run

- Install-time, after `capture-interceptly-personas` finishes.
- When the user says "my ICP changed" or "I keep seeing this
  pattern slip through — let's tighten the rules."
- When the weekly report's Non-ICP Log surfaces a pattern the
  client would have drafted for — that's a signal to re-interview.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists — it provides
  the baseline ICP to anchor against. If not, refuse and point at
  `rockstarr-infra:ingest-workbook`.

## Inputs

- `00_intake/client-profile.md` — baseline ICP section (read, do
  not modify).
- `00_intake/icp-qualifications.md` — prior version if it exists
  (offer to reuse or refresh).

## Interview

Walk one question at a time via `AskUserQuestion`. Pre-read
`client-profile.md` first and draft HIGH/MEDIUM/LOW confidence
proposed answers to each question. Present the pre-draft as the
default answer; the client confirms, amends, or rejects.

### Part A — Target rules ("who IS a target")

Q1. "List the titles or role clusters that ARE targets. If a lead's
title or functional role matches one of these, they qualify on role
alone. (Example: 'founder, CEO, CRO, VP Sales, Head of Sales.')"

Q2. "What company-size signals identify a target? (Employee count
bands, revenue bands, funding stage, growth stage — whatever you
use.)"

Q3. "What industries or business models ARE targets? Separately,
which markets are NOT the fit even when the title is right?"

Q4. "What buying-intent or context signals raise confidence that a
lead is worth the draft? (Recent hire, recent funding, just
launched a product, scaling sales team, etc.)"

### Part B — Not-target rules ("who is NOT a target")

Q5. "List the roles or functions that are NEVER a target, even if
they reply warmly. (Example: recruiters, students, other agencies,
services providers pitching back at you.)"

Q6. "What company types are disqualifiers regardless of role?
(Competitors, agencies selling to your audience, pre-revenue
startups, specific industries — whatever the client rules out.)"

Q7. "What lead behaviors are automatic declines? (Pitching back,
asking for introductions without reciprocity, sales-y replies,
anything else that signals a bad-faith conversation.)"

### Part C — Ambiguous rules ("when the bot should flag, not draft")

Q8. "What makes a lead ambiguous enough that you'd want to read
the thread yourself before the bot drafts? (Mid-senior but odd
industry? Right industry but unclear buying role? New company
without a public footprint?)"

Q9. "If the Interceptly right-panel doesn't give enough signal,
what is the minimum extra context that would tip the lead from
ambiguous to target? (A LinkedIn summary? A company website? A
specific job-change recency?)"

### Part D — Edge cases

Q10. "Name 2-3 real past leads who WERE targets but the bot might
not realize it. What signal did you use?"

Q11. "Name 2-3 real past leads who looked like targets but
weren't. What disqualified them?"

These edge cases become the calibration examples in the file.

## Write file

Write `/rockstarr-ai/00_intake/icp-qualifications.md` in full.
Schema:

```markdown
---
generated_at: <ISO timestamp>
schema_version: 1
source: capture-icp-qualifications
confidence_baseline: <HIGH|MEDIUM|LOW based on how much the client
    confirmed vs. amended>
---

# ICP Qualifications

Source-of-truth for `qualify-lead`. The bot has no opinion on who
is or isn't a target beyond what this file says. Per-campaign ICP
specs can tighten these rules but never loosen them.

## Target rules

### By role

- <role or role cluster>

### By company size / stage

- <firmographic rule>

### By industry / business model

- <industry rule>

### By buying-intent signal

- <signal>

## Not-target rules

### By role

- <role>

### By company type

- <disqualifier>

### By behavior

- <behavior>

## Ambiguous rules

- <rule>

## Minimum extra context for ambiguous-to-target promotion

- <extra context the bot may fetch from LinkedIn before promoting
  an ambiguous lead to target>

## Calibration examples

### Were targets (non-obvious)

- <example with the signal the client used>

### Were NOT targets (looked like targets)

- <example with the disqualifier>
```

## Outputs

- `/rockstarr-ai/00_intake/icp-qualifications.md`

## Gate on the daily loop

The daily loop refuses to run if this file does not exist. When
the daily loop is blocked on this reason, point at this skill.

## Failure modes

- **Client gives every question a one-line answer.** Thin input
  produces thin rules, which produces more ambiguous verdicts. This
  is acceptable for V0.1 — surface it in the weekly report so the
  client can refine over time.
- **Client's rules contradict client-profile.md.** Surface the
  conflict and ask which wins. `icp-qualifications.md` is the
  runtime source-of-truth; update `client-profile.md` separately
  via `rockstarr-infra:ingest-workbook`.
- **Client refuses to name disqualifiers.** Explain that empty
  not-target rules mean the bot will draft for every warm reply,
  including recruiters and agencies. Push back once, then accept
  the client's call.

## What NOT to do

- Do not hardcode target or not-target roles the client didn't
  name. The bot carries zero domain-specific prescription.
- Do not merge target rules across campaigns. Each campaign's ICP
  spec can tighten — that logic lives in the campaign spec, not
  here.
- Do not write this file without running the interview. Empty-file
  fabrication from `client-profile.md` would bake agency opinion
  into bot behavior.
