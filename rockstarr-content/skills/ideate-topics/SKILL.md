---
name: ideate-topics
description: "This skill should be used when the user asks to \"ideate content topics\", \"brainstorm content angles\", \"suggest topics for next month\", \"give me topic ideas\", or \"what should we write about this month\". Reads the client's profile, approved style guide, first-party knowledge base, publish log, AND the stack-cadence fields in stack.md, then proposes 8 to 12 ranked topic angles filtered to lanes the client actually publishes (cadence >= 1). Each angle carries working title, pillar, audience, evidence pointer, and a lane (researched blog, thought leadership, email newsletter, or LinkedIn newsletter). Output is written to 02_inputs/content-topics_YYYY-MM.md monthly for the user to pick from before drafting."
---

# ideate-topics

Propose a month's batch of content angles grounded in the client's
actual profile, voice, and first-party material — not generic
prompts. The user picks which angles to draft from the list this
skill produces. First step of the monthly planning pass.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- First business day of the month — monthly ideation pass.
- User asks for topic ideas across any of the four content lanes.
- Publish log is getting thin and the pipeline needs refilling.
- User has supplied new first-party KB material and wants to turn
  it into content angles.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved
  (front-matter should not flag LOW CONFIDENCE on positioning).
- `/rockstarr-ai/00_intake/stack.md` exists with the v0.2
  content-cadence fields populated:
  `blogs_per_month`, `thought_leadership_per_month`,
  `email_newsletters_per_month`, `linkedin_newsletters_per_month`,
  `records_videos`, `case_studies_per_quarter`.
- `/rockstarr-ai/01_knowledge_base/index.md` exists with at least
  one first-party processed file.

If the style guide is missing or unapproved, stop and tell the user
to run `rockstarr-infra:generate-style-guide` first. Topic ideation
without a locked voice produces generic drift.

If the cadence block is missing from `stack.md`, stop and tell the
user to run `rockstarr-infra:capture-stack` to fill the content-
cadence fields first. Without them this skill has no way to know
which lanes are even live.

## Inputs

Read in this order:

1. `/rockstarr-ai/00_intake/client-profile.md` — positioning,
   audience, offers.
2. `/rockstarr-ai/00_intake/style-guide.md` — voice, tone, pillars,
   banned language.
3. `/rockstarr-ai/00_intake/stack.md` — content-cadence fields.
   This is the gate: only propose lanes whose cadence is >= 1.
4. `/rockstarr-ai/01_knowledge_base/index.md` — summary of every KB
   file with `kb_scope` tags.
5. Every first-party processed KB file (`kb_scope: owned`) — pull
   concrete claims, frameworks, numbers, customer stories, and
   phrases the client has already used. These become evidence
   pointers for topics.
6. Third-party processed KB files (`kb_scope: third_party`) —
   **reference only**. They can inform what competitors or the
   category are saying, but never act as voice signal and never
   get paraphrased into a topic angle.
7. `/rockstarr-ai/05_published/_publish.log` — the last 90 days of
   shipped pieces. Use this to avoid repeating recent topics within
   90 days, and to spot pillars that are under-served.

## Stack-cadence filter — the non-negotiable rule

Every topic this skill emits must belong to a lane the client
actually publishes. For each lane:

- `blogs_per_month` == 0 → no researched-blog topics proposed.
- `thought_leadership_per_month` == 0 → no thought-leadership
  topics proposed.
- `email_newsletters_per_month` == 0 → no email-newsletter
  topics proposed.
- `linkedin_newsletters_per_month` == 0 → no LinkedIn-newsletter
  topics proposed (note: LinkedIn newsletters reuse an approved
  thought-leadership piece via `publish-linkedin-newsletter`, so
  TL cadence must also be >= 1).

Quota guidance: aim to propose roughly twice the month's cadence
per lane (e.g., `blogs_per_month: 2` → propose 4 researched blog
angles so the client has choice). Cap the total at 12 across all
lanes.

Case studies are NOT generated here. They run on a quarterly
reminder via `draft-case-study`, outside the monthly calendar.

## Pillar selection

Derive 3 to 5 content pillars from the profile and style guide
before listing topics. Pillars are the buckets topics fit into —
for a B2B founder-led coach, pillars might be "Positioning",
"Pipeline systems", "Founder-led selling", "The Growth Amplifier
in practice". Every topic you propose must sit under one pillar.
If you can only justify 2 pillars from the inputs, say so — do not
invent pillars to hit a number.

## Output

Write to
`/rockstarr-ai/02_inputs/content-topics_YYYY-MM.md` (ISO
year-month in the filename; if a file for the month already exists,
append `-2`, `-3`).

File structure:

```markdown
# ---
client_id: [from client.toml]
generated_at: [ISO timestamp]
generate_skill_version: "rockstarr-content/ideate-topics@0.2.0"
month: "YYYY-MM"
pillars: ["Pillar A", "Pillar B", ...]
lanes_enabled:
  blog: true   # based on blogs_per_month >= 1
  thought_leadership: true
  email_newsletter: true
  linkedin_newsletter: false
cadence_snapshot:
  blogs_per_month: 2
  thought_leadership_per_month: 2
  email_newsletters_per_month: 4
  linkedin_newsletters_per_month: 0
topic_count: [int]
recent_publish_window_days: 90
kb_sources_scanned: [int]
# ---

# Content topics — [Client name] — [Month Year]

## Pillars

- **Pillar A** — one-line description
- **Pillar B** — one-line description
- ...

## Cadence snapshot

This month's targets from stack.md:
- Researched blogs: N per month
- Thought-leadership pieces: N per month
- Email newsletters: N per month
- LinkedIn newsletters: N per month (gated on thought leadership)

Lanes with cadence 0 are suppressed — no topics proposed.

## Topic list

### 1. [Working title]

- **Pillar:** Pillar A
- **Lane:** blog | thought-leadership | email-newsletter | linkedin-newsletter
- **Audience:** [specific audience from the profile]
- **Angle:** [one sentence — the contrarian / specific POV, not a summary]
- **Enemy** (thought-leadership only): [one-sentence headline of what this angle argues AGAINST. See "Enemy field — thought leadership only" below.]
- **Evidence (first-party):** [KB file or client-profile.md section]
- **References (third-party, optional):** [KB file + link]
- **Why now:** [one line — topical, seasonal, gap in publish log, customer question]
- **Rough length:** [800-word TL | 1500-word researched blog | 700-word newsletter | etc.]
- **Handoff:** outline-blog | outline-thought-leadership | draft-newsletter | publish-linkedin-newsletter

### 2. ...
```

Aim for 8 to 12 topics total across enabled lanes. Rank them by
strength of first-party evidence, not by how viral they feel.
Angles grounded in a real claim the client has made are more
useful than clever frames with no evidence.

## Lane — how to decide

- **Researched blog (1200–2500 words, outline-first):** Single-focus
  informational angle, SEO-friendly, answers a question or teaches
  a system. Cites first-party evidence heavily; may cite
  third-party research in a References section. Routed through
  `outline-blog` (gate) → `draft-blog`.

- **Thought leadership (600–1200 words, outline-first as of v0.3):**
  Stance-led opinion piece. Pushes back on a view, names a
  pattern, reframes a problem. Weight from POV, not research.
  Each TL angle MUST carry an `enemy` field — see the next
  section. Routed through `outline-thought-leadership` (gate) →
  `draft-thought-leadership`. This is also the source for
  LinkedIn newsletter republishes when that lane is active.

- **Email newsletter (600–900 words):** Personal, conversational,
  stitches 2–3 short thoughts, CTA links to the month's blog and
  thought-leadership pieces on the website. Routed through
  `draft-newsletter`.

- **LinkedIn newsletter (reuse of TL):** Not a new angle — a
  thought-leadership piece already in the pipeline that will be
  republished as a LinkedIn article. Call this out when a TL
  angle is especially well-suited for republishing. The handoff
  is `publish-linkedin-newsletter` (DEFER in v0.2), running
  against an approved TL piece.

## Enemy field — thought leadership only

Per the canonical TL rubric at
`rockstarr-infra/skills/_shared/references/tl-rubric.md` §
"Multi-article series — enemy diversity":

> If the newsletter publishes multiple articles in a series, each
> piece must make a different argument with a different enemy.
> Four pieces that all defend the same product around the same
> villain read as one ad repeated four times.

For every thought-leadership angle proposed, capture an `enemy`
field — a one-sentence headline of what the piece argues
AGAINST. Examples:

- "Vague positioning that blocks referrals."
- "Owner-dependent AI use that scales the bottleneck."
- "Buying execution instead of building a system."
- "Confusing pipeline that runs on memory with pipeline that
  runs on infrastructure."

Each enemy must be specific enough to write a counter-piece
against. "Bad marketing" or "doing it wrong" are not enemies —
they're labels. If the writer can't name the specific villain
the piece argues against, the angle isn't ready and should be
re-worked or dropped.

### Enemy diversity check

After proposing the month's TL angles, run a diversity check on
their enemies. The test: can you imagine reading two of these
pieces back-to-back and feeling like the second one repeated
the first one's argument under a different title?

Concretely, for each pair of TL enemies in the month's slate,
compare:

- **Same target?** (Same villain, just rephrased.)
- **Same systemic root cause?** ("Founders not delegating" and
  "Owner-dependent ops" are the same root cause stated two
  ways — that's a rhyme.)
- **Same fix?** (If the implied solution is the same product or
  same methodology in both pieces, the enemies rhyme even if
  they sound different.)

If two TL angles' enemies fail any of those three sub-tests,
flag them as rhyming. Surface to the user via
`AskUserQuestion`:

> Two thought-leadership angles for [Month] have rhyming
> enemies. They'll read as the same argument restated:
>
> 1. "[Title 1]" — enemy: [enemy 1]
> 2. "[Title 2]" — enemy: [enemy 2]
>
> Which would you like to do?
>   - Drop one of these and re-propose
>   - Keep both, accept the redundancy
>   - Sharpen the enemy on one to differentiate
>   - Skip this check (override)

Do NOT silently swap or modify the angle. The user decides.

If only one TL angle is proposed for the month, skip the check.
The diversity test only applies when multiple TL pieces ship in
the same monthly cadence.

## Interview step

After writing the file:

1. Run the enemy diversity check on the TL angles (see "Enemy
   field" above). Surface any rhyming pairs to the user before
   asking for picks. Resolve those before proceeding.
2. Summarize the top picks per lane in chat: count per lane, top
   1–2 topic titles per lane.
3. Ask the user, via `AskUserQuestion`, which topics they want to
   commit to for the month to hit each lane's cadence. Use at
   most 4 options per question; if there are more than 4 topics
   worth presenting, ask across multiple rounds.
4. Do not proceed to outlining or drafting inside this skill.
   Hand off to `content-calendar` (to slot the picks across the
   month) as the next step.

## What NOT to do

- Do not propose a topic for a lane whose cadence is 0. Honor the
  stack as the single source of truth.
- Do not use third-party KB content as a topic angle or an evidence
  pointer. Third-party material may only appear in the optional
  references line and must remain clearly attributed.
- Do not propose topics that contradict the style guide's Do Not
  list or the banned-language list.
- Do not propose topics that repeat a headline from
  `_publish.log` in the last 90 days, unless the user explicitly
  asks for a follow-up or refresh.
- Do not invent customer stories, numbers, or case studies. If a
  proposed topic needs a stat or story the client has not already
  made public, flag it as a blocker in the evidence line.
- Do not emit case-study angles here — those are quarterly via
  `draft-case-study`.
- Do not draft the actual content here. Ideation is the whole job.
- Do not run calendar assignment here — that's `content-calendar`.
