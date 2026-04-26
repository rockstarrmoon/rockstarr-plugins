---
name: notify-reply-ready
description: |
  This skill should be used when an outreach reply has landed and a draft has been staged in /rockstarr-ai/03_drafts/replies/, and the client should be notified urgently to review it. Trigger phrases: "send the reply-ready notification", "notify the client — new replies are ready", "ping the client about Jane's reply", "fire the urgent reply notification". Sends one urgent email per batch summarizing each reply (lead, company, inbound message excerpt, classification, proposed draft body) with a claude:// deep-link per draft so the client opens Claude Desktop directly into the present-for-approval flow. Routes through send-notification with notify_type=urgent. Caller is responsible for accumulating just-staged draft paths — typically detect-replies in an outreach variant after its review-reply task batch completes, or rockstarr-reply:draft-reply directly when running synchronously on a single thread.
---

# notify-reply-ready

The third leg of the cross-bot approvals stack, alongside
`approvals-digest` and `approvals-backlog-alert`. Where the digest
is a routine daily roll-up and the backlog alert is a weekly
strategist nudge, this one is **urgent and immediate**: an inbound
reply landed, a draft is staged, and the client needs to see it
inside the next hour to keep the conversation alive.

Same mailer, same `claude://` deep-link pattern, same front-matter
contract — different routing (`notify_type=urgent`) and different
shape (per-reply card with lead context + the proposed draft body
rendered inline, so the client can read it without clicking).

## When to run

- Called by an outreach variant's `detect-replies` skill at the
  end of its run, with the list of paths to drafts staged during
  that run. detect-replies is the natural batch boundary — five
  replies in a single morning pass produce one email, not five.
- Called by `rockstarr-reply:draft-reply` directly when running
  synchronously on a single inbound (e.g. a manual "draft a reply
  to Jane" invocation that produces one staged draft outside the
  daily loop).
- Called manually by the user with the trigger phrases above.

The skill itself does not poll for new drafts — it operates on
exactly the list of paths the caller hands in. Callers that want
batching implement it at their own layer.

## Preconditions

- `/rockstarr-ai/00_intake/.rockstarr-mailer.env` exists with
  `ROCKSTARR_MAILER_TOKEN`, `ROCKSTARR_CLIENT_ID`, and
  `ROCKSTARR_NOTIFY_TO` filled in. `ROCKSTARR_NOTIFY_URGENT_TO` is
  optional — if set, urgent items route there; if empty, urgent
  items fall back to the default `NOTIFY_TO` (the
  `send-notification` helper handles the resolution).
- `/rockstarr-ai/client.toml` exists. `client_name` is read for
  the email subtitle.
- The shared `send-notification` helper is available
  (`rockstarr-infra/skills/_shared/send-notification/`).
- Each path passed in `staged_paths` exists at
  `/rockstarr-ai/<path>` and carries the front-matter contract
  this skill reads (see below).

## Inputs

- `staged_paths` — array of paths to just-staged reply drafts,
  relative to `/rockstarr-ai/`. Required. Empty array → exit
  silently with no send.
- `dry_run` — optional. Assemble the email and print it to chat,
  do NOT call `send-notification`.

## Front-matter contract

Each staged reply draft carries — written by
`rockstarr-reply:draft-reply` — the following YAML front-matter
fields. This skill reads them; if a non-required field is missing,
surface a "(not captured)" note in the email rather than blocking
the send.

| Field | Required | Used for |
|---|---|---|
| `channel` | yes — must be `"reply"` | sanity check; skip non-reply files |
| `source_channel` | yes | "Sales Nav reply", "Email reply", etc. |
| `lead_name` | yes | subject line + per-reply heading |
| `lead_company` | best-effort | per-reply preamble |
| `lead_title` | best-effort | per-reply preamble |
| `bucket` | yes | classification block |
| `sub_types` | optional | classification block (subordinate flags) |
| `proposed_label` | yes | classification block |
| `proposed_followup_timer` | yes | classification block |
| `inbound_excerpt` | yes | "What Jane said" block |
| `draft_body` | yes | "Drafted reply" block |
| `draft_options` | optional | three-option Warm-non-ICP drafts (array of {label, body}) |
| `path_relative` | yes | the deep-link target |

If `draft_options` is present, render all three options inline with
their labels. Otherwise render `draft_body` as a single block.

## Steps

### 1. Validate the input

If `staged_paths` is empty or missing, exit silently. Print:

> No staged reply drafts passed. Nothing to notify.

### 2. Read each draft's front-matter

For each path:

- Skip files that don't exist (broken state — log a warning but
  don't abort the whole batch).
- Skip files where `channel != "reply"`.
- Skip files where required fields are missing (log a warning;
  surface in the post-run summary so the caller can audit).

If after filtering the list is empty, exit silently.

### 3. Sort

Sort by file mtime descending — newest reply first. Tie-break by
path ascending for determinism.

### 4. Render the markdown body

The mailer renders a restricted markdown subset: `## h2`, `### h3`,
`**bold**`, `*italic*`, `- list`, `1. list`, `` `code` ``, `---`,
`[text](url)`. Allowed URL schemes are `http`, `https`, `mailto`,
and `claude://` (mailer ≥ 0.2.1 — patch the Worker before this
skill ships).

**A note on excerpt formatting.** The mailer's primitive set
excludes blockquotes. Lines that start with `>` render as plain
paragraphs with a literal `&gt;` at the front, which is not what
we want. The right pattern for the inbound-message excerpt and
the drafted reply is `### h3` heading followed by a regular
paragraph — h3 gives clear visual hierarchy, and the paragraph
reads as prose. Do NOT prefix excerpt lines with `>`.

**Single-reply email** (one draft in the batch):

```markdown
## Reply from <lead_name> needs your review

<lead_name> (<lead_title> at <lead_company>) replied via
<source_channel>. Drafted response below — open the workspace to
approve, edit, or reject.

---

### What <first_name> said

<inbound_excerpt>

### Classification

- **Bucket:** <bucket>
- **Sub-types:** <sub_types comma-joined, or "none">
- **Proposed label:** <proposed_label>
- **Proposed follow-up timer:** <proposed_followup_timer>

### Drafted reply

<draft_body>

[Open in Cowork →](claude://cowork/new?q=Show%20me%20the%20reply%20draft%20at%20<URL-encoded path_relative>%20for%20approval.)

---

**To respond:** click the link above to open Claude Desktop with
the draft loaded. Approve, edit, or reject.

*Links open Claude Desktop. If you're reading on a device without
Claude Desktop installed, open your Cowork workspace manually — the
reply draft is at `<path_relative>`.*
```

**Multi-reply email** (more than one draft in the batch):

Top of the email:

```markdown
## <N> new replies need your review

Most recent first. Each reply has its own draft and Cowork link
below.

---
```

Then per reply (newest first), separated by `---`:

```markdown
### <Channel label>: <lead_name>

*<lead_title> at <lead_company> · replied <relative timestamp>*

**What <first_name> said**

<inbound_excerpt>

**Classification:** <bucket> · <proposed_label> · follow-up
`<proposed_followup_timer>`

**Drafted reply**

<draft_body>

[Open in Cowork →](claude://cowork/new?q=Show%20me%20the%20reply%20draft%20at%20<URL-encoded path_relative>%20for%20approval.)
```

End with the same footer as the single-reply case (adjusted for
plurality).

**Three-option draft handling** (Warm-non-ICP, when
`draft_options` is present): replace the single `### Drafted reply`
block with:

```markdown
### Drafted reply (3 options — pick one in Cowork)

**Option A: <draft_options[0].label>**

<draft_options[0].body>

**Option B: <draft_options[1].label>**

<draft_options[1].body>

**Option C: <draft_options[2].label>**

<draft_options[2].body>
```

### 5. Render plaintext body

Flatten the markdown — drop the inline code, drop the markdown
syntax, keep the structure. Required for deliverability and for
mail clients that strip HTML.

### 6. Compose subject

- Single: `Reply from <lead_name> needs your review`
- Multiple: `<N> new replies need your review`

Subtitle: `New replies · <client_name>`.

Reply hint:
`"Reply to this email to respond — it reaches your Rockstarr strategist directly."`.

Tag: `reply_needed`.

### 7. Send

Invoke `send-notification` with **`notify_type=urgent`** so the
helper resolves the recipient via the urgent rules:

```
notify_type   = "urgent"
subject       = <subject>
subtitle      = <subtitle>
body_markdown = <rendered markdown>
body_text     = <rendered plaintext>
reply_hint    = <reply hint>
tag           = "reply_needed"
```

If `dry_run == true`, skip the call and print everything to chat
instead.

### 8. Confirm

Print a one-line confirmation:

> Sent the reply-ready notification. <N> draft<s>. message_id: <id>.

If any drafts were skipped due to missing/broken front-matter,
list them so the caller can audit:

> Skipped 1 draft due to missing front-matter:
> - `03_drafts/replies/foo.md`

## Edge cases

- **A reply lands but `draft-reply` errored and didn't stage a
  draft.** No path gets added to `staged_paths`. This skill
  doesn't see it, doesn't fire. The error surfaces upstream (in
  `rockstarr-reply` itself or the outreach variant's error log).
- **An inbound is hostile, off-topic, or otherwise unsafe.**
  `draft-reply` flags via `flag-for-review` instead of staging a
  draft. No path is staged → no email. Flagged threads are
  handled by the operator separately, not via this notification.
- **A flurry of replies after a campaign goes live (10+).** Render
  all of them in one email up to a soft cap of 8 cards. If more
  than 8 land in one batch, render the 8 most recent and add a
  preamble line: `> + <N-8> additional replies — open your workspace
  to see the full list at /03_drafts/replies/.`
- **A reply was already drafted earlier and the client has not
  yet acted on it.** This skill is fired on a NEW staged draft
  (the caller's batch) and only contains paths from this run —
  prior pending drafts surface in the daily digest, not here.
  The two skills are complementary: urgent is for newly-arrived,
  digest is for everything still open.
- **Three-option Warm-non-ICP draft.** Render all three options
  inline (per the structure above) so the client can pick from
  the email if they want, then commit in Cowork.

## What NOT to do

- Do NOT send when `staged_paths` is empty. The skip is the
  feature.
- Do NOT scan `/03_drafts/replies/` to discover paths. The caller
  passes the batch explicitly. Scanning would make the skill
  fire-on-rerun-of-a-stale-batch by accident.
- Do NOT send via `notify_type=default`. Reply-ready is urgent by
  definition — the whole point is to land in the inbox the client
  watches actively.
- Do NOT include the bearer token, the Resend API key, the
  `NOTIFY_*` values, or any other env-file content in the body.
- Do NOT include the full inbound message body if it's longer than
  ~300 characters — truncate with `…` and let the client open the
  workspace for the full thread. The email is a pointer to the
  conversation, not a transcript.
- Do NOT use markdown primitives outside the supported set
  (tables, nested lists, blockquotes, raw HTML silently fail at
  render). Stick to `##`, `###`, `**`, `*`, `-`, `1.`, `` ` ``,
  `---`, `[link](url)`. In particular: do NOT prefix excerpt
  lines with `>` — the mailer has no blockquote handler and the
  `>` will render as literal text. Use `### h3` headings + plain
  paragraphs for the inbound excerpt and drafted reply per the
  structure in step 4.
- Do NOT batch across runs. One detect-replies run = one email
  (or zero, if no drafts staged). Cross-run batching reintroduces
  exactly the staleness problem that the digest already handles.

## Caller integration contract

The two callers expected to invoke this skill:

1. **`rockstarr-outreach-<variant>:detect-replies`** — at the end
   of its daily-loop run, after all `review-reply` tasks have
   produced their drafts. Calls `notify-reply-ready` once with
   the accumulated `staged_paths` array.
2. **`rockstarr-reply:draft-reply`** — when invoked synchronously
   on a single inbound (manual flow, not via a batched task
   queue). Calls `notify-reply-ready` immediately after staging
   the single draft, with `staged_paths = [<single path>]`.

Either caller MUST guarantee the front-matter contract before
calling — the contract belongs to `draft-reply`, the
notification just reads it.

## Related

- `skills/_shared/send-notification/` — the mailer helper this
  skill calls with `notify_type=urgent`.
- `skills/approvals-digest/` — daily client-bound digest of every
  pending draft. Already-staged reply drafts that haven't been
  approved show up here too on the next morning's run.
- `skills/approvals-backlog-alert/` — weekly strategist alert
  when total pending exceeds threshold. Reply drafts count
  toward that total.
- `rockstarr-reply:draft-reply` — owns the front-matter contract
  this skill reads.
- `rockstarr-reply:present-for-approval` — the in-Cowork
  destination of every deep-link this skill emits.
