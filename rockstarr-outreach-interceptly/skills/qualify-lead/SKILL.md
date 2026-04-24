---
name: qualify-lead
description: "This skill should be used on every reply in the per-reply pipeline, or when the user says \"qualify this lead\", \"does this lead match our ICP\", or \"run ICP qualification on this lead\". Reads the Interceptly right-panel (title, company, Work Experience) plus any optional LinkedIn fallback, checks the lead against /00_intake/icp-qualifications.md (tightened by the active campaign's per-campaign ICP overrides if any), and returns one of four verdicts: target / not_target / ambiguous / unknown — with the matching rule cited. Runs on every reply, not just once per lead — the lead's company or title may have changed since the last check."
---

# qualify-lead

Per-reply qualification. The bot has no opinion on who is or
isn't a target beyond what the client wrote in
`icp-qualifications.md` plus any tightening in the active
campaign's ICP file. This skill is pure rule evaluation — it does
not draft, label, or take any action.

## When to run

- Step 1 of the per-reply pipeline, inside `process-inbox` and
  `process-my-tasks`.
- On demand when the user says "qualify `<lead>`" — useful when
  they want to sanity-check the rules against a specific lead
  before refining `icp-qualifications.md`.

## Preconditions

- `/00_intake/icp-qualifications.md` exists and has target /
  not-target / ambiguous sections. If not, refuse and point at
  `capture-icp-qualifications`.
- The lead's Interceptly right-panel is currently rendered in
  Chrome (or a mirror of its content has been passed to the
  skill).

## Inputs

- `lead_url` — Interceptly thread URL or LinkedIn profile URL
  (required).
- `right_panel_context` — structured data from the right panel:
  `{name, company, title, location, work_experience[], notes}`.
- `active_campaign_slug` — the campaign the lead is enrolled in
  (optional; if absent, skip per-campaign tightening).

## Behavior

### Step 1 — Load rule sources

1. Read `/00_intake/icp-qualifications.md`. Parse the target,
   not-target, and ambiguous rule lists.
2. If `active_campaign_slug` is provided, read
   `/02_inputs/outreach/icps/<slug>.md` and layer its
   `## Per-campaign tightening` block on top. Tightenings ADD
   restrictions; they never loosen.

### Step 2 — Match the lead against rules

Run in this order, returning the first matching verdict:

1. **Not-target rules.** If any not-target rule matches
   (role, company type, behavior), return
   `{verdict: "not_target", rule: "<matching rule>",
   confidence: "high"}`.
2. **Per-campaign tightening excludes.** If the campaign has
   additive exclusions, check them next. Same verdict shape.
3. **Ambiguous rules.** If the lead hits any ambiguous rule,
   return `{verdict: "ambiguous", rule: "<matching rule>",
   needs: "<what would tip it to target>"}`.
4. **Target rules.** If the lead matches target rules (role
   cluster AND company-size AND industry — intersection, not
   union, unless `icp-qualifications.md` explicitly says
   otherwise), return `{verdict: "target", rule: "<matching
   rule>", confidence: "<HIGH|MEDIUM>"}`.
5. **Unknown.** If none of the rule branches matches — the
   right-panel context is too thin to place the lead — return
   `{verdict: "unknown", needs: "more context (LinkedIn
   profile, company website, etc.)"}`.

### Step 3 — Optional LinkedIn fallback

If verdict = `unknown` AND
`icp-qualifications.md.minimum_extra_context_for_ambiguous_to_target_promotion`
specifies the kind of LinkedIn context that would help, call
`process-inbox`'s LinkedIn side-tab helper to fetch the lead's
LinkedIn profile summary (read-only). Re-run Step 2 with the
enriched context.

Do NOT send from the LinkedIn side tab. Ever.

### Step 4 — Record verdict

Append a row to the `Qualifications` sheet of
`outreach-mirror.xlsx`:

| column | value |
|---|---|
| ts | `<ISO>` |
| lead_url | `<URL>` |
| campaign_slug | `<slug or blank>` |
| verdict | `target / not_target / ambiguous / unknown` |
| rule_cited | `<the matching rule text>` |
| confidence | `HIGH / MEDIUM / LOW` |

If verdict = `not_target`, also append to
`/02_inputs/outreach/_non_icp_log.md` with the rule cited.
The Non-ICP Log is the bot's durable memory of "who the client
decided is not a target and why" — weekly reports surface these
for refinement.

### Step 5 — Return

Return the verdict object to the caller. The caller decides what
to do with it (draft, flag, skip).

## Semantics

- `target` → caller hands off to `rockstarr-reply:classify-reply`
  and `rockstarr-reply:draft-reply`.
- `not_target` → no draft (except the non-ICP yes three-option
  flow; `process-inbox` handles that branch). Label via the
  default mapping (`decline` → Not Interested; polite ack →
  Ignore). No follow-up task.
- `ambiguous` → caller hands off to
  `rockstarr-reply:flag-for-review`. No draft.
- `unknown` → caller hands off to
  `rockstarr-reply:flag-for-review` with a note asking the operator
  to supply the missing context.

## Re-qualification

`qualify-lead` runs on EVERY reply, not just once per lead. A
lead who was `not_target` last week may be `target` this week if
they changed companies. Cached verdicts in the mirror are
informational — the current run's verdict wins.

## Failure modes

- **Right-panel context is empty.** Wait 2s and retry reading the
  DOM once. On second failure, return `unknown` with `needs:
  "right-panel unreadable; retry or check Chrome MCP"`.
- **Campaign ICP file missing but slug passed.** Warn and proceed
  with baseline rules only. Write an _errors.md note.
- **Multiple rules match with conflicting verdicts.** Not-target
  wins over ambiguous wins over target. Strictness always wins.

## What NOT to do

- Do not hardcode any rule the client did not write into
  `icp-qualifications.md`. Zero baked-in opinions.
- Do not cache a `target` verdict across reply runs. Always
  re-evaluate — the world changes.
- Do not send a message, apply a label, or create a task from
  this skill. Pure rule evaluation.
- Do not paraphrase the matching rule. Cite it verbatim from
  `icp-qualifications.md` so the operator can audit the
  decision chain.
