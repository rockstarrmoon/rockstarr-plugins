---
name: ingest-workbook
description: "This skill should be used when the user asks to \"ingest the workbook\", \"parse the client workbook\", \"run workbook ingestion\", or \"process the Rockstarr AI Playbook\". It reads the client's Rockstarr AI Playbook from /rockstarr-ai/00_intake/ and produces a structured client-profile.md, a draft stack.md, and a samples/content-library.md that downstream bots and intake skills read before doing anything else."
---

# ingest-workbook

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as frontmatter
> separators. **When writing actual output files, emit real `---`, not
> `# ---`.**

Convert the client's **Rockstarr AI Playbook** (a `.docx` file) into
structured intake artifacts that every Rockstarr bot and intake skill
reads.

The Playbook has one required section — **Getting Started** — which the
client is expected to have filled in before first ingest. Everything
else (Modules 1–11) is optional program content that may or may not be
populated. This skill produces meaningful output from Getting Started
alone, and folds in any filled-in Module content where it adds context.

## When to run

- First intake, after the client uploads their Playbook.
- Any time the client revises the Playbook. Re-running archives the
  prior artifacts (see step 8) and regenerates from the new source.

## Preconditions

- `scaffold-client` has already run, so `/rockstarr-ai/00_intake/` and
  `/rockstarr-ai/00_intake/samples/` exist.
- A Playbook `.docx` exists somewhere under `/rockstarr-ai/00_intake/`.
- The `docx` skill is available for parsing (built-in Cowork skill).

## Workbook structure (what to expect)

The Playbook is delivered as `<Client Name> Rockstarr AI Playbook.docx`.
It follows this fixed top-level structure:

```
The Rockstarr AI Playbook          — front matter (program outcomes, conditions)
Getting Started                    — REQUIRED before first ingest
  Action Item 1: Welcome Note + Background (tech stack + content library)
  Action Item 2: Ideal Client Profile Playbook
  Action Item 3: Top Results / Transformations
  Action Item 4: Competition Crusher
  Action Item 5: Platinum Message Playbook
  Action Item 6: Offer Builder Playbook
  Action Item 7: Book the intake call         (process step; no content to extract)
Module 1  — Build the Foundation             (optional)
Module 2  — Lead Generation Masterclass       (optional)
Module 3  — LinkedIn Authority Masterclass    (optional)
Module 4  — Build the Sales Engine            (optional)
Module 5  — Building Market Authority         (optional)
Module 6  — The Content Creation Masterclass  (optional)
Module 11 — The 90-Day Operating Plan         (optional)
```

Module numbering is non-contiguous by design — the Playbook ships with
a curated subset of the full program modules. Do not assume 7, 8, 9, 10
are present or meaningful if missing.

## Steps

### 1. Locate the Playbook

Glob `/rockstarr-ai/00_intake/*Rockstarr AI Playbook*.docx` (case-
insensitive). If no match, fall back to the first `.docx` at that
path. If zero `.docx` files, stop and tell the user the Playbook is
missing.

### 2. Parse the docx

Use the docx skill's reader to extract text with heading structure and
tables preserved. Do not rely on regex-only heuristics — the Playbook
mixes prose, tables, and comment annotations.

### 3. Verify Getting Started is complete enough to ingest

Locate the `Getting Started` heading. For each of Action Items 1–6,
check that the "Paste your results below:" region (or the equivalent
answer table rows) has **any** non-empty content.

- If Getting Started is missing entirely, or if Action Items 2, 3, 4,
  5, and 6 are all empty, stop with a clear error: the client has not
  completed the prerequisite before first ingest.
- If 1–3 of those six items are empty, proceed but flag them in the
  quality report for the intake call.
- Action Item 7 is a booking step — it has no content to extract.
  Treat its presence as informational only.

### 4. Extract Getting Started into the three output artifacts

| Getting Started source                 | Target artifact                  | Target section / use                                   |
|----------------------------------------|----------------------------------|--------------------------------------------------------|
| AI1 — tech Q&A (website, CRM, socials, brand/style, newsletter, blog, case studies, design) | `stack.md` (draft) | Seed rows for CRM, Website, Social, Email, Design; mark access `unknown` so `capture-stack` fills in |
| AI1 — "content library" links & notes  | `samples/content-library.md`     | Verbatim list of URLs + client's notes, so `kb-ingest` can later fetch first-party content |
| AI2 — Ideal Client Profile             | `client-profile.md`              | **ICP (Ideal Client)** section                         |
| AI3 — Top Results / Transformations    | `client-profile.md`              | **Proof / results** section                            |
| AI4 — Competitive grid                 | `client-profile.md`              | **Competitors** section                                |
| AI4 — Differentiation summary / Brand Edge / Messaging Opportunities | `client-profile.md` | **Positioning** section            |
| AI5 — Platinum Message + Outcome Statement | `client-profile.md`          | **Messaging** section                                  |
| AI6 — Offer description (who / problem / outcome / how it works / etc.) | `client-profile.md` | **Offers / services** section           |

### 5. Fold in optional Module content (only what is filled in)

For any Module whose answer regions are non-empty, fold the content
into the matching `client-profile.md` section. Tag each folded block
with an HTML comment naming the source, e.g.:

```markdown
<!-- source: Module 1 > Action Item 5: Success Conditions -->
```

| Module source (if populated)                          | Target section in client-profile.md |
|------------------------------------------------------|--------------------------------------|
| Module 1 — current-state metrics, 12-month goal       | Goals / constraints                  |
| Module 1 — 90-day and 12-month vision                 | Goals / constraints                  |
| Module 1 — Perception Gap, Niche Evaluator            | Positioning                          |
| Module 1 — Success Conditions                         | Goals / constraints (as qualification checklist) |
| Module 2 — audience plan, LinkedIn outreach messages  | Outreach appendix                    |
| Module 2 — lead reactivation messages                 | Outreach appendix                    |
| Module 3 — LinkedIn headline / About / Business page  | Voice samples                        |
| Module 4 — sales system, objection playbook           | Sales appendix                       |
| Module 5 — case studies, testimonials, PR plan        | Proof / results                      |
| Module 6 — brand style & tone answers                 | Voice samples (mark as input, not the finished style guide) |
| Module 6 — social posts, long-form, newsletter drafts | Voice samples                        |
| Module 11 — targets, initiatives                      | Goals / constraints                  |

Do not synthesize Module content into a fabricated style guide — that
is `generate-style-guide`'s job. Module 6's style-and-tone answers are
inputs for that skill, not a substitute for it.

### 6. Write `client-profile.md`

Produce `/rockstarr-ai/00_intake/client-profile.md` with this
front-matter and section order. Omit a section only if it has zero
content (Getting Started or Module); never write a section with only
a placeholder line.

```yaml
# ---
client_id: "<from client.toml>"
client_name: "<from client.toml>"
source_workbook: "<filename of the .docx that was ingested>"
ingested_at: "<ISO timestamp>"
ingest_skill_version: "0.2.0"
getting_started_complete: true | partial | minimal
modules_present: [1, 5, 11]   # list of module numbers with any filled content
# ---
```

Section order:

1. Company
2. Offers / services
3. ICP (Ideal Client)
4. Messaging
5. Positioning
6. Proof / results
7. Competitors
8. Voice samples            (only if Module 3 or 6 populated)
9. Goals / constraints      (only if Module 1 or 11 populated)
10. Outreach appendix       (only if Module 2 populated)
11. Sales appendix          (only if Module 4 populated)

For any Getting Started item that was empty, write
`*(not provided in Getting Started — capture during intake call)*`
as the body of the corresponding section. Never invent content.

### 7. Write the draft `stack.md` and `samples/content-library.md`

**`stack.md` (draft)** — write `/rockstarr-ai/00_intake/stack.md` with
this shape. Use `unknown` for access status on every row; `capture-
stack` is responsible for confirming access during intake. Mark the
file as a draft so `capture-stack` knows to treat it as pre-fill.

```markdown
# ---
client_id: "<from client.toml>"
captured_at: "<ISO>"
capture_skill_version: "0.2.0-draft-from-ingest"
source: "ingest-workbook v0.2.0 — Getting Started AI1"
status: "draft"
# ---

# <Client Name> — Stack (draft)

| Category         | Tool                    | Access  | Notes                                     |
|------------------|-------------------------|---------|-------------------------------------------|
| CRM              | <from AI1 CRM answer>   | unknown | <free-text from the workbook answer>      |
| Website platform | <from AI1 website>      | unknown | ...                                       |
| Social platforms | <from AI1 socials>      | unknown | ...                                       |
| Brand guide      | <yes/no + notes>        | n/a     | Input for generate-style-guide            |
| Style/tone guide | <yes/no + notes>        | n/a     | Input for generate-style-guide            |
| LinkedIn newsletter | <yes/no + personal/business> | n/a | ...                                   |
| Blog             | <yes/no + where>        | n/a     | ...                                       |
| Case studies     | <yes/no + location>     | n/a     | ...                                       |
| Design tech      | <from AI1 design answer>| n/a     | ...                                       |
```

If `stack.md` already exists and is not a draft, do **not** overwrite —
skip this step and flag in the quality report.

**`samples/content-library.md`** — write
`/rockstarr-ai/00_intake/samples/content-library.md` with the URLs and
notes the client pasted under AI1's content library prompt, preserving
their groupings where possible (e.g., series of articles stays as a
series). Format:

```markdown
# ---
client_id: "<from client.toml>"
source: "Rockstarr AI Playbook > Getting Started > Action Item 1 > content library"
captured_at: "<ISO>"
# ---

# Content library — first-party sources the client provided

<bullet list of titles + URLs; preserve any grouping like "Series X Part 1…7">

## Client notes

<verbatim notes the client wrote alongside the links, if any>
```

This file is for `kb-ingest` to consume later — it should not be
treated as finished knowledge-base content.

### 8. Archive prior artifacts before overwriting

Before writing any file in step 6 or 7, if a non-draft version already
exists, move it to
`/rockstarr-ai/99_archive/<filename>_<ISO timestamp>.md`. Never
silently overwrite a live intake artifact.

### 9. Print the quality report

End with a concise bulleted report:

- Sections populated in `client-profile.md` (Getting Started + Modules).
- Sections empty or sparse — flagged as intake-call punch list.
- Modules detected but empty (so Rachel knows they were offered and
  skipped vs. not present at all).
- Whether `stack.md` was written fresh, pre-filled alongside an
  existing draft, or skipped because a non-draft stack is in place.
- Whether `samples/content-library.md` contained any URLs.

## What NOT to do

- Do not invent content. Empty sections get the "not provided" sentinel
  or are omitted per step 6.
- Do not write `style-guide.md` — that is `generate-style-guide`'s job.
  Module 6's style-and-tone answers feed that skill; do not pretend
  they are the finished guide.
- Do not finalize `stack.md`. This skill only drafts. `capture-stack`
  is the source of truth for tooling and variants.
- Do not fetch or process the URLs in `samples/content-library.md` —
  that is `kb-ingest`'s job.
- Do not touch any folder outside `/rockstarr-ai/00_intake/` (and its
  `samples/` subfolder) and `/rockstarr-ai/99_archive/`.
- Do not include raw workbook tables verbatim in `client-profile.md`
  unless they are the competitive grid in AI4 (which is clearest as a
  table). Summarize other tables into prose.
- Do not treat Action Item 7 as missing content — it is a booking step
  with nothing to extract.
