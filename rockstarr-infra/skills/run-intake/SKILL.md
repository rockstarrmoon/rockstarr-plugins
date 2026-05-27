---
name: run-intake
description: "This skill should be used when the user asks to \"start intake\", \"run intake\", \"continue intake\", \"resume my onboarding\", or \"redo a step of intake\", or when run-onboarding dispatches to the intake phase. Stateful orchestrator. Owns 00_intake/intake/_progress.md, handles the first-run workbook-or-interview fork, dispatches to the six sub-skills (background, icp, transformations, competitors, platinum-message, offer) in order with redo/jump/switch-paths controls, then calls compile-profile. Honors pause/stop cleanly."
---

# run-intake

The intake orchestrator. Sits between `run-onboarding` (the
client-facing entry point) and the six intake sub-skills + the
`ingest-workbook` legacy path + `compile-profile`. Owns one durable
state file: `/rockstarr-ai/00_intake/intake/_progress.md`.

Two paths converge here:

- **Workbook path** — client has a completed Rockstarr AI Playbook
  (.docx). `run-intake` calls `ingest-workbook` once. Done. Legacy
  but preserved indefinitely.
- **Interview path** — client doesn't have a workbook (or wants to
  redo via interview). `run-intake` dispatches to the six intake
  sub-skills in order, then calls `compile-profile`. This is the
  rule going forward.

The path is chosen on first run via the only required path-fork
turn. Subsequent runs read `_progress.md` and resume.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. Discipline rules apply to the orchestrator's own
question turns too — even though most of `run-intake`'s questions
are dispatch decisions rather than evidence capture.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- From `run-onboarding` after `scaffold-client`.
- Standalone, when the operator says "start intake," "continue
  intake," "resume my onboarding," "redo a step of intake," "show
  me where I left off," or similar.
- Standalone, when the operator wants to switch paths
  (interview → workbook is the only meaningful switch direction —
  workbook is one-shot).

## Preconditions

- `client.toml` exists. `scaffold-client` must have run.
- `/rockstarr-ai/00_intake/intake/` exists. `scaffold-client` creates
  it; `run-intake` creates `_progress.md` on first invocation.
- All six intake sub-skills + `compile-profile` + `ingest-workbook`
  + `capture-stack` are installed in the same plugin (they are —
  same plugin, same install).
- `intake-interviewer-voice.md` is on disk in the plugin's shared
  references.
- `client-facing-output-voice.md` is on disk in the plugin's
  shared references. This skill's chat narration (step transitions,
  the top-of-loop turn preamble, the quality report) follows it.

When any precondition fails, surface the issue and exit cleanly.

## Chat narration discipline

The two shared voice references cover different surfaces:

- **`intake-interviewer-voice.md`** governs the AskUserQuestion
  turns themselves (one question at a time, pre-draft / confirm /
  amend / reject / skip pattern, the four discipline rules).
  Sub-skills follow this for their interviews.
- **`client-facing-output-voice.md`** governs everything else
  this orchestrator emits — top-of-loop framing, dispatch
  narration, return-handling acknowledgments, the step 9 quality
  report.

Specifically for this orchestrator:

- **No "Phase 1.2 — Baseline ICP from client-profile.md" framing
  in chat.** That kind of internal-phase labeling is structural
  narration the client doesn't need. Step-transition lines stay
  short and plain ("Got it. Next up: transformations — what your
  best clients walked away with.").
- **No artifact paths in chat by default.** The fact that
  `intake-icp` writes to `00_intake/intake/icp.md` is invisible
  to the client. Mention paths only in collapsed `[details]`
  footers, and only on session-exit summaries.
- **No internal step numbering in chat preambles** beyond "step
  [N] of 6" framing. The canon table uses 1–6 for the step canon
  (background through offer) and that's fine to reference in plain
  English; deeper numbering ("Phase A step 4") stays out.
- **Sub-skill names in chat only when the client genuinely needs
  to invoke them.** Most of the time the user invokes
  `run-intake` and lets the orchestrator dispatch — they never
  need to type `intake-competitors` directly. Skill names appear
  in operator-facing surfaces (AskUserQuestion option labels and
  the `[details]` footer), not in prose narration.

## `_progress.md` shape

`run-intake` owns this file end to end. No sub-skill writes to it.

Front-matter:

~~~yaml
# ---
state_file: "_progress"
state_skill: "run-intake"
state_skill_version: "0.1.0"
created_at: "<ISO 8601>"
updated_at: "<ISO 8601 of last write>"
intake_path: "unknown"     # unknown | interview | workbook
status: "in_progress"      # in_progress | complete
compile_profile_at: null   # null until compile-profile runs; then ISO 8601
steps:
  background:
    status: "not_started"  # not_started | in_progress | complete
    started_at: null
    completed_at: null
    sub_skill_version: null
  icp:
    status: "not_started"
    started_at: null
    completed_at: null
    sub_skill_version: null
  transformations:
    status: "not_started"
    started_at: null
    completed_at: null
    sub_skill_version: null
  competitors:
    status: "not_started"
    started_at: null
    completed_at: null
    sub_skill_version: null
  platinum_message:
    status: "not_started"
    started_at: null
    completed_at: null
    sub_skill_version: null
  offer:
    status: "not_started"
    started_at: null
    completed_at: null
    sub_skill_version: null
# ---
~~~

Body:

~~~markdown
# Intake progress

## Path: <interview | workbook | unknown>

## Status grid

| # | Step                | Status         | Last update         |
|---|---------------------|----------------|---------------------|
| 1 | Background          | ✓ complete     | [ISO]               |
| 2 | ICP                 | ✓ complete     | [ISO]               |
| 3 | Transformations     | • in progress  | [ISO]               |
| 4 | Competitors         | not started    | —                   |
| 5 | Platinum Message    | not started    | —                   |
| 6 | Offer               | not started    | —                   |

## Compile profile

Status: pending (or "complete at [ISO]").

## History

- [ISO] — run-intake initialized; intake_path: unknown
- [ISO] — path chosen: interview
- [ISO] — background marked complete (intake-background v0.1.0)
- [ISO] — icp marked complete (intake-icp v0.1.0)
- [ISO] — transformations started
~~~

The History section is append-only; the Status grid is regenerated
from front-matter on every write. Front-matter is the source of
truth; the body is human-readable rendering.

## The step canon

Six steps in this order. Step number is fixed and stable —
operators can refer to "step 3" in chat and run-intake knows it
means transformations.

| # | Step             | Sub-skill                  | Required upstream artifacts                                              | Feeds (in client-profile.md)              |
|---|------------------|----------------------------|--------------------------------------------------------------------------|-------------------------------------------|
| 1 | Background       | `intake-background`        | (none — runs first)                                                      | Company, Voice samples, Goals/constraints |
| 2 | ICP              | `intake-icp`               | (none mandatory; pre-drafts from background)                             | ICP, Positioning (Phase B)                |
| 3 | Transformations  | `intake-transformations`   | `intake/icp.md`                                                          | Proof / results                           |
| 4 | Competitors      | `intake-competitors`       | `intake/icp.md`, `intake/transformations.md`                             | Competitors, Positioning (5 subsections)  |
| 5 | Platinum Message | `intake-platinum-message`  | `intake/icp.md`, `intake/transformations.md`, `intake/competitors.md`    | Messaging                                 |
| 6 | Offer            | `intake-offer`             | All four upstream intake artifacts                                       | Offers / services                         |

Dependency order isn't strict for step 1 (can re-run anytime), but
steps 2–6 enforce upstream completion. `run-intake` honors the
sub-skills' own precondition checks — if the operator jumps to step
5 without step 4 complete, the sub-skill refuses cleanly and
`run-intake` surfaces that.

## The path fork (first run only)

When `_progress.md` doesn't exist OR `intake_path` is `unknown`,
ask one `AskUserQuestion`:

> "How do you want to capture your business context?"
>
> Options:
> - **I have a completed workbook** — upload the .docx.
> - **No workbook — walk me through it** (six-step guided
>   interview, roughly 1.5 to 3 hours depending on ICP count).
> - **Show me what each path involves before I pick.**

On **Show me**: surface a short summary of each path (what the
client experiences, how long it takes, what the output is), then
re-ask the path question.

On **workbook**: enter the workbook path (below).

On **interview**: set `intake_path: interview` in front-matter,
write the path-chosen audit line to History, save, and enter the
interview path (below).

## The workbook path

Single step. One sub-skill call.

1. Prompt the client to confirm or supply the workbook location.
   Default: `00_intake/client-workbook.docx`. Accept any path the
   client gives.
2. Call `ingest-workbook` against that path.
3. On return, mark all six steps `complete` in `_progress.md`
   front-matter with `sub_skill_version: "ingest-workbook v[X]"`
   and `completed_at: [ISO]`. Add a History line.
4. Set `compile_profile_at: [ISO]` (ingest-workbook produces
   `client-profile.md` directly — `compile-profile` doesn't run
   on this path).
5. Set `status: complete`. Save.
6. Print the quality report and return.

Subsequent invocations on a workbook-path workspace ask:

> "Workbook path complete. Re-ingest a new workbook, switch to
> the interview path (archives the workbook artifacts), or exit?"

Default action: exit.

## The interview path

Multi-step. Six sub-skills dispatched in order, with controls for
redo / jump / stop / switch-paths.

### Top-of-loop turn

On every invocation that lands in the interview path with
`status: in_progress`, render the Status grid and ask one
`AskUserQuestion`. Options depend on state:

- **Continue: <next incomplete step name>** — dispatches to the
  next step where `status != complete`.
- **Redo: <pick a completed step>** — sub-menu listing each
  completed step; dispatches to the picked sub-skill.
- **Jump to: <pick any step>** — sub-menu listing all six steps;
  dispatches to the picked sub-skill (which may refuse if
  upstream isn't ready).
- **Switch to workbook** — archives current intake artifacts,
  resets the orchestrator, runs the workbook path. See
  Switch-paths below.
- **Show progress in detail** — surfaces a per-step summary
  pulled from each completed artifact's front-matter, then
  re-asks the top-of-loop turn.
- **Stop for now** — saves state and exits cleanly.

When five of six steps are complete and the user picks "Continue,"
the next step is the sole incomplete one. When all six are
complete, the loop enters the **final compile gate** (below).

### Dispatch + return

When a sub-skill is dispatched:

1. Set the step's `status: in_progress` and `started_at: [ISO]`
   in `_progress.md` front-matter (when not already in_progress).
   Save.
2. Call the sub-skill.
3. On return, inspect the sub-skill's artifact front-matter for
   `status: complete`. When complete, set the step's
   `status: complete`, `completed_at: [ISO]`, and capture the
   sub-skill's `sub_skill_version` value.
4. When the sub-skill returned with `status: in_progress` (the
   client paused), keep the step `in_progress`. Save.
5. Add a History line either way.
6. Ask one follow-up `AskUserQuestion`:

   - **Continue to the next step** — return to top-of-loop with
     the next-step option pre-selected.
   - **Stop for now** — save and exit cleanly.

### Final compile gate

When all six steps' `status: complete`, ask one
`AskUserQuestion` before dispatching `compile-profile`:

- **Show the assembled summary first** — surface a one-paragraph
  summary of each captured artifact (read each sub-skill's
  artifact front-matter and lead bullets). Re-ask after.
- **Redo a step** — return to the redo sub-menu.
- **Compile now** — dispatch `compile-profile`.
- **Stop for now** — save and exit cleanly.

On **Compile now**:

1. Call `compile-profile`. It refuses if any sub-skill artifact
   is `in_progress`; `run-intake`'s state should prevent this,
   but the check is a belt-and-suspenders.
2. On compile-profile's clean return, set
   `compile_profile_at: [ISO]` in `_progress.md` front-matter,
   `status: complete`, add a History line, save.
3. Print the orchestrator's quality report (below) and return to
   `run-onboarding` or the user.

Subsequent invocations on a complete interview workspace ask:

> "Intake complete. Redo a step, refresh the compiled profile,
> add another ICP via redo, switch to workbook (archives
> interview artifacts), or exit?"

Default action: exit.

## Switch-paths

A capability, not a default. Two flows:

**Interview → workbook.** From a partial-or-complete interview
state. The client confirms they want to switch:

1. Archive every file in `00_intake/intake/` to
   `99_archive/intake/<ISO date>/`. This includes
   `_progress.md`, every `[artifact].md`, and any audit notes.
   The archive preserves the audit trail.
2. Archive any companion files written by `intake-background`
   that the workbook path will also write — specifically
   `samples/content-library.md` and `samples/voice-notes.md` —
   to the same `99_archive/intake/<ISO date>/` directory. The
   workbook path will rewrite them.
3. Create a fresh `_progress.md` with `intake_path: workbook` and
   all six steps `not_started`. Add a History line noting the
   switch and archive path.
4. Enter the workbook path's prompt-for-upload turn.

**Workbook → interview.** From a complete workbook state. Same
archive-and-reset shape:

1. Archive `client-profile.md` (from the workbook ingest) to
   `99_archive/client-profile_<ISO date>.md`.
2. Archive `stack.md`, `samples/content-library.md`,
   `samples/voice-notes.md` to `99_archive/intake/<ISO date>/`.
3. Create a fresh `_progress.md` with `intake_path: interview`
   and all six steps `not_started`.
4. Enter the interview path's top-of-loop turn.

The switch is irreversible without another switch, which is
fine — archives are the safety net.

## Steps

1. **Initialize.** Confirm `client.toml` exists. Confirm
   `00_intake/intake/` exists. When `_progress.md` doesn't exist,
   create it with the empty-state front-matter shape and
   `intake_path: unknown`. Add a History line.

2. **Parse state.** Read front-matter from `_progress.md`.
   Determine the current branch:

   - `intake_path: unknown` → path fork.
   - `intake_path: workbook` + `status: complete` → workbook
     re-run / switch / exit prompt.
   - `intake_path: interview` + `status: in_progress` → interview
     top-of-loop turn.
   - `intake_path: interview` + `status: complete` → interview
     post-compile prompt (redo / refresh / switch / exit).

3. **Run the path fork** when applicable.

4. **Run the workbook path** when chosen. Update state, save, return.

5. **Run the interview top-of-loop turn** when applicable.

6. **Dispatch the chosen sub-skill.** When the option is
   "Continue / Redo / Jump," dispatch the matching sub-skill from
   the canon table. Mark the step `in_progress`, save before
   dispatch.

7. **Return handling.** On sub-skill return, inspect its artifact
   front-matter. Update `_progress.md` per the dispatch + return
   protocol. Save. Ask the follow-up turn (continue / stop).

8. **Final compile gate** when all six steps complete.

9. **Quality report.** When `run-intake` finishes a session — at
   any clean exit, not only at full completion — print a short
   chat summary that follows
   [`skills/_shared/references/client-facing-output-voice.md`].
   The V0.x bullet-list of internal fields (`Path: interview`,
   `compile_profile_at value`, "the path to the assembled
   `client-profile.md`") read as state-dump to non-technical
   clients. Replace with two sentences:

   - **First sentence**: where the client is, in plain English.
     One of:
     - *In progress:* "You're [N] of 6 steps into your business
       intake."
     - *Just completed* (compile fired this session): "Your
       business profile is captured and assembled."
     - *Already complete on session entry*: "Your intake is
       complete — your business profile is ready to read."
   - **Second sentence**: what's next, in user verbs.
     - *In progress, paused*: "Pick it back up any time by saying
       'continue intake'."
     - *Just completed*: "Next: process your knowledge base, then
       build your style guide. I can walk you through both."
     - *Already complete*: "Want to redo a step, refresh the
       profile, or move on to the knowledge base?"

   File paths (the path to `client-profile.md`, the
   `_progress.md` location) go in a collapsed `[details]` footer
   if at all. The skill names `kb-ingest` and
   `generate-style-guide` stay out of the chat summary — they
   appear in the AskUserQuestion options surface (where they're
   labels on the operator side) but not in the prose. Same
   discipline as `run-onboarding`'s Step 3.

## Drop handling

Drops mid-orchestrator are rare — `run-intake` doesn't ask
content-capture questions itself. When a drop lands during the
dispatch turn:

- **A workbook .docx**: hold in memory; pre-select the workbook
  path option on the next path-fork turn (or surface the switch-
  to-workbook option when on the interview path).
- **A voice sample, content URL, or first-party content**: tell
  the client these belong inside the relevant sub-skill, not at
  the orchestrator level. Offer to dispatch to
  `intake-background` so the drop can be routed correctly. Don't
  silently absorb drops at the orchestrator layer — that
  bypasses each sub-skill's routing logic.

## Honor pause everywhere

At any `AskUserQuestion` turn in `run-intake`, the client may say
"stop," "pause," "come back later," or equivalent. The
orchestrator:

1. Saves `_progress.md` with the current state.
2. Adds a History line noting the pause and what the next step
   would have been on resume.
3. Returns cleanly with the quality report.

When the client pauses mid-sub-skill (the sub-skill itself
handles the pause), `run-intake` doesn't see the pause directly —
it sees the sub-skill return with `status: in_progress`. Mark the
step `in_progress` in `_progress.md` and ask the standard follow-
up turn (continue to next step / stop for now). If the client
picks stop, exit. If the client picks continue, dispatch the next
step — but in practice clients usually picked stop because they
just paused.

## Drop and route — special "redo" semantics

When the client picks "Redo: [step]", the sub-skill's own resume
logic kicks in. Every intake sub-skill handles its own
"already complete — redo from scratch?" prompt. `run-intake` just
dispatches; the sub-skill owns the user-facing redo question.

After a redo completes, `compile-profile` (if it has already run)
is stale — the `compile_profile_at` timestamp predates the redo.
Surface this in the quality report:

> "Step [N] was redone after compile-profile ran. The current
> client-profile.md is stale. Run `compile-profile` again, or
> the next time intake is run."

Don't auto-re-run compile-profile. The client decides.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| `client.toml` missing                            | Refuse to run. Tell the client to run `scaffold-client` first.                 |
| `00_intake/intake/` missing                      | Same — tell the client to run `scaffold-client` first.                         |
| `_progress.md` malformed (manual hand-edit broke | Surface the malformation. Offer two options: restore from the most recent      |
|   the YAML)                                      | History timestamp by rebuilding from sub-skill artifacts, or restart from      |
|                                                  | scratch (archives current state to 99_archive/intake/[ISO]/).                  |
| Sub-skill refuses to run due to missing upstream | Surface the sub-skill's error message verbatim. Re-render the status grid and  |
|                                                  | ask the top-of-loop turn again. The client likely needs to run the missing     |
|                                                  | upstream step first.                                                            |
| Compile-profile refuses to run (incomplete       | Show which steps are still incomplete. Re-render the status grid.              |
|   sub-skill artifact)                            |                                                                                |
| Switch-paths archive directory creation fails    | Surface the file-system error. Refuse to reset `_progress.md` — the archive    |
|                                                  | step is non-negotiable, no archives means no switch.                            |
| User picks "Jump to step 5" before steps 2-4 are | Dispatch anyway. intake-platinum-message will refuse with its own error.       |
|   complete                                       | `run-intake` surfaces the refusal and re-asks the top-of-loop turn.            |
| `_progress.md` History section grows past 100    | Truncate to the last 50 entries when writing. The front-matter is the durable  |
|   entries                                        | state; the History body is human-readable audit. Compaction is fine.           |

## What NOT to do

- Do not interview the client about intake content. `run-intake`
  asks dispatch decisions only. Every content capture happens
  inside a sub-skill.
- Do not write to any artifact other than `_progress.md`.
  Sub-skills own their files; `compile-profile` owns
  `client-profile.md`; `ingest-workbook` owns its own outputs.
- Do not auto-dispatch sub-skills without a dispatch turn. Even
  on resume, the client picks the next step. The orchestrator
  proposes a default ("Continue: transformations") but doesn't
  silently run it.
- Do not auto-trigger `compile-profile` when all six steps
  complete. The final compile gate is a deliberate confirm-
  pause; clients sometimes want to redo before compiling.
- Do not auto-chain to `kb-ingest` or `generate-style-guide`
  after compile-profile runs. Surface them as next-step prompts;
  the client invokes them deliberately (`run-onboarding` does the
  same).
- Do not let the switch-paths archive overwrite an existing
  archive directory. The `99_archive/intake/<ISO date>/` path
  uses second-precision; collisions are rare but possible —
  append a `_2` suffix when needed.
- Do not delete artifacts on switch. Archive everything, even the
  files the new path will overwrite. The audit trail is the
  reason switching is safe.
