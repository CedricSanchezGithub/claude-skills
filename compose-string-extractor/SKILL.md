---
name: compose-string-extractor
description: >
  Extrait les chaînes de caractères codées en dur d'un fichier Kotlin Compose
  Multiplatform et les remplace par des ressources localisées. Se déclenche
  quand l'utilisateur fournit un fichier Kotlin avec des strings hardcodées à
  externaliser. Mots-clés déclencheurs : "extraire les strings", "externaliser
  les chaînes", "hardcoded strings", "ressources string", "strings codés en dur",
  "localisation", "composeResources", "strings XML".
---

# Compose String Extractor

## Vue d'ensemble

Ce skill analyse un fichier Kotlin Compose Multiplatform, identifie toutes les
chaînes de caractères codées en dur dans l'UI, génère les ressources XML
correspondantes et produit le code Kotlin modifié — sans toucher à quoi que ce
soit d'autre.

Cible : **KMP / Compose Multiplatform** (structure `composeResources`).

---

## Entrée attendue

Un fichier Kotlin (`.kt`) contenant des composables Compose avec des chaînes
hardcodées. L'utilisateur peut le coller directement ou mentionner son chemin.

---

## Workflow

### Étape 1 — Analyse silencieuse

Lire le fichier et identifier :

- Le **package** déclaré en haut du fichier
- Toutes les **chaînes UI hardcodées** (voir critères ci-dessous)
- Les chaînes contenant des **variables/placeholders** (`$variable` ou
  `${expression}`)

### Étape 2 — Question ciblée (si nécessaire)

Poser **au plus une question** avant de générer, uniquement si :

- Un préfixe de clé est ambigu et ne peut pas être déduit du contenu
- Des strings semblent techniques mais pourraient être de l'UI (demander
  confirmation)

Si tout est clair depuis le code, passer directement à l'étape 3.

### Étape 3 — Génération

Produire dans cet ordre :

1. **Contenu du fichier XML** — complet, prêt à créer ou à fusionner
2. **Fichier Kotlin modifié** — complet, avec toutes les occurrences remplacées
3. **Tableau récapitulatif** — clé · valeur originale · type (simple/placeholder)

---

## Critères d'identification des strings à extraire

### ✅ Extraire

- Littéraux dans les paramètres UI : `Text("…")`, `Button(… ) { Text("…") }`,
  `contentDescription = "…"`, `label = "…"`, `placeholder = "…"`,
  `hint = "…"`, etc.
- Valeurs visibles par l'utilisateur final dans n'importe quel composable

### ❌ Ne pas extraire

- Messages de log : `Log.d("TAG", "…")`, `println("…")`
- Constantes techniques : noms de routes, tags, identifiants, URLs, clés d'API
- Strings dans des annotations (`@` )
- Commentaires
- Strings visiblement non destinées à l'utilisateur final (ex. : `"DEBUG"`,
  `"TODO"`, noms de classes)

En cas de doute sur une string, la signaler dans le tableau récapitulatif avec
la mention `⚠️ à vérifier` plutôt que de l'ignorer silencieusement.

---

## Règles de nommage des clés

### Format général

```
snake_case, tout en minuscules, sans accents, sans articles
```

### Algorithme

1. Partir du contenu de la string
2. Supprimer les articles (`le`, `la`, `les`, `un`, `une`, `des`, `the`, `a`,
   `an`, `l'`, `d'`)
3. Supprimer la ponctuation
4. Remplacer les espaces par `_`
5. Translittérer les accents (`é` → `e`, `à` → `a`, etc.)
6. Tronquer à **5 mots maximum**

Exemples :

| String originale | Clé générée |
|------------------|-------------|
| `"En attente d'une signature unique"` | `en_attente_signature_unique` |
| `"Valider et continuer"` | `valider_continuer` |
| `"Erreur lors du chargement"` | `erreur_chargement` |
| `"Bonjour, $name !"` | `bonjour_nom` |

### Gestion des doublons

Si deux strings identiques existent dans le fichier → **une seule clé**, toutes
les occurrences pointent vers la même ressource.

Si deux strings différentes produisent la même clé → ajouter un suffixe
numérique (`_2`, `_3`…) et signaler l'ambiguïté.

---

## Gestion des placeholders

Les strings Kotlin avec interpolation (`$variable` ou `${expression}`) sont
converties au format XML Android/KMP avec placeholders positionnels.

| Kotlin original | XML généré | Appel Kotlin |
|-----------------|------------|--------------|
| `"Bonjour $name"` | `Bonjour %1$s` | `stringResource(Res.string.bonjour_nom, name)` |
| `"$count élément(s)"` | `%1$d élément(s)` | `stringResource(Res.string.count_elements, count)` |
| `"De $start à $end"` | `De %1$s à %2$s` | `stringResource(Res.string.de_start_a_end, start, end)` |

Types utilisés : `%s` (String), `%d` (Int), dans leur forme positionnelle
`%1$s`, `%2$d`, etc. pour garantir la compatibilité avec les langues à ordre
de mots différent.

---

## Format du fichier XML

Structure `composeResources` standard :

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="en_attente_signature_unique">En attente d'une signature unique</string>
    <string name="valider_continuer">Valider et continuer</string>
</resources>
```

### Chemin du fichier

Dérivé du **dernier segment du package** du fichier source :

```
Package : fr.ca.cats.signature.edi.ui.screens.dashboard.deliver
Fichier : shared/ui/src/commonMain/composeResources/values/deliver.xml
```

Si le fichier XML existe déjà, indiquer les entrées à **ajouter** (ne pas
réécrire les entrées existantes non concernées).

---

## Format de sortie

### 1. Fichier XML

````
### `shared/ui/src/commonMain/composeResources/values/[nom].xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <!-- nouvelles entrées -->
</resources>
```
````

### 2. Fichier Kotlin modifié

````
### `[NomDuFichier].kt` (modifié)

```kotlin
// contenu complet avec les remplacements
```
````

### 3. Tableau récapitulatif

| Clé | Valeur originale | Type |
|-----|------------------|------|
| `en_attente_signature_unique` | `"En attente d'une signature unique"` | simple |
| `bonjour_nom` | `"Bonjour $name"` | placeholder |

---

## Contraintes strictes

- **Ne pas compiler**, ne pas exécuter, ne pas tester
- **Ne pas modifier** d'autres fichiers que le `.kt` fourni et le `.xml` cible
- **Ne pas ajouter** d'imports manquants (les signaler en fin de réponse si
  nécessaire)
- Respecter la casse et l'indentation existantes du fichier Kotlin
