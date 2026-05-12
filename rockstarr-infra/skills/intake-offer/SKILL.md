---
name: intake-offer
description: "This skill should be used when the user asks to \"run the offer step\", \"build the offer\", \"capture our services\", \"map what we sell\", \"run the offer builder\", or when run-intake dispatches to the offer step of the intake flow. Reads the upstream artifacts (icp.md, platinum-message.md, competitors.md, transformations.md) and walks the client through capturing one or more offers in the canonical eight-field shape (Name, Who it's for, Problem, Outcome, Process, Experience, Edge (optional), Why it works, Category-of-one positioning). Runs a Category-of-One check after each offer to confirm a competitor couldn't describe themselves the same way. One question at a time, in the unified intake voice. Checkpoints every answer to /00_intake/intake/offer.md. Feeds the Offers / services section of client-profile.md."
---

# intake-offer

Capture the offers the client sells. Each offer becomes one
`### <Offer Name>` H3 in the canonical profile, with a fixed
eight-field bullet structure underneath. The skill walks an offer-
at-a-time capture loop with field-level pre-drafts pulled from the
four upstream artifacts.

Three stages:

- **Stage 1 — Anchor.** Read company + ICP list from upstream and
  confirm.
- **Stage 2 — Per-offer loop.** For each offer the client wants to
  capture, run the eight-field capture flow, then the Category-of-
  One check.
- **Stage 3 — Wrap.** Mark complete when the client says "no more
  offers."

This is one of six intake sub-skills inside `rockstarr-infra`'s
v0.9.x intake flow. Writes `/rockstarr-ai/00_intake/intake/offer.md`.
`compile-profile` reads that file to assemble the **Offers /
services** section of `client-profile.md`.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule comes from there. The spirit
of the original Offer Builder GPT is preserved (the eight-field
shape, the Category-of-One Check, the "no pricing, no buzzwords,
active verb-led outcomes" style rules), but the unified intake
voice applies from question one.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` after `intake-platinum-message`
  completes.
- Standalone, when an operator re-runs the offer step via
  `run-intake`'s "redo step N" path.
- Standalone, when an operator says "redo the offers" or "we
  launched a new offer and need to capture it."

## Preconditions

- `/rockstarr-ai/00_intake/intake/icp.md` with `status: complete`.
  The "Who it's for" field maps offers to ICPs by name.
- `/rockstarr-ai/00_intake/intake/transformations.md` with
  `status: complete`. The top transformations supply the Outcome
  field's evidence anchors.
- `/rockstarr-ai/00_intake/intake/competitors.md` with
  `status: complete`. Brand Edge feeds the Edge field; the
  Differentiation Summary seeds Category-of-one positioning.
- `/rockstarr-ai/00_intake/intake/platinum-message.md` with
  `status: complete`. Per-ICP messages give the language for Why
  it works.
- `/rockstarr-ai/00_intake/intake/` writable.
- `intake-interviewer-voice.md` on disk inside the plugin.

When any upstream artifact isn't yet `complete`, refuse to run with
a clear message naming the missing step. Offer Builder is the
synthesis step — every upstream input matters.

When `intake/offer.md` exists with `status: in_progress`, this is a
resume. Read the file, check `offers_complete`, pick up at the next
unfinished offer (or at the "add another?" gate if all captured
offers are complete).

When the file exists with `status: complete`, ask: "Offers already
captured. Redo from scratch, redo one offer, add another offer, or
exit?" Default action is exit.

## The eight fields

Every offer is captured in the same fixed shape. `compile-profile`
depends on this exactly — drift means downstream bots reading
`client-profile.md`'s Offers section get inconsistent input.

| Field                              | Type        | Length      | Stop-slop? | Notes                                                                                          |
|------------------------------------|-------------|-------------|------------|------------------------------------------------------------------------------------------------|
| Name                               | string      | 1-6 words   | no         | Title-case. Becomes the H3.                                                                    |
| Who it's for                       | string      | 1 line      | no         | Multi-select from `intake/icp.md` ICP list. One or more ICP names.                             |
| Problem                            | string      | 1 sentence  | no         | One urgent problem this offer solves. Specific. Named pain.                                    |
| Outcome                            | string      | 1 sentence  | no         | Active, verb-led, results-focused. Not activities. "Increased revenue 27% in 4 months," not   |
|                                    |             |             |            | "Provided consulting."                                                                          |
| Process                            | string      | 1-2 lines   | no         | High-level phases or steps of working together. No pricing. No internal SOPs — buyer-facing.   |
| Experience                         | string      | 1-2 lines   | no         | Texture and feel. What's different about how this offer is delivered.                          |
| Edge (optional)                    | string      | 1 line      | no         | What makes this offer accessible / sharp / specific that competitors can't match. Skip is OK. |
| Why it works                       | paragraph   | 2-4 sentences | yes      | The mechanism. Why this offer produces its Outcome.                                            |
| Category-of-one positioning        | string      | 1 sentence  | yes        | The framing line. One sentence stating the offer as a category of one.                         |

Bullet structured fields (Who it's for, Problem, Outcome, Process,
Experience, Edge) ship without stop-slop polish — they're short,
field-level confirmed, and the language is mostly the client's
direct answer.

Prose fields (Why it works, Category-of-one positioning) run stop-
slop before presenting to the client for confirm / amend / reject.

## The Category-of-One check

After all eight fields land for an offer, the skill runs a self-
test on the captured offer as a whole. One question, asked
internally before showing the result to the client:

> Could a competitor described in `intake/competitors.md` truthfully
> describe themselves the same way using these eight fields?

When the answer is **no**, the offer passes the check. The
validation table renders ✓ across the row and the offer is
captured.

When the answer is **yes** for any competitor, the check fails.
The skill identifies which competitor matches, which fields read
generic, and suggests one sharpening move:

- **Outcome reads generic** → sharpen with a number, timeframe, or
  named methodology from `intake/transformations.md`.
- **Process reads generic** → name a specific phase, a duration,
  or a sequencing decision that competitors don't have.
- **Experience reads generic** → name the texture difference (the
  hands-on element, the done-for-you-ness, the speed, the
  intimacy).
- **Category-of-one positioning reads generic** → swap the framing
  for the language the Differentiation Summary uses in
  `intake/competitors.md`.

Present the suggested sharpening as one `AskUserQuestion` with
three options: apply the suggestion / write my own sharpening /
accept the failure (ship the offer with the ✗ visible in the
validation table). Cap sharpening rounds at two; on the third
attempt, the offer ships flagged.

The validation table renders inline under the offer's H3:

~~~markdown
**Category of One check:**

| Could a competitor describe themselves this way? | Result |
|--------------------------------------------------|--------|
| Reviewed against intake/competitors.md           | ✓ or ✗ |
~~~

`compile-profile` does NOT lift this validation table into the
canonical profile. It lives in the intake artifact for audit.

## The three stages

### Stage 1 — Anchor

Two `AskUserQuestion` turns:

1. **Confirm company.** Read company name + URL from
   `intake/competitors.md` Company anchor. Same three options as
   every anchor stage: confirm / edit / no URL yet.

2. **Confirm ICP list.** Read ICP names from `intake/icp.md`
   Phase A. One-sentence recap of each. Same three options.

Append both to the artifact under `## Company anchor` and
`## ICPs available`. Mark `anchor` in `stages_complete`. Save.

### Stage 2 — Per-offer loop

One offer at a time. Pre-suggest the natural number of offers
from upstream context — usually one offer per ICP, but
flexible. Many clients have one core offer that serves multiple
ICPs (e.g., TRANSEARCH); some have one offer per side of the
marketplace (e.g., Beyond Basic Needs: Chemo Care Kit + Corporate
Build Day). Don't enforce a per-ICP mapping.

#### 2a. Open the offer

One `AskUserQuestion`:

> "Capture an offer. What's it called?"

Options:
- **Free text** — the client names the offer.
- **Suggest from upstream** — pre-draft 2 to 4 candidate offer
  names by reading the per-ICP Platinum Messages and pulling out
  recurring offer-shaped phrases. Present as picks.
- **Done — no more offers** — exits the loop.

When the client picks "Done — no more offers" before any offer is
captured, ask once: "No offers at all? compile-profile will mark
the section _not provided._ Confirm or go back?" Accept whichever.

#### 2b. Capture the eight fields

Walk the eight fields in order, one `AskUserQuestion` per field.
Each field uses the pre-draft → confirm / amend / reject / skip
pattern from the shared voice reference. Skip on a required field
captures `_not provided._` for that slot but proceeds — only Edge
is officially optional.

Per-field pre-draft sources:

| Field                         | Pre-draft from                                                                                       |
|-------------------------------|------------------------------------------------------------------------------------------------------|
| Who it's for                  | Multi-select widget over `intake/icp.md` ICP names. Pre-select based on the platinum-message that    |
|                               | shares the most language with the offer name.                                                        |
| Problem                       | Pain points from the selected ICP(s) in `intake/icp.md` Phase A.                                     |
| Outcome                       | Transformation sought from the same ICP(s); narrow to ones echoed in the top transformations.        |
| Process                       | Free draft from the platinum-message Why it works language; no upstream evidence usually.            |
| Experience                    | Free draft from the platinum-message Outcome statements language.                                    |
| Edge                          | Brand Edge bullets in `intake/competitors.md`; pick the ones relevant to this offer's audience.      |
| Why it works                  | Synthesis of Outcome + Process + the transformations that prove the Outcome.                         |
| Category-of-one positioning   | Synthesis of Differentiation Summary + the offer's unique combination of Outcome + Experience.        |

Confidence labels (HIGH / MEDIUM / LOW) accompany every pre-draft.

Stop-slop runs on the Why it works paragraph and the Category-of-
one positioning sentence between the internal draft and the
client-facing presentation.

After each field's resolution, append it to the offer's H3 block
in the artifact and save. Update `captured_at` and the offer's
in-flight slug in front-matter.

#### 2c. Category-of-One check

After all eight fields are captured, run the internal check
described above. When the check passes, render the validation
table with ✓. When it fails, run the sharpening loop (max two
rounds), then render the result (✓ or ✗).

Append the validation table under the offer's H3. Add the offer's
slug to `offers_complete` in front-matter. Save.

#### 2d. Loop control

After each offer completes, ask:

> "Add another offer? (yes / no)"

When yes, loop back to 2a. When no, advance to Stage 3.

Soft warnings:
- At the **fifth** offer captured in one run, surface: "Five
  offers is a lot for one workspace. Consider whether some are
  variants of a parent offer worth merging."
- The warning is informational — accept "yes" anyway.

### Stage 3 — Wrap

Mark `per_offer_loop` in `stages_complete`. Set `status:
complete`. Save.

Return to `run-intake` or to the user with the standard quality
report:

- `offer_count` and the list of captured offer names.
- For each offer, whether the Category-of-One check passed (✓) or
  shipped flagged (✗).
- Any offers where Edge was skipped (signal: the offer may read
  generic to non-ICP buyers).
- Next-step prompt: "Every intake sub-skill is complete. Run
  `compile-profile` now to assemble `client-profile.md`."

## Artifact shape: `intake/offer.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "offer"
sub_skill: "intake-offer"
sub_skill_version: "0.1.0"
status: "in_progress"
captured_at: "<ISO 8601 of last write>"
offer_count: 0
offers_complete: []        # list of offer slugs
stages_complete: []        # subset of: anchor, per_offer_loop
# ---
~~~

Body shape:

~~~markdown
# Offer intake

## Company anchor

<!-- source: 00_intake/intake/competitors.md (Company anchor) -->

- Company: <name>
- URL: <https://...>

## ICPs available

<!-- source: 00_intake/intake/icp.md -->

- <ICP 1 Name>
- <ICP 2 Name>

## Offers

### <Offer Name 1>

<!-- source: client interview, <ISO date> -->

- **Who it's for:** <ICP names>
- **Problem:** <one sentence>
- **Outcome:** <one sentence, active and verb-led>
- **Process:** <one to two lines>
- **Experience:** <one to two lines>
- **Edge:** <one line, or omitted if skipped>
- **Why it works:** <stop-slop polished paragraph, 2-4 sentences>
- **Category-of-one positioning:** <stop-slop polished sentence>

**Category of One check:**

| Could a competitor describe themselves this way? | Result |
|--------------------------------------------------|--------|
| Reviewed against intake/competitors.md           | ✓      |

### <Offer Name 2>

...
~~~

`compile-profile` lifts each `### <Offer Name>` H3 block with the
eight-field bullet structure into the **Offers / services**
section of `client-profile.md`. The validation table stays in the
intake artifact for audit and is not copied.

## Steps

1. **Read preconditions.** Confirm `icp.md`,
   `transformations.md`, `competitors.md`,
   `platinum-message.md` are all `status: complete`. Surface and
   exit cleanly when any is missing. Confirm
   `intake-interviewer-voice.md` is on disk.

2. **Initialize or resume `intake/offer.md`.** Parse front-matter
   when present. Compute the resume point from `stages_complete`
   and `offers_complete`:

   - `stages_complete: []` → Stage 1 question 1.
   - `[anchor]` → Stage 2a for offer #1.
   - For an in-flight offer (slug not in `offers_complete` but
     fields partially captured), read the body to find the next
     unanswered field. Resume at that field.
   - For a complete loop (`per_offer_loop` in `stages_complete`)
     with `status: complete`, ask the redo/refresh/add/exit
     question.

3. **Run Stage 1.** Anchor turns. Append. Add `anchor` to
   `stages_complete`. Save.

4. **Run Stage 2 per-offer loop.** For each offer:
   - 2a opening turn → captured offer name → save.
   - 2b eight-field walk → field-by-field capture → save after
     each.
   - 2c Category-of-One check → internal validation → sharpening
     loop (max two rounds) → validation table → save.
   - 2d "Add another?" turn.

5. **Run Stage 3 wrap.** Mark `per_offer_loop` in
   `stages_complete`. Set `status: complete`. Save.

6. **Return.** Print the quality report.

## Drop handling mid-flow

Per the shared voice reference. Routing for this skill:

- **A brochure or service page** the client wants captured as an
  offer: drop in `01_knowledge_base/raw/`. Log it for `kb-ingest`.
  In the current run, use the dropped file's language to pre-
  draft the eight fields for the next offer the client opens (or
  to enrich the in-flight offer's pre-drafts if it matches).
- **A website "services" page URL**: fetch when web tools are
  available; otherwise ask the client to paste the relevant copy.
  Use as pre-draft input for Problem / Outcome / Process /
  Experience.
- **A pricing card or proposal**: drop in `01_knowledge_base/raw/`.
  Pricing does NOT enter the captured offer — `intake-offer`
  intentionally has no pricing field. Reference the dropped file
  only when its non-pricing prose informs Why it works or
  Category-of-one positioning.

Do not advance the question cursor when handling a drop.

## Stop-slop integration

`stop-slop` runs on:

- The **Why it works** paragraph before the client sees it
  (after internal field synthesis, before the confirm /
  amend / reject turn).
- The **Category-of-one positioning** sentence on the same
  timing.
- Any client-revised version of either field when the client
  picks amend or reject.

Structured fields and bullet content are exempt — short, field-
level confirmed, mostly client-direct language.

## Pre-draft sources

Reading order, in priority:

1. `00_intake/intake/icp.md` — for Who it's for, Problem,
   Outcome.
2. `00_intake/intake/transformations.md` — Outcome anchors, Why
   it works evidence.
3. `00_intake/intake/competitors.md` — Edge (Brand Edge bullets),
   Category-of-one positioning (Differentiation Summary).
4. `00_intake/intake/platinum-message.md` — Process, Experience,
   Why it works language.
5. `00_intake/client-profile.md` Offers section (when a prior
   profile exists — re-runs).
6. `01_knowledge_base/processed/**` — any service pages or
   offer documents already cleaned.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| Any upstream artifact missing or in_progress     | Refuse to run. Tell the client which step still needs to finish.               |
| Client lands at zero offers                      | After Stage 2a's "Done" path, confirm once. If the client confirms zero,       |
|                                                  | mark `status: complete` with `offer_count: 0`. compile-profile renders         |
|                                                  | `_not provided._` for the Offers / services section.                           |
| Edge skipped on every captured offer             | Permitted. Edge is officially optional. The quality report flags it as a       |
|                                                  | signal but doesn't block.                                                      |
| Category-of-One check fails after two sharpening | Capture the offer with ✗ in the validation table. The offer ships;             |
|   rounds                                          | downstream skills can read the table. Don't block on it.                       |
| Client says "stop" mid-field                     | Save current state, leave `status: in_progress`. Resume picks up at the next   |
|                                                  | unanswered field of the in-flight offer.                                       |
| Client wants to add an offer that doesn't map    | Allow it. Capture the offer with Who it's for: `_not provided._`               |
|   to any captured ICP                             | This signals the ICPs may be incomplete; downstream review surfaces it.        |
| Captured Outcome reads as an activity, not a     | The pre-draft synthesizer rewrites activity-shaped outcomes as result-shaped   |
|   result                                          | before presenting. When the client confirms an activity-shaped Outcome anyway, |
|                                                  | capture as-stated; the validation table or Category-of-One check may catch it. |
| Client adds pricing information mid-field        | Capture the field as-stated MINUS the pricing. Note the pricing detail in      |
|                                                  | the artifact under a `_pricing notes:_` italic line — for the client's audit  |
|                                                  | trail — but it does not enter any of the eight canonical fields.               |

## What NOT to do

- Do not batch questions. One `AskUserQuestion` per field.
- Do not capture pricing inside any of the eight fields. Pricing
  is workspace-mutable and lives elsewhere (sales collateral,
  the client's CRM). compile-profile's Offers section is for
  buyer-facing offer description, not sales mechanics.
- Do not normalize "Audience" → "Who it's for" silently if a
  client writes "Audience" in chat. Capture the client's
  phrasing for the value, but the field name in the artifact
  always renders as **Who it's for** for downstream consistency.
- Do not skip the Category-of-One check. Even on offers that look
  obviously distinct, the check is a useful audit signal.
- Do not auto-override a client's ✗ Category-of-One result. The
  capture stands; the table notes the failure for downstream
  awareness.
- Do not partition offers by ICP one-to-one unless the client
  explicitly does so. Many offers serve multiple ICPs; the
  `Who it's for` field captures the mapping.
- Do not preserve corporate-consulting register from the
  original Offer Builder GPT. The unified intake voice applies.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill only owns `intake/offer.md`.
