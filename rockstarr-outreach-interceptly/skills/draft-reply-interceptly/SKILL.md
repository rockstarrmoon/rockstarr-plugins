---
name: draft-reply-interceptly
description: "This skill should be used after classify-reply returns in the per-reply pipeline, or when the user says \"draft the reply\", \"draft a warm-ICP reply\", \"propose 2-3 times for this lead\", \"draft the non-ICP three-option response\", or \"draft a breakup message\". Picks the draft pattern from classify-reply's bucket plus qualify-lead's verdict, layers in the current managed account's persona (signature, persona notes) and the client's style-guide (voice, warm-reply pattern), runs stop-slop on every output, and writes the staged draft to /03_drafts/outreach/replies/<thread>.md for present-for-approval to show."
---

# draft-reply-interceptly

Picks a draft pattern and produces the reply body. Layers are:

1. **Voice** — `style-guide.md` per client.
2. **Persona** — signature + persona notes for the active managed
   account (from `interceptly-accounts.md`).
3. **Pattern** — the structural template selected by verdict ×
   bucket (see `classify-reply`).
4. **Stop-slop** — final AI-tells-removed pass. Always last.

Voice and persona don't overlap. Voice is how the client talks;
persona is who is signing. Same voice, different personas, is
common. Don't conflate them.

## When to run

- Step 3 of the per-reply pipeline, after `classify-reply`.
- Triggered with verdict + bucket + sub_types already resolved.

## Preconditions

- `rockstarr-infra:stop-slop` is available. Refuse if not.
- `/00_intake/style-guide.md` exists and is approved.
- `/00_intake/interceptly-accounts.md` has a row for the current
  managed account. If the thread could be signed under more than
  one persona, pause and ask `present-for-approval` to route
  the selection question first.
- If verdict = `target` AND bucket = `warm` AND the
  warm-reply-pattern subsection of `style-guide.md` is missing,
  fall back to a neutral warm-ICP template and tag the draft
  `warm-reply pattern not captured — drafts may feel generic.`

## Inputs

- `verdict` — from `qualify-lead`.
- `bucket`, `sub_types` — from `classify-reply`.
- `thread` — message history.
- `active_account_label` — the managed account the reply will
  sign under.
- `active_campaign_slug` — optional; used for proof-point
  selection from the campaign's ICP file.

## Behavior

### Step 1 — Resolve persona

From `interceptly-accounts.md` row for `active_account_label`:

- `signature` — verbatim.
- `persona_notes` — shapes tone nuance.
- `default_signer` — only relevant if the thread is ambiguous
  about who should sign.

If the thread is a new inbound under a campaign scoped to one
account, the active account IS the signer. No ambiguity.

### Step 2 — Pick the draft pattern

Use the verdict × bucket matrix from `classify-reply`. Patterns:

#### Hot — proposes a meeting

- Bot-led booking (`booking_mode=automated` +
  `availability_source=booking_link`): call
  `propose-meeting-times` for 2-3 slots. Draft body that names
  those slots and asks the lead to pick one. Do NOT paste the
  booking link.
- `booking_mode=manual` or `availability_source=gcal` in V1:
  still call `propose-meeting-times` (which reads gcal). Draft
  the same shape. The client books outside the bot and calls
  `mark-booked`.
- Include any booking-link-required fields the lead hasn't
  supplied yet in a one-line ask ("If Tuesday works, what
  email should I send the invite to?").
- If the lead has already agreed to a time AND supplied every
  required field, do NOT draft a reply — instead, create a
  `book-meeting` task via `create-followup-task`. The book-
  meeting skill takes over.

#### Warm-ICP — uses the client's warm-reply pattern

Read the warm-reply subsection of `style-guide.md`. Apply the
captured structure, length, opening, closing, and banned moves.
Do NOT invent a pattern the client didn't describe.

If the pattern is thin, the draft is thin. This is acceptable —
thin drafts are feedback for the next `capture-warm-reply-pattern`
pass.

#### Cold bump — short, low-ask

2-3 sentences, low commitment. Reference one specific signal
from the thread (the accept, a prior touch the lead didn't
respond to). No booking link. No overwrought CTA.

#### Skeptical — acknowledge pushback

Acknowledge the pushback verbatim ("Fair — we hear X a lot").
One proof point from first-party KB cited with file pointer
(the approver sees the pointer, the lead sees only the prose).
Redirect to a specific next step that is NOT the booking link.

#### Referral — thank + ask for intro

Warm thank-you, ask for a direct intro path ("happy to make it
easy — do you have an email I can CC, or do you want to forward
this thread to them?"). Don't pitch past the referrer.

#### Graceful exit / non-ICP polite-yes — three-option generator

When verdict = `not_target` AND bucket = `warm` (including
`polite_yes`):

1. Generate THREE distinct response options:
   - **Graceful exit** — acknowledges the lead's interest,
     names the fit mismatch, thanks them warmly. Text is
     generated from the client's positioning + style-guide —
     NOT from a Rockstarr template.
   - **Let-it-hang** — no reply drafted. Label Ignore. The
     option is presented as a non-action the approver can pick.
   - **Throwaway question** — a lightweight curiosity-preserving
     question that maintains the thread without committing to a
     pitch. Stance: the client stays open to a non-sales
     relationship.

No default is pre-selected. `present-for-approval` shows all
three and the approver picks.

The graceful-exit text is NOT a template. It's generated per
lead from:
- `00_intake/client-profile.md` positioning paragraph
- `00_intake/style-guide.md` Brand Approach + Tone Definition

#### Pitch-back — decline graciously

The lead used the thread to sell their own service. Draft a
one-line decline that names the mismatch without being rude.
Label Bad Fit. Create no follow-up task.

### Step 3 — Compose

With persona + pattern + thread context:

1. Generate a draft body in the client's voice. Apply
   `style-guide.md` Brand Personality, Tone Definition, Style
   Rules verbatim.
2. Append the persona's signature block.
3. Run the full body through `rockstarr-infra:stop-slop`. This
   is mandatory. Order is voice first, stop-slop last.
4. If stop-slop flags heavy rewrites, note that in the draft's
   front matter so the approver sees it.

### Step 4 — Stage the draft

Write `/03_drafts/outreach/replies/<thread-id>.md`:

```markdown
---
thread_id: <Interceptly thread id or lead_url slug>
lead_url: <URL>
lead_name: <name>
lead_title: <title>
campaign_slug: <slug or blank>
active_account_label: <account>
signer_persona: <same as active_account_label or explicit pick>
verdict: target | not_target | ambiguous | unknown
bucket: hot | warm | cold | skeptical | decline
sub_types: [<flags>]
pattern: hot | warm_icp | cold_bump | skeptical | referral | graceful_exit | letitthang | throwaway_q | pitch_back
proposed_label: <Interceptly label>
proposed_followup_timer_days: <int>
stop_slop_flagged: true|false
generated_at: <ISO>
awaiting_approval: true
schema_version: 1
---

# Reply draft

## Body

<post-stop-slop reply body, including signature>

## (If non-ICP three-option flow) Alternate options

### Option 1 — Graceful exit

<body>

### Option 2 — Let-it-hang

No reply sent. Label Ignore.

### Option 3 — Throwaway question

<body>

## Evidence / reasoning

- Verdict: `<verdict>` — rule cited: `<rule>`
- Bucket: `<bucket>` — evidence: `<excerpt>`
- Proof point source (if cited): `<file pointer>`
```

### Step 5 — Hand to present-for-approval

Return the staged draft path to `present-for-approval`. That
skill renders the draft for the operator and blocks on the
'send it' gate.

## Constraints

- Booking link is a destination, never pasted into a reply.
- Every output runs through stop-slop. No exceptions.
- Non-ICP polite-yes ALWAYS goes through the three-option flow.
  Never auto-generate a "graceful exit" without showing the
  other two options.
- Voice belongs to `style-guide.md`. Persona belongs to
  `interceptly-accounts.md`. Do not conflate.

## Failure modes

- **Persona missing signature.** Use the persona's first name as
  fallback; flag in front matter.
- **Warm-reply pattern not captured.** Fall back to neutral
  warm-ICP template; tag draft. Don't block the pipeline.
- **First-party KB has no proof point for a skeptical draft.**
  Write the draft without the proof point; note in front matter
  that the skeptical path would be stronger with a proof point,
  and recommend `rockstarr-infra:kb-ingest`.

## What NOT to do

- Do not send. Sending is `send-message`, and only after
  `present-for-approval` unlocks on 'send it'.
- Do not auto-pick one of the three non-ICP options. The approver
  picks, period.
- Do not paraphrase third-party KB as if the client said it.
- Do not paste the booking link. Ever. If a pattern tempts you
  to, re-read Section 3.10 of the plugin spec.
- Do not skip stop-slop. If it's unavailable, refuse the draft.
