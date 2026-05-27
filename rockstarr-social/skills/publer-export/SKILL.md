---
name: publer-export
description: "This skill should be used when the user asks to \"export to Publer\", \"build the Publer CSV\", or \"generate the scheduler import\" for an approved batch. Reads the approved weekly batch manifest + per-post files and writes a Publer-shaped CSV (Date, Time, Account, Post, Media, Link, First Comment, Labels) to 05_published-staging/social/YYYY-WW.csv. Manual trigger only — does not auto-run on approval. Refuses if stack.md.social_scheduler is not 'publer'. Does not post — the client imports the CSV into Publer themselves."
---

# publer-export

Generate the Publer-shaped CSV import file for one approved
weekly social batch. Manual-trigger skill — the user runs it
after `rockstarr-infra:approve` lands the batch.

> **Template convention.** Fenced blocks below show `# ---` where
> YAML front-matter delimiters belong. **When writing the actual
> output file, emit real `---`, not `# ---`.**

## When to run

- User says "export to Publer", "build the Publer CSV", "export
  this week's batch", or names an approved manifest file.
- After `rockstarr-infra:approve` has promoted a weekly batch
  manifest from `03_drafts/social/` to `04_approved/social/`.

This skill does NOT auto-run on approval. The user triggers it
explicitly. (Design choice for V0.1; revisit in V0.2 if the
manual step proves friction.)

## Preconditions

- `stack.md.social_scheduler` is `"publer"`. If `"growthamp"`,
  refuse and route to `social-export-ga` (deferred — build on
  demand). If `"native"`, refuse with a note that the client
  publishes manually and no export file is produced.
- The named approved manifest file exists in
  `04_approved/social/week-[YYYY-WW].md`. If the user named a
  per-post file or a draft file by mistake, refuse with the
  expected path shape.
- `05_published-staging/social/` exists (create on first run).

## Inputs

1. The approved batch manifest at
   `04_approved/social/week-[YYYY-WW].md`.
2. Each per-post file the manifest references (in
   `04_approved/social/`).
3. `stack.md.social_channels` — drives the Account column
   value(s) in the CSV. The client's Publer account labels are
   in `stack.md.publer_accounts`:

   ```yaml
   publer_accounts:
     linkedin: "Jane Doe (LinkedIn)"
     x: "@janedoe (X)"
     instagram: "@janedoe (IG)"
   ```

4. The proposed publish dates from the manifest's `posts[].proposed_publish_date`.
5. `stack.md.social_default_post_time` — `"09:00"` if not
   set; the per-channel time the export uses unless a slot
   overrides.

## Phase 1: Resolve account labels

Read `stack.md.publer_accounts`. If a per-channel label is
missing for any channel the batch posts to, refuse with the
missing key called out — the CSV's Account column must match a
real Publer account name letter-for-letter or Publer drops the
row on import.

## Phase 2: Build the CSV rows

Publer's standard CSV import shape (May 2026):

| Column          | Source                                                   |
|-----------------|----------------------------------------------------------|
| Date            | `posts[i].proposed_publish_date` (`YYYY-MM-DD`)          |
| Time            | `stack.md.social_default_post_time` (24h `HH:MM`)        |
| Account         | `stack.md.publer_accounts[[channel]]`                    |
| Post            | full body text — hook + body + CTA + hashtags, joined    |
| Media           | empty (V0.1 doesn't attach media; surface if a draft references an image) |
| Link            | first URL in the post body if any (Publer auto-attaches) |
| First Comment   | empty by default; populated if the per-post file's front-matter has a `first_comment` field |
| Labels          | `rockstarr-social,[post_type],[iso_week]`                |

### Variant resolution

If a per-post file has variant A and variant B, the CSV uses the
manifest's `recommended_variant` (defaults to A). Variant B is
ignored at export time — the user can hand-edit the CSV before
import if they want B.

### Post body assembly

Per per-post file:

1. Read the recommended-variant section.
2. Concatenate Hook + blank line + Body + blank line + CTA +
   blank line + Hashtags. Single newline-separated paragraphs;
   no markdown headers in the post body.
3. Strip any leading/trailing whitespace.
4. Validate against channel limits:
   - LinkedIn: 3000-char body cap; warn if over 2500.
   - X: 280-char cap; refuse the row if over (X drafts at this
     length should never have left `draft-social`).
   - Instagram: 2200-char caption cap.

If a body fails its channel limit, write the rest of the CSV
but mark the failing row with a `_LIMIT_EXCEEDED_` prefix in the
Post column and surface the failure. The user fixes the source
draft, re-runs `rockstarr-infra:approve` (or the per-post
edit-then-approve loop), then re-runs this skill.

## Phase 3: Write the CSV

Path: `/rockstarr-ai/05_published-staging/social/[YYYY-WW].csv`.
If the file exists from a prior export attempt, append a
timestamp suffix (`[YYYY-WW]-[HHMMSS].csv`) — never overwrite
silently.

CSV format: standard RFC 4180 quoting, UTF-8, comma-delimited,
header row included. Quote any field containing commas, newlines,
or double-quotes. Newlines inside the Post column are preserved
(Publer accepts `\n` inside quoted fields).

## Phase 4: Write a sidecar manifest

Path: `/rockstarr-ai/05_published-staging/social/[YYYY-WW].manifest.md`.
Records what the CSV reflects, for audit:

```markdown
# Publer export — ISO week 2026-W19

**CSV:** `<YYYY-WW>.csv`
**Source manifest:** `04_approved/social/week-2026-W19.md`
**Source per-post files:**
- `04_approved/social/post-2026-05-04-<slug>.md` (slot 1)
- ...
**Account labels used:**
- LinkedIn: "Jane Doe (LinkedIn)"
**Default post time:** 09:00 client-local
**Variant resolution:** all rows used recommended variant
**Channel-limit warnings:** [list, or "none"]
**Channel-limit failures:** [list, or "none"]
**Exported at:** ISO timestamp
**Produced by:** rockstarr-social/publer-export@0.1.0
```

## Phase 5: Chat summary

Print:

- The CSV path.
- Row count + per-channel breakdown.
- Any channel-limit warnings or failures.
- The default post time used.

End with:

> Publer CSV at `05_published-staging/social/[YYYY-WW].csv`.
> Drop it into Publer → Bulk import → CSV. The week is ready
> for the client to schedule.

## Publish logging

This skill does NOT log to `_publish.log`. The publish event is
when the post goes live in Publer; the user (or a future
`detect-publishes` skill) calls `rockstarr-infra:publish-log`
per post once Publer fires.

## What NOT to do

- Do NOT auto-run on approval. V0.1 is manual-trigger.
- Do NOT export drafts. Only files in `04_approved/social/`.
- Do NOT ship a row that exceeds the channel char limit. Mark
  it and surface the failure; the user fixes the draft.
- Do NOT post directly. Publer is the publish surface; this
  skill produces the import file only.
- Do NOT include variant B in the CSV. The recommended variant
  is the export.
- Do NOT log to `_publish.log` from this skill. The publish
  event is the actual scheduler send.
- Do NOT touch the GrowthAmp planner format. That's
  `social-export-ga`'s job (deferred).

## Related

- `rockstarr-infra:approve` — produces the manifest this skill
  reads.
- `rockstarr-infra:publish-log` — per-post publish logging,
  fired after Publer actually publishes.
- `social-export-ga` — sister skill for the GrowthAmp planner
  format. Build on demand when a client's `social_scheduler` is
  `"growthamp"`.
- `fill-week` — produces the manifest one ISO week at a time.
- `draft-social` — produces the per-post files this skill
  reads.
