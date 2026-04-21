---
name: kb-ingest
description: "This skill should be used when the user asks to \"ingest the knowledge base\", \"process raw kb files\", \"refresh the knowledge base\", \"add this article to the kb\", \"pull this URL into the kb\", or drops new files into /rockstarr-ai/01_knowledge_base/raw/. It converts raw PDFs, Word docs, plain text, markdown, and https URLs into cleaned, tagged markdown that Rockstarr bots can read. It also maintains index.md so bots know what is available, and keeps first-party material separate from third-party reference material."
---

# kb-ingest

> **Template convention.** Fenced code blocks in this skill show `# ---`
> where YAML front-matter delimiters belong. That substitution exists
> only to keep Cowork's SKILL.md parser from misreading them as frontmatter
> separators. **When writing actual output files, emit real `---`, not
> `# ---`.**

Take raw source material from the client (PDFs, Word docs, saved web
pages, plain text, https URLs) and turn it into cleaned, tagged markdown
that bots can read deterministically.

v1 is **flat tagged markdown with a keyword-searchable index**. No
embeddings, no vector search. Every file ends up in
`/rockstarr-ai/01_knowledge_base/processed/` with YAML front-matter tags,
and the index lists them for bots to read directly.

## First-party vs. third-party content

The knowledge base holds two scopes of material, kept in separate
directories so bots can tell them apart:

- **owned** — content the client produced or has the rights to speak
  for. Safe to draw from when drafting on the client's behalf. Used by
  every bot when researching topics and gathering proof points.
- **third_party** — saved reference material (articles the client
  bookmarked, competitor posts, industry research, other people's
  writing). Preserved for future content development but **never used
  as a voice or style signal**. `generate-style-guide` ignores it.
  Drafting bots may cite or link it, but never paraphrase it as if it
  were the client's own thinking.

Directory layout:

```
/rockstarr-ai/01_knowledge_base/
├── raw/                            # first-party raw uploads
├── raw/third-party/                # third-party raw uploads
├── processed/                      # first-party cleaned markdown
├── processed/third-party/          # third-party cleaned markdown
└── index.md                        # unified index with scope column
```

## When to run

- Client drops new files into `/rockstarr-ai/01_knowledge_base/raw/`
  or `/raw/third-party/`.
- User explicitly asks to refresh the knowledge base.
- User asks to add a URL to the kb (see "URL ingestion" below).
- User asks to mark or move a file to third-party scope (see "Scope
  routing" below).
- After a workbook ingest, to bring in any supporting materials
  referenced by the workbook.

## Preconditions

- `/rockstarr-ai/01_knowledge_base/raw/` exists. Create
  `raw/third-party/`, `processed/`, and `processed/third-party/` if
  missing.
- The `docx` skill is available for `.docx` parsing; the `pdf` skill is
  available for PDF parsing. Use them rather than hand-rolling parsers.
- `WebFetch` is available for URL ingestion.

## Steps — file ingestion

1. List every file under `/rockstarr-ai/01_knowledge_base/raw/` and its
   subdirectories. Supported extensions: `.pdf`, `.docx`, `.md`, `.txt`,
   `.html`. Skip and warn on other extensions.

2. Determine scope for each raw file:

   - Files anywhere under `raw/third-party/` → `third_party`.
   - All other files under `raw/` → `owned`.
   - If a file has a front-matter or first-line marker
     `kb_scope: third_party` (or a filename prefix `3p_`), treat as
     `third_party` even if it sits outside the third-party directory,
     and note in the summary that it should be moved.

3. For each file, check whether it has already been processed by
   comparing its relative path and modification time against what is
   recorded in `/01_knowledge_base/index.md`. Skip files that have not
   changed.

4. For each new or changed file:

   a. **Extract text.** Use the docx skill for `.docx`, the pdf skill
      for `.pdf`, direct read for `.md` / `.txt`, and a simple HTML-to-
      text extractor for `.html`.

   b. **Clean.** Strip headers, footers, page numbers, and obvious
      navigation chrome. Collapse more than one blank line into a single
      blank line. Preserve headings and list structure.

   c. **Tag.** Generate a short tag list from the cleaned text:
      - topic keywords (3–7 salient nouns or noun phrases)
      - format (`article`, `case-study`, `whitepaper`, `blog-post`,
        `internal-doc`, `transcript`, `webpage`, `other`)
      - source (original filename without extension, or source URL if
        from URL ingestion)
      - approximate date if detectable in the content; otherwise file
        mtime

   d. **Name.** Write to the scope-appropriate folder:

      - owned → `/01_knowledge_base/processed/{yyyy-mm-dd}_{slug}.md`
      - third_party → `/01_knowledge_base/processed/third-party/{yyyy-mm-dd}_{slug}.md`

      where `{yyyy-mm-dd}` is the ingestion date and `{slug}` is a
      kebab-cased, shortened version of the source filename or URL
      title.

   e. **Front-matter.** Every processed file starts with:

      ```yaml
      # ---
      title: "<short descriptive title>"
      kb_scope: "owned"                   # or "third_party"
      source_file: "01_knowledge_base/raw/<relative path>"
      source_url: ""                      # set for URL-ingested files
      source_mtime: "<ISO>"
      ingested_at: "<ISO>"
      topic_tags: [tag1, tag2, tag3]
      format_tag: "article"
      style_guide_eligible: true          # false for third_party
      # ---
      ```

5. Rebuild `/01_knowledge_base/index.md`. The index is a plain Markdown
   table with columns: Title, Scope, File, Topic tags, Format,
   Ingested at. Bots read this file first to pick candidates, then
   read the candidates' full text.

6. Print a summary: files added, files updated, files skipped (with
   reason), files errored, grouped by scope.

## Steps — URL ingestion

When the user says "add this URL to the kb", "pull this article in",
"save this link for reference", or supplies a URL directly:

1. Ask or infer the scope:

   - If the URL is on the client's own domain(s) (the `site` field in
     `client.toml` or any domain listed in `client-profile.md` under
     Company), default to `owned` and confirm.
   - Otherwise default to `third_party` and confirm.
   - The user can always override with an explicit "owned" or
     "third-party" instruction.

2. Fetch the page with `WebFetch`. Use the fetch result's extracted
   article body when available. If `WebFetch` fails or the content
   restrictions block the domain, stop and tell the user — do not try
   alternative fetchers.

3. Clean the fetched content:

   - Remove nav, footer, cookie banners, share buttons, related-posts
     widgets. Keep headings, paragraphs, lists, blockquotes, and
     inline links to sources.
   - Preserve the canonical page title, author (if present), and
     publish date.

4. Compute the slug from the page title (kebab-case, max 60 chars).
   Compute the filename as in step 4d above, using the scope-
   appropriate folder.

5. Write the processed markdown with front-matter. For URL ingestion
   set:

   ```yaml
   source_file: ""
   source_url: "<the https URL>"
   source_domain: "<host>"
   author: "<if detected>"
   published_at: "<detected date or empty>"
   ```

6. **Do not mirror the raw HTML into `raw/`.** URL ingestion goes
   straight to `processed/` (or `processed/third-party/`). The
   canonical source is the URL itself, re-fetchable on demand.

7. Update `/01_knowledge_base/index.md` and print the same summary as
   file ingestion.

## Scope routing — moving files between owned and third-party

If the user says "this is third-party, move it" or "mark X as owned",
move both the raw file (when present) and the processed file to the
appropriate directory, update the `kb_scope` and
`style_guide_eligible` fields, and rebuild the index. Never delete;
move only.

## Conventions

- One raw file maps to exactly one processed file. Do not chunk in v1.
- Never delete from `/raw/` or `/raw/third-party/`. Original sources
  stay forever. Same for URL-ingested processed files — the URL is the
  source of truth.
- If two raw files or URLs produce the same slug in the same scope on
  the same day, suffix with `-2`, `-3`, etc.
- Keep processed file bodies under ~4,000 words each. If a source is
  significantly larger, truncate with a clear marker and note in the
  front-matter `truncated: true`. v2 (embeddings) will solve this.
- `generate-style-guide` MUST only read files where
  `style_guide_eligible: true`. This is enforced by that skill, but
  this skill is responsible for setting the flag correctly.

## What NOT to do

- Do not touch files outside `/rockstarr-ai/01_knowledge_base/`.
- Do not invent content or summarize beyond what the skill steps above
  describe.
- Do not generate the style guide or client profile — those are
  separate skills.
- Do not attempt semantic search or embedding generation in v1. The
  architecture intentionally defers that to v2.
- Do not follow links found inside ingested pages. One URL in, one
  markdown file out.
- Do not use third-party content as a voice signal under any
  circumstance. If in doubt, ask.
