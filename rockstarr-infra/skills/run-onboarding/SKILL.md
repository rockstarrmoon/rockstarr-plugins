---
name: run-onboarding
description: "This skill should be used when the user asks to \"onboard me\", \"start onboarding\", \"set me up\", \"I'm a new client and want to get started\", \"let's get started\", \"run onboarding\", or when a brand-new install of rockstarr-infra hits an empty workspace. Thin client-facing entry point that chains scaffold-client (idempotent) into run-intake (the intake orchestrator), and on return surfaces kb-ingest and generate-style-guide as next-step prompts. No auto-chain past intake — every downstream skill keeps its own entry point. Safe to re-run at any point; idempotent at every layer."
---

# run-onboarding

The single entry point new clients use to set up their Rockstarr AI
Growth Operating System workspace. Thin wrapper — `run-onboarding`
makes no decisions of its own beyond the chain order and the
post-intake next-step prompt. The work happens inside the skills it
calls.

Three calls in order:

1. **`scaffold-client`** — creates the `/rockstarr-ai/` folder layout
   and `client.toml`. Idempotent: re-running is safe.
2. **`run-intake`** — dispatches to the workbook path or the
   interview path depending on the client's choice; orchestrates
   the six intake sub-skills; ends with `compile-profile`.
3. **Post-intake next-step prompt** — surfaces `kb-ingest` and
   `generate-style-guide` as the natural next moves, with an
   optional "run them now" branch.

The post-intake prompt offers but does not silently chain. The two
remaining skills (`kb-ingest`, `generate-style-guide`) have their
own gates and clients sometimes want to pause between intake and
KB processing.

Read `rockstarr-infra/skills/_shared/references/intake-interviewer-voice.md`
before running. The few question turns `run-onboarding` asks
follow the same voice and discipline as every other intake-adjacent
skill.

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. When writing actual
> output files, emit real `---`, not `# ---`.

## When to run

- First-time setup for a brand-new client. The marketplace
  installer's welcome message routes here.
- Operator standalone: "onboard me," "start onboarding," "let's
  get started," "I'm a new client and want to get started."
- Re-run after a pause: `run-onboarding` reads state from
  downstream skills' artifacts and resumes at the right point.

## Preconditions

- The client has Cowork open and a workspace folder selected. If
  no folder is selected, surface the issue and surface the
  `request_cowork_directory` prompt (or its equivalent) so the
  client picks a folder before continuing.
- `rockstarr-infra` is installed (this is the plugin shipping
  `run-onboarding`, so it's installed by definition).

No other preconditions. The chain handles everything else.

## Chat narration discipline

Every chat message this skill emits — step-transition lines, the
post-intake next-step prompt, the deferred-prompt reminder, the
all-set quality report — follows
[`skills/_shared/references/client-facing-output-voice.md`].

Specifically for this orchestrator:

- **No internal skill names in client-facing labels.** "Process
  the knowledge base now" not "Process the knowledge base now
  (`kb-ingest`)". Operators reading the backscroll can ask for the
  skill name if they need it.
- **No "step [N] of 6 on the interview path" framing in transitions.**
  "You're 3 of 6 steps into the intake" is fine — short, plain
  English, no internal label ("the interview path") that distinguishes
  the workbook path from the interview path (the client doesn't
  care which path; they care which step).
- **Quality reports follow the voice-guide pattern.** One sentence
  about what's captured, one sentence about what's next, file
  paths in a `[details]` footer.

## The chain

### Step 1 — `scaffold-client`

Call unconditionally. The skill is idempotent: on a fresh
workspace it creates the folder layout, on a partial workspace it
fills in missing pieces, on a complete workspace it updates the
`rockstarr_infra_version` stamp in `client.toml` and exits.

Surface the skill's normal output to the client. `run-onboarding`
adds no commentary unless something failed.

### Step 2 — `run-intake`

Call unconditionally after `scaffold-client` returns clean.
`run-intake` owns its own state via `_progress.md`; the chain
through `run-onboarding` looks identical on every invocation,
whether it's first-run or the eleventh resume.

On the first invocation, `run-intake` asks the workbook-or-
interview path question and proceeds from there. On a resume,
`run-intake` reads `_progress.md` and asks the top-of-loop turn.

### Step 3 — Post-intake next-step prompt

On `run-intake` return, inspect `_progress.md` for
`status: complete`. Chat narration follows
`skills/_shared/references/client-facing-output-voice.md` —
short, plain-English, file paths in a footer if at all.

**When intake is complete:**

Surface a two-sentence summary:

> "Your business profile is captured and assembled. Open it and
> read it end-to-end; if anything reads off, you can redo that
> step any time."
>
> *(File path in a collapsed Details footer for the user who
> wants to navigate to it directly:
> `00_intake/client-profile.md`.)*

Then ask one `AskUserQuestion`:

- **Process the knowledge base now** — dispatches
  to `kb-ingest`, which reads `samples/content-library.md` plus
  anything in `01_knowledge_base/raw/` and produces the
  processed library. After `kb-ingest` returns clean, ask the
  follow-up about the style guide.
- **Build the style guide now** —
  warns that style-guide quality is meaningfully better when the
  knowledge base has been processed, then dispatches to
  `generate-style-guide` directly. On the client's confirm.
- **Stop here** — exits cleanly with the deferred-prompt reminder
  (below).

Option labels stay in plain English (not "Process the knowledge
base now (`kb-ingest`)" with the skill name in parens). The
follow-up route names the skill in backticks when needed for
operator-level clarity, but the client-facing label is plain.

**When intake is in progress** (the client paused mid-flow):

One short sentence per voice-guide rule 4:

> "You're [N] of 6 steps into the intake. Pick it back up any time
> by saying 'continue intake'."

Exit cleanly. Don't ask about `kb-ingest` or `generate-style-guide`
— those skills aren't useful until intake produces
`client-profile.md`.

**When intake is on the workbook path and complete:**

`ingest-workbook` already produced `client-profile.md`. The
`kb-ingest` and `generate-style-guide` next-step prompts apply
the same way as on the interview path. Use the same turn.

### Step 4 — Optional `kb-ingest` dispatch

When the client picks "Process the knowledge base now," dispatch
`kb-ingest`. Surface its normal output. On return, ask one
`AskUserQuestion`:

- **Build the style guide now (`generate-style-guide`)** —
  dispatches.
- **Stop here** — exits cleanly with the deferred-prompt reminder.

### Step 5 — Optional `generate-style-guide` dispatch

When the client picks "Build the style guide now" at any of the
turns above, dispatch `generate-style-guide`. Surface its
normal output (including the 30 to 45 minute interview budget
warning at entry). On return, run-onboarding exits with the
all-set quality report.

## Deferred-prompt reminder

Whenever `run-onboarding` exits with intake complete but the
client picked "Stop here" before running `kb-ingest` and / or
`generate-style-guide`, the exit message includes a clear
reminder:

> "Two more steps to unlock your drafting bots: process your
> knowledge base and build the style guide. Neither is on a timer
> — pick them up when you have focused time."

(The skill names `kb-ingest` and `generate-style-guide` stay out
of the client-facing message; the user can ask "what are the
next steps?" if they want more detail and the response will
expand. Operators reading the chat backscroll know the right
skill names without surfacing them to the client at exit.)

When `generate-style-guide` is the only one left, the reminder
narrows to that skill.

## Steps

1. **Confirm workspace.** Verify a Cowork workspace folder is
   selected. When none is, surface the request-folder prompt and
   exit until the client picks one.

2. **Call `scaffold-client`.** Idempotent. Pass through whatever
   the skill prints to the client. On any failure, surface the
   error verbatim and exit — don't continue to intake on a
   half-scaffolded workspace.

3. **Call `run-intake`.** Same pass-through behavior. On the first
   invocation, the path-fork turn lives inside `run-intake`. On
   resumes, the top-of-loop turn lives there. `run-onboarding`
   sees only the return.

4. **Inspect `_progress.md`.** Read `status`, `intake_path`,
   `compile_profile_at`. Three branches:

   - `status: complete` → Step 5.
   - `status: in_progress` → Step 8 (paused-exit).
   - Any malformation → surface the error and exit (re-running
     `run-intake` will repair the state file or restart cleanly).

5. **Surface the intake-complete summary.** One paragraph naming
   the path taken and the assembled profile path. Don't quote
   long sections of the profile — the client opens the file
   themselves.

6. **Ask the post-intake next-step turn.** Three options:
   `kb-ingest` now / `generate-style-guide` now (with warning) /
   stop here.

7. **Dispatch the chosen next step.** On return, ask the
   matching follow-up turn until the client picks stop or both
   skills have run.

8. **Quality report.** Print:

   - Workspace path (the selected Cowork folder).
   - `client_id` and `client_name` from `client.toml`.
   - Intake path + completion status.
   - `kb-ingest` status (skipped / ran in this session / already
     ran previously, inferred from
     `01_knowledge_base/processed/index.md` existing).
   - `generate-style-guide` status (skipped / ran in this session
     / already ran previously, inferred from
     `00_intake/style-guide.md` existing).
   - Next-step prompt:
     - All four foundational skills complete → "You're set up.
       Your strategist will enable your bot plugins in the
       marketplace; install them as they appear. Day-to-day:
       draft, approve, publish."
     - Otherwise → the deferred-prompt reminder.

## Re-run semantics

`run-onboarding` is safe to invoke at any point. Its idempotency
comes from the skills it calls:

- `scaffold-client` is idempotent by design.
- `run-intake` resumes from `_progress.md`; never re-asks
  already-answered questions.
- `kb-ingest` is idempotent — re-ingesting an unchanged raw set
  produces no changes; new files in `raw/` get processed.
- `generate-style-guide` is the only non-idempotent piece: it
  always runs its interview when invoked. The post-intake turn
  doesn't pre-select it as the default; the client picks
  deliberately.

On a fully-set-up workspace (all four skills have run), the
post-intake turn shifts. `_progress.md` is `status: complete`
and the next-step prompt acknowledges the all-set state, but
the post-intake turn still offers:

- **Refresh kb-ingest** (when new raw files are present).
- **Regenerate the style guide** (with a warning that this is a
  ~30-45 minute exercise the client should only redo when
  positioning has shifted).
- **Exit** (the default — there's nothing to do).

## Drop handling

Drops landed during `run-onboarding`'s own turns are rare —
its turns are dispatch decisions, not capture. When a drop
does land:

- **A workbook .docx**: hold in memory, surface to `run-intake`
  when it asks the path-fork question. Pre-select workbook on
  that turn.
- **Voice samples, content URLs, KB material**: tell the client
  these belong inside the relevant sub-skill. Offer to dispatch
  to `run-intake` so the drop can be routed to
  `intake-background` for proper handling.
- **A style or brand guide**: same. Route via `intake-background`.

Don't silently absorb drops at the orchestrator level — that
bypasses each sub-skill's routing logic.

## Honor pause

The client may say "stop" or "pause" at any turn `run-onboarding`
asks. Save what the inner skills have written (they handle their
own saves; `run-onboarding` doesn't own state files), print the
deferred-prompt reminder, exit cleanly.

When the client pauses mid-`run-intake`, `run-intake` already
returned with `status: in_progress`. `run-onboarding` honors
that — it doesn't override with `kb-ingest` or `generate-style-guide`
prompts.

## Failure modes

| Failure                                          | Behavior                                                                       |
|--------------------------------------------------|--------------------------------------------------------------------------------|
| No Cowork workspace selected                     | Surface the request-folder prompt. Exit until folder is selected.              |
| `scaffold-client` fails (filesystem error,       | Surface the error verbatim. Exit. Don't proceed to `run-intake` on a half-     |
|   permission denied)                             | scaffolded workspace.                                                          |
| `run-intake` returns with malformed state        | Surface the issue. Recommend re-running `run-intake` directly (its own         |
|                                                  | malformed-state handling has the right tools).                                 |
| `kb-ingest` fails mid-dispatch                   | Surface the error. Re-render the post-intake turn so the client can choose to  |
|                                                  | retry, skip to style guide, or stop.                                            |
| `generate-style-guide` fails or is paused        | Surface the state. The deferred-prompt reminder covers the recovery path.      |
| Client picks "Stop here" at every turn           | Always allowed. The deferred-prompt reminder fires cleanly.                    |

## What NOT to do

- Do not own state. `run-onboarding` writes no files. Inner
  skills (`scaffold-client`, `run-intake`, etc.) own their own
  state.
- Do not silently chain into `kb-ingest` or
  `generate-style-guide` after intake completes. Always ask.
  Clients sometimes want to pause between intake and KB
  processing to read the assembled `client-profile.md`.
- Do not skip `scaffold-client` even when the folder layout
  already exists. The skill is idempotent and updates the
  version stamp — every run-onboarding invocation should pass
  through it.
- Do not interview the client at the orchestrator level. Every
  capture happens inside an inner skill.
- Do not interpret `run-intake`'s pause as a failure. Paused
  intake is a normal exit state; the deferred-prompt reminder
  is the right next-step surface.
- Do not surface `kb-ingest` or `generate-style-guide` next-
  step prompts when intake is still in progress. Those skills
  depend on `client-profile.md` existing.
- Do not auto-install bot plugins. Bot enablement happens via
  the marketplace, gated by the client's strategist. The
  quality report mentions this at the all-set exit but
  `run-onboarding` does no plugin installation.
