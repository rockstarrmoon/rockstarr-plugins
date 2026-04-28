---
name: outreach-weekly-report-interceptly
description: "This skill should be used after metrics-weekly-interceptly finishes, or when the user asks to \"generate the weekly outreach report\", \"write this week's report\", or \"render the Interceptly weekly\". Consumes Metrics (Weekly) rows for the target ISO week and produces a human-readable markdown report at /06_reports/weekly/outreach-YYYY-WW.md with per-account tables, week-over-week deltas, stale review-reply callout, non-ICP log highlights, flagged-leads list, session-failure summary, and a 'what Rachel / Jon should notice' block."
---

# outreach-weekly-report-interceptly

Human-readable wrap-up. The Friday artifact the client reads
before Monday. Every metric on the page has a pointer back to the
workbook so the client can audit.

This plugin owns `outreach-mirror.xlsx` end to end (Campaigns,
Session, Qualifications, Messages, Labels, Tasks, Replies,
MeetingProposals, Metrics). The only reply-side surface this
report reads is `/02_inputs/replies/_flags.md`, which
`rockstarr-reply:flag-for-review` writes when drafting is
refused or the operator asks to flag. If `_flags.md` is missing,
render the report without a flagged-leads section.

## When to run

- Automatically after `metrics-weekly-interceptly` finishes.
- On demand when the user asks for a specific week's report.

## Preconditions

- `Metrics (Weekly)` sheet has rows for the target ISO week
  (produced by `metrics-weekly-interceptly`).
- `outreach-mirror.xlsx` is readable.

## Inputs

- `iso_week` — `YYYY-WW` (defaults to current).

## Behavior

### Step 1 — Read metrics

Read every `Metrics (Weekly)` row for the target ISO week.
Capture `any_day_partial` per account — surfaces a data-quality
caveat below.

### Step 2 — Read adjunct signals

From `outreach-mirror.xlsx`:

- `Campaigns` sheet — campaigns configured, paused, stopped this
  week (based on `configured_at` / `paused_at` / `stopped_at`).
- `Session` sheet — session-failure heartbeats this week.
- `Tasks` sheet — stale `review-reply` and `flagged_review` tasks
  (past due and still pending).
- `Replies` sheet — bookings this week with `meeting_datetime`.
- `Qualifications` sheet — `not_target` rulings this week (feed
  the non-ICP highlights; pull `matching_rule` for the callout).

From the filesystem:

- `/02_inputs/replies/_flags.md` (owned by
  `rockstarr-reply:flag-for-review`) — count of flagged leads
  raised this week, and the subset still unresolved at week end.
  If the file is missing, skip the flagged-leads section and note
  that rockstarr-reply has not yet flagged anything.
- `/02_inputs/outreach/_errors.md` — count session failures,
  Chrome MCP drops, UI regressions.

### Step 3 — Render the report

Write `/06_reports/weekly/outreach-<YYYY-WW>.md`:

```markdown
---
iso_week: <YYYY-WW>
week_start: <YYYY-MM-DD>
week_end: <YYYY-MM-DD>
generated_at: <ISO>
schema_version: 1
---

# Outreach — Week of <week_start>

_Interceptly is source-of-truth; this report reads from the local
audit mirror `/02_inputs/outreach/outreach-mirror.xlsx` plus
`/02_inputs/replies/_flags.md` for the flagged-leads signal._

## Headline numbers

<one-line summary, e.g., "Across 3 managed accounts: 127 unreads
processed, 52 sends, 4 bookings, 18 flagged, 0 session failures.">

## Per-account results

### <account_label 1>

| metric | this week | prev week | Δ |
|---|---|---|---|
| Unreads processed | N | N | +N |
| Sends | N | N | +N |
| Bookings | N | N | +N |
| Non-ICP declines | N | N | +N |
| Flags | N | N | +N |
| Reply-to-send ratio | X% | Y% | +Z pp |
| Booking rate | X% | Y% | +Z pp |

**Label distribution (this week):** interested=N,
follow_up=N, ignore=N, not_interested=N, ...

**Campaign changes this week:** configured=N, paused=N,
stopped=N.

### <account_label 2>

... (same shape) ...

## Bookings

<bulleted list — one line per booking: lead_name (company),
campaign, meeting_datetime, source (automated / manual). Sourced
from the Replies sheet.>

## Flagged leads still open

<bulleted list of unresolved entries in /02_inputs/replies/_flags.md.
Include the reason and the day flagged. If none, say "none." If
the file is missing, say "rockstarr-reply has not flagged any
leads yet.">

## Stale review-reply tasks

<bulleted list of Tasks rows where task_type IN (review-reply,
flagged_review) AND due_date < today AND status = pending. If
none, say "none.">

## Non-ICP highlights

<3-5 most recent not_target rulings from the Qualifications sheet.
For each: lead name, matching_rule cited, date.>

> If you keep seeing patterns here that feel wrong, re-run
> `capture-icp-qualifications` to tighten the baseline rules.
> `draft-icp-campaign-interceptly` only narrows; the baseline lives in
> `/00_intake/icp-qualifications.md`.

## Session + UI health

- Session failures: N (per account if >0)
- Chrome MCP drops: N
- UI regressions (Interceptly / LinkedIn): N

## What Rachel / Jon should notice

<2-4 bullets that translate the numbers into judgment — picked
based on notable deltas, first-time-occurring events, or
shifting patterns. Examples the author might write:>

- Booking rate on <account> jumped from Y% to X% — what
  changed? Worth checking whether the warm-reply pattern tuned
  last week is the signal.
- Non-ICP declines climbed to X% on <account> — the ICP rules
  may be letting through roles that don't convert. Consider
  re-running `capture-icp-qualifications`.
- N stale review-reply tasks sitting past due — block calendar
  time to clear them Monday morning.
- Zero session failures all week — `confirm-session-interceptly`
  doing its job.
```

If `any_day_partial` was true for any account, add at the bottom:

> Note: one or more daily rollups this week were marked partial.
> Numbers may be floors rather than truth. Re-run `metrics-daily-interceptly`
> with the final flag after the affected day's loop completes.

### Step 4 — Return

`{report_path: "/06_reports/weekly/outreach-<YYYY-WW>.md"}`
plus a short chat summary: headline numbers + the "what to
notice" bullets.

## Rhetoric rules

The "What Rachel / Jon should notice" section is the only place
this skill offers interpretation. Everywhere else it cites
numbers. Interpretation should be:

- Specific — name the account, the metric, the delta.
- Actionable — what to look at, what to decide, what to tune.
- Short — 2-4 bullets max. The numbers do most of the work.

Do NOT editorialize in the metrics tables. Tables are facts.

## Failure modes

- **Metrics (Weekly) row is missing for an account.** Skip
  that account's section with a one-line note and continue.
- **`/02_inputs/replies/_flags.md` missing.** Drop the flagged-
  leads section with the note above. This is expected if
  rockstarr-reply is installed but has not flagged anything yet,
  or if it is not installed at all.
- **Prev-week deltas blank** (first week, or a missed week).
  Blank Δ is fine.
- **Long bookings list** (>10 in a week). Cap the rendered list
  at 10 with a "see workbook for full list" line.

## What NOT to do

- Do not fabricate bookings, flags, or error counts. Every
  number in this report comes from the workbook or from
  `_flags.md`.
- Do not editorialize numbers you don't have. If a metric is
  blank, say "no data" rather than assuming.
- Do not edit `_flags.md`. That file is owned by
  `rockstarr-reply:flag-for-review`; this report only reads it.
- Do not recommend changes to `icp-qualifications.md` without
  pointing at `capture-icp-qualifications` — the file should
  only be rewritten by that skill.
