🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7. Gestion des erreurs

## Introduction

La gestion des erreurs est l'un des aspects les plus distinctifs et importants du langage Go. Contrairement à de nombreux autres langages de programmation qui utilisent des exceptions (try/catch), Go adopte une approche explicite et pragmatique pour traiter les erreurs.

Dans ce chapitre, nous allons explorer en profondeur comment Go gère les erreurs, pourquoi cette approche est différente, et comment l'utiliser efficacement dans vos programmes.

## Pourquoi la gestion d'erreur est-elle importante ?

Dans le développement logiciel, les erreurs sont inévitables. Elles peuvent survenir pour de nombreuses raisons :

- **Ressources indisponibles** : fichiers introuvables, serveurs inaccessibles
- **Données invalides** : formats incorrects, valeurs hors limites
- **Problèmes système** : mémoire insuffisante, permissions manquantes
- **Erreurs réseau** : timeouts, connexions interrompues
- **Erreurs logiques** : conditions inattendues dans le code

Une gestion d'erreur appropriée permet de :
- **Rendre les programmes robustes** face aux conditions exceptionnelles
- **Fournir des messages informatifs** aux utilisateurs
- **Faciliter le débogage** lors du développement
- **Éviter les crashes** en production
- **Permettre la récupération** après des erreurs temporaires

## La philosophie Go

### Erreurs explicites

Go considère que les erreurs font partie intégrante du flux normal d'un programme. Au lieu de les traiter comme des "exceptions" à éviter, Go les rend **explicites** et **visibles** dans le code.

```go
// Exemple typique en Go
resultat, err := operation()
if err != nil {
    // Traiter l'erreur
    return err
}
// Utiliser le résultat
```

Cette approche force le développeur à :
- **Anticiper les erreurs** possibles
- **Traiter chaque erreur** de manière consciente
- **Rendre le code plus prévisible** et lisible

### Comparaison avec d'autres langages

**Langages avec exceptions (Java, Python, C#) :**
```java
// Java - exceptions peuvent être "oubliées"
try {
    String contenu = lireFichier("config.txt");
    // traitement...
} catch (IOException e) {
    // gestion optionnelle
}
```

**Go - gestion explicite :**
```go
// Go - impossible d'ignorer l'erreur sans intention
contenu, err := lireFichier("config.txt")
if err != nil {
    // OBLIGÉ de traiter l'erreur
    return fmt.Errorf("lecture config: %w", err)
}
// traitement...
```

## Avantages de l'approche Go

### 1. **Lisibilité**
Le flux d'erreur est visible directement dans le code. Pas de chemins cachés ou de "magic" qui peut surprendre.

### 2. **Performance**
Pas de stack unwinding coûteux comme avec les exceptions. Les erreurs sont de simples valeurs.

### 3. **Simplicité**
Une seule façon de gérer les erreurs = moins de confusion, code plus uniforme.

### 4. **Fiabilité**
Impossible d'oublier accidentellement de gérer une erreur critique.

## Inconvénients apparents

### **Verbosité**
Le code Go peut sembler répétitif avec tous ces `if err != nil`. C'est un choix conscient pour la clarté.

### **Changement d'habitude**
Pour les développeurs habitués aux exceptions, l'adaptation peut prendre du temps.

**Important :** Ces "inconvénients" deviennent rapidement des avantages quand on développe des applications robustes en équipe.

## Types d'erreurs en Go

Go distingue deux catégories principales de problèmes :

### 1. **Erreurs** (errors)
- Conditions attendues mais indésirables
- Font partie du flux normal du programme
- Doivent être gérées explicitement
- Exemples : fichier introuvable, format de données invalide

### 2. **Panics**
- Conditions exceptionnelles et inattendues
- Arrêtent l'exécution normale du programme
- Equivalent aux exceptions non gérées
- Exemples : division par zéro, accès mémoire invalide

## Structure de ce chapitre

Dans les sections suivantes, nous allons explorer :

### **7-1 : Conventions d'erreur en Go**
- Le type `error` et son interface
- Pattern de retour multiple `(result, error)`
- Vérification standard avec `if err != nil`
- Création d'erreurs simples

### **7-2 : Création d'erreurs personnalisées**
- Types d'erreur custom avec struct
- Erreurs avec contexte et métadonnées
- Wrapping et unwrapping d'erreurs
- Erreurs prédéfinies et codes d'erreur

### **7-3 : Panic et recover**
- Quand et comment utiliser `panic()`
- Mécanisme de `recover()` avec `defer`
- Patterns pour des applications robustes
- Différences entre erreurs et panics

### **7-4 : Bonnes pratiques**
- Messages d'erreur efficaces
- Stratégies de gestion d'erreur
- Testing des cas d'erreur
- Logging et monitoring
- Patterns avancés (retry, circuit breaker)

## Premier aperçu

Avant de plonger dans les détails, voici un exemple simple qui illustre l'approche Go :

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Demander à l'utilisateur un nombre
    fmt.Print("Entrez un nombre: ")
    var input string
    fmt.Scanln(&input)

    // Conversion avec gestion d'erreur
    nombre, err := strconv.Atoi(input)
    if err != nil {
        fmt.Printf("Erreur: '%s' n'est pas un nombre valide\n", input)
        return
    }

    // Calcul avec vérification
    if nombre < 0 {
        fmt.Println("Erreur: le nombre doit être positif")
        return
    }

    // Traitement réussi
    resultat := nombre * 2
    fmt.Printf("Le double de %d est %d\n", nombre, resultat)
}
```

**Ce qui se passe ici :**
1. `strconv.Atoi()` retourne `(int, error)` - pattern typique Go
2. On vérifie immédiatement `if err != nil`
3. On gère l'erreur de manière appropriée (message + arrêt)
4. On continue seulement si tout va bien

## Objectifs d'apprentissage

À la fin de ce chapitre, vous saurez :

- ✅ **Comprendre** la philosophie Go pour les erreurs
- ✅ **Utiliser** les conventions standard `(result, error)`
- ✅ **Créer** vos propres types d'erreur personnalisés
- ✅ **Gérer** les situations exceptionnelles avec panic/recover
- ✅ **Appliquer** les bonnes pratiques professionnelles
- ✅ **Écrire** du code robuste et maintenable
- ✅ **Déboguer** efficacement grâce à des erreurs informatives

## Prérequis

Avant de commencer ce chapitre, assurez-vous de maîtriser :
- Les fonctions et leurs valeurs de retour
- Les interfaces de base
- Les types de données (string, struct)
- La déclaration de variables

Ces concepts sont essentiels car la gestion d'erreur en Go s'appuie fortement sur les interfaces et les valeurs de retour multiples.

---

**Prêt à découvrir comment Go rend la gestion d'erreur simple, explicite et efficace ? Commençons par les conventions de base !**

⏭️
