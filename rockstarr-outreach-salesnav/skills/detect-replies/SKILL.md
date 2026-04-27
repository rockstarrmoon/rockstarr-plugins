---
name: detect-replies
description: "This skill should be used in the daily outreach loop after send-scheduled-messages, optionally again as an afternoon pass, or when the user says \"check for new replies\", \"scan the Sales Nav inbox\", or \"pull today's inbound messages\". It reads inbound messages from Sales Navigator and LinkedIn via Chrome MCP, matches each to a Leads row, appends to the Replies sheet, cancels every pending message-step-N and follow-up task for that lead, creates a review-reply task, drives rockstarr-reply synchronously to stage a draft per inbound, and at end-of-run fires rockstarr-infra:notify-reply-ready as a single urgent client email summarizing every newly-staged draft. Booking detection happens via mark-booked, not here."
---

# detect-replies

The hand-off point between outreach and reply. Every inbound message
from a lead in a live campaign comes through here, every one of them
routes to `rockstarr-reply` for client-gated drafting, and at the
end of the run a single urgent email summarizes whichever drafts got
staged on this pass.

## When to run

- Daily loop, after `send-scheduled-messages`, before `daily-connect`.
- Optional afternoon pass — this skill may re-run later in the day
  to pick up afternoon replies. The afternoon pass does NOT re-run
  the connect loop. Each afternoon pass fires its own
  `notify-reply-ready` for whatever it newly staged.
- On-demand when the user asks "did anyone reply?"

## Preconditions

- `confirm-session` passed in this run.
- `rockstarr-infra >= 0.8.0` is installed. The `notify-reply-ready`
  skill required by Step 5 ships in 0.8.0. If unavailable, this skill
  still stages drafts but skips the urgent email and records an
  `infra_too_old` entry in `notify_skipped_reasons`.
- `rockstarr-reply >= 0.2.0` is installed. 0.2.0 is the build that
  writes the v0.8.0 front-matter contract `notify-reply-ready` reads
  (`channel: "reply"` + `source_channel`, `inbound_excerpt`,
  `draft_body`, `path_relative`, optional `draft_options[]`). If
  rockstarr-reply is missing, this skill still creates the
  `review-reply` task and writes the Cowork notification fallback per
  Step 4d but does not stage a draft, does not fire the urgent email.
- The mailer Worker is at version `0.2.1` or later (claude:// URL
  scheme support). Mailer version is enforced by `notify-reply-ready`
  itself, not here — surfaced via `notify_skipped_reasons` if a send
  errors.

## Inputs

- Sales Nav messaging inbox + LinkedIn inbox (via Chrome MCP).
- Leads sheet (to match inbound threads to known leads).
- Tasks sheet (to find pending tasks to cancel).
- The last time this skill ran today (from a `last_detect_replies_at`
  marker in the workbook's metadata area, or derived from the most
  recent Replies row).

## Behavior

1. **Open the Sales Nav messaging inbox** via Chrome MCP. If Sales
   Nav inbox is not available to the client, fall back to the
   LinkedIn messaging inbox at `https://www.linkedin.com/messaging/`.
2. **Scan unread threads + threads with a new inbound since
   last-run.**
3. **For each thread with a new inbound message from the lead:**
   a. **Identify the lead.** Use the thread's counterparty profile
      URL. Match against Leads (primary key `lead_url`).
      - No match → log as "personal inbox message, not a campaign
        lead" and skip.
      - Multiple matches (shouldn't happen — leads are unique
        across active campaigns) → log to `_errors.md` and skip.
   b. **Capture the reply.** Append a row to Replies:
      - `date` = inbound timestamp (ISO)
      - `lead_url`
      - `campaign_slug`
      - `raw_text` = the full inbound message text
      - `classification` = `unclassified` for now; the downstream
        reply skill will classify during draft.
        (Classifications used later: `positive`, `ask-for-info`,
        `not-interested`, `out-of-office`, `booked`. `booked` is set
        by `mark-booked`, not here.)
      - `handoff_state` = `handed_to_reply`
   c. **Cancel pending sequence tasks.** For this lead, find every
      Tasks row where `campaign_slug` matches, `status = pending`,
      and `type IN (message-step-1, message-step-2, message-step-3,
      follow-up, book-meeting)`. Flip `status = cancelled`,
      `completed_at = now`, `cancel_reason = reply_received`.
      Leave `connect` tasks alone (the lead's already connected if
      we're getting replies).
   d. **Merge into an existing review-reply, or create a new one.**
      - If a `review-reply` task already exists for this lead that
        is `status = pending`: leave it alone. The reply append
        above will be picked up when the client opens the existing
        thread in `rockstarr-reply`.
      - Else create a new Tasks row: `type = review-reply`,
        `status = pending`, `due_date = today`, metadata pointing to
        the Replies row id.
   e. **Flip Leads.state.** If `state` is currently `connected` or
      `accepted`, move to `replied`, set `date_replied = today`.
      If `state = booked`, leave it alone (a post-booking message
      is still useful context but doesn't reset the state).
4. **Stage a draft via rockstarr-reply for every new review-reply
   task in this run.** For each inbound that produced a *new*
   `review-reply` task in Step 3d (not the merged-into-existing case
   — those already have a draft from the earlier pass), do, in order:

   a. **Build the channel-agnostic handoff bundle.** Same shape the
      Interceptly variant's `process-inbox` uses, with `channel`
      hard-coded to `"linkedin-salesnav"` so rockstarr-reply applies
      the LinkedIn channel adaptation:

      ```
      {
        channel: "linkedin-salesnav",
        thread: {
          thread_id: "<Sales Nav conversation id, or lead_url slug>",
          messages: [
            { direction: "outbound|inbound", ts: "<ISO>", body: "..." },
            ...
          ],
          thread_open_path: "<Sales Nav messaging URL for this thread>"
        },
        persona: {
          account_label: "<linkedin_expected_profile_url from stack.md>",
          signature: "",
          persona_notes: ""
        },
        lead: {
          url: "<lead_url>",
          name: "<Leads.name>",
          company: "<Leads.company>",
          title: "<Leads.title>",
          campaign_slug: "<slug>"
        },
        icp_verdict: "unknown",
        icp_matching_rule: "",
        intent_hint: null
      }
      ```

      Salesnav 0.1.6 does NOT run a `qualify-lead` pass and does NOT
      maintain `00_intake/icp-qualifications.md` — that capability is
      tracked for the 0.2.0 ICP-qualifications track. Until then,
      every bundle goes out with `icp_verdict: "unknown"`.
      rockstarr-reply still drafts; it just doesn't get a per-reply
      verdict to specialize on.

   b. **Invoke `rockstarr-reply:classify-reply` then
      `rockstarr-reply:draft-reply` synchronously, in that order.**
      Do NOT invoke `rockstarr-reply:present-for-approval` — approval
      is the operator's later move via the deep-link in the urgent
      email. The daily-loop boundary is "draft staged," not
      "draft approved."

   c. **Capture the result.** rockstarr-reply returns one of five
      shapes (per the v0.2.0 handoff contract):

      - **Staged body** (`{draft_path, pattern, proposed_label}`) —
        most common. Append `draft_path` to `staged_paths[]`.
      - **Three-option staged bundle** (`{draft_path,
        three_options: true, ...}`) — Warm-non-ICP polite-yes case.
        Append `draft_path` to `staged_paths[]`. notify-reply-ready
        renders all three options inline from `draft_options[]` in
        the front-matter.
      - **`book-meeting-handoff`** (`{slot, lead_fields}`) — lead
        already agreed and supplied required fields. Do NOT append
        to `staged_paths[]`. Create a `book-meeting` task in the
        workbook (type `book-meeting`, due `today`, payload =
        slot + lead_fields) and continue. `book-meeting` runs in
        its own event-driven path; the urgent email is not the
        right surface for it.
      - **`no-action`** (`{reason}`) — sub_types like
        `out_of_office`, `already_booked`, or
        `not-target × Cold/Skeptical`. Do NOT append. Log the
        reason against the Replies row as
        `handoff_state = no_action`.
      - **`flag`** (`{reason, note}`) — rockstarr-reply hit an
        unsafe combo and wrote to `_flags.md` instead of staging.
        Do NOT append. The flag itself is the operator surface;
        leaving it out of the urgent email is intentional —
        flags get resolved separately, not via reply approval.

   d. **rockstarr-reply not installed.** Skip steps 4a–4c entirely.
      Write a Cowork notification: "Reply from `<name>` in
      `<campaign_slug>` needs your attention — rockstarr-reply is
      not installed. Draft manually via Sales Nav." Log the
      fallback to `_errors.md`. The `review-reply` task remains
      pending in the workbook so it surfaces in the next daily
      digest.

   e. **rockstarr-reply errored on a single thread.** Append a row
      to `_errors.md` with the lead, campaign, and error. Drop the
      path from `staged_paths[]` if it was added. Continue to the
      next thread — do NOT abort the whole loop.

   At the end of Step 4, `staged_paths[]` is the list of *paths
   whose drafts were staged this run by rockstarr-reply with the
   v0.8.0 front-matter contract*. It is the input to Step 5.

5. **Fire the urgent reply-ready notification.** If
   `staged_paths.length > 0`:

   a. **Validate front-matter on each path** before handing the list
      off. For each path, read the YAML front-matter. Required fields
      (per the `notify-reply-ready` contract): `channel == "reply"`,
      `source_channel`, `lead_name`, `bucket`, `proposed_label`,
      `proposed_followup_timer`, `inbound_excerpt`, `draft_body`,
      `path_relative`. If any required field is missing, drop the
      path from the list, append a row to `_errors.md`, and add an
      entry to `notify_skipped_reasons[]` with
      `{path, reason: "front_matter_incomplete", missing: [...]}`.
      This catches the rockstarr-reply-version-skew failure mode early.

   b. **Cap at the soft limit.** notify-reply-ready renders a soft
      cap of 8 cards per email. Do NOT pre-truncate `staged_paths[]`
      — pass the full list. notify-reply-ready does the truncation
      and writes the "+ N additional" line itself.

   c. **Invoke `rockstarr-infra:notify-reply-ready`** with
      `{ staged_paths }`. Capture:

      - On success: `notify_message_id` = the returned Resend
        message id.
      - On error: `notify_message_id = null`. Append a row to
        `_errors.md` with the error response. Add an entry to
        `notify_skipped_reasons[]` with
        `{reason: "notify_send_failed", detail: "<error>"}`. Do NOT
        retry from this skill — `send-notification` already handles
        the one-retry on transient hiccups internally.

   d. **The Tasks rows are already in place.** A failed urgent send
      is not a data-integrity event. The next morning's
      `approvals-digest` will pick the still-pending review-reply
      tasks up. The urgent email is an *acceleration* of the
      digest, not the digest's authoritative replacement.

   If `staged_paths.length == 0` (no new drafts staged this pass —
   either no inbounds, all merged into existing review-reply
   threads, or every result was book-meeting-handoff / no-action /
   flag), skip Step 5 entirely. Set `notify_message_id = null` and
   leave `notify_skipped_reasons[]` as the running list (which may
   already contain `infra_too_old`, `front_matter_incomplete`,
   etc.). The skip is the feature — the email only fires when there
   is something genuinely new to escalate.

6. **Update the `last_detect_replies_at` marker** so the afternoon
   pass (if enabled) only picks up newer inbounds.
7. **Save the workbook.**

## Output

- `new_replies` — count of inbounds captured this run.
- `threads_handed_off` — count of `review-reply` tasks newly created
  this run.
- `already_in_review` — count of inbounds that merged into an
  existing review-reply thread (no new draft staged for these — the
  prior pass already staged one).
- `non_campaign_inbound` — count (logged, not acted on).
- `drafts_staged` — count of paths added to `staged_paths[]` from
  rockstarr-reply (excludes `book-meeting-handoff`, `no-action`,
  and `flag` results). This is the count of cards the urgent email
  was built from.
- `book_meeting_handoffs` — count of inbounds that produced a
  `book-meeting` task (rockstarr-reply returned the slot + fields).
  These do not appear in the urgent email.
- `notify_message_id` — Resend message id returned by
  `notify-reply-ready` on success, or `null` if Step 5 was skipped
  or errored.
- `notify_skipped_reasons[]` — list of `{path?, reason, ...}`
  entries explaining anything that didn't make it into the email.
  Possible reasons: `infra_too_old`, `reply_plugin_missing`,
  `front_matter_incomplete`, `rockstarr_reply_errored`,
  `notify_send_failed`. Populated even when the email itself sent
  cleanly — drops can happen mid-batch.

## What NOT to do

- Do not draft the reply in this skill. Drafting belongs to
  `rockstarr-reply:draft-reply`. This skill only detects, routes,
  drives the staging call, and notifies.
- Do not invoke `rockstarr-reply:present-for-approval` from inside
  this skill. The daily-loop boundary is *draft staged, email
  sent* — approval happens later in Cowork via the deep-link.
- Do not classify as `booked` here. `mark-booked` is the single
  source of truth for booking state.
- Do not send any message from this skill. Not ever. The
  `send-approved-reply` skill ships approved replies; this one
  only stages drafts and notifies.
- Do not cancel `connect` tasks — a replied lead was already
  connected; the connect task is done, not pending.
- Do not mark `review-reply` as done. That happens when
  `rockstarr-reply:approve` fires downstream after the operator
  authorizes the send.
- Do not pre-truncate `staged_paths[]` to fit notify-reply-ready's
  8-card soft cap. Pass the full list — notify-reply-ready handles
  the truncation and the "+ N additional" line on its own.
- Do not retry a failed `notify-reply-ready` call from this skill.
  `send-notification` already retries once on transient errors
  internally. A second failure is real; surface it via
  `notify_skipped_reasons` and let the next morning's
  `approvals-digest` cover the gap.
- Do not call `notify-reply-ready` when `staged_paths` is empty.
  The skip is the feature.
- Do not call `notify-reply-ready` with `notify_type=default`. The
  contract is urgent — the helper resolves the recipient via the
  urgent rules. Passing `default` would silently route to the
  wrong inbox if `ROCKSTARR_NOTIFY_URGENT_TO` is set.
