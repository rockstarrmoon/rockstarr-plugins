---
name: audit-lead
description: "This skill should be used when the user says \"audit [name]\", \"what's going on with [name]\", \"where are we with the [company] deal\", or \"do an old-lead audit\". Walks four sources in fixed order (recorder then CRM then email then outreach), writes a 5-part synthesis to /02_inputs/ops/audit-[lead-slug].md, recommends ONE of six plays (operator can override), then dispatches via rockstarr-reply (plays 1/2/4/5), reengage-lead (play 3), or ops_calendar (play 6). Four-source order is fixed — inversion mis-frames the synthesis. Recorder quotes verbatim; never invented."
---

# audit-lead

Old-lead audit. The heaviest skill in V0.1. Four-source walk +
five-part synthesis + six-way play selection + dispatch to the
right downstream skill.

The four-source order is fixed: **recorder → CRM → email →
outreach**. Reading them in that order means later signal
(cold LinkedIn ghost) is always interpreted in light of what
the lead said when warm (recorder transcript). Reading them out
of order — starting with the cold ghost — mis-frames the
synthesis. This was hard-won from real use; do not reorder.

## When to run

- On demand by the operator with the trigger phrases above. The
  operator names a lead OR pastes a queue-task URL.
- Auto, when an audit task lands in the task system tagged with
  the configured audit-task tag (rare; most audits are operator-
  triggered).

## Preconditions

- `00_intake/pitch.md`, `00_intake/icp-qualifications.md`,
  `00_intake/style-guide.md` exist.
- `stack.md` has the four sources captured: `ops_meeting_recorder`,
  CRM (via the active `rockstarr-ops-bot-[crm]`),
  `ops_email_platform`, `ops_email_outreach_tool`. Any of which
  can be `none` — the corresponding source is skipped if so.
- `rockstarr-reply` is installed (HARD dependency for plays 1,
  2, 4, 5).

## Inputs

- `lead_name` OR `lead_linkedin_url` OR `task_url`.
- `lead_company` — optional; resolved by lookup.
- `play_override` — optional; the operator can pre-specify a
  play to skip the recommendation step ("audit Jane and run
  play 5").

## Behavior

### Step 1 — Resolve the lead

If `task_url` was passed, read the task to get `lead_name` +
`lead_company`. If `lead_linkedin_url` was passed, scrape the
profile for `lead_name` + `lead_company`. Otherwise use the
passed-in fields directly.

Search the four sources by the resolved identity. Capture the
match key per source:

- Recorder: search by name + company, take the most recent
  recorded call.
- CRM: search by email (most reliable) or name + company.
  Capture the contact ID for the dispatch step.
- Email: search inbox + sent for any thread with the lead's
  email.
- Outreach: search the active LinkedIn outreach tool for any
  thread with the lead's LinkedIn URL.

If no match in any source, the lead doesn't exist for the bot
— surface "No history found for `[name]`. Confirm spelling and
re-try." and exit.

### Step 2 — Walk source 1: recorder

This is the highest-signal source. Most audits hinge on what
the lead said when warm.

Pull the most recent recorded call. Capture verbatim:

- **Pain points the lead named.** 3-5 quotes.
- **Voiced objections.** Quotes capturing every objection the
  lead raised.
- **Commitments.** "Operator owed the lead X" + "lead owed
  the operator Y" — captured as quoted lines.
- **Pricing language.** How the lead described the price.
- **Last words on the call.** Did they say `let's do this` or
  `let me think about it` or `not the right time`?
- **Signal of warmth at end-of-call.** Tag as `warm` (engaged,
  asking next steps), `lukewarm` (open but cautious), `cold`
  (not yet sold).

If the recorder transcript is unavailable (recording failed,
retention window expired), the audit continues with sources
2-4 only. The synthesis renders the recorder gap explicitly:
"RECORDER UNAVAILABLE — synthesis is partial."

### Step 3 — Walk source 2: CRM

Read CRM record via the active `rockstarr-ops-bot-[crm]` (or
manual lookup if no CRM ops bot is installed). Capture:

- Stage / status — current pipeline stage.
- Last contact date — when the operator (or bot) last touched
  the record.
- Operator-owed deliverables — anything in the activity log
  saying `to send`, `to share`, `follow up with`.
- Notes from the operator since the last call.
- Tags — any segmentation tags relevant to the audit (industry,
  campaign source, etc.).

### Step 4 — Walk source 3: email

Read email inbox + sent for any thread with the lead. Capture:

- Lead's last few messages, verbatim.
- Operator's last sent message.
- Days since the last lead-initiated message.
- Days since the last operator-initiated message.
- Any forwarded emails involving the lead (referrals,
  introductions).
- Newsletter / sequence engagement: did the lead open / click
  any operator-sent automated email in the last 30 days?

The newsletter / sequence engagement signal feeds Play 3
(curiosity nudge → reengage-lead) directly.

### Step 5 — Walk source 4: outreach

Read the active LinkedIn outreach tool's thread for the lead.
Capture:

- Lead's last message in the thread, verbatim.
- Operator's last sent DM.
- Days since the last lead-initiated DM.
- Days since the last operator-initiated DM.
- Acceptance state (accepted vs. pending) — relevant for
  threads that never got past connect.
- Any specific labels applied to the thread (the outreach tool's
  status — Not Interested, Bad Fit, Followup, etc.).

### Step 6 — Write the 5-part synthesis

Write `/02_inputs/ops/audit-[lead-slug].md`. The lead-slug is
`[lead_name]-[lead_company]` lowercased, hyphenated.

> **Template convention.** Real `---` in the actual file, not
> `# ---`.

```markdown
# ---
schema_version: 1
type: ops-audit-synthesis
lead_name: <name>
lead_company: <company>
lead_linkedin_url: <URL>
last_real_contact_at: <ISO>
last_real_contact_channel: recorder | email | outreach | crm-note
state_of_relationship: warm | lukewarm | cold | wrong-fit | competing
commitments_owed_by_operator: [<list of verbatim commitment quotes>]
commitments_owed_by_lead: [<list>]
engagement_signal: in-orbit | dormant | gone
real_bottleneck: <one-sentence string>
recommended_play: 1 | 2 | 3 | 4 | 5 | 6
recommended_play_label: fulfill | reframe | curiosity-nudge | third-party | clean-break | wait
play_recommendation_reason: <one-paragraph string>
recorder_unavailable: true | false
generated_at: <ISO>
# ---

# Audit synthesis — <lead_name> @ <lead_company>

## Last real contact

When: <ISO date>
Channel: <recorder / email / outreach / crm-note>
What was said (verbatim): <quote>

## State of the relationship

<warm / lukewarm / cold / wrong-fit / competing>

<one paragraph synthesis. Paint the relationship — are they
still curious, are they politely-distant, are they actively
working with a competitor, are they wrong-fit and the operator
should have walked away?>

## Unfulfilled commitments

### Operator owes the lead
- <verbatim quote of commitment 1>
- <verbatim quote of commitment 2>

### Lead owes the operator
- <verbatim quote>

(If neither side owes anything, render: "No outstanding
commitments — both sides clean.")

## Engagement signal

<in-orbit / dormant / gone>

<sub-paragraph: what's the lead doing in our orbit? Newsletter
opens, LinkedIn likes on operator's posts, replies to nurture
sequences? Or have they gone fully dark?>

## Real bottleneck

<one sentence: what's actually stopping them from saying yes
or no, NOT what they said out loud. The synthesis writer's
read.>

(For example: lead said "no budget", but the real bottleneck
is "no executive sponsor — they can't get the conversation
into the room with the people who control the budget.")
```

The act of WRITING the synthesis is what surfaces the right
play. The bot does not skip the synthesis — even when the
recommended play is obvious, the document forces a clean
read.

### Step 7 — Recommend ONE play

Six plays, judged against the synthesis:

**Play 1: Fulfill.** Operator owes the lead something specific
(commitment from Step 3 or Step 6's CRM-noted deliverable).
Move: deliver the thing + a short note. Recommended when there's
a clean operator-owed commitment AND the relationship state is
warm or lukewarm.

**Play 2: Reframe.** The lead voiced an objection at the prior
call that the operator didn't fully resolve, AND the
synthesis shows the real bottleneck differs from the voiced
objection. Move: reframe the offer to address the real
bottleneck. Recommended when there's an unresolved objection
+ a clear real-bottleneck read.

**Play 3: Curiosity nudge → `reengage-lead`.** Lead has gone
dark on direct contact (LinkedIn DMs, email replies) BUT is
still in the orbit (newsletter opens, recent likes on the
operator's content). This is the only play where audit-lead
hands off to a sibling skill — `reengage-lead` owns the
reengagement workflow. Recommended when engagement_signal =
in-orbit AND last real contact was > 30 days AND state is
lukewarm or cold.

**Play 4: Third-party intro.** Lead is wrong-fit for the
operator's offer but might be the right-fit for someone in
the operator's network. Move: graciously refer to a vetted
peer. Recommended when state = wrong-fit AND the operator has
a captured peer-network.

**Play 5: Clean break.** The relationship is past saving — wrong
fit, hostile last message, or has been dark for 90+ days with
no orbit signal. Move: a short final message that closes the
loop without burning the relationship. Recommended when state
= wrong-fit OR engagement_signal = gone AND last real contact
> 90 days.

**Play 6: Wait.** Real bottleneck is timing — the lead is the
right fit but the moment isn't (mid-funding round, mid-
acquisition, mid-leadership-transition). Move: schedule a
calendar reminder for the right re-touch date. NO message is
drafted. Recommended when state = lukewarm AND the synthesis
identifies a specific timing constraint.

The bot recommends ONE. The recommendation lands in
front-matter as `recommended_play` + `recommended_play_label`,
and the synthesis renders the recommendation reason in chat.

### Step 8 — Operator override gate

Surface the recommendation in chat with a clear one-line ask:

> Recommended: **Play [N] — [label]**.
> <one-sentence reason>.
> Reply with `play [N]` to override or `go` to proceed.

The operator can override with one word: `play 4`, `play 5`,
`go`. Anything else is treated as a question / discussion and
the bot answers without locking in a play.

**No play, no draft.** The dispatch step (Step 9) is gated on
explicit confirmation.

### Step 9 — Dispatch

Once the play is locked in:

| Play | Dispatch |
|---|---|
| 1: Fulfill | `rockstarr-reply` with `intent_hint=audit-fulfill` + the operator-owed deliverable referenced from intake |
| 2: Reframe | `rockstarr-reply` with `intent_hint=audit-reframe` + the synthesis's real-bottleneck paragraph as `intent_context` |
| 3: Curiosity nudge | hand off to `reengage-lead` with the resolved lead identity |
| 4: Third-party | `rockstarr-reply` with `intent_hint=audit-third-party` + the named peer + the suggested fit framing |
| 5: Clean break | `rockstarr-reply` with `intent_hint=audit-clean-break` |
| 6: Wait | create a calendar reminder via `ops_calendar` for the named re-touch date; NO draft |

The handoff bundle to `rockstarr-reply` follows the standard
shape from `rockstarr-reply/skills/draft-reply/SKILL.md`:

```
{
  channel: linkedin-<variant> | email-<variant>,
  thread: <full text from outreach + email>,
  persona: { name, title, signature_block, persona_notes },
  icp_verdict: <re-run from intake — leads that audit out as
    wrong-fit on the audit get icp_verdict=not-target here
    even if their original intake verdict was target>,
  matching_rule: <verbatim from icp-qualifications.md>,
  lead: { url, name, company, title, campaign_slug },
  intent_hint: audit-fulfill | audit-reframe | audit-clean-break | audit-third-party,
  intent_context: <synthesis paragraph the bot wants the draft to anchor on>
}
```

The handoff ALSO sets `batch_context: false` (single-thread
audit), so `rockstarr-reply` fires a `notify-reply-ready` urgent
email when the draft lands.

For Play 3, dispatch hands off to `reengage-lead` with the
resolved lead identity, the synthesis path, and the recommended
re-engagement angle. `reengage-lead` runs its own scope check;
if its scope check fails, audit-lead surfaces the failure to
the operator and proposes a different play.

For Play 6, dispatch creates a calendar event in `ops_calendar`:

- Title: `Re-touch [lead_name] @ [lead_company]`.
- Date: per the synthesis's named timing constraint (e.g., "in
  3 months when funding closes"). Default to 30 days out if no
  specific date.
- Description: link to the synthesis file.

### Step 10 — Log to the publish stream

Write a row to
`/05_published/ops/audits-[YYYY-MM].md` capturing the audit
record:

```markdown
## <YYYY-MM-DD> — <lead_name> @ <lead_company>

- Recommended play: <N> (<label>)
- Operator-locked play: <N> (<label>)
- Last real contact: <ISO>
- State: <state>
- Engagement: <signal>
- Synthesis: <path>
- Draft: <path or `n/a — Play 6`>
```

## Outputs

- `/02_inputs/ops/audit-[lead-slug].md` — synthesis.
- `/03_drafts/ops/replies/[channel]/[thread-id].md` — staged
  draft (when applicable; via `rockstarr-reply`).
- A calendar event in `ops_calendar` (Play 6 only).
- A row in `/05_published/ops/audits-[YYYY-MM].md`.
- A row in the Audits sheet of
  `/02_inputs/ops/ops-mirror.xlsx`.

## Approvals

The audit synthesis itself is operator-facing — no `send`
gate. The PLAY recommendation has an explicit operator gate
(Step 8) before dispatch.

The drafted message (when applicable) flows through
`rockstarr-reply`'s `present-for-approval`. Same `send it`
gate as every prose-producing bot in the Growth OS — no local
relaxation here.

For Play 6, the calendar reminder creation does NOT have a
separate gate — the operator approved Play 6, and the
reminder is the dispatch.

## Failure modes

- **One source unavailable.** Render the synthesis with that
  source flagged. Most audits work fine on 3 of 4 sources;
  recorder-unavailable is the most common gap.
- **Lead has multiple records in the CRM** (duplicates).
  Surface in chat: "Multiple CRM contacts match `[name]` —
  pick one." Do not guess. After resolving, re-run the audit.
- **Synthesis has no clear winner** (state could be lukewarm
  OR cold, real bottleneck unclear). Render the synthesis as
  written and surface "Synthesis is ambiguous — pick a play
  manually." Do not lock in a play.
- **Operator picks a play that's structurally wrong** (e.g.,
  Play 5 clean-break on a lead the operator just had a call
  with last week). Soft-warn but proceed: "This is a clean
  break with someone you just spoke to — confirm intent."
  After confirmation, dispatch.
- **`rockstarr-reply` not installed.** Refuse plays 1, 2, 4,
  5 with a hard pointer. Plays 3 and 6 still work (Play 3
  hands to `reengage-lead`, which has its own
  rockstarr-reply check; Play 6 needs no draft).

## What this skill does NOT do

- Does NOT write a draft itself. Drafts come from
  `rockstarr-reply`. Audit dispatches with an `intent_hint`
  and lets the reply skill do its job.
- Does NOT batch audits. Each lead is audited individually,
  with the operator in the loop on play selection.
- Does NOT auto-update the CRM after the audit. CRM updates
  ride along with the eventual sent message via
  `process-call-transcript` (or via the operator's manual
  CRM action). The audit is a synthesis pass, not a write
  pass.
- Does NOT consult `style-guide.md` directly. The voice live
  in `rockstarr-reply`'s draft path — audit just hands off.
- Does NOT run stop-slop on the synthesis. Operator-facing.
