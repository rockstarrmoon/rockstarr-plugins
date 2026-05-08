---
name: daily-call-prep
description: "This skill should be used when the scheduled daily run fires at ops_daily_run_time (default 06:00 local), or when the user says \"run today's call prep\", \"morning prep\", \"do today's prep sweep\", \"prep my day\", \"daily ops sweep\". Orchestrator that classifies every event on today's calendar, sequences the per-event prep skills (prep-call-1 for discovery, prep-call-2 for close, build-client-agenda for client catch-ups), lists internal blocks under 'Other', and posts one mobile-readable summary to chat with computer:// links to every prep / agenda doc. Refuses to run until pitch.md, ops-call-framework.md, and (when ops_deliverability_tool is not none) deliverability-config.md all exist. Archives prep / agenda docs + sidecars to /05_published/ops/ at end-of-day."
---

# daily-call-prep

The orchestration layer for the morning sweep. Owns calendar
classification, per-event sub-skill dispatch, the morning summary
chat post, and the end-of-day archive.

This skill does NOT produce any prep doc itself. It sequences the
skills that do. Anything specific to a single prospect or a single
client lives in `prep-call-1` / `prep-call-2` /
`build-client-agenda` — this skill calls them.

## When to run

- Scheduled, daily at `ops_daily_run_time` (default `06:00`
  local). The schedule is wired by `rockstarr-infra:scaffold-client`
  at install time.
- On demand by the operator with the trigger phrases above.
- Both paths produce the same outputs.

## Inputs

- `mode` — `foreground` (default, operator in chat) or
  `background` (scheduled run, no operator in chat). Foreground
  posts the summary inline; background sends the summary as an
  urgent email via `rockstarr-infra:notify-reply-ready` style
  delivery (this bot uses a separate
  `daily-summary-notification` channel — see Step 8).
- `target_date` — optional ISO date; defaults to today. Used
  for re-runs ("re-run yesterday's prep, the calendar changed").

## Preconditions

- `00_intake/pitch.md` exists.
- `00_intake/ops-call-framework.md` exists.
- `00_intake/deliverability-config.md` exists when
  `stack.md.ops_deliverability_tool != none`.
- `00_intake/icp-qualifications.md` exists.
- `stack.md` has `ops_calendar`, `ops_meeting_recorder`,
  `ops_task_system` set (any of which can be `none`; the
  orchestrator degrades gracefully per Variant B/C/D in the
  spec).

The orchestrator REFUSES to run when any required intake file is
missing. It surfaces a chat message naming the missing file +
the skill that produces it (`capture-pitch`,
`capture-call-framework`, `capture-deliverability-config`) and
exits.

## Behavior

### Step 1 — Read today's calendar

Via `ops_calendar` (Google Calendar via API + Chrome MCP, or
Outlook via API), pull every event between `target_date` 00:00
and `target_date` 23:59 local. For each event, capture:

- Title, description, start, end, attendees (names + emails),
  location / video URL, any booking-source notes from the
  description.

Filter out events the orchestrator should NOT prep:

- `Lunch`, `Coffee`, `Out of office`, `Holiday`, `PTO`, all-day
  events.
- Events the operator has tagged `#noprep` in the description.
- Internal recurring events the operator marks as ignorable in
  `00_intake/calendar-ignore.md` if it exists.

The remaining events are the targets for classification.

### Step 2 — Classify each event

For each filtered event, classify into one of four buckets:

- **Sales Call 1 (discovery).** External attendee + no prior
  recorded call AND no substantive prior thread (3+ messages).
- **Sales Call 2 (close).** External attendee + a prior
  recorded call OR a substantive prior thread.
- **Client catch-up.** Attendee email matches a row in
  `ops_client_roster_source`, OR the event title matches a
  row in the roster.
- **Other.** Internal blocks, team meetings, anything not in
  the three buckets above.

Classification logic for Sales Call 1 vs. Call 2:

1. Resolve the external attendee's identity (most events have
   one external attendee — their email is the lookup key).
2. Search `ops_meeting_recorder` for a prior call with that
   email or that name. Found → Call 2.
3. Search `ops_email_outreach_tool` + `ops_email_platform` for
   a thread with that email. Substantive (3+ messages) → Call 2.
4. Otherwise → Call 1.

### Step 3 — Dispatch per-event sub-skills

For each Sales Call 1 event: call `prep-call-1` with the
event's payload.

For each Sales Call 2 event: call `prep-call-2` with the
event's payload.

For each client catch-up event: call `build-client-agenda` with
the client name and event start.

For each Other event: capture title + start in the summary
under "Other" — no doc produced.

Sub-skills run in series, not parallel. Most days have 1-3
events; the serial cost is ~30s-2min total. Parallelization
would add complexity without payoff at this scale.

### Step 4 — Capture sub-skill results

Each sub-skill returns a `{ docx_path, sidecar_path,
icp_verdict?, recorder_unavailable?, ... }` payload. The
orchestrator collects every payload into a per-day digest:

```
{
  date: <ISO>,
  events: [
    { type: prep-call-1, lead_name, docx_path, sidecar_path,
      icp_verdict, ... },
    { type: prep-call-2, lead_name, docx_path, sidecar_path,
      ... },
    { type: client-agenda, client_name, docx_path, sidecar_path,
      overdue_count, stale_review_count, ... },
    { type: other, title, start },
  ]
}
```

### Step 5 — Render the morning summary (mobile-readable)

The summary is a single chat post the operator reads on their
phone between waking up and the first coffee. Three rules:

1. **Mobile-readable.** No tables wider than ~30 chars per
   row. Bullet list, not numbered list. Each bullet ≤2 lines.
2. **Every link is `computer://`.** Operators tap a
   `computer://` link from chat and it opens directly into
   the docx on their desktop.
3. **Critical info on the line.** ICP verdict, overdue count,
   stale count, and recorder-unavailable flag are visible
   without expanding the doc.

Summary template:

```
## Today, <weekday> <month> <day>

### Sales calls
- <time> — Call 1: <lead_name> @ <company>
  <icp_verdict_chip> · [prep doc](computer://<path>)
- <time> — Call 2: <lead_name> @ <company>
  <commitment_chip> · [prep doc](computer://<path>)

### Client catch-ups
- <time> — <client_name>
  ⚠ <overdue_count> overdue · 🚩 <stale_count> stale ·
  [agenda](computer://<path>)

### Other
- <time> — <title>

### FLAGS (review before the calls)
- <list of any recorder_unavailable / DISQUALIFY / pricing-conflict / etc.>
```

The FLAGS section only renders when at least one sub-skill
surfaced a flag. Empty FLAGS section is omitted.

### Step 6 — Post the summary to chat (foreground)

In foreground mode, render the summary as a chat message in
the active Cowork session. The operator sees it inline.

### Step 7 — Send the summary as an email (background)

In background mode, the operator is not in chat. Route the
summary to email via `rockstarr-infra:send-notification` with
`notify_type=daily-summary` (a non-urgent channel — distinct
from `notify-reply-ready`'s urgent path):

- To: the operator's `ROCKSTARR_NOTIFY_TO`.
- Subject: `[ops] Daily prep — <date> — <event_count> events`.
- Body: rendered summary + the markdown.

When `notify-reply-ready` (the urgent reply path) fires for a
DIFFERENT bot during the same morning run, that's separate —
the daily summary and reply-ready emails coexist in the
operator's inbox without overlap.

### Step 8 — Persist the digest

Write the full digest as markdown to
`/rockstarr-ai/05_published/ops/daily-summary-<date>.md`. This
is what `ops-weekly-report` reads for the weekly rollup.

> **Digest template convention.** Real `---` in the actual
> file, not `# ---`.

```markdown
# ---
schema_version: 1
type: ops-daily-summary
date: <ISO date>
event_count: <integer>
sales_call_1_count: <integer>
sales_call_2_count: <integer>
client_catch_up_count: <integer>
other_count: <integer>
flags: [<list>]
generated_at: <ISO>
mode: foreground | background
# ---

# Daily ops summary — <date>

(... full summary text from Step 5 ...)

## Sub-skill results

| Type | Subject | Docx | Sidecar | Notes |
|---|---|---|---|---|
| prep-call-1 | <name> | <path> | <path> | ICP=GO / CAUTION / DISQUALIFY |
| prep-call-2 | <name> | <path> | <path> | recorder=ok / unavailable, commits=N |
| client-agenda | <client> | <path> | <path> | overdue=N, stale=N, prod=N/N/N |

## FLAGS

<list>
```

### Step 9 — Schedule end-of-day archive

The morning run schedules an end-of-day archive at 11:59 PM
local. The archive moves all of today's prep docs + sidecars
+ agendas from `/03_drafts/ops/sales-prep/` and
`/03_drafts/ops/agendas/` to the date-stamped sub-folder of
`/05_published/ops/`. The archive happens regardless of
whether the operator opened the docs — the published copy is
the durable record for the weekly report.

The archive uses `rockstarr-infra:publish-log` for the
publish-log entry, so the cross-bot publish log captures every
archived prep / agenda doc.

## Outputs

- One docx + one `.md` sidecar per Call 1, Call 2, and client
  catch-up event (produced by the sub-skills).
- One chat post (foreground) OR one email (background) with
  the morning summary.
- One `daily-summary-<date>.md` digest in
  `/rockstarr-ai/05_published/ops/`.
- One scheduled end-of-day archive task per run.

## Approvals

The orchestrator does NOT have a per-event approval gate.
Each sub-skill writes to drafts; the operator opens the doc
when they're ready. The summary is informational — not a
gate.

The end-of-day archive is unattended. The operator cannot
"reject" today's archive — it's the durable record. To remove
a doc, the operator deletes it from `/05_published/ops/`
manually.

## Failure modes

- **Calendar API errors.** Retry once with backoff. If still
  failing, surface in chat (foreground) or email (background)
  with "Calendar unreachable — manual prep today" and exit.
- **One sub-skill errors but others run fine.** Capture the
  error in the digest's FLAGS section; render the rest of
  the summary normally. The errored event gets a
  "PREP FAILED — manual prep" line.
- **Classification is wrong** (e.g., Call 1 → Call 2 because
  a stale recorder hit was matched). The operator catches it
  visually in the summary and asks the orchestrator to
  re-run with a hint ("re-run today's prep, classify
  <name> as Call 1"). Re-runs with hints overwrite the prior
  doc + sidecar.
- **Multiple events at the same time** (back-to-back, or
  overlapping). The orchestrator handles each independently;
  the operator decides on the day which to attend.
- **Operator on PTO** (full-day OOO event in calendar). The
  orchestrator detects the OOO event and exits early with
  "PTO detected — skipping today's prep." Re-running on
  return resumes normal flow.

## What this skill does NOT do

- Does NOT produce any prep doc itself. Sub-skill dispatch
  only.
- Does NOT update the calendar. Read-only.
- Does NOT send any customer-facing message. The whole skill
  is operator-facing.
- Does NOT consult `pitch.md` or `ops-call-framework.md`
  directly — it confirms they exist as a precondition. The
  sub-skills do the actual reads.
- Does NOT run on weekends by default. Configurable via
  `stack.md.ops_daily_run_weekends=true` if a client wants
  weekend prep.
