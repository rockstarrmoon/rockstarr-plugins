---
name: approvals-backlog-alert
description: |
  This skill should be used when the scheduled weekly run fires (Monday 8 am local), or when the user says "send the backlog alert", "ping the strategist about the queue", "escalate the approvals backlog", or "run the backlog alert now". Counts pending items in /rockstarr-ai/03_drafts/ and emails the client's Rockstarr strategist when the count exceeds the configured threshold (default 25, configurable per-client in client.toml as strategist_alert_threshold under the [approvals] section). Re-fires every week as long as the count stays over the threshold; goes silent the moment the queue is back under. Recipient is the strategist via ROCKSTARR_STRATEGIST_EMAIL — not the client. Reply-to is set to the client so the strategist can respond and reach the client directly.
---

# approvals-backlog-alert

The strategist-side counterpart to `approvals-digest`. Daily digest
goes to the client; this one goes to Rachel or Jon when the client's
queue is backed up enough that a human nudge will land better than
another routine email.

The math is simple: if the client has more than `N` items waiting on
their approval, somebody on the R&M side needs to know. The default
threshold is 25. The alert re-fires every week while the count stays
over `N` and goes silent the moment it drops back under.

## When to run

- **Scheduled.** Weekly, Monday 8:00 am local time. Cron:
  `0 8 * * 1`. Schedule is created during `scaffold-client`; this
  skill is the body the scheduler invokes.
- **On demand.** User says "send the backlog alert", "ping the
  strategist about the queue", "escalate the approvals backlog",
  "run the backlog alert now". Same logic, same payload, just
  synchronous.

## Preconditions

- `/rockstarr-ai/00_intake/.rockstarr-mailer.env` exists with
  `ROCKSTARR_MAILER_TOKEN`, `ROCKSTARR_CLIENT_ID`, **and
  `ROCKSTARR_STRATEGIST_EMAIL`** filled in. The strategist email is
  optional in `send-notification` but **required here** — it is
  the recipient. If missing or empty, abort with a clear message:

  > Cannot send the backlog alert: `ROCKSTARR_STRATEGIST_EMAIL` is
  > not set in `/rockstarr-ai/00_intake/.rockstarr-mailer.env`. Add
  > the strategist's address (Rachel or Jon, whoever owns this
  > account) and re-run.

- `ROCKSTARR_NOTIFY_TO` should also be set so the alert's reply-to
  routes back to the client. If missing, fall through to the
  Worker's default reply-to (`hello@rockstarrandmoon.com`) and warn
  the user — that's a usable fallback but not the intended route.
- `/rockstarr-ai/client.toml` exists. Read `client_name` and the
  optional `[approvals]` block.
- `send-notification` helper is available
  (`rockstarr-infra/skills/_shared/send-notification/`).

## Configuration

The threshold and cadence are per-client overrides in
`/rockstarr-ai/client.toml`:

```toml
[approvals]
# Item count above which the strategist gets a weekly alert.
# Default: 25. Set higher for high-volume clients, lower for
# clients on tighter SLAs.
strategist_alert_threshold = 25

# Reserved for future use. The schedule itself is owned by the
# Cowork scheduler, not this field — change cadence by editing
# the scheduled task, not by changing this value.
# strategist_alert_cadence = "weekly"
```

The `[approvals]` block is optional. If absent, defaults are used:

- `strategist_alert_threshold = 25`

If the client wants a different number, Rachel or Jon edits
`client.toml` during onboarding (or any time after). No skill
re-deploy required — this skill reads the file fresh on every run.

## Inputs

None required. Optional override for testing or one-off sends:

- `dry_run = true` — assemble the alert and print it to chat,
  do NOT call `send-notification`.
- `force = true` — fire even if the count is at or below the
  threshold. Useful for previewing the email or testing routing.
  Logs the forced send with tag `approvals_backlog_alert_forced`
  so it doesn't pollute the regular cadence audit.

## Steps

### 1. Read the threshold

Parse `/rockstarr-ai/client.toml`. Look for `[approvals]
strategist_alert_threshold = <int>`. If missing, default to `25`.

Validate: must be a positive integer. If the value is malformed
(zero, negative, non-integer), warn loudly in chat, fall back to
the default of 25, and proceed.

### 2. Resolve the strategist email

Parse `/rockstarr-ai/00_intake/.rockstarr-mailer.env` line by line
(same parser shape as `send-notification`). Read
`ROCKSTARR_STRATEGIST_EMAIL` and `ROCKSTARR_NOTIFY_TO`. If
`ROCKSTARR_STRATEGIST_EMAIL` is missing or empty, abort with the
message above.

### 3. Count pending items

Walk `/rockstarr-ai/03_drafts/` recursively, applying the same
contract `approvals-digest` uses:

- Skip files without YAML front-matter.
- Parse front-matter; keep only files where `approval_status ==
  "pending"`.
- For each kept file, capture: `title`, `channel`, `produced_by`,
  `awaiting_approval_since`, `path_relative`, and the file mtime
  (as `last_updated`).

Build the list. The cross-bot front-matter contract is the same
contract the daily digest reads — adding a new drafting bot
automatically lights up here too.

### 4. Decide whether to fire

Let `count` = the length of the list and `threshold` = the
configured threshold.

- If `count <= threshold` and `force != true`:
  - Do NOT call `send-notification`.
  - Append one line to `/rockstarr-ai/05_published/_mailer.log`
    with tag `approvals_backlog_alert_skipped`, message_id
    `(none)`, and the `count` / `threshold` for audit.
  - Print a one-line confirmation in chat:
    `> Queue at <count> / <threshold>. No alert sent.`
  - Exit clean.
- If `count > threshold` (or `force == true`): proceed.

The skip-when-clear rule is non-negotiable. The point of this
alert is the signal it gives by appearing — false alarms erode it.

### 5. Compute the snapshot

Roll up the pending list into:

- `count` — total pending.
- `oldest_age_days` — days since the oldest item's
  `awaiting_approval_since`. Round to nearest day.
- `median_age_days` — median age across all items. One decimal.
- `breakdown` — count per `channel` label. Sort by count desc.
  Use the same channel labels as `approvals-digest`:
  - `blog` → `Researched blog`
  - `thought-leadership` → `Thought leadership`
  - `newsletter` → `Email newsletter`
  - `case-study` → `Case study`
  - `linkedin-post` → `LinkedIn post`
  - `newsletter-highlight` → `Newsletter highlight`
  - `x-thread` → `X / Threads thread`
  - `video-script` → `Video script`
  - `outreach-campaign` → `Outreach campaign`
  - `reply` → `Reply` (with channel-of-origin prefix if available).
- `oldest_five` — the five items with the earliest
  `awaiting_approval_since`. Tie-break on `last_updated` ascending,
  then path ascending so output is deterministic.

### 6. Render the markdown body

Use the same restricted markdown set the mailer accepts: `## h2`,
`### h3`, `**bold**`, `*italic*`, `- list`, `1. list`, `` `code` ``,
`---`, `[link](url)`. No tables, no nested lists, no blockquotes,
no raw HTML.

Structure (substituting real values):

```markdown
## <count> items pending — <client_name>'s queue is backed up

<client_name> has more than <threshold> items awaiting their
approval. Worth a check-in this week.

---

### Snapshot

- **Total pending:** <count>
- **Threshold:** <threshold>
- **Oldest item:** <oldest_age_days> day<s> ago
- **Median age:** <median_age_days> days

### Breakdown by channel

- <Channel label>: <count>
- <Channel label>: <count>
- ...

### 5 oldest items

- *<MMM D, YYYY>* — <Channel label>: <Title> (`<path_relative>`)
- ... (repeat for up to 5)

---

**Suggested next step:** reply to this email and the response goes
straight to <client_name>. A "want to clear this together on a
call?" usually resets the queue.

This alert re-fires every Monday morning while the count stays
over <threshold>. Goes silent the moment they're back under.
```

Date format: `MMM D, YYYY` (e.g., `April 14, 2026`) for the oldest
items. Strategist's local timezone — whichever timezone the run
fires in. The strategist reads it in their own time; that's the
right default.

### 7. Render the plaintext body

Same content flattened, no markdown markers. Required for
deliverability.

### 8. Compose and send

Subject: ``[Backlog alert] <client_name>: <count> items pending``

Subtitle: ``Strategist alert · <client_name>``

Reply hint: ``"Reply to this email to respond — it reaches <client_name> directly."``

Tag: ``approvals_backlog_alert`` (or
``approvals_backlog_alert_forced`` if `force == true`).

Invoke `send-notification` with **explicit overrides** so the
asymmetric routing is unambiguous:

```
to            = <ROCKSTARR_STRATEGIST_EMAIL from env>
reply_to      = <ROCKSTARR_NOTIFY_TO from env>   # may be empty; helper falls back
subject       = <subject>
subtitle      = <subtitle>
body_markdown = <rendered markdown>
body_text     = <rendered plaintext>
reply_hint    = <reply hint>
tag           = "approvals_backlog_alert"
```

Passing `to` explicitly bypasses the env-file routing entirely —
this email never goes to the client, no matter how `notify_type`
or `ROCKSTARR_NOTIFY_TO` are set. Passing `reply_to` explicitly
ensures the strategist's reply lands in the client's mailbox.

### 9. Log

`send-notification` writes to `_mailer.log` automatically. No
extra logging needed.

Print a one-line confirmation in chat:

> Backlog alert sent. <count> items, threshold <threshold>.
> message_id: <id>.

## Edge cases

- **Threshold value is missing or malformed.** Default to 25.
  Warn in chat so the user knows the file needs a fix.
- **`ROCKSTARR_STRATEGIST_EMAIL` set to a comma-separated list.**
  Treat as a list (Rachel + Jon, for instance). The mailer
  accepts an array for `to`. Split on commas, trim whitespace.
- **`ROCKSTARR_NOTIFY_TO` is empty.** Send anyway, but warn the
  user that the reply-to falls through to the shared R&M inbox.
- **The queue dropped below threshold between alert send and
  next run.** Fine — next Monday's run sees `count <= threshold`
  and skips. No "queue cleared" notification. Silence is the
  signal.
- **First-time fire and `oldest_age_days` is zero (every item
  landed today).** Show `< 1 day ago`. Do not show "0 days" — it
  reads as broken.
- **Fewer than 5 pending items even though count > threshold
  somehow.** Should not happen (threshold default is 25), but if
  it does, list every item in the "5 oldest" section without
  padding.
- **`approvals-digest` was skipped this morning because the queue
  was empty, but this alert fires with a non-zero count.** Should
  not happen — both skills read the same source. If observed,
  surface in the next weekly audit.

## What NOT to do

- Do NOT send to the client. This email is for the strategist
  only. The client gets the daily digest.
- Do NOT send when the count is at or below threshold. The skip
  is the feature.
- Do NOT route through `notify_type=urgent`. The backlog is by
  definition non-urgent — it's a weekly nudge, not a page.
- Do NOT include draft body content. Like the daily digest, this
  is a pointer email. The strategist opens the workspace (or
  asks the client) for content.
- Do NOT use markdown primitives outside the supported set.
  Tables, nested lists, blockquotes, and raw HTML silently fail
  at render. Stick to `##`, `###`, `**`, `*`, `-`, `1.`, `` ` ``,
  `---`, `[link](url)`.
- Do NOT include the bearer token, the Resend API key, or any
  `NOTIFY_*` value in the body.
- Do NOT batch alerts across clients. One client per email so
  the strategist's reply lands in the right mailbox.
- Do NOT auto-retry on `unauthorized` or `invalid_payload`. Per
  `send-notification` rules, those are not transient.

## Schedule wiring

During `scaffold-client`, the scaffolder creates a second
scheduled task alongside `approvals-digest`:

- **Task name:** `approvals-backlog-alert`
- **Cron:** `0 8 * * 1` (Monday 8:00 am local)
- **Prompt:** `"Run the approvals-backlog-alert skill in
  rockstarr-infra. Send the alert if the pending count is over
  the configured threshold; otherwise exit silently."`

Clients who want a different weekday say so during onboarding;
update the cron via the schedule skill rather than editing this
file. The threshold is the per-client knob; the cadence is the
per-org default.

## Related

- `skills/approvals-digest/` — the daily, client-bound counterpart.
  Both skills read the same `03_drafts/**` front-matter contract.
- `skills/_shared/send-notification/` — the mailer helper.
- `skills/scaffold-client/` — wires both scheduled tasks at
  intake time.
- `client.toml` — where `[approvals] strategist_alert_threshold`
  is set per client.
