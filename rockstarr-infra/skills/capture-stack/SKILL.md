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
are variant-specific (e.g., `rockstarr-outreach-bot-interceptly` vs.
`rockstarr-outreach-bot-dripify`); this skill is what decides which
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
  (e.g., `rockstarr-ops-bot-ga`). Never `gha`.

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

## Steps

1. If `/rockstarr-ai/00_intake/stack.md` already exists, read it first and
   use its values as defaults in the questions. Treat this run as an
   update rather than a blank slate.

2. Ask each category as a separate `AskUserQuestion` call. Single question
   per turn; no batching.

3. Write `/rockstarr-ai/00_intake/stack.md`:

   ```markdown
   # ---
   client_id: "<from client.toml>"
   captured_at: "<ISO>"
   capture_skill_version: "0.1.0"
   # ---

   # <Client Name> — Stack

   | Category         | Tool                 | Access    | Notes        |
   |------------------|----------------------|-----------|--------------|
   | CRM              | The Growth Amplifier | granted   | ...          |
   | LI outreach tool | Interceptly          | pending   | ...          |
   | ...              | ...                  | ...       | ...          |
   ```

4. Compute the **variant enable list** — the list of Rockstarr bot
   plugin names that match this stack. Map:

   - CRM = The Growth Amplifier → `rockstarr-ops-bot-ga`,
     `rockstarr-nurture-bot-ga`
   - CRM = HubSpot → `rockstarr-ops-bot-hubspot`,
     `rockstarr-nurture-bot-hubspot`
   - CRM = Pipedrive → `rockstarr-ops-bot-pipedrive`,
     `rockstarr-nurture-bot-pipedrive`
   - CRM = Salesforce → `rockstarr-ops-bot-salesforce`,
     `rockstarr-nurture-bot-salesforce`
   - CRM = Close → `rockstarr-ops-bot-close`,
     `rockstarr-nurture-bot-close`
   - LI tool = Interceptly → `rockstarr-outreach-bot-interceptly`
   - LI tool = MeetAlfred → `rockstarr-outreach-bot-meetalfred`
   - LI tool = Dripify → `rockstarr-outreach-bot-dripify`
   - LI tool = Waalaxy → `rockstarr-outreach-bot-waalaxy`
   - LI tool = Sales Nav only → `rockstarr-outreach-bot-salesnav`
   - Email = Gmail → `rockstarr-reply-bot-gmail`
   - Email = Outlook → `rockstarr-reply-bot-outlook`
   - Email = Same as CRM → no separate reply-bot variant; the CRM's
     ops-bot handles inbox actions.
   - Social scheduler = Publer → `rockstarr-social-bot` (Publer is the
     default; no variant)
   - Social scheduler = Buffer / Hootsuite / Later →
     `rockstarr-social-bot-<tool>`
   - Content Bot and Social Bot (base) are installed for all clients who
     purchased Content / Social.

5. Print the variant enable list to the user with a clear CTA:

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
