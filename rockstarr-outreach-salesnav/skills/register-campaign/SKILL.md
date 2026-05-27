---
name: register-campaign
description: "This skill should be used when the user says \"register this campaign\", \"launch the campaign\", \"go live on this campaign\", or names an approved campaign spec in 04_approved/outreach/. Promotes the approved campaign live: reads campaign_type front-matter, validates the spec shape, enforces ALL stack prerequisites upfront with a single consolidated remediation message, strips the volatile sessionId from the saved-search URL, scaffolds _excluded-companies.md if missing, adds a Campaigns row, invokes crawl-lead-list, surfaces the crawl outcome with the right next-step pointer, and seeds the initial connect tasks. Fails loudly if any lead already belongs to another active campaign."
---

# register-campaign

Promote an approved campaign spec into live state. This is the
irreversible step that moves a campaign from paperwork to a live
pipeline of tasks the bot will execute.

## When to run

- User asks to register / launch / go live on a specific campaign slug.
- A campaign spec has just been approved via `rockstarr-infra:approve`
  and the user wants it live.

## Preconditions

This skill is the gate between paperwork and a live pipeline. It
enforces every key the daily loop will need **upfront** instead of
discovering missing config mid-flight (which is what wasted Jason's
first-campaign morning in the 0.2.x era). Run the full check first;
present a single consolidated remediation message if anything is
missing; do not partially register.

### Required (block registration if missing)

- `/rockstarr-ai/04_approved/outreach/campaign-[slug].md` exists.
- `/rockstarr-ai/00_intake/stack.md` exists and contains:
  - `outreach_tool: salesnav`
  - `linkedin_expected_profile_url` matching `https://www.linkedin.com/in/...`
  - `booking_mode: automated | manual`
  - `availability_source: booking_link | gcal`
  - If `availability_source: booking_link`: `booking_link_url` (URL).
  - If `availability_source: gcal`: `gcal_id` (string).
- `outreach-tasks.xlsx` exists at
  `/rockstarr-ai/02_inputs/outreach/outreach-tasks.xlsx`. Create it
  (with empty sheets following the README schema) if missing.
- No existing `Campaigns` row with the same `campaign_slug` in
  `status = active` or `paused`. (Re-registering a `stopped` campaign
  with the same slug is allowed; leads previously in a `stopped`
  campaign may be re-used.)

### Recommended (warn but don't block)

These don't block registration but the daily loop will be missing
features without them:

- **Reply notification destination.** Check stack.md for
  `outreach_notify_to` (email address). If absent, also check
  `/rockstarr-ai/client.toml` for `[notifications].default_recipient`.
  If neither resolves, warn: "Reply notifications via
  `notify-reply-ready` will be skipped — replies will still stage
  drafts and show up in the next morning's `approvals-digest`, but
  you'll miss the urgent same-day email. Set `outreach_notify_to` in
  stack.md to fix."
- **Timezone for cadence math.** Check stack.md for `timezone` (e.g.
  `"America/Chicago"`). If absent, warn: "Cadence math will default
  to UTC — message-step due-dates may fire at unintended hours in
  your local timezone. Set `timezone` in stack.md to fix."
- **Booking-link field schema** (only when
  `availability_source: booking_link`). Check stack.md for both
  `booking_link_required_fields` (list) AND
  `booking_link_optional_fields` (list — split off in
  `rockstarr-infra:capture-stack@0.9.12+`). If
  `booking_link_required_fields` is unset, warn: "`book-meeting`
  will abort during form-fill on any required field whose source
  value is missing. Capture the field schema in stack.md."
  `booking_link_optional_fields` absent is fine — defaults to empty.

### Remediation

If anything in **Required** is missing, present a single message:

```
Cannot register <slug> yet. Please set up the following first:

Required:
  [✗] stack.md: linkedin_expected_profile_url is missing
  [✗] stack.md: booking_mode is missing

Recommended (not blocking but worth fixing):
  [!] stack.md: outreach_notify_to is missing — urgent reply emails will be skipped
  [!] stack.md: timezone is missing — cadence math will default to UTC

Run `rockstarr-infra:capture-stack` (or edit stack.md directly) to
add the missing keys, then re-invoke `register-campaign`.
```

Do not partially register.

## Inputs

Read:

1. `04_approved/outreach/campaign-[slug].md` — full front-matter and
   body. Extract `campaign_type` (default `full-sequence` if absent
   for back-compat with pre-0.1.7 specs), `saved_search_url`,
   `target_lead_count`, sequence copy (full-sequence only), cadence,
   booking mode (full-sequence only).
2. `00_intake/stack.md` — outreach keys + booking config.
3. `02_inputs/outreach/outreach-tasks.xlsx` — current state of Leads,
   Tasks, Campaigns.

### Campaign-type validation

Before proceeding past Step 1, validate the spec against its
declared `campaign_type`:

- **`full-sequence`** — front-matter must include a non-empty
  `anchor_phrase` and a `cadence` of `"day-of-accept, accept+3,
  accept+7"` (or equivalent). Body must contain a `## Sequence`
  section with Messages 1–4 fully drafted. If any of these are
  missing, abort with a clear message pointing the user at
  `draft-icp-campaign` to redraft.

- **`connect-only`** — front-matter `anchor_phrase` must be empty,
  `cadence` must be `"connect requests only — no post-accept
  sequence"`, and `booking_mode` / `availability_source` must be
  `"n/a"`. Body must NOT contain a `## Sequence` section, a
  `## Pain focus + anchor phrase` section, or a `## Booking flow`
  section. If any of these constraints are violated, abort with a
  message naming the offending section and pointing the user at
  `draft-icp-campaign` to redraft. The validation is loud on
  purpose — the campaign-type field is load-bearing for downstream
  skills (`detect-accepts`, `generate-message-tasks`,
  `outreach-weekly-report`), and a mismatched spec creates silent
  bugs at run time.

## Behavior

Execute in this order. If any step fails, roll back the preceding
writes so the workbook does not end up in a half-registered state.

1. **Normalize the saved-search URL.** Read `saved_search_url` from
   the approved campaign's front-matter. Strip the volatile
   `sessionId` query parameter before storing — Sales Nav regenerates
   `sessionId` on every page load, so anything captured at draft
   time is stale by the time the daily loop runs. What identifies
   the saved search is `savedSearchId=[n]`; everything else is
   noise that hurts URL portability and shows up as confusing diff
   noise when re-comparing the campaign file against the workbook.

   Implementation: parse the URL's query string, drop the
   `sessionId` key if present, re-emit the URL with the remaining
   query parameters preserved in original order. If the user
   supplied a URL without `sessionId` (already-clean), this is a
   no-op. Log nothing on the no-op path; log a one-line note on the
   stripping path ("normalized saved-search URL: stripped sessionId").

2. **Ensure the workspace-level exclusion list exists.** If
   `/rockstarr-ai/02_inputs/outreach/_excluded-companies.md` does
   NOT exist, create it with this header:

   ~~~markdown
   # Excluded companies — workspace-wide

   This file lists company names that ALL outreach campaigns should
   skip during `crawl-lead-list`. It is workspace-scoped — entries
   apply to every active and future campaign, not just the one
   that prompted them.

   This is the right home for **named lists** (your past clients,
   your own entities, partners you don't want to outreach). It is
   NOT the right home for **disqualifier patterns** (titles to
   avoid, company-size bands to skip, geography exclusions). Those
   belong in the per-campaign ICP file at
   `02_inputs/outreach/icps/[slug].md` under the Disqualifiers
   section, and feed back into the Sales Nav saved search itself.

   One company per line. Case-insensitive substring match against
   each lead's company name. Lines starting with `#` are comments.

   ~~~
   ```
   <!-- Examples (delete this block once you've added real entries):
   Acme Corp
   Acme Inc.
   Acme Holdings
   -->
   ```
   ~~~

   Then surface to the user: "Created `_excluded-companies.md`. If
   you have past clients, your own entities, or partner companies
   to skip, add them now — one per line — before continuing.
   `crawl-lead-list` will honor them on this run and every future
   run." Wait for the user to confirm they've seeded it (or
   explicitly skip).

   If the file already exists, no action; do not surface to the user.

3. **Open the workbook.** Use the shared `xlsx` skill. Read the
   Campaigns, Leads, Tasks sheets into memory.
4. **Write the Campaigns row.** Append one row:
   - `campaign_slug`: the slug
   - `campaign_type`: `full-sequence` or `connect-only` (from spec
     front-matter; default to `full-sequence` for back-compat if
     absent on a pre-0.1.7 spec)
   - `icp_ref`: path to the ICP file
   - `lead_list_saved_search_url`: the **normalized** URL from step 1
     (not the raw URL from the spec)
   - `target_lead_count`: from front-matter
   - `status`: `active`
   - `date_started`: today's ISO date
   - `date_stopped`: empty
   - `notes`: the campaign title + anything from the spec header
5. **Invoke `crawl-lead-list`** with the normalized saved-search URL,
   target count, and slug. Wait for completion.
6. **Check for conflicts.** If `crawl-lead-list` reports any leads
   that already belong to another active or paused campaign
   (status `aborted` with non-empty `conflicts`):
   - Do **not** proceed.
   - Log the conflict list to `/02_inputs/outreach/_errors.md` with a
     timestamp, the slug that tried to register, and the conflicting
     lead URLs + their current campaign.
   - Roll back the Campaigns row.
   - Tell the user which leads conflict and ask them to either remove
     the leads from the saved search, or stop the other campaign,
     before retrying.
7. **Seed connect tasks.** For every new lead in Leads with
   `campaign_slug` matching this one and `state = queued`, append a
   `Tasks` row:
   - `task_id`: generated
   - `lead_url`: lead's LinkedIn URL
   - `campaign_slug`: this slug
   - `type`: `connect`
   - `due_date`: empty (daily-connect schedules based on budget)
   - `status`: `pending`
   - `created_at`: now
   - `completed_at`: empty
8. **Save the workbook.** Let the `xlsx` skill handle the write +
   validation.
9. **Log.** Append a line to `/05_published/outreach/[today].md`:
   `registered campaign [slug] — [lead_count] leads seeded, connects queued`.

## Output

Return a short summary to the user, structured as three blocks:

**1. What got registered.**
- Campaign slug + title
- Campaign type (full-sequence / connect-only)
- Saved-search URL after normalization (note if `sessionId` was
  stripped)
- Number of leads seeded with the `crawl-lead-list` status verbatim:
  - `status: complete` (target met on the first pass) — say:
    "[N] leads seeded; target met. Connects begin firing at the
    next scheduled morning run."
  - `status: partial` (some leads seeded, target not yet met — this
    is normal under Chrome MCP × Sales Nav timing for larger
    targets) — say: "[N] of [T] leads seeded on this pass.
    `crawl-lead-list` is idempotent and will resume on subsequent
    runs. **Next step:** make sure your morning loop's
    `crawl-on-shortfall` pre-step is wired up (see README's
    'Scheduling the loop' section) — without it, no further leads
    will be crawled and `daily-connect` will stop sending once it
    exhausts the [N] seeded. Alternative: invoke `crawl-lead-list
    [slug]` manually now to top up before the first send."
  - `status: noop` (target was already met from a prior register)
    — say: "Target already met ([N] leads). No new crawl this run."
- Number of conflicts rejected (should be 0 if we reached this step)

**2. Warnings surfaced during preconditions** (recommended-but-
missing keys). One line per warning, prefixed `[!]`. Empty section
if everything was set.

**3. What happens next.**
- "`daily-connect` will start sending at `stack.md.outreach_daily_run_time`."
- **Scheduling heads-up — say this on every first-time registration
  for a workspace**: scheduled tasks run inside Claude Desktop and
  only fire when the app is open and the laptop is awake. If the
  laptop sleeps at the scheduled time, the task fires on next
  launch — i.e., when you wake the laptop. For most workdays this
  is fine; for strict-window cadence or multi-day absences, run
  the laptop in clamshell mode connected to power. See the README's
  "Scheduling environment" section for the full picture.

## What NOT to do

- Do not send connect requests from this skill. `daily-connect` is the
  only skill allowed to send connects.
- Do not crawl the saved search inline. Delegate to `crawl-lead-list`
  so duplicate-detection and UI-change handling live in one place.
- Do not silently move leads from one campaign to another. A lead
  belongs to exactly one campaign at a time. Resolve conflicts by
  stopping the other campaign first.
- Do not write to `03_drafts/` or touch the campaign spec file.
