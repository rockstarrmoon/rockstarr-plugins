---
name: launch-campaign-interceptly
description: "This skill should be used when the user says \"launch the campaign\", \"configure this campaign in Interceptly\", \"set up the Interceptly campaign for <slug>\", or names an approved campaign spec in 04_approved/outreach/. It configures the approved campaign inside Interceptly via Chrome MCP — filter criteria, 3-step sequence copy, cadence — and records the Interceptly campaign id in the Campaigns sheet. STOPS at CONFIGURED, NOT STARTED. The client operator presses Start inside Interceptly; the bot never starts a campaign on its own."
---

# launch-campaign-interceptly

Translates an approved campaign spec into a live Interceptly
configuration. Stops one button short of sending — the human
operator presses Start.

## When to run

- User says "launch the <slug> campaign" and the approved spec
  exists at `/04_approved/outreach/campaign-<slug>.md`.
- After `rockstarr-infra:approve` promotes a campaign draft.

## Preconditions

- `/04_approved/outreach/campaign-<slug>.md` exists (promoted by
  `approve`). If only the draft exists, refuse and point at
  `approve`.
- `/02_inputs/outreach/outreach-mirror.xlsx` exists. If not, create
  it with the schema (see Section 10 of the plugin spec).
- Chrome MCP is available.
- `confirm-session-interceptly` passes for the first managed
  account the campaign will be configured under. If the spec is
  scoped to a specific account, confirm against that account; if
  it's `all_accounts`, confirm against the first managed account
  and repeat for each.

## Inputs

- `/04_approved/outreach/campaign-<slug>.md` — the approved spec.
- `/00_intake/stack.md` — managed accounts list for `all_accounts`
  scope.

## Behavior

### Step 1 — Resolve scope

If scope is `all_accounts`, this skill runs N Interceptly
configurations — one per managed account. For each, run Steps 2-6
in sequence, switching accounts between via
`switch-account` + `confirm-session-interceptly`.

If scope is a specific `account_label`, run Steps 2-6 once against
that account.

### Step 2 — Open the campaign editor

Navigate Chrome MCP to `https://dash.interceptly.ai/` and open
the Campaigns view. Click `Create campaign` (text match; the
button text has been stable across UI changes).

### Step 3 — Fill filter criteria

From the spec's `## Filter summary`, translate each bullet to the
Interceptly filter UI:

- Titles → Title filter, comma-separated
- Seniority → Seniority filter
- Company size → Employee count bands
- Industry → Industry filter (include + exclude)
- Location → Geography filter
- Excluded companies → Exclude companies list

Known UI quirk: the industry dropdown is a virtualized list. If
the value is not visible, type to filter — do NOT scroll blindly,
it rubber-bands and loses selection.

### Step 4 — Paste sequence copy

Interceptly's sequence editor has three slots: connect note and
two follow-ups. Paste verbatim from the approved spec:

- Step 1 → Connect note field (enforce Interceptly's ~200-char
  limit; the spec should already respect it, but validate).
- Step 2 → Follow-up #1 field.
- Step 3 → Follow-up #2 field.

Set the cadence days per the spec's `## Cadence` section
(default day +3 and day +7).

Known Chrome MCP quirk: Interceptly's composer is a React
textarea. Use the React native-setter pattern (the pattern
preserved in the Rockstarr Chrome MCP automation playbook) so the
app actually registers the input. Direct `.value =` does NOT fire
React's onChange; use the native setter + a synthetic input event.

### Step 5 — Save as CONFIGURED, do NOT press Start

After filling filters and sequence, click Save / Save & Close (do
NOT click Start / Launch). Read back the Interceptly campaign id
from the URL or the campaign-list row the save creates.

If the UI offers a confirmation prompt that includes a Start
option, explicitly choose the save-without-starting path. If there
is no save-without-starting path, abort and write a loud block to
`/02_inputs/outreach/_errors.md` — Interceptly UI has changed and
the bot won't risk auto-start.

### Step 6 — Record in the mirror

Append a row to the `Campaigns` sheet of
`/02_inputs/outreach/outreach-mirror.xlsx`:

| column | value |
|---|---|
| campaign_slug | `<slug>` |
| interceptly_campaign_id | `<id from Step 5>` |
| account_label | `<managed account under which configured>` |
| scope | `all_accounts` or `<account_label>` |
| status | `configured` |
| configured_at | `<ISO>` |
| started_at | (blank — human fills when they press Start) |
| spec_path | `/04_approved/outreach/campaign-<slug>.md` |

If scope is `all_accounts`, this step writes N rows (one per
managed account). The slug is shared; the Interceptly campaign id
differs per account.

### Step 7 — Tell the human

Return a message the user can act on:

> Campaign `<slug>` is configured under `<account_label>` but NOT
> started. Press `Start` in Interceptly when you're ready.
> Interceptly will send the connect note on the cadence you
> configured. Every reply routes through the daily loop's inbox
> pass.

If scope is `all_accounts`, list every account under which the
campaign was configured.

## Outputs

- N new rows in `Campaigns` sheet of `outreach-mirror.xlsx`.
- Campaign live (but not started) in Interceptly.

## Failure modes

- **Filter UI changed.** Abort, write an _errors.md block, tell
  the user to configure manually and then call
  `register-campaign-manually` (V1.1 skill — not in V0.1). For
  V0.1, manual fallback is acceptable if the user can record the
  Interceptly campaign id themselves.
- **Sequence paste silently fails.** Use the React native-setter
  pattern. If that fails twice, abort. Do NOT save a half-filled
  sequence.
- **Accidentally clicked Start.** Immediately pause the campaign
  in Interceptly, write a loud _errors.md block, tell the user.
  This is the one misfire mode this skill guards against.

## What NOT to do

- Do not press Start. Ever. If the UI funnels the save flow into a
  Start action, abort.
- Do not configure a campaign that isn't in `/04_approved/`.
  Drafts in `/03_drafts/` have not been approved.
- Do not edit the spec mid-configuration. If the spec looks wrong,
  abort and ask the user to re-run `draft-icp-campaign`.
- Do not fabricate an Interceptly campaign id if you cannot read
  one from the UI. Abort and say so.
