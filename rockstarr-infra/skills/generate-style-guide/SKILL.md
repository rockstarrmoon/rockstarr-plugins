---
name: generate-style-guide
description: "This skill should be used when the user asks to \"generate the style guide\", \"create the voice guide\", \"build style-guide.md\", \"run the Rockstarr style guide prompt\", or \"interview me for the style guide\". It reads the client's profile plus any first-party voice samples and knowledge-base content, pre-drafts answers to the Rockstarr strategic interview, walks the user through confirming and filling gaps one question at a time, then produces the authoritative /rockstarr-ai/00_intake/style-guide.md in the Rockstarr Brand Voice and Style Architect structure."
---

# generate-style-guide

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as frontmatter
> separators. **When writing actual output files, emit real `---`, not
> `# ---`.**

Produce the client's authoritative Written and Verbal Style Guide.
Every downstream Rockstarr bot (Content, Social, Outreach, Reply,
Nurture) reads this file before producing a single word of output. If
this file is wrong, every output is wrong.

This skill is a hybrid of the Rockstarr Brand Voice and Style Architect
custom GPT and a file-based pre-read. The GPT is strictly interview-
driven; real-world clients arrive with a workbook, samples, and
supporting materials. The skill reads everything first, drafts its own
answers, and then walks the user through confirming and closing gaps
— rather than forcing the human to answer questions whose answers are
already sitting in the profile.

## When to run

- Right after `ingest-workbook` and `kb-ingest` have completed for a
  new client, before any bot runs.
- When the client supplies new voice samples or significantly shifts
  positioning.
- When a human review of an existing guide surfaces a systemic issue
  that calls for a rebuild.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists and has been at
  least lightly reviewed by a human.
- Voice samples, if any, live in `/rockstarr-ai/00_intake/samples/` as
  `.md`, `.txt`, or copy-pasted excerpts.
- `references/style-guide-prompt.md` inside this skill is the canonical
  Rockstarr Brand Voice and Style Architect prompt. Follow it exactly.

## First-party only — hard rule

This skill reads from **first-party sources only**:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/samples/**`
- Files in `/rockstarr-ai/01_knowledge_base/processed/` whose front-
  matter has `style_guide_eligible: true` AND `kb_scope: owned`.

It must **never** read from:

- `/rockstarr-ai/01_knowledge_base/raw/third-party/`
- `/rockstarr-ai/01_knowledge_base/processed/third-party/`
- Any file with `kb_scope: third_party` or
  `style_guide_eligible: false`.

Third-party content is preserved by `kb-ingest` for drafting reference
only. Using it as a voice signal would launder someone else's style
into the client's guide, which is the opposite of what a style guide
is for. If the only available inputs are third-party, stop and tell
the user to supply first-party samples.

## The three phases

The canonical Rockstarr flow is three phases, in strict order. This
skill follows them with one Rockstarr-specific twist in Phase 1 (the
pre-read). See `references/style-guide-prompt.md` for the full prompt.

### Phase 0 — Pre-read (Rockstarr addition)

Before asking a single question:

1. Load `references/style-guide-prompt.md` as the active instruction
   set for this run.
2. Read `client-profile.md` end-to-end.
3. Read every file in `00_intake/samples/`.
4. Read `01_knowledge_base/index.md` (if present) and pull in every
   first-party processed file (`kb_scope == "owned"` AND
   `style_guide_eligible == true`). Ignore everything else.
5. For each interview question in the reference's six areas
   (strategic context, positioning edge, audience sophistication,
   temperament, communication environment, non-negotiables), draft:
   - A proposed answer grounded in the content.
   - A confidence rating: HIGH (clearly stated), MEDIUM (implied but
     not explicit), or LOW (no evidence — gap).
   - A one-line evidence pointer, e.g., `client-profile.md §Positioning`
     or `samples/linkedin-2025-q4.md`.
6. Hold these drafts in memory as the working answer sheet for Phase 1.
   Do not write a file yet.

### Phase 1 — Strategic interview (with pre-drafts)

Run the full interview from `references/style-guide-prompt.md`. One
question at a time, using `AskUserQuestion`. Never batch. Never skip.

For each question:

1. State the pre-drafted answer and its confidence rating, plus the
   evidence pointer. Example:

   > **Q (Positioning edge):** In one sentence, what makes you different
   > that a competitor could not credibly claim?
   >
   > **My draft (confidence: MEDIUM)** — "You sell to B2B founders who
   > have scaled past first-generation marketing and need a system, not
   > another freelancer."
   > **Evidence:** `client-profile.md §Positioning` and
   > `samples/linkedin-pinned-post.md`.

2. Offer the user four response options via `AskUserQuestion`:
   - **Confirm** the draft as-is.
   - **Amend** the draft (free-text edit).
   - **Reject and answer fresh** (ask the user to provide their own
     answer).
   - **Skip for now** (only allowed on non-core questions; mark as a
     GAP in the working answer sheet).

3. For LOW-confidence questions, lead with the underlying question
   rather than the weak draft, and provide the helpful prompts /
   multiple-choice suggestions described in the reference. Never
   fabricate positioning.

4. Adapt the next question based on the last answer. If the user
   rejects the pre-draft for Positioning Edge, probe harder on
   differentiation before moving on.

### Phase 2 — Confirm positioning

Synthesize a concise paragraph (≤120 words) that captures:

- The brand's positioning.
- Its authority posture (challenger / category leader / specialist /
  insurgent / practitioner).
- Its audience sophistication level.

Present it to the user and ask for explicit confirmation:

> Before I write the guide, confirm this positioning summary reads
> correctly. I will not generate the guide until you say "approved" or
> give me edits.

Iterate until the user explicitly approves. Do not move to Phase 3
without approval.

### Phase 3 — Generate the guide

Produce `style-guide.md` using the fixed structure from the reference,
in this exact order:

1. Brand Context
2. Mission or Core Intent
3. Brand Approach
4. Brand Personality (3–5 traits, each with Do / Do Not behaviors)
5. Audience Definition
6. Tone Definition using contrast framing (5–7 "X, not Y" pairs)
7. Style Rules (specific, testable)
8. Channel Adaptation (LinkedIn, blog, email, newsletter, outreach,
   meetings)
9. Tone Examples (3–5 before/after pairs)
10. Consistency Principles (3–5)

Prune the banned language listed in the reference. Do not use
motivational, corporate-cliché, or consultant phrasing.

Write to `/rockstarr-ai/00_intake/style-guide.md` with YAML
front-matter:

```yaml
# ---
client_id: "<from client.toml>"
client_name: "<from client.toml>"
source_profile: "client-profile.md"
sample_count: <int>
kb_sources_used: <int>                 # first-party kb files cited
interview_gaps: ["<area>", ...]        # LOW-confidence areas the user skipped
positioning_summary: "<the confirmed Phase 2 paragraph>"
generated_at: "<ISO timestamp>"
generate_skill_version: "0.3.0"
prompt_version: "brand-voice-architect-v1"
# ---
```

If an existing `style-guide.md` is present, archive it to
`/rockstarr-ai/99_archive/style-guide_<ISO timestamp>.md` before
writing the new one.

Print a summary — sections produced, interview gaps, any pre-drafts
the user rejected — and end with:

> Please review and either approve, or mark specific sections to
> revise. Bots should not run until this guide is approved.

## Interview discipline

- One question at a time. Always.
- Adapt the next question to what the last answer revealed.
- When the user is unsure, offer prompts, examples, or multiple-choice
  suggestions. Do not leave them stuck.
- Never skip Phase 1, even if the content is rich. The interview is
  the moment the human commits to the positioning.

## What NOT to do

- Do not invent voice characteristics not supported by the profile,
  the samples, or the user's interview answers. If there are no
  samples and the profile is thin, say so in the guide body and flag
  it as LOW CONFIDENCE in the front-matter.
- Do not paraphrase or echo third-party content from the knowledge
  base. It is ineligible under the first-party rule.
- Do not write any draft content (blog, social, email) in this skill.
- Do not merge the positioning summary, the interview answers, and
  the guide into one mega-document. Three phases, three artifacts
  (two ephemeral, one on disk).
- Do not generate the guide before explicit Phase 2 confirmation.
