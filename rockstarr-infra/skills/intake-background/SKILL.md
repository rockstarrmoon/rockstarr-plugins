---
name: intake-background
description: "This skill should be used when the user asks to \"start intake\", \"run the background step\", \"capture company background\", \"capture voice samples\", or when run-intake dispatches to the first step. Captures the three foundational inputs every other intake sub-skill depends on — tech stack (via capture-stack), content-library URLs + notes, voice samples + brand/style notes — plus a 1-3 paragraph Company description for the canonical profile's Company section. One question at a time in the unified intake voice. Checkpoints to /00_intake/intake/background.md. Writes downstream artifacts: stack.md, samples/content-library.md, samples/voice-notes.md."
---

# intake-background

The foundational intake step. Runs first in the interview flow. Five
stages:

- **Stage 1 — Tech stack.** Delegates to the existing `capture-stack`
  skill. Produces `00_intake/stack.md`.
- **Stage 2 — Company description.** Three questions plus a synthesis
  pass produces 1 to 3 short paragraphs describing the company. Stop-
  slop polished. Feeds the Company section of `client-profile.md`.
- **Stage 3 — Content library.** Loop capturing URLs and pasted /
  dropped content the client wants in the knowledge base. Tagged
  first-party or third-party. Written to
  `00_intake/samples/content-library.md` so `kb-ingest` can pick up
  the URLs on its next pass.
- **Stage 4 — Voice samples.** Loop capturing 2 to 3 (or more) voice
  samples — recent LinkedIn posts, newsletter excerpts, sales-call
  transcript snippets, anything the client wants the bots to sound
  like. Plus an optional brand / style notes block. Written to
  `00_intake/samples/voice-notes.md` so `generate-style-guide` can
  use it directly.
- **Stage 5 — Goals / constraints.** Optional in v0.9.x. Default
  outcome is `_not provided._` per the locked intake decisions.

This is the first of six intake sub-skills in `rockstarr-infra`'s
v0.9.x intake flow and the foundation every other sub-skill reads
from. Writes `/rockstarr-ai/00_intake/intake/background.md` (the
audit trail) plus three downstream-readable files.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Every discipline rule comes from there. There is no
GPT prompt to channel — the workbook's Action Item 1 covered some
of this surface, but the unified intake voice and capture-stack's
existing interview pattern set the register.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- Dispatched from `run-intake` as the first step of the interview
  path.
- Standalone, when an operator says "start intake," "capture
  background," "set up the knowledge base inputs," or "drop in
  voice samples."
- Re-runnable on demand when the client's tooling shifts (the
  Stage 1 capture-stack delegation handles updates), the company
  description changes (Stage 2), or new voice samples land (Stage 4).

## Preconditions

- `/rockstarr-ai/00_intake/intake/` exists. `run-intake` or
  `scaffold-client` creates it.
- `_progress.md` exists in that directory.
- `intake-interviewer-voice.md` is on disk inside the plugin.
- `capture-stack` is installed in the same plugin (it is — same
  plugin, same install).

When `intake/background.md` already exists with `status:
in_progress`, this is a resume. Read the file, check
`stages_complete`, pick up at the next unfinished stage.

When the file exists with `status: complete`, ask the client:
"Background already captured. Refresh the stack, add more content
library entries or voice samples, redo the company description, or
exit?" Default action is exit.

## Chat narration discipline

Two shared voice references govern what this skill says:

- **`skills/_shared/references/intake-interviewer-voice.md`** —
  the AskUserQuestion turns themselves (one question at a time,
  pre-draft → confirm / amend / reject / skip, the four discipline
  rules).
- **`skills/_shared/references/client-facing-output-voice.md`** —
  everything between the questions: stage-transition lines,
  capture acknowledgments, the post-completion summary.

Apply both. Specifically for this sub-skill:

- **No stage labels in chat transitions.** "Stage 3 — Content
  library" reads as internal structure. Use plain English: "Got
  it. Next: where can we find your published thinking?"
- **No artifact paths in capture acknowledgments.** The fact that
  answers land in `00_intake/intake/background.md` is invisible
  to the client.
- **One short sentence per stage transition.** Not "Captured: 4
  fields. Stage 2 complete."
- **Post-completion summary**: one sentence what's captured, one
  sentence about what comes next (which the orchestrator
  handles). File paths in a collapsed `[details]` footer.

## Stage 1 — Tech stack

Delegate to `capture-stack`. Three flows:

- **First-time intake** — `stack.md` doesn't exist. Call
  `capture-stack`; it runs its full interview. On return, mark
  `stack` complete in `stages_complete`.
- **Resume / re-run** — `stack.md` exists with the current
  `capture_skill_version`. Ask the client: "Stack already
  captured. Skip / refresh / show me what's there?" Default skip.
  On refresh, call `capture-stack`; it reads the existing values
  as defaults. On show, surface the captured tools and access
  status as a quick summary, then ask again.
- **Outdated stack** — `stack.md` exists with an older
  `capture_skill_version`. Tell the client: "Your stack was
  captured under skill version X. We've added fields since.
  Refresh now?" Default yes; refusal preserves the older file.

After the delegation returns, append a one-line audit note to
`intake/background.md` under `## Tech stack` pointing at
`stack.md` and recording the version stamp. Mark `stack` in
`stages_complete`. Save.

## Stage 2 — Company description

Three `AskUserQuestion` turns plus a synthesis pass.

1. **What does the company do?** One sentence summary. Plain
   language. The client supplies. Pre-draft from any seed material
   already on disk (a website-about file in
   `01_knowledge_base/raw/`, the company name + URL the user
   supplied during scaffolding) when available; ask cold otherwise.

2. **Who does it serve?** One sentence audience anchor. Mid-50s
   CEOs of PE-portfolio companies. Cancer patients in the first 90
   days of chemo. AEC-industry firms scaling regional leadership.
   Specific beats general. The interviewer pushes for specificity
   once if the answer is vague; no second push.

3. **Three to five key facts.** Bullet list. Things a stranger
   landing on the website should know within ten seconds: location
   or scope, founding year or scale, notable partnerships,
   methodology names, certifications, headline stat. Free text.
   Captured as a list.

Synthesize the three answers into **1 to 3 short paragraphs** in
the unified intake voice. Run `stop-slop`. Present via
`AskUserQuestion` with confirm / amend / reject.

When the client rejects, regenerate one revision with the
client's specified change. Cap at three revision rounds; on the
third, capture the client's last-stated language verbatim
(no stop-slop unless the client opts in).

Append the polished paragraphs to `intake/background.md` under
`## Company description` with a `<!-- source: client interview,
<ISO date> (stop-slop polished) -->` provenance comment. Mark
`company_description` in `stages_complete`. Save.

## Stage 3 — Content library

Loop capturing what the client wants in the knowledge base. Each
captured entry has three fields:

| Field            | Prompt                                                                                                |
|------------------|-------------------------------------------------------------------------------------------------------|
| **Title or URL** | Either a URL the client wants `kb-ingest` to fetch, or a paste / file the client provides directly.   |
| **Scope**        | first-party (the client created or has rights to speak for it) or third-party (reference only).       |
| **One-line description** | What it is and why it matters for the bots.                                                   |

Routing on capture:

- **URL**: append to `samples/content-library.md` under the
  matching `## First-party content` or `## Third-party content`
  section. `kb-ingest` reads this file on its next run and fetches
  the URLs.
- **Paste**: write to `01_knowledge_base/raw/[slug].md` (first-
  party) or `01_knowledge_base/raw/third-party/[slug].md` (third-
  party). Reference it from `samples/content-library.md`.
- **Dropped file**: route to `01_knowledge_base/raw/` or `raw/third-
  party/` per scope. Reference it from `samples/content-library.md`.

After each entry, ask: "Add another? (yes / no / done)". On no /
done, exit the loop. Encourage at least 3 first-party entries
during the intake — that's the floor that makes
`generate-style-guide`'s output meaningful. Surface a soft warning
when the loop exits with fewer than 3 first-party entries:

> "Style guide quality drops sharply with fewer than 3 first-party
> samples. Add more before continuing?"

Accept the client's answer either way.

Mark `content_library` in `stages_complete`. Save.

## Stage 4 — Voice samples

Loop capturing 2 to 3 (or more) voice samples. Each captured
sample has three fields:

| Field          | Prompt                                                                                            |
|----------------|---------------------------------------------------------------------------------------------------|
| **Title**      | Short label naming the sample (e.g., "Recent LinkedIn post on AI search", "March newsletter").    |
| **Source kind**| Free text, but encourage the client toward: LinkedIn post / newsletter / sales-call transcript / |
|                | blog post / podcast intro / website hero / one-on-one DM thread / keynote excerpt.                |
| **Content**    | Paste-in directly, or a file drop, or a URL the skill fetches and extracts when web tools are    |
|                | available.                                                                                        |

Each sample gets appended to `00_intake/samples/voice-notes.md`
under its own `## [Title]` H2 with a `<!-- source: <kind>,
captured <ISO> -->` provenance comment.

After each sample, ask: "Add another? (yes / no / done)". Encourage
at least 3 samples during the intake. The hard floor is 2 — below
that, surface:

> "Style guide quality is thin with fewer than 2 voice samples.
> Add more before continuing?"

Accept the client's answer either way.

After the samples loop exits, ask one optional turn:

- **Brand / style notes.** "Anything explicit about how the
  bots should write or shouldn't write? Words to never use,
  phrases to avoid, mandated terminology, tone constraints,
  inside jokes?" Free text. Skip is OK.

When the client supplies notes, append them to
`samples/voice-notes.md` under a `## Brand and style notes` H2.

Append an audit summary to `intake/background.md` under `## Voice
samples` — number of samples captured, whether style notes were
captured, the path to the samples file. Mark `voice_samples` in
`stages_complete`. Save.

## Stage 5 — Goals / constraints

Optional in v0.9.x. The plan's "out of scope" decision applies —
the dedicated Goals/Constraints sub-skill (workbook Module 1
equivalent) is deferred to a future intake release.

One `AskUserQuestion`:

> "Anything specific you want the bots to keep in mind — goals,
> constraints, timelines, taboo topics, must-mention partnerships
> that don't fit elsewhere? Skip is fine; we can capture this later."

Three options: capture / skip / capture later.

- **Capture** — free text. Write to `intake/background.md` under
  `## Goals / constraints`.
- **Skip** — write `_not provided._` under `## Goals /
  constraints`.
- **Capture later** — same as Skip; surfaces in the quality report
  that this is on the deferred list.

Mark `goals` in `stages_complete`. Save.

## Artifact shape: `intake/background.md`

Front-matter:

~~~yaml
# ---
intake_artifact: "background"
sub_skill: "intake-background"
sub_skill_version: "0.1.0"
status: "in_progress"
captured_at: "<ISO 8601 of last write>"
stages_complete: []                    # subset of: stack, company_description, content_library, voice_samples, goals
content_library_first_party_count: 0
content_library_third_party_count: 0
voice_samples_count: 0
brand_style_notes_captured: false
goals_captured: false
# ---
~~~

Body shape:

~~~markdown
# Background intake

## Tech stack

<!-- source: 00_intake/stack.md (captured via capture-stack v<version>) -->

See `00_intake/stack.md`. Captured: <ISO date>.

## Company description

<!-- source: client interview, <ISO date> (stop-slop polished) -->

<paragraph 1>

<paragraph 2 — when applicable>

<paragraph 3 — when applicable>

## Content library

<!-- source: client interview, <ISO date> -->

- 5 first-party entries.
- 2 third-party entries.
- Written to `00_intake/samples/content-library.md`.

## Voice samples

<!-- source: client interview, <ISO date> -->

- 3 samples captured.
- Brand / style notes: captured.
- Written to `00_intake/samples/voice-notes.md`.

## Goals / constraints

<!-- source: client interview, <ISO date> OR not provided -->

_not provided._
~~~

Three companion files are written outside `intake/background.md`
and own the actual content:

- `00_intake/stack.md` — owned by `capture-stack`. Background's
  Stage 1 only delegates.
- `00_intake/samples/content-library.md` — owned by this skill.
- `00_intake/samples/voice-notes.md` — owned by this skill.

`compile-profile` reads only `intake/background.md`. Downstream
content skills (`kb-ingest`, `generate-style-guide`) read the
samples files directly.

## File shapes — companion files

### `00_intake/samples/content-library.md`

~~~markdown
# Content library

Captured during intake-background, <ISO date>. `kb-ingest` reads
this file and fetches the URLs on its next pass.

## First-party content (owned)

- [Title](https://...) — One-line description.
- [Title](https://...) — One-line description.

## Third-party content (reference only)

- [Title](https://...) — One-line description.

## Notes

<free text the client supplied during capture, if any>
~~~

When the file already exists from a prior `ingest-workbook` run,
`intake-background` appends new entries under the existing
sections rather than overwriting. The kb-scope flag (first-party
vs. third-party) is preserved on every entry.

### `00_intake/samples/voice-notes.md`

~~~markdown
# Voice samples and notes

Captured during intake-background, <ISO date>.
`generate-style-guide` reads this file directly.

## <Sample 1 title>

<!-- source: <kind>, captured <ISO> -->

<paste content, or `<see 01_knowledge_base/raw/[slug].md>` when the
sample landed as a file drop>

## <Sample 2 title>

<!-- source: <kind>, captured <ISO> -->

<paste content>

## Brand and style notes

<free text from the optional Stage 4 turn — words to avoid,
mandated terminology, tone constraints, etc. Omit this section
when the client skipped the turn.>
~~~

## Steps

1. **Read preconditions.** Confirm `_progress.md` exists. Confirm
   `intake-interviewer-voice.md` is on disk.

2. **Initialize or resume `intake/background.md`.** Parse front-
   matter when present. Compute the resume point from
   `stages_complete`:

   - `[]` → Stage 1.
   - `[stack]` → Stage 2 question 1.
   - `[stack, company_description]` → Stage 3 loop entry.
   - `[stack, company_description, content_library]` → Stage 4
     loop entry.
   - `[stack, company_description, content_library, voice_samples]`
     → Stage 5.
   - All five → ask redo / refresh / exit.

3. **Run Stage 1.** Check `stack.md` state. Delegate to
   `capture-stack` per the three flows above. Append the audit
   note. Save.

4. **Run Stage 2.** Three turns + synthesis + stop-slop + confirm
   loop (cap revisions at 3). Save after each turn and after the
   final paragraph commits.

5. **Run Stage 3.** Content library loop. For each entry,
   capture title/URL + scope + description; route per the
   routing table; append to `samples/content-library.md`; save
   `intake/background.md` audit count after each entry. On loop
   exit, finalize the audit summary.

6. **Run Stage 4.** Voice samples loop. For each sample, capture
   title + source kind + content; append to
   `samples/voice-notes.md`; save audit count. Optional brand /
   style notes turn at loop end. Finalize the audit summary.

7. **Run Stage 5.** Goals / constraints turn. Capture or write
   `_not provided._`. Save.

8. **Mark complete.** Set `status: complete`. Save.

9. **Return.** Print the quality report:

   - Stack version captured (and whether it was a refresh).
   - Number of paragraphs in the Company description.
   - First-party + third-party content library counts.
   - Voice samples count + brand/style notes captured (yes/no).
   - Goals captured (yes/no).
   - Soft warnings — fewer than 3 first-party content entries or
     fewer than 2 voice samples surface here.
   - Next-step prompt: "Run `intake-icp` next — `run-intake` will
     dispatch automatically when running through the orchestrator."

## Drop handling mid-flow

Per the shared voice reference. Routing for this skill is broader
than other intake sub-skills because drops are most likely here:

- **A voice sample** dropped during Stage 2 / 3 / 4: append to
  `samples/voice-notes.md` under its own H2. Return to the
  question in flight. The Stage 4 sample count auto-increments
  even when the drop landed during an earlier stage.
- **A first-party content URL or file**: drop in
  `01_knowledge_base/raw/` and append to `samples/content-library.md`
  first-party section. Return to the question.
- **A third-party content URL or file**: drop in
  `01_knowledge_base/raw/third-party/` and append to
  `samples/content-library.md` third-party section. Return.
- **A website "About" page URL during Stage 2**: fetch when web
  tools are available; use as pre-draft input for the Company
  description synthesis. Return to the question.
- **A style or brand guide PDF / .docx**: drop in
  `01_knowledge_base/raw/`. Use as pre-draft input for the
  Stage 4 brand/style notes turn.

Do not advance the question cursor when handling a drop.

## Stop-slop integration

`stop-slop` runs on:

- The synthesized **Company description** paragraphs in Stage 2,
  before presenting for confirm.
- Any client-revised paragraphs (each revision round).

Bullet structures (Stage 3 content library, Stage 4 voice
samples) are not polished — short, structured, client-direct.

Voice samples themselves are NEVER passed through stop-slop —
they're the client's actual voice that everything downstream
calibrates against. Polishing them would invalidate the signal.

## Pre-draft sources

Background runs first, so upstream artifacts are minimal.
Available sources:

1. `client.toml` — company name from scaffolding.
2. Any seed files already in `01_knowledge_base/raw/` — clients
   often drop a few files between `scaffold-client` and
   `run-onboarding`. The skill scans on entry.
3. Mid-flow drops — files and URLs the client paste during the
   skill itself.

Pre-drafts here are sparser than later sub-skills. The Company
description synthesis is the main place pre-drafting helps —
when an "about" page or company brochure was dropped in
advance, the synthesis can pre-draft the three answers.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| `capture-stack` returns an error                 | Surface the error verbatim. Mark Stage 1 incomplete. Resume restarts Stage 1.  |
| Stack already current, client picks "skip"       | Audit note records the skip and the captured version. Move to Stage 2.         |
| Stage 2 synthesis hits the revision cap          | Capture the client's last-stated language verbatim, no stop-slop. Save.        |
| Content library loop exits with zero first-party | Allow it. Quality report flags it as a `style-guide` quality risk. Don't       |
|   entries                                        | block.                                                                          |
| Voice samples loop exits with zero or one        | Allow it. Quality report flags it. `generate-style-guide` runs in low-evidence |
|   sample                                         | mode but still produces a style guide.                                          |
| File-drop routing target directory missing       | Create the directory (`01_knowledge_base/raw/third-party/` is the common case). |
| Client says "stop" mid-stage                     | Save current state, leave `status: in_progress`. Resume picks up at the next   |
|                                                  | unanswered question.                                                            |
| Existing `samples/content-library.md` from prior | Append under the existing sections; do not overwrite. Preserve scope tags.     |
|   `ingest-workbook` run                          |                                                                                |
| Existing `samples/voice-notes.md` from a prior   | Append new samples under new H2s; do not overwrite the file. Voice samples are |
|   run                                            | additive.                                                                       |

## What NOT to do

- Do not batch questions. One `AskUserQuestion` per field.
- Do not run `stop-slop` on voice samples themselves. The
  client's voice is what we're calibrating against; polishing
  it defeats the purpose.
- Do not skip Stage 1's stack delegation. Even when the client
  says "I'm just doing the messaging stuff," the stack capture
  is required for downstream bot variant enablement.
- Do not fabricate Company description content. The synthesis
  is from the three captured answers + any seed files; if the
  client gives short answers, the paragraphs are short. Padding
  the description with marketing language hurts every
  downstream draft that quotes it.
- Do not partition content library by lane or channel. The
  first-party / third-party scope flag is the only partition
  `kb-ingest` cares about.
- Do not capture the Company URL twice. It lives in
  `client.toml` (set during `scaffold-client`); Stage 2 reads
  it from there and confirms with the client rather than
  recapturing.
- Do not write to `client-profile.md`. That is `compile-profile`'s
  job. This skill owns `intake/background.md`,
  `samples/content-library.md`, and `samples/voice-notes.md`.
- Do not auto-trigger `kb-ingest` on Stage 3 exit. `kb-ingest`
  has its own entry point and gates. The quality report mentions
  it as the next-step prompt; the user invokes it deliberately.
