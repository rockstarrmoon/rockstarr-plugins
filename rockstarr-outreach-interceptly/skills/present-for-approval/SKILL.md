---
name: present-for-approval
description: "This skill should be used after draft-reply-interceptly stages a draft, or when the user says \"show me the draft\", \"present this for approval\", or \"what's waiting for my send it\". Renders the staged draft from /03_drafts/outreach/replies/ with persona, ICP verdict + matching rule, temperature bucket, proposed label, and proposed follow-up timer. Blocks on an explicit 'send it' (or clear equivalent: 'yes send', 'go ahead and send it'). Editing instructions — 'make it shorter', 'try a different angle', 'continue', 'next' — are draft-and-present, NEVER authorization. The send gate language is preserved verbatim because any weakening has caused misfires in the past."
---

# present-for-approval

The send-authorization gate. Every reply draft stops here. Nothing
downstream (send, label, task) runs until this skill returns
"authorized".

## When to run

- Step 4 of the per-reply pipeline, after
  `draft-reply-interceptly` stages a draft.
- On demand when the user says "show me the pending draft for
  `<lead>`" or "what's waiting for approval."

## Preconditions

- Draft file exists at `/03_drafts/outreach/replies/<thread>.md`
  with `awaiting_approval: true` in front matter.

## Behavior

### Step 1 — Render

Show the operator, in one chat message:

- **Lead** — name, title, company, thread URL.
- **Persona** — the account the draft will sign under. If the
  thread could be signed by more than one persona, prompt the
  operator to pick BEFORE rendering the body.
- **Verdict** — `target / not_target / ambiguous / unknown` with
  the rule cited verbatim from `icp-qualifications.md` (+ any
  per-campaign tightening).
- **Temperature** — bucket + sub_types.
- **Proposed label** — the Interceptly label the bot will apply
  after send (from `stack.md.label_mapping` or default taxonomy).
- **Proposed follow-up timer** — from
  `stack.md.followup_timers` (default 2 / 3 / 5 / 5 business
  days for meeting_proposed / general / referral / cold_bump).
- **Draft body** — the post-stop-slop reply body, including
  signature block.
- **stop-slop flagged?** — yes/no. If yes, show a one-line
  summary of what stop-slop rewrote.

For non-ICP warm polite-yes drafts, show ALL THREE options
(graceful exit / let-it-hang / throwaway question) with no
default pre-selected. The operator picks one.

### Step 2 — Wait for authorization

Block until the operator responds. Interpret responses:

**Authorization (sends):**
- "send it"
- "yes send"
- "go ahead and send it"
- "ship it"
- "approved, send"
- any other phrase that explicitly commits to sending. Use
  judgement; if in doubt, treat as NOT authorization and ask.

**Editing instructions (draft-and-re-present, do NOT send):**
- "make it shorter"
- "try a different angle"
- "less formal"
- "continue"
- "next"
- "rework the opening"
- "draft a breakup message"
- any phrase that asks for a different or revised draft.

When you get an editing instruction: call
`draft-reply-interceptly` again with the instruction as
additional context. The new draft lands in the same file; the
front matter gets `revision_count` bumped. Present it again.
Loop until authorization or a different terminal action.

**Pick (non-ICP three-option flow):**
- "option 1" / "graceful exit" — sends option 1 body.
- "option 2" / "let it hang" — no send; label Ignore.
- "option 3" / "throwaway question" — sends option 3 body.

**Skip (no send, no label, no task):**
- "skip" / "no action" — close the draft, flip front matter
  `awaiting_approval: false`, `resolved_action: skip`.

**Flag (route to flag-for-review):**
- "flag" / "flag this" / "I want to look at this one myself" —
  call `flag-for-review`.

### Step 3 — On authorization

- Flip front matter: `awaiting_approval: false`,
  `authorized_at: <ISO>`, `authorized_by: "operator"`,
  `resolved_action: send`.
- Move the draft file to
  `/04_approved/outreach/replies/<thread>.md`.
- Return `{action: "send", draft_path: <path>}` to the caller.
  The caller (`process-inbox` / `process-my-tasks`) calls
  `send-message` next.

### Step 4 — On non-ICP option pick

- If option 1 or 3: handle like authorization — move to approved,
  return `{action: "send", option: 1|3, draft_path}`.
- If option 2 (let-it-hang): flip `resolved_action:
  letitthang`, move to `/04_approved/outreach/replies/` with a
  note, return `{action: "label_only", label: "Ignore"}`. The
  caller applies the Ignore label directly — no send.

### Step 5 — On skip / flag

- Skip: update front matter, leave draft in `/03_drafts/` with
  `awaiting_approval: false` so the approver can find it later
  if they change their mind.
- Flag: hand the lead to `flag-for-review`. Return
  `{action: "flagged"}`.

## The send-gate language — preserved verbatim

The phrases listed above are preserved from the operator-facing
process that has survived real use. Weakening this gate has
caused misfires in the past. If a new phrase arrives that
might be interpreted as authorization, default to "NOT
authorization — ask." Silence is never consent.

> The workflow is always: draft → present → wait for 'send it'
> → send. Editing instructions are NEVER authorization.

When in doubt, present and ask. Never send on assumed approval.

## Failure modes

- **Operator gives a mix** ("shorter AND send it"). Always obey
  the editing instruction first — re-draft shorter, re-present.
  Do NOT send the pre-edit version.
- **Operator types a paragraph of feedback**. Parse for any
  authorization phrase. If present, treat the rest as editing
  context for AFTER this send — don't apply to the current
  draft. If absent, treat the whole thing as editing
  instructions.
- **Operator never replies**. The draft sits in `/03_drafts/`
  with `awaiting_approval: true`. The approvals-digest (future
  rockstarr-infra skill) surfaces it. Until then, stale drafts
  show up in the weekly report.

## What NOT to do

- Do not interpret silence, "ok", "cool", "sure", or any
  ambiguous affirmation as authorization.
- Do not send the body shown after an editing instruction
  arrived. The operator always gets a re-presented draft.
- Do not pre-select a non-ICP option. The approver picks all
  three every time.
- Do not send more than one draft per 'send it'. A second send
  requires a fresh approval cycle.
- Do not weaken the gate language under any circumstance. Every
  shortcut has cost a misfire before.
