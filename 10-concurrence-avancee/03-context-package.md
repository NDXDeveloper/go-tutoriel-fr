🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10-3 : Context package

## Introduction

Le package `context` est l'un des outils les plus importants pour gérer la concurrence en Go. Il permet de gérer les **timeouts**, l'**annulation** de tâches, et le **passage de valeurs** entre goroutines. Si vous développez des applications Go pour la production, maîtriser le context est essentiel.

## Qu'est-ce qu'un Context ?

Un **Context** est comme un "signal de contrôle" qui voyage avec vos goroutines. Il peut porter trois types d'informations :

1. **Signal d'annulation** : "Arrête ce que tu fais"
2. **Deadline/Timeout** : "Tu as jusqu'à telle heure pour finir"
3. **Valeurs** : "Voici des données que tu peux utiliser"

### Analogie du monde réel

Imaginez que vous êtes un chef de projet qui envoie des équipes sur différentes missions :

- **Timeout** : "Vous avez 2 heures maximum pour cette mission"
- **Annulation** : "Mission annulée, revenez immédiatement"
- **Valeurs** : "Voici l'ID de mission et les clés d'accès"

C'est exactement ce que fait un Context !

## Concepts de base

### Interface Context

```go
type Context interface {
    // Deadline retourne le moment où le context expire
    Deadline() (deadline time.Time, ok bool)

    // Done retourne un channel qui se ferme quand le context est annulé
    Done() <-chan struct{}

    // Err retourne l'erreur si le context est annulé
    Err() error

    // Value retourne la valeur associée à une clé
    Value(key interface{}) interface{}
}
```

### Les quatre types de Context

1. **Background** : Context racine, jamais annulé
2. **WithCancel** : Peut être annulé manuellement
3. **WithTimeout** : S'annule après un délai
4. **WithDeadline** : S'annule à un moment précis

## 1. Context Background

C'est le context de base, point de départ de tous les autres.

```go
package main

import (
    "context"
    "fmt"
)

func basicContextExample() {
    // Créer un context de base
    ctx := context.Background()

    fmt.Printf("Context créé: %v\n", ctx)
    fmt.Printf("Deadline: %v\n", ctx.Deadline())  // false, pas de deadline
    fmt.Printf("Done channel: %v\n", ctx.Done())  // nil, jamais fermé
    fmt.Printf("Error: %v\n", ctx.Err())          // nil, pas d'erreur
}
```

**Utilisation** : Point de départ dans `main()`, tests, ou quand aucune annulation n'est nécessaire.

## 2. Context avec annulation manuelle

### WithCancel

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func cancelContextExample() {
    // Créer un context avec possibilité d'annulation
    ctx, cancel := context.WithCancel(context.Background())

    // Lancer une goroutine qui fait du travail
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("🛑 Goroutine arrêtée:", ctx.Err())
                return
            default:
                fmt.Println("⚙️ Travail en cours...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()

    // Laisser tourner 2 secondes puis annuler
    time.Sleep(2 * time.Second)
    fmt.Println("📢 Annulation du context...")
    cancel() // Déclenche l'annulation

    // Attendre que la goroutine se termine
    time.Sleep(100 * time.Millisecond)
    fmt.Println("✅ Programme terminé")
}
```

### Sortie attendue

```
⚙️ Travail en cours...
⚙️ Travail en cours...
⚙️ Travail en cours...
⚙️ Travail en cours...
📢 Annulation du context...
🛑 Goroutine arrêtée: context canceled
✅ Programme terminé
```

### Exemple avec plusieurs goroutines

```go
func multipleGoRoutinesCancel() {
    ctx, cancel := context.WithCancel(context.Background())

    // Lancer plusieurs workers
    for i := 1; i <= 3; i++ {
        go worker(ctx, i)
    }

    // Annuler après 3 secondes
    time.Sleep(3 * time.Second)
    fmt.Println("📢 Arrêt de tous les workers...")
    cancel()

    time.Sleep(500 * time.Millisecond)
}

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("🛑 Worker %d arrêté: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("👷 Worker %d travaille...\n", id)
            time.Sleep(800 * time.Millisecond)
        }
    }
}
```

## 3. Context avec timeout

### WithTimeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func timeoutContextExample() {
    // Context qui s'annule automatiquement après 2 secondes
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel() // Bonne pratique : toujours appeler cancel()

    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("⏰ Timeout atteint:", ctx.Err())
                return
            default:
                fmt.Println("🔄 Traitement en cours...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()

    // Attendre que le timeout se déclenche
    time.Sleep(3 * time.Second)
    fmt.Println("✅ Fin du programme")
}
```

### Exemple pratique : Appel HTTP avec timeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Simulation d'un appel HTTP
func fetchData(ctx context.Context, url string) (string, error) {
    // Channel pour recevoir le résultat
    resultChan := make(chan string, 1)
    errorChan := make(chan error, 1)

    // Lancer la requête dans une goroutine
    go func() {
        // Simulation d'un appel qui prend du temps
        fmt.Printf("🌐 Début de la requête vers %s...\n", url)
        time.Sleep(3 * time.Second) // Simulation latence réseau

        // Vérifier si le context n'est pas annulé
        select {
        case <-ctx.Done():
            errorChan <- ctx.Err()
            return
        default:
            resultChan <- fmt.Sprintf("Données de %s", url)
        }
    }()

    // Attendre le résultat ou le timeout
    select {
    case result := <-resultChan:
        return result, nil
    case err := <-errorChan:
        return "", err
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func httpTimeoutExample() {
    // Context avec timeout de 2 secondes
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    start := time.Now()

    data, err := fetchData(ctx, "https://api.example.com/data")

    elapsed := time.Since(start)

    if err != nil {
        fmt.Printf("❌ Erreur après %v: %v\n", elapsed, err)
    } else {
        fmt.Printf("✅ Succès après %v: %s\n", elapsed, data)
    }
}
```

## 4. Context avec deadline

### WithDeadline

```go
func deadlineContextExample() {
    // Context qui expire à un moment précis
    deadline := time.Now().Add(1500 * time.Millisecond)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    fmt.Printf("⏰ Deadline fixée à: %v\n", deadline.Format("15:04:05.000"))

    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Printf("⏰ Deadline atteinte à %v: %v\n",
                          time.Now().Format("15:04:05.000"), ctx.Err())
                return
            default:
                fmt.Printf("🔄 Travail en cours à %v\n",
                          time.Now().Format("15:04:05.000"))
                time.Sleep(300 * time.Millisecond)
            }
        }
    }()

    time.Sleep(2 * time.Second)
}
```

## 5. Context avec valeurs

### WithValue

```go
package main

import (
    "context"
    "fmt"
)

// Clés typées pour éviter les collisions
type contextKey string

const (
    UserIDKey    contextKey = "userID"
    RequestIDKey contextKey = "requestID"
)

func contextWithValuesExample() {
    // Context avec des valeurs
    ctx := context.Background()

    // Ajouter des valeurs
    ctx = context.WithValue(ctx, UserIDKey, "user123")
    ctx = context.WithValue(ctx, RequestIDKey, "req456")

    // Passer le context à une fonction
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    // Récupérer les valeurs du context
    userID := ctx.Value(UserIDKey)
    requestID := ctx.Value(RequestIDKey)

    fmt.Printf("👤 Traitement pour l'utilisateur: %v\n", userID)
    fmt.Printf("📝 ID de requête: %v\n", requestID)

    // Passer à d'autres fonctions
    saveToDatabase(ctx)
    sendNotification(ctx)
}

func saveToDatabase(ctx context.Context) {
    userID := ctx.Value(UserIDKey)
    fmt.Printf("💾 Sauvegarde en base pour l'utilisateur: %v\n", userID)
}

func sendNotification(ctx context.Context) {
    userID := ctx.Value(UserIDKey)
    requestID := ctx.Value(RequestIDKey)
    fmt.Printf("📧 Notification envoyée (user: %v, req: %v)\n", userID, requestID)
}
```

### Bonnes pratiques pour les valeurs

```go
// ✅ Utiliser des types personnalisés pour les clés
type userContextKey struct{}
type requestContextKey struct{}

func betterContextValues() {
    ctx := context.Background()

    // Clés typées (plus sûr)
    ctx = context.WithValue(ctx, userContextKey{}, "user123")
    ctx = context.WithValue(ctx, requestContextKey{}, "req456")

    // Helper functions pour type safety
    userID := getUserID(ctx)
    requestID := getRequestID(ctx)

    fmt.Printf("User: %s, Request: %s\n", userID, requestID)
}

// Helper functions pour extraire les valeurs
func getUserID(ctx context.Context) string {
    if userID, ok := ctx.Value(userContextKey{}).(string); ok {
        return userID
    }
    return ""
}

func getRequestID(ctx context.Context) string {
    if reqID, ok := ctx.Value(requestContextKey{}).(string); ok {
        return reqID
    }
    return ""
}
```

## Combinaison de contexts

### Timeout + Valeurs + Annulation

```go
func combinedContextExample() {
    // Context de base avec valeurs
    ctx := context.Background()
    ctx = context.WithValue(ctx, UserIDKey, "user789")

    // Ajouter un timeout
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // Lancer plusieurs tâches
    done := make(chan bool, 2)

    go longRunningTask(ctx, "Tâche 1", done)
    go longRunningTask(ctx, "Tâche 2", done)

    // Arrêter manuellement après 2 secondes pour montrer l'annulation
    go func() {
        time.Sleep(2 * time.Second)
        fmt.Println("📢 Annulation manuelle...")
        cancel()
    }()

    // Attendre que les tâches se terminent
    for i := 0; i < 2; i++ {
        <-done
    }

    fmt.Println("✅ Toutes les tâches terminées")
}

func longRunningTask(ctx context.Context, name string, done chan<- bool) {
    defer func() {
        done <- true
    }()

    userID := ctx.Value(UserIDKey)
    fmt.Printf("🚀 %s démarrée pour l'utilisateur %v\n", name, userID)

    for i := 1; i <= 10; i++ {
        select {
        case <-ctx.Done():
            fmt.Printf("🛑 %s arrêtée à l'étape %d: %v\n", name, i, ctx.Err())
            return
        default:
            fmt.Printf("⚙️ %s - étape %d/10\n", name, i)
            time.Sleep(500 * time.Millisecond)
        }
    }

    fmt.Printf("✅ %s terminée avec succès\n", name)
}
```

## Context dans un serveur HTTP

### Exemple pratique avec serveur

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func contextHTTPServerExample() {
    http.HandleFunc("/api/data", dataHandler)
    http.HandleFunc("/api/slow", slowHandler)

    fmt.Println("🚀 Serveur démarré sur :8080")
    fmt.Println("Testez avec:")
    fmt.Println("  curl http://localhost:8080/api/data")
    fmt.Println("  curl http://localhost:8080/api/slow")

    http.ListenAndServe(":8080", nil)
}

func dataHandler(w http.ResponseWriter, r *http.Request) {
    // Le context de la requête HTTP
    ctx := r.Context()

    // Ajouter des informations au context
    ctx = context.WithValue(ctx, RequestIDKey, generateRequestID())

    // Traiter la requête avec le context
    result, err := processDataRequest(ctx)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    fmt.Fprintf(w, "Résultat: %s\n", result)
}

func slowHandler(w http.ResponseWriter, r *http.Request) {
    // Context avec timeout pour cette requête
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    result, err := slowOperation(ctx)
    if err != nil {
        if err == context.DeadlineExceeded {
            http.Error(w, "Timeout: opération trop lente", http.StatusRequestTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    fmt.Fprintf(w, "Résultat lent: %s\n", result)
}

func processDataRequest(ctx context.Context) (string, error) {
    requestID := ctx.Value(RequestIDKey)

    // Simulation d'un traitement rapide
    select {
    case <-time.After(100 * time.Millisecond):
        return fmt.Sprintf("Données traitées (ID: %v)", requestID), nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func slowOperation(ctx context.Context) (string, error) {
    // Simulation d'une opération qui prend 5 secondes
    select {
    case <-time.After(5 * time.Second):
        return "Opération lente terminée", nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func generateRequestID() string {
    return fmt.Sprintf("req_%d", time.Now().UnixNano())
}
```

## Context avec base de données

### Exemple avec requêtes DB

```go
package main

import (
    "context"
    "fmt"
    "time"
)

// Simulation d'une base de données
type Database struct {
    name string
}

func (db *Database) Query(ctx context.Context, query string) ([]string, error) {
    fmt.Printf("🗄️ Exécution de la requête: %s\n", query)

    // Simulation d'une requête qui prend du temps
    select {
    case <-time.After(1 * time.Second):
        return []string{"résultat1", "résultat2", "résultat3"}, nil
    case <-ctx.Done():
        fmt.Printf("❌ Requête annulée: %v\n", ctx.Err())
        return nil, ctx.Err()
    }
}

func databaseContextExample() {
    db := &Database{name: "ProductionDB"}

    // Context avec timeout pour les requêtes DB
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    // Ajouter des informations de traçage
    ctx = context.WithValue(ctx, "trace_id", "trace_12345")

    // Exécuter plusieurs requêtes
    queries := []string{
        "SELECT * FROM users",
        "SELECT * FROM orders",
        "SELECT * FROM products",
    }

    for _, query := range queries {
        result, err := db.Query(ctx, query)
        if err != nil {
            fmt.Printf("❌ Erreur pour la requête '%s': %v\n", query, err)
            continue
        }
        fmt.Printf("✅ Résultats pour '%s': %v\n", query, result)
    }
}
```

## Patterns avancés

### 1. Context pipeline

```go
func contextPipeline() {
    ctx := context.Background()
    ctx = context.WithValue(ctx, "step", 0)

    // Pipeline de traitement
    result := ctx
    result = step1(result)
    result = step2(result)
    result = step3(result)

    fmt.Printf("Résultat final: step = %v\n", result.Value("step"))
}

func step1(ctx context.Context) context.Context {
    step := ctx.Value("step").(int)
    fmt.Printf("🔄 Étape 1 (step=%d)\n", step)
    return context.WithValue(ctx, "step", step+1)
}

func step2(ctx context.Context) context.Context {
    step := ctx.Value("step").(int)
    fmt.Printf("🔄 Étape 2 (step=%d)\n", step)
    return context.WithValue(ctx, "step", step+1)
}

func step3(ctx context.Context) context.Context {
    step := ctx.Value("step").(int)
    fmt.Printf("🔄 Étape 3 (step=%d)\n", step)
    return context.WithValue(ctx, "step", step+1)
}
```

### 2. Context avec retry

```go
func retryWithContext(ctx context.Context, maxAttempts int, operation func() error) error {
    for attempt := 1; attempt <= maxAttempts; attempt++ {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            err := operation()
            if err == nil {
                return nil
            }

            fmt.Printf("⚠️ Tentative %d échouée: %v\n", attempt, err)

            if attempt < maxAttempts {
                // Backoff avec context
                backoff := time.Duration(attempt) * 100 * time.Millisecond
                timer := time.NewTimer(backoff)

                select {
                case <-timer.C:
                    // Continuer
                case <-ctx.Done():
                    timer.Stop()
                    return ctx.Err()
                }
            }
        }
    }

    return fmt.Errorf("échec après %d tentatives", maxAttempts)
}
```

## Bonnes pratiques

### ✅ À faire

1. **Toujours passer Context en premier paramètre**
   ```go
   func myFunction(ctx context.Context, param string) error
   ```

2. **Utiliser defer cancel()**
   ```go
   ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
   defer cancel() // Toujours appeler cancel()
   ```

3. **Vérifier ctx.Done() dans les boucles**
   ```go
   for {
       select {
       case <-ctx.Done():
           return ctx.Err()
       default:
           // Faire le travail
       }
   }
   ```

4. **Utiliser des clés typées pour les valeurs**
   ```go
   type myKey struct{}
   ctx = context.WithValue(ctx, myKey{}, "value")
   ```

### ❌ À éviter

1. **Ne jamais passer nil context**
   ```go
   // ❌ Mauvais
   myFunction(nil, "param")

   // ✅ Bon
   myFunction(context.Background(), "param")
   ```

2. **Ne pas stocker context dans des structs**
   ```go
   // ❌ Mauvais
   type MyStruct struct {
       ctx context.Context
   }

   // ✅ Bon : passer en paramètre
   func (m *MyStruct) DoWork(ctx context.Context)
   ```

3. **Ne pas ignorer l'annulation**
   ```go
   // ❌ Mauvais : ne vérifie pas l'annulation
   for i := 0; i < 1000; i++ {
       heavyWork()
   }

   // ✅ Bon : vérifie l'annulation
   for i := 0; i < 1000; i++ {
       select {
       case <-ctx.Done():
           return ctx.Err()
       default:
           heavyWork()
       }
   }
   ```

## Exercices pratiques

### Exercice 1 : Worker avec timeout

Créez un worker qui traite des tâches avec un timeout global.

```go
func timeoutWorkerExercise() {
    // Votre implémentation ici
    // Indices :
    // - Context avec timeout de 5 secondes
    // - Worker qui traite 10 tâches
    // - Chaque tâche prend 1 seconde
    // - Gérer l'annulation proprement
}
```

### Exercice 2 : Pipeline avec valeurs

Créez un pipeline de traitement qui enrichit les données à chaque étape.

```go
func enrichmentPipelineExercise() {
    // Votre implémentation ici
    // Indices :
    // - Context avec des valeurs initiales
    // - 3 étapes qui ajoutent des informations
    // - Timeout global de 3 secondes
    // - Retourner l'objet enrichi final
}
```

## Résumé

Le package `context` est essentiel pour :

- **Gérer les timeouts** et deadlines
- **Annuler des opérations** en cours
- **Passer des valeurs** entre goroutines
- **Coordonner** plusieurs tâches concurrentes

**Types principaux** :
- `context.Background()` : Point de départ
- `WithCancel()` : Annulation manuelle
- `WithTimeout()` : Timeout automatique
- `WithDeadline()` : Expiration à un moment précis
- `WithValue()` : Transport de valeurs

**Pattern de base** :
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case <-ctx.Done():
    return ctx.Err()
default:
    // Faire le travail
}
```

La maîtrise du package `context` vous permettra de construire des applications Go robustes et professionnelles !

⏭️

# Solutions des exercices pratiques - Context

## Exercice 1 : Worker avec timeout

### Solution complète

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

// Task représente une tâche à traiter
type Task struct {
    ID       int
    Name     string
    Duration time.Duration
}

// TaskResult représente le résultat d'une tâche
type TaskResult struct {
    TaskID    int
    Success   bool
    Error     error
    Elapsed   time.Duration
    Worker    int
}

// timeoutWorker traite les tâches avec gestion du timeout
func timeoutWorker(ctx context.Context, id int, tasks <-chan Task, results chan<- TaskResult, wg *sync.WaitGroup) {
    defer wg.Done()

    processed := 0
    for {
        select {
        case task, ok := <-tasks:
            if !ok {
                fmt.Printf("🏁 Worker %d terminé (%d tâches traitées)\n", id, processed)
                return
            }

            fmt.Printf("⚙️ Worker %d commence la tâche %d (%s)\n", id, task.ID, task.Name)
            start := time.Now()

            // Traiter la tâche avec vérification du context
            result := processTaskWithContext(ctx, task, id)
            result.Elapsed = time.Since(start)

            // Envoyer le résultat (non-bloquant avec context)
            select {
            case results <- result:
                processed++
                if result.Success {
                    fmt.Printf("✅ Worker %d: tâche %d terminée avec succès en %v\n",
                              id, task.ID, result.Elapsed)
                } else {
                    fmt.Printf("❌ Worker %d: tâche %d échouée - %v\n",
                              id, task.ID, result.Error)
                }
            case <-ctx.Done():
                fmt.Printf("🛑 Worker %d: arrêt pendant l'envoi du résultat (tâche %d)\n", id, task.ID)
                return
            }

        case <-ctx.Done():
            fmt.Printf("🛑 Worker %d arrêté par timeout: %v (%d tâches traitées)\n",
                      id, ctx.Err(), processed)
            return
        }
    }
}

// processTaskWithContext traite une tâche en respectant le context
func processTaskWithContext(ctx context.Context, task Task, workerID int) TaskResult {
    result := TaskResult{
        TaskID: task.ID,
        Worker: workerID,
    }

    // Créer un timer pour la durée de la tâche
    timer := time.NewTimer(task.Duration)
    defer timer.Stop()

    select {
    case <-timer.C:
        // Tâche terminée normalement
        result.Success = true
        return result

    case <-ctx.Done():
        // Timeout global ou annulation
        result.Success = false
        result.Error = fmt.Errorf("tâche annulée: %v", ctx.Err())
        return result
    }
}

func timeoutWorkerExercise() {
    const (
        numWorkers    = 3
        numTasks      = 10
        globalTimeout = 5 * time.Second
    )

    // Créer un context avec timeout global
    ctx, cancel := context.WithTimeout(context.Background(), globalTimeout)
    defer cancel()

    // Channels
    tasks := make(chan Task, numTasks)
    results := make(chan TaskResult, numTasks)

    var wg sync.WaitGroup

    fmt.Printf("🚀 Démarrage de %d workers avec timeout global de %v\n", numWorkers, globalTimeout)
    start := time.Now()

    // Démarrer les workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go timeoutWorker(ctx, i, tasks, results, &wg)
    }

    // Générer les tâches
    go func() {
        defer close(tasks)

        for i := 1; i <= numTasks; i++ {
            task := Task{
                ID:       i,
                Name:     fmt.Sprintf("Tâche-%d", i),
                Duration: 1 * time.Second, // Chaque tâche prend 1 seconde
            }

            select {
            case tasks <- task:
                fmt.Printf("📋 Tâche %d ajoutée à la queue\n", i)
            case <-ctx.Done():
                fmt.Printf("⚠️ Arrêt de la génération de tâches: %v\n", ctx.Err())
                return
            }
        }
        fmt.Println("📋 Toutes les tâches ont été ajoutées")
    }()

    // Collecter les résultats dans une goroutine séparée
    go func() {
        wg.Wait()
        close(results)
    }()

    // Analyser les résultats
    var successful, failed int
    var totalTime time.Duration
    var completedTasks []TaskResult

    fmt.Println("📊 Collecte des résultats...")

    for result := range results {
        completedTasks = append(completedTasks, result)
        totalTime += result.Elapsed

        if result.Success {
            successful++
        } else {
            failed++
        }
    }

    elapsed := time.Since(start)

    // Afficher les statistiques
    fmt.Printf("\n🎉 Exercice terminé en %v\n", elapsed)
    fmt.Printf("📈 Statistiques:\n")
    fmt.Printf("   📝 Tâches prévues: %d\n", numTasks)
    fmt.Printf("   ✅ Tâches réussies: %d\n", successful)
    fmt.Printf("   ❌ Tâches échouées: %d\n", failed)
    fmt.Printf("   ⏱️ Temps total de traitement: %v\n", totalTime)

    if len(completedTasks) > 0 {
        avgTime := totalTime / time.Duration(len(completedTasks))
        fmt.Printf("   📊 Temps moyen par tâche: %v\n", avgTime)
    }

    // Détail par worker
    workerStats := make(map[int]int)
    for _, result := range completedTasks {
        if result.Success {
            workerStats[result.Worker]++
        }
    }

    fmt.Println("   👷 Répartition par worker:")
    for workerID, count := range workerStats {
        fmt.Printf("     Worker %d: %d tâches réussies\n", workerID, count)
    }

    // Vérifier si le timeout a été atteint
    if elapsed >= globalTimeout {
        fmt.Printf("⏰ Le timeout global de %v a été atteint\n", globalTimeout)
    } else {
        fmt.Printf("⚡ Terminé avant le timeout (reste %v)\n", globalTimeout-elapsed)
    }
}
```

### Version avec retry automatique

```go
// Version avancée avec retry en cas d'échec
func advancedTimeoutWorkerExercise() {
    const (
        numWorkers    = 2
        numTasks      = 8
        globalTimeout = 6 * time.Second
        maxRetries    = 2
    )

    ctx, cancel := context.WithTimeout(context.Background(), globalTimeout)
    defer cancel()

    tasks := make(chan Task, numTasks*2) // Buffer plus grand pour les retry
    results := make(chan TaskResult, numTasks)

    var wg sync.WaitGroup

    // Workers avec retry
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go advancedTimeoutWorker(ctx, i, tasks, results, &wg, maxRetries)
    }

    // Générer les tâches avec chance d'échec
    go func() {
        defer close(tasks)

        for i := 1; i <= numTasks; i++ {
            task := Task{
                ID:       i,
                Name:     fmt.Sprintf("Tâche-fragile-%d", i),
                Duration: 800 * time.Millisecond,
            }
            tasks <- task
        }
    }()

    // Collecter les résultats
    go func() {
        wg.Wait()
        close(results)
    }()

    // Analyser les résultats
    for result := range results {
        status := "✅"
        if !result.Success {
            status = "❌"
        }
        fmt.Printf("%s Tâche %d (Worker %d): %v\n",
                  status, result.TaskID, result.Worker, result.Error)
    }
}

func advancedTimeoutWorker(ctx context.Context, id int, tasks <-chan Task,
                          results chan<- TaskResult, wg *sync.WaitGroup, maxRetries int) {
    defer wg.Done()

    for {
        select {
        case task, ok := <-tasks:
            if !ok {
                return
            }

            // Essayer la tâche avec retry
            result := processTaskWithRetry(ctx, task, id, maxRetries)

            select {
            case results <- result:
            case <-ctx.Done():
                return
            }

        case <-ctx.Done():
            fmt.Printf("🛑 Worker %d arrêté par timeout\n", id)
            return
        }
    }
}

func processTaskWithRetry(ctx context.Context, task Task, workerID, maxRetries int) TaskResult {
    for attempt := 1; attempt <= maxRetries+1; attempt++ {
        select {
        case <-ctx.Done():
            return TaskResult{
                TaskID:  task.ID,
                Worker:  workerID,
                Success: false,
                Error:   ctx.Err(),
            }
        default:
            // Simulation: 30% de chance d'échec
            if attempt < maxRetries+1 && time.Now().UnixNano()%10 < 3 {
                fmt.Printf("⚠️ Worker %d: échec tâche %d (tentative %d)\n",
                          workerID, task.ID, attempt)
                time.Sleep(100 * time.Millisecond) // Petit délai avant retry
                continue
            }

            // Traitement réussi
            time.Sleep(task.Duration)
            return TaskResult{
                TaskID:  task.ID,
                Worker:  workerID,
                Success: true,
            }
        }
    }

    return TaskResult{
        TaskID:  task.ID,
        Worker:  workerID,
        Success: false,
        Error:   fmt.Errorf("échec après %d tentatives", maxRetries+1),
    }
}
```

---

## Exercice 2 : Pipeline avec valeurs

### Solution complète

```go
package main

import (
    "context"
    "fmt"
    "strings"
    "time"
)

// DataObject représente l'objet enrichi à chaque étape
type DataObject struct {
    ID          string
    RawData     string
    ProcessedBy []string
    Metadata    map[string]interface{}
    Timestamp   time.Time
}

// Clés pour le context
type contextKey string

const (
    UserIDKey     contextKey = "userID"
    RequestIDKey  contextKey = "requestID"
    TraceIDKey    contextKey = "traceID"
    PipelineIDKey contextKey = "pipelineID"
)

// enrichmentStep1 ajoute des informations de base
func enrichmentStep1(ctx context.Context, data DataObject) (DataObject, error) {
    select {
    case <-ctx.Done():
        return data, ctx.Err()
    default:
        // Simulation du traitement (700ms)
        time.Sleep(700 * time.Millisecond)

        userID := ctx.Value(UserIDKey)
        requestID := ctx.Value(RequestIDKey)

        fmt.Printf("🔄 Étape 1: Normalisation des données (User: %v, Request: %v)\n",
                  userID, requestID)

        // Enrichir l'objet
        data.RawData = strings.ToUpper(strings.TrimSpace(data.RawData))
        data.ProcessedBy = append(data.ProcessedBy, "normalizer")
        data.Metadata["normalized"] = true
        data.Metadata["step1_duration"] = 700
        data.Metadata["processor"] = "step1-normalizer-v1.0"

        fmt.Printf("✅ Étape 1 terminée: %s\n", data.RawData)
        return data, nil
    }
}

// enrichmentStep2 ajoute une validation et des métadonnées
func enrichmentStep2(ctx context.Context, data DataObject) (DataObject, error) {
    select {
    case <-ctx.Done():
        return data, ctx.Err()
    default:
        // Simulation du traitement (800ms)
        time.Sleep(800 * time.Millisecond)

        traceID := ctx.Value(TraceIDKey)
        fmt.Printf("🔍 Étape 2: Validation et enrichissement (Trace: %v)\n", traceID)

        // Validation
        if len(data.RawData) < 3 {
            return data, fmt.Errorf("données trop courtes pour validation")
        }

        // Enrichir l'objet
        data.ProcessedBy = append(data.ProcessedBy, "validator")
        data.Metadata["validated"] = true
        data.Metadata["length"] = len(data.RawData)
        data.Metadata["step2_duration"] = 800
        data.Metadata["validation_rules"] = []string{"min_length", "uppercase_check"}

        // Ajouter des données calculées
        if strings.Contains(data.RawData, "URGENT") {
            data.Metadata["priority"] = "high"
        } else {
            data.Metadata["priority"] = "normal"
        }

        fmt.Printf("✅ Étape 2 terminée: validation OK, priorité %v\n",
                  data.Metadata["priority"])
        return data, nil
    }
}

// enrichmentStep3 finalise le traitement
func enrichmentStep3(ctx context.Context, data DataObject) (DataObject, error) {
    select {
    case <-ctx.Done():
        return data, ctx.Err()
    default:
        // Simulation du traitement (900ms)
        time.Sleep(900 * time.Millisecond)

        pipelineID := ctx.Value(PipelineIDKey)
        fmt.Printf("🎯 Étape 3: Finalisation et indexation (Pipeline: %v)\n", pipelineID)

        // Enrichir l'objet final
        data.ProcessedBy = append(data.ProcessedBy, "finalizer")
        data.Metadata["finalized"] = true
        data.Metadata["step3_duration"] = 900
        data.Metadata["index_key"] = fmt.Sprintf("idx_%s_%d", data.ID, time.Now().Unix())
        data.Metadata["total_processing_steps"] = len(data.ProcessedBy)

        // Calculer le temps total estimé
        totalDuration := 0
        for _, key := range []string{"step1_duration", "step2_duration", "step3_duration"} {
            if duration, ok := data.Metadata[key].(int); ok {
                totalDuration += duration
            }
        }
        data.Metadata["total_duration_ms"] = totalDuration

        fmt.Printf("✅ Étape 3 terminée: objet indexé avec clé %v\n",
                  data.Metadata["index_key"])
        return data, nil
    }
}

func enrichmentPipelineExercise() {
    const pipelineTimeout = 3 * time.Second

    // Créer le context de base avec des valeurs
    ctx := context.Background()
    ctx = context.WithValue(ctx, UserIDKey, "user_12345")
    ctx = context.WithValue(ctx, RequestIDKey, "req_67890")
    ctx = context.WithValue(ctx, TraceIDKey, "trace_abcdef")
    ctx = context.WithValue(ctx, PipelineIDKey, "pipeline_xyz789")

    // Ajouter le timeout global
    ctx, cancel := context.WithTimeout(ctx, pipelineTimeout)
    defer cancel()

    // Données initiales à traiter
    initialData := DataObject{
        ID:          "data_001",
        RawData:     "  urgent customer data processing request  ",
        ProcessedBy: []string{},
        Metadata:    make(map[string]interface{}),
        Timestamp:   time.Now(),
    }

    fmt.Printf("🚀 Démarrage du pipeline d'enrichissement\n")
    fmt.Printf("📋 Données initiales: %q\n", initialData.RawData)
    fmt.Printf("⏰ Timeout global: %v\n", pipelineTimeout)

    start := time.Now()

    // Exécuter le pipeline étape par étape
    result, err := executePipeline(ctx, initialData)

    elapsed := time.Since(start)

    // Afficher les résultats
    if err != nil {
        fmt.Printf("\n❌ Pipeline échoué après %v: %v\n", elapsed, err)

        // Afficher l'état partiel s'il y en a un
        if len(result.ProcessedBy) > 0 {
            fmt.Printf("📊 État partiel après %d étapes:\n", len(result.ProcessedBy))
            fmt.Printf("   Traité par: %v\n", result.ProcessedBy)
            fmt.Printf("   Métadonnées: %v\n", result.Metadata)
        }
    } else {
        fmt.Printf("\n🎉 Pipeline terminé avec succès en %v\n", elapsed)
        displayEnrichedObject(result)
    }

    // Vérifier si le timeout a été atteint
    if err == context.DeadlineExceeded {
        fmt.Printf("⏰ Le timeout de %v a été dépassé\n", pipelineTimeout)
        fmt.Printf("💡 Suggestion: augmenter le timeout ou optimiser les étapes\n")
    }
}

func executePipeline(ctx context.Context, data DataObject) (DataObject, error) {
    var err error

    // Étape 1: Normalisation
    data, err = enrichmentStep1(ctx, data)
    if err != nil {
        return data, fmt.Errorf("erreur étape 1: %w", err)
    }

    // Étape 2: Validation
    data, err = enrichmentStep2(ctx, data)
    if err != nil {
        return data, fmt.Errorf("erreur étape 2: %w", err)
    }

    // Étape 3: Finalisation
    data, err = enrichmentStep3(ctx, data)
    if err != nil {
        return data, fmt.Errorf("erreur étape 3: %w", err)
    }

    return data, nil
}

func displayEnrichedObject(data DataObject) {
    fmt.Printf("📊 Objet enrichi final:\n")
    fmt.Printf("   🆔 ID: %s\n", data.ID)
    fmt.Printf("   📝 Données: %q\n", data.RawData)
    fmt.Printf("   🔄 Traité par: %v\n", data.ProcessedBy)
    fmt.Printf("   📅 Timestamp: %v\n", data.Timestamp.Format("15:04:05.000"))

    fmt.Printf("   📋 Métadonnées:\n")
    for key, value := range data.Metadata {
        fmt.Printf("     %s: %v\n", key, value)
    }
}

// Version avec pipeline concurrent
func concurrentEnrichmentPipeline() {
    const pipelineTimeout = 4 * time.Second

    ctx := context.Background()
    ctx = context.WithValue(ctx, UserIDKey, "user_concurrent")
    ctx = context.WithTimeout(ctx, pipelineTimeout)

    // Données à traiter en parallèle
    datasets := []DataObject{
        {ID: "data_001", RawData: "urgent customer request", Metadata: make(map[string]interface{})},
        {ID: "data_002", RawData: "normal processing task", Metadata: make(map[string]interface{})},
        {ID: "data_003", RawData: "high priority order", Metadata: make(map[string]interface{})},
    }

    fmt.Printf("🚀 Pipeline concurrent avec %d objets\n", len(datasets))

    // Channel pour collecter les résultats
    results := make(chan DataObject, len(datasets))
    errors := make(chan error, len(datasets))

    // Lancer le traitement en parallèle
    for _, data := range datasets {
        go func(d DataObject) {
            result, err := executePipeline(ctx, d)
            if err != nil {
                errors <- err
            } else {
                results <- result
            }
        }(data)
    }

    // Collecter les résultats
    var successful []DataObject
    var failed []error

    for i := 0; i < len(datasets); i++ {
        select {
        case result := <-results:
            successful = append(successful, result)
        case err := <-errors:
            failed = append(failed, err)
        case <-ctx.Done():
            failed = append(failed, ctx.Err())
        }
    }

    fmt.Printf("📊 Résultats du pipeline concurrent:\n")
    fmt.Printf("   ✅ Succès: %d\n", len(successful))
    fmt.Printf("   ❌ Échecs: %d\n", len(failed))

    for _, result := range successful {
        fmt.Printf("   🎯 %s: %d étapes, priorité %v\n",
                  result.ID, len(result.ProcessedBy), result.Metadata["priority"])
    }
}
```

### Version avec monitoring en temps réel

```go
func monitoredEnrichmentPipeline() {
    const pipelineTimeout = 5 * time.Second

    ctx := context.Background()
    ctx = context.WithTimeout(ctx, pipelineTimeout)

    // Channel pour les métriques
    metrics := make(chan string, 10)

    // Moniteur en temps réel
    go func() {
        for metric := range metrics {
            fmt.Printf("📊 [MÉTRIQUE] %s\n", metric)
        }
    }()

    // Pipeline avec monitoring
    data := DataObject{
        ID:       "monitored_data",
        RawData:  "data with real-time monitoring",
        Metadata: make(map[string]interface{}),
    }

    start := time.Now()

    // Étape 1 avec métrique
    metrics <- fmt.Sprintf("Début étape 1 à %v", time.Since(start))
    data, err := enrichmentStep1(ctx, data)
    if err == nil {
        metrics <- fmt.Sprintf("Étape 1 terminée à %v", time.Since(start))
    }

    // Étape 2 avec métrique
    metrics <- fmt.Sprintf("Début étape 2 à %v", time.Since(start))
    data, err = enrichmentStep2(ctx, data)
    if err == nil {
        metrics <- fmt.Sprintf("Étape 2 terminée à %v", time.Since(start))
    }

    // Étape 3 avec métrique
    metrics <- fmt.Sprintf("Début étape 3 à %v", time.Since(start))
    data, err = enrichmentStep3(ctx, data)
    if err == nil {
        metrics <- fmt.Sprintf("Étape 3 terminée à %v", time.Since(start))
    }

    close(metrics)

    if err != nil {
        fmt.Printf("❌ Pipeline échoué: %v\n", err)
    } else {
        fmt.Printf("✅ Pipeline monitored terminé en %v\n", time.Since(start))
    }
}

func main() {
    fmt.Println("=== Exercice 1: Worker avec timeout ===")
    timeoutWorkerExercise()

    fmt.Println("\n=== Version avancée avec retry ===")
    advancedTimeoutWorkerExercise()

    fmt.Println("\n=== Exercice 2: Pipeline avec valeurs ===")
    enrichmentPipelineExercise()

    fmt.Println("\n=== Version concurrent ===")
    concurrentEnrichmentPipeline()

    fmt.Println("\n=== Version monitored ===")
    monitoredEnrichmentPipeline()
}
```

## Points clés des solutions

### Exercice 1 - Worker avec timeout

**Fonctionnalités implémentées** :
- ✅ Context avec timeout global de 5 secondes
- ✅ 3 workers traitant 10 tâches de 1 seconde chacune
- ✅ Gestion propre de l'annulation à tous les niveaux
- ✅ Statistiques détaillées et répartition par worker
- ✅ Version avancée avec retry automatique

**Patterns utilisés** :
- Select avec ctx.Done() dans les boucles
- Gestion non-bloquante des channels avec context
- WaitGroup pour coordonner les workers
- Timer avec defer stop pour éviter les fuites

### Exercice 2 - Pipeline avec valeurs

**Fonctionnalités implémentées** :
- ✅ Context avec valeurs initiales (userID, requestID, traceID)
- ✅ 3 étapes d'enrichissement (normalisation, validation, finalisation)
- ✅ Timeout global de 3 secondes avec gestion d'état partiel
- ✅ Objet DataObject complet avec métadonnées
- ✅ Versions concurrent et monitored

**Enrichissements par étape** :
- **Étape 1** : Normalisation + métadonnées de traitement
- **Étape 2** : Validation + calcul de priorité
- **Étape 3** : Indexation + statistiques finales

Ces solutions démontrent l'utilisation avancée du package context pour construire des systèmes robustes et observables !

⏭️
