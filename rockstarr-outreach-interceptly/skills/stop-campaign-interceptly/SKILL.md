---
name: stop-campaign-interceptly
description: "This skill should be used when the user says \"pause this campaign\", \"stop the <slug> campaign\", \"kill this Interceptly campaign\", or names a campaign to wind down. Pauses or stops the campaign inside Interceptly via Chrome MCP, mirrors the state change into the Campaigns sheet, and cancels pending follow-up tasks for that campaign's leads. Paused is resumable; stopped is permanent."
---

# stop-campaign-interceptly

Wind-down path. Runs inside Interceptly and then mirrors the state
change into the local audit workbook.

## When to run

- User says "pause <slug>", "stop <slug>", "kill <slug>", or
  points at a running campaign and asks for it to stop.
- Rare: an upstream skill escalates (e.g., Interceptly reports a
  TOS warning for a campaign).

## Preconditions

- `/02_inputs/outreach/outreach-mirror.xlsx` has a `Campaigns` row
  for the slug with `status` in (`configured`, `running`,
  `paused`). If the row is already `stopped`, refuse with a
  no-op message.
- Chrome MCP is available and `confirm-session-interceptly`
  passed for the account the campaign runs under.

## Inputs

- `slug` — campaign slug (required).
- `mode` — `pause` or `stop` (required). Use `AskUserQuestion` if
  the user didn't say which.

## Behavior

### Step 1 — Resolve scope

If scope is `all_accounts`, the slug maps to multiple Interceptly
campaign ids (one per managed account). Iterate per account,
switching + confirming session between. If scope is a specific
account, operate once against that account's Interceptly campaign
id.

### Step 2 — Find the campaign in Interceptly

Navigate to the Campaigns view. Find the row matching the
Interceptly campaign id from the mirror (not just the slug — the
id is the primary key here; different accounts may label their
campaigns with the same local slug).

### Step 3 — Apply the state change

- `pause` → click the campaign's Pause control. Confirm the UI
  shows paused.
- `stop` → click the campaign's Stop / End control. Confirm the UI
  shows stopped. Interceptly may prompt for confirmation —
  acknowledge, do not cancel.

### Step 4 — Cancel pending follow-up tasks

Open the Tasks sheet of `outreach-mirror.xlsx`. For every row
where `campaign_slug = <slug>` AND `status = pending` AND
`task_type` in (`follow-up`, `review-reply`, `book-meeting`),
flip:

- `status = cancelled`
- `completed_at = now`
- `cancel_reason = campaign_<mode>`

Do NOT cancel `message-step-N` or `connect` tasks — those are
Interceptly-driven and the Interceptly state change already
handles them. The mirror cancellation is for tasks the bot owns.

### Step 5 — Update Campaigns row

Flip the `Campaigns` row:

- `pause` → `status = paused`, `paused_at = now`
- `stop` → `status = stopped`, `stopped_at = now`

### Step 6 — Log

Append to `/05_published/outreach/<today>.md`:

> `stop-campaign-interceptly — <slug> <mode>d under
> <account_label>. <N> pending tasks cancelled.`

### Step 7 — Return a short summary

Tell the user: slug, mode, which accounts were touched, how many
pending tasks were cancelled.

## Semantics

- `pause` is RESUMABLE. The client can press Resume inside
  Interceptly — the bot does not need to do anything. Pending
  tasks stay cancelled; they re-seed from the next accept / reply.
- `stop` is PERMANENT. Interceptly marks the campaign ended;
  resuming requires a new Interceptly campaign (via
  `launch-campaign-interceptly` against a re-approved spec).

## Failure modes

- **UI refuses to pause / stop** (e.g., Interceptly bug). Retry
  once after a 2-3s wait. If still failing, write to
  `_errors.md` and tell the user to do it manually — but still
  update the mirror so the bot stops scheduling tasks.
- **Campaigns row already stopped.** No-op. Return a clear "already
  stopped" message.
- **Slug matches multiple rows** (`all_accounts` scope). Prompt the
  user whether to pause/stop all of them or just one specific
  account's instance.

## What NOT to do

- Do not delete the `Campaigns` row. History matters.
- Do not cancel `message-step-N` Interceptly-driven tasks from
  the mirror — they aren't real tasks, they're Interceptly's own
  sequencing.
- Do not resume a paused campaign from this skill. Resume is a
  human action inside Interceptly.
