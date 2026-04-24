---
name: book-meeting
description: "This skill should be used when a book-meeting task fires in the daily loop (after the lead agreed to a specific time AND supplied the booking-link-required fields), or when the user says \"book the meeting for <lead>\", \"run the booking-link form\", or \"book this lead on the agreed time\". Runs only when stack.md.booking_mode=automated and availability_source=booking_link. Drives the booking-link page via Chrome MCP: selects the agreed slot, fills required fields, submits. On success calls mark-booked. On failure writes to _errors.md and creates a review-failure note so the operator can recover."
---

# book-meeting

Bot-led booking path. Runs AFTER `draft-reply-interceptly`
confirms the lead has agreed to a specific slot AND the booking-
link form's required fields are all supplied. Never guesses.

## When to run

- `book-meeting` task fires in the daily loop (created by
  `create-followup-task` via `draft-reply-interceptly` Hot path).
- On demand when the user says "book `<lead>` at `<time>`" — in
  which case the operator is providing the agreed slot explicitly.

## Preconditions

- `stack.md.booking_mode = automated`.
- `stack.md.availability_source = booking_link`.
- `stack.md.booking_link_url` is set.
- `stack.md.booking_link_required_fields` enumerates the form's
  required fields (default: `[email, phone]`; varies per booking
  link).
- The lead's thread has a `book-meeting` task with:
  - `agreed_start_iso` — the slot the lead agreed to
  - `lead_fields` — a map of every required field → value
    provided by the lead
- `confirm-session-interceptly` passed in this run.

If `booking_mode=manual` or `availability_source=gcal`, refuse
and route the caller to `mark-booked` instead (the client books
manually in that case).

## Inputs

- `thread_id`
- `lead_url`
- `agreed_start_iso` — the slot
- `lead_fields` — `{email: "...", phone: "...", ...}`

## Behavior

### Step 1 — Open the booking link

Navigate Chrome MCP to `stack.md.booking_link_url`. Wait for the
calendar widget to render.

### Step 2 — Select the slot

Navigate to the date and time in `agreed_start_iso`. Click the
slot. If the slot is no longer available (taken between proposal
and this run), abort with reason `slot_taken`:

- Write to `_errors.md`.
- Flip the task status to `stale`.
- Create a new `review-reply` task for the lead with a note
  explaining the race.
- Do NOT attempt to pick a different slot — the lead agreed
  to this one specifically.

### Step 3 — Fill the form

The booking page transitions to a form. Fill every field listed
in `stack.md.booking_link_required_fields` using `lead_fields`.

For each field not in `lead_fields`:

- If the field is optional, leave blank.
- If the field is required but missing from `lead_fields`, abort
  with reason `missing_required_field` and the name of the
  missing field. Write _errors.md. The pipeline should have
  prevented this; flag as a bug to investigate.

### Step 4 — Submit

Click Submit / Confirm / Schedule (text match). Wait for the
success state. Typical success signals: confirmation page,
"booked" text, redirect to a thank-you URL.

If no success signal within 10 seconds, inspect the DOM for an
error message. Write _errors.md with the message; abort.

### Step 5 — On success, call mark-booked

Call `mark-booked` with:

- `lead_url`
- `meeting_datetime = agreed_start_iso`
- `source = automated`
- `notes = "Booked via book-meeting against <booking_link_url>"`

`mark-booked` handles the state transition (Leads row, pending
task cancellation, Replies row, Booked label via apply-label).

### Step 6 — Complete the Interceptly task

Mark the `book-meeting` task done in Interceptly. Mirror
completion in the `Tasks` sheet.

### Step 7 — Return

`{booked: true, meeting_datetime: "<ISO>"}`

## Failure modes

- **Slot taken.** See Step 2. Do NOT pick an alternate.
- **Form requires a field not in `booking_link_required_fields`.**
  Write _errors.md, abort. Operator updates stack.md and retries.
- **Form has a captcha.** Abort. The bot doesn't solve captchas;
  route to `mark-booked` for the operator to handle manually.
- **Submit succeeds but no confirmation signal.** Wait another
  5s. If still ambiguous, write _errors.md and ABORT without
  calling `mark-booked`. A false-positive booking is worse than
  a missed one — the operator can rerun.

## Idempotency

- A `book-meeting` task whose mirror row already has
  `status = completed` is a no-op (refuse with "already booked").
- If `mark-booked` returns `already_booked`, still mark the
  Interceptly task done.

## What NOT to do

- Do not pick an alternate slot if the agreed slot is gone. The
  lead agreed to a specific time; anything else is a new
  conversation.
- Do not fabricate field values. Every required field comes from
  the lead's explicit supply.
- Do not call `mark-booked` on a submit that didn't visibly
  succeed.
- Do not run when `booking_mode=manual`. The client books; the
  bot doesn't.
