---
name: capture-call-framework
description: "This skill should be used at install time after capture-pitch + capture-deliverability-config, or when the user says \"capture the call framework\", \"set up the sales-call framework\", \"refresh ops-call-framework.md\", or \"the call structure changed\". Interviews the operator on the per-client sales-call framework: phase count + names, pricing card, primary pain hooks, disqualify language, deal-breaker filters. Writes to /rockstarr-ai/00_intake/ops-call-framework.md. prep-call-1 and prep-call-2 refuse to run without it — there is NO baked-in framework in this plugin."
---

# capture-call-framework

Intake interview that produces `00_intake/ops-call-framework.md` —
the per-client single source of truth for the sales-call structure.
`prep-call-1` and `prep-call-2` refuse to run without it.

This is the file that makes the prep skills tool-agnostic. There
is no baked-in phase count, no baked-in pitch, no baked-in pricing
card. The operator's framework drives every prep doc the bot
produces.

## When to run

- Install-time, immediately after `capture-pitch` and
  `capture-deliverability-config` finish.
- On demand, any time the framework changes — a new phase added,
  phase names re-ordered, pricing card refreshed, new pain hook
  surfaced, new deal-breaker filter caught from a recent
  disqualify.
- Both `prep-call-1` and `prep-call-2` REFUSE to run when this
  file is missing.

## Preconditions

- `scaffold-client` has run.
- `capture-stack` has run.
- `capture-pitch` has run — `pitch.md` informs the pricing-card
  section here. The two files cover different scopes:
  - `pitch.md` = current offer, current pricing, current
    positioning (refreshed when the offer changes).
  - `ops-call-framework.md` = how the offer is delivered ON the
    call — phase order, what the operator says in each phase,
    which pain hooks open the conversation, disqualify language.

## Behavior

Walk the interview one question at a time via `AskUserQuestion`.
Pre-fill from any existing `ops-call-framework.md` (HIGH / MEDIUM /
LOW confidence per pre-fill).

The interview defaults are biased toward the patterns most
commonly used by founder-led B2B sales. The operator overrides
freely; nothing here is normative.

### Q1 — Phase count

Single-select `AskUserQuestion`:

"How many phases does your sales call have?" Options: `3` /
`4` / `5` / `Other`.

Most founder-led calls land at 4 (open → discovery → pitch →
close + objection handling). The operator names whatever is
real for them.

### Q2 — Phase names

Free text per phase. The bot asks Q2 once per phase named in
Q1 ("What's phase 1 called?", "What's phase 2 called?", etc.).
Capture the operator's exact phrasing — this is what the prep
docs render.

Common 4-phase pattern (offer to operator if Q1=4 and they're
looking for a starting point):
1. Connection / opening
2. Discovery / pain
3. Pitch + price
4. Close + objection handling

### Q3 — Phase content per phase

For each phase named in Q2, ask one free-text question:
"In 1-3 sentences, what HAPPENS in this phase? What's the
operator's job in this phase?"

This is the content the prep docs render under the phase
heading. The prep skill personalizes the phase text to the
specific prospect — but the structural shape comes from this
captured content.

### Q4 — Pricing card

Free text. "When the operator names the price on the call,
what's the EXACT phrasing? Include any setup-fee + recurring
split, tier names, payment cadence — same level of detail as
`pitch.md` Q2 but framed as the script the operator says
out loud."

This is allowed to differ slightly from `pitch.md` Q2 — the
pitch file captures the offer, this file captures how the
offer is verbalized on the call. Most operators paste-the-same;
some have a more conversational variant they use live.

### Q5 — Primary pain hooks

Multi-select `AskUserQuestion`, plus free-text additions:

"Which pain hooks does the operator typically open the
discovery phase with? (Select all that apply.)"

Pre-fill with common patterns the operator can confirm or
reject. Free-text additions for client-specific hooks:

- `"What's the biggest bottleneck right now?"`
- `"What have you already tried?"`
- `"What does success in 90 days look like?"`
- `"What would 10x of this look like?"`
- `"What's stopping you from doing it yourself?"`
- `Other (free text)`

These hooks are what `prep-call-1` selects from when picking
the primary pain hook for a specific prospect. The prospect's
intel + ICP verdict drive the selection — this question
gives the bot the menu of options.

### Q6 — Disqualify language

Free text. "If the prospect is wrong-fit, what does the
operator say to gracefully end the call? Include the exact
phrasing — `not a fit at this stage`, `come back when X`,
`happy to introduce you to Y`."

The bot uses this verbatim in `prep-call-1`'s clean off-ramp
section AND in `audit-lead` Play 5 (clean break) drafts.

### Q7 — Deal-breaker filters

Free text or short list. "What disqualifies a prospect on the
call? Budget below X, no decision-maker present, wrong
industry, wrong company size — whatever ends the call."

This shows up in both prep docs as a pre-call check.

### Q8 — Common objection handlers

Free text per objection (asked once per objection captured in
`pitch.md` Q7). "When a prospect says [objection], how does
the operator typically respond?"

Captured verbatim. The prep docs render these handlers in the
Phase-N section that covers objection handling. The handlers
are NOT auto-generated — the bot has no opinion here.

### Q9 — Booking / next-step language

Free text. "When the operator closes the call, what's the
exact language for booking the next step? Whether that's
`I'll send a contract`, `let's schedule a follow-up for next
week`, `here's the link to my calendar`, etc."

`prep-call-2`'s close-section uses this verbatim.

## Write the file

Write `/rockstarr-ai/00_intake/ops-call-framework.md`. Overwrite
on re-run.

> **Template convention.** Real `---` in the output file, not
> `# ---`.

```markdown
# ---
captured_at: <ISO timestamp>
schema_version: 1
phase_count: <Q1>
# ---

# Sales-call framework

> Per-client structure consumed by `prep-call-1` and
> `prep-call-2`. Both skills refuse to run without this file.
> Re-run `capture-call-framework` any time the framework
> changes.

## Phase 1: <Q2 phase 1 name>

<Q3 phase 1 content>

## Phase 2: <Q2 phase 2 name>

<Q3 phase 2 content>

(... one section per phase named in Q1/Q2 ...)

## Pricing card

<Q4 verbatim>

## Pain hooks (open discovery with one of these)

<Q5 selections + free-text additions, as a bullet list>

## Disqualify language

<Q6 verbatim>

## Deal-breaker filters

<Q7 verbatim, as a bullet list>

## Objection handlers

### <pitch.md Q7 objection 1>

<Q8 handler 1 verbatim>

### <pitch.md Q7 objection 2>

<Q8 handler 2 verbatim>

(... one section per objection ...)

## Booking / next-step language

<Q9 verbatim>
```

## Outputs

- `/rockstarr-ai/00_intake/ops-call-framework.md`.

## Chained intake

After this skill writes the file, check whether
`/rockstarr-ai/00_intake/icp-qualifications.md` exists. If it
does NOT, this is typically because `rockstarr-reply`'s install
hasn't run yet. In that case, chain into the shared
`capture-icp-qualifications` skill (lives in
`rockstarr-outreach-*/skills/capture-icp-qualifications/`, with
canonical source migrating to
`rockstarr-infra/skills/_shared/capture-icp-qualifications/`).
First bot installed wins the chain — the next bot's install
finds the file already present and skips its own chained call.

## What this skill does NOT do

- Does NOT carry default content for any phase. Phase names and
  phase content are captured from the operator. The bot has no
  opinion on whether a 4-phase or 5-phase structure is "right."
- Does NOT validate the framework against industry best practice
  or any external sales-call rubric. The operator's framework
  IS the framework.
- Does NOT pull from `client-profile.md`'s positioning paragraph
  for the pitch content. The positioning paragraph and the
  pitch + close phases are different artifacts.

## Failure modes

- **Operator answers Q3 in one or two words per phase.** Fine —
  thin content produces thin prep docs, which is feedback for
  refining this skill on the next pass. Do not push back; do
  not auto-pad.
- **Operator picks Q1=Other and answers `it depends`.** Push
  back once: "What's the most common shape?" Accept whatever
  number they give the second time. The bot needs A specific
  framework to render against; "it depends" produces no useful
  prep doc.
- **Operator skips Q5 ("we discover everything live").** Accept
  it; tag the section `# operator declined to capture default
  pain hooks — prep-call-1 will use icp-qualifications.md
  + prospect intel only`. The prep doc will be thinner; that's
  acceptable feedback.
- **Operator's Q6 disqualify language is hostile.** Capture
  verbatim — the operator's voice is the operator's voice. The
  prep doc renders it as written; the operator can re-run if
  they decide it reads worse on the page than it sounds out
  loud.

## Re-running

Overwrite is the default. The bot keeps ONE current version of
the framework. Operators who want a phase-renaming history use
git on the `00_intake/` directory.
