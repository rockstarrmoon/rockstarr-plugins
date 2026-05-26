# rockstarr-infra (developer notes)

This file is for Claude Code working on this plugin's source. For what
the plugin does, who runs it, and the operator-facing skill catalog,
read `README.md` in this folder.

## Why this plugin is different from the rest

It is the foundation. Every other Rockstarr plugin assumes it has
already run on the client's workspace. Every Rockstarr client install
includes it, no exceptions. Most other plugins read files this plugin
produced (`client-profile.md`, `style-guide.md`, `stack.md`), call
into helpers this plugin owns (`approve`, `publish-log`,
`send-notification`), or hand off to scheduled tasks this plugin
configured (`approvals-digest`, `approvals-backlog-alert`,
`notify-reply-ready`).

**Practical implication:** changes here have wider blast radius than
changes anywhere else in the monorepo. Treat schema changes, contract
changes, and scheduled-task changes as cross-plugin coordination work,
not local edits.

## Skill groupings (mental map)

The plugin ships ~20 skills. They sort into five buckets:

1. **Onboarding entry points** — `run-onboarding`, `run-intake`,
   `scaffold-client`. These are what clients invoke directly. Stateful;
   own `_progress.md`.
2. **Intake sub-skills** — `intake-background`, `intake-icp`,
   `intake-transformations`, `intake-competitors`,
   `intake-platinum-message`, `intake-offer`. Dispatched by
   `run-intake`. Each follows the unified intake voice (see
   `skills/_shared/references/intake-interviewer-voice.md`).
   Checkpoint-per-answer resume.
3. **Profile + voice + KB authoring** — `ingest-workbook` (legacy
   path), `compile-profile`, `generate-style-guide`, `kb-ingest`,
   `capture-stack`.
4. **Lifecycle primitives consumed by every bot** — `approve`,
   `publish-log`, `request-support`. These are called by other
   plugins as much as by humans.
5. **Notification layer** — `approvals-digest` (daily),
   `approvals-backlog-alert` (weekly), `notify-reply-ready` (event).
   All three route through `_shared/send-notification/` →
   `mail.rockstarr.ai`.

## Critical contracts this plugin owns

Several front-matter and folder-shape contracts originate here. Other
plugins read them. **Don't change them without coordinated PRs in the
consumers** (rockstarr-reply, both outreach variants, content, social,
ops).

- **Approvable-draft front-matter** — every draft a bot writes to
  `03_drafts/` carries `approval_status: pending`, `created_at`,
  `bot`, `lane`. `approvals-digest` and `approvals-backlog-alert`
  scan for these. Adding a new required field breaks every existing
  draft until backfilled.
- **`notify-reply-ready` bundle shape** — the urgent reply
  notification expects `lead_name`, `company`, `inbound_excerpt`,
  `classification`, `proposed_body`, `claude_uri`. Both outreach
  variants emit this; `notify-reply-ready` renders it. The bundle
  is the contract between outreach and infra.
- **`scaffold-client` folder contract** — the `/rockstarr-ai/` layout
  described in `README.md`'s "Folder contract" section. Any bot that
  reads or writes files in a client workspace assumes this exact
  shape. Adding a directory is safe; renaming or removing one is a
  breaking change that requires a deployment-model version bump and
  consumer updates across every plugin.
- **`client.toml` shape** — `[approvals]`, `[mailer]`, `[notifications]`
  sections. Read by the notification skills and by some bots. Adding
  optional fields with defaults is safe; renaming or removing is not.

## Shared assets — read-only for other plugins

Anything under `skills/_shared/` is consumed by other plugins via
direct path reference. They do not fork. If you change one, you change
it for everyone. Specifically:

- `skills/_shared/stop-slop/` — MIT-licensed. Every prose-producing
  plugin runs it. Behavior change here changes every plugin's output.
- `skills/_shared/send-notification/` — the only sanctioned path to
  send email from any Rockstarr plugin. Reads the mailer bearer from
  `.rockstarr-mailer.env` (seeded by `scaffold-client`). Bypassing it
  to call Resend directly from a plugin is a deployability risk —
  the mailer Worker provides domain alignment, branded templating,
  and reply-to routing that's central to inbox reputation.
- `skills/_shared/references/` — the canonical prompts and rubrics
  (style guide prompt, case-study prompt, intake-interviewer voice,
  TL rubric, blog SEO/GEO reference). Update in place. Never fork
  into a consumer plugin.

## Scheduled tasks `scaffold-client` wires up

When a new client installs this plugin, `scaffold-client` registers
two recurring scheduled tasks via Cowork's scheduled-tasks MCP:

- **Daily 6 am local** → `approvals-digest` (silent on empty days)
- **Monday 8 am local** → `approvals-backlog-alert` (silent under
  threshold)

`notify-reply-ready` is event-driven, not scheduled — it fires from
the outreach plugins' `detect-replies` runs or directly from
`rockstarr-reply:draft-reply`.

If you touch `scaffold-client`, verify these tasks still register
correctly. Re-running `scaffold-client` is supposed to be idempotent
— it should not double-register tasks.

## What's high-risk to change in this plugin

Some areas need extra care because they affect every client + every
plugin simultaneously:

- `scaffold-client` — folder contract, scheduled task registration.
  Errors here brick new client installs.
- The `_shared/` contents — see above.
- The notification skills' bundle shapes — see above.
- `client.toml` schema — read by many plugins.
- The intake-voice reference — every intake sub-skill follows it; a
  change here changes the tone of every onboarding interview.

These should land with explicit cross-plugin coordination notes in the
PR description, not as drive-by edits.

## What's safe to change without much ceremony

- New skill additions that don't break existing contracts.
- Documentation, comments, references in skill bodies.
- Internal logic of intake sub-skills that doesn't change the
  artifacts they write.
- New optional `client.toml` fields with sensible defaults.
- Wording of error messages, prompts, confirmations.

## Versioning

See `README.md`'s "Versioning" section for the history. The summary:
patch bumps for behavior fixes that don't change contracts; minor bumps
for new skills or new optional fields; major bumps for contract changes
that require consumers to update. We've been on `0.9.x` since intake
v0.9 landed and have not yet cut `1.0.0`.

When you bump, use the `/bump` slash command at the repo root —
`/bump rockstarr-infra <new-version>`.

## Testing your changes

There's no automated test harness for plugin behavior (the skills are
LLM-prompted, not unit-testable). The manual test approach:

1. Build the `.zip` locally via `scripts/package-plugin.sh
   rockstarr-infra`.
2. Sideload into your own Cowork install (Plugin Settings → install
   from `.zip`).
3. Run the affected skill against a fresh empty workspace folder
   (for `scaffold-client`) or against `RockstarrAI/references/intake-fixtures/`
   (for intake skills).
4. Inspect the produced files / sent notifications.

CI in this monorepo only checks skill-name uniqueness; it doesn't
run the skills.

## When you're stuck

Ask Jon in the PR. Most contract questions about this plugin route
through him.
