---
name: draft-social
description: "This skill should be used when the user asks to \"draft a social post\", \"write a LinkedIn post\", \"draft a post about [topic]\", or \"turn this blog into a LinkedIn post\". Drafts a single short-form social post (LinkedIn primary, X / IG secondary) in the client's approved voice, optionally with two A/B hook variants. Reads first-party KB for anecdotes; third-party content is reference-only. Runs the shared stop-slop pass before saving. Output lands in 03_drafts/social/ as post-YYYY-MM-DD-slug.md for the standard approve flow."
---

# draft-social

Draft a single short-form social post — LinkedIn primary, X / IG
secondary — in the client's approved voice. One-off drafting
flow, separate from the weekly batch (see `fill-week`).

> **Template convention.** Fenced blocks below show `# ---` where
> YAML front-matter delimiters belong. **When writing the actual
> output file, emit real `---`, not `# ---`.**

## When to run

- User says "draft a LinkedIn post about X", "draft today's
  post", "turn this blog into a LinkedIn post", "draft a post
  for this week".
- An item from the weekly mix needs a one-off redraft (e.g., the
  approved blog just landed and the LinkedIn promo needs to be
  written before the rest of the batch).
- A standalone post outside the weekly cadence (event recap,
  spontaneous reaction, founder anecdote).

For full-week generation, call `fill-week` instead — it loops
this skill across the configured mix.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/stack.md` exists. The social block:

  ```yaml
  social_scheduler: "publer"   # publer | growthamp | native
  social_posts_per_week: 5
  social_mix:
    promo: 1          # promote a blog/newsletter/case
    insight: 2        # client POV, no asset link
    case: 1           # customer story or proof
    engagement: 1     # question / poll-adjacent / reaction
  social_channels:
    linkedin: true
    x: false
    instagram: false
  ```

  If the social block is missing, refuse and route to
  `rockstarr-infra:capture-stack`.

## Inputs

Read in this order:

1. `client-profile.md` — audience, ICP, business positioning.
2. `style-guide.md`, full document — voice baseline. Pay special
   attention to `Channel Adaptation → LinkedIn` (and X / IG if
   configured) for short-form rhythm, hook conventions,
   hashtag policy.
3. The **post brief** — whichever of these the user provided:
   - A topic string ("how I think about hiring slow").
   - A URL (blog post, news article, podcast episode).
   - A reference to an approved long-form piece in
     `04_approved/content/` (then this is a `repurpose`-shaped
     task — call out that `repurpose` is the better lane if the
     brief is a clean fan-out).
   - A free-form anecdote pasted into chat.
4. The **post-type** input — one of `promo`, `insight`, `case`,
   `engagement`. Required. If the user didn't specify, ask via
   `AskUserQuestion`.
5. First-party KB entries (`kb_scope: owned`) matching the topic
   — used for anecdotes, proof points, voice signals.
6. Recent approved posts in `04_approved/social/` — read the 3-5
   most recent for style continuity. Newest approved post wins
   on style.
7. `05_published/_publish.log` — for topic de-dup against what
   already shipped.

Third-party material may be cited as a link inside the post but
NEVER paraphrased into the client's voice.

## Phase 1: Resolve the brief and post-type

Ask via `AskUserQuestion` if any of these are missing:

- Channel (LinkedIn default; ask only if multiple channels are
  enabled in `stack.md`).
- Post-type (`promo` / `insight` / `case` / `engagement`).
- The brief itself, if the user gestured vaguely ("draft today's
  post").

Surface the resolved brief + post-type back to the user before
drafting:

> Drafting: a LinkedIn `insight` post on "[brief]" — using
> [KB anecdote source if any]. OK?

## Phase 2: Match the newest approved post's style

Read the 3-5 most recent files in `04_approved/social/`. Pull:

- Hook patterns (question, contrarian claim, anecdote opener,
  stat-led).
- Average length range.
- Paragraph rhythm (single-line breaks vs. dense blocks).
- Emoji policy (most clients none; if the recent approved set
  uses them sparingly, mirror that — never add).
- Hashtag count and mix.
- CTA pattern (link + soft prompt vs. naked question).

If the style guide's `Channel Adaptation → LinkedIn` and the
newest approved posts disagree, the **newest approved post wins**
— it's the most recent human-validated signal.

## Phase 3: Draft the post

Produce one canonical draft body. Hard rules:

1. **Hook first.** First two lines must earn the click — no
   "I'm excited to share...", no "In today's fast-paced world".
2. **One thought per post.** A short-form post that tries to
   make three points dilutes all three. Carry one point well.
3. **First-party voice only.** KB material is the perspective
   source. Stats and quotes from outside sources are cited
   inline with a link if used at all.
4. **CTA matches post-type.**
   - `promo`: link to the asset, one-line teaser, no hard
     "READ NOW".
   - `insight`: optional question to invite a reply; no link.
   - `case`: name the customer (with permission) or anonymize
     consistently; quantify the outcome; one-line takeaway.
   - `engagement`: question or two-side framing; no link.
5. **Hashtags follow the style guide.** End with the brand tag
   if the style guide names one. 3-5 tags is the typical band.
6. **Length per channel.** LinkedIn: ~150-300 words is the
   safe band; some clients prefer shorter. X: ≤280 chars.
   IG: caption budget set in style-guide.
7. **No em dashes in the body.** Use periods or commas. (This
   is a stop-slop catch but easier to avoid up front.)

### A/B hook variants (optional)

If the user asked for variants, or the post-type is `promo` (the
asset-link CTA benefits most from hook A/B testing), produce two
hook lines (the first 1-2 lines only — body stays the same) and
present both with the recommended pick called out.

## Phase 4: Stop-slop pass

Run the shared stop-slop skill at
`rockstarr-infra/skills/_shared/stop-slop/SKILL.md` over every
prose element — hook, body, CTA, alt-text if attached image.
Order: voice (style guide) first, structural rules (length, CTA)
second, stop-slop (AI tells) last.

What stop-slop catches especially in this lane:

- "Delve," "navigate," "unlock," "tapestry."
- "It's not just X, it's Y."
- "The real question is..." / "Here's the thing..."
- "I'm thrilled / excited / honored to..."
- Em dashes anywhere in the body.
- Three same-length sentences in a row (rare in posts but flag).

Record `stop_slop_score` in front-matter. Below 35/50 flags for
reviewer.

## Output

Write to `/rockstarr-ai/03_drafts/social/post-[YYYY-MM-DD]-[slug].md`.
Slug is a 3-6 word kebab from the topic. If the file exists,
append `-v2`, `-v3`. Never overwrite a previous draft without
archiving to `99_archive/`.

Required front-matter:

```yaml
# ---
channel: "linkedin"   # or "x", "instagram"
post_type: "insight"  # promo | insight | case | engagement
brief: "How I think about hiring slow"
brief_source: "topic"  # topic | url | approved-piece | anecdote
client_id: [from client.toml]
style_guide_version: "matched from style-guide.md front-matter"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
external_sources_cited: []
hook_variants: 2  # 1 or 2
recommended_variant: "A"  # or "B"
char_count: 1742
word_count: 287
stop_slop_score: 42
produced_by: "rockstarr-social/draft-social@0.1.0"
produced_at: "ISO timestamp"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# [Channel] post — [Post-type] — [Brief]

**Post-type:** [type] · **Channel:** [channel] · **Word count:** [n]

---

## Variant A (recommended)

**Hook:**

[1-2 line hook]

**Body:**

[main body]

**CTA:**

[link / question / no CTA]

**Hashtags:**

[#tag #tag #tag #brandtag]

---

## Variant B (optional)

**Hook:**

[alternate 1-2 line hook]

(body, CTA, hashtags identical to A unless explicitly forked)

---

## Notes for reviewer

- KB sources: [list]
- External sources: [none / list with links]
- Style reference: [most recent approved post filename]
- Stop-slop score: [n] / 50
```

## After writing

1. Print a chat summary:
   - Channel + post-type.
   - Word count + char count.
   - Hook variant count.
   - Stop-slop score.
   - File path.
2. End with:

   > Draft landed at `03_drafts/social/post-[YYYY-MM-DD]-[slug].md`.
   > Review the draft and run `rockstarr-infra:approve` to promote
   > to `04_approved/social/`.

3. Do not call `approve` yourself.

## What NOT to do

- Do NOT post directly. This plugin never publishes — see
  `publer-export` for the export step.
- Do NOT mix multiple post-types in one draft. One thought per
  post, one type per file.
- Do NOT paraphrase third-party content into the client's voice.
  Cite with a link if relevant; never speak as the client about
  someone else's perspective.
- Do NOT skip the stop-slop pass.
- Do NOT auto-shorten / auto-expand to hit a length target. If
  the draft runs long, surface to the user with a tightened
  candidate.
- Do NOT add emojis unless the recent approved posts use them.

## Related

- `fill-week` — calls this skill in a loop to build the weekly
  batch.
- `repurpose` (in rockstarr-content) — better lane when the
  brief is "fan out this approved blog into a LinkedIn post";
  use that instead of `draft-social` for clean derivatives.
- `style-guide.md` § Channel Adaptation → LinkedIn / X / IG —
  voice rules.
- `rockstarr-infra:approve` — promotion path from
  `03_drafts/social/` to `04_approved/social/`.
- `publer-export` / `social-export-ga` — scheduler export step.
