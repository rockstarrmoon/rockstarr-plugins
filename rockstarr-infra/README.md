# rockstarr-infra

Rockstarr AI infrastructure plugin. This is the foundation every
Rockstarr bot sits on top of. It sets up the per-client folder layout,
ingests the intake workbook, generates the style guide, runs the
knowledge base, captures the client's tool stack, and handles the
approve / publish-log lifecycle that every bot terminates into.

This plugin is installed on **every** Rockstarr client marketplace.
Bot plugins (Content Bot, Social Bot, Outreach Bot variants, etc.)
depend on files produced by these skills.

## Skills

| Skill                       | Purpose                                                                                                                                                          |
|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `scaffold-client`           | Create `/rockstarr-ai/` folder layout, `client.toml` (with `[approvals]` block), and the daily-digest + weekly-backlog scheduled tasks.                          |
| `ingest-workbook`           | Parse `client-workbook.docx` → `client-profile.md`. **Legacy path.** Preserved for clients who arrived with a completed Rockstarr AI Playbook. New clients run the intake-interview flow instead. |
| `intake-icp`                | (v0.9.0) Two-phase ICP interview — Phase A captures the buyer profile, Phase B runs the Perception Gap exercise. Multi-ICP loop. Checkpoint-per-answer resume. Writes `00_intake/intake/icp.md`. |
| `intake-transformations`    | (v0.9.1) Five-stage Top Transformations exercise — ICP review, evidence pre-read, candidate list (5 to 10 with specificity push), three-question pressure-test per candidate, narrow to top 3 to 5. Optional Case studies + Quantifiable proof points subsections. Writes `00_intake/intake/transformations.md`. |
| `intake-competitors`        | (v0.9.2) Seven-stage Competition Crusher exercise — anchor, competitor selection (up to 3), per-competitor research via web fetch with paste-in fallback, competitive grid synthesis, five positioning artifacts (Brand Edge, Differentiation Summary, Messaging Opportunities, Risks, Quick Wins), Assumed-line validation pass, Key takeaways. Writes `00_intake/intake/competitors.md`; feeds both the Competitors section AND five subsections of the Positioning section of `client-profile.md`. |
| `intake-platinum-message`   | (v0.9.3) Three-stage Platinum Message synthesis — anchor, tone capture, per-ICP messaging loop with three pitch options + three outcome statements. Self-validates each draft against the four principles (Appeal / Exclusivity / Clarity / Credibility) and revises before presenting. Always emits one per-ICP H3 block. Writes `00_intake/intake/platinum-message.md`. |
| `intake-offer`              | (v0.9.4) Offer-by-offer capture in the canonical eight-field shape (Name, Who it's for, Problem, Outcome, Process, Experience, optional Edge, Why it works, Category-of-one positioning). Runs the Category-of-One check against `intake/competitors.md` after each offer with up to two sharpening rounds. No pricing by design. Writes `00_intake/intake/offer.md`. |
| `intake-background`         | (v0.9.5) Foundational intake step — runtime Step 1. Five stages: tech stack (delegates to `capture-stack`), Company description (3 questions + stop-slop synthesis), content library loop (URLs + paste-ins + drops, first-party vs. third-party), voice samples loop (samples never stop-slopped — they're the calibration signal), optional Goals/constraints turn. Writes `00_intake/intake/background.md` plus the two downstream-readable files `samples/content-library.md` and `samples/voice-notes.md`. |
| `run-intake`                | (v0.9.6) Stateful orchestrator for the intake flow. Owns `00_intake/intake/_progress.md`, handles the first-run workbook-or-interview path fork, dispatches to the six intake sub-skills in order with continue / redo / jump / stop / switch-paths controls, calls `compile-profile` once every sub-skill is complete. Switch-paths archives the prior path's artifacts to `99_archive/intake/<ISO>/` before resetting. Honors pause cleanly at every decision point. |
| `run-onboarding`            | (v0.9.7) Thin client-facing entry point. Chains `scaffold-client` (idempotent) into `run-intake`, then on intake completion surfaces `kb-ingest` and `generate-style-guide` as next-step prompts. No auto-chain past intake. Owns no state. Idempotent at every layer. The single skill new clients invoke after install. |
| `compile-profile`           | (v0.9.0) Deterministic assembler. Reads the six intake artifacts under `00_intake/intake/` and produces the canonical `00_intake/client-profile.md`. Archives prior versions to `99_archive/`. Replaces `ingest-workbook` on the interview path. |
| `generate-style-guide`      | Produce `style-guide.md` from profile + voice samples using the ported Rockstarr prompt.                                                                         |
| `kb-ingest`                 | Clean and tag raw knowledge-base files into `processed/` with a keyword index.                                                                                   |
| `capture-stack`             | Interview client on CRM / LI tool / website / scheduler / email / task system / analytics; capture content cadence (one question per content type), social channels including Google Business, Sales Nav outreach defaults including `outreach_campaign_mode` preference, and Publer label placeholders (rockstarr-social populates the real labels on first run). |
| `approve`                   | Promote a draft from `03_drafts/` to `04_approved/` with approval metadata.                                                                                      |
| `publish-log`               | Record a shipped output in `05_published/<channel>/` for metrics review.                                                                                         |
| `request-support`           | Draft + send a support email to `ai_support@rockstarrandmoon.com` on client approval.                                                                            |
| `approvals-digest`          | Daily 6 am cross-bot email to the client listing items pending approval, sorted most-recent first, with `claude://` deep-links per item. Silent on empty days.    |
| `approvals-backlog-alert`   | Weekly Monday 8 am email to the client's Rockstarr strategist when pending count exceeds `[approvals].strategist_alert_threshold` (default 25). Silent under it. |
| `notify-reply-ready`        | Urgent email to the client when an outreach reply lands and a draft is staged. Per-reply card with lead context, inbound excerpt, classification, drafted body, and `claude://` deep-link. Routed via `notify_type=urgent`. Called by outreach `detect-replies` (batched) or `rockstarr-reply:draft-reply` (single). |

## Folder contract

Every client repo produced and read by this plugin conforms to:

```
/rockstarr-ai/
├── client.toml
├── README.md
├── 00_intake/
│   ├── client-workbook.docx
│   ├── client-profile.md
│   ├── style-guide.md
│   ├── stack.md
│   └── samples/
├── 01_knowledge_base/
│   ├── raw/
│   ├── raw/third-party/
│   ├── processed/
│   ├── processed/third-party/
│   └── index.md
├── 02_inputs/
├── 03_drafts/
│   ├── content/
│   ├── social/
│   ├── outreach/
│   ├── replies/
│   └── nurture/
├── 04_approved/
│   └── _approvals.log
├── 05_published/
│   ├── _publish.log
│   └── <channel>/
├── 06_reports/
│   ├── weekly/
│   ├── monthly/
│   └── data/
└── 99_archive/
```

The structure is defined in Section 2.3 of the Rockstarr AI Growth
Operating System deployment model. Do not change it in this plugin
without coordinating a deployment-model version bump.

## Getting started — step-by-step

Welcome aboard. This is your walkthrough for getting the Rockstarr AI
Growth Operating System up and running in your Cowork workspace. By
this point, the Rockstarr team has already provisioned your private
marketplace and sent you your marketplace URL — so you're ready to
install.

Plan on about an hour of focused time to get through the first
install. The style-guide interview (step 6) is the longest piece and
benefits from not being rushed.

### Step 1 — Install the rockstarr-infra plugin

Open Cowork and go to **Plugin Settings**. Add the custom
marketplace URL the Rockstarr team sent you (it looks like
`https://github.com/rockstarr-moon/mkt-<your-token>`), authorize
GitHub when prompted, and flip on the **Sync updates** toggle so
you automatically receive improvements over time.

Once the marketplace is connected, install **rockstarr-infra**. This
is the foundation plugin — the bots we'll add later depend on it.

> Nothing else will be visible to install yet. Your bots will appear
> after intake completes and your strategist enables them.

### Step 1b — Allow the Rockstarr AI mailer

Rockstarr bots email you notifications — draft-ready pings,
approvals digests, urgent replies — through a Rockstarr-controlled
mail service at `mail.rockstarr.ai`. Cowork's sandbox
enforces an outbound network allowlist by default, so the mail
service needs to be explicitly permitted before anything can reach
it.

Open **Cowork → Settings → Capabilities** and add this host to
the Network Allowlist:

```
mail.rockstarr.ai
```

Save.

> **Important:** the allowlist is read once when a Cowork
> conversation starts. Your current chat was created *before* the
> change, so it can't see the new entry. **Close this conversation
> and start a new one in the same workspace before continuing.** From
> the new chat, everything in the rest of this walkthrough works
> normally.

If your Cowork plan is Team or Enterprise, your workspace admin may
need to make this change on your behalf. Ask them to allowlist
`mail.rockstarr.ai` in the workspace's Capabilities settings.

### Step 2 — Select your working folder

In Cowork, select (or create) the folder on your computer that will
hold all of your Rockstarr AI work. A folder named `RockstarrAI` on
your Desktop works well. Everything the plugin creates will live
inside this folder.

### Step 3 — Run onboarding

This is the main step. One command chains your folder scaffolding,
your intake interview, and the post-intake prompts for knowledge
base processing and style guide generation. In Cowork, ask Claude
to "run onboarding."

Claude does three things in order:

1. **Scaffolds your folder.** Asks you for a short `client_id`
   (lowercase, hyphens only — e.g. `acme-corp`) and your company
   display name, then builds the full `/rockstarr-ai/` folder
   structure with `client.toml` and a short README. Safe to re-run
   at any point.

2. **Walks you through intake.** Asks how you want to capture your
   business context:

   - **Workbook path** — if you have a completed Rockstarr AI
     Playbook (.docx) ready, upload it and Claude parses it
     directly into your profile.
   - **Interview path** — six guided exercises in chat
     (Background, ICP, Transformations, Competitors, Platinum
     Message, Offer). One question at a time. Pause anywhere;
     resume any time with "continue intake." Total time depends
     on ICP count: usually 1.5–3 hours, split across as many
     sessions as you want. Drop files and links into the
     conversation as they come up — Claude routes each one to
     the right destination without breaking flow.

   Either path produces the same `00_intake/client-profile.md`.

3. **Offers to continue with `kb-ingest` + `generate-style-guide`.**
   After intake completes, Claude surfaces these two skills as the
   natural next moves. Pick "yes, continue" to run them in
   sequence, or pause and run them later. Both are required before
   your drafting bots produce usable output, but neither is on a
   timer.

   - `kb-ingest` reads any content URLs captured during intake
     plus any files in `01_knowledge_base/raw/` and produces the
     cleaned, tagged processed library.
   - `generate-style-guide` reads your profile, samples, and
     processed KB, then walks a focused interview that produces
     `00_intake/style-guide.md` — the single voice document every
     drafting bot reads. Budget 30–45 minutes.

> To redo any single intake exercise later without re-running
> everything, ask Claude to "redo step N" (where N is 1 through 6:
> 1 Background, 2 ICP, 3 Transformations, 4 Competitors,
> 5 Platinum Message, 6 Offer).

### Step 4 — Review your intake artifacts

Before turning on the drafting bots, read through these three files
and fix anything that isn't right:

- `00_intake/client-profile.md`
- `00_intake/style-guide.md`
- `00_intake/stack.md`

Loop in your Rockstarr strategist if you want a second set of eyes.
Edits now are cheap; edits after bots start drafting cost you
content that doesn't sound like you.

### Step 5 — Bots get turned on

Once your intake is solid, let your Rockstarr strategist know. We
will enable your bot plugins in your marketplace on our side. When
they're ready, they'll show up in your Cowork Plugin Settings —
install them the same way you installed rockstarr-infra.

Your first-wave bundle typically includes:

- **rockstarr-content** — long-form drafting (blogs,
  newsletters, articles)
- **rockstarr-social** — social scheduling and short-form posts
- **rockstarr-outreach** — LinkedIn outreach (matched to your
  tool)
- **rockstarr-reply** — inbox handling (matched to Gmail or
  Outlook)
- **rockstarr-nurture** — CRM-driven nurture (matched to your
  CRM)
- **rockstarr-ops** — CRM operations automations (when
  applicable)

### Step 6 — Your day-to-day: draft, approve, publish

Once the bots are installed, your day-to-day cycle looks like this:

1. **Draft.** Ask a bot to ideate a topic, outline a post, draft a
   newsletter, generate social posts, and so on. Drafts land in
   `/rockstarr-ai/03_drafts/<channel>/`.
2. **Review.** Open the draft. Edit freely until it's ready.
3. **Approve.** When you're happy, say "approve this draft." Claude
   moves the file into `/rockstarr-ai/04_approved/<channel>/` and
   logs the approval.
4. **Publish.** After the post goes live, the email sends, or the
   LinkedIn message goes out, say "log that I published this."
   Claude records the send into `/rockstarr-ai/05_published/` so
   your weekly and monthly reports stay accurate.

### Ongoing — keep the knowledge base fresh

Whenever you produce new content or come across great reference
material, drop it into the right folder (`raw/` for yours,
`raw/third-party/` for reference) and ask Claude to "refresh the
knowledge base." The bots will immediately have access to it on
their next draft.

If your positioning shifts materially — a new offer, a new audience,
a rebrand — ask your Rockstarr strategist whether it's time to
regenerate the style guide.

### Need help?

If anything in this walkthrough gets stuck, or a bot does something
that surprises you, message your Rockstarr strategist. We're
watching the rollout closely and will jump in. Don't fight it —
that's what we're here for.

## Shared assets

- `skills/_shared/references/style-guide-prompt.md` — canonical
  Rockstarr Brand Voice and Style Architect prompt. (Currently lives
  under `skills/generate-style-guide/references/`; other bots that
  need voice rules should read from there.)
- `skills/_shared/references/case-study-prompt.md` — canonical
  Rockstarr case-study interview prompt. Referenced by
  `rockstarr-content:draft-case-study` and any future bot that
  produces case-study output. Do not fork — update in place.
- `skills/_shared/references/intake-interviewer-voice.md` —
  (v0.9.0) single source of truth for the unified voice and
  discipline every intake sub-skill follows. Covers the six
  rules (one-question-at-a-time, pre-draft → confirm / amend /
  reject / skip, checkpoint-per-answer, pause-as-first-class-verb,
  accept-drops-mid-flow, no-silent-fabrication), the artifact
  shape every intake-* skill writes, the provenance-comment
  format, and the tone examples that distinguish the unified
  voice from the original GPT registers it replaces. Read by
  `intake-icp` and every future intake sub-skill (background,
  transformations, competitors, platinum-message, offer). Do not
  fork — update in place.
- `skills/_shared/references/tl-rubric.md` — canonical
  thought-leadership rubric. Defines the three tests every TL
  piece must pass (single argument a smart competitor could
  disagree with, one specific story told end-to-end, one quotable
  line a reader would repeat), patterns to cut on sight, the
  structural rewrite checklist, the multi-article enemy-diversity
  standard, and the seven-test quick critique frame. Read by
  `rockstarr-content:outline-thought-leadership`,
  `rockstarr-content:draft-thought-leadership`, and
  `rockstarr-content:ideate-topics`. Do not fork — update in
  place.
- `skills/_shared/references/blog-seo-geo.md` — canonical blog
  SEO + GEO (Generative Engine Optimization) reference. Covers
  the GEO patterns AI search systems reward (cited stats, named
  sources, structured definitions, direct-answer-first,
  consistent terminology, specific numbers and proper nouns),
  the required FAQ section structure, keyword placement and
  density rules, internal linking rules, the inline `[Source: URL]`
  convention, meta title and meta description specs, and the
  13-item quality checklist drafts run as Pass 1 before
  stop-slop. Read by `rockstarr-content:outline-blog` (research
  phase, FAQ outline, keyword and internal-linking plans, meta
  drafts) and `rockstarr-content:draft-blog` (FAQ in body,
  inline sources, keyword density, direct-answer pattern,
  structured definitions, the quality checklist gate). Do not
  fork — update in place.
- `skills/_shared/stop-slop/` — MIT-licensed stop-slop skill. Every
  prose-producing Rockstarr bot must run this as the final pass before
  writing a draft to `03_drafts/`. Shared SKILL.md + `references/`
  (phrases, structures, examples); consumers wire it in, do not fork.
- `skills/_shared/send-notification/` — cross-bot helper that sends a
  single email from Rockstarr AI to the client (or their strategist)
  via the Rockstarr mailer at `mail.rockstarr.ai`. Used by
  `approvals-digest`, `approvals-backlog-alert`, `notify-reply-ready`,
  `request-support`, and any bot that needs to emit an out-of-band
  notification (draft ready, reply landed, digest rollup). Reads the
  bearer token from `/rockstarr-ai/00_intake/.rockstarr-mailer.env`
  (seeded by `scaffold-client`, filled in at onboarding). Do not
  bypass — drift here creates real inbox-deliverability risk.

## Customization

- Bot variant mapping (which outreach / reply / CRM bots to enable for
  a given stack) lives inside `capture-stack`. When a new tool variant
  is added, update the table in `skills/capture-stack/SKILL.md` AND
  publish the matching bot plugin to the marketplace.
- When porting a new Rockstarr custom GPT prompt into this plugin,
  follow the `style-guide-prompt.md` / `case-study-prompt.md` pattern:
  land the canonical text under `skills/_shared/references/` and have
  the consuming skill reference it.

## Versioning

- `0.7.0` — `request-support` skill plus shared `send-notification`
  helper and `.rockstarr-mailer.env` scaffolding. First version where
  bots can email the client out-of-band.
- `0.8.0` — Cross-bot approvals + urgent-reply layer.
  - New skill: `approvals-digest`. Daily 6 am client-bound email with
    a digest of every pending draft across every bot, sorted
    most-recent first by file mtime. Each item carries a
    `claude://cowork/new?q=...` deep-link that opens Claude Desktop
    with `Show me the draft at <path> for review.` pre-typed. Silent
    on empty days.
  - New skill: `approvals-backlog-alert`. Weekly Monday 8 am email
    to the client's Rockstarr strategist when pending count exceeds
    the configured threshold. Routed to `ROCKSTARR_STRATEGIST_EMAIL`
    with `reply_to` set to the client's mailbox so a reply lands with
    them directly.
  - New skill: `notify-reply-ready`. Urgent client-bound email when
    an outreach reply lands and a draft is staged. Per-reply card
    with lead context, inbound excerpt, classification, the drafted
    body rendered inline, and a `claude://` deep-link straight into
    the present-for-approval flow. Routes via `notify_type=urgent`
    and is called by outreach variants' `detect-replies` (batched
    across the daily run) or by `rockstarr-reply:draft-reply` for
    synchronous single-thread use.
  - New per-client config: `[approvals] strategist_alert_threshold`
    in `client.toml` (default 25). Seeded by `scaffold-client`.
  - `scaffold-client` now wires both recurring scheduled tasks
    (digest + backlog alert) via
    `mcp__scheduled-tasks__create_scheduled_task` at intake time.
    `notify-reply-ready` is event-driven, not scheduled — no cron
    wiring required.
  - **Mailer prerequisite:** rockstarr-mailer's `safeUrl()` must
    allow the `claude://` scheme for any of the three skills'
    deep-links to render. Patched in the mailer's matching release;
    the skills still function if the patch isn't deployed (file
    paths work without the link), the convenience layer just stays
    dark until it is.
  - **Caller integration outstanding:** `notify-reply-ready` is a
    shared infra skill but the wiring lives in the callers.
    `rockstarr-reply:draft-reply` and the outreach variants'
    `detect-replies` skills need an explicit invocation step
    appended to call this skill with the just-staged paths. Wire
    that in the matching plugin bumps before the urgent
    notification fires for real.
- `0.8.2` — Shared blog SEO + GEO reference.
  - **New shared reference:
    `skills/_shared/references/blog-seo-geo.md`** — canonical
    SEO and GEO standard for the researched-blog lane. Covers
    GEO patterns (cited stats, named sources, structured
    definitions, direct-answer-first, consistent terminology,
    specific numbers / dates / proper nouns), the required FAQ
    section, keyword placement and density rules, internal
    linking rules, the inline `[Source: URL]` convention, meta
    title and description specs, and the 13-item quality
    checklist that runs as Pass 1 before stop-slop.
  - **Read by rockstarr-content v0.4+** — `outline-blog` (adds
    a WebSearch research phase, FAQ outline, keyword placement
    plan, internal linking plan, meta drafts) and `draft-blog`
    (required FAQ in body, inline source URLs, direct-answer
    pattern, structured definitions, density tracking, quality
    checklist gate before stop-slop).
  - Single source of truth — do not fork.
  - Pure additive change. No skill behavior in this plugin
    changes; the only delta is the new file under
    `skills/_shared/references/`.
- `0.8.1` — Shared thought-leadership rubric.
  - **New shared reference:
    `skills/_shared/references/tl-rubric.md`** — canonical
    thought-leadership rubric. Defines the three tests every TL
    piece must pass (single argument a smart competitor could
    publicly disagree with, one specific story told end-to-end,
    one quotable line a reader would repeat at dinner), patterns
    to cut on sight, the structural rewrite checklist, the
    multi-article enemy-diversity standard, the seven-test quick
    critique frame, the "what compelling actually means"
    definition, and the dual-roles note for founder-as-writer vs.
    bot-as-writer.
  - **Read by rockstarr-content v0.3+** —
    `outline-thought-leadership` (forces the five required
    fields), `draft-thought-leadership` (post-draft rubric pass
    before stop-slop), and `ideate-topics` (enemy-diversity check
    across the month's TL slate).
  - Single source of truth — do not fork. Updates land here and
    propagate automatically to every consuming skill on the next
    plugin sync.
  - Pure additive change. No skill behavior in this plugin
    changes; the only delta is the new file under
    `skills/_shared/references/`.

## Backlog / future

(Empty — every entry that was here in 0.7 is now shipped in 0.8.x.)

## What this plugin does not do

- It does not draft content, post to LinkedIn, or run outreach. Those
  are the bot plugins' jobs. It does provide the `send-notification`
  helper bots use for out-of-band email to the client, but it does not
  decide when those notifications fire.
- It does not send email on behalf of the client to third parties
  (leads, teammates). That flows through `rockstarr-reply-<variant>`
  using the client's own Gmail or Outlook OAuth.
- It does not embed or vector-search the knowledge base. v1 is flat
  tagged markdown; embeddings are a v2 feature.
