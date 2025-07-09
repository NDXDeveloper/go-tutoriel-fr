🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11-4 : Benchmarks et profiling

## Qu'est-ce qu'un benchmark ?

Un **benchmark** est un test qui mesure les performances de votre code. Au lieu de vérifier si votre code fonctionne correctement (comme les tests unitaires), les benchmarks mesurent à quelle vitesse il s'exécute et combien de mémoire il utilise.

**Pourquoi faire des benchmarks ?**
- Identifier les parties lentes de votre code
- Comparer différentes implémentations
- Valider que vos optimisations fonctionnent
- Éviter les régressions de performance
- Prendre des décisions basées sur des données réelles

## Votre premier benchmark

### Étape 1 : Fonction à tester

Créons une fonction simple à benchmarker :

```go
// math.go
package main

import "strings"

// Fonction simple pour compter les voyelles
func CountVowels(text string) int {
    vowels := "aeiouAEIOU"
    count := 0

    for _, char := range text {
        if strings.ContainsRune(vowels, char) {
            count++
        }
    }

    return count
}

// Version optimisée
func CountVowelsOptimized(text string) int {
    count := 0

    for _, char := range text {
        switch char {
        case 'a', 'e', 'i', 'o', 'u', 'A', 'E', 'I', 'O', 'U':
            count++
        }
    }

    return count
}
```

### Étape 2 : Écrire le benchmark

Créez un fichier `math_bench_test.go` :

```go
package main

import (
    "strings"
    "testing"
)

// Votre premier benchmark !
func BenchmarkCountVowels(b *testing.B) {
    text := "Hello World! This is a test string with many vowels."

    // b.N est le nombre d'itérations que Go va exécuter
    for i := 0; i < b.N; i++ {
        CountVowels(text)
    }
}

func BenchmarkCountVowelsOptimized(b *testing.B) {
    text := "Hello World! This is a test string with many vowels."

    for i := 0; i < b.N; i++ {
        CountVowelsOptimized(text)
    }
}
```

### Étape 3 : Exécuter les benchmarks

```bash
go test -bench=.
```

Résultat typique :
```
BenchmarkCountVowels-8              2000000    750 ns/op
BenchmarkCountVowelsOptimized-8     5000000    320 ns/op
PASS
ok      your-package    3.456s
```

**Lecture des résultats :**
- `BenchmarkCountVowels-8` : nom du benchmark, `-8` = nombre de CPU
- `2000000` : nombre d'itérations exécutées
- `750 ns/op` : temps moyen par opération (nanosecondes)

## Anatomie d'un benchmark

### Structure de base

```go
func BenchmarkFunctionName(b *testing.B) {
    // 1. SETUP : Préparer les données (une seule fois)
    data := prepareTestData()

    // 2. RESET TIMER : Ignorer le temps de setup
    b.ResetTimer()

    // 3. BOUCLE DE BENCHMARK : Répéter b.N fois
    for i := 0; i < b.N; i++ {
        // Code à mesurer
        FunctionToTest(data)
    }
}
```

### Règles importantes

1. **Nom du fichier** : doit finir par `_test.go`
2. **Nom de la fonction** : doit commencer par `Benchmark`
3. **Paramètre** : toujours `b *testing.B`
4. **Boucle obligatoire** : `for i := 0; i < b.N; i++`

## Exemples pratiques de benchmarks

### Benchmark avec setup

```go
func BenchmarkStringConcatenation(b *testing.B) {
    // Setup : créer des données de test
    words := []string{"Hello", "World", "This", "Is", "A", "Test"}

    // Reset le timer pour ignorer le temps de setup
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        var result string
        for _, word := range words {
            result += word + " "
        }
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    words := []string{"Hello", "World", "This", "Is", "A", "Test"}

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        var builder strings.Builder
        for _, word := range words {
            builder.WriteString(word)
            builder.WriteString(" ")
        }
        _ = builder.String()
    }
}

func BenchmarkStringJoin(b *testing.B) {
    words := []string{"Hello", "World", "This", "Is", "A", "Test"}

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        _ = strings.Join(words, " ")
    }
}
```

### Benchmark avec différentes tailles de données

```go
func BenchmarkSort10(b *testing.B) {
    benchmarkSort(b, 10)
}

func BenchmarkSort100(b *testing.B) {
    benchmarkSort(b, 100)
}

func BenchmarkSort1000(b *testing.B) {
    benchmarkSort(b, 1000)
}

func BenchmarkSort10000(b *testing.B) {
    benchmarkSort(b, 10000)
}

// Fonction helper pour éviter la duplication
func benchmarkSort(b *testing.B, size int) {
    for i := 0; i < b.N; i++ {
        // Stop le timer pendant la génération de données
        b.StopTimer()
        data := generateRandomSlice(size)
        b.StartTimer()

        // Code à mesurer
        sort.Ints(data)
    }
}

func generateRandomSlice(size int) []int {
    slice := make([]int, size)
    for i := range slice {
        slice[i] = rand.Intn(1000)
    }
    return slice
}
```

### Benchmark table-driven

```go
func BenchmarkStringOperations(b *testing.B) {
    tests := []struct {
        name string
        text string
        fn   func(string) string
    }{
        {"ToUpper_Short", "hello", strings.ToUpper},
        {"ToUpper_Long", strings.Repeat("hello world ", 100), strings.ToUpper},
        {"ToLower_Short", "HELLO", strings.ToLower},
        {"ToLower_Long", strings.Repeat("HELLO WORLD ", 100), strings.ToLower},
    }

    for _, tt := range tests {
        b.Run(tt.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                tt.fn(tt.text)
            }
        })
    }
}
```

## Mesures de mémoire

### Benchmark avec allocation de mémoire

```go
func BenchmarkSliceAppend(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var slice []int
        for j := 0; j < 1000; j++ {
            slice = append(slice, j)
        }
    }
}

func BenchmarkSlicePrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        slice := make([]int, 0, 1000) // Pré-allouer la capacité
        for j := 0; j < 1000; j++ {
            slice = append(slice, j)
        }
    }
}
```

### Exécuter avec mesures de mémoire

```bash
go test -bench=. -benchmem
```

Résultat avec mémoire :
```
BenchmarkSliceAppend-8      20000    85432 ns/op    40960 B/op    10 allocs/op
BenchmarkSlicePrealloc-8    50000    25678 ns/op     8192 B/op     1 allocs/op
```

**Lecture des résultats mémoire :**
- `40960 B/op` : bytes alloués par opération
- `10 allocs/op` : nombre d'allocations par opération

## Benchmark d'algorithmes de tri

```go
// Différents algorithmes de tri à comparer
func bubbleSort(arr []int) {
    n := len(arr)
    for i := 0; i < n; i++ {
        for j := 0; j < n-i-1; j++ {
            if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
            }
        }
    }
}

func quickSort(arr []int) {
    if len(arr) < 2 {
        return
    }

    pivot := partition(arr)
    quickSort(arr[:pivot])
    quickSort(arr[pivot+1:])
}

func partition(arr []int) int {
    pivot := arr[len(arr)-1]
    i := -1

    for j := 0; j < len(arr)-1; j++ {
        if arr[j] <= pivot {
            i++
            arr[i], arr[j] = arr[j], arr[i]
        }
    }

    arr[i+1], arr[len(arr)-1] = arr[len(arr)-1], arr[i+1]
    return i + 1
}

// Benchmarks pour comparer les algorithmes
func BenchmarkBubbleSort(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomSlice(100)
        b.StartTimer()

        bubbleSort(data)
    }
}

func BenchmarkQuickSort(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomSlice(100)
        b.StartTimer()

        quickSort(data)
    }
}

func BenchmarkStandardSort(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := generateRandomSlice(100)
        b.StartTimer()

        sort.Ints(data)
    }
}
```

## Profiling avancé

### Générer des profils de performance

```bash
# Profil CPU
go test -bench=. -cpuprofile=cpu.prof

# Profil mémoire
go test -bench=. -memprofile=mem.prof

# Profil de blocage
go test -bench=. -blockprofile=block.prof
```

### Analyser les profils

```bash
# Analyser le profil CPU
go tool pprof cpu.prof

# Analyser le profil mémoire
go tool pprof mem.prof
```

Dans l'interface pprof :
```
(pprof) top10     # Top 10 des fonctions les plus coûteuses
(pprof) list main.CountVowels  # Code source avec annotations
(pprof) web       # Graphique dans le navigateur
(pprof) help      # Aide
```

### Benchmark avec profiling intégré

```go
func BenchmarkWithProfiling(b *testing.B) {
    // Créer beaucoup de données pour voir l'impact
    data := strings.Repeat("Hello World with many vowels! ", 1000)

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        CountVowels(data)
    }
}
```

## Optimisation basée sur les benchmarks

### Exemple : Optimisation d'une fonction de recherche

```go
// Version naïve
func ContainsWordNaive(text, word string) bool {
    words := strings.Split(text, " ")
    for _, w := range words {
        if w == word {
            return true
        }
    }
    return false
}

// Version optimisée
func ContainsWordOptimized(text, word string) bool {
    return strings.Contains(" "+text+" ", " "+word+" ")
}

// Version avec regex (pour comparaison)
func ContainsWordRegex(text, word string) bool {
    pattern := `\b` + regexp.QuoteMeta(word) + `\b`
    matched, _ := regexp.MatchString(pattern, text)
    return matched
}

// Benchmarks pour comparer
func BenchmarkContainsWordNaive(b *testing.B) {
    text := "The quick brown fox jumps over the lazy dog"
    word := "fox"

    for i := 0; i < b.N; i++ {
        ContainsWordNaive(text, word)
    }
}

func BenchmarkContainsWordOptimized(b *testing.B) {
    text := "The quick brown fox jumps over the lazy dog"
    word := "fox"

    for i := 0; i < b.N; i++ {
        ContainsWordOptimized(text, word)
    }
}

func BenchmarkContainsWordRegex(b *testing.B) {
    text := "The quick brown fox jumps over the lazy dog"
    word := "fox"

    for i := 0; i < b.N; i++ {
        ContainsWordRegex(text, word)
    }
}
```

## Comparaison de benchmarks

### Utiliser benchcmp pour comparer

```bash
# Faire des benchmarks avant optimisation
go test -bench=. > before.txt

# Optimiser le code...

# Faire des benchmarks après optimisation
go test -bench=. > after.txt

# Comparer les résultats
benchcmp before.txt after.txt
```

### Benchmark continu

```go
// Script pour surveiller les performances
func BenchmarkCriticalFunction(b *testing.B) {
    // Fonction critique de votre application
    for i := 0; i < b.N; i++ {
        CriticalFunction()
    }

    // Définir un seuil de performance
    if testing.Short() {
        return
    }

    // Vérifier que les performances restent acceptables
    result := testing.Benchmark(BenchmarkCriticalFunction)
    nsPerOp := result.NsPerOp()

    // Seuil : pas plus de 1000 ns par opération
    if nsPerOp > 1000 {
        b.Fatalf("Performance dégradée: %d ns/op > 1000 ns/op", nsPerOp)
    }
}
```

## Commandes utiles

### Commandes de base

```bash
# Exécuter tous les benchmarks
go test -bench=.

# Exécuter un benchmark spécifique
go test -bench=BenchmarkCountVowels

# Benchmarks avec mesures de mémoire
go test -bench=. -benchmem

# Benchmarks en mode verbose
go test -bench=. -v

# Exécuter plusieurs fois pour plus de précision
go test -bench=. -count=5
```

### Commandes avancées

```bash
# Contrôler la durée minimale
go test -bench=. -benchtime=10s

# Contrôler le nombre d'itérations
go test -bench=. -benchtime=1000000x

# Générer tous les profils
go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof -blockprofile=block.prof

# Benchmark avec race detection
go test -bench=. -race
```

## Bonnes pratiques

### 1. Setup et cleanup

```go
func BenchmarkWithSetup(b *testing.B) {
    // Setup coûteux fait une seule fois
    largeData := generateLargeDataset()

    // Reset le timer pour ignorer le setup
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        // Code à mesurer
        ProcessData(largeData)
    }

    // Cleanup si nécessaire
    b.StopTimer()
    cleanup(largeData)
}
```

### 2. Éviter les optimisations du compilateur

```go
var result int // Variable globale pour éviter l'optimisation

func BenchmarkFunction(b *testing.B) {
    var r int

    for i := 0; i < b.N; i++ {
        r = ExpensiveFunction()
    }

    result = r // Assigner à une variable globale
}
```

### 3. Benchmark de sous-fonctions

```go
func BenchmarkFullWorkflow(b *testing.B) {
    for i := 0; i < b.N; i++ {
        data := LoadData()
        processed := ProcessData(data)
        SaveData(processed)
    }
}

func BenchmarkLoadData(b *testing.B) {
    for i := 0; i < b.N; i++ {
        LoadData()
    }
}

func BenchmarkProcessData(b *testing.B) {
    data := LoadData()
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        ProcessData(data)
    }
}
```

### 4. Benchmark avec différentes conditions

```go
func BenchmarkMapAccess(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("Size%d", size), func(b *testing.B) {
            m := make(map[int]string, size)
            for i := 0; i < size; i++ {
                m[i] = fmt.Sprintf("value%d", i)
            }

            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                _ = m[rand.Intn(size)]
            }
        })
    }
}
```

## Exercices pratiques

### Exercice 1 : Benchmark de recherche
Créez des benchmarks pour comparer ces trois méthodes de recherche :

```go
func LinearSearch(slice []int, target int) int {
    for i, v := range slice {
        if v == target {
            return i
        }
    }
    return -1
}

func BinarySearch(slice []int, target int) int {
    left, right := 0, len(slice)-1

    for left <= right {
        mid := (left + right) / 2
        if slice[mid] == target {
            return mid
        } else if slice[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}

func MapSearch(m map[int]int, target int) int {
    if index, exists := m[target]; exists {
        return index
    }
    return -1
}
```

### Exercice 2 : Optimisation de concaténation
Benchmarkez et optimisez cette fonction :

```go
func BuildHTMLTable(rows [][]string) string {
    html := "<table>"

    for _, row := range rows {
        html += "<tr>"
        for _, cell := range row {
            html += "<td>" + cell + "</td>"
        }
        html += "</tr>"
    }

    html += "</table>"
    return html
}
```

## Récapitulatif

Les **benchmarks et le profiling** permettent de :
- **Mesurer** les performances de votre code
- **Identifier** les goulots d'étranglement
- **Comparer** différentes implémentations
- **Valider** les optimisations
- **Éviter** les régressions de performance

**Points clés à retenir :**
- Les benchmarks mesurent le temps et la mémoire
- Utilisez `b.ResetTimer()` pour ignorer le setup
- Toujours benchmarker avant d'optimiser
- Utilisez `go test -benchmem` pour la mémoire
- Le profiling aide à identifier les problèmes
- Comparez les résultats avec benchcmp
- Évitez les optimisations prématurées

**Commandes essentielles :**
- `go test -bench=.` : exécuter les benchmarks
- `go test -bench=. -benchmem` : avec mesures mémoire
- `go test -bench=. -cpuprofile=cpu.prof` : profiling CPU
- `go tool pprof cpu.prof` : analyser les profils

Les benchmarks sont essentiels pour créer des applications Go performantes et maintenir leur performance dans le temps !

⏭️

# Solutions des exercices pratiques - Benchmarks et profiling

## Exercice 1 : Benchmark de recherche

### Code complet avec fonctions de recherche

```go
package main

import (
    "math/rand"
    "sort"
    "testing"
    "time"
)

// Fonctions de recherche à benchmarker
func LinearSearch(slice []int, target int) int {
    for i, v := range slice {
        if v == target {
            return i
        }
    }
    return -1
}

func BinarySearch(slice []int, target int) int {
    left, right := 0, len(slice)-1

    for left <= right {
        mid := (left + right) / 2
        if slice[mid] == target {
            return mid
        } else if slice[mid] < target {
            left = mid + 1
        } else {
            right = mid - 1
        }
    }
    return -1
}

func MapSearch(m map[int]int, target int) int {
    if index, exists := m[target]; exists {
        return index
    }
    return -1
}

// Fonctions utilitaires pour les benchmarks
func generateSortedSlice(size int) []int {
    slice := make([]int, size)
    for i := range slice {
        slice[i] = i * 2 // Valeurs paires pour éviter les doublons
    }
    return slice
}

func generateRandomSlice(size int) []int {
    rand.Seed(time.Now().UnixNano())
    slice := make([]int, size)
    for i := range slice {
        slice[i] = rand.Intn(size * 2)
    }
    return slice
}

func sliceToMap(slice []int) map[int]int {
    m := make(map[int]int, len(slice))
    for i, v := range slice {
        m[v] = i
    }
    return m
}
```

### Benchmarks de base pour chaque méthode

```go
// Benchmarks avec différentes tailles de données
func BenchmarkLinearSearch_Small(b *testing.B) {
    benchmarkLinearSearch(b, 100)
}

func BenchmarkLinearSearch_Medium(b *testing.B) {
    benchmarkLinearSearch(b, 1000)
}

func BenchmarkLinearSearch_Large(b *testing.B) {
    benchmarkLinearSearch(b, 10000)
}

func BenchmarkLinearSearch_XLarge(b *testing.B) {
    benchmarkLinearSearch(b, 100000)
}

func benchmarkLinearSearch(b *testing.B, size int) {
    // Setup : créer des données de test
    slice := generateSortedSlice(size)
    target := slice[size/2] // Chercher au milieu (cas moyen)

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        LinearSearch(slice, target)
    }
}

// Benchmarks pour Binary Search
func BenchmarkBinarySearch_Small(b *testing.B) {
    benchmarkBinarySearch(b, 100)
}

func BenchmarkBinarySearch_Medium(b *testing.B) {
    benchmarkBinarySearch(b, 1000)
}

func BenchmarkBinarySearch_Large(b *testing.B) {
    benchmarkBinarySearch(b, 10000)
}

func BenchmarkBinarySearch_XLarge(b *testing.B) {
    benchmarkBinarySearch(b, 100000)
}

func benchmarkBinarySearch(b *testing.B, size int) {
    slice := generateSortedSlice(size)
    target := slice[size/2]

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        BinarySearch(slice, target)
    }
}

// Benchmarks pour Map Search
func BenchmarkMapSearch_Small(b *testing.B) {
    benchmarkMapSearch(b, 100)
}

func BenchmarkMapSearch_Medium(b *testing.B) {
    benchmarkMapSearch(b, 1000)
}

func BenchmarkMapSearch_Large(b *testing.B) {
    benchmarkMapSearch(b, 10000)
}

func BenchmarkMapSearch_XLarge(b *testing.B) {
    benchmarkMapSearch(b, 100000)
}

func benchmarkMapSearch(b *testing.B, size int) {
    slice := generateSortedSlice(size)
    m := sliceToMap(slice)
    target := slice[size/2]

    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        MapSearch(m, target)
    }
}
```

### Benchmarks table-driven pour comparaison directe

```go
func BenchmarkSearchMethods(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}

    for _, size := range sizes {
        // Préparer les données pour tous les tests
        slice := generateSortedSlice(size)
        m := sliceToMap(slice)
        target := slice[size/2]

        b.Run(fmt.Sprintf("Linear_%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                LinearSearch(slice, target)
            }
        })

        b.Run(fmt.Sprintf("Binary_%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BinarySearch(slice, target)
            }
        })

        b.Run(fmt.Sprintf("Map_%d", size), func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                MapSearch(m, target)
            }
        })
    }
}
```

### Benchmarks avec différents scénarios de recherche

```go
func BenchmarkSearchScenarios(b *testing.B) {
    size := 10000
    slice := generateSortedSlice(size)
    m := sliceToMap(slice)

    scenarios := []struct {
        name   string
        target int
        desc   string
    }{
        {"FirstElement", slice[0], "Meilleur cas pour linear search"},
        {"MiddleElement", slice[size/2], "Cas moyen"},
        {"LastElement", slice[size-1], "Pire cas pour linear search"},
        {"NotFound", -1, "Élément inexistant"},
    }

    for _, scenario := range scenarios {
        b.Run("Linear_"+scenario.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                LinearSearch(slice, scenario.target)
            }
        })

        b.Run("Binary_"+scenario.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BinarySearch(slice, scenario.target)
            }
        })

        b.Run("Map_"+scenario.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                MapSearch(m, scenario.target)
            }
        })
    }
}
```

### Benchmark avec mesures de mémoire

```go
func BenchmarkSearchWithMemory(b *testing.B) {
    size := 10000

    b.Run("LinearSearch", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            b.StopTimer()
            slice := generateSortedSlice(size)
            target := slice[size/2]
            b.StartTimer()

            LinearSearch(slice, target)
        }
    })

    b.Run("BinarySearch", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            b.StopTimer()
            slice := generateSortedSlice(size)
            target := slice[size/2]
            b.StartTimer()

            BinarySearch(slice, target)
        }
    })

    b.Run("MapSearch_WithCreation", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            b.StopTimer()
            slice := generateSortedSlice(size)
            target := slice[size/2]
            b.StartTimer()

            m := sliceToMap(slice) // Inclure la création de la map
            MapSearch(m, target)
        }
    })

    b.Run("MapSearch_PreCreated", func(b *testing.B) {
        slice := generateSortedSlice(size)
        m := sliceToMap(slice)
        target := slice[size/2]

        b.ResetTimer()

        for i := 0; i < b.N; i++ {
            MapSearch(m, target)
        }
    })
}
```

### Résultats attendus et analyse

```bash
# Exécuter les benchmarks
go test -bench=BenchmarkSearchMethods -benchmem

# Résultats typiques :
# BenchmarkSearchMethods/Linear_100-8         	 2000000	   550 ns/op
# BenchmarkSearchMethods/Binary_100-8         	20000000	    85 ns/op
# BenchmarkSearchMethods/Map_100-8            	50000000	    25 ns/op
# BenchmarkSearchMethods/Linear_10000-8       	   20000	 55000 ns/op
# BenchmarkSearchMethods/Binary_10000-8       	10000000	   120 ns/op
# BenchmarkSearchMethods/Map_10000-8          	50000000	    25 ns/op
```

**Analyse des résultats :**
- **Linear Search** : O(n) - performance dégradée avec la taille
- **Binary Search** : O(log n) - performance stable même sur grandes données
- **Map Search** : O(1) - performance constante, la plus rapide

## Exercice 2 : Optimisation de concaténation

### Version originale (problématique)

```go
package main

import (
    "fmt"
    "strings"
    "testing"
)

// Version originale - inefficace
func BuildHTMLTable(rows [][]string) string {
    html := "<table>"

    for _, row := range rows {
        html += "<tr>"
        for _, cell := range row {
            html += "<td>" + cell + "</td>"
        }
        html += "</tr>"
    }

    html += "</table>"
    return html
}
```

### Versions optimisées

```go
// Version 1 : avec strings.Builder (recommandée)
func BuildHTMLTableBuilder(rows [][]string) string {
    var builder strings.Builder

    builder.WriteString("<table>")

    for _, row := range rows {
        builder.WriteString("<tr>")
        for _, cell := range row {
            builder.WriteString("<td>")
            builder.WriteString(cell)
            builder.WriteString("</td>")
        }
        builder.WriteString("</tr>")
    }

    builder.WriteString("</table>")
    return builder.String()
}

// Version 2 : avec strings.Builder et pré-allocation
func BuildHTMLTableBuilderPrealloc(rows [][]string) string {
    // Estimer la taille nécessaire
    totalCells := 0
    totalChars := 0
    for _, row := range rows {
        totalCells += len(row)
        for _, cell := range row {
            totalChars += len(cell)
        }
    }

    // Estimation: balises + contenu
    estimatedSize := totalChars + totalCells*9 + len(rows)*9 + 15 // 9 chars pour <td></td>, etc.

    var builder strings.Builder
    builder.Grow(estimatedSize) // Pré-allouer la mémoire

    builder.WriteString("<table>")

    for _, row := range rows {
        builder.WriteString("<tr>")
        for _, cell := range row {
            builder.WriteString("<td>")
            builder.WriteString(cell)
            builder.WriteString("</td>")
        }
        builder.WriteString("</tr>")
    }

    builder.WriteString("</table>")
    return builder.String()
}

// Version 3 : avec slice et join
func BuildHTMLTableSlice(rows [][]string) string {
    parts := make([]string, 0, len(rows)*10) // Estimation du nombre de parties

    parts = append(parts, "<table>")

    for _, row := range rows {
        parts = append(parts, "<tr>")
        for _, cell := range row {
            parts = append(parts, "<td>", cell, "</td>")
        }
        parts = append(parts, "</tr>")
    }

    parts = append(parts, "</table>")

    return strings.Join(parts, "")
}

// Version 4 : avec fmt.Sprintf (pour comparaison)
func BuildHTMLTableSprintf(rows [][]string) string {
    var parts []string

    parts = append(parts, "<table>")

    for _, row := range rows {
        parts = append(parts, "<tr>")
        for _, cell := range row {
            parts = append(parts, fmt.Sprintf("<td>%s</td>", cell))
        }
        parts = append(parts, "</tr>")
    }

    parts = append(parts, "</table>")

    return strings.Join(parts, "")
}

// Version 5 : avec bytes.Buffer (alternative)
func BuildHTMLTableBuffer(rows [][]string) string {
    var buffer bytes.Buffer

    buffer.WriteString("<table>")

    for _, row := range rows {
        buffer.WriteString("<tr>")
        for _, cell := range row {
            buffer.WriteString("<td>")
            buffer.WriteString(cell)
            buffer.WriteString("</td>")
        }
        buffer.WriteString("</tr>")
    }

    buffer.WriteString("</table>")
    return buffer.String()
}
```

### Fonction utilitaire pour générer des données de test

```go
func generateTestData(rows, cols int) [][]string {
    data := make([][]string, rows)
    for i := range data {
        data[i] = make([]string, cols)
        for j := range data[i] {
            data[i][j] = fmt.Sprintf("Cell_%d_%d", i, j)
        }
    }
    return data
}
```

### Benchmarks complets

```go
func BenchmarkHTMLTable_Original_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTable(data)
    }
}

func BenchmarkHTMLTable_Builder_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTableBuilder(data)
    }
}

func BenchmarkHTMLTable_BuilderPrealloc_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTableBuilderPrealloc(data)
    }
}

func BenchmarkHTMLTable_Slice_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTableSlice(data)
    }
}

func BenchmarkHTMLTable_Sprintf_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTableSprintf(data)
    }
}

func BenchmarkHTMLTable_Buffer_Small(b *testing.B) {
    data := generateTestData(10, 5)

    for i := 0; i < b.N; i++ {
        BuildHTMLTableBuffer(data)
    }
}

// Benchmarks avec différentes tailles
func BenchmarkHTMLTableComparison(b *testing.B) {
    sizes := []struct {
        name string
        rows int
        cols int
    }{
        {"Small_10x5", 10, 5},
        {"Medium_50x10", 50, 10},
        {"Large_100x20", 100, 20},
        {"XLarge_500x10", 500, 10},
    }

    for _, size := range sizes {
        data := generateTestData(size.rows, size.cols)

        b.Run("Original_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTable(data)
            }
        })

        b.Run("Builder_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTableBuilder(data)
            }
        })

        b.Run("BuilderPrealloc_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTableBuilderPrealloc(data)
            }
        })

        b.Run("Slice_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTableSlice(data)
            }
        })

        b.Run("Sprintf_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTableSprintf(data)
            }
        })

        b.Run("Buffer_"+size.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                BuildHTMLTableBuffer(data)
            }
        })
    }
}
```

### Benchmark avec mesures de mémoire détaillées

```go
func BenchmarkHTMLTableMemory(b *testing.B) {
    data := generateTestData(100, 20) // Données assez grandes pour voir l'impact

    b.Run("Original", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            BuildHTMLTable(data)
        }
    })

    b.Run("Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            BuildHTMLTableBuilder(data)
        }
    })

    b.Run("BuilderPrealloc", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            BuildHTMLTableBuilderPrealloc(data)
        }
    })

    b.Run("Slice", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            BuildHTMLTableSlice(data)
        }
    })

    b.Run("Buffer", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            BuildHTMLTableBuffer(data)
        }
    })
}
```

### Tests de validation (pour vérifier que toutes les versions donnent le même résultat)

```go
func TestHTMLTableImplementations(t *testing.T) {
    data := generateTestData(3, 3)

    original := BuildHTMLTable(data)
    builder := BuildHTMLTableBuilder(data)
    builderPrealloc := BuildHTMLTableBuilderPrealloc(data)
    slice := BuildHTMLTableSlice(data)
    sprintf := BuildHTMLTableSprintf(data)
    buffer := BuildHTMLTableBuffer(data)

    implementations := map[string]string{
        "Builder":        builder,
        "BuilderPrealloc": builderPrealloc,
        "Slice":          slice,
        "Sprintf":        sprintf,
        "Buffer":         buffer,
    }

    for name, result := range implementations {
        if result != original {
            t.Errorf("%s donne un résultat différent", name)
            t.Logf("Original: %s", original)
            t.Logf("%s: %s", name, result)
        }
    }
}
```

### Résultats attendus et analyse

```bash
# Exécuter avec mesures de mémoire
go test -bench=BenchmarkHTMLTableMemory -benchmem

# Résultats typiques pour données 100x20 :
# BenchmarkHTMLTableMemory/Original-8              100  12000000 ns/op  45000000 B/op  2000 allocs/op
# BenchmarkHTMLTableMemory/Builder-8              2000    600000 ns/op    120000 B/op     8 allocs/op
# BenchmarkHTMLTableMemory/BuilderPrealloc-8      3000    400000 ns/op     80000 B/op     2 allocs/op
# BenchmarkHTMLTableMemory/Slice-8                1500    800000 ns/op    200000 B/op    15 allocs/op
# BenchmarkHTMLTableMemory/Buffer-8               2000    650000 ns/op    130000 B/op    10 allocs/op
```

**Analyse des performances :**

1. **Version originale** (concaténation string) :
   - ❌ Très lente : chaque += crée une nouvelle string
   - ❌ Beaucoup d'allocations : O(n²) en mémoire
   - ❌ Plus les données sont grandes, plus c'est dramatique

2. **strings.Builder** :
   - ✅ 20x plus rapide que l'original
   - ✅ Beaucoup moins d'allocations
   - ✅ Conçu spécifiquement pour ce cas d'usage

3. **strings.Builder avec pré-allocation** :
   - ✅ La plus rapide
   - ✅ Le moins d'allocations (souvent juste 1-2)
   - ✅ Idéal quand on peut estimer la taille

4. **Slice + Join** :
   - ✅ Plus rapide que l'original
   - ⚠️ Un peu plus d'allocations que Builder
   - ✅ Lisible et facile à comprendre

5. **fmt.Sprintf** :
   - ❌ Plus lent à cause du parsing des formats
   - ⚠️ Plus d'allocations
   - ⚠️ À éviter pour la performance

6. **bytes.Buffer** :
   - ✅ Performance similaire à strings.Builder
   - ✅ Alternative valide
   - ⚠️ Légèrement plus verbeux

### Recommandations

```go
// ✅ Recommandation 1 : strings.Builder avec pré-allocation
func BuildHTMLTableRecommended(rows [][]string) string {
    var builder strings.Builder

    // Estimation intelligente de la taille
    totalCells := 0
    totalChars := 0
    for _, row := range rows {
        totalCells += len(row)
        for _, cell := range row {
            totalChars += len(cell)
        }
    }
    estimatedSize := totalChars + totalCells*9 + len(rows)*9 + 15
    builder.Grow(estimatedSize)

    builder.WriteString("<table>")
    for _, row := range rows {
        builder.WriteString("<tr>")
        for _, cell := range row {
            builder.WriteString("<td>")
            builder.WriteString(cell)
            builder.WriteString("</td>")
        }
        builder.WriteString("</tr>")
    }
    builder.WriteString("</table>")

    return builder.String()
}

// ✅ Recommandation 2 : Version simple avec strings.Builder
func BuildHTMLTableSimple(rows [][]string) string {
    var builder strings.Builder

    builder.WriteString("<table>")
    for _, row := range rows {
        builder.WriteString("<tr>")
        for _, cell := range row {
            builder.WriteString("<td>")
            builder.WriteString(cell)
            builder.WriteString("</td>")
        }
        builder.WriteString("</tr>")
    }
    builder.WriteString("</table>")

    return builder.String()
}
```

## Récapitulatif des solutions

### **Exercice 1 - Recherche :**
- **Linear Search** : O(n) - bon pour petites données, simple
- **Binary Search** : O(log n) - excellent pour données triées
- **Map Search** : O(1) - le plus rapide, mais nécessite préparation

### **Exercice 2 - Concaténation :**
- **strings.Builder** : solution recommandée (20x plus rapide)
- **Pré-allocation** : optimisation supplémentaire significative
- **Éviter** : concaténation directe de strings (très inefficace)

### **Leçons importantes :**
1. **Mesurer avant d'optimiser** : les benchmarks révèlent la vérité
2. **O(n²) est à éviter** : la concaténation string crée ce problème
3. **La pré-allocation aide** : connaître la taille cible améliore les performances
4. **Go fournit les bons outils** : strings.Builder est conçu pour ces cas
5. **Tester la correction** : s'assurer que l'optimisation ne casse rien

Ces exercices montrent l'importance des benchmarks pour identifier les problèmes de performance et valider les optimisations !

⏭️
