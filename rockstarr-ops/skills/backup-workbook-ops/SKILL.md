---
name: backup-workbook-ops
description: "This skill should be used every Friday end-of-day after ops-weekly-report runs, or when the user says \"back up the ops workbook\", \"snapshot ops-mirror.xlsx\", \"save a weekly ops backup\", \"backup the ops mirror\". Copies the current /02_inputs/ops/ops-mirror.xlsx to /06_reports/data/ops-mirror-backup-[YYYY-WW].xlsx, preserving the small ops mirror (Reengagements, PostCalls, Deliverability, Audits sheets) as it stood at week's end for rollback and comparison. Idempotent — re-running on the same week overwrites the existing backup. Mirror is light in V0.1; backup is small."
---

# backup-workbook-ops

Friday end-of-day snapshot of the small ops mirror. Idempotent —
re-running on the same week overwrites the existing backup.

The mirror is the audit substrate `ops-weekly-report` cross-checks
against the markdown rollups. Backing it up weekly lets the
operator roll back when a downstream skill writes a bad row, or
diff week-over-week for trend investigation.

## When to run

- Scheduled, every Friday at 17:30 local (after
  `ops-weekly-report` finishes its 17:00 run). Wired by
  `rockstarr-infra:scaffold-client` at install time.
- On demand by the operator with the trigger phrases above.

## Preconditions

- `/02_inputs/ops/ops-mirror.xlsx` exists. If it does not yet
  (a brand-new install), surface "No ops mirror to back up —
  run an ops skill first" and exit.

## Inputs

- `target_week` — optional ISO week string. Defaults to the
  current ISO week.

## Behavior

### Step 1 — Resolve paths

Source: `/rockstarr-ai/02_inputs/ops/ops-mirror.xlsx`.
Destination:
`/rockstarr-ai/06_reports/data/ops-mirror-backup-<target_week>.xlsx`.

### Step 2 — Copy

Copy bytes via the file system (no parse / re-write — preserves
formatting, formulas, and any operator annotations). Overwrite
the destination if it exists.

### Step 3 — Log the backup

Append to `/06_reports/data/_backup-log.md` (or create it if
missing):

```markdown
## <YYYY-MM-DD HH:MM> — Backup ops-mirror

Source: <source path>
Destination: <dest path>
Source size: <bytes>
Destination size: <bytes>
```

### Step 4 — Surface (foreground only)

When run on demand, render in chat:

> Backed up `ops-mirror.xlsx` →
> [`ops-mirror-backup-<YYYY-WW>.xlsx`](computer://<path>).

Background runs do not chat-post — the backup-log.md row is
the durable record.

## Outputs

- `/rockstarr-ai/06_reports/data/ops-mirror-backup-<YYYY-WW>.xlsx`.
- One row appended to
  `/rockstarr-ai/06_reports/data/_backup-log.md`.

## Approvals

Unattended. The backup is a non-destructive copy; no operator
gate.

## Failure modes

- **Source file locked** (operator has the workbook open in
  Excel). Retry once after a 10s delay. If still locked, write
  to `_errors.md` with "ops-mirror.xlsx locked — close Excel
  and re-run." Do NOT silently skip.
- **Destination directory missing.** Create
  `/06_reports/data/` if it doesn't exist. (`scaffold-client`
  creates this — but a manually-deleted folder is recoverable.)
- **Disk full.** Surface clearly: "Disk full — backup failed."
  Exit without writing a partial file.

## What this skill does NOT do

- Does NOT parse the workbook contents. Pure file copy.
- Does NOT prune old backups. Operators who want to clean up
  delete files manually. (The 4-week-old backup is small; pruning
  is a deferred feature.)
- Does NOT consult any other intake file.
- Does NOT touch the source workbook. Read-only.
