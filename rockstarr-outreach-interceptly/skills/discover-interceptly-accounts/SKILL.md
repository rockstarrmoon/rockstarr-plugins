---
name: discover-interceptly-accounts
description: "This skill should be used at install time when the user says \"discover the Interceptly accounts\", \"list my Interceptly accounts\", \"set up which accounts the bot manages\", or \"the client just added a new Interceptly account\". It opens https://dash.interceptly.ai/ via Chrome MCP, walks the SWITCH ACCOUNT popup to enumerate every account the client's Interceptly workspace exposes, and asks the client which subset the bot should manage — plus the rotation order. The chosen list is written to stack.md.outreach_accounts[]. It also enumerates the workspace's custom Labels so create-followup-task and apply-label know what exists. Re-runnable on demand when the roster changes."
---

# discover-interceptly-accounts

Install-time entry point for a client on Interceptly. This skill
cannot be inferred or hardcoded — it has to read the client's real
Interceptly workspace. Every downstream skill depends on the list
it produces.

## When to run

- First install against a brand-new Interceptly client.
- Whenever the client adds or removes an account in Interceptly.
- Whenever the client renames an account or changes which accounts
  the bot should drive.
- When the user says "rediscover accounts" or names a new account
  that `interceptly-accounts.md` does not list.

## Preconditions

- `/rockstarr-ai/00_intake/stack.md` exists and has
  `outreach_tool = interceptly`. If `outreach_tool` is anything
  else, refuse and point the user at `rockstarr-infra:capture-stack`.
- Chrome MCP is available. If it is not connected, tell the user to
  install the Claude in Chrome extension before continuing.
- The client has already logged into Interceptly in Cowork's
  Chrome session. This skill does NOT attempt to log in.

## Inputs

- None beyond the stack. The skill reads Interceptly directly.

## Behavior

### Step 1 — Open Interceptly

Navigate Chrome MCP to `https://dash.interceptly.ai/`. Wait for the
left sidebar to render. If redirected to `/login`, stop and tell
the user to sign into Interceptly in the browser, then re-run.

### Step 2 — Open the SWITCH ACCOUNT popup

The Interceptly sidebar shows the currently-active account near the
top-left. Click the account chip to open the SWITCH ACCOUNT popup.
The popup lists every account the signed-in user has access to.

Known quirks:

- The popup sometimes renders inside a portal that is a sibling of
  the sidebar. Read the DOM, do not rely on positional selectors.
- Account labels can repeat when the same user is on multiple teams.
  Disambiguate by the workspace / team name shown under each label.

### Step 3 — Enumerate accounts

For each entry in the popup, capture:

- `account_label` — the display name Interceptly shows.
- `workspace` — the team / workspace name if present.
- `interceptly_account_url` — the URL the entry points to (the
  click target, not the profile URL on LinkedIn).

Return to the caller (or log for the interview) the full list.

### Step 4 — Enumerate Labels

While here, open any account and visit its Labels view (or Inbox
filter → Labels dropdown). Capture every label the workspace
exposes. Write these to `/00_intake/interceptly-accounts.md` under a
top-level `## Labels` section — they are shared across managed
accounts inside a workspace. Call out which of the six defaults are
present (`INTERESTED`, `Booked`, `Referral`, `Follow Up`,
`Contact Later`, `Bad Fit`, plus `Ignore`, `Not Interested`) and
surface any custom labels the client added.

### Step 5 — Present choices to the client

Use `AskUserQuestion` with one question per account enumerated:
"Should the bot manage `<account_label>` (`<workspace>`)?" with
options `Manage` / `Skip`. If there are more than four, batch —
do not exceed four questions in one `AskUserQuestion` call.

After every account has a manage/skip verdict, ask one more
question with `multiSelect` disabled: "Which managed account should
the daily loop process first?" Options are each managed account in
install order. The answer becomes position 0 in
`outreach_accounts[]`.

If more than two accounts were chosen, ask a follow-up to order
positions 1..N (use `AskUserQuestion` iteratively — "of the
remaining accounts, which runs next?" — rather than presenting a
free-text reorder prompt).

### Step 6 — Map custom labels

If any custom label was found in Step 4, ask the client how it
maps to the bot's internal classifications. Default
classifications: `interested`, `booked`, `referral`, `follow_up`,
`contact_later`, `bad_fit`, `ignore`, `not_interested`. For each
custom label, offer the eight options as `AskUserQuestion` answers.
Write the result into `stack.md.label_mapping` as
`"<Interceptly label>": "<internal classification>"`.

### Step 7 — Write stack.md additions

Update `/rockstarr-ai/00_intake/stack.md`:

- `outreach_accounts[]` — ordered list of
  `{account_label, workspace, interceptly_account_url}` entries the
  client chose to manage.
- `label_mapping` — the overrides from Step 6 (empty if the
  workspace only uses the defaults).

Do NOT write `interceptly_profile_url` here — that belongs to
`capture-interceptly-personas` because it requires the matching
LinkedIn profile URL for `confirm-session-interceptly`.

### Step 8 — Chain into capture-interceptly-personas

On clean finish, prompt the user: "Discovery captured N managed
accounts. Run `capture-interceptly-personas` now to record the
signature, persona notes, and LinkedIn profile URL for each?"
If yes, hand control to that skill.

## Outputs

- Updated `/rockstarr-ai/00_intake/stack.md` with
  `outreach_accounts[]` and (optionally) `label_mapping`.
- Seed file at `/rockstarr-ai/00_intake/interceptly-accounts.md`
  with a stub row per managed account (columns filled during
  `capture-interceptly-personas`) and a `## Labels` section listing
  the enumerated labels.

## Failure modes

- **Chrome MCP dropped mid-enumeration.** Retry once with a 2-3s
  wait. If still failing, write a loud block to
  `/02_inputs/outreach/_errors.md` and abort — do NOT write a
  partial `outreach_accounts[]`.
- **SWITCH ACCOUNT popup opens onto a different layout.** Interceptly
  has shipped UI changes before. Abort and write an error asking
  the user to confirm the account list manually; the capture step
  can proceed with the manual list.
- **Client has more than ~8 managed accounts.** That's unusually
  many; confirm with the user they really want to manage every
  account before writing the stack file — multi-account rotation
  cost scales linearly with account count.

## What NOT to do

- Do not attempt to log in.
- Do not guess at the label list. Enumerate from the DOM; empty
  list is better than a fabricated default.
- Do not write `interceptly_profile_url` here — it lives in
  `interceptly-accounts.md` and requires a LinkedIn URL the client
  confirms.
- Do not enable every account the workspace exposes by default.
  The client picks. Silent over-management is a reputational risk.
