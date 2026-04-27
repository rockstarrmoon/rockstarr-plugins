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
| `ingest-workbook`           | Parse `client-workbook.docx` → `client-profile.md`.                                                                                                              |
| `generate-style-guide`      | Produce `style-guide.md` from profile + voice samples using the ported Rockstarr prompt.                                                                         |
| `kb-ingest`                 | Clean and tag raw knowledge-base files into `processed/` with a keyword index.                                                                                   |
| `capture-stack`             | Interview client on CRM / LI tool / website / scheduler / email / analytics.                                                                                     |
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
`https://github.com/rockstarrmoon/mkt-<your-token>`), authorize
GitHub when prompted, and flip on the **Sync updates** toggle so
you automatically receive improvements over time.

Once the marketplace is connected, install **rockstarr-infra**. This
is the foundation plugin — the bots we'll add later depend on it.

> Nothing else will be visible to install yet. Your bots will appear
> after we've captured your tool stack in step 5.

### Step 1b — Allow the Rockstarr AI mailer

Rockstarr bots email you notifications — draft-ready pings,
approvals digests, urgent replies — through a Rockstarr-controlled
mail service at `mail.rockstarrandmoon.com`. Cowork's sandbox
enforces an outbound network allowlist by default, so the mail
service needs to be explicitly permitted before anything can reach
it.

Open **Cowork → Settings → Capabilities** and add this host to
the Network Allowlist:

```
mail.rockstarrandmoon.com
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
`mail.rockstarrandmoon.com` in the workspace's Capabilities settings.

### Step 2 — Select your working folder

In Cowork, select (or create) the folder on your computer that will
hold all of your Rockstarr AI work. A folder named `RockstarrAI` on
your Desktop works well. Everything the plugin creates will live
inside this folder.

### Step 3 — Scaffold the folder layout

In Cowork, ask Claude to "scaffold the Rockstarr client folder." It
will ask you for a short `client_id` (lowercase, hyphens only — e.g.
`acme-corp`) and your company display name. Claude will then build
out the full folder structure, a `client.toml` with your info, and a
short README inside the folder.

> This step is safe to re-run at any point if folders ever get
> deleted or moved.

### Step 4 — Drop in your Getting Started Workbook

Before going further, make sure you've completed your Rockstarr AI
Playbook / Getting Started Workbook (the `.docx` your Rockstarr
strategist sent you). Save your completed copy as:

```
/rockstarr-ai/00_intake/client-workbook.docx
```

Then ask Claude to "ingest the workbook." It will read the document
and turn it into three files you'll use throughout the system:

- `client-profile.md` — your business, audience, and positioning
- `stack.md` (first pass) — the tools you told us you use
- `00_intake/samples/content-library.md` — voice samples extracted
  from the workbook

Take a few minutes to skim `client-profile.md` and make sure it
reflects you accurately. Everything downstream uses it as the
source of truth.

### Step 5 — Capture your tool stack

Ask Claude to "capture the stack." You'll answer a short interview
about which tools you use for:

- CRM (The Growth Amplifier, HubSpot, Pipedrive, Salesforce, Close)
- LinkedIn outreach (Interceptly, MeetAlfred, Dripify, Waalaxy,
  Sales Nav)
- Website platform
- Social scheduler (Publer is our default)
- Email platform (Gmail or Outlook)
- Analytics

This writes the authoritative `00_intake/stack.md` and tells us
which specific bot variants to turn on for you. Send the list back
to your Rockstarr strategist — that's how we enable the right bots
in your marketplace.

### Step 6 — Seed your knowledge base (optional but highly recommended)

Before the style-guide interview, give Claude some raw material to
learn your voice from. The stronger the inputs here, the better
every draft that follows.

Drop content you have created or own the rights to into:

```
/rockstarr-ai/01_knowledge_base/raw/
```

Good things to include: existing blog posts, podcast transcripts,
keynote or webinar decks, website copy, thought-leadership essays.

If you have reference material you want the bots to be aware of but
**not** imitate — competitor posts, industry research, articles you
found useful — drop those into `raw/third-party/` instead. We keep
these scopes strictly separate so your voice never gets diluted by
someone else's.

You can also give Claude a URL directly (e.g. "ingest this post:
https://...") and it will fetch and clean it automatically.

Once the files are in place, ask Claude to "ingest the knowledge
base."

### Step 7 — Build your style guide

This is the most important step in the whole setup, and the one
that earns the longest block of your time (budget 30–45 minutes).

Ask Claude to "generate the style guide." It will:

1. Pre-read your profile, samples, and first-party knowledge base,
   and draft a first pass at every interview question — each marked
   HIGH, MEDIUM, or LOW confidence — so you're never starting from
   a blank page.
2. Walk you through each question one at a time. You can confirm
   Claude's pre-draft, amend it, reject it outright, or skip ahead.
   The interview always runs in full, even when the pre-reads are
   strong — this is where your voice gets made.
3. Ask you to approve a short (≤120-word) positioning paragraph
   before producing the final guide.
4. Produce `00_intake/style-guide.md`, your authoritative voice
   document, covering brand context, mission, brand approach,
   personality, audience, tone, style rules, channel adaptation,
   tone examples, and consistency principles.

> Your style guide is the single reference every bot uses when
> drafting. Take the time to get it right.

### Step 8 — Review your intake artifacts

Before turning on the drafting bots, read through these three files
and fix anything that isn't right:

- `00_intake/client-profile.md`
- `00_intake/style-guide.md`
- `00_intake/stack.md`

Loop in your Rockstarr strategist if you want a second set of eyes.
Edits now are cheap; edits after bots start drafting cost you
content that doesn't sound like you.

### Step 9 — Bots get turned on

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

### Step 10 — Your day-to-day: draft, approve, publish

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
- `skills/_shared/stop-slop/` — MIT-licensed stop-slop skill. Every
  prose-producing Rockstarr bot must run this as the final pass before
  writing a draft to `03_drafts/`. Shared SKILL.md + `references/`
  (phrases, structures, examples); consumers wire it in, do not fork.
- `skills/_shared/send-notification/` — cross-bot helper that sends a
  single email from Rockstarr AI to the client (or their strategist)
  via the Rockstarr mailer at `mail.rockstarrandmoon.com`. Used by
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
  - **New shared reference:
    `skills/_shared/references/tl-rubric.md`** — canonical
    thought-leadership rubric. Read by rockstarr-content's
    `outline-thought-leadership`, `draft-thought-leadership`, and
    `ideate-topics` skills (rockstarr-content v0.3+). Defines
    the three tests, patterns to cut, structural rewrite
    checklist, enemy-diversity standard, and quick critique
    frame. Single source of truth — update in place, don't fork.

## Backlog / future

(Empty — every entry that was here in 0.7 is now shipped in 0.8.)

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
