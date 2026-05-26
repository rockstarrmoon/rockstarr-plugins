# rockstarr-reply (developer notes)

This file is for Claude Code working on this plugin's source. For the
full handoff interface spec, the front-matter contract, and the design
decisions behind every draft pattern, read `README.md` in this folder.
That README is the canonical reference — this CLAUDE.md is its
developer-perspective complement.

## Why this plugin is different from the rest

It is **channel-agnostic**. It does not touch a LinkedIn inbox, a
Gmail thread, or a Sales Nav DM directly. It receives a structured
handoff bundle from a *caller* (currently `rockstarr-outreach-salesnav`
or `rockstarr-outreach-interceptly`; future: email variants), produces
a staged draft + a structured response, and hands control back to the
caller to execute channel-side work.

That clean separation is the whole point. Adding a new channel means
shipping a new caller — outreach-meetalfred, outreach-dripify,
reply-gmail — without modifying this plugin at all. The handoff
interface is the contract.

**Practical implication:** the bundle shape and the staged-draft
front-matter shape are *contracts*, not implementation details. They
are read by callers, by `rockstarr-infra:notify-reply-ready`, and by
`rockstarr-infra:approvals-digest`. Treat changes to either shape as
cross-plugin coordination work.

## Skill groupings (mental map)

Five skills, all forming one tight pipeline plus one escape hatch:

**The pipeline (in order):**

1. `classify-reply` — bucket the inbound (Hot / Warm-ICP /
   Warm-non-ICP / Cold / Out-of-office / etc.) using thread context.
2. `draft-reply` — produce the draft (or three options on
   Warm-non-ICP) using the bucket + caller's persona + style guide.
   Runs stop-slop. Stages to `03_drafts/replies/<channel>/`.
3. `present-for-approval` — surface the staged draft to the operator,
   handle approve / edit / let-it-hang / skip / flag.
4. `follow-up-timer` — compute the next follow-up timer keyword
   (returned as part of `authorized-send`).

**The escape hatch:**

5. `flag-for-review` — when classify-reply can't bucket confidently or
   draft-reply can't produce a safe draft (hostile reply, missing
   context, unknown signal), return a `flag` response. Caller labels
   the thread `Follow Up` and creates a 2-business-day review task.

## Contracts this plugin owns

These two interfaces are the primary thing this plugin is *responsible
for*. Everything else is implementation.

### The handoff interface (caller ↔ this plugin)

Both directions are spec'd in `README.md` under "The handoff
interface." Summary:

- **Caller → this plugin** sends a bundle with `channel`, `thread`,
  `persona`, `icp_verdict`, `lead`, and optionally `intent_hint`
  and `batch_context`.
- **This plugin → caller** returns one of: `authorized-send`,
  `three-option-choice`, `flag`, `book-meeting-handoff`, or
  `no-action`. Each carries the payload the caller needs to do its
  channel-side work.

**Changes here = breaking changes.** Bumping the response shape is a
major version. Adding a new optional input field is a minor.
Renaming a field is a major.

### The front-matter contract (staged draft → infra)

Every draft `draft-reply` writes to `03_drafts/replies/` carries a
specific front-matter shape that other plugins read. Two layers:

- **Cross-bot layer** (read by `rockstarr-infra:notify-reply-ready`
  and `approvals-digest`): `channel: "reply"`, `source_channel`,
  `source_channel_slug`, `title`, `produced_by`, `approval_status`,
  `awaiting_approval_since`, `path_relative`, `inbound_excerpt`,
  `draft_body` / `draft_options`, `stop_slop_score`,
  `stop_slop_flagged`, `bucket`, `sub_types`, `proposed_label`,
  `proposed_followup_timer`, `lead_*` fields.
- **Reply-specific layer** (read only by this plugin's own skills):
  `thread_id`, `campaign_slug`, `persona_*`, `icp_verdict`,
  `matching_rule`, `pattern`, `warm_reply_pattern_missing`,
  `generated_at`, `last_revised_at`, `revision_count`,
  `schema_version`.

The cross-bot layer is the thing that triggers cross-plugin
coordination if you change it. The reply-specific layer is internal
to this plugin and is safe to evolve.

## The notify-reply-ready integration

`rockstarr-infra:notify-reply-ready` is fired from one of two places,
**never both for the same draft**:

- **Synchronous:** when the operator manually invokes a single reply
  draft, `draft-reply` fires `notify-reply-ready` itself with
  `staged_paths=[<the one path>]` after staging. Bundle has no
  `batch_context`.
- **Batched:** when an outreach `detect-replies` dispatches N inbound
  replies in a loop, the caller sets `batch_context=detect-replies`
  in the handoff. `draft-reply` then *skips* the notification. The
  caller accumulates staged paths and fires `notify-reply-ready`
  once at the end with the full list.

If you touch `draft-reply` or this contract, verify both paths still
work — the most common failure mode is double-firing
(`draft-reply` notifies AND the caller notifies) or zero-firing
(both think the other is doing it).

## Channels currently supported

- `linkedin-interceptly` (caller: `rockstarr-outreach-interceptly`)
- `linkedin-salesnav` (caller: `rockstarr-outreach-salesnav`)
- `linkedin-meetalfred`, `linkedin-dripify`, `linkedin-waalaxy` —
  reserved enum values; no caller exists yet.
- `email-gmail`, `email-outlook` — reserved enum values; no caller
  exists yet.

Each channel slug controls a small channel-adaptation switch inside
`draft-reply` (LinkedIn DMs stay short and skip the signature block;
email may include a subject line and appends persona signature).
Adding a new channel = adding a value to the enum + adding the
adaptation case + a caller plugin. This plugin's pipeline doesn't
care which channel is in use beyond that switch.

## What's high-risk to change in this plugin

- **The handoff bundle shape (in or out).** Every caller depends
  on it.
- **The front-matter contract's cross-bot layer.** notify-reply-ready
  and approvals-digest read it.
- **The five draft patterns** (Hot meeting ask, Warm-ICP standard
  reply, Warm-non-ICP three options, Cold bump/breakup, Out-of-office
  no-action). Pattern logic is consulted by every channel.
- **The `produced_by` versioning string.** Some consumers may key off
  it for compatibility checks.

These should land with explicit cross-plugin coordination notes in the
PR description.

## What's safe to change without much ceremony

- Internal drafting heuristics within an existing pattern (tone,
  word choice, structural variations) as long as the output still
  passes stop-slop.
- `classify-reply`'s bucketing logic, as long as the bucket enum
  doesn't change.
- The stop-slop score thresholds (currently `<35` flags).
- Wording of operator-facing prompts in `present-for-approval`.
- Internal-only front-matter fields (the reply-specific layer).

## Versioning

The handoff response shape is the major-version axis. Currently on
`0.2.0`. Bumping the response shape is `1.0.0` territory.

Bump via `/bump rockstarr-reply <new-version>` at the repo root.

## Testing your changes

There's no automated test harness. The manual approach for this
plugin specifically:

1. Build the `.zip` via `scripts/package-plugin.sh rockstarr-reply`.
2. Sideload into your own Cowork install.
3. Invoke directly with a synthetic bundle — easiest path is to
   run from inside an outreach plugin's `detect-replies` test loop,
   but you can also call `draft-reply` standalone by handing it a
   bundle dictionary.
4. Inspect the staged draft's front-matter and body.
5. Verify the returned response shape matches what callers expect
   for the relevant bucket.

CI in this monorepo only checks skill-name uniqueness.

## When you're stuck

Ask Jon in the PR. Most contract questions about this plugin route
through him.
