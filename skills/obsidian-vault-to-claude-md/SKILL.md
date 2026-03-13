---
name: obsidian-vault-to-claude-md
description: Use when enriching or creating a project CLAUDE.md with information from the Obsidian vault. Triggers on requests to update CLAUDE.md from vault docs, sync vault knowledge to CLAUDE.md, or bootstrap a CLAUDE.md for a project that has vault documentation.
---

# Obsidian Vault to CLAUDE.md

Enrich a project's CLAUDE.md with operational knowledge from the Obsidian vault. Starts from the existing CLAUDE.md (or bootstraps one), reads vault docs, proposes targeted additions/updates, and writes them after user approval.

## Step 1: Determine starting point

Check if a CLAUDE.md exists in the current project repo root.

**If CLAUDE.md exists:**
- Read it and parse into sections by `##` headings
- Note the existing structure and what's already covered
- Proceed to Step 2

**If no CLAUDE.md exists:**
- Bootstrap a baseline CLAUDE.md by scanning the repo (equivalent of `/init`):
  - `package.json` — commands/scripts, project name, dependencies
  - Linter/prettier configs — conventions, formatting rules
  - `tsconfig.json` / `babel.config.js` — path aliases
  - Directory structure — key files, architecture overview
- Use a section structure similar to existing rich CLAUDE.md files as reference: Commands, Architecture, Conventions, Key Files
- Write the baseline CLAUDE.md
- Proceed to Step 2

Either way, Step 2 starts with a CLAUDE.md to enrich.

## Step 2: Read vault docs

Find the corresponding Obsidian vault project folder.

**Vault path:** `/Users/FM08ID/Library/CloudStorage/ProtonDrive-dieter.hoeven@pm.me-folder/BRAIN/`

**Matching heuristics (try in order):**
1. Existing CLAUDE.md references — look for project name mentions, Notion/Obsidian links in the content
2. `package.json` name field — map to vault project folder name
3. Repo directory name — fuzzy match against `NN/Projects/` folder names

**Read sources:**
1. All markdown files in `NN/Projects/<Name>/`
2. Cross-project docs in `NN/Documentation/` that reference this project — these may contain shared environment, infrastructure, or setup info

**If no match found:** Ask the user: "Which vault project folder should I use for this repo?"

## Step 3: Analyze and propose changes

Compare the current CLAUDE.md sections against vault doc content. For each vault doc, extract only what's operationally relevant for Claude working in the codebase.

### What to extract from the vault

**Add or update when it fills a gap or corrects stale info:**
- Environment configs (URLs, branch-to-environment mappings, deployment triggers)
- Known issues / gotchas from technical debt docs (high priority items that could trip up development)
- Key contacts / team ownership
- Architecture updates (provider chain changes, new API clients, auth flows)
- Infrastructure endpoints (Kudu, APIM, app URLs, resource groups)

### What to always ensure

- **Obsidian vault reference section** — every CLAUDE.md must have a section pointing to the vault:
  ```markdown
  ## Obsidian Vault Documentation

  For detailed project documentation (architecture, tech debt, infrastructure, release process, etc.), consult the Obsidian vault at `NN/Projects/<Name>/`.
  ```
- **Replace Notion references** — any existing references to Notion documentation are updated to point to the Obsidian vault instead

### What to NOT extract

- Detailed architecture prose (that's what the vault is for — CLAUDE.md just needs the operational summary)
- Release plans for specific versions (ephemeral)
- Full dependency tables (derivable from `package.json`)
- Meeting notes, analysis docs
- Anything already accurately present in the CLAUDE.md

### Boundary reference

| CLAUDE.md (operational) | Vault (detailed) |
|------------------------|------------------|
| Commands to run | Full architecture prose |
| Path aliases | Dependency tables with versions |
| Provider chain summary | Detailed state management docs |
| Environment URLs + triggers | Release plans for specific versions |
| Known issues / gotchas | Full technical debt backlog |
| Key contacts (who to ask) | Meeting notes, analysis |
| Architecture one-liners | Routing/auth deep dives |
| Infrastructure endpoints | Resource provisioning guides |
| Conventions + linter rules | — |
| Vault pointer for deep docs | — |

### Present the proposal

Present a diff-style summary grouped by section:

> **Proposed CLAUDE.md changes:**
>
> - **New section: Obsidian Vault Documentation** — pointer to `NN/Projects/<Name>/`
> - **Update: Environments** — add ACC/PRD URLs from vault Infrastructure doc
> - **New section: Known Issues** — 3 items from vault Technical Debt (high priority)
> - **Remove: Notion reference** — replaced by Obsidian vault pointer
>
> Approve, or tell me what to change?

For each proposed change, show a brief preview of what would be added/changed so the user can make an informed decision.

Wait for user approval before proceeding.

## Step 4: Write changes

After approval, apply the changes:

- Use the Edit tool for modifying existing sections in CLAUDE.md
- Use the Write tool only if creating CLAUDE.md from scratch (Step 1 bootstrap)
- Preserve existing content structure — do not reorganize sections the user has arranged
- Keep additions concise and operational (CLAUDE.md style, not vault prose style)
- **REQUIRED:** Apply the `humanizer` skill when writing prose content
- If a section would exceed ~10 lines, summarize and point to the vault doc instead

Report what was changed:

> **CLAUDE.md updated:**
> - Added: Obsidian Vault Documentation section
> - Added: Known Issues section (3 items)
> - Updated: Environments section
> - Removed: Notion reference

## Key Constraints

- **CLAUDE.md is the authority on its own content.** The skill enriches, never overwrites user-curated sections without approval.
- **Vault is reference material.** Extract operational summaries, not full docs.
- **Always interactive.** No changes without user approval.
- **One project at a time.**
- **Concise additions.** CLAUDE.md should stay scannable — if a section would be longer than ~10 lines, summarize and point to the vault doc instead.
