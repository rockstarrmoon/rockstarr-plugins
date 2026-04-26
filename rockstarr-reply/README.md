# rockstarr-reply

Channel-agnostic reply drafting for the Rockstarr AI Growth OS. Every
`rockstarr-outreach-*` plugin delegates reply drafting here — one
place for voice, temperature classification, the warm-ICP pattern, the
non-ICP three-option flow, the stop-slop pass, and the `send it`
authorization gate. The caller owns the channel I/O (scraping threads,
reading profile evidence, sending the approved body, labelling,
creating follow-up tasks).

Email variants (`rockstarr-reply-gmail`, `rockstarr-reply-outlook`)
ride this same core with read-inbox / write-draft adapters added on
top. V0.2.0 ships the core only.

## What's new in v0.2.0

This release wires `rockstarr-reply` into the cross-bot notification
stack added in `rockstarr-infra` v0.8.0:

- `approvals-digest` — daily 6am client-bound roll-up of every
  pending draft across every Rockstarr bot. Picks up reply drafts
  automatically via the new `approval_status: pending` front-matter
  field.
- `approvals-backlog-alert` — weekly strategist-bound nudge when
  the queue exceeds the configured threshold.
- `notify-reply-ready` — **urgent**, immediate, per-batch email
  that fires when a reply lands and a draft is staged. Renders the
  proposed body inline (single or three-option) with a
  `claude://cowork/new?q=...` deep-link straight back into the
  workspace.

The work is in `draft-reply` (front-matter contract rewrite +
synchronous notify call) and `present-for-approval` (approval_status
transitions). `classify-reply`, `follow-up-timer`, and
`flag-for-review` are unchanged. See **Front-matter contract** below
for the full field list.

## V0.2.0 skills

The same five core pipeline skills as v0.1.0, with v0.2 contract
updates noted in the second column:

| Skill | Purpose | v0.2 changes |
|---|---|---|
| `classify-reply` | Pure-prose analysis. Given thread text + `icp_verdict` from the caller, returns a temperature bucket (Hot / Warm-ICP / Warm-non-ICP / Skeptical / Cold) plus sub-type flags and a `proposed_label` computed from the default mapping overlaid with `stack.md.label_mapping`. No side effects. | unchanged |
| `draft-reply` | Core drafting step. Reads `style-guide.md` (including the warm-reply and optional referral-pattern subsections), picks the bucket's pattern, applies channel-adaptation for LinkedIn vs. email surfaces, calls `propose-meeting-times` on Hot, generates three option bodies on Warm-non-ICP, and runs stop-slop as the mandatory final pass. Writes to `/03_drafts/replies/<source_channel_slug>/<thread-id>.md`. | **Front-matter contract rewrite** to match `rockstarr-infra` v0.8 readers. **New synchronous step**: when `batch_context` is absent on the handoff, fires `notify-reply-ready` with the just-staged path. Skipped on redraft, batched flows, `book-meeting-handoff`, and `no-action`. |
| `present-for-approval` | The approval gate. Renders persona + ICP verdict + matching rule + bucket + proposed label + proposed follow-up timer. Blocks on an explicit `send it` (or clear equivalent). Editing instructions (`make it shorter`, `try a different angle`, `draft a breakup message`) trigger a redraft, never authorization. On authorization, appends the verbatim phrase to `/04_approved/replies/_approvals.log`. | **`approval_status` transitions** replace the v0.1 `awaiting_approval` boolean. `approved` on send; `rejected` with a `rejection_reason` on let-it-hang / skip / flag. Drives the digest's "is this still pending" check. |
| `follow-up-timer` | Maps bucket + `intent_hint` to a timer keyword (`meeting_proposed | general | referral | cold_bump | none`). Non-ICP `Ignore` / `Not Interested` / `Bad Fit` always return `none`. Caller's `create-followup-task` converts the keyword into a due date. | unchanged |
| `flag-for-review` | Refusal-to-draft handoff. Writes to `/02_inputs/replies/_flags.md` and returns `{ reason, note }` to the caller. The caller's own `apply-label` + `create-followup-task` turn that into a `Follow Up` label + 2-biz-day review task. | unchanged. Flagged threads stage no draft, fire no notification, and stay out of the daily digest by design. |

Deferred past V0.1.0:

- `rockstarr-reply-gmail` — read-inbox-gmail + write-draft-gmail.
  Built next (Rockstarr dogfoods Gmail).
- `rockstarr-reply-outlook` — fork-and-swap on top of Gmail when the
  first Outlook client signs.
- `capture-referral-pattern` — the optional companion to
  capture-warm-reply-pattern. Without it, Skeptical drafts default to
  silent-label.
- `refresh-icp-on-flags` — re-run classify-reply on every thread in
  `_flags.md` after icp-qualifications.md is updated.
- `reply-activity-report` — weekly roll-up of drafts / sends / flags
  / bookings / stop-slop-diff. Added after two weeks of dogfood data.
- Any flavor of `autosend`. Non-goal for V1 and V1.1. The human gate
  is the reason the process has survived real operator use.

## The handoff interface

Every caller (outreach-* plugin or email variant) talks to the core
pipeline through the same narrow, versioned interface. Additions are
minor bumps; changes to the response shape are major bumps.

**Caller → rockstarr-reply:**

| Field | Type | Notes |
|---|---|---|
| `channel` | enum | `linkedin-interceptly` \| `linkedin-salesnav` \| `linkedin-meetalfred` \| `linkedin-dripify` \| `linkedin-waalaxy` \| `email-gmail` \| `email-outlook` |
| `thread` | string | Full thread text, oldest to newest. Includes the lead's latest reply and all prior context. |
| `persona` | object | `{ name, title, signature_block, persona_notes }`. rockstarr-reply does NOT look this up itself. |
| `icp_verdict` | enum + meta | `target` \| `not-target` \| `ambiguous` \| `unknown` with `{ matching_rule, evidence }`. Caller runs its own qualify-lead adapter first. |
| `lead` | object | `{ url, name, company, title, campaign_slug? }` |
| `intent_hint` | enum (optional) | `reply` \| `bump` \| `breakup` \| `graceful-exit` \| `referral-pivot` \| `throwaway-question` \| `book-meeting-followup`. Used when the caller is re-entering the pipeline with a specific intent. |
| `batch_context` | string (optional, v0.2+) | Free-form. Set by callers that batch multiple drafts in one run (typically `detect-replies`). When present, draft-reply does NOT fire its own urgent notification — the caller fires `notify-reply-ready` once at the end of its batch. When absent, draft-reply fires urgently for the single just-staged path. |

**rockstarr-reply → caller (one of):**

| Response kind | Payload | Caller's next step |
|---|---|---|
| `authorized-send` | `{ body, subject?, proposed_label, proposed_followup_timer, persona_confirmed }` | Execute own `send-message`, `apply-label`, `create-followup-task`. Label + timer are suggestions; caller may override per `stack.md`. |
| `three-option-choice` | `{ pick: "A" \| "B" \| "C", action_for_caller, proposed_label }` | Only returned when the approver picked option B (let-it-hang). Caller labels `Ignore`, sends nothing, creates no task. Picks A or C re-enter the pipeline and return as `authorized-send` on the next turn. |
| `flag` | `{ reason, note, flags_path }` | Caller labels `Follow Up`, writes the note to its own flags mirror, creates a 2-biz-day review task. |
| `book-meeting-handoff` | `{ slot, lead_fields }` | Hot thread where the lead agreed AND supplied every `booking_link_required_field`. Caller's `book-meeting` skill drives the booking form. Only emitted when `stack.md.booking_mode=automated`. |
| `no-action` | `{ reason }` | Thread needs no reply (out-of-office, already-booked, thread continuation with no signal). Caller logs and moves on. |

## Front-matter contract (v0.2)

`draft-reply` writes every staged draft with the front-matter shape
below. The first block is the **cross-bot contract** that
`rockstarr-infra` v0.8's `notify-reply-ready` and `approvals-digest`
read. The second block is reply-specific and read only by
rockstarr-reply's own skills.

| Field | Required | Notes |
|---|---|---|
| `channel: "reply"` | yes | Lane discriminator. Always literal `"reply"` — NOT the source channel. |
| `source_channel` | yes | Human-readable origin label (e.g., `Sales Nav reply`, `Email reply (Gmail)`). Rendered in email subject + digest heading. |
| `source_channel_slug` | yes | Raw channel slug (`linkedin-salesnav`, `email-gmail`, etc.). Drives the channel-adaptation switch and the path segment. |
| `title` | yes | Human-facing summary. Default `Reply to <lead_name> at <lead_company>` plus pattern qualifier (`(meeting ask)`, `(non-ICP — 3 options)`, etc.). |
| `produced_by` | yes | `rockstarr-reply/draft-reply@0.2.0`. |
| `approval_status` | yes | Enum: `pending` (set on first stage) \| `approved` (set by present-for-approval on send) \| `rejected` (set on let-it-hang / skip / flag). |
| `awaiting_approval_since` | yes | ISO timestamp set on first stage; preserved across redrafts. |
| `path_relative` | yes | Path of this file, relative to `/rockstarr-ai/`. Deep-link target for notify-reply-ready. |
| `inbound_excerpt` | yes | Last inbound segment of the thread, ≤300 chars (`…` truncation). Verbatim, no reflow. |
| `draft_body` | yes (single-draft) | Post-stop-slop body string. Same string as the markdown's `## Body` section. Omitted when `draft_options` is present. |
| `draft_options` | yes (Warm-non-ICP) | Array of `{label, body}`. Three entries: graceful exit, let it hang, throwaway question. |
| `stop_slop_score` | yes | Numeric 0–50. |
| `stop_slop_flagged` | yes | `true` when score < 35. |
| `bucket`, `sub_types`, `proposed_label`, `proposed_followup_timer` | yes | From `classify-reply` and `follow-up-timer`. |
| `lead_name`, `lead_title`, `lead_company`, `lead_url` | yes | From the handoff `lead` object. |
| `thread_id`, `campaign_slug`, `persona_name`, `persona_title`, `icp_verdict`, `matching_rule`, `pattern`, `warm_reply_pattern_missing`, `generated_at`, `last_revised_at`, `revision_count`, `schema_version` | reply-specific | Read only by rockstarr-reply's own skills. |

The body content is duplicated — front-matter `draft_body` /
`draft_options` for programmatic readers (notify-reply-ready, future
digesters), markdown `## Body` / `### Option A/B/C` for
present-for-approval and any human looking at the file. Same string,
two surfaces.

## Caller integration: notify-reply-ready

`notify-reply-ready` (rockstarr-infra v0.8) is invoked from one of
two places — never both for the same draft:

1. **Synchronous, manual invocation.** When the operator says "draft
   a reply to Jane" outside the daily loop, `draft-reply` fires
   `notify-reply-ready` itself with `staged_paths = [<the single
   path>]` immediately after staging. The handoff bundle has no
   `batch_context` field.

2. **Batched, daily-loop invocation.** When an outreach-* plugin's
   `detect-replies` discovers N inbound replies and dispatches them
   to `draft-reply` in sequence, the handoff bundle MUST set
   `batch_context: detect-replies` (or any non-empty string).
   `draft-reply` skips its own notify call. After every reply has
   been drafted, `detect-replies` calls `notify-reply-ready` once
   with the accumulated `staged_paths` array.

   Outreach-* plugin authors: add this to your `detect-replies`:
   - On every handoff to `rockstarr-reply`, set `batch_context:
     detect-replies` in the bundle.
   - Accumulate the returned `draft_path` (from `authorized-send` or
     `three-option-choice` or the staged path on a non-terminal
     return).
   - At end of run, call `rockstarr-infra:notify-reply-ready` with
     `staged_paths = [<all accumulated paths>]`.

`notify-reply-ready` is a no-op when `staged_paths` is empty, so
runs that happen to stage zero new drafts (everything was
out-of-office or already-booked) silently emit no email.

## Hard constraints (preserved verbatim from the spec)

1. **Channel-agnostic.** The core pipeline does not know or care
   where the thread came from. It works off the handoff bundle.
   Channel-adaptation is a small switch in `draft-reply` that shapes
   surface format only — LinkedIn DMs are short with no subject and
   no signature; email may include a subject and appends the persona
   signature on send.
2. **Client-owned voice.** The bot has zero baked-in reply pattern.
   Everything comes from `style-guide.md` (including the warm-reply
   subsection from `capture-warm-reply-pattern` and the optional
   referral-pattern subsection from `capture-referral-pattern`) plus
   stop-slop.
3. **Client-owned ICP.** `classify-reply` reads
   `/00_intake/icp-qualifications.md`. The bot has no hardcoded
   target / not-target opinions. The caller supplies the verdict;
   classify-reply trusts it and flags conflicts without overriding.
4. **Strict `send it` authorization on every outbound.** Editing
   instructions (`make it shorter`, `try a different angle`,
   `draft a breakup message`, `continue`, `next`) are ALWAYS
   draft-and-present, NEVER send. The gate is: draft → present →
   wait for `send it` → send. The verbatim authorization phrase is
   logged to `/04_approved/replies/_approvals.log` on every send.
5. **stop-slop on every piece of prose** before the approver sees
   it. Voice first, stop-slop last, always. Structural outputs (the
   three-option labels themselves, log entries) are exempt.
6. **Booking URL is a destination, never content.** Hot threads
   propose specific times from `propose-meeting-times`; the link is
   driven by the caller's `book-meeting` skill only after the lead
   agrees and supplies required fields.
7. **No auto-send in V1 or V1.1.** Every outbound routes through the
   `send it` gate. A dedicated `rockstarr-reply-autosend` variant
   would require a separate proposal and is not on the roadmap.

## Preconditions (every client)

Before any skill in this plugin runs, `rockstarr-infra` must have
already produced:

- `/rockstarr-ai/00_intake/client-profile.md` (from
  `rockstarr-infra:ingest-workbook`)
- `/rockstarr-ai/00_intake/style-guide.md` (from
  `rockstarr-infra:generate-style-guide`, explicitly approved, and
  with the warm-reply subsection appended by
  `capture-warm-reply-pattern`)
- `/rockstarr-ai/00_intake/icp-qualifications.md` (from
  `capture-icp-qualifications`)
- `/rockstarr-ai/00_intake/stack.md` with reply-bot keys set (from
  `rockstarr-infra:capture-stack`):
  - `followup_timers` — override map. Default
    `meeting_proposed: +2 biz days (Mon-if-Friday)`,
    `general: +3`, `referral: +5`, `cold_bump: +5`,
    `none: (not configurable)`.
  - `label_mapping` — override map for the default label taxonomy
    (`INTERESTED`, `Ignore`, `Not Interested`, `Booked`, `Referral`,
    `Follow Up`, `Contact Later`, `Bad Fit`).
  - `booking_mode: automated | manual`
  - `availability_source: booking_link | gcal`
  - `booking_link_url:` (if `availability_source: booking_link`)
  - `booking_link_required_fields: [email, phone, company]`
  - `gcal_id:` (if `availability_source: gcal`)
  - For email variants: `from_address`, `mailbox_label` (Gmail) /
    `mailbox_folder` (Outlook) — not required for V0.1.0 core.

**rockstarr-reply v0.2.0 requires rockstarr-infra v0.8.0 or later.**
The cross-bot notification stack (`approvals-digest`,
`approvals-backlog-alert`, `notify-reply-ready`) and its supporting
`_shared/send-notification` helper land in v0.8.0; the v0.2 front-
matter contract is what those skills read. Earlier rockstarr-infra
builds will not error, but the urgent-reply and digest paths will
silently no-op.

`rockstarr-infra` must expose these shared skills:

- `skills/_shared/stop-slop/` — **MANDATORY** final pass on every
  piece of outbound prose. `draft-reply` refuses to produce a draft
  if stop-slop is unavailable. Returns a 0–50 score consumed as
  `stop_slop_score`.
- `skills/_shared/send-notification/` — mailer helper used by
  `notify-reply-ready` and `approvals-digest`. Best-effort — a
  failed urgent send does not block draft staging.
- `skills/notify-reply-ready/` — v0.8 cross-bot urgent notifier.
  Called by `draft-reply` (synchronous flow) or by an outreach-*
  `detect-replies` (batched flow). See **Caller integration:
  notify-reply-ready** above.
- `skills/approvals-digest/` — daily digest at 6am local. Reads
  `approval_status: pending` across every `/03_drafts/` lane. Reply
  drafts surface automatically.
- `skills/_shared/propose-meeting-times/` — Chrome MCP / calendar
  helper that returns 2–3 free slots for Hot drafts. In V0.1.x this
  skill also ships inline in `rockstarr-outreach-salesnav` — migrate
  to `_shared/` when a second outreach variant ships.
- `skills/_shared/mark-booked/` — single source of truth for the
  Booked state. Same shim: inline in outreach-salesnav for V0.1,
  migrates to `_shared/` later.
- `skills/_shared/capture-warm-reply-pattern/` and
  `skills/_shared/capture-icp-qualifications/` — intake interviews
  chained into install. For V0.1.x these ship inline in
  `rockstarr-outreach-interceptly-0.1.0`. **Required precondition for
  every rockstarr-reply-enabled client.** If the outreach-* plugin a
  client installs ships them inline, that satisfies the precondition.
  Otherwise, a future rockstarr-infra release will move them to
  `_shared/` and rockstarr-reply can chain into install directly.

The mailer requires `/rockstarr-ai/00_intake/.rockstarr-mailer.env`
with `ROCKSTARR_MAILER_TOKEN`, `ROCKSTARR_CLIENT_ID`,
`ROCKSTARR_NOTIFY_TO`, and (optionally) `ROCKSTARR_NOTIFY_URGENT_TO`.
Onboarding writes the template; Rachel or Jon fills in the values.

## Folder contract

Standard `/rockstarr-ai/` layout, with reply paths created by
`rockstarr-infra:scaffold-client`:

```
/rockstarr-ai/
├── 00_intake/
│   ├── stack.md                     (reply-bot section)
│   ├── icp-qualifications.md        (shared — client-owned)
│   └── style-guide.md               (warm-reply + optional referral-pattern subsections)
├── 02_inputs/
│   └── replies/
│       ├── email-queue.md               (email variants only — V0.2+)
│       └── _flags.md                    (threads the bot refused to draft)
├── 03_drafts/
│   └── replies/
│       ├── linkedin-interceptly/<thread>.md
│       ├── linkedin-salesnav/<thread>.md
│       ├── email-gmail/<thread>.md          (V0.2+)
│       └── email-outlook/<thread>.md        (V0.2+)
├── 04_approved/
│   └── replies/
│       ├── <channel>/<thread>.md            (authorized drafts, moved from /03_drafts/)
│       └── _approvals.log                   (every authorized-send + verbatim `send it` phrase)
└── 06_reports/
    └── data/
        └── reply-activity-<YYYY-WW>.md      (V0.2+)
```

Drafts are organized by channel under `/03_drafts/replies/` so the
client has one place to audit replies regardless of origin.

## Design decisions (preserved verbatim)

- **The reason rockstarr-reply exists as its own plugin** — not as
  a skill inside each outreach-* plugin — is maintenance. Every
  reply-drafting bug fix, every new temperature-bucket refinement,
  every stop-slop rule change lands once and propagates to every
  channel. The old interceptly design (reply drafting embedded in
  the outreach plugin) would have required three copies by the time
  outreach-salesnav shipped, and worse as MeetAlfred / Dripify /
  Waalaxy landed.
- **The handoff interface is intentionally narrow.** Callers do NOT
  get to supply partial drafts, pre-classified buckets, or custom
  prose rules — those break the "one place for reply logic"
  contract. If a caller needs to influence drafting, the influence
  must come through `style-guide.md` (voice),
  `icp-qualifications.md` (rules), or `stack.md` (configuration).
  Anything else is a feature request for rockstarr-reply, not a
  local override in the caller.
- **Channel-agnosticism does not mean channel-blind.** `draft-reply`
  has a channel-adaptation switch: LinkedIn drafts are short, no
  subject line, no signature block (sender account IS the
  signature). Email drafts may include a subject (on first reply)
  and DO append the persona signature block on send. The adaptation
  is metadata — it does not reshape the prose pipeline.
- **The three-option flow for warm-non-ICP is a design decision,
  not a bug to fix.** A polite `Absolutely` from a non-ICP is the
  default failure mode of LinkedIn. The three options (graceful
  exit / let-it-hang / throwaway question) are the answer: never
  pitch, never assume, present the options, let the approver call
  it. No default is ever pre-selected.
- **classify-reply trusts the caller's icp_verdict.** If the
  caller's qualify-lead adapter has a bug, classify-reply will not
  catch it. This is a deliberate separation: the caller knows its
  channel; rockstarr-reply knows voice and logic.
- **stop-slop runs AFTER style-guide shaping, never before.** If you
  scrub before the style guide is applied, you'll strip voice-
  specific turns of phrase the client approved.
- **Booking URL is a destination, not a data source.** The whole flow
  exists to keep the conversation human until there's a real agreed
  time, at which point the caller's book-meeting skill executes the
  booking on the lead's behalf. The URL never touches LinkedIn or —
  except in rare, style-guide-permitted email cases — the reply body.
- **Auto-send is a non-goal.** Every feature ask that sounds like
  "let the bot send this warm reply automatically" is a reason to
  say no. The human gate is the reason the process has survived real
  operator use.

## Pilot outcomes to watch

1. **`send it` false-negative rate** — times the approver wanted to
   send but the gate misread an edit as a no-send. Should be near
   zero.
2. **Three-option pick distribution** — does the approver ever
   actually pick A or C, or always B? If always B, the graceful-exit
   and throwaway-question paths are dead code.
3. **Warm-reply pattern fit** — are captured patterns producing
   drafts that get sent unchanged, or drafts that get rewritten? If
   drafts are rewritten most of the time, re-run
   `capture-warm-reply-pattern`.
4. **stop-slop cut volume** — how much prose is stop-slop removing
   each week? Falling volume means voice is holding; rising volume
   means the style-guide may need a refresh.
5. **Flag-to-follow-up resolution time** — how long a flagged thread
   sits before the operator resolves it.

## Changelog

### v0.2.0 — 2026-04-26

**Wires rockstarr-reply into rockstarr-infra v0.8.0's cross-bot
notification stack.**

- **New synchronous notify call.** `draft-reply` now calls
  `rockstarr-infra:notify-reply-ready` immediately after staging a
  draft, when the handoff bundle does not carry `batch_context`.
  Skipped on redrafts, batched flows, `book-meeting-handoff`, and
  `no-action` returns. Mailer errors are best-effort and never abort
  draft staging.
- **New handoff field: `batch_context` (optional).** Outreach-*
  `detect-replies` skills set this to suppress the per-draft urgent
  notification and fire one batched call at the end of their run.
- **Front-matter contract rewrite.** Drafts now carry the cross-bot
  fields the v0.8 readers expect: `channel: "reply"` (lane
  discriminator), `source_channel`, `source_channel_slug`, `title`,
  `produced_by`, `approval_status`, `awaiting_approval_since`,
  `path_relative`, `inbound_excerpt` (≤300 chars from the most-
  recent inbound), `draft_body` (or `draft_options` array on
  Warm-non-ICP), `stop_slop_score` (0–50 numeric). The draft body
  is now duplicated into front-matter so notify-reply-ready can
  render it inline in the urgent email; markdown body is unchanged.
- **`approval_status` replaces `awaiting_approval` (BREAKING).**
  `present-for-approval` now sets `approval_status: approved` on
  send and `approval_status: rejected` (with `rejection_reason`) on
  let-it-hang / skip / flag. This is what removes the draft from
  tomorrow's `approvals-digest`. Pre-v0.2 drafts (with
  `awaiting_approval: true` and no `approval_status`) will be
  skipped by the v0.8 digest — clean them up by hand or re-run
  through the v0.2 pipeline.
- **`stop_slop_score` numeric capture.** stop-slop's 0–50 score is
  surfaced. `stop_slop_flagged: true` is set when score < 35. On
  three-option Warm-non-ICP drafts, the front-matter score is the
  MIN across the prose-bearing options (A and C).
- **Path segment clarified.** The path is
  `/03_drafts/replies/<source_channel_slug>/<thread-id>.md` (the
  raw channel slug, not the lane discriminator). v0.1 used
  `<channel>` for the same value; the directory layout is
  unchanged.
- **No data migration needed.** The first paying client lands with
  rockstarr-reply v0.2.0 — there are no v0.1 drafts in production
  to migrate.

### v0.1.0 — 2026-04-24

- Initial public release. Core pipeline skills: `classify-reply`,
  `draft-reply`, `present-for-approval`, `follow-up-timer`,
  `flag-for-review`.
- Channel-agnostic rewrite of the reply-drafting pipeline previously
  embedded in `rockstarr-outreach-interceptly`. The salesnav variant
  has delegated here from day one; interceptly now delegates too.
- Handoff interface v1: `channel | thread | persona | icp_verdict |
  lead | intent_hint?` → `authorized-send | three-option-choice |
  flag | book-meeting-handoff | no-action`.
- Depends on `rockstarr-infra` for scaffolding, intake files, and
  shared skills (`stop-slop`, `propose-meeting-times`, `mark-booked`,
  `capture-warm-reply-pattern`, `capture-icp-qualifications`). The
  intake skills currently ship inline in
  `rockstarr-outreach-interceptly-0.1.0`; migration to
  `rockstarr-infra/skills/_shared/` is a planned follow-up release.
