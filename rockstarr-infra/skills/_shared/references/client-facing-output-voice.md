---
title: "Client-facing output voice + discipline"
purpose: "Single source of truth for what every Rockstarr plugin skill says AFTER it finishes work — chat summaries, confirmation lines, next-step prompts, email body templates, and the anchor message that fires at the start of any long-running skill. Companion to intake-interviewer-voice.md, which governs question discipline during interviews. This file governs reporting-back discipline everywhere else."
applies_to: "Every Rockstarr plugin skill that produces user-facing chat output, email body content, or post-completion summaries. Operator-only outputs (test fixtures, dev logs, _errors.md technical traces) are out of scope."
read_by:
  - "every prose-producing skill in rockstarr-infra, rockstarr-content, rockstarr-social, rockstarr-outreach-salesnav, rockstarr-outreach-interceptly, rockstarr-reply, rockstarr-ops, rockstarr-crm"
do_not_fork: true
---

# Client-facing output voice + discipline

The Rockstarr clients are non-technical founders, sales leaders, and
operators. They opened Claude Desktop, asked the bot to do
something, and they want to know: **did it work, what do I have now,
what's next.** Every skill that ends a turn with chat output, or
sends an email body, or shows a confirmation, is talking to that
person.

**This file is the single source of truth for how skills talk to
clients after work is done.** Companion to
`intake-interviewer-voice.md`, which governs how skills ask
questions during interviews. Same tone (direct, calm, no jargon)
applied to a different moment in the conversation.

Variation in output voice creates variation in client trust. A
skill that confirms with "Promoted: 03_drafts/outreach/campaign-X.md
→ 04_approved/outreach/2026-05-22_outreach-salesnav_..." reads like
a CI server. A skill that confirms with "Approved. Your campaign is
ready to go live — let me know when you've posted it" reads like a
colleague. The client experience of working with Rockstarr is the
sum of every one of those small moments.

## The voice

Same baseline as the intake voice — direct, calm, supportive
without being saccharine, no theater. **The shift is from
"interrogating" to "reporting back."** Where the intake voice
asks one focused question and waits, the output voice states what
happened and what's possible next, in plain English, and stops.

The shared prose rules every Rockstarr bot follows still apply
(these come from `stop-slop` and `intake-interviewer-voice.md`):

- No em-dashes.
- No `-ly` adverbs (`really`, `truly`, `genuinely`, `simply`).
- No throat-clearing openers ("Here's the thing," "Let me be
  clear," "The reality is").
- No business jargon ("circle back," "unpack," "lean into").
- No false intimacy ("I promise," "creeps in").
- No announcing structure ("In this section we'll...").

Two new rules apply specifically to reporting-back:

- **No structural narration.** Do not say "Phase 1.2 — Baseline
  ICP from client-profile.md" or "Step 7 of 11" in chat. The
  client does not care about the skill's internal phase numbering.
  They care that you're at a specific point in a recognizable
  process ("you're three questions into the ICP exercise"), or
  that something concrete happened ("your ICP is captured").
- **No "Used 3 tools, edited 2 files" filler.** Cowork sometimes
  surfaces these system lines automatically — that is Cowork's
  job, not the skill's. The skill's first user-facing line after
  finishing should be about the *outcome*, not the *mechanism*.

## The discipline

Eight rules. Each one has a concrete failure mode the rule
prevents. If you find yourself wanting to break a rule, the answer
is to revise the rule in this file, not to deviate inside one
skill.

### 1. Lead with the outcome the user cares about

The first sentence of any post-completion message names the
concrete thing that now exists or now happened, in the user's
terms.

**Wrong:**

> Promoted: 03_drafts/outreach/campaign-2026-05-22-proff-services.md
> → 04_approved/outreach/2026-05-22_outreach-salesnav_campaign-2026-05-22-proff-services.md.
> Approval front-matter added: approved_at, approved_by,
> approved_from, approval_status: approved. Log entry written:
> 04_approved/_approvals.log.

**Right:**

> Approved. Your campaign is ready to go live — let me know when
> you've posted the first batch and I'll log it.

The right version leads with **what now exists** ("approved",
"ready to go live") and **what to do next** ("let me know when you
posted it"). The file paths, log entries, and front-matter mutations
all happened — the user does not need a roll-call.

### 2. Use the user's nouns, not the system's slugs

Translate internal vocabulary into the words the user actually
uses for the thing.

| System slug                       | What to say                                |
|-----------------------------------|--------------------------------------------|
| `draft` / `03_drafts/`            | "your LinkedIn post" / "your blog draft" / "the reply" |
| `bucket: Warm-ICP`                | "warm and a great fit" / "warm and on-ICP" |
| `bucket: Warm-non-ICP`            | "warm but a bit off-target"                |
| `channel-of-origin: linkedin-salesnav` | "the LinkedIn outreach"             |
| `source_channel`                  | (don't show — embed in the sentence: "Jane replied on LinkedIn") |
| `proposed_followup_timer: 2d`     | "I'll check back on this in 2 days"        |
| `stop_slop_score`                 | (don't show externally — internal QA only) |
| `campaign_slug`                   | the campaign's actual title                |
| `approved_at` / `published_at`    | "as of 9:14am" / "yesterday at 4pm"        |

When a structured artifact name needs to appear (file the user
will actually open), name it in plain English: not
`03_drafts/outreach/campaign-X.md` but "the campaign brief I just
drafted." The path lives in the details footer (see rule 5).

### 3. Next-step prompts use verbs the user recognizes

Skill names are internal identifiers. Next-step language uses
verbs the user already knows from their work.

**Wrong:**

> Ready to publish. Run the channel's publish flow, then
> `publish-log` to record what shipped.

**Right:**

> Your post is approved. Post it to LinkedIn now, then tell me
> when it's up and I'll log it for the weekly report.

If a follow-up skill MUST be named (rare), name it in
backticks and explain what it does:

**Acceptable when necessary:**

> Your campaign is registered and the first batch sends tomorrow
> morning. To pause the campaign at any time, say "stop the
> campaign" or run `stop-campaign`.

### 4. One or two sentences for the confirmation

A post-completion message is one sentence about the outcome and
one sentence about what's next. **Maximum.** A multi-bullet
recap of what changed reads as audit log, not confirmation.

**Wrong:**

> Approved. Summary:
>
> - Promoted: 03_drafts/.../campaign-X.md → 04_approved/.../...
> - Approval front-matter added: approved_at, approved_by,
>   approved_from, approval_status: approved
> - Log entry written: 04_approved/_approvals.log
> - Original draft: removed from 03_drafts/ (one file per run,
>   no overwrites)

**Right:**

> Approved. Your campaign is ready to go live — post it when you
> can and I'll log it for the weekly report.

If genuinely meaningful detail exists (a partial outcome, a
warning, a number worth knowing), include it as one short
follow-up sentence. Not as bullets.

**Right (with detail):**

> Crawled 49 of the 250 leads on this pass — Sales Nav's lazy
> loading capped the run. The morning loop will pick up the rest
> over the next few days.

### 5. File paths and metadata go in a collapsed Details footer

Most users never click a file path. Surfacing one in the main
message line implies the user should — which adds cognitive
overhead with no benefit.

If file paths, IDs, or technical details genuinely belong in the
message (audit trail, copy-paste-into-something usefulness), put
them at the end under a small heading or a `<details>` block:

~~~markdown
Approved. Your campaign is ready to go live — post it when you
can and I'll log it for the weekly report.

<details>
<summary>Details</summary>

- File: `04_approved/outreach/2026-05-22_outreach-salesnav_campaign-X.md`
- Approval log: `04_approved/_approvals.log` (entry #14)
</details>
~~~

The user who needs the path can expand. The user who doesn't is
not distracted.

### 6. No structured field-by-field readouts in body content

Tables of `Bucket: Warm-ICP / Sub-types: ask-for-info / Proposed
label: <slug> / Proposed follow-up timer: 2d` belong in operator
debug surface (the workbook, `_errors.md`, the run-summary that
goes to Jon), not in client-facing chat or email body content.

When the classification adds context, write it as one English
sentence at the bottom of the message:

**Wrong (in the `notify-reply-ready` email body today):**

> ### Classification
>
> - **Bucket:** Warm-ICP
> - **Sub-types:** ask-for-info
> - **Proposed label:** warm
> - **Proposed follow-up timer:** 2d

**Right:**

> Jane's reply looks warm and on-ICP — she's asking a clarifying
> question. If she hasn't responded in 2 days, I'll surface the
> thread again.

### 7. Anchor message: name the work fast for any skill >5 seconds

Cowork sometimes takes a few seconds to start producing output —
between the user hitting send and the first content line, the
client may see nothing but a spinner. **Any skill that does more
than ~5 seconds of work emits one short anchor line first**, so
the client immediately sees the bot is alive and working.

#### The rules

The anchor message:

- **Fires as the FIRST text the skill emits**, before any tool
  calls, *and before any precondition file reads*. Even when a
  skill is about to refuse (because a precondition fails or a
  required artifact is missing), the user still sees "I'm
  starting" first — followed by the refusal — rather than
  silence-then-refusal.
- **Is one sentence**, present-tense ("Drafting…"), not
  past-tense ("I drafted…") or future-tense ("I'll draft…").
  The client should see "this is happening right now."
- **Names the user-recognizable noun** ("your campaign", "your
  approvals digest"), never the skill name
  (`draft-icp-campaign`, `approvals-digest`).
- **Is a single sentence.** Not a paragraph. Not a bullet list
  of steps the skill is about to take.
- **Ends with an ellipsis** (`…`) to signal "more coming," not
  a period (which signals "done") or no punctuation (which feels
  unfinished).

For skills that complete in under ~5 seconds (most one-shot
helpers, single-file edits, status checks), the anchor message is
optional — the completion message is plenty.

#### Examples by skill family

| Skill family | Bad (V0.x silence or filler) | Good anchor message |
|---|---|---|
| Intake interviews (`intake-icp`, `intake-transformations`, etc.) | *(no output for ~10s while the bot reads `client-profile.md` and pre-drafts)* | "Reading your profile and drafting a starting ICP — back in a moment…" |
| Profile compile (`compile-profile`) | *(silent while reading 6 artifacts and assembling)* | "Stitching your business profile together from the six intake steps…" |
| Style-guide build (`generate-style-guide`) | "Phase 0 — pre-reading..." | "Reading your voice samples and drafting your style guide — this takes a few minutes…" |
| Campaign drafting (`draft-icp-campaign`) | *(silence while reading profile + KB + style guide)* | "Drafting your campaign — reading your profile and pulling proof points from your knowledge base…" |
| Lead crawl (`crawl-lead-list`) | "Validating session…" | "Opening your Sales Nav saved search to pull leads — this takes a few minutes…" |
| Daily connect (`daily-connect`) | *(silent until the first send confirms)* | "Sending today's connects — 20 to go, ~30 seconds each with pacing…" |
| Approvals digest (`approvals-digest`) | "Scanning 14 draft folders across 7 lanes…" | "Scanning today's drafts to email you the approvals digest…" |
| Long-form drafting (`draft-blog`, `draft-thought-leadership`, etc.) | *(silence for ~2 minutes during research + draft + stop-slop)* | "Drafting your blog post — researching, drafting, then running the quality pass. ~2 minutes…" |

#### Anchor-message anti-patterns

- **Filler-only.** "Working on it…" tells the user nothing. The
  anchor names *what* you're working on in their words.
- **Internal-structure narration.** "Phase 0 — pre-reading…",
  "Step 1 — reading workbook…", "Initializing skill state…"
  All bad. The client doesn't care about phases or steps; they
  care about the work in their terms.
- **Skill-name as noun.** "Running `draft-icp-campaign`…"
  exposes a name the client never types and doesn't recognize.
  "Drafting your campaign…" is the same length and means
  something to them.
- **Multi-line "here's what I'll do" bullet lists.** The anchor
  is one sentence. Step-by-step roadmaps belong in the body's
  step section (if anywhere), not in the chat opening.
- **Time estimates as the lead.** "This takes 3 minutes…" by
  itself is filler. Add the time estimate to the anchor when
  the work is genuinely long (>~30 seconds), as a soft
  ending — not as the headline. "Drafting your blog post…
  ~2 minutes."
- **Past-tense or future-tense framing.** "I read your profile
  and started drafting" implies the work is done. "I'll draft
  your campaign next" implies it hasn't started. Present-tense
  is the only form that matches the client's experience of
  *the bot is working right now*.

#### When the anchor fires relative to precondition checks

Skills that do significant filesystem reads in their precondition
checks (see "Defer expensive preconditions" in
`rockstarr-infra/CLAUDE.md`) often produce no output for several
seconds while those checks run. Two rules:

- **Fast existence checks** (does this file exist? does that
  directory exist?) are fine to run before the anchor message;
  they're typically sub-second.
- **Expensive content reads** (read 6 markdown artifacts, walk
  a directory tree, parse multiple files) happen AFTER the
  anchor. The anchor names what's about to happen — the reads
  are part of that work, not a separate pre-flight surface.

In practice this means most long-running skills' opening
sequence looks like:

1. Cheap existence checks (does `client-profile.md` exist?) —
   silent.
2. Anchor message — "Stitching your business profile together…"
3. Expensive content reads happen here, inside the work.
4. The skill's main work continues.

If a precondition fails after the anchor, the refusal follows
naturally as the next chat line ("…actually, your `intake/icp.md`
hasn't been captured yet. Want me to start that step now?").
Better experience than silence-then-refusal.

### 8. Empty / quiet days are silent or near-silent

When a skill's run produces nothing meaningful (no new drafts to
digest, no replies to detect, no actions taken), the right
output is either silence (for event-driven / scheduled skills)
or one short sentence (for user-invoked runs).

**Wrong:**

> Approvals digest run complete. Scanned 14 draft folders across
> 7 lanes. Found 0 drafts in `approval_status: pending` state.
> Sent 0 emails. No further action required.

**Right (scheduled run, silent):**

> *(no chat output; the run wrote a tiny "0 pending" line to the
> daily log and exited)*

**Right (user-invoked):**

> Nothing waiting for you right now — all clear.

`approvals-digest` already does this for the email (silent on
empty days). The same discipline applies to in-chat output for
every skill.

## Applies to which output surfaces

This file's rules govern:

- **In-chat post-completion messages.** Every "## After writing"
  or "## Output" section of every SKILL.md that ends with text
  the user will read.
- **Email body content** sent via `_shared/send-notification/`,
  including `notify-reply-ready`, `approvals-digest`,
  `approvals-backlog-alert`, and any future notification skill.
- **Anchor messages** fired at the start of long-running skills.
- **AskUserQuestion question bodies** that incorporate
  "here's what I just did" framing (the question and answer
  surface itself is governed by `intake-interviewer-voice.md` —
  this file covers any prose that precedes or follows the
  question).

This file does NOT govern:

- **The actual content artifacts the bots produce on behalf of
  the Rockstarr client.** Replies that get sent to leads (drafted
  by `rockstarr-reply:draft-reply`), blog posts and
  thought-leadership pieces drafted for the client's audience
  (`rockstarr-content:draft-blog`, `draft-thought-leadership`,
  `draft-newsletter`, `draft-case-study`), outreach campaign
  message bodies (`rockstarr-outreach-*:draft-icp-campaign` M2 /
  M3 / M4 / follow-up bodies), and any other artifact the bot
  writes *in the client's voice for the client's audience* — all
  of these stay governed by the established artifact chain:
  the client's `00_intake/style-guide.md`, the relevant pattern
  or rubric (TL rubric, blog SEO/GEO checklist, reply
  bucket pattern, sequence rules), and the mandatory `stop-slop`
  final pass. This file applies to the *framing around* such
  artifacts (the chat message confirming a draft landed, the
  email body announcing a reply needs review), never to the
  artifact bodies themselves. The boundary: "talking *to* the
  Rockstarr client about work" (this file) vs. "writing *on
  behalf of* the Rockstarr client for their audience" (existing
  chain).
- Operator-facing surfaces — `_errors.md`, the workbook itself,
  test fixtures, dev-only logs, request-support emails to
  R&M. Those can be technical because the readers are technical.
- Skill body documentation (the rest of SKILL.md after the
  description). Skill bodies are read by Claude during routing
  and by developers reading source — they can carry the
  implementation detail this file wants to keep out of the user
  experience.
- The structured *return payload* between skills (the JSON-shaped
  Output blocks that callers parse programmatically). Those keep
  their existing field-by-field shape. The voice rule applies to
  the *user-facing chat or email rendering* of that payload, not
  the payload itself.

## Worked examples — wrong / right pairs

### `approve` confirmation

**Wrong (today):**

> Promoted: 03_drafts/outreach/campaign-2026-05-22-proff-services.md
> → 04_approved/outreach/2026-05-22_outreach-salesnav_campaign-2026-05-22-proff-services.md.
> Approval front-matter added: approved_at, approved_by,
> approved_from, approval_status: approved. Log entry written:
> 04_approved/_approvals.log.

**Right:**

> Approved. Your campaign is ready to go live — post the first
> batch when you can and I'll log it.

### `publish-log` confirmation

**Wrong (today):**

> Shipped. When results come in, `metrics-review` will read this
> log and attribute performance back to the draft.

**Right:**

> Logged. Your LinkedIn post is live as of 9:14am — performance
> shows up in this Friday's report.

### `notify-reply-ready` email body

**Wrong (today):**

~~~
## Reply from Jane Smith needs your review

Jane Smith (VP Marketing at Acme Corp) replied via Sales Nav.

### What Jane said
> Can you tell me more about pricing?

### Classification
- **Bucket:** Warm-ICP
- **Sub-types:** ask-for-info
- **Proposed label:** warm
- **Proposed follow-up timer:** 2d

### Drafted reply
[body]

[Open in Cowork →]
~~~

**Right:**

~~~
## Jane Smith replied — needs your eye

Jane (VP Marketing, Acme Corp) replied to your LinkedIn outreach:

> Can you tell me more about pricing?

Here's what I'd send back:

> [body]

[Open in Claude to approve →]

Looks warm and on-ICP — she's asking a clarifying question. If
no answer in 2 days I'll surface the thread again.
~~~

### `register-campaign` confirmation

**Wrong (verbose structured-block version this skill used to
produce in 0.2.x):**

> Campaign registered. Summary:
>
> - Slug: 2026-05-22-proff-services
> - Status: active (Campaigns row written)
> - Workbook: outreach-tasks.xlsx created with 8 sheets
> - Stack.md: canonical yaml keys promoted to front-matter
> - Log: 05_published/outreach/2026-05-22.md entry written
> - Leads seeded: 0 (crawl pending — see below)
> - Conflicts rejected: 0

**Right:**

> Your campaign is live. The first batch of 250 leads is queued
> for tomorrow morning's run, and I'll start sending blank
> connects then.

### Anchor message for a long-running skill

**Wrong (silence for 30 seconds while crawl-lead-list works,
then a wall of text):**

> *(no output)*
> *(30 seconds pass)*
> Crawled 49 of 250 leads. Pages 1-2 hydrated; pages 3-16 stalled.
> Conflicts: 0. Skipped: 12 ...

**Right:**

> Reading your saved search and pulling the first batch of leads…
>
> *(tool calls happen — Cowork's spinner shows activity)*
>
> Pulled the first 49 leads from your saved search. Sales Nav's
> lazy loading capped the run; the morning loop will pick up the
> rest over the next few days.

### Empty-day output for a scheduled skill

**Wrong (today, when `approvals-digest` runs on an empty day):**

> Approvals digest complete. Scanned 14 draft folders across 7
> lanes. Found 0 drafts in `approval_status: pending` state.
> Sent 0 emails. No further action required.

**Right (scheduled, empty day):**

> *(no chat output. Daily log shows "0 pending — silent.")*

**Right (user-invoked on an empty day, e.g., "any approvals
waiting?"):**

> Nothing waiting for you right now — all clear.

## When to deviate

Almost never. The rules are deliberately strict because
inconsistency across 100+ skills is what creates the "this
sometimes feels like talking to a developer" experience.

If you find a case that genuinely doesn't fit:

1. First check whether the audience is actually a client or
   actually an operator. Operator output (Jon, Rachel, an
   installer) is not bound by this file.
2. If genuinely client-facing and the rule still doesn't fit,
   propose a revision to this file via PR rather than
   deviating in one skill. Cross-bot consistency is the whole
   point.

## Provenance

This reference shipped in `rockstarr-infra 0.9.14` as the
foundation of a UX-pass series triggered by client feedback that
the system "feels technical." See ClickUp AI-142 and the parent
series (AI-143 through AI-147) for the cross-bot rollout.
