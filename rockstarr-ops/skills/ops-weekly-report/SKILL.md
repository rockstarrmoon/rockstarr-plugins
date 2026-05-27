---
name: ops-weekly-report
description: "This skill should be used every Friday end-of-day, or when the user says \"run the ops weekly report\", \"close out the ops week\", or \"weekly ops rollup\". Aggregates the week's data from /05_published/ops/ + ops-mirror.xlsx into /06_reports/weekly/ops-[YYYY-WW].md. Sections: sales calls prepped, audits run (play breakdown + override rate), reengagements sent + replied, post-call processings (CRM-write success rate), deliverability trend with low-score callouts, and a stale-review-items section across ALL clients in one place."
---

# ops-weekly-report

Friday end-of-day rollup. Reads the week's outputs across every
ops skill, produces one human-readable markdown report at
`/06_reports/weekly/ops-[YYYY-WW].md`.

This skill writes ONE artifact: the report. It does not draft
prose, send messages, or update any CRM record. The report is
operator-facing — bypasses stop-slop.

## When to run

- Scheduled, every Friday at 17:00 local. The schedule is wired
  by `rockstarr-infra:scaffold-client` at install time.
- On demand by the operator with the trigger phrases above.

## Preconditions

- `/05_published/ops/` exists and has data from the current ISO
  week.
- `/02_inputs/ops/ops-mirror.xlsx` exists.
- `stack.md` is set.

If `/05_published/ops/` is empty for the current week (the
operator was on PTO, or no ops skills ran), produce the report
with empty-state per section ("No sales calls prepped this
week.") rather than skipping the report.

## Inputs

- `target_week` — optional ISO week string (e.g., `2026-W19`).
  Defaults to the current ISO week.

## Behavior

### Step 1 — Resolve the week boundaries

Compute the Monday 00:00 → Sunday 23:59 local boundaries for
`target_week`. Every read in the next steps filters to this
window.

### Step 2 — Aggregate sales-call prep

Read every `daily-summary-[date].md` in `/05_published/ops/`
inside the week boundaries. Count:

- `sales_call_1_count` summed across days.
- `sales_call_2_count` summed.
- `client_catch_up_count` summed.
- ICP verdict breakdown for Call 1: GO / CAUTION / DISQUALIFY
  counts.
- Recorder-unavailable count for Call 2.

Also count overdue commitment count + stale review count
across all client-agenda entries.

### Step 3 — Aggregate audits

Read every `audits-[YYYY-MM].md` row inside the week. Count:

- Total audits run.
- Recommended-play breakdown (Plays 1-6).
- Operator-locked-play breakdown (Plays 1-6).
- Override count + override-to breakdown (which play the
  operator overrode TO most often).

### Step 4 — Aggregate reengagements

Read every `reengagements-[YYYY-MM].md` row inside the week.
Count:

- Reengagements sent.
- Reengagement replies received (matched against the active
  outreach variant's reply log).
- Reply rate as `replies / sent`.
- Booked meetings from reengagement replies.
- Engagement-signal-overridden count (Gate C overrides).

### Step 5 — Aggregate post-call processings

Read every `post-calls-[YYYY-MM].md` row inside the week.
Count:

- Total processings completed.
- CRM-write success rate.
- Manual-punch-list count (no CRM ops bot installed).
- Recap-emails sent vs. drafted vs. skipped.

### Step 6 — Aggregate deliverability runs

Read every `deliverability-[YYYY-MM].md` row inside the week
plus the most recent 8 weeks for a trendline. Capture:

- Score per run.
- Low-score-flag count.
- Trend direction (up / flat / down).
- Sender + segment per run.

### Step 7 — Surface stale review items across ALL clients

Read the latest `ClientAgenda_*.md` sidecar for every client
catch-up that ran this week. Aggregate:

- Every stale-review-flagged task across every client.

This is the cross-client view that's easy to miss when a
single client's agenda only shows their own staleness. The
weekly report puts every stale item from every client in one
list — sorted by days-stale, descending. The operator can
clear the worst offenders in one batch.

### Step 8 — Read the ops mirror for completeness check

Read `/02_inputs/ops/ops-mirror.xlsx`. Cross-check counts in
each sheet (Audits, Reengagements, PostCalls, Deliverability)
against the markdown counts from Steps 2-6. Discrepancies
indicate one of:

- A skill wrote to the markdown but not the mirror (or vice
  versa) — surface as a `data integrity` callout in the
  report.
- A skill ran but errored mid-write — surface as a
  `partial run` callout.

The mirror is the operator's audit substrate; surfacing
discrepancies keeps it honest.

### Step 9 — Render the report

Write to `/06_reports/weekly/ops-[YYYY-WW].md`. Overwrite if
the file exists for this week (re-runs replace).

> **Template convention.** Real `---` in the actual file, not
> `# ---`.

```markdown
# ---
schema_version: 1
type: ops-weekly-report
iso_week: <YYYY-WW>
week_start: <ISO Monday>
week_end: <ISO Sunday>
generated_at: <ISO>
# ---

# Ops weekly — <YYYY-WW>

## Headline

<a 2-3 sentence overview of the week — most important facts:
audit override rate, deliverability trend, stale-review pile-up,
reengagement reply rate.>

## Sales-call prep

| Type | Count |
|---|---|
| Call 1 (discovery) | <N> |
| Call 2 (close) | <N> |
| Client catch-ups | <N> |
| **Total** | **<N>** |

ICP verdict breakdown (Call 1): GO=<N>, CAUTION=<N>,
DISQUALIFY=<N>.

Recorder unavailable on <N> Call 2 preps this week. (<callout
if N > 1>: recurring recorder availability gap — investigate.)

Overdue commitments surfaced: <N>. Stale review items
surfaced: <N>.

## Audits

| Play | Recommended | Operator-locked |
|---|---|---|
| 1: Fulfill | <N> | <N> |
| 2: Reframe | <N> | <N> |
| 3: Curiosity nudge | <N> | <N> |
| 4: Third-party | <N> | <N> |
| 5: Clean break | <N> | <N> |
| 6: Wait | <N> | <N> |

Total audits: <N>. Override rate: <pct>%.
Most-overridden TO: <play name>.

## Reengagements

Sent: <N>. Replies: <N>. Reply rate: <pct>%. Booked: <N>.

(<callout if reply rate < 5%>: reply rate trending low —
consider re-running `capture-warm-reply-pattern`.)

(<callout if engagement-signal-override count > 30% of sent>:
high override rate — operator is reengaging dark leads. Audit
the trade-off.)

## Post-call processings

Completed: <N>. CRM-write success: <pct>%.

Recap email status:
- Sent: <N>
- Drafted (operator hits send): <N>
- Skipped: <N>

Manual punch lists: <N>. (<callout if > 0>: no CRM ops bot
installed — install `rockstarr-ops-bot-<crm>` for
automation.)

## Deliverability

| Date | Sender | Segment | Score | Flag |
|---|---|---|---|---|
| <ISO> | <sender> | <segment> | <score>/10 | <LOW SCORE if applicable> |
| ... | ... | ... | ... | ... |

8-week trendline: <up / flat / down>. Average this week:
<score>/10.

(<callout if any LOW SCORE this week>: bumped to top —
operator should investigate before next send batch.)

## Stale review items across all clients

| Client | Task | Out for review since | Days |
|---|---|---|---|
| <client> | <task> | <ISO> | <N> |
| ... | ... | ... | ... |

(Sorted by days descending. Items shown only when days >= 5
business days. Empty state: "No stale review items — clean
week.")

## Data integrity

- Markdown / mirror discrepancies: <N>. <list paths if any.>
- Partial runs: <N>. <list IDs if any.>

## What Rachel / Jon should notice

<a 3-5 bullet list of patterns the report writer wants to
flag — typically synthesized from the trends above.>
```

### Step 10 — Surface to chat (or email if background)

When run on demand (foreground), render the report path in
chat with a `computer://` link:

> Weekly ops report ready: [`ops-[YYYY-WW].md`](computer://[path]).

When run on the schedule (background), route via
`rockstarr-infra:send-notification` with
`notify_type=weekly-summary`:

- To: `ROCKSTARR_NOTIFY_TO`.
- Subject: `[ops] Weekly report — [YYYY-WW]`.
- Body: report rendered + the markdown.

## Outputs

- `/rockstarr-ai/06_reports/weekly/ops-[YYYY-WW].md`.
- A chat post (foreground) or email (background) with a
  pointer to the report.

## Approvals

Operator-facing report. No `send` step — the report is
informational.

## Failure modes

- **Empty week.** No data in `/05_published/ops/` — produce
  the report with empty-state copy ("No sales calls prepped
  this week.") rather than skipping. The empty report is
  meaningful.
- **Mirror unreadable.** Render the report from markdown only;
  the data-integrity section calls out the mirror failure.
- **One source skill's published files are corrupt** (front-
  matter unparseable). Skip the corrupt file, count one fewer
  in the relevant section, surface the file path under data
  integrity.

## What this skill does NOT do

- Does NOT draft prose. The report is a structured rollup; the
  "What Rachel / Jon should notice" section is light synthesis,
  not generated copy.
- Does NOT consult `style-guide.md`.
- Does NOT update task systems or CRMs. Read-only across every
  source.
- Does NOT run stop-slop. Operator-facing.
