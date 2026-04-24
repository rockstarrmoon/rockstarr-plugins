---
name: process-inbox
description: "This skill should be used in the daily outreach loop after preview-queue, or when the user says \"process the inbox\", \"walk unread replies\", or \"handle today's Interceptly inbox for the current account\". Filters Interceptly Inbox → Replied, walks unread conversations oldest-first, and for each thread runs qualify-lead locally, then hands the thread off to rockstarr-reply via a channel-agnostic handoff bundle. On the bundle returned by rockstarr-reply (authorized-send / three-option-choice / flag / book-meeting-handoff / no-action), this skill executes the Interceptly-side work: send-message → apply-label → create-followup-task. Unreads come before My Tasks on every account. Refuses to run if confirm-session-interceptly has not passed in this run."
---

# process-inbox

The first half of the per-account pass. Every unread reply runs
through the full pipeline before `process-my-tasks` takes over
for this account.

rockstarr-outreach-interceptly owns channel I/O. rockstarr-reply
owns drafting. This skill is the seam between the two. It:

1. Navigates Interceptly and scrapes the per-thread context.
2. Runs `qualify-lead` locally to get an `icp_verdict`.
3. Builds a handoff bundle and calls rockstarr-reply.
4. Executes the channel-side work returned by rockstarr-reply.

## When to run

- Step 3a of the daily loop, for the currently-active managed
  account, after `confirm-session-interceptly` passes and
  `preview-queue` (if enabled) has written its file.
- On demand when the user says "just process the inbox" or "run
  the inbox pass for this account".

## Preconditions

- `confirm-session-interceptly` has passed in this run for the
  currently-active account. If the last confirm result was `fail`,
  REFUSE — the whole pass is aborted.
- `/00_intake/interceptly-accounts.md` has a row for the current
  account (with `linkedin_profile_url`, `signature`, `persona_notes`).
- `/00_intake/icp-qualifications.md` exists.
- `/00_intake/style-guide.md` exists and has an approved voice.
- `rockstarr-reply` is installed. If not, refuse with a pointer —
  drafting is not a responsibility of this plugin.

## Behavior

### Step 1 — Navigate to inbox

Navigate Chrome MCP to Interceptly → Inbox. Apply the filter
`Replied` (leads who have replied to any touch). Sort by unread
state ascending, then by date ascending — unread + oldest first.

### Step 2 — Walk unreads oldest-first

For each unread thread:

1. **Open the thread.** Click the conversation row. Wait for the
   right-panel to render (lead profile + thread body).
2. **Read the right-panel context.** Lead name, company, title,
   Work Experience. If Work Experience is collapsed, scroll to
   expand it. This feeds `qualify-lead`.
3. **Read the thread body.** Capture every inbound and outbound
   message since the last thread open (the new-since-read portion
   is the focus; older context is useful for classification).

### Step 3 — Qualify locally

Call `qualify-lead` with the right-panel context and any active
campaign's per-campaign ICP overrides. Record the verdict:
`target` / `not_target` / `ambiguous` / `unknown`. The verdict is
what we will pass to rockstarr-reply in the handoff.

### Step 4 — Build the handoff bundle

Compose the channel-agnostic bundle for rockstarr-reply:

```
{
  channel: "linkedin",
  thread: {
    thread_id: "<Interceptly thread id>",
    messages: [
      { direction: "outbound|inbound", ts: "<ISO>", body: "..." },
      ...
    ],
    thread_open_path: "<Interceptly URL>"
  },
  persona: {
    account_label: "<active managed account>",
    signature: "<signature block from interceptly-accounts.md>",
    persona_notes: "<persona_notes from interceptly-accounts.md>"
  },
  lead: {
    url: "<lead LinkedIn URL>",
    name: "<right-panel name>",
    company: "<...>",
    title: "<...>",
    campaign_slug: "<slug or blank>"
  },
  icp_verdict: "target | not_target | ambiguous | unknown",
  icp_matching_rule: "<verbatim rule text from qualify-lead>",
  intent_hint: null
}
```

### Step 5 — Call rockstarr-reply

Hand the bundle to `rockstarr-reply`. The internal order is:
`rockstarr-reply:classify-reply` →
`rockstarr-reply:draft-reply` →
`rockstarr-reply:present-for-approval`. rockstarr-reply owns that
internal sequence; this skill only sees the final bundle it
returns.

Possible return bundles:

- **`authorized-send`** — operator said "send it". Contains
  `draft_path`, `proposed_label`, `proposed_followup_timer`
  keyword. Continue to Step 6a.
- **`three-option-choice`** — non-ICP warm case; operator picked A
  (graceful exit) or C (throwaway question) and wants it sent.
  Shape is the same as `authorized-send` with the picked option
  resolved. Continue to Step 6a.
  - Variant: operator picked B (let-it-hang) — return shape is
    `action: label-only`, no send. Continue to Step 6b.
- **`flag`** — rockstarr-reply refused to draft or the operator
  asked to flag. Contains `reason` + `note`. rockstarr-reply has
  already written `/02_inputs/replies/_flags.md`. Continue to
  Step 6c.
- **`book-meeting-handoff`** — the lead agreed to a specific time
  AND supplied all booking-link-required fields. Contains
  `agreed_start_iso` + `lead_fields`. Continue to Step 6d.
- **`no-action`** — out-of-office, already-booked, or a
  polite-ack with no signal. Contains `proposed_label`. Continue
  to Step 6b.

### Step 6 — Execute the channel-side work

#### 6a — authorized-send

1. `send-message` with `draft_path`. On failure, abort this
   thread (do NOT label, do NOT create task). Move to next
   unread.
2. On send success, `apply-label` with `proposed_label`.
3. `create-followup-task` with `proposed_followup_timer` keyword.
   This skill converts keyword → due date using
   `stack.md.followup_timers` + business-days math.

#### 6b — label-only (let-it-hang, no-action)

1. `apply-label` with `proposed_label`. No send. No task.

#### 6c — flag

1. rockstarr-reply already wrote `_flags.md`. This skill
   applies the `Follow Up` label (from
   `stack.md.label_mapping.flagged_review` override, if any) via
   `apply-label`.
2. `create-followup-task` with a 2-business-day review timer
   (overridable via `stack.md.followup_timers.flagged_review`).

#### 6d — book-meeting-handoff

1. Route directly to `book-meeting` with `agreed_start_iso` +
   `lead_fields`. `book-meeting` handles slot selection, form
   fill, submit, and on success calls `mark-booked`. This branch
   does NOT send a reply message — the booking itself is the
   close.

### Step 7 — Move to next thread

Do NOT navigate away from the inbox until all unreads are
processed. Interceptly sometimes re-renders the list after a
label click — re-read the list and continue at the next unread
that is NOT the one just processed.

### Step 8 — Handoff

When the unread count reaches zero (read from Interceptly, not
from the mirror), return control to the daily loop. The loop
then runs `process-my-tasks` against this same account BEFORE
switching to the next account.

## Constraints

- Unreads ALWAYS come before My Tasks on any given account. Do
  not interleave.
- Never move to another account while this account has ANY
  unread in the Replied filter.
- The send gate is strict and lives inside
  `rockstarr-reply:present-for-approval`. This skill never
  interprets a response as authorization — it only executes
  the `authorized-send` bundle if rockstarr-reply returns one.

## Failure modes

- **Chrome MCP drops mid-thread.** Retry the current thread
  once. On second failure, leave the thread unread, write an
  `_errors.md` block, and continue to the next thread.
- **A thread has no visible right-panel context.** Open the
  lead's LinkedIn profile in a read-only side tab via Chrome
  MCP to gather context for `qualify-lead`. Do not send from
  that tab.
- **rockstarr-reply returns an unrecognized bundle shape.**
  Abort this thread, write `_errors.md` with the raw bundle,
  move on.
- **`send-message` fails** after rockstarr-reply returned
  `authorized-send`. Retry once. On second failure, leave the
  thread unread, write `_errors.md`, and move on. Do NOT label
  or create a task if the send failed.
- **`apply-label` fails** after a successful send. Retry once.
  On failure, write `_errors.md` with the specific thread — the
  operator needs to apply the label manually. The task still
  gets created (follow-up timer is independent of label state).
- **The list re-renders into Campaigns view** (known Labels nav
  bug). Navigate back to Inbox → Replied filter and continue.

## What NOT to do

- Do not process My Tasks until this skill finishes.
- Do not switch accounts until this skill and
  `process-my-tasks` both finish.
- Do not draft a reply in this plugin. rockstarr-reply owns
  drafting. If rockstarr-reply is not installed, refuse — don't
  improvise.
- Do not guess approval. Silence is not consent. The send gate
  is `rockstarr-reply:present-for-approval` — if the bundle
  returned is not `authorized-send`, nothing goes out.
- Do not skip `qualify-lead`. The bundle's `icp_verdict` is what
  rockstarr-reply's classify-reply trusts — don't ship the bundle
  without it.
- Do not paste the booking link into any reply. The booking link
  is a destination, not content. (rockstarr-reply enforces this
  in draft-reply; this skill enforces it at send time by refusing
  to paste a body that contains `stack.md.booking_link_url`.)
