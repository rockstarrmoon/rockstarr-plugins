---
name: classify-reply
description: "This skill should be used after qualify-lead returns target or not_target in the per-reply pipeline, or when the user says \"classify this reply\", \"what's the temperature on this thread\", or \"is this a hot / warm / cold reply\". Reads the thread body (most-recent inbound message + one or two earlier messages for context) and assigns a temperature bucket — hot / warm / cold / skeptical / decline — plus a sub-type flag when relevant (meeting_proposed, referral, pitch_back, polite_yes). Temperature drives which draft pattern draft-reply-interceptly picks."
---

# classify-reply

Temperature classification. Takes a qualified lead's most recent
thread state and returns a bucket that maps to a draft pattern.

## When to run

- Step 2 of the per-reply pipeline, after `qualify-lead` returns
  `target` or `not_target`.
- Does NOT run after `ambiguous` or `unknown` — those route to
  `flag-for-review`, not to classification or drafting.

## Inputs

- `thread` — list of messages, each `{direction: in|out, ts,
  body}`. Most-recent first.
- `verdict` from `qualify-lead` — `target` or `not_target`.
- `style_guide` — full `style-guide.md` (for the warm-reply
  pattern reference, not for voice here — this skill only
  classifies).

## Behavior

### Step 1 — Isolate the inbound message

The most-recent inbound message is the primary classification
input. Earlier messages provide context, not the signal.

### Step 2 — Assign a bucket

Buckets:

- **hot** — explicit buying signal: "let's set up a call",
  "send me a time", "yes I'm interested", "I'd like a demo",
  "when can we talk", a specific calendar availability offered.
- **warm** — positive but non-committal: "sure, happy to
  chat", "tell me more", "what does it cost", "how does it
  work", "might be interesting".
  - Sub-type: `polite_yes` — when the reply is a polite
    affirmative that does NOT commit to a next step and doesn't
    ask a specific question. Flagged because the non-ICP polite
    yes is the default failure mode of outreach.
  - Sub-type: `referral` — "not me but try <name>".
- **cold** — neutral or mildly receptive but without signal:
  "got it", "thanks for reaching out", "I'll take a look".
- **skeptical** — engaged but pushing back: "how is this
  different from <competitor>", "we already have X", "I've
  heard this pitch before".
- **decline** — explicit no: "not interested", "please
  remove me", "wrong person", "unsubscribe".

### Step 3 — Flag sub-types

Add optional flags to the return object:

- `meeting_proposed` — the lead named a specific time or asked
  for a time.
- `referral` — the lead redirected to someone else.
- `pitch_back` — the lead used the thread to pitch their own
  service. Always flagged regardless of verdict; `draft-reply-
  interceptly` uses this to decline gracefully.
- `polite_yes` — warm bucket, no commitment, no specific
  question.
- `requested_info` — the lead asked a specific question (case
  studies, pricing, how it works). Not a meeting ask, but
  actionable.

### Step 4 — Return

Return `{bucket: "hot|warm|cold|skeptical|decline",
sub_types: [<flags>], evidence: "<verbatim excerpt from
inbound>"}`.

### Step 5 — Record

Append to the `Classifications` sheet of
`outreach-mirror.xlsx` with ts, lead_url, bucket, sub_types,
evidence. The sheet feeds `outreach-weekly-report` temperature
distribution.

## Semantics — verdict × bucket matrix

| verdict | bucket | draft pattern |
|---|---|---|
| target | hot | Hot — propose 2-3 meeting times (via `propose-meeting-times` if booking_mode=automated or availability_source=gcal) |
| target | warm | Warm-ICP — uses the client's warm-reply pattern from style-guide.md |
| target | cold | Cold bump — short, low-ask |
| target | skeptical | Skeptical — acknowledge pushback, one proof point, redirect to a specific next step (NOT the booking link) |
| target | decline | No draft. Label Not Interested. |
| target | (sub_type=referral) | Referral thank-you + ask for warm intro |
| target | (sub_type=pitch_back) | Graceful exit — see Warm-non-ICP three-option generator |
| not_target | warm / polite_yes | Non-ICP three-option flow (graceful exit / let-it-hang / throwaway question) |
| not_target | cold | No draft. Label Ignore. |
| not_target | decline | No draft. Label Not Interested. |
| not_target | hot | Non-ICP three-option flow with "interested non-target" note — the client may want the graceful-exit path since the lead IS interested, just not a buyer. |

## Failure modes

- **Inbound message is empty** (the lead clicked accept but
  didn't send text). Treat as `warm` with sub-type `empty_body`
  — `draft-reply-interceptly` will produce a light opener that
  doesn't assume a question was asked.
- **Mixed signal** — the inbound has both a decline phrase and
  a buying signal ("not interested in X but curious about Y").
  Classify to `warm` and include both phrases in evidence. The
  draft pattern can handle the nuance.

## What NOT to do

- Do not classify into buckets the verdict × bucket matrix
  doesn't cover. If you reach for a sixth bucket, that's feedback
  — raise it with the operator rather than inventing.
- Do not cache bucket across replies. Every inbound gets its own
  classification.
- Do not read voice here. Voice is `draft-reply-interceptly`'s
  job. This skill is temperature-only.
