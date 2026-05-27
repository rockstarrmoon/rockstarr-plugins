---
name: notify-reply-ready
description: "This skill should be used when an outreach reply has landed and a draft has been staged in /rockstarr-ai/03_drafts/replies/, and the client should be notified urgently to review. Trigger phrases: \"send the reply-ready notification\", \"notify the client about new replies\", \"fire the urgent reply notification\". Sends one urgent email per batch summarizing each reply (lead, inbound excerpt, drafted response) with a claude:// deep-link per draft into the present-for-approval flow. Email body follows client-facing-output-voice.md — natural-language framing, no field/value classification listings. Caller (detect-replies, or rockstarr-reply:draft-reply when synchronous) accumulates the staged paths and hands them in."
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
- The shared client-facing output voice reference is available
  (`rockstarr-infra/skills/_shared/references/client-facing-output-voice.md`).
  This skill follows its email body rules — natural-language
  framing, no field/value classification blocks, one-sentence
  classification summary. Refuse if missing.
- Each path passed in `staged_paths` exists at
  `/rockstarr-ai/[path]` and carries the front-matter contract
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
| `source_channel` | yes | translated to a plain-English channel label in the preamble (e.g. "LinkedIn outreach", "email") — see "Channel label mapping" below |
| `lead_name` | yes | subject line + per-reply heading |
| `lead_company` | best-effort | per-reply preamble |
| `lead_title` | best-effort | per-reply preamble |
| `bucket` | yes | input to the one-sentence classification summary at the bottom of each card |
| `sub_types` | optional | input to the one-sentence classification summary (subordinate flags inform the wording) |
| `proposed_label` | yes | input to the one-sentence classification summary |
| `proposed_followup_timer` | yes | input to the one-sentence classification summary ("If no answer in N days I'll surface this again") |
| `inbound_excerpt` | yes | "What Jane said" block |
| `draft_body` | yes | "Here's what I'd send back" block |
| `draft_options` | optional | three-option Warm-non-ICP drafts (array of {label, body}) |
| `path_relative` | yes | the deep-link target |

If `draft_options` is present, render all three options inline with
their labels. Otherwise render `draft_body` as a single block.

### Channel label mapping

`source_channel` is an internal slug. In the email body, translate
to a plain-English noun the client uses. The translation table:

| `source_channel` value | Plain-English label |
|---|---|
| `linkedin-salesnav` / `linkedin-interceptly` / `linkedin-meetalfred` / `linkedin-dripify` / `linkedin-waalaxy` | "LinkedIn outreach" |
| `email-gmail` / `email-outlook` | "email" |
| any other / unknown | the slug verbatim as a fallback |

Use the plain-English form in body prose ("replied to your LinkedIn
outreach"). The slug itself never appears in the email body.

### Classification summary — natural language, not field listing

Old versions of this skill rendered a four-line `Bucket / Sub-types
/ Proposed label / Proposed follow-up timer` field block in the
email. Per `client-facing-output-voice.md` rule 6, that pattern is
out — it reads like a JIRA ticket to a non-technical client. The
replacement is **one short English sentence** at the bottom of each
reply card that compresses the same information into a colleague's
voice.

Compose the sentence from the bucket + sub_types + followup_timer:

| Bucket × signal | Suggested sentence shape |
|---|---|
| `Hot` | "Looks like she wants to move forward — I proposed times." |
| `Warm-ICP` + `ask-for-info` | "Warm and on-ICP, asking a clarifying question." |
| `Warm-ICP` (other) | "Warm and on-ICP — looks like genuine interest." |
| `Warm-non-ICP` | "Warm but a bit off-target for your ICP — three angles below." |
| `Skeptical` | "Pushed back. The draft addresses the objection." |
| `Cold` | "Short reply, no clear next step. Draft is a soft bump." |
| Out-of-office | "Auto-reply, not a real signal — I'll check back when they're back." |

Then a follow-up clause derived from `proposed_followup_timer`:

- `"2d"` / `"3d"` / `"7d"` etc. → " If no answer in [N] days I'll
  surface this again."
- `"none"` / absent → omit the follow-up clause.

The sentence ends there. Two clauses max, total under ~25 words.
The point is to give the client one beat of context, not to
exhaustively communicate every classification field.

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
## <lead_name> replied — needs your eye

<first_name> (<lead_title>, <lead_company>) replied to your
<plain-English channel label>:

### What <first_name> said

<inbound_excerpt>

### Here's what I'd send back

<draft_body>

[Open in Claude to approve →](claude://cowork/new?q=Show%20me%20the%20reply%20draft%20at%20<URL-encoded path_relative>%20for%20approval.)

*<one-sentence classification summary per the rules in
"Classification summary" above>*

---

*Link opens Claude Desktop. If you're on a device without it, the
draft is at `<path_relative>` in your workspace.*
```

**Multi-reply email** (more than one draft in the batch):

Top of the email:

```markdown
## <N> replies waiting on you

Most recent first. Each one has its own draft and a link to open
in Claude.

---
```

Then per reply (newest first), separated by `---`:

```markdown
### <lead_name>

*<lead_title>, <lead_company> · replied to your <plain-English channel label> <relative timestamp>*

**What <first_name> said**

<inbound_excerpt>

**My draft**

<draft_body>

[Open in Claude to approve →](claude://cowork/new?q=Show%20me%20the%20reply%20draft%20at%20<URL-encoded path_relative>%20for%20approval.)

*<one-sentence classification summary>*
```

End with the same footer as the single-reply case (adjusted for
plurality).

**Three-option draft handling** (Warm-non-ICP, when
`draft_options` is present): replace the single `### Here's what
I'd send back` block (single-reply) or `**My draft**` block
(multi-reply) with the three options. Use first-person, not the
old "Drafted reply (3 options — pick one in Cowork)" header which
read as a command. Heading + soft framing:

```markdown
### Three angles — pick whichever fits

This one's a bit off-target for your ICP. Each option steers the
conversation a different way:

**Option A — <draft_options[0].label>**

<draft_options[0].body>

**Option B — <draft_options[1].label>**

<draft_options[1].body>

**Option C — <draft_options[2].label>**

<draft_options[2].body>
```

When a Warm-non-ICP card uses the three-option block, the
one-sentence classification summary at the bottom is implied by the
"a bit off-target for your ICP" framing in the block itself — do
NOT also append the "Warm but a bit off-target..." sentence
underneath the options. One framing per card.

### 5. Render plaintext body

Flatten the markdown — drop the inline code, drop the markdown
syntax, keep the structure. Required for deliverability and for
mail clients that strip HTML.

### 6. Compose subject

- Single: `[lead_name] replied — needs your eye`
- Multiple: `[N] replies waiting on you`

Subtitle: `New replies · [client_name]`.

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

### 8. Confirm in chat

Per `client-facing-output-voice.md` rules 1 and 4: one short
sentence about what happened, in the user's terms. Examples:

- Single reply sent: `Emailed you about [lead_name]'s reply.`
- Multi-reply sent: `Emailed you about [N] new replies.`
- Skipped due to broken front-matter on one or more drafts: append
  one sentence naming the count, and put the file paths in a
  collapsed Details footer rather than inline:

  ```
  Emailed you about 3 new replies. 1 draft was skipped — its
  front-matter is incomplete.
  ```

  Followed by a Details block (one short bullet per skipped path)
  for the operator audit trail.

The Resend `message_id` does NOT appear in the chat confirmation
to the user — it goes into the structured return payload only
(callers that need it can read it from the payload), per the voice
guide's rule that machine-format identifiers stay out of
client-facing text. Operators debugging a delivery issue can read
the id from the payload or from `_errors.md`.

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
  preamble line: `> + [N-8] additional replies — open your workspace
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
- Do NOT render a `**Bucket:** / **Sub-types:** / **Proposed
  label:** / **Proposed follow-up timer:**` field/value block
  inside the email body. That was the V0.x shape and it reads as
  internal classification metadata to a non-technical client. The
  per-card one-sentence classification summary at the bottom
  carries the same information in colleague's voice — see
  "Classification summary — natural language, not field listing"
  above. The same rule applies to chat confirmations — never
  surface `bucket`, `sub_types`, `proposed_label`, or
  `proposed_followup_timer` as bare field names to the client.
- Do NOT use internal slugs (`linkedin-salesnav`, `email-gmail`)
  in body prose. Use the plain-English channel labels from the
  mapping table ("LinkedIn outreach", "email").
- Do NOT include the Resend `message_id` in the chat confirmation
  to the user. The id is operator-debug surface and goes in the
  structured return payload only.

## Caller integration contract

The two callers expected to invoke this skill:

1. **`rockstarr-outreach-[variant]:detect-replies`** — at the end
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
