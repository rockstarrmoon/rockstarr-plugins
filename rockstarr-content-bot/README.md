# rockstarr-content-bot

Rockstarr AI long-form drafting plugin. Ideates topics and drafts blog
posts, newsletters, and articles in the client's approved voice. Sits
on top of `rockstarr-infra` and hands finished drafts to the standard
`approve` / `publish-log` lifecycle.

This is the second plugin in the Rockstarr AI Growth Operating System.
It does not post, send, or schedule — those are jobs for
`rockstarr-social-bot`, `rockstarr-outreach-bot-<variant>`, and the
email side of the stack. Content Bot only drafts.

## Skills

| Skill              | Purpose                                                                                       |
|--------------------|-----------------------------------------------------------------------------------------------|
| `ideate-topics`    | Propose 8–12 content angles from the profile, style guide, first-party KB, and publish log.   |
| `outline-blog`     | Outline-first gate for blog drafting. H2s, angle, CTA, keyword, KB sources. Approval required.|
| `draft-blog`       | Consume an approved outline and produce a full blog post in the client's voice.               |
| `draft-newsletter` | Single-shot email newsletter: subject line options, preheader, intro, sections, CTA.          |
| `draft-article`    | Long-form research article: dek, lede, body, pull-quotes, conclusion.                         |

## Preconditions (every client)

Before running any skill in this plugin, `rockstarr-infra` must have
already produced:

- `/rockstarr-ai/00_intake/client-profile.md`
- `/rockstarr-ai/00_intake/style-guide.md` (explicitly approved)
- `/rockstarr-ai/01_knowledge_base/index.md` with at least some
  first-party processed files
- `/rockstarr-ai/00_intake/stack.md`

If the style guide is missing or unapproved, every drafting skill
must refuse to run and point the user back at
`rockstarr-infra:generate-style-guide`.

## Drafting rules (non-negotiable)

These rules apply to every drafting skill in this plugin. They mirror
the integrity rules that `generate-style-guide` enforces upstream.

1. **Style guide is canon.** Read `style-guide.md` front to back
   before drafting. Apply Brand Personality, Tone Definition (the
   "X, not Y" pairs), Style Rules, and Channel Adaptation literally.
   Banned language stays banned.
2. **First-party voice only.** Only files with
   `kb_scope: owned` AND `style_guide_eligible: true` may be used as
   voice signal. Third-party KB is reference-only — cite it, do not
   paraphrase it as if the client said it.
3. **Cite the style guide.** Every draft's front-matter records
   `style_guide_version` so `approve` can audit drift when the guide
   is rebuilt.
4. **Drafts land in `03_drafts/content/`.** Never write to
   `04_approved/` or `05_published/`. That is `approve` and
   `publish-log`'s job.
5. **Required front-matter:** every draft ships with
   `channel`, `title`, `produced_by`, `produced_at`,
   `style_guide_version`, `kb_sources_used`, `outline_source`
   (where applicable). `approve` reads these.
6. **Never invent positioning.** If the requested angle is not
   supported by the profile, the style guide, or the first-party KB,
   stop and ask the user to supply evidence — do not fabricate
   authority claims or case studies.
7. **No third-party paraphrase.** Third-party sources appear only in
   "References" sections with explicit attribution and a link.

## Folder contract (reads and writes)

```
rockstarr-content-bot reads:
  00_intake/client-profile.md
  00_intake/style-guide.md
  00_intake/stack.md
  01_knowledge_base/index.md
  01_knowledge_base/processed/**           (owned, style_guide_eligible)
  01_knowledge_base/processed/third-party/ (reference only)
  05_published/_publish.log                (avoid recent topic repeats)

rockstarr-content-bot writes:
  02_inputs/content-topics_<yyyy-mm-dd>.md (ideate-topics)
  03_drafts/content/outline_<slug>.md      (outline-blog)
  03_drafts/content/<slug>.md              (draft-blog, draft-article)
  03_drafts/content/<yyyy-mm-dd>_newsletter_<slug>.md (draft-newsletter)
```

All paths are relative to the client's `/rockstarr-ai/` root.

## Intended flow

```
ideate-topics               → content-topics_<date>.md in 02_inputs/
                              (user picks which to draft)

pick a blog topic
  → outline-blog            → outline_<slug>.md in 03_drafts/content/
                              (user approves outline)
  → draft-blog              → <slug>.md in 03_drafts/content/
  → rockstarr-infra:approve → 04_approved/<date>_blog_<slug>.md
  → ship via Publer/etc.
  → rockstarr-infra:publish-log

pick a newsletter topic
  → draft-newsletter        → <date>_newsletter_<slug>.md in 03_drafts/content/
  → approve → publish-log

pick a long-form article topic
  → draft-article           → <slug>.md in 03_drafts/content/
  → approve → publish-log
```

## Versioning

- `0.1.0` — initial cut (week of April 20, 2026). Five skills:
  `ideate-topics`, `outline-blog`, `draft-blog`, `draft-newsletter`,
  `draft-article`. No dedicated revise skill — re-run the relevant
  draft skill with feedback. No content-calendar / scheduling skill
  (that lives in `rockstarr-social-bot`). Blog is the only format
  with a mandatory outline gate; article and newsletter are
  single-shot.

## What this plugin does not do

- Does not post, schedule, or send. No channel integrations.
- Does not ideate social posts (that is `rockstarr-social-bot`).
- Does not generate outreach or reply copy (those are separate bot
  variants).
- Does not re-open the style guide. If the draft reveals a voice
  issue, run `rockstarr-infra:generate-style-guide` upstream.
- Does not approve its own drafts. Human review through
  `rockstarr-infra:approve` is the only path to `04_approved/`.
