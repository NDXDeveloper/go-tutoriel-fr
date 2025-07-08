🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4-1 : Déclaration et appel de fonctions

## Introduction

La déclaration et l'appel de fonctions sont les concepts les plus fondamentaux à maîtriser. Une fonction est comme une petite machine : vous lui donnez des ingrédients (paramètres), elle fait son travail, et vous récupérez le résultat. Dans cette section, nous allons apprendre à créer et utiliser ces "machines" en Go.

## Anatomie d'une fonction Go

Voici la structure générale d'une fonction en Go :

```go
func nomDeLaFonction(paramètre1 type1, paramètre2 type2) typeDeRetour {
    // Corps de la fonction
    return valeurDeRetour
}
```

Décomposons chaque élément :

- **`func`** : Le mot-clé qui indique que nous déclarons une fonction
- **`nomDeLaFonction`** : Le nom que nous donnons à notre fonction
- **`(paramètres)`** : Les données que la fonction peut recevoir (optionnel)
- **`typeDeRetour`** : Le type de valeur que la fonction renvoie (optionnel)
- **`{}`** : Le corps de la fonction, où le travail s'effectue
- **`return`** : Le mot-clé pour renvoyer une valeur (optionnel)

## Fonction la plus simple : sans paramètres ni retour

Commençons par l'exemple le plus simple possible :

```go
package main

import "fmt"

// Déclaration d'une fonction simple
func direBonjour() {
    fmt.Println("Bonjour le monde !")
}

func main() {
    direBonjour() // Appel de la fonction
}
```

**Explication :**
- `direBonjour()` est une fonction qui ne prend aucun paramètre et ne retourne rien
- Elle se contente d'afficher un message
- Dans `main()`, nous appelons la fonction en écrivant son nom suivi de parenthèses

**Résultat :**
```
Bonjour le monde !
```

## Fonction avec paramètres

Maintenant, créons une fonction qui peut recevoir des informations :

```go
package main

import "fmt"

// Fonction qui prend un paramètre
func direBonjour(nom string) {
    fmt.Println("Bonjour", nom, "!")
}

func main() {
    direBonjour("Alice")  // Appel avec un argument
    direBonjour("Bob")    // Appel avec un autre argument
}
```

**Explication :**
- `nom string` indique que la fonction attend un paramètre appelé `nom` de type `string`
- Lors de l'appel, nous passons une valeur concrète (appelée "argument")
- La fonction utilise cette valeur dans son traitement

**Résultat :**
```
Bonjour Alice !
Bonjour Bob !
```

## Fonction avec plusieurs paramètres

Une fonction peut recevoir plusieurs paramètres :

```go
package main

import "fmt"

// Fonction avec plusieurs paramètres
func presenter(nom string, age int, ville string) {
    fmt.Printf("Je m'appelle %s, j'ai %d ans et je viens de %s\n", nom, age, ville)
}

func main() {
    presenter("Marie", 25, "Paris")
    presenter("Pierre", 30, "Lyon")
}
```

**Explication :**
- Les paramètres sont séparés par des virgules
- Chaque paramètre a son propre type
- L'ordre des arguments lors de l'appel doit correspondre à l'ordre des paramètres

**Résultat :**
```
Je m'appelle Marie, j'ai 25 ans et je viens de Paris
Je m'appelle Pierre, j'ai 30 ans et je viens de Lyon
```

## Fonction avec valeur de retour

Les fonctions peuvent également renvoyer des valeurs :

```go
package main

import "fmt"

// Fonction qui calcule et retourne une valeur
func additionner(a int, b int) int {
    resultat := a + b
    return resultat
}

func main() {
    somme := additionner(5, 3)
    fmt.Println("5 + 3 =", somme)

    // On peut aussi utiliser directement le résultat
    fmt.Println("10 + 7 =", additionner(10, 7))
}
```

**Explication :**
- `int` après les paramètres indique que la fonction retourne un entier
- `return resultat` renvoie la valeur calculée
- Nous pouvons stocker le résultat dans une variable ou l'utiliser directement

**Résultat :**
```
5 + 3 = 8
10 + 7 = 17
```

## Syntaxe raccourcie pour les paramètres du même type

Quand plusieurs paramètres consécutifs ont le même type, on peut utiliser une syntaxe raccourcie :

```go
package main

import "fmt"

// Syntaxe longue
func multiplier1(a int, b int) int {
    return a * b
}

// Syntaxe raccourcie (équivalente)
func multiplier2(a, b int) int {
    return a * b
}

func main() {
    fmt.Println(multiplier1(4, 5))  // 20
    fmt.Println(multiplier2(6, 7))  // 42
}
```

## Fonction avec plusieurs valeurs de retour

Une spécificité de Go est la possibilité de retourner plusieurs valeurs :

```go
package main

import "fmt"

// Fonction qui retourne plusieurs valeurs
func calculerRectangle(longueur, largeur float64) (float64, float64) {
    aire := longueur * largeur
    perimetre := 2 * (longueur + largeur)
    return aire, perimetre
}

func main() {
    aire, perimetre := calculerRectangle(5.0, 3.0)
    fmt.Printf("Aire: %.2f, Périmètre: %.2f\n", aire, perimetre)
}
```

**Explication :**
- `(float64, float64)` indique que la fonction retourne deux valeurs de type `float64`
- `return aire, perimetre` retourne les deux valeurs
- `aire, perimetre := calculerRectangle(5.0, 3.0)` récupère les deux valeurs

**Résultat :**
```
Aire: 15.00, Périmètre: 16.00
```

## Ignorer des valeurs de retour

Si vous ne voulez pas utiliser toutes les valeurs retournées, utilisez `_` :

```go
package main

import "fmt"

func calculerRectangle(longueur, largeur float64) (float64, float64) {
    aire := longueur * largeur
    perimetre := 2 * (longueur + largeur)
    return aire, perimetre
}

func main() {
    // Je ne veux que l'aire, j'ignore le périmètre
    aire, _ := calculerRectangle(5.0, 3.0)
    fmt.Printf("Aire: %.2f\n", aire)
}
```

## Valeurs de retour nommées

Go permet de nommer les valeurs de retour, ce qui peut améliorer la lisibilité :

```go
package main

import "fmt"

// Valeurs de retour nommées
func calculerRectangle(longueur, largeur float64) (aire, perimetre float64) {
    aire = longueur * largeur
    perimetre = 2 * (longueur + largeur)
    return // return sans valeurs explicites
}

func main() {
    a, p := calculerRectangle(5.0, 3.0)
    fmt.Printf("Aire: %.2f, Périmètre: %.2f\n", a, p)
}
```

**Explication :**
- `(aire, perimetre float64)` nomme les valeurs de retour
- On peut assigner directement aux noms des valeurs de retour
- `return` sans valeurs explicites retourne automatiquement les variables nommées

## Ordre d'exécution et appels de fonctions

```go
package main

import "fmt"

func etape1() {
    fmt.Println("Étape 1 exécutée")
}

func etape2() {
    fmt.Println("Étape 2 exécutée")
}

func etape3() {
    fmt.Println("Étape 3 exécutée")
}

func main() {
    fmt.Println("Début du programme")
    etape1()
    etape2()
    etape3()
    fmt.Println("Fin du programme")
}
```

**Résultat :**
```
Début du programme
Étape 1 exécutée
Étape 2 exécutée
Étape 3 exécutée
Fin du programme
```

## Exercices pratiques

### Exercice 1 : Fonction de salutation personnalisée
Créez une fonction `saluer` qui prend un nom et un moment de la journée ("matin", "après-midi", "soir") et affiche un message approprié.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func saluer(nom, moment string) {
    var salutation string
    switch moment {
    case "matin":
        salutation = "Bonjour"
    case "après-midi":
        salutation = "Bon après-midi"
    case "soir":
        salutation = "Bonsoir"
    default:
        salutation = "Salut"
    }
    fmt.Printf("%s %s !\n", salutation, nom)
}

func main() {
    saluer("Alice", "matin")
    saluer("Bob", "soir")
    saluer("Charlie", "après-midi")
}
```
</details>

### Exercice 2 : Calculatrice simple
Créez une fonction `diviser` qui prend deux nombres et retourne le résultat de la division et le reste.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func diviser(dividende, diviseur int) (quotient, reste int) {
    quotient = dividende / diviseur
    reste = dividende % diviseur
    return
}

func main() {
    q, r := diviser(17, 5)
    fmt.Printf("17 ÷ 5 = %d reste %d\n", q, r)
}
```
</details>

### Exercice 3 : Fonction de validation
Créez une fonction `estMajeur` qui prend un âge et retourne `true` si la personne est majeure (18 ans ou plus), `false` sinon.

<details>
<summary>Solution</summary>

```go
package main

import "fmt"

func estMajeur(age int) bool {
    return age >= 18
}

func main() {
    ages := []int{15, 18, 21, 17}
    for _, age := range ages {
        if estMajeur(age) {
            fmt.Printf("Âge %d : Majeur\n", age)
        } else {
            fmt.Printf("Âge %d : Mineur\n", age)
        }
    }
}
```
</details>

## Erreurs courantes à éviter

### 1. Oublier les parenthèses lors de l'appel
```go
// ❌ Incorrect
direBonjour  // Ceci ne fait que référencer la fonction

// ✅ Correct
direBonjour()  // Ceci appelle la fonction
```

### 2. Mauvais ordre des arguments
```go
func presenter(nom string, age int) {
    fmt.Printf("Je suis %s et j'ai %d ans\n", nom, age)
}

// ❌ Incorrect
presenter(25, "Alice")  // L'ordre est inversé

// ✅ Correct
presenter("Alice", 25)
```

### 3. Oublier de retourner une valeur
```go
func additionner(a, b int) int {
    resultat := a + b
    // ❌ Oublier le return provoque une erreur de compilation
}

// ✅ Correct
func additionner(a, b int) int {
    resultat := a + b
    return resultat
}
```

## Résumé

Dans cette section, nous avons appris :

- **Syntaxe de base** : `func nom(paramètres) typeRetour { }`
- **Fonctions simples** : sans paramètres ni retour
- **Paramètres** : comment passer des données aux fonctions
- **Valeurs de retour** : comment récupérer des résultats
- **Retours multiples** : une spécificité puissante de Go
- **Valeurs nommées** : pour améliorer la lisibilité
- **Bonnes pratiques** : nommage, ordre, gestion des erreurs

Les fonctions sont la base de l'organisation du code en Go. Maîtriser leur déclaration et leur appel est essentiel pour écrire du code propre et réutilisable.

Dans la section suivante, nous explorerons plus en détail les paramètres et les valeurs de retour, notamment les techniques avancées pour les manipuler efficacement.

⏭️
