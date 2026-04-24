---
name: flag-for-review
description: "This skill should be used when classify-reply returns an ambiguous or unknown icp_verdict, when present-for-approval returns the 'flag' action, when draft-reply hits a bucket × sub_types combo it cannot safely draft (hostile reply, unsafe topic, missing critical context), or when the operator says \"flag this lead\", \"I want to look at this one myself\", \"put this thread on human review\". Writes a structured block to /02_inputs/replies/_flags.md and returns a flag response to the caller with { reason, note }. The caller's own apply-label and create-followup-task skills turn that into a Follow Up label + a 2-business-day review task in its channel. rockstarr-reply never drafts again for a flagged thread until the operator resolves the flag."
---

# flag-for-review

Refusal-to-draft handoff. When rockstarr-reply is not confident
enough to draft — or when the operator wants to handle the thread by
hand — flag-for-review writes the thread to the reply flags file and
returns a flag response to the caller. The caller does its own
labeling and task creation.

## When to run

- `classify-reply` is invoked with `icp_verdict=ambiguous` or
  `unknown`. The pipeline routes those verdicts here directly —
  classify-reply does not even assign a bucket.
- `classify-reply` returns a bucket + sub_types combo the pipeline
  isn't prepared to handle (e.g., hostile reply, unsafe topic).
- `draft-reply` encounters a path that requires human judgement
  beyond what the bot can safely exercise.
- `present-for-approval` returns `action: flagged` (the operator
  typed `flag this`).
- On demand when the operator says `flag <lead>` or `put this on
  human review` — useful when they see a thread in a daily preview
  they want to handle personally.

## Preconditions

- `/02_inputs/replies/` exists (created by
  `rockstarr-infra:scaffold-client`). Create `_flags.md` if it
  doesn't exist yet.

## Inputs

- `lead` — `{ url, name, company, title, campaign_slug }` from the
  caller's handoff bundle.
- `channel` — the channel the thread lives on.
- `thread_id` — channel-specific thread identifier (e.g., Interceptly
  thread id, Sales Nav conversation URN, Gmail thread id).
- `reason` — free text from an enumerable set. Typical values:
  - `ambiguous_icp` — `icp_verdict=ambiguous`
  - `unknown_icp` — `icp_verdict=unknown`
  - `operator_requested` — the operator said `flag this`
  - `hostile_reply` — the lead's inbound is hostile / angry
  - `unsafe_topic` — the thread touches a topic the bot shouldn't
    draft on (politics, legal threat, personal crisis)
  - `pitch_back` — the lead used the thread to pitch back (rare
    escalation path; usually draft-reply handles pitch_back with a
    graceful decline rather than flagging)
  - `missing_context` — the thread text scraped by the caller is
    incomplete enough that the bot can't safely draft
  - `cold_bump_exhausted` — the bot has already drafted 3 Cold
    bumps with no reply; surfacing the lead for human review
- `context` — optional one-paragraph summary of what the pipeline
  knows so far (icp_verdict if any, bucket if any, evidence
  excerpt, prior-draft count).

## Behavior

### Step 1 — Append to `_flags.md`

Append a block to `/02_inputs/replies/_flags.md`:

```markdown
## <ISO timestamp> — <lead_name> (<lead_url>)

- Channel: <channel>
- Thread: <thread_id>
- Campaign: <campaign_slug or "(none)">
- Reason: <reason>

### Context

<one-paragraph summary>

### What the bot saw

- icp_verdict: <verdict or "ambiguous / unknown">
- bucket: <bucket or "(not classified)">
- sub_types: [<flags>]
- evidence excerpt: <short excerpt or "(none)">

### What the bot needs from you

<a specific question or decision the operator should resolve.
Examples:
- "Is this lead a target despite the VP Marketing title? The
  company size fits, but the role is outside our usual pattern."
- "This reply is hostile. Do you want to respond, or label Not
  Interested and move on?"
- "3 Cold bumps sent, no reply. Continue with a 4th, label Contact
  Later, or label Not Interested?">
```

The flags file is append-only by timestamp. It becomes a review
queue.

### Step 2 — Return a flag response

Return to the caller:

```json
{
  "kind": "flag",
  "payload": {
    "reason": "<reason>",
    "note": "<short operator-facing note summarising the specific doubt>",
    "flags_path": "/02_inputs/replies/_flags.md"
  }
}
```

The caller's own `apply-label` skill sets the `Follow Up` label in
its channel (from `stack.md.label_mapping` override, if any). The
caller's own `create-followup-task` skill creates a 2-business-day
review task (overridable via `stack.md.followup_timers.flagged_review`
if the caller wants to tune it).

flag-for-review does NOT execute the label or the task. The caller
owns channel I/O.

### Step 3 — Log to `_approvals.log` (as a flag event)

Append to `/04_approved/replies/_approvals.log`:

```
<ISO>  <channel>  <thread_id>  <lead_name> (<lead_url>)  FLAGGED  reason=<reason>
```

This keeps the approval trail complete — flags are a terminal state
for the draft pipeline, same as an authorized send.

## Resolution path

When the operator reviews a flag:

- **Resolve by updating intake.** If the flag was
  `ambiguous_icp` or `unknown_icp`, the operator updates
  `/00_intake/icp-qualifications.md` to tighten the rules. Next time
  the thread enters the pipeline (via the caller's daily loop or
  direct invocation), `classify-reply` reads the updated rules.
- **Resolve by drafting manually.** The operator writes a reply
  directly in the channel UI and the caller's
  `apply-label` / `create-followup-task` cycle runs when the caller
  next scans the channel.
- **Resolve by escalating.** The operator may decide the lead is a
  permanent no-draft (for example, a hostile contact in a competitor
  org). Append a resolution note inside `_flags.md` and the caller
  may optionally add the lead to a per-channel suppression list.

flag-for-review does NOT track resolution state. The file is a queue;
the operator works through it manually.

## Deferred feature — refresh-icp-on-flags

Per the V1 spec, a future `refresh-icp-on-flags` skill will re-run
`classify-reply` on every thread currently in `_flags.md` after
`icp-qualifications.md` is updated — surfacing flags that should no
longer BE flags. Deferred past V0.1 intentionally; first confirm the
flags queue behavior on real pilot traffic.

## Failure modes

- **`_flags.md` doesn't exist and `/02_inputs/replies/` is missing.**
  Create both. If `/02_inputs/` itself doesn't exist, that means
  `rockstarr-infra:scaffold-client` never ran — surface that as a
  hard error instead of creating a half-structure.
- **Caller passes an unrecognized `reason`.** Write the reason
  verbatim to `_flags.md`. The enum in this doc is typical values,
  not a closed set.
- **Same lead flagged repeatedly in one day.** Write a new block per
  flag event; don't dedupe. The queue shape is append-only so the
  operator can see the history.

## What NOT to do

- Do not draft a reply for a flagged lead. The whole point is the
  bot wasn't confident.
- Do not apply the `Follow Up` label yourself. The caller's
  apply-label skill owns channel labeling.
- Do not create the follow-up task yourself. Same reason.
- Do not mark a flag resolved from this skill. Resolution is always
  a human decision written by the operator.
- Do not silently re-flag a lead on every pipeline run. If
  classify-reply returns ambiguous again, the existing flag block
  stays — the new block gets written but the caller's
  `apply-label(Follow Up)` is a no-op if the label is already set.
- Do not touch the caller's own mirror / workbook / state store.
  flag-for-review writes only to `/02_inputs/replies/_flags.md` and
  `/04_approved/replies/_approvals.log`.
