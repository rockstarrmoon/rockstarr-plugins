---
name: propose-meeting-times
description: "This skill should be used when the user or another skill (typically rockstarr-reply's draft-reply) asks to \"propose meeting times\", \"find free slots for the lead\", \"pull availability\", or judges that a Sales Nav thread is ready for a meeting ask. It reads the client's availability source — either the booking-link page via Chrome MCP or Google Calendar via the shared calendar helper — and returns 2 to 3 proposed time slots in the next few business days. Optionally biases the slots by lead-provided context (e.g., \"Tuesdays after 3pm\"). Shared skill; ships inline in rockstarr-outreach-salesnav for V0.1 and will migrate to rockstarr-infra/skills/_shared/."
---

# propose-meeting-times

Return 2–3 concrete proposed time slots for the drafting skill to
offer the lead. Never returns a booking link. Never books on its own.

> **Shared-skill notice.** Canonical source will live at
> `rockstarr-infra/skills/_shared/propose-meeting-times/` once more
> than one plugin consumes it. For V0.1 it ships inline in
> `rockstarr-outreach-salesnav`. Keep copies in sync across plugins.

## When to run

- `rockstarr-reply:draft-reply` decides a thread has reached a
  meeting-ask moment.
- User explicitly asks to "propose times" for a named lead.

## Inputs

- `lead_url` — LinkedIn profile URL (primary key in Leads).
- `campaign_slug` — so we can read any campaign-specific constraints.
- Optional `lead_context` string — free-form bias hints the lead gave
  in-thread ("Tuesdays after 3", "mornings only", "next week").
- `stack.md`:
  - `availability_source: booking_link | gcal`
  - `booking_link_url` (if `booking_link`)
  - `gcal_id` (if `gcal`)
  - `booking_link_required_fields` (informational — used by the reply
    draft, not this skill)

## Behavior

### availability_source = booking_link

1. Open `booking_link_url` in Cowork's Chrome session via Chrome MCP.
2. Parse the page's visible availability calendar for the next 5
   business days.
3. Prefer 3 non-adjacent slots across at least 2 different days.
4. If `lead_context` biases the window (e.g., "afternoons"), filter to
   matching slots first; fall back to the original window if fewer
   than 2 match.
5. Return ISO-8601 datetimes in the client's timezone (from
   `stack.md`), plus a short human-readable string each
   ("Wed Apr 29, 2:00 PM CT").

### availability_source = gcal

1. Call the shared Google Calendar helper for the calendar identified
   by `gcal_id`.
2. Pull free/busy for the next 5 business days between the client's
   configured office hours (default 9am–5pm local).
3. Propose 3 slots of 30 minutes each, non-adjacent, across at least
   2 days. Honor `lead_context` as in the booking-link path.
4. Return the same shape (ISO-8601 + human string).

### Error handling

- Chrome MCP cannot open the booking link, or the calendar helper
  errors → write to `/02_inputs/outreach/_errors.md` with the failure,
  return an empty list, and let the caller decide what to do (usually
  the reply draft will ask the client for manual times).
- Calendar shows no free slots in the 5-day window → widen to 10
  business days before giving up.

## Output shape

Return a list, 2–3 items long, each of the form:

```json
{
  "iso": "2026-04-29T19:00:00Z",
  "local": "2026-04-29T14:00:00-05:00",
  "display": "Wed Apr 29, 2:00 PM CT",
  "source": "booking_link | gcal"
}
```

Do **not** write to the workbook from this skill. Do **not** create
tasks. The caller composes the reply; the caller decides when to
create a `book-meeting` task.

## What NOT to do

- Do not return the booking link URL. Ever. Consumers must not see
  it. The reply proposes times, not a link.
- Do not book on the lead's behalf. Booking is `book-meeting`'s job.
- Do not assume timezone. Use `stack.md`'s client timezone.
- Do not propose more than 3 slots. Choice paralysis is real and
  three is plenty.
- Do not propose slots inside the next 2 hours. Leads need lead time.
