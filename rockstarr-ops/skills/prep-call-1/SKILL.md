---
name: prep-call-1
description: "This skill should be used when daily-call-prep classifies a calendar event as Call 1 (discovery), or when the user says \"prep the discovery for [name]\", \"call 1 prep for [name]\", or \"first call prep for [name]\". Produces Call1_Prep_[name]_[date].docx with intel snapshot (LinkedIn + company site + outreach thread + booking source), ICP verdict (go/caution/disqualify), primary pain hook, angle, and phase script (opener, discovery, pitch, price, close, objections, off-ramp). Phase scaffold from ops-call-framework.md; pitch/price from pitch.md. Operator-facing — bypasses stop-slop."
---

# prep-call-1

Per-prospect Call 1 (discovery) prep doc. Produces a docx file at
`/03_drafts/ops/sales-prep/Call1_Prep_[name]_[YYYY-MM-DD].docx`
that the operator reads before the call.

The doc is **operator-facing** — it bypasses stop-slop (stop-slop
is for what the LEAD reads, not what the operator reads). The
doc cites real intel, names real ICP filters, and renders the
operator's framework verbatim against this specific prospect.

## When to run

- Auto, by `daily-call-prep` when an event on today's calendar
  classifies as a Call 1 (no prior recorded call AND no
  substantive prior thread for this prospect).
- On demand by the operator via the trigger phrases above.
- Both paths produce the same docx output.

## Preconditions

- `00_intake/pitch.md` exists.
- `00_intake/ops-call-framework.md` exists.
- `00_intake/icp-qualifications.md` exists (chained from
  `rockstarr-reply` install).
- `stack.md` is set with at minimum
  `ops_email_outreach_tool` (LinkedIn outreach thread reading)
  and `ops_email_platform` (email thread reading).
- The Chrome MCP browser session is authenticated against
  LinkedIn for intel reads.

Refuse with a one-line pointer to the missing intake skill if
any precondition fails.

## Inputs

From the caller (orchestrator or operator):

- `lead_name` — required.
- `lead_company` — optional; resolved from the calendar event
  + LinkedIn lookup if missing.
- `lead_linkedin_url` — optional; resolved by name+company
  search if missing.
- `event_start` — calendar event start time (ISO).
- `booking_source` — optional; how the meeting was booked
  (referrer, outbound campaign, inbound form, etc.). The
  orchestrator surfaces this from the event description /
  booking notes when available.

## Behavior

### Step 1 — Resolve the prospect

If `lead_linkedin_url` is missing, search LinkedIn via Chrome
MCP for `[lead_name] [lead_company]`. Confirm the match by
reading the headline and a snippet of recent activity.

If no clear match, the bot does NOT guess. Surface the
ambiguity in chat: "Couldn't resolve [name] on LinkedIn — name
the URL or skip this prep." Skip means writing the prep doc
with an "Intel: NOT RESOLVED" header.

### Step 2 — Gather intel from four sources

**LinkedIn profile.**
- Title, company, tenure at company.
- Recent posts (last 30 days) — capture 1-2 lines from any
  post that signals current priorities or current frustrations.
- Recent activity (likes, comments) — capture themes, not
  individual posts.

**Company website (homepage + one of about / team / pricing).**
- Positioning paragraph — what the company says it does.
- Stage signal — recent funding, recent hires, recent product
  launches.
- ICP-relevant signal — anything that confirms or denies the
  ICP filter (industry, size, model).

**Outreach thread (when `ops_email_outreach_tool` carries
prior contact).**
- Verbatim quote of the prospect's last 1-2 messages in the
  thread. Quotes are verbatim — the bot does NOT paraphrase.
- The campaign / sequence the prospect was in (when the tool
  surfaces it).

**Booking-source context.**
- For inbound forms: any form-field answers the operator
  captured.
- For referrals: who referred, what the referrer said.
- For outbound: which campaign the prospect was in.

The intel block is rendered as four sub-sections in the docx —
one per source, with verbatim quotes called out as
block-quotes.

### Step 3 — Run ICP verdict

Read `00_intake/icp-qualifications.md`. Apply each rule
against the intel from Step 2. Produce one of three verdicts:

- **GO** — every rule matches.
- **CAUTION** — most rules match but at least one is ambiguous
  (e.g., company size unclear, industry adjacent).
- **DISQUALIFY** — at least one hard-no rule fires.

Render the verdict in the docx with the matching / ambiguous /
failing rule cited verbatim. On DISQUALIFY, the prep doc still
gets written — the operator may want to take the call to
preserve the relationship — but the docx leads with a
DISQUALIFY callout and points the operator at the disqualify
language section.

### Step 4 — Pick the primary pain hook

Read the pain-hook list from `ops-call-framework.md` Q5. Pick
ONE that aligns most directly with the prospect's intel.
Selection logic:

- LinkedIn post in last 30 days that names a frustration → use
  the hook that matches that frustration.
- Recent funding → "What's stopping you from doing it
  yourself?" or similar growth-stage hook.
- Outreach-thread reply that named a specific bottleneck → use
  the hook that matches that bottleneck.
- No clear signal → default to the hook the operator marked
  as their go-to in the framework file.

The selected hook is the OPENING question for the discovery
phase. The prep doc renders it verbatim.

### Step 5 — Build the phase-by-phase script

For each phase named in `ops-call-framework.md`:

- Render the phase heading verbatim from the framework.
- Under the heading, render the phase content from the
  framework, **personalized to this prospect**:
  - Opener (Phase 1) named to the operator's voice.
  - Discovery questions (Phase 2 typically) tuned by the
    primary pain hook + ICP verdict.
  - Pitch (Phase 3 typically) using the EXACT pricing
    language from `pitch.md` (NOT from any transcript).
  - Close + objection handling (final phase) using the
    framework's verbatim handlers.
- Render any prospect-specific notes inline as italicized
  asides ("Mention [signal] here — they posted about it 3
  days ago").

**Conflict rule.** If the framework's pricing card differs
from `pitch.md`'s pricing, the doc uses `pitch.md`. Surface
the conflict in a callout: "Framework pricing card is stale —
re-run `capture-call-framework` after this call." This
prevents the prep doc from quoting one number while the
operator quotes another live.

### Step 6 — Render the clean off-ramp

When the ICP verdict is CAUTION or DISQUALIFY, render the
clean off-ramp section using `ops-call-framework.md` Q6
disqualify language verbatim. Operator reads it before the call
in case the call surfaces the disqualifier in real time.

### Step 7 — Write the docx

Write `/03_drafts/ops/sales-prep/Call1_Prep_[lead_name]_[YYYY-MM-DD].docx`.

Use the docx skill's helpers — every Rockstarr docx writer
goes through there for consistent fonts, headings, and TOC.

The docx structure:

1. **Cover** — `Call 1 Prep — [lead_name]` header. ICP verdict
   chip. `event_start` time. Operator name.
2. **Intel snapshot** — four sub-sections (LinkedIn, website,
   outreach thread, booking-source). Verbatim quotes
   block-quoted.
3. **ICP verdict** — verdict chip + matching / ambiguous /
   failing rule citation.
4. **Primary pain hook** — selected hook + reason for
   selection.
5. **Angle** — one-paragraph framing of how the operator
   approaches THIS prospect (synthesized from intel + ICP +
   pain hook).
6. **Phase-by-phase script** — one section per phase from
   `ops-call-framework.md`, personalized per Step 5.
7. **Pricing** — from `pitch.md` Q2 verbatim. Conflict callout
   if framework differs.
8. **Clean off-ramp** — verbatim from `ops-call-framework.md`
   Q6. Always rendered, even on GO verdicts (the operator may
   need it mid-call).
9. **Open questions** — anything the bot couldn't resolve
   (intel gaps, ambiguous ICP rule, etc.).

### Step 8 — Front-matter

The docx's first page renders front-matter as a small
metadata footer (also written into a `.md` companion file at
the same path with `.md` extension instead of `.docx` — the
markdown sidecar is what `daily-call-prep` reads to render the
chat-summary line):

> **Sidecar template convention.** The fenced code block below
> shows `# ---` where YAML front-matter delimiters belong, only
> to keep Cowork's SKILL.md parser from misreading them as
> front-matter separators. **When writing the actual sidecar,
> emit real `---`, not `# ---`.**

```markdown
# ---
schema_version: 1
type: ops-prep-call-1
lead_name: <name>
lead_company: <company>
lead_linkedin_url: <URL>
event_start: <ISO>
icp_verdict: GO | CAUTION | DISQUALIFY
icp_matching_rule: <verbatim from icp-qualifications.md>
primary_pain_hook: <verbatim from ops-call-framework.md Q5>
generated_at: <ISO>
docx_path: <path to the docx>
booking_source: <from caller>
intel_resolved: true | false
# ---
```

## Outputs

- `/rockstarr-ai/03_drafts/ops/sales-prep/Call1_Prep_[name]_[date].docx`.
- A markdown sidecar at the same path with `.md` extension,
  for the orchestrator's chat-summary read.

## Approvals

The docx is operator-facing. There is no `send` step. The
operator reads it before the meeting and uses it on the call.
Per-prep-doc approval is the operator's read; no explicit
approval gate is needed.

`daily-call-prep` archives the docx + sidecar to
`/05_published/ops/` at end-of-day after the meeting completes
(the orchestrator owns the archive step, not this skill).

## Failure modes

- **LinkedIn intel returns nothing.** Render the intel section
  with "LinkedIn: profile reachable but no recent activity
  signal." Don't make stuff up.
- **Outreach thread is empty.** Render the section with
  "Outreach: no prior thread (cold inbound or referral)." This
  often correlates with the booking source being a referral or
  inbound form.
- **`pitch.md` and `ops-call-framework.md` disagree on
  pricing.** Use `pitch.md`. Render the framework's stale
  number in a strikethrough or italic callout. Recommend a
  re-run of `capture-call-framework` in the chat-side
  daily-summary.
- **Prospect company website blocks Chrome MCP** (auth wall,
  bot-block). Render the section with "Website: not
  reachable — operator verifies live." Continue with the rest.
- **Calendar event has no end-time / unclear duration.** Use
  60 minutes as the default and surface in the open-questions
  section.

## What this skill does NOT do

- Does NOT draft prose for the operator to deliver verbatim.
  The phase-by-phase script renders structural content + the
  operator's framework verbatim — but the operator's actual
  language is the operator's domain.
- Does NOT pull pricing from anywhere except `pitch.md`. Old
  transcripts and old pitch files are NOT consulted.
- Does NOT run stop-slop. Operator-facing docs bypass stop-slop
  by design.
- Does NOT update the CRM. CRM writes route through the
  active `rockstarr-ops-bot-[crm]` and are owned by
  `process-call-transcript` after the call, not before.
- Does NOT post to chat. The orchestrator (or the operator's
  trigger) handles the chat-summary side; this skill only
  produces the docx + sidecar.
