🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4-2 : Paramètres et valeurs de retour

## Introduction

Dans la section précédente, nous avons appris les bases de la déclaration et de l'appel de fonctions. Maintenant, nous allons approfondir deux aspects cruciaux : comment transmettre des données aux fonctions (paramètres) et comment récupérer des résultats (valeurs de retour). Cette section vous donnera une maîtrise complète de ces mécanismes essentiels.

## Comprendre les paramètres

Les paramètres sont comme des "boîtes aux lettres" où vous déposez des informations pour que la fonction puisse les utiliser. Chaque paramètre a un nom et un type, et la fonction peut les utiliser comme des variables locales.

### Paramètres par valeur (comportement par défaut)

En Go, les paramètres sont passés **par valeur** par défaut. Cela signifie que la fonction reçoit une **copie** de la valeur, pas la valeur originale.

```go
package main

import "fmt"

func modifier(x int) {
    fmt.Println("Dans la fonction, x avant modification:", x)
    x = 100
    fmt.Println("Dans la fonction, x après modification:", x)
}

func main() {
    nombre := 42
    fmt.Println("Avant l'appel de fonction:", nombre)

    modifier(nombre)

    fmt.Println("Après l'appel de fonction:", nombre)
}
```

**Résultat :**
```
Avant l'appel de fonction: 42
Dans la fonction, x avant modification: 42
Dans la fonction, x après modification: 100
Après l'appel de fonction: 42
```

**Explication :**
- La variable `nombre` garde sa valeur originale (42)
- Les modifications dans la fonction n'affectent que la copie locale
- C'est un comportement sûr qui évite les effets de bord non désirés

### Paramètres par référence avec les pointeurs

Pour modifier la valeur originale, nous devons utiliser des **pointeurs** :

```go
package main

import "fmt"

func modifierAvecPointeur(x *int) {
    fmt.Println("Dans la fonction, valeur pointée avant:", *x)
    *x = 100
    fmt.Println("Dans la fonction, valeur pointée après:", *x)
}

func main() {
    nombre := 42
    fmt.Println("Avant l'appel de fonction:", nombre)

    modifierAvecPointeur(&nombre)

    fmt.Println("Après l'appel de fonction:", nombre)
}
```

**Résultat :**
```
Avant l'appel de fonction: 42
Dans la fonction, valeur pointée avant: 42
Dans la fonction, valeur pointée après: 100
Après l'appel de fonction: 100
```

**Explication :**
- `*int` indique que le paramètre est un pointeur vers un entier
- `&nombre` passe l'adresse de la variable
- `*x` déréférence le pointeur pour accéder à la valeur originale

### Comparaison pratique : avec et sans pointeurs

```go
package main

import "fmt"

// Sans pointeur : ne modifie pas l'original
func doublerParValeur(x int) int {
    x = x * 2
    return x
}

// Avec pointeur : modifie l'original
func doublerParPointeur(x *int) {
    *x = *x * 2
}

func main() {
    // Test sans pointeur
    nombre1 := 5
    resultat := doublerParValeur(nombre1)
    fmt.Printf("Par valeur - Original: %d, Résultat: %d\n", nombre1, resultat)

    // Test avec pointeur
    nombre2 := 5
    doublerParPointeur(&nombre2)
    fmt.Printf("Par pointeur - Après modification: %d\n", nombre2)
}
```

**Résultat :**
```
Par valeur - Original: 5, Résultat: 10
Par pointeur - Après modification: 10
```

## Paramètres avec des types complexes

### Slices et maps : un cas particulier

Les slices et maps sont des références, leur comportement est donc différent :

```go
package main

import "fmt"

func modifierSlice(s []int) {
    s[0] = 999  // Modifie l'élément original
    fmt.Println("Dans la fonction:", s)
}

func ajouterElementSlice(s []int) []int {
    s = append(s, 100)  // Ceci peut créer un nouveau slice
    fmt.Println("Dans la fonction après append:", s)
    return s
}

func main() {
    // Test avec modification d'élément
    nombres := []int{1, 2, 3}
    fmt.Println("Avant modification:", nombres)
    modifierSlice(nombres)
    fmt.Println("Après modification:", nombres)

    fmt.Println("---")

    // Test avec ajout d'élément
    nombres2 := []int{1, 2, 3}
    fmt.Println("Avant append:", nombres2)
    nouveauSlice := ajouterElementSlice(nombres2)
    fmt.Println("Après append - original:", nombres2)
    fmt.Println("Après append - nouveau:", nouveauSlice)
}
```

**Résultat :**
```
Avant modification: [1 2 3]
Dans la fonction: [999 2 3]
Après modification: [999 2 3]
---
Avant append: [1 2 3]
Dans la fonction après append: [1 2 3 100]
Après append - original: [1 2 3]
Après append - nouveau: [1 2 3 100]
```

### Structs : par valeur et par référence

```go
package main

import "fmt"

type Personne struct {
    Nom string
    Age int
}

// Par valeur : ne modifie pas l'original
func vieillirParValeur(p Personne) {
    p.Age++
    fmt.Printf("Dans la fonction: %s a %d ans\n", p.Nom, p.Age)
}

// Par référence : modifie l'original
func vieillirParPointeur(p *Personne) {
    p.Age++
    fmt.Printf("Dans la fonction: %s a %d ans\n", p.Nom, p.Age)
}

func main() {
    personne1 := Personne{Nom: "Alice", Age: 25}

    // Test par valeur
    fmt.Printf("Avant (par valeur): %s a %d ans\n", personne1.Nom, personne1.Age)
    vieillirParValeur(personne1)
    fmt.Printf("Après (par valeur): %s a %d ans\n", personne1.Nom, personne1.Age)

    fmt.Println("---")

    // Test par pointeur
    fmt.Printf("Avant (par pointeur): %s a %d ans\n", personne1.Nom, personne1.Age)
    vieillirParPointeur(&personne1)
    fmt.Printf("Après (par pointeur): %s a %d ans\n", personne1.Nom, personne1.Age)
}
```

**Résultat :**
```
Avant (par valeur): Alice a 25 ans
Dans la fonction: Alice a 26 ans
Après (par valeur): Alice a 25 ans
---
Avant (par pointeur): Alice a 25 ans
Dans la fonction: Alice a 26 ans
Après (par pointeur): Alice a 26 ans
```

## Valeurs de retour avancées

### Retours multiples et gestion d'erreurs

Le pattern le plus courant en Go est de retourner une valeur et une erreur :

```go
package main

import (
    "fmt"
    "errors"
)

func diviser(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division par zéro impossible")
    }
    return a / b, nil
}

func main() {
    // Cas normal
    resultat, err := diviser(10, 2)
    if err != nil {
        fmt.Println("Erreur:", err)
    } else {
        fmt.Printf("10 ÷ 2 = %.2f\n", resultat)
    }

    // Cas d'erreur
    resultat2, err2 := diviser(10, 0)
    if err2 != nil {
        fmt.Println("Erreur:", err2)
    } else {
        fmt.Printf("Résultat: %.2f\n", resultat2)
    }
}
```

**Résultat :**
```
10 ÷ 2 = 5.00
Erreur: division par zéro impossible
```

### Valeurs de retour nommées avec logique complexe

```go
package main

import "fmt"

func analyserNote(note int) (mention string, reussite bool, commentaire string) {
    // Initialisation par défaut
    reussite = false
    commentaire = "Analyse de la note"

    switch {
    case note >= 90:
        mention = "Excellent"
        reussite = true
        commentaire = "Félicitations, résultat exceptionnel !"
    case note >= 80:
        mention = "Très bien"
        reussite = true
        commentaire = "Très bon travail, continuez ainsi !"
    case note >= 70:
        mention = "Bien"
        reussite = true
        commentaire = "Bon travail, avec des efforts supplémentaires vous pouvez faire mieux."
    case note >= 60:
        mention = "Passable"
        reussite = true
        commentaire = "Résultat juste suffisant, il faut travailler davantage."
    default:
        mention = "Insuffisant"
        reussite = false
        commentaire = "Résultat insuffisant, reprise nécessaire."
    }

    return // Retour automatique des variables nommées
}

func main() {
    notes := []int{95, 85, 75, 65, 45}

    for _, note := range notes {
        mention, reussite, commentaire := analyserNote(note)
        fmt.Printf("Note: %d - %s (Réussite: %v)\n", note, mention, reussite)
        fmt.Printf("Commentaire: %s\n\n", commentaire)
    }
}
```

**Résultat :**
```
Note: 95 - Excellent (Réussite: true)
Commentaire: Félicitations, résultat exceptionnel !

Note: 85 - Très bien (Réussite: true)
Commentaire: Très bon travail, continuez ainsi !

Note: 75 - Bien (Réussite: true)
Commentaire: Bon travail, avec des efforts supplémentaires vous pouvez faire mieux.

Note: 65 - Passable (Réussite: true)
Commentaire: Résultat juste suffisant, il faut travailler davantage.

Note: 45 - Insuffisant (Réussite: false)
Commentaire: Résultat insuffisant, reprise nécessaire.
```

### Retourner des fonctions comme valeurs

En Go, les fonctions sont des "first-class citizens", ce qui signifie qu'on peut les retourner :

```go
package main

import "fmt"

// Fonction qui retourne une fonction
func creerOperateur(operation string) func(int, int) int {
    switch operation {
    case "+":
        return func(a, b int) int { return a + b }
    case "-":
        return func(a, b int) int { return a - b }
    case "*":
        return func(a, b int) int { return a * b }
    default:
        return func(a, b int) int { return 0 }
    }
}

func main() {
    // Créer différents opérateurs
    addition := creerOperateur("+")
    soustraction := creerOperateur("-")
    multiplication := creerOperateur("*")

    // Utiliser les fonctions retournées
    fmt.Println("5 + 3 =", addition(5, 3))
    fmt.Println("5 - 3 =", soustraction(5, 3))
    fmt.Println("5 * 3 =", multiplication(5, 3))
}
```

**Résultat :**
```
5 + 3 = 8
5 - 3 = 2
5 * 3 = 15
```

## Patterns courants avec paramètres et retours

### Pattern de validation avec retour booléen

```go
package main

import "fmt"

func validerEmail(email string) (bool, string) {
    if len(email) == 0 {
        return false, "L'email ne peut pas être vide"
    }
    if !contains(email, "@") {
        return false, "L'email doit contenir un @"
    }
    if !contains(email, ".") {
        return false, "L'email doit contenir un point"
    }
    return true, "Email valide"
}

// Fonction auxiliaire pour chercher un caractère
func contains(s, char string) bool {
    for _, c := range s {
        if string(c) == char {
            return true
        }
    }
    return false
}

func main() {
    emails := []string{
        "test@example.com",
        "invalide",
        "test@",
        "test.com",
        "",
    }

    for _, email := range emails {
        valide, message := validerEmail(email)
        fmt.Printf("'%s' -> %v (%s)\n", email, valide, message)
    }
}
```

**Résultat :**
```
'test@example.com' -> true (Email valide)
'invalide' -> false (L'email doit contenir un @)
'test@' -> false (L'email doit contenir un point)
'test.com' -> false (L'email doit contenir un @)
'' -> false (L'email ne peut pas être vide)
```

### Pattern de transformation avec chaînage

```go
package main

import (
    "fmt"
    "strings"
)

func nettoyer(texte string) string {
    // Supprime les espaces en début et fin
    return strings.TrimSpace(texte)
}

func formater(texte string) string {
    // Met en minuscules
    return strings.ToLower(texte)
}

func valider(texte string) (string, bool) {
    // Vérifie que le texte n'est pas vide
    if len(texte) == 0 {
        return "", false
    }
    return texte, true
}

// Fonction qui combine tout
func traiterTexte(texte string) (string, bool, string) {
    // Chaînage des transformations
    texte = nettoyer(texte)
    texte = formater(texte)
    texte, valide := valider(texte)

    var message string
    if valide {
        message = "Texte traité avec succès"
    } else {
        message = "Texte invalide après traitement"
    }

    return texte, valide, message
}

func main() {
    textes := []string{
        "  HELLO WORLD  ",
        "  Go Programming  ",
        "   ",
        "",
        "Test",
    }

    for _, texte := range textes {
        resultat, valide, message := traiterTexte(texte)
        fmt.Printf("'%s' -> '%s' (Valide: %v) - %s\n", texte, resultat, valide, message)
    }
}
```

**Résultat :**
```
'  HELLO WORLD  ' -> 'hello world' (Valide: true) - Texte traité avec succès
'  Go Programming  ' -> 'go programming' (Valide: true) - Texte traité avec succès
'   ' -> '' (Valide: false) - Texte invalide après traitement
'' -> '' (Valide: false) - Texte invalide après traitement
'Test' -> 'test' (Valide: true) - Texte traité avec succès
```

## Exercices pratiques

### Exercice 1 : Calculatrice avec gestion d'erreurs
Créez une fonction `calculer` qui prend deux nombres et une opération (string) et retourne le résultat et une erreur éventuelle.

<details>
<summary>Solution</summary>

```go
package main

import (
    "fmt"
    "errors"
)

func calculer(a, b float64, operation string) (float64, error) {
    switch operation {
    case "+":
        return a + b, nil
    case "-":
        return a - b, nil
    case "*":
        return a * b, nil
    case "/":
        if b == 0 {
            return 0, errors.New("division par zéro")
        }
        return a / b, nil
    default:
        return 0, errors.New("opération non supportée")
    }
}

func main() {
    operations := []struct {
        a, b float64
        op   string
    }{
        {10, 5, "+"},
        {10, 5, "-"},
        {10, 5, "*"},
        {10, 5, "/"},
        {10, 0, "/"},
        {10, 5, "%"},
    }

    for _, op := range operations {
        resultat, err := calculer(op.a, op.b, op.op)
        if err != nil {
            fmt.Printf("%.1f %s %.1f = Erreur: %v\n", op.a, op.op, op.b, err)
        } else {
            fmt.Printf("%.1f %s %.1f = %.2f\n", op.a, op.op, op.b, resultat)
        }
    }
}
```
</details>

### Exercice 2 : Modification de structure par référence
Créez une fonction qui modifie les propriétés d'une structure `Compte` (nom, solde) par référence.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

type Compte struct {
    Nom   string
    Solde float64
}

func deposer(compte *Compte, montant float64) bool {
    if montant <= 0 {
        return false
    }
    compte.Solde += montant
    return true
}

func retirer(compte *Compte, montant float64) bool {
    if montant <= 0 || montant > compte.Solde {
        return false
    }
    compte.Solde -= montant
    return true
}

func changerNom(compte *Compte, nouveauNom string) {
    if len(nouveauNom) > 0 {
        compte.Nom = nouveauNom
    }
}

func afficherCompte(compte Compte) {
    fmt.Printf("Compte: %s, Solde: %.2f€\n", compte.Nom, compte.Solde)
}

func main() {
    compte := Compte{Nom: "Alice", Solde: 100.0}

    fmt.Println("État initial:")
    afficherCompte(compte)

    // Dépôt
    if deposer(&compte, 50.0) {
        fmt.Println("Dépôt réussi")
    }
    afficherCompte(compte)

    // Retrait
    if retirer(&compte, 30.0) {
        fmt.Println("Retrait réussi")
    }
    afficherCompte(compte)

    // Changement de nom
    changerNom(&compte, "Alice Dupont")
    afficherCompte(compte)
}
```
</details>

### Exercice 3 : Analyse de slice avec statistiques
Créez une fonction qui analyse un slice d'entiers et retourne la moyenne, le minimum, le maximum et la somme.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func analyserNombres(nombres []int) (moyenne float64, min, max, somme int, valide bool) {
    if len(nombres) == 0 {
        return 0, 0, 0, 0, false
    }

    min = nombres[0]
    max = nombres[0]
    somme = 0

    for _, nombre := range nombres {
        somme += nombre
        if nombre < min {
            min = nombre
        }
        if nombre > max {
            max = nombre
        }
    }

    moyenne = float64(somme) / float64(len(nombres))
    valide = true
    return
}

func main() {
    testSlices := [][]int{
        {1, 2, 3, 4, 5},
        {10, -5, 0, 15, -2},
        {100},
        {},
        {-1, -2, -3, -4, -5},
    }

    for i, slice := range testSlices {
        fmt.Printf("Test %d: %v\n", i+1, slice)
        moyenne, min, max, somme, valide := analyserNombres(slice)

        if valide {
            fmt.Printf("  Moyenne: %.2f\n", moyenne)
            fmt.Printf("  Min: %d, Max: %d\n", min, max)
            fmt.Printf("  Somme: %d\n", somme)
        } else {
            fmt.Println("  Slice vide - analyse impossible")
        }
        fmt.Println()
    }
}
```
</details>

## Bonnes pratiques

### 1. Nommage des paramètres
```go
// ❌ Pas assez descriptif
func calculer(a, b float64, c string) (float64, error) { ... }

// ✅ Noms descriptifs
func calculer(nombre1, nombre2 float64, operation string) (float64, error) { ... }
```

### 2. Gestion des erreurs
```go
// ✅ Toujours retourner une erreur quand nécessaire
func diviser(dividende, diviseur float64) (float64, error) {
    if diviseur == 0 {
        return 0, errors.New("division par zéro")
    }
    return dividende / diviseur, nil
}
```

### 3. Utilisation des pointeurs
```go
// ✅ Utilisez des pointeurs pour les gros objets ou quand vous devez modifier
func traiterGrosObjet(obj *GrosObjet) { ... }

// ✅ Passez par valeur pour les petits objets immutables
func calculerAire(largeur, hauteur float64) float64 { ... }
```

### 4. Valeurs de retour cohérentes
```go
// ✅ Ordre cohérent : valeur, erreur
func lireFichier(nom string) ([]byte, error) { ... }

// ✅ Noms de retour pour les fonctions complexes
func analyserDonnees(data []int) (moyenne float64, ecartType float64, err error) { ... }
```

## Résumé

Dans cette section, nous avons exploré :

**Paramètres :**
- **Passage par valeur** : comportement par défaut, sûr mais copie les données
- **Passage par référence** : avec des pointeurs, permet de modifier l'original
- **Types complexes** : slices, maps et structs ont des comportements spécifiques

**Valeurs de retour :**
- **Retours multiples** : spécialité de Go, très utile pour la gestion d'erreurs
- **Retours nommés** : améliorent la lisibilité des fonctions complexes
- **Fonctions comme valeurs** : permettent des patterns avancés

**Patterns courants :**
- **Validation** : retour booléen avec message d'erreur
- **Transformation** : chaînage de fonctions
- **Gestion d'erreurs** : convention (valeur, erreur)

La maîtrise des paramètres et valeurs de retour est essentielle pour écrire du code Go idiomatique et robuste. Dans la section suivante, nous découvrirons les fonctions variadiques, qui permettent de créer des fonctions avec un nombre variable d'arguments.

⏭️
