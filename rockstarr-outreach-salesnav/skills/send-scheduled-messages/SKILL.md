---
name: send-scheduled-messages
description: "This skill should be used in the daily outreach loop after generate-message-tasks, or when the user says \"send today's scheduled Sales Nav messages\", \"execute the messages due today\", or \"run the sequence sends\". It executes every message-step-N task and follow-up task due today by resolving {first_name} and {company} placeholders against the Leads row (with shape-aware parse-failure fallbacks: bare-name comma-led drops the prefix and capitalizes; greeting-led substitutes 'there'), sending the resolved body via Sales Nav messaging through Chrome MCP, logging each send to the Messages sheet, and marking the task done. Refuses to run if confirm-session has not passed in this run. Aborts an individual send if any literal {...} placeholder remains after substitution."
---

# send-scheduled-messages

Sends the sequence bodies that `generate-message-tasks` queued and the
2-day follow-ups that `send-approved-reply` queued. Bodies are read
from the approved campaign file at send time (or drafted fresh via
`rockstarr-reply` for follow-ups).

## When to run

- Daily loop, after `generate-message-tasks`, before `detect-replies`.
- On-demand via `force-send-today` (deferred V0.1).

## Preconditions

- `confirm-session` passed in this run.

## Inputs

- Tasks rows where `status = pending`, `due_date <= today`, and
  `type IN (message-step-1, message-step-2, message-step-3,
  follow-up)`.

## Behavior

1. **Sort due tasks.** Oldest `due_date` first, within a `due_date`
   group sort by `created_at`.
2. **For each task:**
   a. **Resolve body.**
      - `message-step-N`: read the approved campaign body at
        `04_approved/outreach/campaign-<slug>.md`, extract the
        matching step's copy, apply variable substitution per the
        rules in Step 2a-i below.
      - `follow-up`: call `rockstarr-reply:draft-reply` with the
        thread context — follow-up bodies draft fresh on the due-
        date, not at send-time, because the context can drift. If
        `rockstarr-reply:draft-reply` is not yet available, fall back
        to a Cowork notification to the client asking them to draft
        manually. Log the fallback to `_errors.md`.
      - If the draft-reply path is in use, the follow-up body must
        be approved by the client through `rockstarr-reply` before
        this skill sends. Do **not** send an unapproved follow-up.

   ### 2a-i. Variable substitution rules

   Two placeholders are supported in approved campaign bodies:
   `{first_name}` (sourced from `Leads.name`) and `{company}`
   (sourced from `Leads.company`). `draft-icp-campaign` only emits
   `{first_name}` today — `{company}` support is preserved in this
   skill so future Sequence Rule extensions can use it without a
   send-side change. Both placeholders are case-sensitive literal
   tokens.

   **`{first_name}` — primary parse.**
   1. Read `Leads.name` for this lead.
   2. Strip leading/trailing whitespace.
   3. Split on whitespace. Take the first token.
   4. Strip leading/trailing non-alphabetic characters (so `"(Bob)"`
      → `"Bob"`, but `"O'Neill"` → `"O'Neill"` — interior apostrophes
      stay).
   5. The result is `first_name`. The substitution succeeds if the
      result is non-empty AND contains at least one alphabetic
      character.

   **`{first_name}` — substitute on success.**
   Replace every occurrence of the literal token `{first_name}` in
   the body with the parsed first-name. Do NOT case-fold the
   parsed name — `Leads.name` is the source of truth. `"jane"` in
   the workbook substitutes as `"jane"`, not `"Jane"`. Operators
   keep the workbook tidy.

   **`{first_name}` — fallback on parse failure.**
   Parse failure means `Leads.name` is empty, whitespace-only, or
   contains no alphabetic characters after the strip in step 4.
   On parse failure, behavior depends on the *opening shape* of
   the message body. Detect the shape by examining the body's
   first non-whitespace characters:

   - **Bare-name comma-led** (`^\s*\{first_name\},\s*`). The
     placeholder IS the opener, with nothing before it but
     whitespace. On parse failure, drop the entire matched prefix
     (`{first_name},` plus the trailing whitespace) AND capitalize
     the first alphabetic character of the remaining body. Example:
     `"{first_name}, saw your post on Q4 hiring."` →
     `"Saw your post on Q4 hiring."`. The opener becomes a clean
     sentence; no awkward leading comma, no `"Hi there,"` graft
     onto a body that wasn't written for one.
   - **Greeting-led** (`^\s*\S+\s+\{first_name\},\s*` — one or more
     non-whitespace words before the placeholder). Common shapes:
     `"Hi {first_name}, ..."`, `"Hey {first_name}, ..."`,
     `"Hello {first_name} — ..."`. On parse failure, replace
     `{first_name}` with the literal word `there`. Example:
     `"Hi {first_name}, saw your post on Q4 hiring."` →
     `"Hi there, saw your post on Q4 hiring."`. The greeting
     stays intact.

   - **Mid-sentence occurrence** (placeholder appears anywhere
     other than the opener — i.e., neither pattern above
     matches the placeholder's position). On parse failure,
     substitute `there`. The Sequence rules in `draft-icp-campaign`
     don't emit mid-sentence `{first_name}` today, so this is a
     defensive default. If a body has `{first_name}` in *both* the
     opener AND mid-sentence, apply the opener rule to the opener
     occurrence and `there` to the mid-sentence one(s).

   **`{company}` — parse + substitute.**
   1. Read `Leads.company`. Strip whitespace.
   2. If non-empty, replace every `{company}` token with the
      verbatim value. Do not case-fold.
   3. On failure (empty / whitespace-only `Leads.company`), drop
      the placeholder along with the single space immediately
      following it (so `"at {company} now"` → `"at now"` is
      avoided — pattern: replace `{company}\s` with empty
      string; if the placeholder is the last token in a
      paragraph, replace `\s?{company}` with empty string). The
      remaining sentence may read awkwardly; if that happens
      consistently, fix the campaign body, not the substitution.

   **Post-substitution residue check.**
   After all substitutions complete, scan the resolved body for
   any remaining literal `{` character. If found, abort the send
   for THIS task only: log to `_errors.md` with the lead, the
   campaign, the unresolved placeholder, and the resolved body
   so the operator can audit. Mark the task `cancelled` with
   reason `unresolved_placeholder`. Do not flip Leads.state.
   Continue to the next task — one bad placeholder doesn't kill
   the whole loop. Approved-campaign hygiene is the operator's
   job; this check is the safety net.
   b. **Open the Sales Nav thread** for this lead via Chrome MCP.
      Navigate to `https://www.linkedin.com/sales/messaging/` and
      find the thread (or open via the lead's profile panel). If
      the thread does not exist (lead removed the connection, etc.):
      log to `_errors.md`, mark the task `cancelled` with reason
      `thread_missing`, flip Leads.state to `opted-out`, cancel
      remaining sequence tasks for this lead.
   c. **Send the body.** Paste into the message composer, submit.
      Verify the message appears in the thread after send.
   d. **Write to Messages.** Row per send:
      - `date` = now (ISO timestamp)
      - `lead_url`
      - `campaign_slug`
      - `step` = `message-step-1|2|3` or `follow-up`
      - `body_sent` = exact body submitted (post-substitution)
   e. **Mark the task done.** `status = done`, `completed_at = now`.
   f. **Polite rate.** 25–45 seconds random wait between sends.
3. **Save the workbook.**
4. **Log to publish-log.** Append a per-campaign summary line to
   `/05_published/outreach/<today>.md`:
   `send-scheduled-messages — <N> sends across <M> campaigns
   (<slug-a>: <n_a>, <slug-b>: <n_b>)`.

## Output

- `sent` — total messages sent
- `per_campaign` — slug → count
- `skipped_thread_missing` — count
- `skipped_unapproved_followup` — count (waiting on rockstarr-reply
  approval)
- `skipped_unresolved_placeholder` — count (Step 2a-i residue check
  fired; task left `cancelled` with reason
  `unresolved_placeholder` and the operator notified via
  `_errors.md`)
- `first_name_fallback_bare` — count of sends where the bare-name
  comma-led fallback fired (placeholder dropped + first word
  capitalized)
- `first_name_fallback_greeting` — count of sends where the
  greeting-led fallback fired (`there` substituted)

## What NOT to do

- Do not send a `follow-up` before its body has been approved
  through `rockstarr-reply`. Per-reply gate is non-negotiable.
- Do not rewrite the sequence body. Variable substitution only —
  `{first_name}` and `{company}` per Step 2a-i. If the body reads
  awkwardly for a specific lead after substitution, skip + log; the
  client can edit the approved campaign and re-register.
- Do not pre-resolve `{first_name}` into a real name in the
  approved campaign file. Substitution is the send-side's job. A
  hard-coded name in `04_approved/outreach/campaign-<slug>.md`
  means every lead gets the same name. If you find a hard-coded
  name in a campaign body, surface it to the client and have them
  re-draft + re-approve.
- Do not send a body that still contains a literal `{first_name}`,
  `{company}`, or any `{...}` placeholder after substitution. The
  post-substitution residue check in Step 2a-i catches this; if it
  fires, abort the task and log. Sending an unresolved placeholder
  to the lead is the worst kind of bot tell.
- Do not "improve" the bare-name fallback by injecting `Hi there,`
  or any greeting graft. The bare-name shape was authored *without*
  a greeting on purpose — adding one on parse failure changes the
  voice. Drop and capitalize per Step 2a-i; do not substitute a
  greeting.
- Do not silently substitute `there` on bare-name parse failure.
  That's the greeting-led path's fallback, not the bare-name
  path's. The two shapes have different fallbacks for a reason.
- Do not send outside the configured run time window. This skill
  expects to be invoked by the daily loop at the client's
  `outreach_daily_run_time`.
- Do not continue the sequence after a reply. `detect-replies`
  cancels future `message-step-N` tasks for that lead; trust it.
