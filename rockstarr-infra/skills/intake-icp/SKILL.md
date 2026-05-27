---
name: intake-icp
description: "This skill should be used when the user asks to \"run the ICP interview\", \"capture the ideal client profile\", \"build the ICP\", \"run the perception gap exercise\", or when run-intake dispatches to the ICP step. Walks the client through Phase A (7-step ICP interview) and Phase B (7-step Perception Gap exercise), one question at a time in the unified intake voice. Supports multiple ICPs via explicit loop. Checkpoints to /00_intake/intake/icp.md. Phase A output is the ICP section of client-profile.md; Phase B feeds the Positioning section."
---

# intake-icp

Capture the client's Ideal Client Profile(s) in two phases:

- **Phase A — ICP interview.** Seven questions per ICP. Builds the
  profile of the client's buyer (or buyers — multi-ICP is real and
  routine).
- **Phase B — Perception Gap.** Seven questions per ICP. Pressure-
  tests Phase A by comparing what the client thinks they deliver
  to what their clients perceive.

This skill is one of six intake sub-skills inside `rockstarr-infra`'s
v0.9.x intake flow. It writes `/rockstarr-ai/00_intake/intake/icp.md`.
`compile-profile` reads that file and assembles the ICP and
Positioning sections of `client-profile.md`. Downstream bots
(Content's `ideate-topics`, Outreach's `qualify-lead`) eventually
read those sections.

Read
`rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule in this skill comes from
there.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` after `intake-background` completes.
- Standalone, when an operator re-runs the ICP step via `run-intake`'s
  "redo step N" path.
- Standalone, when an operator says "redo the ICP interview" or "the
  ICP has shifted and I need to capture the new one."

## Preconditions

- `/rockstarr-ai/00_intake/intake/` exists. `run-intake` or
  `scaffold-client` creates it.
- `_progress.md` exists in that directory.
- `intake-interviewer-voice.md` is on disk inside the plugin (every
  intake-* skill in this plugin depends on it).

When `intake/icp.md` already exists and its front-matter shows
`status: in_progress`, this is a resume. Read the file, count the
ICPs present and the answers within each, and pick up at the next
unanswered question. Don't re-ask anything already captured.

When `intake/icp.md` exists and shows `status: complete`, ask the
client: "ICP is already captured. Redo from scratch, add another
ICP, or exit?" Default action is exit.

## Chat narration discipline

Two shared voice references govern what this skill says:

- **`skills/_shared/references/intake-interviewer-voice.md`** —
  the AskUserQuestion turns themselves.
- **`skills/_shared/references/client-facing-output-voice.md`** —
  everything between the questions: phase-transition lines,
  capture acknowledgments, the post-completion summary.

Apply both. Specifically for this sub-skill:

- **No "Phase A" / "Phase B" labels in chat transitions.** Use
  plain English: "Now let's pressure-test how your clients describe
  you" not "Phase B — Perception Gap exercise."
- **No artifact paths in capture acknowledgments.** The fact that
  answers land in `00_intake/intake/icp.md` is invisible to the
  client.
- **One short sentence per phase transition.** The pre-draft /
  confirm pattern (from `intake-interviewer-voice.md`) is the
  question discipline; what surrounds it stays short.
- **Multi-ICP loop transitions** stay plain: "Got the first ICP.
  Want to capture another one or stop here?" Not "ICP #1 status:
  complete. Multi-ICP loop continuing."

## The two phases

### Phase A — ICP interview (per ICP)

Seven questions, asked one at a time via `AskUserQuestion`. The
sequence:

1. **Name** — Working name for this ICP (e.g., "Savvy SoloPreneur",
   "Mid-50s CEO", "HR/CSR leader"). Short, memorable, lowercase or
   title-case as the client prefers.

2. **Demographics** — Age range, geography, occupation or industry,
   title, education or background. Pre-draft from
   `client-profile.md` (if a prior profile exists) and from any
   `background.md` notes about the audience. Confidence label
   required.

3. **Sources of information** — Where this buyer finds people like
   the client. Social channels, peer networks, conferences, podcasts,
   trade pubs, search behavior. Pre-draft from KB material in
   `01_knowledge_base/raw/` if any reads like positioning research.

4. **Core situation** — The client's life on a normal Tuesday before
   they meet the seller. What are they doing, what's hurting, what's
   on their plate. Plain prose, two to four sentences. Specific
   beats general.

5. **Pain points** — One pain per turn in a loop, not a bulleted list
   captured in a single answer. After each captured pain, ask "Add
   another? (yes / no)". Aim for three at minimum, five to ten in
   the median case. The interviewer pushes for specificity per pain
   ("anxiety about quarterly numbers" beats "stressed"); same
   one-push discipline as other intake skills. The captured pains
   render as a bulleted list in the artifact, but every bullet is
   its own captured field — never a free-text list typed into one
   answer box.

6. **Transformation sought** — What changes when the buyer engages
   the client's solution. Outcome-focused, not activity-focused.
   Bulleted list. Three to six items.

7. **Decision drivers and objections** — How they decide, what they
   weigh, what makes them hesitate. Two sub-fields, captured
   together: decision drivers (bulleted) and objections (bulleted,
   with the move that overcomes each one when the client can name
   it).

After question 7, synthesize a one-paragraph **narrative ICP
summary** and present it for confirm / amend / reject. Once
confirmed, append the ICP block to `intake/icp.md` and ask the
multi-ICP question (below).

### Phase B — Perception Gap (per ICP)

Seven questions, asked one at a time. The exercise pressure-tests
the Phase A ICP by comparing the client's view of what they deliver
against the buyer's perception of what they get. Same voice as
Phase A — direct, no theater.

1. **Confirm who we're testing.** Read the ICP name back from Phase
   A. One sentence. ("Pressure-testing the Mid-50s CEO ICP. Same
   buyer we captured in Phase A. Yes?")

2. **What you actually deliver.** Three to seven outcomes, plainly
   stated. Pre-draft from Phase A's transformation list — Phase B
   forces a re-read of what's there.

3. **What you think they think you deliver.** The client's
   prediction of how their buyer would describe the service. One
   sentence per prediction.

4. **Capture real perception.** If the client has direct evidence
   (testimonials, review quotes, survey data, recorded calls),
   pull it from `01_knowledge_base/raw/` and surface what's
   actually said. If no evidence on disk, ask the client to recall
   the language a recent client used.

5. **Name the gap.** Side-by-side: prediction vs. reality. The
   interviewer renders a two-column table and shows it to the
   client. The client confirms which row(s) reveal a real gap.

6. **Reframe.** For each gap row the client confirmed, ask: how
   should you describe what you deliver, so the buyer's actual
   perception matches the description? Capture the reframe in plain
   language.

7. **Closing summary.** Synthesize a three-to-five-sentence
   positioning paragraph that uses the reframed language. Present
   for confirm / amend / reject. Once confirmed, append the Phase
   B block to `intake/icp.md` under the matching ICP.

After Phase B for an ICP completes, ask: "Add another ICP? (y/n)"
If yes, loop back to Phase A for the next ICP. If no, mark the
artifact `status: complete` and return.

## Multi-ICP loop

After the FIRST ICP's Phase A + Phase B both complete, the multi-ICP
question fires for the first time. After every subsequent ICP's
Phase A + Phase B, it fires again. The exit condition is the client
saying "no more ICPs" (or equivalent — "done", "that's all",
"finished").

The skill warns the client on the second loop iteration and after:
each additional ICP adds roughly 15 to 25 minutes of interview time.
A two-ICP client takes 45 to 60 minutes; a five-ICP client (like the
Beyond Basic Needs fixture) takes two hours. The warning is
informational, not an attempt to talk the client out of accurate
input.

When multi-ICP runs produce overlapping ICPs, the skill does NOT
auto-merge. The client can redo from scratch (drops everything) or
let the overlapping ICPs stand. Downstream skills handle de-
duplication via the style-guide pass.

## Artifact shape: `intake/icp.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "icp"
sub_skill: "intake-icp"
sub_skill_version: "0.1.0"
status: "in_progress"     # not_started | in_progress | complete
captured_at: "<ISO 8601 of last write>"
icp_count: 1
phase_a_complete_for: ["icp-slug-1"]
phase_b_complete_for: ["icp-slug-1"]
# ---
~~~

The two `phase_*_complete_for` lists are the resume index. On every
write, the skill updates these arrays so resume knows exactly which
ICPs need which phase finished.

Body shape:

~~~markdown
# ICP intake

## Phase A — ICP profiles

### <ICP Name 1>

<!-- source: client interview, <ISO date> -->

- **Demographics:** ...
- **Sources of information:** ...
- **Core situation:** ...
- **Pain points:**
  - ...
- **Transformation sought:**
  - ...
- **Decision drivers:**
  - ...
- **Objections:**
  - ...

**Narrative summary:**

<one paragraph, stop-slop polished, client-confirmed>

### <ICP Name 2>

<!-- source: client interview, <ISO date> -->

...

## Phase B — Perception Gap

### <ICP Name 1>

<!-- source: client interview, <ISO date> -->

**What you deliver:**
- ...

**What you think they think you deliver:**
- ...

**What they actually say:**
- ...
- <!-- source: 01_knowledge_base/raw/<file> --> when evidence was the basis

**Gap table:**

| What clients think we do | What we actually do |
|--------------------------|---------------------|
| ...                      | ...                 |

**Reframed positioning:**

<paragraph, stop-slop polished, client-confirmed>

### <ICP Name 2>

...
~~~

Always H3 per ICP, even with one ICP. Always both phases per ICP,
in the same order.

## Steps

1. **Read preconditions.** Confirm `intake/icp.md` exists or create
   it with the front-matter block (`status: not_started`, empty
   arrays). Confirm `intake-interviewer-voice.md` is on disk.

2. **Resume decision.** Parse the current file. Compute the resume
   point:

   - If `status: not_started`, start at Phase A question 1 for the
     first ICP.
   - If `status: in_progress`, find the first ICP missing from
     `phase_b_complete_for`. For that ICP, find the next
     unanswered question by reading the body. Resume there.
   - If `status: complete`, ask the redo / add / exit question.

3. **Run Phase A for the current ICP.** Questions 1 through 7,
   `AskUserQuestion` per turn. After each answer, write the
   answer to `intake/icp.md` under that ICP's H3, update
   `captured_at`, save. Honor pause: at any turn the client can say
   "stop" and the skill returns.

4. **Run the narrative-summary draft.** After question 7,
   synthesize a one-paragraph summary using the seven captured
   fields. Run `stop-slop` over the paragraph. Present via
   `AskUserQuestion` with options confirm / amend / reject. Capture
   the resolution.

5. **Append ICP to `phase_a_complete_for`.** Update front-matter,
   save.

6. **Run Phase B for the current ICP.** Questions 1 through 7,
   `AskUserQuestion` per turn. Apply the same write-after-every-
   answer discipline.

7. **Run the reframed-positioning draft.** Synthesize the three-to-
   five-sentence positioning paragraph. Run `stop-slop`. Present
   via `AskUserQuestion` with options confirm / amend / reject.
   Capture the resolution.

8. **Append ICP to `phase_b_complete_for`.** Update front-matter,
   save.

9. **Multi-ICP gate.** Ask the client: "Add another ICP? (y/n)"

   - **Yes:** Increment `icp_count` in front-matter, return to step
     3 for the next ICP. Warn about time on the second iteration
     and after.
   - **No:** Set `status: complete`, save, exit.

10. **Return to `run-intake`** (or to the user, when running
    standalone). Print:

    - Number of ICPs captured.
    - File path written.
    - Next-step prompt: "Run `intake-transformations` next, or run
      `compile-profile` only when every intake sub-skill is
      complete."

## Drop handling mid-flow

Per the shared voice reference, the client may paste a URL, upload
a file, or quote a document mid-question. Route as follows:

- **A voice sample** (the client offers a quote they want
  captured): append the quote to `00_intake/samples/voice-notes.md`
  with a one-line provenance comment naming the ICP context. Then
  return to the question in flight.
- **A testimonial or review URL** (relevant to Phase B question 4):
  fetch the page if web tools are available, extract the relevant
  quote, append to `intake/icp.md` under the current ICP's Phase B
  section with a `<!-- source: <URL> -->` provenance comment. If
  web tools are blocked, ask the client to paste the relevant text
  and capture from chat. Then return to the question.
- **A case study or first-party document**: drop in
  `01_knowledge_base/raw/`, log it for `kb-ingest`'s next pass,
  reference it in the current Phase B section with a `<!-- source:
  01_knowledge_base/raw/<file> -->` comment when its content
  shapes an answer. Then return to the question.

Do not advance the question cursor when handling a drop. The client
expects to answer the question after the file is in place.

## Pre-draft sources

Where evidence exists on disk, the skill drafts proposed answers
before asking. Reading order:

1. `/rockstarr-ai/00_intake/client-profile.md` (when a prior
   profile exists — common on re-runs).
2. `/rockstarr-ai/00_intake/intake/background.md` (when
   `intake-background` ran first and captured the audience).
3. `/rockstarr-ai/01_knowledge_base/processed/` (when `kb-ingest`
   has already run on first-party material that reads like ICP
   research — case studies, audience research, sales conversations).

Confidence labels (HIGH / MEDIUM / LOW) accompany each pre-draft.
HIGH means the evidence is direct and recent; MEDIUM means the
evidence is suggestive; LOW means the proposal is the skill's best
read of sparse signal.

## Stop-slop integration

`stop-slop` runs on two prose artifacts inside this skill:

- The Phase A narrative ICP summary (one paragraph, after question
  7, before showing to the client).
- The Phase B reframed positioning paragraph (three to five
  sentences, after question 7, before showing to the client).

It does NOT run after every individual question — those answers are
short, structural, and the client-confirm step already polices them.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| `intake/icp.md` malformed front-matter           | Refuse to resume. Surface the issue, suggest manual fix or redo from scratch.  |
| Web tools blocked during Phase B Q4              | Fall back to asking the client to paste the relevant testimonial text.         |
| Client says "stop" mid-question                  | Save current state, set `status: in_progress`, return cleanly.                 |
| Client contradicts a Phase A answer in Phase B   | Surface the contradiction. Ask which is right. Update Phase A via the same     |
|                                                  | append-and-resave discipline (Phase A block gets a fresh provenance comment).  |
| Stop-slop catches violations the client wrote    | Polish the prose paragraphs but do NOT touch the structural bullet lists.      |
|                                                  | Client review confirms the polished paragraph.                                 |
| Duplicate ICP names in the same run              | Ask the client to disambiguate (e.g., "Mid-50s CEO" → "Mid-50s CEO (energy)"   |
|                                                  | and "Mid-50s CEO (defense)"). Slugs in `phase_*_complete_for` arrays must be   |
|                                                  | unique.                                                                        |

## What NOT to do

- Do not batch questions. One `AskUserQuestion` per field.
- Do not skip Phase B for a "small" ICP. Both phases run on every
  ICP, every time. Skipping Phase B is a separate compile-profile
  failure mode that costs more later than running it now.
- Do not auto-merge overlapping ICPs. The client decides.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill only owns `intake/icp.md` and may append to
  `samples/voice-notes.md` and `01_knowledge_base/raw/`.
- Do not fabricate testimonials in Phase B question 4. Use what's
  on disk; ask the client when nothing is on disk; mark "not
  provided" when the client genuinely has nothing.
- Do not preserve game-show-host or corporate-consulting register
  from the original ChatGPT custom GPTs. The unified intake voice
  applies to every question, including Phase B.
