---
name: approve
description: "This skill should be used when the user says \"approve this draft\", \"move this to approved\", \"sign off on this\", or \"promote this draft\". It moves a file from /rockstarr-ai/03_drafts/ to /rockstarr-ai/04_approved/ with a date-stamped filename and records the approval in an approvals log. Every Rockstarr bot's review flow ends here."
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
   /rockstarr-ai/04_approved/<yyyy-mm-dd>_<channel>_<slug>.md
   ```

   where `<slug>` comes from the draft's front-matter `title` (kebab-
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
   <ISO>  <approver>  <channel>  <approved filename>
   ```

   This log is what `publish-log` and audit reviews read to reconstruct
   history.

7. Print a confirmation with the new path and a link/reference the user
   can copy. End with a single-line next-step hint:

   > Ready to publish. Run the channel's publish flow, then `publish-log`
   > to record what shipped.

## What NOT to do

- Do not edit the draft body. If the reviewer wants changes, they make
  them in `03_drafts/` before approving.
- Do not promote files from anywhere other than `03_drafts/`. Approval
  requires a reviewable draft path.
- Do not publish. Approving is not the same as sending. `publish-log`
  handles the ship event.
- Do not bulk-approve. One file per run. The friction is the feature.
