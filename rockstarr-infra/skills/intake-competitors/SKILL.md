---
name: intake-competitors
description: "This skill should be used when the user asks to \"run the competitors step\", \"do the competition crusher exercise\", \"map the competitive landscape\", or when run-intake dispatches to the competitors step. Walks the client through a seven-stage process: anchor (company + URL), competitor selection (1-3, client-named or suggested), per-competitor research via web fetch (graceful fallback to paste-in), competitive grid synthesis, positioning artifacts (Brand Edge / Differentiation / Messaging opportunities / Risks / Quick wins), validation pass on Assumed items, Key takeaways. One question at a time in the unified intake voice. Checkpoints to /00_intake/intake/competitors.md. Feeds both Competitors AND Positioning sections of client-profile.md."
---

# intake-competitors

Capture the competitive landscape and the positioning artifacts it
reveals. This is the most layered intake sub-skill — it does external
research, produces a structured grid, and emits five subsections that
`compile-profile` lifts into the Positioning section of
`client-profile.md`.

Seven stages:

- **Stage 1 — Anchor.** Pull company name + URL from prior context.
  Pull the ICPs from `intake/icp.md`. Confirm both.
- **Stage 2 — Competitor selection.** Either accept 1 to 3
  client-named competitors, or draft 3 to 5 candidates from the ICP
  + KB material and let the client pick.
- **Stage 3 — Per-competitor research.** For each competitor: web-
  fetch when tools are available (homepage, About, Solutions /
  Products, Pricing, Case Studies), extract value proposition +
  offering + differentiators with confidence labels. When fetch is
  blocked or empty, ask the client to paste relevant copy.
- **Stage 4 — Competitive grid.** Deterministic rendering of the
  per-competitor data as a markdown table.
- **Stage 5 — Positioning artifacts.** Brand Edge, Differentiation
  Summary (first-person, copy-paste-ready), Messaging Opportunities,
  Risks to Watch, Quick Wins.
- **Stage 6 — Validation pass.** Surface every line marked `Assumed:`
  during research. Client confirms or corrects.
- **Stage 7 — Key takeaways.** Closing synthesis of what the grid
  reveals about the firm's position.

This is one of six intake sub-skills inside `rockstarr-infra`'s
v0.9.x intake flow. Writes `/rockstarr-ai/00_intake/intake/competitors.md`.
`compile-profile` reads that file to assemble the **Competitors**
section AND five subsections of the **Positioning** section in
`client-profile.md`.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule comes from there. There is no
GPT prompt to channel — the spirit of the original Competition
Crusher GPT is preserved (what it asks, what it produces, what
quality it enforces), but the unified intake voice applies from
question one.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` after `intake-transformations`
  completes (the plan's natural order: ICP → Transformations →
  Competitors → Platinum Message → Offer → Background).
- Standalone, when an operator re-runs the competitors step via
  `run-intake`'s "redo step N" path.
- Standalone, when an operator says "redo the competitive grid"
  or "the landscape has changed."

## Preconditions

- `/rockstarr-ai/00_intake/intake/icp.md` exists with
  `status: complete`. ICP is the lens for Stage 5's positioning
  artifacts. Refuse to run when ICPs are missing.
- `/rockstarr-ai/00_intake/intake/transformations.md` exists with
  `status: complete`. The Brand Edge subsection draws on the
  pressure-tested top transformations. Refuse to run when
  transformations are missing.
- `/rockstarr-ai/00_intake/intake/` is writable.
- `intake-interviewer-voice.md` is on disk inside the plugin.

When `intake/competitors.md` already exists with
`status: in_progress`, this is a resume. Read the file, count which
stages are complete, and pick up at the next unanswered question.

When the file exists with `status: complete`, ask the client:
"Competitive grid is already captured. Redo from scratch, refresh a
specific competitor, or exit?" Default action is exit.

## Chat narration discipline

Two shared voice references govern what this skill says:

- **`skills/_shared/references/intake-interviewer-voice.md`** —
  the AskUserQuestion turns themselves.
- **`skills/_shared/references/client-facing-output-voice.md`** —
  everything between the questions: stage-transition lines,
  capture acknowledgments, the post-completion summary.

Apply both. Specifically for this sub-skill:

- **No "Stage 1 / Stage 2 / ..." labels in chat transitions.**
  The seven stages (anchor, competitor selection, per-competitor
  research, grid synthesis, positioning artifacts, validation,
  takeaways) are bot-side structure. The client experiences a
  natural flow: "Who are we comparing against? Let me pull what
  I can find on them, then we'll work out where you stand."
- **No artifact paths in capture acknowledgments.** The fact that
  answers land in `00_intake/intake/competitors.md` is invisible
  to the client.
- **Web-fetch source attribution in narration is fine.** "Pulled
  their About page and the latest case study from their site" is
  useful context. URLs go in the artifact, not on every chat line.
- **The Assumed → Validation pass surfaces in chat as one
  natural question.** "I marked a few things as assumed earlier
  — want me to walk through them so we can confirm?" Not "Stage
  6 — Validation pass on Assumed items."

## Web fetch availability

Some intake sessions run with browser / web-fetch tools available;
some run without. The skill detects availability at Stage 3 entry
and branches.

| Tools available                     | Behavior                                                              |
|-------------------------------------|-----------------------------------------------------------------------|
| `WebSearch` + `WebFetch` (any form) | Fetch competitor URLs; extract value prop / offering / differentiators directly. Mark un-confirmed extractions with `Assumed:` prefix for Stage 6 validation. |
| `WebSearch` only                    | Use search to find competitor URLs when the client didn't supply them. Per-competitor research falls back to paste-in. |
| Neither                             | Skip web altogether. Ask the client to paste competitor language (one to three sentences per field) for each competitor. |

Tool availability is checked by attempting a low-cost call (a
search for the company name) at Stage 2's end. The result is
stored in `intake/competitors.md` front-matter as
`web_fetch_available: true | false` so resume sessions don't
re-detect.

When neither tool is available, the skill still produces a
complete artifact — the grid is just sparser and every cell is
client-confirmed rather than pre-drafted.

## The seven stages

### Stage 1 — Anchor

Two `AskUserQuestion` turns:

1. **Confirm company + URL.** Read from
   `00_intake/client-profile.md` if a prior profile exists, or
   from `intake/background.md` once that sub-skill ships, or ask
   the client directly. Show what was found and ask:

   - **Confirm** — proceed using the captured name + URL.
   - **Edit** — capture corrections.
   - **No URL yet** — accept; proceed without a URL. Web fetches
     for the competitor pages still run when applicable, but the
     company-side context comes from KB processed material only.

2. **Confirm ICPs in scope.** Read the ICP names from
   `intake/icp.md` Phase A and present a one-sentence recap of
   each. Same three options as above. ICPs are the lens for
   Stage 5's positioning; they don't change during this skill,
   only get re-confirmed.

No competitors get captured here. Stage 1 is reorientation.

Append both confirmations to the artifact under `## Company
anchor` and `## ICPs in scope`. Mark `anchor` in `stages_complete`.

### Stage 2 — Competitor selection

One `AskUserQuestion` turn with three branches:

1. **Client names competitors.** "Who are 1 to 3 competitors you'd
   like to map?" Free text, accepted as a comma-separated list or
   one per line. For each, capture name + URL when supplied.

2. **Ask for suggestions.** Pre-draft 3 to 5 candidate competitors
   from:
   - `intake/icp.md` Phase A "Sources of information" field.
   - `intake/icp.md` Phase A "Pain points" (which competitors
     also address these pains?).
   - `01_knowledge_base/processed/**` for any positioning research
     the client uploaded.
   - When web tools are available, a search like
     `[company] competitors` or `<ICP-specific keyword>
     companies` to surface obvious candidates.

   Present the candidates with one-sentence "Why this one"
   labels and confidence labels (HIGH / MEDIUM / LOW). Ask the
   client to pick 1 to 3.

3. **Mix.** Client names some, asks for suggestions on the
   remainder. Run branch 1 for the named, then branch 2 to fill
   up to 3.

The cap of 3 is enforced. The original Competition Crusher GPT
landed on 3 as the sweet spot — enough to triangulate a position,
not so many that the grid becomes mush. Accept fewer than 3 when
the client genuinely sees only 1 or 2 real competitors; flag the
gap in front-matter as `competitor_count: [N]`.

After the picks land, run the tool-availability check (described
above) and write `web_fetch_available: [bool]` to front-matter.

Append the competitor list to the artifact and mark `selection`
in `stages_complete`.

### Stage 3 — Per-competitor research

Loop, one competitor at a time. For each:

**3a. Research pre-read (when web tools available).**

Fetch in this order, accepting partial success:
- Homepage / landing page.
- About / About Us page.
- Solutions / Products / Services page.
- Pricing page (when public).
- Case Studies / Customer Stories page.

For each fetched page, extract:
- **Value proposition** — the headline or sub-headline the page
  leads with.
- **Offering** — the product / service categories the page lists.
- **Differentiators** — anything the page calls out as "why
  choose us," "what makes us different," or comparable phrasing.

Draft a per-field proposal with a confidence label:
- HIGH — the language was lifted directly from a clear page
  section.
- MEDIUM — the language is the skill's read of the page, not a
  direct quote.
- LOW — the page was thin or evasive; the proposal is a best
  guess from sparse signal.

Mark every LOW field and every non-HIGH field with the prefix
`Assumed:` so Stage 6 validates it.

**3b. Per-field confirmation.**

For each of the three fields (value prop / offering /
differentiators), one `AskUserQuestion`:

- **Confirm** — capture the pre-draft verbatim, drop the
  `Assumed:` prefix.
- **Amend** — capture the client's revision, drop the prefix.
- **Reject** — replace the pre-draft with the client's answer
  from scratch.
- **Skip** — keep the pre-draft as-is with the `Assumed:`
  prefix; Stage 6 will surface it for explicit validation.

When web tools are unavailable (or the fetches returned
nothing), skip 3a and ask the three fields cold:

- "How does [competitor] describe what they do? One sentence."
- "What products or services do they offer?"
- "What do they call out as their differentiator?"

The client's answer is captured directly (no `Assumed:` prefix,
since the client is the source).

**3c. Nudge prompts.**

If a captured field is vague ("they do consulting," "they have
services"), ask one sharpening question: "What kind of
consulting?" / "Which services specifically?" Capture the
sharper version. Same one-push discipline as
`intake-transformations` Stage 3 — don't push twice.

After each competitor's three fields land, append the competitor
block to the artifact under `## Competitors` as an H3, and save.
Update `captured_at` and the `competitors_researched` count in
front-matter.

When all competitors have been researched, mark `research` in
`stages_complete`.

### Stage 4 — Competitive grid synthesis

Deterministic. No `AskUserQuestion` turns. The skill assembles
the per-competitor data into a markdown table under `##
Competitive grid`:

~~~markdown
## Competitive grid

| Competitor & URL                 | Value proposition | The offering | Differentiators |
|----------------------------------|-------------------|--------------|-----------------|
| **[Name]** (<https://...>)       | ...               | ...          | ...             |
~~~

`Assumed:` prefixes remain visible in the table; Stage 6 will
resolve them.

Append the table to the artifact and mark `grid` in
`stages_complete`. Save.

### Stage 5 — Positioning artifacts

Five subsections, captured in order. Each is one `AskUserQuestion`
turn (or two for Differentiation summary because of the polish
pass). Pre-drafts pull from the ICP, the top transformations, and
the competitive grid.

**5a. Brand Edge.**

The bullet list of attributes that distinguish the client across
the competitive field. Pre-draft from:
- The differentiators column of the grid (what the competitors
  claim — the gaps reveal Brand Edge).
- The top transformations from `intake/transformations.md`
  (proven outcomes are Brand Edge).
- The Phase B reframed positioning from `intake/icp.md` (what
  the firm actually delivers).

Target: 4 to 8 bullets. Each a single specific claim. Present
the draft via `AskUserQuestion` with confirm / amend / reject /
skip.

**5b. Differentiation summary.**

A single paragraph, first-person ("We stand out because we..."),
copy-paste-ready into a website hero block or a sales deck.
Synthesizes the Brand Edge bullets into one flowing claim. Runs
`stop-slop` before presenting to the client. Two `AskUserQuestion`
turns:

- Show the polished paragraph. Confirm / amend / reject.
- When the client picks amend or reject, the skill produces one
  revised paragraph (also stop-slop polished) and re-presents.
  Cap at three revision turns; if the client still isn't happy,
  capture their last-stated language verbatim and move on.

**5c. Messaging opportunities.**

Bullet list of taglines, framings, or hooks the competitive gap
reveals. These are short — three to seven words per bullet —
and ready for use as social headers, ad copy lines, or section
heads in long-form content. Target: 3 to 6 bullets.

Pre-draft from:
- The Differentiation summary (just produced).
- The Phase B "What clients think vs. what we do" reframed
  language from `intake/icp.md`.

Confirm / amend / reject / skip.

**5d. Risks to watch.**

Bullet list of competitive and market risks the gap analysis
surfaces. Target: 3 to 5 bullets. These are NOT operational risks
("we need to hire a salesperson") but positioning risks ("being
grouped into 'general consulting' instead of a distinct
category").

Pre-draft from the grid + ICP. Confirm / amend / reject / skip.

**5e. Quick wins.**

Bullet list of 30-day actions the client can take to lean into
the positioning the gap reveals. Target: 3 to 5 bullets. Each a
specific concrete action ("Add a split message above the fold:
'For Patients' + 'For Companies.'"), not a vague intention
("improve our website").

Pre-draft from the grid + Differentiation summary + Messaging
opportunities. Confirm / amend / reject / skip.

After 5e, mark `positioning` in `stages_complete`. Save.

### Stage 6 — Validation pass

Surface every line in the artifact still carrying the `Assumed:`
prefix. One `AskUserQuestion` per line, with three options:

- **Confirm** — drop the `Assumed:` prefix; capture as-is.
- **Correct** — capture the client's revision; drop the prefix.
- **Skip** — keep the `Assumed:` prefix; the line ships in the
  artifact (and into `compile-profile`'s output) flagged. This is
  fine — the prefix is a deliberate signal downstream skills can
  read.

When zero `Assumed:` lines exist (rare — most live web fetches
produce at least a few), skip Stage 6 entirely.

Mark `validation` in `stages_complete`. Save.

### Stage 7 — Key takeaways

One `AskUserQuestion` turn. Pre-draft a 3-to-5-bullet synthesis
of what the grid reveals about the firm's position. Each bullet a
single sharp observation about the competitive landscape, in the
first-person plural where natural ("We're the only player that
ties X to Y," "Competitors own the corporate relationship but
lack the human outcome").

Pre-draft from the full artifact. Present via `AskUserQuestion`
with confirm / amend / reject.

Append to the artifact under `## Key takeaways from the
competitive landscape`. Mark `takeaways` in `stages_complete`.
Save.

Set `status: complete`. Exit.

## Artifact shape: `intake/competitors.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "competitors"
sub_skill: "intake-competitors"
sub_skill_version: "0.1.0"
status: "in_progress"     # not_started | in_progress | complete
captured_at: "<ISO 8601 of last write>"
competitor_count: 0
competitors_researched: 0
web_fetch_available: null  # null until Stage 2 sets it; then true | false
stages_complete: []        # subset of: anchor, selection, research, grid, positioning, validation, takeaways
# ---
~~~

`stages_complete` plus the counts are the resume index.

Body shape:

~~~markdown
# Competitors intake

## Company anchor

<!-- source: 00_intake/client-profile.md or 00_intake/intake/background.md -->

- Company: [name]
- URL: <https://...>

## ICPs in scope

<!-- source: 00_intake/intake/icp.md -->

- <ICP Name 1>: <one-sentence recap>
- <ICP Name 2>: <one-sentence recap>

## Competitors

### <Competitor 1 Name>

<!-- source: <competitor URL fetched on YYYY-MM-DD> -->

- **URL:** <https://...>
- **Value proposition:** ...
- **Offering:** ...
- **Differentiators:** ...

### <Competitor 2 Name>

...

## Competitive grid

| Competitor & URL                 | Value proposition | The offering | Differentiators |
|----------------------------------|-------------------|--------------|-----------------|
| **[Name]** (<https://...>)       | ...               | ...          | ...             |

## Brand Edge

<!-- source: 00_intake/intake/icp.md (Phase B) + 00_intake/intake/transformations.md + this artifact's Competitive grid -->

- ...
- ...

## Differentiation summary

<!-- source: client interview, <ISO date> (stop-slop polished) -->

> <paragraph, first-person, copy-paste-ready>

## Messaging opportunities

- ...
- ...

## Risks to watch

- ...

## Quick wins

- ...

## Key takeaways from the competitive landscape

<!-- source: client interview, <ISO date> -->

- ...
- ...
~~~

`compile-profile` lifts the Competitive grid + Key takeaways into
the Competitors section, and Brand Edge / Differentiation summary
/ Messaging opportunities / Risks to watch / Quick wins into the
Positioning section.

## Steps

1. **Read preconditions.** Confirm `intake/icp.md` and
   `intake/transformations.md` exist with `status: complete`. If
   either is missing or in_progress, surface the issue and exit
   cleanly. Confirm `intake-interviewer-voice.md` is on disk.

2. **Initialize or resume `intake/competitors.md`.** If the file
   doesn't exist, write front-matter with empty arrays and
   `web_fetch_available: null`. If it does, parse front-matter
   and body. Compute the resume point from `stages_complete`:

   - `[]` → Stage 1 question 1.
   - `[anchor]` → Stage 2 question 1.
   - `[anchor, selection]` → Stage 3 for the first competitor.
   - `[anchor, selection, research]` → Stage 4 (deterministic).
   - `[anchor, selection, research, grid]` → Stage 5a.
   - `[anchor, selection, research, grid, positioning]` → Stage
     6 (or skip if no `Assumed:` lines).
   - `[anchor, selection, research, grid, positioning, validation]`
     → Stage 7.

3. **Run Stage 1.** Anchor turns. Append to artifact. Add
   `anchor` to `stages_complete`. Save.

4. **Run Stage 2.** Competitor selection turn. Tool-availability
   check. Write `web_fetch_available` and the competitor list.
   Add `selection` to `stages_complete`. Save.

5. **Run Stage 3.** Per-competitor research loop. For each
   competitor, run 3a (when fetch available) → 3b → 3c. Append
   each completed competitor block under `## Competitors` and
   save before moving to the next. Increment
   `competitors_researched` per completion. When all are
   researched, add `research` to `stages_complete`. Save.

6. **Run Stage 4.** Build the grid table deterministically from
   the per-competitor blocks. Append under `## Competitive
   grid`. Add `grid` to `stages_complete`. Save.

7. **Run Stage 5.** Five subsections in order (5a → 5e). After
   each subsection's `AskUserQuestion` resolves, append to the
   artifact and save. The Differentiation summary runs
   `stop-slop` before presentation; the bulleted subsections do
   not (they're short structured fields). Add `positioning` to
   `stages_complete`. Save.

8. **Run Stage 6.** Scan the full artifact for `Assumed:`
   prefixes. For each match, one `AskUserQuestion`. Update the
   artifact in place — drop the prefix when the client confirms
   or corrects, leave it when the client skips. Save after each
   resolution. When done, add `validation` to `stages_complete`.
   Save.

9. **Run Stage 7.** Key takeaways pre-draft + confirmation.
   Append under `## Key takeaways from the competitive
   landscape`. Add `takeaways` to `stages_complete`. Save.

10. **Mark complete.** Set `status: complete` in front-matter.
    Save. Return to `run-intake` or to the user with the
    standard quality report:

    - `competitor_count` and `web_fetch_available`.
    - How many `Assumed:` lines remain in the final artifact
      (signal for the client about confidence).
    - Whether all five positioning subsections rendered or any
      were skipped.
    - Next-step prompt: "Run `intake-platinum-message` next, or
      run `compile-profile` only when every intake sub-skill is
      complete."

## Drop handling mid-flow

Per the shared voice reference. Routing for this skill:

- **A competitor URL** mid-Stage-2 or mid-Stage-3: accept it as
  an addition / replacement to the current competitor list (Stage
  2) or as a re-fetch input for the current competitor (Stage 3,
  in which case run 3a fresh against the new URL).
- **A piece of competitive research** the client did themselves
  (a positioning doc, an analyst report .pdf): drop in
  `01_knowledge_base/raw/`, log it for `kb-ingest`'s next pass,
  reference it in the artifact's Brand Edge subsection with a
  `<!-- source: 01_knowledge_base/raw/<file> -->` provenance
  comment when it shapes a bullet.
- **A tagline or value statement** the client wants to test in
  the Differentiation summary: hold it in memory; surface it
  during Stage 5b as a pre-draft variant.

Do not advance the question cursor when handling a drop.

## Stop-slop integration

`stop-slop` runs on one prose artifact:

- The Stage 5b Differentiation summary paragraph, every time the
  skill produces or revises it.

The bullet subsections (Brand Edge, Messaging opportunities,
Risks to watch, Quick wins, Key takeaways) are short structured
fields that the client-confirm step polices; stop-slop does not
run on them. The competitive-grid table is also stop-slop-free —
the language inside is each competitor's, not the firm's.

## Pre-draft sources

Reading order, in priority:

1. `00_intake/intake/icp.md` — mandatory ICP context for Stage 5.
2. `00_intake/intake/transformations.md` — Brand Edge draws on
   pressure-tested top results.
3. `00_intake/client-profile.md` (when a prior profile exists —
   re-runs).
4. `01_knowledge_base/processed/**` — any uploaded competitive
   research already cleaned.
5. `01_knowledge_base/raw/**` — uploaded but not yet processed.
6. Live web (Stage 3) — competitor URLs, fetched directly.

Confidence labels accompany every pre-draft.

## Failure modes

| Failure                                            | Behavior                                                                       |
|----------------------------------------------------|--------------------------------------------------------------------------------|
| ICP or transformations artifact missing            | Refuse to run. Tell the client which intake step still needs to finish.        |
| Stage 2 web search fails                           | Mark `web_fetch_available: false` and proceed in paste-in mode for Stage 3.    |
| Stage 3 fetch returns 404 / paywalled / empty      | Capture the failure in the competitor's block (`Notes: site unreachable on    |
|                                                    | [ISO]`) and fall back to paste-in for that competitor.                         |
| Client lands at 1 competitor instead of 2 or 3     | Accept it. Mark `competitor_count: 1`. The grid still renders; the Brand Edge  |
|                                                    | subsection works with thin signal.                                             |
| Stage 5b revision loop hits cap                    | Capture the client's last-stated language verbatim, skip the polish pass.      |
|                                                    | Stop-slop violations are tolerable when the client explicitly wanted them.     |
| Web fetch tools mid-skill become available         | Don't auto-switch modes. The skill respects the `web_fetch_available` value    |
|                                                    | written at Stage 2. The client can redo Stage 3 manually via "redo step N."    |
| Client says "stop" mid-stage                       | Save current state, leave `status: in_progress`, return cleanly.               |
| Validation pass finds zero `Assumed:` lines        | Skip Stage 6 entirely. Mark it complete in `stages_complete` without asking.   |
| Duplicate competitor names in the same run         | Ask the client to disambiguate (add a parenthetical region or product line).   |

## What NOT to do

- Do not batch questions. One `AskUserQuestion` per field.
- Do not exceed 3 competitors. The original Competition Crusher
  GPT capped at 3 for a reason — past 3, triangulation becomes
  noise. When a client wants to map 5 or 6, capture the top 3 in
  this artifact and tell them the rest can wait for a quarterly
  refresh.
- Do not auto-resolve `Assumed:` lines. Stage 6 exists because
  unconfirmed extractions are worth a deliberate client pass.
- Do not paraphrase a competitor's language in their voice. The
  per-competitor sections capture how the competitor describes
  themselves; the language is theirs, not the firm's. Stop-slop
  does not run on these fields.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill only owns `intake/competitors.md` and may
  append to `01_knowledge_base/raw/` when the client drops files.
- Do not preserve corporate-consulting register from the original
  Competition Crusher GPT. The unified intake voice applies even
  when the subject matter is positioning strategy.
