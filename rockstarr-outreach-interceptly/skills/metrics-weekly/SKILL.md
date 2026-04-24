---
name: metrics-weekly
description: "This skill should be used every Friday end-of-day, or when the user says \"roll up the week\", \"close out the outreach week\", or \"write this week's Metrics (Weekly) rows\". Aggregates Metrics (Daily) rows per managed account per ISO week into Metrics (Weekly): sends, bookings, reply-to-send ratio, label distribution, non-ICP declines, flags, campaigns configured/stopped, session failures, plus week-over-week deltas. Triggers outreach-weekly-report to render the human-readable weekly markdown."
---

# metrics-weekly

Weekly rollup. One row per managed account per ISO week.

Reads `Metrics (Daily)` which `metrics-daily` has already
populated. This skill is a pure aggregation of the daily
scoreboard — it does not re-read the raw activity sheets.

## When to run

- Scheduled: every Friday at client-configured end-of-day.
- On demand when the user says "close the outreach week" or
  "recompute this week's metrics."

## Preconditions

- `Metrics (Daily)` sheet has rows for every business day of
  the target ISO week. Missing days are fine (write partial)
  but are noted.
- `outreach-mirror.xlsx` exists.

## Inputs

- `iso_week` — defaults to the current ISO week.

## Behavior

### Step 1 — Determine the week range

ISO-8601 week. Monday-start to Sunday-end. Compute
`week_start` and `week_end` dates.

### Step 2 — Per-account aggregation for the week

For each managed account in `stack.md.outreach_accounts[]`,
sum the Metrics (Daily) rows for the target week:

- `total_unreads_processed`
- `total_sends`
- `total_bookings`
- `total_non_icp_declines`
- `total_flags` — sourced from `flags` column in Metrics (Daily),
  which counts entries in `/02_inputs/replies/_flags.md`
  (written by `rockstarr-reply:flag-for-review`).
- `total_tasks_created`
- `total_tasks_completed`
- `total_campaigns_configured`
- `total_campaigns_stopped`
- `total_session_failures`
- Label counts by type — `interested`, `booked`, `referral`,
  `follow_up`, `contact_later`, `bad_fit`, `ignore`,
  `not_interested`.
- `any_day_partial` — true if ANY daily row in the week has
  `partial = true`. Surfaces a data-quality caveat to the
  weekly report.

### Step 3 — Derived ratios

For each account:

- **reply_to_send_ratio** — `(total_unreads_processed /
  total_sends)` when `total_sends > 0`, else blank.
- **booking_rate** — `(total_bookings / total_unreads_processed)`
  when processed > 0, else blank.
- **flag_rate** — `(total_flags / total_unreads_processed)` —
  high values mean `icp-qualifications.md` isn't tight enough
  or rockstarr-reply's classify-reply is punting too often.
- **non_icp_rate** — `(total_non_icp_declines /
  total_unreads_processed)` — helps tune ICP rules (owned by
  rockstarr-reply's `capture-icp-qualifications`).

### Step 4 — Week-over-week deltas

For each metric, compute the delta against the same metric from
`iso_week - 1`. If no row exists for the previous week, leave
delta blank rather than writing `+100%` against zero.

### Step 5 — Write to Metrics (Weekly)

Append one row per account per week to the `Metrics (Weekly)`
sheet. Schema:

| column | value |
|---|---|
| iso_week | `<YYYY-WW>` |
| week_start | `YYYY-MM-DD` |
| week_end | `YYYY-MM-DD` |
| account_label | `<account>` |
| total_unreads_processed | N |
| total_sends | N |
| total_bookings | N |
| total_non_icp_declines | N |
| total_flags | N |
| total_tasks_created | N |
| total_tasks_completed | N |
| total_campaigns_configured | N |
| total_campaigns_stopped | N |
| labels_interested | N |
| labels_booked | N |
| labels_referral | N |
| labels_follow_up | N |
| labels_contact_later | N |
| labels_bad_fit | N |
| labels_ignore | N |
| labels_not_interested | N |
| reply_to_send_ratio | X.XX |
| booking_rate | X.XX |
| flag_rate | X.XX |
| non_icp_rate | X.XX |
| delta_sends | +/-N |
| delta_bookings | +/-N |
| delta_flags | +/-N |
| delta_booking_rate_pp | +/-X.XX |
| session_failures | N |
| any_day_partial | true/false |

### Step 6 — Trigger the weekly report

Call `outreach-weekly-report` with `iso_week` = this week.
That skill produces the human-readable markdown at
`/06_reports/weekly/outreach-<YYYY-WW>.md`.

### Step 7 — Return

A brief summary with the per-account headlines: sends,
bookings, notable deltas, session-failure count. Flag if
`any_day_partial` — the week's numbers may be floors rather
than truth.

## Failure modes

- **Missing Metrics (Daily) rows.** Write partial weekly rows
  and note which days were missing. Do NOT fabricate daily
  activity.
- **Previous-week Metrics (Weekly) doesn't exist** (first
  week running). Leave deltas blank; note in the weekly report.
- **Mirror workbook locked by another process.** Retry after
  `backup-workbook` releases the lock. If still locked, abort
  with a clear error.

## What NOT to do

- Do not write to `Metrics (Daily)` from here. That's
  `metrics-daily`'s ownership.
- Do not re-aggregate from the raw activity sheets (Messages,
  Labels, Tasks, Qualifications, etc.) — `metrics-daily`
  already rolled those up into the daily scoreboard. Read the
  daily sheet, not the raw.
- Do not skip accounts with zero activity. Every managed
  account gets a row — zeros are information.
- Do not compute deltas against fabricated baselines. A blank
  delta is honest.
