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

### Step 1 — Connect note (day of send)
[Body of the connect request. Max ~300 characters. No offer link.
No ask. Tone per style guide — land on the "X, not Y" X side.]

### Step 2 — Follow-up (day of accept)
[Body. Opens the conversation. Offers value. No booking link.]

### Step 3 — Follow-up (accept + 3 days)
[Body. Reinforces value, invites a reply.]

### Step 4 — Follow-up (accept + 7 days)
[Body. Soft break. Invites a reply or graceful exit.]

> Note: the campaign's official 3-step sequence is Step 2 + Step 3 +
> Step 4 above. Step 1 is the connect note; the sequence count refers
> to the post-connect messages.

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
4. **Sales Nav character limits.** Keep Step 1 under ~300 characters.
   Keep Step 2–4 short — this is LinkedIn, not an email sequence.
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
