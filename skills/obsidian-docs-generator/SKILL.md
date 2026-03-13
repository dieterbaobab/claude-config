---
name: obsidian-docs-generator
description: Use when generating or updating Obsidian vault project documentation from codebase, Confluence exports, and existing vault notes. Triggers on requests to create project docs, sync documentation to vault, or document a codebase.
---

# Obsidian Docs Generator

Generate and maintain project documentation in the Obsidian vault from multiple sources. Single-phase skill: gather, analyze, propose, write.

## Phase 1: Gather, Analyze, Propose, Write

Follow these steps in order. Each step completes before moving to the next.

### Step 1: Gather local sources

Use the Agent tool (subagent_type: Explore, thoroughness: "very thorough") to collect:

- `CLAUDE.md` — project instructions and conventions
- `MEMORY.md` — Claude memory for this project (check `~/.claude/projects/` for the matching project path)
- Repo docs: everything in `docs/`, plus `REFACTORS.md`, `README.md`, `CHANGELOG.md` at the root
- Codebase exploration:
  - Top-level and `src/` directory structure
  - `package.json` or equivalent (dependencies, scripts, versions)
  - Key architectural files: providers, hooks, navigation, components, API clients
  - Test files: count, location, what's covered
  - Technical debt indicators: `TODO`, `FIXME`, `@ts-ignore`, `as any`, `eslint-disable`
  - CI/CD config: Azure Pipelines YAML, GitHub Actions, Fastlane, Makefile, etc.
- Git log: last 20 commits (for release patterns, recent activity)
- Existing vault notes: scan `NN/Projects/<Name>/` for existing docs to update rather than duplicate

### Step 2: Confluence ingestion

#### 2a: Suggest search keywords

Based on what was found in Step 1, suggest Confluence search keywords to the user. Draw from:
- Project name (from `package.json`, README, or repo directory name)
- Product or brand name (if different from the repo name)
- Team name or domain terms found in docs

Present suggestions:

> "Based on the codebase, here are some keywords to search Confluence with:
> - `[keyword 1]` (from package.json)
> - `[keyword 2]` (from README)
> - `[keyword 3]` (from docs)
>
> Provide the keywords you'd like to search with (comma-separated), or type 'skip' to skip Confluence."

#### 2b: Search Confluence

**Shell safety:** The `$CONFLUENCE_BASE_URL` env var may contain trailing whitespace or newlines that break URL construction. Always trim it first and store in a local variable. Use `jq` (not inline Python) to parse JSON responses — inline Python with special characters like `!=` gets mangled by shell escaping.

First, validate the environment and set up a trimmed base URL:
```bash
BASE_URL=$(echo "$CONFLUENCE_BASE_URL" | tr -d '[:space:]') && echo "Using: $BASE_URL"
```

For each keyword the user provides, run a CQL search. **Important:** Build the URL by assigning the trimmed base to a variable first, then use that variable. Use `jq` for JSON parsing:
```bash
BASE_URL=$(echo "$CONFLUENCE_BASE_URL" | tr -d '[:space:]')
curl -sk --max-time 15 \
  -H "Authorization: Bearer $CONFLUENCE_PAT" \
  "${BASE_URL}/rest/api/content/search?cql=text~%22KEYWORD%22+OR+title~%22KEYWORD%22&limit=20&expand=title,space" \
  | jq -r '.results[] | select(.type == "page") | [.id, .title, .space.name] | @tsv'
```

Note: URL-encode the keyword quotes as `%22` instead of using escaped shell quotes — this avoids nested quote issues entirely.

Deduplicate results across all keyword searches. Then fetch creation/modification dates for each page using `expand=version,history` and build the page link from `_links.webui`.

Present the combined results as a **table sorted by last modified date (newest first)**:

> "Found [N] pages across Confluence:
>
> | # | Title | Space | Created | Last Modified | Versions | Link |
> |---|-------|-------|---------|---------------|----------|------|
> | 1 | **[Title]** | [Space] | YYYY-MM-DD | YYYY-MM-DD | [n] | [Open]([BASE_URL][webui_path]) |
> | ... |
>
> Which pages should I ingest? (e.g., `1, 3, 5` or `all` or `none`)"

#### 2c: Ingest selected pages

For each selected page:
1. Fetch the page content via the Confluence REST API:
   ```bash
   BASE_URL=$(echo "$CONFLUENCE_BASE_URL" | tr -d '[:space:]')
   curl -sk --max-time 15 \
     -H "Authorization: Bearer $CONFLUENCE_PAT" \
     "${BASE_URL}/rest/api/content/{pageId}?expand=body.view,metadata.labels"
   ```
2. Extract the HTML body using `jq`: `jq -r '.body.view.value'`
3. Convert HTML to markdown: strip tags, convert `<h1>`->`#`, `<h2>`->`##`, `<li>`->`-`, `<table>`->pipe tables, `<code>`->backticks, `<a href>`->`[text](url)`. **Write a temp Python or Node script file** for complex HTML-to-markdown conversion instead of using inline code in the Bash tool — inline scripts with special characters get mangled by shell escaping.
4. Also fetch child pages if the page is a parent:
   ```bash
   BASE_URL=$(echo "$CONFLUENCE_BASE_URL" | tr -d '[:space:]')
   curl -sk --max-time 15 \
     -H "Authorization: Bearer $CONFLUENCE_PAT" \
     "${BASE_URL}/rest/api/content/{pageId}/child/page?expand=body.view"
   ```
5. Store each page as a temp `.md` file

#### 2d: Manual URLs

After search-based ingestion:

> "Do you have any additional Confluence page URLs to import directly? Paste them or type 'done'."

For each URL the user provides:
1. Extract the page ID from the URL. Confluence URLs typically contain `pageId=XXXXX` or `/pages/XXXXX/`.
   If the URL contains a page title instead (e.g., `/display/SPACE/Page+Title`), use the search API:
   ```bash
   BASE_URL=$(echo "$CONFLUENCE_BASE_URL" | tr -d '[:space:]')
   curl -sk --max-time 15 \
     -H "Authorization: Bearer $CONFLUENCE_PAT" \
     "${BASE_URL}/rest/api/content?title=PAGE_TITLE&spaceKey=SPACE_KEY&expand=body.view"
   ```
2. Fetch, convert, and store using the same process as Step 2c.
- Loop until the user says done.

#### Confluence post-processing

- Fix HTML-to-markdown conversion artifacts (broken lists, jammed headers, orphaned image refs) but do **not** rewrite the content itself at this stage.
- Flag all Confluence content as **"reference only — may be outdated."** The codebase is the source of truth.
- Images in Confluence exports are ignored.
- **Do NOT run the humanizer here.** The `humanizer` skill is applied later, during Step 5, when writing the final content to the vault. At ingestion time, preserve the original wording.

**Environment variables required:** `$CONFLUENCE_PAT` (Bearer token) and `$CONFLUENCE_BASE_URL` must be set in the shell environment (e.g., `~/.zshrc`).

**Shell pitfalls to avoid:**
- Never use `$CONFLUENCE_BASE_URL` directly in a quoted URL string — always trim it first with `tr -d '[:space:]'`
- Never write inline Python/Node in Bash tool calls — shell escaping will mangle `!=`, quotes, and backslashes. Write a temp script file with the Write tool instead, then execute it.
- URL-encode special characters in CQL queries (`%22` for `"`, `%20` for space) instead of using escaped quotes
- Use `jq` for all JSON extraction — it handles edge cases that sed/grep/awk miss
- Always add `--max-time 15` to curl calls to avoid hanging on network issues
- All curl commands use `-k` to skip SSL certificate verification, as corporate Confluence instances often use internal CA certificates not in the system trust store.

### Step 3: Cross-reference and analyze

Compare all sources against each other:

- **Confluence vs codebase** — Flag anything in Confluence that contradicts the code (versions, architecture, deps)
- **Existing vault notes vs codebase** — Flag what's outdated in `NN/Projects/<Name>/`
- **Gaps** — What's in the code but not documented anywhere?
- **Technical debt** — Compile from TODOs, FIXMEs, large files, type assertions, known issues

Present a summary to the user:

> "Here's what I found:"
> - **Outdated:** [list]
> - **Discrepancies:** [list]
> - **Missing documentation:** [list]
> - **Technical debt items:** [count] across high/medium/low priority

### Step 4: Propose structure and get approval

Read the template at `~/.claude/skills/obsidian-docs-generator/template.md` for formatting conventions and the MOC template — but do **not** treat the page list as a rigid checklist.

**Only the MOC is always created:**
- `NN/Projects/<Name>/<Name>.md` — landing page with wikilinks to all topic pages

**Everything else is project-driven.** Design pages around the project's actual content, not a fixed set of page types. The goal is pages with clear subject boundaries that are easy to find and update later. Ask: "What are the distinct topics someone would look up for this project?" — then make each one a page.

Common pages that naturally recur across projects (like technical debt, architecture, release process) are fine when they fit. But don't force a "Solution Architecture" page if the project is simple enough that the MOC covers it, and don't skip a "Care Products" page just because it's not in a standard list.

**Guidelines:**
- Each page should have a clear, single subject — if a page covers two unrelated topics, split it
- Prefer specific, descriptive names over generic ones (e.g., "Care Product Data Mapping" over "API Documentation")
- Check existing vault notes in `NN/Projects/<Name>/` to determine update vs create
- **Cross-project docs** go to `NN/Documentation/<Topic>.md` tagged `#nn/docs` if they apply beyond a single project

Propose the full file list with a one-line rationale per page and wait for user approval before proceeding.

### Step 5: Write files

After approval, write files directly to the vault:

- Use the Write tool for file creation
- **REQUIRED:** Invoke the `obsidian:obsidian-markdown` skill for Obsidian Flavored Markdown syntax (wikilinks, callouts, frontmatter/properties)
- **REQUIRED:** Apply the `humanizer` skill when writing content
- MOC page uses `[[wikilinks]]` to link to sibling topic pages
- Follow wikilink conventions from the vault's CLAUDE.md:
  - `[[Project Name]]` for MOCs
  - `[[Project Name/Topic|Display Text]]` for docs with duplicate names
  - `[[Topic]]` for unique names
- Tag notes per vault conventions (e.g. `#nn/docs` for cross-project docs)
- If updating existing notes: read first, merge content, don't blindly overwrite

**Verify after writing:**
- Read each file back and confirm key headings are present
- Check that wikilinks resolve (target files exist or are being created in this run)
- Flag any issues in the results table

Report final status:

> **Vault documentation complete.**
>
> | File | Status |
> |------|--------|
> | `NN/Projects/<Name>/<Name>.md` | Created/Updated |
> | ... | ... |
>
> [Any warnings or issues]

### Clean up

1. **Delete temp Confluence files** — Remove any `.md` files created during Step 2's Confluence ingestion
2. **Update CLAUDE.md** — Add the project to the Active Projects section if it's a new project
3. **Update MEMORY.md** — Add the project's vault paths so future sessions can locate the docs
4. **Update template (if approved)** — If Step 4 flagged template changes and the user approved them, update `~/.claude/skills/obsidian-docs-generator/template.md` now

### Resumability

No intermediate spec artifact is produced. If the conversation breaks mid-write, the skill can be re-invoked. Because Step 5 reads existing notes before writing (merge, don't overwrite), re-running is safe — files already written will be detected as existing and merged rather than duplicated.

## Key Constraints

- **Confluence is reference, not truth.** Always cross-reference against the codebase.
- **Codebase is canonical for technical details.** The vault mirrors, not replaces.
- **No image handling.** Confluence images are excluded.
- **One project at a time.**
- **Obsidian does not need to be running.** Direct file writes only.
