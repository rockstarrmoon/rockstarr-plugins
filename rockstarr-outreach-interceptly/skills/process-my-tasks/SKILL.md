---
name: process-my-tasks
description: "This skill should be used in the daily outreach loop after process-inbox completes for the currently-active managed account, or when the user says \"process my tasks\", \"run the Interceptly My Tasks pass\", or \"handle overdue and due-today tasks\". Opens Interceptly → My Tasks, sorts by Due date oldest-first, and for each overdue or due-today task runs qualify-lead locally, then hands the thread off to rockstarr-reply via the same channel-agnostic handoff bundle process-inbox uses. Has the same two modes process-inbox has. In foreground mode (default) it executes the returned bundle's channel-side work (send-message → apply-label → create-followup-task, or flag, or book-meeting-handoff). In background mode it stops after draft-reply stages a draft, accumulates staged paths, and returns them. Always returns staged_paths plus a per-task outcome list. book-meeting tasks always route directly to book-meeting in both modes — the close was authorized on a prior turn. Walks every page — the bot does not stop after page 1."
---

# process-my-tasks

The second half of the per-account pass. Runs ONLY after
`process-inbox` finishes for this account. Same handoff contract
to rockstarr-reply; the difference is the entry point (task
due-date instead of an unread reply).

## When to run

- Step 3b of the daily loop, for the currently-active managed
  account, after `process-inbox` returns zero unreads remaining.
- On demand when the user says "process my tasks", "run overdue
  + due-today for this account", or "the task queue is piling
  up."

## Inputs

- `mode` — `foreground` (default) or `background`. Same semantics
  as `process-inbox`. In background mode, draft-producing branches
  (`follow-up`, `review-reply`, `flagged_review`) stop after
  `draft-reply` stages a draft and accumulate the path; deterministic
  branches (label-only, flag) execute their channel-side work
  immediately. `book-meeting` tasks fire `book-meeting` directly in
  BOTH modes — the booking substance was authorized on a previous
  reply turn, so `book-meeting` is just execution.
- (Implicit) the currently-active managed account, established by
  the most recent `switch-account` call in this run.

## Preconditions

- `process-inbox` has returned clean for the current account.
  If unreads remain, refuse — unreads come first, always.
- `confirm-session-interceptly` last result in this run is
  `pass`. If a `fail` has been logged since, abort.
- `rockstarr-reply` is installed.

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
3. Branch on `task_type`:
   - `book-meeting` — route directly to `book-meeting` in BOTH
     modes. Do NOT call rockstarr-reply. The close is the booking,
     not a reply, and the substance was authorized on a prior turn.
   - `follow-up` / `review-reply` / `flagged_review` — run the same
     handoff as `process-inbox`:
     a. Read right-panel context + current thread body.
     b. `qualify-lead` locally.
     c. Build the handoff bundle (including `source_channel_label`
        per `process-inbox` Step 4). For `follow-up` tasks set
        `intent_hint` per the task metadata (typical: `bump` for
        cold_bump follow-ups; `referral-pivot` for referral
        follow-ups; `null` otherwise).
     d. Call rockstarr-reply. In foreground mode, the chain is
        classify-reply → draft-reply → present-for-approval. In
        background mode, the chain stops at draft-reply; the
        staged path is appended to this run's `staged_paths`
        accumulator and Step 6 is skipped for this task.
     e. Execute the returned bundle per `process-inbox` Step 6
        (a/b/c/d) — same code path. Background mode runs only
        sub-steps 6b and 6c (deterministic non-draft outcomes);
        6a and 6d are skipped because there is no operator
        approval available in this run.
4. If the lead's thread has a new inbound reply since the task
   was created (an unread that `process-inbox` already processed
   OR a race-condition reply that landed between inbox and
   tasks), the bundle's thread content reflects the current
   state, and rockstarr-reply re-classifies from the fresh
   context. Do not draft against a stale task premise.
5. After the pipeline returns, mark the task done in Interceptly
   (click the task's complete checkbox) and mirror the
   completion into the `Tasks` sheet.

### Step 3 — Pagination

Interceptly's My Tasks list paginates. Walk beyond page 1 — the
bot does NOT stop after the first page. Stop only when the filter
(`Due date ≤ today`) returns zero rows.

### Step 4 — Handoff

When no overdue or due-today tasks remain for this account,
return control to the caller. Return shape (matches
`process-inbox`):

```
{
  account_label: "<active managed account>",
  processed_count: <int>,        # tasks worked this run
  staged_paths: [
    "03_drafts/replies/<file>",
    ...
  ],
  outcomes: [
    { task_type, lead_name, action, mode_branch, notes? },
    ...
  ]
}
```

`action` values are the union of `process-inbox`'s set plus
`booked` (from the `book-meeting` branch). The daily-loop caller
accumulates `staged_paths` into the global accumulator before
calling `switch-account` to move to the next managed account.

## Task types the bot handles here

- `follow-up` — a reply is stale; the handoff gets an
  `intent_hint` the caller derives from the task metadata
  (`bump` / `referral-pivot` / null). rockstarr-reply's
  classify-reply may bump, pivot to referral, or flag.
- `review-reply` — a thread was flagged earlier and the flag
  was cleared; the pipeline now runs normally.
- `book-meeting` — lead agreed to a slot and supplied required
  fields; routes to `book-meeting`, NOT through
  rockstarr-reply. The booking is the close.
- `flagged_review` — created 2 business days after a prior
  `flag` bundle. Re-runs the handoff so rockstarr-reply can
  re-evaluate with any updated
  `/00_intake/icp-qualifications.md`.

## Constraints

- Overdue before due-today, within the Due-date sort.
- The daily loop never moves to another account while this
  skill still has overdue or due-today tasks on this account.
- Afternoon re-runs of the daily loop do NOT re-run this skill.
  Only the morning run processes tasks. Afternoon is inbox-only
  (per the schedule spec).
- Background mode never produces an `authorized-send` or
  `book-meeting-handoff` outcome from a `follow-up` /
  `review-reply` / `flagged_review` task. Approval-gated
  outcomes wait for the operator to open the urgent notify email
  and authorize via Cowork.
- `book-meeting` task type ALWAYS executes `book-meeting`,
  including in background mode. The booking substance was
  authorized when the operator approved the prior reply that
  produced the agreed time + lead_fields.

## Failure modes

- **Task row routes to an empty thread** (lead deleted from
  Interceptly). Skip; mark task done with a note in the mirror.
  Write a quiet `_errors.md` note for the operator.
- **Task's underlying lead is now `not_target`.** Let the
  handoff run — rockstarr-reply's classify-reply will
  return a Warm-non-ICP / Skeptical-non-target bucket and the
  non-ICP path applies. Mark the task done after the bundle
  resolves.
- **Pagination breaks.** Interceptly has shipped pagination
  regressions. If clicking "Next page" produces no new rows,
  retry once with a reload. On second failure, write
  `_errors.md` and stop — better to miss page 2 and surface the
  issue than to fake success.

## What NOT to do

- Do not start this skill while `process-inbox` has unreads
  remaining.
- Do not process future-dated tasks. Only overdue + due-today.
- Do not bypass the handoff. Even a `follow-up` task's bump
  reply runs through `qualify-lead` first — the lead's
  situation may have changed since the task was created.
- Do not cross accounts from this skill. Switching is a
  separate step (`switch-account`) run by the daily loop.
- Do not draft a reply in this plugin. rockstarr-reply owns
  drafting.
