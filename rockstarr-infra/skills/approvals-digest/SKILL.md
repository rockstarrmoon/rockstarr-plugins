---
name: approvals-digest
description: |
  This skill should be used when the scheduled daily run fires at 6 am local time, or when the user says "send the approvals digest", "what's waiting on me", "email me my approval queue", or "run the digest now". Scans /rockstarr-ai/03_drafts/ for files with approval_status: pending, sorts most-recent first by mtime, and emails one daily summary via send-notification. Email body follows skills/_shared/references/client-facing-output-voice.md — natural-language per-item cards with the channel as a plain-English noun and a one-line "what it is" line, no stop-slop scores or classification fields exposed. Exits silently when nothing is pending. Cross-bot by design — surfaces drafts from content, reply, outreach-*, and any future drafting bot without bot-specific code.
---

# approvals-digest

One daily email summarizing every draft awaiting client approval
across every Rockstarr AI bot. Lives at the infra layer so individual
bots don't each ship their own notifier.

The contract is simple: every drafting bot writes its drafts under
`/rockstarr-ai/03_drafts/[channel]/` with `approval_status: pending`
and `awaiting_approval_since: [ISO]` in front-matter. This skill
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
- The shared client-facing output voice reference is available
  (`rockstarr-infra/skills/_shared/references/client-facing-output-voice.md`).
  The rendered email body follows its rules. Refuse if missing.

## Inputs

None required. The skill reads everything it needs from the
filesystem.

Optional override (for testing or one-off sends):

- `dry_run = true` — assemble the email and print it to chat,
  do NOT call send-notification.

## Anchor message

User-invoked runs ("what's waiting on me?", "run the digest
now") should emit one anchor line BEFORE the directory walk
in Step 1 — that walk is the bulk of the runtime and is
otherwise invisible. Per `client-facing-output-voice.md`
rule 7:

> Scanning today's drafts to email you the approvals digest…

**Scheduled runs don't emit the anchor.** Scheduled-task
output goes to the email itself (or to silence on empty days
per rule 8). Chat-side anchor messages on scheduled runs wake
unattended Claude Desktop sessions for no reason — same logic
the chat-confirmation rule applies. Branch by invocation type:
emit the anchor on user-invoked runs only.

The Tier 1 precondition checks (mailer env exists, voice
reference exists, send-notification helper exists) are cheap
existence checks and run silently before the anchor. They
refuse fast if any fail.

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
- Print a one-line confirmation in chat per the voice guide's
  rule 8 (empty / quiet days are silent or near-silent):
  - **Scheduled run**: NO chat output at all. The mailer-log line
    is the audit record.
  - **User-invoked** (the user explicitly asked "what's waiting
    on me?"): one short sentence: `Nothing waiting for you right
    now — all clear.`
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
`Show me the draft at [path_relative] for review.`

where `[path_relative]` is the same path shown in the item's
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

Use this structure (substituting the real values):

```markdown
## <N> thing<s> waiting on you

Most recent first. Each one has a link to open it in Claude for
review.

---

### <Channel label>: <Title>

*Updated <human date and time>*

<One short "what it is" line — see "Per-item context line" below>

[Open in Claude to review →](claude://cowork/new?q=Show%20me%20the%20draft%20at%20<URL-encoded path_relative>%20for%20review.)

---

(repeat per item)

---

Anything you don't get to today rolls into tomorrow's email.

*Link opens Claude Desktop. If you're on a device without it, the
drafts are in your workspace under `/03_drafts/`.*
```

Channel label conventions (used in the `### ` heading) — these are
the plain-English labels the client uses, never the internal slug:

- `blog` → `Researched blog`
- `thought-leadership` → `Thought leadership`
- `newsletter` → `Email newsletter`
- `case-study` → `Case study`
- `linkedin-post` → `LinkedIn post`
- `newsletter-highlight` → `Newsletter highlight`
- `x-thread` → `X / Threads thread`
- `video-script` → `Video script`
- `outreach-campaign` → `Outreach campaign`
- `reply` → `<Channel of origin> reply` (e.g., `LinkedIn reply`,
  `Email reply` — translate the source-channel slug to plain
  English the same way `notify-reply-ready` does)

### Per-item context line

One sentence per item, on its own line under the `*Updated …*`
line, in the client's terms. Per voice guide rule 6, do NOT
render the lane-specific data as a field/value block — translate
it to a natural sentence.

| Channel | Context line shape |
|---|---|
| `blog` / `thought-leadership` / `case-study` | "Long-form piece for your audience." (no internal QA scores like stop-slop exposed to the client) |
| `newsletter` / `newsletter-highlight` | "Newsletter draft ready for your eye." |
| `linkedin-post` / `x-thread` | "Repurposed from \"<source title>\"." when the front-matter carries a source; otherwise "Short-form post ready for your eye." |
| `video-script` | "Script for your next video." |
| `outreach-campaign` | "[N] leads, [M]-message sequence ready to register." |
| `reply` | Use the same colleague-voice classification sentence pattern from `notify-reply-ready` — e.g. "Warm and on-ICP, asking a clarifying question." Pull `bucket` + `sub_types` from front-matter to compose. |

**File paths and `produced_by` stay out of the email body.** They
were operator-debug surface even in V0.x — promoting them above
the link added cognitive overhead without adding value. The link
itself is the action; the path is in the linked draft's
front-matter for anyone who opens it. Operators who need a path
inventory can look at the `_mailer.log` or scan the drafts folder
directly.

**No `Stop-slop score: [n] / 50` line.** That's internal QA data,
not a client decision input. If a draft scores below the
stop-slop threshold, that's a signal for the drafting bot to
revise before staging — not a signal for the client to override.
Flagged drafts surface via `approvals-backlog-alert`'s strategist
channel, not the client digest.

Date format: `April 25, 2026 · 8:14am` (client's local timezone, not
UTC). The client reads this in their inbox in their own timezone, so
local is the right default.

### 5. Render the plaintext body

The mailer requires `body_text` for deliverability. Flatten the
same content into plain prose — drop the markdown markers, keep
the structure. The plaintext follows the same voice rules as the
markdown body (no field/value blocks, no internal slugs, no
stop-slop scores). Example:

```
4 things waiting on you. Most recent first.

1. Researched blog: Why founder-led outbound outperforms SDR teams
   in 2026
   Updated April 25, 2026, 8:14am
   Long-form piece for your audience.

2. ...

Anything you don't get to today rolls into tomorrow's email.
```

Open the workspace manually to approve when reading from a
plaintext-only client.

### 6. Compose and send

Subject: `[N] thing[s] waiting on you` (singular vs. plural based
on N). Less formal than the V0.x "items awaiting your approval"
phrasing — matches the body's opening line and reads as a quick
heads-up rather than a formal queue alert.

Subtitle: `Daily digest · [client_name]`.

Reply hint:
`"Reply to this email to respond — it reaches your Rockstarr strategist directly."`.

Tag: `approvals_digest`.

Invoke `send-notification` with these values plus the rendered
`body_markdown` and `body_text`. Use default routing
(`notify_type=default`) — the digest is never urgent, by design.

If `dry_run == true`, skip the call and print the assembled subject,
subtitle, body_markdown, and body_text to chat instead.

### 7. Log + chat confirmation

`send-notification` writes to `_mailer.log` automatically. This
skill does not write its own log; the mailer log plus the file
mtimes are the audit trail.

Chat confirmation per the voice guide's rule 4 (one short
sentence) and rule 8 (scheduled silent on quiet days). Branch on
how the skill was invoked:

- **Scheduled run with items**: NO chat output. The email itself
  is the user-facing surface; chat-side confirmation on a
  scheduled run wakes up an unattended Claude Desktop session for
  no reason.
- **User-invoked with items** (the user explicitly asked for the
  digest): one short sentence:
  `Emailed you about [N] thing[s] waiting.`
- **Empty run**: see Step 2's branch — scheduled silent,
  user-invoked one sentence.

The Resend `message_id` does NOT appear in the chat
confirmation — operator-debug surface, lives in `_mailer.log`
for anyone debugging a delivery issue.

## Edge cases

- **Front-matter missing `approval_status`.** Skip the file. We
  don't infer pending from absence — every drafting skill is
  required to set the field.
- **Front-matter says `approval_status: approved` but the file is
  still in `03_drafts/`.** Skip. Mismatch is a sign the approve
  skill failed to move the file. Surface it in the next weekly
  audit, not the daily digest.
- **More than 25 pending items.** Cap the rendered list at 25.
  Prepend a one-line note at the top:
  `*[N] total are open in your workspace — showing the 25 most
  recently updated.*` Plain English, not the V0.x's "> Showing
  the 25 most recently updated; [N] total pending..." phrasing
  which used the operator word "pending." This prevents a runaway
  digest if a client falls behind for a week.
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
- Do NOT surface `Stop-slop score: [n] / 50`, `Bucket: [bucket]`,
  `Location: [path]`, `Drafted by [produced_by]`, or any other
  field/value bullet pattern in the email body. Internal QA
  scores, classification fields, file paths, and `produced_by`
  stamps all stay out of the client surface. The per-item context
  line is the canonical replacement — one sentence in colleague's
  voice. Operator-facing data is in the linked draft's
  front-matter for anyone who opens it.
- Do NOT use internal slugs (`linkedin-salesnav`, `email-gmail`,
  `outreach-campaign`) in body prose. Use the plain-English
  channel labels from the heading table.
- Do NOT include the Resend `message_id` in the chat
  confirmation. Operator-debug data goes in `_mailer.log`.
- Do NOT emit chat output on a scheduled empty-day or successful
  scheduled run. The email is the user-facing surface. Chat-side
  "Sent today's digest" lines on scheduled runs wake unattended
  Claude Desktop sessions for no reason.

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
  `awaiting_approval_since: [ISO]` to its draft front-matter.
  This skill is what reads it.
