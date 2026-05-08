---
name: capture-deliverability-config
description: "This skill should be used at install time after capture-pitch when ops_deliverability_tool is not set to none, or any time the user says \"capture deliverability config\", \"set up deliverability\", \"refresh deliverability-config.md\", \"the deliverability tool changed\", \"the test segment changed\", \"the tracker URL changed\". Interviews the operator on the per-client deliverability tool configuration: sender address, tracker spreadsheet URL, test seed-inbox segment, template clone source, recurring deliverability task URL, comment format. Writes the result to /rockstarr-ai/00_intake/deliverability-config.md. Skipped entirely when stack.md says ops_deliverability_tool=none. The captured URL pattern (tool-specific) is what lets run-deliverability-test construct the report URL deterministically without reloading the tool's home page after the test code is captured — the single biggest gotcha across MailReach, GMass, and every other tool of this category."
---

# capture-deliverability-config

Intake interview that produces `00_intake/deliverability-config.md`
— the per-client configuration `run-deliverability-test` reads on
every recurring spam-score run.

This file is what makes `run-deliverability-test` tool-agnostic.
The skill ports cleanly across MailReach, GMass, and any future
deliverability tool because the captured URL pattern + sender +
tracker + segment + template-clone source are tool-specific
inputs the operator owns, not bot-side defaults.

## When to run

- Install-time, immediately after `capture-pitch` finishes — but
  ONLY when `stack.md.ops_deliverability_tool` is not `none`. When
  it is `none`, this skill is skipped entirely and
  `deliverability-config.md` is never produced.
- On demand, any time the deliverability tool changes (the
  operator switched from MailReach to GMass), the tracker URL
  changes (the operator moved the score sheet), the test segment
  changes (different seed inbox set), or the template-clone source
  changes.
- `run-deliverability-test` REFUSES to run when this file is
  missing AND `ops_deliverability_tool` is set to a real tool.

## Preconditions

- `scaffold-client` has run.
- `capture-stack` has run. This skill reads
  `stack.md.ops_deliverability_tool` to confirm a tool is set
  before running the interview.
- `capture-stack` has also captured `ops_deliverability_tracker`
  (the tracker container — Google Sheet vs. Airtable). This
  skill captures the SPECIFIC tracker URL inside that container.

## Refusal path

When `stack.md.ops_deliverability_tool=none`, this skill writes
no file and exits with a one-line note in chat:
"Deliverability tool is `none` — skipping. Re-run
`capture-stack` first if this is wrong." Do not write
`deliverability-config.md` as an empty file.

## Behavior

Walk the interview one question at a time via `AskUserQuestion`.
Pre-fill from any existing `deliverability-config.md` and tag
each pre-fill HIGH / MEDIUM / LOW confidence.

The exact wording of some questions changes when
`ops_deliverability_tool=mailreach` vs. `gmass` vs. anything
else — those tool-specific variants are noted inline below.

### Q1 — Sender address

Free text. "What email address sends the deliverability test?"

This is the address the test runs FROM, not the seed inbox set.
Often the operator's primary outbound sender (e.g.,
`hello@<client-domain>`).

### Q2 — Tracker URL

Free text (URL). "What's the URL of the tracker where scores get
logged?" Pre-fill from `ops_deliverability_tracker` if a URL is
already in `stack.md`.

When `ops_deliverability_tracker=google_sheet`, point at the
specific tab + row range the bot should append to. When
`airtable`, point at the specific base + table + view.

### Q3 — Test segment

Free text. "What's the seed-inbox segment / list / audience that
the test sends to?"

Tool-specific phrasing:

- **MailReach:** "What's the seed-inbox PROJECT name?"
- **GMass:** "What's the seed-inbox CAMPAIGN segment / list?"
- **Other:** "What does this tool call the seed-inbox group?"

### Q4 — Template clone source

Free text. "What's the most recent send template the bot
should clone for the test?"

Tool-specific phrasing:

- **MailReach:** "What's the campaign / project the bot should
  clone from? (the bot pastes the test code into the clone)"
- **GMass:** "What's the most recent campaign in your sent
  history the bot should clone?"
- **Other:** "What does this tool call the source send the bot
  clones the test from?"

The operator names a specific source send by name. The bot
clones that specific source on every test run.

### Q5 — Deliverability task URL

Free text (URL). "If your task system has a recurring task for
the deliverability test, what's its URL? (Optional — leave
blank if you run the test manually.)"

When set, `run-deliverability-test` posts its score-comment back
to this task. When blank, the bot logs the score to the tracker
only and writes a daily-summary note instead.

### Q6 — Comment format

Free text (template). "What format should the comment on the
queue task use?"

Default template:
```
{date} — Score: {score}/10 — {sender} → {segment} — {report_url}
```

Operators with a strict format own the override. The bot fills
the placeholders verbatim.

### Q7 — Low-score threshold

Numeric. "Below what score (out of 10) should the bot prefix
the comment with `LOW SCORE` and bump the run to the top of the
weekly report?" Default `8.0`.

### Q8 — URL pattern for the report

This question is what makes the bot tool-agnostic. The deliv
tool generates a fresh test code on every page-load of its home
page, and the report URL is `<base>/<code>`. The bot constructs
the URL deterministically from the code captured on the first
page-load and never reloads the home page after that.

Free text (URL pattern). "What's the URL pattern for an
individual deliverability report?" The pattern uses
`{code}` as the placeholder for the test code.

Tool-specific defaults to OFFER as a pre-fill (operator confirms
or amends):

- **MailReach:** `https://app.mailreach.co/spam-tests/{code}`
- **GMass:** `https://gmass.co/inbox-test/{code}`
- **Other:** ask the operator to find the pattern by running one
  manual test in the tool, copy the report URL, and replace the
  test-code segment with `{code}`.

### Q9 — Test cadence

Single-select `AskUserQuestion`:

"How often should the bot run the deliverability test?"

- `Weekly` (default — most common)
- `Twice a month`
- `Monthly`
- `On-demand only`

When set to `On-demand only`, the bot does not schedule a
recurring run — `run-deliverability-test` only fires when the
operator triggers it manually.

## Write the file

Write `/rockstarr-ai/00_intake/deliverability-config.md`.
Overwrite if it exists.

> **Template convention.** Same `# ---` substitution as in
> capture-pitch — emit real `---` in the actual output file.

```markdown
# ---
captured_at: <ISO timestamp>
schema_version: 1
ops_deliverability_tool: <stack.md value, copied for convenience>
# ---

# Deliverability test config

> Per-client configuration consumed by `run-deliverability-test`
> on every recurring spam-score run. Re-run
> `capture-deliverability-config` when any of these changes.

## Sender

<Q1 verbatim>

## Tracker

URL: <Q2 verbatim>
Container: <stack.md.ops_deliverability_tracker, copied>

## Test segment

<Q3 verbatim>

## Template clone source

<Q4 verbatim>

## Deliverability queue task

URL: <Q5 verbatim, or `none — manual trigger only`>

## Comment format

```
<Q6 verbatim>
```

## Low-score threshold

<Q7 numeric>

## Report URL pattern

```
<Q8 verbatim>
```

## Cadence

<Q9 single-select label>
```

## Outputs

- `/rockstarr-ai/00_intake/deliverability-config.md`.
- When `Q9=Weekly | Twice a month | Monthly`, recommend that the
  operator wire a Cowork scheduled task at the chosen cadence.
  This skill does NOT create the scheduled task itself — that
  belongs to `rockstarr-infra`'s schedule layer.

## What this skill does NOT do

- Does NOT install or authenticate the deliverability tool. Tool
  authentication is a one-time browser action the operator does
  outside Cowork.
- Does NOT pre-validate the URL pattern by hitting it. The first
  real run of `run-deliverability-test` is the validation pass.
- Does NOT create the recurring task in the task system. If the
  operator wants the task system to drive cadence (vs. Cowork
  scheduled tasks), they create the task manually and paste the
  URL into Q5.

## Failure modes

- **Operator gives Q8 as a single static URL** ("here's a
  report I generated last week"). Catch this — explain that the
  pattern needs `{code}` as a placeholder for a fresh test
  code on every run. Walk them through extracting the pattern
  by running ONE manual test in the tool.
- **Q4 names a send the operator just sent live** (not a clone
  source). Reject — the source must be a stable template the
  bot can repeatedly clone without polluting the live send
  history. Ask for a different source.
- **Q5 URL doesn't match the task system in `stack.md`.** Soft
  warn but accept. Some clients use one task system for general
  work and a different tracker for deliverability.

## Re-running

Overwrite is the default. The whole point of re-running is to
refresh the configuration — the bot keeps ONE current version.
