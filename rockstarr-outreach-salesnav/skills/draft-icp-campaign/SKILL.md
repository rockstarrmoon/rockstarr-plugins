---
name: draft-icp-campaign
description: "This skill should be used when the user asks to \"draft a campaign\", \"write an outreach campaign\", \"draft the Sales Nav campaign\", \"turn this ICP into a campaign\", or names a campaign slug that has a lead-list pointer file at 02_inputs/outreach/lead-lists/. It reads the baseline ICP from 00_intake/client-profile.md, walks the user through a per-campaign clarification pass (adjustments apply ONLY to this campaign, never back to client-profile.md), writes the resolved per-campaign ICP to 02_inputs/outreach/icps/, then produces a campaign spec in 03_drafts/outreach/ with filter summary, four-message Sales Nav sequence (blank connect + three post-accept messages), cadence, exit conditions, and booking-flow notes. Cites first-party KB for proof points; third-party material is reference only. SOURCE OF TRUTH: this skill ships inline in rockstarr-outreach-salesnav for V0.1 and will migrate to rockstarr-infra/skills/_shared/ when the Interceptly variant lands so both plugins share one implementation."
---

# draft-icp-campaign

Turn a saved Sales Navigator search + the client's baseline ICP
(from `client-profile.md`) into an approved-ready campaign spec.
Produces two artifacts: a per-campaign ICP file and the campaign
spec. Output lives in `02_inputs/outreach/icps/` and
`03_drafts/outreach/`. The campaign spec routes through
`rockstarr-infra:approve`.

> **Shared-skill notice.** This is a shared skill. The canonical
> source will live at `rockstarr-infra/skills/_shared/draft-icp-campaign/`
> once both `rockstarr-outreach-salesnav` and `rockstarr-outreach-interceptly`
> exist. For V0.1 it ships inline in each plugin. If you edit this file,
> edit the matching copy in the other plugin in the same PR.

## When to run

- User says "draft a campaign from this saved search", names a slug
  that has `02_inputs/outreach/lead-lists/<slug>.md`, or points at
  any saved Sales Nav search URL and asks for a campaign.
- A new `lead-lists/<slug>.md` appears and the user wants the
  campaign drafted.

The per-campaign ICP file (`02_inputs/outreach/icps/<slug>.md`) is
NOT a precondition — this skill produces it. If it already exists
for the slug, this skill offers to reuse it or re-derive fresh from
`client-profile.md`.

## Preconditions

- `/rockstarr-ai/00_intake/client-profile.md` exists and has an ICP
  section. This is the baseline the campaign ICP derives from.
- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/stack.md` has the outreach keys set (see
  README). If `outreach_tool` is not `salesnav`, refuse.
- `02_inputs/outreach/lead-lists/<slug>.md` exists and contains a
  `saved_search_url` pointing at a Sales Nav saved search.

If any of the above is missing, stop and tell the user exactly
which file is missing and which upstream skill produces it.

## Inputs

Read, in this order:

1. `00_intake/client-profile.md` — the BASELINE ICP lives here.
   Firmographics, seniority, signals, pains, disqualifiers, anchor
   language. Treat as canonical.
2. `lead-lists/<slug>.md` — saved-search URL, any crawl notes,
   target lead count.
3. `02_inputs/outreach/icps/<slug>.md` — IF it already exists.
   Optional; prior per-campaign overrides live here.
4. `00_intake/style-guide.md` — full. Pay literal attention to:
   - **Brand Personality** + Do / Do Not behaviors
   - **Tone Definition** contrast pairs ("X, not Y")
   - **Style Rules** and banned phrases
   - **Channel Adaptation — LinkedIn / outreach** block
5. `00_intake/client-profile.md` again — offer names, audience
   phrasing, proof points, booking link URL.
6. First-party KB (`01_knowledge_base/processed/**` where
   `kb_scope: owned` AND `style_guide_eligible: true`) for proof
   points.
7. Third-party KB (`processed/third-party/`) for *industry context
   only*. Never paraphrase as if the client said it.

## Run order

This skill runs in two distinct phases. Do not collapse them — the
ICP clarification has to happen before drafting so the sequence copy
can reference the resolved audience.

1. **Phase 1 — Clarify the campaign ICP** (interactive, below).
2. **Phase 2 — Draft the campaign spec** (produces the draft file).

## Phase 1 — Clarify the campaign ICP

The goal is to give the user one chance to narrow or adjust the
baseline ICP for THIS campaign only. Never edit `client-profile.md`
from this skill.

### Step 1.1 — Handle prior per-campaign ICP (if any)

If `02_inputs/outreach/icps/<slug>.md` already exists:

- Show the user the existing file's audience summary + any
  overrides block.
- Use `AskUserQuestion` with options:
  - **Reuse as-is** — jump to Phase 2 with this file.
  - **Re-derive from client-profile.md** — archive the existing
    file to `99_archive/` and continue to Step 1.2.
  - **Edit specific fields** — jump to Step 1.3 with the existing
    file as the starting point.

### Step 1.2 — Present the baseline ICP

Read the ICP section of `client-profile.md`. Render it to the user
verbatim in chat:

```
From client-profile.md, your baseline ICP is:

<exact block copied from client-profile.md>
```

Immediately follow with `AskUserQuestion`:

> Is this the exact audience this campaign should target, or do you
> want to narrow / adjust it for THIS campaign only? (Adjustments
> apply to this campaign's ICP file, not to client-profile.md.)

Options:

- **Use as-is** — proceed to Phase 2.
- **Narrow for this campaign** — go to Step 1.3.
- **Add disqualifiers only** — go to Step 1.3 (disqualifiers-only
  sub-flow).
- **Swap the audience entirely** — warn the user that a new
  audience should probably get its own slug; confirm before
  continuing to Step 1.3.

### Step 1.3 — Walk the adjustments (one question at a time)

Use `AskUserQuestion` to walk these dimensions, ONE AT A TIME, only
asking about the dimensions the user flagged for adjustment. Do not
batch. Do not try to infer answers the user didn't give.

Dimensions (use the user's own language from the baseline wherever
possible):

- Firmographics (industry, sub-industry, company size band,
  geography)
- Seniority / titles to include
- Seniority / titles to exclude
- Buying signals to require (funding, hiring, tech-stack, recent
  event)
- Disqualifiers (do-not-contact lists, blocked companies, titles)
- Pain focus for this campaign (which of the profile's pains is the
  anchor for the sequence — pick exactly one; see Sequence rules)
- Anchor language — confirm or rewrite the exact phrase for the
  central pain + ceiling (see Rule 4 in Sequence rules)

After each answer, summarize back to the user and ask confirm /
amend before moving on.

### Step 1.4 — Write the per-campaign ICP file

Write `/rockstarr-ai/02_inputs/outreach/icps/<slug>.md`. If a prior
file existed and the user chose "Re-derive," move the old file to
`/rockstarr-ai/99_archive/icps/<slug>-<YYYY-MM-DD>.md` first. Never
overwrite without archiving.

Front-matter:

```yaml
# ---
campaign_slug: "kebab-case-slug"
derived_from: "00_intake/client-profile.md"
derived_at: "ISO timestamp"
produced_by: "rockstarr-outreach-salesnav/draft-icp-campaign@0.1.2"
scope: "campaign-only"
authority_note: "This file governs only this campaign. It does NOT override 00_intake/client-profile.md. Update the profile via rockstarr-infra:ingest-workbook + approve."
overrides:
  - "narrowed industries from [A, B, C] to [A, B]"
  - "added disqualifier: companies under 50 employees"
  - "anchor phrase: 'pipeline gets attention only when there's time, which is never'"
# ---
```

Body sections:

```markdown
# Campaign ICP: <Campaign title>

_Derived from client-profile.md on <date>. Adjustments below apply
only to this campaign._

## Audience (for this campaign)
<Firmographics, seniority, signals — final, post-adjustment.>

## Pain focus (one)
<The single pain this campaign's sequence speaks to. The anchor
phrase below is the verbatim shorthand.>

## Anchor phrase
<The exact verbatim phrase that will appear in Messages 2, 3, and 4
of the sequence. See Sequence Rule 4.>

## Disqualifiers
<Companies, titles, geographies this campaign will not contact.>

## Overrides vs. baseline
<Bulleted diff from client-profile.md for audit.>
```

Then tell the user:

> Per-campaign ICP written to
> `02_inputs/outreach/icps/<slug>.md`. This file governs this
> campaign only. Your `client-profile.md` is unchanged. Moving on
> to draft the campaign spec now.

Proceed to Phase 2.

## Phase 2 — Draft the campaign spec

Write to `/rockstarr-ai/03_drafts/outreach/campaign-<slug>.md`.

Front-matter:

```yaml
# ---
channel: "outreach-salesnav"
stage: "draft"
campaign_slug: "kebab-case-slug"
icp_source: "02_inputs/outreach/icps/<slug>.md"
lead_list_source: "02_inputs/outreach/lead-lists/<slug>.md"
saved_search_url: "https://www.linkedin.com/sales/search/..."
target_lead_count: 250
cadence: "day-of-accept, accept+3, accept+7"
daily_send_cap_contribution: "round-robin share of 20/day + 100/week"
produced_by: "rockstarr-outreach-salesnav/draft-icp-campaign@0.1.2"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md"
anchor_phrase: "<verbatim anchor phrase from per-campaign ICP>"
kb_sources_used:
  - "processed/founder-call-2026-03.md"
third_party_references: []
booking_mode: "automated | manual (from stack.md)"
availability_source: "booking_link | gcal (from stack.md)"
# ---
```

Body sections (in order):

```markdown
# Campaign: [Working title — ICP phrase, not an ad line]

## Filter summary
Plain-English restatement of the Sales Nav saved search. Who is in.
Who is out. Why this cut, in one paragraph.

## Target ICP
Summary pulled from `02_inputs/outreach/icps/<slug>.md`. Do not
invent. One-line audience + 3–5 signals that matter.

## Pain focus + anchor phrase
The single pain this campaign speaks to (one). The anchor phrase
(verbatim). Note: Rule 4 requires the anchor phrase to appear
word-for-word in Messages 2, 3, and 4.

## Why now
The reason this ICP is hearing from the client right now, grounded in
first-party KB. No third-party paraphrase.

## Sequence

The four-message structure below is the canonical Rockstarr Sales
Nav cadence. Each message has a specific, narrow purpose. Do not
collapse them, do not reorder them, do not rewrite them to "add
value" or "sound more confident." The purpose notes AND the
Sequence rules (hard) block below are hard constraints, not vibes.

### Message 1 — Connect request (day of send)

**Body: BLANK.**

Leave the connect-request note empty on purpose. LinkedIn's default
behavior is a cleaner first touch than any custom note, and an
empty note keeps Message 1 from eating into the post-accept
conversation space. Do not write a connect note. Do not paste a
pitch. Do not "introduce yourself briefly." Blank is the spec.

### Message 2 — Curiosity opener (day of accept)

**Purpose.** Earn a reply on its own. This is the curiosity opener,
not a placeholder or a "warm up" for Message 3.

The body must:
- Acknowledge the connection briefly.
- Drop the anchor phrase verbatim in the curiosity line (see
  Sequence Rule 4).
- Mention that something new or relevant to the anchor is in motion
  (kept vague, no product name, no offer name, no positioning).
- Stay neutral and low effort.
- Stand on its own as a reason for the reader to reply.

The body must NOT:
- Ask a direct question (soft curiosity is fine; "what do you think
  about X?" is not).
- Pitch anything.
- Include a call to action.
- Include a positioning statement.
- Use hype words, superlatives, or energy-raising language.
- Say "follow up" in any form (see Sequence Rule 1).
- Announce the absence of a pitch (see Sequence Rule 3).

Keep it short. Two or three sentences. Under ~400 characters.

### Message 3 — First real touch (accept + 3 days)

**Purpose.** Invite engagement without pressure.

The body must:
- Reference what the client is exploring or launching, tied to the
  anchor phrase (verbatim).
- Share a short observation or pattern from first-party KB.
- Ask one soft, non-leading question.
- Ask permission to continue (e.g., "fine to share more?" / "OK if
  I send a quick note later?").

The body must NOT:
- Pitch, name-drop customers, or quote stats without a first-party
  source.
- Ask a hard-commit question (no calendar asks, no "can we hop on a
  call?").
- Use leading questions that assume the pain.
- Use the phrase "following up" or any variant.
- Announce that this is "not a pitch" or "not an ask."

Keep it conversational and neutral. Short paragraphs.

### Message 4 — Ceiling close (accept + 7 days)

**Purpose.** Close the loop respectfully by naming the ceiling that
keeps the pain permanent, not the pain itself.

The body must:
- Name the ceiling verbatim using the anchor phrase.
  Example shape: "Pipeline gets attention only when there's time,
  which is never." The anchor phrase from the ICP goes here,
  unchanged.
- Acknowledge they are likely busy, briefly.
- Leave the door open without asking for a reply.
- Be brief, calm, and final in tone.

The body must NOT:
- Close with well-wishes ("good luck with the quarter," "wishing
  you a great Q2," "best of luck out there"). No quarter-signoff.
- Guilt them for not responding.
- Chase ("just bumping this," "circling back," "one more try").
  Any variant of "following up" is banned per Sequence Rule 1.
- Restate your value proposition.
- Push for a reply.
- State the pain instead of the ceiling. The pain is the surface
  hurt; the ceiling is the structural reason the pain stays.

Short. Three or four sentences. Reads like a polite exit with a
sharp structural observation, not a last-ditch pitch or a greeting
card.

> Note: the campaign's official 3-step post-accept sequence is
> Message 2 + Message 3 + Message 4 above. Message 1 is the connect
> request (intentionally blank). The "3-step" sequence count in
> front-matter refers to the post-connect messages.

## Sequence rules (hard)

These apply to every message body in the sequence (Messages 2, 3,
4). They do NOT apply to the surrounding spec prose — em dashes in
the Filter Summary are fine; em dashes in a message body are not.

1. **Never write "following up."** Not anywhere, not in any form.
   "Following up," "just following up," "circling back," "one more
   follow-up," "quick follow" — all banned. If the intent is a
   second touch, just send the message. Naming the touch is the
   smell.
2. **No em dashes in the sequence.** Use commas, periods, or a line
   break. Em dashes read as written, not written-for-LinkedIn.
3. **Never announce the absence of a pitch.** "No pitch," "not
   pitching," "this isn't a sales ask," "no agenda," "not an ask" —
   all banned. Announcing it signals it.
4. **Anchor language stays verbatim.** The exact anchor phrase from
   the per-campaign ICP must appear word-for-word in Messages 2, 3,
   and 4. The recognition hit depends on the phrase repeating
   exactly. Do not rephrase it, do not synonym-swap, do not
   paraphrase for variety.
5. **Message 2 is the curiosity opener, not a placeholder.** It has
   to earn a reply on its own, independent of Messages 3 and 4. If
   the only reason Message 2 exists is to tee up Message 3, rewrite
   it. The anchor phrase is how it earns the reply.
6. **Message 4 names the ceiling, not the pain.** Shape:
   "Pipeline gets attention only when there's time, which is never."
   That's a ceiling — the structural constraint that keeps the pain
   from going away. The pain itself ("we're missing quota") is not
   what Message 4 says. Do not close with well-wishes like "good
   luck with the quarter." The ceiling is the close.

## Booking flow
- `booking_mode: automated` → `propose-meeting-times` is called when
  the thread reaches a meeting-ask moment. Bot books via the booking
  link.
- `booking_mode: manual` → bot proposes times from Google Calendar;
  client books manually and calls `mark-booked`.
- Never include the booking link in message copy. Times get proposed
  in the reply, not linked.

## Exit conditions
- Lead replies → thread routes to `rockstarr-reply`.
- Lead books → `mark-booked` fires, all pending tasks cancelled.
- Lead replies with opt-out signal → classification `not-interested`;
  future sequence tasks cancelled.
- Sequence complete with no reply → lead marked `queued` for possible
  nurture hand-off (out of scope V0.1; leave as-is).

## Daily-send-cap contribution
Round-robin share of the global 20/day + 100/week cap. This campaign
does not carry its own cap — `daily-connect` manages the global one.

## Open questions for the client
(Only if there are gaps. Mark inline `[CLIENT TO CONFIRM]` in copy
for any specific claim that cannot be traced to the profile or
first-party KB.)
```

## Drafting rules

1. **Style guide is canon.** Re-read the Tone Definition contrast
   pairs after drafting. Every sentence must sit on the X side.
2. **First-party voice only.** All claims, proof points, and specific
   phrasing must trace to `client-profile.md`, the per-campaign ICP,
   or a first-party KB file in `kb_sources_used`. Third-party
   appears only as *industry context* in "Why now," never as the
   client's voice.
3. **No invented proof.** No case studies, customer names, revenue
   figures, or dates the client has not already published.
   `[CLIENT TO CONFIRM]` for gaps.
4. **Sales Nav character limits + Message 1 rule.** Message 1
   (connect request) is intentionally BLANK. Never fill it in, even
   if the ICP screams for it. Keep Messages 2–4 short; this is
   LinkedIn, not an email sequence. Message 2 under ~400 characters.
   Messages 3 and 4 short paragraphs, never more than a short scroll
   on mobile. Respect the Sequence rules (hard) block above for
   banned wording.
5. **No booking link in copy.** Times are proposed later by
   `propose-meeting-times` and injected into reply drafts by
   `rockstarr-reply`. Never paste the booking URL into a sequence
   body.
6. **One campaign, one ICP, one saved search.** If the ICP implies
   two audiences, say so and ask the user to split the campaign into
   two slugs.
7. **Self-check pass before writing.** After drafting Messages 2, 3,
   4, run a literal search-and-check:
   - Anchor phrase appears verbatim in all three? If no, rewrite.
   - "Following up" / "follow up" / "circling back" / "bumping" —
     zero hits across Messages 2–4? If any hit, rewrite.
   - Em dash in any message body? If yes, rewrite.
   - "No pitch" / "not pitching" / "not an ask" — zero hits? If any
     hit, rewrite.
   - Message 4 ends with a ceiling statement, not "good luck"? If
     no, rewrite.

## After writing

Print a summary in chat: slug, target lead count, sequence cadence,
the anchor phrase (verbatim), any `[CLIENT TO CONFIRM]` markers, and
the first-party sources cited. End with:

> Campaign draft landed at `03_drafts/outreach/campaign-<slug>.md`.
> Per-campaign ICP at `02_inputs/outreach/icps/<slug>.md`.
> Review and either edit in place, ask for a revised pass, or run
> `rockstarr-infra:approve` when it is ready. `register-campaign`
> only runs against an approved campaign file.

Do not call `approve` yourself. Human review is the gate.

## What NOT to do

- Do not run without `client-profile.md` and the lead-list pointer
  file.
- Do not modify `00_intake/client-profile.md` from this skill. Ever.
  The per-campaign ICP lives at `02_inputs/outreach/icps/<slug>.md`
  and is scoped to this campaign only.
- Do not skip the Phase 1 clarification step. Even if the baseline
  ICP looks perfect, ask — the user should get one conscious beat
  to confirm or narrow.
- Do not batch the clarification questions. One dimension per
  `AskUserQuestion` call, with a confirm-back before the next.
- Do not include the booking link in message copy.
- Do not paraphrase third-party sources.
- Do not seed leads, crawl the saved search, or write to the
  workbook from this skill. Those are `register-campaign` and
  `crawl-lead-list` jobs.
- Do not move the campaign file to `04_approved/`. That is
  `approve`'s job.
- Do not violate any Sequence rule (hard). Those rules are
  non-negotiable and outrank any stylistic impulse.
