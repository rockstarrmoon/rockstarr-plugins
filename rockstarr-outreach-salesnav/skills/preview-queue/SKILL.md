---
name: preview-queue
description: "This skill should be used as the second step of the daily outreach loop (after confirm-session), or when the user asks to \"preview today's queue\", \"show what the bot plans to do today\", or \"write the daily outreach preview\". It writes a markdown file at 02_inputs/outreach/queue-YYYY-MM-DD.md listing every action the bot plans to take today: connects by lead and campaign with the day's cap math, message-step-N sends, follow-ups due, review-reply handoffs queued. Default enabled; togglable via stack.md outreach_daily_preview=false."
---

# preview-queue

Human-readable sneak peek of what the bot is about to do. Helps the
client catch "oh wait, don't message that lead yet" before a send
goes out.

## When to run

- Immediately after `confirm-session` passes in the daily loop.
- On-demand when the user wants to see today's plan without actually
  running anything.

## Gate

- If `stack.md.outreach_daily_preview` is `false`, skip. Return early
  with a one-line note in the caller's summary.

## Inputs

- `outreach-tasks.xlsx` (Tasks, Leads, Campaigns, plus recent
  Connections to compute weekly cap usage).
- `stack.md` (cap config is fixed — 20/day + 100/week — but we read
  the client timezone and run time for the header).

## Behavior

1. Compute **today's connection budget:**
   - `connections_sent_this_week = count(Connections WHERE iso_week = current AND date <= today)`
   - `carry_forward_this_week = sum(daily_shortfalls on earlier days of the current ISO week)` — a day's shortfall is `max(0, 20 - sent_that_day)` for each earlier day.
   - `today_budget = min(20 + carry_forward_this_week, 100 - connections_sent_this_week)`
2. Compute **round-robin split.** Active campaigns (Campaigns rows
   with `status = active`) split `today_budget` round-robin by slug.
   E.g., 3 active + budget 20 → `7 / 7 / 6` in slug order.
3. **Gather today's sends.** From Tasks:
   - `message-step-N` tasks with `due_date = today` and
     `status = pending`.
   - `follow-up` tasks with `due_date = today` and
     `status = pending`.
4. **Gather today's review handoffs.** Pending `review-reply` tasks
   that hit the preview window — flag as "awaiting client approval"
   without running anything.
5. **Gather pending book-meeting tasks.** `book-meeting` tasks in
   `status = pending` — listed here so the client sees the bot is
   about to run a booking form.
6. **Write the preview file.** Overwrite
   `/02_inputs/outreach/queue-<today>.md`. Template:

```markdown
# Outreach queue — <YYYY-MM-DD> (<client timezone>)

Daily run time: <stack.md.outreach_daily_run_time>
Session check: pass (ran <HH:MM>)
Weekly cap: <used>/100 — <remaining> remaining
Today's budget: <today_budget> = min(20 + carry <c>, 100 − used <used>)

## Connects planned — <today_budget> total

### <campaign-slug-a> — <n> connects
- <Lead Name> — <Title>, <Company> — <linkedin-url>
- ...

### <campaign-slug-b> — <n> connects
- ...

## Messages scheduled — <total> sends

### <campaign-slug-a>
- step-2 → <Lead Name> (<linkedin-url>) — accepted <YYYY-MM-DD>
- step-3 → ...

### <campaign-slug-b>
- ...

## Follow-ups due — <N>
- <Lead Name> (<linkedin-url>) — drafted today by rockstarr-reply

## Awaiting your approval (no bot action today)
- <Lead Name> — review-reply, handed <N> days ago
- ...

## Bookings the bot will attempt today
- <Lead Name> — <agreed time> via <booking-link or calendar>
- (empty if none)

## Notes
- <anything odd: cap close to 0, all campaigns paused, etc.>
```

7. Return a one-line summary to the caller naming the file and
   totals.

## What NOT to do

- Do not send anything from this skill. It's a read-only preview.
- Do not mutate Tasks statuses. Nothing is "started" here.
- Do not include the booking link URL in the preview. List the name
  of the booking source ("via Calendly" / "via GrowthAmp" / "via
  Google Calendar"), not the URL.
- Do not carry forward capacity across ISO weeks. Monday 00:00 local
  is a hard reset.
