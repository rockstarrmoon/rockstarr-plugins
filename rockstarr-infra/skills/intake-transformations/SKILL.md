---
name: intake-transformations
description: "This skill should be used when the user asks to \"run the transformations step\", \"capture top results\", \"do the top transformations exercise\", \"list our wins\", or when run-intake dispatches to the transformations step of the intake flow. Walks the client through a five-stage process — ICP review, evidence pre-read, candidate list (5 to 10 transformations with specifics), pressure-test each (repeatable, why-it-worked, right-fit dependency), and narrow to top 3 to 5. One question at a time, in the unified intake voice. Checkpoints every answer to /00_intake/intake/transformations.md so the long interview survives pauses. Feeds the Proof / results section of client-profile.md."
---

# intake-transformations

Capture the client's top results — the transformations they deliver
that downstream bots cite as proof. Five stages:

- **Stage 1 — ICP review.** Read the ICPs from `intake/icp.md`,
  surface them, confirm with the client.
- **Stage 2 — Evidence pre-read.** Scan
  `01_knowledge_base/raw/` for case studies, testimonials, invoice
  notes, anything that already documents past wins. Surface what's
  there as candidate transformations.
- **Stage 3 — Candidate list.** Walk the client through capturing
  5 to 10 transformations, one at a time. Push for specifics.
- **Stage 4 — Pressure-test each.** For each candidate, ask three
  questions: can you deliver this again, do you know why it
  worked, does it depend on right-fit clients.
- **Stage 5 — Narrow to top 3 to 5.** The client picks the
  transformations that matter most to the ICPs we want more of,
  and are easy to prove.

This is one of six intake sub-skills inside `rockstarr-infra`'s
v0.9.x intake flow. Writes `/rockstarr-ai/00_intake/intake/transformations.md`.
`compile-profile` reads that file to assemble the **Proof / results**
section of `client-profile.md`.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule comes from there. There is no
GPT prompt to channel — this exercise is process-driven, which makes
the unified intake voice the default register from question one.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` after `intake-icp` completes.
- Standalone, when an operator re-runs the transformations step via
  `run-intake`'s "redo step N" path.
- Standalone, when an operator says "redo transformations" or "the
  proof points have changed and I need to recapture."

## Preconditions

- `/rockstarr-ai/00_intake/intake/icp.md` exists and shows
  `status: complete`. Transformations are pressure-tested against
  the ICPs; without ICPs, the pressure-test is meaningless. Refuse
  to run when ICPs are missing or in_progress and tell the client
  to finish `intake-icp` first.
- `/rockstarr-ai/00_intake/intake/` is writable.
- `intake-interviewer-voice.md` is on disk inside the plugin.

When `intake/transformations.md` already exists with `status: in_progress`,
this is a resume. Read the file, count the candidates captured and
which pressure-test fields are filled, and pick up at the next
unanswered question.

When the file exists with `status: complete`, ask the client: "Top
results are already captured. Redo from scratch, add more, or exit?"
Default action is exit.

## Scope decision: transformations are global, not per-ICP

Some clients run a single ICP; some run five. In both cases,
transformations are captured as a flat list at the workspace level,
not partitioned per ICP. The pressure-test (Stage 4) and the
top-N narrowing (Stage 5) reference the ICPs and weight
transformations by their fit to the ICPs the client wants more
of — but the captured artifact is one list.

This matches both fixtures we have. The single-ICP fixture
(TRANSEARCH) and the five-ICP fixture (Beyond Basic Needs) both
produce a flat aggregate set of proof points in `client-profile.md`.

## The five stages

### Stage 1 — ICP review

One `AskUserQuestion` turn:

1. **Confirm the ICPs.** Read the ICP names and a one-sentence
   recap of each from `intake/icp.md` Phase A. Ask the client to
   confirm them as the lens for the pressure-test ahead. Options:

   - **Confirm** — proceed to Stage 2 using the ICPs as captured.
   - **Edit one** — return the client to `run-intake` to redo the
     ICP step. This skill exits cleanly. (When running standalone,
     suggest the client redo `intake-icp` before retrying.)

No transformations get captured here. This stage is just the
re-orientation.

### Stage 2 — Evidence pre-read

Read what's on disk before asking the client to recall wins from
memory. Scan in this order:

1. `/rockstarr-ai/00_intake/client-profile.md` (when one exists
   from a prior run — Proof / results section is the obvious
   starting point).
2. `/rockstarr-ai/01_knowledge_base/processed/` (when `kb-ingest`
   has run on case studies, testimonials, recap docs).
3. `/rockstarr-ai/01_knowledge_base/raw/` (when raw uploads are
   sitting unprocessed — case-study PDFs, testimonial screenshots,
   invoice notes, project recap .docx files).

For each candidate piece of evidence the scan surfaces, draft a
**proposed transformation** in the form:

> We helped `<client / role>` `<specific outcome>` in `<time
> frame>` by `<the core action that made it work>`.

Confidence label per draft (HIGH / MEDIUM / LOW). HIGH = the
evidence cites a quantifiable outcome with a timeframe; MEDIUM =
the evidence names an outcome but no timeframe; LOW = the
evidence hints at an outcome but specifics are sparse.

Present the draft list to the client in one `AskUserQuestion` with
four options:

- **Use as starting point** — the drafts seed Stage 3; the client
  edits and adds during Stage 3.
- **Use selectively** — the client picks which drafts feed Stage
  3.
- **Start from scratch** — the client wants to recall from memory
  without anchoring to the drafts.
- **Add a new file first** — the client wants to drop a case
  study or testimonial that's not yet on disk before this stage
  runs. Honor the drop, then re-scan.

When nothing is on disk, skip the question and go straight to
Stage 3. Write a note in the artifact's "Pre-read evidence"
subsection: `_no evidence on disk at intake time._`

### Stage 3 — Candidate list

Capture 5 to 10 transformations, one per `AskUserQuestion` turn.
The minimum target is 5 — if the client lands at 3, ask once
whether there are more they want to add; accept their answer.
The cap is 10 — past 10, the value falls off and the narrowing
in Stage 5 takes longer.

Each capture has three sub-fields:

| Field          | Prompt / guidance                                                                                |
|----------------|--------------------------------------------------------------------------------------------------|
| **Outcome**    | What changed. Specific. ("Increased recurring revenue by 27% in 4 months" not "Increased revenue.") |
| **Who**        | Which client, role, industry, or anonymized cohort.                                              |
| **How**        | One sentence on the action or methodology that made it work.                                     |

The interviewer pushes for specificity at the Outcome field. If the
client offers a vague answer ("we helped them grow"), ask one
sharpening question: "What does 'grow' mean here — a number, a
timeframe, a milestone?" Capture the sharpened version. Don't
push twice; if the client doesn't have a number, capture the
sharper non-quantified outcome and move on.

After each capture, append the transformation as a numbered entry
to `intake/transformations.md` under `## Candidate transformations`
and save. Update `captured_at` and increment `candidate_count` in
front-matter.

When the client says "that's all" / "done" / "no more" before
hitting 5, accept it and continue. When they hit 10, surface a
prompt: "We have 10. Do you want to add more or move to the
pressure-test?" Default action is move on.

### Stage 4 — Pressure-test each

For every captured candidate, run three questions one at a time:

1. **Repeatable.** "Can you deliver this outcome again for a
   similar client?" Three options: yes / yes-with-conditions /
   no-it-was-unique. When yes-with-conditions, capture the
   conditions in a follow-up free-text turn.

2. **Why it worked.** "Why did this work? What was the key move
   or condition?" Free text. The interviewer pushes for the
   actual mechanism, not the activity ("the candidate's prior
   board experience matched the company's growth stage," not "we
   ran a search").

3. **Right-fit dependency.** "Does this outcome depend on a
   right-fit client, or could you deliver it for any client who
   walked in?" Three options: requires-right-fit /
   helped-by-right-fit / works-for-anyone.

Append each answer to the matching candidate in the artifact under
sub-bullets. After all three answers for a candidate land, save
the artifact.

Stage 4 is the longest stage. A 10-candidate run is 30 turns. The
pause discipline matters — clients reliably stop here.

### Stage 5 — Narrow to top 3 to 5

Present the full candidate list with pressure-test notes
inline. Ask the client to pick the **top 3 to 5** transformations
that meet all three criteria:

- Matter most to the ICPs we want more of (Stage 1's
  re-orientation pays off here).
- Are easy to prove (Stage 4's "repeatable + why it worked"
  answers are the proof basis).
- Align with the work the client actively wants more of.

Single `AskUserQuestion` per pick: "Top transformation #1?"
Options: each remaining candidate, plus "stop picking" when the
client has at least 3 picks.

After the picks, present a **summary paragraph** synthesizing the
top transformations and what they reveal about the firm's value.
Two to four sentences. Run `stop-slop` over the paragraph.
Present for confirm / amend / reject. Capture the resolution.

Set `status: complete`, save, exit.

## Optional sub-sections

Two artifacts in `intake/transformations.md` are optional. Surface
them only when the relevant material exists.

### Case studies

When the Stage 2 pre-read found named case studies (with titles
and URLs in the KB), or when the client during Stage 3 references
named published case studies, capture them in a `## Case studies`
subsection of the artifact as a markdown table:

~~~markdown
## Case studies

<!-- source: 01_knowledge_base/raw/<files> -->

| # | Title | URL or source |
|---|-------|---------------|
| 1 | ...   | ...           |
~~~

`compile-profile` lifts this table into the Proof / results
section under `### Case studies`. When no case studies were
referenced, omit this subsection from the artifact and
compile-profile renders nothing.

### Quantifiable proof points

When the client cites aggregate stats they use publicly
("8,300+ patients reached," "90%+ placement rate vs. 67% industry
average," "$9–$11 per kit"), capture them in a `## Quantifiable
proof points` subsection of the artifact as a bullet list. Ask
the client at the end of Stage 5 with a single `AskUserQuestion`:
"Do you have aggregate stats you cite publicly that we should
surface in your profile?" Capture each one with the same
specificity push as Stage 3 outcomes.

When the client has none, omit this subsection.

## Artifact shape: `intake/transformations.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "transformations"
sub_skill: "intake-transformations"
sub_skill_version: "0.1.0"
status: "in_progress"     # not_started | in_progress | complete
captured_at: "<ISO 8601 of last write>"
candidate_count: 0
top_count: 0
stages_complete: []        # subset of: pre_read, candidates, pressure_test, narrowed
# ---
~~~

`stages_complete` is the resume index alongside `candidate_count` and
`top_count`. The skill's resume logic reads these arrays plus the
body to determine the next question.

Body shape:

~~~markdown
# Top transformations intake

## ICPs in scope

<!-- source: 00_intake/intake/icp.md -->

- <ICP Name 1>: <one-sentence recap>
- <ICP Name 2>: <one-sentence recap>

## Pre-read evidence

<!-- source: 01_knowledge_base/raw/<files>, 01_knowledge_base/processed/<files> -->

<short paragraph describing what the scan found, or "_no evidence
on disk at intake time._">

## Candidate transformations

1. **Outcome:** Increased recurring revenue by 27% in 4 months.
   **Who:** SaaS company, post-Series B, 50 employees.
   **How:** Recruited a head of sales with prior PLG conversion
   experience.

   - **Repeatable:** yes-with-conditions — depends on the buyer's
     comp model.
   - **Why it worked:** The new hire had run the same play at a
     comparable-stage company.
   - **Right-fit dependency:** helped-by-right-fit.

2. ...

## Top transformations (client-confirmed top 3 to 5)

<!-- source: client interview, <ISO date> -->

1. <full transformation entry, lifted from Candidates>
2. ...

**Summary:**

<stop-slop-polished paragraph, two to four sentences,
client-confirmed>

## Case studies

<!-- source: 01_knowledge_base/raw/<files> -->

| # | Title | URL |
|---|-------|-----|
| 1 | ...   | ... |

## Quantifiable proof points

<!-- source: client interview, <ISO date> -->

- 8,300+ patients reached.
- All 50 states.
~~~

The two optional subsections (Case studies, Quantifiable proof
points) appear only when relevant content exists.

## Steps

1. **Read preconditions.** Confirm `intake/icp.md` exists with
   `status: complete`. If not, surface the issue and exit cleanly:
   "Top transformations need the ICPs as the lens. Run intake-icp
   first." Confirm `intake-interviewer-voice.md` is on disk.

2. **Initialize or resume `intake/transformations.md`.** If the
   file doesn't exist, write the front-matter block with empty
   arrays. If it does, parse front-matter and body. Compute the
   resume point:

   - `stages_complete: []` → Stage 1.
   - `stages_complete: [pre_read]` → Stage 3 question 1.
   - `stages_complete: [pre_read, candidates]` → Stage 4 question 1
     for the first candidate.
   - `stages_complete: [pre_read, candidates, pressure_test]` →
     Stage 5 question 1.
   - `stages_complete: [pre_read, candidates, pressure_test, narrowed]`
     → ask the optional-subsections questions, then exit.

3. **Run Stage 1.** ICP confirmation turn. On confirm, mark Stage 1
   passed in `stages_complete` and proceed. On edit, exit and tell
   the client to redo `intake-icp`.

4. **Run Stage 2.** Pre-read the three KB locations. Draft
   candidate transformations with confidence labels. Present via
   `AskUserQuestion` with the four options. Capture the choice.
   On drop-a-new-file, accept the file per the shared voice
   reference, re-scan, present again. On done, append the chosen
   drafts (or "_no evidence on disk at intake time._") to the
   artifact's Pre-read evidence subsection. Add `pre_read` to
   `stages_complete`. Save.

5. **Run Stage 3.** Candidate loop. For each candidate:
   - Outcome → Who → How, one `AskUserQuestion` per field.
   - After all three sub-fields, append the candidate to the
     Candidate transformations subsection. Increment
     `candidate_count`. Save.
   - Sharpening prompt only when the Outcome is vague. Don't
     push twice.
   - After each candidate, ask: "Add another? (y/n)". Track loop
     bounds (5 minimum encouraged, 10 maximum).

   When the loop exits, add `candidates` to `stages_complete`.
   Save.

6. **Run Stage 4.** Pressure-test loop. For each candidate in
   order:
   - Repeatable → Why → Right-fit, one `AskUserQuestion` per
     field. Yes-with-conditions on Repeatable triggers a free-text
     follow-up.
   - After each candidate's three answers, append them as sub-
     bullets under the candidate in the artifact. Save.

   When all candidates are pressure-tested, add `pressure_test` to
   `stages_complete`. Save.

7. **Run Stage 5.** Narrowing loop:
   - Pick 1, Pick 2, Pick 3 in sequence — one `AskUserQuestion`
     each with remaining candidates as options.
   - After Pick 3, add "stop picking" as an option alongside the
     remaining candidates for Picks 4 and 5.
   - Append the picks to the Top transformations subsection.
     Update `top_count`. Save.

   Generate the summary paragraph. Run `stop-slop`. Present for
   confirm / amend / reject. Capture the resolution. Save.

   Add `narrowed` to `stages_complete`. Save.

8. **Optional subsections.**
   - Ask once: "Do you cite aggregate stats publicly that we
     should surface (placement rate, patients reached, etc.)?"
     If yes, loop captures one stat per turn until "no more."
   - If the Stage 2 pre-read surfaced named case studies, render
     the Case studies table. Otherwise omit.

9. **Mark complete.** Set `status: complete` in front-matter.
   Save. Return to `run-intake` or to the user (when running
   standalone) with the standard quality report:

   - Candidate count.
   - Top-N count.
   - Whether Case studies and Quantifiable proof points
     subsections were populated.
   - Next-step prompt: "Run `intake-competitors` next, or run
     `compile-profile` only when every intake sub-skill is
     complete."

## Drop handling mid-flow

Per the shared voice reference, the client may paste a URL or
upload a file mid-question. Routing for this skill:

- **A case study or recap document** (.pdf, .docx, .md): drop in
  `01_knowledge_base/raw/`. Re-scan the KB and present the new
  candidate(s) in the next pre-read prompt; if already past Stage
  2, surface the new file as an additional draft for the current
  candidate or as a new candidate the client can add to the list.
- **A testimonial URL** (a publicly hosted quote or review): fetch
  the page if web tools are available, extract the relevant
  passage, append to the artifact under the relevant candidate
  with a `<!-- source: <URL> -->` provenance comment. If web tools
  are blocked, ask the client to paste the relevant text.
- **An aggregate stat the client cites publicly** (a Quantifiable
  proof point dropped mid-flow): hold it in memory; surface it
  during Stage 8's optional-subsection capture.

Do not advance the question cursor when handling a drop. The
client expects to answer the question after the file is in place.

## Stop-slop integration

`stop-slop` runs on one prose artifact in this skill:

- The Stage 5 summary paragraph (two to four sentences, after the
  picks, before showing to the client).

The bullet structures (candidate entries, pressure-test notes,
proof-point bullets) are short structural answers; stop-slop is not
run on them. The client-confirm step polices the captured prose at
the field level.

## Pre-draft sources

Where evidence exists on disk, the skill drafts proposals before
asking. Reading order, in priority:

1. `00_intake/intake/icp.md` — the ICPs are mandatory context.
2. `00_intake/client-profile.md` Proof / results section, if a
   prior profile exists (re-runs).
3. `01_knowledge_base/processed/**` — case studies, testimonials,
   recap docs already cleaned.
4. `01_knowledge_base/raw/**` — uploaded but not yet processed.

Confidence labels accompany every pre-draft, same convention as
`intake-icp`.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| `intake/icp.md` missing or in_progress           | Refuse to run. Tell the client to finish `intake-icp` first.                   |
| Client lands at fewer than 3 candidates          | Accept it. Mark `candidate_count: 2` (or less) in front-matter; Stage 5 still  |
|                                                  | runs but the picks max out at the candidate count.                             |
| Client says "stop" mid-stage                     | Save current state, leave `status: in_progress`, return cleanly.               |
| Web tools blocked during a testimonial fetch     | Fall back to asking the client to paste the relevant text.                     |
| Stage 2 finds no evidence on disk                | Write the sentinel `_no evidence on disk at intake time._` and proceed.        |
| Client's Stage 5 picks contradict Stage 4 notes  | Surface the contradiction. Ask if the picks should change or if the            |
|                                                  | pressure-test notes need editing. Honor whichever answer the client gives.     |
| Case studies were pre-read but client doesn't    | Capture the case studies in the table regardless. They're separate from the    |
|   pick any in Stage 5                             | top-N picks. compile-profile will surface both.                                |
| Stop-slop catches violations the summary wrote   | Polish the summary paragraph only. The captured bullet structures stay         |
|                                                  | exactly as the client confirmed them.                                          |

## What NOT to do

- Do not batch questions. One `AskUserQuestion` per field.
- Do not auto-quantify a vague outcome ("Increased revenue by an
  unspecified amount in an unspecified timeframe"). When the
  client can't quantify, capture the sharper non-quantified
  outcome as-is. Vagueness is a signal downstream skills can use.
- Do not partition transformations per-ICP in this artifact. The
  flat aggregate list is what `compile-profile` expects.
- Do not skip the pressure-test stage. Even when Stage 3 captured
  only 3 candidates, the pressure-test still runs on all 3. Why-
  it-worked is the proof basis for downstream drafting.
- Do not fabricate case studies or proof points. Use what's on
  disk or what the client confirms in chat. Omit the optional
  subsections when the underlying content is absent.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill only owns `intake/transformations.md` and may
  append to `01_knowledge_base/raw/` when the client drops files.
