---
name: detect-replies
description: "This skill should be used in the daily outreach loop after send-scheduled-messages, optionally again as an afternoon pass, or when the user says \"check for new replies\", \"scan the Sales Nav inbox\", or \"pull today's inbound messages\". It reads inbound messages from Sales Navigator and LinkedIn via Chrome MCP, matches each to a Leads row, appends to the Replies sheet, cancels every pending message-step-N and follow-up task for that lead, and creates a review-reply task that hands the thread to rockstarr-reply's draft-reply for client-approved response. Booking detection happens via mark-booked, not here."
---

# detect-replies

The hand-off point between outreach and reply. Every inbound message
from a lead in a live campaign comes through here, and every one of
them routes to `rockstarr-reply` for client-gated response.

## When to run

- Daily loop, after `send-scheduled-messages`, before `daily-connect`.
- Optional afternoon pass — this skill may re-run later in the day
  to pick up afternoon replies. The afternoon pass does NOT re-run
  the connect loop.
- On-demand when the user asks "did anyone reply?"

## Preconditions

- `confirm-session` passed in this run.

## Inputs

- Sales Nav messaging inbox + LinkedIn inbox (via Chrome MCP).
- Leads sheet (to match inbound threads to known leads).
- Tasks sheet (to find pending tasks to cancel).
- The last time this skill ran today (from a `last_detect_replies_at`
  marker in the workbook's metadata area, or derived from the most
  recent Replies row).

## Behavior

1. **Open the Sales Nav messaging inbox** via Chrome MCP. If Sales
   Nav inbox is not available to the client, fall back to the
   LinkedIn messaging inbox at `https://www.linkedin.com/messaging/`.
2. **Scan unread threads + threads with a new inbound since
   last-run.**
3. **For each thread with a new inbound message from the lead:**
   a. **Identify the lead.** Use the thread's counterparty profile
      URL. Match against Leads (primary key `lead_url`).
      - No match → log as "personal inbox message, not a campaign
        lead" and skip.
      - Multiple matches (shouldn't happen — leads are unique
        across active campaigns) → log to `_errors.md` and skip.
   b. **Capture the reply.** Append a row to Replies:
      - `date` = inbound timestamp (ISO)
      - `lead_url`
      - `campaign_slug`
      - `raw_text` = the full inbound message text
      - `classification` = `unclassified` for now; the downstream
        reply skill will classify during draft.
        (Classifications used later: `positive`, `ask-for-info`,
        `not-interested`, `out-of-office`, `booked`. `booked` is set
        by `mark-booked`, not here.)
      - `handoff_state` = `handed_to_reply`
   c. **Cancel pending sequence tasks.** For this lead, find every
      Tasks row where `campaign_slug` matches, `status = pending`,
      and `type IN (message-step-1, message-step-2, message-step-3,
      follow-up, book-meeting)`. Flip `status = cancelled`,
      `completed_at = now`, `cancel_reason = reply_received`.
      Leave `connect` tasks alone (the lead's already connected if
      we're getting replies).
   d. **Merge into an existing review-reply, or create a new one.**
      - If a `review-reply` task already exists for this lead that
        is `status = pending`: leave it alone. The reply append
        above will be picked up when the client opens the existing
        thread in `rockstarr-reply`.
      - Else create a new Tasks row: `type = review-reply`,
        `status = pending`, `due_date = today`, metadata pointing to
        the Replies row id.
   e. **Flip Leads.state.** If `state` is currently `connected` or
      `accepted`, move to `replied`, set `date_replied = today`.
      If `state = booked`, leave it alone (a post-booking message
      is still useful context but doesn't reset the state).
4. **Hand off to rockstarr-reply.** For every new `review-reply`
   task created in this run, signal `rockstarr-reply:draft-reply`
   with the thread context (lead_url, campaign_slug, reply-row id).
   If `rockstarr-reply` is not yet installed in the client's
   workspace, write a Cowork notification: "Reply from `<name>` in
   `<campaign_slug>` needs your attention — rockstarr-reply is not
   installed. Draft manually via Sales Nav." Log the fallback.
5. **Update the `last_detect_replies_at` marker** so the afternoon
   pass (if enabled) only picks up newer inbounds.
6. **Save the workbook.**

## Output

- `new_replies` — count
- `threads_handed_off` — count of `review-reply` tasks newly created
- `already_in_review` — count of inbound that merged into an existing
  review thread
- `non_campaign_inbound` — count (logged, not acted on)

## What NOT to do

- Do not draft the reply in this skill. `rockstarr-reply:draft-reply`
  is the reply surface; this skill only detects + routes.
- Do not classify as `booked` here. `mark-booked` is the single
  source of truth for booking state.
- Do not send any message from this skill. Not ever.
- Do not cancel `connect` tasks — a replied lead was already
  connected; the connect task is done, not pending.
- Do not mark `review-reply` as done. That happens when
  `rockstarr-reply:approve` fires downstream.
