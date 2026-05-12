---
name: intake-platinum-message
description: "This skill should be used when the user asks to \"run the platinum message step\", \"build the platinum message\", \"draft the elevator pitch\", \"capture the messaging\", or when run-intake dispatches to the platinum-message step of the intake flow. Reads the three upstream artifacts (icp.md, transformations.md, competitors.md), captures the client's desired tone, and walks through a per-ICP loop: draft three pitch options, self-validate each against the four principles (Appeal, Exclusivity, Clarity, Credibility), revise any failure before presenting, capture the client's pick as the canonical Platinum Message, then draft three outcome statements with the same validation. Always emits one ### <ICP> — Platinum message H3 per ICP, even when there is one ICP. One question at a time, in the unified intake voice. Checkpoints every answer to /00_intake/intake/platinum-message.md."
---

# intake-platinum-message

Produce the client's canonical Platinum Message and outcome statements
per ICP. Three stages:

- **Stage 1 — Anchor.** Read company name from upstream and confirm.
  Read the ICP list from `intake/icp.md` and confirm.
- **Stage 2 — Tone.** Capture the client's desired tone (one or more
  of: corporate, professional, friendly, direct, bold,
  conversational, plus free-text). Applied workspace-wide.
- **Stage 3 — Per-ICP messaging loop.** For each ICP, draft three
  elevator-pitch options, self-validate against the four principles
  (Appeal / Exclusivity / Clarity / Credibility), revise any
  failure before presenting, capture the client's pick. Then draft
  three outcome statements in Outcome / Promise format, validate,
  capture the picks.

This is one of six intake sub-skills inside `rockstarr-infra`'s
v0.9.x intake flow. Writes `/rockstarr-ai/00_intake/intake/platinum-message.md`.
`compile-profile` reads that file to assemble the **Messaging**
section of `client-profile.md`.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule comes from there. The spirit
of the original Platinum Message GPT is preserved (the four
principles, the Outcome / Promise format, the "no buzzwords, no em
dashes, lead with outcomes" style rules), but the unified intake
voice applies from question one — no corporate-consulting register.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` after `intake-competitors` completes.
- Standalone, when an operator re-runs the platinum-message step
  via `run-intake`'s "redo step N" path.
- Standalone, when an operator says "redo the messaging" or "I
  want to refresh the pitch."

## Preconditions

- `/rockstarr-ai/00_intake/intake/icp.md` with `status: complete`.
  Per-ICP messages branch on the captured ICP list; the Phase B
  reframed positioning is a primary pre-draft source.
- `/rockstarr-ai/00_intake/intake/transformations.md` with
  `status: complete`. The top transformations supply the
  credibility evidence each pitch needs.
- `/rockstarr-ai/00_intake/intake/competitors.md` with
  `status: complete`. The Brand Edge and Differentiation Summary
  supply the exclusivity evidence each pitch needs.
- `/rockstarr-ai/00_intake/intake/` writable.
- `intake-interviewer-voice.md` on disk inside the plugin.

When all three upstream artifacts aren't yet `complete`, refuse to
run with a clear message naming the missing step. The Platinum
Message exercise depends on all three — it's the synthesis step,
not a standalone.

When `intake/platinum-message.md` already exists with
`status: in_progress`, this is a resume. Read the file, check
`icps_complete`, pick up at the next un-messaged ICP.

When the file exists with `status: complete`, ask: "Platinum
message is already captured. Redo from scratch, redo one ICP, or
exit?" Default action is exit.

## The four principles

Every pitch and every outcome statement must meet all four. The
skill self-validates each draft before presenting to the client.
When a draft fails any principle, the skill revises the draft (one
revision pass internally) before showing it.

| Principle    | Test the draft answers                          | Common failure mode                                                |
|--------------|-------------------------------------------------|--------------------------------------------------------------------|
| Appeal       | "I want it." Does it state a wanted outcome?    | Draft describes the activity instead of the outcome.               |
| Exclusivity  | "I can't get it anywhere else."                 | Draft names a generic capability competitors also have.            |
| Clarity      | "I understand it."                              | Draft uses buzzwords / jargon / vague abstractions.                |
| Credibility  | "I believe it."                                 | Draft makes a claim with no evidence anchor (number, named client, |
|              |                                                 |   methodology, duration).                                          |

The validation is recorded inline in the artifact for each captured
message — a four-cell checkbox table — so the audit trail is
visible. The principles do NOT appear in `client-profile.md`; they
live in the intake artifact only.

## The three stages

### Stage 1 — Anchor

Two `AskUserQuestion` turns:

1. **Confirm company.** Read company name + URL from
   `intake/competitors.md` Company anchor (or
   `intake/background.md` once that ships, or
   `client-profile.md` from a prior run). Present and ask:
   confirm / edit / no URL yet.

2. **Confirm ICP list.** Read the ICP names from `intake/icp.md`
   Phase A. Present a one-sentence recap of each. Same three
   options.

Append both confirmations to the artifact under `## Company
anchor` and `## ICPs in scope`. Mark `anchor` in `stages_complete`.

### Stage 2 — Tone

One `AskUserQuestion` with multi-select options:

- **Tone descriptors.** Six checkbox options the client picks one
  or more of: corporate, professional, friendly, direct, bold,
  conversational. Plus a "Other (specify)" free-text option for
  combinations the six don't cover.

Then one follow-up free-text turn:

- **Anything to add?** Open prompt for tone nuance not captured
  by the descriptors ("not too aggressive — we sell to
  hospital systems"). Optional; skip is allowed.

The captured tone is workspace-wide. It applies to every ICP's
messages in Stage 3. When a client genuinely needs different tones
per ICP, they can redo the skill per ICP — but flag this as
unusual; most clients want one tone across the board.

Append to the artifact under `## Desired tone`. Mark `tone` in
`stages_complete`. Save.

### Stage 3 — Per-ICP messaging loop

For each ICP from `intake/icp.md` Phase A, in the order they were
captured. Per ICP, run the pitch flow then the outcome-statements
flow.

#### 3a. Pitch flow

**Draft three elevator-pitch options.** Read the per-ICP
context:
- Phase A pain points → what hurts.
- Phase A transformation sought → what changes.
- Phase B reframed positioning → how the client should describe
  what they deliver.
- `transformations.md` top transformations → evidence (the
  Credibility anchor).
- `competitors.md` Brand Edge + Differentiation Summary →
  exclusivity (the Exclusivity anchor).
- Stage 2 tone → register.

Compose three distinct pitches. Each in the form: a short
paragraph (two to four sentences), copy-paste-ready, in the
captured tone. Each pitch leads with the outcome the client
delivers, not the activity.

**Self-validate.** For each of the three drafts, check the four
principles. When any principle fails, revise the draft once
internally:
- Appeal fails → restate from the buyer's outcome, not the
  client's activity.
- Exclusivity fails → swap a generic capability for a specific
  one named in Brand Edge or the Differentiation Summary.
- Clarity fails → strip buzzwords; rephrase in plain spoken
  language.
- Credibility fails → add an evidence anchor from
  `transformations.md` (a number, a named outcome, a
  methodology).

After the internal revision pass, run `stop-slop` over all
three drafts.

**Present.** One `AskUserQuestion` with five options:

- **Pitch 1** — capture as the canonical Platinum Message for
  this ICP.
- **Pitch 2** — capture as the canonical.
- **Pitch 3** — capture as the canonical.
- **Iterate** — none of the three lands; the skill regenerates
  with the failure pattern the client points to. Cap at three
  iterate rounds per ICP; on the third, fall back to "write
  from scratch."
- **Write from scratch** — client supplies the message
  directly. Skill validates the four principles on the
  client's text; flags any failure as a follow-up
  `AskUserQuestion` rather than rewriting. Stop-slop runs on
  the client's text only when the client says go ahead and
  polish.

**Record principles validation.** Capture the four-cell
checkbox table for the chosen pitch under the ICP's H3.

#### 3b. Outcome statements flow

After the pitch is captured, draft three **outcome
statements** in the canonical format:

> We create [outcome] for [client type] [qualifier].

Or a near-variant:

> We help [client type] [achieve outcome] [qualifier].

Each statement is one sentence. Each must meet the same four
principles. Same self-validate + revise + stop-slop discipline
as the pitch.

**Present.** Three separate `AskUserQuestion` turns, one per
statement slot:

- Slot 1: confirm draft / amend / reject (write my own) / skip.
- Slot 2: same options.
- Slot 3: same options.

After each slot resolves, append the captured statement to the
ICP's H3 under **Outcome statements:**. Skip on any slot writes
`_not provided._` for that slot rather than omitting it — the
canonical shape always has three bullet entries per ICP.

After all three slots land, mark the ICP in `icps_complete` in
front-matter. Save.

#### Loop control

After each ICP's pitch + outcome statements are captured, advance
to the next ICP automatically. The loop is driven by the
`intake/icp.md` ICP count; no "add another?" prompt — the
ICPs are fixed upstream.

When all ICPs are messaged, mark `per_icp_messages` in
`stages_complete`, set `status: complete`, save, exit.

## Artifact shape: `intake/platinum-message.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "platinum-message"
sub_skill: "intake-platinum-message"
sub_skill_version: "0.1.0"
status: "in_progress"
captured_at: "<ISO 8601 of last write>"
icp_count: <N>
icps_complete: []          # subset of ICP slugs from intake/icp.md
desired_tone: []           # subset of: corporate, professional, friendly, direct, bold, conversational
stages_complete: []        # subset of: anchor, tone, per_icp_messages
# ---
~~~

Body shape:

~~~markdown
# Platinum Message intake

## Company anchor

<!-- source: 00_intake/intake/competitors.md (Company anchor) -->

- Company: <name>
- URL: <https://...>

## ICPs in scope

<!-- source: 00_intake/intake/icp.md -->

- <ICP 1 Name>
- <ICP 2 Name>

## Desired tone

<!-- source: client interview, <ISO date> -->

- **Tone descriptors:** corporate, direct
- **Notes:** <free-text from the follow-up turn, or "_none_">

## Platinum messages

### <ICP 1 Name> — Platinum message

<!-- source: client interview, <ISO date> (stop-slop polished, principles validated) -->

> <canonical Platinum Message paragraph — the client's pick from
> the three drafts, or their from-scratch text>

**Outcome statements:**

- <statement 1>
- <statement 2>
- <statement 3>

**Principles validation:**

| Appeal | Exclusivity | Clarity | Credibility |
|--------|-------------|---------|-------------|
| ✓      | ✓           | ✓       | ✓           |

### <ICP 2 Name> — Platinum message

...
~~~

`compile-profile` lifts the `### <ICP> — Platinum message` H3
blocks (blockquote + Outcome statements bullets) into the
**Messaging** section of `client-profile.md`. The principles
validation table stays in the intake artifact for audit; it does
NOT get copied into the canonical profile.

## Steps

1. **Read preconditions.** Confirm all three upstream artifacts
   (`icp.md`, `transformations.md`, `competitors.md`) are
   `status: complete`. Surface and exit cleanly when any is
   missing or in_progress. Confirm `intake-interviewer-voice.md`
   is on disk.

2. **Initialize or resume `intake/platinum-message.md`.** Parse
   front-matter when present. Compute the resume point from
   `stages_complete` and `icps_complete`:

   - `stages_complete: []` → Stage 1 question 1.
   - `[anchor]` → Stage 2 question 1.
   - `[anchor, tone]` → Stage 3 for the first ICP not in
     `icps_complete`.
   - `[anchor, tone, per_icp_messages]` → done; ask the
     redo/refresh/exit question if status is somehow `complete`,
     otherwise exit cleanly.

3. **Run Stage 1.** Anchor turns. Append to artifact. Add
   `anchor` to `stages_complete`. Save.

4. **Run Stage 2.** Tone capture. Append. Add `tone` to
   `stages_complete`. Save.

5. **Run Stage 3 per-ICP loop.** For each ICP in
   `intake/icp.md` Phase A, in order:
   - Run 3a pitch flow. Capture the chosen pitch + principles
     validation table. Save.
   - Run 3b outcome statements flow. Capture three statements
     (or `_not provided._` per slot when the client skips).
     Save.
   - Append ICP slug to `icps_complete`. Save.

6. **Mark complete.** When all ICPs are in `icps_complete`, set
   `status: complete` and `per_icp_messages` in
   `stages_complete`. Save.

7. **Return.** Print the quality report:

   - Number of ICPs messaged.
   - How many pitches required "Iterate" or "Write from
     scratch" (signals which ICPs were hardest to message).
   - Whether any outcome-statement slots were skipped (`_not
     provided._` count).
   - Next-step prompt: "Run `intake-offer` next, or run
     `compile-profile` only when every intake sub-skill is
     complete."

## Drop handling mid-flow

Per the shared voice reference. Routing for this skill:

- **An existing pitch or tagline the client has been using**:
  hold in memory; surface as a fourth pre-draft variant in the
  current ICP's Stage 3a presentation, or as the seed for
  "Write from scratch" if the client picks that option mid-
  flow.
- **A competitor's tagline** the client wants to position
  against: append to the current ICP's section as a `<!--
  source: client interview -->` note; the skill uses it as a
  contrast point in the next pitch revision.
- **A piece of voice or brand language** (a website headline,
  a tone-of-voice doc): drop in `01_knowledge_base/raw/`,
  log for `kb-ingest`'s next pass. The current run uses the
  language directly; future runs read it from processed.

Do not advance the question cursor when handling a drop.

## Stop-slop integration

`stop-slop` runs on:

- Each of the three pitch drafts before presenting them in Stage
  3a (one polish pass after the internal principle-revision
  pass, before the client sees the options).
- Each of the three outcome-statement drafts before presenting
  them in Stage 3b.
- The client's "Write from scratch" text **only when the client
  explicitly opts in** to a polish pass. Otherwise the client's
  text ships verbatim.

Bullet structure (the tone descriptors list, the principles
validation table) is exempt — short structured content the
client confirms at the field level.

## Pre-draft sources

Reading order, in priority:

1. `00_intake/intake/icp.md` — per-ICP pain points, transformation,
   reframed positioning.
2. `00_intake/intake/transformations.md` — top transformations as
   the Credibility anchor.
3. `00_intake/intake/competitors.md` — Brand Edge + Differentiation
   Summary as the Exclusivity anchor.
4. Stage 2 tone — the register for every draft.
5. `00_intake/client-profile.md` Messaging section (when a prior
   profile exists — re-runs).
6. `01_knowledge_base/processed/**` — any voice or brand language
   already cleaned.

Confidence labels are not needed here — the drafts are syntheses,
not extractions. The four-principles validation is the quality
check.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| Any upstream artifact missing or in_progress     | Refuse to run. Tell the client which step still needs to finish.               |
| Iterate-rounds cap reached on a pitch            | Fall back to "Write from scratch" with the client's last-stated language.      |
|                                                  | Run the principles check as a follow-up rather than rewriting.                 |
| Client picks pitch that fails a principle        | Don't override. Capture the pitch as the client confirmed it. Note the failed   |
|                                                  | principle in the validation table (✗ in the matching cell). Downstream skills  |
|                                                  | can read the table and flag if needed.                                          |
| Client says "stop" mid-ICP                       | Save current state, leave `status: in_progress`, exit cleanly. Resume picks up |
|                                                  | mid-ICP at the next question slot.                                              |
| Client skips all three outcome-statement slots   | Each slot renders `_not provided._` in the artifact. The Messaging section of  |
|   for an ICP                                     | client-profile.md will show the blockquote with three "not provided" bullets;  |
|                                                  | compile-profile does not block on this — it's a signal, not an error.          |
| Stop-slop catches violations after client-write  | Only polish when the client opts in. The client's text is their voice.         |
|   from-scratch                                   |                                                                                |
| Tone descriptors conflict (corporate + bold)     | Capture both. The combination is the client's intent; the drafts honor both    |
|                                                  | (an earned, confident register without consultancy hedge).                     |

## What NOT to do

- Do not batch questions. Three pitch slots are ONE
  `AskUserQuestion` because they're the same decision (pick or
  iterate). Three outcome-statement slots are THREE
  `AskUserQuestion` turns because each is a separate decision.
- Do not present a pitch that fails the principle check without
  attempting an internal revision first. Show the client only
  drafts the skill itself rates as principle-passing — unless
  the client explicitly asked to see a failing draft.
- Do not auto-override a client's chosen pitch on a principles
  failure. The client's pick stands; the validation table notes
  the failure.
- Do not partition tone per ICP. Tone is workspace-wide. When
  the client genuinely wants tone-per-ICP, they redo the skill
  per ICP via `run-intake`'s "redo step N" path.
- Do not preserve corporate-consulting register from the
  original Platinum Message GPT. The unified intake voice
  applies — including during pitch presentation.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill only owns `intake/platinum-message.md`.
