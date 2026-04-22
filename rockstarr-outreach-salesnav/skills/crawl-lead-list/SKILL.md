---
name: crawl-lead-list
description: "This skill should be used when register-campaign asks to \"crawl the saved search\", \"populate leads from Sales Nav\", or \"pull the lead list for a campaign\". It drives Sales Navigator through Chrome MCP: opens the saved-search URL, paginates through results up to target_lead_count, extracts lead fields (name, company, title, LinkedIn URL, location), and writes unique leads into the Leads sheet of outreach-tasks.xlsx. Any lead already present in Leads for another active or paused campaign is logged to _errors.md and causes the register operation to abort."
---

# crawl-lead-list

Eager, one-shot crawl of a Sales Nav saved search into the Leads
sheet. Owned by `register-campaign`; not meant to be called
standalone in V0.1 (a `refresh-lead-list` variant is deferred).

## When to run

- `register-campaign` delegates to this skill with a saved-search URL
  + target count + slug.
- Not recommended standalone. If the user asks for it outside
  `register-campaign`, confirm they're not trying to re-crawl a
  running campaign (`refresh-lead-list` is the deferred path).

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

1. **Verify session.** Navigate to `https://www.linkedin.com/sales/`
   via Chrome MCP. Confirm the signed-in user matches
   `stack.md.linkedin_expected_profile_url`. Mismatch → abort, write
   to `_errors.md`, surface to the caller.
2. **Open the saved search.** Navigate to `saved_search_url`. Wait
   for the results list to load. If Sales Nav prompts for a login or
   returns an error state, abort and write to `_errors.md`.
3. **Paginate.** Walk through result pages until either:
   - `target_lead_count` unique leads have been collected, OR
   - Sales Nav runs out of results, OR
   - The 2500-result ceiling is reached.
   Use the Chrome MCP to read each card's DOM. Do not rely on
   pixel-level scraping.
4. **Extract per lead.** For each result card, capture:
   - `lead_url` — the LinkedIn profile URL (primary key)
   - `name` — full name
   - `company` — current company
   - `title` — current title
   - `location` — free text
5. **Check for duplicates.** For each extracted lead:
   - If `lead_url` is already in Leads under a campaign whose status
     is `active` or `paused`, add it to a `conflicts` list with the
     conflicting campaign slug. Do **not** add it to Leads.
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
7. **Insert.** For each non-conflicting lead, append a Leads row:
   - `lead_url`, `name`, `company`, `title`, `location`,
     `campaign_slug`, `state = queued`, `date_queued = now`.
   Other state-date columns stay empty until the lead moves.
8. **Save.** Persist the workbook via the shared `xlsx` skill.
9. **Summarize.** Return to the caller:
   - total results scanned
   - new leads inserted
   - leads skipped as conflicts (should be 0 on success)
   - any UI anomalies noticed (e.g., Sales Nav showed fewer results
     than expected). Anomalies go to `_errors.md` as warnings, not
     aborts.

## Rate limiting

- Wait 1.5–3 seconds between page navigations. Randomize within that
  window. LinkedIn rate-limits aggressive pagination.
- If LinkedIn serves a CAPTCHA or a "you've been viewing a lot of
  profiles" modal, abort, write to `_errors.md`, and surface to the
  caller. Do **not** attempt to click through.

## What NOT to do

- Do not send connect requests from this skill. Ever. Connects are
  `daily-connect`'s exclusive responsibility.
- Do not attempt to re-crawl a running campaign. Use
  `refresh-lead-list` (deferred past V0.1) when it ships.
- Do not write to `Tasks`. `register-campaign` seeds connect tasks
  after this skill returns successfully.
- Do not silently skip conflicts. The abort-on-conflict contract
  exists so the client notices and fixes the saved search.
- Do not try to solve CAPTCHAs, 2FA prompts, or re-logins. Abort and
  let the client handle it.
