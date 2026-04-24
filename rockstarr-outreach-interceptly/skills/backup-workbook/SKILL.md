---
name: backup-workbook
description: "This skill should be used every Friday end-of-day, or when the user says \"back up the outreach mirror\", \"snapshot outreach-mirror.xlsx\", or \"save a weekly outreach backup\". Copies the current outreach-mirror.xlsx to /06_reports/data/outreach-mirror-backup-YYYY-WW.xlsx, preserving the shared audit mirror (written by this plugin + rockstarr-reply) as it stood at week's end for rollback and comparison."
---

# backup-workbook

A weekly snapshot of the shared audit mirror. Both this plugin and
`rockstarr-reply` write to `outreach-mirror.xlsx`. This skill's
scope is the file — what's inside it belongs to whichever plugin
owns that sheet.

Cheap insurance for the cases where a future run corrupts the file,
a Chrome MCP UI change causes a bad write, or the client wants to
compare "what did the mirror show on Friday" against "what did
Interceptly show Monday."

Note: Interceptly itself is the source of truth. This backup is of
the LOCAL audit mirror only — Interceptly's own state is
Interceptly's to protect.

## When to run

- Friday end-of-day, after `metrics-weekly` and
  `outreach-weekly-report`.
- On-demand before doing something risky (stopping several campaigns
  at once, rebuilding qualification rules, changing persona mappings).

## Inputs

- `02_inputs/outreach/outreach-mirror.xlsx`
- `iso_week` (YYYY-WW) — defaults to the current ISO week.

## Behavior

1. **Verify the source exists.** If the mirror is missing, write a
   loud line to `_errors.md` — the mirror is how both plugins
   reconcile against Interceptly, so a missing file is a real
   incident — and refuse. Tell the operator to rebuild via
   `rockstarr-reply`'s inbox pass before retrying.
2. **Compute the target path.**
   `/rockstarr-ai/06_reports/data/outreach-mirror-backup-<iso_week>.xlsx`.
3. **Check for collisions.** If a backup already exists for this
   ISO week, keep both:
   - Rename the existing file to
     `outreach-mirror-backup-<iso_week>-v1.xlsx`.
   - Write the new one to the canonical name.
   - If both v1 and v2 already exist, append v3, v4, etc.
4. **Copy the file.** Byte-for-byte. Do not open + re-serialize —
   a copy preserves the workbook exactly, including any manual
   client edits to sheets we did not touch.
5. **Log.** Append to `/05_published/outreach/<today>.md`:
   `backup-workbook — snapshot written to <target path>`.

## Output

- `backup_path`
- `size_bytes`
- `iso_week`

## What NOT to do

- Do not delete older backups. Disk space is cheap; audit history
  is not. If storage becomes a problem, add a separate retention
  skill later.
- Do not "clean up" the workbook on backup. The whole point is a
  faithful snapshot — including sheets this plugin does not own.
- Do not back up `_errors.md`, `_flags.md`, `_non_icp_log.md`, or
  publish-log files from this skill. Backup scope is the mirror
  workbook.
- Do not assume the mirror is authoritative. It isn't — Interceptly
  is. A backed-up mirror row that disagrees with Interceptly means
  the pipeline missed a reconcile, not that Interceptly is wrong.
