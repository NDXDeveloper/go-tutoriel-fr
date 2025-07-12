🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18-3. Garbage collector

## Introduction

Le garbage collector (GC) de Go est comme un assistant invisible qui nettoie automatiquement la mémoire de votre programme. Imaginez que vous cuisinez dans une cuisine : vous utilisez des ustensiles, des ingrédients, et vous créez des déchets. Le GC est comme quelqu'un qui passerait régulièrement pour nettoyer et ranger ce qui n'est plus utilisé.

Comprendre le GC est crucial car il peut affecter les performances de votre application, surtout si vous créez beaucoup d'objets ou si vous avez besoin de très faibles latences.

## Qu'est-ce que le garbage collector ?

### Définition simple
Le garbage collector libère automatiquement la mémoire qui n'est plus utilisée par votre programme. Vous n'avez pas besoin d'appeler `free()` comme en C - Go s'en occupe pour vous !

### Pourquoi c'est important
```go
func createData() {
    data := make([]byte, 1024*1024) // 1MB d'allocation
    // Traitement des données...
    // À la fin de la fonction, 'data' n'est plus accessible
    // Le GC va automatiquement libérer cette mémoire
}

func main() {
    for i := 0; i < 1000; i++ {
        createData() // Sans GC, on utiliserait 1GB de RAM !
    }
    // Grâce au GC, la mémoire est réutilisée
}
```

## Comment fonctionne le GC de Go

### Le modèle tricolore (simplifié)

Le GC de Go utilise un algorithme "tricolore" qui classe les objets en trois catégories :

1. **Blanc** : Objets potentiellement inutiles (candidats à la suppression)
2. **Gris** : Objets en cours d'analyse
3. **Noir** : Objets confirmés comme encore utilisés

```go
// Exemple conceptuel
func demonstrateGC() {
    // Objet racine (toujours "noir" - jamais supprimé)
    mainData := &Data{value: "important"}

    // Objet référencé (devient "noir" pendant le GC)
    linkedData := &Data{value: "also important"}
    mainData.next = linkedData

    // Objet temporaire (devient "blanc" et sera supprimé)
    tempData := &Data{value: "temporary"}
    _ = tempData // Plus de référence -> éligible pour le GC
}
```

### Phases du GC

1. **Mark** (Marquage) : Identifie quels objets sont encore utilisés
2. **Sweep** (Balayage) : Libère les objets non marqués
3. **Concurrent** : Ces opérations se font en parallèle de votre programme

## Types de collecte en Go

### GC concurrent
Le GC de Go fonctionne en arrière-plan pendant que votre programme s'exécute :

```go
func main() {
    for i := 0; i < 1000000; i++ {
        data := make([]int, 100)
        // Votre programme continue à tourner
        // Le GC nettoie en parallèle
        processData(data)
    }
}
```

### Avantages vs inconvénients

**Avantages :**
- Pas de pauses longues
- Votre programme reste réactif
- Gestion automatique de la mémoire

**Inconvénients :**
- Utilise un peu de CPU pour le nettoyage
- Peut créer de petites latences imprévisibles

## Observer le comportement du GC

### Variables d'environnement utiles

```bash
# Affiche les statistiques du GC
export GODEBUG=gctrace=1
go run main.go

# Exemple de sortie :
# gc 1 @0.002s 0%: 0.018+1.3+0.076 ms clock, 0.074+0.35/1.2/3.0+0.30 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
```

**Explication de la sortie :**
- `gc 1` : Premier cycle de GC
- `@0.002s` : Temps depuis le démarrage
- `0%` : Pourcentage de temps CPU utilisé par le GC
- `4->4->0 MB` : Mémoire avant -> pendant -> après le GC

### Programme pour observer le GC

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func allocateMemory() {
    // Alloue beaucoup de petits objets
    var data [][]byte
    for i := 0; i < 10000; i++ {
        data = append(data, make([]byte, 1024))
    }
    // Les objets deviennent éligibles pour le GC à la fin de la fonction
}

func printGCStats() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    fmt.Printf("Allocs = %d KB", bToKb(m.Alloc))
    fmt.Printf(", TotalAllocs = %d KB", bToKb(m.TotalAlloc))
    fmt.Printf(", Sys = %d KB", bToKb(m.Sys))
    fmt.Printf(", NumGC = %d\n", m.NumGC)
}

func bToKb(b uint64) uint64 {
    return b / 1024
}

func main() {
    fmt.Println("Avant allocations:")
    printGCStats()

    // Alloue de la mémoire
    for i := 0; i < 5; i++ {
        allocateMemory()
        fmt.Printf("Après allocation %d:\n", i+1)
        printGCStats()
        time.Sleep(100 * time.Millisecond)
    }

    // Force un garbage collection
    runtime.GC()
    fmt.Println("Après GC forcé:")
    printGCStats()
}
```

## Optimiser pour le GC

### 1. Réduire la pression sur le GC

```go
// ❌ Mauvais : Crée beaucoup d'objets temporaires
func processDataBad(items []string) []string {
    var results []string
    for _, item := range items {
        // Allocation temporaire à chaque itération
        processed := strings.ToUpper(item)
        trimmed := strings.TrimSpace(processed)
        results = append(results, trimmed)
    }
    return results
}

// ✅ Bon : Réduit les allocations temporaires
func processDataGood(items []string) []string {
    results := make([]string, 0, len(items)) // Pré-alloue
    for _, item := range items {
        // Une seule allocation par étape
        processed := strings.TrimSpace(strings.ToUpper(item))
        results = append(results, processed)
    }
    return results
}
```

### 2. Réutiliser les objets (Object Pooling)

```go
import "sync"

// Pool pour réutiliser les buffers
var bufferPool = sync.Pool{
    New: func() interface{} {
        // Crée un nouveau buffer quand le pool est vide
        return make([]byte, 0, 1024)
    },
}

// ❌ Mauvais : Nouvelle allocation à chaque appel
func formatDataBad(data []int) string {
    buffer := make([]byte, 0, 1024) // Allocation !
    for _, num := range data {
        buffer = append(buffer, []byte(fmt.Sprintf("%d,", num))...)
    }
    return string(buffer)
}

// ✅ Bon : Réutilise un buffer du pool
func formatDataGood(data []int) string {
    buffer := bufferPool.Get().([]byte)
    defer bufferPool.Put(buffer[:0]) // Remet dans le pool, capacité préservée

    for _, num := range data {
        buffer = append(buffer, []byte(fmt.Sprintf("%d,", num))...)
    }

    // Important : retourne une copie car le buffer sera réutilisé
    return string(buffer)
}
```

### 3. Utiliser des structures plus efficaces

```go
// ❌ Mauvais : Beaucoup de petits objets
type NodeBad struct {
    Value string
    Next  *NodeBad // Pointeur = allocation séparée
}

// ✅ Bon : Moins d'allocations avec des slices
type ListGood struct {
    Values []string // Tous les éléments dans un seul bloc mémoire
}

func (l *ListGood) Add(value string) {
    l.Values = append(l.Values, value)
}
```

### 4. Contrôler la fréquence du GC

```go
package main

import (
    "runtime"
    "runtime/debug"
)

func optimizeGCSettings() {
    // Ajuste le pourcentage cible pour le prochain GC
    // Plus élevé = GC moins fréquent mais plus de mémoire utilisée
    debug.SetGCPercent(200) // Par défaut : 100

    // Pour des applications avec beaucoup de mémoire disponible
    // debug.SetGCPercent(500) // GC très peu fréquent

    // Pour des applications avec peu de mémoire
    // debug.SetGCPercent(50) // GC plus fréquent
}

func main() {
    optimizeGCSettings()

    // Votre application...
}
```

## Patterns pour minimiser l'impact du GC

### 1. Batch processing

```go
// ❌ Mauvais : Traite un élément à la fois
func processItemsOneByOne(items []Data) {
    for _, item := range items {
        result := processItem(item) // Peut créer des objets temporaires
        saveResult(result)
        // GC peut se déclencher à chaque itération
    }
}

// ✅ Bon : Traite par lots
func processItemsBatch(items []Data) {
    const batchSize = 100

    for i := 0; i < len(items); i += batchSize {
        end := i + batchSize
        if end > len(items) {
            end = len(items)
        }

        batch := items[i:end]
        results := make([]Result, 0, len(batch))

        // Traite tout le lot d'un coup
        for _, item := range batch {
            results = append(results, processItem(item))
        }

        saveResults(results) // Sauvegarde par lot

        // Optionnel : force un GC entre les lots
        runtime.GC()
    }
}
```

### 2. Pré-allocation intelligente

```go
// ❌ Mauvais : Laisse le slice grandir dynamiquement
func collectResultsBad() []Result {
    var results []Result
    for i := 0; i < 10000; i++ {
        // Le slice va être réalloué plusieurs fois
        results = append(results, processData(i))
    }
    return results
}

// ✅ Bon : Pré-alloue la taille finale
func collectResultsGood() []Result {
    results := make([]Result, 0, 10000) // Capacité connue à l'avance
    for i := 0; i < 10000; i++ {
        results = append(results, processData(i))
    }
    return results
}
```

### 3. Réutilisation de structures

```go
type DataProcessor struct {
    buffer    []byte     // Réutilisé pour chaque traitement
    tempSlice []int      // Réutilisé pour les calculs temporaires
    cache     map[string]Result // Cache pour éviter les recalculs
}

func NewDataProcessor() *DataProcessor {
    return &DataProcessor{
        buffer:    make([]byte, 0, 1024),
        tempSlice: make([]int, 0, 100),
        cache:     make(map[string]Result),
    }
}

func (dp *DataProcessor) Process(data string) Result {
    // Vérifie le cache d'abord
    if result, found := dp.cache[data]; found {
        return result
    }

    // Réinitialise les buffers sans réallouer
    dp.buffer = dp.buffer[:0]
    dp.tempSlice = dp.tempSlice[:0]

    // Utilise les buffers réutilisables
    dp.buffer = append(dp.buffer, []byte(data)...)

    // ... traitement ...

    result := Result{/* ... */}
    dp.cache[data] = result
    return result
}
```

## Mesurer l'impact du GC

### Métriques importantes

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

type GCStats struct {
    NumGC        uint32
    PauseTotal   time.Duration
    LastPause    time.Duration
    AllocatedMB  uint64
}

func getGCStats() GCStats {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    return GCStats{
        NumGC:        m.NumGC,
        PauseTotal:   time.Duration(m.PauseTotalNs),
        LastPause:    time.Duration(m.PauseNs[(m.NumGC+255)%256]),
        AllocatedMB:  m.Alloc / 1024 / 1024,
    }
}

func monitorGC(duration time.Duration) {
    start := getGCStats()
    startTime := time.Now()

    time.Sleep(duration)

    end := getGCStats()
    elapsed := time.Since(startTime)

    fmt.Printf("Période de monitoring: %v\n", elapsed)
    fmt.Printf("Cycles GC: %d\n", end.NumGC-start.NumGC)
    fmt.Printf("Temps total de pause: %v\n", end.PauseTotal-start.PauseTotal)
    fmt.Printf("Dernière pause: %v\n", end.LastPause)
    fmt.Printf("Mémoire allouée: %d MB\n", end.AllocatedMB)

    if end.NumGC > start.NumGC {
        avgPause := (end.PauseTotal - start.PauseTotal) / time.Duration(end.NumGC-start.NumGC)
        fmt.Printf("Pause moyenne: %v\n", avgPause)
    }
}
```

### Benchmark avec mesure GC

```go
func BenchmarkWithGCStats(b *testing.B) {
    var startGC, endGC runtime.MemStats

    // Mesure avant
    runtime.GC()
    runtime.ReadMemStats(&startGC)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        // Votre code à benchmarker
        processData()
    }
    b.StopTimer()

    // Mesure après
    runtime.ReadMemStats(&endGC)

    // Rapporte les statistiques GC
    b.ReportMetric(float64(endGC.NumGC-startGC.NumGC), "gcs")
    b.ReportMetric(float64(endGC.TotalAlloc-startGC.TotalAlloc)/1024/1024, "MB-allocated")
}
```

## Cas d'usage spécifiques

### Applications web avec faible latence

```go
func optimizeForWebServer() {
    // Réduit la fréquence du GC pour éviter les pauses pendant les requêtes
    debug.SetGCPercent(300)

    // Utilise des pools d'objets pour les structures communes
    var responsePool = sync.Pool{
        New: func() interface{} {
            return &Response{
                Headers: make(map[string]string),
                Body:    make([]byte, 0, 1024),
            }
        },
    }
}
```

### Applications de traitement de données

```go
func optimizeForDataProcessing() {
    // GC plus fréquent pour éviter l'accumulation de mémoire
    debug.SetGCPercent(50)

    // Force le GC après chaque lot de données
    func processBatch(data []Record) {
        // ... traitement ...
        runtime.GC() // Nettoie immédiatement
    }
}
```

### Applications en temps réel

```go
func optimizeForRealTime() {
    // Pré-alloue toute la mémoire nécessaire au démarrage
    preallocateMemory()

    // Désactive temporairement le GC pendant les sections critiques
    func criticalSection() {
        debug.SetGCPercent(-1) // Désactive le GC
        defer debug.SetGCPercent(100) // Réactive après

        // Code critique qui ne doit pas être interrompu
        performRealTimeTask()
    }
}
```

## Outils de debugging du GC

### 1. go tool trace

```go
func main() {
    f, err := os.Create("trace.out")
    if err != nil {
        panic(err)
    }
    defer f.Close()

    trace.Start(f)
    defer trace.Stop()

    // Votre code ici
    doWork()
}
```

```bash
go tool trace trace.out
```

### 2. Profiling mémoire avec focus GC

```bash
# Génère un profil pendant que le GC est actif
go tool pprof http://localhost:6060/debug/pprof/allocs

# Dans pprof, analyser les allocations
(pprof) top10
(pprof) list main.functionName
```

## Bonnes pratiques récapitulatives

### Do's (À faire)
- ✅ Pré-allouer les slices et maps quand la taille est connue
- ✅ Réutiliser les objets avec des pools
- ✅ Mesurer l'impact du GC sur votre application
- ✅ Traiter les données par lots
- ✅ Utiliser des buffers réutilisables

### Don'ts (À éviter)
- ❌ Créer beaucoup de petits objets temporaires
- ❌ Forcer le GC trop souvent (sauf cas spéciaux)
- ❌ Ignorer les métriques de GC en production
- ❌ Optimiser le GC sans mesurer l'impact réel
- ❌ Désactiver le GC sans très bonne raison

## Exercices pratiques

### Exercice 1 : Observer le GC

Créez un programme qui :
1. Alloue progressivement de la mémoire
2. Affiche les statistiques GC à intervalles réguliers
3. Observe comment le comportement change avec différents patterns d'allocation

### Exercice 2 : Optimiser une fonction

Optimisez cette fonction pour réduire la pression sur le GC :

```go
func processLogs(logs []string) map[string]int {
    counts := make(map[string]int)
    for _, log := range logs {
        parts := strings.Split(log, " ")
        if len(parts) > 0 {
            level := strings.ToUpper(strings.TrimSpace(parts[0]))
            counts[level]++
        }
    }
    return counts
}
```

**Indices :**
- Pré-allouer la map
- Éviter les allocations temporaires
- Réutiliser des buffers

## Résumé

Le garbage collector de Go est un outil puissant mais qu'il faut comprendre pour optimiser ses applications :

**Points clés :**
- Le GC fonctionne en parallèle de votre programme
- Il peut affecter les performances si mal géré
- L'objectif est de réduire la "pression" sur le GC

**Stratégies principales :**
- Réduire les allocations temporaires
- Réutiliser les objets avec des pools
- Pré-allouer quand c'est possible
- Mesurer l'impact régulièrement

**Outils de mesure :**
- Variables d'environnement `GODEBUG`
- Package `runtime` pour les métriques
- Profiling mémoire avec `go tool pprof`

Dans le prochain chapitre, nous verrons comment détecter et corriger les fuites mémoire, un problème qui peut court-circuiter l'efficacité du garbage collector.

⏭️
