---
name: compose-decomposer
description: >
  Décompose un écran Compose Multiplatform trop complexe en sous-composables
  modulaires, lisibles et maintenables. Se déclenche quand l'utilisateur fournit
  un composable trop long, trop imbriqué, ou avec trop de responsabilités.
  Mots-clés déclencheurs : "décomposer mon composable", "refactoriser mon écran",
  "mon écran est trop grand", "extraire des composants", "modulariser mon UI",
  "trop complexe", "nettoyer mon compose", "découper mon écran".
  Pose des questions ciblées avant de générer le code refactorisé.
---

# Compose Multiplatform — Screen Decomposer

## Vue d'ensemble

Ce skill analyse un écran Compose Multiplatform trop complexe et le refactorise
en sous-composables cohérents — en respectant les bonnes pratiques Compose :
state hoisting, single responsibility, paramètres minimaux, pas de logique métier.

Cible : **Compose Multiplatform (Android, iOS, Desktop, Web)**.
Sortie : **un seul fichier Kotlin** avec le composable principal et ses sous-composables privés.

---

## Workflow

### Étape 1 — Questions d'analyse (toujours avant de générer)

Après avoir lu le code fourni, poser uniquement les questions pertinentes parmi celles-ci.
Si le code est explicite, se limiter à 1 ou 2 questions.

Questions candidates :

- **Objectif de l'écran** : quel est son rôle en une phrase ?
- **Sections déjà identifiées** : y a-t-il des blocs que tu vois toi-même comme extractibles ?
  (ex : "le header", "la liste", "le formulaire du bas")
- **Réutilisabilité** : certaines parties pourraient-elles être utilisées ailleurs dans l'app ?
  Si oui, les rendre plus génériques.
- **Contraintes de nommage** : un préfixe ou une convention de nom à respecter ?

### Étape 2 — Analyse du code

Identifier les signaux de complexité suivants :

| Signal | Seuil d'alerte |
|--------|----------------|
| Longueur | > 150 lignes dans un seul composable |
| Paramètres | > 5 paramètres sur un composable |
| Nesting | > 4 niveaux d'imbrication |
| Sections visuelles | Blocs visuellement distincts sans séparation logique |
| Responsabilités mélangées | Un composable gère plusieurs préoccupations visuelles sans lien |

### Étape 3 — Génération du code refactorisé

Produire le code dans **un seul fichier Kotlin**, structuré ainsi :

```kotlin
// 1. Composable principal (public)
@Composable
fun XxxScreen(
    modifier: Modifier = Modifier,
    ...
) { ... }

// 2. Sous-composables extraits (private)
@Composable
private fun XxxHeader(
    modifier: Modifier = Modifier,
    ...
) { ... }

@Composable
private fun XxxContent(
    modifier: Modifier = Modifier,
    ...
) { ... }
```

---

## Bonnes pratiques Compose à appliquer

### State hoisting
- Remonter l'état au niveau le plus haut nécessaire
- Les sous-composables reçoivent des **valeurs** et des **lambdas**, jamais un ViewModel
- Préférer `onAction: () -> Unit` à `viewModel: XxxViewModel`
- Un composable affiche — il ne décide pas

### Single responsibility
- Chaque sous-composable a **une seule raison de changer**
- Séparer : affichage d'un item / affichage d'une liste / affichage d'une section / layout global

### Paramètres minimaux
- Ne passer que ce dont le composable a réellement besoin
- Si > 4 paramètres liés entre eux : les regrouper dans un `data class` dédié
- Éviter de passer l'objet UiState entier si seul un champ est consommé

### Modifier
- Tout composable extractible doit accepter `modifier: Modifier = Modifier` en premier paramètre optionnel
- Ne jamais appliquer de taille ou de padding fixe à l'intérieur d'un composable générique — laisser le parent décider

### Nommage

| Type | Convention |
|------|-----------|
| Écran principal | `XxxScreen` |
| Sections | `XxxHeader`, `XxxBody`, `XxxFooter` |
| Sous-sections | `XxxInfoSection`, `XxxActionsSection` |
| Éléments de liste | `XxxItem`, `XxxCard`, `XxxRow` |
| Dialogs / Sheets | `XxxDialog`, `XxxBottomSheet` |

### Pas de logique métier dans les composables
- Aucun appel à un repository, use case ou service
- Les lambdas remontent les événements, elles ne les traitent pas
- Pas d'effets de bord (pas de `LaunchedEffect` avec logique métier dans les sous-composables)

### Multiplatform
- Pas d'API platform-spécifique dans les composables partagés
- `expect`/`actual` uniquement si un comportement natif est explicitement nécessaire
- Préférer les composants du common module Compose

---

## Format de sortie

1. **Plan de décomposition** — liste des composables extraits avec leur rôle en une ligne
2. **Code Kotlin complet** dans un bloc de code unique
3. **Note de refactoring** (si des choix méritent une explication : regroupement en data class, décision de hoisting, etc.)
