🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7-1 : Conventions d'erreur en Go

## Introduction

En Go, la gestion des erreurs est différente de nombreux autres langages de programmation. Au lieu d'utiliser des exceptions (try/catch), Go utilise des valeurs d'erreur explicites. Cette approche rend le code plus prévisible et force le développeur à traiter les erreurs de manière consciente.

## Principe fondamental

En Go, les erreurs sont des **valeurs** comme les autres. Elles sont représentées par l'interface `error` qui est définie dans le package standard :

```go
type error interface {
    Error() string
}
```

## Convention de base : valeur de retour multiple

La convention principale en Go est que les fonctions qui peuvent échouer retournent **deux valeurs** :
1. Le résultat (si tout va bien)
2. Une erreur (si quelque chose ne va pas)

```go
func diviser(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division par zéro impossible")
    }
    return a / b, nil
}
```

## Vérification d'erreur standard

La façon standard de vérifier une erreur en Go suit ce pattern :

```go
func main() {
    resultat, err := diviser(10, 2)
    if err != nil {
        fmt.Println("Erreur:", err)
        return
    }
    fmt.Println("Résultat:", resultat)
}
```

**Points importants :**
- On vérifie toujours `err != nil` pour détecter une erreur
- Si `err` est `nil`, cela signifie qu'il n'y a pas d'erreur
- La gestion d'erreur se fait **immédiatement** après l'appel de fonction

## Exemples pratiques

### Lecture d'un fichier

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // Tentative de lecture d'un fichier
    contenu, err := os.ReadFile("monfichier.txt")
    if err != nil {
        fmt.Println("Impossible de lire le fichier:", err)
        return
    }

    fmt.Println("Contenu du fichier:", string(contenu))
}
```

### Conversion de chaîne en nombre

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // Tentative de conversion
    nombre, err := strconv.Atoi("123")
    if err != nil {
        fmt.Println("Conversion impossible:", err)
        return
    }

    fmt.Println("Nombre converti:", nombre)
}
```

## Création d'erreurs

### Avec `fmt.Errorf()`

```go
import "fmt"

func verifierAge(age int) error {
    if age < 0 {
        return fmt.Errorf("l'âge ne peut pas être négatif: %d", age)
    }
    if age > 150 {
        return fmt.Errorf("l'âge semble irréaliste: %d", age)
    }
    return nil
}
```

### Avec `errors.New()`

```go
import "errors"

func verifierMotDePasse(mdp string) error {
    if len(mdp) < 8 {
        return errors.New("le mot de passe doit contenir au moins 8 caractères")
    }
    return nil
}
```

## Bonnes pratiques

### 1. Toujours vérifier les erreurs

```go
// ✅ Bon
fichier, err := os.Open("test.txt")
if err != nil {
    return err
}
defer fichier.Close()

// ❌ Mauvais - ignorer l'erreur
fichier, _ := os.Open("test.txt")
```

### 2. Messages d'erreur en minuscules

```go
// ✅ Bon
return errors.New("fichier introuvable")

// ❌ Mauvais
return errors.New("Fichier introuvable")
```

### 3. Pas de ponctuation finale

```go
// ✅ Bon
return errors.New("connexion échouée")

// ❌ Mauvais
return errors.New("connexion échouée.")
```

### 4. Contexte dans les messages

```go
// ✅ Bon - donne du contexte
return fmt.Errorf("impossible de se connecter à la base de données: %v", err)

// ❌ Moins bon - pas de contexte
return err
```

## Exemple complet

```go
package main

import (
    "fmt"
    "strconv"
)

// Fonction qui peut échouer
func calculerMoyenne(notes []string) (float64, error) {
    if len(notes) == 0 {
        return 0, fmt.Errorf("impossible de calculer la moyenne: aucune note fournie")
    }

    var somme float64
    for i, noteStr := range notes {
        note, err := strconv.ParseFloat(noteStr, 64)
        if err != nil {
            return 0, fmt.Errorf("note %d invalide (%s): %v", i+1, noteStr, err)
        }

        if note < 0 || note > 20 {
            return 0, fmt.Errorf("note %d hors limite (0-20): %.2f", i+1, note)
        }

        somme += note
    }

    return somme / float64(len(notes)), nil
}

func main() {
    // Test avec des notes valides
    notes := []string{"15.5", "12", "18.25", "14"}
    moyenne, err := calculerMoyenne(notes)
    if err != nil {
        fmt.Println("Erreur:", err)
        return
    }
    fmt.Printf("Moyenne: %.2f\n", moyenne)

    // Test avec une note invalide
    notesInvalides := []string{"15.5", "abc", "18.25"}
    _, err = calculerMoyenne(notesInvalides)
    if err != nil {
        fmt.Println("Erreur détectée:", err)
    }
}
```

## Points clés à retenir

1. **Les erreurs sont des valeurs** : elles se manipulent comme n'importe quelle autre valeur
2. **Convention du retour multiple** : `(résultat, error)`
3. **Vérification immédiate** : toujours vérifier `err != nil` après un appel
4. **Messages clairs** : les erreurs doivent être compréhensibles pour le développeur
5. **Pas d'exceptions** : Go n'utilise pas le système try/catch

Cette approche peut sembler verbeuse au début, mais elle rend le code plus prévisible et force à traiter les cas d'erreur de manière explicite, ce qui améliore la robustesse des programmes.

⏭️
