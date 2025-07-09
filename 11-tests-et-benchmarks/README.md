🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11. Tests et benchmarks

## Introduction

Les tests constituent un pilier fondamental du développement logiciel moderne, et Go excelle dans ce domaine en fournissant un écosystème de test intégré, simple et puissant. Contrairement à de nombreux langages qui nécessitent des frameworks externes complexes, Go intègre directement les outils de test dans sa bibliothèque standard, rendant l'écriture et l'exécution de tests naturelle et accessible.

## Pourquoi tester en Go ?

### Philosophie du test en Go

Go adopte une approche pragmatique des tests, privilégiant la simplicité et l'efficacité. Les créateurs du langage ont conçu un système de test qui encourage les bonnes pratiques sans imposer de complexité inutile. Cette philosophie se traduit par :

- **Simplicité avant tout** : Les tests sont écrits en Go standard, sans syntaxe particulière
- **Intégration native** : Pas besoin de frameworks externes pour commencer
- **Performance** : Les tests s'exécutent rapidement et peuvent être parallélisés
- **Lisibilité** : Les tests servent de documentation vivante du code

### Avantages du système de test Go

**1. Zéro configuration**
```go
// Pas de setup complexe, juste une fonction
func TestAddition(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Expected 5, got %d", result)
    }
}
```

**2. Outils intégrés**
- `go test` : Exécution des tests
- `go test -bench` : Benchmarks
- `go test -cover` : Couverture de code
- `go test -race` : Détection de race conditions

**3. Parallélisation native**
```go
func TestParallel(t *testing.T) {
    t.Parallel() // Ce test peut s'exécuter en parallèle
    // Test code...
}
```

## Types de tests en Go

### Tests unitaires
Les tests unitaires vérifient le comportement d'unités individuelles de code (fonctions, méthodes). Ils sont rapides, isolés et constituent la base de votre suite de tests.

### Tests d'intégration
Ces tests vérifient l'interaction entre différents composants du système. Ils sont plus lents mais permettent de détecter les problèmes d'intégration.

### Benchmarks
Les benchmarks mesurent les performances du code, permettant d'identifier les goulots d'étranglement et de valider les optimisations.

### Tests de propriétés (Property-based testing)
Bien que non intégré nativement, Go supporte les tests de propriétés via des bibliothèques tierces, permettant de générer automatiquement des cas de test.

## Structure des tests

### Convention de nommage
```
projet/
├── main.go
├── main_test.go
├── utils/
│   ├── helper.go
│   └── helper_test.go
└── models/
    ├── user.go
    └── user_test.go
```

### Règles de base
- Les fichiers de test se terminent par `_test.go`
- Les fonctions de test commencent par `Test`
- Les benchmarks commencent par `Benchmark`
- Les exemples commencent par `Example`

## Écosystème de test

### Package testing
Le package `testing` fournit les outils de base :
- `*testing.T` : Pour les tests unitaires
- `*testing.B` : Pour les benchmarks
- `*testing.M` : Pour les tests principaux (setup/teardown)

### Outils complémentaires
- **Testify** : Assertions plus expressives
- **GoMock** : Génération de mocks
- **Ginkgo/Gomega** : Framework BDD
- **Go-fuzz** : Fuzzing

## Métriques et qualité

### Couverture de code
```bash
go test -cover
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

### Détection de race conditions
```bash
go test -race
```

### Profiling
```bash
go test -cpuprofile=cpu.prof
go test -memprofile=mem.prof
```

## Bonnes pratiques préliminaires

### 1. Tests comme documentation
Vos tests doivent être suffisamment clairs pour servir de documentation sur le comportement attendu de votre code.

### 2. Isolation
Chaque test doit être indépendant et pouvoir s'exécuter dans n'importe quel ordre.

### 3. Nommage descriptif
```go
func TestUserValidation_WithInvalidEmail_ReturnsError(t *testing.T) {
    // Le nom décrit exactement ce qui est testé
}
```

### 4. Arrange, Act, Assert
```go
func TestCalculateTotal(t *testing.T) {
    // Arrange
    items := []Item{{Price: 10}, {Price: 20}}

    // Act
    total := CalculateTotal(items)

    // Assert
    if total != 30 {
        t.Errorf("Expected 30, got %d", total)
    }
}
```

## Intégration dans le workflow

### Automatisation
```bash
# Exécuter tous les tests
go test ./...

# Tests avec couverture
go test -cover ./...

# Tests en mode verbose
go test -v ./...

# Tests d'un package spécifique
go test ./pkg/utils
```

### CI/CD
Les tests Go s'intègrent parfaitement dans les pipelines CI/CD, permettant une validation automatique à chaque commit.

## Prochaines étapes

Dans les sections suivantes, nous explorerons en détail :
- L'écriture de tests unitaires efficaces
- Les table-driven tests pour réduire la duplication
- Les techniques de mocking et d'injection de dépendances
- L'optimisation des performances avec les benchmarks

Cette base solide vous permettra d'aborder chaque aspect du testing en Go avec confiance et de développer des applications robustes et maintenables.

⏭️
