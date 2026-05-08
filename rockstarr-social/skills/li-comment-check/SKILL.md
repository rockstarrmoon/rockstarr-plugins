---
name: li-comment-check
description: "This skill should be used when the user asks to \"run the LI comments\", \"check LinkedIn comments\", \"daily LinkedIn check\", \"comment check\", or names one of the configured managed LinkedIn accounts. Daily LinkedIn comment check and response workflow for one or more managed accounts. Walks the configured account list one at a time, verifies the correct account is signed in (identity gate via profile-photo alt-text), reads notifications + the account's recent activity, drafts in-voice responses to unanswered comments, gates every reply on operator approval before posting, and closes the matching ClickUp task on success when ClickUp closure is enabled in stack.md. Generic and config-driven: managed accounts (name, profile slug, voice notes, ClickUp task ID) are read from stack.md.li_comment_accounts[]. Refuses to send anything without explicit operator approval per comment."
---

# li-comment-check

Daily comment-monitoring and reply workflow for one or more
managed LinkedIn accounts. Each account is processed in
sequence. The operator (typically the strategist running the
session) handles browser sign-in switches between accounts; the
skill handles the rest.

The merged source for this skill is the BBN-internal Daily LI
Comment Check SOP and the canonical SKILL.md drafted alongside
it. The Rockstarr-framework version is generic — managed
accounts come from `stack.md`, not hard-coded — and the ClickUp
closure step is opt-in.

## When to run

- The scheduled daily run fires (default 08:30 client-local;
  configured in `stack.md.li_comment_check_cron`).
- User says "run the LI comments", "check LinkedIn comments",
  "daily LinkedIn check", "comment check".
- User names one of the configured managed accounts ("run
  Phil's comment check") — the skill scopes to that one
  account.

## Preconditions

- `/rockstarr-ai/00_intake/stack.md` has the LI-comment block:

  ```yaml
  li_comment_check_enabled: true
  li_comment_check_cron: "30 8 * * 1-5"   # default 08:30 weekdays
  li_comment_clickup_enabled: false       # default false; opt-in
  li_comment_accounts:
    - name: "Phil Katz"
      slug: "philipkatz1"               # linkedin.com/in/<slug>
      voice_notes: |
        Corporate, professional. Seasoned executive in his 70s.
        Measured, appreciative, authoritative. No slang, no
        exclamation-heavy enthusiasm.
      brand_kit_url: "..."              # optional
      clickup_task_id: "868fzfgrf"      # required only if clickup enabled
    - name: "Rachel Minion"
      slug: "rachelminion"
      voice_notes: |
        Warm, direct, energetic but not over-the-top. Marketing
        strategist who connects with people personally.
      clickup_task_id: "868fzfh1h"
    - name: "Debbie Katz"
      slug: "debbie-katz"
      voice_notes: "Match her Brand Kit. Professional and approachable."
      brand_kit_url: "..."
      clickup_task_id: "868fzfgqg"
  ```

  If `li_comment_check_enabled` is `false` or absent, refuse
  cleanly with `{status: "skipped", reason: "li_comment_check not enabled"}`.

  If `li_comment_accounts` is empty, refuse and route to
  `rockstarr-infra:capture-stack`.

- Chrome MCP is connected. If not, ask the operator to install
  the Chrome extension before continuing — do not try to fall
  through to computer-use clicks on LinkedIn.

- For each account, the operator must be able to sign into the
  LinkedIn account in Chrome. The skill never handles
  passwords or automates the sign-in — that's an operator
  step, intentionally.

- If `li_comment_clickup_enabled: true`, the ClickUp connector
  must be installed in the workspace and authorized.

## Inputs

1. `stack.md.li_comment_accounts[]` — the rotation list.
2. `style-guide.md` if present — read for general voice rules
   that apply across all accounts. Per-account `voice_notes` in
   stack.md override style-guide on conflicts (different people
   = different voices).
3. The account's optional `brand_kit_url` — fetched on demand
   for additional voice context if the operator wants to dig
   in. Not required for routine runs.
4. `style-guide.md` § Channel Adaptation → LinkedIn comments
   subsection if it exists — provides cross-account rules
   (length band, hashtag policy, emoji policy).

## Account rotation overview

Process accounts in the order listed in
`stack.md.li_comment_accounts[]`. The operator handles browser
sign-in/sign-out between each account.

If a single account is named in the user's invocation ("run
Phil's comment check"), scope the run to that one account and
skip the rest.

For each account, run the workflow below in order. Any account
that can't be processed (operator unavailable, wrong account
signed in and not corrected, browser issue) gets logged and the
skill moves on to the next — partial completion is better than
all-or-nothing.

## Workflow per account

### Step 1 — Identity gate (CRITICAL)

Wrong account = posting in someone else's voice on someone
else's network. Two checks, both must pass.

**1a. Confirm operator-driven sign-in.** Ask the operator to
sign into the target account in Chrome. Wait for explicit
confirmation before continuing.

**1b. Verify the signed-in account.** Via Chrome MCP:

1. Navigate to `https://www.linkedin.com/feed/`.
2. Read the page and take a screenshot.
3. Check the profile photo / nav-bar name in the top-left.
   The expected match is the account's `name` from stack.md.
4. Optional belt-and-suspenders: query the alt text:

   ```js
   document.querySelector('.global-nav__me-photo, img[alt*="Photo of"]')?.alt
   ```

   The returned alt should contain the account `name`
   (case-insensitive substring).

If the wrong account is signed in (e.g., a client account, or
the operator's personal account):

- Tell the operator the actual signed-in name and the expected
  name.
- Wait for them to fix it (sign out, sign in as the correct
  account).
- Re-verify before continuing.
- Do NOT proceed past Step 1 until the correct account is
  confirmed on screen.

If no one is signed in (the page redirects to a login screen),
ask the operator to sign in as the target account before
continuing.

If the operator cannot or will not switch (e.g., session
issues, MFA failure), abort this account, log to
`/05_published/social/li-comments/<YYYY-MM-DD>.md` with
`status: "skipped"` and `reason: "operator could not switch to <name>"`,
and move on.

### Step 2 — Check notifications

Navigate to:

```
https://www.linkedin.com/notifications/?filter=my_posts_all
```

Read the page. Look for notifications mentioning **comments**,
not just reactions/likes. Comments show "commented on your
post" with comment-preview text and a comments count. Reactions
without comments don't need responses — note them only if the
operator wants engagement metrics.

**Time window:**

- On Monday, the window is the trailing Friday → Sunday (a
  full weekend's worth of activity).
- All other days, the window is the trailing 24 hours.
- Manual override: if the operator says "check the last
  week" or names a window, honor that.

Capture each comment notification: the post topic excerpt, the
commenter's name, and the comment-preview text.

### Step 3 — Double-check recent activity

Navigate to:

```
https://www.linkedin.com/in/<slug>/recent-activity/all/
```

Scan visible posts for any showing a comment count. This catches
comments that didn't fire a notification (e.g., reply-to-a-reply,
older posts that just got attention).

For each post with a comment count higher than what
notifications already covered, drill in (Step 4).

### Step 4 — Read full comments

For each post with comments:

1. Click into the post via Chrome MCP `find` (use accessible
   names; never pixel coordinates).
2. Read the full comment text — notifications truncate.
3. Check whether a reply already exists from the account
   holder. If yes, skip — that comment is handled.
4. Capture each unanswered comment: commenter name, comment
   text, post topic, whether it's a question / agreement /
   disagreement / spam / sensitive.

### Step 5 — Draft responses

For each unanswered comment, draft a reply in the account's
voice using:

- The account's `voice_notes` from `stack.md` (the primary
  voice signal — these are per-person, not per-client).
- The account's `brand_kit_url` if the operator wants depth.
- The cross-account `style-guide.md` § Channel Adaptation →
  LinkedIn comments subsection if it exists (length band,
  hashtag policy, emoji policy).

Drafting rules:

1. **Voice match the account, not the firm.** Phil's voice ≠
   Rachel's voice ≠ Debbie's voice. The same comment thread
   on each account's post gets a different reply.
2. **Length: 1-3 sentences.** Comment replies are short. A
   four-sentence reply reads as performance.
3. **Acknowledge specifically.** Don't write "thanks for your
   comment". Reference what the commenter said.
4. **Use the commenter's first name when natural.** Not every
   reply needs the name; don't force it.
5. **No AI tells.** "Delve," "navigate," "unlock," "tapestry,"
   "the real question is...," "in today's fast-paced world."
   No em dashes — periods or commas.
6. **No links unless specifically warranted.** Comment-thread
   links read as promotion.
7. **No hashtags in replies.** Hashtags are for posts, not
   comment replies.

### Step 6 — Present for approval

Render every drafted reply for the operator in this format:

```
**Account:** Phil Katz
**Post topic:** [brief description]
**Commenter:** [first last]
**Their comment:** [verbatim text]
**Drafted reply:** [draft]
```

Wait for explicit approval per comment. The operator can:

- **Approve as-is** → proceed to post.
- **Edit** → operator hands over the edited text; use that
  verbatim.
- **Reject / skip** → don't post; log it in the run summary
  with `outcome: "skipped"` and the operator's reason if
  given.
- **Flag** → operator wants to handle the reply themselves
  later. Log `outcome: "flagged"` and move on.

Do NOT send anything without explicit approval per comment.
"Approve all" is acceptable shorthand if the operator says it
unambiguously, but log each individual approval.

### Step 7 — Post approved replies

For each approved (or operator-edited) reply, via Chrome MCP:

1. Click "Reply" on the comment (use Chrome MCP `find` with
   the comment's accessible-name context — avoid pixel
   coordinates).
2. Type the approved text.
3. Wait for the operator's per-reply confirmation before
   clicking Submit. (Belt-and-suspenders gate; operator may
   waive on accounts where they trust the post-and-go flow.)
4. Click Submit.
5. Capture the resulting reply URL or thread state for the run
   log.

If a Submit click fails (LinkedIn rate-limited, modal didn't
close), pause 30 seconds, re-read the page, and retry once. If
the second attempt also fails, log the failure with the comment
context and skip — do not power through repeated failures.

### Step 8 — Close the ClickUp task (if enabled)

Only if `stack.md.li_comment_clickup_enabled: true` AND the
account has a `clickup_task_id` in stack.md.

Update the ClickUp task to "Closed" via the connected ClickUp
MCP. Only close the task if **all of these are true**:

- Step 1's identity gate passed.
- Step 2 (notifications) and Step 3 (recent activity) both
  ran without error.
- Every unanswered comment got a decision (approved + posted,
  operator-edited + posted, skipped, or flagged).

If any of those failed, leave the task open and add a ClickUp
comment naming what blocked the close — "wrong account signed
in," "operator unavailable for approvals," "Chrome MCP
disconnected mid-run," etc.

If `li_comment_clickup_enabled: false`, skip this step. Other
clients may use Asana, Linear, Notion, or no task tracker at
all; future variants of this skill (or `stack.md` extensions)
can wire those in.

### Step 9 — Switch account

If more accounts remain in the rotation, ask the operator to
sign out and sign into the next account. Loop to Step 1.

## After all accounts

1. Ask the operator to sign out of the last account and back
   into their own usual session (so they don't accidentally
   post as a managed account in their own work later in the
   day).
2. Confirm in chat that:
   - Accounts processed cleanly are listed with their reply
     counts.
   - Accounts that were skipped are listed with the reason.
   - ClickUp tasks (if enabled) are closed only for cleanly-
     processed accounts.
3. Write the run summary to
   `/05_published/social/li-comments/<YYYY-MM-DD>.md`.

## Run log shape

Path: `/05_published/social/li-comments/<YYYY-MM-DD>.md`. One
file per day; if multiple runs happen the same day (rare —
operator catch-up after a missed scheduled fire), append.

Front-matter:

```yaml
# ---
run_kind: "scheduled"   # scheduled | manual | catch-up | scoped
client_id: [from client.toml]
run_date: "2026-05-08"
run_started_at: "ISO timestamp"
run_finished_at: "ISO timestamp"
accounts_processed: 3
accounts_skipped: 0
total_comments_seen: 7
total_replies_posted: 5
total_replies_skipped: 1
total_replies_flagged: 1
clickup_closures: 3   # 0 if clickup disabled
produced_by: "rockstarr-social/li-comment-check@0.1.0"
# ---
```

Body shape:

```markdown
# Daily LI comment check — 2026-05-08

## Account: Phil Katz (philipkatz1)

- **Identity gate:** passed
- **Comments seen:** 3 (2 from notifications, 1 caught on recent-activity)
- **Replies posted:** 2
- **Replies skipped:** 0
- **Replies flagged:** 1 — "sensitive comment about hiring decision; operator handling"
- **ClickUp:** task 868fzfgrf closed

### Comment-by-comment

1. Post: "Why we hire slow" — Commenter: Jane Doe — Status:
   posted reply (URL captured below).
2. Post: "Why we hire slow" — Commenter: John Smith — Status:
   posted reply.
3. Post: "Q1 markets" — Commenter: Sarah Lee — Status: flagged
   for operator follow-up.

## Account: Rachel Minion (rachelminion)

(same shape per account)

## Account: Debbie Katz (debbie-katz)

(same shape per account)

## After all accounts

- Operator signed out of Debbie's account and back into their
  own session at <ISO timestamp>.
```

## Chat summary

Print at the end of the run:

- Per account: identity-gate result, comments-seen count,
  replies-posted count, ClickUp closure (if enabled), any
  flagged items.
- Total replies posted across all accounts.
- The run-log file path.

End with:

> Daily LI comment check complete. <N> replies posted, <M>
> flagged for follow-up. Run log at
> `05_published/social/li-comments/<YYYY-MM-DD>.md`.

## Edge cases

- **No comments on any account:** Run Steps 1-3 anyway, then
  log "no comments today" per account and close the relevant
  ClickUp tasks (if enabled). Visibility matters even on quiet
  days.
- **Spam or irrelevant comment:** Flag it for the operator
  with the comment text quoted. Operator decides ignore /
  delete / reply. Don't auto-respond to spam.
- **Hostile, sensitive, or legally-charged comment:** Flag.
  Don't draft a reply — surface the comment with the post
  context and let the operator handle.
- **Comment on a post that's older than the time window but
  was just engaged with:** Treat as in-window; the time
  window is about when the comment landed, not when the post
  was published.
- **Brand kit URL provided:** If the operator wants extra
  voice depth, fetch and read it before drafting that
  account's replies. Skip otherwise — `voice_notes` in
  stack.md is the primary signal.
- **Account locked / "we're checking your sign-in":** Abort
  this account cleanly, leave the ClickUp task open with a
  comment, and move on. Do not ask the operator to recover —
  account-recovery is out of scope.

## What NOT to do

- Do NOT proceed past Step 1 if the wrong account is signed
  in. Wrong-account replies are the highest-stakes failure
  mode in this skill — there is no soft-fail path.
- Do NOT post any reply without explicit operator approval per
  comment. Even an "approve all" shorthand requires the
  operator to type it.
- Do NOT close a ClickUp task you didn't actually complete
  the work for. The close is the operator's audit trail; lying
  to the audit trail is worse than leaving the task open.
- Do NOT use pixel coordinates for any LinkedIn click. Chrome
  MCP `find` with accessible-name queries is the only stable
  lookup.
- Do NOT auto-handle account sign-in. The operator handles
  switches; that's a deliberate trust boundary.
- Do NOT mix voices across accounts. Each account's
  `voice_notes` is its own primary voice signal.
- Do NOT add hashtags or links to comment replies unless the
  account's `voice_notes` explicitly says to.
- Do NOT retry a failed Submit click more than once (Step 7).
  A second silent send risks double-posting.
- Do NOT skip the run log. Even a "no comments today" outcome
  gets a row.

## Schedule wiring

The scheduled run is wired during onboarding via the
workspace's `schedule` skill, pointing at this skill with the
cron expression from
`stack.md.li_comment_check_cron`. Default `30 8 * * 1-5`
(08:30 weekdays, client-local).

## Related

- `invite-page-followers` — sister skill in this plugin.
  Different surface (page-follower audience growth) and
  cadence (monthly), but uses the same Chrome MCP signed-in
  session pattern.
- `draft-social` / `fill-week` — the post-drafting side of
  social. This skill engages with replies on already-published
  posts; those skills produce posts in the first place.
- `rockstarr-infra:capture-stack` — the install-time skill
  that collects the LI-comment-check config. Until
  capture-stack prompts for `li_comment_accounts[]`, operators
  add the block to `stack.md` by hand.
- `rockstarr-infra:publish-log` — NOT called by this skill.
  Comment replies are not "publishes" in the Rockstarr sense
  (they're not approved drafts moving through 03 → 04 → 05).
  The run log inside `05_published/social/li-comments/` is the
  audit record.
