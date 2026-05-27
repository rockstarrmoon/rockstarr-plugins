---
name: compile-profile
description: "This skill should be used when the user asks to \"compile the profile\", \"assemble the client profile\", \"build client-profile.md from the intake artifacts\", or when run-intake reports every intake sub-skill is complete. Reads the six intake artifacts from 00_intake/intake/ (icp, transformations, competitors, platinum-message, offer, background) and assembles the canonical /rockstarr-ai/00_intake/client-profile.md in the single shape every downstream bot reads. Deterministic stitcher — no editing, no fabrication. Archives the prior profile before overwriting."
---

# compile-profile

Assemble the authoritative `client-profile.md` from the six artifacts
the intake sub-skills produce. `compile-profile` is the contract every
intake sub-skill writes toward. It is deterministic, never asks the
client a question, and never invents content. If a sub-skill artifact
is missing or marked `status: in_progress`, this skill refuses to run.

The intake path is the rule going forward. `ingest-workbook` remains as
the legacy path for clients who arrive with a completed Rockstarr AI
Playbook. The two paths converge at `client-profile.md` — downstream
bots cannot tell which path produced it (except by reading the
front-matter `source` field, which they generally don't).

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as
> frontmatter separators. **When writing actual output files, emit
> real `---`, not `# ---`.**

## When to run

- At the end of `run-intake`'s interview path, once every sub-skill
  has marked itself `complete` in `_progress.md`.
- After `run-intake`'s "redo step N" loop, when a regenerated
  sub-skill artifact needs to flow back into the profile.
- Manually, when an operator has hand-edited an intake artifact and
  wants the profile re-stitched.

## Preconditions

Per `rockstarr-infra/CLAUDE.md`'s "Defer expensive preconditions"
section, split checks into two tiers:

**Tier 1 — cheap existence checks (run silently, before the
anchor message):**

- `/rockstarr-ai/00_intake/intake/_progress.md` exists.
- All six intake artifact files exist at their expected paths
  (use `os.path.exists`, not `read` — this is a sub-second stat).
- `client.toml` exists at the workspace root.

If any of these fail, refuse FAST with a clear remediation
pointer naming what's missing. The user sees the refusal
immediately, not after a multi-second silence.

**Tier 2 — content-level validation (deferred to Step 1 of
Behavior):**

- Each intake artifact carries `status: complete` in
  front-matter. This requires reading + parsing the front-matter
  block of each file (still cheap individually, but together
  with the body reads in Step 1 it makes more sense to do as
  part of the actual work, not pre-flight).
- Cross-artifact consistency (the campaign_slug referenced in
  `competitors.md` matches the one in `icp.md`, etc.).

These run AFTER the anchor message has fired, as part of the
main assembly work.

## Anchor message

Per `client-facing-output-voice.md` rule 7, fire this as the
FIRST chat output after Tier 1 existence checks pass:

> Stitching your business profile together from the six intake
> steps…

If Tier 1 fails (a file is missing), the anchor doesn't fire —
the refusal is the only output. The user sees the refusal
immediately without bot-looking-dead silence.

## Input artifacts

| Artifact path                                             | Produced by                  | Feeds section(s)                                                  |
|-----------------------------------------------------------|------------------------------|-------------------------------------------------------------------|
| `00_intake/intake/background.md`                          | `intake-background`          | Company; Voice samples (references `samples/voice-notes.md`)      |
| `00_intake/intake/icp.md`                                 | `intake-icp` (Phase A + B)   | ICP (Phase A); Positioning (Phase B Perception Gap output)        |
| `00_intake/intake/transformations.md`                     | `intake-transformations`     | Proof / results                                                   |
| `00_intake/intake/competitors.md`                         | `intake-competitors`         | Competitors; Positioning (Differentiation summary subsection)     |
| `00_intake/intake/platinum-message.md`                    | `intake-platinum-message`    | Messaging                                                         |
| `00_intake/intake/offer.md`                               | `intake-offer`               | Offers / services                                                 |

Two destinations have multiple inputs (Positioning pulls from both
`icp.md` Phase B and `competitors.md`; Voice samples references the
`samples/voice-notes.md` file produced by `intake-background`). The
assembler handles the merge; the sub-skills do not coordinate.

## Output: canonical `client-profile.md` shape

The output always has the same nine sections in the same order, with
the same front-matter shape. Variation across clients is in content,
not in structure.

### Front-matter

~~~yaml
# ---
client_id: "<from client.toml>"
client_name: "<from client.toml>"
source: "intake-interview v0.1.0"
compiled_at: "<ISO 8601, now>"
compile_skill_version: "0.1.0"
intake_status: "complete"
icp_count: <integer N>
# ---
~~~

`source` discriminates between paths. `compile-profile` always writes
`"intake-interview v[version]"`. `ingest-workbook` writes its own
source string (and may include workbook-specific fields like
`source_workbook`, `getting_started_complete`, `modules_present` —
those belong only to the legacy path and are not added here).

`icp_count` is the number of ICPs `intake-icp` captured. Downstream
parsers (Content Bot's `ideate-topics`, Reply Bot's `qualify-lead`)
key off this field.

### Body section order — fixed

1. `# <Client Name> — Client profile` (H1)
2. `## Company`
3. `## Offers / services`
4. `## ICP (Ideal Client)`
5. `## Messaging`
6. `## Positioning`
7. `## Proof / results`
8. `## Competitors`
9. `## Voice samples`
10. `## Goals / constraints`

Every section is always present, always in this order. When a sub-
skill emitted "not provided" sentinels for every field in a section,
the section still appears with a single line: `_not provided._` That
sentinel tells downstream skills the client passed on the section;
they handle the empty case rather than crash on a missing heading.

### Provenance comments

Every section opens with a single HTML comment naming its
source artifact:

~~~markdown
## ICP (Ideal Client)

<!-- source: 00_intake/intake/icp.md -->

...
~~~

When a section is fed by two artifacts (Positioning), emit both:

~~~markdown
## Positioning

<!-- source: 00_intake/intake/icp.md (Phase B) -->
<!-- source: 00_intake/intake/competitors.md (Brand Edge + Differentiation Summary) -->

...
~~~

Do not preserve the sub-skill artifacts' inner provenance comments
(`<!-- source: client interview, ... -->`) in the compiled profile.
They live in the artifacts under `intake/` for audit; they don't
need to leak into the canonical profile.

## Section-by-section assembly rules

### Company

Reads `intake/background.md` → "Company description" subsection.
One to three short paragraphs. No bullets. The interviewer writes
this section in tight prose; `compile-profile` lifts it verbatim.

### Offers / services

Reads `intake/offer.md` body. Each offer is rendered as a `### <Offer
Name>` H3 with a fixed bullet structure underneath:

- **Who it's for**
- **Problem**
- **Outcome**
- **Process**
- **Experience**
- **Edge** (optional)
- **Why it works**
- **Category-of-one positioning**

When `intake-offer` produced more than one offer, emit each as its
own H3 in the order they were captured.

### ICP (Ideal Client)

Reads `intake/icp.md` Phase A. Always renders the lead paragraph
naming the count and split of ICPs ("N distinct ICPs across two
sides of the marketplace" or "one primary ICP," depending on what
intake-icp captured), then one `### <ICP name>` H3 per ICP with the
fixed bullet structure underneath:

- **Core situation**
- **Where they discover support**
- **Pain points**
- **Surface-level want**
- **What they really want**
- **Ultimate goal**
- **Decision drivers**
- **Objections**

Always use H3 per ICP, even when N=1. Downstream parsers expect a
predictable structure.

### Messaging

Reads `intake/platinum-message.md`. Always renders one `### <ICP
name> — Platinum message` H3 per ICP captured. Each H3's body is a
blockquote (the canonical Platinum Message) followed by an
**Outcome statements** sub-list with three statements. When N=1,
the same shape applies — single H3.

### Positioning

Reads `intake/icp.md` Phase B (Perception Gap output) and
`intake/competitors.md` (Brand Edge, Differentiation Summary,
Messaging Opportunities, Risks to Watch, Quick Wins). Structure:

- `### Brand Edge` — bullet list from `competitors.md`.
- `### Differentiation summary` — single paragraph blockquote from
  `competitors.md`.
- `### What clients think vs. what we do` — two-column markdown
  table from `icp.md` Phase B, with the closing positioning
  summary as a short paragraph below.
- `### Messaging opportunities` — bullet list from
  `competitors.md`.
- `### Risks to watch` — bullet list from `competitors.md`.
- `### Quick wins` — bullet list from `competitors.md`.

Every subsection is always present. Empty subsections render
`_not provided._` Lines in any subsection still carrying the
`Assumed:` prefix from `intake-competitors`' validation pass
ship through to the canonical profile flagged — that's a
deliberate downstream signal, not a leak.

### Proof / results

Reads `intake/transformations.md`. Structure:

- A short lead paragraph naming how many transformations were
  captured (e.g., "Four core transformations the firm consistently
  delivers for its ICPs.").
- `### Top transformations` — numbered list. Each transformation
  is one line: a specific outcome with quantification when
  available.
- `### Case studies` (only when transformations are backed by named
  case studies) — markdown table with columns `#`, `Title`, `URL`.
- `### Quantifiable proof points` (only when present) — bullet
  list.

When the transformations artifact contains only "not provided"
sentinels — common for new clients pre-track-record — render the
section as:

~~~markdown
## Proof / results

<!-- source: 00_intake/intake/transformations.md -->

_Sparse — capture during the first quarter of operation._
~~~

### Competitors

Reads `intake/competitors.md`. Structure:

- A 4-column markdown table with columns `Competitor & URL`, `Value
  proposition`, `The offering`, `Differentiators`. One row per
  competitor.
- `### Key takeaways from the competitive landscape` — bullet list.

When fewer than two competitors were captured, the table still
renders (one row) but a trailing italic note flags the gap:
`_Competitive grid is thin — revisit during the first quarterly
review._`

### Voice samples

Reads `intake/background.md` → "Voice samples" subsection. Does not
duplicate the samples themselves; renders a one-paragraph summary
naming where the samples live and how many were captured:

~~~markdown
## Voice samples

<!-- source: 00_intake/intake/background.md -->

Three voice samples are available at
`00_intake/samples/voice-notes.md`: a recent LinkedIn post, a
newsletter the client is proud of, and a sales-call transcript
snippet. `generate-style-guide` reads them on its next run.
~~~

### Goals / constraints

Reads `intake/background.md` → "Goals and constraints" subsection
if present. For v0.9.x interview clients, this section is typically
empty and renders:

~~~markdown
## Goals / constraints

<!-- source: 00_intake/intake/background.md -->

_not provided._
~~~

The Goals/constraints exercise is deferred to a future intake sub-
skill (workbook Module 1 equivalent). Current intake clients can
fill the section by hand or wait for the dedicated sub-skill to
ship.

## Steps

1. **Read `client.toml`.** Capture `client_id` and `client_name`.
   If `client.toml` is missing, write a clear error and exit.

2. **Read `_progress.md`.** Confirm every intake sub-skill is
   `complete`. If any is `not_started` or `in_progress`, write a
   message naming which step still needs work and exit cleanly.

3. **Read every intake artifact.** For each of the six artifacts
   listed in the Input artifacts table, read the file and parse its
   front-matter. Verify `status: complete`. If any artifact's front-
   matter shows `in_progress` or `not_started`, exit with the same
   message as step 2.

4. **Compute `icp_count`.** Count `### ` H3 headings under the
   ICP section of `intake/icp.md` (Phase A only). This number drives
   the per-ICP rendering in Messaging and the lead paragraph in
   ICP.

5. **Archive the prior profile, if any.** If
   `/rockstarr-ai/00_intake/client-profile.md` exists, copy it to
   `/rockstarr-ai/99_archive/client-profile_<ISO date>.md` before
   overwriting. Do not delete the archive copies; they are the
   audit trail.

6. **Assemble the new profile in memory.** Emit the front-matter
   block, then each of the nine sections in order, each with its
   provenance comment(s) and the body assembled per the rules
   above. Render `_not provided._` for entirely empty sections;
   render structured "not provided" notes inside subsections of
   partially empty sections.

7. **Write `/rockstarr-ai/00_intake/client-profile.md`.** Overwrite
   the existing file. Use real `---` for front-matter (not `# ---`
   — that substitution is only for SKILL.md template safety).

8. **Run `stop-slop` over the assembled profile.** The shared stop-
   slop pass runs once at the end, after the full profile is
   assembled, before writing. Applies to the prose sections
   (Company, Differentiation summary, the closing paragraph after
   the Perception Gap table); structural artifacts (tables, the
   front-matter, provenance comments, bullet-list mechanical
   structure) are exempt.

9. **Print a chat summary** per
   [`skills/_shared/references/client-facing-output-voice.md`].
   V0.x bullet-listed the internal fields (`icp_count` and
   `source` field, "which sections rendered `_not provided._`",
   "whether `samples/voice-notes.md` exists") — that read as
   audit log to non-technical clients. The new shape:

   - **First sentence**: confident past tense, what now exists in
     the user's terms.
     - Clean run, no gaps: "Your business profile is assembled
       and ready to read."
     - Partial — one or more sections empty or gappy: "Your
       business profile is assembled. A few sections are still
       waiting on input — you can fill them in any time by
       redoing those intake steps."
   - **Second sentence**: what comes next, in user verbs.
     "Next: process your knowledge base and build your style
     guide. I can walk you through both whenever you're ready."

   Gap detail (which sections came up empty or partial, ICP count,
   voice-samples presence) goes in a collapsed `[details]` footer
   for the user who wants to navigate to specifics:

   ~~~markdown
   **Details**

   - File: `00_intake/client-profile.md`
   - ICPs captured: [N]
   - Sections still waiting on input: <comma list, or "none">
   - Voice samples available: <yes / no, with a one-line note
     when no, since the absence affects style-guide quality>
   ~~~

   The skill names `run-intake`, `kb-ingest`, and
   `generate-style-guide` stay out of the chat summary's prose.
   The orchestrator (`run-onboarding` / `run-intake`) is the
   surface that asks the next-step AskUserQuestion using
   user-facing labels — this skill just hands back cleanly.

## Resume and idempotency

`compile-profile` is fully deterministic. Running it twice in a row
with no intervening artifact changes produces an identical output
(modulo `compiled_at` ISO timestamps). The archive step prevents
data loss across multiple runs.

`compile-profile` does NOT mutate any intake artifact. Edits flow in
the other direction: an operator (or `run-intake`'s redo path) edits
the artifact, then re-runs `compile-profile`.

## What NOT to do

- Do not interview the client. Every input is already on disk.
- Do not edit the prose in the artifacts. Lift sections as written.
  Voice polish is each sub-skill's job at write time.
- Do not preserve the artifacts' inner provenance comments. Rewrite
  them to point at the artifact path.
- Do not fabricate content for empty sections. Use the `_not
  provided._` sentinel.
- Do not skip the archive step. Even a one-character edit warrants
  an archive copy.
- Do not auto-chain into `kb-ingest` or `generate-style-guide`.
  Those skills have their own entry points and gates. Surface
  them as next steps in the quality report and stop.

## Failure modes

| Failure                                   | Behavior                                                         |
|-------------------------------------------|------------------------------------------------------------------|
| Missing `client.toml`                     | Error: scaffold-client must run first. Exit.                     |
| Missing `_progress.md`                    | Error: run-intake must run first. Exit.                          |
| One or more artifacts `in_progress`       | Print list of incomplete steps. Exit cleanly (no error tone).    |
| Artifact present but front-matter missing | Error: artifact is malformed; re-run that intake sub-skill. Exit. |
| Existing `client-profile.md` not writable | Error: surface the permission issue. Don't overwrite.            |
| `samples/voice-notes.md` missing          | Voice samples section renders the "samples missing" variant.     |

All errors are surfaced in plain English, naming the offending file
or step. Never silent failure.
