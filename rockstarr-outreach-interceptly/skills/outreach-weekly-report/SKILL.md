---
name: outreach-weekly-report
description: "This skill should be used after metrics-weekly finishes, or when the user asks to \"generate the weekly outreach report\", \"write this week's report\", or \"render the Interceptly weekly\". Consumes Metrics (Weekly) rows for the target ISO week and produces a human-readable markdown report at /06_reports/weekly/outreach-YYYY-WW.md with per-account tables, week-over-week deltas, stale review-reply callout, non-ICP log highlights, flagged-leads list, session-failure summary, and a 'what Rachel / Jon should notice' block."
---

# outreach-weekly-report

Human-readable wrap-up. The Friday artifact the client reads
before Monday. Every metric on the page has a pointer back to the
workbook so the client can audit.

## When to run

- Automatically after `metrics-weekly` finishes.
- On demand when the user asks for a specific week's report.

## Preconditions

- `Metrics (Weekly)` sheet has rows for the target ISO week
  (produced by `metrics-weekly`).
- `outreach-mirror.xlsx` is readable.

## Inputs

- `iso_week` — `YYYY-WW` (defaults to current).

## Behavior

### Step 1 — Read metrics

Read every `Metrics (Weekly)` row for the target ISO week.

### Step 2 — Read adjunct signals

- `Flags` sheet — count of open (unresolved) flagged leads at
  week end.
- `_non_icp_log.md` — pull the 3-5 most recent not-target
  rulings for the "refine icp-qualifications.md" callout.
- `_errors.md` — count session failures, Chrome MCP drops,
  Labels-nav-bug hits.
- `Tasks` sheet — stale `review-reply` tasks (past due and
  still pending).
- `Replies` sheet — bookings this week with meeting_datetime.

### Step 3 — Render the report

Write `/06_reports/weekly/outreach-<YYYY-WW>.md`:

```markdown
---
iso_week: <YYYY-WW>
week_start: <YYYY-MM-DD>
week_end: <YYYY-MM-DD>
generated_at: <ISO>
schema_version: 1
---

# Outreach — Week of <week_start>

_Interceptly is source-of-truth; this report reads from the local
audit mirror `/02_inputs/outreach/outreach-mirror.xlsx`._

## Headline numbers

<one-line summary, e.g., "Across 3 managed accounts: 127 unreads
processed, 52 sends, 4 bookings, 18 flagged, 0 session failures.">

## Per-account results

### <account_label 1>

| metric | this week | prev week | Δ |
|---|---|---|---|
| Unreads processed | N | N | +N |
| Sends | N | N | +N |
| Bookings | N | N | +N |
| Non-ICP declines | N | N | +N |
| Flags | N | N | +N |
| Reply-to-send ratio | X% | Y% | +Z pp |
| Booking rate | X% | Y% | +Z pp |

**Label distribution (this week):** interested=N,
follow_up=N, ignore=N, not_interested=N, ...

### <account_label 2>

... (same shape) ...

## Bookings

<bulleted list — one line per booking: lead_name (company),
campaign, meeting_datetime, source (automated / manual).>

## Flagged leads still open

<bulleted list of Flags rows with resolution = blank. Include
the reason and the day flagged. If none, say "none.">

## Stale review-reply tasks

<bulleted list of Tasks rows where task_type = review-reply AND
due_date < today AND status = pending. If none, say "none.">

## Non-ICP Log highlights

<3-5 most recent not-target rulings. For each: lead name,
rule cited, date.>

> If you keep seeing patterns here that feel wrong, re-run
> `capture-icp-qualifications` to tighten the rules.

## Session + UI health

- Session failures: N (per account if >0)
- Chrome MCP drops: N
- Labels-nav-bug hits: N

## What Rachel / Jon should notice

<2-4 bullets that translate the numbers into judgment — picked
based on notable deltas, first-time-occurring events, or
shifting patterns. Examples the author might write:>

- Booking rate on <account> jumped from Y% to X% — what
  changed? Worth checking whether the warm-reply pattern tuned
  last week is the signal.
- Non-ICP declines climbed to X% on <account> — the ICP rules
  may be letting through roles that don't convert. Consider
  refining.
- N stale review-reply tasks sitting past due — block calendar
  time to clear them Monday morning.
- Zero session failures all week — confirm-session-interceptly
  doing its job.
```

### Step 4 — Return

`{report_path: "/06_reports/weekly/outreach-<YYYY-WW>.md"}`
plus a short chat summary: headline numbers + the "what to
notice" bullets.

## Rhetoric rules

The "What Rachel / Jon should notice" section is the only place
this skill offers interpretation. Everywhere else it cites
numbers. Interpretation should be:

- Specific — name the account, the metric, the delta.
- Actionable — what to look at, what to decide, what to tune.
- Short — 2-4 bullets max. The numbers do most of the work.

Do NOT editorialize in the metrics tables. Tables are facts.

## Failure modes

- **Metrics (Weekly) row is missing for an account.** Skip
  that account's section with a one-line note and continue.
- **Prev-week deltas blank** (first week, or a missed week).
  Blank Δ is fine.
- **Long bookings list** (>10 in a week). Cap the rendered list
  at 10 with a "see workbook for full list" line.

## What NOT to do

- Do not fabricate bookings, flags, or error counts. Every
  number in this report comes from the workbook.
- Do not editorialize numbers you don't have. If a metric is
  blank, say "no data" rather than assuming.
- Do not write recommendations that require changing
  `icp-qualifications.md` without telling the operator to re-
  run `capture-icp-qualifications`. The bot carries zero
  opinion on ICP rules.
