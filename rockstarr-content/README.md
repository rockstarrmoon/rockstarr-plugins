# rockstarr-content

Rockstarr AI long-form content drafting plugin. Four publishing
lanes gated by the client's stack cadence, plus interview-driven
case studies and repurposed derivatives. Sits on top of
`rockstarr-infra` and hands finished drafts to the standard
`approve` / `publish-log` lifecycle.

This is the second plugin in the Rockstarr AI Growth Operating
System. It does not post, send, or schedule — those are jobs for
`rockstarr-social`, `rockstarr-outreach-bot-<variant>`, and the
email side of the stack. This plugin drafts long-form content
only. Short-form formats — including LinkedIn polls — live in
the `rockstarr-social` plugin. Publish connectors
(`publish-wp`, `publish-ga`, `publish-linkedin-newsletter`) are
deferred until a signed client's stack lights up the matching
field.

## At a glance

- **Four publishing lanes** gated by `stack.md` cadence:
  researched blog, thought leadership, email newsletter, LinkedIn
  newsletter. Plus interview-driven case studies (quarterly) and
  on-demand repurposed derivatives.
- **Both long-form lanes are outline-first.** Researched blog
  flows through `outline-blog` → `draft-blog`. Thought leadership
  flows through `outline-thought-leadership` → `draft-thought-leadership`
  (added in v0.3 — see below).
- **Monthly cadence:** `ideate-topics` runs once per month,
  `content-calendar` lays the picks on the calendar, per-piece
  drafting runs on its scheduled date.
- **Mandatory stop-slop pass on every prose output.** Style guide
  shapes voice; stop-slop strips AI tells. Structural artifacts
  (topic lists, calendars, outlines, interview transcripts) are
  exempt.
- **Mandatory TL rubric pass (v0.3).** Thought-leadership drafts
  run the canonical rubric BEFORE stop-slop — argument quality
  first, prose hygiene second. Slogans don't get polished.
- **Mandatory blog SEO/GEO checklist (v0.4).** Researched-blog
  drafts run the 13-item canonical SEO + GEO checklist BEFORE
  stop-slop. FAQ section is required, every external fact
  carries an inline `[Source: URL]`, keyword placement and
  density are tracked, meta title and meta description live in
  front-matter. The checklist closes the gap between
  professionally-written content and content that AI search
  systems actually cite.
- **Strategic content planning (v0.5).** `seo-strategy` runs a
  six-phase keyword research + topic clustering workflow per
  client. Output: a strategy document plus a canonical backlog
  of 25-32 prioritized blog topics organized into 4-5 clusters
  (one pillar page + 4-6 supports each). `ideate-topics`,
  `content-calendar`, and `outline-blog` are all backlog-aware
  — strategy decides which topics matter; the monthly skills
  sequence them.
## v0.7 — LinkedIn polls moved to rockstarr-social

The polls lane added in v0.6 was the first explicitly
short-form lane in rockstarr-content. Once we built the
broader `rockstarr-social` plugin, polls fit better there —
alongside open-ended social posts, scheduling, and other
short-form formats with their own publishing cadence.

- `draft-polls` is gone from rockstarr-content. The skill,
  with all its altitude-check and character-limit logic
  intact, now lives in `rockstarr-social`. The port
  preserved the architectural decisions: set-as-approval-unit,
  enum cadence, all polls config in `style-guide.md` §
  Channel Adaptation → LinkedIn polls.
- `stack.md.polls_cadence` and the `Channel Adaptation →
  LinkedIn polls` subsection of `style-guide.md` are still
  the source-of-truth locations — clients with polls config
  in place don't need workspace migration. Only the
  implementing skill changed plugins.
- The plugin scope tightens back to long-form-only.
  Short-form (polls and otherwise) is `rockstarr-social`'s
  responsibility going forward.

## v0.5 — strategic content planning

The researched-blog lane in v0.4 produced individual blogs
that ranked and got cited. v0.5 adds the layer above:
strategic planning that decides which blogs matter for the
client across 6-12 months and organizes them into pillar +
supporting clusters that build topical authority over time.

- **New skill: `seo-strategy`.** Run on demand, typically
  quarterly. Six phases: gather inputs from intake artifacts
  → research the client website + keyword landscape →
  generate 30 seed keywords + 50 long-tail variants →
  identify 5 competitor gaps and build 4-5 topic clusters →
  write the strategy document and the canonical backlog →
  verify cluster integrity. Two outputs: a dated strategy doc
  at `02_inputs/seo/strategy_<YYYY-MM-DD>.md` (audit history
  preserved across runs) and a canonical backlog at
  `02_inputs/seo/backlog.md` (regenerated each run). No
  ClickUp integration — the backlog file is the
  production-tracking artifact in this stack.
- **`ideate-topics` is backlog-aware.** When the SEO backlog
  exists, `ideate-topics` reads it and prefers backlog items
  over free ideation for the blog lane. Each backlog-sourced
  angle carries `from_backlog: true`, slug, cluster,
  cluster role, parent pillar slug, target keyword, search
  intent, difficulty, and quick-win flag. The skill warns the
  user when fewer than 2× monthly cadence remains in the
  backlog so the strategist can refresh before it empties.
  Other lanes (TL, newsletter, LinkedIn) ideate as before.
- **`content-calendar` enforces cluster ordering.** When
  cluster-tagged picks land in the same month, the pillar's
  full chain (outline → draft → publish) calendars BEFORE
  any supporting post in that cluster begins. Cross-month
  dependencies are surfaced in the calendar's notes so the
  strategist knows what unblocks next month.
- **`outline-blog` defaults to cluster-aware internal
  linking.** Supporting posts default to one link to the
  parent pillar + 1-2 peer supports. Pillar pages default to
  one outbound link per supporting post in their cluster
  (the pillar is the cluster's navigation hub). The
  human reviewer can adjust at outline-approval time.
- **`draft-article` shim removed.** The v0.2 deprecation
  finished its scheduled lifecycle. Any saved workflow
  still referencing `draft-article` returns a "not found"
  and routes the user to `outline-blog` or
  `outline-thought-leadership` directly.

## v0.4 — researched-blog SEO + GEO integration

The researched-blog lane in v0.3 produced clean prose but
under-performed on AI citation rates. The fix is a canonical
SEO + GEO reference enforced at two layers (outline + draft),
parallel to the v0.3 thought-leadership work:

- **`outline-blog`** now runs an external WebSearch research
  phase on top of the first-party KB read. The outline now
  carries SEO/GEO targets, a required FAQ section (3-5 H3
  questions), a keyword placement plan, an internal linking
  plan, an external sources table, and meta title +
  description drafts.
- **`draft-blog`** refuses to run if the outline lacks the
  SEO/GEO scaffolding. Body REQUIRES the FAQ section. Every
  external fact carries an inline `[Source: URL]` next to
  the claim. The 13-item quality checklist runs as Pass 1
  before stop-slop runs as Pass 2.
- **The reference** lives at
  `rockstarr-infra/skills/_shared/references/blog-seo-geo.md`
  (rockstarr-infra v0.8.2+). It's the single source of truth
  for the GEO patterns, FAQ structure, keyword placement
  rules, internal linking rules, inline source URL
  convention, meta specs, and the quality checklist.
- **Research stays first-party-voice only.** External sources
  are CITED inline and listed in References. Their phrasing
  is never paraphrased into the client's voice. The
  rockstarr-content posture from v0.2 stands.

## v0.3 — thought-leadership rubric integration

Thought-leadership pieces in v0.2 were single-shot. They came out
flat. The fix isn't editing — it's catching fuzziness before
drafting begins. v0.3 introduces a rubric-driven outline gate
parallel to the researched-blog flow:

- **`outline-thought-leadership`** forces five required fields
  before any prose runs: thesis, smart-competitor counter-argument,
  opening scene, quotable line, buried proprietary term. Two
  attempts per field; if a field is fuzzy after two tries, the
  skill stops and surfaces the failure to the human. The point is
  to catch fuzziness before drafting wastes compute.
- **`draft-thought-leadership`** now requires the approved
  outline. After drafting, runs Pass 1 (TL rubric — seven-test
  critique frame from the canonical rubric) before Pass 2
  (stop-slop). Order is fixed.
- **`ideate-topics`** captures an `enemy` field on every TL
  angle and runs an enemy-diversity check across the month's TL
  slate. Two TL pieces with rhyming enemies surface to the user
  before the calendar gets approved.

The rubric itself lives once, as a shared reference at
`rockstarr-infra/skills/_shared/references/tl-rubric.md`. Single
source of truth.

## v0.2 historical note

v0.2 was the lane-and-skill-identity restructure that landed the
four-lane model:

- **v0.1's `draft-blog` → v0.2's `draft-thought-leadership`.**
  The shorter, opinion-driven piece named for what it is.
- **v0.1's `draft-article` → v0.2's `draft-blog`.** The longer,
  researched, outline-first piece is now the default "blog".
- **`draft-article` retained as a deprecated alias.** Scheduled
  for removal in v0.4 (deferred from v0.3 to keep that release
  focused on the rubric work).
- **Mandatory stop-slop pass added in 0.2.1.** Every drafting
  skill runs the shared stop-slop skill at
  `rockstarr-infra/skills/_shared/stop-slop/` as the final step
  before writing.

## Lanes and cadence

Every lane is gated by a field in `stack.md`. Cadence 0 means the
lane is suppressed — no topics proposed, no slots scheduled, no
drafts generated.

| Lane | Primary skill | Cadence field | Repurpose path |
|------|---------------|---------------|----------------|
| Researched blog | `outline-blog` (research + FAQ + keyword + linking plan) → `draft-blog` (FAQ body + inline sources + 13-item checklist) | `blogs_per_month` | Referenced from newsletters; optional `repurpose` to LinkedIn post. |
| Thought leadership | `outline-thought-leadership` → `draft-thought-leadership` | `thought_leadership_per_month` | Source for LinkedIn newsletters (via `publish-linkedin-newsletter`, DEFER); referenced from newsletters; optional `repurpose`. |
| Email newsletter | `draft-newsletter` | `email_newsletters_per_month` | Links to the month's blog + TL pieces. Not repurposed back. |
| LinkedIn newsletter | `publish-linkedin-newsletter` (DEFER) | `linkedin_newsletters_per_month` | Reuse of an approved TL piece. Not a new draft. |
| Case study | `draft-case-study` | `case_studies_per_quarter` | On-demand (quarterly reminder), outside the monthly calendar. |
| Derivatives | `repurpose` | n/a — source-gated on approved blog/TL | Video-script output gated on `records_videos`. |

## Skills

| Skill | Status | Purpose |
|-------|--------|---------|
| `seo-strategy` | NEW (0.5) | Run on demand (typically quarterly). Six-phase keyword research + topic clustering workflow. Reads `client-profile.md`, `stack.md`, and the publish log; runs WebFetch on the client site and WebSearch on the keyword landscape; generates 30 seed keywords + 50 long-tail variants + 8-12 quick wins; identifies 5 competitor gaps; builds 4-5 topic clusters with one pillar page + 4-6 supports each. Outputs: `02_inputs/seo/strategy_<YYYY-MM-DD>.md` (audit-preserved) and `02_inputs/seo/backlog.md` (canonical, regenerated per run). 25-32 prioritized blog topics ready for `ideate-topics` to draw from monthly. |
| `ideate-topics` | UPDATED (0.5) | Monthly. Reads profile, style guide, first-party KB, publish log, stack-cadence. **As of v0.5, also reads `02_inputs/seo/backlog.md` if it exists and prefers backlog items for the blog lane.** Each TL angle still carries an `enemy` field with diversity check; backlog-sourced blog angles carry slug + cluster + cluster role + parent pillar + target keyword + search intent + difficulty + quick-win flag. Surfaces a low-backlog warning when fewer than 2× monthly cadence remains. Output: `02_inputs/content-topics_YYYY-MM.md`. |
| `content-calendar` | UPDATED (0.5) | Monthly. Slots the user's picks across the month. Blog and TL outline-to-publish paths first (both lanes are outline-first); newsletters anchored to preferred weekday; LinkedIn newsletters aligned to an approved TL. **As of v0.5, enforces cluster-aware ordering for backlog-sourced picks: pillar pages calendar before their supporting posts within the same cluster, and cross-month dependencies are flagged in the calendar's notes.** Output: `02_inputs/content-calendar_YYYY-MM.md`. |
| `outline-blog` | UPDATED (0.5) | Outline-first gate for the researched blog lane. Runs a WebSearch research phase on top of the first-party KB read, requires an FAQ section (3-5 questions) in the outline, produces a keyword placement plan, internal linking plan, external sources table, and meta title + description drafts. **As of v0.5, defaults the internal linking plan to cluster-aware patterns when the topic comes from the SEO backlog**: supporting posts link to their parent pillar + 1-2 peer supports; pillar pages link out to one supporting post per H2 section. Reads the shared blog-seo-geo reference. Approval required before `draft-blog` runs. |
| `outline-thought-leadership` | EXISTING (0.3) | Outline-first gate for the thought-leadership lane. Forces five required fields — thesis, smart-competitor counter-argument, opening scene, quotable line, buried proprietary term — applied from the canonical TL rubric. Approval required before `draft-thought-leadership` runs. |
| `draft-blog` | EXISTING (0.4) | Researched, informational blog. Consumes the approved outline, produces a full post citing first-party KB and external sources. Body REQUIRES an FAQ section. Every external fact carries an inline `[Source: URL]` next to the claim plus a References section at the bottom. Runs the 13-item SEO/GEO quality checklist as Pass 1 before stop-slop (Pass 2). |
| `draft-thought-leadership` | EXISTING (0.3) | Opinion-driven piece. Outline-gated. Drafts the named thesis, opens on the named scene, places the named quotable line in the first third, buries the proprietary term until the second half. Runs the canonical TL rubric as Pass 1, then stop-slop as Pass 2. |
| `draft-newsletter` | EXISTING | Single-shot email newsletter. CTA links the month's approved blog + TL pieces. Mandatory stop-slop pass on subject lines, preheader, and body. |
| `draft-case-study` | EXISTING | Quarterly. Interview-driven via `AskUserQuestion` using the Rockstarr custom-GPT case-study prompt (sourced from rockstarr-infra). Transcript in `02_inputs/content/`; polished draft only after every required question is answered. Mandatory stop-slop on the polished prose; transcript exempt. |
| `repurpose` | EXISTING | Takes one approved long-form piece and fans it into LinkedIn post, newsletter highlight, X/Threads thread, and (only when `records_videos=true`) a short video script. Mandatory stop-slop pass per derivative. |
| `draft-polls` | MOVED OUT (0.7) | Introduced in v0.6, moved to the new `rockstarr-social` plugin in v0.7 where it fits alongside other short-form social content. Polls cadence (`polls_cadence` enum), brand hashtag, and persona-list config still live in `stack.md` and `style-guide.md` § Channel Adaptation → LinkedIn polls — those workspace conventions are unchanged; only the implementing skill moved plugins. |
| `draft-article` | REMOVED (0.5) | Deprecated v0.2 shim deleted on schedule. Any saved workflow still referencing `draft-article` returns a "not found" error and routes the user to `outline-blog` or `outline-thought-leadership` directly. |
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
- `rockstarr-infra/skills/_shared/references/tl-rubric.md` — the
  canonical thought-leadership rubric. Read by
  `outline-thought-leadership` (forces the five required fields),
  `draft-thought-leadership` (post-draft rubric pass before
  stop-slop), and `ideate-topics` (enemy-diversity check). Single
  source of truth — do not fork.

If any of these are missing or out of date, skills refuse and
point the user back at the relevant infra skill.

## rockstarr-infra dependencies

This plugin requires **rockstarr-infra v0.8.2+** at minimum
(carrying the shared blog SEO + GEO reference the v0.4
researched-blog flow reads). Individual dependencies and the
infra version that introduced each:

1. **`capture-stack`** — content-cadence block with six fields:
   `blogs_per_month`, `thought_leadership_per_month`,
   `email_newsletters_per_month`,
   `linkedin_newsletters_per_month`, `records_videos`,
   `case_studies_per_quarter`. (rockstarr-infra v0.4+)
2. **`scaffold-client`** — content subdirectories:
   `02_inputs/content/` (case-study interview transcripts),
   `04_approved/content/`, `05_published/content/`,
   `06_reports/monthly/`. (rockstarr-infra v0.4+)
3. **Shared reference:
   `skills/_shared/references/case-study-prompt.md`** — the
   Rockstarr custom-GPT case-study interview prompt.
   `draft-case-study` reads from this path. (rockstarr-infra v0.4+)
4. **Shared skill: `skills/_shared/stop-slop/`** — MIT-licensed
   stop-slop skill (derived from
   [hardikpandya/stop-slop](https://github.com/hardikpandya/stop-slop)).
   Every drafting skill in this plugin calls it as the mandatory
   final pass. Every other Growth-OS bot that produces prose
   (social, outreach, reply, nurture) calls the same canonical
   copy — one upstream skill, zero drift. (rockstarr-infra v0.4+)
5. **Shared reference:
   `skills/_shared/references/tl-rubric.md`** — canonical
   thought-leadership rubric. Read by `outline-thought-leadership`,
   `draft-thought-leadership`, and `ideate-topics`. Defines the
   three tests, patterns to cut, structural rewrite checklist,
   enemy-diversity standard, and quick critique frame. Single
   source of truth — do not fork. (rockstarr-infra **v0.8.1+**)
6. **Shared reference:
   `skills/_shared/references/blog-seo-geo.md`** — canonical
   blog SEO + GEO reference. Read by `outline-blog` (research
   phase, FAQ outline, keyword + internal-linking plans, meta
   drafts) and `draft-blog` (FAQ body, inline source URLs,
   keyword density, direct-answer pattern, structured
   definitions, the 13-item quality checklist that runs as
   Pass 1 before stop-slop). Single source of truth — do not
   fork. (rockstarr-infra **v0.8.2+** — this is the floor for
   v0.4 of this plugin)

**Approvals layer** (rockstarr-infra v0.8.0+). Every draft this
plugin emits carries `approval_status: pending` and
`awaiting_approval_since` front-matter — these are read by
`approvals-digest` (daily client-bound email summarizing pending
items) and `approvals-backlog-alert` (weekly strategist alert
when the queue exceeds threshold). No work required in this
plugin to participate; the front-matter contract is already
in place across every skill.

### Resolving the floor

This plugin's hard floor is rockstarr-infra **v0.8.2+** — the
floor advances with each plugin minor that adds a shared
reference dependency:

- v0.3 of this plugin requires infra v0.8.1+ (TL rubric).
- v0.4 of this plugin requires infra v0.8.2+ (blog SEO/GEO
  reference) — and v0.4 still carries the v0.3 work, so the
  TL flow needs v0.8.1 as well.

Both infra bumps (0.8.0 → 0.8.1 → 0.8.2) are pure additive —
each adds one new file under `skills/_shared/references/` and
changes no skill behavior. The upgrade chain is cheap to run.

A client running rockstarr-infra v0.8.0 or v0.8.1 will see this
plugin's affected skills refuse to run with a clear pointer
back at the infra version they need. Other lanes (newsletter,
case study, repurpose) stay functional regardless.

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
  00_intake/stack.md                       (content-cadence block + website_base_url)
  01_knowledge_base/index.md
  01_knowledge_base/processed/**           (owned, style_guide_eligible)
  01_knowledge_base/processed/third-party/ (reference only)
  02_inputs/seo/backlog.md                 (ideate-topics, when present)
  03_drafts/**                             (filter out in-flight slugs)
  04_approved/content/                     (draft-newsletter, repurpose, slug filter)
  05_published/_publish.log                (avoid repeats, resolve pillar URLs)
  rockstarr-infra/skills/_shared/references/case-study-prompt.md
  rockstarr-infra/skills/_shared/references/tl-rubric.md
  rockstarr-infra/skills/_shared/references/blog-seo-geo.md
  rockstarr-infra/skills/_shared/stop-slop/     (final pass, every drafting skill)

rockstarr-content writes:
  02_inputs/seo/strategy_<YYYY-MM-DD>.md   (seo-strategy, dated audit history)
  02_inputs/seo/backlog.md                 (seo-strategy, canonical, regenerated per run)
  02_inputs/content-topics_YYYY-MM.md      (ideate-topics)
  02_inputs/content-calendar_YYYY-MM.md    (content-calendar)
  02_inputs/content/case-study-interview-<slug>.md (draft-case-study)
  03_drafts/content/outline_<slug>.md      (outline-blog)
  03_drafts/content/outline-tl_<slug>.md   (outline-thought-leadership)
  03_drafts/content/<slug>.md              (draft-blog, draft-thought-leadership)
  03_drafts/content/<yyyy-mm-dd>_newsletter_<slug>.md (draft-newsletter)
  03_drafts/content/case-study-<slug>.md   (draft-case-study)
  03_drafts/social/linkedin_<slug>.md      (repurpose)
  03_drafts/social/thread_<slug>.md        (repurpose)
  03_drafts/content/newsletter-highlight_<slug>.md (repurpose)
  03_drafts/video/script_<slug>.md         (repurpose, only when records_videos=true)
```

All paths are relative to the client's `/rockstarr-ai/` root.

## Strategic + monthly flow

```
Quarterly or on-demand (strategy layer, v0.5)
  seo-strategy             → 02_inputs/seo/strategy_<YYYY-MM-DD>.md
                             02_inputs/seo/backlog.md (canonical, regenerated)
                             (25-32 prioritized blog topics, 4-5 clusters)

Day 1 (first business day, monthly)
  ideate-topics            → content-topics_YYYY-MM.md in 02_inputs/
                             (reads SEO backlog when present, prefers backlog
                              items for blog lane; runs free ideation for
                              other lanes; user picks which to draft per lane)

Day 2-3
  user marks Pick: yes in content-topics_YYYY-MM.md

Day 3
  content-calendar         → content-calendar_YYYY-MM.md in 02_inputs/
                             (enforces pillar-before-supports ordering when
                              cluster-tagged picks land in the same month)
                             rockstarr-infra:approve (monthly gate)

Days 4 through end-of-month (on calendar dates)
  blog outline slot        → outline-blog   → outline_<slug>.md
                             (research phase + FAQ + cluster-aware linking;
                              user approves outline)
  blog draft slot          → draft-blog     → <slug>.md
                             (FAQ body + inline sources + 13-item checklist
                              Pass 1 + stop-slop Pass 2)
                             approve → publish

  TL outline slot          → outline-thought-leadership → outline-tl_<slug>.md
                             (5 required fields; user approves outline)
  TL draft slot            → draft-thought-leadership → <slug>.md
                             (TL rubric Pass 1 + stop-slop Pass 2)
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
- `0.7.0` — LinkedIn polls moved out.
  - `draft-polls` removed from this plugin. The skill (with
    all altitude-check + character-limit + style-continuity
    logic intact) now lives in the new `rockstarr-social`
    plugin. Polls fit better there alongside other
    short-form social content.
  - Workspace conventions unchanged. `stack.md.polls_cadence`
    enum and the `Channel Adaptation → LinkedIn polls`
    subsection of `style-guide.md` are still the
    source-of-truth locations — clients with polls config
    don't need workspace migration. Only the implementing
    skill changed plugins.
  - The `04_approved/social/polls_set-*.md` reads-line and
    the `03_drafts/social/polls_set-<N>_<persona-slug>.md`
    writes-line are removed from this plugin's folder
    contract.
  - Plugin scope tightens to long-form-only. Short-form
    (polls and other formats) is `rockstarr-social`'s
    responsibility going forward.
  - The "does not ideate social posts" rule that softened
    in v0.6 reverts and broadens — this plugin no longer
    drafts any short-form content of any kind.
  - Skills count: 11 in v0.6 → 10 in v0.7.
- `0.6.0` — LinkedIn polls lane.
  - **New skill: `draft-polls`.** Writes a batched set of
    10 LinkedIn polls per persona in a single file. The
    batch is the approval unit. Each poll carries post
    copy + question (≤140 chars) + 2-4 options (≤30 chars
    each) + hashtags ending in the brand hashtag.
  - **Audience-altitude pre-check.** Before any prose,
    the skill generates 12-15 candidate topics and
    cross-checks them against `client-profile.md`'s
    audience definition. ≥3 altitude failures triggers a
    full batch re-pick — don't patch individual polls
    when altitude drifts. This is the lane's #1 failure
    mode per the source SOP.
  - **Hard character-limit enforcement.** Counted
    programmatically at write time, with auto-fix
    suggestions for common 31-char option mistakes
    (`/` spacing, "ops" for "operational", "comp" for
    "compensation", "&" for "and").
  - **Style continuity.** Reads the most recent approved
    set in `04_approved/social/polls_set-*.md` and
    pattern-matches its voice rhythm. Newest approved
    style wins.
  - **Topic diversity.** Filters candidates against the
    last 2-3 approved sets so multi-month cadence
    doesn't read as a template.
  - **Cadence enum.** `polls_cadence` in `stack.md` is
    `monthly` / `quarterly` / `on-demand` / `off`. Polls
    don't fit the per-month numeric model of long-form
    lanes — set is the unit, not piece.
  - **Polls config in style-guide.md.** Brand hashtag,
    persona list, voice rhythm, hashtag mix all live in
    `Channel Adaptation → LinkedIn polls`. Skills refuse
    if missing.
  - **Single-file output per set.**
    `03_drafts/social/polls_set-<N>_<persona-slug>.md`
    holds all 10 polls for the set. Approved as one
    batch via `rockstarr-infra:approve`.
  - **Plugin scope softens.** The "does not ideate
    social posts" rule narrows to "does not ideate
    open-ended social posts" — LinkedIn polls are a
    structured short-form format that fits this plugin
    rather than waiting for `rockstarr-social-bot`.
  - No infra bump strictly required. Lightweight ship —
    the polls subsection in style-guide.md and the
    `polls_cadence` field in stack.md are currently
    manual additions, with infra-side automation
    (`generate-style-guide` and `capture-stack` updates)
    landing in a future infra cohort.
- `0.5.0` — Strategic content planning layer.
  - **New skill: `seo-strategy`.** Runs the six-phase keyword
    research + topic clustering workflow on demand
    (typically quarterly). Reads existing intake artifacts
    (no re-asking ICP and niche), runs WebFetch on the
    client site + WebSearch on the keyword landscape,
    generates 30 seed keywords + 50 long-tail variants,
    identifies 5 competitor gaps, builds 4-5 topic clusters
    with one pillar page + 4-6 supporting posts each.
    Outputs a dated strategy document (audit-preserved
    across runs) plus a canonical backlog at
    `02_inputs/seo/backlog.md` (regenerated per run).
    Backlog typically holds 25-32 prioritized blog topics.
    No ClickUp integration — the backlog file is the
    production-tracking artifact.
  - **`ideate-topics` is backlog-aware.** When the SEO
    backlog exists, blog-lane angles are sourced from the
    backlog by default. Each angle carries the slug,
    cluster name, cluster role (pillar / supporting),
    parent pillar slug, target keyword, search intent,
    difficulty estimate, and quick-win flag. Items already
    in the workflow (published, approved, drafting, in the
    current month's slate) are filtered out. A low-backlog
    warning fires when fewer than 2× monthly cadence
    remains so the strategist can refresh proactively.
    TL, newsletter, and LinkedIn lanes ideate as before —
    the backlog is blog-specific.
  - **`content-calendar` enforces cluster ordering.** When
    backlog-sourced picks land in the same month, the
    pillar's full chain (outline → draft → publish) is
    calendared before any supporting post in the cluster
    starts. Cross-month dependencies (pillar this month,
    support unblocked next month) surface in the
    calendar's notes. A backwards dependency (support
    picked, pillar not yet shipped) is flagged loudly and
    the user is asked to resolve before approval.
  - **`outline-blog` defaults to cluster-aware internal
    linking.** Supporting posts default to one link to the
    parent pillar + 1-2 peer supports. Pillar pages
    default to one outbound link per supporting post
    (the pillar is the cluster's navigation hub). Pillars
    intentionally exceed the standard 3-5 link guidance —
    the SEO/GEO checklist's link-count test reads
    `cluster_role: pillar` and interprets "3-5 links" as
    "≥3" rather than "exactly 3-5".
  - **`draft-article` shim removed on schedule.** The v0.2
    deprecation finished its lifecycle. Saved workflows
    still referencing `draft-article` return a "not found"
    error.
  - **Folder additions.** New directory
    `02_inputs/seo/` is created on first run of
    `seo-strategy`. The `02_inputs/seo/strategy_*.md`
    files are dated for audit; `02_inputs/seo/backlog.md`
    is canonical and regenerated per run.
  - No infra version bump required for this cohort. v0.5
    introduces no new shared references — the
    `seo-strategy` patterns are read by exactly one skill
    and live inside `seo-strategy/SKILL.md`. If a second
    consumer emerges later (e.g. a channel-specific
    strategy variant), refactor to a shared reference at
    that point.
- `0.4.0` — Researched-blog SEO + GEO integration.
  - **Lane structural change.** The researched-blog lane now
    runs an external WebSearch research phase at outline time
    on top of the first-party KB read. The body now requires
    an FAQ section (3-5 H3 questions, direct-answer-first)
    because that's where AI extraction is densest. Every
    external fact carries an inline `[Source: URL]` placement
    next to the claim, alongside the existing References
    section at the bottom.
  - **`outline-blog` updated.** New research phase. New
    front-matter fields: `meta_title_draft`,
    `meta_description_draft`, `faq_questions`,
    `keyword_placement_plan`, `internal_links_planned`,
    `external_sources_planned`. Body adds an SEO/GEO targets
    section, a keyword placement plan, an internal linking
    plan with anchor + target URL per entry, an external
    sources table, meta title and meta description drafts.
    Stack precondition tightened: `website_base_url` must be
    set in `stack.md` for the internal linking plan to build.
  - **`draft-blog` updated.** Refuses to run if the outline
    doesn't carry the v0.4 SEO/GEO fields (re-run
    `outline-blog`). Body REQUIRES the FAQ section. Every
    external fact gets an inline `[Source: URL]`. Front-matter
    carries `meta_title`, `meta_description`,
    `keyword_density`, and counts the reviewer can audit at a
    glance. Runs the 13-item SEO/GEO quality checklist as
    Pass 1 before stop-slop (Pass 2). Order is fixed:
    structural quality first, prose hygiene second.
  - **New shared reference:
    `rockstarr-infra/skills/_shared/references/blog-seo-geo.md`** —
    canonical SEO + GEO standard. Defines GEO patterns, FAQ
    structure, keyword placement and density rules, internal
    linking rules, the inline `[Source: URL]` convention, meta
    title and description specs, and the 13-item quality
    checklist. Single source of truth.
  - **Research posture preserved.** External research provides
    cited facts and references; voice stays first-party-only.
    External sources are never paraphrased into the client's
    voice — they appear as `[Source: URL]` placements and in
    the References section.
  - Keyword density is tracked and reported but NOT
    hard-enforced. If the actual density falls outside the
    0.5-1% band, the chat summary surfaces it for the
    reviewer; the file still writes.
  - Depends on `rockstarr-infra` **v0.8.2+** for the shared
    reference. v0.8.1 had the TL rubric but not blog-seo-geo;
    upgrading is pure additive (one new file).
  - **Note:** the deprecated `draft-article` shim was
    scheduled for v0.5 removal; that landed alongside the
    SEO strategy work in v0.5.0.
- `0.3.0` — Thought-leadership rubric integration.
  - **Lane structural change.** Thought leadership is no longer
    single-shot. The lane now flows through
    `outline-thought-leadership` (gate) → `draft-thought-leadership`,
    parallel to how the researched-blog lane works.
  - **New skill: `outline-thought-leadership`.** Forces five
    required fields before any prose — thesis, smart-competitor
    counter-argument, opening scene, quotable line, buried
    proprietary term. Two attempts per field; if a field is fuzzy
    after two tries, surface to the human. The point of the gate
    is to catch fuzziness before drafting wastes compute.
  - **`draft-thought-leadership` updated.** Refuses to run without
    an approved outline. Adds a post-draft rubric pass (Pass 1)
    that runs the canonical TL rubric's seven-test critique frame
    before stop-slop (Pass 2). Order is fixed: argument quality
    first, prose hygiene second.
  - **`ideate-topics` updated.** Each thought-leadership angle
    carries an `enemy` field (one-sentence headline of what the
    piece argues against). Post-proposal, runs an enemy-diversity
    check across the month's TL slate; flags rhyming enemies for
    user resolution.
  - **New shared reference:
    `rockstarr-infra/skills/_shared/references/tl-rubric.md`** —
    canonical TL rubric. Read by all three skills above. Single
    source of truth.
  - Depends on `rockstarr-infra` **v0.8.1+** for the shared
    rubric reference (`tl-rubric.md`). v0.8.0 is not enough — the
    rubric was carved out into its own patch release. The v0.3
    TL skills refuse to run if the rubric file is missing and
    point the user back at the infra upgrade.
  - **Still planned for a later cut:** remove the deprecated
    `draft-article` shim (now scheduled for v0.4); wire publish
    connectors as signed clients request them.

## What this plugin does not do

- Does not post, schedule, or send. Publish connectors deferred.
- Does not ideate or draft any short-form content — that's
  `rockstarr-social` (LinkedIn polls live there as of v0.7;
  open-ended social posts and other short-form formats also
  belong there). This plugin is long-form only.
- Does not generate outreach or reply copy (those are separate
  bot variants).
- Does not re-open the style guide. If the draft reveals a voice
  issue, run `rockstarr-infra:generate-style-guide` upstream.
- Does not approve its own drafts. Human review through
  `rockstarr-infra:approve` is the only path to `04_approved/`.
- Does not accept a pre-written case-study document. The
  case-study lane is interview-first by design.
