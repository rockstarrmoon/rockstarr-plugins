# rockstarr-outreach-salesnav (developer notes)

This file is for Claude Code working on this plugin's source. For the
full skill catalog, hard constraints, campaign types, and the workbook
schema, read `README.md` in this folder. This CLAUDE.md is its
developer-perspective complement.

## Why this plugin is different from the rest

This is the Sales Navigator outreach variant — the no-extension
fallback to `rockstarr-outreach-interceptly` (the default). It shares
much of the interceptly variant's shape (Chrome MCP, per-reply
pipeline, rockstarr-reply caller) but with three meaningful
simplifications and one meaningful complication:

**Simplifications vs interceptly:**

1. **Single LinkedIn account.** No multi-account rotation, no
   `switch-account` skill. `confirm-session` runs once per
   daily-loop and gates everything.
2. **No Interceptly account/workspace plumbing.** No discovery
   skill, no per-workspace API token, no `discover-accounts`
   step. Sales Nav lives at one URL and the bot signs in once.
3. **Skill names are unsuffixed.** This was the original outreach
   variant; interceptly came later and inherited the
   `-interceptly` suffix convention. New skills added here stay
   unsuffixed — if interceptly needs an equivalent later, it
   takes the suffix.

**Chrome MCP × Sales Nav reality (re-evaluated 0.3.0).** Earlier
drafts of this CLAUDE.md called the Sales Nav UI "stable" with
"far fewer quirk-recovery cases than interceptly." Field data from
the first full-volume rollout (RigTex) overturned that:

- **Saved-search rows are virtualization-locked under Chrome MCP.**
  Sales Nav lazy-loads each result row's `<article>` on scroll;
  the rows often do not hydrate within Chrome MCP's
  `Runtime.evaluate` window. `daily-connect`'s old "click name →
  row three-dot" canonical path was therefore unreachable, and
  `crawl-lead-list`'s old whole-page-wait approach extracted
  partial rows. Both are fixed in 0.3.0 (see the README's
  Versioning section), but anyone touching these skills should
  assume Chrome MCP + Sales Nav is fragile, not stable, and design
  defensively: per-row `scrollIntoView` + hydration polls,
  pure-JS one-shot click patterns (refs from `read_page` expire
  mid-batch), and full page refreshes after every 3 successful
  sends to clear SPA state.
- **The em-dash matters.** Verify regexes that look for "Connect —
  Pending" must use U+2014 explicitly. The ASCII hyphen produces
  silent false negatives on every confirmed send.
- **Cowork's scheduled-task turn budget caps a single morning
  loop.** Steps 1–7 of the canonical daily loop, at full volume,
  blow past the ~100-turn ceiling. The split-task pattern
  documented in the README is the practical answer.

**Complication vs interceptly:**

The workbook (`outreach-tasks.xlsx`) is **the state-of-truth**, not
an audit mirror. The interceptly variant treats its workbook as a
mirror — Interceptly itself is authoritative. Here, the workbook
*is* authoritative. The bot reads task state, lead state, and
campaign state from the workbook on every run. There is no separate
upstream source to reconcile against. This makes the workbook schema
load-bearing in a way the interceptly mirror is not.

## Hard constraints (from README; surfacing the operationally critical ones)

1. All sends happen in Sales Nav. Never bypass to LinkedIn directly.
2. A lead belongs to exactly one campaign at a time — no duplicates across active campaigns.
3. **20 connections per day, 100 per ISO week, hard cap across all active campaigns.** Unused daily capacity carries forward inside the current ISO week only. The weekly ceiling is the LinkedIn-account-health budget; respect it.
4. Every reply routes through `rockstarr-reply` for approval before sending.
5. `confirm-session` runs before any daily send. Mismatch aborts the whole loop — wrong-account sending is a reputational incident.
6. The booking link is **never handed to the lead**. Either the bot books on the lead's behalf (after they agree to a specific proposed time) or the operator books manually and calls `mark-booked`.

## Campaign types

Two campaign types share the global cap and the same workflow up to
the accept; they diverge after:

- **`full-sequence`** (default) — connect request + 3-message
  post-accept conversation. The original campaign shape; every
  pre-0.1.7 campaign is implicitly this type.
- **`connect-only`** — connect requests only, no post-accept
  sequence. The campaign ends at `state=accepted`. Used for
  network-building / first-degree audience growth.

The 20/day + 100/week cap is global across types — adding a
connect-only campaign doesn't get you a separate budget. LinkedIn
account health is one budget.

## Skill groupings (mental map)

19 skills sort into six buckets:

1. **Campaign lifecycle** — `draft-icp-campaign`,
   `register-campaign`, `stop-campaign`. Draft → approve → register
   → (optionally later) stop.
2. **Daily loop (fixed order; each step degrades gracefully if the
   prior produced nothing)** — `confirm-session` → `preview-queue`
   → `detect-accepts` → `generate-message-tasks` →
   `send-scheduled-messages` → `detect-replies` → `daily-connect`
   → `metrics-daily`. The canonical order is what the skills
   assume; **how** the order is split across scheduled tasks is a
   separate concern documented in the README's "Scheduling the
   loop" section (single-task pattern for low-volume workspaces;
   split-task pattern — messages + connects, 60-min offset — for
   full-volume).
3. **Per-reply pipeline (caller side of rockstarr-reply)** —
   `send-approved-reply`. The skill that executes on an
   `authorized-send` bundle returned by
   `rockstarr-reply:present-for-approval`.
4. **Meeting & booking** — `propose-meeting-times`, `book-meeting`,
   `mark-booked`.
5. **Lead acquisition** — `crawl-lead-list`. Invoked inside
   `register-campaign` to populate the Leads sheet from a
   saved-search URL.
6. **Metrics & reporting** — `metrics-daily`, `metrics-weekly`,
   `outreach-weekly-report`, `backup-workbook`.

## The connect budget math

The 20/day + 100/week cap is the single most important rule in this
plugin and the thing most likely to be subtly broken by a
well-meaning change. The math:

- Daily allowance starts at 20.
- Unused daily allowance carries forward *within the current ISO
  week only* — Monday-to-Sunday counter resets every Monday.
- Weekly ceiling of 100 is hard. If the week-to-date counter is
  already at 100, daily-connect sends zero regardless of daily
  carry-forward.
- All active campaigns share one budget. `daily-connect`
  round-robins across campaigns; it does not allocate per-campaign
  quotas.

If you touch `daily-connect` or `metrics-daily`, verify the
carry-forward math survives. The weekly cap is what protects the
client's LinkedIn account from a temporary-restriction warning —
it's not a soft target.

## Caller side of the rockstarr-reply contract

This plugin is a caller of `rockstarr-reply` (see
`plugins/rockstarr-reply/CLAUDE.md` for the contract). Caller
responsibilities specific to this variant:

- `detect-replies` is the primary caller. It runs once per
  daily-loop, accumulates `staged_paths` from every reply it
  drafts, and at end-of-run fires `rockstarr-infra:notify-reply-ready`
  with the global accumulator. Set `batch_context=detect-replies`
  on every handoff to ensure `draft-reply` doesn't double-fire
  the notification.
- `send-approved-reply` is the executor for `authorized-send`
  bundles returned from `present-for-approval`. Sends via Sales
  Nav, logs to the Messages sheet, creates a 2-day follow-up task.
- Both must verify `confirm-session` passed before they
  Chrome-MCP-touch Sales Nav.

## The workbook — read carefully before modifying

`/rockstarr-ai/02_inputs/outreach/outreach-tasks.xlsx` is shared
across all campaigns and is the bot's only state. Every skill
reads from it and most write to it. The full schema is in the
README's "The workbook — single state-of-truth" section.

Key invariants to preserve:

- A lead URL appears in `Leads` exactly once per workspace at a
  time. Cross-campaign uniqueness is enforced at `register-campaign`.
- `campaign_slug` appears on every sheet — it's the join key.
- `state` transitions on `Leads` are monotonic: queued →
  connected → accepted → replied → booked / opted-out.
  Backward transitions are bugs.
- Task `status` transitions: pending → done / cancelled.
  Terminal. Don't re-open.

## What's high-risk to change in this plugin

- **`confirm-session`.** Wrong-account sends are the top reputational risk.
- **`daily-connect`'s budget math.** A bug here can blow past the LinkedIn cap and trigger a restriction warning.
- **`daily-connect`'s UI path + pacing rules.** The
  `lead_profile_overflow` canonical, pure-JS click pattern, em-dash
  verify regex, and page-refresh-every-3-sends pacing are all
  load-bearing under Chrome MCP. Each one was validated against
  field-tested failure modes; reverting any of them re-introduces a
  documented breakage (silent send failures, send-not-confirmed
  cascades, 3–5 send ceiling).
- **The workbook schema.** Every skill reads it. Add columns at the end with sensible defaults; never rename or drop existing ones without coordinated updates across all skills.
- **The handoff bundle this plugin sends to `rockstarr-reply`.** Thin upstream → "(not captured)" in the urgent email downstream.
- **The fixed daily-loop step order.** Some steps assume earlier ones have run (e.g. `generate-message-tasks` runs after `detect-accepts` because it operates on the just-accepted leads).

## What's safe to change without much ceremony

- Selectors and waits inside Chrome-MCP skills.
- Per-skill prompts and operator-facing messages.
- Metric calculations within the existing column schema.
- New optional `stack.md` config fields with sensible defaults.
- Internal logic of `draft-icp-campaign` (within the same output
  shape).

## Versioning

Major version axis: workbook schema changes, handoff bundle shape
changes, daily-loop ordering changes. Minor: new skills, new
optional config. Patch: bug fixes, UI selector updates.

Bump via `/bump rockstarr-outreach-salesnav <new-version>` at the
repo root.

## Testing your changes

Same shape as interceptly's testing approach:

1. **Full sideload + drive.** Build the `.zip`, sideload into your
   Cowork, run against a test Sales Nav account.
2. **Unit-style dry run.** Non-Chrome-MCP skills (campaign drafting,
   metrics rollup, workbook reads/writes) can run against a fixture
   client folder.
3. **Read-only inspection.** For Chrome-MCP skills, manually trace
   selectors against the current Sales Nav UI.

Sales Nav is stable enough that selector drift is rare — when it
happens it's worth noting in the README's quirks section if there
isn't one yet.

CI in this monorepo only checks skill-name uniqueness.

## When you're stuck

Ask Jon in the PR. Cap-math questions and workbook-schema questions
especially.
