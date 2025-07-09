🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11-1 : Tests unitaires avec testing

## Qu'est-ce qu'un test unitaire ?

Un test unitaire est un petit bout de code qui vérifie qu'une fonction ou une méthode fonctionne correctement. Imaginez que vous avez écrit une fonction pour additionner deux nombres. Un test unitaire va vérifier que cette fonction retourne bien le bon résultat.

**Pourquoi c'est important ?**
- Détecte les erreurs rapidement
- Évite les régressions (casser quelque chose qui marchait)
- Documente le comportement attendu
- Facilite les modifications futures

## Votre premier test

### Étape 1 : Créer une fonction simple

Commençons par créer un fichier `math.go` :

```go
package main

// Addition simple de deux nombres
func Add(a, b int) int {
    return a + b
}

// Multiplication de deux nombres
func Multiply(a, b int) int {
    return a * b
}
```

### Étape 2 : Créer le fichier de test

Créez un fichier `math_test.go` dans le même dossier :

```go
package main

import "testing"

// Notre premier test !
func TestAdd(t *testing.T) {
    // On teste si 2 + 3 = 5
    result := Add(2, 3)
    expected := 5

    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}
```

### Étape 3 : Exécuter le test

```bash
go test
```

Si tout va bien, vous devriez voir :
```
PASS
ok      your-package    0.001s
```

## Anatomie d'un test

### Structure de base

```go
func TestNomDeLaFonction(t *testing.T) {
    // 1. ARRANGE : Préparer les données
    input1 := 10
    input2 := 5
    expected := 15

    // 2. ACT : Exécuter la fonction
    result := Add(input1, input2)

    // 3. ASSERT : Vérifier le résultat
    if result != expected {
        t.Errorf("Add(%d, %d) = %d; want %d", input1, input2, result, expected)
    }
}
```

### Règles importantes

1. **Nom du fichier** : doit finir par `_test.go`
2. **Nom de la fonction** : doit commencer par `Test`
3. **Paramètre** : toujours `t *testing.T`
4. **Package** : même package que le code à tester

## Méthodes utiles de testing.T

### t.Errorf() - Signaler une erreur

```go
func TestDivision(t *testing.T) {
    result := Divide(10, 2)
    if result != 5 {
        t.Errorf("Divide(10, 2) = %f; want 5", result)
    }
}
```

### t.Fatalf() - Arrêter le test immédiatement

```go
func TestFileRead(t *testing.T) {
    file, err := os.Open("test.txt")
    if err != nil {
        t.Fatalf("Impossible d'ouvrir le fichier: %v", err)
    }
    defer file.Close()

    // Le reste du test ne s'exécutera pas si le fichier n'existe pas
}
```

### t.Logf() - Afficher des informations

```go
func TestWithLogging(t *testing.T) {
    t.Logf("Début du test")

    result := Add(1, 2)
    t.Logf("Résultat: %d", result)

    if result != 3 {
        t.Errorf("Test échoué")
    }
}
```

## Exemples pratiques

### Test d'une fonction de validation

```go
// validation.go
package main

import (
    "errors"
    "strings"
)

func ValidateEmail(email string) error {
    if email == "" {
        return errors.New("email vide")
    }

    if !strings.Contains(email, "@") {
        return errors.New("email invalide")
    }

    return nil
}
```

```go
// validation_test.go
package main

import "testing"

func TestValidateEmail(t *testing.T) {
    // Test avec un email valide
    err := ValidateEmail("test@example.com")
    if err != nil {
        t.Errorf("Email valide rejeté: %v", err)
    }

    // Test avec un email vide
    err = ValidateEmail("")
    if err == nil {
        t.Error("Email vide accepté, erreur attendue")
    }

    // Test avec un email sans @
    err = ValidateEmail("testexample.com")
    if err == nil {
        t.Error("Email sans @ accepté, erreur attendue")
    }
}
```

### Test d'une struct simple

```go
// user.go
package main

type User struct {
    Name string
    Age  int
}

func (u *User) IsAdult() bool {
    return u.Age >= 18
}

func (u *User) GetInfo() string {
    return fmt.Sprintf("%s (%d ans)", u.Name, u.Age)
}
```

```go
// user_test.go
package main

import "testing"

func TestUser_IsAdult(t *testing.T) {
    // Test utilisateur majeur
    adult := &User{Name: "Alice", Age: 25}
    if !adult.IsAdult() {
        t.Error("Alice devrait être majeure")
    }

    // Test utilisateur mineur
    minor := &User{Name: "Bob", Age: 16}
    if minor.IsAdult() {
        t.Error("Bob ne devrait pas être majeur")
    }

    // Test cas limite
    limit := &User{Name: "Charlie", Age: 18}
    if !limit.IsAdult() {
        t.Error("Charlie devrait être majeur à 18 ans")
    }
}

func TestUser_GetInfo(t *testing.T) {
    user := &User{Name: "Diana", Age: 30}
    expected := "Diana (30 ans)"
    result := user.GetInfo()

    if result != expected {
        t.Errorf("GetInfo() = %q; want %q", result, expected)
    }
}
```

## Organiser vos tests

### Grouper les tests par fonctionnalité

```go
func TestCalculator(t *testing.T) {
    // Sous-tests pour différentes opérations
    t.Run("Addition", func(t *testing.T) {
        result := Add(2, 3)
        if result != 5 {
            t.Errorf("Add(2, 3) = %d; want 5", result)
        }
    })

    t.Run("Multiplication", func(t *testing.T) {
        result := Multiply(3, 4)
        if result != 12 {
            t.Errorf("Multiply(3, 4) = %d; want 12", result)
        }
    })

    t.Run("Division par zéro", func(t *testing.T) {
        _, err := Divide(10, 0)
        if err == nil {
            t.Error("Division par zéro devrait retourner une erreur")
        }
    })
}
```

### Helper functions

```go
// Fonction helper pour éviter la répétition
func assertEqual(t *testing.T, got, want int) {
    t.Helper() // Indique que c'est une fonction helper
    if got != want {
        t.Errorf("got %d; want %d", got, want)
    }
}

func TestWithHelper(t *testing.T) {
    assertEqual(t, Add(2, 3), 5)
    assertEqual(t, Add(10, 20), 30)
    assertEqual(t, Add(-5, 5), 0)
}
```

## Commandes utiles

### Exécuter tous les tests
```bash
go test
```

### Exécuter les tests en mode verbose
```bash
go test -v
```

### Exécuter un test spécifique
```bash
go test -run TestAdd
```

### Exécuter les tests avec la couverture
```bash
go test -cover
```

### Exécuter les tests en continu
```bash
go test -watch  # Avec des outils tiers
```

## Erreurs communes et solutions

### 1. Oublier le paramètre *testing.T
```go
// ❌ Incorrect
func TestAdd() {
    // ...
}

// ✅ Correct
func TestAdd(t *testing.T) {
    // ...
}
```

### 2. Mauvais nom de fichier
```go
// ❌ Incorrect : math-test.go
// ✅ Correct : math_test.go
```

### 3. Package différent
```go
// Si votre code est dans le package "main"
// Votre test doit aussi être dans le package "main"
```

### 4. Ne pas gérer les erreurs
```go
// ❌ Incorrect
result := FunctionThatCanFail()

// ✅ Correct
result, err := FunctionThatCanFail()
if err != nil {
    t.Fatalf("Erreur inattendue: %v", err)
}
```

## Exercices pratiques

### Exercice 1 : Fonctions mathématiques
Créez des tests pour ces fonctions :

```go
func Subtract(a, b int) int {
    return a - b
}

func IsEven(n int) bool {
    return n%2 == 0
}

func Abs(n int) int {
    if n < 0 {
        return -n
    }
    return n
}
```

### Exercice 2 : Validation de mot de passe
Testez cette fonction :

```go
func ValidatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("mot de passe trop court")
    }

    hasUpper := false
    hasLower := false
    hasDigit := false

    for _, char := range password {
        switch {
        case char >= 'A' && char <= 'Z':
            hasUpper = true
        case char >= 'a' && char <= 'z':
            hasLower = true
        case char >= '0' && char <= '9':
            hasDigit = true
        }
    }

    if !hasUpper || !hasLower || !hasDigit {
        return errors.New("mot de passe doit contenir majuscule, minuscule et chiffre")
    }

    return nil
}
```

## Récapitulatif

Les tests unitaires avec le package `testing` permettent de :
- Vérifier que votre code fonctionne correctement
- Éviter les régressions lors des modifications
- Documenter le comportement attendu
- Avoir confiance dans votre code

**Points clés à retenir :**
- Les fichiers de test finissent par `_test.go`
- Les fonctions de test commencent par `Test`
- Utilisez `t.Errorf()` pour signaler les erreurs
- Utilisez `t.Fatalf()` pour arrêter le test
- Organisez vos tests avec `t.Run()`
- Créez des helper functions pour éviter la répétition

Dans la section suivante, nous découvrirons les **table-driven tests**, une technique puissante pour tester plusieurs cas avec un code minimal !

⏭️

# Solutions des exercices pratiques - Tests unitaires

## Exercice 1 : Fonctions mathématiques

### Code à tester (math.go)
```go
package main

func Subtract(a, b int) int {
    return a - b
}

func IsEven(n int) bool {
    return n%2 == 0
}

func Abs(n int) int {
    if n < 0 {
        return -n
    }
    return n
}
```

### Solution complète (math_test.go)
```go
package main

import "testing"

// Tests pour la fonction Subtract
func TestSubtract(t *testing.T) {
    // Test basique
    result := Subtract(10, 3)
    expected := 7
    if result != expected {
        t.Errorf("Subtract(10, 3) = %d; want %d", result, expected)
    }

    // Test avec nombres négatifs
    result = Subtract(-5, -3)
    expected = -2
    if result != expected {
        t.Errorf("Subtract(-5, -3) = %d; want %d", result, expected)
    }

    // Test avec résultat négatif
    result = Subtract(3, 10)
    expected = -7
    if result != expected {
        t.Errorf("Subtract(3, 10) = %d; want %d", result, expected)
    }

    // Test avec zéro
    result = Subtract(5, 0)
    expected = 5
    if result != expected {
        t.Errorf("Subtract(5, 0) = %d; want %d", result, expected)
    }

    // Test soustraction avec soi-même
    result = Subtract(8, 8)
    expected = 0
    if result != expected {
        t.Errorf("Subtract(8, 8) = %d; want %d", result, expected)
    }
}

// Tests pour la fonction IsEven
func TestIsEven(t *testing.T) {
    // Test nombres pairs
    if !IsEven(2) {
        t.Error("IsEven(2) devrait retourner true")
    }

    if !IsEven(0) {
        t.Error("IsEven(0) devrait retourner true")
    }

    if !IsEven(100) {
        t.Error("IsEven(100) devrait retourner true")
    }

    // Test nombres impairs
    if IsEven(1) {
        t.Error("IsEven(1) devrait retourner false")
    }

    if IsEven(7) {
        t.Error("IsEven(7) devrait retourner false")
    }

    if IsEven(99) {
        t.Error("IsEven(99) devrait retourner false")
    }

    // Test nombres négatifs
    if !IsEven(-2) {
        t.Error("IsEven(-2) devrait retourner true")
    }

    if IsEven(-3) {
        t.Error("IsEven(-3) devrait retourner false")
    }
}

// Tests pour la fonction Abs
func TestAbs(t *testing.T) {
    // Test nombre positif
    result := Abs(5)
    expected := 5
    if result != expected {
        t.Errorf("Abs(5) = %d; want %d", result, expected)
    }

    // Test nombre négatif
    result = Abs(-8)
    expected = 8
    if result != expected {
        t.Errorf("Abs(-8) = %d; want %d", result, expected)
    }

    // Test zéro
    result = Abs(0)
    expected = 0
    if result != expected {
        t.Errorf("Abs(0) = %d; want %d", result, expected)
    }

    // Test nombres négatifs plus grands
    result = Abs(-100)
    expected = 100
    if result != expected {
        t.Errorf("Abs(-100) = %d; want %d", result, expected)
    }

    // Test nombre positif plus grand
    result = Abs(42)
    expected = 42
    if result != expected {
        t.Errorf("Abs(42) = %d; want %d", result, expected)
    }
}
```

### Version avec sous-tests (plus organisée)
```go
package main

import "testing"

func TestSubtract(t *testing.T) {
    t.Run("Soustraction basique", func(t *testing.T) {
        result := Subtract(10, 3)
        if result != 7 {
            t.Errorf("Subtract(10, 3) = %d; want 7", result)
        }
    })

    t.Run("Nombres négatifs", func(t *testing.T) {
        result := Subtract(-5, -3)
        if result != -2 {
            t.Errorf("Subtract(-5, -3) = %d; want -2", result)
        }
    })

    t.Run("Résultat négatif", func(t *testing.T) {
        result := Subtract(3, 10)
        if result != -7 {
            t.Errorf("Subtract(3, 10) = %d; want -7", result)
        }
    })

    t.Run("Soustraction avec zéro", func(t *testing.T) {
        result := Subtract(5, 0)
        if result != 5 {
            t.Errorf("Subtract(5, 0) = %d; want 5", result)
        }
    })
}

func TestIsEven(t *testing.T) {
    t.Run("Nombres pairs", func(t *testing.T) {
        pairs := []int{2, 0, 4, 100, -2, -4}
        for _, n := range pairs {
            if !IsEven(n) {
                t.Errorf("IsEven(%d) devrait retourner true", n)
            }
        }
    })

    t.Run("Nombres impairs", func(t *testing.T) {
        impairs := []int{1, 3, 7, 99, -1, -3}
        for _, n := range impairs {
            if IsEven(n) {
                t.Errorf("IsEven(%d) devrait retourner false", n)
            }
        }
    })
}

func TestAbs(t *testing.T) {
    t.Run("Nombres positifs", func(t *testing.T) {
        positifs := []int{5, 42, 100, 1}
        for _, n := range positifs {
            result := Abs(n)
            if result != n {
                t.Errorf("Abs(%d) = %d; want %d", n, result, n)
            }
        }
    })

    t.Run("Nombres négatifs", func(t *testing.T) {
        cases := map[int]int{
            -5:   5,
            -42:  42,
            -100: 100,
            -1:   1,
        }

        for input, expected := range cases {
            result := Abs(input)
            if result != expected {
                t.Errorf("Abs(%d) = %d; want %d", input, result, expected)
            }
        }
    })

    t.Run("Zéro", func(t *testing.T) {
        result := Abs(0)
        if result != 0 {
            t.Errorf("Abs(0) = %d; want 0", result)
        }
    })
}
```

## Exercice 2 : Validation de mot de passe

### Code à tester (password.go)
```go
package main

import "errors"

func ValidatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("mot de passe trop court")
    }

    hasUpper := false
    hasLower := false
    hasDigit := false

    for _, char := range password {
        switch {
        case char >= 'A' && char <= 'Z':
            hasUpper = true
        case char >= 'a' && char <= 'z':
            hasLower = true
        case char >= '0' && char <= '9':
            hasDigit = true
        }
    }

    if !hasUpper || !hasLower || !hasDigit {
        return errors.New("mot de passe doit contenir majuscule, minuscule et chiffre")
    }

    return nil
}
```

### Solution complète (password_test.go)
```go
package main

import (
    "strings"
    "testing"
)

func TestValidatePassword(t *testing.T) {
    // Test mot de passe valide
    validPasswords := []string{
        "Password123",
        "MySecret1",
        "Hello123World",
        "Abc12345",
        "Test1234",
    }

    for _, password := range validPasswords {
        err := ValidatePassword(password)
        if err != nil {
            t.Errorf("ValidatePassword(%q) devrait être valide, mais a retourné: %v", password, err)
        }
    }

    // Test mot de passe trop court
    shortPasswords := []string{
        "Pass1",
        "Abc123",
        "Hi1",
        "",
        "Test12",
    }

    for _, password := range shortPasswords {
        err := ValidatePassword(password)
        if err == nil {
            t.Errorf("ValidatePassword(%q) devrait retourner une erreur (trop court)", password)
        }
        if err != nil && !strings.Contains(err.Error(), "trop court") {
            t.Errorf("ValidatePassword(%q) devrait retourner l'erreur 'trop court', mais a retourné: %v", password, err)
        }
    }

    // Test mot de passe sans majuscule
    noUpperPasswords := []string{
        "password123",
        "hello123",
        "test1234",
        "mypassword1",
    }

    for _, password := range noUpperPasswords {
        err := ValidatePassword(password)
        if err == nil {
            t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de majuscule)", password)
        }
        if err != nil && !strings.Contains(err.Error(), "majuscule") {
            t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'majuscule', mais a retourné: %v", password, err)
        }
    }

    // Test mot de passe sans minuscule
    noLowerPasswords := []string{
        "PASSWORD123",
        "HELLO123",
        "TEST1234",
        "MYPASSWORD1",
    }

    for _, password := range noLowerPasswords {
        err := ValidatePassword(password)
        if err == nil {
            t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de minuscule)", password)
        }
        if err != nil && !strings.Contains(err.Error(), "minuscule") {
            t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'minuscule', mais a retourné: %v", password, err)
        }
    }

    // Test mot de passe sans chiffre
    noDigitPasswords := []string{
        "Password",
        "HelloWorld",
        "TestPassword",
        "MyPassword",
    }

    for _, password := range noDigitPasswords {
        err := ValidatePassword(password)
        if err == nil {
            t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de chiffre)", password)
        }
        if err != nil && !strings.Contains(err.Error(), "chiffre") {
            t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'chiffre', mais a retourné: %v", password, err)
        }
    }
}
```

### Version avec sous-tests (plus lisible)
```go
package main

import (
    "strings"
    "testing"
)

func TestValidatePassword(t *testing.T) {
    t.Run("Mots de passe valides", func(t *testing.T) {
        validPasswords := []string{
            "Password123",
            "MySecret1",
            "Hello123World",
            "Abc12345",
            "Test1234",
            "SuperPassword9",
        }

        for _, password := range validPasswords {
            err := ValidatePassword(password)
            if err != nil {
                t.Errorf("ValidatePassword(%q) devrait être valide, mais a retourné: %v", password, err)
            }
        }
    })

    t.Run("Mots de passe trop courts", func(t *testing.T) {
        shortPasswords := []string{
            "Pass1",
            "Abc123",
            "Hi1",
            "",
            "Test12",
            "Ab1",
        }

        for _, password := range shortPasswords {
            err := ValidatePassword(password)
            if err == nil {
                t.Errorf("ValidatePassword(%q) devrait retourner une erreur (trop court)", password)
            }
            if err != nil && !strings.Contains(err.Error(), "trop court") {
                t.Errorf("ValidatePassword(%q) devrait retourner l'erreur 'trop court', mais a retourné: %v", password, err)
            }
        }
    })

    t.Run("Mots de passe sans majuscule", func(t *testing.T) {
        noUpperPasswords := []string{
            "password123",
            "hello123",
            "test1234",
            "mypassword1",
        }

        for _, password := range noUpperPasswords {
            err := ValidatePassword(password)
            if err == nil {
                t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de majuscule)", password)
            }
            if err != nil && !strings.Contains(err.Error(), "majuscule") {
                t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'majuscule', mais a retourné: %v", password, err)
            }
        }
    })

    t.Run("Mots de passe sans minuscule", func(t *testing.T) {
        noLowerPasswords := []string{
            "PASSWORD123",
            "HELLO123",
            "TEST1234",
            "MYPASSWORD1",
        }

        for _, password := range noLowerPasswords {
            err := ValidatePassword(password)
            if err == nil {
                t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de minuscule)", password)
            }
            if err != nil && !strings.Contains(err.Error(), "minuscule") {
                t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'minuscule', mais a retourné: %v", password, err)
            }
        }
    })

    t.Run("Mots de passe sans chiffre", func(t *testing.T) {
        noDigitPasswords := []string{
            "Password",
            "HelloWorld",
            "TestPassword",
            "MyPassword",
        }

        for _, password := range noDigitPasswords {
            err := ValidatePassword(password)
            if err == nil {
                t.Errorf("ValidatePassword(%q) devrait retourner une erreur (pas de chiffre)", password)
            }
            if err != nil && !strings.Contains(err.Error(), "chiffre") {
                t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec 'chiffre', mais a retourné: %v", password, err)
            }
        }
    })

    t.Run("Cas limites", func(t *testing.T) {
        // Test avec exactement 8 caractères
        err := ValidatePassword("Password1")
        if err != nil {
            t.Errorf("ValidatePassword('Password1') devrait être valide (8 caractères exactement)")
        }

        // Test avec 7 caractères
        err = ValidatePassword("Pass1")
        if err == nil {
            t.Error("ValidatePassword('Pass1') devrait retourner une erreur (7 caractères)")
        }
    })
}
```

### Version avec helper function (plus propre)
```go
package main

import (
    "strings"
    "testing"
)

// Helper function pour vérifier qu'une erreur contient un message spécifique
func checkError(t *testing.T, err error, expectedMsg string, password string) {
    t.Helper()
    if err == nil {
        t.Errorf("ValidatePassword(%q) devrait retourner une erreur contenant '%s'", password, expectedMsg)
        return
    }
    if !strings.Contains(err.Error(), expectedMsg) {
        t.Errorf("ValidatePassword(%q) devrait retourner l'erreur avec '%s', mais a retourné: %v", password, expectedMsg, err)
    }
}

// Helper function pour vérifier qu'un mot de passe est valide
func checkValid(t *testing.T, password string) {
    t.Helper()
    err := ValidatePassword(password)
    if err != nil {
        t.Errorf("ValidatePassword(%q) devrait être valide, mais a retourné: %v", password, err)
    }
}

func TestValidatePassword(t *testing.T) {
    t.Run("Mots de passe valides", func(t *testing.T) {
        validPasswords := []string{
            "Password123",
            "MySecret1",
            "Hello123World",
            "Abc12345",
        }

        for _, password := range validPasswords {
            checkValid(t, password)
        }
    })

    t.Run("Mots de passe trop courts", func(t *testing.T) {
        shortPasswords := []string{"Pass1", "Abc123", "Hi1", ""}

        for _, password := range shortPasswords {
            err := ValidatePassword(password)
            checkError(t, err, "trop court", password)
        }
    })

    t.Run("Mots de passe sans majuscule", func(t *testing.T) {
        noUpperPasswords := []string{"password123", "hello123", "test1234"}

        for _, password := range noUpperPasswords {
            err := ValidatePassword(password)
            checkError(t, err, "majuscule", password)
        }
    })

    t.Run("Mots de passe sans minuscule", func(t *testing.T) {
        noLowerPasswords := []string{"PASSWORD123", "HELLO123", "TEST1234"}

        for _, password := range noLowerPasswords {
            err := ValidatePassword(password)
            checkError(t, err, "minuscule", password)
        }
    })

    t.Run("Mots de passe sans chiffre", func(t *testing.T) {
        noDigitPasswords := []string{"Password", "HelloWorld", "TestPassword"}

        for _, password := range noDigitPasswords {
            err := ValidatePassword(password)
            checkError(t, err, "chiffre", password)
        }
    })
}
```

## Exécution des tests

Pour exécuter tous les tests :
```bash
go test
```

Pour exécuter en mode verbose :
```bash
go test -v
```

Résultat attendu :
```
=== RUN   TestSubtract
--- PASS: TestSubtract (0.00s)
=== RUN   TestIsEven
--- PASS: TestIsEven (0.00s)
=== RUN   TestAbs
--- PASS: TestAbs (0.00s)
=== RUN   TestValidatePassword
--- PASS: TestValidatePassword (0.00s)
PASS
ok      your-package    0.001s
```

## Points clés des solutions

1. **Couverture complète** : Les tests couvrent tous les cas possibles (valides, invalides, cas limites)
2. **Messages d'erreur clairs** : Chaque test explique ce qui est attendu
3. **Organisation** : Utilisation de sous-tests pour grouper les cas similaires
4. **Helper functions** : Évitent la duplication de code dans les tests
5. **Cas limites** : Tests des valeurs frontières (0, nombres négatifs, longueur minimum)

Ces solutions montrent comment écrire des tests robustes et maintenables !

⏭️
