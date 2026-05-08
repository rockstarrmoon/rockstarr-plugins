# rockstarr-social

Rockstarr AI social plugin. Drafts daily / weekly LinkedIn (and
X / IG) short-form posts in the client's approved voice, builds
a balanced weekly batch, and exports to the client's scheduler
(Publer default; GrowthAmp planner on demand). Sits on top of
`rockstarr-infra` and hands every batch through the standard
`approve` / `publish-log` lifecycle. The plugin never publishes
directly — that's the scheduler's job.

This is the social pillar of the Rockstarr AI Growth Operating
System. It also absorbs three formerly-adjacent surfaces in
v0.1: `draft-polls` (moved out of `rockstarr-content`),
`invite-page-followers` (moved out of
`rockstarr-outreach-salesnav`), and a brand-new daily LinkedIn
comment-check workflow.

## At a glance

- **Daily / weekly drafting.** `draft-social` produces
  one-offs; `fill-week` builds the configured weekly batch by
  looping `draft-social` once per slot.
- **Config-driven cadence and mix.** Per-week post count and
  the promo / insight / case / engagement mix are set in
  `stack.md` (`social_posts_per_week`, `social_mix`). Cadence
  knobs live with the rest of the stack; voice rules stay in
  `style-guide.md`.
- **Manual scheduler export.** `publer-export` writes the
  Publer CSV to `05_published-staging/social/<YYYY-WW>.csv`
  after the operator runs it. The export is intentionally
  **not** auto-triggered on approval in V0.1 — the operator
  decides when the import lands. Revisit if the manual step
  proves friction.
- **Shared stop-slop pass on every prose output.** Voice
  shapes the draft; stop-slop strips AI tells. Structural
  artifacts (the weekly manifest, the run logs) are exempt.
- **LinkedIn polls (moved here from rockstarr-content v0.6).**
  `draft-polls` writes batched 10-poll sets per persona with
  hard char-limit enforcement, audience-altitude pre-check,
  and topic-diversity gates. The set is the approval unit.
- **LinkedIn page-follower invites (moved here from
  rockstarr-outreach-salesnav v0.1.6).**
  `invite-page-followers` runs once per credit cycle, gated by
  a strict identity check (profile-photo alt-text + admin
  redirect), and logs to `/05_published/social/page-invites/`.
- **Daily LinkedIn comment check (new in v0.1).**
  `li-comment-check` walks a configurable list of managed
  accounts, drafts in-voice replies, gates every send on
  per-comment operator approval, and (optionally) closes the
  matching ClickUp task on success.

## Skills

| Skill | Status | What it does |
|---|---|---|
| `draft-social` | NEW v0.1 | Single short-form post, optional A/B hook variants, one slot at a time. |
| `fill-week` | NEW v0.1 | Weekly batch — loops `draft-social` over the configured `social_mix`, writes a manifest at `03_drafts/social/week-<YYYY-WW>.md`. |
| `publer-export` | NEW v0.1 | Manual-trigger CSV export for one approved weekly batch. |
| `draft-polls` | MOVED from rockstarr-content v0.6 | Batched 10-poll sets per persona; structural lane with hard char limits + altitude check. |
| `invite-page-followers` | MOVED from rockstarr-outreach-salesnav v0.1.6 | Monthly LinkedIn page-follow invites with identity gate, cycle dedup, and run log. |
| `li-comment-check` | NEW v0.1 | Daily comment monitor + reply workflow across one or more managed accounts. Operator-gated sends. Optional ClickUp closure. |

Deferred for V1.1+:

- `social-export-ga` — GrowthAmp planner export. Build when a
  client's `stack.md.social_scheduler` first lights up
  `growthamp`.
- `draft-engagement` — one-off comment drafts (the spec
  explicitly defers this to V1.1; for now, manual handling or
  the per-comment drafting inside `li-comment-check` covers
  it).

## stack.md additions

The plugin reads these blocks from `/rockstarr-ai/00_intake/stack.md`:

```yaml
# Drafting + scheduler
social_scheduler: "publer"          # publer | growthamp | native
social_posts_per_week: 5
social_mix:
  promo: 1
  insight: 2
  case: 1
  engagement: 1
social_channels:
  linkedin: true
  x: false
  instagram: false
social_default_post_time: "09:00"   # 24h client-local
social_fill_week_cron: "0 9 * * 5"  # default Fri 09:00 client-local

publer_accounts:
  linkedin: "Jane Doe (LinkedIn)"   # exact label — must match Publer
  x: "@janedoe (X)"
  instagram: "@janedoe (IG)"

# Polls (carried over from rockstarr-content v0.6)
polls_cadence: "monthly"            # monthly | quarterly | on-demand | off

# Page-follower invites (carried over from rockstarr-outreach-salesnav v0.1.6)
page_invite_enabled: true
page_invite_target_url: "https://www.linkedin.com/company/<slug>/"
page_invite_company_id: "<numeric id>"
page_invite_admin_display_name: "<full name>"
page_invite_schedule_cron: "0 14 8-14 * 2"   # 2pm 2nd Tuesday default
page_invite_credit_target: 50

# Daily LI comment check (new in v0.1)
li_comment_check_enabled: true
li_comment_check_cron: "30 8 * * 1-5"        # 08:30 weekdays default
li_comment_clickup_enabled: false            # opt-in
li_comment_accounts:
  - name: "Phil Katz"
    slug: "philipkatz1"
    voice_notes: |
      Corporate, professional. Seasoned executive in his 70s.
      Measured, appreciative, authoritative.
    brand_kit_url: "..."
    clickup_task_id: "868fzfgrf"
  - name: "Rachel Minion"
    slug: "rachelminion"
    voice_notes: "Warm, direct, energetic but not over-the-top."
    clickup_task_id: "868fzfh1h"
```

`rockstarr-infra:capture-stack` will need a follow-up bump to
prompt for these keys interactively. Until that ships,
operators add the block to `stack.md` by hand.

## Folder contract additions

The plugin extends the canonical `/rockstarr-ai/` layout with:

```
03_drafts/social/                  # per-post + weekly manifests + poll sets
04_approved/social/                # post-approval landing
05_published-staging/social/       # Publer CSVs + sidecar manifests
05_published/social/page-invites/  # monthly invite-page-followers run logs
05_published/social/li-comments/   # daily li-comment-check run logs
```

`03_drafts/social/`, `04_approved/social/`, and the staging
directory are created by `rockstarr-infra:scaffold-client`.
Plugin skills create the run-log directories on first write if
they don't exist.

## Migration path — the two moved skills

### `draft-polls` (was: `rockstarr-content` v0.6)

`draft-polls` was originally added to `rockstarr-content` in
v0.6 because polls were the first short-form lane and a social
plugin didn't yet exist. With v0.1 of `rockstarr-social`, polls
have a proper home.

The skill is moved verbatim — same altitude check, same
char-limit enforcement, same `polls_cadence` enum. Existing
clients running `rockstarr-content` v0.6 should:

1. Install `rockstarr-social` v0.1.0 to pick up the new home.
2. Bump `rockstarr-content` to v0.7.0 (a follow-up release
   that drops the `draft-polls` skill folder and the related
   README sections).

Until `rockstarr-content` v0.7 ships, both plugins will carry a
`draft-polls` skill — Cowork's skill-name uniqueness check will
silently drop the second installer (per the workspace memory on
duplicate skill names). To avoid that, install
`rockstarr-content` v0.7 alongside `rockstarr-social` v0.1, or
hold the `rockstarr-social` install until the content bump is
ready.

The output path (`03_drafts/social/`) is unchanged, so already-
approved poll sets in `04_approved/social/` are still valid for
style continuity and topic dedup.

### `invite-page-followers` (was: `rockstarr-outreach-salesnav` v0.1.6)

`invite-page-followers` was originally bolted onto the
salesnav-outreach plugin because both surfaces talk to LinkedIn
through Chrome MCP. The audience-growth purpose is socially-
shaped, not outreach-shaped, so v0.1 of `rockstarr-social`
takes ownership.

Differences from the salesnav-resident version:

- The skill log path moves from
  `/05_published/outreach/page-invites/` to
  `/05_published/social/page-invites/`. Existing logs at the
  old path can be left as historical record or moved by hand.
- The `confirm-session` precondition is softened: the salesnav
  plugin's `confirm-session` still satisfies it when both
  plugins are installed; if only `rockstarr-social` is
  installed, the skill's own Step 1 identity gate is the only
  check.
- The "do not touch the campaign workbook" rule is now
  trivially honored — this plugin doesn't carry that
  workbook.

Existing clients should:

1. Install `rockstarr-social` v0.1.0.
2. Bump `rockstarr-outreach-salesnav` to v0.2.0 (a follow-up
   release that drops the `invite-page-followers` skill
   folder).

Same name-collision caveat as `draft-polls`: until the salesnav
bump ships, run them in series (drop salesnav first, then
install social) or you'll lose one of the two skills silently.

## Approval lifecycle

Every drafting skill in this plugin lands files in
`03_drafts/social/`. The operator (or strategist) reviews, then
runs `rockstarr-infra:approve` against:

- A per-post file → promoted to `04_approved/social/`.
- A weekly batch manifest → manifest + per-post files all
  promoted.
- A poll-set file → promoted as one batch.

After approval, the operator runs `publer-export` (or
`social-export-ga` once that ships) to produce the scheduler
import file. The CSV stays in `05_published-staging/` until the
operator drops it into Publer; the actual publish is logged via
`rockstarr-infra:publish-log` once the post fires.

`li-comment-check` and `invite-page-followers` are exceptions
to the draft → approve → publish flow:

- `li-comment-check` operates on already-published posts, with
  per-comment approval gates inside the skill itself.
- `invite-page-followers` is a one-step action with an
  identity gate; there's no draft to review.

Both write run logs to `05_published/social/...` directly, not
through `publish-log`.

## Build sequence

Per the spec — adjusted for what shipped:

1. ✅ Plugin skeleton.
2. ✅ `draft-social`.
3. ✅ `fill-week`.
4. ✅ `publer-export`.
5. ✅ `draft-polls` (moved).
6. ✅ `invite-page-followers` (moved).
7. ✅ `li-comment-check` (new).
8. → Ship v0.1.0; install in the Rockstarr internal pilot.
9. → Bump `rockstarr-content` v0.7 to drop `draft-polls`.
10. → Bump `rockstarr-outreach-salesnav` v0.2 to drop
    `invite-page-followers`.
11. → Bump `rockstarr-infra` capture-stack to prompt for the
    new social config keys.
12. → Build `social-export-ga` when the first GrowthAmp client
    arrives.
13. → Build `draft-engagement` for V1.1 if `li-comment-check`'s
    per-comment drafting doesn't fully cover the need.

## Open questions, V0.1

These are deliberately deferred — answer when a real client
forces the call:

- Should `fill-week` honor proposed publish *times* per slot
  (Mon 9am, Tue 7am, Wed 12pm) or stay date-only and let
  Publer's "Best time" feature handle the hour? V0.1 uses
  `social_default_post_time` for every row — flat.
- Should `publer-export` auto-trigger on approval? V0.1 says
  no; revisit if the manual step proves friction.
- Should `li-comment-check` support task systems other than
  ClickUp? The skill is shaped to add Asana / Linear / Notion
  variants by extending `stack.md` and the close step;
  cross-system support waits for a real client need.
- Should the polls cadence enum extend to `bi-weekly` or stay
  monthly / quarterly / on-demand / off? Carried over
  unchanged from rockstarr-content v0.6.

## Dependencies

- `rockstarr-infra` v0.8.2+ — required for the shared
  stop-slop skill, `approve`, `publish-log`,
  `scaffold-client`, `kb-ingest`, and `generate-style-guide`.
- Chrome MCP — required for `invite-page-followers` and
  `li-comment-check`. The plugin refuses these skills cleanly
  if the Chrome extension isn't connected.
- `rockstarr-content` v0.7+ — install in series with this
  plugin to avoid the `draft-polls` name collision (see
  Migration path above).
- `rockstarr-outreach-salesnav` v0.2+ — install in series to
  avoid the `invite-page-followers` name collision.

## Changelog

### v0.1.0 — 2026-05-08

- Initial release.
- Ships `draft-social`, `fill-week`, `publer-export` (per the
  rockstarr-social-spec.docx V1 scope).
- Absorbs `draft-polls` from rockstarr-content v0.6.
- Absorbs `invite-page-followers` from rockstarr-outreach-salesnav
  v0.1.6.
- Adds `li-comment-check` (merged from the BBN-internal Daily
  LI Comment Check SOP and the canonical SKILL.md drafted
  alongside it; generalized to read managed accounts from
  `stack.md.li_comment_accounts[]` and made ClickUp-closure
  opt-in).
