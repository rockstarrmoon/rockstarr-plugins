---
name: register-campaign
description: "This skill should be used when the user says \"register this campaign\", \"launch the campaign\", \"go live on this campaign\", or names an approved campaign spec in 04_approved/outreach/. It promotes the approved campaign, reads the campaign_type front-matter (full-sequence or connect-only), validates that connect-only specs do not contain a Sequence section, adds a Campaigns row to outreach-tasks.xlsx with the type recorded, invokes crawl-lead-list to populate Leads from the saved Sales Navigator search URL, and seeds the initial connect tasks. Fails loudly if any lead already belongs to another active campaign."
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

- `/rockstarr-ai/04_approved/outreach/campaign-<slug>.md` exists.
- `/rockstarr-ai/00_intake/stack.md` has `outreach_tool: salesnav` and
  a valid `linkedin_expected_profile_url`.
- `outreach-tasks.xlsx` exists at
  `/rockstarr-ai/02_inputs/outreach/outreach-tasks.xlsx`. Create it
  (with empty sheets following the README schema) if missing.
- No existing `Campaigns` row with the same `campaign_slug` in
  `status = active` or `paused`. (Re-registering a `stopped` campaign
  with the same slug is allowed; leads previously in a `stopped`
  campaign may be re-used.)

If any precondition fails, stop and explain what is missing. Do not
partially register.

## Inputs

Read:

1. `04_approved/outreach/campaign-<slug>.md` — full front-matter and
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

1. **Open the workbook.** Use the shared `xlsx` skill. Read the
   Campaigns, Leads, Tasks sheets into memory.
2. **Write the Campaigns row.** Append one row:
   - `campaign_slug`: the slug
   - `campaign_type`: `full-sequence` or `connect-only` (from spec
     front-matter; default to `full-sequence` for back-compat if
     absent on a pre-0.1.7 spec)
   - `icp_ref`: path to the ICP file
   - `lead_list_saved_search_url`: from front-matter
   - `target_lead_count`: from front-matter
   - `status`: `active`
   - `date_started`: today's ISO date
   - `date_stopped`: empty
   - `notes`: the campaign title + anything from the spec header
3. **Invoke `crawl-lead-list`** with the saved-search URL, target
   count, and slug. Wait for completion.
4. **Check for conflicts.** If `crawl-lead-list` reports any leads
   that already belong to another active or paused campaign:
   - Do **not** proceed.
   - Log the conflict list to `/02_inputs/outreach/_errors.md` with a
     timestamp, the slug that tried to register, and the conflicting
     lead URLs + their current campaign.
   - Roll back the Campaigns row.
   - Tell the user which leads conflict and ask them to either remove
     the leads from the saved search, or stop the other campaign,
     before retrying.
5. **Seed connect tasks.** For every new lead in Leads with
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
6. **Save the workbook.** Let the `xlsx` skill handle the write +
   validation.
7. **Log.** Append a line to `/05_published/outreach/<today>.md`:
   `registered campaign <slug> — <lead_count> leads seeded, connects queued`.

## Output

Return a short summary to the user:

- Campaign slug + title
- Number of leads seeded
- Number of conflicts rejected (should be 0 if we reached this step)
- A reminder that `daily-connect` will start sending at
  `stack.md.outreach_daily_run_time`.

## What NOT to do

- Do not send connect requests from this skill. `daily-connect` is the
  only skill allowed to send connects.
- Do not crawl the saved search inline. Delegate to `crawl-lead-list`
  so duplicate-detection and UI-change handling live in one place.
- Do not silently move leads from one campaign to another. A lead
  belongs to exactly one campaign at a time. Resolve conflicts by
  stopping the other campaign first.
- Do not write to `03_drafts/` or touch the campaign spec file.
