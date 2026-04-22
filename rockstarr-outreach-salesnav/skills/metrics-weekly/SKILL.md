---
name: metrics-weekly
description: "This skill should be used every Friday end-of-day, or when the user says \"roll up the week\", \"close out the outreach week\", or \"write this week's Metrics (Weekly) rows\". It aggregates Metrics (Daily) rows per campaign per ISO week into Metrics (Weekly): connections, accepts, messages, replies, bookings, opt-outs, plus accept rate, reply rate, booking rate, plus weekly cap used and remaining. Then triggers outreach-weekly-report to render the human-readable weekly markdown."
---

# metrics-weekly

Weekly scoreboard builder. Rolls the daily rows into one row per
campaign per ISO week and hands off to `outreach-weekly-report` to
produce the client-facing markdown.

## When to run

- Scheduled: Friday end-of-day in the client's timezone.
- On-demand if the user wants a mid-week preview (write to a
  temporary row and mark `status = preview`, or compute in-memory
  and skip the write — V0.1 simply refuses a mid-week run and asks
  the user to wait until Friday).

## Inputs

- Metrics (Daily) filtered to the ISO week ending today.
- Campaigns (to include `paused` campaigns that had activity this
  week — they still get a weekly row).

## Behavior

1. **Determine the ISO week.** Use the client's timezone.
2. **For each campaign with any Metrics (Daily) row in this week:**
   Append one Metrics (Weekly) row:
   - `iso_week` = current ISO week (YYYY-WW)
   - `campaign_slug`
   - `connections_sent` = sum across daily rows
   - `accepts` = sum
   - `messages_sent` = sum
   - `replies` = sum
   - `bookings` = sum
   - `opt_outs` = sum
   - `accept_rate` = `accepts / max(connections_sent, 1)` — formatted
     as a percentage string like `"28.6%"` for readability; keep a
     raw decimal column (`accept_rate_raw`) too for downstream
     analytics.
   - `reply_rate` = `replies / max(accepts, 1)`
   - `booking_rate` = `bookings / max(replies, 1)`
   - `weekly_cap_used` = sum(connections_sent across all campaigns
     this week) — duplicated on every campaign row for easy read.
   - `weekly_cap_remaining` = `100 - weekly_cap_used`.
3. **Trigger `outreach-weekly-report`.** Pass the ISO week.
4. **Save the workbook.**

## Output

- Number of weekly rows written
- Pointer to the rendered weekly report (from
  `outreach-weekly-report`)

## What NOT to do

- Do not recompute Metrics (Daily) from scratch here. Trust what
  `metrics-daily` wrote. If it's wrong, fix `metrics-daily` and
  rerun; don't silently correct downstream.
- Do not include `stopped` campaigns that had no activity in this
  week. Noise.
- Do not run mid-week. The weekly cap math + the "week-over-week"
  framing only makes sense at end-of-week.
