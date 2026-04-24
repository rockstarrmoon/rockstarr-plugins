---
name: flag-for-review
description: "This skill should be used when qualify-lead returns ambiguous or unknown, when present-for-approval returns the 'flag' action, or when the user says \"flag <lead>\", \"I want to look at this one myself\", \"put <lead> on human review\", or \"this thread is ambiguous\". Drops a note into /02_inputs/outreach/_flags.md with the lead + thread + reason, applies the Follow Up label inside Interceptly, and creates a review-reply task so the operator can come back to it. The bot never drafts for a flagged lead until the operator resolves the flag."
---

# flag-for-review

Handoff path. When the bot is not confident enough to draft — or
when the operator just wants to handle this one by hand — the
thread gets flagged, labeled Follow Up, and a task is created.

## When to run

- `qualify-lead` returns `ambiguous` or `unknown`.
- `present-for-approval` returns `action: "flagged"`.
- `classify-reply` returns a bucket + sub_types combo the
  pipeline isn't prepared to handle (e.g., a hostile reply —
  extremely rare, but possible).
- On demand when the user says "flag `<lead>`" — useful when
  they see a thread in the preview queue they want to handle
  personally.

## Preconditions

- `/02_inputs/outreach/_flags.md` exists (create if not).
- Thread is open in Chrome.
- `confirm-session-interceptly` passed in this run.

## Inputs

- `lead_url`
- `thread_id`
- `reason` — free text. Typical values:
  `ambiguous_qualification`, `unknown_qualification`,
  `operator_requested`, `hostile_reply`, `pitch_back`,
  `complex_situation`.
- `context` — optional: what the pipeline knows so far (verdict
  if any, bucket if any, evidence excerpt).

## Behavior

### Step 1 — Append to _flags.md

Append a block:

```markdown
## <ISO timestamp> — <lead_name> (<lead_url>)

- Thread: <thread_id>
- Account: <active_account_label>
- Campaign: <campaign_slug or "(none)">
- Reason: <reason>
- Context: <short summary>

### What the bot saw

<qualify-lead verdict and rule, classify-reply bucket, any
evidence excerpt>

### What the bot needs from you

<a specific question or decision — if qualify-lead ambiguous,
name the missing signal; if hostile reply, suggest handling
options; if operator-requested, note that the operator
asked>
```

Flag order is append-only by timestamp so the _flags.md file is a
review queue.

### Step 2 — Apply Follow Up label

Call `apply-label` with `label = "Follow Up"` (or the
`stack.md.label_mapping` override for "flag"). The Follow Up
label is the universal visual marker for "needs human attention."

### Step 3 — Create a review-reply task

Call `create-followup-task` with:

- `reason = "review-reply"` (a special type — reviewed by the
  pipeline when the operator comes back to the thread)
- `days = 2` business days (overridable via
  `stack.md.followup_timers.flagged_review`)
- `title = "Review flagged lead: <lead_name>"`
- description referencing the `_flags.md` block

### Step 4 — Mirror in Leads

Flip the `Leads` row:

- `last_label = Follow Up`
- `flagged_at = now`
- `flag_reason = <reason>`

### Step 5 — Log to flagged-events

Append to the `Flags` sheet of `outreach-mirror.xlsx`:

| column | value |
|---|---|
| ts | `<ISO>` |
| lead_url | `<URL>` |
| reason | `<reason>` |
| account_label | `<active account>` |
| context | `<short summary>` |

### Step 6 — Return

`{flagged: true, flags_path: "/02_inputs/outreach/_flags.md",
label_applied: "Follow Up"}`

## Resolution path

When the operator reviews a flag:

- If they draft a reply themselves inside Interceptly, they
  should manually remove the Follow Up label (or apply the
  correct final label). No bot cleanup is automatic — the
  mirror will be out of sync until the next time the thread
  enters `process-inbox` or `process-my-tasks`, at which point
  the pipeline re-reads the current Interceptly state.
- If they want the bot to draft again with more context, they
  update `icp-qualifications.md` (or add a `_flags.md`
  resolution note) and re-run the daily loop. The next time the
  thread comes up, `qualify-lead` reads the updated rules.

## What NOT to do

- Do not draft a reply for a flagged lead. The whole point is
  that the bot wasn't confident.
- Do not create multiple review-reply tasks for the same lead.
  If one is already pending, update its due date rather than
  duplicating.
- Do not mark the flag resolved from this skill. Resolution is
  always a human decision.
- Do not silently re-flag a lead on every pipeline run. If
  `qualify-lead` returns ambiguous again, the existing flag row
  stays; don't pile on.
