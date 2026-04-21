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

| Skill                    | Purpose                                                                                  |
|--------------------------|------------------------------------------------------------------------------------------|
| `scaffold-client`        | Create `/rockstarr-ai/` folder layout and `client.toml`.                                 |
| `ingest-workbook`        | Parse `client-workbook.docx` → `client-profile.md`.                                      |
| `generate-style-guide`   | Produce `style-guide.md` from profile + voice samples using the ported Rockstarr prompt. |
| `kb-ingest`              | Clean and tag raw knowledge-base files into `processed/` with a keyword index.           |
| `capture-stack`          | Interview client on CRM / LI tool / website / scheduler / email / analytics.             |
| `approve`                | Promote a draft from `03_drafts/` to `04_approved/` with approval metadata.              |
| `publish-log`            | Record a shipped output in `05_published/<channel>/` for metrics review.                 |

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

## Setup

1. Install this plugin from the Rockstarr marketplace (it is enabled
   by default for every active client).
2. Run `scaffold-client` once per new client.
3. Drop the client's Getting Started Workbook into
   `/rockstarr-ai/00_intake/client-workbook.docx`.
4. Run `ingest-workbook`, then `generate-style-guide`, then
   `capture-stack`. This is the intake sequence.
5. As supporting materials arrive, drop them into
   `/rockstarr-ai/01_knowledge_base/raw/` (first-party) or
   `/rockstarr-ai/01_knowledge_base/raw/third-party/` (reference
   material) and run `kb-ingest`. You can also hand `kb-ingest` an
   https URL directly and it will fetch, clean, and tag the page.

After that, bot plugins take over drafting; `approve` and `publish-log`
are used by every bot's review/ship loop.

## Customization

- `skills/generate-style-guide/references/style-guide-prompt.md` is a
  placeholder. Paste the real Rockstarr custom GPT prompt here to
  finish v1 of the style-guide skill.
- Bot variant mapping (which outreach / reply / CRM bots to enable for
  a given stack) lives inside `capture-stack`. When a new tool variant
  is added, update the table in `skills/capture-stack/SKILL.md` AND
  publish the matching bot plugin to the marketplace.

## What this plugin does not do

- It does not install bot plugins. Installation happens by flipping
  entries in the client's marketplace KV record — see the
  `rockstarr-marketplace` repo and the deployment model Section 2.2.
- It does not draft content, post to LinkedIn, send email, or run
  outreach. Those are the bot plugins' jobs.
- It does not embed or vector-search the knowledge base. v1 is flat
  tagged markdown; embeddings are a v2 feature.

## Versioning

- `0.1.0` — initial v1 cut (first paying client, April 20, 2026).
  Seven skills; style-guide prompt placeholder; kb v1 flat markdown.
- `0.2.0` — naming tweaks per Jon's intake review (April 20, 2026):
  LI outreach default is Interceptly; Growth Amplifier slug is `ga`
  (never `gha`); never use "GHL". Added third-party KB scope with
  its own raw/ and processed/ directories, plus https URL ingestion
  via `WebFetch`. `generate-style-guide` now explicitly refuses
  third-party content as a voice signal.
- `0.3.0` — ported the Rockstarr Brand Voice and Style Architect
  custom GPT prompt into `generate-style-guide` as the canonical
  reference, and rewrote the skill as a hybrid flow: it pre-reads
  profile + samples + first-party kb, drafts answers to each
  interview question with confidence ratings, then walks the user
  through confirming / amending / filling gaps one question at a
  time before generating the guide in the GPT's fixed structure
  (Brand Context, Mission, Brand Approach, Brand Personality,
  Audience Definition, Tone Definition with contrast framing, Style
  Rules, Channel Adaptation, Tone Examples, Consistency
  Principles).
