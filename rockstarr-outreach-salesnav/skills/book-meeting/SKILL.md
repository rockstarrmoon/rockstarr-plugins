---
name: book-meeting
description: "This skill should be used when a book-meeting task fires in the daily loop, or when the user says \"book the meeting\", \"run the booking-link form\", or \"book this lead on the agreed time\". It runs only when stack.md.booking_mode is automated + availability_source is booking_link. It drives the booking-link page via Chrome MCP: selects the agreed time slot, fills the required fields (email, phone, whatever booking_link_required_fields lists), submits, and on success calls mark-booked. On failure (slot taken between proposal and submit, form changed, network error), it writes to _errors.md and creates a new review-reply task with the failure context so the client can recover."
---

# book-meeting

Automated booking path. Drives the booking link form on the lead's
behalf after the conversation converged on a specific time. Not
triggered by polling — triggered by the `book-meeting` task that
`draft-reply` created when it judged the thread was ready.

## When to run

- A `book-meeting` Tasks row is due.
- `stack.md.booking_mode = automated` AND
  `stack.md.availability_source = booking_link`. If either is not
  set, refuse — this skill's contract requires both.

## Preconditions

- Recent `confirm-session` pass.
- Booking link is reachable via Chrome MCP.
- The task's metadata includes:
  - `lead_url`
  - `campaign_slug`
  - `agreed_time` — ISO-8601 datetime the lead confirmed
  - `lead_fields` — object with the fields named in
    `stack.md.booking_link_required_fields` (e.g., `{email, phone,
    company}`)

If `lead_fields` is incomplete, refuse — `draft-reply` should only
have created a `book-meeting` task once every required field was in
hand. Log and drop a `review-reply` task asking the client to
collect the missing field.

## Behavior

1. **Open the booking link** via Chrome MCP at
   `stack.md.booking_link_url`.
2. **Navigate to the agreed time.** Most booking pages (Calendly,
   GrowthAmp, SavvyCal, etc.) let you pass date/time as URL query
   parameters or pick from a calendar. Prefer the URL-param path
   when the platform supports it; otherwise click through.
3. **Verify the slot is still available.** Read the page's confirm
   UI. If the slot shows as unavailable or the page surfaces a
   "slot no longer available" message:
   - Log to `_errors.md`: `slot_taken <lead_url> <agreed_time>`
   - Mark the `book-meeting` task `cancelled` with reason
     `slot_taken`.
   - Create a `review-reply` task for this lead with metadata
     `{reason: "slot_taken_repropose", prev_agreed_time: ...}` so
     the client can propose a new time.
   - Return `{booked: false, reason: "slot_taken"}`.
4. **Fill the form.** For each field in
   `stack.md.booking_link_required_fields`, paste the matching
   value from `lead_fields`. If the form asks for a field the stack
   config did not list:
   - Try a best-effort pull from Leads (`name`, `company`, `title`)
     — but only for those three known columns.
   - Otherwise: log to `_errors.md` as "form field changed, needs
     stack.md update," mark the task `cancelled` with reason
     `form_changed`, and drop a `review-reply` so the client can
     book manually.
5. **Submit.** Click the submit button. Wait for the confirmation
   screen.
6. **Verify success.** The booking page must confirm the booking
   (usually a "You're booked for X" screen). If instead the page
   shows an error, times out, or returns to the calendar view:
   - Log to `_errors.md` as `submit_failed` with the error text.
   - Mark the task `cancelled` with reason `submit_failed`.
   - Create a `review-reply` task with metadata
     `{reason: "booking_submit_failed"}`.
   - Return `{booked: false, reason: "submit_failed"}`.
7. **On success: call `mark-booked`.** Pass:
   - `lead_url`
   - `meeting_datetime` = `agreed_time`
   - `source` = `automated`
   `mark-booked` flips Leads.state, cancels sibling tasks, and
   writes the Replies row with `classification = booked`. This is
   the single source of truth — do NOT mutate Leads.state directly
   from this skill.
8. **Mark the `book-meeting` task done.** After `mark-booked`
   returns. `status = done`, `completed_at = now`.
9. **Log.** Append to `/05_published/outreach/<today>.md`:
   `book-meeting — booked <lead_url> for <agreed_time> via <booking_link_url>`.

## Output

- `{booked: true, meeting_datetime}` on success.
- `{booked: false, reason}` on failure (with a companion
  `review-reply` task already created).

## What NOT to do

- Do not book a different slot from the one the lead agreed to.
  Mismatched bookings destroy trust.
- Do not run when `booking_mode = manual`. That path is client-led
  via `mark-booked`.
- Do not retry a failed submit. Fail fast and surface to the client.
  A silent retry can double-book or send the form through twice.
- Do not ever hand the booking link to the lead. The booking link
  is a destination the bot uses — it's not message copy.
- Do not update Leads.state from this skill. Always funnel through
  `mark-booked`.
