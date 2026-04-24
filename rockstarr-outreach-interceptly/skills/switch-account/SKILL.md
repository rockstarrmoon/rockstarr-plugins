---
name: switch-account
description: "This skill should be used when the daily loop finishes process-inbox + process-my-tasks on the current managed account and needs to move to the next one, or when the user says \"switch to <account>\", \"move to the next Interceptly account\", or \"force-switch to account\". Opens the Interceptly SWITCH ACCOUNT popup via Chrome MCP, selects the target account, waits for the dashboard to re-render, then re-runs confirm-session-interceptly against the new account's expected profiles. Refuses to return success unless confirm-session-interceptly passes."
---

# switch-account

Cross-account hop. Paired with `confirm-session-interceptly` on
both sides — it runs BEFORE the switch to confirm the current
account was the expected one, and AFTER the switch to confirm the
new account is the expected next one.

## When to run

- Between accounts in the daily loop rotation, after
  `process-inbox` AND `process-my-tasks` have finished on the
  current account with zero remaining items.
- On demand when the user says "switch to `<account_label>`" —
  e.g., they want to process a specific account out of rotation.
- After `launch-campaign-interceptly` configures one account and
  needs to configure another (for `all_accounts`-scoped
  campaigns).

## Preconditions

- For scheduled daily-loop use: `process-inbox` and
  `process-my-tasks` returned zero-remaining on the current
  account.
- For on-demand use: the caller must pass an explicit
  `target_account_label`.
- `/00_intake/interceptly-accounts.md` has a row for the target
  account with both `interceptly_profile_url` and
  `linkedin_profile_url`.

## Inputs

- `target_account_label` — which managed account to switch to
  (required). The daily loop passes the next entry in
  `outreach_accounts[]`.

## Behavior

### Step 1 — Open the SWITCH ACCOUNT popup

From anywhere inside `https://dash.interceptly.ai/`, click the
account chip at the top of the left sidebar. The SWITCH ACCOUNT
popup opens (same popup `discover-interceptly-accounts` reads).

### Step 2 — Select the target

Click the row whose label matches `target_account_label`. Wait
for Interceptly to re-render the dashboard. Typical wait: 2-4
seconds. On a 6+ second wait with no visible change, retry the
click once.

### Step 3 — Re-confirm session

Call `confirm-session-interceptly` with
`expected_account_label = target_account_label`. This is the
non-negotiable check.

- If confirm returns `pass`, return success.
- If confirm returns `fail`, return `fail` to the caller and
  write `_errors.md` per the confirm skill. Do NOT attempt
  another switch automatically — the caller decides whether to
  retry against a different target or abort the daily loop.

### Step 4 — Tell the caller

Return a short summary:

> Switched to `<target_account_label>`. Session confirmed.

Or on failure:

> Failed to switch to `<target_account_label>`:
> `<confirm failure reason>`. Daily loop should abort or skip
> this account.

## Failure modes

- **Target not in the popup.** The client removed the account
  from Interceptly but it's still in `outreach_accounts[]`.
  Abort this account, write _errors.md, and recommend
  re-running `discover-interceptly-accounts`. Do NOT remove the
  row from the stack file automatically.
- **Popup layout changed.** Interceptly has shipped UI changes
  before. Fail loud to _errors.md; the operator reassigns the
  mapping manually.
- **Session confirmation fails after switch.** That usually
  means the LinkedIn profile URL stored for the target account
  doesn't match what's actually signed in — a
  `capture-interceptly-personas` refresh is needed.

## What NOT to do

- Do not switch without a post-switch `confirm-session-interceptly`
  call. Wrong-account sending is the P0 risk in this plugin.
- Do not attempt to pre-confirm BEFORE the switch happens — that
  confirms the current account, not the target. The post-switch
  confirm is the one that matters.
- Do not retry a failed switch silently. If confirm-session fails
  after the switch, write the error and stop. The caller decides
  next steps.
- Do not switch during `process-inbox` or `process-my-tasks`. A
  mid-pass switch is always a bug.
