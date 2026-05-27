# rockstarr-social (developer notes)

This file is for Claude Code working on this plugin's source. For the
full skill catalog, configuration keys, folder contract additions, and
migration history, read `README.md` in this folder. This CLAUDE.md is
its developer-perspective complement.

## Why this plugin is different from the rest

It is a **hybrid plugin** — half content factory (like
`rockstarr-content`), half Chrome-MCP-driving automation (like the
outreach plugins). That mix is unusual in this monorepo. Practically:

- The drafting skills (`draft-social`, `fill-week`, `draft-polls`) are
  pure file producers — read profile + style guide + KB, write to
  `03_drafts/social/`. Failure modes are quality-of-output, same as
  the content plugin.
- The action skills (`invite-page-followers`, `li-comment-check`,
  `linkedin-profile-audit`) drive LinkedIn through Chrome MCP.
  Failure modes are operational — wrong account, broken UI, send
  without approval — same as the outreach plugins.

When working on this plugin, mentally categorize each skill into one
of those two buckets before reasoning about it. The right test
approach and the right risk profile depend on which bucket the skill
is in.

## Skill groupings (mental map)

7 active skills sort into four groups (plus two deferred):

1. **Pure drafting** — `draft-social` (single short-form post),
   `fill-week` (weekly batch via `draft-social` loop),
   `draft-polls` (batched 10-poll sets per persona).
2. **Scheduler export** — `publer-export` (manual-trigger Publer
   CSV for one approved weekly batch).
3. **LinkedIn admin actions** — `invite-page-followers` (monthly
   credit-cycle page-follow invites with identity gate),
   `li-comment-check` (daily comment monitor + reply workflow,
   per-comment approval gate).
4. **One-off interview-driven deliverables** — `linkedin-profile-audit`
   (eight-item playbook + two inline interviews, produces a
   workspace-markdown deliverable for personal profiles only).

Deferred for V1.1+: `social-export-ga` (GrowthAmp planner export
when a client lights that up), `draft-engagement` (one-off comment
drafts; for now `li-comment-check`'s per-comment drafting covers
this).

## Migration history — read before touching draft-polls or invite-page-followers

Two skills migrated into this plugin from elsewhere when it was
created in v0.1.0. This matters for cross-plugin coordination:

- **`draft-polls`** was in `rockstarr-content` v0.6. Moved here in
  the `rockstarr-content` v0.7 / `rockstarr-social` v0.1
  coordinated release. Workspace conventions (the `polls_cadence`
  enum in `stack.md`, the "Channel Adaptation → LinkedIn polls"
  subsection of `style-guide.md`) are **unchanged** — only the
  implementing skill moved plugins. The altitude check
  (audience-altitude pre-check, the #1 failure mode for the lane)
  came over verbatim.
- **`invite-page-followers`** was in `rockstarr-outreach-salesnav`
  v0.1.6. Moved here in the salesnav v0.2.0 / `rockstarr-social`
  v0.1 coordinated release. Audience-growth is socially shaped,
  not outreach shaped; lives here now. Log path changed from
  `/05_published/outreach/page-invites/` to
  `/05_published/social/page-invites/`. The salesnav
  `confirm-session` precondition is softened — the skill's own
  Step 1 identity gate (profile-photo alt-text + admin redirect)
  is the only required check when this plugin is installed
  standalone.

If you find yourself debugging either skill, the README's
"Migration path" section has the full story including the
name-collision caveat (Cowork silently drops the second
installer when two installed plugins both carry the same skill
name — so the moves had to be coordinated).

## Outstanding backlog item — Publer label fetch

`rockstarr-infra:capture-stack` currently writes the
`publer_accounts.*` fields as null placeholders during intake.
The original plan was to populate them by hand via dashboard
inspection; the better plan is for this plugin to add a Chrome-MCP
fetch routine that pulls the labels automatically from Publer's
account dropdown on first run.

This isn't implemented yet. If you're touching `publer-export` or
adding a new Publer skill, consider closing this gap. The
workspace memory note tagged `publer_label_fetch_deferred`
carries the design intent.

## Cadence + mix — config-driven, lane-gated

This plugin reads two cadence knobs from `stack.md`:

- `social_posts_per_week` — integer; how many slots `fill-week`
  produces in a weekly batch.
- `social_mix` — distribution across post types (promo /
  insight / case / engagement). `fill-week` uses this to decide
  which approved long-form pieces or KB anecdotes to surface per
  slot.

The `polls_cadence` enum (separate from `social_posts_per_week`)
gates `draft-polls`. Cadence 0 / off / disabled means the skill
refuses to emit — same pattern as the content lanes.

Voice rules (per-persona LinkedIn polls subsection, banned
phrases, channel-specific adaptations) stay in `style-guide.md`,
not here. This plugin reads style guide; doesn't author it.

## Chrome MCP — same rules as the outreach plugins

For the three skills that touch LinkedIn directly:

- `invite-page-followers` has a strict identity gate — verifies
  profile-photo alt-text matches the configured admin display
  name AND that the admin redirect succeeded — before
  enumerating invitable connections. Wrong-admin invites are a
  reputational risk on the client's company page.
- `li-comment-check` runs across one or more managed accounts
  per `stack.md.li_comment_accounts[]`; for each it verifies the
  correct account is signed in via the same profile-photo
  alt-text gate, then walks notifications + recent activity,
  drafts in-voice responses, **gates every reply on operator
  approval per-comment** before posting. No batch send. Closes
  matching ClickUp tasks on success when ClickUp closure is
  enabled in `stack.md`.
- `linkedin-profile-audit` is **personal profiles only**. Never
  runs against a company page. Produces a workspace-markdown
  deliverable; never shares externally without an explicit
  "send it" from the operator.

If you're modifying any of these, the same UI-quirk discipline
the outreach plugins use applies — explicit recovery for known
LinkedIn UI weirdness, no sleeps as cover for race conditions.

## Mandatory passes

Drafting skills (`draft-social`, `fill-week`, `draft-polls`,
`li-comment-check`'s reply drafts, `linkedin-profile-audit`'s
headline + About prose) run the shared stop-slop pass as the
final step before write. Same pattern as `rockstarr-content`.

Structural artifacts — the weekly manifest, the page-invite run
log, the comment-check run log, the poll set's wrapper — are
exempt from stop-slop by design.

`draft-polls` additionally enforces the **audience-altitude check**
before any prose runs. This is the #1 quality gate for the polls
lane and is what makes the difference between polls that read as
strategic and polls that read as quizzes. Don't bypass.

## Publer export — manual trigger, intentionally

`publer-export` is **manual-trigger only**. It doesn't fire on
approval. The design choice: the operator decides when the import
lands in Publer, not the bot.

If you find yourself wanting to auto-trigger it on approval, get
operator sign-off first. The README marks this as "revisit if
the manual step proves friction" — V0.1 hasn't surfaced that
friction yet.

## What's high-risk to change in this plugin

- **The identity gates on `invite-page-followers` and
  `li-comment-check`.** Wrong-account or wrong-page actions are
  reputational incidents.
- **The per-comment approval gate in `li-comment-check`.** Batch
  send is a non-starter.
- **The "personal profiles only" rule in
  `linkedin-profile-audit`.** Running against a company page
  produces a deliverable that doesn't apply to the eight-item
  playbook the skill is built around.
- **The cadence-gate logic.** Drafts in lanes a client doesn't
  publish surface as noise.
- **The `stack.md` social config schema.**
  `social_posts_per_week`, `social_mix`, `polls_cadence`,
  `li_comment_accounts[]` — read by multiple skills.
- **The draft front-matter contract.** Shared with `rockstarr-content`
  and read by `rockstarr-infra:approvals-digest`.

## What's safe to change without much ceremony

- Internal logic of any single drafting skill (slot picking,
  post-type heuristics, A/B variant generation) as long as the
  pass order and the front-matter shape are preserved.
- Selectors and waits inside Chrome-MCP skills.
- Run-log formats under `05_published/social/`.
- New optional `stack.md` fields with sensible defaults.
- The Publer CSV column shape (within what Publer accepts).

## Versioning

This plugin is the newest in the monorepo (v0.1 in May 2026).
Major version axis: the cadence-gate schema, the draft
front-matter contract, removal of any of the three Chrome-MCP
skills. Minor: new skills, new optional config, new lanes.

Bump via `/bump rockstarr-social <new-version>` at the repo root.

## Testing your changes

Because this plugin is hybrid, the test approach is per-skill:

- **Drafting skills** — sideload, run against a real test client
  workspace with profile + style guide + KB. Read the output as
  a human. Verify the front-matter and the stop-slop score.
- **Chrome-MCP skills** — sideload, run against a real test
  LinkedIn account / page. The identity gate is the part you
  most need to confirm survives changes.
- **Publer export** — pure file output; verify the CSV against
  Publer's import requirements by spot-importing into a Publer
  workspace.

CI in this monorepo only checks skill-name uniqueness.

## When you're stuck

Ask Jon in the PR. The hybrid nature means cross-cutting
questions (Chrome MCP + drafting in one skill, like
`li-comment-check`) most reliably route through him.
