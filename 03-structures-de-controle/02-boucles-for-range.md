🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3-2 : Boucles (for, range)

## Introduction

Les boucles permettent de répéter des actions. Imaginez que vous devez dire "Bonjour" à 100 personnes : plutôt que d'écrire 100 fois la même instruction, vous utilisez une boucle !

En Go, il n'existe qu'**un seul type de boucle** : `for`. Mais ne vous inquiétez pas, elle est très polyvalente et peut remplacer tous les types de boucles que vous connaissez peut-être dans d'autres langages.

## 1. La boucle for classique

### Syntaxe de base

```go
for initialisation; condition; mise_à_jour {
    // Code à répéter
}
```

**Exemple simple :**
```go
package main

import "fmt"

func main() {
    // Compter de 1 à 5
    for i := 1; i <= 5; i++ {
        fmt.Printf("Compteur : %d\n", i)
    }
}
```

**Résultat :**
```
Compteur : 1
Compteur : 2
Compteur : 3
Compteur : 4
Compteur : 5
```

### Décomposition de la boucle

1. **Initialisation** (`i := 1`) : Exécutée une seule fois au début
2. **Condition** (`i <= 5`) : Vérifiée avant chaque itération
3. **Code de la boucle** : Exécuté si la condition est vraie
4. **Mise à jour** (`i++`) : Exécutée après chaque itération

## 2. Différentes formes de la boucle for

### Boucle infinie
```go
package main

import "fmt"

func main() {
    compteur := 0

    for {
        fmt.Printf("Itération %d\n", compteur)
        compteur++

        // Condition de sortie
        if compteur >= 3 {
            break
        }
    }
    fmt.Println("Boucle terminée")
}
```

### Boucle while (avec seulement une condition)
```go
package main

import "fmt"

func main() {
    i := 1

    // Équivalent d'un "while" dans d'autres langages
    for i <= 3 {
        fmt.Printf("i = %d\n", i)
        i++
    }
}
```

### Boucle sans initialisation
```go
package main

import "fmt"

func main() {
    i := 10

    // Pas d'initialisation, commence directement
    for ; i > 0; i-- {
        fmt.Printf("Décompte : %d\n", i)
    }
    fmt.Println("Décollage !")
}
```

## 3. Mots-clés de contrôle : break et continue

### break : Sortir de la boucle
```go
package main

import "fmt"

func main() {
    for i := 1; i <= 10; i++ {
        if i == 6 {
            fmt.Println("On s'arrête à 6")
            break
        }
        fmt.Printf("Nombre : %d\n", i)
    }
    fmt.Println("Fin du programme")
}
```

### continue : Passer à l'itération suivante
```go
package main

import "fmt"

func main() {
    fmt.Println("Nombres pairs de 1 à 10 :")

    for i := 1; i <= 10; i++ {
        // Si le nombre est impair, on passe au suivant
        if i%2 != 0 {
            continue
        }
        fmt.Printf("%d ", i)
    }
    fmt.Println()
}
```

## 4. La boucle for avec range

Le mot-clé `range` est très utile pour parcourir des collections (arrays, slices, maps, strings).

### Parcourir un slice
```go
package main

import "fmt"

func main() {
    fruits := []string{"pomme", "banane", "orange", "kiwi"}

    // Avec index et valeur
    for index, fruit := range fruits {
        fmt.Printf("Index %d : %s\n", index, fruit)
    }

    fmt.Println("---")

    // Seulement la valeur (ignorer l'index avec _)
    for _, fruit := range fruits {
        fmt.Printf("Fruit : %s\n", fruit)
    }

    fmt.Println("---")

    // Seulement l'index
    for index := range fruits {
        fmt.Printf("Index : %d\n", index)
    }
}
```

### Parcourir une string
```go
package main

import "fmt"

func main() {
    message := "Bonjour"

    // Parcourir caractère par caractère
    for index, caractere := range message {
        fmt.Printf("Position %d : %c\n", index, caractere)
    }
}
```

### Parcourir un array
```go
package main

import "fmt"

func main() {
    nombres := [5]int{10, 20, 30, 40, 50}

    fmt.Println("Contenu de l'array :")
    for i, nombre := range nombres {
        fmt.Printf("nombres[%d] = %d\n", i, nombre)
    }
}
```

## 5. Exemples pratiques

### Exemple 1 : Table de multiplication
```go
package main

import "fmt"

func main() {
    nombre := 7

    fmt.Printf("Table de multiplication de %d :\n", nombre)
    for i := 1; i <= 10; i++ {
        resultat := nombre * i
        fmt.Printf("%d x %d = %d\n", nombre, i, resultat)
    }
}
```

### Exemple 2 : Calculer une somme
```go
package main

import "fmt"

func main() {
    nombres := []int{10, 25, 30, 15, 20}
    somme := 0

    fmt.Println("Calcul de la somme :")
    for _, nombre := range nombres {
        somme += nombre
        fmt.Printf("Ajout de %d, somme actuelle : %d\n", nombre, somme)
    }

    fmt.Printf("Somme totale : %d\n", somme)
}
```

### Exemple 3 : Chercher un élément
```go
package main

import "fmt"

func main() {
    noms := []string{"Alice", "Bob", "Charlie", "Diana"}
    recherche := "Charlie"
    trouve := false
    position := -1

    for index, nom := range noms {
        if nom == recherche {
            trouve = true
            position = index
            break
        }
    }

    if trouve {
        fmt.Printf("'%s' trouvé à la position %d\n", recherche, position)
    } else {
        fmt.Printf("'%s' non trouvé\n", recherche)
    }
}
```

### Exemple 4 : Compter les voyelles
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    phrase := "Bonjour le monde"
    voyelles := "aeiouAEIOU"
    compteur := 0

    fmt.Printf("Phrase : %s\n", phrase)

    for _, caractere := range phrase {
        if strings.ContainsRune(voyelles, caractere) {
            compteur++
            fmt.Printf("Voyelle trouvée : %c\n", caractere)
        }
    }

    fmt.Printf("Nombre total de voyelles : %d\n", compteur)
}
```

## 6. Boucles imbriquées

Vous pouvez mettre une boucle à l'intérieur d'une autre boucle.

### Exemple : Table de multiplication complète
```go
package main

import "fmt"

func main() {
    fmt.Println("Tables de multiplication de 1 à 5 :")

    for i := 1; i <= 5; i++ {
        fmt.Printf("\nTable de %d :\n", i)
        for j := 1; j <= 5; j++ {
            fmt.Printf("%d x %d = %d\n", i, j, i*j)
        }
    }
}
```

### Exemple : Motif d'étoiles
```go
package main

import "fmt"

func main() {
    hauteur := 5

    fmt.Println("Pyramide d'étoiles :")
    for ligne := 1; ligne <= hauteur; ligne++ {
        // Imprimer les étoiles pour cette ligne
        for etoile := 1; etoile <= ligne; etoile++ {
            fmt.Print("* ")
        }
        fmt.Println() // Nouvelle ligne
    }
}
```

**Résultat :**
```
*
* *
* * *
* * * *
* * * * *
```

## 7. Range avec les maps

Quand nous verrons les maps plus tard, voici comment les parcourir :

```go
package main

import "fmt"

func main() {
    ages := map[string]int{
        "Alice": 25,
        "Bob":   30,
        "Charlie": 35,
    }

    fmt.Println("Ages des personnes :")
    for nom, age := range ages {
        fmt.Printf("%s a %d ans\n", nom, age)
    }
}
```

## 8. Erreurs courantes à éviter

### Erreur 1 : Modifier la variable de boucle dans range
```go
// ❌ Incorrect
fruits := []string{"pomme", "banane"}
for i, fruit := range fruits {
    i = 10 // N'affecte pas la boucle
    fmt.Println(i, fruit)
}

// ✅ Correct
for i := 0; i < len(fruits); i++ {
    // Ici on peut modifier i si nécessaire
    fmt.Println(i, fruits[i])
}
```

### Erreur 2 : Boucle infinie accidentelle
```go
// ❌ Attention : boucle infinie !
i := 0
for i < 5 {
    fmt.Println(i)
    // Oubli d'incrémenter i !
}

// ✅ Correct
i := 0
for i < 5 {
    fmt.Println(i)
    i++ // Ne pas oublier !
}
```

## 9. Bonnes pratiques

### 1. Noms de variables clairs
```go
// ✅ Bon
for index, utilisateur := range utilisateurs {
    fmt.Printf("Utilisateur %d : %s\n", index, utilisateur.nom)
}

// ❌ Moins bon
for i, u := range utilisateurs {
    fmt.Printf("Utilisateur %d : %s\n", i, u.nom)
}
```

### 2. Utiliser range quand c'est possible
```go
slice := []int{1, 2, 3, 4, 5}

// ✅ Préféré
for _, valeur := range slice {
    fmt.Println(valeur)
}

// ❌ Moins idiomatique
for i := 0; i < len(slice); i++ {
    fmt.Println(slice[i])
}
```

### 3. Éviter les boucles trop profondes
```go
// ❌ Difficile à lire
for i := 0; i < 10; i++ {
    for j := 0; j < 10; j++ {
        for k := 0; k < 10; k++ {
            // Trop d'imbrication
        }
    }
}

// ✅ Mieux : extraire en fonctions
func traiterLigne(i int) {
    for j := 0; j < 10; j++ {
        traiterColonne(i, j)
    }
}
```

## 10. Exercices pratiques

### Exercice 1 : Nombre premier
Créez un programme qui vérifie si un nombre est premier.

### Exercice 2 : Factorielle
Calculez la factorielle d'un nombre (n! = 1 × 2 × 3 × ... × n).

### Exercice 3 : Inverser une chaîne
Inversez une chaîne de caractères sans utiliser de fonctions prédéfinies.

### Exercice 4 : Maximum dans un slice
Trouvez le plus grand nombre dans un slice d'entiers.

### Exercice 5 : Triangle de Pascal
Générez les n premières lignes du triangle de Pascal.

## Solutions des Exercices - Boucles (for, range)

### Exercice 1 : Nombre premier

#### Solution simple
```go
package main

import "fmt"

func estPremier(n int) bool {
    // Les nombres <= 1 ne sont pas premiers
    if n <= 1 {
        return false
    }

    // 2 est le seul nombre pair premier
    if n == 2 {
        return true
    }

    // Les nombres pairs > 2 ne sont pas premiers
    if n%2 == 0 {
        return false
    }

    // Vérifier les diviseurs impairs jusqu'à √n
    for i := 3; i*i <= n; i += 2 {
        if n%i == 0 {
            return false
        }
    }

    return true
}

func main() {
    // Test avec différents nombres
    nombres := []int{1, 2, 3, 4, 5, 17, 25, 29, 100, 101}

    fmt.Println("Vérification des nombres premiers :")
    for _, nombre := range nombres {
        if estPremier(nombre) {
            fmt.Printf("%d est premier ✓\n", nombre)
        } else {
            fmt.Printf("%d n'est pas premier ✗\n", nombre)
        }
    }
}
```

#### Solution avec explications détaillées
```go
package main

import (
    "fmt"
    "math"
)

func estPremierDetaille(n int) (bool, string) {
    // Cas spéciaux
    if n <= 1 {
        return false, fmt.Sprintf("%d ≤ 1, donc pas premier", n)
    }

    if n == 2 {
        return true, "2 est le seul nombre pair premier"
    }

    if n%2 == 0 {
        return false, fmt.Sprintf("%d est pair (divisible par 2)", n)
    }

    // Vérification des diviseurs
    limite := int(math.Sqrt(float64(n)))
    fmt.Printf("Vérification des diviseurs de 3 à %d :\n", limite)

    for i := 3; i <= limite; i += 2 {
        fmt.Printf("  %d ÷ %d = ", n, i)
        if n%i == 0 {
            fmt.Printf("%d (divisible)\n", n/i)
            return false, fmt.Sprintf("%d est divisible par %d", n, i)
        }
        fmt.Printf("%.2f (non divisible)\n", float64(n)/float64(i))
    }

    return true, fmt.Sprintf("%d est premier", n)
}

func main() {
    var nombre int
    fmt.Print("Entrez un nombre à vérifier : ")
    fmt.Scanln(&nombre)

    premier, explication := estPremierDetaille(nombre)

    fmt.Println("\n" + "="*40)
    if premier {
        fmt.Printf("✓ %s\n", explication)
    } else {
        fmt.Printf("✗ %s\n", explication)
    }
}
```

#### Solution interactive avec historique
```go
package main

import "fmt"

func main() {
    fmt.Println("=== Détecteur de nombres premiers ===")
    fmt.Println("Entrez des nombres (0 pour quitter)")

    var premiers []int
    var nonPremiers []int

    for {
        var nombre int
        fmt.Print("\nNombre : ")
        fmt.Scanln(&nombre)

        if nombre == 0 {
            break
        }

        if estPremier(nombre) {
            fmt.Printf("✓ %d est premier\n", nombre)
            premiers = append(premiers, nombre)
        } else {
            fmt.Printf("✗ %d n'est pas premier\n", nombre)
            nonPremiers = append(nonPremiers, nombre)
        }
    }

    // Affichage du résumé
    fmt.Println("\n=== RÉSUMÉ ===")
    fmt.Printf("Nombres premiers trouvés (%d) : ", len(premiers))
    for _, p := range premiers {
        fmt.Printf("%d ", p)
    }
    fmt.Printf("\nNombres non premiers (%d) : ", len(nonPremiers))
    for _, np := range nonPremiers {
        fmt.Printf("%d ", np)
    }
    fmt.Println()
}

func estPremier(n int) bool {
    if n <= 1 {
        return false
    }
    if n <= 3 {
        return true
    }
    if n%2 == 0 || n%3 == 0 {
        return false
    }

    for i := 5; i*i <= n; i += 6 {
        if n%i == 0 || n%(i+2) == 0 {
            return false
        }
    }
    return true
}
```

### Exercice 2 : Factorielle

#### Solution basique
```go
package main

import "fmt"

func factorielle(n int) int {
    if n < 0 {
        return -1 // Erreur : factorielle non définie pour les négatifs
    }

    if n == 0 || n == 1 {
        return 1
    }

    resultat := 1
    for i := 2; i <= n; i++ {
        resultat *= i
    }

    return resultat
}

func main() {
    // Test avec différents nombres
    nombres := []int{0, 1, 5, 7, 10}

    fmt.Println("Calcul des factorielles :")
    for _, n := range nombres {
        result := factorielle(n)
        if result == -1 {
            fmt.Printf("%d! = Erreur (nombre négatif)\n", n)
        } else {
            fmt.Printf("%d! = %d\n", n, result)
        }
    }
}
```

#### Solution avec affichage du calcul
```go
package main

import (
    "fmt"
    "strings"
)

func factorielleDetaillee(n int) (int, string) {
    if n < 0 {
        return -1, "Erreur : la factorielle n'est pas définie pour les nombres négatifs"
    }

    if n == 0 || n == 1 {
        return 1, fmt.Sprintf("%d! = 1 (par définition)", n)
    }

    resultat := 1
    var calcul []string

    for i := 1; i <= n; i++ {
        calcul = append(calcul, fmt.Sprintf("%d", i))
        resultat *= i
    }

    details := fmt.Sprintf("%d! = %s = %d", n, strings.Join(calcul, " × "), resultat)
    return resultat, details
}

func main() {
    var nombre int
    fmt.Print("Entrez un nombre pour calculer sa factorielle : ")
    fmt.Scanln(&nombre)

    resultat, details := factorielleDetaillee(nombre)

    fmt.Println("\n" + "="*50)
    if resultat == -1 {
        fmt.Println(details)
    } else {
        fmt.Println(details)
        fmt.Printf("Résultat : %d\n", resultat)
    }
}
```

#### Solution avec gestion des grands nombres
```go
package main

import (
    "fmt"
    "math/big"
)

// Factorielle simple pour petits nombres
func factoriellePetite(n int) int {
    if n <= 1 {
        return 1
    }
    resultat := 1
    for i := 2; i <= n; i++ {
        resultat *= i
    }
    return resultat
}

// Factorielle pour grands nombres avec big.Int
func factorielleGrande(n int) *big.Int {
    if n < 0 {
        return nil
    }

    resultat := big.NewInt(1)
    for i := 2; i <= n; i++ {
        resultat.Mul(resultat, big.NewInt(int64(i)))
    }
    return resultat
}

func main() {
    fmt.Println("=== Calculateur de factorielles ===")

    // Petits nombres
    fmt.Println("\nFactorielles jusqu'à 10 :")
    for i := 0; i <= 10; i++ {
        fmt.Printf("%2d! = %d\n", i, factoriellePetite(i))
    }

    // Grands nombres
    fmt.Println("\nGrandes factorielles :")
    grandsNombres := []int{20, 50, 100}

    for _, n := range grandsNombres {
        resultat := factorielleGrande(n)
        if resultat != nil {
            fmt.Printf("%d! = %s\n", n, resultat.String())
            fmt.Printf("    (nombre de chiffres : %d)\n", len(resultat.String()))
        }
    }
}
```

### Exercice 3 : Inverser une chaîne

#### Solution simple
```go
package main

import "fmt"

func inverserChaine(s string) string {
    // Convertir en slice de runes pour gérer les caractères Unicode
    runes := []rune(s)
    longueur := len(runes)

    // Créer un slice pour le résultat
    resultat := make([]rune, longueur)

    // Copier en ordre inverse
    for i := 0; i < longueur; i++ {
        resultat[i] = runes[longueur-1-i]
    }

    return string(resultat)
}

func main() {
    // Test avec différentes chaînes
    chaines := []string{
        "hello",
        "Bonjour",
        "12345",
        "A man a plan a canal Panama",
        "racecar",
        "été", // Teste les accents
    }

    fmt.Println("Inversion de chaînes :")
    for _, chaine := range chaines {
        inverse := inverserChaine(chaine)
        fmt.Printf("'%s' → '%s'\n", chaine, inverse)
    }
}
```

#### Solution avec échange in-place
```go
package main

import "fmt"

func inverserChaineSurPlace(s string) string {
    runes := []rune(s)
    longueur := len(runes)

    // Échanger les éléments symétriques
    for i := 0; i < longueur/2; i++ {
        // Échanger runes[i] avec runes[longueur-1-i]
        runes[i], runes[longueur-1-i] = runes[longueur-1-i], runes[i]
    }

    return string(runes)
}

func inverserChainePasPar(s string, pas int) string {
    runes := []rune(s)
    longueur := len(runes)
    resultat := make([]rune, longueur)

    // Copier en ordre inverse avec un pas donné
    for i := 0; i < longueur; i += pas {
        for j := 0; j < pas && i+j < longueur; j++ {
            if longueur-1-i-j >= 0 {
                resultat[i+j] = runes[longueur-1-i-j]
            }
        }
    }

    return string(resultat)
}

func main() {
    phrase := "Hello World"

    fmt.Printf("Original : '%s'\n", phrase)
    fmt.Printf("Inversé : '%s'\n", inverserChaineSurPlace(phrase))

    // Test interactif
    var saisie string
    fmt.Print("\nEntrez une phrase à inverser : ")
    fmt.Scanln(&saisie)

    fmt.Printf("Résultat : '%s'\n", inverserChaineSurPlace(saisie))
}
```

#### Solution avec analyse détaillée
```go
package main

import "fmt"

func analyserEtInverser(s string) {
    fmt.Printf("\n=== Analyse de '%s' ===\n", s)

    runes := []rune(s)
    longueur := len(runes)

    fmt.Printf("Longueur : %d caractères\n", longueur)
    fmt.Println("Caractères (position → caractère) :")

    for i, r := range runes {
        fmt.Printf("  %d → '%c'\n", i, r)
    }

    // Processus d'inversion
    fmt.Println("\nProcessus d'inversion :")
    resultat := make([]rune, longueur)

    for i := 0; i < longueur; i++ {
        nouvellePosition := longueur - 1 - i
        resultat[nouvellePosition] = runes[i]
        fmt.Printf("  '%c' (pos %d) → pos %d\n", runes[i], i, nouvellePosition)
    }

    fmt.Printf("\nRésultat : '%s'\n", string(resultat))
}

func main() {
    exemples := []string{"abc", "Hello", "12345"}

    for _, exemple := range exemples {
        analyserEtInverser(exemple)
    }
}
```

### Exercice 4 : Maximum dans un slice

#### Solution basique
```go
package main

import "fmt"

func trouverMax(nombres []int) (int, int) {
    if len(nombres) == 0 {
        return 0, -1 // Valeur par défaut et index invalide
    }

    max := nombres[0]
    indexMax := 0

    for i, nombre := range nombres {
        if nombre > max {
            max = nombre
            indexMax = i
        }
    }

    return max, indexMax
}

func main() {
    // Test avec différents slices
    slices := [][]int{
        {3, 7, 2, 9, 1},
        {-5, -2, -8, -1},
        {42},
        {1, 2, 3, 4, 5},
        {},
    }

    fmt.Println("Recherche du maximum :")
    for i, slice := range slices {
        fmt.Printf("\nSlice %d : %v\n", i+1, slice)

        if len(slice) == 0 {
            fmt.Println("Slice vide !")
            continue
        }

        max, index := trouverMax(slice)
        fmt.Printf("Maximum : %d à l'index %d\n", max, index)
    }
}
```

#### Solution avec statistiques complètes
```go
package main

import "fmt"

type Statistiques struct {
    Max        int
    Min        int
    IndexMax   int
    IndexMin   int
    Somme      int
    Moyenne    float64
    NbElements int
}

func analyserSlice(nombres []int) Statistiques {
    if len(nombres) == 0 {
        return Statistiques{}
    }

    stats := Statistiques{
        Max:        nombres[0],
        Min:        nombres[0],
        IndexMax:   0,
        IndexMin:   0,
        NbElements: len(nombres),
    }

    for i, nombre := range nombres {
        stats.Somme += nombre

        if nombre > stats.Max {
            stats.Max = nombre
            stats.IndexMax = i
        }

        if nombre < stats.Min {
            stats.Min = nombre
            stats.IndexMin = i
        }
    }

    stats.Moyenne = float64(stats.Somme) / float64(stats.NbElements)
    return stats
}

func main() {
    nombres := []int{10, 3, 87, 22, 91, 7, 65, 4}

    fmt.Printf("Slice analysé : %v\n", nombres)
    fmt.Println("="*40)

    stats := analyserSlice(nombres)

    fmt.Printf("Nombre d'éléments : %d\n", stats.NbElements)
    fmt.Printf("Maximum : %d (index %d)\n", stats.Max, stats.IndexMax)
    fmt.Printf("Minimum : %d (index %d)\n", stats.Min, stats.IndexMin)
    fmt.Printf("Somme : %d\n", stats.Somme)
    fmt.Printf("Moyenne : %.2f\n", stats.Moyenne)

    // Visualisation
    fmt.Println("\nVisualisation :")
    for i, nombre := range nombres {
        marqueur := " "
        if i == stats.IndexMax {
            marqueur = "↑MAX"
        } else if i == stats.IndexMin {
            marqueur = "↓MIN"
        }
        fmt.Printf("%2d: %3d %s\n", i, nombre, marqueur)
    }
}
```

#### Solution interactive avec recherche personnalisée
```go
package main

import "fmt"

func main() {
    fmt.Println("=== Analyseur de slice ===")

    // Saisie des nombres
    var nombres []int
    fmt.Println("Entrez des nombres (0 pour terminer) :")

    for {
        var nombre int
        fmt.Print("Nombre : ")
        fmt.Scanln(&nombre)

        if nombre == 0 {
            break
        }

        nombres = append(nombres, nombre)
    }

    if len(nombres) == 0 {
        fmt.Println("Aucun nombre saisi.")
        return
    }

    fmt.Printf("\nVotre slice : %v\n", nombres)

    // Menu d'options
    for {
        fmt.Println("\nQue voulez-vous faire ?")
        fmt.Println("1. Trouver le maximum")
        fmt.Println("2. Trouver le minimum")
        fmt.Println("3. Afficher tous les maximums (en cas d'égalité)")
        fmt.Println("4. Quitter")

        var choix int
        fmt.Print("Choix : ")
        fmt.Scanln(&choix)

        switch choix {
        case 1:
            max, index := trouverMax(nombres)
            fmt.Printf("Maximum : %d à l'index %d\n", max, index)

        case 2:
            min, index := trouverMin(nombres)
            fmt.Printf("Minimum : %d à l'index %d\n", min, index)

        case 3:
            max, indices := trouverTousLesMax(nombres)
            fmt.Printf("Maximum : %d\n", max)
            fmt.Printf("Trouvé aux index : %v\n", indices)

        case 4:
            fmt.Println("Au revoir !")
            return

        default:
            fmt.Println("Choix invalide.")
        }
    }
}

func trouverMax(nombres []int) (int, int) {
    max, index := nombres[0], 0
    for i, n := range nombres {
        if n > max {
            max, index = n, i
        }
    }
    return max, index
}

func trouverMin(nombres []int) (int, int) {
    min, index := nombres[0], 0
    for i, n := range nombres {
        if n < min {
            min, index = n, i
        }
    }
    return min, index
}

func trouverTousLesMax(nombres []int) (int, []int) {
    max := nombres[0]
    var indices []int

    // Première passe : trouver la valeur max
    for _, n := range nombres {
        if n > max {
            max = n
        }
    }

    // Deuxième passe : trouver tous les indices
    for i, n := range nombres {
        if n == max {
            indices = append(indices, i)
        }
    }

    return max, indices
}
```

### Exercice 5 : Triangle de Pascal

#### Solution basique
```go
package main

import "fmt"

func genererTrianglePascal(n int) [][]int {
    triangle := make([][]int, n)

    for i := 0; i < n; i++ {
        // Chaque ligne a i+1 éléments
        triangle[i] = make([]int, i+1)

        // Premier et dernier élément de chaque ligne = 1
        triangle[i][0] = 1
        triangle[i][i] = 1

        // Éléments du milieu = somme des deux éléments au-dessus
        for j := 1; j < i; j++ {
            triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j]
        }
    }

    return triangle
}

func afficherTriangle(triangle [][]int) {
    n := len(triangle)

    for i, ligne := range triangle {
        // Espacement pour centrer
        for k := 0; k < n-i-1; k++ {
            fmt.Print("  ")
        }

        // Afficher les nombres
        for _, nombre := range ligne {
            fmt.Printf("%4d", nombre)
        }
        fmt.Println()
    }
}

func main() {
    var n int
    fmt.Print("Entrez le nombre de lignes : ")
    fmt.Scanln(&n)

    if n <= 0 {
        fmt.Println("Le nombre doit être positif.")
        return
    }

    fmt.Printf("\nTriangle de Pascal (%d lignes) :\n\n", n)
    triangle := genererTrianglePascal(n)
    afficherTriangle(triangle)
}
```

#### Solution avec explications étape par étape
```go
package main

import "fmt"

func genererTriangleAvecDetails(n int) {
    fmt.Printf("Construction du triangle de Pascal (%d lignes) :\n\n", n)

    triangle := make([][]int, n)

    for i := 0; i < n; i++ {
        fmt.Printf("--- Ligne %d ---\n", i+1)
        triangle[i] = make([]int, i+1)

        // Premier élément
        triangle[i][0] = 1
        fmt.Printf("Position 0 : 1 (toujours 1)\n")

        // Éléments du milieu
        for j := 1; j < i; j++ {
            triangle[i][j] = triangle[i-1][j-1] + triangle[i-1][j]
            fmt.Printf("Position %d : %d + %d = %d\n",
                j, triangle[i-1][j-1], triangle[i-1][j], triangle[i][j])
        }

        // Dernier élément
        if i > 0 {
            triangle[i][i] = 1
            fmt.Printf("Position %d : 1 (toujours 1)\n", i)
        }

        // Afficher la ligne complète
        fmt.Printf("Ligne complète : ")
        for _, val := range triangle[i] {
            fmt.Printf("%d ", val)
        }
        fmt.Println("\n")
    }

    fmt.Println("Triangle final :")
    afficherTriangleFormate(triangle)
}

func afficherTriangleFormate(triangle [][]int) {
    n := len(triangle)

    for i, ligne := range triangle {
        // Espacement pour centrer
        for k := 0; k < n-i-1; k++ {
            fmt.Print("   ")
        }

        // Afficher les nombres avec espacement
        for j, nombre := range ligne {
            if j > 0 {
                fmt.Print("   ")
            }
            fmt.Printf("%3d", nombre)
        }
        fmt.Println()
    }
}

func main() {
    genererTriangleAvecDetails(6)
}
```

#### Solution avancée avec propriétés mathématiques
```go
package main

import "fmt"

// Calcul direct d'un coefficient binomial C(n,k)
func coefficientBinomial(n, k int) int {
    if k > n || k < 0 {
        return 0
    }
    if k == 0 || k == n {
        return 1
    }

    // Optimisation : C(n,k) = C(n,n-k)
    if k > n-k {
        k = n - k
    }

    resultat := 1
    for i := 0; i < k; i++ {
        resultat = resultat * (n - i) / (i + 1)
    }

    return resultat
}

func afficherTriangleComplet(n int) {
    fmt.Printf("Triangle de Pascal (%d lignes) :\n\n", n)

    // Calcul de la largeur maximale pour l'alignement
    maxVal := coefficientBinomial(n-1, (n-1)/2)
    largeur := len(fmt.Sprintf("%d", maxVal)) + 1

    for i := 0; i < n; i++ {
        // Espacement pour centrer
        espaces := (n - i - 1) * largeur / 2
        for k := 0; k < espaces; k++ {
            fmt.Print(" ")
        }

        // Afficher les coefficients
        for j := 0; j <= i; j++ {
            coeff := coefficientBinomial(i, j)
            if j > 0 {
                fmt.Print(" ")
            }
            fmt.Printf("%*d", largeur, coeff)
        }
        fmt.Println()
    }
}

func analyserProprietes(n int) {
    fmt.Printf("\n=== Propriétés du triangle (%d lignes) ===\n", n)

    for i := 0; i < n; i++ {
        somme := 0
        fmt.Printf("Ligne %d : ", i)

        for j := 0; j <= i; j++ {
            coeff := coefficientBinomial(i, j)
            fmt.Printf("%d ", coeff)
            somme += coeff
        }

        fmt.Printf("(somme = %d = 2^%d)\n", somme, i)
    }

    // Diagonales
    fmt.Println("\nDiagonales importantes :")
    fmt.Print("Nombres naturels (2ème diagonale) : ")
    for i := 1; i < n; i++ {
        fmt.Printf("%d ", coefficientBinomial(i, 1))
    }

    fmt.Print("\nNombres triangulaires (3ème diagonale) : ")
    for i := 2; i < n; i++ {
        fmt.Printf("%d ", coefficientBinomial(i, 2))
    }
    fmt.Println()
}

func main() {
    fmt.Println("=== Générateur de Triangle de Pascal ===")

    var n int
    fmt.Print("Nombre de lignes : ")
    fmt.Scanln(&n)

    if n <= 0 {
        fmt.Println("Le nombre doit être positif.")
        return
    }

    afficherTriangleComplet(n)
    analyserProprietes(n)

    // Test de coefficient spécifique
    fmt.Print("\nCalculer C(n,k) - Entrez n : ")
    var nTest, kTest int
    fmt.Scanln(&nTest)
    fmt.Print("Entrez k : ")
    fmt.Scanln(&kTest)

    resultat := coefficientBinomial(nTest, kTest)
    fmt.Printf("C(%d,%d) = %d\n", nTest, kTest, resultat)
}
```

### Points clés des solutions

#### Exercice 1 - Nombre premier
- **Optimisation** : Vérification seulement jusqu'à √n
- **Cas spéciaux** : 2 est le seul pair premier
- **Interface** : Version interactive avec historique

#### Exercice 2 - Factorielle
- **Gestion d'erreurs** : Nombres négatifs
- **Grands nombres** : Utilisation de big.Int
- **Visualisation** : Affichage du calcul complet

#### Exercice 3 - Inverser chaîne
- **Unicode** : Utilisation de []rune pour les accents
- **Efficacité** : Échange in-place
- **Analyse** : Processus étape par étape

#### Exercice 4 - Maximum
- **Robustesse** : Gestion des slices vides
- **Statistiques** : Analyse complète (min, max, moyenne)
- **Interactivité** : Menu avec options multiples

#### Exercice 5 - Triangle Pascal
- **Algorithme** : Construction ligne par ligne
- **Mathématiques** : Coefficients binomiaux directs
- **Visualisation** : Formatage centré et propriétés


## 11. Comparaison avec d'autres langages

Si vous connaissez d'autres langages de programmation :

| Autre langage | Go équivalent |
|---------------|---------------|
| `while (condition)` | `for condition` |
| `do...while` | `for` avec `break` conditionnel |
| `foreach (item in items)` | `for _, item := range items` |
| `for (int i=0; i<10; i++)` | `for i := 0; i < 10; i++` |

## 12. Cas d'usage avancés

### Parcourir avec un pas différent de 1
```go
package main

import "fmt"

func main() {
    // Compter de 2 en 2
    for i := 0; i <= 10; i += 2 {
        fmt.Printf("%d ", i)
    }
    fmt.Println()

    // Compter à l'envers
    for i := 10; i >= 0; i -= 2 {
        fmt.Printf("%d ", i)
    }
    fmt.Println()
}
```

### Boucle avec plusieurs conditions
```go
package main

import "fmt"

func main() {
    x, y := 1, 1

    for x < 10 && y < 20 {
        fmt.Printf("x=%d, y=%d\n", x, y)
        x += 2
        y += 3
    }
}
```

## Résumé

Les boucles `for` en Go sont simples mais puissantes :

- **Une seule syntaxe** pour tous les types de boucles
- **Range** pour parcourir facilement les collections
- **Break** et **continue** pour contrôler le flux
- **Syntaxe claire** sans parenthèses obligatoires
- **Polyvalence** : peut simuler while, do-while, foreach

**Points clés à retenir :**
- `for` est la seule boucle en Go
- `range` simplifie le parcours des collections
- Attention aux boucles infinies
- Privilégier la clarté du code

Dans la prochaine section, nous verrons comment gérer les erreurs en Go !

⏭️
