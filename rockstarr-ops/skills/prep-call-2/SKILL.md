---
name: prep-call-2
description: "This skill should be used when daily-call-prep classifies a calendar event as Call 2 (close), or when the user says \"prep the close for [name]\", \"call 2 prep for [name]\", or \"second call prep for [name]\". Produces Call2_Prep_[name]_[date].docx with intel recap (verbatim quotes from ops_meeting_recorder + LinkedIn thread), the angle, phase script with mirror lines, pitch tuned to skip rejected framings, price + close + objection handlers. Phase scaffold from ops-call-framework.md; pitch/price from pitch.md (trusted on conflict). Operator-facing — bypasses stop-slop."
---

# prep-call-2

Per-prospect Call 2 (close) prep doc. Produces a docx file at
`/03_drafts/ops/sales-prep/Call2_Prep_[name]_[YYYY-MM-DD].docx`.

The big difference between this skill and `prep-call-1`: this
skill has a recorder transcript and a substantive prior thread
to work from. The doc's value is in how faithfully it surfaces
verbatim quotes from the prior call AND skips pain framings the
prospect already rejected.

The doc is **operator-facing** — bypasses stop-slop. Quotes are
verbatim — the bot does NOT invent quotes.

## When to run

- Auto, by `daily-call-prep` when an event on today's calendar
  classifies as a Call 2 (a prior recorded call exists for this
  prospect OR a substantive prior thread exists).
- On demand by the operator via the trigger phrases above.

## Preconditions

- `00_intake/pitch.md`, `00_intake/ops-call-framework.md`,
  `00_intake/icp-qualifications.md` exist.
- `stack.md` is set with at minimum `ops_meeting_recorder`
  (recorder integration) and `ops_email_outreach_tool`. When
  `ops_meeting_recorder=none`, this skill falls back to thread
  reading only and surfaces the missing-recorder gap clearly in
  the docx.
- Prior recorded call exists in `ops_meeting_recorder` for this
  prospect OR a substantive thread (3+ messages) exists in
  `ops_email_outreach_tool` / `ops_email_platform`.
- Chrome MCP is authenticated against the recorder + outreach
  tool.

## Inputs

Same shape as `prep-call-1`:

- `lead_name`, `lead_company`, `lead_linkedin_url`,
  `event_start`, `booking_source`.

## Behavior

### Step 1 — Pull the prior recorder transcript

Via Chrome MCP, navigate to the recorder's UI (Read.ai, Otter,
Fathom, Fireflies — whatever `ops_meeting_recorder` names).
Search for the prior call by `lead_name` + `lead_company`.
When multiple prior calls exist, pull the most recent one.

The recorder's search / copilot surface is what extracts the
verbatim quotes. The exact tool-specific selector lives in this
skill's `references/recorder-selectors.md` and is consulted
based on `stack.md.ops_meeting_recorder`.

Pull these specific items as VERBATIM strings (no paraphrasing):

- **Pain points the prospect named.** Three to five quotes,
  100-300 chars each.
- **Voiced objections.** Quotes capturing any objection the
  prospect raised on the call — pricing, timing, scope,
  authority, fit.
- **Commitments owed.** Quotes where someone said `I'll send
  you...`, `I'll get back to you with...`, `I'll loop in...`.
  Tag each commitment as `prospect-owed` (operator owes them
  X) vs. `operator-owed` (prospect owes operator Y).
- **Pricing language the prospect used.** Quotes capturing how
  the prospect described the offer's price — rejection,
  acceptance, comparison to alternatives.
- **Pain framings the prospect REJECTED.** Quotes where the
  prospect explicitly pushed back on a framing the operator
  proposed ("that's not really what we're trying to solve",
  "we already tried that").

If the recorder transcript is unavailable (recording failed,
retention window expired, recorder not authenticated), the
skill writes the docx with a clear "RECORDER UNAVAILABLE"
callout at the top and falls back to thread-reading only. It
does NOT fabricate quotes.

### Step 2 — Pull the LinkedIn / email thread

Via Chrome MCP, navigate to `ops_email_outreach_tool` (for
LinkedIn) and `ops_email_platform` (for email). Find any
thread with `lead_name`. Capture the prospect's last 3-5
messages verbatim.

Tag each message with its channel (LinkedIn vs. email) and
its timestamp.

### Step 3 — Build the intel recap section

Render in the docx as four sub-sections:

- **Pain points (verbatim).** Block-quote each.
- **Voiced objections (verbatim).** Block-quote each.
- **Commitments.** Two columns: "you owe them" /
  "they owe you", each with the verbatim quote that captures
  the commitment.
- **Pricing language (verbatim).** Block-quote how the
  prospect described the price last time.
- **Rejected pain framings (verbatim).** Block-quote each.
  This is the section that drives the next step.

### Step 4 — Synthesize the angle

One paragraph: the angle for THIS specific close call. The
synthesis pulls from:

- The pain point that resonated MOST (judged by which one the
  prospect kept returning to).
- The pricing comparison the prospect anchored on.
- The objection the operator did NOT fully resolve last call.

The angle is the operator's read on `how to land this close`
for this specific prospect, synthesized from real quotes.

### Step 5 — Build the phase-by-phase script

Same structural rule as `prep-call-1` (phases come from
`ops-call-framework.md`), with two additions:

**Mirror lines in the prospect's exact words.** For each phase
where the operator names a pain point or recaps the prior call,
render a mirror line using the prospect's exact verbatim
phrasing from Step 1. Example: instead of "you mentioned
hiring is a bottleneck", render "you said
'we just can't hire fast enough' — does that still feel
true?".

**Skip rejected pain framings.** The pitch section in the
phase script does NOT use any framing the prospect rejected in
Step 1's "rejected pain framings" sub-section. The bot
substitutes a different framing from `ops-call-framework.md`
Q5's hook list, picked to avoid the rejected one.

The pricing card uses `pitch.md` Q2 verbatim. Conflict callout
applies if `ops-call-framework.md` differs (same rule as
`prep-call-1`).

### Step 6 — Build sized objection handlers

Read `ops-call-framework.md` Q8 objection handlers. For each
voiced objection captured in Step 1, render the matching
handler from Q8.

If the prospect raised a NEW objection not in Q8 (caught by
matching against the captured Q1 from `pitch.md` and the Q8
handlers), surface it as an open question with "no captured
handler — improvise or re-run `capture-call-framework`."

### Step 7 — Render the close + booking language

Use `ops-call-framework.md` Q9 booking / next-step language
verbatim. This is the language the operator already uses for
booking the next step — no novel phrasing.

### Step 8 — Render the commitment-recap callout

The docx's cover-page-1 callout names the operator-owed
commitments captured in Step 1. The operator should walk in
having delivered on them; the callout is the cross-check.

### Step 9 — Write the docx

Write `/03_drafts/ops/sales-prep/Call2_Prep_[lead_name]_[YYYY-MM-DD].docx`.

The docx structure:

1. **Cover** — `Call 2 Prep — [lead_name]` header.
   Commitment-recap callout (operator-owed only). Event start
   time. Operator name.
2. **Intel recap** — pain points + voiced objections +
   commitments (both directions) + pricing language + rejected
   pain framings, all verbatim.
3. **The angle** — Step 4 synthesis paragraph.
4. **Phase-by-phase script** — phases from `ops-call-framework.md`,
   with mirror lines and skip-rejected-framings tuning.
5. **Pricing** — from `pitch.md` verbatim.
6. **Sized objection handlers** — Q8 handlers per voiced
   objection.
7. **Close + booking language** — Q9 verbatim.
8. **Open questions** — gaps from intel pull, novel objections
   without captured handlers, etc.

### Step 10 — Sidecar markdown

Write a markdown sidecar at the same path with `.md` extension.
Same rule as `prep-call-1`'s sidecar — front-matter only, used
by the orchestrator for the chat-summary line.

> **Sidecar template convention.** Real `---` in the actual
> file, not `# ---`.

```markdown
# ---
schema_version: 1
type: ops-prep-call-2
lead_name: <name>
lead_company: <company>
lead_linkedin_url: <URL>
event_start: <ISO>
prior_call_id: <recorder transcript id>
prior_call_at: <ISO>
icp_verdict: GO | CAUTION | DISQUALIFY     # re-run from new intel
verbatim_quote_count: <integer>
recorder_unavailable: true | false
generated_at: <ISO>
docx_path: <path to the docx>
operator_owed_commitments: <integer count>
prospect_owed_commitments: <integer count>
# ---
```

## Outputs

- `/rockstarr-ai/03_drafts/ops/sales-prep/Call2_Prep_[name]_[date].docx`.
- Markdown sidecar at the same path with `.md` extension.

## Approvals

Same as `prep-call-1` — operator-facing, no `send` step.

## Failure modes

- **Recorder unavailable.** Render docx with a top-of-doc
  "RECORDER UNAVAILABLE" callout. Fall back to thread-only
  intel. Mark `recorder_unavailable: true` in the sidecar so
  the daily-summary line reflects the gap.
- **Recorder retention window expired** (transcript existed
  but was deleted). Same handling as unavailable. Recommend
  the operator update `stack.md.ops_meeting_recorder_retention`
  and consider an export.
- **Prospect's verbatim quotes contain PII** (a personal phone
  number, a customer name they shouldn't have shared). Render
  as quoted, but flag in chat: "Quote includes potential PII
  — consider redacting before publishing the prep doc beyond
  this session." Do not auto-redact — the operator decides.
- **Recorder + thread both empty for this prospect, but
  calendar event still classifies as Call 2.** This usually
  means the orchestrator mis-classified — surface a chat
  warning and recommend re-classification. The skill writes a
  doc that's effectively a Call 1 prep but mis-named, which
  the orchestrator catches in its sweep summary.
- **Recorder search returned multiple prior calls and the
  most-recent one is wrong.** Surface in chat: "Multiple prior
  calls found for [name] — confirm the right one before
  proceeding." Ask the operator which call to use.

## What this skill does NOT do

- Does NOT paraphrase recorder quotes. Verbatim only. When the
  recorder's transcript is unavailable, the skill says so —
  it does not guess.
- Does NOT pull pricing from any transcript. Pricing comes
  from `pitch.md` only. Old recordings often quote stale
  prices; the conflict rule is `pitch.md` wins, every time.
- Does NOT propose a NEW pain framing in the prospect's exact
  words. Mirror lines are surfaced from prior verbatim quotes;
  novel framings are not invented.
- Does NOT run stop-slop. Operator-facing.
- Does NOT update the CRM (post-call processing is a different
  skill).
