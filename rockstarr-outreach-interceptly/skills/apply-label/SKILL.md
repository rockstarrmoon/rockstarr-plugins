---
name: apply-label
description: "This skill should be used after send-message returns success in the per-reply pipeline, when present-for-approval's let-it-hang option routes to label-only, or when the user says \"label this thread <label>\", \"tag <lead> as Not Interested\", or \"apply the <label> label\". Uses the Interceptly Labels UI via Chrome MCP (label.click() pattern) to toggle the proposed label on the current thread, with Labels-nav-bug recovery if the UI navigates away to Campaigns. Refuses to apply more than one label per thread per pipeline run — labels are single-value."
---

# apply-label

Labels the just-processed thread. Exactly one label per thread
per pipeline run.

## When to run

- Immediately after `send-message` returns success, called by
  the per-reply pipeline.
- From `present-for-approval` when the operator picks "let it
  hang" — label-only, no send.
- From `qualify-lead` → `process-inbox` → non-ICP decline path,
  label-only (no send, no task).
- On demand when the user says "label `<lead>` as `<label>`" —
  validates the label is in the workspace's label list and the
  thread is visible.

## Preconditions

- Thread is currently open in Chrome.
- `confirm-session-interceptly` passed in this run.
- Proposed label is in the workspace's label list (from
  `interceptly-accounts.md → ## Labels` or
  `stack.md.label_mapping`).

## Inputs

- `thread_id` — Interceptly thread reference (should match the
  currently-open thread).
- `label` — exact label text as it appears in Interceptly's
  Labels UI.

## Behavior

### Step 1 — Open the Labels panel

From the open thread view, click the Labels icon / link. This
opens the label checkbox list for the current thread.

**Known UI quirk:** clicking Labels has been known to navigate
away to the Campaigns view on certain UI revisions. If the URL
or page title changes to Campaigns, recover:

1. Navigate back to `/inbox` (or wherever the thread lives).
2. Re-open the thread.
3. Try the Labels panel again.
4. On second Labels-nav-bug hit, abort: write _errors.md and
   tell the operator to apply manually.

### Step 2 — Toggle the correct label

The Labels panel is a list of checkboxes. Use
`label.click()` (matching the visible label text) to toggle
the target label on. Prefer clicking the `<label>` element
over the `<input>` — Interceptly's onClick handler is on the
label, and the preserved Rockstarr Chrome MCP pattern does the
same.

If other labels are already applied to this thread, DO NOT
remove them. This skill is additive only — a thread can have
exactly one canonical Rockstarr label (Booked / Not Interested
/ Follow Up / etc.), but the client may have other taxonomy
labels applied for their own reasons.

Exception: if the target label is mutually exclusive with
another label the thread already has (e.g., the thread is
labeled Ignore and the new label is Booked), REMOVE the
conflicting label first. The six-label Rockstarr taxonomy
(INTERESTED / Booked / Referral / Follow Up / Contact Later /
Bad Fit / Ignore / Not Interested) treats these as mutually
exclusive — one per thread at a time.

### Step 3 — Confirm

Wait for the checkbox to visibly toggle (checked state
change in the DOM). If no change within 3 seconds, retry the
click once. On second failure, abort: write _errors.md.

### Step 4 — Close the Labels panel

Click outside or press Escape. Some revisions auto-close;
others don't. Both are fine as long as the panel dismisses
before the next skill runs.

### Step 5 — Log to the Labels sheet

Append to the `Labels` sheet of `outreach-mirror.xlsx`:

| column | value |
|---|---|
| ts | `<ISO>` |
| thread_id | `<id>` |
| lead_url | `<URL>` |
| account_label | `<active managed account>` |
| label_applied | `<label>` |
| label_removed | `<conflicting label removed, if any>` |
| reason | `<send / let-it-hang / non-ICP-decline / booked / manual>` |

### Step 6 — Return

`{labeled: true, label: "<label>", conflicts_removed: ["<...>"]}`

## Label taxonomy and default mapping

Default mapping (overridable via `stack.md.label_mapping`):

| situation | label |
|---|---|
| Successfully-sent hot reply (meeting proposed) | INTERESTED |
| Successfully-sent warm-ICP reply | Follow Up |
| Successfully-sent cold bump | Contact Later |
| Successfully-sent skeptical reply | Follow Up |
| Referral thank-you sent | Referral |
| Pitch-back decline sent | Bad Fit |
| Non-ICP polite-yes — Graceful exit sent | Bad Fit |
| Non-ICP polite-yes — Let-it-hang | Ignore |
| Non-ICP polite-yes — Throwaway question sent | Follow Up |
| Explicit decline received | Not Interested |
| Polite ack with no signal | Ignore |
| Meeting booked (from `mark-booked`) | Booked |
| Flagged for review | Follow Up |

If `stack.md.label_mapping` specifies a different mapping for
any of these situations, use that. Otherwise fall back to the
table above.

## Failure modes

- **Labels-nav-bug fires.** Recover per Step 1. On second
  occurrence in the same pipeline run, abort; the operator
  labels manually.
- **Label not in the workspace's label list.** Refuse. Do NOT
  create new labels from this skill — the workspace's label
  list is owned by the client.
- **Thread is no longer open.** Whoever called this skill
  navigated away between send and label. Re-open the thread
  and try once. On second failure, abort.

## What NOT to do

- Do not apply more than one canonical label per thread per
  pipeline run. If the pipeline wants to label twice, that's
  a pipeline bug.
- Do not create new labels.
- Do not remove non-conflicting labels.
- Do not apply Booked from here — `mark-booked` applies
  Booked as part of the booking-state transition. This skill
  is for all other labels.
