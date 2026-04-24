---
name: preview-queue
description: "This skill should be used as the second step of the daily outreach loop (after confirm-session-interceptly passes), or when the user asks to \"preview today's queue\", \"show what the bot plans to do today\", or \"write the daily outreach preview\". Writes a markdown file at 02_inputs/outreach/queue-YYYY-MM-DD.md summarizing planned work per managed account: unread inbox count, overdue task count, due-today task count, active campaigns, flagged-leads count. Default enabled; togglable via stack.md.outreach_daily_preview=false."
---

# preview-queue

Non-sending step. Purely observational — tells the operator what
the bot plans to do today, per managed account, before it starts
doing it.

## When to run

- Step 2 of the daily loop, after `confirm-session-interceptly`
  passes for the first managed account.
- On demand when the user says "what's the queue look like today?"
  or "preview today's outreach."

## Preconditions

- `/00_intake/stack.md` has `outreach_accounts[]` non-empty.
- `/00_intake/stack.md.outreach_daily_preview` is `true`
  (default). If `false`, no-op and return quietly.
- `confirm-session-interceptly` passed for the first managed
  account in this run.

## Behavior

### Step 1 — Walk managed accounts

For each account in `outreach_accounts[]` order:

1. If this is NOT the first account, just read the workbook
   state for the counts below — do NOT switch accounts in the
   browser here. This skill is non-sending and non-UI-driving
   past the current account. Switch + confirm happen in the
   real per-account pass, not in the preview.
2. If this IS the first account (already on screen from
   `confirm-session-interceptly`), read counts from Interceptly
   directly so the preview reflects Interceptly's actual state,
   not the possibly-stale mirror.

For each account, collect:

- **Unread inbox count.** Interceptly → Inbox → Replied filter.
  Count unread rows.
- **Overdue task count.** Interceptly → My Tasks → Due date
  filter: date < today. Count rows.
- **Due-today task count.** My Tasks → date = today. Count rows.
- **Active campaigns.** Read the `Campaigns` sheet of
  `outreach-mirror.xlsx`; count rows with `status = running` or
  `started_at` set.
- **Flagged leads.** Read `/02_inputs/replies/_flags.md` (written
  by `rockstarr-reply:flag-for-review`); count entries for this
  account awaiting human review. If the file does not exist,
  treat the count as 0 — no flags have been raised yet.

### Step 2 — Format the preview

Write `/02_inputs/outreach/queue-<YYYY-MM-DD>.md`:

```markdown
---
date: <YYYY-MM-DD>
generated_at: <ISO>
schema_version: 1
---

# Daily outreach queue — <YYYY-MM-DD>

Session confirmed: <first account>. Rotation: <N> managed
accounts.

## Per-account summary

### <account_label 1>

- Unread inbox: <N>
- Overdue tasks: <N>
- Due-today tasks: <N>
- Active campaigns: <N>
- Flagged leads awaiting review: <N>

### <account_label 2>

... (same structure) ...

## Flagged leads awaiting review

<list from /02_inputs/replies/_flags.md — empty if none>

## Today's plan

1. <account 1>: process inbox (N unreads), then My Tasks
   (N overdue + N due-today).
2. Switch to <account 2>, confirm session, process inbox
   (N unreads), then My Tasks.
3. (...)
4. End-of-day: metrics-daily rollup.
```

### Step 3 — Return summary

Send a one-line summary to the user:

> Preview written to `queue-<YYYY-MM-DD>.md`. Today: <total
> unreads> unreads, <total overdue> overdue, <total due-today>
> due-today across <N> accounts. Flagged: <N>.

### Step 4 — Togglability

If `stack.md.outreach_daily_preview = false`, write nothing and
return the one-line summary only. The toggle is there for clients
who find the preview file noise instead of signal.

## Outputs

- `/02_inputs/outreach/queue-<YYYY-MM-DD>.md` (if enabled).

## Failure modes

- **Chrome MCP drops while reading the first account's counts.**
  Retry once. On second failure, write the preview with mirror-
  derived counts for every account and add a footer:
  "Counts derived from local mirror; Interceptly live counts
  unavailable this run."
- **No managed accounts configured.** Refuse; point at
  `discover-interceptly-accounts`.
- **rockstarr-reply not installed and `_flags.md` absent.** Drop
  the flagged-leads section; write a footer noting
  rockstarr-reply is not installed so no flag queue exists yet.

## What NOT to do

- Do not switch accounts in the browser from this skill. Account
  switches happen in the real per-account pass, paired with
  `confirm-session-interceptly`.
- Do not draft, send, label, or create tasks from this skill.
  Preview is read-only.
- Do not overwrite yesterday's queue file. Each date gets its
  own file.
