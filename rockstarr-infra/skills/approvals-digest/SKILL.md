---
name: approvals-digest
description: |
  This skill should be used when the scheduled daily run fires at 6 am local time, or when the user says "send the approvals digest", "what's waiting on me", "email me my approval queue", or "run the digest now". Scans every drafting bot's output folder under /rockstarr-ai/03_drafts/ for files with approval_status: pending front-matter, sorts most-recent first by file mtime, and emails one daily summary via the send-notification helper. Exits silently with no send if nothing is pending — the digest is a no-op on quiet days. Cross-bot by design: surfaces pending items from rockstarr-content, rockstarr-reply, rockstarr-outreach-* and any future drafting bot without bot-specific code.
---

# approvals-digest

One daily email summarizing every draft awaiting client approval
across every Rockstarr AI bot. Lives at the infra layer so individual
bots don't each ship their own notifier.

The contract is simple: every drafting bot writes its drafts under
`/rockstarr-ai/03_drafts/<channel>/` with `approval_status: pending`
and `awaiting_approval_since: <ISO>` in front-matter. This skill
reads that contract and assembles the digest. No bot-specific
adapters required.

## When to run

- **Scheduled.** Daily at 6:00 am local time via the Cowork
  scheduler. Cron: `0 6 * * *`. Schedule is created once per client
  during scaffold-client; this skill is the body the scheduler
  invokes.
- **On demand.** User says "send the approvals digest", "email me
  my approval queue", "run the digest now", "what's waiting on me".
  Same logic, same payload, just synchronous.

## Preconditions

- `/rockstarr-ai/00_intake/.rockstarr-mailer.env` exists with
  `ROCKSTARR_MAILER_TOKEN`, `ROCKSTARR_CLIENT_ID`, and
  `ROCKSTARR_NOTIFY_TO` filled in. If missing, abort with a clear
  message — the client needs to complete onboarding.
- `/rockstarr-ai/client.toml` exists. Read `client_name` for the
  email subtitle.
- The shared `send-notification` helper is available
  (`rockstarr-infra/skills/_shared/send-notification/`).

## Inputs

None required. The skill reads everything it needs from the
filesystem.

Optional override (for testing or one-off sends):

- `dry_run = true` — assemble the email and print it to chat,
  do NOT call send-notification.

## Steps

### 1. Collect pending items

Walk `/rockstarr-ai/03_drafts/` recursively. For each file:

- Skip files that don't carry YAML front-matter (logs, READMEs).
- Parse front-matter. Skip files where `approval_status` is missing
  or is anything other than `pending`.
- Capture, in this order of preference:
  - `title` — from front-matter, required by every drafting skill.
  - `channel` — from front-matter (e.g., `blog`, `thought-leadership`,
    `newsletter`, `case-study`, `linkedin-post`, `x-thread`,
    `newsletter-highlight`, `video-script`, `outreach-campaign`,
    `reply`).
  - `produced_by` — from front-matter (e.g.,
    `rockstarr-content/draft-blog@0.2.1`).
  - `stop_slop_score` — if present in front-matter.
  - `awaiting_approval_since` — front-matter ISO timestamp,
    written when the draft landed.
  - `last_updated` — file mtime. Captures both the original landing
    AND any in-place edits the user made before approving.
  - `path_relative` — path relative to `/rockstarr-ai/`.
  - `bucket` (replies only) — if `channel == "reply"`, also pull
    the temperature bucket and any sub-type flags from front-matter.

Build a list of records with these fields.

### 2. Decide whether to fire

If the list is empty:

- Do NOT call send-notification.
- Append one line to
  `/rockstarr-ai/05_published/_mailer.log` with tag
  `approvals_digest_skipped` and `(none)` for message_id.
- Print a one-line confirmation in chat:
  `> Nothing pending. Skipped today's digest.`
- Exit clean.

The no-fire-when-empty rule is non-negotiable. A "you have zero
items" email teaches the client to ignore the inbox.

### 3. Sort

Sort the list by `last_updated` (file mtime) **descending** —
most recently updated drafts first. Tie-break on
`awaiting_approval_since` descending. Tie-break again on path
ascending so output is deterministic.

The user asked for "most recent drafts first" specifically;
`last_updated` (mtime) honors both newly-landed drafts and drafts
the user edited in-place since the last digest.

### 4. Render the markdown body

The mailer renders a restricted markdown subset: `## h2`, `### h3`,
`**bold**`, `*italic*`, `- list`, `1. list`, `` `code` ``, `---`,
`[text](url)`. No tables, no nested lists, no blockquotes, no raw
HTML. Stay inside that set.

**Cowork deep-link per item.** The mailer's allowed URL schemes are
`http`, `https`, `mailto`, AND `claude` (the Anthropic-controlled
desktop-app URL scheme). Each item ends with a deep-link of the
shape `claude://cowork/new?q=<URL-encoded prompt>` that opens
Claude Desktop on the client's machine with a review prompt
pre-typed, so they can read the draft and act without navigating.

**The pre-typed prompt MUST include the file path** (not just the
slug). A fresh Cowork session has no conversation history — it
opens with the workspace folder mounted and the Rockstarr plugins
available, but with zero context about what's been happening.
A path-bearing prompt makes the assistant's first action a
deterministic `Read` of the named file. A slug-only prompt forces
the assistant to glob for it, which fails or asks for help when
slugs collide (v2/v3 versions are the common case).

The prompt format is:
`Show me the draft at <path_relative> for review.`

where `<path_relative>` is the same path shown in the item's
"Location:" bullet (relative to `/rockstarr-ai/`, since Cowork's
working directory is the workspace root and resolves it as-is).

URL-encode the prompt: spaces → `%20`, slashes → `%2F` (for
maximum email-client compatibility — bare `/` works in most
parsers but `%2F` works in all of them), quotes → `%22`. Other
reserved characters per RFC 3986.

The prompt deliberately opens for review, not for auto-approval.
Approve/edit/reject stays the client's call after they've seen
the draft. Auto-approval from an email link would punch a hole
through the human-in-the-loop discipline the system is built on.

If the client reads the email on a device that doesn't have
Claude Desktop installed (phone, web-only mailbox), the link does
nothing — but the file-path bullet stays visible above it, so
they can act manually. The deep-link is a convenience layer on
top of a structure that already works without it.

**Mailer prerequisite.** This skill assumes
`rockstarr-mailer/src/index.ts`'s `safeUrl()` allows the
`claude://` scheme. If the mailer is on an older build that
rejects it (the link renders as `#`), the skill still functions
— file paths are unchanged — but the convenience layer is dead.
Patch the mailer and redeploy to light it up.

Use this exact structure (substituting the real values):

```markdown
## <N> items awaiting your approval

Most recent first. Click any item to open it in Cowork for review.

---

### <Channel label>: <Title>

*Updated <human date and time>*

- Location: `<path_relative>`
- Drafted by `<produced_by>`
- <One lane-specific line — see "Lane-specific lines" below>

[Open in Cowork →](claude://cowork/new?q=Show%20me%20the%20draft%20at%20<URL-encoded path_relative>%20for%20review.)

---

(repeat per item)

---

**To approve:** click any item above to open it in Cowork. After
review, say "approve" — or edit, or reject. Anything you don't
get to today rolls into tomorrow's digest.

*Links open Claude Desktop. If you're reading on a device without
Claude Desktop installed, open your Cowork workspace manually and
reference the file paths above.*
```

Channel label conventions (used in the `### ` heading):

- `blog` → `Researched blog`
- `thought-leadership` → `Thought leadership`
- `newsletter` → `Email newsletter`
- `case-study` → `Case study`
- `linkedin-post` → `LinkedIn post`
- `newsletter-highlight` → `Newsletter highlight`
- `x-thread` → `X / Threads thread`
- `video-script` → `Video script`
- `outreach-campaign` → `Outreach campaign`
- `reply` → `<Channel of origin> reply` (e.g., `Sales Nav reply`,
  `Email reply`) — pull from front-matter if available, else
  default `Reply`.

Lane-specific bullet (one line per item):

- Blog / thought leadership / newsletter / case study /
  newsletter-highlight: `Stop-slop score: <n> / 50`. Flag with
  ` ⚠️ flagged for review` if the score is below 35. (Emoji is
  allowed in body content — it's plain Unicode, not a formatting
  primitive.)
- LinkedIn post / X thread / video script: source line — e.g.
  `Repurposed from "<source title>"`.
- Outreach campaign: `<lead count> leads · <message-count>-step sequence`.
- Reply: `Bucket: <bucket>` plus a one-line action hint (e.g.
  `proposes 3 times next week`, `breakup`, `not interested`).

Date format: `April 25, 2026 · 8:14am` (client's local timezone, not
UTC). The client reads this in their inbox in their own timezone, so
local is the right default.

### 5. Render the plaintext body

The mailer requires `body_text` for deliverability. Flatten the
same content into plain prose — drop the markdown markers, keep
the structure. Example:

```
4 items awaiting your approval. Most recent first.

1. Researched blog: Why founder-led outbound outperforms SDR teams
   in 2026
   Updated April 25, 2026, 8:14am
   Location: 03_drafts/content/why-founder-led-outbound-2026.md
   Drafted by rockstarr-content/draft-blog
   Stop-slop score: 42/50

2. ...

To approve: open your Cowork workspace and say "approve [slug]"
on any item.
```

### 6. Compose and send

Subject: `<N> item<s> awaiting your approval` (singular vs. plural
based on N).

Subtitle: `Daily digest · <client_name>`.

Reply hint:
`"Reply to this email to respond — it reaches your Rockstarr strategist directly."`.

Tag: `approvals_digest`.

Invoke `send-notification` with these values plus the rendered
`body_markdown` and `body_text`. Use default routing
(`notify_type=default`) — the digest is never urgent, by design.

If `dry_run == true`, skip the call and print the assembled subject,
subtitle, body_markdown, and body_text to chat instead.

### 7. Log

`send-notification` writes to `_mailer.log` automatically. This
skill does not write its own log; the mailer log plus the file
mtimes are the audit trail.

Print a one-line confirmation in chat:

> Sent today's digest. <N> items. message_id: <id>.

## Edge cases

- **Front-matter missing `approval_status`.** Skip the file. We
  don't infer pending from absence — every drafting skill is
  required to set the field.
- **Front-matter says `approval_status: approved` but the file is
  still in `03_drafts/`.** Skip. Mismatch is a sign the approve
  skill failed to move the file. Surface it in the next weekly
  audit, not the daily digest.
- **More than 25 pending items.** Cap the rendered list at 25.
  Prepend a one-line warning at the top: `> Showing the 25 most
  recently updated; <N> total pending in your workspace.` This
  prevents a runaway digest if a client falls behind for a week.
- **All pending items are from a single bot.** Render normally;
  don't try to be clever about grouping. The mtime sort is the
  organizing principle, not bot identity.
- **A draft was approved and re-drafted (`-v2`, `-v3`).** Each
  version is its own file, each with its own `approval_status`.
  Show them all — the client sees the latest at the top by mtime
  and decides which to act on.

## What NOT to do

- Do NOT send an email when nothing is pending. The skip is the
  feature, not a bug.
- Do NOT route through `notify_type=urgent`. The digest is by
  definition non-urgent. Urgent items emit their own
  `reply_needed` notification at the time they land.
- Do NOT introduce bot-specific adapters or hard-code which bots
  emit which channels. New drafting bots that follow the
  front-matter contract will appear in the digest automatically.
- Do NOT rewrite or re-flow draft titles. Use front-matter
  verbatim. If a draft has a poor title, that's an upstream fix.
- Do NOT include a draft's body content. The digest is a pointer,
  not a preview. The client opens the workspace to read the prose.
- Do NOT use markdown primitives outside the supported set.
  Tables, nested lists, blockquotes, and raw HTML will silently
  fail at render time. Stick to `##`, `###`, `**`, `*`, `-`, `1.`,
  `` ` ``, `---`, `[link](url)`.
- Do NOT include the bearer token, the Resend API key, or any
  `NOTIFY_*` value in the email body.
- Do NOT batch multiple clients' digests into one email. One
  client, one workspace, one digest.

## Schedule wiring

During `scaffold-client`, the scaffolder creates a scheduled task:

- **Task name:** `approvals-digest`
- **Cron:** `0 6 * * *` (daily 6:00 am local)
- **Prompt:** `"Run the approvals-digest skill in
  rockstarr-infra. Email the digest if there are pending items;
  otherwise exit silently."`

Clients who want a different time say so during onboarding;
update the cron via the schedule skill rather than editing this
file.

## Related

- `skills/_shared/send-notification/` — the mailer helper.
- `skills/approve/` — the destination of every item this digest
  surfaces. Approving a draft removes it from the next digest
  automatically because the file moves out of `03_drafts/`.
- Cross-bot front-matter contract: every drafting skill in every
  Rockstarr AI plugin writes `approval_status: pending` and
  `awaiting_approval_since: <ISO>` to its draft front-matter.
  This skill is what reads it.
