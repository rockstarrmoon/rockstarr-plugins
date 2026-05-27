---
name: crawl-lead-list
description: "This skill should be used when register-campaign asks to \"crawl the saved search\", \"populate leads from Sales Nav\", \"pull the lead list for a campaign\", or when a daily-loop self-heal step says a campaign has fewer queued leads than target. Idempotent against the same campaign: re-runs URL-dedup against existing leads and pick up where partial crawls left off, until target_lead_count is reached or Sales Nav runs out. Drives Sales Nav through Chrome MCP with per-row scrollIntoView + hydration poll before extraction. Writes only fully-hydrated leads. Any lead already in Leads for a DIFFERENT active or paused campaign is logged and aborts the register."
---

# crawl-lead-list

Idempotent crawl of a Sales Nav saved search into the Leads sheet.
Re-running against the same campaign skips leads already present
(URL-dedup against `Leads.lead_url` filtered to this campaign) and
continues from where the prior run left off, until
`target_lead_count` is reached.

## When to run

- `register-campaign` delegates to this skill with a saved-search URL
  + target count + slug. First run for a campaign typically pulls
  most of `target_lead_count` in one pass.
- **Daily-loop self-heal.** When a daily loop detects that an active
  campaign has fewer queued leads than `target_lead_count` (e.g., a
  prior crawl returned partial results due to Sales Nav hydration
  flakiness), the loop calls this skill again to top up. Idempotency
  makes this safe — the skill picks up where the last partial crawl
  stopped without redoing work.
- **Manual top-up.** The user can also invoke this skill directly to
  resume a stalled crawl. The skill detects the existing-lead count
  for the campaign in step 0 and continues pagination from there.

## Preconditions

- `confirm-session` has passed within the current run, OR the user is
  starting a register flow from the Cowork UI (the first crawl can
  happen before the first daily loop; we still verify the session in
  step 1 below).
- Sales Nav is accessible to the logged-in LinkedIn account.
- `outreach-tasks.xlsx` exists.

## Inputs

- `saved_search_url` — the full Sales Nav saved-search URL.
- `campaign_slug` — the slug being registered.
- `target_lead_count` — how many leads to pull (Sales Nav's hard
  ceiling on saved searches is ~2500; treat that as the max).

## Behavior

0. **Resumption check.** Read Leads to count rows where
   `campaign_slug == campaign_slug` (existing-lead count). Compute
   `remaining = target_lead_count - existing_count`.
   - If `remaining <= 0`, return early:
     `{ "status": "noop", "reason": "target_already_met",
        "existing_count": [N], "target": [T] }`.
   - Otherwise, capture an `existing_lead_urls` set from the same
     query. Pagination will skip leads whose URLs are already in
     this set without treating them as conflicts.
1. **Verify session.** Navigate to `https://www.linkedin.com/sales/`
   via Chrome MCP. Confirm the signed-in user matches
   `stack.md.linkedin_expected_profile_url`. Mismatch → abort, write
   to `_errors.md`, surface to the caller.
2. **Open the saved search.** Navigate to `saved_search_url`. Wait
   for the results list shell to render (results count badge present,
   page-1 results container present). If Sales Nav prompts for a
   login or returns an error state, abort and write to `_errors.md`.
3. **Paginate with per-row hydration.** For each result page:
   a. **Read the rendered row count.** Sales Nav's saved-search
      results page is virtualized — typically 25 placeholder rows
      render immediately, but each row's `[article]` children
      (name link, company, title, location) lazy-load on scroll.
      Do NOT rely on whole-page hydration: a single `wait` after
      page-load returns long before all 25 rows finish loading.
   b. **For each row index 0..24 on this page**, drive Chrome MCP
      to scroll the row into view and wait for it to hydrate
      individually:
      - Use a pure-JS one-shot inside `javascript_tool`:
        `document.querySelectorAll('[data-x-search-result]')[i].scrollIntoView({block: 'center'})`,
        then poll for the row's `[article]` having both a non-empty
        accessible-name on the lead-link anchor AND a non-empty
        company/title text node. Polling cadence: 250ms, ceiling
        4000ms per row.
      - If the row hydrates fully within the ceiling, extract its
        fields (step 4) and continue to the next row.
      - If the row does NOT hydrate within the ceiling, log to
        `_errors.md` with `row_did_not_hydrate` + page number +
        row index + the page URL, do NOT extract partial fields,
        continue to the next row.
   c. **Commit-only-fully-hydrated.** A row's fields go into the
      `extracted_this_page` set only if `name`, `lead_url`, AND
      `company` are all non-empty. A row with only `name` and
      `lead_url` (no company/title) is incomplete — the exclusion
      filter in step 5 cannot match without company, and partial
      rows poison downstream targeting. Log incomplete rows as
      `row_partial_hydration` warnings.
   d. **Pagination loop.** After all 25 rows on this page are
      processed, advance to the next page (click "Next") and repeat
      from 3a until either:
      - The total of `existing_count + len(extracted)` reaches
        `target_lead_count`, OR
      - Sales Nav has no more pages, OR
      - The 2500-result ceiling is reached.
   e. **Partial-run is acceptable.** It is normal for Chrome MCP +
      Sales Nav timing to leave some pages with `0 hydrated` rows
      under load. Log the partial-page count and continue. The
      daily-loop self-heal step (or a manual re-run) will resume.
4. **Extract per lead.** From each fully-hydrated row, capture:
   - `lead_url` — the LinkedIn profile URL (primary key)
   - `name` — full name
   - `company` — current company
   - `title` — current title
   - `location` — free text
5. **Check for duplicates.** For each extracted lead:
   - If `lead_url` is in `existing_lead_urls` (the set from step 0,
     i.e. already crawled for THIS campaign in a prior run), skip
     it silently. This is the resumption path — not a conflict.
   - If `lead_url` is in Leads under a DIFFERENT campaign whose
     status is `active` or `paused`, add it to a `conflicts` list
     with the conflicting campaign slug. Do **not** add it to Leads.
   - If `lead_url` is in Leads under a `stopped` campaign, it's
     available to re-use; continue to step 6.
   - If `lead_url` is not in Leads at all, queue it for insert.
6. **Abort-on-conflict.** If `conflicts` is non-empty:
   - Write a timestamped block to
     `/02_inputs/outreach/_errors.md` with the new campaign slug, the
     list of `(lead_url, conflicting_slug)` pairs, and a short
     remediation note (stop the other campaign, or remove leads from
     the saved search).
   - Return `{ "status": "aborted", "conflicts": [...] }` to the
     caller. Do **not** insert any leads.
7. **Insert.** For each non-conflicting, non-resumption lead, append
   a Leads row:
   - `lead_url`, `name`, `company`, `title`, `location`,
     `campaign_slug`, `state = queued`, `date_queued = now`.
   Other state-date columns stay empty until the lead moves.
8. **Save.** Persist the workbook via the shared `xlsx` skill.
9. **First-crawl ICP sanity check.** This step fires ONLY when
   `existing_count_before == 0` AND `inserted_this_run > 0` — i.e.,
   this is the first batch of leads ever crawled for this campaign,
   and there's something to look at. Skip on resumption runs (the
   user already saw the sample on the first pass) and on noop /
   zero-insert runs.

   - Read the per-campaign ICP at
     `02_inputs/outreach/icps/[slug].md` for the audience summary
     and disqualifier patterns.
   - Pick the first 5 (or up to 10 if `inserted_this_run >= 10`)
     leads from the rows just inserted, in insertion order.
   - Surface to the user with one `AskUserQuestion` turn:
     "Quick sanity check — do these first [N] leads look like a
     reasonable fit for the campaign's ICP? Listed by name +
     headline + company so you can eyeball them against the saved
     search filters."
     Options:
       - "Looks right — proceed" (no further action)
       - "A few off but acceptable — proceed" (no further action;
         log warning to `_errors.md` for later review)
       - "Mostly off — pause and let me fix the saved search"
         (log warning; return `status: "partial"` with a note in
         `ui_anomalies` that the user wants to revise the search;
         the caller — typically `register-campaign` — should
         surface this to the user with a clear "review the saved
         search before re-running" pointer; do NOT roll back leads
         that were inserted, since `crawl-lead-list` is idempotent
         and any future re-invoke will resume cleanly)
   - The check is a sanity net, not a hard validation — it surfaces
     obvious saved-search miscalibration (board members instead of
     operators, wrong function filter, off-ICP companies) at a
     moment when fixing it is cheap. Bot does not enforce the
     answer beyond logging; the user is the final judge.
10. **Summarize.** Return to the caller:
   - `status` — `"complete"` (target met), `"partial"` (work done
     but target not met yet — caller should re-invoke later),
     `"aborted"` (conflict), or `"noop"` (target already met).
   - `existing_count_before` — Leads count for this campaign before
     this run.
   - `inserted_this_run` — new leads added.
   - `total_after` — Leads count for this campaign after this run.
   - `target` — the input `target_lead_count`.
   - `pages_walked` — number of result pages traversed.
   - `rows_hydrated_fully` — rows fully hydrated this run.
   - `rows_did_not_hydrate` — rows the bot gave up on this run.
   - `rows_partial_hydration` — rows with name+URL but no company.
   - `conflicts` — list of cross-campaign duplicate `(lead_url,
     conflicting_slug)` pairs (should be empty on success).
   - `ui_anomalies` — free-form warnings (Sales Nav showed fewer
     results than expected, etc.).

## Caller-side smart-defer

`crawl-lead-list` is safe to call any time and short-circuits when
the target is already met (step 0 → `noop`). Callers that have
cheaper information should still gate their invocations:

- A daily loop with `today_budget = 20` and `Leads.count(campaign,
  state=queued) >= 20` does not need to crawl this run — today's
  20 connect attempts will draw from the existing queue. The
  loop's `crawl-on-shortfall` step should compare
  `Leads.count(campaign, state=queued) + today_budget` against
  `target_lead_count` and only call this skill if there's a
  meaningful gap (e.g., shortfall > today's budget × 1.5).
- `register-campaign` should always call this skill on first
  registration (existing_count = 0 → resumption set is empty →
  normal first-pass crawl).

The skill itself does NOT gate on send-budget math — that is the
caller's responsibility. The skill's contract is "given a saved
search and a target, make idempotent progress toward the target."

## Rate limiting

- Wait 1.5–3 seconds between page navigations. Randomize within that
  window. LinkedIn rate-limits aggressive pagination.
- If LinkedIn serves a CAPTCHA or a "you've been viewing a lot of
  profiles" modal, abort, write to `_errors.md`, and surface to the
  caller. Do **not** attempt to click through.

## What NOT to do

- Do not send connect requests from this skill. Ever. Connects are
  `daily-connect`'s exclusive responsibility.
- Do not write to `Tasks`. `register-campaign` seeds connect tasks
  after this skill returns successfully.
- Do not silently skip cross-campaign conflicts. The
  abort-on-conflict contract exists so the client notices and fixes
  the saved search.
- Do not commit partially-hydrated rows. A row with `name` and
  `lead_url` but blank `company` or `title` is incomplete — the
  exclusion filter cannot match without `company`, and downstream
  ICP sanity checks rely on `title`. Log to `_errors.md` as
  `row_partial_hydration` and skip the row. Partial commits poison
  the Leads sheet in ways that are expensive to detect later.
- Do not wait on whole-page hydration. The Sales Nav saved-search
  results page hydrates rows lazily on scroll; a single `wait`
  after page-load returns before most rows are populated. Per-row
  `scrollIntoView` + per-row hydration poll (step 3b) is the only
  way to reliably read the rows inside Chrome MCP's
  `Runtime.evaluate` window.
- Do not treat a partial run as a failure. Returning fewer leads
  than `target_lead_count` is a normal outcome under Chrome MCP +
  Sales Nav timing — the skill's contract is idempotent
  progress-toward-target, not all-or-nothing. The caller (loop or
  `register-campaign`) will re-invoke until the target is met.
- Do not try to solve CAPTCHAs, 2FA prompts, or re-logins. Abort
  and let the client handle it.
