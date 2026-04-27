---
title: "Thought-leadership rubric"
purpose: "Canonical standard for what makes a Rockstarr thought-leadership piece compelling. Applied at outline time and again at draft time."
applies_to: "thought-leadership lane only — opinion / stance pieces, not researched blogs or newsletters"
read_by:
  - "rockstarr-content/outline-thought-leadership"
  - "rockstarr-content/draft-thought-leadership"
  - "rockstarr-content/ideate-topics" (enemy-diversity check only)
do_not_fork: true
---

# Thought-leadership rubric

Use this when reviewing or revising a thought-leadership piece that
is supposed to read as thought leadership but feels flat, generic,
or pitchy. The goal is articles that have a real perspective, a
specific point of view, and make the author stand out — not
articles that restate the same positioning frame in different
words.

## The three tests every article must pass

Before any draft ships, it has to clear all three. If it fails any
one of them, send it back.

### 1. One argument a smart competitor could publicly disagree with

A slogan ("Marketing isn't broken, you are") is not an argument. A
real argument is specific enough that another expert in the field
could write a counter-piece. If the central claim of the article
cannot be argued against, it is positioning, not perspective.

Force the writer to state the argument as a single sentence at the
top of the outline. If they can't, the article doesn't have a
thesis yet.

**The enforcement trick:** require the writer to produce the
counter-argument as a discrete artifact — write the smart
competitor's rebuttal in one sentence as part of the outline. If
the counter is itself slogan-shaped or trivially wrong, the
original argument isn't real either.

### 2. One specific story, told in detail

Replace lists of stats and testimonials with one client or
scenario, walked through end-to-end. What was the person doing on
a Tuesday before? What did they try? What broke? What changed?
What are they doing now? Specifics earn authority. Generic plurals
("most owners," "every founder we talk to") don't. One full story
beats five name-drops.

### 3. One line a reader could repeat at dinner

A crystallized take the reader steals from the author. Place it
high in the article — not buried in paragraph four. Every piece
should have one of these on purpose. If you can't pull a quotable
sentence out of the draft, the article hasn't said anything sharp
enough yet.

**The enforcement trick:** require the writer to flag the line
explicitly, in front-matter and in the body. The human reads it
first and judges it before reading the rest.

## Patterns to cut on sight

These show up across drafts and signal template, not original
thinking. Strip them and replace with something specific.

- "Here is the part most owners never get told."
- "Every owner we talk to."
- "Most owners."
- "It is not a small distinction."
- Any opener that begins with a vague plural ("Every founder,"
  "Most leaders," "All consultants").
- Any reveal phrase that has already been used in another article
  in the same newsletter cadence.

When the same rhetorical move appears in multiple articles in the
same newsletter cadence, it stops reading as voice and starts
reading as a fill-in-the-blank template.

## Structural rewrite checklist

Run every draft through this list:

- **Open with a specific person in a specific moment.** Not "every
  owner." A named role, a named situation, a named time of day.
  The reader needs to see themselves in the first 100 words or
  they leave.
- **State the thesis as one sentence in the first third of the
  article.** No throat-clearing. No "let me set this up." Lead
  with the argument.
- **Bury proprietary terms until the second half.** If the article
  exists to teach a brand term (a product name, a methodology
  name, a coined phrase), it is internal product marketing, not
  thought leadership. The reader should care about the IDEA
  before they care about what the author calls the thing.
- **One story per article, told end-to-end.** Cut stat lists. Cut
  multi-client name-drops. Pick one and go deep.
- **End with the crystallized line, then the CTA.** Not the CTA
  alone. The take-home sentence comes first; the booking link
  comes after.
- **No more than one rhetorical "reveal" per article.** "Here's
  the thing nobody tells you" is fine once. Twice is a tic.

## Multi-article series — enemy diversity

If the newsletter publishes multiple articles in a series, each
piece must make a different argument with a different enemy. Four
pieces that all defend the same product around the same villain
read as one ad repeated four times — even if the topics are
nominally different.

**Test for this explicitly.** Write the central argument of each
article in the series as a one-sentence headline. If two of them
rhyme, one of them needs to change.

Examples of distinct enemies for a four-part series:

- Vague positioning that blocks referrals.
- Owner-dependent AI use that scales the bottleneck.
- Buying execution instead of building a system.
- Confusing pipeline that runs on memory with pipeline that runs
  on infrastructure.

Each gets one argument, one story, one quotable line. Not four
versions of the same point of view.

## Quick critique frame

When a draft is in front of you, run this in order:

1. **What is the single argument?** State it in one sentence. If
   you can't, send it back.
2. **Could a smart competitor publicly disagree with it?** If no,
   it's a slogan — send it back.
3. **Is there one specific story told in detail?** If it's a list
   of names and stats, send it back.
4. **What is the line a reader will repeat?** Highlight it. If
   there isn't one, send it back.
5. **Does the proprietary term appear before the idea is
   established?** If yes, move it later.
6. **Does the article share rhetorical tics with other articles
   in the series?** If yes, cut them.
7. **Does the closing line crystallize the take, or is it just a
   CTA?** If just a CTA, add a take-home line above it.

## What "compelling" actually means

Compelling is not "well-written." It's not "tight prose." It's not
"clean structure." Those are table stakes. Compelling means:

- The reader encounters an idea they hadn't articulated before.
- The reader sees themselves in a specific moment described in
  the piece.
- The reader leaves with a sentence they want to repeat.
- The reader can imagine someone else publicly disagreeing — and
  would defend the argument anyway.

If a draft is well-written but doesn't do those things, it's
professional content. It is not thought leadership. The fix is
not editing — it's adding a real argument.

## Handing this to a writer

When you send a draft back, frame the feedback as:

- "What is the one argument here? State it in a sentence."
- "Who is the specific person in the opening? I need a role, a
  situation, a moment."
- "Pick one client and tell me the whole story. Cut the rest."
- "Pull out the line you want the reader to repeat. Move it up."
- "What is a smart competitor's counter-argument to this piece?
  If you can't name one, the argument isn't sharp enough yet."

Keep the questions short. Don't rewrite for the writer. Make them
answer the questions and produce v2.

## Note on dual roles

In Rockstarr-AI workflows, "the writer" can be the founder
themselves OR the drafting bot.

- When the founder is the author: the rubric's questions are
  surfaced via `AskUserQuestion`. The founder produces the v2
  themselves. The skill does not rewrite for them.
- When the bot is the author: the rubric's failures trigger
  re-generation, with the failed tests passed in as the
  correction prompt. After two failed regeneration attempts, the
  skill stops and surfaces the failure to the human — at that
  point the issue is upstream of writing (the founder is fuzzy
  on what they actually think, and no number of regenerations
  will rescue it).

This rubric helps a writer who has something to say but is buried
under template language. It does NOT rescue a writer who is
genuinely fuzzy on what they think. Skills using this rubric
should fail loudly in that case rather than burning compute
trying to manufacture conviction.
