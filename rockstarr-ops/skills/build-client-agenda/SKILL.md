---
name: build-client-agenda
description: "This skill should be used when daily-call-prep classifies a calendar event as a recurring client catch-up, or when the user says \"build the agenda for [client]\" or \"catch-up agenda for [client]\". Produces ClientAgenda_[client]_[date].docx: action items from the most recent recorded call (split 'you owe them' vs 'they owe you', overdue flagged), out-for-review tasks (5+ business-day stale flagged), coming-up-for-production in next 14 days, open questions, 3-5 quick-reference bullets. Operator-facing — bypasses stop-slop."
---

# build-client-agenda

Per-client recurring catch-up agenda. Produces a docx file at
`/03_drafts/ops/agendas/ClientAgenda_[client]_[YYYY-MM-DD].docx`
that the operator reads at the top of the catch-up call.

The doc is **operator-facing** — bypasses stop-slop. The agenda
is what makes a 30-minute catch-up feel like 5 minutes of
useful talk: the operator and the client both walk in knowing
exactly what's open, what's stale, and what's coming.

## When to run

- Auto, by `daily-call-prep` when an event on today's calendar
  classifies as a recurring client catch-up (the event title
  matches a row in `ops_client_roster_source` — see the
  client-roster matching logic below).
- On demand by the operator via the trigger phrases above.

## Preconditions

- `00_intake/pitch.md` — only used for context if the agenda
  references the offer or pricing. Most agendas don't.
- `stack.md` is set with at minimum `ops_meeting_recorder` and
  `ops_task_system`. When `ops_task_system=none`, the agenda's
  out-for-review and coming-up-for-production sections are
  omitted (the agenda becomes recorder-action-items-only).
- `00_intake/client-roster.md` exists when
  `ops_client_roster_source=static_list`. Otherwise the roster
  is read live from the task system.
- Chrome MCP authenticated against the recorder + task system.

## Inputs

From the caller (orchestrator or operator):

- `client_name` — required.
- `event_start` — calendar event start time (ISO).

## Behavior

### Step 1 — Resolve the client

Match `client_name` against `ops_client_roster_source`:

- `static_list` → grep `00_intake/client-roster.md` for the
  closest match by name. Use the matched row's task-system
  identifier (project ID, list ID, tag, etc.) for the rest of
  the run.
- `task_system_clients_list` → look up the live clients list
  in the task system (e.g., the ClickUp `Clients` list, the
  Asana `Clients` portfolio) and match by name.

If multiple matches, surface in chat: "Multiple clients match
`[name]` — pick one." If no match, surface "Client `[name]` not
in roster — add via `capture-stack` or
`00_intake/client-roster.md`." Do not guess.

### Step 2 — Pull action items from the most recent recorded call

Via Chrome MCP, navigate to `ops_meeting_recorder`. Search for
the most recent recorded call with this client (matching by
client name + recent dates). The recorder's search / copilot
extracts action items.

Tag each action item:

- **You owe them** — operator owes the client. Quote the
  verbatim line that captures the commitment.
- **They owe you** — client owes the operator.
- **Overdue** — when the action item has an explicit deadline
  that has passed. The recorder typically captures deadlines
  as "by Friday", "by end of week", etc. The bot resolves
  these to dates against the call date and flags items past
  their resolved date.

Render in the docx as two columns: "You owe them" / "They owe
you". Overdue items get a 🚩 icon and bold-face.

If the recorder transcript is unavailable, render this section
with "RECORDER UNAVAILABLE — operator surfaces from memory."
Do not invent action items.

### Step 3 — Pull out-for-review tasks

When `ops_task_system != none`: query the task system for
tasks tagged with this client + status = `out_for_review` (or
the equivalent `In Review`, `Awaiting Client`, etc., per the
task-system convention captured in `stack.md`).

For each task, capture:

- Task name
- Date the task moved to out-for-review
- Days-since-out-for-review
- Operator owner

**Stale flag.** When days-since-out-for-review ≥ 5 business
days, flag the task with 🚩 STALE. Five business days is the
hard-won default for "this has rotted long enough that the
client probably forgot about it". Configurable via
`stack.md.ops_review_stale_days` if a client wants a different
threshold.

Render in the docx as a table: Task / Out-for-review since /
Days / Owner / Stale.

### Step 4 — Pull coming-up-for-production tasks

When `ops_task_system != none`: query the task system for
tasks tagged with this client + due-date in the next 14 days
+ status NOT in [`done`, `archived`, `out_for_review`].

Group the tasks by ISO week (current week → +1 → +2):

- **This week** — due-date this week.
- **Next week** — due-date the following week.
- **Two weeks out** — due-date the week after.

Render in the docx as three sub-sections, one per week.

### Step 5 — Pull open questions blocking production

When `ops_task_system != none`: query the task system for
tasks tagged with this client + status = `blocked` (or the
equivalent `Blocked`, `Question`, `Awaiting`, etc., per
`stack.md`).

For each blocked task, capture the task's most recent comment
or note (typically a question the operator asked the client).

Render in the docx as a bullet list of "We're blocked on:"
items.

### Step 6 — Build 3-5 quick-reference bullets

Synthesize 3-5 bullets the operator can skim mid-call:

- Highest-priority overdue commitment from Step 2.
- Stalest review item from Step 3.
- Earliest production deadline from Step 4.
- Most-pressing blocker from Step 5.
- Anything from the recorder transcript that surfaces as a
  client-relationship note worth raising (a kudos the client
  gave, a frustration they expressed).

Render in the docx as a callout box on cover-page-1.

### Step 7 — Write the docx

Write `/03_drafts/ops/agendas/ClientAgenda_[client]_[YYYY-MM-DD].docx`.

The docx structure:

1. **Cover** — `Catch-up Agenda — [client]` header. Quick-
   reference callout (Step 6 bullets). Event start time.
2. **Action items from last call** — two columns (Step 2).
   Overdue flagged.
3. **Out for review** — table (Step 3). Stale flagged.
4. **Coming up for production (next 14 days)** — three
   sub-sections by week (Step 4).
5. **Blockers / open questions** — bullet list (Step 5).
6. **Notes from last call** — any recorder-extracted notes
   that aren't action items (kudos, frustrations, things the
   client mentioned in passing).

### Step 8 — Sidecar markdown

Write a `.md` sidecar with the same shape as the prep skills'
sidecars. The orchestrator reads it for the chat-summary line.

> **Sidecar template convention.** Real `---` in the actual
> file, not `# ---`.

```markdown
# ---
schema_version: 1
type: ops-client-agenda
client_name: <name>
event_start: <ISO>
last_call_at: <ISO from recorder>
recorder_unavailable: true | false
overdue_count: <integer>
stale_review_count: <integer>
production_this_week: <integer>
production_next_week: <integer>
production_two_weeks: <integer>
blocker_count: <integer>
generated_at: <ISO>
docx_path: <path to the docx>
# ---
```

## Outputs

- `/rockstarr-ai/03_drafts/ops/agendas/ClientAgenda_[client]_[date].docx`.
- Markdown sidecar at the same path with `.md` extension.

## Approvals

Operator-facing. No `send` step. The orchestrator archives the
docx + sidecar to `/05_published/ops/` after the meeting.

## Failure modes

- **Recorder transcript missing for the most recent call.**
  Render Section 2 with the unavailable callout. Other
  sections continue.
- **Task system unreachable** (auth dropped, API down). Render
  Sections 3-5 with "TASK SYSTEM UNREACHABLE — operator
  surfaces from memory." Section 2 + 6 still render from the
  recorder.
- **Client roster ambiguity.** Surface in chat, do not
  produce a doc against the wrong client.
- **No tasks at all in any of Sections 3-5** (a brand-new
  client). Render each section with "No items at this time"
  rather than omitting the section. The empty-state is
  meaningful — it tells the operator the agenda is light.
- **Stale-review threshold disputed by the client.** The
  operator can override per-run by passing
  `stale_review_days_override`. The default lives in
  `stack.md.ops_review_stale_days`.

## What this skill does NOT do

- Does NOT update task statuses. Read-only against the task
  system.
- Does NOT post the agenda in any chat tool, email, or
  shared doc. The operator owns distribution; this skill
  produces the local docx.
- Does NOT consult `style-guide.md`. The agenda is operator-
  facing internal.
- Does NOT mark items "discussed" / "completed" after the
  call. That's `process-call-transcript`'s job (it pulls the
  next call's transcript and updates the next agenda).
- Does NOT run stop-slop.
