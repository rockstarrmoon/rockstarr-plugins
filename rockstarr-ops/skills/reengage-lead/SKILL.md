---
name: reengage-lead
description: "This skill should be used when the user says \"reengage [name]\", \"[name] went dark — try again\", \"cold lead reengagement for [name]\", \"reengage this lead\", \"send a curiosity nudge to [name]\", or when audit-lead Play 3 hands off the workflow. Dead-lead revival. Hard scope: prior recorded call (non-overridable) plus LinkedIn ghosting (non-overridable) plus recent newsletter or sequence engagement signal (overridable with explicit \"reengage anyway\" — the bot flags the missing signal but proceeds). Pulls history plus engagement plus a verbatim pain quote from the recorder, hands the draft to rockstarr-reply with intent_hint=reengagement, sends via the active rockstarr-outreach-* variant with a 5-business-day follow-up timer. No batch-send anywhere; the operator picks one lead at a time."
---

# reengage-lead

Dead-lead revival workflow. Strict scope check, single-lead-at-a-
time. Reengagement only works when there's signal the lead is
still in the orbit; sending a curiosity-nudge to someone who
never opens email wastes trust.

This skill is the standalone version of `audit-lead` Play 3, and
it is also called BY `audit-lead` Play 3 directly. The standalone
trigger and the audit-routed trigger run the exact same workflow.

## When to run

- On demand by the operator with the trigger phrases above.
- Routed by `audit-lead` Play 3 dispatch.

## Preconditions

- `00_intake/pitch.md`, `00_intake/icp-qualifications.md`,
  `00_intake/style-guide.md` exist.
- `style-guide.md`'s warm-reply pattern subsection (captured by
  `rockstarr-outreach-*/skills/capture-warm-reply-pattern/`)
  exists. Without it, the draft falls back to a neutral warm
  pattern — fine, but flagged.
- `rockstarr-reply` is installed (HARD).
- The active `rockstarr-outreach-*` is installed for the LinkedIn
  send path (SOFT — when missing, the send step degrades to a
  Gmail / Outlook draft).

## Inputs

- `lead_name` OR `lead_linkedin_url` OR routed payload from
  `audit-lead`.
- `lead_company` — optional; resolved by lookup.
- `audit_synthesis_path` — set when routed from `audit-lead`;
  otherwise the skill runs its own light intel pull.
- `reengage_anyway` — optional boolean. When `true`, skips the
  engagement-signal scope check (the operator explicitly
  acknowledges the missing signal).

## Behavior

### Step 1 — Resolve the lead

Same resolution pattern as `audit-lead` Step 1.

### Step 2 — Run the scope check

The scope check has three gates:

**Gate A: Prior recorded call exists.** Search
`ops_meeting_recorder` for any prior call with this lead. If
no prior recorded call exists, refuse with:

> No prior recorded call for `<lead_name>`. Reengagement
> requires a prior conversation. Use `cold_bump` instead (run
> through the active outreach variant).

This gate is **non-overridable**. An unrecorded prospect is
not a reengagement candidate by design.

**Gate B: LinkedIn ghosting confirmed.** The lead must have
gone dark on LinkedIn — i.e., the operator's last DM in the
LinkedIn outreach thread is unresponded for 14+ days. If the
lead has replied in the last 14 days, refuse with:

> `<lead_name>` has replied on LinkedIn within the last 14 days.
> Reengagement is for fully-dark leads. Reply to their last
> DM directly via the outreach plugin's `present-for-approval`.

This gate is **non-overridable**. Reengagement messages on
warm threads collide with regular reply drafting.

**Gate C: Recent newsletter / sequence engagement.** Read
`ops_email_platform` (or whatever automated-email tracker the
client uses) for any open / click signal from the lead in the
last 30 days. The bot looks for:

- Newsletter open OR click in last 30 days.
- Marketing sequence email open OR click in last 30 days.

If neither exists, this gate FAILS by default. The operator
can override with explicit `reengage_anyway=true`. Override
behavior:

> No engagement signal in the last 30 days for `<lead_name>`.
> Reengaging dark leads with no orbit signal wastes trust.
> Confirm `reengage anyway` to proceed.

When overridden, the bot proceeds but tags the front-matter
`engagement_signal_missing: true` so the approver sees it.

### Step 3 — Pull intel

When `audit_synthesis_path` is set, read the existing synthesis
— it has all the intel needed.

When this skill is running standalone, do a light intel pull:

- Recorder: most recent transcript. Pull 1-2 verbatim pain-
  point quotes. NOT the full audit pull — just enough to
  anchor the reengagement message.
- Outreach: the lead's LAST DM in the LinkedIn thread, the
  operator's last DM, days-since-ghost.
- Email: any operator-noted commitments since the last call
  (light — don't over-walk).

The intel forms the `intent_context` for the draft handoff.

### Step 4 — Pick the pain quote

From the recorder transcript, pick ONE verbatim pain-point
quote that anchors the reengagement message. Selection logic:

- A pain point the lead said they were ACTIVELY trying to
  solve (use a verb cue: "we're working on", "we're trying
  to", "we need to figure out").
- One that's still likely true given the lead's current
  signal (a recent newsletter open about a topic adjacent to
  the pain point is a strong signal).
- The pain point that appeared MOST often in the recorder
  transcript.

The selected quote becomes the anchor of the draft. The
reengagement message references it verbatim.

### Step 5 — Hand off to `rockstarr-reply`

Build the standard handoff bundle:

```
{
  channel: linkedin-<active outreach variant>,
  thread: <full LinkedIn outreach thread text>,
  persona: <the operator's persona for this client / account>,
  icp_verdict: <re-run from current intel; usually still target
    or the audit re-classified to wrong-fit, in which case route
    to clean-break instead>,
  matching_rule: <verbatim from icp-qualifications.md>,
  lead: { url, name, company, title, campaign_slug },
  intent_hint: reengagement,
  intent_context: <verbatim pain quote from Step 4 + the
    one-paragraph framing from the synthesis>,
  batch_context: false
}
```

Set `batch_context: false` so `rockstarr-reply` fires its
synchronous `notify-reply-ready` urgent email when the draft
lands.

`rockstarr-reply` reads the warm-reply subsection of
`style-guide.md` and applies the captured pattern. Without
that subsection, the draft uses a neutral warm template and
flags `warm_reply_pattern_missing: true`.

### Step 6 — Operator approves via `present-for-approval`

The drafted reengagement is presented in the standard reply-
approval flow. The operator picks one of:

- `send it` (or clear equivalent) — authorize the send.
- An editing instruction ("make it shorter") — triggers a
  redraft via `draft-reply`. Editing instructions are NEVER
  authorization (inherited from `rockstarr-reply`'s gate).
- `flag` — surfaces the lead for human review; does not
  send.

### Step 7 — Send via the active outreach variant

On `send it`, the active `rockstarr-outreach-*`'s
`send-message` skill takes the authorized-send bundle and
executes the LinkedIn DM send via Chrome MCP.

When no outreach variant is installed (soft-degrade), the
send step writes a Gmail / Outlook draft instead and surfaces
"No LinkedIn outreach plugin installed. Drafted to email."
The operator hits send manually.

### Step 8 — Apply label + create follow-up task

After a successful send, the active outreach variant's
`apply-label` and `create-followup-task` skills run:

- Label: the proposed label from
  `rockstarr-reply:follow-up-timer` (typically `Followup`).
- Follow-up timer: **5 business days**. This is the hard-won
  default — sooner reads desperate, later goes cold.

### Step 9 — Log the reengagement

Write a row to
`/05_published/ops/reengagements-<YYYY-MM>.md`:

```markdown
## <YYYY-MM-DD> — <lead_name> @ <lead_company>

- Last real contact: <ISO>
- LinkedIn dark since: <ISO>
- Engagement signal: <newsletter opens last 30d / sequence
  clicks / etc.>, OR `missing — overridden by operator`
- Pain quote anchor: "<verbatim quote>"
- Sent via: <linkedin-interceptly | linkedin-salesnav |
  email-gmail-fallback>
- Follow-up due: <ISO + 5 business days>
- Routed from: standalone | audit-lead Play 3
```

Also append a row to the Reengagements sheet of
`/02_inputs/ops/ops-mirror.xlsx` for `ops-weekly-report` to
roll up reply rate.

## Outputs

- A staged reply draft in
  `/03_drafts/ops/replies/<channel>/<thread-id>.md`
  (via `rockstarr-reply`).
- An approved + sent LinkedIn DM (or email draft on degrade).
- A label + follow-up task in the active outreach variant.
- A row in `/05_published/ops/reengagements-<YYYY-MM>.md`.
- A row in the Reengagements sheet of `ops-mirror.xlsx`.

## Approvals

Strict, inherited from `rockstarr-reply`'s
`present-for-approval`. `send it` or clear equivalent. Editing
instructions are NEVER authorization.

The scope check (Step 2) is its own gate before drafting even
starts. Hard refuses on Gates A and B; soft-overridable refuse
on Gate C with an explicit `reengage anyway`.

## Failure modes

- **No prior recorded call.** Refuse (Gate A). Recommend
  `cold_bump` via the active outreach variant instead.
- **Lead replied recently on LinkedIn.** Refuse (Gate B).
  Route to the standard outreach reply path.
- **No engagement signal.** Soft refuse (Gate C). Operator
  override with explicit `reengage anyway` proceeds.
- **Recorder transcript expired.** Use the synthesis file from
  `audit-lead` if routed; otherwise the bot has no anchor and
  refuses with "No recorder transcript and no audit synthesis.
  Run `audit-lead` first to pull intel."
- **Lead's profile says they changed companies** (caught in
  Step 1's lookup). Refuse with: "Lead is now at `<new
  company>`. Re-target — old context doesn't apply." Operator
  may want to start fresh with a discovery cold-bump rather
  than reengagement.
- **Active outreach variant returns send-failure** (LinkedIn
  rate limited, session dropped). Capture the error, surface
  in chat, write `_errors.md`. The draft stays staged in
  `04_approved/replies/` so the operator can re-trigger the
  send when the channel is back.

## What this skill does NOT do

- Does NOT batch reengagements. The whole point of the scope
  check is one-lead-at-a-time. "Reengage these 50 leads from
  last quarter" is a misuse — the bot refuses batch trigger
  phrases ("reengage all", "batch reengage", etc.) with a
  pointer to this rule.
- Does NOT draft prose itself. Drafts come from
  `rockstarr-reply`.
- Does NOT update the CRM. CRM ride-alongs happen through
  `process-call-transcript` after a meeting lands.
- Does NOT skip the scope check on routed input from
  `audit-lead`. Even when audit Play 3 hands off, the scope
  check runs again — redundancy is the point. The bot does
  not trust upstream skills to pre-validate scope.
