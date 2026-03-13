# Obsidian Documentation Template

This is the living template for project documentation in the Obsidian vault. It defines the page structure, formatting conventions, and section templates for each page type. Updated as new projects inform better conventions.

## Formatting Conventions

### Obsidian Flavored Markdown rules

- **Frontmatter** (YAML properties) at the top of every note — required to prevent the Linter plugin from auto-adding defaults
- **Wikilinks** for all internal vault links: `[[Note Name]]`, `[[Note Name|Display Text]]`, `[[Note Name#Heading]]`
- **Standard markdown links** for external URLs only: `[text](url)`
- **Callouts** use `> [!type]` syntax:
  ```markdown
  > [!info] Title
  > Content here.
  ```
- **Tables** use standard pipe table syntax:
  ```markdown
  | Column 1 | Column 2 |
  |----------|----------|
  | Value 1  | Value 2  |
  ```
- **Tags** use `#tag` inline or in frontmatter `tags` property. Nested tags: `#nn/project`, `#nn/docs`
- **Highlights** use `==highlighted text==` for key information
- **Code blocks** use triple backticks with language identifier

### Wikilink conventions

Follow the vault's CLAUDE.md conventions:
- `[[Project Name]]` for MOC landing pages
- `[[Project Name/Topic|Display Text]]` for docs with duplicate names across projects
- `[[Topic]]` for uniquely named docs

### Callout types

| Usage | Callout |
|-------|---------|
| Important notices, warnings | `> [!warning]` |
| Credentials, secrets, sensitive info | `> [!danger]` |
| Tips, best practices | `> [!tip]` |
| Additional context, background | `> [!info]` |
| Known issues, bugs | `> [!bug]` |
| Action items, things to do | `> [!todo]` |

---

## Page Templates

### MOC — Root page (always created)

File: `NN/Projects/<Name>/<Name>.md`

The landing page for the project. Links to all topic docs via wikilinks. No container pages — link directly to topic docs.

**Standard frontmatter:**
```yaml
---
tags:
  - nn/project
aliases:
  - <Alternative Name if applicable>
---
```

**Sections in order:**

1. **About** — A few sentences about what the project is, what it does, who it's for
2. **Documentation** — Wikilinks to all topic pages created for this project
3. **Project shortcuts** — Tables with DevOps links and operational info (repo, pipelines, boards, environments)
4. **Reference** — Table with project identifiers (bundle ID, app ID, etc.) if applicable

### Topic pages — project-driven

All pages beyond the MOC are driven by what the project actually needs. Design pages around the project's content, not a fixed checklist. The goal is pages with **clear subject boundaries** that are easy to find and update.

**Guidelines:**
- Each page should cover one distinct topic someone would look up
- Prefer specific, descriptive names (e.g., "Care Product Data Mapping" over "API Documentation")
- If a page covers two unrelated topics, split it
- If a topic is small enough to fit in the MOC, don't create a separate page for it

**Standard frontmatter for all topic pages:**
```yaml
---
tags:
  - nn/project
---
```

**Common pages that naturally recur** (use when they fit, skip when they don't):
- Technical overview / architecture — tech stack, structure, providers, routing
- Technical debt — prioritized table with Item, Impact, Effort, Notes columns
- Release process — CI/CD workflows, pipelines, deployment steps
- Infrastructure — cloud resources, environments, endpoints

**Section templates to reuse within pages:**

- Technical debt table:
  ```markdown
  | Item | Impact | Effort | Notes |
  |------|--------|--------|-------|
  ```
  Group by priority (High / Medium / Low) when the list is long enough.

- Dependency table:
  ```markdown
  | Package | Version | Purpose |
  |---------|---------|---------|
  ```

- Environment table:
  ```markdown
  | Environment | Purpose | URL |
  |-------------|---------|-----|
  ```

### Cross-project docs

File: `NN/Documentation/<Topic>.md`

For documentation that applies beyond a single project.

**Standard frontmatter:**
```yaml
---
tags:
  - nn/docs
---
```
