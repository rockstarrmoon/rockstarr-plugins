---
name: draft-icp-campaign
description: "This skill should be used when the user asks to \"draft a campaign\", \"write an outreach campaign\", \"draft the Interceptly campaign\", \"turn this ICP into a campaign\", or names a campaign slug with an ICP spec at 02_inputs/outreach/icps/. Reads the baseline ICP from 00_intake/client-profile.md and the qualification rules from 00_intake/icp-qualifications.md, walks the user through a per-campaign clarification pass (tightens rules for this campaign only — never writes back to icp-qualifications.md), then produces a campaign spec in 03_drafts/outreach/ with filter summary, 3-step Interceptly sequence (connect note + 2 post-accept messages), cadence, exit conditions, and optional per-campaign ICP overrides. Every piece of message copy runs through stop-slop before the draft saves. Shared skill; canonical source will migrate to rockstarr-infra/skills/_shared/ when that tree exists."
---

# draft-icp-campaign

Shared skill. Canonical source will live at
`rockstarr-infra/skills/_shared/draft-icp-campaign/` once
`rockstarr-outreach-salesnav` and `rockstarr-outreach-interceptly`
both ship. For V0.1 it ships inline in each plugin; if you edit
this file, edit the matching copy in the other plugin in the same
PR.

Produces two artifacts: a per-campaign ICP file and a draft
campaign spec. The spec routes through `rockstarr-infra:approve`.

## When to run

- User says "draft a campaign", names a slug that has
  `02_inputs/outreach/icps/<slug>.md`, or supplies an ICP inline
  and asks for a campaign.
- A new `icps/<slug>.md` file appears and the user wants the
  campaign drafted.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists with an ICP
  section.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/icp-qualifications.md` exists (produced
  by `capture-icp-qualifications`). The baseline qualification
  rules are the default for this campaign.
- `/rockstarr-ai/00_intake/stack.md` has `outreach_tool =
  interceptly`. If the tool is anything else, refuse and point at
  the right variant.
- `rockstarr-infra:stop-slop` is available. This skill is
  mandatory on every piece of message copy. If `stop-slop` is not
  discoverable, refuse and tell the user `rockstarr-infra` needs
  to be installed or updated.

If any of the above is missing, stop and tell the user which file
is missing and which upstream skill produces it.

## Inputs

Read, in this order:

1. `00_intake/client-profile.md` — BASELINE ICP: firmographics,
   pains, anchor language, disqualifiers, proof points.
2. `00_intake/icp-qualifications.md` — the client's own
   target / not-target / ambiguous rules (set by
   `capture-icp-qualifications`). Baseline for this campaign.
3. `02_inputs/outreach/icps/<slug>.md` — IF it already exists.
   Per-campaign ICP + any prior tightening overrides.
4. `00_intake/style-guide.md` — read in full. Pay attention to:
   - Brand Personality + Do / Do Not behaviors
   - Tone Definition contrast pairs ("X, not Y")
   - Style Rules and banned phrases
   - Channel Adaptation → LinkedIn block
   - Channel Adaptation → LinkedIn replies — warm-ICP pattern
     subsection (written by `capture-warm-reply-pattern`)
5. `00_intake/interceptly-accounts.md` — all managed accounts;
   campaigns can be scoped to one account or run across all.
6. First-party KB (`01_knowledge_base/processed/**` where
   `kb_scope: owned` AND `style_guide_eligible: true`) for proof
   points.
7. Third-party KB (`processed/third-party/`) for industry context
   ONLY. Never paraphrase third-party as if the client said it.

## Run order

Two phases. Do not collapse them.

1. **Phase 1 — Clarify the campaign ICP and account scope.**
2. **Phase 2 — Draft the campaign spec.**

## Phase 1 — Clarify

### Step 1.1 — Handle prior ICP file

If `02_inputs/outreach/icps/<slug>.md` already exists, use
`AskUserQuestion`:

- `Reuse as-is` — jump to Phase 2.
- `Tighten for this campaign` — continue to 1.2.
- `Re-derive from baseline` — archive to `99_archive/` and
  continue to 1.2.

### Step 1.2 — Per-campaign tightening

Offer tightening options only — the baseline
`icp-qualifications.md` rules can be NARROWED for a single campaign
but not LOOSENED. Present fields from `icp-qualifications.md` and
ask for per-campaign overrides:

- Narrow the target role set? (show baseline list, let the client
  drop or add specifics)
- Narrow company size / stage? (show baseline, let the client
  tighten)
- Add an industry carve-out? (only certain industries even though
  baseline allows several)
- Add a disqualifier specific to this campaign? (e.g., competitors
  of the target account)

Every answer is an ADDITIVE tightening. Write the overrides block
into the per-campaign ICP file — do NOT touch
`icp-qualifications.md`.

### Step 1.3 — Account scope

Options: `Run across all managed accounts` /
`Scope to one managed account`. If scoped, use a follow-up
`AskUserQuestion` with the managed accounts as options.

A campaign scoped to one account is configured inside Interceptly
only for that account. A campaign across all accounts is
configured per-account during `launch-campaign-interceptly` — one
Interceptly campaign per managed account, sharing the same spec.

### Step 1.4 — Write the per-campaign ICP file

Write `/rockstarr-ai/02_inputs/outreach/icps/<slug>.md`:

```markdown
---
slug: <slug>
derived_from: icp-qualifications.md (baseline)
scope: all_accounts | <account_label>
overrides: true|false
generated_at: <ISO>
---

# <Campaign name>

## Baseline qualification rules

(From `/00_intake/icp-qualifications.md`. Do not edit here —
edit there and re-run this skill.)

## Per-campaign tightening

<additive narrowings from Step 1.2 — empty if none>

## Account scope

<all_accounts | specific account_label>

## Notes

<any free-text context from the client>
```

## Phase 2 — Draft the campaign spec

### Step 2.1 — Filter summary

Describe the Interceptly filter criteria the campaign will use:
title filters, seniority filters, company size, industry, location,
excluded industries / companies. This is human-readable — the
person who runs `launch-campaign-interceptly` will translate it
into Interceptly's filter UI.

### Step 2.2 — 3-step sequence

Write three messages in the client's approved voice. Interceptly's
sequence shape is connect note + two post-accept follow-ups.

1. **Connect note** (≤200 chars, Interceptly's connect-note limit).
   Personal, specific, references one signal from the filter
   criteria. No links. No emojis unless the style guide blesses
   them.
2. **Follow-up #1** (after accept; default day+3). One-line value
   + soft qualifier. Not a pitch. Reads like the client's warm-ICP
   pattern (captured in style-guide.md) — same shape as a reply,
   since the lead just accepted and a reply is the natural next
   move.
3. **Follow-up #2** (default day+7 from accept, day+4 from
   follow-up #1). Slightly more direct. One proof point from
   first-party KB. Offers a specific next step — NOT the booking
   link.

Run `stop-slop` on each of the three bodies before writing the
draft. stop-slop runs AFTER style-guide voice shaping, never
before — the order is voice first, AI-tells-removed last.

### Step 2.3 — Cadence + exit conditions

- Default cadence: `day 0 connect`, `day +3 follow-up #1`, `day +7
  follow-up #2`. Overridable inline.
- Exit conditions: any reply (handed to the per-reply pipeline),
  accept without reply (enters follow-up schedule normally),
  decline / no-response after follow-up #2 (Interceptly marks
  inactive, bot does nothing).

### Step 2.4 — Per-campaign qualification override note

If Step 1.2 added overrides, reference them in the spec so the
approver can see the narrowing at a glance. Do not restate the
full baseline — point at `icps/<slug>.md`.

### Step 2.5 — Write the draft

Write `/rockstarr-ai/03_drafts/outreach/campaign-<slug>.md`:

```markdown
---
slug: <slug>
status: draft
scope: all_accounts | <account_label>
icp_file: 02_inputs/outreach/icps/<slug>.md
generated_at: <ISO>
schema_version: 1
---

# Campaign: <name>

## Filter summary

<Step 2.1 output>

## Sequence

### Step 1 — Connect note (day 0)

<post-stop-slop body>

### Step 2 — Follow-up #1 (day +3 after accept)

<post-stop-slop body>

### Step 3 — Follow-up #2 (day +7 after accept)

<post-stop-slop body>

## Cadence

<Step 2.3 details>

## Exit conditions

<Step 2.3 details>

## Per-campaign qualification overrides

<reference or 'none'>
```

### Step 2.6 — Route to approve

Tell the user the draft is written. To launch, they need to:

1. Run `rockstarr-infra:approve` on this file. It moves the file
   to `/04_approved/outreach/campaign-<slug>.md`.
2. Run `launch-campaign-interceptly` to configure the campaign in
   Interceptly's UI. The bot will stop at CONFIGURED, NOT STARTED;
   the human operator presses Start.

## Outputs

- `/rockstarr-ai/02_inputs/outreach/icps/<slug>.md`
- `/rockstarr-ai/03_drafts/outreach/campaign-<slug>.md`

## Failure modes

- **Baseline ICP missing.** Refuse; point at
  `capture-icp-qualifications`.
- **Client asks to loosen baseline rules for this campaign.**
  Refuse. Per-campaign overrides are additive tightenings only.
  Loosening requires re-running `capture-icp-qualifications`.
- **stop-slop unavailable.** Refuse. Do not ship prose that has
  not been scrubbed.
- **Client asks to paste the booking link into a sequence step.**
  Refuse. The booking link is a destination, not content. Repeat
  this rule in the chat reply so the user understands it's a
  design decision.

## What NOT to do

- Do not edit `icp-qualifications.md` from this skill. Per-campaign
  tightening lives in the campaign-specific ICP file only.
- Do not run `stop-slop` BEFORE style-guide voice shaping. Order is
  voice first, AI tells last.
- Do not write to `/04_approved/` directly. Drafts go to
  `/03_drafts/` and route through `rockstarr-infra:approve`.
- Do not auto-start the Interceptly campaign from here. Campaign
  start stays a human action.
