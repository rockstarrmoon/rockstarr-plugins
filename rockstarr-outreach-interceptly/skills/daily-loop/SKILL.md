---
name: daily-loop
description: "This skill should be used when the user says \"run today's outreach loop\", \"run the Interceptly daily loop\", \"do today's outreach run\", or when a scheduled task in rockstarr-infra fires the daily Interceptly run. Orchestrates the full per-account pass for every managed Interceptly account: confirm-session-interceptly → preview-queue-interceptly (optional) → process-inbox → process-my-tasks → switch-account → repeat. Has two modes. In foreground mode (default, operator in chat) it threads operator approvals through inline as today. In background mode (used by the scheduled run) it accumulates staged_paths from every account and at the end fires rockstarr-infra:notify-reply-ready once with the global accumulator if non-empty. Refuses to run if confirm-session-interceptly fails on any account — that account is skipped, and the loop continues with the next account."
---

# daily-loop

The orchestration layer for an Interceptly daily run. Owns the
per-account iteration order, the global `staged_paths`
accumulator, and the single end-of-run `notify-reply-ready` call
that the rockstarr-infra v0.8.0 cross-bot approvals layer
expects from an outreach variant.

This skill is what the scaffold-time scheduled task calls. It is
also what an operator types when they say "run today's outreach
loop" — same skill, different mode.

## When to run

- Every weekday morning, fired by the scheduled task that
  `rockstarr-infra:scaffold-client` wires at install time. Mode
  for the scheduled run is `background`.
- On demand by the operator with the trigger phrases above. Mode
  for the on-demand run is `foreground`.
- An optional afternoon scheduled run that is inbox-only (no
  `process-my-tasks` second pass). Mode is `background`.

## Inputs

- `mode` — `foreground` (default) or `background`. Foreground
  threads operator approvals inline; background accumulates and
  emits one urgent email per run.
- `inbox_only` — `false` (default) or `true`. When `true`, skip
  `process-my-tasks` for every account. Used by the optional
  afternoon scheduled run.
- `accounts_filter` — optional array of `account_label` strings.
  When omitted, iterate every active account from
  `00_intake/interceptly-accounts.md`. When present, iterate only
  the named ones (useful for "just run the alex account" requests
  and for partial reruns after an account-specific failure).
- `force_notify` — optional boolean, default `false`. When `true`
  in foreground mode, still fires `notify-reply-ready` at the end
  with whatever was staged (smoke-tests the urgent path without a
  scheduled run). Ignored in background mode (which always
  notifies when non-empty).

## Preconditions

- `rockstarr-reply` is installed. Drafting lives there. Refuse
  with a pointer if missing.
- `rockstarr-infra` is installed at version >= 0.8.0. The notify
  skill lives there. Refuse with a pointer if missing.
- `00_intake/interceptly-accounts.md` exists and lists at least
  one active account.
- `00_intake/icp-qualifications.md` exists.
- `00_intake/style-guide.md` exists with an approved voice.

## Behavior

### Step 1 — Resolve the account list

Read `00_intake/interceptly-accounts.md`. Filter to rows where
`status = active`. If `accounts_filter` was supplied, intersect
with the filter. If the resulting list is empty, refuse with a
clear message ("no active managed accounts; nothing to do") and
exit.

Iterate in the file's row order — that is the operator's chosen
rotation. Account-skipping due to a session failure does not
change the order on subsequent runs.

### Step 2 — Initialize the global accumulator

```
{
  staged_paths: [],
  per_account_results: [],
  errors: []
}
```

### Step 3 — Per-account pass

For each account in the resolved list:

1. **`switch-account`** with the account's label. Confirms the
   Interceptly sidebar is on the right account. (For the first
   iteration this is a no-op if `switch-account` already left
   the bot on the right account, but call it explicitly for
   determinism.)
2. **`confirm-session-interceptly`**. If FAIL: write the failure
   to `errors[]`, write `_errors.md`, SKIP the rest of this
   account, and continue to the next account in the list. Do NOT
   abort the whole run — the other accounts may be fine, and a
   partial run is more useful than nothing.
3. **`preview-queue-interceptly`** if `stack.md.outreach_daily_preview =
   true` (default). Writes `02_inputs/outreach/queue-YYYY-MM-DD-
   <account>.md`. Background runs still write the preview — it
   is a useful audit record even when no operator reads it
   in real time.
4. **`process-inbox`** with the resolved `mode`. Append the
   returned `staged_paths` to the global accumulator. Append
   `{account_label, processed_count, outcomes}` to
   `per_account_results`.
5. **`process-my-tasks`** with the resolved `mode`, unless
   `inbox_only = true`. Append the returned `staged_paths` to
   the global accumulator. Append the account result.
6. Move to the next account.

### Step 4 — Global rollup

Write a per-run summary to
`02_inputs/outreach/daily-loop-YYYY-MM-DD.md` with: mode, accounts
processed, accounts skipped (with reason), per-account counts
(unreads worked, tasks worked, sends, labels, flags, drafts
staged for approval, errors), and the total `staged_paths` count.

This file is the operator's single source of truth for "what
happened in today's run." It is also what the next morning's
`approvals-digest` cross-references when listing reply drafts
that have been waiting since yesterday.

### Step 5 — Fire notify-reply-ready (background only, or
forced)

Decision matrix:

| mode | force_notify | staged_paths empty | action |
|---|---|---|---|
| background | n/a | yes | skip, log "no new drafts" |
| background | n/a | no | call notify-reply-ready with the accumulator |
| foreground | false | n/a | skip (operator is in chat; approvals happened inline) |
| foreground | true | yes | skip, log "force_notify but nothing to notify" |
| foreground | true | no | call notify-reply-ready with the accumulator |

When calling `rockstarr-infra:notify-reply-ready`, pass:

```
staged_paths = <global accumulator>
dry_run      = false
```

The notify skill handles its own front-matter validation, 8-card
soft cap, mtime sort, single-vs-multi shape, and urgent routing.
This skill does NOT pre-render the email and does NOT decide
recipient.

### Step 6 — Daily metrics

Run `metrics-daily-interceptly` once at the end (not per-account). It rolls
up sends, accepts, replies, bookings, opt-outs by ISO date and
campaign. Background-mode runs typically show low send counts
(approvals deferred); that is expected and correct.

### Step 7 — Return

Return a short summary to the caller / operator chat:

> Daily loop complete. Mode: <mode>. Accounts processed:
> <N>/<M>. Drafts staged for approval: <K>. Sends: <S>.
> Bookings: <B>. Errors: <E>. Notify email sent: <yes/no/—>.

If the notify email was sent, include the `message_id`. If any
account was skipped, list it with the failure reason so the
operator can rerun that account specifically with
`accounts_filter = [<account_label>]`.

## Constraints

- Per-account pass order is fixed by `interceptly-accounts.md`
  row order. No interleaving across accounts.
- Within an account: unreads BEFORE tasks. Always.
- A `confirm-session-interceptly` failure on one account NEVER
  aborts the whole run. It skips that account.
- This skill is the ONLY caller of `notify-reply-ready` from this
  plugin. `process-inbox` and `process-my-tasks` accumulate but
  never notify themselves — that would mean one email per
  account per pass instead of one per run.
- Background mode does NOT send messages, does NOT book meetings
  via book-meeting-handoff, does NOT apply approval-gated labels.
  It only stages drafts and executes deterministic non-draft
  branches (label-only, flag, book-meeting task type).
- Foreground mode is the legacy interactive path. Approvals
  happen inline. No email is fired unless `force_notify = true`.

## Failure modes

- **Every account fails confirm-session.** No drafts staged, no
  sends. Return early with a blunt summary: "All <N> accounts
  failed session confirmation. Inspect `_errors.md` and rerun
  after fixing the underlying logins."
- **notify-reply-ready returns an error** (mailer egress block,
  bearer issue, Resend rejection). Do NOT retry the loop — the
  drafts are already staged and `approvals-digest` will pick
  them up tomorrow morning. Surface the error in the daily-loop
  summary so the operator knows the urgent email did not go out.
- **rockstarr-reply errors mid-account.** The per-account skill
  already handles this and writes `_errors.md`. The loop carries
  on; the affected account's `outcomes[]` notes the failure.
- **A draft path is staged but the file is missing by the time
  notify-reply-ready reads it** (race condition, deletion). The
  notify skill skips that path with a warning. The loop's job is
  just to hand over what it accumulated.

## What NOT to do

- Do not call `notify-reply-ready` per account. Once per run.
- Do not interleave accounts. Finish unreads + tasks for one
  account before moving to the next.
- Do not promote `mode = foreground` for scheduled runs. The
  schedule has no operator and inline approvals would block the
  scheduled job indefinitely. The schedule MUST run with
  `mode = background`.
- Do not let a single account's failure abort the run. The whole
  point of multi-account rotation is that one account's session
  going stale doesn't pause everyone else.
- Do not write the urgent email's content here. The notify skill
  owns rendering. This skill only assembles the `staged_paths`
  list.
- Do not skip the per-run summary file even on a no-op run. A
  zero-row summary is still an audit record.

## Caller integration contract

Two callers expected to invoke this skill:

1. **rockstarr-infra's scheduled task** (wired at scaffold time):
   morning run with `mode = background`, optional afternoon run
   with `mode = background, inbox_only = true`. The schedule
   itself is rockstarr-infra's responsibility; this skill is the
   entry point it calls.
2. **Operator chat** with trigger phrases like "run today's
   outreach loop": `mode = foreground` (default).

Either way, the skill is self-contained — it owns iteration,
accumulation, notification, and the per-run summary.

## Related

- `rockstarr-infra:notify-reply-ready` — the urgent reply-ready
  notification this skill fires.
- `rockstarr-infra:approvals-digest` — the daily digest that
  picks up drafts still pending the morning after.
- `process-inbox` — per-account unread pass; returns
  `staged_paths` this skill accumulates.
- `process-my-tasks` — per-account task pass; same accumulation.
- `confirm-session-interceptly` — per-account session check that
  gates whether the rest of the per-account pass runs.
- `metrics-daily-interceptly` — end-of-loop daily roll-up.
