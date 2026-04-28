---
name: capture-interceptly-personas
description: "This skill should be used at install time after discover-interceptly-accounts finishes, or whenever the user says \"capture Interceptly personas\", \"record the signatures for each managed account\", \"update the persona for a managed account\", or \"refresh interceptly-accounts.md\". Interviews the client per managed account on account_label, Interceptly profile URL, LinkedIn profile URL, signature block, and persona notes (subject-matter lens, what this person is known for). Writes /00_intake/interceptly-accounts.md so campaign sequences can be written under the right persona, so rockstarr-reply can draft replies under the right signer, and so confirm-session-interceptly can verify the active LinkedIn account."
---

# capture-interceptly-personas

This is the interview that turns the bare account list produced by
`discover-interceptly-accounts` into the rich persona file every
skill that touches Interceptly needs. Every managed account gets
one row with a signature, a LinkedIn profile URL (for session
confirmation), and free-text persona notes.

`interceptly-accounts.md` is a shared artifact:

- `draft-icp-campaign-interceptly` (this plugin) reads it to match outbound
  sequence copy to the right signer.
- `rockstarr-reply` reads it so draft-reply signs under the right
  persona.
- `confirm-session-interceptly` reads the `linkedin_profile_url`
  to verify the browser is on the correct LinkedIn account.

Voice is one layer, captured in `style-guide.md`. Persona is a
separate layer, captured here. A single client voice can sign
under multiple personas. Do not conflate them.

## When to run

- Immediately after `discover-interceptly-accounts` completes.
- When the client adds a new managed account.
- When the client asks to update a signature, persona note, or
  profile URL for an existing managed account.

## Preconditions

- `/rockstarr-ai/00_intake/stack.md` has a non-empty
  `outreach_accounts[]`. If it does not, refuse and point at
  `discover-interceptly-accounts`.
- `/rockstarr-ai/00_intake/interceptly-accounts.md` exists, at
  minimum as a stub seeded by discovery.

## Inputs

- The ordered list of managed accounts from
  `stack.md.outreach_accounts[]`.
- Any existing rows in `interceptly-accounts.md` for accounts being
  refreshed (offer to reuse vs. re-interview).

## Interview — per managed account

For each account in `outreach_accounts[]`, run the interview below
via `AskUserQuestion`. Do not batch questions across accounts —
finish one account before starting the next so the client's answers
stay grouped in their head.

Open with a one-line summary: "We're capturing the persona for
`<account_label>` (`<workspace>`). Five questions."

### Question 1 — Interceptly profile URL

Free-text via `AskUserQuestion` "Other" path:
"What is the Interceptly profile URL for `<account_label>`? (The
URL you land on after clicking the account in SWITCH ACCOUNT.)"

Validate it starts with `https://dash.interceptly.ai/`.

### Question 2 — LinkedIn profile URL

Free-text:
"What is the LinkedIn profile URL for the person signed in under
this account? (e.g., `https://www.linkedin.com/in/jane-doe/` — the
bot compares this to the signed-in LinkedIn profile on every
session-confirm pass.)"

Validate it starts with `https://www.linkedin.com/in/`. Normalize
(strip trailing slash, query strings, lowercase).

### Question 3 — Signature block

Free-text. "Paste the signature block this persona uses on
LinkedIn replies (name, role, optional extra line). Leave blank if
this persona signs with just their first name."

Store verbatim, including line breaks. Do not auto-format.

### Question 4 — Persona notes

Free-text. "In 2-4 sentences, describe what this persona is known
for. What is the subject-matter lens they speak from? What topics
are natural for them and what topics would sound wrong in their
voice?"

This is the primary input that lets both campaign drafting (this
plugin) and reply drafting (rockstarr-reply) choose which persona
to sign under if a message could go out under more than one.

### Question 5 — Default for uncategorized threads

Options: `Yes — make this the default signer` / `No — ask at
draft time`.

"If a new reply arrives and rockstarr-reply cannot match the thread
to a specific persona, should this be the default signer, or
should the bot pause and ask?"

Exactly one account may have `default_signer = true`. If the client
picks Yes for a second account, warn them and ask which of the two
should keep the default.

## Write file

After the last account finishes, write
`/rockstarr-ai/00_intake/interceptly-accounts.md` in full. Preserve
the enumerated `## Labels` section (if discovery wrote one) —
do NOT drop it on refresh.

```markdown
---
generated_at: <ISO timestamp>
account_count: <N>
schema_version: 1
---

# Interceptly accounts

Source-of-truth for which personas sign which messages. Voice is in
style-guide.md; persona is here. Do not conflate them.

## <account_label 1>

- workspace: <workspace>
- interceptly_profile_url: <URL>
- linkedin_profile_url: <URL>
- default_signer: true|false
- persona_notes: |
    <multi-line free text from Question 4>
- signature: |
    <multi-line verbatim signature from Question 3>

## <account_label 2>
...

## Labels

<preserved or re-enumerated label list from discovery>
```

## Outputs

- `/rockstarr-ai/00_intake/interceptly-accounts.md` — authoritative
  persona file, consumed by this plugin and by rockstarr-reply.

## Gate on Interceptly work

Any Interceptly skill that needs an account's persona or
LinkedIn URL refuses to run when a managed account in
`outreach_accounts[]` is missing a row here, or when a row is
missing `linkedin_profile_url` (`confirm-session-interceptly`
cannot function without it). When a skill is blocked for this
reason, point at this skill.

## Failure modes

- **User skips LinkedIn profile URL.** Refuse to finalize the row.
  Explain that `confirm-session-interceptly` uses this as the
  wrong-account-sending check and wrong-account sending is the top
  reputational risk in the Interceptly pillar.
- **User pastes a LinkedIn vanity URL that redirects.** Follow the
  redirect via Chrome MCP once; store the canonical URL the server
  returns. Fine to verify with the user.
- **User provides no signature.** Confirm with the user this is
  intentional (some personas sign with just their first name
  inline). If yes, write an empty signature block.

## What NOT to do

- Do not fabricate persona notes from training data. If the client
  gives a thin answer, that's fine — thin notes produce thin
  drafts, which is feedback for the next refinement pass.
- Do not write a signature in a format the client didn't give. If
  they paste one line, store one line. If they paste five, store
  five.
- Do not auto-promote accounts into `outreach_accounts[]` from this
  skill — discovery is the only writer for that list.
- Do not ask voice questions here. Voice belongs to
  `style-guide.md`. Persona is a different layer.
- Do not ask ICP qualification questions here. Those live in
  `rockstarr-reply:capture-icp-qualifications` and write to
  `00_intake/icp-qualifications.md`.
