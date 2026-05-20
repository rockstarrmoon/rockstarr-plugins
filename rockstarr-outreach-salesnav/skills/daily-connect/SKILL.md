---
name: daily-connect
description: "This skill should be used as the final send-step of the daily outreach loop, or when the user says \"run today's connects\", \"send today's Sales Nav connection requests\", or \"execute the connect loop\". It computes the day's budget as min(20 + carry_forward_this_week, 100 − connections_sent_this_week), picks the highest-priority un-contacted leads across all active campaigns (full-sequence + connect-only, round-robin by campaign), and for each lead clicks the name first to register a profile view in the Sales Nav side preview (so the lead drops off tomorrow's already-viewed-filtered saved search) BEFORE opening the row-level three-dot menu to send a BLANK connect request through Chrome MCP. Applies the three skip rules (Connect–Pending / 1st-degree / no-Connect-option) — skipped leads still get the profile-view-first treatment, so they drop off tomorrow's queue too. Logs to Connections and marks connect tasks done. This is the ONLY skill allowed to send connection requests in this plugin."
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

The canonical UI path is **click the lead's name first** (to
register a profile view in the side preview, which removes them
from tomorrow's already-viewed-filtered saved search), THEN the
**row-level three-dot menu** on each search result card. NOT the
side panel's overflow button — that tends to navigate to the full
profile page instead of opening a dropdown. The row-level button
reliably exposes Connect / View profile / Add to map. Locate both
elements via Chrome MCP `find` with the lead's name and click by
ref ID rather than by pixel coordinate — viewport changes will
move absolute coordinates but accessible-name lookups stay stable.

The operator's saved searches are configured to hide profiles the
account has already viewed, so each day's queue is clean.
Skipping the profile-view click keeps already-touched leads in
tomorrow's results and corrupts the "start at the top" workflow.

For each lead selected:

1. **Pre-flight the count.** If the cumulative send this run would
   push `connections_sent_this_week` past 100, stop immediately.
   Do not send one more than the ceiling.
2. **Locate the lead in the Sales Nav search result list.** If
   you're working from a specific saved-search results page, the
   lead's row should be visible. If you've navigated past the lead's
   row (or the lead is not on the current page), paginate via the
   "Next" button or open the lead's row by URL — but always operate
   on the row-level UI, not the full-profile UI.
3. **Click the lead's name to register a profile view.** Use
   Chrome MCP `find` to locate the lead's name link in the search
   result row (query the lead's name as the accessible name on an
   `<a>` element inside the result card). Click it by ref ID. The
   right-side preview panel populates with the lead's profile —
   confirm by `find`ing a recognizable side-panel element (e.g.,
   the side-panel's headline or the "Send InMail" button that
   appears at the top of the preview) before continuing. Do NOT
   skip this step. The profile-view registration is what removes
   this lead from tomorrow's already-viewed-filtered saved search,
   and it happens whether or not the connect goes through — so
   leads that hit any of Step 4's skip cases still drop off the
   next-day queue. If clicking the name accidentally navigates
   away from the search results page (some Sales Nav UI variants
   route the name link to the full profile page instead of the
   side preview), use the browser back button to return to the
   results page; the navigation itself counts as a profile view,
   so the dedup goal is still met.
4. **Open the row-level three-dot menu.** Use Chrome MCP `find` with
   `"See more actions for [Lead's Name]"` and click by ref ID. A
   dropdown appears. Three skip cases the dropdown surfaces:

   - **"Connect" is clickable** — proceed to step 5.
   - **"Connect — Pending"** (grayed out) — the lead already has a
     pending invitation from a prior run. Press Escape to close
     the dropdown, log a skip with reason
     `connect_pending_already_invited` to `_errors.md`, do NOT
     decrement the budget, do NOT mark the connect task done (it's
     still pending in our state if we're surprised), continue.
   - **No "Connect" option appears** (only "View profile," "Add to
     map," etc.) — the profile has restrictions or the lead is
     1st-degree already (already connected). Log a skip with reason
     `no_connect_option`, flip Leads.state to `connected` if the
     lead is 1st-degree (the row UI sometimes shows "Message"
     instead of "Connect" for 1st-degrees), otherwise leave
     `state=queued` for a future retry, do NOT decrement the
     budget, continue.

   In every skip case above, the profile view from Step 3 has
   already registered, so the lead drops off tomorrow's queue
   regardless. That is the point — skipped leads should not
   re-surface tomorrow either.

   Side-panel overflow can open the full profile page instead of a
   dropdown. If you find yourself on a profile page after clicking
   the three-dot, you used the wrong button — back out and use the
   row-level one.

5. **Click Connect.** From the row-level dropdown, click "Connect."
   The "Send invitation" dialog opens.
6. **Verify "Save as lead" is unchecked** in the dialog. Row-level
   sends typically have it unchecked by default; first send of a
   session may have it checked if the prior session opened a full
   profile. Uncheck it before sending — checking it auto-saves the
   lead into Sales Nav's lead-list which is not what we want for a
   connect-only or full-sequence campaign.
7. **Send the request WITHOUT a note.** Message 1 in the approved
   campaign is intentionally BLANK — LinkedIn's default connect
   behavior is the cleanest first touch and keeps Message 2 free to
   do the real work (full-sequence campaigns) or simply minimizes
   the spam-flag risk (connect-only campaigns). Do NOT attach a
   note, even if one is present in the campaign file (it should not
   be). If the campaign's Message 1 section contains any body copy,
   treat it as a config error: log to `_errors.md` asking the
   client to re-approve the campaign with Message 1 blank, skip
   this lead, and do not decrement the budget. Click the blue
   "Send Invitation" button.
8. **Verify the send.** Wait for the confirmation UI. If the
   request fails or the UI does not confirm, log to `_errors.md`
   and skip without decrementing the budget.
9. **Write to Connections.**
   - `date` = today, ISO
   - `lead_url`
   - `campaign_slug`
   - `note_sent` = `""` (always empty — Message 1 is blank by spec)
   - `source` = `sales_nav`
10. **Update Leads.** `state = connected`, `date_connected = today`.
11. **Mark the Tasks row done.** `status = done`, `completed_at = now`.
12. **Polite rate.** Wait 25–45 seconds (randomized) between sends.
    LinkedIn rate-limits aggressive clicking far below 20/day if
    they all hit in the same minute.

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
- Do not skip the click-the-name-first step (Step 3 of the Send
  loop). The operator's saved searches are configured to hide
  already-viewed profiles, so each day's queue is meant to start
  at the top with fresh names. Skipping the profile-view click
  leaves already-touched leads in tomorrow's results and breaks
  the operator's workflow. This applies to skipped leads too — a
  Connect-Pending or no-Connect-option lead still needs the
  profile view registered so it drops off the next-day queue.
- Do not open the row-level three-dot menu before clicking the
  lead's name. Even if the connect call goes through, skipping
  the profile-view-first ordering leaves the lead in tomorrow's
  saved-search results because LinkedIn does not register a view
  from the row-level overflow menu — only from a name-link click
  that loads the side preview.
