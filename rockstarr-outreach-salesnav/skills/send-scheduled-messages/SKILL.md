---
name: send-scheduled-messages
description: "This skill should be used in the daily outreach loop after generate-message-tasks, or when the user says \"send today's scheduled Sales Nav messages\", \"execute the messages due today\", or \"run the sequence sends\". It executes every message-step-N task and follow-up task due today by sending the approved body via Sales Nav messaging through Chrome MCP, logs each send to the Messages sheet, and marks the task done. Refuses to run if confirm-session has not passed in this run."
---

# send-scheduled-messages

Sends the sequence bodies that `generate-message-tasks` queued and the
2-day follow-ups that `send-approved-reply` queued. Bodies are read
from the approved campaign file at send time (or drafted fresh via
`rockstarr-reply` for follow-ups).

## When to run

- Daily loop, after `generate-message-tasks`, before `detect-replies`.
- On-demand via `force-send-today` (deferred V0.1).

## Preconditions

- `confirm-session` passed in this run.

## Inputs

- Tasks rows where `status = pending`, `due_date <= today`, and
  `type IN (message-step-1, message-step-2, message-step-3,
  follow-up)`.

## Behavior

1. **Sort due tasks.** Oldest `due_date` first, within a `due_date`
   group sort by `created_at`.
2. **For each task:**
   a. **Resolve body.**
      - `message-step-N`: read the approved campaign body at
        `04_approved/outreach/campaign-<slug>.md`, extract the
        matching step's copy, apply minimal variable substitution
        (`{first_name}`, `{company}`) from Leads.
      - `follow-up`: call `rockstarr-reply:draft-reply` with the
        thread context — follow-up bodies draft fresh on the due-
        date, not at send-time, because the context can drift. If
        `rockstarr-reply:draft-reply` is not yet available, fall back
        to a Cowork notification to the client asking them to draft
        manually. Log the fallback to `_errors.md`.
      - If the draft-reply path is in use, the follow-up body must
        be approved by the client through `rockstarr-reply` before
        this skill sends. Do **not** send an unapproved follow-up.
   b. **Open the Sales Nav thread** for this lead via Chrome MCP.
      Navigate to `https://www.linkedin.com/sales/messaging/` and
      find the thread (or open via the lead's profile panel). If
      the thread does not exist (lead removed the connection, etc.):
      log to `_errors.md`, mark the task `cancelled` with reason
      `thread_missing`, flip Leads.state to `opted-out`, cancel
      remaining sequence tasks for this lead.
   c. **Send the body.** Paste into the message composer, submit.
      Verify the message appears in the thread after send.
   d. **Write to Messages.** Row per send:
      - `date` = now (ISO timestamp)
      - `lead_url`
      - `campaign_slug`
      - `step` = `message-step-1|2|3` or `follow-up`
      - `body_sent` = exact body submitted (post-substitution)
   e. **Mark the task done.** `status = done`, `completed_at = now`.
   f. **Polite rate.** 25–45 seconds random wait between sends.
3. **Save the workbook.**
4. **Log to publish-log.** Append a per-campaign summary line to
   `/05_published/outreach/<today>.md`:
   `send-scheduled-messages — <N> sends across <M> campaigns
   (<slug-a>: <n_a>, <slug-b>: <n_b>)`.

## Output

- `sent` — total messages sent
- `per_campaign` — slug → count
- `skipped_thread_missing` — count
- `skipped_unapproved_followup` — count (waiting on rockstarr-reply
  approval)

## What NOT to do

- Do not send a `follow-up` before its body has been approved
  through `rockstarr-reply`. Per-reply gate is non-negotiable.
- Do not rewrite the sequence body. Variable substitution only
  (`{first_name}`, `{company}`). If the body reads awkwardly for a
  specific lead, skip + log; the client can edit the approved
  campaign and re-register.
- Do not send outside the configured run time window. This skill
  expects to be invoked by the daily loop at the client's
  `outreach_daily_run_time`.
- Do not continue the sequence after a reply. `detect-replies`
  cancels future `message-step-N` tasks for that lead; trust it.
