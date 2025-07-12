🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18-4. Memory leaks (Fuites mémoire)

## Introduction

Les fuites mémoire en Go sont comme un robinet qui goutte : au début, vous ne le remarquez pas, mais avec le temps, cela peut créer de gros problèmes. Même si Go a un garbage collector, il ne peut pas nettoyer la mémoire que votre programme garde "en vie" par accident.

Imaginez que vous gardez toutes vos factures dans une boîte "au cas où". Au bout de 10 ans, votre maison est pleine de boîtes de factures que vous n'utiliserez jamais. C'est exactement ce qui se passe avec les fuites mémoire : votre programme garde des références vers des données qu'il n'utilise plus.

## Qu'est-ce qu'une fuite mémoire ?

### Définition simple
Une fuite mémoire se produit quand votre programme alloue de la mémoire mais ne la libère jamais, même quand elle n'est plus nécessaire.

### En Go vs autres langages
```go
// En C (gestion manuelle) - Fuite évidente
void memoryLeakC() {
    char* data = malloc(1024); // Alloue
    // Oublie d'appeler free(data) -> FUITE !
}

// En Go (avec GC) - Fuite plus subtile
func memoryLeakGo() {
    var globalSlice [][]byte // Slice global
    data := make([]byte, 1024)
    globalSlice = append(globalSlice, data) // Référence gardée !
    // Le GC ne peut pas nettoyer car globalSlice référence encore data
}
```

### Pourquoi c'est dangereux
- **Consommation croissante de RAM** : Votre programme utilise de plus en plus de mémoire
- **Performance dégradée** : Plus de mémoire = plus de travail pour le GC
- **Crash possible** : Votre programme peut manquer de mémoire
- **Coûts serveur** : Plus de RAM nécessaire en production

## Types de fuites mémoire courantes

### 1. Références globales qui s'accumulent

```go
// ❌ FUITE : Slice global qui grandit indéfiniment
var globalCache []ExpensiveData

func processData(input string) {
    data := ExpensiveData{
        Content: input,
        Buffer:  make([]byte, 1024*1024), // 1MB par objet !
    }

    // Ajoute à un cache global qui n'est jamais nettoyé
    globalCache = append(globalCache, data)

    // Même si 'data' n'est plus utilisé dans cette fonction,
    // il reste en vie via globalCache
}

// ✅ SOLUTION : Cache avec limite de taille
type SafeCache struct {
    data []ExpensiveData
    maxSize int
}

func (c *SafeCache) Add(item ExpensiveData) {
    c.data = append(c.data, item)

    // Nettoie les anciens éléments
    if len(c.data) > c.maxSize {
        // Garde seulement les plus récents
        copy(c.data, c.data[len(c.data)-c.maxSize:])
        c.data = c.data[:c.maxSize]
    }
}
```

### 2. Goroutines qui ne se terminent jamais

```go
// ❌ FUITE : Goroutine qui boucle indéfiniment
func startWorkerWithLeak() {
    data := make([]byte, 1024*1024) // 1MB

    go func() {
        for {
            // Boucle infinie sans condition de sortie !
            // 'data' reste accessible donc ne peut pas être collecté
            processBuffer(data)
            time.Sleep(time.Second)
        }
    }() // Cette goroutine ne se termine jamais
}

// ✅ SOLUTION : Goroutine avec condition de sortie
func startWorkerSafe(ctx context.Context) {
    data := make([]byte, 1024*1024)

    go func() {
        for {
            select {
            case <-ctx.Done():
                // Condition de sortie propre
                return // La goroutine se termine, 'data' peut être collecté
            default:
                processBuffer(data)
                time.Sleep(time.Second)
            }
        }
    }()
}

// Usage
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    startWorkerSafe(ctx)

    // Plus tard...
    cancel() // Arrête proprement la goroutine
}
```

### 3. Closures qui capturent de gros objets

```go
// ❌ FUITE : Closure capture plus que nécessaire
func createHandlersWithLeak() []func() {
    largeData := make([]byte, 1024*1024*100) // 100MB !

    var handlers []func()

    for i := 0; i < 1000; i++ {
        index := i
        handler := func() {
            // Cette closure capture toute la variable largeData
            // même si elle n'utilise que l'index !
            fmt.Printf("Handler %d with large data size: %d\n",
                      index, len(largeData))
        }
        handlers = append(handlers, handler)
    }

    // largeData ne peut pas être collecté car toutes les closures
    // y font référence !
    return handlers
}

// ✅ SOLUTION : Capture seulement ce qui est nécessaire
func createHandlersSafe() []func() {
    largeData := make([]byte, 1024*1024*100)
    dataSize := len(largeData) // Capture seulement la taille

    // largeData peut maintenant être collecté
    largeData = nil

    var handlers []func()

    for i := 0; i < 1000; i++ {
        index := i
        handler := func() {
            // Capture seulement les variables nécessaires
            fmt.Printf("Handler %d with data size: %d\n", index, dataSize)
        }
        handlers = append(handlers, handler)
    }

    return handlers
}
```

### 4. Maps qui ne sont jamais nettoyées

```go
// ❌ FUITE : Map qui grandit indéfiniment
type UserSession struct {
    sessions map[string]*SessionData
    mutex    sync.RWMutex
}

func (us *UserSession) CreateSession(userID string) {
    us.mutex.Lock()
    defer us.mutex.Unlock()

    us.sessions[userID] = &SessionData{
        Created: time.Now(),
        Data:    make([]byte, 1024*1024), // 1MB par session
    }
    // Problème : les sessions ne sont jamais supprimées !
}

// ✅ SOLUTION : Nettoyage périodique
type SafeUserSession struct {
    sessions map[string]*SessionData
    mutex    sync.RWMutex
}

func (us *SafeUserSession) CreateSession(userID string) {
    us.mutex.Lock()
    defer us.mutex.Unlock()

    us.sessions[userID] = &SessionData{
        Created: time.Now(),
        Data:    make([]byte, 1024*1024),
    }
}

func (us *SafeUserSession) CleanupExpiredSessions() {
    us.mutex.Lock()
    defer us.mutex.Unlock()

    now := time.Now()
    for userID, session := range us.sessions {
        // Supprime les sessions de plus d'1 heure
        if now.Sub(session.Created) > time.Hour {
            delete(us.sessions, userID)
        }
    }
}

// Démarre un nettoyage périodique
func (us *SafeUserSession) StartCleanupRoutine(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            us.CleanupExpiredSessions()
        }
    }
}
```

### 5. Slices qui gardent des références vers de gros tableaux

```go
// ❌ FUITE : Slice garde une référence vers un gros tableau
func processLargeDataWithLeak() []byte {
    // Gros tableau de 100MB
    largeArray := make([]byte, 1024*1024*100)

    // Remplit avec des données
    for i := range largeArray {
        largeArray[i] = byte(i % 256)
    }

    // On veut juste garder les 1000 premiers bytes
    result := largeArray[:1000]

    // PROBLÈME : result garde une référence vers tout largeArray !
    // Les 100MB ne peuvent pas être collectés
    return result
}

// ✅ SOLUTION : Copie explicite
func processLargeDataSafe() []byte {
    largeArray := make([]byte, 1024*1024*100)

    for i := range largeArray {
        largeArray[i] = byte(i % 256)
    }

    // Copie explicite pour casser la référence
    result := make([]byte, 1000)
    copy(result, largeArray[:1000])

    // largeArray peut maintenant être collecté
    return result
}
```

## Détecter les fuites mémoire

### 1. Surveillance basique avec runtime

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func monitorMemory() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            var m runtime.MemStats
            runtime.ReadMemStats(&m)

            fmt.Printf("Alloc = %d KB", m.Alloc/1024)
            fmt.Printf(", TotalAlloc = %d KB", m.TotalAlloc/1024)
            fmt.Printf(", Sys = %d KB", m.Sys/1024)
            fmt.Printf(", NumGC = %d", m.NumGC)
            fmt.Printf(", Goroutines = %d\n", runtime.NumGoroutine())

            // Signe d'alerte : mémoire qui augmente constamment
            if m.Alloc/1024 > 500*1024 { // Plus de 500MB
                fmt.Println("⚠️ ALERTE : Consommation mémoire élevée !")
            }
        }
    }
}

func main() {
    go monitorMemory()

    // Simule une fuite mémoire
    simulateMemoryLeak()
}

func simulateMemoryLeak() {
    var leakySlice [][]byte

    for i := 0; i < 1000; i++ {
        // Ajoute 1MB à chaque itération
        data := make([]byte, 1024*1024)
        leakySlice = append(leakySlice, data)

        time.Sleep(100 * time.Millisecond)

        // Les données s'accumulent et ne sont jamais libérées !
    }
}
```

### 2. Profiling mémoire avancé

```go
// Programme pour analyser les fuites
package main

import (
    "log"
    "net/http"
    _ "net/http/pprof"
    "time"
)

func main() {
    // Active le serveur de profiling
    go func() {
        log.Println("Serveur pprof sur :6060")
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Simule une application avec une fuite
    go simulateApplicationWithLeak()

    // Garde le programme en vie
    select {}
}

func simulateApplicationWithLeak() {
    var globalData [][]byte

    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Simule du travail qui crée une fuite
            data := make([]byte, 1024*1024) // 1MB
            globalData = append(globalData, data)

            // Simule un "nettoyage" insuffisant
            if len(globalData) > 100 {
                // Ne supprime que le premier élément au lieu de faire un vrai nettoyage
                globalData = globalData[1:]
            }
        }
    }
}
```

```bash
# Analyser la fuite mémoire
go run main.go

# Dans un autre terminal, analyser le profil mémoire
go tool pprof http://localhost:6060/debug/pprof/heap

# Dans pprof
(pprof) top10
(pprof) list main.simulateApplicationWithLeak
(pprof) web  # Visualisation graphique
```

### 3. Tests automatiques pour détecter les fuites

```go
func TestNoMemoryLeak(t *testing.T) {
    // Mesure la mémoire avant
    runtime.GC()
    var m1 runtime.MemStats
    runtime.ReadMemStats(&m1)

    // Exécute la fonction suspecte plusieurs fois
    for i := 0; i < 1000; i++ {
        functionToTest()
    }

    // Force un garbage collection
    runtime.GC()
    runtime.GC() // Double GC pour être sûr

    // Mesure la mémoire après
    var m2 runtime.MemStats
    runtime.ReadMemStats(&m2)

    // Vérifie qu'il n'y a pas de fuite significative
    memoryIncrease := m2.Alloc - m1.Alloc
    maxAcceptableIncrease := uint64(1024 * 1024) // 1MB max

    if memoryIncrease > maxAcceptableIncrease {
        t.Errorf("Possible fuite mémoire détectée: %d bytes",
                 memoryIncrease)
    }
}

func BenchmarkMemoryUsage(b *testing.B) {
    b.ReportAllocs() // Affiche les allocations

    for i := 0; i < b.N; i++ {
        functionToTest()
    }

    // Si allocs/op augmente à chaque run, il y a probablement une fuite
}
```

## Corriger les fuites mémoire

### 1. Pattern Context pour les goroutines

```go
type WorkerManager struct {
    workers []Worker
    ctx     context.Context
    cancel  context.CancelFunc
}

func NewWorkerManager() *WorkerManager {
    ctx, cancel := context.WithCancel(context.Background())
    return &WorkerManager{
        ctx:    ctx,
        cancel: cancel,
    }
}

func (wm *WorkerManager) StartWorker(id int) {
    worker := Worker{
        ID:   id,
        data: make([]byte, 1024*1024), // Données par worker
    }

    go func() {
        defer func() {
            // Nettoyage automatique quand la goroutine se termine
            worker.data = nil
        }()

        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-wm.ctx.Done():
                // Sortie propre
                return
            case <-ticker.C:
                worker.DoWork()
            }
        }
    }()

    wm.workers = append(wm.workers, worker)
}

func (wm *WorkerManager) Shutdown() {
    wm.cancel() // Arrête toutes les goroutines
    // Toutes les données des workers seront libérées
}
```

### 2. Pattern Reference Counting

```go
type SharedResource struct {
    data     []byte
    refCount int32
    mutex    sync.Mutex
}

func NewSharedResource() *SharedResource {
    return &SharedResource{
        data:     make([]byte, 1024*1024),
        refCount: 1, // Compteur initial
    }
}

func (sr *SharedResource) AddRef() {
    atomic.AddInt32(&sr.refCount, 1)
}

func (sr *SharedResource) Release() {
    if atomic.AddInt32(&sr.refCount, -1) == 0 {
        // Plus de références, libère la mémoire
        sr.mutex.Lock()
        sr.data = nil
        sr.mutex.Unlock()
    }
}

func (sr *SharedResource) GetData() []byte {
    sr.mutex.Lock()
    defer sr.mutex.Unlock()
    return sr.data
}
```

### 3. Pattern Weak References (simulation)

```go
type WeakRef struct {
    target interface{}
    valid  bool
    mutex  sync.RWMutex
}

func NewWeakRef(target interface{}) *WeakRef {
    return &WeakRef{
        target: target,
        valid:  true,
    }
}

func (wr *WeakRef) Get() interface{} {
    wr.mutex.RLock()
    defer wr.mutex.RUnlock()

    if wr.valid {
        return wr.target
    }
    return nil
}

func (wr *WeakRef) Invalidate() {
    wr.mutex.Lock()
    defer wr.mutex.Unlock()

    wr.target = nil
    wr.valid = false
}

// Usage dans un cache avec weak references
type WeakCache struct {
    items map[string]*WeakRef
    mutex sync.RWMutex
}

func (wc *WeakCache) Set(key string, value interface{}) {
    wc.mutex.Lock()
    defer wc.mutex.Unlock()

    wc.items[key] = NewWeakRef(value)
}

func (wc *WeakCache) Get(key string) interface{} {
    wc.mutex.RLock()
    defer wc.mutex.RUnlock()

    if ref, exists := wc.items[key]; exists {
        return ref.Get()
    }
    return nil
}
```

## Outils de détection automatique

### 1. Script de monitoring

```bash
#!/bin/bash
# monitor_memory.sh

echo "Monitoring de la mémoire pour le processus Go"
while true; do
    # Trouve le PID du processus Go
    PID=$(pgrep -f "your-go-app")

    if [ ! -z "$PID" ]; then
        # Mesure l'utilisation mémoire
        MEMORY=$(ps -o pid,vsz,rss,comm -p $PID | tail -1)
        echo "$(date): $MEMORY"

        # Alerte si plus de 1GB
        RSS=$(echo $MEMORY | awk '{print $3}')
        if [ $RSS -gt 1048576 ]; then
            echo "⚠️ ALERTE: Consommation mémoire élevée: ${RSS}KB"
        fi
    fi

    sleep 5
done
```

### 2. Middleware de surveillance HTTP

```go
func memoryMonitoringMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Mesure avant
        var m1 runtime.MemStats
        runtime.ReadMemStats(&m1)

        start := time.Now()
        next.ServeHTTP(w, r)
        duration := time.Since(start)

        // Mesure après
        var m2 runtime.MemStats
        runtime.ReadMemStats(&m2)

        // Log si augmentation significative
        memIncrease := m2.Alloc - m1.Alloc
        if memIncrease > 1024*1024 { // Plus de 1MB
            log.Printf("⚠️ Requête %s a alloué %d KB en %v",
                      r.URL.Path, memIncrease/1024, duration)
        }
    })
}
```

## Exemples de fuites réelles et leurs corrections

### Exemple 1 : Fuite dans un serveur HTTP

```go
// ❌ PROBLÈME : Accumulation des connexions
type Server struct {
    connections map[string]*Connection
    mutex       sync.RWMutex
}

func (s *Server) HandleConnection(id string, conn *Connection) {
    s.mutex.Lock()
    s.connections[id] = conn
    s.mutex.Unlock()

    // PROBLÈME : Connexion jamais supprimée de la map !
}

// ✅ SOLUTION : Nettoyage automatique
type SafeServer struct {
    connections map[string]*Connection
    mutex       sync.RWMutex
    cleanup     chan string
}

func NewSafeServer() *SafeServer {
    s := &SafeServer{
        connections: make(map[string]*Connection),
        cleanup:     make(chan string, 100),
    }

    // Goroutine de nettoyage
    go s.cleanupRoutine()
    return s
}

func (s *SafeServer) HandleConnection(id string, conn *Connection) {
    s.mutex.Lock()
    s.connections[id] = conn
    s.mutex.Unlock()

    // Nettoie automatiquement quand la connexion se ferme
    go func() {
        <-conn.Done() // Attend que la connexion se ferme
        s.cleanup <- id
    }()
}

func (s *SafeServer) cleanupRoutine() {
    for id := range s.cleanup {
        s.mutex.Lock()
        delete(s.connections, id)
        s.mutex.Unlock()
    }
}
```

### Exemple 2 : Fuite dans un cache LRU

```go
// ✅ Implémentation correcte d'un cache LRU sans fuite
type LRUCache struct {
    capacity int
    items    map[string]*Item
    order    *list.List
    mutex    sync.RWMutex
}

type Item struct {
    key   string
    value interface{}
    elem  *list.Element
}

func NewLRUCache(capacity int) *LRUCache {
    return &LRUCache{
        capacity: capacity,
        items:    make(map[string]*Item),
        order:    list.New(),
    }
}

func (c *LRUCache) Set(key string, value interface{}) {
    c.mutex.Lock()
    defer c.mutex.Unlock()

    if item, exists := c.items[key]; exists {
        // Met à jour l'élément existant
        item.value = value
        c.order.MoveToFront(item.elem)
        return
    }

    // Nouvel élément
    item := &Item{
        key:   key,
        value: value,
    }
    item.elem = c.order.PushFront(item)
    c.items[key] = item

    // Éviction si nécessaire (évite la fuite)
    if c.order.Len() > c.capacity {
        c.evictOldest()
    }
}

func (c *LRUCache) evictOldest() {
    elem := c.order.Back()
    if elem != nil {
        item := elem.Value.(*Item)
        delete(c.items, item.key)
        c.order.Remove(elem)
    }
}
```

## Checklist anti-fuite mémoire

### Avant de coder
- [ ] Définir clairement le cycle de vie des objets
- [ ] Prévoir les mécanismes de nettoyage
- [ ] Identifier les références globales nécessaires

### Pendant le développement
- [ ] Utiliser context.Context pour les goroutines
- [ ] Limiter la taille des caches et collections
- [ ] Éviter les closures qui capturent de gros objets
- [ ] Implémenter des mécanismes de nettoyage périodique

### Tests et monitoring
- [ ] Écrire des tests de non-régression mémoire
- [ ] Surveiller l'utilisation mémoire en continu
- [ ] Profiler régulièrement l'application
- [ ] Mettre en place des alertes sur la consommation mémoire

## Exercices pratiques

### Exercice 1 : Détecter la fuite

Trouvez et corrigez la fuite dans ce code :

```go
type EventLogger struct {
    events []Event
}

func (el *EventLogger) LogEvent(event Event) {
    el.events = append(el.events, event)
    // Traite l'événement...
}

func (el *EventLogger) ProcessEvents() {
    for _, event := range el.events {
        handleEvent(event)
    }
    // Où est le problème ?
}
```

### Exercice 2 : Optimiser un worker pool

Corrigez cette implémentation qui fuit de la mémoire :

```go
func ProcessJobs(jobs []Job) {
    for _, job := range jobs {
        go func(j Job) {
            result := processJob(j)
            // Stocke le résultat quelque part...
            storeResult(result)
        }(job)
    }
}
```

## Résumé

Les fuites mémoire en Go sont souvent causées par :

**Causes principales :**
- Références globales qui s'accumulent
- Goroutines qui ne se terminent jamais
- Closures qui capturent trop de données
- Collections qui ne sont jamais nettoyées

**Stratégies de prévention :**
- Utiliser context.Context pour contrôler le cycle de vie
- Implémenter des mécanismes de nettoyage
- Limiter la taille des caches et collections
- Tester régulièrement la consommation mémoire

**Outils de détection :**
- Monitoring avec le package runtime
- Profiling avec go tool pprof
- Tests automatiques de non-régression

**Principe fondamental :** Chaque allocation doit avoir un chemin vers la libération !

La gestion proactive des fuites mémoire est essentielle pour maintenir des applications Go performantes et stables en production.

⏭️
