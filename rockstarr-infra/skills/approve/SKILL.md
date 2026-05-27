---
name: approve
description: "This skill should be used when the user says \"approve this draft\", \"move this to approved\", \"sign off on this\", or \"promote this draft\". It moves a file from /rockstarr-ai/03_drafts/ to /rockstarr-ai/04_approved/ with a date-stamped filename and records the approval in an approvals log. Chat confirmation follows skills/_shared/references/client-facing-output-voice.md — one short sentence about what now exists plus one short sentence about the next step in the user's terms (channel-aware: \"post it to LinkedIn\", \"send the newsletter\", etc.), with file paths in a collapsed Details footer. Every Rockstarr bot's review flow ends here."
---

# approve

Promote a reviewed draft to `04_approved/`. This is the single gate
between "AI wrote it" and "Rockstarr is willing to ship it." Bots never
write straight to `04_approved/`; only this skill does.

## When to run

- A human has reviewed a file in `/rockstarr-ai/03_drafts/` and wants to
  sign off on it.
- A bot has finished a draft and the reviewer says "approve".

## Preconditions

- Target draft exists under `/rockstarr-ai/03_drafts/`.
- `/rockstarr-ai/04_approved/` exists.
- The user running this skill is authorized to approve on the client's
  behalf (typically Rockstarr staff or the client themselves).
- The shared client-facing output voice reference is available at
  `rockstarr-infra/skills/_shared/references/client-facing-output-voice.md`.
  Step 7 follows its rules. Refuse if missing.

## Steps

1. Identify the draft to approve. If the user did not name a specific
   file, list everything in `/rockstarr-ai/03_drafts/` (sorted by mtime
   descending) and ask which one. Never guess.

2. Read the draft's YAML front-matter. Expect these fields at minimum:

   - `channel` — e.g., `blog`, `linkedin`, `email`, `outreach`
   - `produced_by` — which bot produced the draft
   - `produced_at` — ISO timestamp
   - `style_guide_version` — the style-guide version the bot used

   If any are missing, warn the user and ask whether to continue.

3. Compute the approved filename:

   ```
   /rockstarr-ai/04_approved/[yyyy-mm-dd]_[channel]_[slug].md
   ```

   where `[slug]` comes from the draft's front-matter `title` (kebab-
   cased, max 60 chars) or the source filename if `title` is absent.

4. Add approval front-matter fields to the file before writing:

   ```yaml
   approved_at: "<ISO timestamp>"
   approved_by: "<user's email or name>"
   approved_from: "03_drafts/<original filename>"
   ```

   Preserve all existing front-matter fields.

5. Write the file to its new path under `04_approved/` and delete the
   original from `03_drafts/`. If the destination already exists, append
   `-2`, `-3` etc. to the slug — never overwrite.

6. Append a line to `/rockstarr-ai/04_approved/_approvals.log` (create
   if missing):

   ```
   [ISO]  [approver]  [channel]  <approved filename>
   ```

   This log is what `publish-log` and audit reviews read to reconstruct
   history.

7. Print a confirmation in chat per the voice guide's rule 1 (lead
   with the outcome) and rule 4 (one or two sentences). The shape:

   - **First sentence**: what now exists, in the user's terms.
     "Approved. Your <channel-aware noun> is ready to go live."
   - **Second sentence**: the next step in the user's terms, with
     the channel-appropriate action verb. Most cases use plain
     English ("Post it to LinkedIn when you can and I'll log it
     for the weekly report"). The outreach case is the one
     exception that names a skill — see the table.
   - **File paths and log entries**: go in a collapsed `[details]`
     footer, never inline in the main message.

   ### Channel → action mapping for the next-step sentence

   | `channel` value | Next-step sentence |
   |---|---|
   | `blog` | "Publish it to your blog when you can and I'll log it for the weekly report." |
   | `thought-leadership` | "Post the article when you can and I'll log it for the weekly report." |
   | `linkedin-post` / `x-thread` / `newsletter-highlight` | "Post it to <LinkedIn / X / your newsletter audience> when you can and I'll log it." |
   | `linkedin-newsletter` / `newsletter` / `email` | "Send the newsletter when you can and I'll log it." |
   | `case-study` | "Publish the case study when you can and I'll log it." |
   | `video-script` | "Record it when you can and I'll log the upload." |
   | `outreach-campaign` / `outreach` | "Your campaign is ready to go live — say 'register the campaign' or run `register-campaign` and I'll crawl the leads and start sending." |
   | `reply` | (replies route through `send-approved-reply`, not `approve` — this case shouldn't normally hit here. If it does: "Approved. Run `send-approved-reply` to send it.") |
   | any other / unknown | "Ship it when you can and I'll log it." |

   ### Shape

   ~~~markdown
   Approved. Your <channel-aware noun> is ready to go live.
   <Channel-specific next-step sentence>

   **Details**

   - File: `<new path under 04_approved/>`
   - Approval log: `04_approved/_approvals.log`
   ~~~

   The user who needs the file path can expand. The user who
   doesn't (most of them, most of the time) reads two sentences
   and moves on.

## What NOT to do

- Do not edit the draft body. If the reviewer wants changes, they make
  them in `03_drafts/` before approving.
- Do not promote files from anywhere other than `03_drafts/`. Approval
  requires a reviewable draft path.
- Do not publish. Approving is not the same as sending. `publish-log`
  handles the ship event.
- Do not bulk-approve. One file per run. The friction is the feature.
- Do not surface the bullet-list "Summary: Promoted X → Y. Approval
  front-matter added: approved_at, approved_by, approved_from,
  approval_status: approved. Log entry written: ..." pattern in chat
  output. That was the V0.x shape and it reads as audit log to a
  non-technical client. The two-sentence + collapsed Details footer
  shape in Step 7 is the canonical replacement.
- Do not say "Run the channel's publish flow." That phrase is
  meaningless to a non-technical client (which "channel"? which
  "flow"?). Use the channel-specific action verb from the Step 7
  mapping table.
