---
name: send-approved-reply
description: "This skill should be used when rockstarr-reply's approve flow fires on a Sales Nav reply thread, or when the user says \"send the approved reply to this lead\", \"ship the reply\", or \"send Jane's approved response\". It sends the client-approved reply body via Sales Navigator through Chrome MCP, logs the send to the Messages sheet, and creates a single follow-up task due two days out (body drafts fresh on its due-date via rockstarr-reply unless cancelled by a subsequent reply or booking)."
---

# send-approved-reply

Event-driven. The only skill in this plugin that sends a reply. Fires
when `rockstarr-reply:approve` closes the loop on a `review-reply`
task this bot created.

## When to run

- `rockstarr-reply:approve` signals that a reply draft has been
  approved for a specific lead / campaign / thread.
- On-demand when the user approves a reply manually and wants the
  bot to execute the send.

## Preconditions

- A recent `confirm-session` pass (within 30 min). If stale, run
  `confirm-session` first. Event-driven sends are not exempt.
- The approved reply body exists as output of `rockstarr-reply`
  (either in `03_drafts/outreach/replies/<thread>.md` moved to
  `04_approved/outreach/replies/<thread>.md`, or passed inline by
  the caller).
- The target `review-reply` Tasks row is still pending (not cancelled
  by a newer reply).

## Inputs

- `lead_url`
- `campaign_slug`
- `thread_ref` — pointer to the approved reply body (file path or
  inline string)
- `source` — `rockstarr-reply:approve` or `manual`

## Behavior

1. **Load the approved body** from `thread_ref`. Minimal variable
   substitution is already baked in by `rockstarr-reply`; do not
   rewrite.
2. **Open the Sales Nav thread** for this lead via Chrome MCP.
3. **Send the body.** Paste into the composer, submit, verify send.
   - If the thread is missing (lead removed, Sales Nav UI changed):
     log to `_errors.md`, mark the approved reply's `review-reply`
     task `cancelled` with reason `thread_missing`, flip Leads.state
     to `opted-out`. Do not create a follow-up task.
4. **Mark the `review-reply` task done.** `status = done`,
   `completed_at = now`.
5. **Write to Messages.**
   - `date`, `lead_url`, `campaign_slug`
   - `step` = `reply`
   - `body_sent` = exact body
6. **Seed the 2-day follow-up.** Append to Tasks:
   - `type` = `follow-up`
   - `due_date` = today + 2 business days
   - `status` = `pending`
   - `metadata.prev_reply_id` = the Replies row id that triggered
     the review-reply
   The follow-up's body is NOT generated now. On the due-date,
   `send-scheduled-messages` calls `rockstarr-reply:draft-reply` to
   write a fresh body from the current thread context and the
   client approves before send. If a new inbound arrives before the
   due-date, `detect-replies` cancels this follow-up.
7. **Update Leads.** If state is `replied`, leave it; the follow-up
   thread is still an open conversation.
8. **Save the workbook.**
9. **Log.** Append to `/05_published/outreach/<today>.md`:
   `send-approved-reply — sent reply to <lead_url> (<campaign_slug>), 2-day follow-up queued for <due_date>`.

## Output

- `sent: true | false`
- `followup_task_id` (if created)
- `skipped_reason` (if `sent: false`)

## What NOT to do

- Do not create a follow-up task if the send failed. A thread
  without a send does not need a 2-day follow-up.
- Do not draft the follow-up body right now. The whole point of
  deferring is that thread context can shift; the fresh draft on
  the due-date is more honest.
- Do not bypass `confirm-session`. A recent pass is required even
  for event-driven sends.
- Do not send more than one reply per `rockstarr-reply:approve`
  event. Each approval = one send.
