---
name: create-followup-task
description: "This skill should be used after apply-label completes in the per-reply pipeline, or when the user says \"create a follow-up task for <lead>\", \"schedule the bump\", or \"put <lead> on a 3-day timer\". Creates a task in Interceptly's Tasks tab with the correct timer from stack.md.followup_timers (defaults 2 / 3 / 5 / 5 business days for meeting_proposed / general / referral / cold_bump; Friday→Monday shift on meeting_proposed), and mirrors the task into the Tasks sheet of outreach-mirror.xlsx. Non-ICP Ignore and Not Interested do NOT get tasks — this skill refuses those paths."
---

# create-followup-task

Schedules the bot's next touch on the thread. Interceptly holds
the authoritative task; the mirror keeps a copy for client
visibility.

## When to run

- Immediately after `apply-label` returns success, for labels
  that map to a timer (everything EXCEPT Ignore and Not
  Interested — those are terminal).
- Called by `mark-booked` to create a meeting-reminder task if
  the client wants one (optional; off by default).
- On demand when the user says "create a bump task for
  `<lead>`" — validates timer and reason are known.

## Preconditions

- Thread is open in Chrome.
- `confirm-session-interceptly` passed in this run.
- `apply-label` just completed successfully.
- The applied label is NOT `Ignore` or `Not Interested`. If it
  is, refuse with a clear no-op message — these are terminal.

## Inputs

- `thread_id` — Interceptly thread reference.
- `reason` — one of: `meeting_proposed`, `general`, `referral`,
  `cold_bump`, `book-meeting` (if routing to book-meeting),
  `custom` (for `mark-booked`-initiated meeting reminders).
- `days` — optional override; defaults to the
  `stack.md.followup_timers` value for the reason.

## Behavior

### Step 1 — Resolve the timer

Read `stack.md.followup_timers`. Defaults:

| reason | business days |
|---|---|
| meeting_proposed | 2 |
| general | 3 |
| referral | 5 |
| cold_bump | 5 |

If the reason is not in the map, use `general` as fallback.

### Step 2 — Business-day math + Friday→Monday shift

Compute the due date = today + `days` business days (skip
weekends).

Special case — `meeting_proposed`: if the resulting due date
falls on a Friday, shift to the following Monday. Rationale:
meetings proposed on a Friday reminder often get lost over the
weekend; Monday surfaces them fresh for the operator.

### Step 3 — Open Tasks tab on the thread

From the open thread view, click the Tasks tab. Known UI quirk:
this tab sometimes shows an empty state even when tasks exist
— wait 2s before concluding no tasks are present.

Click `Create task` (text match).

### Step 4 — Fill the task form

- Title: a short human-readable string, e.g.,
  `Bump: <reason> — <lead name>`. Keep under 80 chars.
- Due date: use the Interceptly date picker.
  **Known quirk:** the Same-day dropdown is labeled oddly in
  some revisions; avoid it by always explicitly picking the
  calendar date.
- Description: include the reason, the pipeline verdict +
  bucket, and a pointer to the Messages sheet row. This lets
  the operator understand why the task fired when it does.

### Step 5 — Save

Click Save. Wait for the task to appear in the thread's Tasks
tab list. On failure, retry once. On second failure, abort:
write _errors.md and tell the operator the label was applied
but the task wasn't created (so they can create it manually).

### Step 6 — Mirror

Append to the `Tasks` sheet of `outreach-mirror.xlsx`:

| column | value |
|---|---|
| task_id | `<Interceptly task id if readable; else blank>` |
| created_at | `<ISO>` |
| lead_url | `<URL>` |
| thread_id | `<id>` |
| campaign_slug | `<slug or blank>` |
| account_label | `<active managed account>` |
| reason | `<reason>` |
| due_date | `<YYYY-MM-DD>` |
| status | `pending` |
| title | `<Interceptly task title>` |
| source | `create-followup-task` |

### Step 7 — Return

`{created: true, due_date: "<YYYY-MM-DD>", task_id: "<id>"}`

## Special case — book-meeting task

When a `draft-reply-interceptly` Hot path determines the lead
has agreed to a time AND supplied all required fields,
`draft-reply-interceptly` asks THIS skill to create a
`book-meeting` task instead of scheduling a bump. Use:

- reason = `book-meeting`
- due_date = today (same-day)
- title = `Book meeting: <lead name>` + proposed time

The `book-meeting` skill polls for these tasks (or is called
directly by the pipeline on the same run).

## Which situations get no task

- `Ignore` label applied → no task.
- `Not Interested` label applied → no task.
- `Bad Fit` label applied → no task (graceful exits, pitch-back
  declines — all terminal).
- `Booked` label applied → no BUMP task. `mark-booked` may
  optionally create a meeting-reminder task if the client
  configures it in `stack.md`.

## Failure modes

- **Interceptly rejects the form** (missing field, bad date
  format). Inspect the form's validation output; if salvageable,
  correct and resave. If not, abort.
- **Task form closes without saving** (clicked outside). Re-open
  the thread, re-open Tasks tab, retry once.
- **Same-day dropdown defaults to an unexpected date.** Avoid
  the dropdown; use the full date picker.

## What NOT to do

- Do not create multiple tasks per thread per pipeline run. One
  label → one task (or zero, for terminal labels).
- Do not create tasks for Ignore / Not Interested / Bad Fit.
- Do not create a task with a past due date.
- Do not fabricate an Interceptly task id if you cannot read
  one. Leave the mirror's `task_id` column blank. The mirror
  is still useful without the Interceptly id.
