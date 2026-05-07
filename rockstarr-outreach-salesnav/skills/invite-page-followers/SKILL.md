---
name: invite-page-followers
description: "This skill should be used when the scheduled monthly run fires (default cron 0 14 8-14 * 2 — 2pm second Tuesday of the month, configurable per workspace), or when the user says \"invite page followers\", \"run the monthly page-follow invites\", \"invite my connections to follow the company page\", or \"do the page-invite run for this month\". Drives the LinkedIn admin dashboard via Chrome MCP for the configured company page, opens the Invite Connections modal at /company/<id>/admin/dashboard/?invite=true, verifies the signed-in profile photo matches the configured admin display name AND that the admin redirect succeeded, reads the credit balance, lazy-loads the connections list, selects up to min(credits_available, page_invite_credit_target) connections, clicks Invite, captures the confirmation, and logs the run to /05_published/outreach/page-invites/YYYY-MM.md. Cycle-deduplicated — refuses to fire twice in the same credit cycle without an explicit force_rerun override. Page-invite credits are page-level and refill monthly; they do NOT share the 20/day + 100/week connect-request cap that governs daily-connect."
---

# invite-page-followers

Monthly action that uses up the configured company page's invite-credit
pool by inviting the admin's first-degree LinkedIn connections to
follow the page. Parallel to `daily-connect` in mechanism (Sales Nav
UI driven through Chrome MCP) but otherwise unrelated — this is a
page-growth task, not a lead-pipeline campaign. It does not touch
the Leads / Tasks / Connections / Replies workbook sheets.

## When to run

- A scheduled monthly cron fires the skill at the configured time
  (default `0 14 8-14 * 2` — 2pm on the second Tuesday of the month).
  Different workspaces can pick different days inside the 8–14 day
  window via `stack.md.page_invite_schedule_cron`.
- On-demand when the user asks "invite page followers," "run the
  monthly page-follow invites," "invite my connections to follow the
  company page," or names the configured page.
- Catch-up after a missed scheduled run. Cycle-dedup (Step 5 below)
  governs whether the on-demand run actually proceeds.

## Preconditions

- `confirm-session` passed in this run (within the last 30 min).
- `/rockstarr-ai/00_intake/stack.md` has the page-invite block:

  ```yaml
  page_invite_enabled: true
  page_invite_target_url: "https://www.linkedin.com/company/<slug>/"
  page_invite_company_id: "<numeric id>"          # used in admin URLs
  page_invite_admin_display_name: "<full name>"   # for alt-text check
  page_invite_schedule_cron: "0 14 8-14 * 2"
  page_invite_credit_target: 250                  # optional, default 250
  ```

  If `page_invite_enabled` is `false` or absent, refuse and exit
  cleanly with `{status: "skipped", reason: "page_invite not enabled in stack.md"}`.

  If any of `page_invite_target_url`, `page_invite_company_id`, or
  `page_invite_admin_display_name` is missing, refuse and tell the
  user which key is missing and to capture it via
  `rockstarr-infra:capture-stack` (or edit stack.md directly).

## Inputs

- The page-invite block from `stack.md`.
- The run log at `/05_published/outreach/page-invites/<YYYY-MM>.md`
  for the current month (read for the cycle-dedup check; created if
  missing on first run of the month).

## Behavior

Execute in order. Any failed step aborts cleanly with a written log
entry — never proceed past a broken precondition.

### Step 1 — Identity gate (CRITICAL)

This is the highest-stakes check in the skill. Wrong account =
silently burning someone else's monthly invite quota and spamming
their network. Two checks, both must pass:

**1a. Signed-in profile-photo alt-text.** Navigate Chrome MCP to any
LinkedIn page (e.g. the configured `page_invite_target_url`'s public
URL) and execute:

```js
document.querySelector('.global-nav__me-photo, img[alt*="Photo of"]')?.alt
```

The returned alt text MUST contain the configured
`page_invite_admin_display_name` (case-insensitive substring match).
If the selector returns nothing or the alt doesn't contain the name,
abort with:

```
Identity check failed. Expected alt text containing
"<page_invite_admin_display_name>"; got "<actual or null>". Will not
proceed — running on the wrong logged-in account would burn that
person's monthly invite quota.
```

Log to `_errors.md`. Do NOT navigate to the admin dashboard. Exit.

**1b. Admin-redirect.** Navigate to
`https://www.linkedin.com/company/<page_invite_company_id>/admin/dashboard/?invite=true`.
If LinkedIn redirects to a non-admin page (the public company page,
or LinkedIn's home), or returns a 404, the configured admin user
does not have admin rights on this page. Abort with:

```
Admin-redirect check failed. The configured admin user
"<page_invite_admin_display_name>" does not have admin rights on
the page at company id "<page_invite_company_id>". Verify admin
access on linkedin.com first, then re-run.
```

Log to `_errors.md`. Exit.

Both checks must pass before any further step runs. The redirect to
`/company/<id>/admin/dashboard/?invite=true` is the legitimate
landing state; verify the URL before proceeding.

### Step 2 — Open the Invite Connections modal

The `?invite=true` URL parameter should auto-open the modal. If it
does not appear within 5 seconds, fall back to clicking
"Grow followers" or "Invite to follow" in the admin sidebar. Use
Chrome MCP `find` with the accessible-name query rather than pixel
coordinates.

If the modal still does not appear, abort with a clear message
naming the page and the URL. Log to `_errors.md`.

### Step 3 — Read the credit balance

Parse the modal header for the credit display (typical shape:
`<N>/<max> credits available · Credit refill: <date>`). Capture:

- `credits_available` — current cycle balance
- `credits_max` — should be 250 for page admins; if it shows 50, the
  account is acting as a regular user, not a page admin (a clear
  signal of a wrong-account state that the identity gate missed —
  abort with a note in `_errors.md`)
- `next_refill_date` — captured for the run log

If `credits_available == 0`:

- Write a "credits exhausted this cycle, skipping" entry to the run
  log at `/05_published/outreach/page-invites/<YYYY-MM>.md`.
- Exit cleanly with `{status: "skipped", reason: "no_credits_this_cycle", next_refill: <date>}`.

Never click into an empty pool — the modal will let you select
checkboxes but the Invite button stays disabled, and the
appearance-of-progress is misleading in logs.

### Step 4 — Compute the target select count

```
target_select = min(credits_available, page_invite_credit_target)
```

`page_invite_credit_target` defaults to 250 (the LinkedIn admin max)
and exists for workspaces that want to dial down deliberately.

### Step 5 — Cycle-dedup

Read `/05_published/outreach/page-invites/<YYYY-MM>.md`. If a prior
successful run exists THIS MONTH:

- If that prior run sent the full credit pool (left
  `credits_remaining: 0`), refuse: this cycle is done. Exit with
  `{status: "skipped", reason: "already_ran_this_cycle"}`. The
  scheduled cron only fires once a month so this only matters for
  manual invocations.
- If that prior run had `credits_remaining > 0` (the connection
  pool was smaller than the credit pool that month — uncommon),
  this is a legitimate catch-up case. Allow the re-run and log it
  as a follow-on (`run_kind: catch-up` in the run log).
- If the caller passes `force_rerun: true`, override the dedup and
  proceed regardless. Log `run_kind: forced` in the run log.

### Step 6 — Lazy-load the connection list

Scroll the `.invitee-picker__results-container` to the bottom and
click any "Show more results" button until the loaded checkbox count
is at least `target_select + 10`. The `+10` is headroom — it
prevents a race where the credit decrement lands before the click
registers, leaving `target_select - 1` selected.

If after repeated scrolls the loaded count plateaus below
`target_select`, the connection pool is smaller than the credit
pool. Continue with whatever's loaded; the run log will record
`credits_remaining > 0` and the dedup logic in Step 5 will allow a
catch-up run later if leftover credits are still useful.

### Step 7 — Select connections

Click each connection's `<label>` element (NOT the raw `<input>`
checkbox — the BBN process docs name this specifically; LinkedIn's
click handler binds to the label). Pause briefly every ~20 clicks
for the selection counter to keep up.

Stop when:

- `target_select` clicks have been registered AND the dialog footer
  reads `<target_select> selected` AND the Invite button label
  shows `Invite <target_select>`. Verify all three.
- OR the connection pool is exhausted before reaching `target_select`
  (handle per Step 6's plateau case).

If the footer count and the click count diverge by more than 2 (a
real selection sync bug), pause 3 seconds, re-read the footer, and
adjust click cadence. Do NOT power through — over-clicking can
double-select and cause the Invite button to show a count higher
than `credits_available`, which LinkedIn rejects.

### Step 8 — Send

Click the blue "Invite <N>" button at the bottom-right of the modal.
Wait for the confirmation screen:

- Confirmation: `<N> connections are invited to follow your Page`
- Toast: `<N> people invited to follow Page`

If neither appears within 10 seconds, log to `_errors.md` with the
modal state captured (use Chrome MCP `read_page` for context) and
abort the rest of the run. DO NOT retry the click — the first click
may have succeeded silently and a retry would attempt to re-send to
the same connections.

### Step 9 — Dismiss the upsell

LinkedIn shows a "Start a post" prompt after a successful send.
Click "No thanks." If the prompt doesn't appear, no action needed.

### Step 10 — Log the run

Append a row to `/05_published/outreach/page-invites/<YYYY-MM>.md`
(create the file with a header if it doesn't exist):

```markdown
# Page-follow invite runs — <YYYY-MM>

| Timestamp | Page | Sent | Credits remaining | Next refill | Run kind | Notes |
|---|---|---:|---:|---|---|---|
| <ISO> | <slug> | <N> | <M> | <date> | scheduled | — |
| <ISO> | <slug> | <N> | <M> | <date> | catch-up | <reason> |
| <ISO> | <slug> | <N> | <M> | <date> | forced | <reason> |
```

`run_kind` values: `scheduled` (the cron fired), `manual` (operator
ran it on-demand within the cycle), `catch-up` (Step 5's leftover-
credits case), `forced` (Step 5's force_rerun override).

Also append a one-liner to the day's `/05_published/outreach/<today>.md`
(the daily activity log) for the cross-cutting view:

```
invite-page-followers — sent <N>/<credits_available> to <slug>;
<credits_remaining> credits remaining; next refill <date>.
```

### Step 11 — Return

Return a structured summary to the caller:

- `sent` — count of invites sent
- `credits_remaining` — should be 0 if we hit `target_select == credits_available`, else the leftover
- `credits_max` — 250 for admin accounts (sanity-check value)
- `next_refill_date` — captured from the dialog
- `run_kind` — `scheduled` / `manual` / `catch-up` / `forced`
- `run_summary_path` — the path to the monthly run log file

## Output

Returns the structured summary in Step 11 plus a short message in
chat:

> Invited <N> connections to follow <page slug>. <credits_remaining>
> credits remaining for this cycle. Next refill: <date>.

If skipped:

> Page-follow invite skipped: <reason>. Next eligible run: <date>.

## What NOT to do

- Do NOT proceed past Step 1 if either identity check fails. Wrong-
  account sending burns someone else's quota and spams their
  network. There is no "soft-fail" path here.
- Do NOT use pixel coordinates for any click. Chrome MCP `find` with
  accessible-name queries is the only stable lookup. Viewport
  changes will move absolute coordinates but accessible names stay.
- Do NOT click `<input>` checkboxes directly. Click the `<label>`.
  LinkedIn's handler binds to the label and bare-input clicks
  occasionally fail silently.
- Do NOT exceed `credits_available`. The Invite button will reject
  but the lead-up clicks are wasted effort and the over-selection
  state is hard to back out of cleanly.
- Do NOT retry on a confirmation timeout in Step 8. The send may
  have succeeded silently; a retry would double-invite. Log and
  abort instead.
- Do NOT touch the campaign workbook (`outreach-tasks.xlsx`) from
  this skill. Page-follow invites are a separate audience-growth
  surface, not part of the campaign lead pipeline.
- Do NOT run twice in the same credit cycle without explicit
  authorization. Step 5's dedup is the gate; honor it.
- Do NOT proceed if the credits-max value reads 50 instead of 250.
  That's a personal-account state, not an admin state, and a sign
  the identity gate missed something. Abort and ask the operator
  to verify admin rights on the page.
- Do NOT include the booking link, the saved-search URL, or any
  campaign-related material in the run log. The run log is purely
  about the page-invite action.

## Schedule wiring

The scheduled run is wired during onboarding via the workspace's
`schedule` skill, pointing at this skill with the cron expression
from `stack.md.page_invite_schedule_cron`. The cron is evaluated in
the workspace's local timezone per the schedule skill's behavior.

Reference cron expressions:

- `0 14 8-14 * 2` — 2pm EST on the second Tuesday of each month
  (matches the Rockstarr & Moon page run cadence in the V0.1
  reference).
- `0 14 8-14 * 3` — 2pm EST on the second Wednesday of each month
  (matches the BBN page run cadence in the V0.1 reference).

The day-range `8-14` plus a weekday filter (`* 2` for Tuesday, `* 3`
for Wednesday) is the standard "Nth weekday of the month"
construction in standard cron — there's no native "second Tuesday"
operator, so the day-range constrains it.

## Related

- `confirm-session` — the upstream gate that runs before this skill
  (and every other Sales Nav skill in this plugin).
- `daily-connect` — parallel skill, runs against a different audience
  (search-derived leads) with a different cap (20/day + 100/week
  account-level) and a different cadence (every weekday). The two
  skills do not share a credit pool or interact in any way.
- `rockstarr-infra:capture-stack` — the install-time skill that
  collects the page-invite config keys. Updates to capture-stack to
  prompt for these keys are tracked separately; until that ships,
  operators add the keys to `stack.md` by hand.
