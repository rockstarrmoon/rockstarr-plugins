---
name: confirm-session-interceptly
description: "This skill should be used as the first step of every Interceptly pass (outreach campaign configuration, rockstarr-reply's daily inbox + tasks loop), again after every switch-account, or when the user says \"confirm the Interceptly session\", \"check we're signed in as the right account\", or \"verify the Interceptly session\". Opens https://dash.interceptly.ai/ via Chrome MCP, reads the active account in the left sidebar, visits the configured interceptly_profile_url + linkedin_profile_url for the expected account, and verifies both match. Mismatch or logged-out aborts the entire pass for that account with a loud _errors.md entry. Wrong-account sending is the top reputational risk in this variant — this check is non-negotiable."
---

# confirm-session-interceptly

Safety gate. Runs at the start of every Interceptly pass — whether
this plugin is configuring a campaign or `rockstarr-reply` is
walking the inbox — and again after every account switch. If this
skill fails, the caller aborts. No UI actions, no sends, no labels,
no task writes.

This skill is an explicit contract point between
`rockstarr-outreach-interceptly` and `rockstarr-reply`. Both
plugins call it. The Session sheet of `outreach-mirror.xlsx` holds
its heartbeat rows.

## When to run

- Step 1 of any Interceptly UI pass, BEFORE any inbox / tasks /
  campaign work.
- Immediately after every `switch-account` to the next managed
  account.
- Before any on-demand Chrome-MCP skill that acts on Interceptly
  (stop-campaign, launch-campaign, or any reply-pipeline skill in
  rockstarr-reply) that isn't already part of a run that just
  passed confirm.
- Whenever the user asks to verify the session manually.

## Inputs

- `expected_account_label` — the managed account the caller
  expects to see active (e.g., the current position in
  `outreach_accounts[]`).
- `00_intake/interceptly-accounts.md` — row for
  `expected_account_label` with `interceptly_profile_url` and
  `linkedin_profile_url`.

If the row doesn't exist, refuse and point at
`capture-interceptly-personas`. If `linkedin_profile_url` is
blank, refuse with the same pointer — missing LinkedIn URL means
we cannot check wrong-account sending.

## Behavior

### Step 1 — Open Interceptly

Navigate Chrome MCP to `https://dash.interceptly.ai/`. Wait for
the sidebar to render.

- If the page redirects to `/login`, result = `fail`, reason =
  `logged_out`.

### Step 2 — Read the active account

Inspect the account chip at the top of the left sidebar. Extract
the displayed label.

- If the label cannot be read from the DOM after a 2s wait, retry
  once. On second failure, result = `fail`, reason =
  `dom_unreadable`.
- If the label does NOT match `expected_account_label`, result =
  `fail`, reason = `wrong_account`.

### Step 3 — Cross-check LinkedIn

In a new tab, navigate to `https://www.linkedin.com/feed/` (or
`https://www.linkedin.com/sales/` if Sales Nav is the habitual
landing spot). Read the signed-in LinkedIn profile URL from the
top-right "Me" chip. Normalize (strip trailing slash, query
strings, lowercase).

- If LinkedIn is logged out, result = `fail`, reason =
  `linkedin_logged_out`.
- If the URL does NOT match the `linkedin_profile_url` recorded
  for `expected_account_label`, result = `fail`, reason =
  `linkedin_wrong_account` and include the detected URL.
- If match, continue.

The LinkedIn cross-check is the core of this skill — Interceptly
drives LinkedIn via the browser session, so if LinkedIn is the
wrong account, sends will go out as the wrong person even though
Interceptly's sidebar says the right account label.

### Step 4 — Write a heartbeat row

On pass, append one row to the `Session` sheet of
`outreach-mirror.xlsx` (create the sheet if it doesn't exist):

| column | value |
|---|---|
| ts | `<ISO>` |
| account_label | `<expected_account_label>` |
| caller | `<plugin or skill that invoked confirm, e.g., 'rockstarr-outreach-interceptly:process-inbox'>` |
| result | `pass` |
| reason | (blank) |

Quiet on pass — no chat noise. The heartbeat lets weekly reports
see how often confirm failed, and lets both plugins share one
session-health record.

### Step 5 — On fail

Write a loud block to `/02_inputs/outreach/_errors.md`:

```
## <ISO timestamp> — confirm-session-interceptly FAIL (<reason>)

Caller: <caller>
Expected account: <expected_account_label>
Expected Interceptly URL: <interceptly_profile_url>
Expected LinkedIn URL: <linkedin_profile_url>
Detected: <detected label / URL, or "(logged out)">

Action: caller aborted for this account. Sign into the correct
LinkedIn and Interceptly in Cowork's browser, then re-run.
```

Also write the heartbeat row with `result = fail` and the reason.

Return `fail` to the caller. The caller MUST halt all downstream
steps for this account. Do NOT proceed to other managed accounts
from a failed state — the `switch-account` flow owns that
decision, and it re-runs confirm.

## What NOT to do

- Do not attempt to log in. Credentials are not in scope.
- Do not "try again in a few minutes." A fail is a fail.
- Do not downgrade a wrong-account fail to a warning. The whole
  point is that wrong-account sending is unrecoverable.
- Do not skip this check for "urgent" on-demand sends. Every send
  path requires a recent pass against the correct account.
- Do not click Switch Account here — that's what `switch-account`
  is for, and this skill runs BEFORE and AFTER it, not during.

## Why this is non-negotiable

Multi-account rotation is the most boring and most important part
of the Interceptly pillar. Walking managed accounts in order with
`confirm-session-interceptly` between each is what keeps the wrong
persona from replying on the wrong thread. If this check ever
silently passes on the wrong account, it is a P0 for both this
plugin and `rockstarr-reply`.
