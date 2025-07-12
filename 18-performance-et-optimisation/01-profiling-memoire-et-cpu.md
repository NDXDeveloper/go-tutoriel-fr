🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18-1. Profiling mémoire et CPU

## Introduction

Le profiling est l'art de mesurer les performances de votre programme pour identifier où il passe le plus de temps (profiling CPU) et où il utilise le plus de mémoire (profiling mémoire). C'est comme avoir une radiographie de votre application en fonctionnement !

En Go, les outils de profiling sont intégrés au langage et très faciles à utiliser. Pas besoin d'outils externes compliqués.

## Qu'est-ce que le profiling ?

Imaginez que vous voulez savoir pourquoi votre voiture consomme trop d'essence. Vous pourriez :
- Mesurer la consommation sur autoroute vs en ville
- Vérifier si le moteur travaille plus dur dans certaines conditions
- Identifier les pièces qui consomment le plus d'énergie

Le profiling fait la même chose pour votre programme : il vous montre où votre code "consomme" le plus de ressources.

## Types de profiling en Go

### 1. Profiling CPU
- **Quoi** : Mesure où votre programme passe le plus de temps
- **Pourquoi** : Identifier les fonctions lentes
- **Quand** : Votre programme est lent à s'exécuter

### 2. Profiling mémoire
- **Quoi** : Mesure quelles parties allouent le plus de mémoire
- **Pourquoi** : Réduire l'utilisation mémoire et améliorer le GC
- **Quand** : Votre programme consomme trop de RAM

## Méthodes de profiling

### Méthode 1 : Profiling avec les tests (le plus simple)

C'est la méthode la plus facile pour débuter. Vous créez un test qui exécute votre code et Go génère automatiquement les profils.

#### Exemple pratique : Profiling d'une fonction de tri

```go
// main.go
package main

import (
    "math/rand"
    "sort"
    "time"
)

// Fonction simple qui trie des nombres
func sortNumbers(n int) []int {
    numbers := make([]int, n)
    for i := 0; i < n; i++ {
        numbers[i] = rand.Intn(1000)
    }
    sort.Ints(numbers)
    return numbers
}

// Fonction qui fait beaucoup d'allocations mémoire
func createManyStrings(n int) []string {
    var result []string
    for i := 0; i < n; i++ {
        // Attention : cette façon est inefficace !
        result = append(result, "string number "+string(rune(i)))
    }
    return result
}

func main() {
    rand.Seed(time.Now().UnixNano())
    sortNumbers(100000)
    createManyStrings(10000)
}
```

```go
// main_test.go
package main

import (
    "testing"
)

func BenchmarkSortNumbers(b *testing.B) {
    for i := 0; i < b.N; i++ {
        sortNumbers(10000)
    }
}

func BenchmarkCreateManyStrings(b *testing.B) {
    for i := 0; i < b.N; i++ {
        createManyStrings(1000)
    }
}
```

#### Générer les profils

```bash
# Profiling CPU
go test -bench=. -cpuprofile=cpu.prof

# Profiling mémoire
go test -bench=. -memprofile=mem.prof

# Les deux à la fois
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof
```

#### Analyser les résultats

```bash
# Analyser le profil CPU
go tool pprof cpu.prof

# Analyser le profil mémoire
go tool pprof mem.prof
```

### Méthode 2 : Profiling dans une application web

Pour une application web, vous pouvez activer le profiling en ajoutant quelques lignes :

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    _ "net/http/pprof" // Import pour activer le profiling
    "time"
)

func slowHandler(w http.ResponseWriter, r *http.Request) {
    // Simulation d'une opération lente
    time.Sleep(100 * time.Millisecond)

    // Simulation d'allocations mémoire
    data := make([]byte, 1024*1024) // 1MB
    for i := range data {
        data[i] = byte(i % 256)
    }

    fmt.Fprintf(w, "Hello, cette page a pris du temps à charger!")
}

func main() {
    http.HandleFunc("/slow", slowHandler)

    fmt.Println("Serveur démarré sur :8080")
    fmt.Println("Profiling disponible sur :8080/debug/pprof/")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Avec cette configuration, vous pouvez :

```bash
# Voir les profils disponibles
curl http://localhost:8080/debug/pprof/

# Générer un profil CPU de 30 secondes
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30

# Générer un profil mémoire
go tool pprof http://localhost:8080/debug/pprof/heap
```

## Utiliser pprof : Guide pas à pas

### Interface en ligne de commande

Quand vous exécutez `go tool pprof`, vous entrez dans un mode interactif :

```bash
$ go tool pprof cpu.prof
Type: cpu
Time: Dec 13, 2024 at 2:30pm (CET)
Duration: 2.05s, Total samples = 1.85s (90.24%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
```

#### Commandes essentielles

```bash
# Voir les 10 fonctions qui consomment le plus de CPU
(pprof) top10

# Voir les détails d'une fonction spécifique
(pprof) list main.sortNumbers

# Voir qui appelle une fonction
(pprof) peek sortNumbers

# Générer un graphique (nécessite Graphviz)
(pprof) png > profile.png

# Voir l'aide
(pprof) help
```

### Interface web (plus visuelle)

```bash
# Lancer l'interface web
go tool pprof -http=:8081 cpu.prof
```

Ouvrez votre navigateur sur `http://localhost:8081` pour voir :
- **Graph** : Graphique des appels de fonctions
- **Top** : Liste des fonctions les plus coûteuses
- **Flame Graph** : Visualisation en flamme (très utile !)

## Interpréter les résultats

### Profiling CPU

```bash
(pprof) top10
Showing nodes accounting for 1.23s, 66.49% of 1.85s total
Dropped 15 nodes (< 0.09s)
      flat  flat%   sum%        cum   cum%
     0.45s 24.32% 24.32%      0.45s 24.32%  runtime.memclrNoHeapPointers
     0.23s 12.43% 36.76%      0.23s 12.43%  sort.insertionSort
     0.21s 11.35% 48.11%      0.65s 35.14%  sort.quickSort
     0.18s  9.73% 57.84%      0.18s  9.73%  runtime.mallocgc
     0.16s  8.65% 66.49%      0.16s  8.65%  math/rand.Intn
```

**Explication :**
- **flat** : Temps passé dans cette fonction uniquement
- **flat%** : Pourcentage du temps total
- **sum%** : Pourcentage cumulé
- **cum** : Temps total incluant les fonctions appelées
- **cum%** : Pourcentage cumulé incluant les sous-fonctions

### Profiling mémoire

```bash
(pprof) top10
Showing nodes accounting for 512.19MB, 96.61% of 530.05MB total
Dropped 8 nodes (< 26.50MB)
      flat  flat%   sum%        cum   cum%
  256.06MB 48.31% 48.31%   256.06MB 48.31%  main.createManyStrings
  128.05MB 24.16% 72.47%   128.05MB 24.16%  main.sortNumbers
  128.08MB 24.14% 96.61%   128.08MB 24.14%  runtime.makeslice
```

**Explication :**
- **flat** : Mémoire allouée directement par cette fonction
- **cum** : Mémoire totale incluant les fonctions appelées

## Exemple complet : Optimiser une fonction lente

### Problème : Fonction lente

```go
// Version lente
func processData(data []string) map[string]int {
    result := make(map[string]int)
    for _, item := range data {
        // Opération coûteuse : conversion répétée
        key := strings.ToLower(strings.TrimSpace(item))
        result[key]++
    }
    return result
}
```

### Étape 1 : Mesurer

```go
func BenchmarkProcessData(b *testing.B) {
    data := make([]string, 1000)
    for i := range data {
        data[i] = fmt.Sprintf("  Item %d  ", i%100)
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        processData(data)
    }
}
```

```bash
go test -bench=BenchmarkProcessData -cpuprofile=cpu.prof
```

### Étape 2 : Analyser

```bash
go tool pprof cpu.prof
(pprof) top10
```

Vous verrez probablement que `strings.ToLower` et `strings.TrimSpace` consomment beaucoup de CPU.

### Étape 3 : Optimiser

```go
// Version optimisée
func processDataOptimized(data []string) map[string]int {
    result := make(map[string]int, len(data)/2) // Pré-allouer

    for _, item := range data {
        // Éviter les allocations répétées
        key := strings.ToLower(strings.TrimSpace(item))
        result[key]++
    }
    return result
}
```

### Étape 4 : Vérifier l'amélioration

```bash
go test -bench=. -cpuprofile=cpu_optimized.prof
```

## Profiling mémoire : Cas pratiques

### Détecter les fuites mémoire

```go
func memoryLeakExample() {
    // Simulation d'une fuite mémoire
    var leakySlice [][]byte

    for i := 0; i < 1000; i++ {
        // Chaque itération ajoute 1MB qui ne sera jamais libéré
        data := make([]byte, 1024*1024)
        leakySlice = append(leakySlice, data)
    }

    // La slice leakySlice garde des références, empêchant le GC
    // de libérer la mémoire
}
```

### Optimiser les allocations

```go
// Version qui alloue beaucoup
func inefficientStringBuilder(n int) string {
    var result string
    for i := 0; i < n; i++ {
        result += fmt.Sprintf("item-%d,", i) // Allocation à chaque itération !
    }
    return result
}

// Version optimisée
func efficientStringBuilder(n int) string {
    var builder strings.Builder
    builder.Grow(n * 10) // Pré-allouer la capacité

    for i := 0; i < n; i++ {
        builder.WriteString(fmt.Sprintf("item-%d,", i))
    }
    return builder.String()
}
```

## Outils complémentaires

### go tool trace

Pour analyser les goroutines et la concurrence :

```go
func BenchmarkConcurrentWork(b *testing.B) {
    for i := 0; i < b.N; i++ {
        concurrentWork()
    }
}
```

```bash
go test -bench=BenchmarkConcurrentWork -trace=trace.out
go tool trace trace.out
```

### Benchstat

Pour comparer les performances avant/après optimisation :

```bash
# Générer les benchmarks avant optimisation
go test -bench=. -count=10 > before.txt

# Après optimisation
go test -bench=. -count=10 > after.txt

# Comparer
go install golang.org/x/perf/cmd/benchstat@latest
benchstat before.txt after.txt
```

## Bonnes pratiques

### 1. Profiler régulièrement
- Intégrez le profiling dans votre workflow de développement
- Mesurez avant et après chaque optimisation

### 2. Profiler en conditions réelles
- Utilisez des données similaires à la production
- Testez avec la charge réelle

### 3. Ne pas sur-optimiser
- Concentrez-vous sur les vrais goulots d'étranglement
- 80% des problèmes viennent souvent de 20% du code

### 4. Garder les profils
- Conservez les profils pour comparer les évolutions
- Documentez vos optimisations

## Exercices pratiques

### Exercice 1 : Profiler une fonction simple

```go
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
```

**À faire :**
1. Créer un benchmark pour cette fonction
2. Générer un profil CPU
3. Identifier pourquoi elle est lente
4. Proposer une version optimisée

### Exercice 2 : Analyser les allocations mémoire

```go
func processSlice(data []int) []int {
    var result []int
    for _, v := range data {
        if v%2 == 0 {
            result = append(result, v*2)
        }
    }
    return result
}
```

**À faire :**
1. Créer un benchmark avec un grand slice
2. Générer un profil mémoire
3. Compter les allocations
4. Optimiser pour réduire les allocations

## Résumé

Le profiling est essentiel pour :
- **Identifier** les vrais problèmes de performance
- **Mesurer** l'impact des optimisations
- **Comprendre** le comportement de votre application

**Outils clés :**
- `go test -cpuprofile` et `-memprofile`
- `go tool pprof` avec interface CLI et web
- `net/http/pprof` pour les applications web

**Méthodologie :**
1. Mesurer d'abord
2. Identifier les goulots d'étranglement
3. Optimiser de manière ciblée
4. Valider l'amélioration

Dans le prochain chapitre, nous verrons les optimisations courantes que vous pouvez appliquer une fois que vous avez identifié les problèmes avec le profiling.

⏭️
