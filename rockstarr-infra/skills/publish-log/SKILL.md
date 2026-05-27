---
name: publish-log
description: "This skill should be used when the user says \"log that I published this\", \"record the send\", \"mark this shipped\", or after a Rockstarr bot finishes posting/sending something. It writes a publish record to /rockstarr-ai/05_published/ with the metadata needed for performance review and attribution. Chat confirmation follows skills/_shared/references/client-facing-output-voice.md — past tense, channel-aware noun, names when performance shows up next (e.g. \"Logged. Your LinkedIn post is live as of 9:14am — performance shows up in this Friday's report.\")."
---

# publish-log

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as frontmatter
> separators. **When writing actual output files, emit real `---`, not
> `# ---`.**

Record that something actually shipped. This is what closes the loop
between "approved" and "measured" — performance-review and
metrics-review skills in v1.1 will read `/05_published/` to know what
ran when.

## When to run

- Immediately after a bot with a publishing connector (LinkedIn, email,
  scheduler) sends or schedules a piece of content.
- Manually, when the client publishes something outside of Cowork (e.g.,
  posts a LinkedIn article themselves after Rockstarr drafted it).

## Preconditions

- The content was approved. Either an entry exists in
  `/04_approved/_approvals.log`, or the user confirms it was approved
  and explains why it bypassed the log (e.g., client edited post-
  approval).
- `/rockstarr-ai/05_published/[channel]/` exists; create it if missing.
- The shared client-facing output voice reference is available at
  `rockstarr-infra/skills/_shared/references/client-facing-output-voice.md`.
  Step 5 follows its rules. Refuse if missing.

## Steps

1. Gather publish metadata. Ask (or accept from the caller):

   - `channel` — one of `blog`, `linkedin`, `email`, `outreach`,
     `newsletter`, `social-other`.
   - `source_approved_file` — path under `/04_approved/` that this
     ship corresponds to, if any.
   - `published_at` — ISO timestamp. Default to now.
   - `published_by` — who pushed the button (bot name, or person name).
   - `external_url` — public URL if the thing is publicly addressable
     (LinkedIn post URL, blog post URL). Empty string if not (email,
     outreach DM, newsletter send).
   - `external_id` — platform ID if available (e.g., LinkedIn urn,
     email send ID, newsletter broadcast ID).
   - `audience_size` — integer, optional. How many recipients or
     impressions expected. For a LinkedIn post, leave blank; the
     metrics skill will fill it later.
   - `notes` — free text, optional.

2. Compute the log filename:

   ```
   /rockstarr-ai/05_published/[channel]/[yyyy-mm-dd]_[slug].md
   ```

   where `[slug]` is derived from the approved file's slug. If two
   publishes share a slug on the same day, append `-2`, `-3` etc.

3. Write the log file with this front-matter:

   ```yaml
   # ---
   channel: "[channel]"
   published_at: "[ISO]"
   published_by: "[actor]"
   source_approved_file: "04_approved/<filename or empty>"
   external_url: "<url or empty>"
   external_id: "<id or empty>"
   audience_size: <int or null>
   notes: "<text or empty>"
   log_skill_version: "0.1.0"
   # ---
   ```

   Body is a short human summary: one paragraph describing what was
   shipped, pulled from the approved file's title or first line.

4. Append a line to `/rockstarr-ai/05_published/_publish.log` (create
   if missing):

   ```
   [ISO]  [channel]  [published_by]  <published filename>  [external_url]
   ```

5. Print a confirmation in chat per the voice guide's rule 1 (lead
   with the outcome, past tense) and rule 4 (one or two sentences).
   The shape:

   - **First sentence**: confident past tense, what is now live, in
     the user's terms. Includes a humanized timestamp from
     `published_at` (e.g. "9:14am today", "yesterday at 4pm",
     "April 25, 2026 at 8:14am" — match the client's reading
     pattern, not raw ISO).
   - **Second sentence**: when performance shows up. Use "this
     Friday's report" for sends that happen during the week and
     "next Friday's report" for weekend sends. Outreach sends
     surface in the weekly outreach report; content sends in the
     weekly content report; both are Friday-cadence today.
   - **File paths**: in a collapsed `[details]` footer.

   ### Channel → noun mapping for the confirmation

   | `channel` value | Noun for the first sentence |
   |---|---|
   | `blog` | "blog post" |
   | `linkedin` (when it's a post; the `linkedin-post` lane goes here too) | "LinkedIn post" |
   | `email` / `newsletter` | "newsletter" |
   | `outreach` | "outreach send" |
   | `social-other` | "social post" |
   | any other / unknown | "post" (generic fallback) |

   For `outreach` specifically, the noun pluralizes naturally
   ("Your outreach send is logged — 20 connects went out today")
   because connects are batches, not single posts.

   ### Shape

   ~~~markdown
   Logged. Your <channel-aware noun> is live as of <humanized
   timestamp>. Performance shows up in <this/next> Friday's
   report.

   **Details**

   - Publish log: `05_published/[channel]/[filename]`
   - External URL: <url, if external_url was provided>
   ~~~

   The external URL bullet appears in the Details footer only when
   `external_url` is non-empty — outreach / DM sends often have no
   public URL, so the bullet is omitted in those cases.

## Idempotency and duplicates

- If the user asks to log the same `external_url` twice within an hour,
  warn and ask for confirmation. Accidental double-logs pollute the
  metrics view.
- `external_id` is the canonical dedup key when present.

## What NOT to do

- Do not attempt to fetch metrics from the platform here. That is the
  metrics-review skill (v1.1).
- Do not edit anything in `/04_approved/`. Approved files are immutable
  once shipped.
- Do not create publish records for drafts that were never approved.
  If the user insists, require an explicit `bypass_reason` field in the
  front-matter, and print a prominent warning.
- Do not use the passive-tense "When results come in, `metrics-review`
  will read this log..." phrasing in chat output. That sentence read
  as vague-future-tense to clients (they wanted confirmation the
  thing actually went live, not a forward-looking promise about an
  unfamiliar skill). The Step 5 confident-past-tense shape is the
  canonical replacement.
- Do not surface internal skill names like `metrics-review`,
  `outreach-weekly-report`, or `metrics-weekly` in the chat
  confirmation. The user-facing phrase is "this Friday's report" —
  same content, the user's noun.
