---
name: classify-reply
description: "This skill should be used as step 1 of the rockstarr-reply core pipeline, called by any outreach-* plugin or email variant after it has scraped a thread and run its own qualify-lead adapter. Trigger phrases: \"classify this reply\", \"what's the temperature on this thread\", \"is this hot / warm / cold\", \"bucket this inbound\". Reads the thread body (most-recent inbound plus one or two earlier messages for context) and the caller's icp_verdict, returns a temperature bucket (Hot / Warm-ICP / Warm-non-ICP / Skeptical / Cold) plus sub-type flags and a proposed_label computed from the default mapping rules overlaid with stack.md.label_mapping. Does NOT read channel DOM, does NOT send or label. Trusts the caller's icp_verdict — if the caller's qualify-lead has a bug, classify-reply does not catch it."
---

# classify-reply

Channel-agnostic temperature classification. Step 1 of the
rockstarr-reply core pipeline. Given thread text plus the caller's
icp_verdict, returns the bucket that draft-reply will use to pick a
pattern, plus the proposed_label the caller's apply-label skill will
likely set once a send is authorized.

No side effects. No DOM reading. No file writes. Pure analysis over
the handoff bundle.

## When to run

- Step 1 of the per-reply pipeline — always called first, before
  `draft-reply`.
- Re-entered later in the pipeline when a three-option pick is
  resolved (pick A or C): the caller passes back the same thread with
  `intent_hint=graceful-exit` or `intent_hint=throwaway-question` and
  classify-reply confirms the bucket / sub_types before draft-reply
  produces the chosen body.
- Does NOT run when `icp_verdict=ambiguous` or `unknown` — the
  pipeline routes those directly to `flag-for-review` without
  classifying or drafting.

## Inputs

From the handoff bundle supplied by the caller:

- `channel` — `linkedin-interceptly | linkedin-salesnav |
  linkedin-meetalfred | linkedin-dripify | linkedin-waalaxy |
  email-gmail | email-outlook`. Used only to influence one or two
  sub-type heuristics (e.g., an out-of-office auto-reply looks
  different on email vs. LinkedIn). Bucket choice is channel-agnostic.
- `thread` — string containing the full thread text, oldest to
  newest. The most-recent inbound message is the primary signal;
  earlier messages provide context.
- `icp_verdict` — `target | not-target | ambiguous | unknown` with
  `{ matching_rule, evidence }`. The caller runs its own qualify-lead
  adapter and hands the verdict in. classify-reply trusts it.
- `intent_hint` (optional) — `reply | bump | breakup |
  graceful-exit | referral-pivot | throwaway-question |
  book-meeting-followup`. Present when the caller is re-entering the
  pipeline with a specific intent. On a fresh reply this is omitted.

And from the client's intake:

- `/00_intake/icp-qualifications.md` — read for sanity-checking the
  verdict against the client's own rules. classify-reply does NOT
  override the caller's verdict, but if the rules contradict the
  verdict in an obvious way, it flags the conflict in `evidence` so
  the operator sees it at approval time.
- `/00_intake/stack.md` — the `label_mapping` section, used to
  overlay any per-client overrides on top of the default label
  taxonomy.

## Behavior

### Step 1 — Isolate the inbound signal

The most-recent inbound message is the primary classification input.
Earlier messages provide context, not the signal.

If the thread is empty (lead accepted a connection request but sent
no text), treat as `Warm-ICP` for `icp_verdict=target` or
`Warm-non-ICP` for `icp_verdict=not-target`, with
`sub_type=empty_body`. draft-reply will produce a light opener.

### Step 2 — Assign a bucket

Buckets (the five the spec defines):

- **Hot** — explicit buying energy. Asks about price, timing,
  process, or "tell me more." Specific calendar availability offered.
  Phrases: "let's set up a call", "send me a time", "I'd like a
  demo", "when can we talk", "what does it cost".
- **Warm-ICP** — positive but non-committal, FROM a verified target
  (`icp_verdict=target`). Phrases: "sure, happy to chat", "sounds
  good", "absolutely", "might be interesting", "tell me more" when
  it's closer to conversation than buying.
- **Warm-non-ICP** — same warm acknowledgment, but the caller
  verified NOT a target (`icp_verdict=not-target`). This is the
  default failure mode of LinkedIn — a real human says yes to a soft
  opener even when they're not a buyer.
- **Skeptical** — engaged but pushing back, or explicit decline.
  Phrases: "not interested", "we already have X", "please remove
  me", "wrong person", "I'm all set", "selling the business". The
  Skeptical bucket covers both pushback AND flat decline — the
  draft-reply handler branches on tone.
- **Cold** — lead has not responded to prior touches (no inbound),
  or a neutral non-signal like "got it", "thanks for reaching out",
  "I'll take a look."

### Step 3 — Flag sub-types

Add optional flags to the return object. These are secondary signals
draft-reply uses to pick the exact pattern:

- `meeting_proposed` — the lead named a specific time or asked for a
  time.
- `referral` — the lead redirected to someone else ("not me but try
  <name>"). Overlays on any bucket.
- `pitch_back` — the lead used the thread to pitch their own service.
  Always flagged regardless of verdict; draft-reply uses this to
  produce a graceful decline.
- `polite_yes` — Warm bucket, no commitment, no specific question.
  The three-option trigger on the `not-target` side.
- `requested_info` — the lead asked a specific question (case
  studies, pricing, how it works). Not a meeting ask, but actionable
  — draft-reply addresses the question directly.
- `empty_body` — accepted-without-text (see Step 1).
- `out_of_office` — auto-reply pattern ("I'm out until X", "on
  vacation", "limited email access"). Triggers `no-action` downstream.
- `already_booked` — lead references an existing booking in the
  thread ("looking forward to Thursday's call"). Triggers `no-action`.

### Step 4 — Compute the proposed_label

Start from the default mapping rules (see table below). Then apply
per-client overrides from `stack.md.label_mapping` — keys in stack.md
win over defaults.

Default mapping:

| Situation | Proposed label |
|---|---|
| Hot / explicit buying energy | `INTERESTED` |
| Warm-ICP polite yes / soft ack | (no label change — leave to caller) |
| Warm-non-ICP polite yes | `Ignore` (after option A / B / C resolves) |
| Skeptical — explicit decline | `Not Interested` |
| Skeptical — pushback, referral pattern available | `Referral` if the client opts into a referral pivot; else `Not Interested` |
| Cold non-signal | (no label change — Cold bump is mid-conversation) |
| Lead booked a meeting (via `mark-booked`) | `Booked` — set by the caller, not proposed here |
| Ambiguous / unknown verdict | `Follow Up` (from flag-for-review, not this skill) |
| Lead signaled bad fit without declining | `Bad Fit` |
| Lead asked for a later reconnection | `Contact Later` |

If `stack.md.label_mapping` has any of these keys set to a non-default
value, use that.

### Step 5 — Return

Return a pure JSON-shaped object to the caller / to draft-reply:

```json
{
  "bucket": "Hot | Warm-ICP | Warm-non-ICP | Skeptical | Cold",
  "sub_types": ["<flag>", "..."],
  "proposed_label": "<label from Step 4>",
  "evidence": "<verbatim excerpt from the inbound that drove the bucket choice>",
  "icp_verdict_echo": "<target | not-target>",
  "matching_rule_echo": "<the rule the caller cited>",
  "rules_conflict": false | "<summary if classify-reply spotted a conflict between the verdict and icp-qualifications.md>"
}
```

No side effects. Nothing written to disk.

## Semantics — verdict × bucket matrix

The bucket × verdict combination determines the draft-reply path:

| icp_verdict | bucket | draft-reply path |
|---|---|---|
| target | Hot | Hot — propose 2-3 meeting times via `propose-meeting-times` |
| target | Warm-ICP | Warm-ICP — reads style-guide.md warm-reply pattern |
| target | Cold | Cold bump — short, low-ask, references one concrete signal |
| target | Skeptical | Skeptical — optional referral pivot if style-guide has the subsection; else silent-label `Not Interested` |
| target | (sub_type=referral) | Referral thank-you + ask for warm intro |
| target | (sub_type=pitch_back) | Graceful exit (same generator as non-ICP option A) |
| not-target | Warm-non-ICP | Three-option flow (graceful exit / let-it-hang / throwaway question) |
| not-target | Cold | `no-action` — label Ignore, no draft |
| not-target | Skeptical | `no-action` — label Not Interested, no draft |
| not-target | Hot | Three-option flow with "interested non-target" note — the client may want the graceful-exit path since the lead IS interested, just not a buyer |
| (any) | (sub_type=out_of_office OR already_booked) | `no-action` — caller logs and moves on |

## Failure modes

- **Inbound message is empty** — handled in Step 1 via
  `sub_type=empty_body`. Don't refuse to classify.
- **Mixed signal** — the inbound has both a decline phrase and a
  buying signal ("not interested in X but curious about Y"). Classify
  to Warm-ICP (or Warm-non-ICP per verdict) and include both phrases
  in `evidence`. draft-reply handles the nuance.
- **Verdict contradicts icp-qualifications.md** — e.g., caller said
  `target` but the thread shows the lead is a competitor. Do NOT
  override the verdict. Return `rules_conflict` with a one-line
  summary; draft-reply surfaces it to the operator at approval time.
- **`intent_hint` present but inconsistent with the thread** — e.g.,
  caller passes `intent_hint=book-meeting-followup` on a thread where
  no meeting was proposed. Return the bucket as if `intent_hint` were
  absent and note the inconsistency in `evidence`.

## What NOT to do

- Do not read the channel DOM. Thread scraping is the caller's job.
- Do not write to any file. All persistence downstream of this skill
  is draft-reply's responsibility.
- Do not send or label. classify-reply PROPOSES a label; the caller
  DISPOSES.
- Do not override the caller's icp_verdict. Trust the verdict; flag
  the conflict; let draft-reply / the operator decide.
- Do not cache bucket decisions across replies. Every inbound gets
  its own classification.
- Do not invent a sixth bucket. If a thread resists all five, return
  `Warm-ICP` or `Warm-non-ICP` per verdict with a `sub_types` note —
  and raise it with the operator as feedback for a future refinement.
