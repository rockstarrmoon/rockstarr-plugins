# rockstarr-outreach-interceptly

Rockstarr AI LinkedIn outreach for clients who run **Interceptly**.
This plugin drives the Interceptly dashboard through the Chrome MCP,
rotates across every managed LinkedIn account the client has connected
in Interceptly, and handles the full reply-and-follow-up pipeline under
the client's approved voice — with a strict human gate before any
message leaves the client's account.

Unlike `rockstarr-outreach-salesnav`, where the local workbook is the
single state-of-truth, **Interceptly itself is source-of-truth here.**
`/rockstarr-ai/02_inputs/outreach/outreach-mirror.xlsx` is an audit
mirror only — if Interceptly and the mirror disagree, Interceptly wins
and the pipeline reconciles on the next pass.

The plugin is generic-for-any-client. No hardcoded persona, voice,
ICP, or reply pattern. Everything the bot needs is captured at install
through four interviews and stored in the client folder.

## Hard constraints

1. All sends happen through Interceptly's messaging surface via Chrome
   MCP. The bot never bypasses Interceptly to go straight at
   LinkedIn.
2. Every reply that is about to leave the client's account must pass
   `present-for-approval` with a verbatim "send it" authorization.
   Editing instructions, emoji, or "looks good" **do not** authorize.
3. Before any per-reply pipeline runs, `confirm-session-interceptly`
   verifies both the Interceptly sidebar AND the underlying LinkedIn
   profile URL match `interceptly-accounts.md` for the account about
   to be worked. A mismatch aborts the loop and writes `_errors.md`.
4. When the loop rotates between accounts, `confirm-session-interceptly`
   runs again after every `switch-account`. Non-negotiable.
5. `process-my-tasks` refuses to run while the current account's
   Inbox still has unreads. Inbox drains first.
6. The booking link is **never** pasted into a message. The bot either
   books on the lead's behalf via `book-meeting` (automated mode) or
   the client books out-of-band and calls `mark-booked`.
7. Third-party content is reference-only. `draft-reply-interceptly`
   never paraphrases third-party material as if the client said it.
8. All state that is ours to own lives in `/rockstarr-ai/` per
   `rockstarr-infra`'s scaffold. Interceptly owns its own state.

## Skills (27)

### Install-time intake

| Skill | Purpose |
|-------|---------|
| `discover-interceptly-accounts` | Walks the Interceptly SWITCH ACCOUNT popup via Chrome MCP, enumerates every managed LinkedIn account + the labels each account has, asks the client (via `AskUserQuestion`) which accounts this deployment should manage, writes `stack.md.outreach_accounts[]`. |
| `capture-interceptly-personas` | Per-account persona interview (5 questions: Interceptly account URL, backing LinkedIn profile URL, signature, persona notes, default signer). Writes `00_intake/interceptly-accounts.md`. |
| `capture-icp-qualifications` | 11-question interview on target / not-target / ambiguous rules. Writes `00_intake/icp-qualifications.md`. Shared skill. |
| `capture-warm-reply-pattern` | 7-question interview on warm-reply structure, length, opening, closing, banned moves. Appends a subsection to `style-guide.md`. Shared skill. |

### Campaign lifecycle

| Skill | Purpose |
|-------|---------|
| `draft-icp-campaign` | Two-phase: per-campaign ICP clarification (never writes back to `client-profile.md`) + draft the Interceptly campaign spec with the 3-step sequence (connect note + 2 follow-ups). Shared skill. |
| `launch-campaign-interceptly` | Configures the campaign inside Interceptly's UI via Chrome MCP. **Stops at CONFIGURED-NOT-STARTED** — a human presses Start. |
| `stop-campaign-interceptly` | Pause (resumable) or Stop (permanent). Cancels pending follow-up tasks for that campaign's leads. |

### Daily loop (per managed account, rotated)

| Skill | Purpose |
|-------|---------|
| `confirm-session-interceptly` | Safety gate. Checks Interceptly sidebar AND LinkedIn profile URL against `interceptly-accounts.md`. Gate on the entire per-account loop. |
| `preview-queue` | Writes `02_inputs/outreach/queue-<date>.md` listing per-account unread counts, overdue tasks, due-today tasks, known session issues. Togglable via `stack.md`. |
| `process-inbox` | Filters Interceptly Inbox to Replied, walks unreads oldest-first, runs the full per-reply pipeline for each. |
| `process-my-tasks` | Opens the My Tasks tab, walks overdue + due-today, runs the per-reply pipeline. Refuses if Inbox has unreads remaining. |
| `switch-account` | Drives the Interceptly SWITCH ACCOUNT popup, then calls `confirm-session-interceptly`. No post-switch work happens until session confirms. |

### Per-reply pipeline

| Skill | Purpose |
|-------|---------|
| `qualify-lead` | Target / not_target / ambiguous / unknown against `icp-qualifications.md` + per-campaign overrides. Writes the Qualifications sheet. |
| `classify-reply` | Bucket (hot / warm / cold / skeptical / decline) + sub_types (meeting_proposed / referral / pitch_back / polite_yes / requested_info). Writes the Classifications sheet. |
| `draft-reply-interceptly` | Picks a pattern from the verdict × bucket matrix. Applies voice (style-guide), persona (interceptly-accounts.md), then `stop-slop` as the final pre-write pass. Stages to `03_drafts/outreach/replies/<thread>.md`. |
| `present-for-approval` | Strict "send it" gate. The acceptable-authorization language is verbatim-preserved — editing instructions never authorize. Returns `{action: send | edit | flag | skip}`. |
| `send-message` | React native-setter paste + Send button with text-match verification. Logs to Messages sheet. Refuses without `/04_approved/` + `authorized_at` timestamp. |
| `apply-label` | Interceptly `label.click()` pattern with Labels-nav-bug recovery (clicking Labels sometimes navigates to Campaigns). One label per thread per pipeline run. |
| `create-followup-task` | Creates an Interceptly task with business-day math + Friday→Monday shift on `meeting_proposed`. No tasks created for Ignore / Not Interested / Bad Fit. |

### Meetings + handoff

| Skill | Purpose |
|-------|---------|
| `propose-meeting-times` | Reads `booking_link` or `gcal` per `stack.md`, returns 2–3 slots for the reply body. Never pastes the booking URL. Shared skill. |
| `book-meeting` | Automated booking path: fills the booking-link form via Chrome MCP once the lead agreed to a time. Only runs when `booking_mode=automated` AND `availability_source=booking_link`. Calls `mark-booked` on success. |
| `mark-booked` | Single source of truth for "a meeting got booked." Flips the Leads row, mirrors the Booked label into Interceptly via `apply-label`, cancels pending tasks, writes a Replies row. Shared skill. |
| `flag-for-review` | When the pipeline isn't confident, hand it off: append to `_flags.md`, apply Follow Up label, create a `review-reply` task. The bot never drafts for a flagged lead until resolved. |

### Metrics + reporting

| Skill | Purpose |
|-------|---------|
| `metrics-daily` | End-of-day rollup, one row per managed account per day into Metrics (Daily) + per-account summary appended to `05_published/outreach/<date>.md`. |
| `metrics-weekly` | Friday EOD: aggregate Metrics (Daily) rows per account per ISO week into Metrics (Weekly) with WoW deltas, then trigger `outreach-weekly-report`. |
| `outreach-weekly-report` | Markdown report at `06_reports/weekly/outreach-<yyyy-ww>.md`: per-account tables, WoW deltas, bookings, flagged-open, stale review-reply tasks, Non-ICP Log highlights, session/UI health, "What Rachel / Jon should notice." |
| `backup-workbook` | Friday EOD: snapshot `outreach-mirror.xlsx` to `06_reports/data/outreach-mirror-backup-<yyyy-ww>.xlsx`. |

Shared skills (`capture-icp-qualifications`, `capture-warm-reply-pattern`,
`draft-icp-campaign`, `propose-meeting-times`, `mark-booked`) ship inline
in this plugin for V0.1. Canonical source will migrate to
`rockstarr-infra/skills/_shared/` once that tree is populated; at that
point the plugin-level wrappers will call through to the shared source.

## Preconditions (every client)

Before any skill in this plugin runs, `rockstarr-infra` must have
already produced:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/style-guide.md` (explicitly approved)
- `/rockstarr-ai/00_intake/stack.md` with Interceptly keys set (see
  below)
- `/rockstarr-ai/01_knowledge_base/index.md` with first-party processed
  files (drafts cite first-party proof).

`rockstarr-infra` must also expose:

- `skills/_shared/stop-slop/` — mandatory final pass on every prose
  draft. The prose-producing skills in this plugin
  (`draft-icp-campaign`, `draft-reply-interceptly`,
  `outreach-weekly-report`) call it after the style-guide pass and
  before the file lands. If `stop-slop` is not discoverable, these
  skills refuse.

Then at install, run in order:

1. `discover-interceptly-accounts`
2. `capture-interceptly-personas`
3. `capture-icp-qualifications`
4. `capture-warm-reply-pattern`

If any of the install-time artifacts are missing, daily-loop skills
refuse and point the operator back at the relevant intake skill.

## Configuration keys (`stack.md`)

```
outreach_tool: interceptly
outreach_accounts:              # populated by discover-interceptly-accounts
  - label: <account label>
    interceptly_account_url: https://dash.interceptly.ai/...
    linkedin_expected_profile_url: https://www.linkedin.com/in/...
    managed: true|false
outreach_daily_run_time: "09:00"     # client-local
outreach_daily_preview: true
booking_mode: automated | manual
availability_source: booking_link | gcal
booking_link_url: https://...        # if booking_link
booking_link_required_fields: [email, phone, company]
gcal_id: primary                     # if gcal
followup_timers:
  meeting_proposed: 2                # business days
  general: 3
  referral: 5
  cold_bump: 5
  flagged_review: 2
label_mapping:                       # optional; override the defaults
  flag: "Follow Up"
  interested: "INTERESTED"
  booked: "Booked"
  referral: "Referral"
  follow_up: "Follow Up"
  contact_later: "Contact Later"
  bad_fit: "Bad Fit"
  ignore: "Ignore"
  not_interested: "Not Interested"
```

If any required key is missing, skills refuse with a pointer to
`rockstarr-infra:capture-stack`.

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
  02_inputs/outreach/icps/<slug>.md            (per-campaign ICP)
  04_approved/outreach/campaign-<slug>.md      (approved campaign spec)
  04_approved/outreach/replies/<thread>.md     (approved reply body)

rockstarr-outreach-interceptly writes:
  02_inputs/outreach/outreach-mirror.xlsx      (audit mirror)
  02_inputs/outreach/queue-<yyyy-mm-dd>.md     (daily preview)
  02_inputs/outreach/_errors.md                (loud failures)
  02_inputs/outreach/_flags.md                 (leads awaiting operator review)
  02_inputs/outreach/_non_icp_log.md           (not-target rulings)
  03_drafts/outreach/campaign-<slug>.md        (campaign draft)
  03_drafts/outreach/replies/<thread>.md       (reply drafts → present-for-approval)
  05_published/outreach/<yyyy-mm-dd>.md        (daily activity log)
  06_reports/weekly/outreach-<yyyy-ww>.md      (weekly report)
  06_reports/data/outreach-mirror-backup-<yyyy-ww>.xlsx
```

All paths are relative to the client's `/rockstarr-ai/` root.

## The mirror — audit, not source-of-truth

`/rockstarr-ai/02_inputs/outreach/outreach-mirror.xlsx` records what
the bot did so we can audit, roll up metrics, and reconcile against
Interceptly. **It is not authoritative.** Interceptly is.

| Sheet | Role |
|-------|------|
| Campaigns | One row per managed campaign. Slug, account label, status, dates. |
| Leads | One row per lead the bot has touched. Thread URL, LinkedIn URL, last label, campaign slug, flags, booking state. |
| Messages | Every send the bot made (account, thread, step, body). |
| Labels | Every label the bot applied (or tried to). |
| Tasks | Interceptly tasks the bot created (cancel / complete mirrored). |
| Qualifications | `qualify-lead` verdicts with rule cited + evidence. |
| Classifications | `classify-reply` buckets + sub_types. |
| Session | Every `confirm-session-interceptly` result (pass / fail with reason). |
| Flags | Flagged leads with reason + resolution state. |
| MeetingProposals | Slots proposed by `propose-meeting-times`. |
| Replies | Inbound reply records + booking events. |
| Metrics (Daily) | One row per managed account per day. |
| Metrics (Weekly) | One row per managed account per ISO week with WoW deltas. |

## Daily operational loop

At `stack.md.outreach_daily_run_time` (client's local time), for each
managed account in rotation:

1. `switch-account` — drive the Interceptly SWITCH ACCOUNT popup.
2. `confirm-session-interceptly` — verify the account. Hard gate.
3. `preview-queue` — update today's preview file (first account only).
4. `process-inbox` — walk unreads oldest-first through the per-reply
   pipeline. Must drain before step 5.
5. `process-my-tasks` — walk overdue + due-today tasks through the
   pipeline.
6. Rotate to next managed account, return to step 1.

After every managed account has finished:

7. `metrics-daily` — roll up today's per-account numbers, append the
   summary to `05_published/outreach/<date>.md`.

Every step fails gracefully if the previous produced nothing. Order is
fixed. Any `confirm-session-interceptly` failure aborts that account's
pass and surfaces loudly — subsequent accounts in the rotation still
run as long as their own session check passes.

Event-driven paths — `send-message`, `apply-label`,
`create-followup-task`, `mark-booked`, `book-meeting` — run whenever
they're triggered by the pipeline or the operator.

## Approvals

| Gate | Routes through |
|------|----------------|
| Campaign spec | `rockstarr-infra:approve` on `03_drafts/outreach/campaign-<slug>.md`. `launch-campaign-interceptly` only runs on approved files. |
| Campaign messages | Covered by campaign approval. Interceptly owns the send cadence from there. |
| Replies | Every draft → `present-for-approval` → verbatim "send it" → `send-message`. Editing instructions, emoji, or a typed "ok" **do not** authorize. |
| Follow-ups | Due-date follow-ups draft fresh and re-route through the per-reply pipeline. Per-reply gate applies. |
| Bookings | Covered by the approved reply that proposed the times. `book-meeting` runs without a second approval. Set `booking_mode: manual` to disable `book-meeting` entirely. |
| Flagged leads | `flag-for-review` only labels + tasks — the bot never drafts. The operator handles the thread by hand inside Interceptly, or updates `icp-qualifications.md` and re-runs. |
| Launch / stop | Client triggers `launch-campaign-interceptly` and `stop-campaign-interceptly`. The bot never starts or stops a campaign on its own. |

No SLA on `review-reply` approvals — stale items surface in the weekly
report. The cross-bot daily approvals digest on `rockstarr-infra`'s
backlog will pick them up via standardized `awaiting-approval`
front-matter when it ships.

## Known Interceptly UI quirks

The pipeline handles these explicitly. If you see the bot work around
something odd, this is why:

- **Labels-nav-bug.** Clicking a label chip from certain screens
  navigates to the Campaigns tab instead of applying the label.
  `apply-label` watches for the navigation and retries from the thread
  view.
- **Same-day dropdown quirk.** When creating a task due today,
  Interceptly's date picker renders the current day in a non-obvious
  position in the dropdown. `create-followup-task` uses the explicit
  date-string path rather than the dropdown.
- **SWITCH ACCOUNT popup timing.** The popup sometimes dismisses
  before the accounts list paints. `switch-account` waits for the
  list before clicking.
- **React-textarea setter.** Interceptly's message composer is a
  controlled React textarea — direct `.value =` does not fire the
  onChange handler. `send-message` uses the React native-setter
  pattern and only confirms the send after matching the button text.

## Versioning

- `0.1.0` — initial cut. 27 skills. Multi-account rotation with
  per-account session confirmation. Inbox-drains-before-tasks. Three-
  step Interceptly sequence (connect + 2 follow-ups). Verbatim
  "send it" authorization gate. Three-layer voice (style-guide +
  persona + stop-slop-last). Non-ICP polite-yes three-option flow.
  Bot-led bookings via booking link; client-led bookings via
  `mark-booked`. Audit mirror at `outreach-mirror.xlsx`.

Deferred past V0.1:

- Parallel-account pipelines (V0.1 rotates serially per account).
- Automatic reconciliation sweep (V0.1 reconciles lazily on the next
  time a thread is touched).
- Cross-variant shared-skills tree (V0.1 ships shared skills inline).
- SLA escalations on stale `review-reply` tasks (V0.1 surfaces them
  in the weekly report only).
- Calendar-invite-accepted auto-booking (V0.1 requires explicit
  `mark-booked`).

## What this plugin does not do

- Does not install or talk to Sales Navigator, MeetAlfred, Dripify,
  Waalaxy.
- Does not send email, schedule social posts, or drive CRM.
- Does not approve its own replies or bookings.
- Does not hand the booking link to the lead. Ever.
- Does not silently retry after a `confirm-session-interceptly`
  failure. Wrong-account sending is the top reputational risk in this
  pillar.
- Does not rewrite `client-profile.md` or `icp-qualifications.md` from
  runtime activity — both are intake artifacts the operator owns.

## Troubleshooting

**"Session failed for account X, but the browser looks logged in."**
`confirm-session-interceptly` checks the LinkedIn profile URL
underneath the Interceptly session, not just the Interceptly sidebar.
A common cause is the client swapped the backing LinkedIn account
without updating Interceptly. Re-run `capture-interceptly-personas`
for that account.

**"The bot keeps flagging leads as ambiguous."** Read
`02_inputs/outreach/_flags.md` + `_non_icp_log.md`. Patterns there
usually mean `icp-qualifications.md` is under-specified. Re-run
`capture-icp-qualifications`.

**"A reply got sent that I didn't expect."** Every send is logged to
the Messages sheet with the approval reference. Check
`04_approved/outreach/replies/` for the `authorized_at` timestamp.
If there is no `authorized_at`, that's a bug — file it with the full
`_errors.md` entry from that day.

**"Interceptly says the task is done but the mirror says pending
(or vice versa)."** Interceptly wins. The next time that thread
enters `process-inbox` or `process-my-tasks`, the mirror reconciles.
If the thread hasn't been touched in a while, manually re-run
`process-my-tasks` for the account.

**"The weekly report is missing an account."** `metrics-weekly` skips
accounts with zero Metrics (Daily) rows for the week but still writes
a zero-row per account, so absence usually means the daily loop
didn't run at all for that account. Check `Session` sheet + `_errors.md`.
