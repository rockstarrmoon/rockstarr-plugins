---
name: follow-up-timer
description: "This skill should be used by draft-reply or present-for-approval to compute the proposed follow-up timer keyword attached to every authorized-send bundle. Trigger phrases: \"what's the follow-up timer for this bucket\", \"compute the follow-up cadence\", \"resolve the timer keyword\". Takes temperature bucket + optional intent_hint + stack.md.followup_timers and returns one of: meeting_proposed | general | referral | cold_bump | none. Non-ICP Ignore / Not Interested / Bad Fit always return none. The caller's create-followup-task skill converts the keyword into an actual date using its own business-days math."
---

# follow-up-timer

Small contract-completion skill. Given the bucket + optional
intent_hint + `stack.md.followup_timers`, returns the timer keyword
the caller will pass to its own create-followup-task skill.

rockstarr-reply proposes the keyword; the caller disposes. If the
caller's `stack.md.followup_timers` has a different day count for
that keyword, the caller's create-followup-task applies the
override. This skill only returns the keyword, not the day count.

## When to run

- Called by `draft-reply` as part of the front-matter compute, so
  `proposed_followup_timer` can be written alongside the staged
  draft.
- Called by `present-for-approval` to render the proposed timer to
  the operator at approval time.
- Called directly when the operator asks "what's the timer for
  this?"

## Inputs

- `bucket` — `Hot | Warm-ICP | Warm-non-ICP | Skeptical | Cold`
  (from classify-reply).
- `icp_verdict` — `target | not-target | ambiguous | unknown`.
- `intent_hint` (optional) — from the handoff bundle. Overrides the
  bucket default when present.
- `sub_types` (optional) — used to route pitch_back, empty_body, etc.
- `/00_intake/stack.md` — read the `followup_timers` section for
  override keys. This skill does NOT apply the overrides itself; it
  returns the keyword, and the caller reads stack.md.followup_timers
  when it creates the task. The stack.md read here is only to
  sanity-check that the keyword is a recognized key.

## Behavior

### Step 1 — Resolve the keyword

Precedence order:

1. If `intent_hint` is set and maps to a keyword, use it:
   - `book-meeting-followup` → `meeting_proposed`
   - `referral-pivot` → `referral`
   - `bump` → `cold_bump`
   - `breakup` → `none`
   - `graceful-exit` / `throwaway-question` → `none`
2. Else route on bucket × icp_verdict:
   - `Hot` (target) → `meeting_proposed`
   - `Warm-ICP` (target) → `general`
   - `Skeptical` (target) with referral pivot sent → `referral`
   - `Skeptical` (target) without referral — label Not Interested → `none`
   - `Cold` (target) after a bump is drafted → `cold_bump`
   - `Warm-non-ICP` option A sent → `none` (label Ignore)
   - `Warm-non-ICP` option B (let-it-hang) → `none`
   - `Warm-non-ICP` option C sent → `none` (label Ignore; the
     throwaway question is a close-the-loop move, not a follow-up
     trigger)
   - `Warm-non-ICP` (Hot but non-target) graceful-exit path → `none`
   - Anything with `icp_verdict != target` that labels `Ignore`,
     `Not Interested`, or `Bad Fit` → `none` (hard rule — the spec
     is explicit that these are not configurable)
3. Else, for sub_types:
   - `pitch_back` → `none` (caller labels Bad Fit, no task)
   - `out_of_office` / `already_booked` → `none` (these route to
     `no-action` anyway; the keyword is returned for completeness)

### Step 2 — Return

Return a single keyword string:

```
"meeting_proposed" | "general" | "referral" | "cold_bump" | "none"
```

No side effects, no file writes.

## Default day counts (caller uses these unless stack.md overrides)

The caller's create-followup-task skill converts the keyword to a
due date. For context, the default day counts (from the spec) are:

| Keyword | Default due date |
|---|---|
| `meeting_proposed` | Monday-if-Friday, else +2 business days |
| `general` | +3 business days |
| `referral` | +5 business days |
| `cold_bump` | +5 business days |
| `none` | No task created |

Overridable in `stack.md.followup_timers` (except `none`, which is
hardcoded per the spec). If the caller sees a stack.md override for
the keyword this skill returns, the caller applies the override.

## Failure modes

- **Unknown bucket / intent_hint combination.** Default to
  `general`. Log the unrecognized combo to the draft front-matter as
  `followup_timer_fallback: true` so the operator sees it at approval
  time.
- **stack.md.followup_timers not yet captured.** Return the keyword
  anyway. The caller's create-followup-task will use the default day
  count.

## What NOT to do

- Do not compute the actual due date. That's the caller's job — it
  needs to know its own business-days calendar, which may differ
  from another caller's.
- Do not write to any file. This is a pure function.
- Do not send, label, or create a task. Keyword only.
- Do not return a day count — the caller maps keyword to days.
- Do not override a Non-ICP Ignore / Not Interested / Bad Fit to
  produce a non-`none` keyword. The spec explicitly hardcodes those
  to `none`.
