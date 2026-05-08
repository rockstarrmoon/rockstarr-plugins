---
name: process-call-transcript
description: "This skill should be used when the user says \"process the call recap for [task-id]\", \"post-call follow-up for [name]\", \"process the transcript for the [name] call\", \"do the post-call write-up for [name]\", \"run the post-call queue\", or when a new task with the configured ops_post_call_queue_tag lands in ops_task_system (event-driven webhook trigger when supported, or scheduled queue walk otherwise). Walks the post-call queue one task at a time. Per task: reads the transcript via ops_meeting_recorder, extracts the per-client field set, hands a structured CRM update bundle to the active rockstarr-ops-bot-crm variant for execution (or surfaces a manual punch list when no CRM ops bot is installed), hands a recap-note draft request to rockstarr-reply with intent_hint=post-call-recap, comments back on the queue task using the configured comment format. Each task gets a per-CRM-write gate before its update bundle ships — no batch processing. The recap note also flows through rockstarr-reply's send-it gate when the operator wants it sent as a follow-up email vs. only pasted into the CRM."
---

# process-call-transcript

Post-call queue walker. Reads transcripts, extracts CRM update
bundles, drafts recap notes, comments back on queue tasks. Runs
one task at a time with a per-task per-CRM-write gate.

The whole reason this workflow survives real operator use is the
human gate. Every shortcut to remove the human (auto-approve all
recap notes, auto-write all CRM bundles in one batch, auto-skip
the comment) has cost misfires in the past.

## When to run

- Event-driven, when a new task with the configured
  `ops_post_call_queue_tag` lands in `ops_task_system`. Some
  task systems support webhooks (ClickUp, Asana). When they
  don't, the bot polls the queue on its own scheduled cadence
  (default every 4 hours during business hours).
- On demand by the operator with the trigger phrases above.

## Preconditions

- `00_intake/pitch.md`, `00_intake/icp-qualifications.md`,
  `00_intake/style-guide.md` exist.
- `00_intake/ops-call-framework.md` exists — the per-client
  field set the bot extracts is documented here (or in a
  sibling `ops-fields.md` file when the operator chose to
  separate fields from the framework).
- `stack.md` has `ops_meeting_recorder`, `ops_task_system`,
  `ops_post_call_queue_list`, `ops_post_call_queue_tag`,
  `ops_post_call_comment_format` set.
- `rockstarr-reply` is installed (HARD — recap-note drafting
  lives there).
- The active `rockstarr-ops-bot-<crm>` is installed (SOFT — when
  missing, CRM bundles become manual punch lists).

## Inputs

- `task_id` OR `task_url` — when the operator names a single
  task. When neither, the bot walks the queue.
- `walk_queue` — boolean, default `true` when no task is
  named. The bot walks newest-first and processes until it
  hits a task that fails the per-CRM-write gate (operator
  paused) OR runs out of tasks.

## Behavior

### Step 1 — Resolve the task

Read the task from `ops_task_system`. Capture:

- Task title, description, comments to date.
- Tags applied (the queue tag confirms scope).
- Linked attendee email or contact reference (most task systems
  let the operator paste a CRM contact link or email into the
  task description).
- The recorder transcript link (most task systems let the
  recorder integration auto-paste the transcript URL).

### Step 2 — Read the transcript

Via Chrome MCP, navigate to the recorder transcript URL. Pull:

- Full transcript text.
- The recorder's auto-extracted action items (if the recorder
  has a copilot / summary).
- Verbatim quotes for the per-client field set.

The per-client field set is captured in `ops-call-framework.md`
(or `ops-fields.md` when separated). It typically covers:

- Lead status (e.g., `engaged`, `cautious`, `not-now`,
  `disqualified`).
- Pain points named (verbatim).
- Voiced objections (verbatim).
- Pricing language used.
- Commitments owed (operator-owed vs. lead-owed).
- Next steps agreed (verbatim).
- Follow-up window agreed (verbatim).

V0.1 supports any field set — the bot extracts whatever
fields the framework names. Field count is open-ended.

### Step 3 — Build the structured CRM update bundle

The bundle shape (from the cross-bot contract documented in
the rockstarr-ops README):

```
{
  contact_id: <CRM contact ID — resolved from the task linked
    contact, or by lookup against name + email>,
  field_updates: {
    <field_name>: <value>,
    ...
  },
  note_payload: {
    sections: [
      { heading: "Pain points", body: "<verbatim quotes>" },
      { heading: "Voiced objections", body: "<verbatim quotes>" },
      ...
    ],
    links: [
      { label: "Recorder transcript", url: "<URL>" },
      { label: "Source task", url: "<task URL>" },
    ]
  },
  status_transition: <stage value if the field set named a
    status change, else null>
}
```

When `contact_id` cannot be resolved (lead is new — no CRM
record), the bundle's `contact_id` field becomes a creation
payload:

```
contact_id: {
  first_name: <verbatim>,
  last_name: <verbatim>,
  email: <verbatim>,
  phone: <verbatim or null>,
  business_name: <verbatim>,
  source: "post_call_processing"
}
```

The active `rockstarr-ops-bot-<crm>` knows how to handle both
shapes (existing-contact-update vs. new-contact-create).

### Step 4 — Build the recap-note draft request

Build the standard `rockstarr-reply` handoff bundle with
`intent_hint=post-call-recap`:

```
{
  channel: email-<ops_email_platform variant>,
  thread: <transcript text + any prior thread context>,
  persona: <operator's persona for this client>,
  icp_verdict: <re-run from current intel>,
  matching_rule: <verbatim>,
  lead: { url, name, company, title, campaign_slug },
  intent_hint: post-call-recap,
  intent_context: <the field set's verbatim section bodies +
    the operator-owed commitments>,
  batch_context: false
}
```

`rockstarr-reply` drafts a recap note that:

- Opens with a thank-you for the call.
- Lists the operator-owed commitments (so the lead has them
  in writing).
- Restates the next steps agreed.
- Closes with the agreed follow-up window.

The recap note is structured as an EMAIL — the lead receives it
in their inbox. It's also pasted INTO the CRM as a note (via
the bundle's `note_payload`), so the CRM record carries the
same artifact.

### Step 5 — Per-CRM-write gate

Surface the bundle to the operator before execution:

> Recap for **<lead_name> @ <lead_company>** ready.
>
> - CRM updates: <field count> fields, <new contact OR
>   existing contact ID> + status transition to <X>.
> - Note: pasting <N> sections + <M> links.
> - Recap email draft: see `present-for-approval` flow next.
>
> Reply `approve` to write to CRM, `edit fields` to tune
> the bundle, `skip` to drop this task.

The operator inspects the field updates one at a time if
needed. `edit fields` triggers a redraft of the bundle from
the same transcript with operator-supplied corrections. `skip`
drops the task — no CRM write, no recap, comment back as
"skipped by operator".

On `approve`, the bundle is dispatched to the active
`rockstarr-ops-bot-<crm>`'s write-skill. The CRM ops bot
returns `{ contact_id, success: true | false, errors: [...] }`.

### Step 6 — Manual punch list when no CRM ops bot

When no `rockstarr-ops-bot-<crm>` is installed, the bundle
becomes a manual punch list rendered in chat:

> No CRM ops bot installed. Paste these into the CRM record
> for **<lead_name>**:
>
> - <field_name>: <value>
> - <field_name>: <value>
>
> Note to add:
> ```
> <note_payload rendered as plain text>
> ```
>
> Reply `done` once you've pasted; reply `skip` to drop the
> task without pasting.

The recap-email draft step still runs — it's a separate
artifact, not blocked by CRM tooling.

### Step 7 — Recap email gate (rockstarr-reply)

The recap-note draft from Step 4 flows through
`rockstarr-reply`'s `present-for-approval`. Same `send it`
gate. The operator picks one of:

- `send it` — the recap email is sent (or staged as a draft
  in the email platform — the operator hits send if the
  client's `stack.md.ops_email_send_mode=draft`).
- An editing instruction — triggers a redraft.
- `skip email` — the email is not sent; the recap-note still
  exists as a CRM note via the bundle in Step 5.

The send path uses the operator's email platform — Gmail or
Outlook via API or Chrome MCP, per `stack.md.ops_email_platform`.

### Step 8 — Comment back on the queue task

After the CRM write succeeds (or the manual punch-list
"done" lands) AND the recap-email gate resolves, comment back
on the queue task using `ops_post_call_comment_format`. The
comment template is captured at install in
`capture-stack` or via the `ops_post_call_comment_format` field.

Default template:

```
{date} — Recap processed.
- CRM: {field_count} fields updated, contact {contact_id}.
- Note: pasted {section_count} sections.
- Recap email: {email_status}.
- Operator: {operator_name}.
```

The `email_status` placeholder fills with `sent`,
`drafted (operator hits send)`, or `skipped`.

### Step 9 — Status transition on the queue task

After the comment lands, transition the queue task's status
per `stack.md.ops_post_call_completed_status` (default
`done` or `complete`). The operator can override default
status in `stack.md` if their workflow uses a different
final status.

### Step 10 — Log the processing run

Write a row to
`/05_published/ops/post-calls-<YYYY-MM>.md`:

```markdown
## <YYYY-MM-DD> — <lead_name> @ <lead_company>

- Task: <task URL>
- Recorder transcript: <URL>
- Field updates: <count>
- Recap email: sent | drafted | skipped
- CRM status transition: <X> → <Y>
- Manual punch list: <yes / no>
- Operator: <operator name>
```

Also append a row to the PostCalls sheet of
`/02_inputs/ops/ops-mirror.xlsx`.

## Outputs

- One CRM contact updated (or created) per task.
- One recap note pasted into the CRM record.
- One email draft (or sent email) per task in the operator's
  email platform.
- One comment + one status transition on the queue task.
- One row in `/05_published/ops/post-calls-<YYYY-MM>.md`.
- One row in the PostCalls sheet of `ops-mirror.xlsx`.

## Approvals

- **Per-CRM-write gate (Step 5).** Mandatory. Operator
  approves field set + recap-note body before execution.
- **Per-recap-email gate (Step 7).** Mandatory. Inherited
  from `rockstarr-reply`'s `present-for-approval`.
- **No batch processing.** Each task gets both gates
  individually. The walk loop pauses on operator
  intervention; it does NOT auto-resume to the next task
  without a fresh trigger phrase or a task-system event.

## Failure modes

- **Recorder transcript not yet ready.** Some recorders take
  10-30 min post-call to publish the transcript. The bot
  catches this (the URL returns 404 or the recorder's UI
  shows "processing") and writes `_errors.md` + skips the
  task. The next queue walk picks it up.
- **Linked contact missing on the task.** Surface in chat:
  "Task `<title>` has no linked contact. Add the contact
  link or paste an email + I'll resolve from there." The
  operator updates the task; the bot retries.
- **CRM write fails** (validation error, required field
  missing). The bundle returns `success: false` with errors.
  Surface to the operator: "CRM write rejected: <errors>.
  Tune the bundle and re-approve, or skip." Do NOT silently
  drop.
- **Verbatim quote contains PII** (someone the lead named
  on the call who shouldn't be in the CRM). Surface in
  chat at the per-CRM-write gate: "This quote contains
  potential PII — review before approving." Operator
  decides to keep / redact.
- **Multiple recorder transcripts for the same call** (the
  recorder accidentally captured both legs of a recorded
  Zoom). Surface in chat, ask the operator to pick the
  canonical transcript, do not guess.
- **Field set in `ops-call-framework.md` doesn't match the
  CRM's actual fields.** Surface a soft warning — the bundle
  ships with whatever the framework named; the CRM ops bot
  silently drops fields it doesn't recognize. Recommend
  re-running `capture-call-framework` to align.

## What this skill does NOT do

- Does NOT batch-process the queue. Each task is its own
  flow.
- Does NOT write to the CRM directly. Every CRM mutation
  routes through the active `rockstarr-ops-bot-<crm>`.
  Routing around the CRM ops bot is a P0 audit-trail break.
- Does NOT skip the recap email when the operator says `skip
  email` AND the bundle would have been a CRM-only update.
  The CRM update still ships; the email is the only thing
  skipped.
- Does NOT auto-approve recap-email drafts on a redraft. The
  gate is per-version: every redraft re-presents.
- Does NOT consult `style-guide.md` directly. Voice lives in
  `rockstarr-reply`'s draft path.
- Does NOT generate any stop-slop pass on the recap email.
  `rockstarr-reply` runs it inside its draft path; this
  skill just hands off.
