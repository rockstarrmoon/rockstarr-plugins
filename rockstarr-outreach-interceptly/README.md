# rockstarr-outreach-interceptly

Rockstarr AI LinkedIn outreach for clients who run **Interceptly**.
This plugin owns the Interceptly integration end-to-end: account
discovery, per-account persona capture, campaign lifecycle (draft →
launch → pause / stop), multi-account session confirmation, the
daily inbox + tasks loop, per-reply channel I/O (send, label,
follow-up task, meeting proposal, booking, booked-state single
source of truth), and outreach-side metrics + weekly reporting.

Reply **drafting** lives in the sister plugin
[`rockstarr-reply`](../rockstarr-reply): classification, draft
composition, operator approval gate, follow-up-timer keyword
derivation, and flag-for-review. This plugin navigates Interceptly,
scrapes the per-thread context, runs ICP qualification locally,
calls `rockstarr-reply` with a channel-agnostic handoff bundle, then
executes the channel-side work returned by the bundle.

Unlike `rockstarr-outreach-salesnav`, where the local workbook is
the single state-of-truth, **Interceptly itself is source-of-truth
here.** `/rockstarr-ai/02_inputs/outreach/outreach-mirror.xlsx` is an
audit mirror only — if Interceptly and the mirror disagree,
Interceptly wins and the pipeline reconciles on the next pass.

The plugin is generic-for-any-client. No hardcoded persona, voice,
or ICP. Everything the bot needs is captured at install through
four interviews and stored in the client folder.

## Hard constraints

1. All Interceptly UI actions happen through Chrome MCP. The bot
   never bypasses Interceptly to go straight at LinkedIn.
2. Before any Interceptly pass runs, `confirm-session-interceptly`
   verifies both the Interceptly sidebar AND the underlying
   LinkedIn profile URL match `interceptly-accounts.md` for the
   account about to be worked. A mismatch aborts and writes
   `_errors.md`.
3. When work rotates between accounts, `confirm-session-interceptly`
   runs again after every `switch-account`. Non-negotiable.
4. The bot never presses Start on an Interceptly campaign.
   `launch-campaign-interceptly` stops at CONFIGURED-NOT-STARTED —
   a human presses Start.
5. Nothing goes out on a reply thread without operator approval.
   The send gate is `rockstarr-reply:present-for-approval` — this
   plugin's `send-message` only runs when the handoff bundle
   returns `authorized-send`.
6. Third-party content is reference-only. `draft-icp-campaign`
   never paraphrases third-party material as if the client said
   it.
7. All state that is ours to own lives in `/rockstarr-ai/` per
   `rockstarr-infra`'s scaffold. Interceptly owns its own state.

## Skills (23)

### Install-time intake (4)

| Skill | Purpose |
|-------|---------|
| `discover-interceptly-accounts` | Walks the Interceptly SWITCH ACCOUNT popup via Chrome MCP, enumerates every managed LinkedIn account, asks the client which accounts this deployment should manage, writes `stack.md.outreach_accounts[]`. |
| `capture-interceptly-personas` | Per-account persona interview (5 questions). Writes `00_intake/interceptly-accounts.md`. |
| `capture-icp-qualifications` | Interview-driven capture of the baseline ICP rules used by `qualify-lead`. Writes `00_intake/icp-qualifications.md`. `draft-icp-campaign` reads this as the campaign baseline and only narrows it per-campaign. |
| `capture-warm-reply-pattern` | Captures the client's preferred warm-reply style and appends it as a subsection of `style-guide.md`. `rockstarr-reply:draft-reply` reads it when drafting warm-ICP replies. |

### Campaign lifecycle (3)

| Skill | Purpose |
|-------|---------|
| `draft-icp-campaign` | Per-campaign ICP clarification (additive tightening of the baseline) plus drafting the Interceptly campaign spec with the 3-step sequence (connect note + 2 follow-ups). Runs `stop-slop` on every message body. |
| `launch-campaign-interceptly` | Configures the campaign inside Interceptly's UI via Chrome MCP. **Stops at CONFIGURED-NOT-STARTED.** |
| `stop-campaign-interceptly` | Pause (resumable) or Stop (permanent). Updates Interceptly + the Campaigns row, and cancels dependent follow-up tasks. |

### Session + rotation (2)

| Skill | Purpose |
|-------|---------|
| `confirm-session-interceptly` | Safety gate. Checks Interceptly sidebar AND LinkedIn profile URL against `interceptly-accounts.md`. Writes a heartbeat row to the `Session` sheet. Refuses the pass on mismatch. |
| `switch-account` | Drives the Interceptly SWITCH ACCOUNT popup, then calls `confirm-session-interceptly` against the target. |

### Daily loop (3)

| Skill | Purpose |
|-------|---------|
| `preview-queue` | Writes `02_inputs/outreach/queue-<date>.md` with per-account unread / overdue / due-today / flagged counts and a plan of today's work. Togglable. |
| `process-inbox` | Walks Interceptly Inbox → Replied filter, oldest-first. For each unread thread: scrape context → `qualify-lead` → build handoff bundle → call `rockstarr-reply` → execute the returned bundle (send + label + task, or label-only, or flag, or book-meeting-handoff). |
| `process-my-tasks` | Runs after `process-inbox` returns zero unreads for the account. Walks Interceptly → My Tasks (overdue + due-today). `book-meeting` task type routes direct; other types run the same handoff as `process-inbox` with an `intent_hint` drawn from task metadata. |

### Per-reply channel I/O (4)

| Skill | Purpose |
|-------|---------|
| `qualify-lead` | Reads the right-panel context + any per-campaign ICP overrides. Returns `target` / `not_target` / `ambiguous` / `unknown` with a matching-rule string. Feeds the handoff bundle's `icp_verdict`. |
| `send-message` | Sends the approved body into the open Interceptly thread via Chrome MCP. Uses the React native-setter for the composer. Refuses to paste a body containing `stack.md.booking_link_url`. Logs to Messages sheet. |
| `apply-label` | Applies one Interceptly label to the current thread. Handles the Labels-nav-bug (clicking Labels sometimes navigates to Campaigns). Logs to Labels sheet. |
| `create-followup-task` | Converts a follow-up-timer keyword (from `rockstarr-reply:follow-up-timer`) + `stack.md.followup_timers` overrides into an Interceptly task with business-days math and the Friday→Monday shift for `meeting_proposed`. Logs to Tasks sheet. |

### Meetings + booking (3)

| Skill | Purpose |
|-------|---------|
| `propose-meeting-times` | Reads the client's availability source (booking-link page or Google Calendar) and returns 2-3 slots. Called by `rockstarr-reply:draft-reply` when it decides the thread is ready for a meeting ask. Logs to MeetingProposals sheet. |
| `book-meeting` | Runs only when `booking_mode = automated` + `availability_source = booking_link`. Drives the booking-link page, fills required fields, submits. On success calls `mark-booked`. On failure writes `_errors.md` and creates a `review-reply` task for the operator. |
| `mark-booked` | Single source of truth for booked state. Called by `book-meeting` (automated path) OR by the operator (manual path — "I booked Jane by phone"). Flips Leads state to booked, cancels every pending task for the lead, writes a Replies row with `classification = booked`. |

### Metrics + reporting (4)

| Skill | Purpose |
|-------|---------|
| `metrics-daily` | End-of-day rollup. One row per managed account per day into `Metrics (Daily)` — unreads processed, sends, labels by type, bookings, non-ICP declines, flags (sourced from `/02_inputs/replies/_flags.md`), session failures, campaigns configured/stopped. |
| `metrics-weekly` | Friday EOD: aggregate Metrics (Daily) into Metrics (Weekly) per account per ISO week with WoW deltas, then trigger `outreach-weekly-report`. |
| `outreach-weekly-report` | Markdown report at `06_reports/weekly/outreach-<yyyy-ww>.md`. Per-account tables, WoW deltas, bookings, flagged-open, stale review-reply tasks, non-ICP highlights, campaign changes, session/UI health, "What Rachel / Jon should notice." |
| `backup-workbook` | Friday EOD: snapshot `outreach-mirror.xlsx` to `06_reports/data/outreach-mirror-backup-<yyyy-ww>.xlsx`. |

Shared skills (`draft-icp-campaign`, `confirm-session-interceptly`,
`switch-account`, `propose-meeting-times`, `mark-booked`) ship
inline in this plugin for V0.1. Canonical source will migrate to
`rockstarr-infra/skills/_shared/` when the Interceptly variant
lands alongside other channel variants; at that point the
plugin-level wrappers will call through to the shared source.

## The handoff contract with rockstarr-reply

This plugin owns **channel I/O**; `rockstarr-reply` owns
**drafting**. The seam is a channel-agnostic bundle built by
`process-inbox` and `process-my-tasks`.

**Caller → rockstarr-reply:**

```
{
  channel: "linkedin",
  thread: { thread_id, messages[], thread_open_path },
  persona: { account_label, signature, persona_notes },
  lead:    { url, name, company, title, campaign_slug },
  icp_verdict: "target" | "not_target" | "ambiguous" | "unknown",
  icp_matching_rule: "<rule text from qualify-lead>",
  intent_hint: null | "bump" | "referral-pivot" | ...
}
```

**rockstarr-reply → caller** (one of):

- `authorized-send` — operator said send. Body path, proposed
  label, proposed follow-up-timer keyword. Caller runs
  `send-message` → `apply-label` → `create-followup-task`.
- `three-option-choice` — non-ICP warm case, operator picked
  option A or C. Same shape as `authorized-send`. Variant where
  operator picked B (let-it-hang) returns `label-only`.
- `label-only` — apply label, no send, no task.
- `flag` — rockstarr-reply refused to draft or operator flagged.
  `_flags.md` is already written. Caller applies the Follow Up
  label and creates a 2-business-day flagged_review task.
- `book-meeting-handoff` — lead agreed to a slot + supplied
  required fields. Caller routes direct to `book-meeting`.
- `no-action` — OOO, already booked, polite-ack with no signal.
  Caller applies the proposed label; no send, no task.

The send gate is strict and lives inside
`rockstarr-reply:present-for-approval`. Silence is not consent.
If the bundle returned is not `authorized-send` (or a resolved
`three-option-choice` with A or C), nothing goes out.

## Preconditions (every client)

Before any skill in this plugin runs, `rockstarr-infra` must have
already produced:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/style-guide.md` (explicitly approved)
- `/rockstarr-ai/00_intake/stack.md` with Interceptly keys set
- `/rockstarr-ai/01_knowledge_base/index.md` with first-party
  processed files (campaign drafts cite first-party proof).

`rockstarr-infra` must also expose:

- `skills/_shared/stop-slop/` — mandatory final pass on every
  prose draft. `draft-icp-campaign` calls it. If `stop-slop` is
  not discoverable, `draft-icp-campaign` refuses.

`rockstarr-reply` must be installed for the daily loop to do
anything past `confirm-session-interceptly` and `preview-queue`.
Without it, `process-inbox` refuses (drafting is not this
plugin's responsibility).

Then at install, run in order:

1. `discover-interceptly-accounts`
2. `capture-interceptly-personas`
3. `capture-icp-qualifications`
4. `capture-warm-reply-pattern`

If any install-time artifact is missing, the relevant skill
refuses and points the operator at the interview that produces it.

## Configuration keys (`stack.md`)

```
outreach_tool: interceptly
outreach_accounts:              # populated by discover-interceptly-accounts
  - label: <account label>
    interceptly_account_url: https://dash.interceptly.ai/...
    linkedin_expected_profile_url: https://www.linkedin.com/in/...
    managed: true|false
outreach_daily_preview: true
outreach_daily_run_time: "08:30"
booking_mode: automated|manual
availability_source: booking_link|google_calendar
booking_link_url: https://...
booking_link_required_fields: [email, phone]
followup_timers:                # optional overrides (business days)
  meeting_proposed: 2
  general: 3
  referral: 5
  cold_bump: 5
  flagged_review: 2
label_mapping:                  # optional overrides
  interested: "INTERESTED"
  booked: "Booked"
  referral: "Referral"
  follow_up: "Follow Up"
  contact_later: "Contact Later"
  bad_fit: "Bad Fit"
  ignore: "Ignore"
  not_interested: "Not Interested"
  flagged_review: "Follow Up"
```

## Folder contract

```
rockstarr-outreach-interceptly reads:
  00_intake/client-profile.md
  00_intake/style-guide.md           (includes warm-reply-pattern subsection)
  00_intake/stack.md
  00_intake/interceptly-accounts.md
  00_intake/icp-qualifications.md
  01_knowledge_base/index.md
  01_knowledge_base/processed/**
  02_inputs/outreach/icps/<slug>.md
  02_inputs/replies/_flags.md        (written by rockstarr-reply)
  04_approved/outreach/campaign-<slug>.md

rockstarr-outreach-interceptly writes:
  00_intake/interceptly-accounts.md
  00_intake/icp-qualifications.md
  00_intake/style-guide.md            (append-only warm-reply-pattern subsection)
  02_inputs/outreach/outreach-mirror.xlsx
  02_inputs/outreach/icps/<slug>.md
  02_inputs/outreach/queue-<yyyy-mm-dd>.md
  02_inputs/outreach/_errors.md
  03_drafts/outreach/campaign-<slug>.md
  05_published/outreach/<yyyy-mm-dd>.md
  06_reports/weekly/outreach-<yyyy-ww>.md
  06_reports/data/outreach-mirror-backup-<yyyy-ww>.xlsx
```

All paths are relative to the client's `/rockstarr-ai/` root.

## The mirror — audit, not source-of-truth

`/rockstarr-ai/02_inputs/outreach/outreach-mirror.xlsx` records
what the bot did so we can audit, roll up metrics, and reconcile
against Interceptly. **It is not authoritative.** Interceptly is.

This plugin owns every sheet. `rockstarr-reply` never touches
the workbook.

| Sheet | Role |
|-------|------|
| Campaigns | One row per managed campaign. Slug, account label, status, dates. `stop-campaign-interceptly` flips status + cancels dependent tasks. |
| Session | Every `confirm-session-interceptly` result, with `caller` column. |
| Qualifications | `qualify-lead` verdicts with matching rule cited. |
| Messages | Every reply the bot sent, with draft_path pointer. |
| Labels | Every label applied (or tried), with Labels-nav-bug notes. |
| Tasks | Interceptly tasks created, cancelled, or completed. |
| Replies | Inbound replies + booking events (written by `mark-booked`). |
| MeetingProposals | Slots proposed by `propose-meeting-times`. |
| Metrics (Daily) | One row per managed account per day. |
| Metrics (Weekly) | One row per managed account per ISO week with WoW deltas. |

The only reply-side surface this plugin reads is
`/02_inputs/replies/_flags.md`, written by
`rockstarr-reply:flag-for-review`. It is a markdown file of
appended blocks, not a sheet. `preview-queue`,
`metrics-daily`, and `outreach-weekly-report` read it for flag
counts; none of them edit it.

## Operational overview

**Campaign launch.** Operator triggers `draft-icp-campaign` →
review → `rockstarr-infra:approve` → `launch-campaign-interceptly`
→ (human presses Start inside Interceptly).

**Campaign wind-down.** Operator triggers
`stop-campaign-interceptly` with `mode=pause` (resumable) or
`mode=stop` (permanent). The Campaigns row flips; pending tasks
for the campaign's leads are cancelled.

**Daily loop (per scheduled run).**

1. For each managed account (in `stack.md.outreach_accounts[]`
   order):
   1. `confirm-session-interceptly` — abort the whole loop on
      fail.
   2. On the first account only: `preview-queue`.
   3. `process-inbox` — walk unreads, handoff to
      `rockstarr-reply`, execute returned bundle.
   4. `process-my-tasks` — walk overdue + due-today, same handoff.
   5. `switch-account` → next account.
2. `metrics-daily` rolls up the day.

Afternoon re-runs skip `process-my-tasks` and run inbox-only per
the schedule spec.

**End of week.** `metrics-weekly` → `outreach-weekly-report` →
`backup-workbook`.

## Approvals

| Gate | Routes through |
|------|----------------|
| Campaign spec | `rockstarr-infra:approve` on `03_drafts/outreach/campaign-<slug>.md`. |
| Campaign launch/stop | Operator triggers `launch-campaign-interceptly` / `stop-campaign-interceptly`. |
| Every outbound reply | `rockstarr-reply:present-for-approval`. `send-message` only runs on an `authorized-send` bundle. |
| Meeting booking | Either the operator books manually (calls `mark-booked`) or the bot runs `book-meeting` on an `agreed_start_iso` + required fields supplied through the reply thread and confirmed by the operator via `rockstarr-reply:present-for-approval`. |

## Known Interceptly UI quirks

The channel-I/O skills handle these explicitly:

- **React-textarea setter.** Interceptly's composer is a controlled
  React textarea — direct `.value =` does not fire the onChange
  handler. `send-message` and `launch-campaign-interceptly` use
  the React native-setter pattern.
- **SWITCH ACCOUNT popup timing.** Popup sometimes dismisses
  before the accounts list paints. `switch-account` waits for
  the list before clicking.
- **Virtualized industry dropdown.** Scrolling rubber-bands;
  `launch-campaign-interceptly` types to filter instead of
  scrolling.
- **Labels-nav-bug.** Clicking Labels sometimes navigates to
  Campaigns. `apply-label` detects and recovers.
- **My Tasks empty-state race.** Tasks tab occasionally shows
  empty even when tasks exist. `create-followup-task` waits 2s
  before concluding.
- **Same-day date dropdown.** Labeled oddly in some revisions.
  `create-followup-task` always uses the calendar date picker
  explicitly.

## Versioning

- `0.1.0` — initial cut. 27 skills, reply pipeline drafting
  lived here.
- `0.1.1` — **scope split with a tighter seam.** Drafting
  (classify-reply, draft-reply, present-for-approval,
  follow-up-timer, flag-for-review) moved to the sister plugin
  `rockstarr-reply`. This plugin keeps channel I/O, qualifies
  leads, and calls `rockstarr-reply` across a channel-agnostic
  handoff bundle. 23 skills. Cross-plugin contract points:
  the handoff bundle shape (`process-inbox` Step 4), the
  `/02_inputs/replies/_flags.md` file, and the
  `stack.md.followup_timers` keyword map consumed by
  `create-followup-task`.

Deferred past V0.1:

- Parallel-account pipelines (V0.1 rotates serially per account).
- Automatic reconciliation sweep (V0.1 reconciles lazily on the
  next time a thread is touched).
- Cross-variant shared-skills tree (V0.1 ships shared skills
  inline).
- Calendar-invite-accepted auto-booking (V0.1 requires explicit
  `mark-booked`).

## What this plugin does not do

- Does not install or talk to Sales Navigator, MeetAlfred,
  Dripify, Waalaxy.
- Does not send email, schedule social posts, or drive CRM.
- Does not draft reply copy, classify replies, derive the
  follow-up-timer keyword, or present drafts for approval. All
  of that is `rockstarr-reply`.
- Does not press Start on an Interceptly campaign.
- Does not silently retry after a `confirm-session-interceptly`
  failure. Wrong-account sending is the top reputational risk
  in this pillar.
- Does not edit `client-profile.md` from runtime activity.
- Does not paste the booking link into any reply. The booking
  link is a destination, not content.

## Troubleshooting

**"Session failed for account X, but the browser looks logged
in."** `confirm-session-interceptly` checks the LinkedIn profile
URL underneath the Interceptly session, not just the Interceptly
sidebar. A common cause is the client swapped the backing
LinkedIn account without updating Interceptly. Re-run
`capture-interceptly-personas` for that account.

**"`process-inbox` refuses to run."** It needs
`rockstarr-reply` installed. If rockstarr-reply is installed,
check that the last `confirm-session-interceptly` result is
`pass` — a `fail` heartbeat aborts the pass.

**"A paused campaign is still queuing follow-up tasks."** Check
that the Campaigns row `status` flipped (look at `paused_at`).
If the row is correct, `create-followup-task` will refuse to
create new tasks for that campaign's leads — existing tasks
should already be cancelled by `stop-campaign-interceptly`.

**"Interceptly says the campaign is running but the Campaigns
row says configured."** Someone pressed Start in Interceptly
without updating the mirror. Update `status` to `running` and
set `started_at` by hand. This plugin never presses Start; a
human did, and the mirror is behind.

**"The weekly report is missing flagged-leads."** Check
`/02_inputs/replies/_flags.md`. If the file is missing or empty,
`rockstarr-reply` hasn't flagged anything this week — that's
fine. If flags are in the file but not the report, confirm the
dates: `outreach-weekly-report` counts entries timestamped
within the ISO week.

**"A booking happened but the mirror doesn't show it."** The
operator may have booked manually and not called `mark-booked`.
Run `mark-booked` with the lead's thread reference and the
meeting datetime. The Replies sheet will then carry
`classification = booked` and the Tasks sheet will show the
cancellations.
