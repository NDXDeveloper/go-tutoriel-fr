🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18-2. Optimisations courantes

## Introduction

Maintenant que vous savez utiliser le profiling pour identifier les problèmes de performance, apprenons les techniques d'optimisation les plus courantes en Go. Ces optimisations sont comme des "recettes" que vous pouvez appliquer dans vos programmes.

Pensez à ces optimisations comme à des conseils pour conduire de manière plus économique : éviter les freinages brusques, maintenir une vitesse constante, bien gonfler les pneus... Chaque petite amélioration contribue à un meilleur résultat global.

## Principe général : Optimiser intelligemment

### La règle des 80/20
80% des problèmes de performance viennent de 20% du code. Concentrez-vous sur les parties qui comptent vraiment !

### Ordre d'optimisation (par impact)
1. **Algorithmes** : Choisir le bon algorithme (O(n) vs O(n²))
2. **Structures de données** : Utiliser les bonnes structures
3. **Allocations mémoire** : Réduire les allocations inutiles
4. **I/O** : Optimiser les lectures/écritures
5. **Micro-optimisations** : Petites améliorations de code

## 1. Optimisations algorithmiques

### Choisir la bonne complexité

```go
// ❌ Mauvais : O(n²) - Lent pour de grandes données
func findDuplicatesSlow(data []string) []string {
    var duplicates []string
    for i := 0; i < len(data); i++ {
        for j := i + 1; j < len(data); j++ {
            if data[i] == data[j] {
                duplicates = append(duplicates, data[i])
                break
            }
        }
    }
    return duplicates
}

// ✅ Bon : O(n) - Rapide même avec beaucoup de données
func findDuplicatesFast(data []string) []string {
    seen := make(map[string]bool)
    var duplicates []string

    for _, item := range data {
        if seen[item] {
            duplicates = append(duplicates, item)
        } else {
            seen[item] = true
        }
    }
    return duplicates
}
```

### Éviter les calculs redondants

```go
// ❌ Mauvais : Calcule la longueur à chaque itération
func processItemsSlow(items []string) {
    for i := 0; i < len(items); i++ { // len(items) appelé à chaque fois !
        fmt.Println(items[i])
    }
}

// ✅ Bon : Calcule la longueur une seule fois
func processItemsFast(items []string) {
    length := len(items) // Calculé une seule fois
    for i := 0; i < length; i++ {
        fmt.Println(items[i])
    }
}

// ✅ Encore mieux : Utilise range (plus idiomatique en Go)
func processItemsBest(items []string) {
    for _, item := range items {
        fmt.Println(item)
    }
}
```

## 2. Optimisations des structures de données

### Choisir la bonne structure

```go
// Pour chercher des éléments
type UserStore struct {
    // ❌ Mauvais : Recherche O(n)
    users []User

    // ✅ Bon : Recherche O(1)
    usersByID map[int]User
    usersByEmail map[string]User
}

// Pour des données ordonnées
// ❌ Mauvais : Tri à chaque recherche
func findInSlice(data []int, target int) bool {
    sort.Ints(data) // Tri coûteux !
    i := sort.SearchInts(data, target)
    return i < len(data) && data[i] == target
}

// ✅ Bon : Tri une seule fois
type SortedData struct {
    data []int
    sorted bool
}

func (s *SortedData) Add(value int) {
    s.data = append(s.data, value)
    s.sorted = false
}

func (s *SortedData) Contains(target int) bool {
    if !s.sorted {
        sort.Ints(s.data)
        s.sorted = true
    }
    i := sort.SearchInts(s.data, target)
    return i < len(s.data) && s.data[i] == target
}
```

### Pré-allouer les slices

```go
// ❌ Mauvais : Réallocations multiples
func createNumbersSlow(n int) []int {
    var numbers []int
    for i := 0; i < n; i++ {
        numbers = append(numbers, i) // Réallocation possible à chaque append !
    }
    return numbers
}

// ✅ Bon : Allocation unique
func createNumbersFast(n int) []int {
    numbers := make([]int, 0, n) // Pré-alloue la capacité
    for i := 0; i < n; i++ {
        numbers = append(numbers, i)
    }
    return numbers
}

// ✅ Encore mieux : Utilise l'index directement
func createNumbersBest(n int) []int {
    numbers := make([]int, n) // Pré-alloue la taille
    for i := 0; i < n; i++ {
        numbers[i] = i
    }
    return numbers
}
```

## 3. Optimisations mémoire

### Réutiliser les objets (Object Pooling)

```go
import "sync"

// Pool d'objets pour éviter les allocations répétées
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 1024) // Buffer de 1KB
    },
}

// ❌ Mauvais : Allocation à chaque appel
func processDataSlow(data []byte) []byte {
    buffer := make([]byte, len(data)*2) // Nouvelle allocation !
    copy(buffer, data)
    // ... traitement
    return buffer
}

// ✅ Bon : Réutilise un buffer du pool
func processDataFast(data []byte) []byte {
    buffer := bufferPool.Get().([]byte)
    defer bufferPool.Put(buffer[:0]) // Remet dans le pool

    // S'assure que le buffer est assez grand
    if cap(buffer) < len(data)*2 {
        buffer = make([]byte, 0, len(data)*2)
    }

    buffer = buffer[:len(data)*2]
    copy(buffer, data)
    // ... traitement

    // Retourne une copie car le buffer sera réutilisé
    result := make([]byte, len(buffer))
    copy(result, buffer)
    return result
}
```

### Optimiser les conversions de chaînes

```go
// ❌ Mauvais : Conversions répétées
func buildURLSlow(parts []string) string {
    var result string
    for _, part := range parts {
        result += "/" + part // Allocation à chaque concaténation !
    }
    return result
}

// ✅ Bon : Utilise strings.Builder
func buildURLFast(parts []string) string {
    var builder strings.Builder

    // Estime la taille finale pour éviter les réallocations
    totalSize := len(parts) // Pour les "/"
    for _, part := range parts {
        totalSize += len(part)
    }
    builder.Grow(totalSize)

    for _, part := range parts {
        builder.WriteString("/")
        builder.WriteString(part)
    }
    return builder.String()
}

// ✅ Alternative : Utilise strings.Join
func buildURLBest(parts []string) string {
    return "/" + strings.Join(parts, "/")
}
```

### Éviter les conversions inutiles

```go
// ❌ Mauvais : Conversions répétées
func processNumbersSlow(numbers []string) int {
    sum := 0
    for _, numStr := range numbers {
        num, err := strconv.Atoi(numStr)
        if err != nil {
            continue
        }
        // Conversion inutile pour la comparaison
        if strconv.Itoa(num) == numStr {
            sum += num
        }
    }
    return sum
}

// ✅ Bon : Évite les conversions inutiles
func processNumbersFast(numbers []string) int {
    sum := 0
    for _, numStr := range numbers {
        num, err := strconv.Atoi(numStr)
        if err != nil {
            continue
        }
        // Pas besoin de reconvertir !
        sum += num
    }
    return sum
}
```

## 4. Optimisations I/O

### Utiliser le buffering

```go
// ❌ Mauvais : Écrit ligne par ligne
func writeLinesSlow(filename string, lines []string) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    for _, line := range lines {
        _, err := file.WriteString(line + "\n") // Appel système à chaque ligne !
        if err != nil {
            return err
        }
    }
    return nil
}

// ✅ Bon : Utilise un buffer
func writeLinesFast(filename string, lines []string) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    writer := bufio.NewWriter(file)
    defer writer.Flush()

    for _, line := range lines {
        _, err := writer.WriteString(line + "\n") // Écrit dans le buffer
        if err != nil {
            return err
        }
    }
    return nil
}
```

### Lire par chunks

```go
// ❌ Mauvais : Lit tout en mémoire
func processLargeFileSlow(filename string) error {
    data, err := os.ReadFile(filename) // Charge tout le fichier !
    if err != nil {
        return err
    }

    // Traite toutes les données
    for _, b := range data {
        // ... traitement
        _ = b
    }
    return nil
}

// ✅ Bon : Lit par chunks
func processLargeFileFast(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    reader := bufio.NewReader(file)
    buffer := make([]byte, 4096) // Buffer de 4KB

    for {
        n, err := reader.Read(buffer)
        if err != nil && err != io.EOF {
            return err
        }
        if n == 0 {
            break
        }

        // Traite seulement les données lues
        for i := 0; i < n; i++ {
            // ... traitement
            _ = buffer[i]
        }
    }
    return nil
}
```

## 5. Optimisations de concurrence

### Utiliser des worker pools

```go
// ❌ Mauvais : Crée une goroutine par tâche
func processTasksSlow(tasks []Task) {
    var wg sync.WaitGroup

    for _, task := range tasks {
        wg.Add(1)
        go func(t Task) { // Peut créer des milliers de goroutines !
            defer wg.Done()
            t.Process()
        }(task)
    }

    wg.Wait()
}

// ✅ Bon : Utilise un pool de workers
func processTasksFast(tasks []Task) {
    const numWorkers = 10
    taskChan := make(chan Task, len(tasks))
    var wg sync.WaitGroup

    // Démarre les workers
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range taskChan {
                task.Process()
            }
        }()
    }

    // Envoie les tâches
    for _, task := range tasks {
        taskChan <- task
    }
    close(taskChan)

    wg.Wait()
}
```

### Éviter la contention sur les mutex

```go
// ❌ Mauvais : Mutex global pour tout
type CounterSlow struct {
    mu    sync.Mutex
    count int
    data  map[string]int
}

func (c *CounterSlow) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *CounterSlow) Get(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.data[key] // Même mutex pour des données différentes !
}

// ✅ Bon : Mutex séparés
type CounterFast struct {
    countMu sync.Mutex
    count   int

    dataMu sync.RWMutex
    data   map[string]int
}

func (c *CounterFast) Increment() {
    c.countMu.Lock()
    defer c.countMu.Unlock()
    c.count++
}

func (c *CounterFast) Get(key string) int {
    c.dataMu.RLock() // Lecture concurrente possible
    defer c.dataMu.RUnlock()
    return c.data[key]
}
```

## 6. Optimisations de compilation

### Utiliser les build tags pour les optimisations

```go
// +build optimize

// optimized.go - Version optimisée
func expensiveOperation() {
    // Version optimisée mais plus complexe
}
```

```go
// +build !optimize

// normal.go - Version normale
func expensiveOperation() {
    // Version simple mais moins optimisée
}
```

```bash
# Compilation avec optimisations
go build -tags optimize

# Compilation normale
go build
```

### Optimisations du compilateur

```bash
# Désactiver les vérifications de bounds (dangereux !)
go build -gcflags=-B

# Optimisations agressives
go build -ldflags="-s -w"
```

## 7. Micro-optimisations

### Éviter les allocations dans les boucles

```go
// ❌ Mauvais : Allocation dans la boucle
func processItemsSlow(items []string) {
    for _, item := range items {
        parts := strings.Split(item, ",") // Allocation à chaque itération !
        // ... traitement
        _ = parts
    }
}

// ✅ Bon : Réutilise le slice
func processItemsFast(items []string) {
    var parts []string
    for _, item := range items {
        parts = parts[:0] // Réinitialise sans allouer
        parts = strings.Split(item, ",")
        // ... traitement
        _ = parts
    }
}
```

### Utiliser les types appropriés

```go
// ❌ Mauvais : Interface vide coûteuse
func processValuesSlow(values []interface{}) {
    for _, v := range values {
        // Type assertion coûteuse
        if str, ok := v.(string); ok {
            fmt.Println(str)
        }
    }
}

// ✅ Bon : Type spécifique
func processValuesFast(values []string) {
    for _, v := range values {
        fmt.Println(v) // Pas d'assertion de type
    }
}
```

## 8. Exemple complet d'optimisation

### Problème : Analyser un fichier de logs

```go
// ❌ Version lente
func analyzeLogsSlow(filename string) (map[string]int, error) {
    // Lit tout le fichier
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, err
    }

    lines := strings.Split(string(data), "\n")
    results := make(map[string]int)

    for _, line := range lines {
        // Parsing coûteux
        parts := strings.Split(line, " ")
        if len(parts) > 0 {
            // Conversion coûteuse
            key := strings.ToLower(strings.TrimSpace(parts[0]))
            results[key]++
        }
    }

    return results, nil
}

// ✅ Version optimisée
func analyzeLogsFast(filename string) (map[string]int, error) {
    file, err := os.Open(filename)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    // Estime la taille de la map
    results := make(map[string]int, 1000)

    scanner := bufio.NewScanner(file)
    // Buffer plus grand pour les longues lignes
    buffer := make([]byte, 64*1024)
    scanner.Buffer(buffer, 1024*1024)

    for scanner.Scan() {
        line := scanner.Text()
        if len(line) == 0 {
            continue
        }

        // Trouve le premier espace sans Split
        spaceIdx := strings.Index(line, " ")
        if spaceIdx == -1 {
            continue
        }

        key := strings.ToLower(strings.TrimSpace(line[:spaceIdx]))
        results[key]++
    }

    return results, scanner.Err()
}
```

## Outils pour mesurer les optimisations

### Benchmark comparatif

```go
func BenchmarkAnalyzeLogsSlow(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _, err := analyzeLogsSlow("test.log")
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkAnalyzeLogsFast(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _, err := analyzeLogsFast("test.log")
        if err != nil {
            b.Fatal(err)
        }
    }
}
```

### Mesurer les allocations

```go
func BenchmarkStringBuilding(b *testing.B) {
    b.Run("Slow", func(b *testing.B) {
        b.ReportAllocs() // Affiche les allocations
        for i := 0; i < b.N; i++ {
            buildURLSlow([]string{"api", "v1", "users", "123"})
        }
    })

    b.Run("Fast", func(b *testing.B) {
        b.ReportAllocs()
        for i := 0; i < b.N; i++ {
            buildURLFast([]string{"api", "v1", "users", "123"})
        }
    })
}
```

## Checklist d'optimisation

### Avant d'optimiser
- [ ] Profiler le code pour identifier les vrais goulots d'étranglement
- [ ] Mesurer les performances actuelles
- [ ] Définir des objectifs clairs

### Pendant l'optimisation
- [ ] Optimiser une chose à la fois
- [ ] Mesurer l'impact de chaque changement
- [ ] Conserver la lisibilité du code

### Après l'optimisation
- [ ] Valider que les optimisations fonctionnent
- [ ] Tester que le comportement reste correct
- [ ] Documenter les changements importants

## Pièges à éviter

### 1. Optimisation prématurée
```go
// ❌ Mauvais : Optimiser sans mesurer
func processData(data []string) {
    // Code compliqué "pour la performance"
    // mais le gain est négligeable
}
```

### 2. Sacrifier la lisibilité
```go
// ❌ Mauvais : Code illisible pour un gain minimal
func calculate(a, b int) int {
    return ((a<<1)^b)&((b<<1)^a) // Qu'est-ce que ça fait ?
}

// ✅ Bon : Code clair
func calculate(a, b int) int {
    return 2*a*b / (a + b) // Formule claire
}
```

### 3. Optimiser au mauvais endroit
Optimiser une fonction appelée une fois par heure n'aura pas d'impact sur les performances globales.

## Exercices pratiques

### Exercice 1 : Optimiser une recherche
```go
// Optimisez cette fonction
func findUsers(users []User, criteria SearchCriteria) []User {
    var results []User
    for _, user := range users {
        if matchesCriteria(user, criteria) {
            results = append(results, user)
        }
    }
    return results
}
```

### Exercice 2 : Optimiser un traitement de données
```go
// Optimisez cette fonction qui traite un CSV
func processCSV(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    reader := csv.NewReader(file)
    records, err := reader.ReadAll()
    if err != nil {
        return err
    }

    for _, record := range records {
        // Traitement ligne par ligne
        processRecord(record)
    }

    return nil
}
```

## Résumé

Les optimisations courantes en Go suivent des patterns bien établis :

1. **Algorithmes** : Choisir la bonne complexité
2. **Structures de données** : Utiliser les bonnes structures et pré-allouer
3. **Mémoire** : Réutiliser les objets et éviter les allocations
4. **I/O** : Utiliser le buffering et traiter par chunks
5. **Concurrence** : Worker pools et mutex appropriés

**Règle d'or** : Toujours mesurer avant et après vos optimisations !

Dans le prochain chapitre, nous verrons comment optimiser le garbage collector et comprendre son impact sur les performances.

⏭️
