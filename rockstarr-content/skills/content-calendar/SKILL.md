---
name: content-calendar
description: "This skill should be used when the user asks to \"build the content calendar\", \"plan the month\", \"schedule this month's content\", \"assign dates to the picked topics\", or \"run the calendar\". Reads the picked angles from the month's content-topics file plus the stack-cadence fields and publish log, assigns dates across the month (blog outlines first because of the outline gate, thought-leadership next, newsletters anchored to a chosen weekday, LinkedIn newsletters aligned to an approved TL piece), and writes one monthly calendar file to 02_inputs/content-calendar_YYYY-MM.md. One approval gate for the whole month."
---

# content-calendar

Take the month's picked angles and lay them on the calendar. This
is the monthly planning pass — the client approves the whole
month at once so per-piece drafting can run on schedule without
per-piece approval gates for scheduling decisions.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- Third business day of the month, after `ideate-topics` has
  produced `content-topics_YYYY-MM.md` and the client has picked
  angles.
- User asks to plan / schedule / calendar the month's content.
- A prior month's calendar is being amended mid-month (in which
  case the skill re-runs against the same file, appends `-v2`).

## Preconditions

- `/rockstarr-ai/02_inputs/content-topics_YYYY-MM.md` exists for
  the target month, with the user's picks marked.
- `/rockstarr-ai/00_intake/stack.md` exists with the content-cadence
  fields populated.
- `/rockstarr-ai/05_published/_publish.log` exists (empty is fine
  for a new client — used to avoid stepping on last month's dates).

If the topics file is missing or has no picks flagged, refuse and
tell the user to pick angles from the topics list first.

If the cadence block is missing from `stack.md`, refuse and point
the user at `rockstarr-infra:capture-stack`.

## Marking picks in the topics file

The client marks picks by adding a line `- **Pick:** yes` (or
`- **Pick:** no`) under each topic in
`content-topics_YYYY-MM.md`. Topics without an explicit `Pick: yes`
are excluded from the calendar. If the count of picks per lane does
not match the month's cadence from `stack.md`, flag the gap in
chat and ask the user how to resolve (add picks, accept under-
scheduling, or skip a lane this month).

## Inputs

Read in this order:

1. `/rockstarr-ai/02_inputs/content-topics_YYYY-MM.md` — the
   month's topics and picks.
2. `/rockstarr-ai/00_intake/stack.md` — cadence fields and any
   schedule preferences (preferred newsletter weekday, blog
   publish weekday, timezone).
3. `/rockstarr-ai/00_intake/client-profile.md` — for holidays,
   known travel / dark weeks, and the audience timezone.
4. `/rockstarr-ai/05_published/_publish.log` — last month's dates
   so back-to-back scheduling on the same weekday does not recur
   without intent.

## Scheduling rules

Slot order matters. Apply these in order:

1. **Researched blog slots first.** Blogs gate on outline approval,
   and the outline must land early enough that draft-blog can run
   before the blog's publish date. For each blog pick:
   - Outline target date: within the first 5 business days of the
     month.
   - Draft target date: at least 5 business days after outline
     approval.
   - Publish target date: at least 2 business days after draft
     approval. Space blog publish dates at least 7 days apart.
2. **Thought-leadership slots next.** Single-shot, so only the
   draft and publish dates matter. For each TL pick:
   - Draft target date: spaced so the client has real review time
     (at least 3 business days before publish).
   - Publish target date: space TL publish dates at least 5 days
     apart and stagger with blog publish dates so the client
     does not publish two long-form pieces on the same day.
3. **LinkedIn newsletter slots.** Each LinkedIn newsletter aligns
   to one approved TL piece from this month (or a prior month if
   the client is ramping). If `linkedin_newsletters_per_month >
   thought_leadership_per_month`, flag the gap — there aren't
   enough TL pieces to feed the cadence.
4. **Email newsletter slots last.** Anchor to a preferred weekday
   (default Tuesday if not set in stack.md). Space sends evenly
   across the month to hit `email_newsletters_per_month`. The
   newsletter slot date is the send date; drafting happens 1-2
   business days ahead.

If two slots land on the same business day across lanes, push the
lower-priority one (email < TL < blog) one business day later.

Avoid the 1st, 15th (often US holidays nearby), and any date the
client has flagged as unavailable.

## Output

Write to
`/rockstarr-ai/02_inputs/content-calendar_YYYY-MM.md` (ISO
year-month in the filename). If the file exists, append `-v2`,
`-v3`.

Required front-matter:

```yaml
# ---
client_id: [from client.toml]
month: "YYYY-MM"
produced_by: "rockstarr-content/content-calendar@0.2.0"
produced_at: "ISO timestamp"
topics_source: "02_inputs/content-topics_YYYY-MM.md"
cadence_snapshot:
  blogs_per_month: 2
  thought_leadership_per_month: 2
  email_newsletters_per_month: 4
  linkedin_newsletters_per_month: 0
slots_scheduled:
  blog: 2
  thought_leadership: 2
  email_newsletter: 4
  linkedin_newsletter: 0
cadence_gaps: []   # e.g., "blog: 1 picked vs. 2 target"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# Content calendar — [Client name] — [Month Year]

## Summary

- Cadence: N blogs, N TL, N newsletters, N LinkedIn newsletters.
- Picks scheduled: [counts per lane].
- Gaps: [e.g., "only 1 blog pick for cadence of 2 — user flagged OK to under-ship"].

## Calendar (chronological)

| Date | Weekday | Lane | Topic | Action | Slug | Draft due |
|------|---------|------|-------|--------|------|-----------|
| YYYY-MM-03 | Tue | blog | [Working title] | outline-blog | [slug] | — |
| YYYY-MM-05 | Thu | email-newsletter | [topic] | draft-newsletter | [slug] | YYYY-MM-04 |
| YYYY-MM-10 | Tue | blog | [Working title] | draft-blog | [slug] | YYYY-MM-10 |
| YYYY-MM-12 | Thu | email-newsletter | ... |
| YYYY-MM-14 | Sat | blog | [Working title] | publish | [slug] | — |
| YYYY-MM-17 | Tue | thought-leadership | [Working title] | draft-thought-leadership | [slug] | YYYY-MM-17 |
| YYYY-MM-19 | Thu | email-newsletter | ... |
| YYYY-MM-21 | Sat | thought-leadership | [Working title] | publish | [slug] | — |

## Per-lane detail

### Researched blogs
- [slug] — outline YYYY-MM-03, draft YYYY-MM-10, publish YYYY-MM-14
  - Pillar: ...
  - Angle: ...
  - KB evidence: ...

### Thought leadership
- [slug] — draft YYYY-MM-17, publish YYYY-MM-21
  - Pillar: ...
  - Stance: ...
  - LinkedIn newsletter republish? (if scheduled) YYYY-MM-28

### Email newsletters
- [slug] — send YYYY-MM-05
  - Theme / beat: ...
  - Monthly pieces linked: [slugs approved by send date]

### LinkedIn newsletters
- [slug] — publish YYYY-MM-28 (source: TL piece [slug])

## Notes / flags

- Any gaps between cadence and picks.
- Any schedule pushes caused by holidays or client-flagged unavailability.
- Any dependencies that could slip the month (e.g., "blog 2 outline
  must be approved by YYYY-MM-06 or publish date slips").
```

## Approval gate

After writing the file:

1. Summarize in chat: picks per lane, first and last publish dates
   of the month, any gaps or pushes.
2. Tell the user this is a monthly gate — one approval for the
   whole month via `rockstarr-infra:approve`.
3. Do not call `approve` yourself.
4. Once the calendar is approved, per-piece drafting runs on the
   calendar dates (outline-blog, draft-blog, draft-thought-
   leadership, draft-newsletter). LinkedIn-newsletter publishes
   require `publish-linkedin-newsletter` (DEFER in v0.2).

## What NOT to do

- Do not schedule a lane whose cadence in `stack.md` is 0.
- Do not schedule more picks than the client flagged with
  `Pick: yes`. If the count is short, flag the gap — do not
  invent picks.
- Do not schedule a LinkedIn newsletter with no corresponding
  approved or in-pipeline thought-leadership source.
- Do not silently push dates across months. If a blog's outline-
  to-publish timeline cannot fit in the remaining days, flag it
  and ask the user whether to accept slipping into next month.
- Do not run outline-blog or any drafting skill from inside this
  skill. Calendar-building and drafting are separate events
  triggered on their calendar dates.
- Do not overwrite an approved calendar. Amendments get a `-v2`
  file and a fresh approval gate.
