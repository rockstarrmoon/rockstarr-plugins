---
name: detect-accepts
description: "This skill should be used in the daily outreach loop after preview-queue, or when the user says \"detect new accepts\", \"check who accepted my connection requests\", or \"scan for new connections\". It checks Sales Navigator and LinkedIn notifications via Chrome MCP for newly accepted connection requests, matches each accepted connection to a Leads row, and flips Leads.state to accepted (with date_accepted). For full-sequence campaigns, chains directly into generate-message-tasks so the 3-step sequence starts on the same run; for connect-only campaigns, the accept is the terminal state and no message tasks are seeded."
---

# detect-accepts

Finds newly accepted connection requests and moves those leads
forward in the pipeline.

## When to run

- Daily loop, after `preview-queue`, before `generate-message-tasks`.
- On-demand when the user asks "did anyone accept today?"

## Preconditions

- `confirm-session` passed in this run.

## Inputs

- `Leads` sheet filtered to `state = connected` (leads with a sent
  connect request that have not yet accepted or replied).
- `Connections` sheet (to know which connection notes were sent, in
  case we want to cross-check recency).

## Behavior

1. **Open the LinkedIn notifications page** via Chrome MCP at
   `https://www.linkedin.com/mynetwork/` (and the Sales Nav
   notifications panel if the client uses it). Both surfaces show
   "X accepted your invitation" events.
2. **Parse accept events.** For each event, extract the profile
   URL of the person who accepted.
3. **Match to Leads.** Find the Leads row whose `lead_url` matches.
   - No match → log to `_errors.md` as a low-severity note: "accept
     notification for URL not in Leads; likely a non-campaign
     personal connect." Continue.
   - Match with `state != connected` → skip (already processed or
     moved past this state). Log at debug level only.
   - Match with `state = connected` → proceed.
4. **Flip Leads.** `state = accepted`, `date_accepted = today`. This
   happens for BOTH campaign types — even connect-only leads get
   the visibility flip so the workbook and weekly report can count
   accepts.
5. **Save the workbook.**
6. **Look up each newly-accepted lead's campaign_type** (read the
   matching Campaigns row). Partition the newly-accepted list:
   - `full_sequence_accepts` — leads in campaigns with
     `campaign_type=full-sequence`. Pass to
     `generate-message-tasks` so the 3-step sequence is seeded.
   - `connect_only_accepts` — leads in campaigns with
     `campaign_type=connect-only`. The accept is the terminal
     state; do NOT call `generate-message-tasks` for these. They
     simply sit at `state=accepted` and feed the campaign's
     accept-rate metric.
7. **Return** the structured summary below to the caller.

## Output

Structured summary:

- `newly_accepted` — list of `{lead_url, campaign_slug,
  campaign_type, accepted_on}`. Includes both full-sequence and
  connect-only.
- `full_sequence_accepts` — count of newly-accepted leads whose
  campaign is full-sequence (these chained into
  `generate-message-tasks`).
- `connect_only_accepts` — count of newly-accepted leads whose
  campaign is connect-only (these did not chain into
  `generate-message-tasks` — their accept is terminal).
- `ignored_non_campaign` — count of accept events that didn't match
  any campaign lead
- `already_processed` — count

## What NOT to do

- Do not send a "thanks for connecting" message from this skill. The
  3-step sequence is what the client approved; that's what ships.
  `generate-message-tasks` seeds those tasks (full-sequence
  campaigns only).
- Do not invent accepts. LinkedIn's notification feed is the only
  source of truth here.
- Do not retroactively backdate `date_accepted` if the notification
  feed paginated past the event. Record today's date; note in
  `_errors.md` if the feed looked truncated.
- Do not call `generate-message-tasks` for leads in connect-only
  campaigns. Their accept is the terminal state. The partition in
  Step 6 is the gate — respect it.
- Do not silently treat a connect-only lead as full-sequence if the
  Campaigns row's `campaign_type` is missing or unrecognized.
  Default to `full-sequence` (back-compat with pre-0.1.7
  registrations) and log a low-severity note to `_errors.md` so
  the operator can fix the row.
