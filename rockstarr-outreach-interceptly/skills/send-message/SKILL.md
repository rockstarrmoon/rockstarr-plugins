---
name: send-message
description: "This skill should be used when rockstarr-reply:present-for-approval unlocks with 'send it' (or a clear equivalent) and returns an authorized-send bundle, or when the user says \"send the approved reply for this lead\", \"ship the message\", or \"send the approved draft\". Pastes the approved body into Interceptly's composer via Chrome MCP using the React native-setter pattern, clicks Send (button match by text), logs the send to the Messages sheet, and returns success/failure. Refuses to run on drafts not in /04_approved/replies/."
---

# send-message

The only skill that actually posts a message to Interceptly.
Runs AFTER `rockstarr-reply:present-for-approval` has unlocked
the draft. Ships one message per call.

## When to run

- Immediately after `rockstarr-reply:present-for-approval`
  returns an `authorized-send` bundle with a `draft_path`.
- Never independent of the per-reply pipeline. If someone calls
  this skill with a draft that is not in `/04_approved/` with
  `awaiting_approval: false` and `authorized_at` set, refuse.

## Preconditions

- `confirm-session-interceptly` has passed for the currently-
  active account within this run. If not, refuse.
- Draft file exists at `/04_approved/outreach/replies/<thread>.md`
  with `awaiting_approval: false`, `authorized_at` set,
  `resolved_action: send`.
- Thread is open in Chrome (the per-reply pipeline navigated
  here before calling this skill).

## Inputs

- `draft_path` — absolute path to the approved reply file.

## Behavior

### Step 1 — Read the approved body

Open the draft file. Extract the reply body (including
signature) from the `## Body` section OR the option-specific
section (for non-ICP three-option picks, the front matter
records which option was picked; use that section).

### Step 2 — Focus the composer

Click Interceptly's composer textarea for the current thread.
Wait for the cursor to be visible in the input.

### Step 3 — Paste with the React native-setter pattern

Direct `.value = ...` does NOT fire React's `onChange`, so
Interceptly's send button stays disabled. Use the preserved
pattern:

```js
const nativeSetter = Object.getOwnPropertyDescriptor(
  window.HTMLTextAreaElement.prototype, 'value'
).set;
nativeSetter.call(composerElement, body);
composerElement.dispatchEvent(new Event('input', { bubbles: true }));
```

Via Chrome MCP's `javascript_tool`. After dispatch, wait for the
Send button to become enabled (state visible in the DOM).

### Step 4 — Click Send

Match the Send button by visible text ("Send" or the localized
equivalent Interceptly is shipping). Do NOT rely on a positional
selector; the button re-orders across UI revisions.

Known quirk: if Enter-to-send is enabled, avoid pressing Enter
from the composer — a stray newline in the body becomes a
premature send. Always use the button click.

### Step 5 — Confirm send

Wait for Interceptly's UI state to change: the composer clears,
the new message appears in the thread body as an outbound entry,
and / or a transient "Sent" indicator flashes. Any of these
signals confirms success.

If none of these signals appears within 6 seconds, retry once —
clear the composer, re-paste, re-click. On second failure,
abort: write a loud _errors.md block, leave the draft in
`/04_approved/` with `send_attempted: true,
send_failed_reason: <reason>`, do NOT apply the label, do NOT
create the follow-up task. The approver can retry manually.

### Step 6 — Log to the Messages sheet

Append to the `Messages` sheet of `outreach-mirror.xlsx`:

| column | value |
|---|---|
| ts | `<ISO>` |
| lead_url | `<URL>` |
| thread_id | `<Interceptly thread id>` |
| campaign_slug | `<slug or blank>` |
| account_label | `<active managed account>` |
| signer_persona | `<same>` |
| verdict | `<target / not_target>` |
| bucket | `<hot / warm / cold / skeptical>` |
| pattern | `<hot / warm_icp / cold_bump / ...>` |
| body_snippet | `<first 80 chars of the send body>` |
| draft_path | `/04_approved/outreach/replies/<thread>.md` |

### Step 7 — Return

Return `{sent: true, ts: <ISO>, thread_id, lead_url}` to the
caller. The caller (the per-reply pipeline) then calls
`apply-label` and `create-followup-task` in that order.

## Failure modes

- **Composer does not accept the paste.** React native-setter
  didn't fire onChange. Retry with a fresh selector lookup. On
  second failure, abort. Do not retry a third time — you risk
  producing a duplicate if the first paste actually succeeded.
- **Send button never enables.** Composer probably has hidden
  validation (max-length, empty body, unrecognized characters).
  Inspect the DOM for an error message. Write to _errors.md;
  abort.
- **Thread navigates away mid-send.** Usually the Labels nav bug
  (known issue: clicking Labels filter navigates to Campaigns).
  Abort; the approver retries manually.
- **Interceptly reports rate limit.** Write to _errors.md; abort
  this send and every subsequent send in the run. Over-sending
  past a rate limit is an account-health risk.

## Idempotency

- If the Messages sheet already has a row with the same `ts`
  (within 30 seconds) AND the same `thread_id`, treat as a
  duplicate and refuse a second send. A re-run of a draft
  requires a fresh authorization cycle.

## What NOT to do

- Do not send from a draft in `/03_drafts/`. Only
  `/04_approved/`.
- Do not press Enter to send. Always click the Send button.
- Do not edit the body before sending. The approved file is
  the authoritative body. If the approver wanted edits, they
  would have given editing instructions in `present-for-
  approval` and received a re-presented draft.
- Do not continue to `apply-label` or `create-followup-task` if
  the send failed. Both are downstream of a successful send.
- Do not retry a failed send more than once. A second failure
  means human intervention is required.
