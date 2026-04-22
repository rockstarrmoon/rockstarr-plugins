---
name: mark-booked
description: "This skill should be used whenever a lead has booked a meeting — whether the bot booked it via book-meeting, or the client booked it manually by phone, email, or directly through their calendar. Trigger phrases: \"mark this lead as booked\", \"Jane booked a meeting\", \"I booked this lead manually\", \"log the booking\". This is the single source of truth for booking state: it flips Leads.state=booked, cancels every pending task for that lead, and writes a Replies row with classification=booked. Both the automated and manual booking paths converge here."
---

# mark-booked

The single source of truth for "a meeting got booked." Every path
— automated `book-meeting` success, client-booked manually by phone,
client-booked via their own calendar link — funnels through here so
state transitions stay consistent.

## When to run

- `book-meeting` calls it on successful form submission.
- Client calls it directly from Cowork: "I booked Jane manually for
  Friday at 2."
- Client calls it when Google Calendar or the booking link shows an
  event they recognize belongs to a campaign lead.

## Inputs

- `lead_url` — required.
- `meeting_datetime` — optional ISO-8601. If omitted, store as
  `meeting_datetime = "unspecified"`.
- `source` — `automated` (from `book-meeting`) or `manual` (from the
  client). Defaults to `manual` if not supplied.
- Optional `notes` — free text the client wants captured in the
  Replies row.

## Preconditions

- `02_inputs/outreach/outreach-tasks.xlsx` exists.
- Leads has a row for `lead_url`. If not, this is either a
  non-campaign booking (fine, just skip with a note to the user) or
  a data-entry error (tell the user which campaigns contain a lead
  with a similar name and ask them to confirm).

## Behavior

1. **Find the Leads row.** If `state = booked` already, refuse with
   a clear message: "Jane is already marked booked on
   `<date_booked>`. No action taken." Do not double-book history.
2. **Flip Leads.state.** `state = booked`, `date_booked = today`.
   If `meeting_datetime` was provided, store it in
   `Leads.meeting_datetime` (a nullable column; fine if blank
   otherwise).
3. **Cancel pending tasks for this lead.** Every Tasks row where
   `lead_url` matches and `status = pending`. Flip
   `status = cancelled`, `completed_at = now`, `cancel_reason =
   booked`. This covers `message-step-N`, `follow-up`,
   `review-reply`, `book-meeting` (important — if the client booked
   manually while an automated `book-meeting` task was pending,
   cancel it so the bot doesn't double-book).
4. **Write a Replies row.**
   - `date` = now (ISO)
   - `lead_url`
   - `campaign_slug` = from the Leads row
   - `raw_text` = `"[BOOKED — source=<source>] <notes or ''>"`
   - `classification` = `booked`
   - `handoff_state` = `closed`
5. **Save the workbook.**
6. **Log.** Append to `/05_published/outreach/<today>.md`:
   `mark-booked — <lead_url> (<campaign_slug>) booked <meeting_datetime> via <source>`.
7. **Return.** A short summary the caller can show the user: lead,
   campaign, meeting time, number of tasks cancelled.

## Idempotency

- A second `mark-booked` call for the same lead must be a no-op
  (refuse with "already booked"), not a silent re-flip. This
  prevents the `book-meeting` → `mark-booked` path from colliding
  with a client's parallel manual call.

## What NOT to do

- Do not send any message. Booked is booked; the conversation is
  complete for this campaign.
- Do not move the Leads row to another campaign. A booked lead
  stays with the campaign that sourced them; that's how attribution
  stays clean.
- Do not fabricate `meeting_datetime`. If the client omits it, the
  column stays empty. We'd rather have honest ambiguity than
  invented timestamps in the metrics data.
- Do not remove the Leads row. History matters.
