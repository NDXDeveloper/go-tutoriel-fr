🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10-2 : Worker pools

## Introduction

Un **worker pool** (pool de travailleurs) est un pattern de concurrence très populaire en Go. Il permet de traiter efficacement un grand nombre de tâches en parallèle en utilisant un nombre fixe de goroutines "travailleuses". C'est l'équivalent d'une équipe de travailleurs qui se partagent une pile de tâches à accomplir.

## Pourquoi utiliser un worker pool ?

### Problème sans worker pool

Imaginez que vous devez traiter 10 000 fichiers. Une approche naïve serait :

```go
// ❌ MAUVAISE APPROCHE : Trop de goroutines !
for i := 0; i < 10000; i++ {
    go func(fileID int) {
        processFile(fileID)
    }(i)
}
```

**Problèmes** :
- **10 000 goroutines** créées simultanément
- **Surcharge mémoire** énorme
- **Contention des ressources** (CPU, réseau, disque)
- **Possible crash** du système

### Solution avec worker pool

```go
// ✅ BONNE APPROCHE : Nombre limité de workers
jobs := make(chan int, 100)
numWorkers := 10

// Créer 10 workers seulement
for w := 1; w <= numWorkers; w++ {
    go worker(w, jobs)
}

// Envoyer les tâches
for i := 0; i < 10000; i++ {
    jobs <- i
}
close(jobs)
```

**Avantages** :
- **Nombre contrôlé** de goroutines (10 au lieu de 10 000)
- **Utilisation optimale** des ressources
- **Stabilité** du système
- **Performance** prévisible

## Anatomie d'un worker pool

### Composants principaux

1. **Jobs channel** : Canal contenant les tâches à traiter
2. **Workers** : Goroutines qui traitent les tâches
3. **Results channel** (optionnel) : Canal pour collecter les résultats
4. **Coordination** : Mécanisme pour savoir quand tout est terminé

### Schéma conceptuel

```
[Tâches] → [Jobs Channel] → [Worker 1] → [Results Channel]
                         → [Worker 2] → [Results Channel]
                         → [Worker 3] → [Results Channel]
                         → [Worker N] → [Results Channel]
```

## Exemple de base

### Worker pool simple

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Job représente une tâche à traiter
type Job struct {
    ID   int
    Data string
}

// Result représente le résultat d'une tâche
type Result struct {
    JobID  int
    Output string
    Error  error
}

// Worker function qui traite les jobs
func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("👷 Worker %d traite le job %d\n", id, job.ID)

        // Simulation du travail
        time.Sleep(time.Duration(100+job.ID*10) * time.Millisecond)

        // Créer le résultat
        result := Result{
            JobID:  job.ID,
            Output: fmt.Sprintf("Traité: %s", job.Data),
            Error:  nil,
        }

        results <- result
        fmt.Printf("✅ Worker %d terminé job %d\n", id, job.ID)
    }

    fmt.Printf("🏁 Worker %d s'arrête\n", id)
}

func basicWorkerPool() {
    numWorkers := 3
    numJobs := 10

    // Créer les channels
    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    // WaitGroup pour attendre tous les workers
    var wg sync.WaitGroup

    // Lancer les workers
    fmt.Printf("🚀 Lancement de %d workers\n", numWorkers)
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Envoyer les jobs
    fmt.Printf("📋 Envoi de %d jobs\n", numJobs)
    for j := 1; j <= numJobs; j++ {
        job := Job{
            ID:   j,
            Data: fmt.Sprintf("données-%d", j),
        }
        jobs <- job
    }
    close(jobs) // Important : fermer le channel des jobs

    // Collecter les résultats dans une goroutine séparée
    go func() {
        wg.Wait()
        close(results) // Fermer quand tous les workers sont terminés
    }()

    // Lire les résultats
    fmt.Println("📊 Collecte des résultats:")
    for result := range results {
        if result.Error != nil {
            fmt.Printf("❌ Erreur job %d: %v\n", result.JobID, result.Error)
        } else {
            fmt.Printf("✅ Résultat job %d: %s\n", result.JobID, result.Output)
        }
    }

    fmt.Println("🎉 Tous les jobs sont terminés !")
}

func main() {
    basicWorkerPool()
}
```

### Sortie attendue

```
🚀 Lancement de 3 workers
📋 Envoi de 10 jobs
👷 Worker 1 traite le job 1
👷 Worker 2 traite le job 2
👷 Worker 3 traite le job 3
✅ Worker 1 terminé job 1
👷 Worker 1 traite le job 4
✅ Worker 2 terminé job 2
👷 Worker 2 traite le job 5
...
```

## Worker pool avec gestion d'erreurs

### Version robuste

```go
package main

import (
    "errors"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

type Task struct {
    ID       int
    URL      string
    Retries  int
}

type TaskResult struct {
    TaskID   int
    Success  bool
    Data     string
    Error    error
    Duration time.Duration
}

// Simulation d'une tâche qui peut échouer
func processTask(task Task) TaskResult {
    start := time.Now()

    // Simulation d'un traitement aléatoire
    processingTime := time.Duration(rand.Intn(500)) * time.Millisecond
    time.Sleep(processingTime)

    // 20% de chance d'échec
    if rand.Float32() < 0.2 {
        return TaskResult{
            TaskID:   task.ID,
            Success:  false,
            Error:    errors.New("traitement échoué"),
            Duration: time.Since(start),
        }
    }

    return TaskResult{
        TaskID:   task.ID,
        Success:  true,
        Data:     fmt.Sprintf("Données traitées pour %s", task.URL),
        Duration: time.Since(start),
    }
}

func robustWorker(id int, tasks <-chan Task, results chan<- TaskResult, wg *sync.WaitGroup) {
    defer wg.Done()

    processed := 0
    for task := range tasks {
        fmt.Printf("🔧 Worker %d traite la tâche %d (tentative %d)\n",
                  id, task.ID, task.Retries+1)

        result := processTask(task)

        // Si échec et des tentatives restantes
        if !result.Success && task.Retries < 2 {
            fmt.Printf("⚠️ Worker %d: échec tâche %d, nouvelle tentative...\n",
                      id, task.ID)

            // Remettre la tâche en queue avec un retry
            retryTask := task
            retryTask.Retries++

            // Utiliser un délai avant retry
            go func() {
                time.Sleep(100 * time.Millisecond)
                tasks <- retryTask
            }()
        } else {
            results <- result
            processed++
        }
    }

    fmt.Printf("🏁 Worker %d terminé (%d tâches traitées)\n", id, processed)
}

func robustWorkerPool() {
    numWorkers := 4
    numTasks := 15

    tasks := make(chan Task, numTasks*2) // Buffer plus large pour les retries
    results := make(chan TaskResult, numTasks)

    var wg sync.WaitGroup

    // Lancer les workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go robustWorker(w, tasks, results, &wg)
    }

    // Envoyer les tâches initiales
    for i := 1; i <= numTasks; i++ {
        task := Task{
            ID:      i,
            URL:     fmt.Sprintf("https://api.example.com/data/%d", i),
            Retries: 0,
        }
        tasks <- task
    }

    // Fermer et collecter les résultats
    go func() {
        wg.Wait()
        close(results)
    }()

    // Statistiques
    var successful, failed int
    var totalDuration time.Duration

    fmt.Println("📊 Résultats:")
    for result := range results {
        totalDuration += result.Duration

        if result.Success {
            successful++
            fmt.Printf("✅ Tâche %d réussie en %v\n",
                      result.TaskID, result.Duration)
        } else {
            failed++
            fmt.Printf("❌ Tâche %d échouée: %v\n",
                      result.TaskID, result.Error)
        }
    }

    fmt.Printf("\n📈 Statistiques finales:\n")
    fmt.Printf("   ✅ Réussies: %d\n", successful)
    fmt.Printf("   ❌ Échouées: %d\n", failed)
    fmt.Printf("   ⏱️ Temps moyen: %v\n", totalDuration/time.Duration(successful+failed))

    close(tasks)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    robustWorkerPool()
}
```

## Worker pool configurable

### Version flexible et réutilisable

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// WorkerPool représente un pool de workers configurable
type WorkerPool struct {
    workerCount int
    jobQueue    chan Job
    resultQueue chan Result
    wg          sync.WaitGroup
    ctx         context.Context
    cancel      context.CancelFunc
}

// NewWorkerPool crée un nouveau worker pool
func NewWorkerPool(workerCount, queueSize int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())

    return &WorkerPool{
        workerCount: workerCount,
        jobQueue:    make(chan Job, queueSize),
        resultQueue: make(chan Result, queueSize),
        ctx:         ctx,
        cancel:      cancel,
    }
}

// Start démarre tous les workers
func (wp *WorkerPool) Start() {
    for i := 1; i <= wp.workerCount; i++ {
        wp.wg.Add(1)
        go wp.worker(i)
    }

    fmt.Printf("🚀 Worker pool démarré avec %d workers\n", wp.workerCount)
}

// Stop arrête proprement le worker pool
func (wp *WorkerPool) Stop() {
    close(wp.jobQueue)
    wp.wg.Wait()
    close(wp.resultQueue)
    wp.cancel()
    fmt.Println("🛑 Worker pool arrêté")
}

// SubmitJob soumet un job au pool
func (wp *WorkerPool) SubmitJob(job Job) {
    select {
    case wp.jobQueue <- job:
        // Job soumis avec succès
    case <-wp.ctx.Done():
        // Pool fermé
        fmt.Printf("⚠️ Pool fermé, job %d rejeté\n", job.ID)
    }
}

// GetResults retourne le canal des résultats
func (wp *WorkerPool) GetResults() <-chan Result {
    return wp.resultQueue
}

// worker est la fonction exécutée par chaque worker
func (wp *WorkerPool) worker(id int) {
    defer wp.wg.Done()

    jobsProcessed := 0

    for {
        select {
        case job, ok := <-wp.jobQueue:
            if !ok {
                fmt.Printf("🏁 Worker %d terminé (%d jobs traités)\n",
                          id, jobsProcessed)
                return
            }

            // Traiter le job
            result := wp.processJob(id, job)

            // Envoyer le résultat
            select {
            case wp.resultQueue <- result:
                jobsProcessed++
            case <-wp.ctx.Done():
                return
            }

        case <-wp.ctx.Done():
            fmt.Printf("🛑 Worker %d arrêté par contexte\n", id)
            return
        }
    }
}

// processJob traite un job individuel
func (wp *WorkerPool) processJob(workerID int, job Job) Result {
    fmt.Printf("⚙️ Worker %d traite job %d\n", workerID, job.ID)

    start := time.Now()

    // Simulation du travail
    time.Sleep(time.Duration(job.ID*50) * time.Millisecond)

    return Result{
        JobID:  job.ID,
        Output: fmt.Sprintf("Job %d traité par worker %d", job.ID, workerID),
        Error:  nil,
    }
}

// Exemple d'utilisation
func configurableWorkerPoolExample() {
    // Créer un pool avec 3 workers et une queue de 10
    pool := NewWorkerPool(3, 10)

    // Démarrer le pool
    pool.Start()

    // Soumettre des jobs
    numJobs := 8
    fmt.Printf("📋 Soumission de %d jobs\n", numJobs)

    for i := 1; i <= numJobs; i++ {
        job := Job{
            ID:   i,
            Data: fmt.Sprintf("Données job %d", i),
        }
        pool.SubmitJob(job)
    }

    // Collecter les résultats
    go func() {
        for result := range pool.GetResults() {
            fmt.Printf("📊 Résultat: %s\n", result.Output)
        }
    }()

    // Attendre un peu puis arrêter
    time.Sleep(2 * time.Second)
    pool.Stop()

    // Laisser du temps pour afficher les derniers résultats
    time.Sleep(500 * time.Millisecond)
}

func main() {
    configurableWorkerPoolExample()
}
```

## Patterns avancés

### 1. Worker pool avec priorités

```go
type PriorityJob struct {
    Job
    Priority int // Plus élevé = plus prioritaire
}

func priorityWorkerPool() {
    // Utiliser plusieurs channels pour différentes priorités
    highPriority := make(chan Job, 50)
    normalPriority := make(chan Job, 100)
    lowPriority := make(chan Job, 200)

    // Worker qui traite les priorités
    go func() {
        for {
            select {
            case job := <-highPriority:
                processJob(job)
            case job := <-normalPriority:
                processJob(job)
            case job := <-lowPriority:
                processJob(job)
            }
        }
    }()
}
```

### 2. Worker pool avec limitation de débit

```go
func rateLimitedWorkerPool() {
    rateLimiter := time.NewTicker(100 * time.Millisecond) // Max 10 req/sec
    defer rateLimiter.Stop()

    jobs := make(chan Job, 100)

    go func() {
        for job := range jobs {
            <-rateLimiter.C // Attendre le rate limiter
            processJob(job)
        }
    }()
}
```

### 3. Worker pool dynamique

```go
type DynamicWorkerPool struct {
    minWorkers    int
    maxWorkers    int
    currentWorkers int
    jobQueue      chan Job
    workersWG     sync.WaitGroup
    mu            sync.Mutex
}

func (dwp *DynamicWorkerPool) scaleUp() {
    dwp.mu.Lock()
    defer dwp.mu.Unlock()

    if dwp.currentWorkers < dwp.maxWorkers {
        dwp.currentWorkers++
        dwp.workersWG.Add(1)
        go dwp.worker(dwp.currentWorkers)
        fmt.Printf("📈 Nouveau worker ajouté (total: %d)\n", dwp.currentWorkers)
    }
}

func (dwp *DynamicWorkerPool) monitor() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        queueLength := len(dwp.jobQueue)

        // Scale up si la queue est pleine à 80%
        if queueLength > cap(dwp.jobQueue)*8/10 {
            dwp.scaleUp()
        }

        fmt.Printf("📊 Queue: %d/%d, Workers: %d\n",
                  queueLength, cap(dwp.jobQueue), dwp.currentWorkers)
    }
}
```

## Cas d'usage réels

### 1. Traitement d'images

```go
func imageProcessingPool() {
    type ImageJob struct {
        InputPath  string
        OutputPath string
        Width      int
        Height     int
    }

    jobs := make(chan ImageJob, 100)

    // Workers spécialisés dans le traitement d'images
    for i := 0; i < 4; i++ {
        go func(workerID int) {
            for job := range jobs {
                fmt.Printf("🖼️ Worker %d redimensionne %s\n",
                          workerID, job.InputPath)

                // Simulation du redimensionnement
                time.Sleep(200 * time.Millisecond)

                fmt.Printf("✅ Image sauvée: %s (%dx%d)\n",
                          job.OutputPath, job.Width, job.Height)
            }
        }(i)
    }
}
```

### 2. Crawling web

```go
func webCrawlerPool() {
    type CrawlJob struct {
        URL   string
        Depth int
    }

    jobs := make(chan CrawlJob, 1000)
    results := make(chan []string, 1000)

    // Workers pour crawler
    for i := 0; i < 10; i++ {
        go func(workerID int) {
            for job := range jobs {
                fmt.Printf("🕷️ Worker %d crawle %s\n", workerID, job.URL)

                // Simulation du crawling
                time.Sleep(500 * time.Millisecond)

                // Résultats simulés
                links := []string{
                    job.URL + "/page1",
                    job.URL + "/page2",
                    job.URL + "/page3",
                }

                results <- links
            }
        }(i)
    }
}
```

## Bonnes pratiques

### ✅ À faire

1. **Taille appropriée** : Choisir le bon nombre de workers (généralement = nombre de CPU cores)
2. **Buffer des channels** : Utiliser des channels bufferisés pour éviter les blocages
3. **Fermeture propre** : Toujours fermer les channels et attendre les workers
4. **Gestion d'erreurs** : Inclure une gestion robuste des erreurs
5. **Monitoring** : Ajouter des métriques et du logging

### ❌ À éviter

1. **Trop de workers** : Ne pas créer plus de workers que nécessaire
2. **Channels non fermés** : Peut causer des fuites de goroutines
3. **Pas de timeout** : Toujours avoir un mécanisme d'arrêt
4. **Ignorer les erreurs** : Gérer tous les cas d'erreur possibles

## Exercices pratiques

### Exercice 1 : Calculateur de nombres premiers

Créez un worker pool qui trouve tous les nombres premiers entre 1 et 10000.

```go
func primeWorkerPool() {
    // Votre implémentation ici
    // Indices :
    // - Chaque job = un nombre à tester
    // - Résultat = nombre + bool (est premier)
    // - 4-6 workers recommandés
}
```

### Exercice 2 : Simulateur de requêtes HTTP

Créez un worker pool qui simule des appels API avec gestion d'erreurs et retry.

```go
func httpSimulatorPool() {
    // Votre implémentation ici
    // Indices :
    // - Simuler des latences variables
    // - 20% de chance d'erreur
    // - 3 tentatives maximum
    // - Collecter les statistiques
}
```

## Résumé

Les worker pools sont un pattern essentiel en Go pour :

- **Contrôler** le nombre de goroutines
- **Optimiser** l'utilisation des ressources
- **Traiter** efficacement de gros volumes
- **Maintenir** la stabilité du système

**Pattern de base** :
1. Créer des channels pour jobs et résultats
2. Lancer un nombre fixe de workers
3. Distribuer les tâches via le channel jobs
4. Collecter les résultats
5. Fermer proprement le système

La maîtrise des worker pools vous permettra de construire des applications Go performantes et scalables !

⏭️

# Solutions des exercices pratiques - Worker Pools

## Exercice 1 : Calculateur de nombres premiers

### Solution complète

```go
package main

import (
    "fmt"
    "math"
    "sync"
    "time"
)

// PrimeJob représente un nombre à tester
type PrimeJob struct {
    Number int
}

// PrimeResult représente le résultat du test
type PrimeResult struct {
    Number  int
    IsPrime bool
    Elapsed time.Duration
}

// isPrime vérifie si un nombre est premier
func isPrime(n int) bool {
    if n < 2 {
        return false
    }
    if n == 2 {
        return true
    }
    if n%2 == 0 {
        return false
    }

    // Vérifier les diviseurs impairs jusqu'à √n
    sqrt := int(math.Sqrt(float64(n)))
    for i := 3; i <= sqrt; i += 2 {
        if n%i == 0 {
            return false
        }
    }
    return true
}

// primeWorker traite les jobs de calcul de nombres premiers
func primeWorker(id int, jobs <-chan PrimeJob, results chan<- PrimeResult, wg *sync.WaitGroup) {
    defer wg.Done()

    processed := 0
    for job := range jobs {
        start := time.Now()

        // Tester si le nombre est premier
        prime := isPrime(job.Number)
        elapsed := time.Since(start)

        // Créer le résultat
        result := PrimeResult{
            Number:  job.Number,
            IsPrime: prime,
            Elapsed: elapsed,
        }

        results <- result
        processed++

        // Log pour les nombres premiers trouvés
        if prime {
            fmt.Printf("🔢 Worker %d: %d est PREMIER (traité en %v)\n",
                      id, job.Number, elapsed)
        }
    }

    fmt.Printf("🏁 Worker %d terminé (%d nombres testés)\n", id, processed)
}

func primeWorkerPool() {
    const (
        numWorkers = 6
        maxNumber  = 10000
    )

    // Créer les channels
    jobs := make(chan PrimeJob, 100)
    results := make(chan PrimeResult, 100)

    var wg sync.WaitGroup

    // Démarrer les workers
    fmt.Printf("🚀 Démarrage de %d workers pour trouver les nombres premiers jusqu'à %d\n",
              numWorkers, maxNumber)

    startTime := time.Now()

    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go primeWorker(w, jobs, results, &wg)
    }

    // Envoyer tous les jobs
    go func() {
        for i := 1; i <= maxNumber; i++ {
            jobs <- PrimeJob{Number: i}
        }
        close(jobs)
    }()

    // Collecter les résultats dans une goroutine séparée
    go func() {
        wg.Wait()
        close(results)
    }()

    // Analyser les résultats
    var primes []int
    var totalTests int
    var totalTime time.Duration
    var slowestTest PrimeResult

    fmt.Println("📊 Collecte des résultats...")

    for result := range results {
        totalTests++
        totalTime += result.Elapsed

        if result.IsPrime {
            primes = append(primes, result.Number)
        }

        // Suivre le test le plus lent
        if result.Elapsed > slowestTest.Elapsed {
            slowestTest = result
        }
    }

    totalElapsed := time.Since(startTime)

    // Afficher les statistiques
    fmt.Printf("\n🎉 Calcul terminé en %v\n", totalElapsed)
    fmt.Printf("📈 Statistiques:\n")
    fmt.Printf("   🔢 Nombres testés: %d\n", totalTests)
    fmt.Printf("   ✨ Nombres premiers trouvés: %d\n", len(primes))
    fmt.Printf("   ⏱️ Temps moyen par test: %v\n", totalTime/time.Duration(totalTests))
    fmt.Printf("   🐌 Test le plus lent: %d (%v)\n", slowestTest.Number, slowestTest.Elapsed)
    fmt.Printf("   ⚡ Tests par seconde: %.0f\n", float64(totalTests)/totalElapsed.Seconds())

    // Afficher quelques nombres premiers
    fmt.Printf("\n🔥 Premiers nombres premiers trouvés: ")
    for i, prime := range primes {
        if i >= 20 { // Limiter l'affichage
            fmt.Printf("... (et %d autres)", len(primes)-20)
            break
        }
        fmt.Printf("%d ", prime)
    }
    fmt.Println()

    // Afficher les plus grands nombres premiers
    if len(primes) >= 10 {
        fmt.Printf("🏆 Les 10 plus grands nombres premiers: ")
        for i := len(primes) - 10; i < len(primes); i++ {
            fmt.Printf("%d ", primes[i])
        }
        fmt.Println()
    }
}
```

### Version optimisée avec batch processing

```go
// PrimeBatch représente un lot de nombres à tester
type PrimeBatch struct {
    Start int
    End   int
}

// PrimeBatchResult représente les résultats d'un lot
type PrimeBatchResult struct {
    Primes  []int
    Count   int
    Elapsed time.Duration
}

func primeBatchWorker(id int, batches <-chan PrimeBatch, results chan<- PrimeBatchResult, wg *sync.WaitGroup) {
    defer wg.Done()

    batchesProcessed := 0
    for batch := range batches {
        start := time.Now()

        var primes []int
        for n := batch.Start; n <= batch.End; n++ {
            if isPrime(n) {
                primes = append(primes, n)
            }
        }

        result := PrimeBatchResult{
            Primes:  primes,
            Count:   batch.End - batch.Start + 1,
            Elapsed: time.Since(start),
        }

        results <- result
        batchesProcessed++

        fmt.Printf("📦 Worker %d: batch %d-%d terminé (%d premiers trouvés)\n",
                  id, batch.Start, batch.End, len(primes))
    }

    fmt.Printf("🏁 Worker %d terminé (%d batches traités)\n", id, batchesProcessed)
}

func optimizedPrimeWorkerPool() {
    const (
        numWorkers = 4
        maxNumber  = 10000
        batchSize  = 500
    )

    batches := make(chan PrimeBatch, 20)
    results := make(chan PrimeBatchResult, 20)

    var wg sync.WaitGroup

    startTime := time.Now()

    // Démarrer les workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go primeBatchWorker(w, batches, results, &wg)
    }

    // Créer les batches
    go func() {
        for start := 1; start <= maxNumber; start += batchSize {
            end := start + batchSize - 1
            if end > maxNumber {
                end = maxNumber
            }

            batch := PrimeBatch{
                Start: start,
                End:   end,
            }
            batches <- batch
        }
        close(batches)
    }()

    // Collecter les résultats
    go func() {
        wg.Wait()
        close(results)
    }()

    // Analyser les résultats
    var allPrimes []int
    var totalNumbers int

    for result := range results {
        allPrimes = append(allPrimes, result.Primes...)
        totalNumbers += result.Count
    }

    fmt.Printf("🎉 Version optimisée terminée en %v\n", time.Since(startTime))
    fmt.Printf("📊 %d nombres premiers trouvés sur %d testés\n", len(allPrimes), totalNumbers)
}
```

---

## Exercice 2 : Simulateur de requêtes HTTP

### Solution complète

```go
package main

import (
    "errors"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// HTTPRequest représente une requête HTTP à traiter
type HTTPRequest struct {
    ID       int
    URL      string
    Method   string
    Retries  int
    MaxRetries int
}

// HTTPResponse représente la réponse d'une requête
type HTTPResponse struct {
    RequestID    int
    URL          string
    StatusCode   int
    Success      bool
    Error        error
    Duration     time.Duration
    Attempts     int
}

// HTTPStats contient les statistiques globales
type HTTPStats struct {
    TotalRequests    int
    SuccessfulReqs   int
    FailedReqs       int
    TotalDuration    time.Duration
    AverageDuration  time.Duration
    TotalRetries     int
    MaxDuration      time.Duration
    MinDuration      time.Duration
}

// simulateHTTPRequest simule un appel HTTP avec latence et erreurs aléatoires
func simulateHTTPRequest(req HTTPRequest) HTTPResponse {
    start := time.Now()

    // Latence variable entre 50ms et 500ms
    latency := time.Duration(50+rand.Intn(450)) * time.Millisecond
    time.Sleep(latency)

    // 20% de chance d'erreur
    success := rand.Float32() > 0.2

    response := HTTPResponse{
        RequestID: req.ID,
        URL:       req.URL,
        Duration:  time.Since(start),
        Attempts:  req.Retries + 1,
    }

    if success {
        // Codes de succès possibles
        codes := []int{200, 201, 202, 204}
        response.StatusCode = codes[rand.Intn(len(codes))]
        response.Success = true
    } else {
        // Codes d'erreur possibles
        codes := []int{400, 401, 403, 404, 500, 502, 503, 504}
        response.StatusCode = codes[rand.Intn(len(codes))]
        response.Success = false
        response.Error = fmt.Errorf("HTTP %d: request failed", response.StatusCode)
    }

    return response
}

// httpWorker traite les requêtes HTTP avec retry automatique
func httpWorker(id int, requests <-chan HTTPRequest, responses chan<- HTTPResponse,
                retryQueue chan<- HTTPRequest, wg *sync.WaitGroup) {
    defer wg.Done()

    processed := 0
    for req := range requests {
        fmt.Printf("🌐 Worker %d traite requête %d: %s (tentative %d)\n",
                  id, req.ID, req.URL, req.Retries+1)

        response := simulateHTTPRequest(req)

        // Si échec et des tentatives restantes
        if !response.Success && req.Retries < req.MaxRetries {
            fmt.Printf("⚠️ Worker %d: échec requête %d (code %d), retry...\n",
                      id, req.ID, response.StatusCode)

            // Programmer un retry avec backoff
            retryReq := req
            retryReq.Retries++

            go func() {
                // Backoff exponentiel: 100ms, 200ms, 400ms...
                backoff := time.Duration(100*(1<<uint(req.Retries))) * time.Millisecond
                time.Sleep(backoff)
                retryQueue <- retryReq
            }()
        } else {
            // Requête terminée (succès ou échecs max atteints)
            if response.Success {
                fmt.Printf("✅ Worker %d: requête %d réussie (code %d) en %v\n",
                          id, req.ID, response.StatusCode, response.Duration)
            } else {
                fmt.Printf("❌ Worker %d: requête %d définitivement échouée après %d tentatives\n",
                          id, req.ID, response.Attempts)
            }

            responses <- response
            processed++
        }
    }

    fmt.Printf("🏁 Worker %d terminé (%d requêtes traitées)\n", id, processed)
}

// retryHandler gère la file des retry
func retryHandler(retryQueue <-chan HTTPRequest, mainQueue chan<- HTTPRequest, done <-chan bool) {
    for {
        select {
        case req := <-retryQueue:
            mainQueue <- req
        case <-done:
            return
        }
    }
}

func httpSimulatorPool() {
    const (
        numWorkers     = 5
        numRequests    = 50
        maxRetries     = 2
        queueSize      = 100
    )

    // Channels
    requests := make(chan HTTPRequest, queueSize)
    responses := make(chan HTTPResponse, queueSize)
    retryQueue := make(chan HTTPRequest, queueSize)
    done := make(chan bool)

    var wg sync.WaitGroup

    fmt.Printf("🚀 Démarrage du simulateur HTTP avec %d workers\n", numWorkers)
    fmt.Printf("📊 Configuration: %d requêtes, %d tentatives max par requête\n",
              numRequests, maxRetries+1)

    startTime := time.Now()

    // Démarrer le gestionnaire de retry
    go retryHandler(retryQueue, requests, done)

    // Démarrer les workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go httpWorker(w, requests, responses, retryQueue, &wg)
    }

    // Générer les requêtes initiales
    go func() {
        endpoints := []string{
            "https://api.example.com/users",
            "https://api.example.com/posts",
            "https://api.example.com/comments",
            "https://api.example.com/albums",
            "https://api.example.com/photos",
        }

        methods := []string{"GET", "POST", "PUT", "DELETE"}

        for i := 1; i <= numRequests; i++ {
            req := HTTPRequest{
                ID:         i,
                URL:        endpoints[rand.Intn(len(endpoints))],
                Method:     methods[rand.Intn(len(methods))],
                Retries:    0,
                MaxRetries: maxRetries,
            }
            requests <- req
        }
    }()

    // Collecter les réponses
    go func() {
        wg.Wait()
        close(responses)
        done <- true
    }()

    // Analyser les résultats
    stats := HTTPStats{
        MinDuration: time.Hour, // Valeur initiale élevée
    }

    statusCodes := make(map[int]int)
    var successDurations []time.Duration

    fmt.Println("📊 Collecte des résultats...")

    for response := range responses {
        stats.TotalRequests++
        stats.TotalDuration += response.Duration
        stats.TotalRetries += response.Attempts - 1

        // Mettre à jour min/max duration
        if response.Duration > stats.MaxDuration {
            stats.MaxDuration = response.Duration
        }
        if response.Duration < stats.MinDuration {
            stats.MinDuration = response.Duration
        }

        // Compter les codes de statut
        statusCodes[response.StatusCode]++

        if response.Success {
            stats.SuccessfulReqs++
            successDurations = append(successDurations, response.Duration)
        } else {
            stats.FailedReqs++
        }
    }

    // Calculer la durée moyenne
    if stats.TotalRequests > 0 {
        stats.AverageDuration = stats.TotalDuration / time.Duration(stats.TotalRequests)
    }

    totalElapsed := time.Since(startTime)

    // Afficher les statistiques détaillées
    fmt.Printf("\n🎉 Simulation terminée en %v\n", totalElapsed)
    fmt.Printf("📈 Statistiques globales:\n")
    fmt.Printf("   📊 Requêtes totales: %d\n", stats.TotalRequests)
    fmt.Printf("   ✅ Succès: %d (%.1f%%)\n", stats.SuccessfulReqs,
              float64(stats.SuccessfulReqs)/float64(stats.TotalRequests)*100)
    fmt.Printf("   ❌ Échecs: %d (%.1f%%)\n", stats.FailedReqs,
              float64(stats.FailedReqs)/float64(stats.TotalRequests)*100)
    fmt.Printf("   🔄 Total retry: %d\n", stats.TotalRetries)
    fmt.Printf("   ⏱️ Durée moyenne: %v\n", stats.AverageDuration)
    fmt.Printf("   ⚡ Durée min: %v\n", stats.MinDuration)
    fmt.Printf("   🐌 Durée max: %v\n", stats.MaxDuration)
    fmt.Printf("   🚀 Requêtes/sec: %.1f\n", float64(stats.TotalRequests)/totalElapsed.Seconds())

    // Afficher la distribution des codes de statut
    fmt.Printf("\n📊 Distribution des codes de statut:\n")
    for code, count := range statusCodes {
        percentage := float64(count) / float64(stats.TotalRequests) * 100
        emoji := "✅"
        if code >= 400 {
            emoji = "❌"
        }
        fmt.Printf("   %s %d: %d (%.1f%%)\n", emoji, code, count, percentage)
    }

    // Calculer la médiane pour les requêtes réussies
    if len(successDurations) > 0 {
        // Tri simple pour trouver la médiane
        for i := 0; i < len(successDurations)-1; i++ {
            for j := 0; j < len(successDurations)-i-1; j++ {
                if successDurations[j] > successDurations[j+1] {
                    successDurations[j], successDurations[j+1] = successDurations[j+1], successDurations[j]
                }
            }
        }

        median := successDurations[len(successDurations)/2]
        fmt.Printf("\n📐 Médiane des requêtes réussies: %v\n", median)
    }
}

// Version avec métriques en temps réel
func httpSimulatorWithMetrics() {
    // Métriques en temps réel
    metrics := struct {
        sync.RWMutex
        completed   int
        successful  int
        failed      int
        inProgress  int
    }{}

    // Moniteur des métriques
    go func() {
        ticker := time.NewTicker(2 * time.Second)
        defer ticker.Stop()

        for range ticker.C {
            metrics.RLock()
            fmt.Printf("📊 Temps réel - Terminées: %d, Succès: %d, Échecs: %d, En cours: %d\n",
                      metrics.completed, metrics.successful, metrics.failed, metrics.inProgress)
            metrics.RUnlock()
        }
    }()

    // Le reste de l'implémentation avec mise à jour des métriques...
}

func main() {
    rand.Seed(time.Now().UnixNano())

    fmt.Println("=== Exercice 1: Calculateur de nombres premiers ===")
    primeWorkerPool()

    fmt.Println("\n=== Version optimisée avec batches ===")
    optimizedPrimeWorkerPool()

    fmt.Println("\n=== Exercice 2: Simulateur de requêtes HTTP ===")
    httpSimulatorPool()
}
```

## Points clés des solutions

### Exercice 1 - Nombres premiers

**Fonctionnalités** :
- ✅ Algorithme optimisé de test de primalité
- ✅ Worker pool avec 6 workers
- ✅ Statistiques détaillées (temps, nombres trouvés)
- ✅ Version optimisée avec batch processing
- ✅ Suivi des performances en temps réel

**Optimisations** :
- Test jusqu'à √n seulement
- Éviter les nombres pairs
- Batch processing pour réduire l'overhead

### Exercice 2 - Simulateur HTTP

**Fonctionnalités** :
- ✅ Simulation réaliste avec latences variables
- ✅ 20% de taux d'erreur comme demandé
- ✅ Retry automatique avec backoff exponentiel
- ✅ 3 tentatives maximum par requête
- ✅ Statistiques complètes et codes de statut
- ✅ Métriques en temps réel optionnelles

**Patterns utilisés** :
- Queue séparée pour les retry
- Backoff exponentiel (100ms, 200ms, 400ms...)
- Gestion des codes HTTP réalistes
- Monitoring en temps réel

Ces solutions démontrent l'utilisation pratique des worker pools pour des problèmes réels avec gestion d'erreurs, retry, et monitoring !

⏭️
