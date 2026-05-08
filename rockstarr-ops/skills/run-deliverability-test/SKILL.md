---
name: run-deliverability-test
description: "This skill should be used when the recurring scheduled run fires at the cadence captured in deliverability-config.md (typically weekly), when a task with the configured ops_post_call_queue_tag for deliverability lands in the task system, or when the user says \"run the deliverability test\", \"run the spam-test\", \"check the spam score\", \"do the mail test\", \"run mailreach\", \"run the deliverability check\". Executes the recurring spam-score test against whichever deliverability tool the client uses (MailReach, GMass, future tools). Nine steps: clone the latest configured send template, paste the test code, send to the configured seed-inbox segment, retrieve the score from the deterministically-constructed report URL, log to the configured tracker, clean up the clone, and comment back on the queue task. The single biggest gotcha is code rotation — the test code regenerates on every page-load of the tool's home page; the bot constructs the report URL deterministically from the code captured on first page-load and never reloads the home page after capture. Tool-agnostic by design: every tool-specific URL pattern, sender, tracker, segment, and clone source comes from deliverability-config.md."
---

# run-deliverability-test

Recurring spam-score check. Tool-agnostic — every tool-specific
detail comes from `00_intake/deliverability-config.md`. The flow
is well-defined nine steps and ports cleanly across MailReach,
GMass, and future deliverability tools.

The single biggest gotcha across every deliverability tool of this
category is **code rotation**. The test code is regenerated on
every page-load of the tool's home page. The bot constructs the
report URL deterministically from the code captured on the first
page-load and **never reloads the home page after capture**.
Reloading invalidates the captured code; the score will look up
against the stale code and silently miss the actual run.

## When to run

- Recurring on the cadence captured in
  `deliverability-config.md` Q9. The bot is content-agnostic on
  trigger source: a Cowork scheduled task fires it, OR a recurring
  task in the task system with the configured
  `deliverability_task_url` fires it. Either path is fine.
- On demand by the operator with the trigger phrases above.
- A new task with the configured `ops_post_call_queue_tag` for
  deliverability landing in the task system, when the system
  supports webhooks.

## Preconditions

- `00_intake/deliverability-config.md` exists and is approved.
  Refuse with a pointer to `capture-deliverability-config` if not.
- `stack.md.ops_deliverability_tool` is not `none`. Refuse with a
  one-line note if it is.
- The deliverability tool is authenticated in the browser session
  the Chrome MCP will drive. The bot does NOT manage tool
  authentication — that's a one-time browser action the operator
  does outside Cowork.
- `stack.md.ops_deliverability_tracker` is set and the tracker URL
  in `deliverability-config.md` Q2 is reachable.

## Inputs

Read on every run from `00_intake/deliverability-config.md`:

- `sender` — Q1
- `tracker_url` — Q2
- `tracker_container` — `stack.md.ops_deliverability_tracker`
- `test_segment` — Q3
- `clone_source` — Q4
- `task_url` — Q5 (optional)
- `comment_format` — Q6
- `low_score_threshold` — Q7 (default `8.0`)
- `report_url_pattern` — Q8 (with `{code}` placeholder)
- `cadence` — Q9

## Behavior

### Step 1 — Open the tool's home / dashboard page (ONCE)

Via Chrome MCP (`mcp__Claude_in_Chrome__navigate`), open the
deliverability tool's home page. This is the page that displays
the current test code. Do NOT reload after this point — code
rotation happens on every reload, and the captured code below
must remain the same code the actual test sends use.

For MailReach: `https://app.mailreach.co/spam-tests/new`.
For GMass: the inbox-test page in the GMass dashboard.
For other tools: the URL pattern captured at install through Q8
implies the home-page location.

### Step 2 — Capture the test code

Read the test code from the page DOM. The exact selector is
tool-specific; common patterns:

- MailReach: a `<code>` block in the new-test panel, or the
  `data-test-code` attribute on the test-instructions row.
- GMass: an `id`-bearing input or `<code>` block in the test
  setup section.

Capture the code as a string. This is the single value that
parameterizes the rest of the run.

### Step 3 — Construct the report URL deterministically

Take `report_url_pattern` from the config and substitute the
captured code into `{code}`. Store the resulting URL as
`report_url`. This is the URL Step 7 navigates to AFTER the
test propagates — without ever reloading the home page.

Example, MailReach:
- Pattern: `https://app.mailreach.co/spam-tests/{code}`
- Captured code: `abc123xyz`
- `report_url` → `https://app.mailreach.co/spam-tests/abc123xyz`

### Step 4 — Open the clone source send

Via Chrome MCP, navigate to the most recent send / campaign /
project the operator named in `clone_source`. The path is
tool-specific:

- MailReach: campaigns list → click the named project.
- GMass: campaigns list → click the named campaign.

### Step 5 — Clone the source send

Click the tool's clone / duplicate action. The clone is named
with the tool's default suffix (most use `(clone)` or
`Copy of...`). Capture the clone's name and URL.

**Safety check.** Verify that `(clone)` (or the equivalent
suffix the tool uses) is in the clone's name. If it is NOT
— if the clone action silently overwrote the source instead of
creating a duplicate — abort the run, write to `_errors.md`,
and notify the operator. This protects scheduled non-clone
sends in the same folder from being deleted by a wrong-row
click in Step 8.

### Step 6 — Paste the test code into the clone, send to test segment

Open the clone for editing. Paste the captured test code into
the body using the tool's compose surface. The test code is
typically a tracking pixel or a unique header — paste it into
the body where the tool's documentation specifies.

For MailReach: paste the code into a body cell anywhere that
will pass through unchanged.

For GMass: paste the code into the body of the campaign draft.

Set the recipient to `test_segment`. The bot does NOT change
the From: address from `sender` — that came in on the clone
and is verified against `deliverability-config.md` Q1.

Click Send. Capture the send timestamp.

### Step 7 — Wait for the score, then navigate directly to the report URL

Wait the tool's typical propagation window. MailReach: ~2-3
minutes; GMass: similar. The wait is tool-specific and captured
in the skill's `propagation_wait_seconds` constant per tool
(default 180s; OPERATOR-overridable per run via a
`wait_override_seconds` argument).

After the wait, navigate via Chrome MCP DIRECTLY to the
`report_url` constructed in Step 3. **Do not pass through the
home page.** The home page reload would rotate the code and
you'd be looking at the wrong report.

Read the score from the report page. Common selectors:

- MailReach: a numeric `<span>` near the heading "Spam Score"
  or "Deliverability Score".
- GMass: the "Inbox Test Result" panel.

Capture as a numeric value (typically 0-10 scale).

### Step 8 — Clean up the clone

Navigate back to the campaigns / sends list. Find the clone
created in Step 5 by name (the captured name from Step 5 is
the truth — DO NOT search by partial match, which is what
collides with non-clone sends).

Re-verify the safety check from Step 5: confirm `(clone)` is
in the row's name BEFORE the delete action. Then delete.

### Step 9 — Log the score and comment back

**Log to tracker.** Append a row to `tracker_url` with:
- `date` — ISO date of the run
- `sender` — `sender`
- `segment` — `test_segment`
- `score` — captured numeric value
- `report_url` — `report_url`
- `clone_name` — clone name from Step 5

Tracker write path is tool-specific:
- Google Sheet: append-row API or Chrome MCP if no API access.
- Airtable: append-record via Airtable API or Chrome MCP.

**Comment back on the queue task.** When `task_url` is set,
post a comment using `comment_format` from
`deliverability-config.md` Q6, with placeholders filled
verbatim. When `task_url` is blank, skip this sub-step and
write a one-line note to
`/rockstarr-ai/05_published/ops/deliverability-<YYYY-MM>.md`
instead.

**LOW SCORE prefix.** When `score < low_score_threshold`,
prefix the comment with `LOW SCORE — `. Same prefix on the
daily-summary line. The weekly report bumps low-score runs to
the top of the deliverability section.

## Outputs

- One row appended to the tracker at `tracker_url`.
- One comment appended to the queue task at `task_url` (when
  set), formatted via `comment_format`.
- One row appended to
  `/rockstarr-ai/05_published/ops/deliverability-<YYYY-MM>.md`
  (one file per month, append-only).
- One row appended to the Deliverability sheet of
  `/rockstarr-ai/02_inputs/ops/ops-mirror.xlsx` (so
  `ops-weekly-report` can roll up scores without re-walking the
  tracker).

## Approvals

This skill executes its full nine-step flow without per-step
approval. The test pattern is well-defined and recurring;
inserting a per-step approval would block every weekly run on
operator availability.

The single human gate is the score-comment on the queue task.
The operator reads the comment in their task system and decides
whether the score warrants action. Low-score runs bubble up
visually via the `LOW SCORE` prefix and the weekly report.

## Failure modes

- **Code rotation snuck in.** The bot reloaded the home page
  somewhere it shouldn't have, or the tool rotated the code
  out-of-band. Step 7 reads a stale report. The score for the
  OLD code is logged; the new send hasn't been read yet.

  Mitigation: defensive — capture the code in Step 2, compute
  the URL in Step 3, and immediately navigate to it in a
  background tab to verify it returns 404 (no test against it
  yet) BEFORE Step 4. If it returns 200, the code is stale and
  the bot must abort + re-capture.

- **Clone safety check fails.** The clone action overwrote the
  source. Abort, write `_errors.md` with the source-name
  contamination detail, and notify the operator. Do NOT delete
  anything.

- **Score < low threshold.** Prefix the comment with `LOW SCORE`
  and bump in the weekly report. Do NOT auto-pause sending or
  auto-take any other action — the operator decides remediation.

- **Tracker append fails.** Write the score to the daily-summary
  file and `_errors.md`. The operator can paste the row into
  the tracker manually. The clone cleanup in Step 8 still runs
  — the run completed, just the logging step partially failed.

- **Clone source not found** (the named campaign was renamed or
  archived). Abort, write `_errors.md`, surface in chat:
  "Clone source `<name>` not found in <tool>. Run
  `capture-deliverability-config` to update Q4."

- **Browser session not authenticated.** The first DOM read
  bounces to the tool's login page. Abort, surface the
  authentication failure to the operator: "Reauthenticate
  <tool> in Chrome and re-run `run-deliverability-test`."

## What this skill does NOT do

- Does NOT change deliverability tool settings, sender warm-up
  state, domain authentication records, or anything outside
  the test send + clone cleanup. The tool's own admin surface
  is the operator's domain.
- Does NOT auto-pause outbound sending when a low score lands.
  The operator decides remediation.
- Does NOT batch multiple runs in one Chrome MCP session.
  Code rotation is per-run; mixing runs would cross-contaminate
  the captured codes.
- Does NOT consult `pitch.md` or `style-guide.md`. This skill is
  pure infrastructure — no prose generation.

## Re-runnability

Idempotent within a tool's design. Re-running this skill on the
same calendar day creates a new clone, runs a new test, and
appends a new tracker row. Multiple-runs-per-day is supported
but unusual — the typical cadence is weekly.
