---
name: confirm-session
description: "This skill should be used as the first step of every daily outreach run, or whenever the user says \"confirm the LinkedIn session\", \"check we're logged in as the client\", or \"verify the Sales Nav session\". It opens LinkedIn in Cowork's Chrome session via Chrome MCP, reads the signed-in profile URL, and compares it to stack.md's linkedin_expected_profile_url. A mismatch or logged-out state aborts the entire daily loop: the failure is written loudly to _errors.md, surfaced in the daily summary, and no downstream skill runs. Wrong-account sending is the top reputational risk in this pillar — this check is non-negotiable."
---

# confirm-session

Safety gate. Runs first on every daily pass. If this skill fails, the
entire outreach loop aborts — no connects, no messages, no booking,
nothing.

## When to run

- First step of every scheduled daily run.
- Before any on-demand `force-send-today`, `send-approved-reply`,
  `book-meeting` run. Event-driven sends still require a session
  check within the last few minutes.
- Whenever the user asks to verify the session manually.

## Inputs

- `stack.md.linkedin_expected_profile_url` — the canonical URL for
  the client's LinkedIn profile (e.g.,
  `https://www.linkedin.com/in/jane-doe-client/`).

If that key is missing or empty, refuse and tell the user to run
`rockstarr-infra:capture-stack` to set it.

## Behavior

1. **Open LinkedIn.** Navigate Chrome MCP to
   `https://www.linkedin.com/feed/` or `https://www.linkedin.com/sales/`.
2. **Read identity.** Inspect the top-right "Me" element or the
   Sales Nav profile chip. Extract the signed-in user's profile URL.
   Normalize (strip query strings, lowercase).
3. **Compare.**
   - If logged-out (LinkedIn redirects to `/login`), result = `fail`
     with reason `logged_out`.
   - If URL does not match `linkedin_expected_profile_url`, result =
     `fail` with reason `wrong_account` and include the actual URL
     detected.
   - If URL matches, result = `pass`.
4. **Write to workbook.** Append a one-row status log to the
   Metrics (Daily) sheet's header area (or a `Session` sheet if one
   exists; if not, log to `_errors.md` only — do not proliferate
   sheets in V0.1). Daily log format in `_errors.md`:
   - Timestamp
   - Result
   - Reason (if fail)
   - Detected URL (if fail)
5. **On fail:**
   - Write a loud block to `/02_inputs/outreach/_errors.md`:
     ```
     ## <YYYY-MM-DD HH:MM> — confirm-session FAIL (<reason>)
     Expected: <expected URL>
     Detected: <detected URL or "(none — logged out)">
     Action: daily loop aborted. Log into the correct LinkedIn account
     in Cowork's browser, then re-run the daily loop or call
     force-send-today.
     ```
   - Return `fail` to the caller. The caller (the daily scheduler)
     must halt downstream steps.
   - Do **not** retry. Do **not** prompt for credentials. Do **not**
     click "Switch account."
6. **On pass:** return `pass` immediately. No logging noise on a
   clean pass; metrics rolls up the daily heartbeat at end-of-day.

## What NOT to do

- Do not attempt to log in. The client's credentials are not in scope.
- Do not "try again in five minutes." A fail is a fail.
- Do not downgrade a fail to a warning. The whole point is that wrong-
  account sending is unrecoverable — we don't have a second chance at
  reputation.
- Do not skip this check for "urgent" on-demand sends. Every send path
  requires a recent pass.
