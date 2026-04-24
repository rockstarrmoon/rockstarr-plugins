---
name: switch-account
description: "This skill should be used when an Interceptly pass finishes on the current managed account and needs to move to the next one (rockstarr-reply's daily loop is the primary driver), or when the user says \"switch to a specific account\", \"move to the next Interceptly account\", or \"force-switch to account\". Opens the Interceptly SWITCH ACCOUNT popup via Chrome MCP, selects the target account, waits for the dashboard to re-render, then re-runs confirm-session-interceptly against the new account's expected profiles. Refuses to return success unless confirm-session-interceptly passes."
---

# switch-account

Cross-account hop. Paired with `confirm-session-interceptly` on
both sides — the caller confirms the current account beforehand
(so it knows the switch is leaving a known-good state), and this
skill confirms the new account after the switch before returning
success.

This skill is a shared contract point with `rockstarr-reply`,
which is the primary driver (its daily loop walks managed accounts
in rotation). This plugin calls it too when
`launch-campaign-interceptly` configures one account and needs to
hop to the next for an `all_accounts`-scoped campaign.

## When to run

- Between accounts in a daily rotation, after the caller finished
  its per-account work on the current account.
- On demand when the user says "switch to `<account_label>`" —
  e.g., they want to process a specific account out of rotation.
- After `launch-campaign-interceptly` configures one account and
  needs to configure another (for `all_accounts`-scoped
  campaigns).

## Preconditions

- The caller has a definite reason to move on from the current
  account (finished its pass, or the user said so).
- The caller must pass an explicit `target_account_label`.
- `/00_intake/interceptly-accounts.md` has a row for the target
  account with both `interceptly_profile_url` and
  `linkedin_profile_url`.

## Inputs

- `target_account_label` — which managed account to switch to
  (required). In the rockstarr-reply daily loop this is the next
  entry in `outreach_accounts[]`.

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
  retry against a different target or abort the pass.

### Step 4 — Tell the caller

Return a short summary:

> Switched to `<target_account_label>`. Session confirmed.

Or on failure:

> Failed to switch to `<target_account_label>`:
> `<confirm failure reason>`. Caller should abort or skip
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
  call. Wrong-account sending is the P0 risk in this pillar.
- Do not attempt to pre-confirm BEFORE the switch happens — that
  confirms the current account, not the target. The post-switch
  confirm is the one that matters.
- Do not retry a failed switch silently. If confirm-session fails
  after the switch, write the error and stop. The caller decides
  next steps.
- Do not switch mid-pass from this skill. A mid-pass switch is
  always a bug. The caller finishes its per-account work first,
  then calls switch-account.
