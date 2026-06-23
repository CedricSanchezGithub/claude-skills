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

**Example prompts:**
```
Crée une issue GitLab : l'app crash au démarrage sur iOS 17 quand on arrive depuis une notification push.

Ouvre un ticket pour ajouter la signature multiple sur l'écran de livraison.

Signale un bug : la liste des documents ne se rafraîchit pas après une synchronisation.

Crée une issue de maintenance pour mettre à jour Kotlin vers 2.1.
```

→ [View skill](./gitlab-issue/SKILL.md)

---

### `gitlab-mr`
Generates standardized GitLab merge request descriptions from a draft and/or a `git log` history.

**Input:** a `mergerequest.md` draft, a pasted `git log --oneline -n 20`, or a verbal description  
**Output:** a complete MR with title `[TYPE] Description` and structured sections

Supported types: `FEAT` · `FIX` · `DOCS` · `REFACTOR` · `TEST` · `CHORE`

**Example prompts:**
```
Génère une MR GitLab à partir de ce git log : [coller git log --oneline -n 20]

Rédige une merge request : j'ai ajouté l'authentification SSO via Azure AD.

Voici mon brouillon mergerequest.md, génère la MR complète : [coller le contenu]

Crée une MR pour ce refactoring du module de paiement.
```

→ [View skill](./gitlab-mr/SKILL.md)

---

### `compose-decomposer`
Refactors an overly complex Compose Multiplatform screen into modular, maintainable composables.

**Input:** a Kotlin composable file  
**Output:** a single Kotlin file with a public root composable and private sub-composables

Applies: state hoisting, single responsibility, `modifier: Modifier = Modifier`, no business logic in composables.  
Targets all Compose Multiplatform platforms: Android, iOS, Desktop, Web.

**Example prompts:**
```
Décompose mon composable, il fait 300 lignes : [coller le fichier Kotlin]

Mon écran est trop grand, refactorise-le en sous-composables : [coller le code]

Extraire des composants de cet écran Compose Multiplatform.

Nettoie mon compose, il y a trop de responsabilités mélangées : [coller le fichier]
```

→ [View skill](./compose-decomposer/SKILL.md)

---

### `compose-string-extractor`
Extracts hardcoded strings from a Kotlin Compose Multiplatform file and generates the corresponding XML resources and updated Kotlin code.

**Input:** a Kotlin file with hardcoded strings in composables  
**Output:** XML resource file (create or append) + updated Kotlin file with `stringResource(...)` calls

Handles: duplicates, placeholders (`$variable` → `%1$s`), content-derived key naming, XML path derived from package name.

**Example prompts:**
```
Extrais les strings hardcodées de ce fichier Kotlin : [coller le fichier]

Externalise les chaînes de caractères de cet écran pour la localisation : [coller le code]

Ce composable a des strings codés en dur, génère les ressources XML et le code modifié.

Prépare ce fichier Compose Multiplatform pour la localisation.
```

→ [View skill](./compose-string-extractor/SKILL.md)

---

### `legacy-screen-flow-analysis`
Analyzes how a screen in a legacy mobile codebase behaves between user arrival and stable render, and produces a structured markdown report covering lifecycle, state management, data fetching, business rules, and side effects.

**Input:** screen name, entry-point file path, framework (Flutter/Dart by default, or Android/Kotlin, iOS/Swift, React Native)  
**Output:** structured markdown report saved as `<screen-name>-analysis.md`

**Example prompts:**
```
Analyse le flow de l'écran HomeScreen — le fichier d'entrée est lib/screens/home_screen.dart.

Je veux comprendre ce que fait cet écran avant de le migrer vers Compose : [coller le fichier]

Documente le cycle de vie de la vue ProfileViewController.swift pour préparer une réécriture.

Cartographie le comportement de l'écran de livraison (DeliveryScreen) dans ce projet Flutter/Riverpod.
```

→ [View skill](./legacy-screen-flow-analysis/SKILL.md)

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
├── compose-string-extractor/
│   └── SKILL.md
└── legacy-screen-flow-analysis/
    ├── SKILL.md
    └── references/
        └── report-template.md
```

---

## Contributing

Skills follow a simple convention:

- `SKILL.md` at the root of each skill directory, with a YAML frontmatter block (`name`, `description`)
- Assets (templates, examples) in an `assets/` subdirectory
- Skills are text-only by default; bash execution is opt-in and always requires user confirmation
