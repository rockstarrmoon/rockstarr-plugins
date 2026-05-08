# rockstarr-ops

Sales-ops bot for the Rockstarr AI Growth OS. Runs the human-facing
sales-operations workflows for any B2B founder-led client: daily call
prep, sales-call prep (discovery + close), client catch-up agendas,
old-lead audits, lead reengagement, post-call transcript processing,
and recurring email-deliverability tests.

This plugin is **tool-agnostic by design**. There is no baked-in CRM,
recorder, task system, calendar, deliverability tool, outreach tool,
voice, pitch, or call framework. Every shaping decision comes from
intake-produced files (`pitch.md`, `ops-call-framework.md`,
`deliverability-config.md`, `icp-qualifications.md`, `style-guide.md`,
optional `client-roster.md`) plus `stack.md`. Installing this bot for
a new client is a few intake interviews and a pilot period — not a
fork of the plugin.

## What this bot does NOT do

- **Drafting prose.** Every customer-facing message (audit
  recommendations, reengagement messages, post-call recap notes,
  post-call follow-up emails) is handed off to `rockstarr-reply` via
  the standard handoff bundle with an `intent_hint`. This bot never
  re-implements reply drafting.
- **CRM writes.** Every structured CRM update is handed to the
  active `rockstarr-ops-bot-<crm>` variant (today: `-ga`; on the
  roadmap: `-hubspot`, `-pipedrive`, `-salesforce`, `-close`). When
  no CRM ops bot is installed, the bundle becomes a manual punch
  list — never a silent skip.
- **LinkedIn DM sends.** Every LinkedIn DM is sent by the active
  `rockstarr-outreach-*` variant via its `send-message`. When no
  outreach plugin is installed, audit Play 3 + reengage-lead's
  send step degrade to email-draft instead.

## V0.1 skills

| Skill | Type | Purpose |
|---|---|---|
| `capture-pitch` | intake | Interviews the operator on offer / pricing / positioning. Writes `00_intake/pitch.md`. Single source of truth for current pricing — old transcripts are NOT trusted on conflict. |
| `capture-deliverability-config` | intake | Interviews the operator on sender / tracker URL / test segment / template clone source / deliverability task URL / comment format. Writes `00_intake/deliverability-config.md`. Skipped when `ops_deliverability_tool=none`. |
| `capture-call-framework` | intake | Interviews the operator on call phases / pricing card / pain hooks / disqualify language / deal-breaker filters. Writes `00_intake/ops-call-framework.md`. Required by both prep skills. |
| `daily-call-prep` | orchestrator | Daily 06:00 local sweep. Classifies every event on today's calendar, sequences the per-event prep skills, posts one mobile-readable summary to chat with `computer://` links. |
| `prep-call-1` | per-prospect | Discovery prep doc to `03_drafts/ops/sales-prep/`. Intel from LinkedIn + website + outreach thread + booking source. ICP verdict + primary pain hook + phase-by-phase script tuned to the per-client framework. |
| `prep-call-2` | per-prospect | Close prep doc to `03_drafts/ops/sales-prep/`. Intel from recorder (verbatim quotes) + email + LinkedIn thread. Mirror lines in the prospect's exact words; pitch tuned to skip rejected pain framings. |
| `build-client-agenda` | per-client | Catch-up agenda doc to `03_drafts/ops/agendas/`. Recorder action items split owe-them / owe-you, out-for-review tasks (stale flagged), 14-day production window grouped by week, open questions, 3-5 quick-reference bullets. |
| `audit-lead` | on-demand | Old-lead audit. Four-source walk in fixed order (recorder → CRM → email → outreach), 5-part synthesis, recommends ONE play out of six, hands draft to rockstarr-reply (or calendar reminder for Play 6). |
| `reengage-lead` | on-demand | Dead-lead revival. Hard scope: prior recorded call + LinkedIn ghosting + recent newsletter / sequence engagement. Hands draft to rockstarr-reply with `intent_hint=reengagement`. Routes send through the active outreach variant. |
| `process-call-transcript` | event-driven | Walks the post-call queue. Per task: read transcript, extract per-client field set, hand CRM update bundle to active `rockstarr-ops-bot-<crm>` (or surface manual punch list), hand recap-note draft to rockstarr-reply, comment back on queue task. |
| `run-deliverability-test` | recurring | 9-step recurring spam-score check. Deterministic report URL construction, clone-safety check, score logged to deliverability tracker. LOW SCORE prefix when score below configured threshold. |
| `ops-weekly-report` | weekly | Friday EOD rollup. Sales-prep counts, audits, reengagements sent + replied, post-call processings, deliverability scores trend. Surfaces stale review items across all clients. |
| `backup-workbook-ops` | weekly | Friday EOD snapshot of the small ops mirror at `02_inputs/ops/ops-mirror.xlsx` to `06_reports/data/ops-mirror-backup-<YYYY-WW>.xlsx`. |

## Dependencies

- **`rockstarr-infra` (HARD).** `>= 0.8.4`. Provides `scaffold-client`
  (creates the ops folder layout), `capture-stack` (captures every
  `ops_*` field listed below), `kb-ingest`, `generate-style-guide`,
  the cross-bot mailer (`approvals-digest`,
  `approvals-backlog-alert`, `notify-reply-ready`), and the
  `_shared/stop-slop/` reference consumed inside rockstarr-reply.
- **`rockstarr-reply` (HARD, peer plugin).** `>= 0.2.0`. Owns
  classify + draft + present-for-approval + stop-slop. Every
  customer-facing message this bot proposes routes through it.
- **`rockstarr-ops-bot-<crm>` (SOFT, peer plugin).** Today: `-ga`;
  roadmap: `-hubspot`, `-pipedrive`, `-salesforce`, `-close`. When
  installed, this bot hands a CRM update bundle to it for execution.
  When NOT installed, the bundle becomes a manual punch list.
- **`rockstarr-outreach-*` (SOFT, peer plugin).** When installed,
  reengagement + audit Play 3 LinkedIn DMs route through its
  `send-message`. When NOT installed, those steps degrade to
  email-draft.

## Stack fields read from `00_intake/stack.md`

| Field | Common values | Purpose |
|---|---|---|
| `ops_calendar` | `google_calendar` / `outlook` | Today's events for daily-call-prep; single-event create for audit-lead Play 6. |
| `ops_meeting_recorder` | `read_ai` / `otter` / `fathom` / `fireflies` / `none` | Prior call reports + verbatim quote extraction. |
| `ops_task_system` | `clickup` / `asana` / `linear` / `notion` / `trello` / `basecamp` / `none` | Client roster, post-call queue, deliverability queue, comments + status changes. |
| `ops_email_outreach_tool` | matches active `rockstarr-outreach-*` | LinkedIn thread reading. |
| `ops_email_platform` | `gmail` / `outlook` | Email thread reading + draft creation. |
| `ops_deliverability_tool` | `mailreach` / `gmass` / `none` | Test-code generation, score retrieval. |
| `ops_deliverability_tracker` | `google_sheet` / `airtable` | Score logging (append-only). |
| `ops_daily_run_time` | `06:00` (default) | When daily-call-prep fires. |
| `ops_client_roster_source` | `static_list` / `task_system_clients_list` | Where to read the catch-up client list. |
| `ops_post_call_queue_list` | string | The list / project in the task system that holds the post-call queue. |
| `ops_post_call_queue_tag` | string | The tag identifying tasks the post-call queue should pick up. |
| `ops_post_call_comment_format` | string | Template for the comment posted back on a processed queue task. |

## Folder layout (added by `scaffold-client`)

```
/rockstarr-ai/
├── 00_intake/
│   ├── pitch.md                      (capture-pitch)
│   ├── deliverability-config.md      (capture-deliverability-config; skipped when none)
│   ├── ops-call-framework.md         (capture-call-framework)
│   └── client-roster.md              (only when ops_client_roster_source=static_list)
├── 02_inputs/
│   └── ops/
│       ├── audit-<lead-slug>.md      (audit-lead synthesis before drafting)
│       ├── ops-mirror.xlsx           (small mirror; reengagements + post-calls + deliv runs)
│       └── _flags.md
├── 03_drafts/
│   └── ops/
│       ├── sales-prep/               (Call1_Prep_<name>_<date>.docx, Call2_Prep_<name>_<date>.docx)
│       ├── agendas/                  (ClientAgenda_<client>_<date>.docx)
│       ├── recaps/                   (post-call recap notes awaiting CRM write)
│       └── replies/                  (reply drafts staged by rockstarr-reply on this bot's behalf)
├── 04_approved/
│   └── ops/
├── 05_published/
│   └── ops/
│       ├── daily-summary-<date>.md
│       ├── reengagements-<YYYY-MM>.md
│       ├── post-calls-<YYYY-MM>.md
│       └── deliverability-<YYYY-MM>.md
└── 06_reports/
    ├── weekly/
    │   └── ops-<YYYY-WW>.md
    └── data/
        └── ops-mirror-backup-<YYYY-WW>.xlsx
```

## Approval discipline

- **Per-prep-doc.** Prep docs and agendas write to `03_drafts/ops/`.
  The operator reads them before the meeting. No `send` step —
  these are operator-facing artifacts.
- **Per-message (strict, inherited from rockstarr-reply).** Every
  customer-facing message routes through rockstarr-reply's
  `present-for-approval`. `send it` or clear equivalent. Editing
  instructions are NEVER authorization.
- **Per-CRM-write.** Every CRM update bundle is presented before
  execution. The operator approves field set + recap-note body, then
  the active CRM ops bot writes.
- **Per-deliverability-run.** Nine-step flow runs without per-step
  approval. The single human gate is the score-comment on the queue
  task. Scores below threshold get a `LOW SCORE` prefix and a flag
  in the weekly report.
- **Audit-play.** `audit-lead` recommends ONE play. The operator
  can override with a one-word reply. No play, no draft.
- **Reengagement scope.** Hard refuse without a prior recorded call.
  Engagement-signal check overridable with explicit
  `reengage anyway`.
- **Post-call queue.** No batch-process. Each task gets the per-CRM-
  write gate before its bundle ships.

## Hard rules — do not weaken

- **No batch-send anywhere.** Every customer-facing message ships
  through one explicit `send it`. No `reengage these 50 leads` path.
  Batch reengagement is a misuse of the workflow.
- **`pitch.md` over the transcript on every conflict.** Old
  recordings quote stale pricing; the bot trusts `pitch.md`.
- **Quotes from the recorder are verbatim.** The bot does NOT
  invent quotes, and does NOT paraphrase what the lead said.
  When the transcript is unavailable, say so plainly rather than
  guessing.
- **Four-source order in `audit-lead` is fixed.** Recorder → CRM →
  email → outreach. Reading them out of order mis-frames the
  synthesis. This was hard-won from real use.
- **Six audit plays. No more, no less.** Pile-on multiple plays is
  a misuse — it produces noise the lead reads as desperation.
- **`process-call-transcript` writes via `rockstarr-ops-bot-<crm>`,
  never via Chrome MCP directly.** The CRM ops bot is the single
  point of audit for every CRM mutation in the Growth OS.
- **Internal docs (prep / agenda / audit synthesis / recap-note
  draft) bypass stop-slop.** Stop-slop is for what the LEAD reads,
  not what the operator reads. Customer-facing prose handed to
  rockstarr-reply DOES run through stop-slop inside reply.

## Cross-bot handoff bundles

### To `rockstarr-reply` (HARD)

Standard handoff bundle from
`rockstarr-reply/skills/draft-reply/SKILL.md`. The intent hints used
by this bot are:

- `reengagement` — reengage-lead.
- `post-call-recap` — process-call-transcript's recap-note draft.
- `audit-fulfill` — audit-lead Play 1 (operator-owed deliverable).
- `audit-reframe` — audit-lead Play 2 (reframe the offer).
- `audit-clean-break` — audit-lead Play 5 (graceful exit).
- `audit-third-party` — audit-lead Play 4 (third-party intro).

(Audit Play 3 hands off to `reengage-lead` instead of drafting.
Play 6 creates a calendar reminder via `ops_calendar` — no draft.)

### To `rockstarr-ops-bot-<crm>` (SOFT)

```
{
  contact_id: <id> OR { first_name, last_name, email, phone, business_name },
  field_updates: { <field_name>: <value>, ... },
  note_payload: { sections: [...], links: [...] },
  status_transition: <enum value, optional>
}
```

Every CRM ops bot variant accepts the same bundle. When no CRM ops
bot is installed, the bundle is rendered as a manual punch list in
chat — never silently skipped.

### To `rockstarr-outreach-*` (SOFT)

The same authorized-send bundle `rockstarr-reply:present-for-approval`
returns. The active outreach variant's `send-message` takes over.
When no outreach plugin is installed, the LinkedIn-DM step degrades
to a Gmail / Outlook draft instead.

## Schedule

- **Daily** at `ops_daily_run_time` (default `06:00` local):
  `daily-call-prep` runs end-to-end.
- **Recurring** on the cadence captured in `deliverability-config.md`
  (typically weekly): `run-deliverability-test`. Either picked up
  from the recurring task in the task system OR fired by a Cowork
  scheduled task.
- **Weekly Friday EOD:** `ops-weekly-report` + `backup-workbook-ops`.

## Pilot outcomes to watch

The first dogfood pass on Rockstarr & Moon's own ops workflow tracks
six metrics: (a) Call 1 vs. Call 2 detection accuracy in
`daily-call-prep`; (b) client-roster matching accuracy for catch-up
titles; (c) audit-lead play-selection precision (override rate +
which play the operator overrides TO); (d) `reengage-lead` reply
rate vs. cold-bump baseline; (e) `run-deliverability-test` stability
across the configured tool (Chrome MCP drops, code-rotation
gotchas, iframe-unresponsive-after-send patterns); (f) post-call
queue throughput (tasks landed per week, per-task processing time
stability as the field count grows).
