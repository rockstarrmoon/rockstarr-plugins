---
name: metrics-daily
description: "This skill should be used as the final step of the daily outreach loop, or when the user says \"roll up today's metrics\", \"close out the day\", or \"update the daily outreach scoreboard\". It writes one row per active campaign into the Metrics (Daily) sheet — connections sent, accepts, messages sent, replies, bookings, opt-outs for today — and updates the ISO-week running totals that daily-connect's budget math reads the next morning."
---

# metrics-daily

End-of-day roll-up. Closes the loop on today's run and sets up the
state `daily-connect` needs tomorrow.

## When to run

- Final step of the scheduled daily loop.
- On-demand if the user wants a clean end-of-day snapshot after a
  manual run.

## Inputs

- Connections, Messages, Replies, Leads, Tasks (reads).
- Campaigns (to know which campaigns are `active` — `paused` and
  `stopped` campaigns do not get a new Metrics (Daily) row, but we
  still include them in the ISO-week totals if they had activity
  before being paused).

## Behavior

1. **Scope to today.** "Today" is defined by the client's timezone
   in `stack.md`. ISO week is computed from the same timezone.
2. **For each campaign with any activity today OR `status = active`:**
   Append one Metrics (Daily) row:
   - `date` = today
   - `iso_week` = current ISO week (YYYY-WW format)
   - `campaign_slug`
   - `connections_sent` = count(Connections today for this slug)
   - `accepts` = count(Leads with `date_accepted = today` for this
     slug)
   - `messages_sent` = count(Messages today, step in
     message-step-1|2|3 or reply or follow-up, for this slug)
   - `replies` = count(Replies today for this slug)
   - `bookings` = count(Leads with `date_booked = today` for this
     slug)
   - `opt_outs` = count(Leads that moved to `state = opted-out`
     today for this slug)
3. **Compute weekly running totals.** For each active campaign,
   store or update:
   - `Metrics (Weekly) in-progress` — not a separate sheet, but a
     computed running total that `daily-connect` reads tomorrow to
     derive its budget math. Two options:
     a. Derive it fresh from Connections on each `daily-connect`
        run (simpler, always correct).
     b. Cache it on the workbook's metadata row.
     V0.1: use option (a). Simpler and resilient to manual edits.
4. **Heartbeat check.** If `confirm-session` failed earlier today,
   append a loud note to today's publish-log file:
   `DAILY LOOP ABORTED — confirm-session failure. No Metrics (Daily)
   row written.` Skip steps 1–3 entirely in that case.
5. **Save.**
6. **Append to publish-log.** `/05_published/outreach/<today>.md`
   gets a trailing summary block:
   ```
   # Daily metrics (<today>)
   | campaign         | connects | accepts | msgs | replies | bookings | opt-outs |
   |------------------|---------:|--------:|-----:|--------:|---------:|---------:|
   | <slug-a>         |   7      |   2     |  5   |   1     |   0      |   0      |
   | <slug-b>         |   6      |   1     |  3   |   0     |   0      |   0      |
   | **total**        |  13      |   3     |  8   |   1     |   0      |   0      |
   weekly cap: <used>/100 — <remaining> remaining
   ```

## Output

- Number of rows written
- Total connects for the day
- `weekly_used` / `weekly_remaining`
- `session_aborted: true|false`

## What NOT to do

- Do not retroactively edit earlier Metrics (Daily) rows. If
  yesterday's numbers are wrong, it's because a lead's state changed
  after the roll-up — that's fine. Weekly metrics aggregate at end
  of week; small daily drift is honest.
- Do not write Metrics (Weekly) rows from this skill. That's
  `metrics-weekly`'s Friday job.
- Do not touch the client's timezone. Read from `stack.md`.
- Do not skip the heartbeat note on a session-aborted day. The
  client needs to see why their scoreboard didn't move.
