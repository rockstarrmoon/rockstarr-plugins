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
| Analytics          | Google Analytics       | Plausible, Fathom, built-in CRM analytics        |

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

| Key                             | Type              | Prompt / guidance                                                                                       |
|---------------------------------|-------------------|---------------------------------------------------------------------------------------------------------|
| `outreach_tool`                 | fixed             | Always `salesnav` for this variant. Write it; don't ask.                                                |
| `linkedin_expected_profile_url` | URL               | The client's personal LinkedIn profile URL (the account sends will go out from). Full `https://www.linkedin.com/in/...` form. |
| `outreach_daily_run_time`       | HH:MM (24h)       | What local time of day the daily send loop should fire. Default `"09:00"` client local.                 |
| `outreach_daily_preview`        | boolean           | Write the daily preview file before sending? Default `true`. Offer `true` / `false`.                    |
| `booking_mode`                  | automated / manual | Should the bot drive the booking link itself (`automated`) or does the client book manually (`manual`)? |
| `availability_source`           | booking_link / gcal | Where does the bot read availability from? Booking link form, or Google Calendar?                      |
| `booking_link_url`              | URL               | Ask **only if** `availability_source: booking_link`. The booking URL (Calendly, GrowthAmp calendar, etc.). |
| `booking_link_required_fields`  | list              | Ask **only if** `availability_source: booking_link`. Which fields the form requires (e.g., `[email, phone, company]`). Free-text; accept a comma-separated list and normalize to a YAML sequence. |
| `gcal_id`                       | string            | Ask **only if** `availability_source: gcal`. The Google Calendar ID to read availability from.          |

Rules:

- Do not hand the booking link to the lead. Ever. `booking_link_url` is
  for the bot to drive the form, not to share in messages.
- If `booking_mode: manual`, still capture `availability_source` — the
  bot reads it to propose times in replies, even though a human books.
- If the client is on a different LI outreach tool (Interceptly,
  MeetAlfred, Dripify, Waalaxy), skip this whole section.

## Content cadence

`rockstarr-content` gates every drafting lane on these numbers.
Always capture them during intake, even when the client is not yet on a
Content package — defaults of `0` / `false` / `1` are valid and mean
"don't run that lane." One `AskUserQuestion` per field; no batching.
Offer the default as the pre-filled answer.

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

4. Ask each content-cadence key, one `AskUserQuestion` per turn, using
   the defaults as pre-filled answers. Always run this — even if the
   client has not bought Content yet, capture zeros so downstream bots
   can read a complete stack.

5. Write `/rockstarr-ai/00_intake/stack.md`. The file has up to four
   sections: front-matter, the categories table, a content-cadence
   YAML block (always), and — when the LI outreach tool is Sales Nav
   only — an "Outreach configuration" YAML block.

   Front-matter and table:

   ~~~markdown
   # ---
   client_id: "<from client.toml>"
   captured_at: "<ISO>"
   capture_skill_version: "0.3.0"
   # ---

   # <Client Name> — Stack

   | Category         | Tool                 | Access    | Notes        |
   |------------------|----------------------|-----------|--------------|
   | CRM              | The Growth Amplifier | granted   | ...          |
   | LI outreach tool | Sales Nav only       | pending   | ...          |
   | ...              | ...                  | ...       | ...          |
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
   outreach_daily_run_time: "09:00"
   outreach_daily_preview: true
   booking_mode: automated
   availability_source: booking_link
   booking_link_url: https://calendly.com/<client>/<slug>
   booking_link_required_fields:
     - email
     - phone
     - company
   # gcal_id only present when availability_source: gcal
   ~~~

   Omit the "Outreach configuration" block entirely when the client is
   on a different LI outreach tool. Only include the key for the chosen
   `availability_source` — booking-link keys OR `gcal_id`, never both.

6. Compute the **variant enable list** — the list of Rockstarr bot
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
   - `rockstarr-content` and `rockstarr-social` (base) are installed
     for all clients who purchased Content / Social.

7. Print the variant enable list to the user with a clear CTA:

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
