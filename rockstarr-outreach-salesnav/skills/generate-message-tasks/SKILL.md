---
name: generate-message-tasks
description: "This skill should be used right after detect-accepts, or when the user says \"seed the sequence\", \"generate message tasks for newly accepted leads\", or \"schedule the follow-ups for this lead\". For each lead that just moved to state=accepted, it reads the approved campaign's 3-step sequence and writes message-step-1, message-step-2, and message-step-3 tasks into the Tasks sheet with due-dates at day-of-accept, accept+3, and accept+7 respectively. Idempotent — re-running on the same lead does not duplicate tasks."
---

# generate-message-tasks

Turns an "accepted" signal into a concrete three-row plan the
`send-scheduled-messages` skill will execute on schedule.

## When to run

- Immediately after `detect-accepts` produces new accepts.
- On-demand when the user says "seed the sequence for Jane" (e.g.,
  if a lead's `date_accepted` was set manually and tasks were never
  generated).

## Inputs

- List of `(lead_url, campaign_slug, accepted_on)` from the caller,
  OR the Leads sheet filtered to `state = accepted` with no pending
  `message-step-*` tasks.

## Behavior

For each lead:

1. **Read the approved campaign spec** at
   `04_approved/outreach/campaign-<slug>.md`. Extract the bodies for
   Step 2 (day of accept), Step 3 (accept + 3), Step 4 (accept + 7).
2. **Check idempotency.** If the lead already has Tasks rows of type
   `message-step-1`, `message-step-2`, `message-step-3` for this
   campaign (regardless of status), skip. Never create duplicates.
3. **Create three tasks.** Append to Tasks:
   - `message-step-1`, due = `accepted_on`
   - `message-step-2`, due = `accepted_on + 3 business days`
   - `message-step-3`, due = `accepted_on + 7 business days`
   Each row:
   - `task_id` = generated
   - `lead_url` = the lead's URL
   - `campaign_slug` = the slug
   - `due_date` = as above (ISO date, no time)
   - `status` = `pending`
   - `created_at` = now
   - `completed_at` = empty
   - `metadata.body_ref` = pointer to the campaign file + step section
     so `send-scheduled-messages` reads the canonical copy rather
     than a cached snapshot (if the client edits an approved
     campaign, they must re-register; see `register-campaign`).
4. **Save.**

## "Business days" rule

V0.1: skip Saturdays and Sundays when computing `+3` and `+7`. If the
client's timezone is configured in `stack.md` such that the weekend
falls differently, respect that. No holiday calendar in V0.1 —
holidays surface as a "why did this go out on a holiday" weekly-
report question, and the client can pause the campaign if it
matters.

## Output

Return `{created: N, skipped_already_seeded: M}` to the caller.

## What NOT to do

- Do not send messages from this skill. Scheduling only.
- Do not paraphrase the approved sequence copy. Bodies live in the
  approved campaign file; `send-scheduled-messages` reads them at
  send time.
- Do not create tasks for leads in `state != accepted`.
- Do not re-create tasks after `detect-replies` cancelled them.
  Cancelled is cancelled — a subsequent accept cannot happen for the
  same lead/campaign because the reply already moved the lead off
  the sequence rails.
