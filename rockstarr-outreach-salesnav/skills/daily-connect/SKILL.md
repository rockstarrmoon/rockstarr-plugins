---
name: daily-connect
description: "This skill should be used as the final send-step of the daily outreach loop, or when the user says \"run today's connects\", \"send today's Sales Nav connection requests\", or \"execute the connect loop\". It computes the day's budget as min(20 + carry_forward_this_week, 100 − connections_sent_this_week), picks the highest-priority un-contacted leads across active campaigns via round-robin by campaign, sends connect requests with the campaign-specific note via Sales Nav through Chrome MCP, logs to Connections, and marks connect tasks done. This is the ONLY skill allowed to send connection requests in this plugin."
---

# daily-connect

Enforces the global cap. Only skill in this plugin that actually sends
a Sales Nav connection request.

## When to run

- Last action of the scheduled daily loop, after `confirm-session`,
  `preview-queue`, `detect-accepts` → `generate-message-tasks`,
  `send-scheduled-messages`, `detect-replies`.
- On-demand from `force-send-today` (deferred V0.1).

## Preconditions

- `confirm-session` passed in this run (within the last 30 min).
- Sales Nav is reachable (Chrome MCP can navigate to it without a
  login prompt).

If `confirm-session` has not passed this run, refuse and return
`{status: "skipped", reason: "no confirmed session"}` to the caller.

## Budget math

- `week` = current ISO week in the client's timezone (from stack.md).
- `connections_sent_this_week` = count(Connections rows where
  iso_week == week).
- `daily_shortfall[d]` for each earlier day of the week =
  `max(0, 20 - sent_that_day)`.
- `carry_forward_this_week` = sum(daily_shortfall[d] for d in earlier
  days of week).
- `today_budget` = `min(20 + carry_forward_this_week, 100 - connections_sent_this_week)`.
- If `today_budget <= 0`, skip. Weekly cap is the hard ceiling; no
  overrides.
- Monday 00:00 local is a hard reset — unused weekly capacity does
  NOT roll into the next week.

## Selection + round-robin

1. Active campaigns = Campaigns rows with `status = active`.
2. If no active campaigns, return `{status: "skipped", reason: "no active campaigns"}`.
3. Split `today_budget` round-robin by campaign slug order:
   - 3 campaigns + budget 20 → `7 / 7 / 6`
   - 2 campaigns + budget 15 → `8 / 7`
   - 1 campaign + budget 20 → `20`
   If a campaign has fewer eligible leads than its quota, spill the
   remainder to the next campaign in the round-robin order.
4. Eligible lead = Leads row where:
   - `campaign_slug` matches the campaign
   - `state = queued` (never been contacted)
   - There is a pending `connect` Tasks row for this lead
   - The lead has not been sent a connect request previously (no
     `Connections` row referencing its URL)
5. Priority within a campaign: take leads in Leads-row order (i.e.,
   the order they came out of `crawl-lead-list`). V0.1 does not
   rank by signal strength.

## Send loop

For each lead selected:

1. **Pre-flight the count.** If the cumulative send this run would
   push `connections_sent_this_week` past 100, stop immediately.
   Do not send one more than the ceiling.
2. **Open the lead's profile.** Navigate Chrome MCP to the lead's
   LinkedIn URL in the Sales Nav panel.
3. **Click Connect.** Use Chrome MCP. If "Connect" is not available
   (e.g., the lead is out-of-network, already connected, or the UI
   changed):
   - Log the skip with a reason to `_errors.md`.
   - Do not decrement the budget; move to the next lead.
4. **Send the request WITHOUT a note.** Message 1 in the approved
   campaign is intentionally BLANK — LinkedIn's default connect
   behavior is the cleanest first touch and keeps Message 2 free to
   do the real work. Do NOT attach a note, even if one is present in
   the campaign file (it should not be). If the campaign's Message 1
   section contains any body copy, treat it as a config error: log to
   `_errors.md` asking the client to re-approve the campaign with
   Message 1 blank, skip this lead, and do not decrement the budget.
   Submit the connect request with no note.
5. **Verify the send.** Wait for the confirmation UI. If the request
   fails or the UI does not confirm, log to `_errors.md` and skip
   without decrementing the budget.
6. **Write to Connections.**
   - `date` = today, ISO
   - `lead_url`
   - `campaign_slug`
   - `note_sent` = `""` (always empty — Message 1 is blank by spec)
   - `source` = `sales_nav`
7. **Update Leads.** `state = connected`, `date_connected = today`.
8. **Mark the Tasks row done.** `status = done`, `completed_at = now`.
9. **Polite rate.** Wait 25–45 seconds (randomized) between sends.
   LinkedIn rate-limits aggressive clicking far below 20/day if they
   all hit in the same minute.

## Save + publish-log

After the loop:

1. Save the workbook via shared `xlsx`.
2. Append to `/05_published/outreach/<today>.md`:
   ```
   daily-connect — sent <N>/<today_budget> connects
     <slug-a>: <n_a> sent (<skipped_a> skipped)
     <slug-b>: <n_b> sent
   weekly cap: <used>/100 — <remaining> remaining
   ```

## Output

Return a structured summary to the caller:

- `today_budget`
- `sent` (total)
- `per_campaign` (map of slug → sent count)
- `skipped` (list of lead_url + reason)
- `weekly_used` after this run
- `weekly_remaining`

## What NOT to do

- Do not exceed 20 in a single calendar day under any circumstance.
- Do not exceed 100 in a single ISO week. Once 100 is hit, skip the
  rest of the week. Monday resets.
- Do not send connects to leads in `state != queued`.
- Do not send without a recent `confirm-session` pass.
- Do not attach a note to the connect request. Ever. Message 1 is
  spec'd blank so LinkedIn's default behavior ships. If you find
  yourself tempted to add "a quick intro" because the campaign feels
  cold, stop — the cadence is designed to do its work in Messages
  2–4, not in a 300-character note. If the approved campaign file
  contains body copy under Message 1, treat it as a config error and
  surface it to the client; do not silently send the note.
- Do not retry on LinkedIn UI anomalies. Skip the lead, log to
  `_errors.md`, move on. A silent retry masks UI breakage the
  weekly report needs to surface.
- Do not touch leads in paused campaigns.
