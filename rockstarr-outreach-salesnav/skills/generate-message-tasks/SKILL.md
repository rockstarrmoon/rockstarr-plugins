---
name: generate-message-tasks
description: "This skill should be used right after detect-accepts, or when the user says \"seed the sequence\", \"generate message tasks for newly accepted leads\", or \"schedule the follow-ups for this lead\". For each lead that just moved to state=accepted in a full-sequence campaign, reads the approved 3-step sequence and writes message-step-1/2/3 tasks with due-dates at day-of-accept, accept+3, accept+7. Refuses to seed for connect-only campaigns (no post-accept sequence by design). Idempotent."
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

1. **Look up the campaign's `campaign_type`** from the matching
   Campaigns row.
   - If `connect-only`: REFUSE to seed message tasks for this lead.
     Append a row to `_errors.md` with the lead, the campaign, and
     a note: "generate-message-tasks invoked on connect-only
     campaign — no message sequence by design; caller should not
     have routed this lead here. Check detect-accepts'
     full_sequence/connect_only partition." Continue to the next
     lead. The accept on a connect-only lead is the terminal
     state; the lead simply sits at `state=accepted` and feeds the
     accept-rate metric. This is belt-and-suspenders defense —
     `detect-accepts` should not call this skill for connect-only
     leads, but if a programming error or an out-of-band invocation
     gets us here, we fail safe.
   - If `full-sequence` (or absent — default for back-compat): proceed.
2. **Read the approved campaign spec** at
   `04_approved/outreach/campaign-[slug].md`. Extract the bodies for
   Message 2 (day of accept), Message 3 (accept + 3), Message 4
   (accept + 7). Message 1 is the connect request and is
   intentionally BLANK — it does not map to a `message-step-N` task.
3. **Check idempotency.** If the lead already has Tasks rows of type
   `message-step-1`, `message-step-2`, `message-step-3` for this
   campaign (regardless of status), skip. Never create duplicates.
4. **Create three tasks.** Append to Tasks (step numbers refer to the
   post-connect sequence — step-1 is Message 2 in the campaign file,
   step-2 is Message 3, step-3 is Message 4):
   - `message-step-1` (campaign Message 2), due = `accepted_on`
   - `message-step-2` (campaign Message 3), due = `accepted_on + 3 business days`
   - `message-step-3` (campaign Message 4), due = `accepted_on + 7 business days`
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
5. **Save.**

## "Business days" rule

V0.1: skip Saturdays and Sundays when computing `+3` and `+7`. If the
client's timezone is configured in `stack.md` such that the weekend
falls differently, respect that. No holiday calendar in V0.1 —
holidays surface as a "why did this go out on a holiday" weekly-
report question, and the client can pause the campaign if it
matters.

## Output

Return `{created: N, skipped_already_seeded: M, refused_connect_only: K}`
to the caller. `refused_connect_only` should normally be 0 — it
counts the belt-and-suspenders refusals from Step 1, which only
fire if `detect-accepts`'s partition is broken or this skill was
invoked manually on a connect-only lead.

## What NOT to do

- Do not send messages from this skill. Scheduling only.
- Do not paraphrase the approved sequence copy. Bodies live in the
  approved campaign file; `send-scheduled-messages` reads them at
  send time.
- Do not create tasks for leads in `state != accepted`.
- Do not create tasks for leads in connect-only campaigns. Step 1's
  refusal is the gate. The accept on a connect-only lead is the
  terminal state by design — there are no Messages 2/3/4 in the
  approved campaign spec to seed from.
- Do not re-create tasks after `detect-replies` cancelled them.
  Cancelled is cancelled — a subsequent accept cannot happen for the
  same lead/campaign because the reply already moved the lead off
  the sequence rails.
