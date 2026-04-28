---
name: create-followup-task
description: "This skill should be used after apply-label completes in the per-reply pipeline, or when the user says \"create a follow-up task\", \"schedule the bump\", or \"put this lead on a three-day timer\". Takes the follow-up-timer keyword proposed by rockstarr-reply:follow-up-timer (meeting_proposed / general / referral / cold_bump / none) and converts it to an Interceptly task with Interceptly's business-days math + Friday→Monday shift for meeting_proposed. Applies stack.md.followup_timers overrides. Mirrors the task into the Tasks sheet of outreach-mirror.xlsx. The `none` keyword is a no-op; Ignore / Not Interested / Bad Fit never get tasks."
---

# create-followup-task

Schedules the bot's next touch on the thread. Interceptly holds
the authoritative task; the mirror keeps a copy for client
visibility.

rockstarr-reply proposes; this skill disposes. rockstarr-reply's
`follow-up-timer` skill returns a keyword (`meeting_proposed` /
`general` / `referral` / `cold_bump` / `none`). This skill takes
the keyword, applies `stack.md.followup_timers` overrides, runs
the business-days math, and creates the Interceptly task.

## When to run

- Immediately after `apply-label` returns success, for labels
  that map to a timer (everything EXCEPT Ignore, Not Interested,
  and Bad Fit — those are terminal).
- After `process-inbox` / `process-my-tasks` receives an
  `authorized-send` bundle from rockstarr-reply with a
  non-`none` `proposed_followup_timer` keyword.
- Called by `mark-booked-interceptly` to create a meeting-reminder task if
  the client wants one (optional; off by default).
- On demand when the user says "create a bump task for this
  lead" — validates keyword and reason are known.

## Preconditions

- Thread is open in Chrome.
- `confirm-session-interceptly` passed in this run.
- `apply-label` just completed successfully.
- The applied label is NOT `Ignore` or `Not Interested`. If it
  is, refuse with a clear no-op message — these are terminal.

## Inputs

- `thread_id` — Interceptly thread reference.
- `reason` — the follow-up-timer keyword from
  `rockstarr-reply:follow-up-timer`: one of `meeting_proposed`,
  `general`, `referral`, `cold_bump`, `none`. Plus the two
  channel-side values this plugin generates itself:
  `book-meeting-interceptly` (from the `book-meeting-handoff` bundle) and
  `custom` (for `mark-booked-interceptly`-initiated meeting reminders).
- `days` — optional override; if absent, this skill reads
  `stack.md.followup_timers[reason]` and falls back to the
  default day count below.

If `reason` is `none`, refuse with a no-op message. This matches
rockstarr-reply's contract: `none` means "the keyword-to-task
contract declines this one."

## Behavior

### Step 1 — Resolve the timer

Precedence:

1. Explicit `days` input — use it verbatim.
2. Else `stack.md.followup_timers[reason]` — client override.
3. Else the default day count below.

| reason (keyword) | default business days |
|---|---|
| meeting_proposed | 2 |
| general | 3 |
| referral | 5 |
| cold_bump | 5 |
| flagged_review | 2 (only when caller is the flag branch) |
| book-meeting | 0 (same-day) |

If the reason is `none`, refuse. If the reason is not in the map
above and not in `stack.md.followup_timers`, fall back to
`general` and log `fallback_reason: <original>` into the mirror.

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

When rockstarr-reply returns a `book-meeting-handoff` bundle,
`process-inbox` / `process-my-tasks` routes directly to
`book-meeting-interceptly` — NOT through this skill. This skill is only
involved if the pipeline wants a scheduled book-meeting task
(rare; used when the lead agreed to a slot but supplied fields
that need operator confirmation before the form runs).

Inputs in that case:

- reason = `book-meeting-interceptly`
- days = 0 (same-day)
- title = `Book meeting: <lead name>` + proposed time

The `book-meeting-interceptly` skill polls for these tasks (or is called
directly by the pipeline on the same run).

## Which situations get no task

- `Ignore` label applied → no task.
- `Not Interested` label applied → no task.
- `Bad Fit` label applied → no task (graceful exits, pitch-back
  declines — all terminal).
- `Booked` label applied → no BUMP task. `mark-booked-interceptly` may
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
