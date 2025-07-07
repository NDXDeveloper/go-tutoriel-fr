🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2-3 : Opérateurs arithmétiques et logiques

## Introduction

Les opérateurs sont les **outils** qui permettent de manipuler et combiner vos données. Imaginez-les comme des outils dans une boîte à outils : la calculatrice pour les calculs, la loupe pour les comparaisons, et les connecteurs logiques pour assembler les conditions.

## Qu'est-ce qu'un opérateur ?

### Définition simple

Un **opérateur** est un symbole qui indique au programme quelle opération effectuer sur une ou plusieurs valeurs (appelées **opérandes**).

### Structure de base

```
opérande1  opérateur  opérande2  =  résultat
    5         +           3      =      8
   true      &&         false    =    false
```

## Opérateurs arithmétiques

### Les opérateurs de base

| Opérateur | Nom | Description | Exemple | Résultat |
|-----------|-----|-------------|---------|----------|
| `+` | Addition | Additionne deux nombres | `5 + 3` | `8` |
| `-` | Soustraction | Soustrait le second du premier | `5 - 3` | `2` |
| `*` | Multiplication | Multiplie deux nombres | `5 * 3` | `15` |
| `/` | Division | Divise le premier par le second | `15 / 3` | `5` |
| `%` | Modulo (reste) | Reste de la division entière | `5 % 3` | `2` |

### Exemples pratiques de base

```go
package main

import "fmt"

func main() {
    // Variables pour les calculs
    a := 10
    b := 3

    // Opérations arithmétiques de base
    addition := a + b
    soustraction := a - b
    multiplication := a * b
    division := a / b          // Division entière !
    reste := a % b

    fmt.Printf("%d + %d = %d\n", a, b, addition)        // 10 + 3 = 13
    fmt.Printf("%d - %d = %d\n", a, b, soustraction)    // 10 - 3 = 7
    fmt.Printf("%d * %d = %d\n", a, b, multiplication)  // 10 * 3 = 30
    fmt.Printf("%d / %d = %d\n", a, b, division)        // 10 / 3 = 3 (division entière)
    fmt.Printf("%d %% %d = %d\n", a, b, reste)          // 10 % 3 = 1
}
```

### Division : entière vs décimale

**Important :** Le type des opérandes détermine le type du résultat !

```go
package main

import "fmt"

func main() {
    // Division entière (int / int = int)
    var a int = 10
    var b int = 3
    resultaEntier := a / b
    fmt.Printf("Division entière: %d / %d = %d\n", a, b, resultaEntier)  // 3

    // Division décimale (float64 / float64 = float64)
    var x float64 = 10.0
    var y float64 = 3.0
    resultatDecimal := x / y
    fmt.Printf("Division décimale: %.1f / %.1f = %.3f\n", x, y, resultatDecimal)  // 3.333

    // Conversion pour obtenir un résultat décimal
    resultatConverti := float64(a) / float64(b)
    fmt.Printf("Avec conversion: %d / %d = %.3f\n", a, b, resultatConverti)  // 3.333
}
```

### L'opérateur modulo (%)

Le modulo retourne le **reste** d'une division entière. Très utile !

```go
package main

import "fmt"

func main() {
    // Exemples de modulo
    fmt.Printf("5 %% 2 = %d\n", 5%2)    // 1 (5 = 2*2 + 1)
    fmt.Printf("8 %% 3 = %d\n", 8%3)    // 2 (8 = 3*2 + 2)
    fmt.Printf("10 %% 5 = %d\n", 10%5)  // 0 (10 = 5*2 + 0)

    // Applications pratiques
    nombre := 17

    // Vérifier si un nombre est pair ou impair
    if nombre%2 == 0 {
        fmt.Printf("%d est pair\n", nombre)
    } else {
        fmt.Printf("%d est impair\n", nombre)
    }

    // Créer un cycle (ex: jours de la semaine)
    for i := 0; i < 10; i++ {
        jourSemaine := i % 7
        fmt.Printf("Jour %d -> Jour de la semaine %d\n", i, jourSemaine)
    }
}
```

### Opérateurs d'assignation combinés

**Raccourcis pour modifier une variable :**

| Opérateur | Équivalent | Description |
|-----------|------------|-------------|
| `+=` | `a = a + b` | Addition et assignation |
| `-=` | `a = a - b` | Soustraction et assignation |
| `*=` | `a = a * b` | Multiplication et assignation |
| `/=` | `a = a / b` | Division et assignation |
| `%=` | `a = a % b` | Modulo et assignation |

```go
package main

import "fmt"

func main() {
    score := 100

    fmt.Printf("Score initial: %d\n", score)

    // Ajouter des points
    score += 50  // Équivalent à: score = score + 50
    fmt.Printf("Après bonus: %d\n", score)

    // Perdre des points
    score -= 20  // Équivalent à: score = score - 20
    fmt.Printf("Après pénalité: %d\n", score)

    // Doubler le score
    score *= 2   // Équivalent à: score = score * 2
    fmt.Printf("Score doublé: %d\n", score)

    // Diviser par 3
    score /= 3   // Équivalent à: score = score / 3
    fmt.Printf("Score final: %d\n", score)
}
```

### Opérateurs d'incrémentation et décrémentation

```go
package main

import "fmt"

func main() {
    compteur := 5

    fmt.Printf("Compteur initial: %d\n", compteur)

    // Incrémenter (augmenter de 1)
    compteur++  // Équivalent à: compteur = compteur + 1
    fmt.Printf("Après compteur++: %d\n", compteur)

    // Décrémenter (diminuer de 1)
    compteur--  // Équivalent à: compteur = compteur - 1
    fmt.Printf("Après compteur--: %d\n", compteur)

    // Note: En Go, ++ et -- sont des instructions, pas des expressions
    // Donc ceci ne fonctionne PAS: result := compteur++
    // Il faut faire:
    compteur++
    result := compteur
    fmt.Printf("Résultat: %d\n", result)
}
```

## Opérateurs de comparaison

### Les opérateurs disponibles

| Opérateur | Nom | Description | Exemple | Résultat |
|-----------|-----|-------------|---------|----------|
| `==` | Égal à | Vérifie l'égalité | `5 == 5` | `true` |
| `!=` | Différent de | Vérifie l'inégalité | `5 != 3` | `true` |
| `<` | Inférieur à | Strictement plus petit | `3 < 5` | `true` |
| `<=` | Inférieur ou égal | Plus petit ou égal | `5 <= 5` | `true` |
| `>` | Supérieur à | Strictement plus grand | `5 > 3` | `true` |
| `>=` | Supérieur ou égal | Plus grand ou égal | `3 >= 5` | `false` |

### Exemples pratiques

```go
package main

import "fmt"

func main() {
    age := 20
    ageMinimum := 18
    note := 15.5
    notePasse := 10.0

    // Comparaisons numériques
    estMajeur := age >= ageMinimum
    aReussi := note >= notePasse
    estExcellent := note >= 18.0

    fmt.Printf("Age: %d, Age minimum: %d\n", age, ageMinimum)
    fmt.Printf("Est majeur: %t\n", estMajeur)
    fmt.Printf("Note: %.1f, Note de passage: %.1f\n", note, notePasse)
    fmt.Printf("A réussi: %t\n", aReussi)
    fmt.Printf("Est excellent: %t\n", estExcellent)

    // Comparaisons de chaînes
    nom1 := "Alice"
    nom2 := "Alice"
    nom3 := "Bob"

    fmt.Printf("'%s' == '%s': %t\n", nom1, nom2, nom1 == nom2)  // true
    fmt.Printf("'%s' == '%s': %t\n", nom1, nom3, nom1 == nom3)  // false
    fmt.Printf("'%s' < '%s': %t\n", nom1, nom3, nom1 < nom3)    // true (ordre alphabétique)
}
```

### Comparaisons de chaînes

```go
package main

import "fmt"

func main() {
    // Comparaisons alphabétiques
    fmt.Printf("'a' < 'b': %t\n", "a" < "b")        // true
    fmt.Printf("'Apple' < 'Banana': %t\n", "Apple" < "Banana")  // true
    fmt.Printf("'abc' == 'abc': %t\n", "abc" == "abc")          // true

    // Attention à la casse !
    fmt.Printf("'Alice' == 'alice': %t\n", "Alice" == "alice")  // false

    // Comparaison de longueur
    texte := "Bonjour"
    estCourt := len(texte) < 10
    fmt.Printf("'%s' est court (< 10 caractères): %t\n", texte, estCourt)
}
```

## Opérateurs logiques

### Les trois opérateurs principaux

| Opérateur | Nom | Description | Exemple | Résultat |
|-----------|-----|-------------|---------|----------|
| `&&` | ET logique (AND) | Vrai si TOUS les opérandes sont vrais | `true && true` | `true` |
| `\|\|` | OU logique (OR) | Vrai si AU MOINS UN opérande est vrai | `true \|\| false` | `true` |
| `!` | NON logique (NOT) | Inverse la valeur booléenne | `!true` | `false` |

### Tables de vérité

**ET logique (&&) :**
| A | B | A && B |
|---|---|--------|
| `true` | `true` | `true` |
| `true` | `false` | `false` |
| `false` | `true` | `false` |
| `false` | `false` | `false` |

**OU logique (||) :**
| A | B | A \|\| B |
|---|---|----------|
| `true` | `true` | `true` |
| `true` | `false` | `true` |
| `false` | `true` | `true` |
| `false` | `false` | `false` |

**NON logique (!) :**
| A | !A |
|---|-----|
| `true` | `false` |
| `false` | `true` |

### Exemples pratiques

```go
package main

import "fmt"

func main() {
    age := 20
    aPermis := true
    aVoiture := false
    soldeCompte := 1500.0

    // ET logique - toutes les conditions doivent être vraies
    peutConduire := age >= 18 && aPermis
    peutVoyager := peutConduire && aVoiture

    // OU logique - au moins une condition doit être vraie
    aArgent := soldeCompte > 1000 || age < 25  // Compte fourni ou tarif jeune

    // NON logique - inverse la condition
    estMineur := !(age >= 18)  // Équivalent à: age < 18

    fmt.Printf("Age: %d, A permis: %t, A voiture: %t\n", age, aPermis, aVoiture)
    fmt.Printf("Peut conduire: %t\n", peutConduire)
    fmt.Printf("Peut voyager: %t\n", peutVoyager)
    fmt.Printf("A de l'argent: %t\n", aArgent)
    fmt.Printf("Est mineur: %t\n", estMineur)
}
```

### Évaluation paresseuse (Short-circuit)

**Important :** Go utilise l'évaluation paresseuse pour les opérateurs logiques.

```go
package main

import "fmt"

func main() {
    a := 5
    b := 0

    // Avec &&: si la première condition est false,
    // la seconde n'est pas évaluée
    if b != 0 && a/b > 2 {  // Sûr: division par zéro évitée
        fmt.Println("Division OK")
    } else {
        fmt.Println("Division évitée ou résultat < 2")
    }

    // Avec ||: si la première condition est true,
    // la seconde n'est pas évaluée
    if b == 0 || a/b < 10 {  // Sûr: si b == 0, pas de division
        fmt.Println("Condition remplie")
    }
}
```

### Combinaisons complexes

```go
package main

import "fmt"

func main() {
    age := 25
    experience := 3
    diplome := true
    salaireDemande := 45000

    // Condition complexe pour un recrutement
    estQualifie := (age >= 21 && experience >= 2) || (diplome && age >= 23)
    salaireAcceptable := salaireDemande <= 50000
    estRetenu := estQualifie && salaireAcceptable

    fmt.Printf("Candidat: Age %d, Expérience %d ans, Diplôme %t\n",
               age, experience, diplome)
    fmt.Printf("Salaire demandé: %d€\n", salaireDemande)
    fmt.Printf("Est qualifié: %t\n", estQualifie)
    fmt.Printf("Salaire acceptable: %t\n", salaireAcceptable)
    fmt.Printf("Est retenu: %t\n", estRetenu)
}
```

## Priorité des opérateurs

### Ordre de priorité (du plus prioritaire au moins prioritaire)

1. **Parenthèses** `()`
2. **Unaires** `!`, `+`, `-`
3. **Multiplicatifs** `*`, `/`, `%`
4. **Additifs** `+`, `-`
5. **Comparaisons** `<`, `<=`, `>`, `>=`
6. **Égalité** `==`, `!=`
7. **ET logique** `&&`
8. **OU logique** `||`

### Exemples avec priorité

```go
package main

import "fmt"

func main() {
    // Sans parenthèses - ordre de priorité naturel
    resultat1 := 2 + 3 * 4        // 2 + (3 * 4) = 2 + 12 = 14

    // Avec parenthèses - forcer l'ordre
    resultat2 := (2 + 3) * 4      // (2 + 3) * 4 = 5 * 4 = 20

    // Comparaisons et logique
    x, y, z := 5, 10, 15
    condition1 := x < y && y < z   // (x < y) && (y < z) = true && true = true
    condition2 := x + y > z || z > 20  // ((x + y) > z) || (z > 20) = false || false = false

    fmt.Printf("2 + 3 * 4 = %d\n", resultat1)
    fmt.Printf("(2 + 3) * 4 = %d\n", resultat2)
    fmt.Printf("%d < %d && %d < %d = %t\n", x, y, y, z, condition1)
    fmt.Printf("%d + %d > %d || %d > 20 = %t\n", x, y, z, z, condition2)
}
```

### Bonnes pratiques avec les parenthèses

```go
package main

import "fmt"

func main() {
    age := 25
    experience := 3
    diplome := true

    // ❌ Difficile à lire
    eligible := age >= 18 && experience >= 2 || diplome && age >= 21

    // ✅ Clair avec parenthèses
    eligibleClair := (age >= 18 && experience >= 2) || (diplome && age >= 21)

    fmt.Printf("Eligible (sans parenthèses): %t\n", eligible)
    fmt.Printf("Eligible (avec parenthèses): %t\n", eligibleClair)

    // Les deux donnent le même résultat, mais le second est plus lisible !
}
```

## Opérateurs sur les bits (bonus)

### Opérateurs bit à bit

| Opérateur | Nom | Description | Exemple |
|-----------|-----|-------------|---------|
| `&` | ET bit à bit | ET sur chaque bit | `5 & 3 = 1` |
| `\|` | OU bit à bit | OU sur chaque bit | `5 \| 3 = 7` |
| `^` | OU exclusif (XOR) | XOR sur chaque bit | `5 ^ 3 = 6` |
| `<<` | Décalage à gauche | Décale les bits vers la gauche | `5 << 1 = 10` |
| `>>` | Décalage à droite | Décale les bits vers la droite | `5 >> 1 = 2` |

### Exemple simple

```go
package main

import "fmt"

func main() {
    a := 5  // 101 en binaire
    b := 3  // 011 en binaire

    fmt.Printf("a = %d (%08b)\n", a, a)
    fmt.Printf("b = %d (%08b)\n", b, b)
    fmt.Printf("a & b = %d (%08b)\n", a&b, a&b)  // ET: 001 = 1
    fmt.Printf("a | b = %d (%08b)\n", a|b, a|b)  // OU: 111 = 7
    fmt.Printf("a ^ b = %d (%08b)\n", a^b, a^b)  // XOR: 110 = 6

    // Décalages
    fmt.Printf("a << 1 = %d (%08b)\n", a<<1, a<<1)  // 1010 = 10
    fmt.Printf("a >> 1 = %d (%08b)\n", a>>1, a>>1)  // 010 = 2
}
```

## Exemples d'applications pratiques

### Exemple 1 : Calculateur de remise

```go
package main

import "fmt"

func main() {
    prixOriginal := 100.0
    estMembre := true
    quantite := 5

    // Calcul de la remise
    var tauxRemise float64

    if estMembre && quantite >= 3 {
        tauxRemise = 0.20  // 20% pour les membres avec 3+ articles
    } else if estMembre {
        tauxRemise = 0.10  // 10% pour les membres
    } else if quantite >= 5 {
        tauxRemise = 0.05  // 5% pour 5+ articles
    } else {
        tauxRemise = 0.0   // Pas de remise
    }

    remise := prixOriginal * tauxRemise
    prixFinal := prixOriginal - remise

    fmt.Printf("Prix original: %.2f€\n", prixOriginal)
    fmt.Printf("Membre: %t, Quantité: %d\n", estMembre, quantite)
    fmt.Printf("Taux de remise: %.0f%%\n", tauxRemise*100)
    fmt.Printf("Remise: %.2f€\n", remise)
    fmt.Printf("Prix final: %.2f€\n", prixFinal)
}
```

### Exemple 2 : Système de notation

```go
package main

import "fmt"

func main() {
    note := 15.5

    // Détermination de la mention
    var mention string
    var aReussi bool

    aReussi = note >= 10

    if note >= 16 {
        mention = "Très bien"
    } else if note >= 14 {
        mention = "Bien"
    } else if note >= 12 {
        mention = "Assez bien"
    } else if note >= 10 {
        mention = "Passable"
    } else {
        mention = "Échec"
    }

    // Conditions pour les félicitations
    estExcellent := note >= 18
    estTresBon := note >= 16 && note < 18

    fmt.Printf("Note: %.1f/20\n", note)
    fmt.Printf("A réussi: %t\n", aReussi)
    fmt.Printf("Mention: %s\n", mention)

    if estExcellent {
        fmt.Println("🏆 Félicitations ! Excellence !")
    } else if estTresBon {
        fmt.Println("👏 Très beau résultat !")
    } else if aReussi {
        fmt.Println("✅ Réussite !")
    } else {
        fmt.Println("❌ Doit repasser l'examen")
    }
}
```

### Exemple 3 : Validateur de mot de passe

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    motDePasse := "MonMotDePasse123!"

    // Critères de validation
    longueurMin := len(motDePasse) >= 8
    longueurMax := len(motDePasse) <= 50
    aMajuscule := strings.ToLower(motDePasse) != motDePasse
    aMinuscule := strings.ToUpper(motDePasse) != motDePasse
    aChiffre := strings.ContainsAny(motDePasse, "0123456789")
    aSpecial := strings.ContainsAny(motDePasse, "!@#$%^&*()_+-=[]{}|;:,.<>?")

    // Validation globale
    estValide := longueurMin && longueurMax && aMajuscule &&
                 aMinuscule && aChiffre && aSpecial

    // Force du mot de passe
    score := 0
    if longueurMin { score++ }
    if aMajuscule { score++ }
    if aMinuscule { score++ }
    if aChiffre { score++ }
    if aSpecial { score++ }

    var force string
    if score >= 5 {
        force = "Très fort"
    } else if score >= 4 {
        force = "Fort"
    } else if score >= 3 {
        force = "Moyen"
    } else {
        force = "Faible"
    }

    // Affichage des résultats
    fmt.Printf("Mot de passe: %s\n", motDePasse)
    fmt.Printf("Longueur appropriée (8-50): %t\n", longueurMin && longueurMax)
    fmt.Printf("Contient majuscule: %t\n", aMajuscule)
    fmt.Printf("Contient minuscule: %t\n", aMinuscule)
    fmt.Printf("Contient chiffre: %t\n", aChiffre)
    fmt.Printf("Contient caractère spécial: %t\n", aSpecial)
    fmt.Printf("Est valide: %t\n", estValide)
    fmt.Printf("Force: %s (%d/5)\n", force, score)
}
```

## Exercices pratiques

### Exercice 1 : Calculateur simple

Créez un programme qui :
1. Déclare deux nombres
2. Effectue toutes les opérations arithmétiques de base
3. Affiche les résultats avec des messages clairs

### Exercice 2 : Vérificateur d'âge

Créez un programme qui :
1. Déclare un âge
2. Vérifie si la personne est mineure, majeure, ou senior (65+)
3. Détermine les droits (voter, conduire, etc.)

### Exercice 3 : Calculateur d'IMC avec recommandations

Créez un programme qui :
1. Calcule l'IMC
2. Détermine la catégorie (sous-poids, normal, surpoids, obèse)
3. Donne des recommandations selon l'âge et l'IMC

### Exercice 4 : Système de tarification

Créez un programme qui calcule le prix d'un billet de transport selon :
1. L'âge (enfant, adulte, senior)
2. Le type de billet (simple, aller-retour)
3. Le jour (semaine ou weekend)
4. Le statut d'abonnement

### Solutions

**Exercice 1 :**
```go
package main

import "fmt"

func main() {
    a := 15.0
    b := 4.0

    fmt.Printf("Nombres: %.1f et %.1f\n", a, b)
    fmt.Println("=== OPÉRATIONS ARITHMÉTIQUES ===")

    fmt.Printf("Addition: %.1f + %.1f = %.1f\n", a, b, a+b)
    fmt.Printf("Soustraction: %.1f - %.1f = %.1f\n", a, b, a-b)
    fmt.Printf("Multiplication: %.1f × %.1f = %.1f\n", a, b, a*b)
    fmt.Printf("Division: %.1f ÷ %.1f = %.3f\n", a, b, a/b)

    // Conversion en entiers pour le modulo
    entierA := int(a)
    entierB := int(b)
    fmt.Printf("Modulo: %d %% %d = %d\n", entierA, entierB, entierA%entierB)

    // Opérations avancées
    carre := a * a
    puissance := a * a * a  // a³

    fmt.Printf("Carré de %.1f: %.1f\n", a, carre)
    fmt.Printf("Cube de %.1f: %.1f\n", a, puissance)
}
```

**Exercice 2 :**
```go
package main

import "fmt"

func main() {
    age := 17

    fmt.Printf("Age: %d ans\n", age)
    fmt.Println("=== STATUT LÉGAL ===")

    // Catégories d'âge
    estMineur := age < 18
    estMajeur := age >= 18
    estSenior := age >= 65

    // Droits et capacités
    peutVoter := age >= 18
    peutConduire := age >= 18  // En France
    peutAcheterAlcool := age >= 18
    aTarifReduit := age < 18 || age >= 65

    if estMineur {
        fmt.Println("Statut: Mineur")
    } else if estSenior {
        fmt.Println("Statut: Senior")
    } else {
        fmt.Println("Statut: Majeur")
    }

    fmt.Printf("Peut voter: %t\n", peutVoter)
    fmt.Printf("Peut conduire: %t\n", peutConduire)
    fmt.Printf("Peut acheter de l'alcool: %t\n", peutAcheterAlcool)
    fmt.Printf("A droit à un tarif réduit: %t\n", aTarifReduit)
}
```

**Exercice 3 :**
```go
package main

import "fmt"

func main() {
    poids := 70.0  // kg
    taille := 1.75 // m
    age := 30

    // Calcul IMC
    imc := poids / (taille * taille)

    // Catégorisation
    var categorie string
    var estSain bool

    if imc < 18.5 {
        categorie = "Insuffisance pondérale"
        estSain = false
    } else if imc < 25 {
        categorie = "Poids normal"
        estSain = true
    } else if imc < 30 {
        categorie = "Surpoids"
        estSain = false
    } else {
        categorie = "Obésité"
        estSain = false
    }

    // Recommandations selon l'âge
    var recommandation string
    if age < 25 && !estSain {
        recommandation = "Consultez un médecin pour un plan adapté à votre jeune âge"
    } else if age >= 65 && !estSain {
        recommandation = "Consultez un gériatre pour un plan adapté"
    } else if !estSain {
        recommandation = "Consultez un nutritionniste"
    } else {
        recommandation = "Continuez vos bonnes habitudes !"
    }

    // Affichage des résultats
    fmt.Printf("Poids: %.1f kg, Taille: %.2f m, Age: %d ans\n", poids, taille, age)
    fmt.Printf("IMC: %.2f\n", imc)
    fmt.Printf("Catégorie: %s\n", categorie)
    fmt.Printf("Poids santé: %t\n", estSain)
    fmt.Printf("Recommandation: %s\n", recommandation)
}
```

**Exercice 4 :**
```go
package main

import "fmt"

func main() {
    age := 25
    estWeekend := false
    estAbonne := true
    typeTrajet := "aller-retour"  // "simple" ou "aller-retour"

    // Prix de base
    var prixBase float64 = 10.0

    // Modificateurs selon l'âge
    var coefficientAge float64
    if age < 12 {
        coefficientAge = 0.5  // -50% pour les enfants
    } else if age >= 65 {
        coefficientAge = 0.7  // -30% pour les seniors
    } else {
        coefficientAge = 1.0  // Prix normal pour les adultes
    }

    // Modificateur pour le type de trajet
    var coefficientTrajet float64
    if typeTrajet == "aller-retour" {
        coefficientTrajet = 1.8  // 80% de plus (au lieu de 100%)
    } else {
        coefficientTrajet = 1.0
    }

    // Modificateur weekend
    var coefficientWeekend float64
    if estWeekend {
        coefficientWeekend = 1.2  // +20% le weekend
    } else {
        coefficientWeekend = 1.0
    }

    // Modificateur abonnement
    var coefficientAbonnement float64
    if estAbonne {
        coefficientAbonnement = 0.85  // -15% pour les abonnés
    } else {
        coefficientAbonnement = 1.0
    }

    // Calcul du prix final
    prixFinal := prixBase * coefficientAge * coefficientTrajet *
                 coefficientWeekend * coefficientAbonnement

    // Calcul des économies
    prixSansReduction := prixBase * coefficientTrajet * coefficientWeekend
    economie := prixSansReduction - prixFinal

    // Affichage détaillé
    fmt.Println("=== CALCULATEUR DE TARIF TRANSPORT ===")
    fmt.Printf("Age: %d ans\n", age)
    fmt.Printf("Type de trajet: %s\n", typeTrajet)
    fmt.Printf("Weekend: %t\n", estWeekend)
    fmt.Printf("Abonné: %t\n", estAbonne)
    fmt.Println()

    fmt.Printf("Prix de base: %.2f€\n", prixBase)
    fmt.Printf("Coefficient âge: x%.2f\n", coefficientAge)
    fmt.Printf("Coefficient trajet: x%.2f\n", coefficientTrajet)
    fmt.Printf("Coefficient weekend: x%.2f\n", coefficientWeekend)
    fmt.Printf("Coefficient abonnement: x%.2f\n", coefficientAbonnement)
    fmt.Println()

    fmt.Printf("Prix sans réductions d'âge/abonnement: %.2f€\n", prixSansReduction)
    fmt.Printf("Prix final: %.2f€\n", prixFinal)

    if economie > 0 {
        fmt.Printf("Vous économisez: %.2f€ (%.0f%%)\n",
                   economie, (economie/prixSansReduction)*100)
    }
}
```

## Erreurs courantes avec les opérateurs

### 1. Division entière inattendue

```go
// ❌ ERREUR : résultat inattendu
var resultat float64 = 5 / 2  // Résultat: 2.0 (pas 2.5 !)

// ✅ CORRECT : conversion avant la division
var resultat float64 = 5.0 / 2.0  // ou float64(5) / float64(2)
```

### 2. Comparaison de nombres flottants

```go
// ❌ PROBLÈME : imprécision des flottants
var a float64 = 0.1 + 0.2
var b float64 = 0.3
estEgal := a == b  // Peut être false !

// ✅ SOLUTION : comparaison avec tolérance
epsilon := 1e-9
estEgal := math.Abs(a-b) < epsilon
```

### 3. Confusion entre = et ==

```go
// ❌ ERREUR : assignation au lieu de comparaison
if age = 18 {  // Assignation ! Erreur de compilation
    fmt.Println("Majeur")
}

// ✅ CORRECT : comparaison
if age == 18 {
    fmt.Println("Majeur")
}
```

### 4. Oubli de parenthèses

```go
// ❌ AMBIGU : priorité des opérateurs
resultat := 2 + 3 * 4 || false  // Que fait-on en premier ?

// ✅ CLAIR : avec parenthèses
resultat := ((2 + 3) * 4) || false
```

### 5. Division par zéro

```go
// ❌ DANGER : division par zéro
diviseur := 0
// resultat := 10 / diviseur  // Panic !

// ✅ SÉCURISÉ : vérification
if diviseur != 0 {
    resultat := 10 / diviseur
    fmt.Println(resultat)
} else {
    fmt.Println("Division par zéro impossible")
}
```

## Conseils pour bien utiliser les opérateurs

### 1. Lisibilité avant tout

```go
// ❌ Difficile à lire
if a > 18 && b < 100 || c == 0 && d != e || f >= g {
    // ...
}

// ✅ Plus lisible
estMajeur := a > 18
estValidAge := b < 100
estZero := c == 0
estDifferent := d != e
estSuperieur := f >= g

if (estMajeur && estValidAge) || (estZero && estDifferent) || estSuperieur {
    // ...
}
```

### 2. Utiliser des constantes pour les valeurs magiques

```go
// ❌ Valeurs "magiques"
if age >= 18 && age < 65 {
    // ...
}

// ✅ Constantes nommées
const (
    AGE_MAJORITE = 18
    AGE_RETRAITE = 65
)

if age >= AGE_MAJORITE && age < AGE_RETRAITE {
    // ...
}
```

### 3. Éviter les expressions trop complexes

```go
// ❌ Trop complexe en une ligne
resultat := ((a + b) * c - d) / (e + f) > ((g - h) * i + j) / (k - l)

// ✅ Décomposer en étapes
numerateur1 := ((a + b) * c - d) / (e + f)
numerateur2 := ((g - h) * i + j) / (k - l)
resultat := numerateur1 > numerateur2
```

### 4. Attention aux effets de bord

```go
// ❌ Effets de bord surprenants
compteur := 0
if compteur++ > 0 || compteur++ > 1 {  // compteur peut être incrémenté 1 ou 2 fois !
    // ...
}

// ✅ Clarté
compteur := 0
compteur++
if compteur > 0 || compteur > 1 {
    // ...
}
```

## Cas d'usage avancés

### 1. Validation de formulaire

```go
package main

import (
    "fmt"
    "regexp"
    "strings"
)

func main() {
    // Données du formulaire
    nom := "Alice"
    email := "alice@example.com"
    age := 25
    motDePasse := "MonMotDePasse123!"
    confirmationMDP := "MonMotDePasse123!"
    cguAcceptees := true

    // Validations
    nomValide := len(strings.TrimSpace(nom)) >= 2
    emailValide := strings.Contains(email, "@") && strings.Contains(email, ".")
    ageValide := age >= 13 && age <= 120
    mdpValide := len(motDePasse) >= 8
    mdpConfirme := motDePasse == confirmationMDP
    cguOK := cguAcceptees

    // Validation globale
    formulaireValide := nomValide && emailValide && ageValide &&
                       mdpValide && mdpConfirme && cguOK

    // Affichage des résultats
    fmt.Println("=== VALIDATION DE FORMULAIRE ===")
    fmt.Printf("Nom valide (≥2 caractères): %t\n", nomValide)
    fmt.Printf("Email valide (contient @ et .): %t\n", emailValide)
    fmt.Printf("Age valide (13-120): %t\n", ageValide)
    fmt.Printf("Mot de passe valide (≥8 caractères): %t\n", mdpValide)
    fmt.Printf("Mot de passe confirmé: %t\n", mdpConfirme)
    fmt.Printf("CGU acceptées: %t\n", cguOK)
    fmt.Printf("Formulaire valide: %t\n", formulaireValide)

    if formulaireValide {
        fmt.Println("✅ Inscription réussie !")
    } else {
        fmt.Println("❌ Veuillez corriger les erreurs")
    }
}
```

### 2. Calculateur de notes avec bonus

```go
package main

import "fmt"

func main() {
    // Notes et coefficients
    noteExamen := 14.5
    noteDevoirs := 16.0
    noteProjet := 12.0
    participation := 85  // Pourcentage de participation

    coeffExamen := 0.5
    coeffDevoirs := 0.3
    coeffProjet := 0.2

    // Calcul de la moyenne
    moyenne := noteExamen*coeffExamen + noteDevoirs*coeffDevoirs + noteProjet*coeffProjet

    // Bonus de participation
    var bonusParticipation float64
    if participation >= 90 {
        bonusParticipation = 1.0
    } else if participation >= 80 {
        bonusParticipation = 0.5
    } else if participation >= 70 {
        bonusParticipation = 0.2
    } else {
        bonusParticipation = 0.0
    }

    // Note finale avec bonus
    noteFinale := moyenne + bonusParticipation
    if noteFinale > 20 {
        noteFinale = 20  // Plafonner à 20
    }

    // Conditions de réussite
    aReussi := noteFinale >= 10
    aBienReussi := noteFinale >= 12
    aExcelle := noteFinale >= 16

    // Mention
    var mention string
    if noteFinale >= 16 {
        mention = "Très bien"
    } else if noteFinale >= 14 {
        mention = "Bien"
    } else if noteFinale >= 12 {
        mention = "Assez bien"
    } else if noteFinale >= 10 {
        mention = "Passable"
    } else {
        mention = "Insuffisant"
    }

    // Affichage
    fmt.Println("=== CALCUL DE NOTES ===")
    fmt.Printf("Note examen (coeff %.1f): %.1f\n", coeffExamen, noteExamen)
    fmt.Printf("Note devoirs (coeff %.1f): %.1f\n", coeffDevoirs, noteDevoirs)
    fmt.Printf("Note projet (coeff %.1f): %.1f\n", coeffProjet, noteProjet)
    fmt.Printf("Participation: %d%%\n", participation)
    fmt.Println()
    fmt.Printf("Moyenne calculée: %.2f\n", moyenne)
    fmt.Printf("Bonus participation: +%.1f\n", bonusParticipation)
    fmt.Printf("Note finale: %.2f/20\n", noteFinale)
    fmt.Printf("Mention: %s\n", mention)
    fmt.Println()

    if aExcelle {
        fmt.Println("🏆 Excellents résultats ! Félicitations !")
    } else if aBienReussi {
        fmt.Println("👏 Très bon travail !")
    } else if aReussi {
        fmt.Println("✅ Réussite !")
    } else {
        fmt.Println("❌ Échec - Il faut reprendre")
    }
}
```

### 3. Système de recommandation de contenu

```go
package main

import "fmt"

func main() {
    // Profil utilisateur
    age := 28
    genresPreferes := []string{"Action", "Sci-Fi", "Thriller"}
    noteMoyenne := 4.2  // Sur 5
    tempsVisionnage := 120  // Minutes par semaine
    estAbonne := true

    // Film à évaluer
    genreFilm := "Sci-Fi"
    noteFilm := 4.5
    dureeFilm := 140  // Minutes
    estPremium := true

    // Critères de recommandation
    ageApproprie := age >= 18  // Film pour adultes
    genreAime := false
    for _, genre := range genresPreferes {
        if genre == genreFilm {
            genreAime = true
            break
        }
    }

    bienNote := noteFilm >= noteMoyenne
    dureeOK := dureeFilm <= 180  // Pas trop long
    peutVoir := !estPremium || estAbonne

    // Score de recommandation
    score := 0
    if ageApproprie { score += 20 }
    if genreAime { score += 30 }
    if bienNote { score += 25 }
    if dureeOK { score += 15 }
    if peutVoir { score += 10 }

    // Décision de recommandation
    var recommandation string
    var couleur string

    if score >= 80 {
        recommandation = "Fortement recommandé"
        couleur = "🟢"
    } else if score >= 60 {
        recommandation = "Recommandé"
        couleur = "🟡"
    } else if score >= 40 {
        recommandation = "Peut-être"
        couleur = "🟠"
    } else {
        recommandation = "Non recommandé"
        couleur = "🔴"
    }

    // Affichage
    fmt.Println("=== SYSTÈME DE RECOMMANDATION ===")
    fmt.Printf("Film: %s (Note: %.1f/5, Durée: %d min)\n",
               genreFilm, noteFilm, dureeFilm)
    fmt.Printf("Premium: %t\n", estPremium)
    fmt.Println()
    fmt.Printf("Age approprié (18+): %t (+20 points)\n", ageApproprie)
    fmt.Printf("Genre aimé (%v): %t (+30 points)\n", genresPreferes, genreAime)
    fmt.Printf("Bien noté (≥%.1f): %t (+25 points)\n", noteMoyenne, bienNote)
    fmt.Printf("Durée OK (≤180 min): %t (+15 points)\n", dureeOK)
    fmt.Printf("Peut voir (abonné ou gratuit): %t (+10 points)\n", peutVoir)
    fmt.Println()
    fmt.Printf("Score total: %d/100\n", score)
    fmt.Printf("Recommandation: %s %s\n", couleur, recommandation)
}
```

## Récapitulatif

**Ce que vous avez appris :**
- ✅ Opérateurs arithmétiques (+, -, *, /, %)
- ✅ Opérateurs d'assignation (+=, -=, ++, --)
- ✅ Opérateurs de comparaison (==, !=, <, >, <=, >=)
- ✅ Opérateurs logiques (&&, ||, !)
- ✅ Priorité des opérateurs et parenthèses
- ✅ Opérateurs bit à bit (bonus)
- ✅ Applications pratiques complexes

**Points clés à retenir :**
1. **Division entière** : `5 / 2 = 2` (pas 2.5)
2. **Évaluation paresseuse** : `&&` et `||` s'arrêtent dès que le résultat est connu
3. **Parenthèses** : toujours préférer la clarté à l'optimisation
4. **Comparaisons** : attention aux flottants imprécis
5. **Lisibilité** : décomposer les expressions complexes

**Opérateurs essentiels pour débuter :**
```go
// Arithmétiques
a + b    // Addition
a - b    // Soustraction
a * b    // Multiplication
a / b    // Division
a % b    // Modulo (reste)

// Comparaisons
a == b   // Égal
a != b   // Différent
a < b    // Inférieur
a > b    // Supérieur

// Logiques
a && b   // ET
a || b   // OU
!a       // NON
```

**Bonnes pratiques :**
- Utilisez des parenthèses pour clarifier les expressions complexes
- Préférez plusieurs variables intermédiaires à une expression géante
- Attention à la division entière vs décimale
- Validez toujours les données avant les opérations (division par zéro, etc.)

---

*Excellent ! Vous maîtrisez maintenant les opérateurs en Go. Dans la prochaine section, nous découvrirons comment bien commenter et documenter votre code selon les conventions Go !*

⏭️
