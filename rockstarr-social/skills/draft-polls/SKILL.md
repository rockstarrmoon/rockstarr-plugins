---
name: draft-polls
description: "This skill should be used when the user asks to \"draft the polls\", \"write the LinkedIn polls\", \"build the polls for [client]\", \"draft 10 polls for [persona]\", \"build set [N] polls\", or otherwise wants to produce a batched set of LinkedIn polls. Originally shipped in rockstarr-content v0.6; moved to rockstarr-social v0.1 — polls are short-form social, not long-form content. Drafts a full set of LinkedIn polls (default 10, configurable) for a given client persona, applying the canonical structure (post copy + question + 4 options + hashtags), enforcing LinkedIn's hard character limits (question ≤140, options ≤30, 2-4 options), running an audience-altitude pre-check before any prose runs (the #1 failure mode for this lane), pattern-matching the style of the most recent approved set, and writing the whole set as a single batched file in 03_drafts/social/. Reads brand hashtag and persona context from the LinkedIn polls subsection of style-guide.md. Cadence is gated by polls_cadence enum in stack.md (monthly | quarterly | on-demand | off)."
---

# draft-polls

Write a batched set of LinkedIn polls for a client persona.
Polls are short, structured, and shipped 10 at a time — the
batch is the unit of approval, not the individual poll.

This is rockstarr-social's structured short-form lane.
`draft-social` covers free-form short-form. Polls have hard
structural constraints (question ≤140, options ≤30, 2-4 options)
that make them their own lane. The skill originally shipped in
rockstarr-content v0.6 and was moved here in rockstarr-social
v0.1 because polls are a social surface, not a content one.
Long-form lanes (blog, thought leadership, newsletter, LinkedIn
newsletter, case study) and the `repurpose` derivative lane stay
in rockstarr-content.

> **Template convention.** Fenced code blocks below show `# ---`
> where YAML front-matter delimiters belong, to keep Cowork's
> SKILL.md parser from misreading them. **When writing the
> actual output file, emit real `---`, not `# ---`.**

## When to run

- User asks to draft a poll set: "draft the polls for Chris",
  "build set 8 polls", "write 10 polls for [persona]".
- A poll-set due date fires from the calendar (when
  `polls_cadence` resolves to `monthly` or `quarterly` and a
  set is due).
- User invokes on-demand for a client whose `polls_cadence` is
  `on-demand`.
- The trigger phrases above; or a saved workflow referencing
  `draft-polls`.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists with a
  clear audience definition (the altitude check needs it).
- `/rockstarr-ai/00_intake/style-guide.md` exists and is
  approved. **Specifically: the style guide must carry a
  `Channel Adaptation → LinkedIn polls` subsection** that
  contains:
  - Brand hashtag(s) — e.g., `#transearch`. Required.
  - Persona / practice list (if multi-persona client) —
    each persona's name, slug, and audience focus.
  - Voice rhythm notes specific to polls (post copy
    pattern, "but" pivot, length expectations).
  - Hashtag mix pattern (e.g., 3-5 tags per poll, mix of
    industry + brand).

  If this subsection is missing, refuse with this message:

  > Cannot run: style-guide.md doesn't have a
  > `Channel Adaptation → LinkedIn polls` subsection. The
  > brand hashtag, persona list, and voice rhythm for polls
  > all live there. Add the subsection (or re-run
  > `rockstarr-infra:generate-style-guide` once it carries
  > the polls questions in a future infra version), then
  > come back.

- `/rockstarr-ai/00_intake/stack.md` exists with
  `polls_cadence` set to one of `monthly`, `quarterly`, or
  `on-demand`. If `polls_cadence` is `off` or missing, refuse:

  > Cannot run: polls lane is not enabled for this client
  > (`polls_cadence: off` in stack.md). Re-run
  > `rockstarr-infra:capture-stack` to enable the lane, or
  > set `polls_cadence` to `monthly`, `quarterly`, or
  > `on-demand` directly.

- `/rockstarr-ai/04_approved/social/` directory exists (or
  is created on first run) — it's where set-number
  determination reads from.

## Inputs

Read in this order:

1. **`client-profile.md`** — audience definition is the
   spine of the altitude check. Pull the audience's industry,
   role level, company size, decision-making concerns. This
   is what the polls' altitude is measured against.
2. **`style-guide.md`** — full document for voice baseline.
   The `Channel Adaptation → LinkedIn polls` subsection is
   the polls-specific rules: brand hashtag, persona list,
   voice rhythm, hashtag mix pattern.
3. **`stack.md`** — `polls_cadence` enum confirms the lane
   is enabled.
4. **The most recent 1-3 approved poll sets** in
   `04_approved/social/` matching the pattern
   `polls_set-<N>_<persona-slug>.md`. The newest set is the
   primary style reference; older sets are for topic
   de-duplication only.
5. **First-party KB files** (`kb_scope: owned`) — for
   topic source material. Polls live or die on whether the
   client has a real point of view on the topic; KB carries
   the points of view.
6. **`05_published/_publish.log`** — additional topic
   de-duplication (a poll topic already published in any set
   is off the table for this run).

## Phase 1: Resolve persona and set number

### 1A. Persona / practice resolution

Most clients are single-founder — one persona, no decision
needed. Multi-persona clients (e.g., TRANSEARCH with Chris,
John, etc.) require the user to specify which persona's set
this is.

If the user's invocation names a persona ("draft polls for
Chris"), use it. If not, ask via `AskUserQuestion`:

> Which persona is this set for?
>   - [Persona A name]
>   - [Persona B name]
>   - [Persona C name]
>   - This is a single-founder client (no persona
>     selection needed)

Resolve the persona's slug from the style guide's persona
list. The slug is what determines the filename pattern.

### 1B. Set number resolution

List filenames in `04_approved/social/` matching
`polls_set-<N>_<persona-slug>.md`. Take the maximum N, add 1.
If no files match (first set for this persona), N = 1.

Surface the chosen set number to the user before drafting.
"Drafting Set 8 for Chris" — gives the user a chance to
correct if something's off.

## Phase 2: Audience altitude pre-check

**This is the #1 failure mode for the lane.** Polls about
field-level topics for an executive search firm fail the same
way regardless of how well-crafted the prose is. Catch
altitude drift before drafting any prose.

### 2A. Generate candidate topics (12-15)

Brainstorm 12-15 candidate topics across 3-4 themes:

- **Talent / leadership** (succession, hiring, retention,
  team composition).
- **Growth / market** (expansion, deals, market shifts).
- **Culture / retention** (engagement, comp, work patterns).
- **Trends / tech** (industry-specific shifts, regulatory
  changes, technology adoption).

Pull source material for each candidate from the first-party
KB and the client profile. A candidate topic without
KB-traceable source material gets cut — polls that aren't
grounded in the client's actual perspective sound generic.

### 2B. Altitude check

For each candidate topic, ask: **"Would the audience defined
in client-profile.md actually care about this question?"**

- If the audience is C-suite / boards / HR partners (e.g.
  TRANSEARCH), topics live at the executive level —
  succession, board composition, exec comp, PE ownership,
  CEO focus, leadership retention. NOT field-level (jobsite
  supers, trades pipeline, Gen Z entry-level workers,
  prefab factory workers, field safety).
- If the audience is business owners / founders / PE,
  topics live at the deal-readiness level — valuation,
  sale timing, owner-operator transitions.
- If the audience is something else, re-read the
  client-profile.md audience section and mirror that
  altitude.

Mark each candidate `pass` or `fail` on altitude. Surface
the failure rate to the user.

### 2C. Altitude-failure response

If 3 or more candidates fail altitude (out of 12-15):

- The candidate set is drifting. Do NOT proceed to drafting.
- Surface the failures to the user with the audience
  definition cited verbatim from client-profile.md and the
  failed topics listed.
- Ask the user to confirm the audience interpretation OR
  re-pick candidates with the corrected altitude.
- Per the SOP: don't try to patch individual polls — the
  whole batch needs to be re-picked at the right altitude.

If 1-2 candidates fail, drop them silently and proceed with
the remaining 10-13 candidates. Note the drops in the chat
summary so the human reviewer sees the filter at work.

## Phase 3: Topic diversity check

Read titles from the most recent 2-3 approved sets in
`04_approved/social/`. Filter out any candidate that repeats
a topic from those sets — even if the angle differs.

Spread the final 10 picks across at least 3 of the 4 themes
(talent, growth, culture, trends). A set where 8 of 10 polls
are talent-themed reads as monothematic.

If the topic diversity check leaves fewer than 10 viable
candidates, surface to the user before drafting. Either
generate more candidates (Phase 2A) or accept a smaller set.

## Phase 4: Draft each poll

For each of the 10 picks, produce this structure (matching
the most recent approved set's style, per the style guide's
LinkedIn polls subsection):

```
## Poll N: [Topic Title]

**Post Copy:**

[1-2 short lines of setup. Often a "but" pivot — the first
line names the obvious, the second line names what's harder.
Contractions are fine. No filler openers ("In today's...",
"It's worth noting...").]

[A line that points the reader to vote.]

**Poll Question:** [Punchy question, ≤140 chars]

**Options:**

- [Option 1, ≤30 chars]
- [Option 2, ≤30 chars]
- [Option 3, ≤30 chars]
- [Option 4, ≤30 chars]

**Hashtags:** [3-5 tags per the style guide's hashtag mix,
ending with the brand hashtag]
```

### Drafting rules

1. **Style match the newest approved set.** If Set 7 was
   tighter and more provocative than Set 3, match Set 7.
   Newest approved style wins.
2. **First-party voice only.** Same rule as every other
   rockstarr-content lane. The KB carries the perspective;
   the prose draws from KB material, not from web research
   or competitor poll content.
3. **No AI tells.** Stop-slop pass at the end catches
   common ones. Specifically watch for in this lane:
   "delve," "navigate," "unlock," "tapestry," "the real
   question is," "in today's fast-paced world," "it's not
   just X, it's Y."
4. **The "but" pivot** is the canonical post-copy rhythm.
   Setup line names the obvious; the second line names
   what's actually hard. Example: "Closing the deal is the
   easy part. Keeping the executives who built the firm is
   harder."
5. **Question is short and provocative.** A question that
   reads as one-sided isn't a poll — it's an opinion. Keep
   the question genuinely contested.
6. **Options span the realistic answer space.** Not "yes /
   no / maybe / I don't know" — those are filler. Each
   option is a real answer a thoughtful respondent might
   actually pick.
7. **Hashtags follow the style guide's mix pattern.** End
   with the brand hashtag. Don't pad with generic
   `#leadership` if the style guide's pattern uses
   industry-specific tags.

## Phase 5: Character limit enforcement

LinkedIn's hard constraints. Counted at write time, not
eyeballed:

| Element | Limit |
|---|---|
| Poll question | 140 characters |
| Each option | 30 characters |
| Options count | 2-4 (default 4) |

For each poll, count and verify. Common 31-char mistakes
(catch and auto-fix):

- "Chief Innovation / Tech Officer" (31) → "Chief
  Innovation/Tech Officer" (29) — remove spaces around `/`.
- "Margin & operational discipline" (31) → "Margin & ops
  discipline" (23) — abbreviate "operational" to "ops".
- "Competitive compensation & bonus" (31) → "Comp & bonus
  structure" (22) — abbreviate "compensation" to "comp".
- "and" → "&" when the option starts running long.

If any option still exceeds 30 chars after auto-fix
attempts, surface the specific option to the user with two
or three rewrite suggestions and ask them to pick or supply
their own.

If a question exceeds 140 chars, the question itself needs
trimming — not auto-fixable in most cases. Surface to the
user with a tightened candidate.

Capture per-poll character budgets in the file's
front-matter so the human reviewer sees the tightness at a
glance.

## Phase 6: Stop-slop pass

Run the shared stop-slop skill at
`rockstarr-infra/skills/_shared/stop-slop/SKILL.md` on every
prose element across all 10 polls — every post-copy block,
every question, every option. Order is fixed: style guide
shapes voice first, character limits enforce structure
second, stop-slop strips AI tells last.

What stop-slop catches especially in this lane:

- "Delve," "navigate," "unlock," "tapestry."
- "The real question is..." / "Here's the thing..." /
  "What's interesting is..." / "It's not just X, it's Y."
- "In today's fast-paced world."
- Em dashes in post copy — replace with periods or commas.
- Three-in-a-row same-length sentences (rare in polls but
  flag if it happens).

Record a `stop_slop_score` in the file's front-matter — a
single score for the whole set. Surface to the chat
summary. Below 35/50 flags for reviewer.

## Output

Write to
`/rockstarr-ai/03_drafts/social/polls_set-<N>_<persona-slug>.md`.
If the file exists, append `-v2`, `-v3`. Never overwrite a
previous draft without archiving to `99_archive/`.

Required front-matter:

```yaml
# ---
channel: "linkedin-polls"
set_number: 8
persona: "Chris Swan"
persona_slug: "chris"
client_id: [from client.toml]
poll_count: 10
brand_hashtag: "#transearch"
audience_summary: "C-suite / boards / HR partners (from client-profile.md)"
altitude_check_status: "passed (12 candidates checked, 2 dropped on altitude, 10 selected)"
themes_covered:
  - "talent / leadership"
  - "growth / market"
  - "culture / retention"
  - "trends / tech"
style_reference_set: "polls_set-7_chris.md"
char_budgets:
  - poll: 1
    question_chars: 124
    max_option_chars: 28
    tightness_flag: false
  - poll: 2
    question_chars: 138
    max_option_chars: 30
    tightness_flag: true   # one or both fields within 5 chars of cap
  # ... per poll
produced_by: "rockstarr-social/draft-polls@0.1.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
stop_slop_score: 41
# ---
```

Body structure:

```markdown
# LinkedIn Polls — Set [N] — [Client] [Persona]

**Brand hashtag:** [from style guide]
**Audience altitude:** [one-line summary from client-profile.md]
**Style reference:** [previous approved set filename]

---

## Poll 1: [Topic Title]

**Post Copy:**

[Setup line.]

[Pivot line.]

[Vote-prompt line.]

**Poll Question:** [Punchy question, max 140 chars]

**Options:**

- [Option 1]
- [Option 2]
- [Option 3]
- [Option 4]

**Hashtags:** [tags ending in brand hashtag]

**Char budget:** Q [n]/140 · max option [n]/30

---

## Poll 2: [Topic Title]

(same structure)

---

(repeat through Poll 10)

---

## Set summary

- **Themes covered:** [list]
- **Style reference:** Set [N-1] (`polls_set-N-1_persona.md`)
- **Topics deduplicated against:** Sets [N-3, N-2, N-1]
- **Tightness flags:** [poll numbers where char budget is
  within 5 chars of cap]
- **Stop-slop score:** [n] / 50
```

## After writing

1. Print a summary in chat:
   - Set number and persona.
   - 10 topic titles, numbered.
   - Tightness flags (any polls within 5 chars of caps).
   - Topics dropped on altitude (if any).
   - Topics dropped on diversity (if any).
   - Stop-slop score.
   - The file path.

2. End with:

   > Polls Set [N] for [Persona] landed at
   > `03_drafts/social/polls_set-<N>_<persona-slug>.md`.
   > Review the whole set as one approval — polls aren't
   > approved individually. When ready, run
   > `rockstarr-infra:approve` to promote the set to
   > `04_approved/social/`.

3. Do not call `approve` yourself. The whole set is one
   human-review event.

## Approval and publishing

The standard rockstarr-infra `approve` flow promotes the file
from `03_drafts/social/` to `04_approved/social/` with a
date stamp. The strategist or client then publishes the set
on LinkedIn over the following days/weeks (typically one
poll per day or every other day, on the persona's account).
Each individual poll publish gets logged via
`rockstarr-infra:publish-log` with a `channel: linkedin-polls`
entry that references the approved set's filename and the
specific poll number within the set.

The set is the approval unit; individual polls within the
set are the publish unit.

## What NOT to do

- Do NOT skip the altitude check. It's the #1 failure mode
  for this lane. A polished set of off-altitude polls wastes
  every later step.
- Do NOT patch individual polls when altitude fails. Re-pick
  the whole batch at the corrected altitude. Trying to
  rescue 3 of 10 off-altitude picks produces an inconsistent
  set.
- Do NOT eyeball character counts. Count programmatically.
  31-char options pass visual scan and fail LinkedIn at
  publish time.
- Do NOT mix personas in one set. Each set is one persona's
  voice. If a multi-persona client wants two sets in the
  same month, run this skill twice (once per persona).
- Do NOT propose topics already covered in the last 2-3
  sets, even with a different angle. Topic freshness across
  sets is what keeps a multi-month polls cadence from
  reading as a fill-in-the-blank template.
- Do NOT paraphrase third-party content into the client's
  voice. The first-party voice rule still applies — KB
  carries the perspective, web research is reference-only
  (and rarely needed for polls).
- Do NOT skip the stop-slop pass. Polls are short enough
  that one AI tell stands out as much as five do in a long
  blog. The pass is mandatory.
- Do NOT batch-deliver more than 10 polls per call unless
  the user explicitly asks. The 10-poll set is the canonical
  unit. Larger sets dilute the altitude-and-diversity
  discipline that makes them work.
- Do NOT move the file to `04_approved/`. That is `approve`'s
  job.

## Related

- `style-guide.md` § Channel Adaptation → LinkedIn polls —
  brand hashtag, persona list, voice rhythm, hashtag mix.
  Single source of truth for polls voice.
- `04_approved/social/polls_set-*.md` — past approved sets;
  read for style continuity (newest set wins) and topic
  de-duplication.
- `rockstarr-infra:approve` — the promotion path from
  `03_drafts/social/` to `04_approved/social/`.
- `rockstarr-infra:publish-log` — per-poll publish logging
  on LinkedIn.
- `rockstarr-content:repurpose` — different lane. Repurpose
  creates short-form derivatives FROM approved long-form
  pieces. Polls are original short-form, not derivatives.
- `draft-social` — sister skill in this plugin for free-form
  short-form posts. Polls have a hard structural constraint
  (the 4-option question); free posts do not.
