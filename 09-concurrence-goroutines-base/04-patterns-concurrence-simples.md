🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9-4 : Patterns de concurrence simples

## Qu'est-ce qu'un pattern de concurrence ?

Un pattern de concurrence est une solution éprouvée pour résoudre des problèmes courants en programmation parallèle. C'est comme une recette de cuisine : une fois que vous connaissez la recette, vous pouvez l'adapter à différents ingrédients.

**Analogie simple** : Dans une cuisine de restaurant, il y a des "patterns" établis :
- Le chef principal coordonne (orchestrateur)
- Plusieurs cuisiniers travaillent en parallèle (workers)
- Les plats passent par différentes stations (pipeline)
- Les commandes arrivent et sont distribuées (fan-out)

En Go, nous utilisons des patterns similaires avec goroutines et channels.

## Pourquoi apprendre ces patterns ?

Dans les sections précédentes, nous avons vu les bases : goroutines, channels, select. Maintenant, nous allons apprendre à les combiner efficacement pour résoudre des problèmes réels :

- ✅ Code plus lisible et maintenable
- ✅ Solutions performantes et scalables
- ✅ Éviter les erreurs communes (deadlocks, race conditions)
- ✅ Réutiliser des solutions éprouvées

## Pattern 1 : Worker Pool (Pool de Workers)

### Problème
Vous avez beaucoup de tâches à traiter et voulez limiter le nombre de workers simultanés.

### Solution

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Tache struct {
    ID   int
    Data string
}

type Resultat struct {
    TacheID int
    Message string
}

func worker(id int, taches <-chan Tache, resultats chan<- Resultat, wg *sync.WaitGroup) {
    defer wg.Done()

    for tache := range taches {
        fmt.Printf("Worker %d: Traitement tâche %d\n", id, tache.ID)

        // Simulation du travail
        time.Sleep(500 * time.Millisecond)

        resultats <- Resultat{
            TacheID: tache.ID,
            Message: fmt.Sprintf("Tâche %d terminée par worker %d", tache.ID, id),
        }
    }

    fmt.Printf("Worker %d: Terminé\n", id)
}

func workerPool() {
    fmt.Println("=== Pattern: Worker Pool ===")

    const nbWorkers = 3
    const nbTaches = 10

    // Channels
    taches := make(chan Tache, nbTaches)
    resultats := make(chan Resultat, nbTaches)

    // WaitGroup pour attendre tous les workers
    var wg sync.WaitGroup

    // Lancement des workers
    for i := 1; i <= nbWorkers; i++ {
        wg.Add(1)
        go worker(i, taches, resultats, &wg)
    }

    // Envoi des tâches
    for i := 1; i <= nbTaches; i++ {
        taches <- Tache{
            ID:   i,
            Data: fmt.Sprintf("Données de la tâche %d", i),
        }
    }
    close(taches) // Important : signaler qu'il n'y a plus de tâches

    // Goroutine pour fermer le channel des résultats quand tous les workers terminent
    go func() {
        wg.Wait()
        close(resultats)
    }()

    // Collecte des résultats
    for resultat := range resultats {
        fmt.Printf("Résultat: %s\n", resultat.Message)
    }

    fmt.Println("Toutes les tâches terminées")
}

func main() {
    workerPool()
}
```

### Quand l'utiliser ?
- Traitement d'images en lot
- Appels API en parallèle
- Traitement de fichiers
- Calculs intensifs

---

## Pattern 2 : Pipeline

### Problème
Vous voulez traiter des données par étapes séquentielles, mais en parallèle.

### Solution

```go
package main

import (
    "fmt"
    "strings"
)

// Étape 1 : Génération de données
func generer(donnees []string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for _, data := range donnees {
            out <- data
        }
    }()

    return out
}

// Étape 2 : Transformation en majuscules
func majuscules(in <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for data := range in {
            out <- strings.ToUpper(data)
        }
    }()

    return out
}

// Étape 3 : Ajout d'un préfixe
func prefixer(in <-chan string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for data := range in {
            out <- fmt.Sprintf("TRAITÉ: %s", data)
        }
    }()

    return out
}

func pipeline() {
    fmt.Println("=== Pattern: Pipeline ===")

    // Données d'entrée
    donnees := []string{"hello", "world", "golang", "pipeline"}

    // Construction du pipeline
    etape1 := generer(donnees)
    etape2 := majuscules(etape1)
    etape3 := prefixer(etape2)

    // Consommation des résultats finaux
    for resultat := range etape3 {
        fmt.Println("Résultat final:", resultat)
    }
}

func main() {
    pipeline()
}
```

### Version avec buffering pour plus de performance

```go
func pipelineOptimise() {
    fmt.Println("=== Pipeline optimisé ===")

    // Pipeline avec channels bufferisés
    donnees := []string{"hello", "world", "golang", "pipeline", "concurrent"}

    // Étapes avec buffer
    etape1 := genererAvecBuffer(donnees, 2)
    etape2 := majusculesAvecBuffer(etape1, 2)
    etape3 := prefixerAvecBuffer(etape2, 2)

    for resultat := range etape3 {
        fmt.Println("Résultat optimisé:", resultat)
    }
}

func genererAvecBuffer(donnees []string, bufferSize int) <-chan string {
    out := make(chan string, bufferSize)

    go func() {
        defer close(out)
        for _, data := range donnees {
            fmt.Printf("Génération: %s\n", data)
            out <- data
        }
    }()

    return out
}
```

### Quand l'utiliser ?
- Traitement de logs
- Transformation de données
- Compression/décompression
- Validation de données en étapes

---

## Pattern 3 : Fan-Out / Fan-In

### Problème
Vous voulez distribuer le travail à plusieurs workers, puis recombiner les résultats.

### Solution

```go
package main

import (
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// Fan-Out : Distribution du travail
func distribuer(donnees []int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)
        for _, data := range donnees {
            out <- data
        }
    }()

    return out
}

// Worker qui calcule le carré avec délai aléatoire
func calculerCarre(in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)
        for nombre := range in {
            // Simulation de travail avec durée variable
            delai := time.Duration(rand.Intn(1000)) * time.Millisecond
            time.Sleep(delai)

            resultat := nombre * nombre
            fmt.Printf("Carré de %d = %d (délai: %v)\n", nombre, resultat, delai)
            out <- resultat
        }
    }()

    return out
}

// Fan-In : Recombiner les résultats de plusieurs workers
func combiner(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    // Fonction pour copier d'un channel vers la sortie
    copier := func(c <-chan int) {
        defer wg.Done()
        for val := range c {
            out <- val
        }
    }

    // Lancement d'une goroutine pour chaque channel d'entrée
    wg.Add(len(channels))
    for _, c := range channels {
        go copier(c)
    }

    // Goroutine pour fermer le channel de sortie quand tous sont terminés
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func fanOutFanIn() {
    fmt.Println("=== Pattern: Fan-Out / Fan-In ===")

    // Données à traiter
    donnees := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    // Fan-Out : Distribution
    entree := distribuer(donnees)

    // Création de 3 workers en parallèle
    worker1 := calculerCarre(entree)
    worker2 := calculerCarre(entree)
    worker3 := calculerCarre(entree)

    // Fan-In : Recombiner les résultats
    resultats := combiner(worker1, worker2, worker3)

    // Collecte des résultats
    var total int
    for resultat := range resultats {
        total += resultat
        fmt.Printf("Résultat reçu: %d\n", resultat)
    }

    fmt.Printf("Somme totale des carrés: %d\n", total)
}

func main() {
    rand.Seed(time.Now().UnixNano())
    fanOutFanIn()
}
```

### Quand l'utiliser ?
- Recherche dans plusieurs bases de données
- Traitement d'images avec différents filtres
- Appels API parallèles avec agrégation
- Calculs scientifiques distribués

---

## Pattern 4 : Publisher/Subscriber (Pub/Sub)

### Problème
Vous voulez diffuser des informations à plusieurs consommateurs intéressés.

### Solution

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Message struct {
    Type    string
    Contenu string
    Heure   time.Time
}

type Subscriber struct {
    ID       string
    Messages chan Message
    quit     chan bool
}

type Publisher struct {
    subscribers map[string]*Subscriber
    mu          sync.RWMutex
}

func NewPublisher() *Publisher {
    return &Publisher{
        subscribers: make(map[string]*Subscriber),
    }
}

func (p *Publisher) Subscribe(id string) *Subscriber {
    p.mu.Lock()
    defer p.mu.Unlock()

    sub := &Subscriber{
        ID:       id,
        Messages: make(chan Message, 10), // Buffer pour éviter les blocages
        quit:     make(chan bool),
    }

    p.subscribers[id] = sub
    fmt.Printf("📋 %s s'est abonné\n", id)
    return sub
}

func (p *Publisher) Unsubscribe(id string) {
    p.mu.Lock()
    defer p.mu.Unlock()

    if sub, exists := p.subscribers[id]; exists {
        close(sub.Messages)
        delete(p.subscribers, id)
        fmt.Printf("❌ %s s'est désabonné\n", id)
    }
}

func (p *Publisher) Publish(msg Message) {
    p.mu.RLock()
    defer p.mu.RUnlock()

    fmt.Printf("📢 Publication: [%s] %s\n", msg.Type, msg.Contenu)

    for _, sub := range p.subscribers {
        select {
        case sub.Messages <- msg:
            // Message envoyé avec succès
        default:
            // Channel du subscriber plein, on ignore
            fmt.Printf("⚠️ Channel de %s plein, message ignoré\n", sub.ID)
        }
    }
}

func (s *Subscriber) Listen() {
    fmt.Printf("👂 %s commence à écouter\n", s.ID)

    for {
        select {
        case msg, ok := <-s.Messages:
            if !ok {
                fmt.Printf("🔇 %s: Channel fermé\n", s.ID)
                return
            }
            fmt.Printf("📨 %s reçu: [%s] %s à %s\n",
                s.ID, msg.Type, msg.Contenu, msg.Heure.Format("15:04:05"))

        case <-s.quit:
            fmt.Printf("🛑 %s: Arrêt demandé\n", s.ID)
            return
        }
    }
}

func (s *Subscriber) Stop() {
    close(s.quit)
}

func pubSub() {
    fmt.Println("=== Pattern: Publisher/Subscriber ===")

    // Création du publisher
    pub := NewPublisher()

    // Création des subscribers
    logger := pub.Subscribe("Logger")
    emailer := pub.Subscribe("Emailer")
    analytics := pub.Subscribe("Analytics")

    // Démarrage de l'écoute
    go logger.Listen()
    go emailer.Listen()
    go analytics.Listen()

    // Publication de messages
    messages := []Message{
        {Type: "INFO", Contenu: "Application démarrée", Heure: time.Now()},
        {Type: "ERROR", Contenu: "Erreur de connexion", Heure: time.Now()},
        {Type: "USER", Contenu: "Nouvel utilisateur inscrit", Heure: time.Now()},
        {Type: "INFO", Contenu: "Sauvegarde terminée", Heure: time.Now()},
    }

    for _, msg := range messages {
        pub.Publish(msg)
        time.Sleep(500 * time.Millisecond)
    }

    // Désabonnement d'un subscriber
    time.Sleep(1 * time.Second)
    pub.Unsubscribe("Emailer")

    // Encore quelques messages
    pub.Publish(Message{Type: "INFO", Contenu: "Message final", Heure: time.Now()})

    time.Sleep(1 * time.Second)
    fmt.Println("Fin de la démonstration")
}

func main() {
    pubSub()
}
```

### Quand l'utiliser ?
- Systèmes de notifications
- Logs distribués
- Événements d'application
- Systèmes de chat

---

## Pattern 5 : Rate Limiting (Limitation de débit)

### Problème
Vous voulez limiter le nombre d'opérations par seconde.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

// Rate Limiter simple avec ticker
func rateLimiterSimple() {
    fmt.Println("=== Pattern: Rate Limiter Simple ===")

    // Limiteur : maximum 2 opérations par seconde
    limiter := time.NewTicker(500 * time.Millisecond)
    defer limiter.Stop()

    // Simuler 5 requêtes qui arrivent rapidement
    requetes := []string{"Req1", "Req2", "Req3", "Req4", "Req5"}

    for _, req := range requetes {
        <-limiter.C // Attendre l'autorisation
        fmt.Printf("Traitement de %s à %s\n", req, time.Now().Format("15:04:05.000"))
    }
}

// Rate Limiter avec burst (rafales autorisées)
func rateLimiterAvecBurst() {
    fmt.Println("\n=== Rate Limiter avec Burst ===")

    // Permet 3 opérations immédiates, puis 1 par seconde
    burstLimit := 3
    tokens := make(chan struct{}, burstLimit)

    // Remplir le bucket initial
    for i := 0; i < burstLimit; i++ {
        tokens <- struct{}{}
    }

    // Ajouter un token chaque seconde
    go func() {
        ticker := time.NewTicker(1 * time.Second)
        defer ticker.Stop()

        for range ticker.C {
            select {
            case tokens <- struct{}{}:
                // Token ajouté
            default:
                // Bucket plein, on ignore
            }
        }
    }()

    // Simuler des requêtes
    for i := 1; i <= 7; i++ {
        <-tokens // Consommer un token
        fmt.Printf("Requête %d traitée à %s\n", i, time.Now().Format("15:04:05.000"))

        if i == 3 {
            fmt.Println("  (Burst terminé, maintenant 1 req/sec)")
        }

        time.Sleep(200 * time.Millisecond) // Simuler l'arrivée de requêtes
    }
}

func rateLimiting() {
    rateLimiterSimple()
    time.Sleep(2 * time.Second)
    rateLimiterAvecBurst()
}

func main() {
    rateLimiting()
}
```

### Quand l'utiliser ?
- APIs avec limite de taux
- Protection contre le spam
- Contrôle de charge système
- Appels vers services externes

---

## Pattern 6 : Circuit Breaker (Disjoncteur)

### Problème
Vous voulez arrêter d'appeler un service qui échoue fréquemment.

### Solution

```go
package main

import (
    "errors"
    "fmt"
    "math/rand"
    "time"
)

type CircuitState int

const (
    Closed CircuitState = iota
    Open
    HalfOpen
)

type CircuitBreaker struct {
    maxFailures  int
    timeout      time.Duration
    failures     int
    lastFailTime time.Time
    state        CircuitState
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures: maxFailures,
        timeout:     timeout,
        state:       Closed,
    }
}

func (cb *CircuitBreaker) Call(operation func() error) error {
    switch cb.state {
    case Open:
        if time.Since(cb.lastFailTime) > cb.timeout {
            cb.state = HalfOpen
            fmt.Println("🔄 Circuit HALF-OPEN - Test d'une requête")
        } else {
            return errors.New("circuit breaker ouvert")
        }
    case HalfOpen:
        // En mode half-open, on teste une requête
    case Closed:
        // Fonctionnement normal
    }

    err := operation()

    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = Open
            fmt.Printf("⚠️ Circuit OUVERT après %d échecs\n", cb.failures)
        }
        return err
    }

    // Succès
    if cb.state == HalfOpen {
        cb.state = Closed
        cb.failures = 0
        fmt.Println("✅ Circuit FERMÉ - Service rétabli")
    } else if cb.state == Closed {
        cb.failures = 0 // Reset le compteur en cas de succès
    }

    return nil
}

func (cb *CircuitBreaker) GetState() string {
    switch cb.state {
    case Closed:
        return "FERMÉ"
    case Open:
        return "OUVERT"
    case HalfOpen:
        return "SEMI-OUVERT"
    default:
        return "INCONNU"
    }
}

// Service simulé qui échoue parfois
func serviceInstable() error {
    // 70% de chance d'échec pour simuler un service en panne
    if rand.Float32() < 0.7 {
        return errors.New("service indisponible")
    }
    return nil
}

func circuitBreaker() {
    fmt.Println("=== Pattern: Circuit Breaker ===")

    // Circuit breaker : 3 échecs max, timeout de 3 secondes
    cb := NewCircuitBreaker(3, 3*time.Second)

    // Simuler 15 appels
    for i := 1; i <= 15; i++ {
        fmt.Printf("\nAppel %d - Circuit: %s\n", i, cb.GetState())

        err := cb.Call(serviceInstable)

        if err != nil {
            fmt.Printf("❌ Échec: %v\n", err)
        } else {
            fmt.Printf("✅ Succès\n")
        }

        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    rand.Seed(time.Now().UnixNano())
    circuitBreaker()
}
```

### Quand l'utiliser ?
- Appels vers des APIs externes
- Connexions base de données
- Services de microservices
- Systèmes distribués

---

## Résumé des patterns

| Pattern | Usage | Avantages |
|---------|-------|-----------|
| **Worker Pool** | Limiter la concurrence | Contrôle des ressources |
| **Pipeline** | Traitement par étapes | Parallélisme et organisation |
| **Fan-Out/Fan-In** | Distribution puis agrégation | Performance et scalabilité |
| **Pub/Sub** | Diffusion d'événements | Découplage et flexibilité |
| **Rate Limiting** | Contrôle du débit | Protection et stabilité |
| **Circuit Breaker** | Gestion des pannes | Résilience et récupération |

## Conseils pour choisir le bon pattern

### 1. **Analysez votre problème**
- Avez-vous besoin de limiter la concurrence ? → Worker Pool
- Traitez-vous des données par étapes ? → Pipeline
- Voulez-vous distribuer puis recombiner ? → Fan-Out/Fan-In

### 2. **Considérez les contraintes**
- Ressources limitées ? → Worker Pool + Rate Limiting
- Services externes instables ? → Circuit Breaker
- Plusieurs consommateurs ? → Pub/Sub

### 3. **Combinez les patterns**
Vous pouvez combiner plusieurs patterns :

```go
// Worker Pool + Rate Limiting + Circuit Breaker
func systemeComplet() {
    // Rate limiter pour contrôler les entrées
    // Worker pool pour traiter les tâches
    // Circuit breaker pour les appels externes
}
```

## Prochaines étapes

Ces patterns constituent les fondations de la programmation concurrente en Go. Dans la partie suivante (Concurrence avancée), nous verrons :

- Context package pour l'annulation
- Channels buffered vs unbuffered
- Sync package (Mutex, WaitGroup, etc.)
- Patterns plus complexes

**Conseil** : Maîtrisez bien ces patterns simples avant de passer aux concepts avancés. Ils résolvent 80% des problèmes de concurrence que vous rencontrerez !

## Exercices pratiques

### Exercice 1 : Web Scraper
Créez un worker pool qui télécharge le contenu de plusieurs URLs en parallèle.

### Exercice 2 : Pipeline de traitement d'images
Implémentez un pipeline : chargement → redimensionnement → sauvegarde.

### Exercice 3 : Système de notifications
Créez un pub/sub qui distribue des notifications à différents types d'abonnés (email, SMS, push).

# Solutions des exercices - Patterns de concurrence

## Exercice 1 : Web Scraper

**Énoncé** : Créez un worker pool qui télécharge le contenu de plusieurs URLs en parallèle.

### Solution

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type URLTask struct {
    URL string
    ID  int
}

type ScrapResult struct {
    URL          string
    ID           int
    ContentSize  int
    Duration     time.Duration
    Error        error
    StatusCode   int
}

func scrapeWorker(id int, tasks <-chan URLTask, results chan<- ScrapResult, wg *sync.WaitGroup) {
    defer wg.Done()

    fmt.Printf("🤖 Worker %d: Démarré\n", id)

    for task := range tasks {
        start := time.Now()

        fmt.Printf("🌐 Worker %d: Téléchargement de %s\n", id, task.URL)

        // Création du client HTTP avec timeout
        client := &http.Client{
            Timeout: 10 * time.Second,
        }

        resp, err := client.Get(task.URL)
        result := ScrapResult{
            URL:      task.URL,
            ID:       task.ID,
            Duration: time.Since(start),
        }

        if err != nil {
            result.Error = err
            fmt.Printf("❌ Worker %d: Erreur pour %s - %v\n", id, task.URL, err)
        } else {
            defer resp.Body.Close()
            result.StatusCode = resp.StatusCode

            // Lire le contenu
            content, err := io.ReadAll(resp.Body)
            if err != nil {
                result.Error = err
            } else {
                result.ContentSize = len(content)
                fmt.Printf("✅ Worker %d: %s - %d bytes (%v)\n",
                    id, task.URL, result.ContentSize, result.Duration)
            }
        }

        results <- result
    }

    fmt.Printf("🔄 Worker %d: Terminé\n", id)
}

func webScraper() {
    fmt.Println("=== Web Scraper avec Worker Pool ===")

    // Configuration
    const numWorkers = 3

    // URLs à télécharger
    urls := []string{
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://jsonplaceholder.typicode.com/posts/2",
        "https://httpbin.org/json",
        "https://httpbin.org/uuid",
        "https://httpbin.org/user-agent",
        "https://httpbin.org/headers",
    }

    // Channels
    tasks := make(chan URLTask, len(urls))
    results := make(chan ScrapResult, len(urls))

    // WaitGroup pour les workers
    var wg sync.WaitGroup

    // Lancement des workers
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go scrapeWorker(i, tasks, results, &wg)
    }

    // Envoi des tâches
    fmt.Printf("📤 Envoi de %d URLs à traiter\n", len(urls))
    for i, url := range urls {
        tasks <- URLTask{
            URL: url,
            ID:  i + 1,
        }
    }
    close(tasks)

    // Goroutine pour fermer le channel des résultats
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collecte et analyse des résultats
    var totalSize int
    var totalDuration time.Duration
    var successCount int
    var errorCount int

    fmt.Println("\n📊 === RÉSULTATS ===")
    for result := range results {
        if result.Error != nil {
            errorCount++
            fmt.Printf("❌ URL %d (%s): ERREUR - %v\n",
                result.ID, result.URL, result.Error)
        } else {
            successCount++
            totalSize += result.ContentSize
            totalDuration += result.Duration
            fmt.Printf("✅ URL %d (%s): %d bytes, %v, status %d\n",
                result.ID, result.URL, result.ContentSize,
                result.Duration, result.StatusCode)
        }
    }

    // Statistiques finales
    fmt.Println("\n📈 === STATISTIQUES ===")
    fmt.Printf("Total URLs: %d\n", len(urls))
    fmt.Printf("Succès: %d\n", successCount)
    fmt.Printf("Erreurs: %d\n", errorCount)
    fmt.Printf("Taille totale téléchargée: %d bytes\n", totalSize)
    if successCount > 0 {
        fmt.Printf("Durée moyenne par URL: %v\n", totalDuration/time.Duration(successCount))
    }
}

func main() {
    webScraper()
}
```

### Version avec retry et circuit breaker

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type AdvancedScraper struct {
    client       *http.Client
    maxRetries   int
    retryDelay   time.Duration
    circuitBreaker map[string]*CircuitBreaker
    mu           sync.RWMutex
}

type CircuitBreaker struct {
    failures    int
    maxFailures int
    lastFail    time.Time
    timeout     time.Duration
    isOpen      bool
}

func NewAdvancedScraper() *AdvancedScraper {
    return &AdvancedScraper{
        client: &http.Client{
            Timeout: 10 * time.Second,
        },
        maxRetries:     3,
        retryDelay:     1 * time.Second,
        circuitBreaker: make(map[string]*CircuitBreaker),
    }
}

func (s *AdvancedScraper) scrapeWithRetry(url string) ScrapResult {
    start := time.Now()

    for attempt := 1; attempt <= s.maxRetries; attempt++ {
        if attempt > 1 {
            fmt.Printf("🔄 Tentative %d/%d pour %s\n", attempt, s.maxRetries, url)
            time.Sleep(s.retryDelay)
        }

        resp, err := s.client.Get(url)
        if err != nil {
            if attempt == s.maxRetries {
                return ScrapResult{
                    URL:      url,
                    Error:    err,
                    Duration: time.Since(start),
                }
            }
            continue
        }

        defer resp.Body.Close()
        content, err := io.ReadAll(resp.Body)

        return ScrapResult{
            URL:         url,
            ContentSize: len(content),
            Duration:    time.Since(start),
            StatusCode:  resp.StatusCode,
            Error:       err,
        }
    }

    return ScrapResult{
        URL:      url,
        Error:    fmt.Errorf("max retries exceeded"),
        Duration: time.Since(start),
    }
}

func advancedWebScraper() {
    fmt.Println("=== Advanced Web Scraper ===")

    scraper := NewAdvancedScraper()

    urls := []string{
        "https://httpbin.org/delay/1",
        "https://httpbin.org/status/500", // Va échouer
        "https://jsonplaceholder.typicode.com/posts/1",
        "https://httpbin.org/status/200",
        "https://invalid-url-that-will-fail.com", // Va échouer
    }

    var wg sync.WaitGroup
    results := make(chan ScrapResult, len(urls))

    // Lancement des téléchargements
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            result := scraper.scrapeWithRetry(u)
            results <- result
        }(url)
    }

    // Fermeture du channel
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collecte des résultats
    for result := range results {
        if result.Error != nil {
            fmt.Printf("❌ %s: %v (%v)\n", result.URL, result.Error, result.Duration)
        } else {
            fmt.Printf("✅ %s: %d bytes (%v)\n", result.URL, result.ContentSize, result.Duration)
        }
    }
}
```

---

## Exercice 2 : Pipeline de traitement d'images

**Énoncé** : Implémentez un pipeline : chargement → redimensionnement → sauvegarde.

### Solution

```go
package main

import (
    "fmt"
    "path/filepath"
    "strings"
    "time"
)

type Image struct {
    Name     string
    Path     string
    Width    int
    Height   int
    Size     int
    Format   string
}

type ProcessedImage struct {
    Original  Image
    NewWidth  int
    NewHeight int
    NewSize   int
    NewPath   string
    Duration  time.Duration
}

// Étape 1: Chargement des images
func loadImages(imagePaths []string) <-chan Image {
    out := make(chan Image)

    go func() {
        defer close(out)

        for i, path := range imagePaths {
            fmt.Printf("📁 Chargement: %s\n", path)

            // Simulation du chargement
            time.Sleep(200 * time.Millisecond)

            // Extraction des informations (simulées)
            name := filepath.Base(path)
            ext := strings.ToLower(filepath.Ext(path))
            format := strings.TrimPrefix(ext, ".")

            image := Image{
                Name:   name,
                Path:   path,
                Width:  1920 + i*100, // Dimensions simulées
                Height: 1080 + i*50,
                Size:   500000 + i*100000, // Taille simulée en bytes
                Format: format,
            }

            fmt.Printf("✅ Chargé: %s (%dx%d, %d bytes)\n",
                image.Name, image.Width, image.Height, image.Size)

            out <- image
        }
    }()

    return out
}

// Étape 2: Redimensionnement
func resizeImages(in <-chan Image, targetWidth, targetHeight int) <-chan ProcessedImage {
    out := make(chan ProcessedImage)

    go func() {
        defer close(out)

        for img := range in {
            start := time.Now()

            fmt.Printf("🔄 Redimensionnement: %s (%dx%d → %dx%d)\n",
                img.Name, img.Width, img.Height, targetWidth, targetHeight)

            // Simulation du redimensionnement
            time.Sleep(500 * time.Millisecond)

            // Calcul de la nouvelle taille (approximative)
            ratio := float64(targetWidth*targetHeight) / float64(img.Width*img.Height)
            newSize := int(float64(img.Size) * ratio)

            processed := ProcessedImage{
                Original:  img,
                NewWidth:  targetWidth,
                NewHeight: targetHeight,
                NewSize:   newSize,
                NewPath:   generateResizedPath(img.Path, targetWidth, targetHeight),
                Duration:  time.Since(start),
            }

            fmt.Printf("✅ Redimensionné: %s en %v\n", img.Name, processed.Duration)

            out <- processed
        }
    }()

    return out
}

// Étape 3: Sauvegarde
func saveImages(in <-chan ProcessedImage) <-chan ProcessedImage {
    out := make(chan ProcessedImage)

    go func() {
        defer close(out)

        for processed := range in {
            start := time.Now()

            fmt.Printf("💾 Sauvegarde: %s → %s\n",
                processed.Original.Name, processed.NewPath)

            // Simulation de la sauvegarde
            time.Sleep(300 * time.Millisecond)

            // Mise à jour de la durée totale
            processed.Duration += time.Since(start)

            fmt.Printf("✅ Sauvegardé: %s (durée totale: %v)\n",
                processed.NewPath, processed.Duration)

            out <- processed
        }
    }()

    return out
}

func generateResizedPath(originalPath string, width, height int) string {
    dir := filepath.Dir(originalPath)
    name := strings.TrimSuffix(filepath.Base(originalPath), filepath.Ext(originalPath))
    ext := filepath.Ext(originalPath)

    return filepath.Join(dir, fmt.Sprintf("%s_%dx%d%s", name, width, height, ext))
}

func imagePipeline() {
    fmt.Println("=== Pipeline de traitement d'images ===")

    // Images à traiter
    imagePaths := []string{
        "/photos/vacation/beach.jpg",
        "/photos/vacation/mountain.png",
        "/photos/family/portrait.jpg",
        "/photos/work/presentation.png",
        "/photos/nature/sunset.jpg",
    }

    // Configuration du redimensionnement
    targetWidth := 800
    targetHeight := 600

    fmt.Printf("🎯 Objectif: redimensionner en %dx%d\n\n", targetWidth, targetHeight)

    // Construction du pipeline
    stage1 := loadImages(imagePaths)
    stage2 := resizeImages(stage1, targetWidth, targetHeight)
    stage3 := saveImages(stage2)

    // Collecte des résultats finaux
    var totalOriginalSize int
    var totalNewSize int
    var totalDuration time.Duration
    var processedCount int

    fmt.Println("📊 === RÉSULTATS FINAUX ===")
    for result := range stage3 {
        processedCount++
        totalOriginalSize += result.Original.Size
        totalNewSize += result.NewSize
        totalDuration += result.Duration

        compression := float64(result.Original.Size-result.NewSize) / float64(result.Original.Size) * 100

        fmt.Printf("🎉 Image %d: %s\n", processedCount, result.Original.Name)
        fmt.Printf("   Taille: %d → %d bytes (%.1f%% compression)\n",
            result.Original.Size, result.NewSize, compression)
        fmt.Printf("   Durée: %v\n", result.Duration)
        fmt.Printf("   Sauvegardé: %s\n\n", result.NewPath)
    }

    // Statistiques globales
    fmt.Println("📈 === STATISTIQUES GLOBALES ===")
    fmt.Printf("Images traitées: %d\n", processedCount)
    fmt.Printf("Taille originale totale: %d bytes\n", totalOriginalSize)
    fmt.Printf("Taille finale totale: %d bytes\n", totalNewSize)
    if totalOriginalSize > 0 {
        globalCompression := float64(totalOriginalSize-totalNewSize) / float64(totalOriginalSize) * 100
        fmt.Printf("Compression globale: %.1f%%\n", globalCompression)
    }
    fmt.Printf("Durée totale: %v\n", totalDuration)
    if processedCount > 0 {
        fmt.Printf("Durée moyenne par image: %v\n", totalDuration/time.Duration(processedCount))
    }
}

func main() {
    imagePipeline()
}
```

### Version avec multiple workers par étape

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Pipeline avec workers parallèles à chaque étape
func parallelImagePipeline() {
    fmt.Println("=== Pipeline parallèle ===")

    imagePaths := []string{
        "/photos/img1.jpg", "/photos/img2.jpg", "/photos/img3.jpg",
        "/photos/img4.jpg", "/photos/img5.jpg", "/photos/img6.jpg",
    }

    // Channels avec buffers
    loadedImages := make(chan Image, 10)
    resizedImages := make(chan ProcessedImage, 10)
    savedImages := make(chan ProcessedImage, 10)

    var wg sync.WaitGroup

    // Étape 1: 2 workers pour le chargement
    wg.Add(2)
    go loadWorker(1, imagePaths[:3], loadedImages, &wg)
    go loadWorker(2, imagePaths[3:], loadedImages, &wg)

    // Fermer le channel après tous les chargements
    go func() {
        wg.Wait()
        close(loadedImages)
    }()

    // Étape 2: 3 workers pour le redimensionnement
    var resizeWg sync.WaitGroup
    for i := 1; i <= 3; i++ {
        resizeWg.Add(1)
        go resizeWorker(i, loadedImages, resizedImages, &resizeWg)
    }

    go func() {
        resizeWg.Wait()
        close(resizedImages)
    }()

    // Étape 3: 2 workers pour la sauvegarde
    var saveWg sync.WaitGroup
    for i := 1; i <= 2; i++ {
        saveWg.Add(1)
        go saveWorker(i, resizedImages, savedImages, &saveWg)
    }

    go func() {
        saveWg.Wait()
        close(savedImages)
    }()

    // Collecte des résultats
    for result := range savedImages {
        fmt.Printf("🎉 Terminé: %s\n", result.Original.Name)
    }
}

func loadWorker(id int, paths []string, out chan<- Image, wg *sync.WaitGroup) {
    defer wg.Done()

    for _, path := range paths {
        fmt.Printf("📁 Loader %d: %s\n", id, path)
        time.Sleep(200 * time.Millisecond)

        // Simulation d'image chargée
        out <- Image{Name: filepath.Base(path), Path: path}
    }
}

func resizeWorker(id int, in <-chan Image, out chan<- ProcessedImage, wg *sync.WaitGroup) {
    defer wg.Done()

    for img := range in {
        fmt.Printf("🔄 Resizer %d: %s\n", id, img.Name)
        time.Sleep(500 * time.Millisecond)

        out <- ProcessedImage{Original: img, NewPath: "resized_" + img.Name}
    }
}

func saveWorker(id int, in <-chan ProcessedImage, out chan<- ProcessedImage, wg *sync.WaitGroup) {
    defer wg.Done()

    for processed := range in {
        fmt.Printf("💾 Saver %d: %s\n", id, processed.NewPath)
        time.Sleep(300 * time.Millisecond)

        out <- processed
    }
}
```

---

## Exercice 3 : Système de notifications

**Énoncé** : Créez un pub/sub qui distribue des notifications à différents types d'abonnés (email, SMS, push).

### Solution

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type NotificationType string

const (
    EMAIL NotificationType = "EMAIL"
    SMS   NotificationType = "SMS"
    PUSH  NotificationType = "PUSH"
    ALL   NotificationType = "ALL"
)

type Notification struct {
    ID       string
    Type     NotificationType
    Title    string
    Message  string
    Priority int // 1=low, 2=medium, 3=high, 4=urgent
    Metadata map[string]string
    Created  time.Time
}

type Subscriber interface {
    GetID() string
    GetTypes() []NotificationType
    Notify(notification Notification) error
    Start()
    Stop()
}

// Implémentation Email Subscriber
type EmailSubscriber struct {
    ID       string
    Email    string
    messages chan Notification
    quit     chan bool
    wg       sync.WaitGroup
}

func NewEmailSubscriber(id, email string) *EmailSubscriber {
    return &EmailSubscriber{
        ID:       id,
        Email:    email,
        messages: make(chan Notification, 50),
        quit:     make(chan bool),
    }
}

func (e *EmailSubscriber) GetID() string {
    return e.ID
}

func (e *EmailSubscriber) GetTypes() []NotificationType {
    return []NotificationType{EMAIL, ALL}
}

func (e *EmailSubscriber) Notify(notification Notification) error {
    select {
    case e.messages <- notification:
        return nil
    default:
        return fmt.Errorf("email queue full for %s", e.ID)
    }
}

func (e *EmailSubscriber) Start() {
    e.wg.Add(1)
    go func() {
        defer e.wg.Done()

        for {
            select {
            case notification := <-e.messages:
                e.processEmailNotification(notification)
            case <-e.quit:
                fmt.Printf("📧 Email subscriber %s arrêté\n", e.ID)
                return
            }
        }
    }()
}

func (e *EmailSubscriber) processEmailNotification(n Notification) {
    // Simulation d'envoi d'email
    fmt.Printf("📧 EMAIL [%s]: Envoi à %s\n", e.ID, e.Email)
    fmt.Printf("   Sujet: %s\n", n.Title)
    fmt.Printf("   Message: %s\n", n.Message)

    // Simulation du temps d'envoi
    time.Sleep(200 * time.Millisecond)

    fmt.Printf("✅ EMAIL [%s]: Envoyé avec succès\n", e.ID)
}

func (e *EmailSubscriber) Stop() {
    close(e.quit)
    e.wg.Wait()
}

// Implémentation SMS Subscriber
type SMSSubscriber struct {
    ID       string
    Phone    string
    messages chan Notification
    quit     chan bool
    wg       sync.WaitGroup
}

func NewSMSSubscriber(id, phone string) *SMSSubscriber {
    return &SMSSubscriber{
        ID:       id,
        Phone:    phone,
        messages: make(chan Notification, 30),
        quit:     make(chan bool),
    }
}

func (s *SMSSubscriber) GetID() string {
    return s.ID
}

func (s *SMSSubscriber) GetTypes() []NotificationType {
    return []NotificationType{SMS, ALL}
}

func (s *SMSSubscriber) Notify(notification Notification) error {
    select {
    case s.messages <- notification:
        return nil
    default:
        return fmt.Errorf("SMS queue full for %s", s.ID)
    }
}

func (s *SMSSubscriber) Start() {
    s.wg.Add(1)
    go func() {
        defer s.wg.Done()

        for {
            select {
            case notification := <-s.messages:
                s.processSMSNotification(notification)
            case <-s.quit:
                fmt.Printf("📱 SMS subscriber %s arrêté\n", s.ID)
                return
            }
        }
    }()
}

func (s *SMSSubscriber) processSMSNotification(n Notification) {
    // Simulation d'envoi de SMS
    fmt.Printf("📱 SMS [%s]: Envoi à %s\n", s.ID, s.Phone)

    // Raccourcir le message pour SMS (limite 160 caractères)
    message := n.Message
    if len(message) > 100 {
        message = message[:97] + "..."
    }

    fmt.Printf("   Message: %s\n", message)

    // Simulation du temps d'envoi
    time.Sleep(100 * time.Millisecond)

    fmt.Printf("✅ SMS [%s]: Envoyé avec succès\n", s.ID)
}

func (s *SMSSubscriber) Stop() {
    close(s.quit)
    s.wg.Wait()
}

// Implémentation Push Subscriber
type PushSubscriber struct {
    ID       string
    DeviceID string
    messages chan Notification
    quit     chan bool
    wg       sync.WaitGroup
}

func NewPushSubscriber(id, deviceID string) *PushSubscriber {
    return &PushSubscriber{
        ID:       id,
        DeviceID: deviceID,
        messages: make(chan Notification, 100),
        quit:     make(chan bool),
    }
}

func (p *PushSubscriber) GetID() string {
    return p.ID
}

func (p *PushSubscriber) GetTypes() []NotificationType {
    return []NotificationType{PUSH, ALL}
}

func (p *PushSubscriber) Notify(notification Notification) error {
    select {
    case p.messages <- notification:
        return nil
    default:
        return fmt.Errorf("push queue full for %s", p.ID)
    }
}

func (p *PushSubscriber) Start() {
    p.wg.Add(1)
    go func() {
        defer p.wg.Done()

        for {
            select {
            case notification := <-p.messages:
                p.processPushNotification(notification)
            case <-p.quit:
                fmt.Printf("🔔 Push subscriber %s arrêté\n", p.ID)
                return
            }
        }
    }()
}

func (p *PushSubscriber) processPushNotification(n Notification) {
    // Simulation d'envoi de push notification
    priorityIcon := map[int]string{1: "🟢", 2: "🟡", 3: "🟠", 4: "🔴"}
    icon := priorityIcon[n.Priority]

    fmt.Printf("🔔 PUSH [%s]: %s Device %s\n", p.ID, icon, p.DeviceID)
    fmt.Printf("   Titre: %s\n", n.Title)
    fmt.Printf("   Message: %s\n", n.Message)

    // Simulation du temps d'envoi
    time.Sleep(50 * time.Millisecond)

    fmt.Printf("✅ PUSH [%s]: Envoyé avec succès\n", p.ID)
}

func (p *PushSubscriber) Stop() {
    close(p.quit)
    p.wg.Wait()
}

// Notification Hub (Publisher)
type NotificationHub struct {
    subscribers map[string]Subscriber
    mutex       sync.RWMutex
    stats       map[NotificationType]int
}

func NewNotificationHub() *NotificationHub {
    return &NotificationHub{
        subscribers: make(map[string]Subscriber),
        stats:       make(map[NotificationType]int),
    }
}

func (h *NotificationHub) Subscribe(subscriber Subscriber) {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    h.subscribers[subscriber.GetID()] = subscriber
    subscriber.Start()

    fmt.Printf("📋 Abonnement: %s (%v)\n", subscriber.GetID(), subscriber.GetTypes())
}

func (h *NotificationHub) Unsubscribe(subscriberID string) {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    if subscriber, exists := h.subscribers[subscriberID]; exists {
        subscriber.Stop()
        delete(h.subscribers, subscriberID)
        fmt.Printf("❌ Désabonnement: %s\n", subscriberID)
    }
}

func (h *NotificationHub) Publish(notification Notification) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    fmt.Printf("\n📢 PUBLICATION [%s]: %s\n", notification.Type, notification.Title)
    h.stats[notification.Type]++

    sent := 0
    failed := 0

    for _, subscriber := range h.subscribers {
        // Vérifier si le subscriber accepte ce type de notification
        accepts := false
        for _, acceptedType := range subscriber.GetTypes() {
            if acceptedType == notification.Type || acceptedType == ALL {
                accepts = true
                break
            }
        }

        if accepts {
            err := subscriber.Notify(notification)
            if err != nil {
                fmt.Printf("⚠️ Erreur envoi à %s: %v\n", subscriber.GetID(), err)
                failed++
            } else {
                sent++
            }
        }
    }

    fmt.Printf("📊 Résultats: %d envoyés, %d échecs\n", sent, failed)
}

func (h *NotificationHub) GetStats() map[NotificationType]int {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    // Copie pour éviter les race conditions
    stats := make(map[NotificationType]int)
    for k, v := range h.stats {
        stats[k] = v
    }
    return stats
}

func (h *NotificationHub) Shutdown() {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    fmt.Println("\n🛑 Arrêt du hub de notifications...")

    for _, subscriber := range h.subscribers {
        subscriber.Stop()
    }

    fmt.Println("✅ Hub arrêté")
}

func notificationSystem() {
    fmt.Println("=== Système de notifications Pub/Sub ===")

    // Création du hub
    hub := NewNotificationHub()
    defer hub.Shutdown()

    // Création des abonnés
    emailSub1 := NewEmailSubscriber("user1-email", "alice@example.com")
    emailSub2 := NewEmailSubscriber("user2-email", "bob@example.com")
    smsSub1 := NewSMSSubscriber("user1-sms", "+33123456789")
    smsSub2 := NewSMSSubscriber("admin-sms", "+33987654321")
    pushSub1 := NewPushSubscriber("user1-push", "device-ios-001")
    pushSub2 := NewPushSubscriber("user2-push", "device-android-002")

    // Abonnement au hub
    hub.Subscribe(emailSub1)
    hub.Subscribe(emailSub2)
    hub.Subscribe(smsSub1)
    hub.Subscribe(smsSub2)
    hub.Subscribe(pushSub1)
    hub.Subscribe(pushSub2)

    time.Sleep(500 * time.Millisecond) // Laisser le temps aux goroutines de démarrer

    // Création et envoi de notifications
    notifications := []Notification{
        {
            ID:       "notif-001",
            Type:     EMAIL,
            Title:    "Bienvenue !",
            Message:  "Merci de vous être inscrit à notre service. Votre compte est maintenant actif.",
            Priority: 2,
            Metadata: map[string]string{"category": "welcome"},
            Created:  time.Now(),
        },
        {
            ID:       "notif-002",
            Type:     SMS,
            Title:    "Code de vérification",
            Message:  "Votre code de vérification est: 123456. Il expire dans 5 minutes.",
            Priority: 3,
            Metadata: map[string]string{"category": "security"},
            Created:  time.Now(),
        },
        {
            ID:       "notif-003",
            Type:     PUSH,
            Title:    "Nouvelle fonctionnalité",
            Message:  "Découvrez la nouvelle fonctionnalité de partage dans l'application !",
            Priority: 1,
            Metadata: map[string]string{"category": "feature"},
            Created:  time.Now(),
        },
        {
            ID:       "notif-004",
            Type:     ALL,
            Title:    "Maintenance programmée",
            Message:  "Une maintenance est programmée demain de 2h à 4h. Le service pourrait être temporairement indisponible.",
            Priority: 4,
            Metadata: map[string]string{"category": "maintenance"},
            Created:  time.Now(),
        },
        {
            ID:       "notif-005",
            Type:     EMAIL,
            Title:    "Rapport mensuel",
            Message:  "Votre rapport d'activité mensuel est disponible dans votre espace personnel.",
            Priority: 1,
            Metadata: map[string]string{"category": "report"},
            Created:  time.Now(),
        },
    }

    // Envoi des notifications avec délai
    for i, notification := range notifications {
        fmt.Printf("\n⏰ Envoi de la notification %d/%d\n", i+1, len(notifications))
        hub.Publish(notification)
        time.Sleep(1 * time.Second) // Délai entre les envois
    }

    // Attendre que toutes les notifications soient traitées
    time.Sleep(2 * time.Second)

    // Désabonnement d'un utilisateur
    fmt.Println("\n🔄 Simulation de désabonnement...")
    hub.Unsubscribe("user1-email")

    // Envoi d'une notification après désabonnement
    fmt.Println("\n📤 Test après désabonnement:")
    hub.Publish(Notification{
        ID:       "notif-006",
        Type:     EMAIL,
        Title:    "Test après désabonnement",
        Message:  "Cette notification ne devrait pas être reçue par user1-email",
        Priority: 2,
        Created:  time.Now(),
    })

    time.Sleep(1 * time.Second)

    // Affichage des statistiques
    fmt.Println("\n📊 === STATISTIQUES FINALES ===")
    stats := hub.GetStats()
    total := 0
    for notifType, count := range stats {
        fmt.Printf("%s: %d notifications\n", notifType, count)
        total += count
    }
    fmt.Printf("TOTAL: %d notifications publiées\n", total)
}

func main() {
    // Lancement de tous les exercices
    fmt.Println("🚀 === DÉMARRAGE DES EXERCICES PATTERNS ===\n")

    // Exercice 1
    webScraper()
    fmt.Println("\n" + strings.Repeat("=", 60) + "\n")

    // Exercice 2
    imagePipeline()
    fmt.Println("\n" + strings.Repeat("=", 60) + "\n")

    // Exercice 3
    notificationSystem()

    fmt.Println("\n🎉 === TOUS LES EXERCICES TERMINÉS ===")
}
```

### Version avancée avec filtres et priorités

```go
package main

import (
    "fmt"
    "strings"
    "sync"
    "time"
)

// Filtre pour les notifications
type NotificationFilter struct {
    MinPriority    int
    AllowedTypes   []NotificationType
    AllowedKeywords []string
}

// Subscriber avancé avec filtres
type AdvancedSubscriber struct {
    ID       string
    Type     string
    Filter   NotificationFilter
    messages chan Notification
    quit     chan bool
    wg       sync.WaitGroup
    stats    map[string]int
    mu       sync.Mutex
}

func NewAdvancedSubscriber(id, subType string, filter NotificationFilter) *AdvancedSubscriber {
    return &AdvancedSubscriber{
        ID:       id,
        Type:     subType,
        Filter:   filter,
        messages: make(chan Notification, 100),
        quit:     make(chan bool),
        stats:    make(map[string]int),
    }
}

func (s *AdvancedSubscriber) AcceptsNotification(n Notification) bool {
    // Vérifier la priorité
    if n.Priority < s.Filter.MinPriority {
        return false
    }

    // Vérifier le type
    typeAllowed := false
    for _, allowedType := range s.Filter.AllowedTypes {
        if allowedType == n.Type || allowedType == ALL {
            typeAllowed = true
            break
        }
    }
    if !typeAllowed {
        return false
    }

    // Vérifier les mots-clés (si spécifiés)
    if len(s.Filter.AllowedKeywords) > 0 {
        keywordFound := false
        text := strings.ToLower(n.Title + " " + n.Message)
        for _, keyword := range s.Filter.AllowedKeywords {
            if strings.Contains(text, strings.ToLower(keyword)) {
                keywordFound = true
                break
            }
        }
        if !keywordFound {
            return false
        }
    }

    return true
}

func (s *AdvancedSubscriber) Notify(notification Notification) error {
    if !s.AcceptsNotification(notification) {
        s.mu.Lock()
        s.stats["filtered"]++
        s.mu.Unlock()
        return nil // Pas d'erreur, juste filtré
    }

    select {
    case s.messages <- notification:
        return nil
    default:
        return fmt.Errorf("queue full for %s", s.ID)
    }
}

func (s *AdvancedSubscriber) Start() {
    s.wg.Add(1)
    go func() {
        defer s.wg.Done()

        for {
            select {
            case notification := <-s.messages:
                s.processNotification(notification)
            case <-s.quit:
                fmt.Printf("🛑 %s subscriber %s arrêté\n", s.Type, s.ID)
                return
            }
        }
    }()
}

func (s *AdvancedSubscriber) processNotification(n Notification) {
    s.mu.Lock()
    s.stats["processed"]++
    s.stats[string(n.Type)]++
    s.mu.Unlock()

    priorityEmoji := map[int]string{1: "🟢", 2: "🟡", 3: "🟠", 4: "🔴"}
    emoji := priorityEmoji[n.Priority]

    // Simulation de traitement basé sur le type de subscriber
    var processingTime time.Duration
    var icon string

    switch s.Type {
    case "EMAIL":
        icon = "📧"
        processingTime = 200 * time.Millisecond
    case "SMS":
        icon = "📱"
        processingTime = 100 * time.Millisecond
    case "PUSH":
        icon = "🔔"
        processingTime = 50 * time.Millisecond
    case "WEBHOOK":
        icon = "🌐"
        processingTime = 300 * time.Millisecond
    default:
        icon = "📤"
        processingTime = 150 * time.Millisecond
    }

    fmt.Printf("%s %s [%s]: %s %s\n", icon, s.Type, s.ID, emoji, n.Title)
    fmt.Printf("   Message: %s\n", n.Message)

    // Simulation du traitement
    time.Sleep(processingTime)

    fmt.Printf("✅ %s [%s]: Traité en %v\n", s.Type, s.ID, processingTime)
}

func (s *AdvancedSubscriber) Stop() {
    close(s.quit)
    s.wg.Wait()
}

func (s *AdvancedSubscriber) GetStats() map[string]int {
    s.mu.Lock()
    defer s.mu.Unlock()

    stats := make(map[string]int)
    for k, v := range s.stats {
        stats[k] = v
    }
    return stats
}

// Hub avancé avec métriques
type AdvancedNotificationHub struct {
    subscribers    map[string]*AdvancedSubscriber
    mutex          sync.RWMutex
    totalPublished int
    publishedByType map[NotificationType]int
    startTime      time.Time
}

func NewAdvancedNotificationHub() *AdvancedNotificationHub {
    return &AdvancedNotificationHub{
        subscribers:     make(map[string]*AdvancedSubscriber),
        publishedByType: make(map[NotificationType]int),
        startTime:       time.Now(),
    }
}

func (h *AdvancedNotificationHub) Subscribe(subscriber *AdvancedSubscriber) {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    h.subscribers[subscriber.ID] = subscriber
    subscriber.Start()

    fmt.Printf("📋 Abonnement: %s (%s, priorité min: %d)\n",
        subscriber.ID, subscriber.Type, subscriber.Filter.MinPriority)
}

func (h *AdvancedNotificationHub) Publish(notification Notification) {
    h.mutex.Lock()
    h.totalPublished++
    h.publishedByType[notification.Type]++
    subscribers := make([]*AdvancedSubscriber, 0, len(h.subscribers))
    for _, sub := range h.subscribers {
        subscribers = append(subscribers, sub)
    }
    h.mutex.Unlock()

    fmt.Printf("\n📢 PUBLICATION [%s]: %s\n", notification.Type, notification.Title)

    var wg sync.WaitGroup
    sent := make(chan int, len(subscribers))
    failed := make(chan int, len(subscribers))

    // Envoi parallèle à tous les subscribers
    for _, subscriber := range subscribers {
        wg.Add(1)
        go func(sub *AdvancedSubscriber) {
            defer wg.Done()

            err := sub.Notify(notification)
            if err != nil {
                fmt.Printf("⚠️ Erreur envoi à %s: %v\n", sub.ID, err)
                failed <- 1
            } else {
                sent <- 1
            }
        }(subscriber)
    }

    wg.Wait()
    close(sent)
    close(failed)

    // Compter les résultats
    sentCount := 0
    failedCount := 0
    for range sent {
        sentCount++
    }
    for range failed {
        failedCount++
    }

    fmt.Printf("📊 Résultats: %d envoyés, %d échecs\n", sentCount, failedCount)
}

func (h *AdvancedNotificationHub) GetGlobalStats() {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    fmt.Println("\n📊 === STATISTIQUES GLOBALES ===")
    fmt.Printf("Durée de fonctionnement: %v\n", time.Since(h.startTime).Round(time.Second))
    fmt.Printf("Total publié: %d notifications\n", h.totalPublished)

    fmt.Println("\nPar type:")
    for notifType, count := range h.publishedByType {
        fmt.Printf("  %s: %d\n", notifType, count)
    }

    fmt.Println("\nPar subscriber:")
    for _, sub := range h.subscribers {
        stats := sub.GetStats()
        fmt.Printf("  %s (%s):\n", sub.ID, sub.Type)
        for stat, value := range stats {
            fmt.Printf("    %s: %d\n", stat, value)
        }
    }
}

func (h *AdvancedNotificationHub) Shutdown() {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    fmt.Println("\n🛑 Arrêt du hub avancé...")

    var wg sync.WaitGroup
    for _, subscriber := range h.subscribers {
        wg.Add(1)
        go func(sub *AdvancedSubscriber) {
            defer wg.Done()
            sub.Stop()
        }(subscriber)
    }

    wg.Wait()
    fmt.Println("✅ Hub avancé arrêté")
}

func advancedNotificationSystem() {
    fmt.Println("=== Système de notifications avancé ===")

    hub := NewAdvancedNotificationHub()
    defer hub.Shutdown()

    // Création de subscribers avec différents filtres

    // Admin - reçoit tout en priorité élevée
    adminSub := NewAdvancedSubscriber("admin", "EMAIL", NotificationFilter{
        MinPriority:  3,
        AllowedTypes: []NotificationType{ALL},
    })

    // User normal - emails et push de priorité moyenne+
    userSub := NewAdvancedSubscriber("user1", "EMAIL", NotificationFilter{
        MinPriority:  2,
        AllowedTypes: []NotificationType{EMAIL, PUSH},
    })

    // SMS d'urgence uniquement
    smsSub := NewAdvancedSubscriber("emergency-sms", "SMS", NotificationFilter{
        MinPriority:  4,
        AllowedTypes: []NotificationType{SMS, ALL},
    })

    // Push avec filtre par mot-clé
    pushSub := NewAdvancedSubscriber("feature-push", "PUSH", NotificationFilter{
        MinPriority:     1,
        AllowedTypes:    []NotificationType{PUSH},
        AllowedKeywords: []string{"fonctionnalité", "feature", "update"},
    })

    // Webhook pour intégrations
    webhookSub := NewAdvancedSubscriber("webhook-integration", "WEBHOOK", NotificationFilter{
        MinPriority:  2,
        AllowedTypes: []NotificationType{ALL},
    })

    // Abonnements
    hub.Subscribe(adminSub)
    hub.Subscribe(userSub)
    hub.Subscribe(smsSub)
    hub.Subscribe(pushSub)
    hub.Subscribe(webhookSub)

    time.Sleep(500 * time.Millisecond)

    // Notifications de test avec différentes priorités
    testNotifications := []Notification{
        {
            ID:       "test-001",
            Type:     EMAIL,
            Title:    "Message de bienvenue",
            Message:  "Bienvenue dans notre application !",
            Priority: 1, // Faible - filtré par admin et SMS
        },
        {
            ID:       "test-002",
            Type:     PUSH,
            Title:    "Nouvelle fonctionnalité disponible",
            Message:  "Une nouvelle fonctionnalité de partage est maintenant disponible !",
            Priority: 2, // Moyen - reçu par feature-push (mot-clé match)
        },
        {
            ID:       "test-003",
            Type:     ALL,
            Title:    "Maintenance d'urgence",
            Message:  "Maintenance d'urgence en cours, service interrompu",
            Priority: 4, // Urgent - reçu par tous
        },
        {
            ID:       "test-004",
            Type:     SMS,
            Title:    "Code de sécurité",
            Message:  "Votre code: 789012",
            Priority: 3, // Élevé - reçu par admin et webhook
        },
        {
            ID:       "test-005",
            Type:     PUSH,
            Title:    "Rappel quotidien",
            Message:  "N'oubliez pas de consulter vos notifications",
            Priority: 1, // Faible - pas de mot-clé match pour feature-push
        },
    }

    // Envoi des notifications
    for i, notification := range testNotifications {
        fmt.Printf("\n⏰ Test %d/%d\n", i+1, len(testNotifications))
        hub.Publish(notification)
        time.Sleep(800 * time.Millisecond)
    }

    // Attendre le traitement
    time.Sleep(1 * time.Second)

    // Statistiques finales
    hub.GetGlobalStats()
}

func main() {
    advancedNotificationSystem()
}
```

## Points importants à retenir

### 1. **Worker Pool (Exercice 1)**
- **Contrôle des ressources** : Limite le nombre de connexions simultanées
- **Gestion d'erreurs** : Retry automatique et timeouts
- **Monitoring** : Statistiques de performance et taux de réussite
- **Scalabilité** : Facile d'ajuster le nombre de workers

### 2. **Pipeline (Exercice 2)**
- **Traitement par étapes** : Chaque étape peut être optimisée indépendamment
- **Parallélisme** : Multiple workers par étape pour plus de performance
- **Buffers** : Channels bufferisés pour éviter les goulots d'étranglement
- **Backpressure** : Gestion de la charge entre les étapes

### 3. **Pub/Sub (Exercice 3)**
- **Découplage** : Publishers et subscribers indépendants
- **Filtrage** : Notifications ciblées selon les critères
- **Résilience** : Gestion des queues pleines et des pannes
- **Métriques** : Suivi des performances et de la délivrance

## Patterns combinés

Ces exercices montrent comment combiner plusieurs patterns :

```go
// Web Scraper = Worker Pool + Rate Limiting + Circuit Breaker
// Pipeline = Pipeline + Worker Pool (par étape)
// Notifications = Pub/Sub + Filtering + Rate Limiting
```

## Conseils pour la production

### 1. **Monitoring et logging**
- Ajoutez des métriques pour chaque pattern
- Loggez les erreurs et les performances
- Utilisez des dashboards pour surveiller

### 2. **Gestion des erreurs**
- Implémentez des retry avec backoff exponentiel
- Utilisez des circuit breakers pour les services externes
- Gérez gracieusement les timeouts

### 3. **Configuration**
- Rendez les paramètres configurables (nombre de workers, tailles de buffers)
- Permettez l'ajustement dynamique de la charge
- Documentez les compromis performance/ressources

### 4. **Tests**
- Testez avec différentes charges
- Simulez des pannes de réseau
- Vérifiez les fuites de goroutines

Ces patterns forment la base de la plupart des applications concurrentes en Go. Maîtrisez-les bien avant de passer aux concepts plus avancés !

⏭️
