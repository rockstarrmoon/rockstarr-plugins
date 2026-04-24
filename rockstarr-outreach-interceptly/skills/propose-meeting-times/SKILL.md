---
name: propose-meeting-times
description: "This skill should be used when draft-reply-interceptly decides a thread is Hot and needs meeting times, or when the user says \"propose meeting times\", \"find 2-3 slots for <lead>\", or \"pull availability\". Reads the client's availability source — either the booking-link page (Calendly / GrowthAmp / similar) via Chrome MCP, or Google Calendar via the shared calendar helper — and returns 2-3 proposed time slots in the next few business days. Optionally biases slots by lead-provided context ('Tuesdays after 3pm'). Shared skill; canonical source will migrate to rockstarr-infra/skills/_shared/ when both outreach variants and reply ship."
---

# propose-meeting-times

Shared skill. Canonical source will live at
`rockstarr-infra/skills/_shared/propose-meeting-times/` once the
shared tree is populated. For V0.1 it ships inline in
`rockstarr-outreach-interceptly`. If you edit this file, edit the
matching copy in `rockstarr-outreach-salesnav` (and eventually
`rockstarr-reply`) in the same PR.

Returns a small slate of specific times the approved reply can
propose to the lead. The booking link is never pasted — the bot
names times, the lead picks one, and either `book-meeting` (bot-
led) or `mark-booked` (client-led) records it.

## When to run

- Called by `draft-reply-interceptly` when the thread is Hot
  (verdict=target + bucket=hot, or any bucket with
  `sub_type=meeting_proposed`).
- On demand when the user says "propose 2-3 times for `<lead>`"
  — useful for manual re-qualification of a draft.

## Preconditions

- `/00_intake/stack.md` has `booking_mode` and
  `availability_source` set. If missing, refuse and point at
  `rockstarr-infra:capture-stack`.
- Supported `availability_source` values:
  - `booking_link` — the client's booking page
    (Calendly / GrowthAmp / similar). `booking_link_url` must be
    set.
  - `gcal` — Google Calendar read via the shared calendar
    helper. `gcal_id` must be set.

## Inputs

- `lead_context` — optional. Free-text preferences the lead
  named ("afternoon works best", "Tuesdays after 3pm").
- `timezone` — optional. If the lead's timezone is known,
  convert the slate accordingly. Default: client's local.
- `horizon_days` — default 5 business days. Do not look further
  than 10.

## Behavior

### Step 1 — Branch on availability_source

#### Branch A — booking_link

1. Navigate Chrome MCP to `stack.md.booking_link_url`.
2. Wait for the calendar widget to render.
3. Read the next `horizon_days` of business days. Extract
   available time slots visible on the page. Do NOT click any
   slot — this is read-only.
4. Normalize: each slot = `{start_iso, end_iso, timezone}`.

#### Branch B — gcal

1. Call the shared calendar helper to fetch the primary
   calendar's free/busy for `horizon_days` of business days.
2. Apply the client's working-hours window (defaults 9am-5pm
   local, respecting weekends; overridable via
   `stack.md.calendar_working_hours`).
3. Derive free slots of the client's default meeting length
   (defaults 30 minutes; overridable via
   `stack.md.default_meeting_length_minutes`).

### Step 2 — Apply lead preferences

If `lead_context` names a preference (afternoon, Tuesdays,
after-3pm, etc.), filter the free-slot list to matching slots.
If filter returns zero, fall back to unfiltered and note that
the preference couldn't be met.

### Step 3 — Pick 2-3 slots

Select 2 or 3 slots spread across the horizon — do NOT return
three consecutive slots on the same day. Rough rule:

- Slot 1: earliest acceptable (today or tomorrow if possible).
- Slot 2: middle of horizon (3-4 business days out).
- Slot 3: later (5 business days out) — OPTIONAL. Only include
  if slot 2 is less than 48 hours away; otherwise 2 slots is
  plenty.

### Step 4 — Format for the draft

Return a list the draft can interpolate:

```
1. Tuesday, May 5 at 10:00 AM ET
2. Wednesday, May 6 at 2:30 PM ET
3. Friday, May 8 at 11:00 AM ET
```

Use the lead's timezone if known; otherwise the client's. Include
the timezone abbreviation in the human-readable string — leads
routinely read a bare time as their own tz and miss by hours.

### Step 5 — Return

```
{
  slots: [
    {start_iso, end_iso, timezone, display_string},
    ...
  ],
  source: "booking_link" | "gcal",
  preference_applied: true | false,
  notes: "<any fallbacks the caller should know>"
}
```

### Step 6 — Log

Append a row to the `MeetingProposals` sheet of
`outreach-mirror.xlsx` with ts, thread_id, lead_url,
slot_count, source, preference_applied.

## Semantics

- The slots are PROPOSALS, not holds. Neither branch writes
  to the source. If a slot is booked between proposal and the
  lead's reply, `book-meeting` fails loudly and `draft-reply`
  regenerates the draft with fresh slots.
- For `booking_mode=automated`, the next step is `book-meeting`
  when the lead picks one and supplies required fields.
- For `booking_mode=manual`, the next step is the client
  running `mark-booked` when they see the lead accept a slot.

## Failure modes

- **booking_link page didn't render.** Retry once with a 3s
  wait. On second failure, return `source: "unavailable"` and
  let `draft-reply-interceptly` fall back to "what times work
  for you this week?" language.
- **gcal helper returns zero free slots in horizon.** Extend
  horizon to 10 business days and retry. If still zero, return
  `source: "unavailable"`.
- **Lead preference filter returns zero slots.** Fall back
  unfiltered; note preference-met = false.

## What NOT to do

- Do not paste the booking link URL anywhere. This skill
  returns times, not links.
- Do not click slots. Reading the availability source is
  read-only here.
- Do not propose a slot less than 12 hours out unless
  stack.md explicitly allows short-notice proposals.
- Do not return more than 3 slots. The draft body gets noisy
  past three.
