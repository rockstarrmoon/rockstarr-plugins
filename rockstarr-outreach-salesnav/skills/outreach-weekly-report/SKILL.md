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
- Leads sheet, filtered per campaign to rows where `state` is
  `accepted` or later AND `date_accepted` falls in the ISO week
  (for the connect-only named-accepts block — see Render rules).
- Campaigns sheet (for `campaign_type` lookup driving the
  full-sequence vs connect-only render branch).
- `_errors.md` lines timestamped within the week (for the bot-
  heartbeat callout).

## Output

Write to `/rockstarr-ai/06_reports/weekly/outreach-[iso_week].md`.
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

## Per campaign — full-sequence

| Campaign | Connects | Accepts | Accept % | Msgs | Replies | Reply % | Bookings | Booking % |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| <slug-a> | 35 | 10 | 28.6% | 22 | 4 | 40.0% | 1 | 25.0% |
| <slug-b> | 28 | 5  | 17.9% | 14 | 2 | 40.0% | 0 | 0%    |

## Per campaign — connect-only

Connect-only campaigns send connection requests only — no
post-accept sequence, no reply funnel by design. Accept rate is the
primary signal.

| Campaign | Connects | Accepts | Accept % | Saved search status |
|---|---:|---:|---:|---|
| <slug-c> | 100 | 31 | 31.0% | active |
| <slug-d> | 80  | 12 | 15.0% | exhausted — operator should stop |

_Connect-only campaigns do not contribute to messages, replies, or
bookings in the at-a-glance table above (their values are zero by
design). They DO contribute to connects sent and accepts received._

### Accepted this week — named list

For each connect-only campaign with one or more accepts this week,
name the accepting leads. The accept IS the success event for these
campaigns; surfacing names is what makes the report actionable
beyond an accept-rate percentage.

**<slug-c>** — 31 accepts this week:
- Jane Doe — CMO at SaaS Co
- John Smith — Head of RevOps at Mid-Co
- Sarah Lee — CTO at Series-B Co
- ... (12 more shown; +16 over the soft cap, see workbook Leads
  filtered to `campaign_slug=<slug-c>`, `state=accepted`,
  `date_accepted` in this week)

**<slug-d>** — 12 accepts this week:
- Michael Brown — VP Eng at Enterprise Co
- Pat Johnson — COO at Manufacturing Co
- ...

### Week-over-week
- <slug-a>: accept rate up from X → Y; +1 booking WoW.
- <slug-b>: accept rate flat; reply rate up.
- <slug-c>: connect-only — accept rate up 4 points; saved search
  yielding fresh leads each day.
- <slug-d>: connect-only — saved search exhausted; recommend stop
  or refresh the search.

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

## Render rules — campaign-type branching

The per-campaign rendering forks on `campaign_type` from the
Campaigns sheet:

- **`full-sequence` campaigns** render in the "Per campaign —
  full-sequence" table with the full column set: Connects, Accepts,
  Accept %, Msgs, Replies, Reply %, Bookings, Booking %.
- **`connect-only` campaigns** render in the "Per campaign —
  connect-only" table with a narrower column set: Connects,
  Accepts, Accept %, Saved search status. Reply / booking columns
  are omitted entirely (zero by design is not the same as
  underperforming, and showing zeros invites the wrong reading).

Skip a table entirely if there are no campaigns of that type for
the week. Don't render an empty "Per campaign — connect-only"
section just to keep the structure symmetric.

The "Accepted this week — named list" sub-block under the
connect-only section follows the same skip rule: omit it entirely
if no connect-only campaigns posted accepts this week. Per-campaign
list:

- Query Leads sheet for rows where `campaign_slug` matches a
  connect-only campaign, `state` is `accepted` or later, AND
  `date_accepted` falls inside the ISO week being rendered.
- Render each as `[lead_name] — [lead_title] at [lead_company]`.
  If `title` or `company` is missing on a row, render the name
  alone; never render `— at` or trailing punctuation that implies
  missing data.
- Sort by `date_accepted` descending (most recent first) within
  each campaign so the operator's eye lands on the latest names.
- Soft cap at 15 names per campaign. Past the cap, render
  `... (X more shown; +[N] over the soft cap, see workbook Leads
  filtered to <filter expression>)` so the operator has a clear
  path to the full list without bloating the report.
- Connect-only campaigns with zero accepts this week appear in the
  per-campaign table (with `Accepts: 0` and `Accept %: 0%`) but
  do NOT appear in the named-list sub-block at all. A 0-accepts
  campaign needs a different conversation than a long list of
  names.

Full-sequence campaigns do NOT get a parallel named-accepts block.
Their accept is an intermediate step, not the success event;
downstream replies and bookings are where the named events live,
and naming accepts there would clutter the report with leads who
may never reply. The asymmetry is deliberate.

The "At a glance" totals at the top of the report aggregate
honestly across BOTH types:

- `Connections sent`, `Accepts`, `Opt-outs` — sum across
  full-sequence + connect-only.
- `Messages sent`, `Replies`, `Bookings` — sum across
  full-sequence only. Connect-only campaigns contribute zero to
  these by design, so the sum is naturally correct, but be explicit
  about which type's denominator is in play if the per-type counts
  could mislead.

## Write the "What Rachel / Jon should notice" block by hand

The bullets and per-campaign commentary cannot be pure template.
They have to read the actual Metrics (Weekly) deltas, spot the one
or two load-bearing stories, and say them in plain prose. Use the
client's style guide for tone; this is a client-facing report.

- If nothing interesting moved, say so honestly: "Quiet week —
  numbers are in range. Worth watching next week."
- If a number moved dramatically, say *why* as a hypothesis, not a
  conclusion: "Accept rate dropped from 28% to 14% in [slug-b].
  Message 1 is blank by spec, so suspect the saved-search filter or
  a change in the ICP's current environment — not the copy."
- If the weekly cap bound (we hit 100 mid-week), flag it.
- If `confirm-session` ever failed, surface it loudly in the
  heartbeat AND in this block.
- For connect-only campaigns, accept rate is the only quality
  signal. Don't invent a "reply rate looks low" line — the campaign
  has no reply funnel by design, and zero replies is not a problem,
  it's the point. Talk about accept rate, the saved-search yield,
  and whether the campaign is approaching its target_lead_count.

## Stop-slop pass (mandatory)

Before writing the report file, run every prose block through
`rockstarr-infra:stop-slop` as the final pass. The prose blocks in
this report are:

- `### Week-over-week` bullets
- `## What Rachel / Jon should notice` paragraphs
- `## Actions for next week` narrative bullets (the `continue /
  pause / tweak` reasoning text, not the slug labels themselves)

The tables, heartbeat lines, stale-task callout, and the connect-
only named-accepts list are structural artifacts and are exempt
(lead names, titles, and companies are data — not prose, and not
something stop-slop should rewrite). Fix any slop flags (filler
phrases, binary contrasts, passive voice, em dashes, vague
declaratives, metronomic rhythm) before the report lands in
`/06_reports/weekly/`. Order is style-guide first, stop-slop last.
If stop-slop is unavailable, refuse and tell the user
`rockstarr-infra` needs to be installed or updated.

## What NOT to do

- Do not invent metrics. Everything in the tables traces to
  Metrics (Weekly).
- Do not include the booking link URL in the report. It's not
  client-facing content.
- Do not paraphrase third-party content to pad the report.
- Do not soften a bad week. Honest reporting is the whole point.
- Do not send the report. Render only. The client reads it in
  place or forwards it.
