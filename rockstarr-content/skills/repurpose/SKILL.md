---
name: repurpose
description: "This skill should be used when the user asks to \"repurpose this blog\", \"turn this thought-leadership piece into a LinkedIn post\", \"fan out this approved piece\", \"generate derivatives\", or names an approved long-form piece and asks for social / newsletter / video derivatives. Takes one approved piece from 04_approved/content/ and emits derivative drafts: a LinkedIn post, a newsletter highlight, an X / Threads thread, and (only when stack.md.records_videos=true) a short video script. Each derivative lands in 03_drafts/[channel]/ for separate review."
---

# repurpose

Take an approved long-form piece and fan it out into channel-native
derivatives. Repurpose does not re-research or reframe — it
compresses and reshapes the approved argument for each channel's
native format. Voice stays the same.

> **Template convention.** Fenced code blocks below show `# ---` where
> YAML front-matter delimiters belong, to keep Cowork's SKILL.md
> parser from misreading them. **When writing the actual output file,
> emit real `---`, not `# ---`.**

## When to run

- User names an approved blog or thought-leadership piece and asks
  to repurpose it.
- User asks to "turn this into a LinkedIn post" / "make a thread
  out of this" / "pull a newsletter highlight from this".
- A downstream workflow (social bot, newsletter bot) is planned
  but not yet built — in the meantime this skill produces the
  derivative drafts for human review.

## Preconditions

- Target file exists at
  `/rockstarr-ai/04_approved/content/[slug].md` with
  `channel: "blog"` or `channel: "thought-leadership"` in its
  front-matter.
  - Case studies can be repurposed too; same approval path.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/stack.md` exists.

If the source piece is a newsletter (`channel: "newsletter"`),
refuse — newsletters are the ingestion point, not a repurpose
source. If the user insists, point them at the underlying blog or
TL piece referenced in the newsletter's `monthly_pieces_linked`.

## Inputs

Read in this order:

1. The approved source piece — full body and front-matter.
2. Style guide — **Channel Adaptation** sections drive per-channel
   format. Tone Definition pairs still apply to every derivative.
3. Client profile — the client's sign-off conventions on LinkedIn
   and the tone they use on each channel.
4. `/rockstarr-ai/00_intake/stack.md` — key field:
   `records_videos`. Controls whether the video-script derivative
   is written.

## Derivatives

The skill always attempts these three derivatives:

1. **LinkedIn post.** 1200-1500 characters, native format. Hooks
   on the strongest line from the source, compresses the argument
   into a post that stands on its own without requiring a
   click-through, ends with the source link as a PS. No hashtag
   stuffing — match the style guide's hashtag convention (if
   any).
2. **Newsletter highlight.** 150-250 words. A compressed version
   of the argument suitable for inclusion in the client's next
   email newsletter — or for teasing the full piece. Ends with a
   one-line CTA and the source URL.
3. **X / Threads thread.** 5 to 9 posts, each ≤ 280 chars. First
   post hooks, middle posts advance the argument, last post
   converts (links to the source or the client's offer). Respect
   the style guide's conventions on thread shape.

The skill ALSO attempts this derivative, but only when
`stack.md.records_videos == true`:

4. **Short video script.** 60-120 seconds of spoken copy. Hook in
   the first 10 seconds, argument in the middle, CTA at the end.
   Formatted as speakable lines (not paragraphs). Note camera /
   visual directions only if the client's past scripts show that
   convention.

If `records_videos == false` or is missing, skip the video-script
derivative entirely and note it in the run summary. Do not write
an empty file.

## Output

Each derivative writes to its own file under
`/rockstarr-ai/03_drafts/[channel]/`:

- `/rockstarr-ai/03_drafts/social/linkedin_[slug].md`
- `/rockstarr-ai/03_drafts/content/newsletter-highlight_[slug].md`
- `/rockstarr-ai/03_drafts/social/thread_[slug].md`
- `/rockstarr-ai/03_drafts/video/script_[slug].md` (only when
  `records_videos == true`)

If any of these directories do not exist yet, create them.

Each derivative file carries its own front-matter:

```yaml
# ---
channel: "linkedin-post" | "newsletter-highlight" | "x-thread" | "video-script"
source_channel: "blog" | "thought-leadership" | "case-study"
source_slug: "[source slug]"
source_path: "04_approved/content/[slug].md"
slug: "kebab-cased-slug"
title: "Working title for the derivative"
char_count: 1240          # LinkedIn / thread
word_count: 210           # newsletter-highlight
spoken_seconds: 78        # video-script
produced_by: "rockstarr-content/repurpose@0.2.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md front-matter"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

### LinkedIn post body structure

```markdown
# [Working title — internal only]

[Hook line — the strongest sentence from the source, rewritten
to stand on its own.]

[2-3 short paragraphs that advance the argument. Line breaks
every 1-2 sentences for LinkedIn readability.]

[Closing line — one-line stance or question.]

PS: [Full-source link line. Match the client's convention —
"Full post →" or similar.]

Source: [URL]
```

### Newsletter highlight body structure

```markdown
# [Working title — internal only]

[150-250 words. Opens with a specific scene or claim. Compresses
the source into one clear idea. Ends with a one-line CTA
pointing at the full source.]

Read the full piece → [URL]
```

### X / Threads thread body structure

```markdown
# [Working title — internal only]

## Thread plan
- 5-9 posts
- Hook → argument → CTA

## Posts

**1 / [total]** (hook)
[≤ 280 chars]

**2 / [total]**
[≤ 280 chars]

...

**[total] / [total]** (CTA)
[≤ 280 chars — includes the source link]
```

### Video script body structure (only when records_videos=true)

```markdown
# [Working title — internal only]

**Target length:** 60-120 seconds spoken
**Estimated spoken seconds:** 78

## Script

[HOOK — 5-10 seconds]
[Spoken line 1]
[Spoken line 2]

[ARGUMENT — 30-60 seconds]
[Spoken line N]
...

[CTA — 10-15 seconds]
[Spoken line N]
[CTA action — from stack.md]
```

## Drafting rules

1. **Compress, do not re-research.** The source piece is the
   canon. Derivatives shrink or reshape it; they do not
   introduce new claims, new numbers, or new customers.
2. **Voice stays the same.** Tone Definition pairs still apply
   to every derivative. Short-form channels sometimes tempt
   towards LinkedIn-speak — the style guide still governs.
3. **Banned language stays banned.** Same search-and-replace
   pass as drafting skills.
4. **No invented facts.** Same rule. Any stat or customer
   reference must trace back to the source piece.
5. **Stack gates the video script.** `records_videos == false`
   means no video file is written. Do not ask the user to
   override — changing the flag lives in `capture-stack`.
6. **Character and word counts matter.** LinkedIn and X/Threads
   drop silently if over-limit. Count at write time; flag if a
   post blows the cap.
7. **Source-link fidelity.** Pull the live URL from
   `stack.md.website_base_url + /blog/<slug>` (or the client's
   actual pattern). Do not invent.

## After writing

1. Print a summary in chat: derivative list, file paths, source
   piece, any skipped derivatives and the reason (e.g., "video
   script skipped — records_videos=false").
2. End with:

   > Derivatives landed in `03_drafts/`. Each is a separate
   > approval — review and run `rockstarr-infra:approve` on
   > each one individually.

3. Do not call `approve` yourself.

## What NOT to do

- Do not repurpose an unapproved piece. Source must be in
  `04_approved/content/`.
- Do not emit a video-script derivative when
  `records_videos != true`. Silence is the correct behavior.
- Do not invent new claims, numbers, or customer references in
  the derivatives.
- Do not batch-approve the derivatives — each channel gets its
  own review.
- Do not write the derivatives to `04_approved/` or log them as
  published. Those are downstream skills.
- Do not repurpose a newsletter or a derivative of a derivative.
  Source must be an approved long-form piece (blog, TL, or case
  study).
