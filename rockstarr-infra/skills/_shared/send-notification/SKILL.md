---
name: send-notification
description: |
  Cross-bot helper. Sends a single email from Rockstarr AI to the client through the Rockstarr mailer. Use when another skill says "notify the client", "send the approvals digest email", "ping the client about the new draft", "let them know a reply landed", or otherwise needs to emit an out-of-band email from a Rockstarr bot to the human running the workspace. Wraps the POST to mail.rockstarrandmoon.com/send with payload validation, bearer lookup, error handling, and a local log so individual bot skills do not re-implement the contract. Not for sending email on behalf of the client to third parties — that flows through the reply plugin using the client's own mailbox OAuth.
---

# send-notification

Send one email from Rockstarr AI to the client via the Rockstarr
mailer at `https://mail.rockstarrandmoon.com/send`.

Every bot that needs to nudge a human routes through here. Keeps the
endpoint URL, bearer lookup, payload shape, and retry policy in one
place so individual skills do not drift.

## When to run

- A bot finishes a draft and wants to tell the client it is ready for
  review.
- The daily approvals digest is assembled and needs to go out.
- An outreach reply has landed and needs a fast-turn response from the
  client.
- Any other out-of-band notification the client should see in email
  rather than only in the workspace.

Do NOT use this to send email ON BEHALF of the client to leads or
teammates. That flows through the `rockstarr-reply-<variant>` plugin
using the client's own Gmail or Outlook OAuth.

## Preconditions

- `/rockstarr-ai/00_intake/.rockstarr-mailer.env` exists on the local
  machine with at minimum:

  ```
  ROCKSTARR_MAILER_TOKEN=<bearer>
  ROCKSTARR_CLIENT_ID=<client slug, e.g., rockstarr-internal>
  ROCKSTARR_NOTIFY_TO=<default recipient for this workspace>
  # optional:
  ROCKSTARR_NOTIFY_URGENT_TO=<recipient for notify_type=urgent>
  ```

  The token is the shared Rockstarr mailer bearer, issued during
  onboarding. If the file or `ROCKSTARR_MAILER_TOKEN` /
  `ROCKSTARR_CLIENT_ID` is missing or empty, abort and tell the user to
  complete onboarding (the `scaffold-client` skill seeds a template;
  Rachel or Jon fills in the values). `NOTIFY_*` values are only
  required at recipient-resolution time — see step 2.

- Network reachability to `mail.rockstarrandmoon.com` (Cloudflare
  anycast — almost always fine).

## Inputs

The calling skill supplies:

- `to` — OPTIONAL. One email address, or an array of addresses. If
  supplied, this wins over every env-file default.
- `notify_type` — OPTIONAL. One of `urgent` or `default` (default =
  `default`). Selects which env-file address is used when `to` is not
  supplied. Unknown values fall back to `default`.
- `subject` — plain-text subject line.
- `body_text` — required. Plain-text body.
- `body_html` — optional. HTML body. If provided, most mail clients
  will show this instead of `body_text`.
- `tag` — optional. Short tag used for Resend analytics, e.g.,
  `approvals_digest`, `draft_ready`, `reply_needed`.
- `reply_to` — optional override; default is the mailer's
  `DEFAULT_REPLY_TO` (`hello@rockstarrandmoon.com`).

## Steps

1. Parse `/rockstarr-ai/00_intake/.rockstarr-mailer.env` line-by-line.
   Do NOT `source` it blindly. Read only `KEY=VALUE` lines, ignoring
   blank lines and `#` comments. Extract:

   - `ROCKSTARR_MAILER_TOKEN` — required, non-empty.
   - `ROCKSTARR_CLIENT_ID` — required, non-empty.
   - `ROCKSTARR_NOTIFY_TO` — required for any send where the caller
     did not pass an explicit `to`. May be empty if caller always
     passes `to`.
   - `ROCKSTARR_NOTIFY_URGENT_TO` — optional.

   Ignore any unknown keys (e.g. a legacy `ROCKSTARR_NOTIFY_DIGEST_TO`
   left in place from an older scaffold).

   Abort with a clear message if `ROCKSTARR_MAILER_TOKEN` or
   `ROCKSTARR_CLIENT_ID` is missing or empty.

2. Resolve the recipient — this is the `to` value that ends up in the
   payload. Apply the first rule that matches, top to bottom:

   1. Caller passed `to` → use it verbatim (string or array).
   2. `notify_type == "urgent"` and `ROCKSTARR_NOTIFY_URGENT_TO` is set
      and non-empty → use it.
   3. `ROCKSTARR_NOTIFY_TO` is set and non-empty → use it.
   4. Otherwise → abort. Tell the user that no recipient was resolved
      and they either need to pass an explicit `to` or fill in
      `ROCKSTARR_NOTIFY_TO` (and optionally `ROCKSTARR_NOTIFY_URGENT_TO`)
      in `/rockstarr-ai/00_intake/.rockstarr-mailer.env`.

   `ROCKSTARR_NOTIFY_URGENT_TO` only kicks in for `notify_type=urgent`;
   leaving it empty is the normal case and routes urgent items to the
   default. Unknown `notify_type` values fall through to rule 3 as if
   the caller had passed `default`.

3. Assemble the JSON payload:

   ```json
   {
     "client_id": "<from env>",
     "to": "<resolved recipient — string or array>",
     "subject": "<subject>",
     "text": "<body_text>",
     "html": "<body_html, if provided>",
     "reply_to": "<reply_to, if provided>",
     "tag": "<tag, if provided>"
   }
   ```

   Omit optional keys that were not supplied. Do not send empty
   strings — prefer to drop the key.

4. Write the payload to a temp file (to avoid shell-escaping issues
   with quotes, emojis, or embedded newlines) and POST via Bash:

   Capture curl's exit code AND verbose output so the response
   interpreter (step 5) can distinguish a sandbox-level egress block
   from a Worker-level error:

   ```bash
   payload=$(mktemp)
   cat > "$payload" <<'JSON'
   <payload>
   JSON
   curl_out=$(curl -sS -v -X POST https://mail.rockstarrandmoon.com/send \
     -H "Authorization: Bearer $ROCKSTARR_MAILER_TOKEN" \
     -H "Content-Type: application/json" \
     --data @"$payload" 2>&1)
   curl_exit=$?
   rm -f "$payload"
   ```

5. Interpret the response. Handle the sandbox-egress case BEFORE the
   generic error cases — its failure mode (curl exit 56 plus an
   `HTTP/1.1 403 Forbidden` from the CONNECT proxy, often carrying an
   `X-Proxy-Error: blocked-by-allowlist` header) looks superficially
   like an unrelated 403 but is actually a Cowork-level block, not a
   Worker-level one.

   - **Egress blocked by Cowork allowlist** — curl exit 56 AND the
     verbose output contains `blocked-by-allowlist` (or
     `403 Forbidden` specifically from `CONNECT`). Do NOT retry. Show
     the user this exact message:

     > The Cowork sandbox can't reach `mail.rockstarrandmoon.com` yet.
     > To fix:
     >
     > 1. Open **Cowork → Settings → Capabilities**
     > 2. Add `mail.rockstarrandmoon.com` to the Network Allowlist
     > 3. Save
     > 4. **Open a new Cowork conversation** — the allowlist is pinned
     >    at session start, so your current chat can't pick up the
     >    change
     >
     > Then ask the new conversation to retry this email.

   - `{"ok": true, "message_id": "..."}` — success. Return the
     `message_id` to the caller.
   - `{"error": "invalid_payload", "detail": "..."}` — caller bug.
     Show `detail` to the user, abort, do not retry.
   - `{"error": "unauthorized"}` — bearer is wrong or expired. Tell the
     user to fix `.rockstarr-mailer.env`. Do not retry.
   - `{"error": "resend_failed", ...}` — Resend rejected the send
     (bounce, suppression, verification issue). Show `detail` so the
     user can triage in the Resend dashboard. Do not auto-retry.
   - Non-2xx with no JSON body AND not an egress block — transient
     hiccup. Retry once after a two-second wait. If it still fails,
     abort and surface the HTTP status.

6. Append one line to `/rockstarr-ai/05_published/_mailer.log` (create
   the file if missing):

   ```
   <ISO timestamp>  <client_id>  <tag or "(none)">  <message_id>  <first recipient>
   ```

   This log is what the approvals-digest skill and audit reviews
   reconcile against Resend's own delivery log.

## What NOT to do

- Do NOT send email from individual bot skills directly. Always go
  through this helper so endpoint, auth, payload shape, and log
  format stay in one place.
- Do NOT write the bearer token into any persistent Rockstarr file
  other than `.rockstarr-mailer.env`. It does not belong in
  `client-profile.md`, `stack.md`, `client.toml`, or any committed
  plugin asset.
- Do NOT retry `unauthorized` or `invalid_payload`. Those are not
  transient.
- Do NOT batch. One POST per email. Rate limits are generous and
  batching is a V0.1 concern at best.
- Do NOT send email without an explicit `tag` for recurring
  notifications (digests, reply-ready pings). The tag is how we
  measure open / click rates per flow in Resend.
- Do NOT hard-code recipient addresses in calling skills. Prefer
  `notify_type` and let this helper resolve the address from the env
  file. Passing an explicit `to` is reserved for one-off cases (a
  single named stakeholder for this message, a test, a forward).

## Example call shapes

Default-path notification (common case — no `to` supplied, routes to
`ROCKSTARR_NOTIFY_TO`):

```
Invoke send-notification with:
  subject   = "3 drafts awaiting your review"
  body_text = "Open your workspace — 3 items are in /04_approved/ queue."
  tag       = "approvals_digest"
```

Urgent notification (routes to `ROCKSTARR_NOTIFY_URGENT_TO` if set,
otherwise the default):

```
Invoke send-notification with:
  notify_type = "urgent"
  subject     = "Reply from Jane Doe needs your review"
  body_text   = "Sales Nav thread: …"
  tag         = "reply_needed"
```

Explicit one-off recipient (overrides env routing entirely — used by
`request-support` to route to `ai_support@rockstarrandmoon.com`, and by
smoke tests):

```
Invoke send-notification with:
  to        = "jon@rockstarrandmoon.com"
  subject   = "Mailer smoke test"
  body_text = "If you see this, the pipe is working."
  tag       = "smoke_test"
```

## Related

- Mailer Worker source + runbook:
  `rockstarrmoon/rockstarr-mailer` on GitHub (local path during
  development: `~/Desktop/Claude/RockstarrAI/rockstarr-mailer/`).
- Onboarding template seed: `scaffold-client` creates
  `.rockstarr-mailer.env` with placeholders; Rachel or Jon pastes the
  real token value during client onboarding.
- Planned consumer: `skills/approvals-digest/` (see
  project memory `project_approvals_digest.md`).
