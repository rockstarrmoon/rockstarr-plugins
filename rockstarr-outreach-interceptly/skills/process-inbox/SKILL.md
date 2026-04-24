---
name: process-inbox
description: "This skill should be used in the daily outreach loop after preview-queue, or when the user says \"process the inbox\", \"walk unread replies\", or \"handle today's Interceptly inbox for <account>\". Filters Interceptly Inbox → Replied, walks unread conversations oldest-first, and for each thread runs the full per-reply pipeline: qualify-lead → classify-reply → draft-reply-interceptly → present-for-approval → (on 'send it') send-message → apply-label → create-followup-task. Unreads come before My Tasks on every account. Refuses to run if confirm-session-interceptly has not passed in this run."
---

# process-inbox

The first half of the per-account pass. Every unread reply runs
through the full pipeline before `process-my-tasks` takes over
for this account.

## When to run

- Step 3a of the daily loop, for the currently-active managed
  account, after `confirm-session-interceptly` passes and
  `preview-queue` (if enabled) has written its file.
- On demand when the user says "just process the inbox" or "run
  the inbox pass for `<account>`."

## Preconditions

- `confirm-session-interceptly` has passed in this run for the
  currently-active account. If the last confirm result was `fail`,
  REFUSE — the whole pass is aborted.
- `/00_intake/interceptly-accounts.md` has a row for the current
  account.
- `/00_intake/icp-qualifications.md` exists.
- `/00_intake/style-guide.md` exists and has an approved voice.

## Behavior

### Step 1 — Navigate to inbox

Navigate Chrome MCP to Interceptly → Inbox. Apply the filter
`Replied` (leads who have replied to any touch). Sort by unread
state ascending, then by date ascending — unread + oldest first.

### Step 2 — Walk unreads oldest-first

For each unread thread:

1. **Open the thread.** Click the conversation row. Wait for the
   right-panel to render (lead profile + thread body).
2. **Read the right-panel context.** Lead name, company, title,
   Work Experience. If Work Experience is collapsed, scroll to
   expand it. This feeds `qualify-lead`.
3. **Read the thread body.** Capture every inbound and outbound
   message since the last thread open (the new-since-read
   portion is the focus; older context is useful for
   classification).

### Step 3 — Per-reply pipeline

For each thread, in order:

1. Call `qualify-lead` with the right-panel context and any
   active campaign's per-campaign ICP overrides. Record the
   verdict (`target` / `not_target` / `ambiguous` / `unknown`).
2. If verdict = `ambiguous`, call `flag-for-review` and move to
   the next thread. Do NOT draft.
3. If verdict = `not_target`:
   - Call `classify-reply` to get temperature (`hot` / `warm` /
     `cold` / `decline`).
   - If temperature is `warm` AND the reply is a polite "yes"
     (not an explicit decline), this is the non-ICP yes case.
     Call `draft-reply-interceptly` which will present the three
     options (graceful exit / let-it-hang / throwaway question)
     via `present-for-approval`. On the approver's choice, the
     appropriate path runs (send + label / no-send + label Ignore
     / send + label Follow Up).
   - Otherwise, skip drafting. Label per mapping (`decline` →
     Not Interested; polite ack → Ignore). Record in the Non-ICP
     Log (`/02_inputs/outreach/_non_icp_log.md`).
4. If verdict = `target`:
   - Call `classify-reply` to get temperature.
   - Call `draft-reply-interceptly` with verdict + temperature +
     current account's persona + style-guide. Every output runs
     through `stop-slop`.
   - Call `present-for-approval`. Block on 'send it' or clear
     equivalent. Editing instructions never authorize.
   - On 'send it', call `send-message` → then
     `apply-label` → then `create-followup-task`.
   - On 'flag', call `flag-for-review`. No send.
   - On 'skip', move to next thread without sending, labeling,
     or task-ing.
5. If verdict = `unknown` (the bot genuinely cannot tell even
   with LinkedIn fallback), call `flag-for-review`.

### Step 4 — Move to next thread

Do NOT navigate away from the inbox until all unreads are
processed. Interceptly sometimes re-renders the list after a
label click — re-read the list and continue at the next unread
that is NOT the one just processed.

### Step 5 — Handoff

When the unread count reaches zero (read from Interceptly, not
from the mirror), return control to the daily loop. The loop
then runs `process-my-tasks` against this same account BEFORE
switching to the next account.

## Constraints

- Unreads ALWAYS come before My Tasks on any given account.
  Do not interleave.
- Never move to another account while this account has ANY
  unread in the Replied filter.
- The send gate is strict. Only 'send it' or a clear equivalent
  sends a message. Editing instructions — 'make it shorter',
  'try a different angle', 'next', 'continue' — are draft-and-
  present, NEVER authorization.

## Failure modes

- **Chrome MCP drops mid-thread.** Retry the current thread once.
  On second failure, leave the thread unread, write an _errors.md
  block, and continue to the next thread.
- **A thread has no visible right-panel context.** Open the
  lead's LinkedIn profile in a read-only side tab via Chrome MCP
  to gather context for `qualify-lead`. Do not send from that tab.
- **`send-message` fails** after 'send it' was authorized. Retry
  once. On second failure, leave the thread unread, write
  _errors.md, and move on. The approver can retry on the next
  daily pass. Do NOT label or create a task if the send failed.
- **`apply-label` fails** after a successful send. Retry once.
  On failure, write _errors.md with the specific thread — the
  operator needs to apply the label manually. The task still
  gets created (follow-up timer is independent of label state).
- **The list re-renders into Campaigns view** (known Labels nav
  bug). Navigate back to Inbox → Replied filter and continue.

## What NOT to do

- Do not process My Tasks until this skill finishes.
- Do not switch accounts until this skill and
  `process-my-tasks` both finish.
- Do not guess approval. Silence is not consent. When in doubt,
  present and wait.
- Do not skip `stop-slop`. Every piece of prose runs through it.
- Do not paste the booking link into any reply. The booking link
  is a destination, not content.
