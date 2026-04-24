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

- `channel` — controls the channel-adaptation switch
- `thread` — full text, oldest to newest
- `persona` — `{ name, title, signature_block, persona_notes }`
- `icp_verdict` with `{ matching_rule, evidence }`
- `lead` — `{ url, name, company, title, campaign_slug }`
- `intent_hint` (optional)

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
   **This is mandatory.** Order is voice first, stop-slop last. If
   stop-slop flags heavy rewrites, note that in the draft's
   front-matter (`stop_slop_flagged: true`) so the approver sees it.

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
`/03_drafts/replies/<channel>/<thread-id>.md`. The `<channel>` path
segment is the raw channel string from the handoff (e.g.
`linkedin-interceptly`, `email-gmail`).

File shape:

```markdown
---
thread_id: <channel-specific thread id, or lead_url slug>
lead_url: <URL>
lead_name: <name>
lead_title: <title>
lead_company: <company>
campaign_slug: <slug or blank>
channel: <channel>
persona_name: <persona.name>
persona_title: <persona.title>
icp_verdict: target | not-target | ambiguous | unknown
matching_rule: <rule from handoff bundle>
bucket: Hot | Warm-ICP | Warm-non-ICP | Skeptical | Cold
sub_types: [<flag>, ...]
pattern: hot | warm_icp | cold_bump | skeptical | referral | graceful_exit | letitthang | throwaway_q | pitch_back | book_meeting_handoff
proposed_label: <label>
proposed_followup_timer: <keyword — computed by follow-up-timer>
stop_slop_ran: true
stop_slop_flagged: true | false
warm_reply_pattern_missing: true | false
generated_at: <ISO>
revision_count: 0
awaiting_approval: true
schema_version: 1
---

# Reply draft

## Body

<post-stop-slop reply body. Email variants end with a
[SIGNATURE_BLOCK] marker where the signature will be injected on
send. LinkedIn variants omit the marker.>

## (If non-ICP three-option flow) Alternate options

### Option A — Graceful exit

<body>

### Option B — Let-it-hang

No reply sent. Label Ignore.

### Option C — Throwaway question

<body>

## Evidence / reasoning

- icp_verdict: `<verdict>` — rule cited: `<matching_rule>`
- bucket: `<bucket>` — evidence: `<excerpt from inbound>`
- proposed meeting times (Hot only): `<ISO slots from propose-meeting-times>`
- proof point source (if cited): `<file pointer into 01_knowledge_base/processed/>`
```

On a redraft triggered by present-for-approval, OVERWRITE the same
file and bump `revision_count`. Preserve `generated_at` from the
first write; add `last_revised_at`.

### Step 5 — Return

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
  body is drafted.
- `no-action` — when sub_types include `out_of_office`,
  `already_booked`, or `not-target` × `Cold` / `Skeptical`. Returns
  `{ reason }` for the caller to log and move on. No body drafted,
  no approval needed.

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
  The draft file is the only output.
