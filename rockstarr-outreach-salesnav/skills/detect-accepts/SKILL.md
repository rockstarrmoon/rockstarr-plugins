---
name: detect-accepts
description: "This skill should be used in the daily outreach loop after preview-queue, or when the user says \"detect new accepts\", \"check who accepted my connection requests\", or \"scan for new connections\". It checks Sales Navigator and LinkedIn notifications via Chrome MCP for newly accepted connection requests, matches each accepted connection to a Leads row, and flips Leads.state to accepted (with date_accepted). Chains directly into generate-message-tasks so the 3-step sequence starts on the same run."
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
4. **Flip Leads.** `state = accepted`, `date_accepted = today`.
5. **Save the workbook.**
6. **Return** the list of newly accepted `(lead_url, campaign_slug)`
   tuples to the caller so `generate-message-tasks` can act on them.

## Output

Structured summary:

- `newly_accepted` — list of `{lead_url, campaign_slug, accepted_on}`
- `ignored_non_campaign` — count of accept events that didn't match
  any campaign lead
- `already_processed` — count

## What NOT to do

- Do not send a "thanks for connecting" message from this skill. The
  3-step sequence is what the client approved; that's what ships.
  `generate-message-tasks` seeds those tasks.
- Do not invent accepts. LinkedIn's notification feed is the only
  source of truth here.
- Do not retroactively backdate `date_accepted` if the notification
  feed paginated past the event. Record today's date; note in
  `_errors.md` if the feed looked truncated.
