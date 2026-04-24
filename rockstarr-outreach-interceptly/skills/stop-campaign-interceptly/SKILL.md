---
name: stop-campaign-interceptly
description: "This skill should be used when the user says \"pause this campaign\", \"stop the campaign\", \"kill this Interceptly campaign\", or names a campaign to wind down. Pauses or stops the campaign inside Interceptly via Chrome MCP and mirrors the state change into the Campaigns sheet of outreach-mirror.xlsx. Paused is resumable; stopped is permanent. rockstarr-reply reads the Campaigns row status and cancels its own pending reply-side tasks — this plugin does NOT reach into those tables."
---

# stop-campaign-interceptly

Wind-down path. Runs inside Interceptly and then mirrors the state
change into the local audit workbook. This plugin is only
responsible for Interceptly state + the Campaigns row.
Reply-side task cancellation (review-reply, follow-up,
book-meeting) is handled by `rockstarr-reply`, which reads the
Campaigns row status as its cancel signal.

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

### Step 4 — Update Campaigns row

Flip the `Campaigns` row of
`/02_inputs/outreach/outreach-mirror.xlsx`:

- `pause` → `status = paused`, `paused_at = now`
- `stop` → `status = stopped`, `stopped_at = now`

This is the signal `rockstarr-reply` reads on its next daily pass
to cancel its own pending review-reply, follow-up, and
book-meeting tasks for the campaign's leads. This plugin does NOT
reach into those tables.

### Step 5 — Log

Append to `/05_published/outreach/<today>.md`:

> `stop-campaign-interceptly — <slug> <mode>d under
> <account_label>. Interceptly state + Campaigns row updated.
> rockstarr-reply will cancel dependent tasks on its next pass.`

### Step 6 — Return a short summary

Tell the user: slug, mode, which accounts were touched, and a
one-line reminder that `rockstarr-reply` will pick up the status
change and cancel its dependent tasks on its next daily pass.

## Semantics

- `pause` is RESUMABLE. The client can press Resume inside
  Interceptly — the bot does not need to do anything.
  `rockstarr-reply` re-seeds dependent tasks the next time a
  reply or accept lands on a lead in this campaign.
- `stop` is PERMANENT. Interceptly marks the campaign ended;
  resuming requires a new Interceptly campaign (via
  `launch-campaign-interceptly` against a re-approved spec).

## Failure modes

- **UI refuses to pause / stop** (e.g., Interceptly bug). Retry
  once after a 2-3s wait. If still failing, write to
  `_errors.md` and tell the user to do it manually — but still
  update the Campaigns row status so both this plugin and
  `rockstarr-reply` stop scheduling work.
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
- Do not touch reply-side tasks (review-reply, follow-up,
  book-meeting). Those rows belong to `rockstarr-reply` and that
  plugin cancels them based on the Campaigns row status.
- Do not resume a paused campaign from this skill. Resume is a
  human action inside Interceptly.
