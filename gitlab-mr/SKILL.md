---
name: gitlab-mr
description: >
  Génère des descriptions de merge requests GitLab standardisées et uniformes.
  Utiliser ce skill dès qu'un utilisateur veut créer, rédiger, formater ou améliorer
  une merge request (MR) GitLab. Se déclenche sur les mots-clés "merge request", "MR",
  "créer une MR", "rédiger une MR", "template MR", "gitlab MR", "pull request",
  "ouvrir une MR", ou lorsque l'utilisateur fournit un fichier mergerequest.md ou un
  extrait de git log dans l'intention de préparer une MR. Prend en entrée un brouillon
  et/ou un historique de commits, et produit une MR complète, homogène et conforme aux
  conventions de l'équipe — sans jamais exécuter de commandes bash.
---

# GitLab Merge Request Generator

## Vue d'ensemble

Ce skill génère des descriptions de merge requests GitLab **standardisées** à partir d'inputs fournis par l'utilisateur. Il ne nécessite aucune commande bash : tout se base sur les informations transmises en texte.

## Entrées attendues

L'utilisateur peut fournir **un ou plusieurs** des éléments suivants :

| Entrée | Description |
|--------|-------------|
| Fichier `mergerequest.md` | Brouillon libre : contexte, objectif, notes de l'auteur |
| `git log --oneline -n 20` | Historique des commits récents (texte brut copié-collé) |
| Description orale | L'utilisateur décrit ce qu'il a fait en langage naturel |

> Si des informations essentielles manquent pour remplir une section, poser des questions ciblées **avant** de générer la MR — ne jamais inventer de détails techniques.

---

## Format du titre

```
[TYPE] Brève description en français (impératif, 72 caractères max tout compris)
```

### Types disponibles

| Type | Usage |
|------|-------|
| `[FEAT]` | Nouvelle fonctionnalité |
| `[FIX]` | Correction de bug |
| `[DOCS]` | Documentation uniquement |
| `[REFACTOR]` | Refactoring sans changement de comportement observable |
| `[TEST]` | Ajout ou modification de tests |
| `[CHORE]` | Maintenance, dépendances, CI/CD, configuration |

### Règles du titre

- Verbe à **l'impératif** : "Ajouter", "Corriger", "Extraire", "Migrer"…
- **Pas de point final**
- Maximum **72 caractères** tout compris (crochets inclus)
- Choisir le type le **plus précis** ; si une MR mélange FEAT et FIX, suggérer de la découper si possible
- Exemple valide : `[FEAT] Ajouter l'authentification SSO via Azure AD`
- Exemple invalide : `[feat] ajout de l'auth sso.` (casse, point, minuscules)

---

## Structure de la description

Lire le template dans `assets/mr_template.md` et l'adapter au contexte.

### Règles de remplissage par section

**Description / Contexte**
- Expliquer le *pourquoi*, pas le *quoi*
- Répondre à : quel problème est résolu ? quel besoin est couvert ? quel ticket est adressé ?
- 2 à 5 phrases maximum

**Changements effectués**
- Liste à puces avec des **verbes d'action au passé** : "Ajouté", "Supprimé", "Migré", "Extrait"
- Rester factuel et technique
- Regrouper par domaine fonctionnel si beaucoup de changements

**Breaking changes**
- Inclure **uniquement si** il y a un impact sur la compatibilité (API, schéma BDD, contrats, config)
- Sinon : écrire simplement `Aucun`

**Checklist**
- Afficher toutes les cases **décochées** `[ ]` — c'est l'auteur qui les cochera
- Ne jamais pré-cocher des cases à la place de l'auteur

**Notes pour les reviewers**
- Points d'attention particuliers, choix d'architecture à valider, fichiers critiques à relire en priorité
- Peut rester vide si rien de spécifique : écrire `RAS`

---

## Processus de génération

Suivre ces étapes dans l'ordre :

1. **Analyser les inputs** — lire le brouillon et/ou le git log pour comprendre le périmètre, les fichiers touchés et l'intention
2. **Inférer le type** — déterminer le `[TYPE]` le plus adapté selon la table ci-dessus
3. **Formuler le titre** — court, clair, impératif, ≤ 72 caractères
4. **Remplir le template** — en chargeant `assets/mr_template.md` et en complétant chaque section
5. **Présenter la MR finale** — afficher le bloc markdown complet, prêt à être copié dans GitLab

---

## Langue et conventions

- Rédiger en **français** sauf les termes techniques (noms de fonctions, variables, classes, paths)
- Être **concis mais exhaustif** : pas de remplissage, pas d'omission
- Les listes de changements commencent par une majuscule, pas de point final à chaque item
- Utiliser le formatage Markdown natif GitLab (émojis sparingly, titres `##`, listes `-`)
