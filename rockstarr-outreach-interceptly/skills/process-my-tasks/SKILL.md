---
name: process-my-tasks
description: "This skill should be used in the daily outreach loop after process-inbox completes for the currently-active managed account, or when the user says \"process my tasks\", \"run the Interceptly My Tasks pass\", or \"handle overdue + due-today for <account>\". Opens Interceptly → My Tasks, sorts by Due date oldest-first, and for each overdue or due-today task runs the same per-reply pipeline as process-inbox (qualify-lead → classify-reply → draft-reply-interceptly → present-for-approval → send-message → apply-label → create-followup-task). Walks every page of tasks — the bot does not stop after page 1."
---

# process-my-tasks

The second half of the per-account pass. Runs ONLY after
`process-inbox` finishes for this account. Same per-reply
pipeline; the difference is the entry point (task due-date instead
of an unread reply).

## When to run

- Step 3b of the daily loop, for the currently-active managed
  account, after `process-inbox` returns zero unreads remaining.
- On demand when the user says "process my tasks", "run overdue +
  due-today for `<account>`", or "the task queue is piling up."

## Preconditions

- `process-inbox` has returned clean for the current account.
  If unreads remain, refuse — unreads come first, always.
- `confirm-session-interceptly` last result in this run is
  `pass`. If a `fail` has been logged since, abort.

## Behavior

### Step 1 — Open My Tasks

Navigate Chrome MCP to Interceptly → My Tasks. Apply filter:
Due date ≤ today. Sort ascending.

### Step 2 — Walk overdue + due-today

For each task row, oldest-first:

1. Read `lead_identifier`, `task_type`, `due_date`, `notes`
   from the row.
2. Open the lead's thread (click the row; Interceptly routes
   task rows to the underlying thread).
3. Run the per-reply pipeline (same as `process-inbox` Step 3):
   `qualify-lead` → `classify-reply` → `draft-reply-interceptly`
   → `present-for-approval` → on 'send it' →
   `send-message` → `apply-label` → `create-followup-task`.
4. If the lead's thread has a new inbound reply since the task
   was created (an unread that `process-inbox` already processed
   OR a race-condition reply that landed between inbox and
   tasks), the pipeline re-qualifies and the state transitions
   reflect the current thread state. Do not draft against a
   stale task premise.
5. After the pipeline returns, mark the task done in Interceptly
   (click the task's complete checkbox) and mirror the
   completion into the `Tasks` sheet.

### Step 3 — Pagination

Interceptly's My Tasks list paginates. Walk beyond page 1 — the
bot does NOT stop after the first page. Stop only when the filter
(`Due date ≤ today`) returns zero rows.

### Step 4 — Handoff

When no overdue or due-today tasks remain for this account,
return control to the daily loop. The loop calls
`switch-account` to move to the next managed account.

## Task types the bot handles here

- `follow-up` — a reply is stale; the bot drafts a bump (cold
  or warm depending on `classify-reply`).
- `review-reply` — a thread was flagged earlier and the flag
  was cleared; the pipeline now drafts normally.
- `book-meeting` — lead agreed to a slot and supplied required
  fields; this task is handled by `book-meeting`, NOT by the
  reply pipeline. Route to `book-meeting` directly from here
  rather than drafting a reply.

## Constraints

- Overdue before due-today, within the Due-date sort.
- The daily loop never moves to another account while this
  skill still has overdue or due-today tasks on this account.
- Afternoon re-runs of the daily loop do NOT re-run this skill.
  Only the morning run processes tasks. Afternoon is inbox-only
  (per the schedule spec).

## Failure modes

- **Task row routes to an empty thread** (lead deleted from
  Interceptly). Skip; mark task done with a note in the mirror.
  Write a quiet `_errors.md` note for the operator.
- **Task's underlying lead is now `not_target`.** Let the
  pipeline run — `qualify-lead` will return the verdict and
  the non-ICP path applies. Mark the task done after the
  pipeline handles the thread.
- **Pagination breaks.** Interceptly has shipped pagination
  regressions. If clicking "Next page" produces no new rows,
  retry once with a reload. On second failure, write _errors.md
  and stop — better to miss page 2 and surface the issue than
  to fake success.

## What NOT to do

- Do not start this skill while `process-inbox` has unreads
  remaining.
- Do not process future-dated tasks. Only overdue + due-today.
- Do not bypass the per-reply pipeline. Even a `follow-up`
  task's bump reply runs through `qualify-lead` first — the
  lead's situation may have changed since the task was created.
- Do not cross accounts from this skill. Switching is a
  separate step (`switch-account`) run by the daily loop.
