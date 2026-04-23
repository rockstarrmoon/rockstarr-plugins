# rockstarr-content

Rockstarr AI long-form drafting plugin. Four publishing lanes
gated by the client's stack cadence, plus interview-driven case
studies and repurposed derivatives. Sits on top of
`rockstarr-infra` and hands finished drafts to the standard
`approve` / `publish-log` lifecycle.

This is the second plugin in the Rockstarr AI Growth Operating
System. It does not post, send, or schedule — those are jobs for
`rockstarr-social-bot`, `rockstarr-outreach-bot-<variant>`, and the
email side of the stack. This plugin only drafts. Publish
connectors (`publish-wp`, `publish-ga`,
`publish-linkedin-newsletter`) are deferred until a signed
client's stack lights up the matching field.

## v0.2 at a glance

v0.2 is a significant restructuring of v0.1. The lane and skill
identities changed:

- **v0.1's `draft-blog` → v0.2's `draft-thought-leadership`.**
  The shorter, opinion-driven, single-shot piece is now named
  for what it is.
- **v0.1's `draft-article` → v0.2's `draft-blog`.** The longer,
  researched, outline-first piece is now the default "blog" — the
  outline-blog gate feeds it.
- **`draft-article` remains in v0.2 as a deprecated alias.** It
  prompts the user to pick a lane and routes them. Scheduled for
  removal in v0.3.
- **New monthly cadence.** `ideate-topics` runs once per month,
  `content-calendar` lays the picks on the calendar, per-piece
  drafting runs on its scheduled date.
- **Mandatory stop-slop pass (added in 0.2.1).** Every drafting
  skill in this plugin runs the shared stop-slop skill at
  `rockstarr-infra/skills/_shared/stop-slop/` as the final step
  before writing to `03_drafts/`. Style guide shapes voice;
  stop-slop strips the AI tells that survive even on-voice
  writing. Structural artifacts (topic lists, calendars,
  outlines, interview transcripts) are exempt.

## Lanes and cadence

Every lane is gated by a field in `stack.md`. Cadence 0 means the
lane is suppressed — no topics proposed, no slots scheduled, no
drafts generated.

| Lane | Primary skill | Cadence field | Repurpose path |
|------|---------------|---------------|----------------|
| Researched blog | `outline-blog` → `draft-blog` | `blogs_per_month` | Referenced from newsletters; optional `repurpose` to LinkedIn post. |
| Thought leadership | `draft-thought-leadership` | `thought_leadership_per_month` | Source for LinkedIn newsletters (via `publish-linkedin-newsletter`, DEFER); referenced from newsletters; optional `repurpose`. |
| Email newsletter | `draft-newsletter` | `email_newsletters_per_month` | Links to the month's blog + TL pieces. Not repurposed back. |
| LinkedIn newsletter | `publish-linkedin-newsletter` (DEFER) | `linkedin_newsletters_per_month` | Reuse of an approved TL piece. Not a new draft. |
| Case study | `draft-case-study` | `case_studies_per_quarter` | On-demand (quarterly reminder), outside the monthly calendar. |
| Derivatives | `repurpose` | n/a — source-gated on approved blog/TL | Video-script output gated on `records_videos`. |

## Skills

| Skill | Status | Purpose |
|-------|--------|---------|
| `ideate-topics` | UPDATED | Monthly. Reads profile, style guide, first-party KB, publish log, AND stack-cadence. Proposes 8–12 ranked angles ONLY for enabled lanes. Output: `02_inputs/content-topics_YYYY-MM.md`. |
| `content-calendar` | NEW | Monthly. Slots the user's picks across the month. Blog outline-to-publish paths first; TL next; newsletters anchored to preferred weekday; LinkedIn newsletters aligned to an approved TL. Output: `02_inputs/content-calendar_YYYY-MM.md`. |
| `outline-blog` | UPDATED | Outline-first gate for the researched blog lane. Approval required before `draft-blog` runs. |
| `draft-blog` | IDENTITY SWAP | Researched, informational blog. Consumes an approved outline, produces a full post citing first-party KB (and optionally third-party references). Mandatory stop-slop final pass. Replaces v0.1's `draft-article`. |
| `draft-thought-leadership` | NEW (renamed) | Shorter, opinion-driven, single-shot piece. Weight from POV, not research. Mandatory stop-slop final pass. Replaces v0.1's `draft-blog`. |
| `draft-article` | DEPRECATED | Back-compat shim. Asks the user which lane they meant and routes to `draft-blog` or `draft-thought-leadership`. Does not draft. Removed in v0.3. |
| `draft-newsletter` | UPDATED | Single-shot email newsletter. CTA links the month's approved blog + TL pieces. Mandatory stop-slop pass on subject lines, preheader, and body. |
| `draft-case-study` | NEW | Quarterly. Interview-driven via `AskUserQuestion` using the Rockstarr custom-GPT case-study prompt (sourced from rockstarr-infra). Transcript in `02_inputs/content/`; polished draft only after every required question is answered. Mandatory stop-slop on the polished prose; transcript exempt. |
| `repurpose` | NEW | Takes one approved long-form piece and fans it into LinkedIn post, newsletter highlight, X/Threads thread, and (only when `records_videos=true`) a short video script. Mandatory stop-slop pass per derivative. |
| `publish-wp` | DEFER | WordPress publish connector. Builds when first client's stack lights up. |
| `publish-ga` | DEFER | GrowthAmp blog publish connector. Builds when first client's stack lights up. |
| `publish-linkedin-newsletter` | DEFER | Republish an approved TL piece as a LinkedIn newsletter via Chrome MCP. Builds when first client's `linkedin_newsletters_per_month >= 1`. |

## Preconditions (every client)

Before running any skill in this plugin, `rockstarr-infra` must
already have produced:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/style-guide.md` (explicitly approved)
- `/rockstarr-ai/00_intake/stack.md` — **including the v0.2
  content-cadence block**: `blogs_per_month`,
  `thought_leadership_per_month`, `email_newsletters_per_month`,
  `linkedin_newsletters_per_month`, `records_videos`,
  `case_studies_per_quarter`.
- `/rockstarr-ai/01_knowledge_base/index.md` with at least some
  first-party processed files.
- `rockstarr-infra/skills/_shared/references/case-study-prompt.md`
  — the shared interview script for `draft-case-study`.
- `rockstarr-infra/skills/_shared/stop-slop/SKILL.md` — the
  shared stop-slop skill every drafting skill runs as its final
  pass. MIT-licensed, based on
  [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop).

If any of these are missing or out of date, skills refuse and
point the user back at the relevant infra skill.

## rockstarr-infra dependencies

The infra dependencies required by this plugin are shipped as of
rockstarr-infra v0.4:

1. **`capture-stack`** — content-cadence block with six fields:
   `blogs_per_month`, `thought_leadership_per_month`,
   `email_newsletters_per_month`,
   `linkedin_newsletters_per_month`, `records_videos`,
   `case_studies_per_quarter`.
2. **`scaffold-client`** — content subdirectories:
   `02_inputs/content/` (case-study interview transcripts),
   `04_approved/content/`, `05_published/content/`,
   `06_reports/monthly/`.
3. **Shared reference:
   `skills/_shared/references/case-study-prompt.md`** — the
   Rockstarr custom-GPT case-study interview prompt.
   `draft-case-study` reads from this path.
4. **Shared skill: `skills/_shared/stop-slop/`** — MIT-licensed
   stop-slop skill (derived from
   [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop)).
   Every drafting skill in this plugin calls it as the mandatory
   final pass. Every other Growth-OS bot that produces prose
   (social, outreach, reply, nurture) calls the same canonical
   copy — one upstream skill, zero drift.

**(Future backlog) `approvals-digest`.** Every draft this plugin
emits carries `awaiting_approval_since` front-matter so a future
infra digest can surface pending reviews across bots without
bot-specific code.

## Drafting rules (non-negotiable)

Applied by every drafting skill.

1. **Style guide is canon.** Read `style-guide.md` front to back
   before drafting. Apply Brand Personality, Tone Definition (the
   "X, not Y" pairs), Style Rules, and Channel Adaptation
   literally. Banned language stays banned.
2. **First-party voice only.** Only files with `kb_scope: owned`
   AND `style_guide_eligible: true` may be used as voice signal.
   Third-party KB is reference-only — cite it, do not paraphrase
   it as if the client said it.
3. **Cite the style guide.** Every draft's front-matter records
   `style_guide_version` so `approve` can audit drift.
4. **Drafts land in `03_drafts/content/`** (or the appropriate
   `03_drafts/<channel>/` for derivatives). Never write to
   `04_approved/` or `05_published/`.
5. **Required front-matter:** `channel`, `title`, `slug`,
   `produced_by`, `produced_at`, `style_guide_version`,
   `kb_sources_used`, `cta_text`, `cta_destination`,
   `approval_status`, `awaiting_approval_since`. Plus
   lane-specific fields (e.g., `outline_source` for blogs,
   `stance` for TL, `monthly_pieces_linked` for newsletters,
   `interview_source` for case studies, `source_path` for
   derivatives).
6. **Never invent positioning.** If the requested angle is not
   supported by the profile, the style guide, or the first-party
   KB, stop and ask the user to supply evidence.
7. **No third-party paraphrase.** Third-party sources appear only
   in "References" sections with explicit attribution and a link.
8. **Cadence is binding.** Skills refuse to emit drafts for lanes
   the client does not publish (cadence 0 in `stack.md`).
9. **stop-slop is the mandatory final pass.** Every drafting skill
   runs `rockstarr-infra/skills/_shared/stop-slop/` on its prose
   output immediately before writing the file. Order: style guide
   shapes the voice, then stop-slop strips AI tells. A draft that
   skips the pass ships broken. Every prose-emitting skill writes
   a `stop_slop_score` to the draft's front-matter and surfaces it
   in the chat summary. Structural artifacts (topic lists,
   calendars, outlines, interview transcripts) are exempt.

## Folder contract (reads and writes)

```
rockstarr-content reads:
  00_intake/client-profile.md
  00_intake/style-guide.md
  00_intake/stack.md                       (content-cadence block required)
  01_knowledge_base/index.md
  01_knowledge_base/processed/**           (owned, style_guide_eligible)
  01_knowledge_base/processed/third-party/ (reference only)
  04_approved/content/                     (draft-newsletter, repurpose)
  05_published/_publish.log                (avoid recent topic repeats)
  rockstarr-infra/skills/_shared/references/case-study-prompt.md
  rockstarr-infra/skills/_shared/stop-slop/     (final pass, every drafting skill)

rockstarr-content writes:
  02_inputs/content-topics_YYYY-MM.md      (ideate-topics)
  02_inputs/content-calendar_YYYY-MM.md    (content-calendar)
  02_inputs/content/case-study-interview-<slug>.md (draft-case-study)
  03_drafts/content/outline_<slug>.md      (outline-blog)
  03_drafts/content/<slug>.md              (draft-blog, draft-thought-leadership)
  03_drafts/content/<yyyy-mm-dd>_newsletter_<slug>.md (draft-newsletter)
  03_drafts/content/case-study-<slug>.md   (draft-case-study)
  03_drafts/social/linkedin_<slug>.md      (repurpose)
  03_drafts/social/thread_<slug>.md        (repurpose)
  03_drafts/content/newsletter-highlight_<slug>.md (repurpose)
  03_drafts/video/script_<slug>.md         (repurpose, only when records_videos=true)
```

All paths are relative to the client's `/rockstarr-ai/` root.

## Monthly flow

```
Day 1 (first business day)
  ideate-topics            → content-topics_YYYY-MM.md in 02_inputs/
                             (user picks which to draft per lane)

Day 2-3
  user marks Pick: yes in content-topics_YYYY-MM.md

Day 3
  content-calendar         → content-calendar_YYYY-MM.md in 02_inputs/
                             rockstarr-infra:approve (monthly gate)

Days 4 through end-of-month (on calendar dates)
  blog outline slot        → outline-blog   → outline_<slug>.md
                             (user approves outline)
  blog draft slot          → draft-blog     → <slug>.md
                             approve → publish

  TL draft slot            → draft-thought-leadership → <slug>.md
                             approve → publish
                             (publish-linkedin-newsletter eligible if TL slot has LinkedIn aligned)

  newsletter slot          → draft-newsletter → <yyyy-mm-dd>_newsletter_<slug>.md
                             approve → send

Quarterly (event-driven)
  case-study interview     → draft-case-study walks interview via AskUserQuestion
                             → case-study-interview-<slug>.md (transcript)
                             → case-study-<slug>.md (draft, after full interview)
                             approve → ship

On-demand
  approved long-form piece → repurpose → LinkedIn post + newsletter highlight + X/Threads thread
                             (+ video script if records_videos=true)
```

## Versioning

- `0.1.0` — initial cut (week of April 20, 2026). Five skills:
  `ideate-topics`, `outline-blog`, `draft-blog`,
  `draft-newsletter`, `draft-article`.
- `0.2.0` — significant restructuring (week of April 27, 2026).
  - Four publishing lanes gated by stack cadence.
  - Identity swap: `draft-blog` ↔ `draft-article` (see v0.2 at a
    glance).
  - `draft-article` retained as deprecated alias.
  - New skills: `content-calendar`, `draft-case-study`,
    `repurpose`.
  - `draft-newsletter` CTA links the month's blog + TL pieces.
  - `ideate-topics` filters by stack cadence; monthly output
    naming.
  - Paired bump to `rockstarr-infra` required (see dependencies
    section).
- `0.2.1` — mandatory stop-slop pass.
  - Every drafting skill (`draft-blog`,
    `draft-thought-leadership`, `draft-newsletter`,
    `draft-case-study`, `repurpose`) runs the shared stop-slop
    skill at `rockstarr-infra/skills/_shared/stop-slop/` as the
    final step before writing. Structural artifacts (topic lists,
    calendars, outlines, interview transcripts) are exempt.
  - Every draft carries a `stop_slop_score` in front-matter and
    in the chat summary. Scores below 35/50 flag for review.
  - Depends on `rockstarr-infra` v0.4 which ships the shared
    stop-slop skill.
- `0.3.0` (planned) — remove `draft-article` shim; wire publish
  connectors as signed clients request them.

## What this plugin does not do

- Does not post, schedule, or send. Publish connectors deferred.
- Does not ideate social posts (that is `rockstarr-social-bot`).
- Does not generate outreach or reply copy (those are separate
  bot variants).
- Does not re-open the style guide. If the draft reveals a voice
  issue, run `rockstarr-infra:generate-style-guide` upstream.
- Does not approve its own drafts. Human review through
  `rockstarr-infra:approve` is the only path to `04_approved/`.
- Does not accept a pre-written case-study document. The
  case-study lane is interview-first by design.
