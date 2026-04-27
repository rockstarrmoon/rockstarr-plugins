---
name: outline-thought-leadership
description: "This skill should be used when the user asks to \"outline a thought-leadership piece\", \"plan an opinion post before drafting\", \"structure my TL piece\", \"sharpen the argument before I write\", or names a thought-leadership-format topic. It is the first step of the two-step thought-leadership flow added in v0.3 and must complete before draft-thought-leadership runs. Forces five required fields — thesis, smart-competitor counter-argument, opening scene, quotable line, buried proprietary term — applied from the canonical TL rubric. Writes an outline file in 03_drafts/content/ with full front-matter; outline must be explicitly approved by the user before drafting begins. Researched / informational topics route to outline-blog instead."
---

# outline-thought-leadership

The outline gate for the **opinion / stance** lane. New in v0.3.

In v0.2 the thought-leadership lane was single-shot — go from
conviction to page fast. That works when the writer has a sharp
point of view fully loaded. It fails when the argument is fuzzy:
the bot produces a confident-sounding piece that has no real
thesis, the reviewer rejects it, multiple regeneration rounds
chase the symptoms instead of the cause.

This skill is the cause-level fix. It forces the writer (founder
or bot) to commit to five concrete artifacts BEFORE prose runs:
the thesis, the counter-argument a smart competitor could make,
the specific opening scene, the line the reader will repeat at
dinner, and the proprietary term that gets buried until the
second half. If any of these is fuzzy at outline time, no draft
will rescue it.

> **Template convention.** Fenced code blocks below show `# ---`
> where YAML front-matter delimiters belong, to keep Cowork's
> SKILL.md parser from misreading them. **When writing the actual
> output file, emit real `---`, not `# ---`.**

## When to run

- User picks a thought-leadership-format topic from the month's
  `02_inputs/content-topics_YYYY-MM.md`.
- User asks to outline an opinion post, a stance piece, or a TL
  piece on a fresh topic.
- A previous TL draft was rejected on rubric grounds (failed two or
  more of the rubric's tests) and the reviewer wants to re-outline
  before regenerating.

If the user brings a topic that's clearly research-heavy /
informational ("explain how X works", "step-by-step guide to Y"),
steer them to `outline-blog` and the researched-blog lane instead.
The TL outline gate is for opinion, not for explainers.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/00_intake/stack.md` exists with
  `thought_leadership_per_month >= 1`. If TL cadence is 0, this
  skill refuses and points the user back at `capture-stack`.
- `/rockstarr-ai/01_knowledge_base/index.md` exists.
- The shared TL rubric at
  `rockstarr-infra/skills/_shared/references/tl-rubric.md` exists.
  This file ships in **rockstarr-infra v0.8.1+**. If the file is
  missing, refuse with this message:

  > Cannot run: the shared TL rubric is missing. Update
  > rockstarr-infra to v0.8.1 or later — the rubric ships at
  > `skills/_shared/references/tl-rubric.md` and v0.8.0 doesn't
  > carry it. The upgrade is pure additive (one new file), so
  > there's nothing else to coordinate.
- Either a specific topic (working title + angle) from the user,
  OR a pointer to a TL line item in
  `02_inputs/content-topics_YYYY-MM.md`.

## Inputs

Read in this order:

1. The shared TL rubric — full. The five required fields trace
   directly to the rubric's three tests plus the structural
   rewrite checklist. Do not reinvent — the rubric IS the
   contract.
2. Style guide — full. Brand Personality, Tone Definition contrast
   pairs, Style Rules. The Channel Adaptation section if a
   thought-leadership-specific block exists.
3. Client profile — positioning, the recurring arguments or
   unpopular truths the client has made before, the audience,
   the offer the CTA will point to.
4. The source topic — preserve angle and enemy from the topic
   file. Do not soften.
5. First-party KB files (`kb_scope: owned`) relevant to the
   stance — founder calls, internal memos, past writing. Cite by
   filename in the outline.
6. Recent TL entries in `05_published/_publish.log` — to flag if
   this argument rhymes with a recent piece (rubric's enemy
   diversity test, applied retroactively).

## The interview

Walk the user through the five required fields one at a time via
`AskUserQuestion`. Order matters — each field builds on the
previous one.

### Field 1 — Thesis (one sentence)

Question: "State the argument in one sentence. What does this
piece say that the reader doesn't already believe?"

Acceptance: a single declarative sentence that takes a position.
Reject:

- Topic labels ("Why positioning matters") — these are subjects,
  not arguments.
- Slogans ("Marketing isn't broken, you are") — these are not
  arguments per the rubric's first test.
- Hedged claims ("Sometimes founders should consider...") — if
  the writer is willing to walk it back in the thesis, the
  argument isn't real.

If the user's first attempt fails, offer one redirect: "That reads
as a topic / slogan / hedge. Re-state it as 'I think X, and here's
why' — what's the X?" If the second attempt also fails, surface
the failure to the human and stop. The piece needs a real position
that the writer hasn't committed to yet — no outline will fix
that.

### Field 2 — Counter-argument (one sentence)

Question: "What's the smart competitor's counter-piece? State
their best one-sentence rebuttal."

Acceptance: a credible counter-argument from someone with
expertise — not a strawman.

Reject:

- "Some people might disagree" — vague, not a real counter.
- An obviously wrong rebuttal (rubric's "trivially wrong" check) —
  the counter has to be the strongest version of the opposing
  view. If the bot can't produce a credible counter, the original
  thesis isn't sharp enough to disagree with.
- A counter that defeats a different thesis than the one stated in
  Field 1.

If two attempts at a credible counter both fail, stop. Per the
rubric: "If the counter is itself slogan-shaped or trivially
wrong, the original argument isn't real either." Surface to the
human.

### Field 3 — Opening scene (one short paragraph)

Question: "Describe the specific person in the first 100 words.
Role, situation, time of day. Who is the reader seeing themselves
as in the opening?"

Acceptance: a concrete person doing a concrete thing at a concrete
time. Examples of strong openings:

- "It's Tuesday at 4pm. Eleanor, the founder of a 22-person
  fintech, is staring at a Notion doc her ops contractor sent
  her three weeks ago..."
- "The CRO of a Series-B SaaS company is in the back of an Uber
  on the way to the airport. He just got a Slack from his head
  of marketing..."

Reject:

- "Most founders..." / "Every owner..." / "Many leaders..." —
  vague plurals are exactly what the rubric's pattern-list bans.
- A scene with no time-of-day, no role specificity, no concrete
  action.

### Field 4 — Quotable line (one sentence)

Question: "What's the line you want the reader to repeat at
dinner? Pull it out word-for-word — it goes high in the article
and gets flagged in front-matter."

Acceptance: a sentence that crystallizes the take and stands on
its own when extracted.

Reject:

- A sentence that only makes sense in context — strip the
  surrounding prose and re-read it. If it's flat alone, it isn't
  the line.
- A sentence that re-states the thesis verbatim. The quotable
  line is the thesis-with-bite — sharper, more memorable, more
  surprising.
- Multiple lines. Pick one. Per the rubric: "Every piece should
  have one of these on purpose."

### Field 5 — Buried proprietary term (one phrase, or "none")

Question: "What proprietary term — product name, methodology
name, coined phrase — gets used in this piece? It does NOT
appear before the idea is established. List the term, or say
'none' if there isn't one."

Acceptance: either a specific proprietary term, or an explicit
"none." If the user lists a term, it must not appear in the body
before the second half (per the rubric's structural checklist).

If "none," skip the burial check at draft time.

## Optional fields

After the five required fields are answered and accepted, ask
about three optional fields. Skip if not relevant — these are
helpful but not gates.

- **H2 structure** — 1 to 3 short H2 headings (TL pieces typically
  run 2-3 H2s). If the user says "let the draft figure it out,"
  that's fine.
- **Word-count target** — 600 to 1200 words for this lane.
  Default 800 if not specified.
- **Enemy** — one-sentence headline of what this piece argues
  AGAINST. (See ideate-topics' enemy-diversity check — if the
  enemy was named at ideation time, it carries through here.)

## Output

Write to
`/rockstarr-ai/03_drafts/content/outline-tl_[slug].md` where
`[slug]` is the kebab-cased working title, max 60 chars. If the
file exists, append `-2`, `-3`. Never overwrite an existing
outline without archiving to `99_archive/`.

Required front-matter:

```yaml
# ---
channel: "thought-leadership"
stage: "outline"
title: "Working title"
slug: "kebab-cased-slug"
pillar: "Pillar name"
audience: "One-line audience"
thesis: "The single-sentence argument from Field 1"
counter_argument: "The smart competitor's rebuttal from Field 2"
opening_scene: "The specific opening paragraph from Field 3"
quotable_line: "The dinner-table sentence from Field 4"
proprietary_term_buried: "Term name, or null if none"
enemy: "What this argues against, if named at ideation time"
word_count_target: 800
produced_by: "rockstarr-content/outline-thought-leadership@0.3.0"
produced_at: "ISO timestamp"
style_guide_version: "from style-guide.md front-matter"
rubric_version: "matched from tl-rubric.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# [Working title]

## Thesis
[Single declarative sentence. Field 1.]

## Counter-argument
A smart competitor would say: [Field 2.]

## Opening scene
[The concrete-person paragraph from Field 3. This is the literal
opening of the piece — `draft-thought-leadership` uses it as the
hook, lightly polished.]

## Quotable line
> [Field 4.]

This is the line the reader takes home. It belongs in the first
third of the article, not buried at the end.

## Proprietary term — handling
- Term: [Field 5, or "none"]
- If a term: bury until [section name or paragraph N]. Idea is
  established before the term is named.

## H2 structure (optional)
1. [H2 heading 1] — [one-line summary of section]
2. [H2 heading 2] — [one-line summary]
3. [H2 heading 3] — [one-line summary, if used]

## Word-count target
[600 / 800 / 1000 / 1200 words]

## Evidence map
| Section | First-party KB source |
|---|---|
| Opening scene | [filename or "client knowledge"] |
| H2 1 | [filename] |
| H2 2 | [filename] |
| Quotable line | [drawn from filename, or original to this piece] |

If any section has no first-party source, flag it. The draft can
still proceed but the writer's own claim is the only backing.

## Open questions
[Anything the outline can't resolve without the client. Example:
"Need confirmation that the Eleanor scenario is composite, not a
single named client."]
```

## Approval gate

After writing the file:

1. Summarize the outline in chat: thesis (one line), counter-
   argument (one line), quotable line, proprietary-term burial
   plan.
2. Ask the user via `AskUserQuestion`:
   - Approve outline as-is
   - Amend (free-text direction)
   - Reject and restart
3. On approve, update front-matter `approval_status: "approved"`
   and add `approved_at` and `approved_by`. This is the signal
   `draft-thought-leadership` reads.
4. On amend, apply the edits and re-present.
5. On reject, archive to `99_archive/` with a timestamp and
   stop.

`draft-thought-leadership` refuses to run if
`approval_status != "approved"`.

## What NOT to do

- Do NOT generate prose paragraphs in the outline (other than the
  opening scene from Field 3, which IS literal opening prose
  the draft uses). Bullets and short structural notes only.
- Do NOT accept a fuzzy thesis or trivially-wrong counter-argument.
  Two attempts each, then surface to the human. The whole point of
  this gate is to catch fuzziness before drafting wastes compute.
- Do NOT invent proprietary terms. If the user says "none," respect
  it and skip the burial logic.
- Do NOT proceed to drafting from inside this skill. Approval and
  drafting are two separate events.
- Do NOT use this gate for researched-blog topics. Route those to
  `outline-blog` instead.
- Do NOT silently mutate the topic-file entry. If the outline
  process surfaces a sharper thesis than the topic file held, ask
  the user whether to update the topic file too.
