---
name: daily-connect
description: "This skill should be used as the final send-step of the daily outreach loop, or when the user says \"run today's connects\", \"send today's Sales Nav connection requests\", or \"execute the connect loop\". It computes the day's budget as min(20 + carry_forward_this_week, 100 − connections_sent_this_week), picks the highest-priority un-contacted leads across all active campaigns (full-sequence + connect-only, round-robin by campaign), and for each lead navigates directly to the lead's Sales Nav profile URL (the canonical lead_profile_overflow path; the visit itself registers the profile view that drops the lead off tomorrow's already-viewed-filtered saved search) and uses the lead-page overflow menu to send a BLANK connect request through Chrome MCP. Pacing is 60–90s per-lead jitter plus a FULL page refresh after every 3 successful sends — Sales Nav SPA JS state accumulates and a refresh is the only thing that reliably clears it. Applies the four skip rules (Connect–Pending / 1st-degree / no-Connect-option / requires-email — the last is terminal because the lead's privacy setting blocks blank invites). Logs to Connections and marks connect tasks done. This is the ONLY skill allowed to send connection requests in this plugin."
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

### UI path — `lead_profile_overflow` is canonical

Two paths exist for "open the row's overflow and click Connect":

- **`lead_profile_overflow` (canonical, Chrome MCP).** Navigate the
  same tab directly to the lead's Sales Nav profile URL
  (`/sales/lead/<urn>`, stored on the Leads row). The page loads
  the lead's full record. The "..." overflow at the top of that
  page exposes Connect / View profile / Add to map. **This is the
  only path that reliably works inside Chrome MCP today.** The
  direct navigation itself registers as a profile view in Sales
  Nav, so it satisfies the dedup goal (the lead drops off
  tomorrow's already-viewed-filtered saved search) as a side
  effect of how the path works — no separate "click the name
  first" step is needed.
- **`side_preview_overflow` (fallback, human-natural; degraded
  under Chrome MCP).** The path a human user actually clicks:
  open the saved-search results page, click the lead's name to
  populate the side preview pane on the right, click the "..." at
  the top of the preview, click Connect. Inside Chrome MCP this
  path is currently unreliable because Sales Nav virtualizes the
  saved-search result rows and lazy-loads them on scroll — the
  rows often don't hydrate within Chrome MCP's `Runtime.evaluate`
  window, so the bot can't see the name link to click. Keep this
  path documented as a fallback that the bot can attempt if
  `lead_profile_overflow` fails on a specific lead, and revisit
  promoting it to canonical only if Sales Nav fixes the
  virtualization timing.

The canonical / fallback split is also encoded in `stack.md`:

```yaml
daily_connect_path: lead_profile_overflow
daily_connect_fallback_path: side_preview_overflow
```

The bot reads those keys at run start. If they're missing, default
canonical = `lead_profile_overflow`, fallback = `side_preview_overflow`.

### Click pattern — pure JS, not element refs

Sales Nav's SPA invalidates element refs from `read_page` /
`browser_snapshot` mid-batch — by the third or fourth send in a
session, refs captured earlier in the run point at stale DOM nodes
and clicks silently fail. **Use a pure-JS one-shot click pattern as
the primary click method:** locate the target element inside
`javascript_tool` (or `mcp__Claude_in_Chrome__javascript_tool`)
by accessible name / aria-label / text content, dispatch the click
in the same JS execution, and read the resulting DOM state from the
same script's return value. Do not split "find ref" and "click ref"
across two MCP calls.

If a pure-JS click fails (element not present, role mismatch),
fall back to a fresh `find` + click — but never reuse a ref captured
more than one MCP call ago.

### Pacing — page refresh every 3 sends

Two-part rule, both required:

- **Per-lead jitter: 60–90 seconds, randomized.** LinkedIn's
  anti-spam heuristics flag accounts whose actions cluster too
  tightly. Older 25–45s pacing was too aggressive and increased
  send-not-confirmed failures on Sales Nav specifically.
- **Full page refresh after every 3 successful sends.** Navigate
  the tab to `https://www.linkedin.com/sales/`, wait 10s for the
  shell to settle, then continue with the next lead. The Sales Nav
  SPA accumulates JS state across rapid actions in the same browser
  session — modals stop opening, screenshots time out, send
  buttons hang. A passive cooldown (sleep without reloading) does
  not clear this; only a fresh page load does. Reset the
  successful-send counter to 0 after each refresh. This is the
  single most load-bearing change relative to V0.2.x — without it,
  daily-connect hits a hard ceiling around send #3–5 per session
  even with longer jitter.

### Per-lead procedure

For each lead selected:

1. **Pre-flight the count.** If the cumulative send this run would
   push `connections_sent_this_week` past 100, stop immediately. Do
   not send one more than the ceiling.
2. **Navigate to the lead's profile URL.** Read `lead_url` from the
   Leads row (it is a `/sales/lead/<urn>...` URL). Navigate the same
   tab there. Wait up to 15 seconds for the lead-page shell to
   render (title element, headline, the "..." overflow button
   present in the DOM). If the page does not render within 15
   seconds — body still shows the `.initial-load-animation` SPA
   loader, no headline — log this lead's URL + the timeout to
   `_errors.md` with reason `lead_page_did_not_render`, skip the
   lead, leave its task pending, run a single 90-second cooldown
   nav to `https://www.linkedin.com/sales/`, then resume with the
   next lead. The direct navigation itself is what registers as a
   profile view, so the dedup goal is met even when the page hangs
   on the loader.
3. **Inspect the degree badge.** Read the lead's degree (1st / 2nd
   / 3rd / out-of-network) from the page header. Three skip cases
   surface here, before the overflow even opens:

   - **1st-degree.** The lead is already a connection. Log a skip
     with reason `no_connect_option`, flip `Leads.state` to
     `connected` (it was already connected — workbook just lagged),
     do NOT decrement the budget, continue. The page-view already
     dropped the lead from tomorrow's queue.
   - **A pending "Connect" badge** ("Pending" / "Invitation sent")
     visible on the header without opening the overflow. Log a
     skip with reason `connect_pending_already_invited`, leave
     `Leads.state` and the connect task alone (the prior send is
     still in flight), do NOT decrement the budget, continue.
   - **Out-of-network with no Connect surface available.** Log a
     skip with reason `no_connect_option`, leave `Leads.state =
     queued`, do NOT decrement the budget, continue.

4. **Open the lead-page overflow menu.** Use the pure-JS click
   pattern (see "Click pattern" above) to click the "..." button at
   the top of the lead page. The dropdown should expose Connect,
   View profile, and a handful of other actions.

5. **Inspect the Connect surface.** Two skip cases the overflow
   surfaces:

   - **No "Connect" item in the dropdown.** The profile has
     restrictions. Log a skip with reason `no_connect_option`,
     leave `Leads.state = queued`, do NOT decrement the budget,
     press Escape, continue.
   - **"Connect — Pending"** (em-dash, grayed out). Already
     invited. Log a skip with reason
     `connect_pending_already_invited`, do NOT decrement the
     budget, press Escape, continue.

6. **Click Connect.** Pure-JS click on the "Connect" item in the
   dropdown. The "Send invitation" dialog opens. Two skip cases the
   dialog surfaces:

   - **"Add a personal note" modal that requires an email
     address** ("We need to verify they know you" / "Please enter
     [lead's] email"). This is a Sales Nav privacy setting on the
     lead's side that blocks blank invites — it is NOT a transient
     error and NOT a retry case. Log a skip with reason
     `requires_email` (a NEW terminal skip reason as of V0.3.0),
     flip `Leads.state` to `requires_email_skip` so the lead is
     never re-surfaced for connect attempts, mark the connect
     `Tasks` row `cancelled` with reason `requires_email`,
     dismiss the modal, do NOT decrement the budget, continue.
   - **Dialog never opens** (the click registered but no modal
     appeared after 5 seconds). Treat as a UI anomaly: log to
     `_errors.md` as `send_not_confirmed`, leave the task pending
     (it will be retried at the end of the same run after the
     next page refresh — see "Retry semantics" below), do NOT
     decrement the budget, continue.

7. **Verify "Save as lead" is unchecked.** First send of a session
   may have it pre-checked. Uncheck before sending — Sales Nav's
   workbook is NOT our source of truth (the outreach workbook is)
   and we don't want duplicate state.

8. **Send the request WITHOUT a note.** Message 1 in the approved
   campaign is intentionally BLANK. Do NOT attach a note even if
   the campaign file contains body copy under Message 1 — if it
   does, treat as a config error: log to `_errors.md` asking the
   client to re-approve, skip the lead, do not decrement the
   budget. Click the blue "Send Invitation" button (pure-JS click).

9. **Verify the send.** Re-open the lead-page overflow within 3
   seconds and look for **"Connect — Pending"** (with an em-dash,
   not an ASCII hyphen — the literal UI string is the em-dash;
   regex must match `Connect\s*—\s*Pending`, not
   `Connect\s*-\s*Pending`). If "Connect — Pending" is present, the
   send is confirmed.

   If the overflow does not show "Connect — Pending" after the click
   (regardless of why), log to `_errors.md` with reason
   `send_not_confirmed`, leave the task pending for the in-run
   retry queue (see "Retry semantics"), do NOT decrement the
   budget, continue.

10. **Write to Connections.**
    - `date` = today, ISO
    - `lead_url`
    - `campaign_slug`
    - `note_sent` = `""` (always empty — Message 1 is blank by spec)
    - `source` = `sales_nav`
    - `path` = `lead_profile_overflow` (or `side_preview_overflow`
      if the bot used the fallback path on this lead)

11. **Update Leads.** `state = connected`, `date_connected = today`.

12. **Mark the Tasks row done.** `status = done`, `completed_at = now`.

13. **Pacing — apply both rules.**
    - Wait 60–90 seconds (randomized) before starting the next lead.
    - Increment a per-run successful-send counter. If the counter
      reaches 3, perform a full page refresh: navigate to
      `https://www.linkedin.com/sales/`, wait 10 seconds, reset the
      counter to 0. The next lead's navigation in step 2 will load
      its profile from the freshly-loaded shell.

### Retry semantics for `send_not_confirmed`

When a send hits `send_not_confirmed` (dialog never opened, or the
overflow did not show "Connect — Pending" after the click), the
lead's task is left `pending` and added to an in-memory retry
queue scoped to THIS run only. After the normal queue completes (or
after the budget is otherwise exhausted), attempt each retry-queue
lead exactly ONE more time, each preceded by a full page refresh.
If the retry also fails, log to `_errors.md` with reason
`send_not_confirmed_retry_failed` and abandon the lead for this
run — tomorrow's loop will pick it up fresh.

Do not retry a lead more than once per run. Retries do not
double-count toward the daily / weekly cap; only confirmed sends
count.

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
- `per_path` (map of `lead_profile_overflow` / `side_preview_overflow`
  → sent count — confirms which UI path is actually doing the work)
- `skipped` (list of `{lead_url, reason}`; reasons are
  `connect_pending_already_invited`, `no_connect_option`,
  `requires_email`, `lead_page_did_not_render`)
- `retried` (count of leads that hit `send_not_confirmed` and were
  re-attempted; reported separately from `sent`)
- `retry_failed` (count of leads abandoned after the in-run retry)
- `page_refreshes` (count of Rule B refreshes during this run —
  expect `floor(sent / 3)`)
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
- Do not retry on LinkedIn UI anomalies inside the normal queue.
  Skip the lead, log to `_errors.md`, move on. The in-run retry
  for `send_not_confirmed` (one pass at end-of-queue, after a
  page refresh) is the ONLY retry permitted; a silent in-line
  retry masks UI breakage the weekly report needs to surface.
- Do not touch leads in paused campaigns.
- Do not treat `requires_email` as a retry case. The "We need to
  verify they know you / please enter [lead's] email" modal is a
  Sales Nav privacy setting on the lead's side. Retrying produces
  the same modal every time. `requires_email` is terminal — flip
  `Leads.state` to `requires_email_skip` and cancel the connect
  task. The lead is unreachable through a blank invite forever.
- Do not reuse element refs captured from `read_page` /
  `browser_snapshot` across more than one MCP call. The Sales Nav
  SPA invalidates them mid-batch. Use the pure-JS one-shot click
  pattern (see "Click pattern") as the primary; a fresh find is
  only acceptable as a one-shot fallback when the JS click fails.
- Do not skip the Rule B page refresh after every 3 successful
  sends. Even at 60–90s per-lead jitter, the SPA's JS state
  accumulates and the send pipeline degrades to silent failures.
  Passive cooldowns (sleeping without a reload) do not clear this.
- Do not use the `side_preview_overflow` path as the primary in
  Chrome MCP today. Sales Nav virtualizes the saved-search rows
  and they do not hydrate inside Chrome MCP's `Runtime.evaluate`
  window reliably. Treat `side_preview_overflow` as the documented
  fallback only.
- Do not assume "Connect — Pending" uses an ASCII hyphen. The
  literal UI string is an em-dash (Unicode U+2014). The verify
  regex must match the em-dash explicitly, e.g.
  `Connect\s*—\s*Pending`. A regex matching the ASCII hyphen
  produces false negatives on every confirmed send, which then
  triggers spurious `send_not_confirmed` retries.
