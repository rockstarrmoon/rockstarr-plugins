---
name: backup-workbook
description: "This skill should be used every Friday end-of-day, or when the user says \"back up the outreach workbook\", \"snapshot outreach-tasks.xlsx\", or \"save a weekly outreach backup\". It copies the current outreach-tasks.xlsx to /06_reports/data/outreach-tasks-backup-YYYY-WW.xlsx, preserving the full workbook as it stood at week's end for rollback and audit."
---

# backup-workbook

A weekly snapshot of the single state-of-truth workbook. Cheap
insurance for the cases where a future run corrupts the file, a
Chrome MCP UI change causes a bad write, or the client wants to
compare "what did we plan on Friday" against "what happened Monday."

## When to run

- Friday end-of-day, after `metrics-weekly` and
  `outreach-weekly-report`.
- On-demand if the user asks for a manual backup before doing
  something risky (e.g., stopping several campaigns at once).

## Inputs

- `02_inputs/outreach/outreach-tasks.xlsx`
- `iso_week` (YYYY-WW) — defaults to the current ISO week.

## Behavior

1. **Verify the source exists.** If the workbook is missing, write a
   loud line to `_errors.md` — we've lost the state-of-truth, which
   is a real incident — and refuse.
2. **Compute the target path.**
   `/rockstarr-ai/06_reports/data/outreach-tasks-backup-<iso_week>.xlsx`.
3. **Check for collisions.** If a backup already exists for this
   ISO week, keep both:
   - Rename the existing file to
     `outreach-tasks-backup-<iso_week>-v1.xlsx`.
   - Write the new one to the canonical name.
   - If both V1 and V2 already exist, append V3, V4, etc.
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
  faithful snapshot.
- Do not back up `_errors.md` or publish-log files from this skill.
  Backup scope is the workbook.
