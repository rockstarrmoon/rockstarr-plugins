# rockstarr-outreach-interceptly (developer notes)

This file is for Claude Code working on this plugin's source. For the
full skill catalog, hard constraints, configuration keys, and design
decisions, read `README.md` in this folder. This CLAUDE.md is its
developer-perspective complement.

## Why this plugin is different from the rest

It is **Chrome-MCP heavy**. Nearly every skill in this plugin drives
the Interceptly web app through the browser — clicking, typing,
waiting for popups, recovering from UI bugs. Three properties follow
from that:

1. **Brittleness budget is non-zero.** Interceptly's UI changes; selectors break; popups dismiss out of order. The plugin carries explicit recovery code for known quirks (see "Known Interceptly UI quirks" in the README). Treat new flakiness as a real-bug-fixed-in-code, not a test that needs re-running.
2. **Session safety is paramount.** This bot drives a real LinkedIn account through Interceptly. A wrong-account send is a reputational incident. `confirm-session-interceptly` is the gate — it runs before *every* per-account pass and after every `switch-account`. Non-negotiable.
3. **It is the default outreach variant.** Other variants (`-salesnav`, future `-meetalfred`, `-dripify`, `-waalaxy`) share much of this plugin's shape. Pattern decisions made here propagate.

## Hard constraints (from README; surfacing here for emphasis)

These are non-negotiable. If you find yourself working around one,
stop and ask:

1. All Interceptly UI actions go through Chrome MCP. Never bypass Interceptly to go straight at LinkedIn.
2. `confirm-session-interceptly` runs before any per-account pass and after every `switch-account`.
3. The bot never presses Start on a campaign. `launch-campaign-interceptly` stops at CONFIGURED-NOT-STARTED.
4. No outbound reply goes out without operator approval. `send-message` only runs on an `authorized-send` bundle from `rockstarr-reply:present-for-approval`.
5. Third-party content is reference-only. Never paraphrase third-party as if the client said it.

## Skill groupings (mental map)

24 skills sort into six buckets:

1. **One-time setup** — `discover-interceptly-accounts`,
   `capture-interceptly-personas`, `capture-icp-qualifications`,
   `capture-warm-reply-pattern`. Run during onboarding to populate
   the configuration files this plugin reads.
2. **Campaign lifecycle** — `draft-icp-campaign-interceptly`,
   `launch-campaign-interceptly`, `stop-campaign-interceptly`.
3. **Daily loop orchestration** — `daily-loop`, `switch-account`,
   `confirm-session-interceptly`, `preview-queue-interceptly`,
   `process-inbox`, `process-my-tasks`. These are how the bot
   walks accounts and processes work in either mode.
4. **Per-thread pipeline** — `qualify-lead`, `send-message`,
   `apply-label`, `create-followup-task`. The channel-side work
   that follows a `rockstarr-reply:authorized-send` bundle.
5. **Meeting & booking** — `propose-meeting-times-interceptly`,
   `book-meeting-interceptly`, `mark-booked-interceptly`.
6. **Metrics & reporting** — `metrics-daily-interceptly`,
   `metrics-weekly-interceptly`, `outreach-weekly-report-interceptly`,
   `backup-workbook-interceptly`. End-of-day and end-of-week
   rollups.

Note the `-interceptly` suffix on most skill names. This is the
collision-prevention convention — every variant-specific skill is
suffixed to avoid colliding with the same-named skill in
`rockstarr-outreach-salesnav`. (See the root CLAUDE.md's "Skill
description rules" for context.) Internal helper skills shared by
both variants (`qualify-lead`, `send-message`, `apply-label`,
`create-followup-task`) intentionally stay unsuffixed to make the
shared pattern obvious — they live in only ONE outreach variant
each in practice (Interceptly's `send-message` operates an
Interceptly composer; salesnav's a Sales Nav composer). If you add
a new skill that doesn't have a counterpart in the other variant,
think about whether it should be suffixed too.

## Foreground vs. background mode

`daily-loop`, `process-inbox`, and `process-my-tasks` accept a `mode`
parameter (`foreground` default; `background` for scheduled runs).
Both modes share the per-account work; they differ in how they treat
approval-gated outcomes:

- **Foreground:** operator is in chat. `present-for-approval` runs
  inline. The operator says "send it" and channel-side work
  executes immediately.
- **Background:** scheduled run. Nobody to present to. Pipeline
  stages drafts to `03_drafts/replies/` and stops. Deterministic
  non-draft branches (label-only, flag, book-meeting on prior-
  authorized substance) still execute. At end of run, `daily-loop`
  fires `rockstarr-infra:notify-reply-ready` once with the global
  accumulator — one urgent email per run, never per account.

If you touch any of those three skills, verify both modes still
behave correctly. The most common failure mode is calling
`present-for-approval` in background mode (which has no operator)
or failing to accumulate `staged_paths` so the end-of-run
notification fires empty.

## Caller side of the rockstarr-reply contract

This plugin is a caller of `rockstarr-reply`. The handoff contract
is owned by that plugin (see `plugins/rockstarr-reply/CLAUDE.md`),
but this plugin's responsibilities as a caller:

- Send a rich handoff bundle. `lead.*`, `thread.messages[]`, and
  `source_channel_label` all feed downstream into the front-matter
  `notify-reply-ready` reads. A thin bundle becomes "(not captured)"
  in the urgent email.
- Set `batch_context: detect-replies` (or any non-empty string) on
  the handoff when in a batched daily-loop run. `draft-reply` then
  skips its own notification and lets `daily-loop` fire once at the
  end with the accumulated paths.
- Don't omit `batch_context` in background mode — that path expects
  batched notification. Conversely, *do* omit it on synchronous
  single-thread invocations so `draft-reply` notifies immediately.
- Execute the channel-side work returned in the response payload:
  `authorized-send` → `send-message` + `apply-label` +
  `create-followup-task`; `flag` → `apply-label('Follow Up')` +
  task; `book-meeting-handoff` → `book-meeting-interceptly`;
  `no-action` → log and move on.

## The mirror — audit, not source-of-truth

`/rockstarr-ai/02_inputs/outreach/outreach-mirror.xlsx` is this
plugin's audit log, not its source of truth. Interceptly is the
source of truth. The mirror records what the bot did so we can roll
up metrics and reconcile against the platform. **The bot reads
state from Interceptly via Chrome MCP, never from the mirror.**

This plugin owns every sheet in the workbook. `rockstarr-reply`
never touches it. The one cross-plugin read this plugin does on
the reply side is `/02_inputs/replies/_flags.md` (written by
`rockstarr-reply:flag-for-review`) for flag counts in the
preview-queue and metrics skills. Read-only.

## Chrome MCP UI quirks — explicit handling required

Six known quirks have explicit recovery code in the relevant
skills. Listed in the README's "Known Interceptly UI quirks." If
you're writing or modifying a Chrome-MCP-driving skill, scan that
list and confirm your skill handles the relevant ones:

- React-textarea setter (composer)
- SWITCH ACCOUNT popup timing
- Virtualized industry dropdown
- Labels-nav-bug recovery
- My Tasks empty-state race
- Same-day date dropdown labelling

Don't paper over these with sleeps. The recovery patterns in the
existing skills are the right templates.

## What's high-risk to change in this plugin

- **`confirm-session-interceptly`.** Wrong-account sends are the
  top reputational risk in the entire system. Don't weaken this
  check.
- **The daily-loop background/foreground branching.** Notification
  double-fire or zero-fire is the failure mode.
- **The mirror schema.** Metrics + reporting skills read it; the
  weekly report depends on its column shape.
- **The handoff bundle this plugin sends to `rockstarr-reply`.**
  Anything thin upstream becomes "(not captured)" in the urgent
  email downstream.
- **Anything that touches `outreach_accounts[]` order or
  rotation** — `daily-loop` iterates in that order; some clients
  rely on it for compliance.

## What's safe to change without much ceremony

- Selectors and waits inside Chrome-MCP skills (these are
  inherently brittle; refactoring is expected).
- Per-skill prompts and operator-facing messages.
- Metric calculations within the existing column schema.
- Recovery logic for newly-discovered UI quirks (add, don't
  rewrite existing ones).
- Internal logic of qualify-lead's ICP-rule evaluation, as long
  as the verdict enum (`target` / `not-target` / `ambiguous` /
  `unknown`) stays unchanged.

## Versioning

Major version axis: the contracts other plugins read (mirror
schema, handoff bundle going to rockstarr-reply). Minor: new
skills, new optional config in `stack.md`. Patch: bug fixes,
quirk recovery additions.

Bump via `/bump rockstarr-outreach-interceptly <new-version>` at
the repo root.

## Testing your changes

This one is harder than the other plugins because you need a real
Interceptly account to drive. Three approaches in order of
fidelity:

1. **Full sideload + drive.** Build the `.zip`, sideload into your
   own Cowork, run the relevant skill against a test Interceptly
   account. Highest fidelity, slowest, requires you to have an
   account configured.
2. **Unit-style dry run.** For non-Chrome-MCP skills (campaign
   drafting, metrics rollup, mirror writes), invoke the skill
   against a fixture client folder under
   `RockstarrAI/references/` (or wherever you keep test fixtures).
3. **Read-only inspection.** For the Chrome-MCP skills, manually
   trace the skill's body against the current Interceptly UI by
   opening Interceptly in a browser and confirming the selectors
   the skill targets still exist. This catches the most common
   bug class — UI drift — without needing a full run.

CI in this monorepo only checks skill-name uniqueness.

## When you're stuck

Ask Jon in the PR. UI-quirk questions especially — he's seen most
of the Interceptly weirdness firsthand.
