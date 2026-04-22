---
name: draft-icp-campaign
description: "This skill should be used when the user asks to \"draft a campaign\", \"write an outreach campaign\", \"draft the Sales Nav campaign\", \"turn this ICP into a campaign\", or points at an ICP + saved-search pair under 02_inputs/outreach/. It produces a campaign spec in 03_drafts/outreach/ with filter summary, 3-step Sales Nav sequence copy (connect note + two follow-ups) at the default day-of-accept / +3 / +7 cadence, daily-send-cap contribution, exit conditions, and booking-flow notes. Cites first-party KB for proof points; third-party material is reference only. SOURCE OF TRUTH: this skill ships inline in rockstarr-outreach-salesnav for V0.1 and will migrate to rockstarr-infra/skills/_shared/ when the Interceptly variant lands so both plugins share one implementation."
---

# draft-icp-campaign

Turn a client-supplied ICP and saved Sales Navigator search URL into
an approved-ready campaign spec. Output lives in
`03_drafts/outreach/` and routes through `rockstarr-infra:approve`.

> **Shared-skill notice.** This is a shared skill. The canonical
> source will live at `rockstarr-infra/skills/_shared/draft-icp-campaign/`
> once both `rockstarr-outreach-salesnav` and `rockstarr-outreach-interceptly`
> exist. For V0.1 it ships inline in each plugin. If you edit this file,
> edit the matching copy in the other plugin in the same PR.

## When to run

- User says "draft a campaign from this ICP", names a slug that has
  both `02_inputs/outreach/icps/<slug>.md` and
  `02_inputs/outreach/lead-lists/<slug>.md`.
- A new `lead-lists/<slug>.md` appears and the matching `icps/<slug>.md`
  already exists.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and is approved.
- `/rockstarr-ai/00_intake/stack.md` has the outreach keys set (see
  README). If `outreach_tool` is not `salesnav`, refuse.
- `02_inputs/outreach/icps/<slug>.md` exists.
- `02_inputs/outreach/lead-lists/<slug>.md` exists and contains a
  `saved_search_url` pointing at a Sales Nav saved search.

If any are missing, stop and tell the user exactly which file is
missing and which upstream skill produces it.

## Inputs

Read, in this order:

1. `icps/<slug>.md` — the client-supplied ICP. Firmographics,
   seniority, signals, pains, disqualifiers.
2. `lead-lists/<slug>.md` — saved-search URL, any crawl notes, target
   lead count.
3. `00_intake/style-guide.md` — full. Pay literal attention to:
   - **Brand Personality** + Do / Do Not behaviors
   - **Tone Definition** contrast pairs ("X, not Y")
   - **Style Rules** and banned phrases
   - **Channel Adaptation — LinkedIn / outreach** block
4. `00_intake/client-profile.md` — offer names, audience phrasing,
   proof points, booking link URL.
5. First-party KB (`01_knowledge_base/processed/**` where
   `kb_scope: owned` AND `style_guide_eligible: true`) for proof
   points.
6. Third-party KB (`processed/third-party/`) for *industry context
   only*. Never paraphrase as if the client said it.

## Output

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
produced_by: "rockstarr-outreach-salesnav/draft-icp-campaign@0.1.0"
produced_at: "ISO timestamp"
style_guide_version: "matched from style-guide.md"
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
One-line restatement + the 3–5 signals that matter. Pulled from the
ICP file, not invented.

## Why now
The reason this ICP is hearing from the client right now, grounded in
first-party KB. No third-party paraphrase.

## Sequence

The four-message structure below is the canonical Rockstarr Sales
Nav cadence. Each message has a specific, narrow purpose — do not
collapse them, do not reorder them, do not rewrite them to "add
value" or "sound more confident." The purpose notes are hard
constraints, not vibes.

### Message 1 — Connect request (day of send)

**Body: BLANK.**

Leave the connect-request note empty on purpose. LinkedIn's default
behavior is a cleaner first touch than any custom note, and an
empty note keeps Step 1 from eating into the post-accept conversation
space. Do not write a connect note. Do not paste a pitch. Do not
"introduce yourself briefly." Blank is the spec.

### Message 2 — Connection accepted (day of accept)

**Purpose.** Acknowledge the connection and set light context
without pressure.

The body must:
- Thank them for connecting.
- Mention that something new or interesting is launching soon (kept
  vague — no product name, no offer name, no positioning).
- Signal you will follow up later.
- Stay neutral and low effort.

The body must NOT:
- Ask a question.
- Pitch anything.
- Include a call to action.
- Include a positioning statement.
- Use hype words, superlatives, or energy-raising language.

Keep it short. Two or three sentences. Under ~400 characters.

### Message 3 — First real touch (accept + 3 days)

**Purpose.** Invite engagement without pressure.

The body must:
- Reference what you (the client) are exploring or launching.
- Connect naturally to the core pain point from the ICP.
- Share a short observation or pattern (grounded in first-party KB,
  never third-party paraphrase).
- Ask one soft, non-leading question.
- Ask permission to continue (e.g., "fine to share more?" / "OK if
  I send a quick note next week?").

The body must NOT:
- Pitch, name-drop customers, or quote stats without a first-party
  source.
- Ask a hard-commit question (no calendar asks, no "can we hop on a
  call?").
- Use leading questions that assume the pain.
- Open with "Just following up."

Keep it conversational and neutral. Short paragraphs.

### Message 4 — Soft close (accept + 7 days)

**Purpose.** Close the loop respectfully if they have not responded.

The body must:
- Acknowledge they are likely busy.
- Communicate respect for their time.
- Gently reference the core pain point one last time.
- Leave the door open without pressure.
- Be brief, calm, and final in tone.

The body must NOT:
- Guilt them for not responding.
- Chase ("just bumping this," "following up again," "circling back").
- Restate your value proposition.
- Push for a reply. Make clear you are available IF the pain point
  becomes relevant — do not ask them to confirm or respond.

Short. Three or four sentences. Reads like a polite exit, not a
last-ditch pitch.

> Note: the campaign's official 3-step post-accept sequence is
> Message 2 + Message 3 + Message 4 above. Message 1 is the connect
> request (intentionally blank). The "3-step" sequence count in
> front-matter refers to the post-connect messages.

## Booking flow
- `booking_mode: automated` → propose-meeting-times is called when the
  thread reaches a meeting-ask moment. Bot books via the booking link.
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
   phrasing must trace to `client-profile.md` or a first-party KB
   file in `kb_sources_used`. Third-party appears only as *industry
   context* in "Why now," never as the client's voice.
3. **No invented proof.** No case studies, customer names, revenue
   figures, or dates the client has not already published.
   `[CLIENT TO CONFIRM]` for gaps.
4. **Sales Nav character limits + Message 1 rule.** Message 1
   (connect request) is intentionally BLANK — LinkedIn's default is
   cleaner than any custom note. Never fill it in, even if the ICP
   screams for it. Keep Messages 2–4 short; this is LinkedIn, not an
   email sequence. Message 2 under ~400 characters. Messages 3 and 4
   short paragraphs, never more than a short scroll on mobile.
5. **No booking link in copy.** Times are proposed later by
   `propose-meeting-times` and injected into reply drafts by
   `rockstarr-reply`. Never paste the booking URL into a sequence
   body.
6. **One campaign, one ICP, one saved search.** If the ICP implies two
   audiences, say so and ask the user to split the campaign.

## After writing

Print a summary in chat: slug, target lead count, sequence cadence,
any `[CLIENT TO CONFIRM]` markers, and the first-party sources cited.
End with:

> Campaign draft landed at `03_drafts/outreach/campaign-<slug>.md`.
> Review and either edit in place, ask for a revised pass, or run
> `rockstarr-infra:approve` when it is ready. `register-campaign`
> only runs against an approved file.

Do not call `approve` yourself. Human review is the gate.

## What NOT to do

- Do not run without both the ICP and the lead-list pointer file.
- Do not include the booking link in message copy.
- Do not paraphrase third-party sources.
- Do not seed leads, crawl the saved search, or write to the workbook
  from this skill. Those are `register-campaign` and `crawl-lead-list`
  jobs.
- Do not move the file to `04_approved/`. That is `approve`'s job.
