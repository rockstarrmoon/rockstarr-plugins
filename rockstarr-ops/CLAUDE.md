# rockstarr-ops (developer notes)

This file is for Claude Code working on this plugin's source. For the
full skill catalog, the "what this bot does NOT do" boundary, the
hard rules, the cross-bot handoff bundle shapes, and the schedule,
read `README.md` in this folder. This CLAUDE.md is its
developer-perspective complement.

## Why this plugin is different from the rest

It is **a pure orchestrator**. It does not draft customer-facing prose
(hands off to `rockstarr-reply`). It does not write to the CRM (hands
off to the active `rockstarr-ops-bot-<crm>` variant). It does not send
LinkedIn DMs (hands off to the active `rockstarr-outreach-*` variant).
What it owns is the *sequence* — which sources to consult in which
order, which play to recommend, which fields to pull from a
transcript, which prep doc to build for which calendar event.

Three properties follow:

1. **Most "logic" in this plugin is sequencing and synthesis, not
   I/O.** A change to drafting tone goes in `rockstarr-reply`; a
   change to CRM write semantics goes in the relevant ops-bot
   variant; a change to LinkedIn DM mechanics goes in the relevant
   outreach variant. Changes here are about *what we look at*,
   *what we recommend*, and *what we hand off*.
2. **Cross-bot handoff bundles are the primary contract.** Three
   bundles (HARD to reply; SOFT to ops-bot-`<crm>` and to
   outreach-*). Most cross-plugin coordination this plugin does
   passes through one of those shapes.
3. **The "hard rules" section in the README is real.** Eight
   non-negotiables, each one earned from real operational use.
   If you find yourself wanting to weaken one, that's the moment
   to stop and ask — these rules failed in their previous, weaker
   forms.

## What this plugin explicitly does NOT do

Repeating the boundary from the README because it's load-bearing
for reasoning about this plugin:

- **Draft prose.** Every audit recommendation, reengagement
  message, post-call recap, and post-call follow-up email goes
  through `rockstarr-reply` via the standard handoff bundle with
  an `intent_hint`.
- **Write to the CRM.** Every structured CRM update goes through
  the active `rockstarr-ops-bot-<crm>` variant. When no CRM ops
  bot is installed, the bundle becomes a manual punch list in
  chat — **never a silent skip**.
- **Send LinkedIn DMs.** Every LinkedIn DM goes through the
  active `rockstarr-outreach-*` variant via its `send-message`.
  When no outreach plugin is installed, the relevant send paths
  degrade to email-draft instead.

If you find yourself reaching for one of these — drafting prose
inline, calling a CRM API directly, sending a DM via Chrome MCP —
stop. The right answer is a handoff.

## Skill groupings (mental map)

13 skills sort into six groups:

1. **Intake / configuration capture** — `capture-pitch`,
   `capture-deliverability-config`, `capture-call-framework`.
   Each writes a config file under `00_intake/`. Other skills
   in this plugin refuse to run without them.
2. **Daily orchestration** — `daily-call-prep` (the 06:00 local
   sweep) dispatches to per-event skills: `prep-call-1` (Call
   1 / discovery), `prep-call-2` (Call 2 / close),
   `build-client-agenda` (recurring catch-ups).
3. **Lead operations (on-demand)** — `audit-lead` (old-lead
   audit, 4-source walk, 6 plays), `reengage-lead` (dead-lead
   revival with strict scope).
4. **Post-call processing (event-driven)** —
   `process-call-transcript` (walks the post-call queue,
   per-task gates).
5. **Deliverability (recurring)** — `run-deliverability-test`
   (9-step weekly spam-score check).
6. **Reporting (weekly)** — `ops-weekly-report`,
   `backup-workbook-ops`.

## Hard rules — do not weaken

These are from the README. Surfacing here because they're the most
likely things a refactor will subtly break:

1. **No batch-send anywhere.** Every customer-facing message ships through one explicit `send it`. Batch reengagement is a misuse of the workflow.
2. **`pitch.md` over the transcript on every conflict.** Old recordings quote stale pricing; the bot trusts `pitch.md`. Every prep / audit / recap-note draft + every reengagement message respects this on conflict.
3. **Quotes from the recorder are verbatim.** No invented quotes, no paraphrasing what the lead said. When the transcript is unavailable, say so plainly rather than guessing.
4. **Four-source order in `audit-lead` is fixed.** Recorder → CRM → email → outreach. Reading them out of order mis-frames the synthesis. Hard-won.
5. **Six audit plays. No more, no less.** Pile-on is a misuse — it produces noise the lead reads as desperation.
6. **`process-call-transcript` writes via `rockstarr-ops-bot-<crm>`, never via Chrome MCP directly.** The CRM ops bot is the single point of audit for every CRM mutation in the Growth OS.
7. **Internal docs (prep / agenda / audit synthesis / recap-note draft) bypass stop-slop.** Stop-slop is for what the LEAD reads, not what the operator reads. Customer-facing prose handed to `rockstarr-reply` DOES run through stop-slop inside reply.
8. **`pitch.md`, `ops-call-framework.md`, and `deliverability-config.md` must exist before the relevant skills run.** The intake captures aren't optional setup; they're the source of truth.

## Cross-bot handoff bundles

This plugin sends to three other plugin families. Bundle shapes are
the primary cross-plugin contract.

### → `rockstarr-reply` (HARD)

Standard handoff bundle from `rockstarr-reply` (see that plugin's
CLAUDE.md for the spec). This plugin uses six `intent_hint` values:

- `reengagement` — from `reengage-lead`.
- `post-call-recap` — from `process-call-transcript`.
- `audit-fulfill` — from `audit-lead` Play 1.
- `audit-reframe` — from `audit-lead` Play 2.
- `audit-clean-break` — from `audit-lead` Play 5.
- `audit-third-party` — from `audit-lead` Play 4.

(Audit Play 3 hands off to `reengage-lead`. Play 6 creates a calendar
reminder via `ops_calendar` — no draft.)

When adding a new ops flow that produces customer-facing prose, add
a new `intent_hint` value and route through `rockstarr-reply`.
Don't draft inline.

### → `rockstarr-ops-bot-<crm>` (SOFT)

Same bundle shape across every CRM variant:

```
{
  contact_id: <id> OR { first_name, last_name, email, phone, business_name },
  field_updates: { <field_name>: <value>, ... },
  note_payload: { sections: [...], links: [...] },
  status_transition: <enum value, optional>
}
```

Today the active variant is `-ga` (The Growth Amplifier). Roadmap:
`-hubspot`, `-pipedrive`, `-salesforce`, `-close`. Every variant accepts
the same bundle.

The "SOFT" designation means: when no CRM ops bot is installed, the
bundle renders as a manual punch list in chat. Never silently
skipped.

### → `rockstarr-outreach-*` (SOFT)

The same `authorized-send` bundle `rockstarr-reply:present-for-approval`
returns. The active outreach variant's `send-message` takes over.
When no outreach plugin is installed, the LinkedIn DM step degrades
to a Gmail / Outlook draft instead.

## The intake config files — load-bearing

Three files this plugin captures during onboarding. Other skills
refuse to run without them:

- `00_intake/pitch.md` — current offer / pricing / positioning.
  Single source of truth. Every prep / audit / recap-note draft +
  every reengagement message reads it. Trusted over transcripts
  on conflict.
- `00_intake/ops-call-framework.md` — per-client call phases /
  pricing card / pain hooks / disqualify language / deal-breaker
  filters. Both prep skills require it.
- `00_intake/deliverability-config.md` — sender / tracker URL /
  test segment / template clone source / deliverability task URL
  / comment format. `run-deliverability-test` requires it.
  Skipped at intake when `ops_deliverability_tool=none`.

If you touch any of the three capture skills, verify the file
shapes they produce still match what the consumer skills read.
Schema drift here surfaces as cryptic refusals from downstream
skills.

## The ops mirror — audit, not source-of-truth

`/rockstarr-ai/02_inputs/ops/ops-mirror.xlsx` is this plugin's
small audit log. Like the interceptly mirror, it is **not
authoritative**. The CRM is authoritative. The recorder is
authoritative. The calendar is authoritative. The mirror exists
so `ops-weekly-report` and stale-item callouts have something to
read.

This plugin owns every sheet (Reengagements, PostCalls,
Deliverability, Audits). Light by design in V0.1; expect modest
schema additions as more flows mature.

## What's high-risk to change in this plugin

- **The eight hard rules.** Each one is a guardrail against a
  real failure mode. Weakening any without a clear reason is a
  regression.
- **The cross-bot bundle shapes.** Three contracts; consumers
  across multiple plugins.
- **The four-source order in `audit-lead`.** Inversion produces
  bad synthesis.
- **The six-play audit framework.** Adding a seventh play is a
  bigger decision than it looks — pile-on is what we're
  protecting against.
- **The intake config file schemas.** Other ops skills refuse
  to run without them.
- **The recorder-quote-verbatim rule.** Paraphrasing a lead is
  silently corrosive.
- **The `pitch.md` > transcript precedence.** Old recordings
  quote stale pricing; the bot must trust `pitch.md`.

## What's safe to change without much ceremony

- Internal synthesis logic within an existing play / prep doc /
  agenda structure, as long as the output sections and the
  handoff bundle shape are preserved.
- Calendar event classification logic in `daily-call-prep`.
- Stale-item thresholds (currently 5 business days for
  out-for-review).
- Wording of operator-facing prompts, run summaries, and
  weekly-report narrative sections.
- New optional `stack.md` ops config fields.
- New `intent_hint` values when adding a new flow (paired with
  consumer-side handling in `rockstarr-reply`).

## Versioning

This plugin is V0.1 (April 2026). Major version axis: the
cross-bot bundle shapes, the eight hard rules, the intake
config schemas, the audit play set. Minor: new skills, new
optional config, new intent_hint values, new mirror sheets.

Bump via `/bump rockstarr-ops <new-version>` at the repo root.

## Testing your changes

Per-skill, because the shapes vary:

- **Intake skills** — run against a fresh empty `00_intake/`
  folder; verify the produced file matches the documented schema.
- **Daily orchestration** — needs a real client's calendar with
  classifiable events; or run the per-event skills (`prep-call-1`,
  `prep-call-2`, `build-client-agenda`) standalone with a
  hand-built event context object.
- **`audit-lead` and `reengage-lead`** — these dispatch heavily
  to other plugins. Easiest test path is a dry-run mode that
  prints the handoff bundle instead of dispatching, then a full
  end-to-end test with a real lead.
- **`process-call-transcript`** — needs a real transcript URL +
  the configured queue tag in the task system. Test against a
  fixture transcript first.
- **`run-deliverability-test`** — needs the deliverability tool
  (MailReach / GMass / similar) plus a configured test segment.
  Hard to test without the real tool.

CI in this monorepo only checks skill-name uniqueness.

## When you're stuck

Ask Jon in the PR. Hard-rule questions especially — those failed
in their weaker forms first, and the current form is the lesson.
