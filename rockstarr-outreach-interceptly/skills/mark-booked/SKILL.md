---
name: mark-booked
description: "This skill should be used whenever a lead has booked a meeting — whether the bot booked via book-meeting, or the client booked manually by phone, email, calendar, or any other path. Trigger phrases: \"mark this lead as booked\", \"Jane booked a meeting\", \"I booked this lead manually\", \"log the booking\". This is the single source of truth for booking state in the Interceptly variant: it flips the Leads row last-label=Booked, mirrors the Booked label into Interceptly via apply-label, cancels pending Interceptly follow-up tasks for that lead, and writes a Replies row with label=Booked. Both the automated and manual booking paths converge here."
---

# mark-booked

Shared skill. Canonical source will live at
`rockstarr-infra/skills/_shared/mark-booked/` once the shared
tree is populated. For V0.1 it ships inline in
`rockstarr-outreach-interceptly`. If you edit this file, edit
the matching copy in `rockstarr-outreach-salesnav` in the same
PR.

The single source of truth for "a meeting got booked." Every path
— `book-meeting` success, client-booked manually by phone,
client-booked via their own calendar link — funnels through here
so state transitions stay consistent.

## When to run

- `book-meeting` calls it on successful form submission.
- Client calls it directly from Cowork: "I booked <lead>
  manually for Friday at 2."
- Client calls it when Google Calendar or the booking link shows
  an event they recognize belongs to a campaign lead.
- Integration path: any future "calendar invite accepted" signal
  that triggers automatically.

## Inputs

- `lead_url` — required. Either a LinkedIn profile URL or an
  Interceptly thread URL — both map to the same Leads row.
- `meeting_datetime` — optional ISO-8601. If omitted, store
  `"unspecified"`.
- `source` — `automated` (from `book-meeting`) or `manual`
  (from the client). Defaults to `manual`.
- `notes` — optional free text.

## Preconditions

- `/02_inputs/outreach/outreach-mirror.xlsx` exists.
- `Leads` sheet has a row for `lead_url`. If not, this is either
  a non-campaign booking (skip with a note to the user) or a
  data-entry error (offer the user similar leads from active
  campaigns to disambiguate).

## Behavior

### Step 1 — Find the Leads row

If `last_label = Booked` already, refuse with a clear message:
"<Lead> is already marked booked on `<date_booked>`. No action
taken." Do not double-book history.

### Step 2 — Flip Leads state

- `last_label = Booked`
- `date_booked = today`
- `meeting_datetime = <supplied or "unspecified">`

### Step 3 — Apply the Booked label inside Interceptly

Call `apply-label` with the current thread + `label = "Booked"`.
This is the path that mirrors the state change into Interceptly.

If `apply-label` fails (thread not open, session expired, UI
bug), write _errors.md and proceed anyway — the Leads row is
still updated. The operator can apply Booked manually in
Interceptly; the mirror is already consistent.

### Step 4 — Cancel pending Interceptly tasks for this lead

Every `Tasks` row where `lead_url` matches AND `status =
pending`. Flip:

- `status = cancelled`
- `completed_at = now`
- `cancel_reason = booked`

This covers `follow-up`, `review-reply`, `book-meeting`
(important: if the client booked manually while an automated
`book-meeting` task was pending, cancel it so the bot doesn't
double-book).

Also mark those tasks done inside Interceptly via Chrome MCP
if the thread is still navigable. If not, leave them for
`process-my-tasks` to reconcile on the next run.

### Step 5 — Write a Replies row

- `ts = now`
- `lead_url`
- `thread_id` (from the Leads row if known)
- `campaign_slug` (from the Leads row)
- `raw_text = "[BOOKED — source=<source>] <notes or ''>"`
- `classification = booked`
- `handoff_state = closed`

### Step 6 — Log

Append to `/05_published/outreach/<today>.md`:

> `mark-booked — <lead_url> (<campaign_slug>) booked
> <meeting_datetime> via <source>`

### Step 7 — Return

A short summary for the caller: lead, campaign, meeting time,
number of tasks cancelled, label applied result.

## Idempotency

- A second `mark-booked` call for the same lead must be a no-op
  (refuse with "already booked"), not a silent re-flip. This
  prevents the `book-meeting` → `mark-booked` path from
  colliding with a client's parallel manual call.

## Client-led path quirks

When `booking_mode=manual`, this skill is how Booked state
enters the system. The client typically calls it from Cowork:

> "I booked Jane for next Tuesday at 11am."

Parse the lead reference ("Jane") against the `Leads` sheet. If
multiple matches, use `AskUserQuestion` to disambiguate. Parse
the datetime; if ambiguous ("next Tuesday" without a time),
confirm before flipping state.

## What NOT to do

- Do not send a message from this skill. Booked is booked.
- Do not move the Leads row to another campaign. A booked lead
  stays with the campaign that sourced them.
- Do not fabricate `meeting_datetime`. Empty is honest; a
  timestamp you invented is not.
- Do not remove any row. History matters.
- Do not skip the label-apply step just because the Leads row
  flipped. Interceptly is the source-of-truth; the label has to
  go there too.
