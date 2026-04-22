---
name: stop-campaign
description: "This skill should be used when the user says \"pause this campaign\", \"stop the campaign\", \"kill the campaign\", or names a campaign slug to wind down. It flips the Campaigns row status to paused (resumable) or stopped (permanent), and cancels every pending task for that campaign's leads. Paused-campaign leads remain reserved to the campaign; stopped-campaign leads become available for re-use in future campaigns."
---

# stop-campaign

Pause or stop a running campaign. The only skill that changes a
campaign's run state after registration.

## When to run

- User says to pause, stop, kill, or wind down a specific campaign
  slug.
- Wrong-account incident after `confirm-session` flagged the wrong
  LinkedIn — the client may pause every active campaign while they
  sort it.

## Inputs

- `campaign_slug`
- `action` — `pause` or `stop`.

If the user is ambiguous about which action they want, ask. The
distinction matters: `pause` keeps the slug reserved to its leads;
`stop` frees the leads for another campaign.

## Preconditions

- `02_inputs/outreach/outreach-tasks.xlsx` exists.
- Campaigns sheet has a row for the slug with
  `status in (active, paused)`. If it's already `stopped`, refuse
  and tell the user the campaign is already done.

## Behavior

1. **Open the workbook.** Shared `xlsx` skill.
2. **Update Campaigns row.**
   - If `action = pause`: `status = paused`. `date_stopped` stays
     empty.
   - If `action = stop`: `status = stopped`. `date_stopped = today`.
3. **Cancel pending tasks.** For every Tasks row where
   `campaign_slug` matches and `status = pending`, flip `status` to
   `cancelled` and set `completed_at = now`. This includes
   `connect`, `message-step-N`, `follow-up`, `review-reply`, and
   `book-meeting` tasks. `review-reply` tasks that were already
   handed to `rockstarr-reply` are cancelled here too — the reply
   thread will surface as an orphan in the weekly report, which is
   the correct behavior (the client decided to stop the campaign;
   they can still reply manually if they want).
4. **Leave Leads state alone.** Do not flip lead states retroactively.
   A lead that reached `accepted` or `replied` stays in that state so
   the metrics history is honest.
5. **Save.**
6. **Log.** Append to `/05_published/outreach/<today>.md`:
   `<action>d campaign <slug> — <N> tasks cancelled`.

## Output

Return a summary: slug, action, number of tasks cancelled, number of
leads whose tasks were touched. Remind the user whether the leads
are still reserved (paused) or freed for re-use (stopped).

## What NOT to do

- Do not delete the Campaigns row. We keep the history.
- Do not delete Leads rows. Historical state matters for metrics.
- Do not send any final messages, bookings, or connects. Stop means
  stop.
- Do not retroactively revise Metrics (Daily / Weekly) rows. What
  happened happened.
