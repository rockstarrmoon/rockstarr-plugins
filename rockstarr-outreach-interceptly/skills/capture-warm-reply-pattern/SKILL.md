---
name: capture-warm-reply-pattern
description: "This skill should be used at install time after capture-icp-qualifications, or when the user says \"capture my warm-reply pattern\", \"interview me on how I reply to warm leads\", \"my reply style changed\", or \"update the warm-reply subsection of style-guide.md\". Interviews the client on their go-to move when a curious / warm ICP reply lands — structure, length, opening, closing, what to avoid — and appends the pattern as a subsection under style-guide.md's Channel Adaptation → LinkedIn replies section. The bot has no hardcoded reply pattern; draft-reply-interceptly reads this subsection on every warm-ICP draft."
---

# capture-warm-reply-pattern

Shared intake interview. Canonical source lives at
`rockstarr-infra/skills/_shared/capture-warm-reply-pattern/` once
the shared tree exists; for V0.1 it ships inline in
`rockstarr-outreach-interceptly`. If you edit this file, edit the
matching copy in any sibling plugin that ships its own inline copy
in the same PR.

The warm-reply pattern is where the single most common drafting
mistake lives — "end with worth-a-chat fluff" or "end with a
link." The pattern captured here tells `draft-reply-interceptly`
exactly what to end on. Thin answers produce thin drafts, which is
fine feedback to refine the next time this skill runs.

## When to run

- Install-time, after `capture-icp-qualifications` finishes.
- When the client says "stop ending with X" or "keep ending with
  Y" — re-run to update the pattern.
- When the weekly report shows the client is routinely rewriting
  the closing move on warm-ICP drafts — re-run so the captured
  pattern matches real client behavior.

## Preconditions

- `/rockstarr-ai/00_intake/style-guide.md` exists and has an
  approved Channel Adaptation section. If not, refuse and point at
  `rockstarr-infra:generate-style-guide` — the warm-reply pattern
  is a subsection of the guide, not a standalone file.

## Inputs

- `00_intake/style-guide.md` — full.
- `00_intake/client-profile.md` — positioning paragraph for
  context.
- Any past approved warm-ICP replies in `04_approved/outreach/` (if
  any exist — none at install).

## Interview

Walk one question at a time via `AskUserQuestion`. Pre-read past
replies if any, and propose an answer with HIGH/MEDIUM/LOW
confidence for the client to confirm, amend, or reject.

### Q1 — Structure

"When a warm ICP reply lands, what's the shape of your go-to
response?" Options:

- `Acknowledge → one-line value → soft question` (short, 2-3
  sentences)
- `Acknowledge → build rapport → direct ask` (medium, 3-5
  sentences)
- `Reframe the conversation → specific offer` (direct, 2-4
  sentences)
- `Other (free text)`

### Q2 — Length target

"Target length?" Options: `1-2 sentences` / `3-4 sentences` /
`5-7 sentences` / `Other`.

### Q3 — Opening move

"How do you typically OPEN a warm-ICP reply?" Options:

- `Thank them by name for the reply`
- `Mirror one line from their reply`
- `Skip acknowledgement — go straight to substance`
- `Other (free text)`

### Q4 — Closing move

"How do you typically CLOSE a warm-ICP reply?" Options:

- `Propose a specific next step (call / intro / send something)`
- `Ask a qualifying question`
- `Leave it open — let them respond first`
- `Share the booking link` (flagged as anti-pattern — ask the
  client to reconsider, since the booking link is a destination,
  not content)
- `Other (free text)`

### Q5 — Banned moves

Multi-select `AskUserQuestion`:
"What do you NOT want in warm-ICP replies?"

Options: `Worth-a-chat fluff` / `Corporate hedging language` /
`Calendar link pasted directly` / `Overly formal closers (e.g.,
"Best regards,")` / `Emojis` / `Exclamation marks` /
`Other (free text)`.

### Q6 — Example reply

Free text: "Paste one warm-ICP reply you've sent recently that
feels like you." This becomes the calibration example in the
subsection.

### Q7 — Contrast example

Free text: "Paste one reply that feels WRONG — the kind of thing
an agency would send and you'd hate. This is our `not-this`
reference."

## Write subsection

Append to `/rockstarr-ai/00_intake/style-guide.md` inside the
Channel Adaptation section. If a warm-reply subsection already
exists, REPLACE it (do not append — that would duplicate). Preserve
all other style-guide content.

Subsection format:

```markdown
### LinkedIn replies — warm-ICP pattern

_Captured by `capture-warm-reply-pattern` on <ISO date>. Re-run
this skill to refresh._

**Structure.** <Q1 answer>

**Length target.** <Q2 answer>

**Opening.** <Q3 answer>

**Closing.** <Q4 answer>

**Do NOT include.** <Q5 multi-select list>

**Calibration example (this is us):**

> <Q6 verbatim>

**Anti-example (this is NOT us):**

> <Q7 verbatim>
```

## Outputs

- Updated `/rockstarr-ai/00_intake/style-guide.md` with the
  replaced / inserted subsection. The file's top-of-file
  `approved_at` timestamp does NOT change — subsection refresh is
  a minor update. Bump only when the top-level style guide is
  regenerated by `generate-style-guide`.

## Gate on the daily loop

The daily loop does not hard-gate on this subsection. If it's
missing, `draft-reply-interceptly` falls back to a neutral
warm-ICP template and flags every such draft with "warm-reply
pattern not captured — replies may feel generic; run
`capture-warm-reply-pattern`." So nothing BREAKS; drafts just read
as generic until the pattern is captured.

## Failure modes

- **Client picks "Share the booking link" as the default close.**
  Explain the design rule (booking link is a destination, not
  content) and ask them to reconsider. If they insist, write it
  into the subsection with a visible comment —
  `# conflicts with bot design: booking link never pasted in
  replies`. `draft-reply-interceptly` will still refuse to paste
  the link; this is the one case where the captured pattern is
  overridden by design.
- **Client gives thin one-word answers.** Write them as-is.
  Surface in weekly report if drafts routinely get rewritten.

## What NOT to do

- Do not write a warm-reply pattern the client did not confirm.
  The whole point of this skill is that the bot has no hardcoded
  pattern of its own.
- Do not copy the pattern into `draft-reply-interceptly` or
  anywhere else — it lives in `style-guide.md` and is read at
  draft time.
- Do not overwrite unrelated parts of the style guide. This skill
  only touches the warm-reply subsection.
