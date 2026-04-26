---
name: draft-reply
description: "This skill should be used as step 2 of the rockstarr-reply core pipeline, after classify-reply returns a bucket. Trigger phrases: \"draft the reply\", \"draft a warm-ICP reply\", \"propose 2-3 times for this lead\", \"draft the non-ICP three-option response\", \"draft a breakup message\", \"redraft shorter\". Picks the draft pattern from bucket + icp_verdict + sub_types, reads style-guide.md (including the warm-reply and optional referral-pattern subsections), applies channel-specific formatting (LinkedIn DMs stay short and sign-block-free; email may include a subject line and appends persona signature), calls propose-meeting-times on Hot threads, generates three option bodies on Warm-non-ICP, and runs stop-slop on every prose path as the mandatory final pass. Writes the staged draft to the channel-scoped drafts folder under /03_drafts/replies/ with full front-matter. Does NOT send — sending is the caller's job after present-for-approval returns authorized-send."
---

# draft-reply

Channel-agnostic reply drafting. Step 2 of the rockstarr-reply core
pipeline. Picks a pattern from the bucket + icp_verdict + sub_types
combination that classify-reply returned, produces the body in the
client's voice, and stages the draft for present-for-approval.

Layers are fixed and ordered:

1. **Voice** — `/00_intake/style-guide.md` per client (including the
   warm-reply subsection from capture-warm-reply-pattern, and the
   optional referral-pattern subsection).
2. **Persona** — signature block + persona notes supplied by the
   caller in the handoff bundle. draft-reply does NOT look up
   personas itself.
3. **Pattern** — the structural template selected by bucket ×
   icp_verdict (see Step 2 below).
4. **Channel adaptation** — LinkedIn vs. email surface differences
   (length, subject line, signature block, booking-link policy).
5. **Stop-slop** — final AI-tells-removed pass. Always last.

Voice and persona don't overlap. Voice is how the client talks;
persona is who is signing. Same voice, different personas, is common
(rockstarr-outreach-interceptly runs under multiple accounts against
the same client style guide). Don't conflate them.

## When to run

- Step 2 of the per-reply pipeline, immediately after
  `classify-reply`.
- Re-entered with `intent_hint=graceful-exit` or
  `intent_hint=throwaway-question` when the caller resolves a prior
  three-option pick (option A or C).
- Re-entered with an operator editing instruction ("make it
  shorter", "try a different angle", "draft a breakup message") when
  present-for-approval loops back for a redraft.

## Preconditions

- `rockstarr-infra` is installed and `skills/_shared/stop-slop/`
  exists. Refuse to draft if stop-slop is unavailable.
- `/00_intake/style-guide.md` exists and is approved.
- `/00_intake/icp-qualifications.md` exists (already checked by
  classify-reply; re-verified here because this skill can also be
  invoked directly with a redraft).
- If `bucket=Warm-ICP` AND the warm-reply subsection of
  style-guide.md is missing, fall back to a neutral warm-ICP template
  and tag the draft front-matter
  `warm_reply_pattern_missing: true` so the approver sees it.

## Inputs

From classify-reply:

- `bucket` — `Hot | Warm-ICP | Warm-non-ICP | Skeptical | Cold`
- `sub_types` — list of flags
- `proposed_label`
- `evidence`

From the caller's handoff bundle:

- `channel` (a.k.a. `source_channel_slug` in the front-matter
  contract) — controls the channel-adaptation switch and the path
  segment for the staged draft. Values:
  `linkedin-interceptly | linkedin-salesnav | linkedin-meetalfred |
  linkedin-dripify | linkedin-waalaxy | email-gmail | email-outlook`.
- `thread` — full text, oldest to newest
- `persona` — `{ name, title, signature_block, persona_notes }`
- `icp_verdict` with `{ matching_rule, evidence }`
- `lead` — `{ url, name, company, title, campaign_slug }`
- `intent_hint` (optional)
- `batch_context` (optional, v0.2+) — free-form string set by
  callers that batch multiple drafts in one run (typically
  `detect-replies` from an outreach-* plugin). When present,
  draft-reply does NOT fire its own urgent notification — the caller
  fires `notify-reply-ready` once at the end of its batch with the
  accumulated path list. When absent (manual / one-off invocation),
  draft-reply fires the urgent notification itself with
  `staged_paths = [the single path it just wrote]`. See Step 6.

## Behavior

### Step 1 — Resolve the pattern

From the bucket × icp_verdict matrix (see classify-reply's semantics
table):

**Hot — propose a meeting.**
Call `rockstarr-infra:propose-meeting-times` (shared) for 2–3 slots
from the client's availability source (booking-link page or Google
Calendar, per `stack.md.availability_source`). Draft a body that
names the specific slots and asks the lead to pick one. Include a
one-line ask for any `booking_link_required_fields` the lead hasn't
supplied yet ("If Tuesday works, what email should I send the invite
to?").

**Never paste the booking link.** The link is a destination the
caller's `book-meeting` skill will drive — it is not content in the
reply.

If the lead has already agreed to a time AND supplied every required
field (detectable from the thread text), do NOT draft a new
conversational reply. Instead, emit `book-meeting-handoff` in the
return payload with `{ slot, lead_fields }` and let the caller's
`book-meeting` skill take over. This only fires when
`stack.md.booking_mode=automated`; on `manual`, draft a brief
confirmation reply and let the client book outside the bot.

**Warm-ICP — client's warm-reply pattern.**
Read the warm-reply subsection of style-guide.md. Apply the captured
structure, length, opening line, closing move, and banned moves
verbatim. Do NOT invent a pattern the client didn't describe.

If the captured pattern is thin, the draft is thin. That's
acceptable — thin drafts are feedback for the next
`capture-warm-reply-pattern` pass.

**Warm-non-ICP — three-option generator.**
Do NOT produce one draft. Produce THREE distinct option bodies:

- **Option A — graceful exit.** Acknowledges the lead's interest,
  names the fit mismatch in the client's own positioning language,
  thanks them warmly. Generated from `/00_intake/client-profile.md`
  positioning paragraph + style-guide.md Brand Approach + Tone
  Definition — NOT from a stock Rockstarr template.
- **Option B — let it hang.** No body. Structural output only —
  presented as a non-action the approver can pick. Label Ignore, no
  send, no task.
- **Option C — throwaway question.** A lightweight
  curiosity-preserving question that maintains the thread without
  committing to a pitch. Stance: the client stays open to a non-sales
  relationship.

**No default is pre-selected.** present-for-approval renders all
three and the approver picks. The three-option flow is preserved
verbatim because auto-defaulting to one option in the past has sent
replies the client would have killed.

**Skeptical — default silent-label, optional referral pivot.**
Default path: propose `Not Interested` (or `stack.md.label_mapping`
override) and draft no body. The caller labels and moves on.

If the tone is cold-but-friendly AND a `referral-pattern` subsection
exists in style-guide.md (from the deferred
`capture-referral-pattern` skill), optionally draft a referral pivot
per that subsection's structure. No proof-point pitching past the
referrer — the pivot asks for a direct intro ("happy to make it easy
— do you have an email I can CC, or do you want to forward this
thread?").

No follow-up task once labeled.

**Cold — short specific bump.**
Two to three sentences, low commitment. Reference one concrete
signal from the thread or the lead's profile summary (a prior touch
the lead didn't respond to, the accept, a specific claim in their
profile). No booking link. No overwrought CTA.

After 3 bumps with no reply on the same thread (detectable from the
thread log + caller-supplied campaign context), return `flag` instead
— the caller surfaces the lead for human review.

**Pitch-back (sub_type) — decline graciously.**
The lead used the thread to sell their own service. Draft a one-line
decline that names the mismatch without being rude. Proposed label:
`Bad Fit`. No follow-up task.

**intent_hint overrides.**
If `intent_hint=graceful-exit` the caller resolved a prior
three-option pick to option A — draft that body directly (same
generator as the Warm-non-ICP option A). If
`intent_hint=throwaway-question`, draft option C. If
`intent_hint=breakup`, draft a short, no-pressure final message and
propose `Ignore` as the label (or `Not Interested` if the thread has
gone cold after 3 bumps). If
`intent_hint=book-meeting-followup`, return
`book-meeting-handoff` (see Hot path above).

### Step 2 — Compose the body

With the pattern + persona + thread context + channel:

1. Generate the draft body in the client's voice. Apply
   style-guide.md Brand Personality, Tone Definition, and Style
   Rules verbatim.
2. Apply the channel-adaptation switch (Step 3 below).
3. Run the full body through `rockstarr-infra:_shared/stop-slop`.
   **This is mandatory.** Order is voice first, stop-slop last.
   stop-slop returns a 0–50 score (5 dimensions × 10). Capture the
   numeric value as `stop_slop_score` in front-matter and set
   `stop_slop_flagged: true` when the score is below 35 (the
   threshold stop-slop itself flags as "revise"). On the
   three-option Warm-non-ICP path, run stop-slop on each prose
   option independently and capture the MIN score across the
   prose-bearing options (A and C) as the front-matter value.

Structural outputs (the three-option labels themselves, front-matter
fields, log entries) are exempt from stop-slop. It runs on prose
only.

### Step 3 — Channel adaptation

draft-reply has a small switch that shapes the surface of the draft
based on `channel`:

**LinkedIn channels** (`linkedin-interceptly | linkedin-salesnav |
linkedin-meetalfred | linkedin-dripify | linkedin-waalaxy`):

- Short. 1–2 paragraphs typical.
- No subject line.
- No signature block appended — the sender account IS the signature.
- Booking URL NEVER pasted, regardless of style-guide permissions.
- Emoji tolerance follows style-guide.md.

**Email channels** (`email-gmail | email-outlook`):

- Supports a subject line. Draft one on first-reply threads;
  subsequent replies inherit the existing subject (the caller
  supplies it in the thread context).
- Longer paragraphs acceptable if style-guide.md doesn't restrict.
- Persona `signature_block` IS appended on send (the write-draft-
  variant skill handles this; draft-reply includes a signature-block
  marker in the draft body).
- Booking link MAY be included as a plain URL if style-guide.md
  explicitly permits it — but the conversational "propose 2–3 times"
  pattern is still the default because it reads as more human.

The adaptation is metadata. It does not reshape the prose pipeline;
it only changes surface format.

### Step 4 — Stage the draft

Write the draft file to
`/03_drafts/replies/<source_channel_slug>/<thread-id>.md`. The path
segment is the raw channel string from the handoff (e.g.
`linkedin-interceptly`, `email-gmail`).

The front-matter follows the **cross-bot contract** that
`rockstarr-infra` v0.8's `notify-reply-ready` and `approvals-digest`
read. Field names are non-negotiable — match them exactly, including
the deliberately surprising `channel: "reply"` (the lane
discriminator) sitting alongside `source_channel_slug` (the origin
channel slug).

File shape:

```markdown
---
# --- cross-bot contract (read by rockstarr-infra v0.8 readers) ---
channel: "reply"                      # lane discriminator — always literal "reply"
source_channel: "Sales Nav reply"     # human-readable origin label (display map below)
source_channel_slug: linkedin-salesnav  # the raw channel string from the handoff
title: "Reply to <lead_name> at <lead_company>"  # human-facing summary; see Title rule below
produced_by: "rockstarr-reply/draft-reply@0.2.0"
approval_status: pending              # enum — pending | approved | rejected
awaiting_approval_since: <ISO>        # set on first stage; preserved across redrafts
path_relative: 03_drafts/replies/<slug>/<thread-id>.md
inbound_excerpt: "<≤300 chars, see Inbound Excerpt rule below>"
draft_body: |
  <post-stop-slop reply body. Email variants end with a
  [SIGNATURE_BLOCK] marker. LinkedIn variants omit the marker.>
draft_options:                        # only when bucket=Warm-non-ICP; otherwise omit
  - label: "Graceful exit"
    body: |
      <Option A body, post stop-slop>
  - label: "Let it hang"
    body: "(no body — label Ignore, no send)"
  - label: "Throwaway question"
    body: |
      <Option C body, post stop-slop>
stop_slop_score: <int 0-50>           # numeric score from stop-slop
stop_slop_flagged: true | false       # true when score < 35

# --- reply-specific fields (read by classify-reply, present-for-approval) ---
thread_id: <channel-specific thread id, or lead_url slug>
lead_url: <URL>
lead_name: <name>
lead_title: <title>
lead_company: <company>
campaign_slug: <slug or blank>
persona_name: <persona.name>
persona_title: <persona.title>
icp_verdict: target | not-target | ambiguous | unknown
matching_rule: <rule from handoff bundle>
bucket: Hot | Warm-ICP | Warm-non-ICP | Skeptical | Cold
sub_types: [<flag>, ...]
pattern: hot | warm_icp | cold_bump | skeptical | referral | graceful_exit | letitthang | throwaway_q | pitch_back | book_meeting_handoff
proposed_label: <label>
proposed_followup_timer: <keyword — computed by follow-up-timer>
warm_reply_pattern_missing: true | false
generated_at: <ISO>
last_revised_at: <ISO>                # only after the first redraft
revision_count: 0
schema_version: 2                     # v0.2 contract
---

# Reply draft

## Body

<post-stop-slop reply body — same string as front-matter draft_body.
Email variants end with a [SIGNATURE_BLOCK] marker where the
signature will be injected on send. LinkedIn variants omit the
marker.>

## (If non-ICP three-option flow) Alternate options

### Option A — Graceful exit

<body — same string as front-matter draft_options[0].body>

### Option B — Let-it-hang

No reply sent. Label Ignore.

### Option C — Throwaway question

<body — same string as front-matter draft_options[2].body>

## Evidence / reasoning

- icp_verdict: `<verdict>` — rule cited: `<matching_rule>`
- bucket: `<bucket>` — evidence: `<excerpt from inbound>`
- proposed meeting times (Hot only): `<ISO slots from propose-meeting-times>`
- proof point source (if cited): `<file pointer into 01_knowledge_base/processed/>`
```

Notes on the contract:

- **`channel: "reply"` is the lane discriminator.** It always carries
  the literal string `"reply"` so the cross-bot digest knows which
  lane to render this draft as. This is NOT the source channel.
- **`source_channel_slug` carries the origin channel.** The same
  string driving the channel-adaptation switch and the path segment
  (`linkedin-salesnav`, `email-gmail`, etc.).
- **`source_channel` is the human-readable label** rendered in
  email subjects and digest headings. Compute via this map:

  | source_channel_slug | source_channel |
  |---|---|
  | `linkedin-interceptly` | `LinkedIn (Interceptly) reply` |
  | `linkedin-salesnav` | `Sales Nav reply` |
  | `linkedin-meetalfred` | `LinkedIn (MeetAlfred) reply` |
  | `linkedin-dripify` | `LinkedIn (Dripify) reply` |
  | `linkedin-waalaxy` | `LinkedIn (Waalaxy) reply` |
  | `email-gmail` | `Email reply (Gmail)` |
  | `email-outlook` | `Email reply (Outlook)` |

- **`title` rule.** Default:
  `Reply to <lead_name> at <lead_company>`. Append a qualifier from
  pattern / sub_types when one is informative:
  - `pattern=hot` → ` (meeting ask)`
  - `pattern=cold_bump` → ` (cold bump)`
  - `pattern=referral` → ` (referral pivot)`
  - `pattern=skeptical` → ` (Skeptical — silent label)` (or
    `(Skeptical — referral pivot)` when the optional pivot fires)
  - `pattern=graceful_exit | throwaway_q | letitthang` → ` (non-ICP — 3 options)`
  - `intent_hint=breakup` → ` (breakup)`

  Example: `Reply to Jane Doe at Acme (meeting ask)`.

- **`inbound_excerpt` rule.** Pull the LAST inbound segment from the
  handoff `thread`. Strip channel-noise: signature blocks, quoted
  prior-message bodies, "On <date> <name> wrote:" headers,
  `--` Reply-Above-This-Line markers. Trim to 300 chars max; if
  truncated, append `…` (single Unicode ellipsis, not three dots).
  Do NOT reflow whitespace or paraphrase — keep the lead's voice
  verbatim. notify-reply-ready renders it under a `### What
  <first_name> said` heading.

- **`draft_body` vs. markdown body.** The same post-stop-slop string
  is written in two places — into front-matter as `draft_body` (so
  notify-reply-ready can render it inline in the email) and into
  the markdown after the `## Body` heading (so present-for-approval
  has a clean human-readable surface). When the bucket is
  `Warm-non-ICP`, OMIT `draft_body` and write `draft_options`
  instead. notify-reply-ready picks the rendering branch based on
  which is present.

- **`stop_slop_score`.** stop-slop returns a score 0–50 (5
  dimensions × 10). Capture the numeric value as `stop_slop_score`.
  Set `stop_slop_flagged: true` when score < 35 (the threshold
  stop-slop itself flags as "revise").

- **`approval_status: pending`** is set on first stage and stays
  `pending` across operator-driven redrafts. present-for-approval is
  the only skill that flips it to `approved` or `rejected` (see
  that skill's spec).

- **`awaiting_approval_since`** is set on first stage and PRESERVED
  across redrafts — the file mtime captures revision activity, this
  field captures the original landing time the digest's "Updated"
  formatter reads via mtime separately.

On a redraft triggered by present-for-approval, OVERWRITE the same
file. Bump `revision_count`. Preserve `generated_at`,
`awaiting_approval_since`, and `approval_status: pending`. Set
`last_revised_at`. Re-run stop-slop and recompute `stop_slop_score`
+ `stop_slop_flagged` on the new body.

### Step 5 — Fire the urgent notification (v0.2+, conditional)

When draft-reply has just staged a draft AND the handoff bundle does
NOT carry `batch_context`, immediately call
`rockstarr-infra:notify-reply-ready` with
`staged_paths = [<the path just written>]`. This is the synchronous
notification path for manual / one-off invocations — the operator
gets an urgent email in their inbox with the proposed body rendered
inline plus a `claude://cowork/new?q=...` deep-link back into
present-for-approval.

When `batch_context` IS present, SKIP this step. The caller (an
outreach-* `detect-replies`, an email-variant `read-inbox-*`, or any
other batched flow) is responsible for accumulating the `staged_paths`
list across its run and calling `notify-reply-ready` once at the end
of its batch. Cross-run batching is the digest's job, not this
skill's.

Skip this step entirely on the `book-meeting-handoff` and
`no-action` returns — no draft was staged, so there is nothing to
notify on. The caller may still emit its own non-urgent log entry;
the urgent mailer is reserved for actually-staged reply drafts the
client can act on.

Skip this step on a redraft (revision_count > 0). The first
notification fires when the draft first lands; subsequent operator-
driven redrafts happen inside the present-for-approval gate, and a
re-notification would just spam the operator's inbox while they're
already in the loop.

If `notify-reply-ready` errors (mailer unreachable, missing
`.rockstarr-mailer.env`, etc.), log the error to chat but do NOT
abort the draft — the draft file is still on disk and the next
day's `approvals-digest` will surface it. Urgent notifications are
best-effort; the digest is the durable safety net.

### Step 6 — Return

Return one of the following to the caller (via present-for-approval):

- Staged body — most common path. Returns `{draft_path,
  pattern, proposed_label}` for present-for-approval to render.
- Three-option staged bundle — for Warm-non-ICP. Returns
  `{draft_path, three_options: true, option_a_body, option_c_body}`.
  present-for-approval renders all three; option B is rendered as
  "let it hang — no body sent".
- `book-meeting-handoff` — when the Hot path detects the lead has
  already agreed + supplied required fields. Returns `{ slot,
  lead_fields }`. The caller's `book-meeting` skill takes over; no
  body is drafted, no notification fires.
- `no-action` — when sub_types include `out_of_office`,
  `already_booked`, or `not-target` × `Cold` / `Skeptical`. Returns
  `{ reason }` for the caller to log and move on. No body drafted,
  no approval needed, no notification fires.

## Constraints — preserved verbatim

- **Booking link is a destination, not content.** LinkedIn replies
  never contain the URL. Email replies may include it only when
  style-guide.md explicitly permits AND the conversational
  "propose times" pattern is not the better choice.
- **Every prose output runs through stop-slop.** No exceptions.
  Voice first, stop-slop last, always.
- **Non-ICP polite-yes ALWAYS goes through the three-option flow.**
  Never auto-generate a graceful exit without showing the other two
  options. Never pre-select a default.
- **Voice belongs to style-guide.md. Persona belongs to the caller's
  handoff bundle.** Do not conflate.
- **stop-slop is MANDATORY.** If it's unavailable, refuse the draft
  and raise the missing shared skill to the operator.

## Failure modes

- **Persona signature_block missing.** Use `persona.name` as
  fallback; flag in front-matter `persona_signature_missing: true`.
  LinkedIn paths don't need the block anyway.
- **Warm-reply pattern subsection missing.** Fall back to neutral
  warm-ICP template; tag draft `warm_reply_pattern_missing: true`.
  Don't block the pipeline.
- **First-party KB has no proof point for a Skeptical draft.** Write
  the draft without the proof point; note in front-matter that the
  Skeptical path would be stronger with a proof point, and recommend
  `rockstarr-infra:kb-ingest` to the operator at approval time.
- **propose-meeting-times returns no slots (calendar conflict /
  booking-link page unreachable).** Draft a Hot reply that
  acknowledges interest and asks the lead to name a preferred window,
  without proposing specific times. Tag
  `meeting_times_unavailable: true`.
- **Channel unknown / unsupported** — refuse and raise. Do not
  default to a LinkedIn surface for an unknown channel; the
  channel-adaptation switch is deliberately narrow.

## What NOT to do

- Do not send. Sending is the caller's job, and only after
  present-for-approval returns `authorized-send`.
- Do not auto-pick one of the three non-ICP options. The approver
  picks every time.
- Do not paraphrase third-party KB as if the client said it. Cite
  only first-party KB (`kb_scope: owned`) in the reasoning block.
- Do not paste the booking link on any LinkedIn channel, ever.
- Do not skip stop-slop. If the shared skill is unavailable, refuse
  the draft.
- Do not hardcode reply patterns. Every pattern the bot produces
  comes from either style-guide.md (voice + warm-reply + optional
  referral-pattern subsections) or the structural rules in this
  skill (Hot = propose times; Warm-non-ICP = three options; etc.).
- Do not write anything outside `/03_drafts/replies/<channel>/`.
  The draft file is the only output of this skill's file-staging
  step. (Step 5's notify-reply-ready call writes to the mailer log
  via `send-notification`; that's expected and lives in
  rockstarr-infra, not here.)
- Do not fire notify-reply-ready on a redraft. The first staging
  fires the urgent notification once; redrafts inside the
  present-for-approval loop don't re-notify.
- Do not fire notify-reply-ready when `batch_context` is set. The
  batching caller fires once at the end of its run.
- Do not fire notify-reply-ready on `book-meeting-handoff` or
  `no-action` returns. No draft was staged, so there's nothing to
  notify on.
- Do not block draft staging on a notify-reply-ready error. The
  mailer is best-effort; the durable surface is the draft file
  plus tomorrow's approvals-digest.
