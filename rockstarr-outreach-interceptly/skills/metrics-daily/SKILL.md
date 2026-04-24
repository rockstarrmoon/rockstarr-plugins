---
name: metrics-daily
description: "This skill should be used as the final step of the daily outreach loop, or when the user says \"roll up today's metrics\", \"close out the day\", or \"update the daily outreach scoreboard\". Writes one row per managed account into the Metrics (Daily) sheet — unreads processed, sends, labels applied (broken down by label), non-ICP declines, bookings, flags, session failures — and appends today's per-account summary into /05_published/outreach/YYYY-MM-DD.md. Runs AFTER every managed account's process-inbox + process-my-tasks have finished."
---

# metrics-daily

End-of-day rollup. One row per managed account per day. Feeds
`metrics-weekly` and `outreach-weekly-report`.

## When to run

- Final step of the scheduled daily loop, after every managed
  account has completed its inbox + tasks pass.
- On demand when the user says "close out today's metrics" or
  "recompute today's numbers."

## Preconditions

- Daily loop has finished (or the user explicitly wants a mid-
  day snapshot). For mid-day, tag the row `partial = true`.
- `outreach-mirror.xlsx` exists and the daily state sheets owned
  by this plugin (Campaigns, Session, Qualifications, Messages,
  Labels, Tasks, MeetingProposals) have today's rows.
- `/02_inputs/replies/_flags.md` (owned by
  `rockstarr-reply:flag-for-review`) is read-only to this skill;
  flag counts come from there.

## Inputs

- `date` — defaults to today.
- `partial` — optional, defaults to false.

## Behavior

### Step 1 — Per-account aggregation

For each account in `stack.md.outreach_accounts[]`, aggregate
today's activity:

- **unreads_processed** — count of threads whose most recent
  inbound was read and qualified today (Qualifications sheet
  rows where `account_label = <account>` AND `ts` is today).
- **sends** — count of Messages rows for this account today.
- **labels_applied** — count of Labels rows for this account
  today, broken down by label:
  - interested, booked, referral, follow_up, contact_later,
    bad_fit, ignore, not_interested, plus any custom.
- **non_icp_declines** — count of Qualifications rows with
  verdict = `not_target` today.
- **bookings** — count of Replies rows with
  `classification = booked` today, for this account (written by
  `mark-booked`).
- **flags** — count of blocks in `/02_inputs/replies/_flags.md`
  timestamped today for this account. If the file is missing,
  treat as 0.
- **tasks_created** — count of Tasks rows created today.
- **tasks_completed** — count of Tasks rows completed today
  (for this account).
- **session_failures** — count of Session sheet rows with
  `result = fail` today for this account.
- **campaigns_configured_today** — count of Campaigns rows with
  `configured_at` timestamped today.
- **campaigns_stopped_today** — count of Campaigns rows whose
  status changed to `paused` or `stopped` today.

### Step 2 — Write to Metrics (Daily)

Append one row per account to the `Metrics (Daily)` sheet:

| column | value |
|---|---|
| date | `YYYY-MM-DD` |
| account_label | `<account>` |
| unreads_processed | N |
| sends | N |
| labels_interested | N |
| labels_booked | N |
| labels_referral | N |
| labels_follow_up | N |
| labels_contact_later | N |
| labels_bad_fit | N |
| labels_ignore | N |
| labels_not_interested | N |
| non_icp_declines | N |
| bookings | N |
| flags | N |
| tasks_created | N |
| tasks_completed | N |
| session_failures | N |
| campaigns_configured_today | N |
| campaigns_stopped_today | N |
| partial | true/false |

### Step 3 — Append per-account summary to published log

Append to `/05_published/outreach/<YYYY-MM-DD>.md`:

```markdown
## End-of-day summary — <YYYY-MM-DD>

### <account_label 1>
- Unreads processed: N
- Sends: N
- Bookings: N
- Non-ICP declines: N
- Flags: N
- Labels: interested=N, follow_up=N, ignore=N, ...

### <account_label 2>
... (same shape) ...

### Across all managed accounts
- Total sends: N
- Total bookings: N
- Total flags: N
- Session failures today: N
- Campaigns configured today: N
- Campaigns paused/stopped today: N

### Notes
<any session failures, error-log highlights, or partial-day flags>
```

If the file already has content from earlier in the day
(e.g., per-account pass logs from `mark-booked`), APPEND the
summary at the end rather than overwriting.

### Step 4 — Surface session failures

If `session_failures > 0` for any account, include a call-out
block at the top of the daily log:

```
> Session failure(s) today: <account_label> — see
> /02_inputs/outreach/_errors.md. Resolve before the next
> scheduled run.
```

### Step 5 — Return

A brief summary the caller can show the user:

> Daily metrics rolled up. Across <N> accounts: <total sends>
> sends, <total bookings> bookings, <total flags> flags. Full
> summary at `/05_published/outreach/<YYYY-MM-DD>.md`.

## Failure modes

- **Messages sheet missing for today.** Treat as zero sends for
  every account (no misleading fabrication). Note in the summary.
- **Some managed account's counts can't be aggregated** (mirror
  corruption). Write whatever is available and note the gap in
  the summary. The operator can rerun after fixing the mirror.
- **`/02_inputs/replies/_flags.md` missing.** Treat flag count as
  0 and note rockstarr-reply is not installed or has not yet
  flagged anything.

## What NOT to do

- Do not send anything. This skill is pure aggregation.
- Do not compute weekly rollups here. `metrics-weekly` owns that.
- Do not overwrite yesterday's row. Each date × account row is
  immutable; re-runs for the same date should update in place
  (via `partial` → final transition), not create a second row.
