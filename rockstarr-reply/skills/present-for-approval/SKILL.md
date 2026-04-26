---
name: present-for-approval
description: "This skill should be used as step 3 of the rockstarr-reply core pipeline, after draft-reply stages a draft. Trigger phrases: \"show me the draft\", \"present this for approval\", \"what's waiting for my send it\", \"present the three options\". Renders the staged draft with persona, ICP verdict + matching rule, temperature bucket, proposed label, and proposed follow-up timer. Blocks on an explicit 'send it' (or clear equivalent: 'yes send', 'go ahead and send it'). Editing instructions — 'make it shorter', 'try a different angle', 'draft a breakup message', 'continue', 'next' — trigger a redraft via draft-reply, NEVER authorization. On three-option threads, renders all three with NO default pre-selected. On authorization, appends the verbatim 'send it' phrase to /04_approved/replies/_approvals.log and returns an authorized-send bundle to the caller."
---

# present-for-approval

The send-authorization gate. Step 3 of the rockstarr-reply core
pipeline. Every reply stops here. Nothing downstream (send, label,
task) runs until this skill returns `authorized-send` (or the
caller-visible equivalent for picks and skips).

The gate language is preserved verbatim from real operator use.
Weakening it has caused misfires in every bot that handled it
loosely. Don't weaken it here.

## When to run

- Step 3 of the per-reply pipeline, after `draft-reply` stages a
  draft or a three-option bundle.
- On demand when the operator says "show me the pending draft for
  `<lead>`" or "what's waiting for approval."

## Preconditions

- Draft file exists at
  `/03_drafts/replies/<source_channel_slug>/<thread-id>.md` with
  `approval_status: pending` in front-matter (v0.2 contract — the
  pre-v0.2 `awaiting_approval: true` boolean is removed).
- `stop_slop_score` is present in the draft front-matter
  (present-for-approval refuses to present a draft that hasn't been
  scrubbed). If only the legacy `stop_slop_flagged` is present
  without a numeric score, the draft was produced by a pre-v0.2
  draft-reply — surface a one-line note to the operator and proceed.

## Behavior

### Step 1 — Render

Render the operator one message with these fields. No bullet points
where prose reads better; use clear section headers:

- **Lead** — name, title, company, thread URL from the draft
  front-matter.
- **Persona** — the account the draft will sign under (persona.name
  + persona.title from the handoff bundle). If the caller passed
  multiple eligible personas (rare — only on shared LinkedIn
  accounts), prompt for the persona pick BEFORE rendering the body.
- **ICP verdict** — `target | not-target | ambiguous | unknown`
  with the `matching_rule` cited verbatim from
  `icp-qualifications.md` (plus any per-campaign tightening the
  caller supplied).
- **Temperature** — `bucket` + `sub_types`.
- **Proposed label** — the label the caller's apply-label skill
  will likely set after send (from classify-reply's computation,
  reading `stack.md.label_mapping`).
- **Proposed follow-up timer** — the keyword from follow-up-timer
  (`meeting_proposed | general | referral | cold_bump | none`).
- **Draft body** — the post-stop-slop reply body.
  - LinkedIn channels: body only, no signature block shown (the
    sender account IS the signature).
  - Email channels: subject line (if any), body, plus
    `[SIGNATURE_BLOCK]` marker — note that the signature block from
    `persona.signature_block` is injected by the caller's
    `write-draft-<variant>` skill on send.
- **stop-slop flagged?** — `yes` / `no`. If yes, one-line summary
  of what stop-slop rewrote (helps the operator trust the voice).
- **Proof-point pointer** (Skeptical path only) — the first-party
  KB file the draft cites. The pointer is operator-visible; the
  lead sees only the prose.

For Warm-non-ICP three-option drafts, render **all three options**
(graceful exit / let-it-hang / throwaway question) with no default
pre-selected. Show option B as explicitly a no-send action, not a
blank line.

For `book-meeting-handoff` returns from draft-reply, DO NOT render a
body — instead render the slot + lead_fields and ask the operator
to confirm the handoff ("Ready to let book-meeting drive the
booking form?"). A confirmation "yes book it" / "send it to
book-meeting" is treated as authorization; edits trigger a redraft
back to the Hot path.

For `no-action` returns from draft-reply, DO NOT render anything
gate-like. Surface the thread + the `reason` and let the caller log
and move on.

### Step 2 — Wait for authorization

Block until the operator responds. Interpret responses:

**Authorization (sends):**
- `send it`
- `yes send`
- `go ahead and send it`
- `ship it`
- `approved, send`
- any other phrase that explicitly commits to sending. Use
  judgement; if in doubt, treat as NOT authorization and ask.

**Editing instructions (draft-and-re-present, do NOT send):**
- `make it shorter`
- `try a different angle`
- `less formal`
- `continue`
- `next`
- `rework the opening`
- `draft a breakup message`
- any phrase that asks for a different or revised draft.

When you get an editing instruction: call `draft-reply` again with
the instruction as additional context (appended to the pattern
input). The new draft lands in the same file; the front-matter gets
`revision_count` bumped and `last_revised_at` set. Present it
again. Loop until authorization or a different terminal action.

**Pick (Warm-non-ICP three-option flow):**
- `option A` / `graceful exit` — option A body is authorized.
- `option B` / `let it hang` — no send; label Ignore.
- `option C` / `throwaway question` — option C body is authorized.

Option A or C triggers a fresh classify-reply → draft-reply cycle
with `intent_hint=graceful-exit` or
`intent_hint=throwaway-question` (see draft-reply Step 1) — this is
deliberate: the chosen option is rewritten as the actual outbound
body, with its own stop-slop pass. Then present-for-approval runs
again to gate the rewritten body. Option B short-circuits the
pipeline.

**Skip (no send, no label, no task):**
- `skip` / `no action` — close the draft, flip front-matter
  `approval_status: rejected`, `rejection_reason: operator_skip`,
  `resolved_action: skip`. Leave the draft file in
  `/03_drafts/replies/<source_channel_slug>/` so the operator can
  find it later. The next morning's approvals-digest will skip it
  because `approval_status != pending`.

**Flag (route to flag-for-review):**
- `flag` / `flag this` / `I want to look at this one myself` — call
  `flag-for-review` with the lead + thread + reason
  (`operator_requested` unless the operator supplies more context).

### Step 3 — On authorization

- Flip draft front-matter: `approval_status: approved`,
  `authorized_at: <ISO>`, `authorized_by: operator`,
  `resolved_action: send`, `authorization_phrase: <verbatim operator
  phrase>`. (The pre-v0.2 `awaiting_approval: false` is gone — the
  cross-bot digest reads `approval_status` only.)
- Move the draft file to
  `/04_approved/replies/<source_channel_slug>/<thread-id>.md`.
- Append a row to `/04_approved/replies/_approvals.log`:

  ```
  <ISO>  <channel>  <thread_id>  <lead_name> (<lead_url>)  bucket=<bucket>  label=<proposed_label>  timer=<timer keyword>  phrase="<verbatim send-it phrase>"
  ```

- Return to the caller:

  ```json
  {
    "kind": "authorized-send",
    "payload": {
      "body": "<post-stop-slop body>",
      "subject": "<email-only, else null>",
      "proposed_label": "<label>",
      "proposed_followup_timer": "<keyword>",
      "persona_confirmed": { "name": "...", "title": "..." }
    }
  }
  ```

The caller's next step (outreach-* plugin or email-variant
`write-draft-<variant>`) executes its own `send-message`,
`apply-label`, and `create-followup-task`. The label and timer are
suggestions; the caller respects `stack.md.label_mapping` /
`followup_timers` if it has overrides.

### Step 4 — On non-ICP option pick

- If option A or C: treat exactly like an editing instruction that
  redirects the pattern. Return to draft-reply with
  `intent_hint=graceful-exit` or `intent_hint=throwaway-question`;
  the new body comes back through present-for-approval for a fresh
  'send it' gate. The initial three-option presentation does NOT
  authorize sending — it selects the intent.
- If option B: flip front-matter `approval_status: rejected`,
  `rejection_reason: let_it_hang`, `resolved_action: letitthang`,
  move the draft to `/04_approved/replies/<source_channel_slug>/`
  with a note, return:

  ```json
  {
    "kind": "three-option-choice",
    "payload": {
      "pick": "B",
      "action_for_caller": "label_only",
      "proposed_label": "Ignore"
    }
  }
  ```

  The caller applies the Ignore label directly — no send, no
  follow-up task.

### Step 5 — On skip / flag

- Skip: update front-matter `approval_status: rejected`,
  `rejection_reason: operator_skip`, `resolved_action: skip`. Leave
  draft in `/03_drafts/replies/<source_channel_slug>/`. Return
  `{ kind: "no-action", payload: { reason: "operator_skip" } }`.
- Flag: update front-matter `approval_status: rejected`,
  `rejection_reason: flagged`. Hand the lead to `flag-for-review`
  which writes `/02_inputs/replies/_flags.md` (the operator's
  separate review queue — flagged threads are intentionally absent
  from the daily digest). Return
  `{ kind: "flag", payload: { reason: <reason>, note: <flag body> } }`.

### Step 6 — On book-meeting-handoff confirmation

When draft-reply returned `book-meeting-handoff` and the operator
said `yes book it` / `send it to book-meeting`:

- No body is sent by the caller; no reply draft exists to move.
- Append to `_approvals.log`:
  `<ISO>  <channel>  <thread_id>  <lead_name>  book-meeting-handoff
  slot=<slot>  phrase="<verbatim phrase>"`
- Return:

  ```json
  {
    "kind": "book-meeting-handoff",
    "payload": {
      "slot": "<ISO 8601>",
      "lead_fields": { "email": "...", "phone": "...", "...": "..." }
    }
  }
  ```

  The caller's `book-meeting` skill drives the booking-link form.

## The send-gate language — preserved verbatim

The phrases listed above are preserved from the operator-facing
process that has survived real use. Weakening this gate has caused
misfires in the past. If a new phrase arrives that might be
interpreted as authorization, default to **"NOT authorization —
ask."** Silence is never consent.

> The workflow is always: draft → present → wait for 'send it' →
> send. Editing instructions are NEVER authorization.

When in doubt, present and ask. Never send on assumed approval.

## Failure modes

- **Operator gives a mix** ("shorter AND send it"). Always obey the
  editing instruction first — redraft shorter, re-present. Do NOT
  send the pre-edit version.
- **Operator types a paragraph of feedback**. Parse for any
  authorization phrase. If present, treat the rest as editing
  context for AFTER this send — don't apply to the current draft.
  If absent, treat the whole thing as editing instructions.
- **Operator never replies**. The draft sits in
  `/03_drafts/replies/<source_channel_slug>/` with
  `approval_status: pending`. The cross-bot `approvals-digest`
  (rockstarr-infra v0.8+) picks it up via the `approval_status`
  front-matter field on its 6am daily run. The
  `approvals-backlog-alert` (also v0.8+) escalates to a
  Rockstarr strategist when total pending exceeds the threshold.
  Together those two replace the V0.1 "stale drafts get noticed
  weekly" gap.
- **Operator authorizes, but the caller's write-draft or
  send-message skill fails downstream**. The approval is already
  logged and the draft already lives in `/04_approved/replies/`.
  That's the intended design — `_approvals.log` records that the
  operator said yes, even if the channel layer failed to deliver.
  The caller is responsible for surfacing its own delivery failures.

## What NOT to do

- Do not interpret silence, `ok`, `cool`, `sure`, or any ambiguous
  affirmation as authorization.
- Do not send the body shown after an editing instruction arrived.
  The operator always gets a re-presented draft.
- Do not pre-select a non-ICP option. The approver picks all three
  every time.
- Do not send more than one draft per 'send it'. A second send
  requires a fresh approval cycle.
- Do not weaken the gate language under any circumstance. Every
  shortcut has cost a misfire before.
- Do not execute the send itself. present-for-approval returns
  `authorized-send` to the caller; the caller's channel-specific
  write path does the actual delivery.
