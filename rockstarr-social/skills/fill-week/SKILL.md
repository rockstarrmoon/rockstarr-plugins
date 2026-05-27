---
name: fill-week
description: "This skill should be used when the user asks to \"fill the week\", \"build next week's social batch\", or when the scheduled Friday-morning run fires. Builds a balanced weekly batch of 5-10 short-form posts: reads social_posts_per_week + social_mix from stack.md, picks topics from first-party KB + recently approved long-form, dedups against the publish log, loops draft-social per slot. Writes a batch manifest week-YYYY-WW.md in 03_drafts/social/ plus per-post drafts. Batch is the approval unit; once approved, run publer-export."
---

# fill-week

Build the weekly social batch — one slot per post in the
configured mix, looping `draft-social` per slot. The batch is
the approval unit; individual posts inside are the publish unit.

> **Template convention.** Fenced blocks below show `# ---` where
> YAML front-matter delimiters belong. **When writing the actual
> output file, emit real `---`, not `# ---`.**

## When to run

- The scheduled Friday-morning run fires (default Fri 09:00
  client-local; configured in `stack.md.social_fill_week_cron`).
- User says "fill the week", "build next week's social batch",
  "draft this week's posts".
- Mid-week catch-up if the scheduled run was skipped.

## Preconditions

- `client-profile.md`, `style-guide.md`, `stack.md` all present.
- `stack.md` has the social block with `social_posts_per_week`
  and `social_mix`. If missing, refuse and route to
  `rockstarr-infra:capture-stack`.
- `04_approved/social/` exists.
- `05_published/_publish.log` exists (created on first publish).

## Inputs

1. `stack.md` — `social_posts_per_week`, `social_mix`,
   `social_channels`. The mix dictates slot count by type.
2. `style-guide.md` — voice baseline.
3. `client-profile.md` — audience.
4. **Approved long-form** in `04_approved/content/` from the
   trailing 30 days — candidates for `promo` slots.
5. **First-party KB** (`kb_scope: owned`) — anecdote pool for
   `insight` and `case` slots.
6. **`05_published/_publish.log`** — topic de-dup. Anything
   shipped in the trailing 14 days is off-limits.
7. **`04_approved/social/`** — read the most recent batch's
   topics for week-over-week diversity.
8. The current ISO week number, computed from today's date in
   the workspace's local TZ.

## Phase 1: Resolve the slot map

Read `social_mix` and expand into a flat slot list. Example:

```yaml
social_mix:
  promo: 1
  insight: 2
  case: 1
  engagement: 1
```

Becomes 5 slots: `promo`, `insight`, `insight`, `case`,
`engagement`. The order in the slot list is also the proposed
publish order across the week (Monday → Friday).

If `social_posts_per_week` and the sum of `social_mix` values
disagree, surface to the user — the mix is the source of truth
for distribution; the count is the budget. If the mix sums to
fewer slots, ask whether to drop the difference or add slots
(propose `insight` for the extra slots — it's the safest fill).

## Phase 2: Pick topics per slot

Loop the slot list. For each slot, pick a topic per these rules:

### `promo` slots

- Candidate pool: approved long-form pieces in
  `04_approved/content/` from the trailing 30 days.
- One promo per long-form piece across all of social — never
  promo the same piece twice in two weeks.
- If no eligible long-form, **drop the slot** and add an
  `insight` slot instead. Note the swap in the batch summary.

### `insight` slots

- Candidate pool: first-party KB topics with no `_publish.log`
  entry in the trailing 14 days.
- Prefer KB material the client has not posted about in 30+
  days — fresh perspective beats recently-rehashed perspective.
- 1-2 candidates per slot; surface to the user if 0 candidates
  match.

### `case` slots

- Candidate pool: customer wins / case-study material in the KB
  AND any customer-named case study approved in the trailing 90
  days.
- Slot can be skipped if no eligible material AND the user
  prefers not to backfill — case slots without real material
  produce vague-sounding posts. Default behavior: surface and
  ask.

### `engagement` slots

- Candidate pool: open-ended questions tied to the audience's
  current concerns (read `client-profile.md`'s audience
  description and the most recent KB entries for cues).
- Engagement posts are the fastest to draft and the easiest to
  swap; prioritize a topic the audience can answer in one
  sentence.

### Diversity check across the week

After picking 5-10 candidate topics, check:

- No two slots cover the same topic from different angles.
- The mix of themes (talent, growth, product, market) spans at
  least 2-3 themes.
- No slot's topic appears in the trailing 14 days of
  `_publish.log` or in the prior week's batch.

If diversity fails, re-pick the failing slot from its candidate
pool. If the candidate pool can't satisfy diversity, surface to
the user with the conflict named.

## Phase 3: Draft each slot via draft-social

For each slot, invoke `draft-social` with:

- `channel`: from `stack.md.social_channels` (LinkedIn primary;
  if X / IG enabled and the slot type is one the client posts
  cross-channel, draft for LinkedIn first and let `repurpose` or
  a manual cross-post handle the others).
- `post_type`: the slot's type.
- `brief`: the picked topic.
- `brief_source`: `approved-piece` (promo) | `topic` (insight,
  engagement) | `case` (case).

`draft-social` runs its own stop-slop pass and writes a
per-post file. Collect each per-post file path for the batch
manifest.

If `draft-social` surfaces a question (missing brief, ambiguous
post-type), pause the loop, answer it, and resume. Do not
silently skip a slot.

## Phase 4: Write the batch manifest

Write `/rockstarr-ai/03_drafts/social/week-[YYYY-WW].md`. This
is the human-review entry point; per-post files are the
artifacts.

Required front-matter:

```yaml
# ---
batch_kind: "weekly-social"
iso_week: "2026-W19"
week_start: "2026-05-04"
week_end:   "2026-05-10"
client_id: [from client.toml]
post_count: 5
mix:
  promo: 1
  insight: 2
  case: 1
  engagement: 1
slot_swaps: []   # any promo->insight type swaps
posts:
  - slot: 1
    type: "promo"
    brief: "..."
    file: "post-2026-05-04-...-promo.md"
    proposed_publish_date: "2026-05-04"
  - slot: 2
    type: "insight"
    brief: "..."
    file: "post-2026-05-05-...-insight.md"
    proposed_publish_date: "2026-05-05"
  # ... per slot
produced_by: "rockstarr-social/fill-week@0.1.0"
produced_at: "ISO timestamp"
approval_status: "pending"
awaiting_approval_since: "ISO timestamp"
# ---
```

Body structure:

```markdown
# Weekly social batch — ISO week 2026-W19

**Week:** Mon 2026-05-04 → Sun 2026-05-10
**Channels:** LinkedIn (primary)
**Mix:** 1 promo / 2 insight / 1 case / 1 engagement

---

## Slot 1 — Mon 2026-05-04 — promo

**Brief:** [topic / link to approved piece]
**Draft file:** `post-2026-05-04-<slug>.md`
**Hook (variant A):**
> [first line of post]

---

## Slot 2 — Tue 2026-05-05 — insight

(same shape per slot)

---

## Batch summary

- **Slot swaps:** [none, or "Slot 3 promo → insight (no eligible long-form in trailing 30 days)"]
- **Topics deduped against:** ISO weeks W17, W18 + trailing 14 days of `_publish.log`
- **Per-post stop-slop scores:** [Slot 1: 42, Slot 2: 38, ...]

---

## Approval

Review each slot's draft file. Edits go in the per-post file.
When the whole batch is satisfactory, run
`rockstarr-infra:approve` against THIS file (the manifest) — the
approve flow promotes the manifest to `04_approved/social/` and
the per-post files follow.

After approval, run `publer-export` (or `social-export-ga`) to
write the scheduler import to
`05_published-staging/social/<YYYY-WW>.csv`.
```

## Phase 5: Chat summary

Print a single concise summary:

- ISO week + week-of dates.
- Slot list: position, type, hook excerpt (first ~10 words),
  per-post stop-slop score.
- Slot swaps if any.
- Topic dedup window applied.
- The manifest file path.

End with:

> Weekly batch landed at `03_drafts/social/week-[YYYY-WW].md`.
> Review the manifest, edit per-slot drafts as needed, then run
> `rockstarr-infra:approve` to promote the whole batch.

Do not call `approve` yourself.

## What NOT to do

- Do NOT skip slots silently. Surface every swap or skip.
- Do NOT promo a long-form piece that's already been promoed in
  the trailing 30 days.
- Do NOT generate posts for channels the client hasn't enabled
  in `social_channels`. LinkedIn-only clients don't get X
  drafts.
- Do NOT batch-deliver more than `social_posts_per_week` posts.
  The configured count is the budget; mid-week extras are a
  separate `draft-social` call.
- Do NOT auto-trigger `publer-export`. The user runs the export
  manually after approval (V0.1 design choice).
- Do NOT promote any file to `04_approved/`. That's `approve`'s
  job.

## Related

- `draft-social` — called once per slot.
- `repurpose` (in rockstarr-content) — produces social
  derivatives from approved long-form; use it instead when the
  whole brief is fan-out from one approved piece.
- `publer-export` / `social-export-ga` — runs after approval.
- `rockstarr-infra:approve` — batch promotion path.
- `rockstarr-infra:publish-log` — per-post publish logging.
