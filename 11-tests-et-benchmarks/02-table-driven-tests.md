🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11-2 : Table-driven tests

## Qu'est-ce qu'un table-driven test ?

Un **table-driven test** (test dirigé par table) est une technique qui permet de tester plusieurs cas avec un seul test en utilisant un tableau de données. Au lieu d'écrire 10 fonctions de test similaires, vous créez un tableau avec 10 cas de test et une seule boucle pour les exécuter tous.

**Pourquoi c'est génial ?**
- Moins de code répétitif
- Plus facile d'ajouter de nouveaux cas
- Tests plus lisibles et organisés
- Maintenance simplifiée

## Le problème : répétition de code

### Exemple sans table-driven tests (répétitif)

```go
func TestAdd_Case1(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2, 3) = %d; want 5", result)
    }
}

func TestAdd_Case2(t *testing.T) {
    result := Add(0, 0)
    if result != 0 {
        t.Errorf("Add(0, 0) = %d; want 0", result)
    }
}

func TestAdd_Case3(t *testing.T) {
    result := Add(-1, 1)
    if result != 0 {
        t.Errorf("Add(-1, 1) = %d; want 0", result)
    }
}

func TestAdd_Case4(t *testing.T) {
    result := Add(10, -5)
    if result != 5 {
        t.Errorf("Add(10, -5) = %d; want 5", result)
    }
}
```

**Problèmes :**
- Beaucoup de répétition
- Difficile d'ajouter de nouveaux cas
- Code verbeux et difficile à maintenir

## La solution : table-driven tests

### Exemple avec table-driven tests (élégant)

```go
func TestAdd(t *testing.T) {
    // Tableau des cas de test
    tests := []struct {
        name     string // Description du test
        a, b     int    // Paramètres d'entrée
        expected int    // Résultat attendu
    }{
        {"addition simple", 2, 3, 5},
        {"addition avec zéro", 0, 0, 0},
        {"addition négative", -1, 1, 0},
        {"addition mixte", 10, -5, 5},
        {"grands nombres", 1000, 2000, 3000},
    }

    // Boucle à travers tous les cas
    for _, test := range tests {
        t.Run(test.name, func(t *testing.T) {
            result := Add(test.a, test.b)
            if result != test.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    test.a, test.b, result, test.expected)
            }
        })
    }
}
```

**Avantages :**
- Code plus court et plus lisible
- Facile d'ajouter des cas : juste une ligne !
- Structure claire et organisée
- Tests exécutés individuellement avec `t.Run()`

## Structure d'un table-driven test

### Anatomie complète

```go
func TestFunctionName(t *testing.T) {
    // 1. Définir la structure des cas de test
    tests := []struct {
        name     string  // Description du cas
        input1   type1   // Premier paramètre
        input2   type2   // Deuxième paramètre (si nécessaire)
        expected type3   // Résultat attendu
        wantErr  bool    // Si on attend une erreur (optionnel)
    }{
        // 2. Lister tous les cas de test
        {
            name:     "description du cas 1",
            input1:   valeur1,
            input2:   valeur2,
            expected: resultat1,
        },
        {
            name:     "description du cas 2",
            input1:   valeur3,
            input2:   valeur4,
            expected: resultat2,
        },
        // ... autres cas
    }

    // 3. Boucle d'exécution
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 4. Exécuter la fonction
            result := FunctionName(tt.input1, tt.input2)

            // 5. Vérifier le résultat
            if result != tt.expected {
                t.Errorf("got %v; want %v", result, tt.expected)
            }
        })
    }
}
```

## Exemples pratiques

### Exemple 1 : Test de fonction mathématique

```go
// Code à tester
func Multiply(a, b int) int {
    return a * b
}

// Table-driven test
func TestMultiply(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"multiplication simple", 3, 4, 12},
        {"multiplication par zéro", 5, 0, 0},
        {"multiplication par un", 7, 1, 7},
        {"nombres négatifs", -3, -4, 12},
        {"négatif et positif", -5, 3, -15},
        {"grands nombres", 100, 200, 20000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Multiply(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Multiply(%d, %d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### Exemple 2 : Test avec chaînes de caractères

```go
// Code à tester
func Reverse(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

// Table-driven test
func TestReverse(t *testing.T) {
    tests := []struct {
        name     string
        input    string
        expected string
    }{
        {"chaîne simple", "hello", "olleh"},
        {"un caractère", "a", "a"},
        {"chaîne vide", "", ""},
        {"palindrome", "radar", "radar"},
        {"avec espaces", "hello world", "dlrow olleh"},
        {"avec chiffres", "abc123", "321cba"},
        {"caractères spéciaux", "!@#", "#@!"},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Reverse(tt.input)
            if result != tt.expected {
                t.Errorf("Reverse(%q) = %q; want %q",
                    tt.input, result, tt.expected)
            }
        })
    }
}
```

### Exemple 3 : Test avec gestion d'erreurs

```go
// Code à tester
func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division par zéro")
    }
    return a / b, nil
}

// Table-driven test avec erreurs
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        expected  float64
        expectErr bool
    }{
        {"division simple", 10.0, 2.0, 5.0, false},
        {"division par zéro", 5.0, 0.0, 0.0, true},
        {"division négative", -10.0, 2.0, -5.0, false},
        {"division décimale", 7.0, 2.0, 3.5, false},
        {"division de zéro", 0.0, 5.0, 0.0, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Divide(tt.a, tt.b)

            // Vérifier si on attend une erreur
            if tt.expectErr {
                if err == nil {
                    t.Errorf("Divide(%f, %f) devrait retourner une erreur", tt.a, tt.b)
                }
                return // Pas besoin de vérifier le résultat si erreur attendue
            }

            // Vérifier qu'il n'y a pas d'erreur inattendue
            if err != nil {
                t.Errorf("Divide(%f, %f) erreur inattendue: %v", tt.a, tt.b, err)
                return
            }

            // Vérifier le résultat
            if result != tt.expected {
                t.Errorf("Divide(%f, %f) = %f; want %f",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### Exemple 4 : Test de validation complexe

```go
// Code à tester
type User struct {
    Name  string
    Email string
    Age   int
}

func ValidateUser(user User) []string {
    var errors []string

    if user.Name == "" {
        errors = append(errors, "nom requis")
    }

    if !strings.Contains(user.Email, "@") {
        errors = append(errors, "email invalide")
    }

    if user.Age < 0 || user.Age > 120 {
        errors = append(errors, "âge invalide")
    }

    return errors
}

// Table-driven test pour validation
func TestValidateUser(t *testing.T) {
    tests := []struct {
        name           string
        user           User
        expectedErrors []string
    }{
        {
            name: "utilisateur valide",
            user: User{Name: "Alice", Email: "alice@example.com", Age: 25},
            expectedErrors: nil,
        },
        {
            name: "nom manquant",
            user: User{Name: "", Email: "test@example.com", Age: 25},
            expectedErrors: []string{"nom requis"},
        },
        {
            name: "email invalide",
            user: User{Name: "Bob", Email: "invalid-email", Age: 30},
            expectedErrors: []string{"email invalide"},
        },
        {
            name: "âge négatif",
            user: User{Name: "Charlie", Email: "charlie@example.com", Age: -5},
            expectedErrors: []string{"âge invalide"},
        },
        {
            name: "multiples erreurs",
            user: User{Name: "", Email: "bad-email", Age: 150},
            expectedErrors: []string{"nom requis", "email invalide", "âge invalide"},
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            errors := ValidateUser(tt.user)

            // Comparer les slices d'erreurs
            if len(errors) != len(tt.expectedErrors) {
                t.Errorf("ValidateUser() retourné %d erreurs; want %d",
                    len(errors), len(tt.expectedErrors))
                return
            }

            for i, expectedErr := range tt.expectedErrors {
                if i >= len(errors) || errors[i] != expectedErr {
                    t.Errorf("ValidateUser() erreur[%d] = %q; want %q",
                        i, errors[i], expectedErr)
                }
            }
        })
    }
}
```

## Techniques avancées

### 1. Tests avec setup et cleanup

```go
func TestDatabaseOperations(t *testing.T) {
    tests := []struct {
        name     string
        userID   int
        expected User
    }{
        {"utilisateur existant", 1, User{ID: 1, Name: "Alice"}},
        {"utilisateur inexistant", 999, User{}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup spécifique au test
            db := setupTestDatabase(t)
            defer cleanupTestDatabase(t, db)

            // Test
            result := GetUser(db, tt.userID)
            if result != tt.expected {
                t.Errorf("got %+v; want %+v", result, tt.expected)
            }
        })
    }
}
```

### 2. Tests avec données complexes

```go
func TestJSONMarshaling(t *testing.T) {
    tests := []struct {
        name     string
        input    interface{}
        expected string
    }{
        {
            name:     "simple object",
            input:    map[string]string{"name": "Alice", "city": "Paris"},
            expected: `{"city":"Paris","name":"Alice"}`,
        },
        {
            name:     "array",
            input:    []int{1, 2, 3},
            expected: `[1,2,3]`,
        },
        {
            name:     "nested object",
            input:    map[string]interface{}{
                "user": map[string]string{"name": "Bob"},
                "count": 42,
            },
            expected: `{"count":42,"user":{"name":"Bob"}}`,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := json.Marshal(tt.input)
            if err != nil {
                t.Fatalf("json.Marshal() erreur = %v", err)
            }

            if string(result) != tt.expected {
                t.Errorf("json.Marshal() = %s; want %s", result, tt.expected)
            }
        })
    }
}
```

### 3. Tests parallèles

```go
func TestParallelOperations(t *testing.T) {
    tests := []struct {
        name  string
        input int
        want  int
    }{
        {"cas 1", 1, 2},
        {"cas 2", 2, 4},
        {"cas 3", 3, 6},
        {"cas 4", 4, 8},
    }

    for _, tt := range tests {
        tt := tt // Capture de la variable pour la goroutine
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Exécution en parallèle

            result := SlowOperation(tt.input)
            if result != tt.want {
                t.Errorf("got %d; want %d", result, tt.want)
            }
        })
    }
}
```

## Comparaison : avant et après

### Avant (tests répétitifs)
```go
// 50 lignes pour 5 cas de test
func TestIsValidEmail_Valid(t *testing.T) {
    if !IsValidEmail("test@example.com") {
        t.Error("Email valide rejeté")
    }
}

func TestIsValidEmail_NoAt(t *testing.T) {
    if IsValidEmail("testexample.com") {
        t.Error("Email sans @ accepté")
    }
}

func TestIsValidEmail_Empty(t *testing.T) {
    if IsValidEmail("") {
        t.Error("Email vide accepté")
    }
}

func TestIsValidEmail_OnlyAt(t *testing.T) {
    if IsValidEmail("@") {
        t.Error("Email avec seulement @ accepté")
    }
}

func TestIsValidEmail_Complex(t *testing.T) {
    if !IsValidEmail("user+tag@domain.co.uk") {
        t.Error("Email complexe valide rejeté")
    }
}
```

### Après (table-driven)
```go
// 25 lignes pour 5 cas de test + facilement extensible
func TestIsValidEmail(t *testing.T) {
    tests := []struct {
        name     string
        email    string
        expected bool
    }{
        {"email valide", "test@example.com", true},
        {"sans @", "testexample.com", false},
        {"email vide", "", false},
        {"seulement @", "@", false},
        {"email complexe", "user+tag@domain.co.uk", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := IsValidEmail(tt.email)
            if result != tt.expected {
                t.Errorf("IsValidEmail(%q) = %v; want %v",
                    tt.email, result, tt.expected)
            }
        })
    }
}
```

## Bonnes pratiques

### 1. Noms descriptifs
```go
// ✅ Bon : descriptif
{"addition de nombres positifs", 5, 3, 8},
{"addition avec zéro", 0, 5, 5},
{"addition de nombres négatifs", -2, -3, -5},

// ❌ Mauvais : non descriptif
{"test1", 5, 3, 8},
{"test2", 0, 5, 5},
{"test3", -2, -3, -5},
```

### 2. Ordre logique des cas
```go
tests := []struct{...}{
    // D'abord les cas normaux
    {"cas normal 1", ...},
    {"cas normal 2", ...},

    // Puis les cas limites
    {"avec zéro", ...},
    {"valeur maximale", ...},

    // Enfin les cas d'erreur
    {"entrée invalide", ...},
    {"erreur attendue", ...},
}
```

### 3. Structure claire
```go
tests := []struct {
    name     string  // Toujours en premier
    // Puis les inputs
    input1   type1
    input2   type2
    // Puis les outputs attendus
    expected type3
    wantErr  bool
}{...}
```

### 4. Gestion des erreurs
```go
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        result, err := Function(tt.input)

        if tt.wantErr {
            if err == nil {
                t.Error("Erreur attendue mais non reçue")
            }
            return
        }

        if err != nil {
            t.Errorf("Erreur inattendue: %v", err)
            return
        }

        // Vérifications du résultat...
    })
}
```

## Exercices pratiques

### Exercice 1 : Convertir en table-driven test
Convertissez ces tests répétitifs en table-driven test :

```go
func TestIsPositive_Positive(t *testing.T) {
    if !IsPositive(5) {
        t.Error("5 devrait être positif")
    }
}

func TestIsPositive_Negative(t *testing.T) {
    if IsPositive(-3) {
        t.Error("-3 ne devrait pas être positif")
    }
}

func TestIsPositive_Zero(t *testing.T) {
    if IsPositive(0) {
        t.Error("0 ne devrait pas être positif")
    }
}
```

### Exercice 2 : Test de calcul de pourcentage
Créez un table-driven test pour cette fonction :

```go
func CalculatePercentage(value, total float64) (float64, error) {
    if total == 0 {
        return 0, errors.New("total ne peut pas être zéro")
    }
    return (value / total) * 100, nil
}
```

Testez les cas : calcul normal, total zéro, valeur zéro, pourcentages > 100%.

### Exercice 3 : Validation de numéro de téléphone
Créez un table-driven test pour valider des formats de téléphone français :

```go
func IsValidFrenchPhone(phone string) bool {
    // Accepte: 01 23 45 67 89, 0123456789, +33123456789
    // Rejette: autres formats
}
```

## Récapitulatif

Les table-driven tests sont un **outil puissant** pour :
- **Réduire la duplication** dans vos tests
- **Améliorer la lisibilité** et l'organisation
- **Faciliter l'ajout** de nouveaux cas
- **Standardiser** la structure des tests

**Points clés à retenir :**
- Utilisez une struct pour définir les cas de test
- Nommez clairement chaque cas avec `name`
- Utilisez `t.Run()` pour exécuter chaque cas individuellement
- Organisez les cas par ordre logique (normal → limites → erreurs)
- Ajoutez `t.Parallel()` pour les tests lents
- Capturez les variables de boucle pour les tests parallèles

Dans la section suivante, nous découvrirons le **mocking et les interfaces**, essentiels pour tester du code avec des dépendances externes !

⏭️
