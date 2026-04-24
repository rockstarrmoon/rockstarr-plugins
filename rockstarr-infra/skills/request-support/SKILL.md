---
name: request-support
description: |
  This skill should be used when the client says "email Rockstarr support", "I need help from R&M", "contact support", "file a support ticket", "something is broken, tell Rockstarr", "ask Jon and Rachel for help", or otherwise asks to get an issue to the Rockstarr AI team by email. Collects the client's description of the issue, gathers workspace context automatically (client_id, plugin versions, recent errors if relevant), shows the draft for approval, then sends it to ai_support@rockstarrandmoon.com via the send-notification helper. Reply-to is set to the client's own ROCKSTARR_NOTIFY_TO so Rachel or Jon can reply directly. Not for sending email to leads or teammates — that flows through the reply plugin.
---

# request-support

Give the client a one-step way to get an issue to the Rockstarr AI
support inbox from inside their Cowork workspace. Drafts the email,
shows it for approval, sends it to
`ai_support@rockstarrandmoon.com` via `send-notification`, and logs
the send so both sides have a record.

## When to run

- Client says any of: "email Rockstarr support", "contact support",
  "file a support ticket", "tell R&M what's broken", "ask Jon/Rachel
  for help", "I need help from Rockstarr".
- A Rockstarr skill has hit a blocker the client cannot resolve
  themselves (corrupted workspace, missing entitlement, auth failure
  that isn't theirs to fix) and prompts the client: "Want me to email
  support about this?"

Do NOT run this for:

- Emailing the client's own leads, teammates, or external contacts.
  Those flow through `rockstarr-reply-<variant>` using the client's
  mailbox OAuth, not the Rockstarr mailer.
- Silent auto-send. This skill ALWAYS shows the draft to the client
  and waits for explicit approval before sending.

## Preconditions

- `/rockstarr-ai/00_intake/.rockstarr-mailer.env` exists and has
  `ROCKSTARR_MAILER_TOKEN`, `ROCKSTARR_CLIENT_ID`, and
  `ROCKSTARR_NOTIFY_TO` filled in. If any are missing or empty, abort
  with a clear message — the client needs to complete onboarding
  first. `send-notification` will also check this; we check here so
  we can fail fast before drafting.
- `/rockstarr-ai/client.toml` exists. Read `client_name` from it for
  the subject and signature. If missing, fall back to `ROCKSTARR_CLIENT_ID`.

## Inputs (from the client)

The client supplies, via chat or `AskUserQuestion`:

- `issue_summary` — a short one-line version of what's wrong.
- `issue_detail` — a longer free-form description. Walk the client
  through this if they only gave a headline.
- `urgency` — one of `blocker`, `high`, `normal`, `low`. Default `normal`.
  This only affects the subject-line prefix; the email still routes
  to the same support inbox.

If the client invoked the skill with only a brief phrase ("something
broke"), use `AskUserQuestion` to gather the above one question at a
time. If they pasted a long description up front, don't re-ask —
offer them an edit pass on the draft instead.

## Steps

1. Gather the client's description per the Inputs section above.

2. Collect workspace context automatically — the support team can
   triage faster with this up front. Capture:

   - `client_id` and `client_name` (from `client.toml` or env).
   - `rockstarr_infra_version` (from `client.toml`).
   - List of installed Rockstarr plugins you can see in the
     workspace's plugin settings, with their versions. Best-effort —
     if you cannot enumerate them, skip this bullet rather than
     blocking on it.
   - If the client's issue mentions a specific skill or draft, tail
     the last 20 lines of any relevant log file they point to (e.g.
     `/rockstarr-ai/05_published/_mailer.log`,
     `/rockstarr-ai/03_drafts/outreach/_errors.md`). Never include
     raw credentials or the mailer token.

3. Draft the email body. Structure it so the support team can scan
   it quickly:

   ```
   SUMMARY
   <issue_summary>

   DETAIL
   <issue_detail, verbatim from the client>

   WORKSPACE
   - client:            <client_name> (<client_id>)
   - rockstarr-infra:   <version>
   - installed plugins: <list or "(not captured)">
   - urgency:           <blocker|high|normal|low>

   CONTEXT (if gathered)
   <any log tails or file pointers, each fenced and labeled>

   — Sent from <client_name>'s Rockstarr AI workspace via /request-support
   ```

   Keep it plain text. Never include the mailer bearer token, the
   Resend API key, or any `NOTIFY_*` address in the body.

4. Compose the subject:

   ```
   [<URGENCY>] <client_name>: <issue_summary>
   ```

   where `<URGENCY>` is uppercase (`BLOCKER`, `HIGH`, `NORMAL`, `LOW`).
   Example: `[BLOCKER] Acme Corp: approve skill is throwing mailer auth errors`.

5. Show the client the full draft — subject and body — and ask for
   explicit approval to send. Offer three options: send as-is, edit
   first, or cancel. If the client edits, loop back to step 5 with
   the updated draft. Do NOT send without an affirmative "yes, send"
   (or equivalent) in the chat.

6. On approval, invoke `send-notification` with:

   ```
   to         = "ai_support@rockstarrandmoon.com"
   reply_to   = <ROCKSTARR_NOTIFY_TO from env>
   subject    = <subject from step 4>
   body_text  = <body from step 3>
   tag        = "support_request"
   ```

   `to` is explicit, so env-file routing rules don't apply — every
   support request from every client lands in the same R&M inbox.
   `reply_to` is set to the client's own notification address so
   Rachel or Jon can reply directly and the client sees it in the
   mailbox they already monitor.

7. On success, tell the client:

   - The `message_id` returned by the mailer (for their records).
   - That Rachel or Jon will reply to `<ROCKSTARR_NOTIFY_TO>`
     directly.
   - The local log line that was appended to
     `/rockstarr-ai/05_published/_mailer.log`.

   On failure, surface the specific error from `send-notification`
   and suggest the appropriate fix (re-auth, fix env file, wait and
   retry).

## What NOT to do

- Do NOT auto-send. Every support email ships only after explicit
  client approval in the chat. "Sending messages on behalf of the
  user" requires explicit permission per the safety rules.
- Do NOT include secrets in the body. No bearer tokens, no API keys,
  no raw OAuth tokens. If a log tail contains anything that looks
  like a credential, redact it as `<redacted>` before including.
- Do NOT bypass `send-notification`. All email from Rockstarr AI
  skills must go through the shared helper so the endpoint, auth,
  payload shape, and mailer log stay consistent.
- Do NOT use `notify_type` for this. Support always goes to the
  fixed R&M inbox, never to a client-side address.
- Do NOT retry a support email on the client's behalf if the first
  send failed. Tell them what went wrong, let them decide whether to
  re-try or escalate another way.

## Related

- `skills/_shared/send-notification/SKILL.md` — the mailer helper
  this skill wraps.
- Support inbox: `ai_support@rockstarrandmoon.com`. Needs to be
  live as a receiving alias on `rockstarrandmoon.com` before this
  skill ships. See the mailer README for DNS / routing setup notes.
