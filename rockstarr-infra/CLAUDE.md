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

## Defer expensive preconditions

The "is it even working?" perception problem (every client asks
this during their first week) has two contributors plugins can
address:

1. **Skill descriptions** trimmed and on-point so routing
   resolves fast. Covered by the description-cap rule and
   `client-facing-output-voice.md` rule 7 (anchor message).
2. **Precondition checks** that don't burn seconds reading
   files before deciding whether the skill can even run.

The second one is what this section is about. Several skills
have historically opened with multi-file reads to "validate
preconditions" — the audit script flagged
`compile-profile` (reads 6 intake artifacts),
`generate-style-guide` (reads `01_knowledge_base/index.md` plus
every first-party processed file),
`draft-icp-campaign` (reads `client-profile.md` + lead-list +
style-guide + KB), and others. Those reads fire on every
invocation, even when the skill ultimately refuses (a downstream
key is missing, the user is on the wrong path, etc.). The user
sees nothing for several seconds and assumes the bot is broken.

### The pattern

Split preconditions into two tiers:

- **Tier 1: cheap existence checks.** Does this file exist?
  Does this directory exist? Is this key set in `client.toml`?
  These are sub-second filesystem stats and small text reads.
  Run them first, before the anchor message. If any fail,
  refuse fast with a clear remediation pointer — the user sees
  the refusal without the bot looking dead.
- **Tier 2: expensive content reads.** Reading a 2KB markdown
  artifact and parsing front-matter, walking a directory tree
  to count files, validating cross-references between
  artifacts. These belong inside the skill's main work, AFTER
  the anchor message has fired. The user sees "I'm starting"
  and then the work happens.

Pseudo-code for a long-running skill:

```
def run():
    # Tier 1: cheap existence checks. Silent. ~10ms total.
    if not exists("00_intake/client-profile.md"):
        refuse("Your business profile isn't captured yet. Run
                onboarding first to set it up.")
        return
    if not exists("00_intake/intake/icp.md"):
        ...

    # Anchor message. Fires after cheap checks, before the work.
    print_anchor("Stitching your business profile together…")

    # Tier 2: expensive content reads. Happens as part of the
    # work, not as gatekeeping.
    artifacts = read_all_intake_artifacts()  # ~2-3s
    validate_internal_shape(artifacts)
    ...
```

### Skills that should adopt the pattern

The audit's specifically-named-as-heavy list:

- **`compile-profile`** — reads 6 intake artifacts upfront. The
  existence-check on each is cheap; the content reads can defer
  to step 1 of the main assembly.
- **`generate-style-guide`** — reads `01_knowledge_base/index.md`
  plus every first-party processed file. The KB index is a small
  read but the file walk is heavy. Defer the walk to the
  pre-read step where the skill actually uses the content.
- **`draft-icp-campaign`** (in `rockstarr-outreach-salesnav`) —
  reads `client-profile.md`, `lead-list/<slug>.md`, style-guide,
  and KB during precondition validation. All four can move to
  the start of Phase 1 where they're actually used.
- **`approvals-digest`** — walks `/03_drafts/` recursively to
  find pending items. This IS the skill's work, not a precondition,
  but the anchor message should fire before the walk so the
  user sees activity immediately on a user-invoked run.

### What the cheap checks should still cover

The Tier 1 checks are still load-bearing — they're the safety net
that prevents the skill from doing partial work and leaving the
workspace in a weird state. Keep them strict:

- File existence (use `os.path.exists`, not `read`).
- Required config keys (parse stack.md / client.toml front-matter
  only — the small front-matter block, not the full body).
- Plugin / shared-reference availability (does
  `_shared/stop-slop/` exist?).
- Authorization signals (is the user authorized to act on this
  client's behalf?).

These are sub-second. Defer everything that requires reading a
multi-paragraph artifact, walking a tree, or parsing structured
content.

### Don't defer the anchor message past expensive checks

If a precondition check is genuinely expensive (e.g., a Chrome
MCP session probe in `confirm-session`), the anchor message
fires BEFORE the probe, not after. The user should see "Checking
your LinkedIn session…" rather than 10 seconds of silence
followed by either the probe result or a refusal. Per
`client-facing-output-voice.md` rule 7, the anchor message is
the first user-facing line of the skill, full stop.

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
