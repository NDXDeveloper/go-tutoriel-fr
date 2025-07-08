🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7-2 : Création d'erreurs personnalisées

## Introduction

Dans la section précédente, nous avons vu comment utiliser les erreurs de base avec `errors.New()` et `fmt.Errorf()`. Maintenant, nous allons apprendre à créer nos propres types d'erreurs personnalisées pour des cas plus complexes.

## Pourquoi créer des erreurs personnalisées ?

Les erreurs personnalisées permettent de :
- **Transporter plus d'informations** que juste un message
- **Identifier précisément** le type d'erreur
- **Traiter différemment** selon le type d'erreur
- **Améliorer le débogage** avec des détails contextuels

## Interface error : rappel

Toute erreur en Go doit implémenter l'interface `error` :

```go
type error interface {
    Error() string
}
```

Cela signifie que n'importe quel type avec une méthode `Error() string` peut être une erreur.

## Méthode 1 : Struct simple

### Création d'une erreur basique

```go
package main

import "fmt"

// Définition d'un type d'erreur personnalisé
type ErreurValidation struct {
    Champ   string
    Valeur  string
    Message string
}

// Implémentation de l'interface error
func (e ErreurValidation) Error() string {
    return fmt.Sprintf("erreur de validation sur '%s' (valeur: '%s'): %s",
        e.Champ, e.Valeur, e.Message)
}

// Fonction qui utilise notre erreur personnalisée
func validerEmail(email string) error {
    if email == "" {
        return ErreurValidation{
            Champ:   "email",
            Valeur:  email,
            Message: "ne peut pas être vide",
        }
    }

    if !strings.Contains(email, "@") {
        return ErreurValidation{
            Champ:   "email",
            Valeur:  email,
            Message: "doit contenir un @",
        }
    }

    return nil
}

func main() {
    // Test avec un email invalide
    err := validerEmail("utilisateur")
    if err != nil {
        fmt.Println("Erreur détectée:", err)
    }
}
```

### Avantages de cette approche

- **Lisibilité** : l'erreur est très descriptive
- **Structuré** : les informations sont organisées
- **Extensible** : on peut ajouter d'autres champs facilement

## Méthode 2 : Erreurs avec codes

Pour des applications plus complexes, on peut utiliser des codes d'erreur :

```go
package main

import "fmt"

// Énumération des codes d'erreur
type CodeErreur int

const (
    CodeChampVide CodeErreur = iota + 1
    CodeFormatInvalide
    CodeTropCourt
    CodeTropLong
)

// Mapping des codes vers des messages
var messagesErreur = map[CodeErreur]string{
    CodeChampVide:      "le champ ne peut pas être vide",
    CodeFormatInvalide: "le format est invalide",
    CodeTropCourt:      "la valeur est trop courte",
    CodeTropLong:       "la valeur est trop longue",
}

// Structure d'erreur avec code
type ErreurMetier struct {
    Code    CodeErreur
    Champ   string
    Valeur  string
    Details string
}

func (e ErreurMetier) Error() string {
    message := messagesErreur[e.Code]
    if e.Details != "" {
        message += " (" + e.Details + ")"
    }
    return fmt.Sprintf("[%s] %s: %s", e.Champ, message, e.Valeur)
}

// Fonction de validation avec codes d'erreur
func validerMotDePasse(mdp string) error {
    if mdp == "" {
        return ErreurMetier{
            Code:   CodeChampVide,
            Champ:  "mot_de_passe",
            Valeur: mdp,
        }
    }

    if len(mdp) < 8 {
        return ErreurMetier{
            Code:    CodeTropCourt,
            Champ:   "mot_de_passe",
            Valeur:  mdp,
            Details: "minimum 8 caractères requis",
        }
    }

    return nil
}

func main() {
    err := validerMotDePasse("123")
    if err != nil {
        fmt.Println("Erreur:", err)

        // On peut récupérer le code d'erreur
        if errMetier, ok := err.(ErreurMetier); ok {
            fmt.Printf("Code d'erreur: %d\n", errMetier.Code)
        }
    }
}
```

## Méthode 3 : Erreurs avec wrapping (Go 1.13+)

Depuis Go 1.13, on peut "emballer" des erreurs pour conserver la chaîne d'erreurs :

```go
package main

import (
    "fmt"
    "errors"
    "os"
)

// Erreur personnalisée qui peut emballer une autre erreur
type ErreurFichier struct {
    Fichier string
    Erreur  error
}

func (e ErreurFichier) Error() string {
    return fmt.Sprintf("erreur avec le fichier '%s': %v", e.Fichier, e.Erreur)
}

// Implémentation de l'interface Unwrap (Go 1.13+)
func (e ErreurFichier) Unwrap() error {
    return e.Erreur
}

// Fonction qui lit un fichier avec erreur personnalisée
func lireFichierConfig(nom string) (string, error) {
    contenu, err := os.ReadFile(nom)
    if err != nil {
        return "", ErreurFichier{
            Fichier: nom,
            Erreur:  err,
        }
    }
    return string(contenu), nil
}

func main() {
    _, err := lireFichierConfig("config.txt")
    if err != nil {
        fmt.Println("Erreur:", err)

        // Vérifier si c'est une erreur de fichier inexistant
        if errors.Is(err, os.ErrNotExist) {
            fmt.Println("Le fichier n'existe pas")
        }

        // Ou récupérer l'erreur originale
        var errFichier ErreurFichier
        if errors.As(err, &errFichier) {
            fmt.Printf("Problème avec le fichier: %s\n", errFichier.Fichier)
        }
    }
}
```

## Méthode 4 : Erreurs prédéfinies

Pour des erreurs courantes, on peut créer des variables d'erreur prédéfinies :

```go
package main

import (
    "errors"
    "fmt"
)

// Erreurs prédéfinies
var (
    ErrUtilisateurInexistant = errors.New("utilisateur inexistant")
    ErrMotDePasseIncorrect   = errors.New("mot de passe incorrect")
    ErrSessionExpiree        = errors.New("session expirée")
)

// Fonction d'authentification
func authentifier(nom, mdp string) error {
    // Simulation de vérifications
    if nom == "" {
        return ErrUtilisateurInexistant
    }

    if mdp != "secret123" {
        return ErrMotDePasseIncorrect
    }

    return nil
}

func main() {
    err := authentifier("", "secret123")
    if err != nil {
        // On peut comparer directement avec nos erreurs prédéfinies
        switch err {
        case ErrUtilisateurInexistant:
            fmt.Println("Veuillez créer un compte")
        case ErrMotDePasseIncorrect:
            fmt.Println("Mot de passe incorrect")
        case ErrSessionExpiree:
            fmt.Println("Veuillez vous reconnecter")
        default:
            fmt.Println("Erreur inconnue:", err)
        }
    }
}
```

## Exemple complet : Système de validation

```go
package main

import (
    "fmt"
    "regexp"
    "strings"
)

// Types d'erreurs de validation
type TypeValidation string

const (
    ValidationVide     TypeValidation = "VIDE"
    ValidationFormat   TypeValidation = "FORMAT"
    ValidationLongueur TypeValidation = "LONGUEUR"
)

// Erreur de validation complète
type ErreurValidation struct {
    Type    TypeValidation
    Champ   string
    Valeur  string
    Message string
    Min     int // Pour les erreurs de longueur
    Max     int
}

func (e ErreurValidation) Error() string {
    switch e.Type {
    case ValidationVide:
        return fmt.Sprintf("le champ '%s' est obligatoire", e.Champ)
    case ValidationFormat:
        return fmt.Sprintf("le format du champ '%s' est invalide: %s", e.Champ, e.Message)
    case ValidationLongueur:
        return fmt.Sprintf("le champ '%s' doit contenir entre %d et %d caractères (actuellement: %d)",
            e.Champ, e.Min, e.Max, len(e.Valeur))
    default:
        return fmt.Sprintf("erreur de validation sur '%s': %s", e.Champ, e.Message)
    }
}

// Structure utilisateur
type Utilisateur struct {
    Nom     string
    Email   string
    Age     int
}

// Fonction de validation complète
func validerUtilisateur(u Utilisateur) []error {
    var erreurs []error

    // Validation du nom
    if strings.TrimSpace(u.Nom) == "" {
        erreurs = append(erreurs, ErreurValidation{
            Type:  ValidationVide,
            Champ: "nom",
            Valeur: u.Nom,
        })
    } else if len(u.Nom) < 2 || len(u.Nom) > 50 {
        erreurs = append(erreurs, ErreurValidation{
            Type:  ValidationLongueur,
            Champ: "nom",
            Valeur: u.Nom,
            Min:   2,
            Max:   50,
        })
    }

    // Validation de l'email
    if strings.TrimSpace(u.Email) == "" {
        erreurs = append(erreurs, ErreurValidation{
            Type:  ValidationVide,
            Champ: "email",
            Valeur: u.Email,
        })
    } else {
        emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
        if !emailRegex.MatchString(u.Email) {
            erreurs = append(erreurs, ErreurValidation{
                Type:    ValidationFormat,
                Champ:   "email",
                Valeur:  u.Email,
                Message: "format email invalide",
            })
        }
    }

    // Validation de l'âge
    if u.Age < 0 || u.Age > 150 {
        erreurs = append(erreurs, ErreurValidation{
            Type:    ValidationFormat,
            Champ:   "age",
            Valeur:  fmt.Sprintf("%d", u.Age),
            Message: "l'âge doit être entre 0 et 150",
        })
    }

    return erreurs
}

func main() {
    // Test avec un utilisateur invalide
    utilisateur := Utilisateur{
        Nom:   "",
        Email: "email-invalide",
        Age:   -5,
    }

    erreurs := validerUtilisateur(utilisateur)
    if len(erreurs) > 0 {
        fmt.Println("Erreurs de validation détectées:")
        for i, err := range erreurs {
            fmt.Printf("%d. %s\n", i+1, err)
        }
    }

    // Test avec un utilisateur valide
    utilisateurValide := Utilisateur{
        Nom:   "Jean Dupont",
        Email: "jean.dupont@email.com",
        Age:   30,
    }

    erreurs = validerUtilisateur(utilisateurValide)
    if len(erreurs) == 0 {
        fmt.Println("Utilisateur valide!")
    }
}
```

## Bonnes pratiques

### 1. Nommage cohérent
```go
// ✅ Bon - préfixe "Err" pour les types d'erreur
type ErreurValidation struct { ... }

// ✅ Bon - préfixe "Err" pour les variables d'erreur
var ErrUtilisateurInexistant = errors.New("...")
```

### 2. Messages d'erreur informatifs
```go
// ✅ Bon - détaillé et actionnable
return fmt.Errorf("impossible de se connecter à la base de données %s:%d: %v",
    host, port, err)

// ❌ Mauvais - trop vague
return errors.New("erreur de connexion")
```

### 3. Utiliser les erreurs prédéfinies quand c'est possible
```go
// ✅ Bon - réutilise les erreurs standard
if file == "" {
    return os.ErrNotExist
}

// ❌ Moins bon - recrée une erreur similaire
if file == "" {
    return errors.New("file not found")
}
```

### 4. Permettre l'inspection des erreurs
```go
// ✅ Bon - les champs sont exportés pour inspection
type ErreurValidation struct {
    Champ   string  // Exporté
    Valeur  string  // Exporté
    Message string  // Exporté
}
```

## Points clés à retenir

1. **Implémenter l'interface error** : méthode `Error() string`
2. **Transporter plus d'informations** : utiliser des structs pour les détails
3. **Codes d'erreur** : pour les systèmes complexes
4. **Erreurs prédéfinies** : pour les cas courants
5. **Wrapping** : pour conserver la chaîne d'erreurs (Go 1.13+)
6. **Inspection** : permettre aux utilisateurs d'analyser les erreurs

Les erreurs personnalisées rendent votre code plus robuste et facilitent le débogage. Elles permettent aussi aux utilisateurs de votre code de traiter différemment chaque type d'erreur selon leurs besoins.

⏭️
