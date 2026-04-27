# rockstarr-outreach-salesnav

Rockstarr AI LinkedIn outreach for clients who won't or can't install a
third-party campaign tool (Interceptly, MeetAlfred, Dripify, Waalaxy).
This plugin drives Sales Navigator directly through the Chrome MCP and
keeps every lead, task, message, and metric in a single Excel workbook
inside the standard `/rockstarr-ai/` folder.

This variant is deliberately minimal. Advanced campaign features
(A/B sequence tests, weighted multi-campaign pacing, per-lead cap
coordination across outreach + nurture, SLA escalations) belong in
`rockstarr-outreach-interceptly` / MeetAlfred / Dripify variants.

## Hard constraints

1. All sends happen in Sales Nav — no 3rd-party outreach tool.
2. A lead belongs to exactly one campaign at a time. No duplicates.
3. Connection requests are capped at **20 per day + 100 per ISO week**
   across all active campaigns. Unused daily capacity carries forward
   inside the current ISO week only. Weekly ceiling is hard.
4. Every reply routes through `rockstarr-reply` for client approval
   before sending.
5. Before any daily send, `confirm-session` verifies the browser is
   signed into the client's LinkedIn. Mismatch aborts the whole loop.
6. The booking link is **never** handed to the lead. The bot either
   books on the lead's behalf after they agree to a proposed time, or
   the client books manually and calls `mark-booked`.
7. All state lives in `/rockstarr-ai/` per `rockstarr-infra`'s scaffold.

## Skills

| Skill | Purpose |
|-------|---------|
| `draft-icp-campaign` | Given an ICP + saved Sales Nav search URL, write the campaign spec (filters, 3-step sequence, cadence, exit conditions). Ships inline for V0.1; will migrate to `rockstarr-infra/skills/_shared/`. |
| `register-campaign` | Promote an approved campaign spec, crawl the saved search into Leads, seed initial connect tasks. |
| `crawl-lead-list` | Paginate a saved Sales Nav search via Chrome MCP and populate Leads. Enforces "one campaign per lead". |
| `confirm-session` | Pre-flight: verify the browser is signed in to the client's LinkedIn. Gate on the entire daily loop. |
| `preview-queue` | Daily preview file listing every action the bot plans to take today. Togglable via `stack.md`. |
| `daily-connect` | Compute today's budget (20/day + 100/week math), round-robin across active campaigns, send connect requests. Only skill that sends connects. |
| `detect-accepts` | Detect newly accepted connections and flip `Leads.state=accepted`. |
| `generate-message-tasks` | On accept, seed a 3-step sequence at day-of-accept / +3 / +7. |
| `send-scheduled-messages` | Execute `message-step-N` and follow-up tasks due today via Sales Nav messaging. |
| `detect-replies` | Pull inbound messages, append to Replies, cancel pending tasks for the lead, drive `rockstarr-reply` synchronously to stage a draft per inbound, then fire one urgent client email summarizing every newly-staged draft via `rockstarr-infra:notify-reply-ready`. |
| `send-approved-reply` | Send a reply approved by `rockstarr-reply`, seed a 2-day follow-up task. |
| `propose-meeting-times` | Read the availability source (booking link OR Google Calendar) and return 2–3 proposed slots for the reply. Ships inline for V0.1; will migrate to `rockstarr-infra/skills/_shared/`. |
| `book-meeting` | Automated booking path: drive the booking-link form via Chrome MCP once the lead has agreed to a time and supplied the required fields. |
| `mark-booked` | Single source of truth for "a meeting got booked." Flips `Leads.state=booked`, cancels pending tasks, writes a Replies row. |
| `stop-campaign` | Pause (resumable) or stop (permanent) a campaign; cancel pending tasks. |
| `metrics-daily` | End-of-day: one row per active campaign into Metrics (Daily); update ISO-week running totals. |
| `metrics-weekly` | Friday: per-campaign weekly roll-up into Metrics (Weekly). |
| `outreach-weekly-report` | Markdown weekly report from Metrics (Weekly). |
| `backup-workbook` | Weekly snapshot of `outreach-tasks.xlsx` into `/06_reports/data/`. |

Deferred past V0.1: `force-send-today`, `refresh-lead-list`,
`gcal-auto-booking`.

## Preconditions (every client)

Before any skill in this plugin runs, `rockstarr-infra` must have
already produced:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/style-guide.md` (explicitly approved)
- `/rockstarr-ai/00_intake/stack.md` with outreach keys set:
  - `outreach_tool: salesnav`
  - `linkedin_expected_profile_url: https://www.linkedin.com/in/...`
  - `outreach_daily_run_time: "09:00"` (client's local time)
  - `outreach_daily_preview: true` (default on)
  - `booking_mode: automated | manual`
  - `availability_source: booking_link | gcal`
  - `booking_link_url:` (if `availability_source: booking_link`)
  - `booking_link_required_fields: [email, phone, company]` (list)
  - `gcal_id:` (if `availability_source: gcal`)
- `/rockstarr-ai/01_knowledge_base/index.md` with first-party processed
  files (campaign copy cites first-party proof).

`rockstarr-infra` must also expose:

- `skills/_shared/stop-slop/` — mandatory final pass on every draft.
  The prose-producing skills in this plugin (`draft-icp-campaign`,
  `outreach-weekly-report`) call it after the style-guide pass and
  before the file lands in `03_drafts/` or `06_reports/`. If
  `stop-slop` is not discoverable, these skills refuse and point the
  user back at `rockstarr-infra`.
- `skills/notify-reply-ready/` — fired by `detect-replies` at the end
  of each run as a single urgent email summarizing every reply draft
  staged on this pass. Ships in `rockstarr-infra >= 0.8.0`. If
  unavailable, `detect-replies` still stages drafts and creates the
  `review-reply` tasks but skips the urgent email and surfaces an
  `infra_too_old` entry in its `notify_skipped_reasons` output. The
  next morning's `approvals-digest` covers the gap.

This plugin also requires:

- `rockstarr-reply >= 0.2.0` — the build that writes the v0.8.0
  front-matter contract `notify-reply-ready` reads (`channel: "reply"`
  + `source_channel`, `inbound_excerpt`, `draft_body`, `path_relative`,
  optional `draft_options[]`). `detect-replies` validates the contract
  on every staged path before handing the list off and drops paths
  with incomplete front-matter.
- `rockstarr-mailer >= 0.2.1` — claude:// URL scheme support for the
  per-draft deep-link in the urgent email.

If `stack.md` is missing any of the outreach keys, skills refuse to run
and point the user back at `rockstarr-infra:capture-stack`.

## Folder contract

```
rockstarr-outreach-salesnav reads:
  00_intake/client-profile.md
  00_intake/style-guide.md
  00_intake/stack.md
  01_knowledge_base/index.md
  01_knowledge_base/processed/**
  02_inputs/outreach/icps/<slug>.md          (client-supplied ICP)
  02_inputs/outreach/lead-lists/<slug>.md    (saved-search URL + notes)
  04_approved/outreach/campaign-<slug>.md    (approved campaign spec)

rockstarr-outreach-salesnav writes:
  02_inputs/outreach/outreach-tasks.xlsx     (SINGLE state-of-truth)
  02_inputs/outreach/queue-<yyyy-mm-dd>.md   (daily preview)
  02_inputs/outreach/_errors.md              (loud failures)
  03_drafts/outreach/campaign-<slug>.md      (campaign draft)
  03_drafts/outreach/replies/<thread>.md     (reply drafts for rockstarr-reply)
  05_published/outreach/<yyyy-mm-dd>.md      (daily activity log)
  06_reports/weekly/outreach-<yyyy-ww>.md    (weekly report)
  06_reports/data/outreach-tasks-backup-<yyyy-ww>.xlsx
```

All paths are relative to the client's `/rockstarr-ai/` root.

## The workbook — single state-of-truth

Every skill reads and writes
`/rockstarr-ai/02_inputs/outreach/outreach-tasks.xlsx`. Every sheet
carries a `campaign_slug` column; all active campaigns share the same
workbook.

| Sheet | Role |
|-------|------|
| Campaigns | One row per campaign. Slug, ICP ref, lead-list saved-search URL, status (active / paused / stopped), date started, date stopped, target_lead_count, notes. |
| Leads | Union of all leads. LinkedIn URL (primary key), name, company, title, `campaign_slug`, state (queued / connected / accepted / replied / booked / opted-out), date entered each state. A lead belongs to exactly one campaign at a time. |
| Tasks | Work queue. id, lead URL, `campaign_slug`, type (connect / message-step-N / follow-up / review-reply / book-meeting), due date, status (pending / done / cancelled), created-at, completed-at. |
| Connections | Daily log of connection requests sent. |
| Messages | Log of messages sent to connected leads (step number, body). |
| Replies | Log of inbound replies (raw text, classification, handoff state). |
| Metrics (Daily) | One row per campaign per day. |
| Metrics (Weekly) | One row per campaign per ISO week + rates + weekly cap used/remaining. |

Task types:

| Task type | Created by | Completed / cancelled by |
|-----------|------------|--------------------------|
| `connect` | `register-campaign` seeds one per lead; `daily-connect` schedules them. | `daily-connect` on send. Cancelled only by `stop-campaign`. |
| `message-step-N` | `generate-message-tasks` on accept. | `send-scheduled-messages` on send. Cancelled by `detect-replies` or `mark-booked`. |
| `review-reply` | `detect-replies` (hands off to `rockstarr-reply`). | Closed when `rockstarr-reply:approve` fires. Merged if lead sends another message first. |
| `follow-up` | `send-approved-reply` (due = today + 2). | Cancelled by `detect-replies` (new reply) or `mark-booked`; otherwise executed on due-date. |
| `book-meeting` | `draft-reply` when lead has agreed to a time AND supplied the booking-link's required fields. | `book-meeting` runs the form via Chrome MCP; calls `mark-booked` on success, drops `review-reply` on failure. |

## Daily operational loop

At `stack.md.outreach_daily_run_time` (client's local time):

1. `confirm-session` — verify LinkedIn session. Gate on everything below.
2. `preview-queue` — write `queue-<date>.md` (if `outreach_daily_preview: true`).
3. `detect-accepts` → `generate-message-tasks` — seed sequences.
4. `send-scheduled-messages` — execute message-step-N + follow-up tasks due today.
5. `detect-replies` — capture inbound, cancel pending tasks for the lead, drive `rockstarr-reply` synchronously to stage a draft per inbound, fire one urgent email summarizing the batch.
6. `daily-connect` — compute budget, round-robin across campaigns, send connects.
7. `metrics-daily` — roll up today's numbers per campaign; update ISO-week totals.

Every step fails gracefully if the previous produced nothing. Order is fixed.

An optional afternoon pass re-runs `detect-replies` only (no connect
loop). Event-driven paths — `send-approved-reply`, `book-meeting`,
`mark-booked` — run whenever they're triggered.

## Approvals

| Gate | Routes through |
|------|----------------|
| Campaign spec | `rockstarr-infra:approve` on `03_drafts/outreach/campaign-<slug>.md`. `register-campaign` only runs on approved files. |
| Campaign messages | Covered by campaign approval. Per-send does not re-prompt. |
| Replies | Every inbound reply → `rockstarr-reply` → human approve → `send-approved-reply`. |
| Follow-ups | 2-day follow-ups draft fresh on due-date via `rockstarr-reply`. Per-reply gate applies. |
| Bookings | Covered by the approved reply that proposed the times. `book-meeting` runs without a second approval. Set `booking_mode: manual` to disable `book-meeting` entirely. |
| Start / stop | Client triggers `register-campaign` and `stop-campaign`. The bot never starts or stops a campaign on its own. |

No SLA on `review-reply` approvals — stale items surface in the weekly
report's "bot heartbeat" callout. The cross-bot
`rockstarr-infra:approvals-digest` runs every morning and surfaces every
still-pending `review-reply` (and every other awaiting-approval draft
across bots) on the standardized front-matter contract that
`rockstarr-reply >= 0.2.0` writes. The `notify-reply-ready` urgent
email fired by `detect-replies` is an *acceleration* of the digest for
the just-staged drafts, not a replacement — a draft staged at 9:14am
appears in the urgent email at 9:15am AND in the next morning's
digest if it hasn't been approved by then.

## Versioning

- `0.1.0` — initial cut. 19 skills, Sales Nav only, booking link OR
  Google Calendar (manual book) for availability.
- `0.1.1` — `draft-icp-campaign` sequence tightened: Message 1 blank
  (LinkedIn default), Message 2 acknowledge/tease, Message 3 invite
  engagement, Message 4 respectful close. No prose-body changes to
  other skills.
- `0.1.2` — `draft-icp-campaign` hardened further: campaign ICP now
  derives from `00_intake/client-profile.md` via an interactive,
  non-destructive per-campaign clarification pass (adjustments
  NEVER write back to `client-profile.md`). Six hard Sequence rules
  added: no "following up" language anywhere, no em dashes in the
  sequence, never announce absence of pitch, anchor phrase stays
  verbatim across Messages 2/3/4, Message 2 is a curiosity opener
  that earns a reply on its own, Message 4 names the ceiling (not
  the pain) in the "X gets attention only when there is time, which
  is never" shape. Self-check pass added as drafting rule 7.
- `0.1.3` — `draft-icp-campaign` tightens Messages 3 and 4 and
  narrows the anchor-phrase rule. Anchor phrase now appears verbatim
  in Message 2 ONLY — Messages 3 and 4 do not repeat it (repetition
  reads as scripted and kills the conversational tone). Message 3
  drops the trailing permission softener ("fine if I send another
  note?" / "OK to stay in touch?") — that kind of line is filler,
  not invitation. Message 4 is renamed from "Ceiling close" to
  "Respectful close" and capped at two or three sentences: one
  specific "weeks get full"–style acknowledgment + one open-door
  line + stop. No anchor restatement, no ceiling restatement, no
  well-wishes, no chase. Rule 4 rewritten, Rule 6 rewritten, new
  Rule 7 added for the softener ban, self-check pass updated.
- `0.1.4` — Stop-slop integration per the Growth OS cross-bot
  contract. `draft-icp-campaign` gains Drafting rule 8: after the
  self-check rewrites, the campaign spec prose and all three
  sequence message bodies (Messages 2, 3, 4) must pass
  `rockstarr-infra:stop-slop` as the final pre-write pass. Order is
  style-guide first, stop-slop last. `outreach-weekly-report` adds
  an equivalent requirement for the prose blocks (week-over-week
  bullets, "What Rachel / Jon should notice," and the narrative
  portion of "Actions for next week"); tables, heartbeat lines, and
  the stale-task callout are exempt as structural artifacts. Both
  skills refuse if `stop-slop` is not discoverable. Produced-by
  stamps bumped.
- `0.1.6` — Three changes shipped together.
  1. `notify-reply-ready` integration. `detect-replies` now drives
     `rockstarr-reply` synchronously to stage a reply draft per
     inbound (instead of just signalling), accumulates the resulting
     paths into `staged_paths[]`, and at end-of-run fires a single
     urgent client email via `rockstarr-infra:notify-reply-ready`
     that summarizes every newly-staged draft with a `claude://`
     deep-link per card. Empty batch = silent skip (the feature).
     `detect-replies` validates each staged path's front-matter
     against the v0.8.0 contract before the call and drops paths
     with incomplete front-matter into `notify_skipped_reasons`.
     Output block gains `drafts_staged`, `book_meeting_handoffs`,
     `notify_message_id`, and `notify_skipped_reasons[]`. Requires
     `rockstarr-infra >= 0.8.0`, `rockstarr-reply >= 0.2.0`,
     `rockstarr-mailer >= 0.2.1`. Existing `rockstarr-reply not
     installed` and per-thread error fallbacks preserved — those
     paths skip the urgent email and let the next morning's
     `approvals-digest` cover the gap.
  2. `draft-icp-campaign` sequence personalization + Message 4 hook
     rewrite. Messages 2 and 3 now open with a literal
     `{first_name}` placeholder (bare-name comma-led OR
     greeting-led, writer's pick per the client's voice);
     `send-scheduled-messages` substitutes from `Leads.name` at
     send time. Message 4 hook is rewritten from a generic empathy
     opener ("weeks get full," "I know you're busy") to a
     benefit-to-ICP hook framed as a future-condition ("when X
     becomes useful," "if Y starts costing you Z") drawn from the
     per-campaign ICP's pain focus + a `kb_sources_used` proof
     point. New Sequence rules 8 (`{first_name}` placeholder
     convention) and 9 (Message 4 benefit-hook spec) added. Self-
     check pass extended with eight new checks covering placeholder
     presence on Messages 2/3, placeholder absence on Message 4,
     literal-token validation, benefit-hook shape, first-party
     trace, and pitch/measured-outcome bans. Sequence Rule 6
     updated to reference the benefit hook instead of the empathy
     line. `produced_by` stamps bumped @0.1.4 → @0.1.6.
  3. `send-scheduled-messages` placeholder substitution spec'd. Step
     2a-i now defines the parse rules for `{first_name}` (whitespace
     split on `Leads.name`, take token zero, strip non-alphabetic
     edges) and `{company}` (verbatim from `Leads.company`). On
     `{first_name}` parse failure the fallback is shape-aware:
     bare-name comma-led (`{first_name}, ...`) drops the prefix and
     capitalizes the next word's first letter, so the body opens
     cleanly without an awkward leading comma; greeting-led
     (`Hi {first_name}, ...`, `Hey {first_name}, ...`) substitutes
     `there`, keeping the greeting intact. Mid-sentence fallback
     defaults to `there`. New post-substitution residue check
     aborts a single send if any literal `{...}` token survives —
     unresolved placeholders never reach the lead. Output block
     gains `skipped_unresolved_placeholder`,
     `first_name_fallback_bare`, and `first_name_fallback_greeting`
     so operators can spot dirty workbook data without grepping
     `_errors.md`. What-NOT-to-do block hardened with five new
     rules covering pre-resolution in approved files, the
     difference between the two fallback paths, and the residue
     check.
- Deferred to later versions: `force-send-today`, `refresh-lead-list`,
  `gcal-auto-booking`, weighted multi-campaign pacing, cross-bot touch
  caps.
- The two "shared" skills (`draft-icp-campaign`, `propose-meeting-times`)
  ship inline in this plugin for V0.1 so it's self-contained. When
  `rockstarr-outreach-interceptly` lands, both move to
  `rockstarr-infra/skills/_shared/` and the plugin-level wrappers call
  the shared source.

## What this plugin does not do

- Does not install or talk to Interceptly, MeetAlfred, Dripify, Waalaxy.
- Does not send email, schedule social posts, or drive CRM.
- Does not approve its own campaigns, replies, or bookings.
- Does not hand the booking link to the lead. Ever.
- Does not silently retry after a `confirm-session` failure. Wrong-
  account sending is the top reputational risk in this pillar.
