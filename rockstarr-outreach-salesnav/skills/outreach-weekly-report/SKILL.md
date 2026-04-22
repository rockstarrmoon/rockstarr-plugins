---
name: outreach-weekly-report
description: "This skill should be used after metrics-weekly finishes, or when the user asks to \"generate the weekly outreach report\", \"write this week's report\", or \"render the Sales Nav campaign weekly\". It consumes Metrics (Weekly) rows for the target ISO week and produces a human-readable markdown report at /06_reports/weekly/outreach-YYYY-WW.md with per-campaign tables, week-over-week deltas, stale review-reply callout, weekly cap usage, and a \"what Rachel / Jon should notice\" block."
---

# outreach-weekly-report

Writes the weekly markdown the client actually reads. Everything
else in the plugin feeds this moment.

## When to run

- Called by `metrics-weekly` on Friday.
- On-demand when the user asks for a re-render of a specific ISO
  week.

## Inputs

- `iso_week` (YYYY-WW) — which week to render.
- Metrics (Weekly) rows for that week (all campaigns).
- Metrics (Weekly) rows for the prior week (for deltas).
- Tasks sheet filtered to `type = review-reply`, `status = pending`
  (for the "awaiting you" callout).
- `_errors.md` lines timestamped within the week (for the bot-
  heartbeat callout).

## Output

Write to `/rockstarr-ai/06_reports/weekly/outreach-<iso_week>.md`.
Overwrite existing.

### Template

```markdown
# Outreach weekly — <iso_week>

_Generated <YYYY-MM-DD HH:MM> by rockstarr-outreach-salesnav_

## At a glance

| | This week | Last week | Δ |
|---|---:|---:|---:|
| Connections sent | <N> | <N> | <+/- N> |
| Accepts | <N> | <N> | <+/- N> |
| Messages sent | <N> | <N> | <+/- N> |
| Replies | <N> | <N> | <+/- N> |
| Bookings | <N> | <N> | <+/- N> |
| Opt-outs | <N> | <N> | <+/- N> |

Weekly cap used: **<used>/100** (<remaining> left)

## Per campaign

| Campaign | Connects | Accepts | Accept % | Msgs | Replies | Reply % | Bookings | Booking % |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| <slug-a> | 35 | 10 | 28.6% | 22 | 4 | 40.0% | 1 | 25.0% |
| <slug-b> | 28 | 5  | 17.9% | 14 | 2 | 40.0% | 0 | 0%    |

### Week-over-week
- <slug-a>: accept rate up from X → Y; +1 booking WoW.
- <slug-b>: accept rate flat; reply rate up.

## Awaiting your approval

Stale review-reply tasks (handed to rockstarr-reply, not yet approved):

| Lead | Campaign | Handed off | Days waiting |
|---|---|---|---:|
| <Name> | <slug> | <date> | <N> |

_No SLA is enforced on these. This callout exists so nothing rots._

## Bot heartbeat

- Daily session checks: <N passes> / <N failures> this week.
- Session failures (if any):
  - <date> — <reason>
- LinkedIn UI anomalies logged to _errors.md:
  - <date> — <summary>
- If confirm-session ever failed silently, treat as P0.

## What Rachel / Jon should notice

- <One-paragraph prose callout on the most load-bearing number this
  week — e.g., accept rate drop in <slug-b>, or a new booking in
  <slug-a> worth celebrating, or a cap that will bind next week.>
- <Second callout if there's another distinct story.>
- <A question for the client to answer before next Monday.>

## Actions for next week

- Continue / pause / tweak calls per campaign
  - <slug-a>: continue
  - <slug-b>: tweak Message 2 copy — reply rate is solid but accept
    rate slipped; accept rate is a function of the saved-search
    filter (Message 1 is blank by spec), so review the ICP and the
    saved search before changing any message body.
- Any campaigns proposed for `stop-campaign` given the data
- Backup: `/06_reports/data/outreach-tasks-backup-<iso_week>.xlsx`
  will be written by `backup-workbook` at end-of-week.
```

## Write the "What Rachel / Jon should notice" block by hand

The bullets and per-campaign commentary cannot be pure template.
They have to read the actual Metrics (Weekly) deltas, spot the one
or two load-bearing stories, and say them in plain prose. Use the
client's style guide for tone; this is a client-facing report.

- If nothing interesting moved, say so honestly: "Quiet week —
  numbers are in range. Worth watching next week."
- If a number moved dramatically, say *why* as a hypothesis, not a
  conclusion: "Accept rate dropped from 28% to 14% in <slug-b>.
  Message 1 is blank by spec, so suspect the saved-search filter or
  a change in the ICP's current environment — not the copy."
- If the weekly cap bound (we hit 100 mid-week), flag it.
- If `confirm-session` ever failed, surface it loudly in the
  heartbeat AND in this block.

## What NOT to do

- Do not invent metrics. Everything in the tables traces to
  Metrics (Weekly).
- Do not include the booking link URL in the report. It's not
  client-facing content.
- Do not paraphrase third-party content to pad the report.
- Do not soften a bad week. Honest reporting is the whole point.
- Do not send the report. Render only. The client reads it in
  place or forwards it.
