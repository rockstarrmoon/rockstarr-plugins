---
name: capture-stack
description: "This skill should be used when the user asks to \"capture the stack\", \"record the client stack\", \"set up stack.md\", or during intake when configuring which tools the client uses. It interactively asks the client (or the installer) which CRM, LinkedIn outreach tool, website platform, social scheduler, email platform, and analytics tool the client uses, then writes /rockstarr-ai/00_intake/stack.md and returns the list of bot variants that should be installed."
---

# capture-stack

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as frontmatter
> separators. **When writing actual output files, emit real `---`, not
> `# ---`.**

Interview the client (or the Rockstarr installer working with the
client) about their tooling and write `stack.md`. Downstream bot plugins
are variant-specific (e.g., `rockstarr-outreach-interceptly` vs.
`rockstarr-outreach-dripify`); this skill is what decides which
variants get enabled for this client.

## When to run

- During the 60-minute intake call, right after the workbook review.
- Any time the client switches tools (e.g., moves from HubSpot to The
  Growth Amplifier).

## Preconditions

- `scaffold-client` has run.
- The user running this skill has consent to speak for / with the client
  on tooling choices. When there is ambiguity, ask the client directly
  rather than guessing.

## Naming conventions

- When referring to Rockstarr's CRM product, always write "The Growth
  Amplifier" (formal) or "GrowthAmp" (informal). Never use the
  abbreviation "GHL".
- In plugin slugs, the Growth Amplifier is abbreviated `ga`
  (e.g., `rockstarr-ops-ga`). Never `gha`.

## Categories and default options

Ask about each of these categories. Use `AskUserQuestion` with a
multiple-choice list where the options below are the choices. Always
allow a free-text "Other" answer.

| Category           | Rockstarr default      | Common alternatives                              |
|--------------------|------------------------|--------------------------------------------------|
| CRM                | The Growth Amplifier   | HubSpot, Pipedrive, Salesforce, Close            |
| LI outreach tool   | Interceptly            | MeetAlfred, Dripify, Waalaxy, Sales Nav only     |
| Website platform   | WordPress              | Webflow, Squarespace, Wix, Framer, GrowthAmp     |
| Social scheduler   | Publer                 | Buffer, Hootsuite, Later, GrowthAmp social       |
| Email platform     | Same as CRM            | Gmail, Outlook, Mailchimp, ConvertKit            |
| Task system        | ClickUp                | Asana, Linear, Notion, none                      |
| Analytics          | Google Analytics       | Plausible, Fathom, built-in CRM analytics        |

The Task system row gates every task-management integration in the
plugin set (today: the ClickUp-closure step inside
`rockstarr-social:li-comment-check`). Capture it even when the client
isn't on a Rockstarr feature that uses it — the answer is what tells
downstream skills whether to ask about task IDs at all. Pick `none`
for clients who don't run a task system; that suppresses every
task-related capture and makes every task-closure feature a no-op.
Only `clickup` is wired into a feature today; `asana` / `linear` /
`notion` are accepted for forward-compat and will light up when their
adapters ship.

For each category, capture:

1. The chosen tool.
2. Whether admin access has been granted to Rockstarr (yes / pending / no).
3. Any notes (multiple accounts, sub-accounts, region, etc.).

## Variant-specific follow-up keys

After the main categories, some variants need extra configuration. Ask
these only when the matching tool was selected. Like the main categories,
each is its own `AskUserQuestion` turn — no batching.

### LI outreach tool = Sales Nav only

The `rockstarr-outreach-salesnav` plugin refuses to run without these
keys. Capture every one during intake:

| Key                             | Type                              | Prompt / guidance                                                                                       |
|---------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------|
| `outreach_tool`                 | fixed                             | Always `salesnav` for this variant. Write it; don't ask.                                                |
| `linkedin_expected_profile_url` | URL                               | The client's personal LinkedIn profile URL (the account sends will go out from). Full `https://www.linkedin.com/in/...` form. |
| `outreach_campaign_mode`        | full / connect_only / none        | The client's **default campaign preference**. `full` = full-sequence (connect + 3 post-accept messages) is the default; `connect_only` = connect-only is the default; `none` = no campaigns by default (e.g., the client bought the plugin to draft copy but isn't ready to execute). This is a default, not a hard gate — `draft-icp-campaign` still runs on demand and surfaces an explicit confirm prompt when the requested campaign type deviates from the captured preference. Default value at intake: `full`. |
| `outreach_daily_run_time`       | HH:MM (24h)                       | What local time of day the daily send loop should fire. Default `"09:00"` client local.                 |
| `outreach_daily_preview`        | boolean                           | Write the daily preview file before sending? Default `true`. Offer `true` / `false`.                    |
| `booking_mode`                  | automated / manual                | Should the bot drive the booking link itself (`automated`) or does the client book manually (`manual`)? |
| `availability_source`           | booking_link / gcal               | Where does the bot read availability from? Booking link form, or Google Calendar?                      |
| `booking_link_url`              | URL                               | Ask **only if** `availability_source: booking_link`. The booking URL (Calendly, GrowthAmp calendar, etc.). |
| `booking_link_required_fields`  | list                              | Ask **only if** `availability_source: booking_link`. Which fields the form **requires** (booking aborts if any of these has no source value at send-time — typically just `[email]` for Cal.com, sometimes `[email, phone, company]` for richer forms). Free-text; accept a comma-separated list and normalize to a YAML sequence. |
| `booking_link_optional_fields`  | list                              | Ask **only if** `availability_source: booking_link`. Which fields the form accepts but does not require (e.g., `[company, notes]`). Booking fills these from `Leads.company` etc. when a value is available and leaves them empty otherwise — they never block. Free-text; accept a comma-separated list and normalize to a YAML sequence. If the client doesn't know, default to `[]`. |
| `gcal_id`                       | string                            | Ask **only if** `availability_source: gcal`. The Google Calendar ID to read availability from.          |

Rules:

- Do not hand the booking link to the lead. Ever. `booking_link_url` is
  for the bot to drive the form, not to share in messages.
- If `booking_mode: manual`, still capture `availability_source` — the
  bot reads it to propose times in replies, even though a human books.
- If the client is on a different LI outreach tool (Interceptly,
  MeetAlfred, Dripify, Waalaxy), skip this whole section.

### Local-area face-to-face handling (toggle-gated)

Some clients have a strong local presence (e.g., Austin / Leander, TX
for RigTex) and want **in-person meeting requests handled differently
from remote bookings** — typically routed to manual review so the
client schedules coffee personally instead of the bot auto-proposing
booking-link slots.

Ask `local_meet_handoff_enabled` first. If `false`, skip the rest
of this subsection.

| Key                              | Type    | Default        | Prompt / guidance                                                                                                                                                                                                              |
|----------------------------------|---------|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `local_meet_handoff_enabled`     | boolean | `false`        | Should `rockstarr-reply` route in-person meeting requests to manual review instead of proposing booking-link slots? Set to `true` only if the client wants coffee meetings handled personally.                                  |
| `local_meet_trigger_terms`       | list    | (see below)    | Words / phrases that signal a local face-to-face ask. Ask the client to confirm the suggested defaults and add local city names. Suggested defaults: `["coffee", "in person", "in-person", "meet up", "meetup", "stop by", "grab coffee", "grab a coffee", "lunch"]` + local cities (e.g., for an Austin-based client: `["austin", "round rock", "cedar park", "leander", "georgetown", "pflugerville"]`). Case-insensitive substring match against the inbound message text. |

Rules:

- The trigger-terms list applies to ALL outreach variants the
  client runs (Sales Nav, Interceptly, etc.) — `rockstarr-reply`
  reads it from `stack.md` regardless of caller.
- Empty list with `local_meet_handoff_enabled: true` is allowed
  but pointless — the feature won't fire on any message. Warn the
  user.
- When a match fires, `rockstarr-reply` routes the thread to its
  flag-for-review path with reason `local_face_to_face_request`.
  The operator handles the coffee scheduling outside the bot.

## Social configuration (rockstarr-social)

`rockstarr-social` reads several blocks out of `stack.md` to drive
weekly drafting, channel routing, scheduler export, polls, page-follower
invites, and the daily LinkedIn comment check. Mirror the
content-cadence pattern: always capture the always-blocks (defaults of
`0` / `false` / `off` are valid and mean "don't run that lane"), and
ask the conditional blocks only when the gating answer warrants them.
One `AskUserQuestion` per field; no batching.

### Drafting + weekly batch (always)

| Key                          | Type        | Default        | Prompt / guidance                                                                                  |
|------------------------------|-------------|----------------|----------------------------------------------------------------------------------------------------|
| `social_posts_per_week`      | integer     | `5`            | How many short-form posts per week should Social Bot draft? `0` is a valid answer — `fill-week` no-ops. |
| `social_mix.promo`           | integer     | `1`            | Promo slots per week.                                                                              |
| `social_mix.insight`         | integer     | `2`            | Insight slots per week.                                                                            |
| `social_mix.case`            | integer     | `1`            | Case-story slots per week.                                                                         |
| `social_mix.engagement`      | integer     | `1`            | Engagement-prompt slots per week.                                                                  |
| `social_default_post_time`   | HH:MM (24h) | `"09:00"`      | Default time-of-day for scheduled posts (client-local).                                            |
| `social_fill_week_cron`      | cron        | `"0 9 * * 5"`  | When `fill-week` runs each week. Default Friday 09:00 client-local.                                |

Rules:

- The four `social_mix` slots should sum to `social_posts_per_week`. If
  they don't, ask the client to reconcile rather than silently
  re-balancing.
- A `social_posts_per_week: 0` is acceptable for clients who buy
  Social later — capture the defaults of the mix anyway so the file is
  shaped consistently.

### Channels (always)

For each channel, ask y/n in its **own** `AskUserQuestion` turn.
LinkedIn defaults `true`; X, Instagram, and Google Business
default `false`.

| Key                                | Type    | Default  | Prompt                                          |
|------------------------------------|---------|----------|-------------------------------------------------|
| `social_channels.linkedin`         | boolean | `true`   | Publish to LinkedIn?                            |
| `social_channels.x`                | boolean | `false`  | Publish to X (formerly Twitter)?                |
| `social_channels.instagram`        | boolean | `false`  | Publish to Instagram?                           |
| `social_channels.google_business`  | boolean | `false`  | Publish to Google Business Profile?             |

### Publer account labels (deferred to `rockstarr-social`)

**Do not ask the client to type Publer labels during onboarding.**
Most clients don't know the exact label strings, and asking them
to guess produces wrong values that break `publer-export` silently.

Instead, write the `publer_accounts` block to `stack.md` with every
enabled channel's value set to `null` (a placeholder). When the
client first runs `rockstarr-social`, that plugin opens Publer via
Chrome MCP, reads the Social Accounts list, and lets the client
pick the correct label per channel — then writes the resolved
labels back into `stack.md`'s `publer_accounts` block. Same
deferral pattern `rockstarr-outreach-salesnav` uses for discovering
managed accounts.

Skip this entire subsection if `social_scheduler` is `growthamp` or
`native` — Publer labels don't apply there.

| Key                                  | Type           | Behavior                                                                            |
|--------------------------------------|----------------|-------------------------------------------------------------------------------------|
| `publer_accounts.linkedin`           | string \| null | Write `null`. Populated by `rockstarr-social` on first run when LinkedIn is enabled. |
| `publer_accounts.x`                  | string \| null | Write `null` only if `social_channels.x: true`. Populated by `rockstarr-social`.     |
| `publer_accounts.instagram`          | string \| null | Write `null` only if `social_channels.instagram: true`. Populated by `rockstarr-social`. |
| `publer_accounts.google_business`    | string \| null | Write `null` only if `social_channels.google_business: true`. Populated by `rockstarr-social`. |

### Polls cadence (always)

| Key              | Type | Default     | Prompt                                                                                          |
|------------------|------|-------------|-------------------------------------------------------------------------------------------------|
| `polls_cadence`  | enum | `"monthly"` | LinkedIn polls cadence. Choose `monthly` / `quarterly` / `on-demand` / `off`.                   |

### Page-follower invites (toggle-gated)

Ask `page_invite_enabled` first. If `false`, skip the rest of this
subsection.

| Key                              | Type    | Default               | Prompt / guidance                                                                                          |
|----------------------------------|---------|-----------------------|------------------------------------------------------------------------------------------------------------|
| `page_invite_enabled`            | boolean | `false`               | Run the monthly LinkedIn page-follower invite?                                                             |
| `page_invite_target_url`         | URL     | —                     | Public URL for the LinkedIn company page (`https://www.linkedin.com/company/<slug>/`).                     |
| `page_invite_company_id`         | string  | —                     | Numeric LinkedIn company id (visible in the page admin URL).                                               |
| `page_invite_admin_display_name` | string  | —                     | Admin's full name as shown on their LinkedIn profile photo. Used in the strict identity gate.              |
| `page_invite_schedule_cron`      | cron    | `"0 14 8-14 * 2"`     | When the monthly invite run fires. Default 2pm second Tuesday client-local.                                |
| `page_invite_credit_target`      | integer | `50`                  | Target invites per cycle. The skill caps at the lower of this number and the credits LinkedIn shows.       |

### Daily LinkedIn comment check (toggle-gated)

Ask `li_comment_check_enabled` first. If `false`, skip the rest of
this subsection.

| Key                              | Type    | Default                | Prompt / guidance                                                                                       |
|----------------------------------|---------|------------------------|---------------------------------------------------------------------------------------------------------|
| `li_comment_check_enabled`       | boolean | `false`                | Run the daily LinkedIn comment-check workflow?                                                          |
| `li_comment_check_cron`          | cron    | `"30 8 * * 1-5"`       | When the daily comment check fires. Default 08:30 weekdays client-local.                                |
| `li_comment_clickup_enabled`     | boolean | `false`                | **Ask only when `task_system: clickup`.** Auto-close the matching ClickUp task after a comment is sent? Opt-in. When `task_system` is anything else, write `false` without asking. |
| `li_comment_accounts`            | list    | `[]`                   | One entry per managed LinkedIn account the bot monitors. Captured in a sub-loop (see below).            |

For `li_comment_accounts`, run a per-account sub-loop. Per managed
account, ask one `AskUserQuestion` at a time:

| Field             | Type   | Required | Prompt                                                                                                |
|-------------------|--------|----------|-------------------------------------------------------------------------------------------------------|
| `name`            | string | yes      | Account holder's full name (e.g. `"Phil Katz"`).                                                      |
| `slug`            | string | yes      | The `linkedin.com/in/<slug>` slug for that account (e.g. `"philipkatz1"`).                            |
| `voice_notes`     | text   | yes      | Short description of the account holder's reply voice. The skill drafts replies in this voice.        |
| `brand_kit_url`   | URL    | no       | Optional link to a brand kit or style sheet the bot should reference for this account.                |
| `clickup_task_id` | string | no       | **Ask only when `task_system: clickup`.** ClickUp task id the comment-check loop closes when a send completes. Required only when `li_comment_clickup_enabled: true`. Omit the field entirely when `task_system` is anything else. |

After each account, ask "Add another managed account? (y/n)". Stop the
sub-loop on `n`.

Rules:

- A list of zero managed accounts is valid — the skill no-ops cleanly.
- Every ClickUp-touching capture is gated on the categories-table
  Task system answer. If `task_system != clickup`, write
  `li_comment_clickup_enabled: false` without asking, omit
  `clickup_task_id` from every account record, and proceed. The
  comment-check skill silently skips its closure step when the
  toggle is off, so the workflow still runs end-to-end.
- `clickup_task_id` is irrelevant when `li_comment_clickup_enabled:
  false` even within a ClickUp workspace; capture it anyway when
  offered, so flipping the flag later doesn't require re-
  interviewing.
- Voice notes are short — one or two sentences. Treat anything longer
  than a paragraph as a smell that the brand-kit URL would serve
  better.

## Content cadence

`rockstarr-content` gates every drafting lane on these numbers.
Always capture them during intake, even when the client is not yet on a
Content package — defaults of `0` / `false` / `1` are valid and mean
"don't run that lane." **One `AskUserQuestion` per field; no batching.**
Offer the default as the pre-filled answer.

**Concrete:** do NOT ask "What's your content cadence?" as a single
question expecting the client to type six numbers. Ask six separate
turns, one per row in the table below. Each turn shows the field's
prompt and accepts a single free-text integer (or yes/no for
`records_videos`). Six questions; six short answers. The shared
voice reference's "one question at a time" rule applies here without
exception.

| Key                              | Type    | Default | Prompt / guidance                                                                                   |
|----------------------------------|---------|---------|-----------------------------------------------------------------------------------------------------|
| `blogs_per_month`                | integer | `0`     | How many long-form blog posts per month should Content Bot draft?                                   |
| `thought_leadership_per_month`   | integer | `0`     | How many thought-leadership articles (anchor pieces) per month?                                     |
| `email_newsletters_per_month`    | integer | `0`     | How many email newsletters per month?                                                               |
| `linkedin_newsletters_per_month` | integer | `0`     | How many LinkedIn newsletter sends per month?                                                       |
| `records_videos`                 | boolean | `false` | Does the client record videos (YouTube / podcast / keynote) that Content Bot can transcribe and repurpose? |
| `case_studies_per_quarter`       | integer | `1`     | How many case studies per quarter? Drives the `draft-case-study` interview cadence.                 |

Rules:

- A `0` is a valid answer and is common. Don't badger clients into a
  higher number.
- `records_videos: true` unlocks video-derived drafting lanes in
  Content Bot; `false` keeps them quiet.
- Cadence is a planning signal, not a throttle. Content Bot checks it
  before ideating or drafting; it does not auto-publish on a schedule.

## Steps

1. If `/rockstarr-ai/00_intake/stack.md` already exists, read it first and
   use its values as defaults in the questions. Treat this run as an
   update rather than a blank slate.

2. Ask each category as a separate `AskUserQuestion` call. Single question
   per turn; no batching.

3. If the LI outreach tool answer was "Sales Nav only", ask each
   variant-specific key from the table above, one `AskUserQuestion` per
   turn. Skip the booking-link keys or the `gcal_id` key based on the
   `availability_source` answer.

4. Walk the **Social configuration** section. Always ask the always-
   blocks (drafting + weekly batch, channels, polls cadence). Ask the
   Publer-account-labels subsection only when `social_scheduler:
   publer`. Ask the page-invite block only when
   `page_invite_enabled: true`. Ask the comment-check block only when
   `li_comment_check_enabled: true`, and run the per-account sub-loop
   inside it until the user says no. Always capture this section even
   when the client has not bought Social yet — defaults of `0` /
   `false` / `off` keep the file shaped consistently.

5. Ask each content-cadence key, one `AskUserQuestion` per turn, using
   the defaults as pre-filled answers. Always run this — even if the
   client has not bought Content yet, capture zeros so downstream bots
   can read a complete stack.

6. Write `/rockstarr-ai/00_intake/stack.md`. The file has up to five
   sections: front-matter, the categories table, the social-
   configuration YAML block (always), the content-cadence YAML block
   (always), and — when the LI outreach tool is Sales Nav only — an
   "Outreach configuration" YAML block.

   Front-matter and table:

   ~~~markdown
   # ---
   client_id: "<from client.toml>"
   captured_at: "<ISO>"
   capture_skill_version: "0.5.0"
   # ---

   # <Client Name> — Stack

   | Category         | Tool                 | Access    | Notes        |
   |------------------|----------------------|-----------|--------------|
   | CRM              | The Growth Amplifier | granted   | ...          |
   | LI outreach tool | Sales Nav only       | pending   | ...          |
   | ...              | ...                  | ...       | ...          |
   ~~~

   Always append this social-configuration YAML block under a
   `## Social configuration` heading. Show only the subsections that
   apply — omit `publer_accounts` when `social_scheduler` is not
   `publer`, omit the page-invite block when `page_invite_enabled:
   false`, omit the comment-check block when
   `li_comment_check_enabled: false`. Within an enabled section,
   always emit every key:

   ~~~yaml
   # Drafting + weekly batch
   social_posts_per_week: 5
   social_mix:
     promo: 1
     insight: 2
     case: 1
     engagement: 1
   social_default_post_time: "09:00"
   social_fill_week_cron: "0 9 * * 5"

   # Channels
   social_channels:
     linkedin: true
     x: false
     instagram: false
     google_business: false

   # Publer account labels (only when social_scheduler: publer)
   # Every value is null at intake — rockstarr-social populates them
   # on its first run via Chrome MCP. Emit a key per enabled channel
   # so the schema is stable; downstream skills key off the presence
   # of the field, not its value.
   publer_accounts:
     linkedin: null
     # x, instagram, google_business only when the matching
     # social_channels.* is true

   # Polls
   polls_cadence: "monthly"

   # Page-follower invites (only when page_invite_enabled: true)
   page_invite_enabled: false
   page_invite_target_url: "https://www.linkedin.com/company/<slug>/"
   page_invite_company_id: "<numeric id>"
   page_invite_admin_display_name: "<full name>"
   page_invite_schedule_cron: "0 14 8-14 * 2"
   page_invite_credit_target: 50

   # Daily LI comment check (only when li_comment_check_enabled: true)
   # Omit li_comment_clickup_enabled and every account's
   # clickup_task_id when task_system is not "clickup".
   li_comment_check_enabled: false
   li_comment_check_cron: "30 8 * * 1-5"
   li_comment_clickup_enabled: false           # only when task_system: clickup
   li_comment_accounts:
     - name: "Phil Katz"
       slug: "philipkatz1"
       voice_notes: |
         Corporate, professional. Measured, appreciative,
         authoritative.
       brand_kit_url: "..."
       clickup_task_id: "868fzfgrf"            # only when task_system: clickup
   ~~~

   Always append this content-cadence YAML block under a
   `## Content cadence` heading:

   ~~~yaml
   blogs_per_month: 0
   thought_leadership_per_month: 0
   email_newsletters_per_month: 0
   linkedin_newsletters_per_month: 0
   records_videos: false
   case_studies_per_quarter: 1
   ~~~

   When `LI outreach tool = Sales Nav only`, also append this YAML block
   under a `## Outreach configuration (Sales Nav variant)` heading:

   ~~~yaml
   outreach_tool: salesnav
   linkedin_expected_profile_url: https://www.linkedin.com/in/<handle>
   outreach_campaign_mode: full   # full | connect_only | none — default preference, not a hard gate
   outreach_daily_run_time: "09:00"
   outreach_daily_preview: true
   booking_mode: automated
   availability_source: booking_link
   booking_link_url: https://calendly.com/<client>/<slug>
   booking_link_required_fields:
     - email
   booking_link_optional_fields:
     - company
     - notes
   # gcal_id only present when availability_source: gcal

   # Local-area face-to-face handling — only present when
   # local_meet_handoff_enabled: true.
   local_meet_handoff_enabled: false
   local_meet_trigger_terms:
     - "coffee"
     - "in person"
     - "in-person"
     - "meet up"
     - "stop by"
     - "lunch"
     # plus local city names captured from the client
   ~~~

   Omit the "Outreach configuration" block entirely when the client is
   on a different LI outreach tool. Only include the key for the chosen
   `availability_source` — booking-link keys OR `gcal_id`, never both.

7. Compute the **variant enable list** — the list of Rockstarr bot
   plugin names that match this stack. Map:

   - CRM = The Growth Amplifier → `rockstarr-ops-ga`,
     `rockstarr-nurture-ga`
   - CRM = HubSpot → `rockstarr-ops-hubspot`,
     `rockstarr-nurture-hubspot`
   - CRM = Pipedrive → `rockstarr-ops-pipedrive`,
     `rockstarr-nurture-pipedrive`
   - CRM = Salesforce → `rockstarr-ops-salesforce`,
     `rockstarr-nurture-salesforce`
   - CRM = Close → `rockstarr-ops-close`,
     `rockstarr-nurture-close`
   - LI tool = Interceptly → `rockstarr-outreach-interceptly`
   - LI tool = MeetAlfred → `rockstarr-outreach-meetalfred`
   - LI tool = Dripify → `rockstarr-outreach-dripify`
   - LI tool = Waalaxy → `rockstarr-outreach-waalaxy`
   - LI tool = Sales Nav only → `rockstarr-outreach-salesnav`
   - Email = Gmail → `rockstarr-reply-gmail`
   - Email = Outlook → `rockstarr-reply-outlook`
   - Email = Same as CRM → no separate `rockstarr-reply` variant; the
     CRM's `rockstarr-ops-<crm>` handles inbox actions.
   - Social scheduler = Publer → `rockstarr-social` (Publer is the
     default; no variant)
   - Social scheduler = Buffer / Hootsuite / Later →
     `rockstarr-social-<tool>`
   - Task system = ClickUp / Asana / Linear / Notion / none → no
     plugin variant attaches today. The answer is captured to gate
     task-closure features inside other plugins (currently the
     ClickUp closure step in
     `rockstarr-social:li-comment-check`).
   - `rockstarr-content` and `rockstarr-social` (base) are installed
     for all clients who purchased Content / Social.

8. Print the variant enable list to the user with a clear CTA:

   > Hand this list to the Rockstarr installer. They will enable these
   > plugins in the client's marketplace record.

## What NOT to do

- Do not install any bot plugins from this skill. Installation happens
  through the marketplace by flipping entries in the client's KV record.
- Do not infer tooling from the workbook alone. Always ask — clients
  change tools between workbook submission and intake.
- Do not write admin credentials into `stack.md`. Access status only
  (granted / pending / no).
- Do not use "GHL" anywhere in output — always "The Growth Amplifier"
  or "GrowthAmp".
