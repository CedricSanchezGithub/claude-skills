# claude-skills

A collection of reusable [agent skills](https://github.com/agentskills/agentskills) — standardized, opinionated prompts that encode team conventions and workflows.

Skills use the open Agent Skills standard and work with **Claude Code**, **GitHub Copilot** (cloud agent, CLI, VS Code agent mode), and any other compatible AI tool.

---

## Skills

### `gitlab-issue`
Generates a structured GitLab issue (bug, feature, task) from a natural language description, and outputs a ready-to-run `glab issue create` command.

**Input:** free-form description, logs, stack trace, observed behavior  
**Output:** complete issue (title + type-specific description template) + `glab` command

With Claude Code (bash access), can create the issue directly on GitLab after confirmation.

→ [View skill](./gitlab-issue/SKILL.md)

---

### `gitlab-mr`
Generates standardized GitLab merge request descriptions from a draft and/or a `git log` history.

**Input:** a `mergerequest.md` draft, a pasted `git log --oneline -n 20`, or a verbal description  
**Output:** a complete MR with title `[TYPE] Description` and structured sections

Supported types: `FEAT` · `FIX` · `DOCS` · `REFACTOR` · `TEST` · `CHORE`

→ [View skill](./gitlab-mr/SKILL.md)

---

### `compose-decomposer`
Refactors an overly complex Compose Multiplatform screen into modular, maintainable composables.

**Input:** a Kotlin composable file  
**Output:** a single Kotlin file with a public root composable and private sub-composables

Applies: state hoisting, single responsibility, `modifier: Modifier = Modifier`, no business logic in composables.  
Targets all Compose Multiplatform platforms: Android, iOS, Desktop, Web.

→ [View skill](./compose-decomposer/SKILL.md)

---

### `compose-string-extractor`
Extracts hardcoded strings from a Kotlin Compose Multiplatform file and generates the corresponding XML resources and updated Kotlin code.

**Input:** a Kotlin file with hardcoded strings in composables  
**Output:** XML resource file (create or append) + updated Kotlin file with `stringResource(...)` calls

Handles: duplicates, placeholders (`$variable` → `%1$s`), content-derived key naming, XML path derived from package name.

→ [View skill](./compose-string-extractor/SKILL.md)

---

## Installation

Each skill is a folder containing a `SKILL.md` file (and optional assets). Copy the folder into your project's skills directory — the tool picks it up automatically.

### Claude Code

```bash
# Project-level
cp -r gitlab-mr /your/project/.claude/skills/

# User-level (available in all projects)
cp -r gitlab-mr ~/.claude/skills/
```

### GitHub Copilot (cloud agent / CLI / VS Code agent mode)

```bash
# Project-level
cp -r gitlab-mr /your/project/.github/skills/

# Personal (shared across projects)
cp -r gitlab-mr ~/.copilot/skills/
```

> Skills use the open [Agent Skills](https://github.com/agentskills/agentskills) standard — the same `SKILL.md` works for both tools, no adaptation needed.

---

## Structure

```
claude-skills/
├── README.md
├── gitlab-issue/
│   └── SKILL.md
├── gitlab-mr/
│   ├── SKILL.md
│   └── assets/
│       └── mr_template.md
├── compose-decomposer/
│   └── SKILL.md
└── compose-string-extractor/
    └── SKILL.md
```

---

## Contributing

Skills follow a simple convention:

- `SKILL.md` at the root of each skill directory, with a YAML frontmatter block (`name`, `description`)
- Assets (templates, examples) in an `assets/` subdirectory
- Skills are text-only by default; bash execution is opt-in and always requires user confirmation
