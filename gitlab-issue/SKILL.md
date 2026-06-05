---
name: gitlab-issue
description: >
  Génère une issue GitLab structurée à partir d'une description en langage
  naturel et produit la commande glab prête à exécuter. Se déclenche quand
  l'utilisateur veut créer, rédiger ou ouvrir une issue GitLab.
  Mots-clés déclencheurs : "créer une issue", "ouvrir une issue", "ticket GitLab",
  "signaler un bug", "demande de feature", "issue GitLab", "nouvelle issue",
  "rédiger un ticket".
---

# GitLab Issue Generator

## Vue d'ensemble

Ce skill génère une issue GitLab complète et homogène à partir d'une description
libre, puis produit la commande `glab issue create` prête à copier-coller ou à
exécuter directement (Claude Code avec accès bash).

**Prérequis :** [`glab`](https://gitlab.com/gitlab-org/cli) installé et
authentifié (`glab auth login`).

---

## Entrées attendues

L'utilisateur peut fournir **n'importe quelle combinaison** de :

- Description libre du problème ou de la feature (langage naturel)
- Logs, stack traces, messages d'erreur
- Captures d'écran ou descriptions de comportement observé
- Contexte technique (module, écran, version, plateforme)

---

## Workflow

### Étape 1 — Qualifier le type

Identifier immédiatement le type d'issue depuis la description :

| Type | Signal |
|------|--------|
| `BUG` | Comportement inattendu, crash, erreur, régression |
| `FEAT` | Nouvelle fonctionnalité, amélioration, demande utilisateur |
| `TASK` | Maintenance, dette technique, refactoring, mise à jour de dépendance |
| `QUESTION` | Clarification, discussion, investigation |

Si le type n'est pas clair, poser **une seule question** : `"C'est un bug ou
une demande de feature ?"`

### Étape 2 — Questions ciblées (si nécessaire)

Poser uniquement les questions dont la réponse manque et qui sont bloquantes
pour remplir l'issue. Maximum **2 questions** avant de générer.

Questions candidates :

- **Reproduction** (BUG) : les étapes pour reproduire sont-elles connues ?
- **Impact** (BUG) : toujours reproductible, aléatoire, ou isolé ?
- **Plateforme** : Android / iOS / Desktop / Web / toutes ?
- **Module/écran** : quel module ou écran est concerné ?
- **Labels du projet** : y a-t-il des labels spécifiques à utiliser ?

### Étape 3 — Génération de l'issue

Remplir le template adapté au type, puis produire la commande `glab`.

---

## Format du titre

```
[TYPE] Description courte en français (impératif, 72 caractères max tout compris)
```

Même convention que les MRs : verbe à l'impératif, pas de point final, type en
majuscules entre crochets.

Exemples :
- `[BUG] Corriger le crash au démarrage sur iOS 17`
- `[FEAT] Ajouter la signature multiple sur l'écran de livraison`
- `[TASK] Mettre à jour Kotlin vers 2.1`

---

## Templates de description

### Bug

```markdown
## Description

[Ce qui se passe et pourquoi c'est un problème]

## Étapes pour reproduire

1. 
2. 
3. 

## Comportement attendu

[Ce qui devrait se passer]

## Comportement observé

[Ce qui se passe réellement]

## Contexte technique

- **Plateforme :** [Android / iOS / Desktop / Web]
- **Version de l'app :** [si connue]
- **Environnement :** [dev / staging / prod]

## Logs / Stack trace

```
[coller ici si disponible]
```
```

### Feature

```markdown
## Description / Contexte

[Pourquoi cette feature est nécessaire, quel besoin elle couvre]

## Comportement souhaité

[Ce que l'utilisateur pourra faire une fois la feature implémentée]

## Critères d'acceptance

- [ ] 
- [ ] 
- [ ] 

## Notes techniques

[Contraintes, dépendances, points d'attention — laisser vide si rien de spécifique]
```

### Task

```markdown
## Description

[Ce qui doit être fait et pourquoi]

## Périmètre

[Fichiers, modules ou composants concernés]

## Critères de complétion

- [ ] 
- [ ] 
```

---

## Commande glab

Après avoir généré la description, produire la commande complète.

### Format standard (description inline)

```bash
glab issue create \
  --title "[TYPE] Description" \
  --description "$(cat <<'EOF'
[contenu markdown de la description]
EOF
)" \
  --label "bug"
```

### Options utiles à inclure selon le contexte

| Option | Usage |
|--------|-------|
| `--label "bug,ios"` | Labels (séparés par des virgules) |
| `--assignee "@me"` | S'assigner l'issue |
| `--milestone "Sprint 42"` | Associer un milestone |
| `--confidential` | Issue confidentielle |

Si l'utilisateur n'a pas précisé de labels, suggérer les plus pertinents
selon le type : `bug`, `feature`, `maintenance`, `ios`, `android`, etc.

### Avec Claude Code (exécution directe)

Si le contexte est Claude Code avec accès bash, proposer d'exécuter la commande
après confirmation :

> La commande est prête. Je l'exécute pour créer l'issue sur GitLab ?

---

## Langue et conventions

- Rédiger en **français** sauf les termes techniques
- Être factuel : pas de formulations vagues ("parfois", "il semblerait que")
- Les critères d'acceptance commencent par un verbe à l'infinitif :
  "Afficher un message d'erreur", "Persister le choix de l'utilisateur"
- Ne pas pré-cocher les cases de la checklist
