🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10-4 : Sync package (WaitGroup, Mutex, etc.)

## Introduction

Le package `sync` fournit des primitives de synchronisation essentielles pour la programmation concurrente en Go. Bien que Go privilégie l'usage des channels ("Don't communicate by sharing memory; share memory by communicating"), il existe des cas où les outils du package `sync` sont plus appropriés et plus efficaces.

## Pourquoi utiliser le package sync ?

### Channels vs Sync

**Channels** sont parfaits pour :
- Communication entre goroutines
- Passage de données
- Coordination d'événements

**Package sync** est meilleur pour :
- Protection de données partagées
- Synchronisation simple (attendre que des tâches se terminent)
- Contrôle d'accès exclusif à des ressources
- Optimisations de performance spécifiques

### Analogie du monde réel

Imaginez un bureau partagé :
- **Mutex** = Clé de bureau (une seule personne peut l'utiliser à la fois)
- **RWMutex** = Bibliothèque (plusieurs peuvent lire, un seul peut écrire)
- **WaitGroup** = Point de rendez-vous (attendre que tout le monde finisse)
- **Once** = Instruction qu'on ne donne qu'une fois

## 1. WaitGroup - Attendre un groupe de goroutines

### Qu'est-ce qu'un WaitGroup ?

Un `WaitGroup` permet d'attendre qu'un groupe de goroutines termine leur exécution. C'est comme un compteur intelligent qui sait quand toutes les tâches sont finies.

### Méthodes principales

```go
var wg sync.WaitGroup

wg.Add(n)    // Ajouter n goroutines à attendre
wg.Done()    // Signaler qu'une goroutine a terminé (équivaut à Add(-1))
wg.Wait()    // Bloquer jusqu'à ce que toutes les goroutines soient terminées
```

### Exemple de base

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func basicWaitGroupExample() {
    var wg sync.WaitGroup

    // Lancer 3 workers
    for i := 1; i <= 3; i++ {
        wg.Add(1) // Indiquer qu'on attend une goroutine de plus

        go func(workerID int) {
            defer wg.Done() // Signaler la fin à la sortie de la fonction

            fmt.Printf("🔧 Worker %d démarre\n", workerID)
            time.Sleep(time.Duration(workerID) * time.Second)
            fmt.Printf("✅ Worker %d terminé\n", workerID)
        }(i)
    }

    fmt.Println("⏳ Attente de tous les workers...")
    wg.Wait() // Bloquer jusqu'à ce que tous les workers appellent Done()
    fmt.Println("🎉 Tous les workers ont terminé !")
}
```

### Sortie attendue

```
🔧 Worker 1 démarre
🔧 Worker 2 démarre
🔧 Worker 3 démarre
⏳ Attente de tous les workers...
✅ Worker 1 terminé
✅ Worker 2 terminé
✅ Worker 3 terminé
🎉 Tous les workers ont terminé !
```

### Exemple pratique : Téléchargement parallèle

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type DownloadResult struct {
    URL      string
    Success  bool
    Duration time.Duration
    Error    error
}

func downloadFile(url string) DownloadResult {
    start := time.Now()

    // Simulation d'un téléchargement
    fmt.Printf("📥 Début téléchargement: %s\n", url)

    // Temps de téléchargement variable
    downloadTime := time.Duration(1+len(url)%3) * time.Second
    time.Sleep(downloadTime)

    // 20% de chance d'échec
    success := time.Now().UnixNano()%5 != 0

    result := DownloadResult{
        URL:      url,
        Success:  success,
        Duration: time.Since(start),
    }

    if !success {
        result.Error = fmt.Errorf("échec du téléchargement")
        fmt.Printf("❌ Échec: %s\n", url)
    } else {
        fmt.Printf("✅ Succès: %s (%v)\n", url, result.Duration)
    }

    return result
}

func parallelDownloadExample() {
    urls := []string{
        "https://example.com/file1.zip",
        "https://example.com/file2.pdf",
        "https://example.com/file3.mp4",
        "https://example.com/file4.jpg",
        "https://example.com/file5.txt",
    }

    var wg sync.WaitGroup
    results := make([]DownloadResult, len(urls))

    fmt.Printf("🚀 Démarrage du téléchargement de %d fichiers\n", len(urls))
    start := time.Now()

    for i, url := range urls {
        wg.Add(1)

        go func(index int, fileURL string) {
            defer wg.Done()
            results[index] = downloadFile(fileURL)
        }(i, url)
    }

    wg.Wait()
    totalTime := time.Since(start)

    // Analyser les résultats
    var successful, failed int
    for _, result := range results {
        if result.Success {
            successful++
        } else {
            failed++
        }
    }

    fmt.Printf("\n📊 Résultats du téléchargement parallèle:\n")
    fmt.Printf("   ⏱️ Temps total: %v\n", totalTime)
    fmt.Printf("   ✅ Succès: %d\n", successful)
    fmt.Printf("   ❌ Échecs: %d\n", failed)
    fmt.Printf("   📈 Taux de succès: %.1f%%\n", float64(successful)/float64(len(urls))*100)
}
```

## 2. Mutex - Exclusion mutuelle

### Qu'est-ce qu'un Mutex ?

Un `Mutex` (Mutual Exclusion) garantit qu'une seule goroutine peut accéder à une ressource partagée à la fois. C'est comme une clé : seule la goroutine qui a la clé peut entrer.

### Méthodes principales

```go
var mu sync.Mutex

mu.Lock()    // Prendre le verrou (bloquer si déjà pris)
mu.Unlock()  // Libérer le verrou
```

### Problème sans Mutex : Race Condition

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var counter int // Variable partagée (DANGER sans protection !)

func incrementWithoutMutex() {
    var wg sync.WaitGroup

    // Lancer 10 goroutines qui incrémentent 1000 fois chacune
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                counter++ // RACE CONDITION !
            }
        }()
    }

    wg.Wait()
    fmt.Printf("❌ Sans Mutex - Résultat: %d (attendu: 10000)\n", counter)
    // Le résultat sera probablement différent de 10000 !
}
```

### Solution avec Mutex

```go
package main

import (
    "fmt"
    "sync"
)

var (
    safeCounter int
    mu         sync.Mutex
)

func incrementWithMutex() {
    var wg sync.WaitGroup

    // Lancer 10 goroutines qui incrémentent 1000 fois chacune
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 1000; j++ {
                mu.Lock()     // Prendre le verrou
                safeCounter++ // Section critique protégée
                mu.Unlock()   // Libérer le verrou
            }
        }()
    }

    wg.Wait()
    fmt.Printf("✅ Avec Mutex - Résultat: %d (attendu: 10000)\n", safeCounter)
    // Résultat toujours correct : 10000
}
```

### Exemple pratique : Cache thread-safe

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Cache thread-safe
type SafeCache struct {
    mu   sync.Mutex
    data map[string]string
}

func NewSafeCache() *SafeCache {
    return &SafeCache{
        data: make(map[string]string),
    }
}

func (c *SafeCache) Get(key string) (string, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    value, exists := c.data[key]
    return value, exists
}

func (c *SafeCache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.data[key] = value
}

func (c *SafeCache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()

    delete(c.data, key)
}

func (c *SafeCache) Size() int {
    c.mu.Lock()
    defer c.mu.Unlock()

    return len(c.data)
}

func cacheExample() {
    cache := NewSafeCache()
    var wg sync.WaitGroup

    // Plusieurs goroutines accèdent au cache simultanément

    // Writers
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 10; j++ {
                key := fmt.Sprintf("key_%d_%d", id, j)
                value := fmt.Sprintf("value_%d_%d", id, j)
                cache.Set(key, value)
                fmt.Printf("📝 Writer %d: Set %s\n", id, key)
                time.Sleep(10 * time.Millisecond)
            }
        }(i)
    }

    // Readers
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 15; j++ {
                key := fmt.Sprintf("key_%d_%d", j%5, j%10)
                if value, exists := cache.Get(key); exists {
                    fmt.Printf("📖 Reader %d: Got %s = %s\n", id, key, value)
                }
                time.Sleep(15 * time.Millisecond)
            }
        }(i)
    }

    wg.Wait()
    fmt.Printf("🎯 Taille finale du cache: %d éléments\n", cache.Size())
}
```

## 3. RWMutex - Lecteurs/Écrivains

### Qu'est-ce qu'un RWMutex ?

Un `RWMutex` (Read-Write Mutex) permet soit :
- **Plusieurs lecteurs** simultanés (lecture partagée)
- **Un seul écrivain** à la fois (écriture exclusive)

C'est plus efficace qu'un Mutex normal quand on a beaucoup de lectures et peu d'écritures.

### Méthodes principales

```go
var rwMu sync.RWMutex

// Pour les lecteurs
rwMu.RLock()    // Prendre un verrou de lecture
rwMu.RUnlock()  // Libérer le verrou de lecture

// Pour les écrivains
rwMu.Lock()     // Prendre un verrou d'écriture (exclusif)
rwMu.Unlock()   // Libérer le verrou d'écriture
```

### Exemple : Configuration partagée

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Configuration thread-safe avec lectures optimisées
type Config struct {
    rwMu sync.RWMutex
    data map[string]string
}

func NewConfig() *Config {
    return &Config{
        data: make(map[string]string),
    }
}

func (c *Config) Get(key string) (string, bool) {
    c.rwMu.RLock()         // Verrou de lecture (partagé)
    defer c.rwMu.RUnlock()

    fmt.Printf("📖 Lecture de %s\n", key)
    time.Sleep(10 * time.Millisecond) // Simulation lecture

    value, exists := c.data[key]
    return value, exists
}

func (c *Config) Set(key, value string) {
    c.rwMu.Lock()          // Verrou d'écriture (exclusif)
    defer c.rwMu.Unlock()

    fmt.Printf("📝 Écriture de %s = %s\n", key, value)
    time.Sleep(50 * time.Millisecond) // Simulation écriture (plus lente)

    c.data[key] = value
}

func (c *Config) GetAll() map[string]string {
    c.rwMu.RLock()
    defer c.rwMu.RUnlock()

    fmt.Println("📚 Lecture complète de la configuration")
    time.Sleep(20 * time.Millisecond)

    // Copier pour éviter les modifications concurrentes
    result := make(map[string]string)
    for k, v := range c.data {
        result[k] = v
    }
    return result
}

func rwMutexExample() {
    config := NewConfig()
    var wg sync.WaitGroup

    // Initialiser quelques valeurs
    config.Set("database_url", "postgres://localhost/mydb")
    config.Set("api_key", "secret123")
    config.Set("timeout", "30s")

    fmt.Println("🚀 Démarrage des lectures et écritures concurrentes")
    start := time.Now()

    // Beaucoup de lecteurs (les lectures peuvent être simultanées)
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            keys := []string{"database_url", "api_key", "timeout", "nonexistent"}
            for j := 0; j < 5; j++ {
                key := keys[j%len(keys)]
                if value, exists := config.Get(key); exists {
                    fmt.Printf("👤 Reader %d: %s = %s\n", id, key, value)
                } else {
                    fmt.Printf("👤 Reader %d: %s non trouvé\n", id, key)
                }
                time.Sleep(20 * time.Millisecond)
            }
        }(i)
    }

    // Quelques écrivains (les écritures sont exclusives)
    for i := 0; i < 2; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 0; j < 3; j++ {
                key := fmt.Sprintf("dynamic_key_%d_%d", id, j)
                value := fmt.Sprintf("value_%d_%d", id, j)
                config.Set(key, value)
                time.Sleep(100 * time.Millisecond)
            }
        }(i)
    }

    // Un lecteur qui lit tout
    wg.Add(1)
    go func() {
        defer wg.Done()

        for i := 0; i < 3; i++ {
            all := config.GetAll()
            fmt.Printf("📚 Configuration complète (%d clés): %v\n", len(all), all)
            time.Sleep(200 * time.Millisecond)
        }
    }()

    wg.Wait()
    elapsed := time.Since(start)

    fmt.Printf("⏱️ Opérations terminées en %v\n", elapsed)
    fmt.Printf("📊 Configuration finale: %v\n", config.GetAll())
}
```

## 4. Once - Exécution unique

### Qu'est-ce qu'un Once ?

`sync.Once` garantit qu'une fonction ne sera exécutée qu'une seule fois, même si elle est appelée par plusieurs goroutines simultanément. Parfait pour l'initialisation.

### Méthode principale

```go
var once sync.Once

once.Do(func() {
    // Code exécuté une seule fois
})
```

### Exemple : Initialisation singleton

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Singleton avec initialisation paresseuse
type Database struct {
    connection string
}

var (
    dbInstance *Database
    dbOnce     sync.Once
)

func GetDatabase() *Database {
    dbOnce.Do(func() {
        fmt.Println("🔧 Initialisation de la base de données...")
        time.Sleep(500 * time.Millisecond) // Simulation initialisation coûteuse
        dbInstance = &Database{
            connection: "postgres://localhost:5432/myapp",
        }
        fmt.Println("✅ Base de données initialisée !")
    })

    return dbInstance
}

func (db *Database) Query(sql string) string {
    return fmt.Sprintf("Résultat de: %s", sql)
}

func onceExample() {
    var wg sync.WaitGroup

    fmt.Println("🚀 Plusieurs goroutines tentent d'accéder à la DB")

    // Lancer plusieurs goroutines qui tentent d'initialiser la DB
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            fmt.Printf("👤 Goroutine %d demande la DB...\n", id)
            db := GetDatabase() // Initialisation thread-safe

            result := db.Query(fmt.Sprintf("SELECT * FROM users WHERE id = %d", id))
            fmt.Printf("👤 Goroutine %d: %s\n", id, result)
        }(i)
    }

    wg.Wait()
    fmt.Println("🎉 Toutes les goroutines ont terminé")

    // Même instance pour tous
    db1 := GetDatabase()
    db2 := GetDatabase()
    fmt.Printf("🔍 Même instance ? %t\n", db1 == db2)
}
```

### Exemple pratique : Configuration globale

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type AppConfig struct {
    DatabaseURL string
    APIKey      string
    Debug       bool
    Timeout     time.Duration
}

var (
    appConfig *AppConfig
    configOnce sync.Once
)

func loadConfig() *AppConfig {
    fmt.Println("📋 Chargement de la configuration...")
    time.Sleep(200 * time.Millisecond) // Simulation lecture fichier/env

    return &AppConfig{
        DatabaseURL: "postgres://localhost/app",
        APIKey:      "sk-1234567890abcdef",
        Debug:       true,
        Timeout:     30 * time.Second,
    }
}

func GetConfig() *AppConfig {
    configOnce.Do(func() {
        appConfig = loadConfig()
        fmt.Println("✅ Configuration chargée et mise en cache")
    })

    return appConfig
}

func configExample() {
    var wg sync.WaitGroup

    // Plusieurs services démarrent en parallèle
    services := []string{"WebServer", "DatabaseService", "APIClient", "Logger", "Metrics"}

    for _, service := range services {
        wg.Add(1)
        go func(serviceName string) {
            defer wg.Done()

            fmt.Printf("🚀 Démarrage du service %s\n", serviceName)

            config := GetConfig() // Chargement thread-safe

            fmt.Printf("⚙️ %s configuré avec DB: %s, Debug: %t\n",
                      serviceName, config.DatabaseURL, config.Debug)

            time.Sleep(100 * time.Millisecond) // Simulation initialisation service
        }(service)
    }

    wg.Wait()
    fmt.Println("🎉 Tous les services sont démarrés")
}
```

## 5. Cond - Condition Variable

### Qu'est-ce qu'un Cond ?

`sync.Cond` permet à des goroutines d'attendre qu'une condition soit vraie. C'est moins courant mais utile pour des synchronisations complexes.

### Exemple : Producteur-Consommateur avec condition

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Buffer struct {
    mu   sync.Mutex
    cond *sync.Cond
    data []int
    max  int
}

func NewBuffer(maxSize int) *Buffer {
    b := &Buffer{
        data: make([]int, 0),
        max:  maxSize,
    }
    b.cond = sync.NewCond(&b.mu)
    return b
}

func (b *Buffer) Put(item int) {
    b.mu.Lock()
    defer b.mu.Unlock()

    // Attendre que le buffer ne soit pas plein
    for len(b.data) >= b.max {
        fmt.Printf("📦 Buffer plein, producteur attend...\n")
        b.cond.Wait() // Libère le mutex et attend
    }

    b.data = append(b.data, item)
    fmt.Printf("➕ Ajouté %d (buffer: %d/%d)\n", item, len(b.data), b.max)

    b.cond.Broadcast() // Réveiller tous les consommateurs
}

func (b *Buffer) Get() int {
    b.mu.Lock()
    defer b.mu.Unlock()

    // Attendre que le buffer ne soit pas vide
    for len(b.data) == 0 {
        fmt.Printf("📭 Buffer vide, consommateur attend...\n")
        b.cond.Wait()
    }

    item := b.data[0]
    b.data = b.data[1:]
    fmt.Printf("➖ Retiré %d (buffer: %d/%d)\n", item, len(b.data), b.max)

    b.cond.Broadcast() // Réveiller tous les producteurs
    return item
}

func condExample() {
    buffer := NewBuffer(3)
    var wg sync.WaitGroup

    // Producteurs
    for i := 1; i <= 2; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 1; j <= 5; j++ {
                item := id*10 + j
                buffer.Put(item)
                time.Sleep(200 * time.Millisecond)
            }
            fmt.Printf("🏁 Producteur %d terminé\n", id)
        }(i)
    }

    // Consommateurs
    for i := 1; i <= 2; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 1; j <= 5; j++ {
                item := buffer.Get()
                fmt.Printf("🍽️ Consommateur %d a traité %d\n", id, item)
                time.Sleep(300 * time.Millisecond)
            }
            fmt.Printf("🏁 Consommateur %d terminé\n", id)
        }(i)
    }

    wg.Wait()
    fmt.Println("🎉 Tous terminés")
}
```

## 6. Pool - Pool d'objets

### Qu'est-ce qu'un Pool ?

`sync.Pool` est un cache thread-safe d'objets temporaires réutilisables. Il aide à réduire la pression sur le garbage collector.

### Exemple : Pool de buffers

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

var bufferPool = sync.Pool{
    New: func() interface{} {
        fmt.Println("🆕 Création d'un nouveau buffer")
        return &bytes.Buffer{}
    },
}

func getBuffer() *bytes.Buffer {
    return bufferPool.Get().(*bytes.Buffer)
}

func putBuffer(buf *bytes.Buffer) {
    buf.Reset() // Nettoyer avant de remettre dans le pool
    bufferPool.Put(buf)
}

func processData(data string) string {
    // Récupérer un buffer du pool
    buf := getBuffer()
    defer putBuffer(buf) // Remettre dans le pool à la fin

    // Utiliser le buffer
    buf.WriteString("Processed: ")
    buf.WriteString(data)
    buf.WriteString(" [DONE]")

    return buf.String()
}

func poolExample() {
    var wg sync.WaitGroup

    data := []string{
        "task1", "task2", "task3", "task4", "task5",
        "task6", "task7", "task8", "task9", "task10",
    }

    for i, task := range data {
        wg.Add(1)
        go func(id int, taskData string) {
            defer wg.Done()

            result := processData(taskData)
            fmt.Printf("📋 Worker %d: %s\n", id, result)
        }(i, task)
    }

    wg.Wait()
    fmt.Println("🎉 Traitement terminé")
}
```

## Comparaison et bonnes pratiques

### Quand utiliser quoi ?

| Outil | Cas d'usage | Exemple |
|-------|-------------|---------|
| **WaitGroup** | Attendre plusieurs goroutines | Téléchargements parallèles |
| **Mutex** | Protéger données partagées | Compteur, cache |
| **RWMutex** | Beaucoup de lectures, peu d'écritures | Configuration, cache en lecture |
| **Once** | Initialisation unique | Singleton, chargement config |
| **Cond** | Synchronisation complexe | Producteur-consommateur |
| **Pool** | Réutilisation d'objets | Buffers, connexions |

### ✅ Bonnes pratiques

1. **Toujours utiliser defer avec Unlock**
   ```go
   mu.Lock()
   defer mu.Unlock()
   // Code protégé
   ```

2. **Préférer RWMutex pour les lectures fréquentes**
   ```go
   var rwMu sync.RWMutex

   // Lecture
   rwMu.RLock()
   defer rwMu.RUnlock()

   // Écriture
   rwMu.Lock()
   defer rwMu.Unlock()
   ```

3. **Encapsuler la synchronisation**
   ```go
   type SafeCounter struct {
       mu    sync.Mutex
       count int
   }

   func (c *SafeCounter) Increment() {
       c.mu.Lock()
       defer c.mu.Unlock()
       c.count++
   }
   ```

### ❌ Pièges à éviter

1. **Deadlock avec verrous multiples**
   ```go
   // ❌ Risque de deadlock
   mu1.Lock()
   mu2.Lock()  // Si autre goroutine fait mu2.Lock() puis mu1.Lock()

   // ✅ Toujours même ordre
   mu1.Lock()
   mu2.Lock()
   defer mu2.Unlock()
   defer mu1.Unlock()
   ```

2. **Copier un Mutex**
   ```go
   // ❌ Ne jamais copier un struct avec Mutex
   var original StructWithMutex
   copy := original  // ERREUR !

   // ✅ Utiliser des pointeurs
   original := &StructWithMutex{}
   ```

3. **Oublier de libérer le verrou**
   ```go
   // ❌ Risque d'oubli
   mu.Lock()
   if condition {
       return // ERREUR : verrou pas libéré !
   }
   mu.Unlock()

   // ✅ Toujours utiliser defer
   mu.Lock()
   defer mu.Unlock()
   if condition {
       return // OK : defer s'exécute
   }
   ```

## Exercices pratiques

### Exercice 1 : Compteur thread-safe

Créez un compteur thread-safe avec opérations d'incrémentation, décrémentation et lecture.

```go
type SafeCounter struct {
    // Votre implémentation ici
}

func (c *SafeCounter) Increment() {
    // Votre implémentation ici
}

func (c *SafeCounter) Decrement() {
    // Votre implémentation ici
}

func (c *SafeCounter) Value() int {
    // Votre implémentation ici
}
```

### Exercice 2 : Cache avec expiration

Créez un cache thread-safe où les entrées expirent après un délai.

```go
type ExpiringCache struct {
    // Votre implémentation ici
    // Indices :
    // - Utiliser RWMutex pour optimiser les lectures
    // - Stocker timestamp avec chaque valeur
    // - Méthode de nettoyage automatique
}
```

## Résumé

Le package `sync` fournit des outils essentiels pour la synchronisation :

- **WaitGroup** : Attendre qu'un groupe de goroutines termine
- **Mutex** : Protection exclusive d'une ressource partagée
- **RWMutex** : Lectures partagées, écriture exclusive
- **Once** : Exécution unique d'une fonction
- **Cond** : Synchronisation basée sur des conditions
- **Pool** : Réutilisation d'objets temporaires

### Patterns fondamentaux

```go
// Pattern WaitGroup
var wg sync.WaitGroup
wg.Add(n)
go func() { defer wg.Done(); /* travail */ }()
wg.Wait()

// Pattern Mutex
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// Section critique

// Pattern Once
var once sync.Once
once.Do(func() { /* initialisation unique */ })
```

## Solutions des exercices pratiques

### Exercice 1 : Compteur thread-safe

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// SafeCounter - Compteur thread-safe avec toutes les opérations
type SafeCounter struct {
    mu    sync.RWMutex // RWMutex pour optimiser les lectures
    count int
}

// NewSafeCounter crée un nouveau compteur
func NewSafeCounter() *SafeCounter {
    return &SafeCounter{
        count: 0,
    }
}

// Increment augmente le compteur de 1
func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// Decrement diminue le compteur de 1
func (c *SafeCounter) Decrement() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count--
}

// Add ajoute une valeur au compteur
func (c *SafeCounter) Add(delta int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count += delta
}

// Value retourne la valeur actuelle (lecture thread-safe)
func (c *SafeCounter) Value() int {
    c.mu.RLock() // Verrou de lecture partagé
    defer c.mu.RUnlock()
    return c.count
}

// Reset remet le compteur à zéro
func (c *SafeCounter) Reset() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count = 0
}

// IsPositive vérifie si le compteur est positif
func (c *SafeCounter) IsPositive() bool {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.count > 0
}

// CompareAndSet définit une nouvelle valeur si la valeur actuelle correspond
func (c *SafeCounter) CompareAndSet(expected, newValue int) bool {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.count == expected {
        c.count = newValue
        return true
    }
    return false
}

func testSafeCounter() {
    counter := NewSafeCounter()
    var wg sync.WaitGroup

    fmt.Println("🧪 Test du compteur thread-safe")
    fmt.Printf("Valeur initiale: %d\n", counter.Value())

    // Test avec plusieurs goroutines qui incrémentent
    numIncrementers := 10
    incrementsPerGoroutine := 1000

    fmt.Printf("🚀 Lancement de %d goroutines d'incrémentation (%d incréments chacune)\n",
              numIncrementers, incrementsPerGoroutine)

    start := time.Now()

    for i := 0; i < numIncrementers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 0; j < incrementsPerGoroutine; j++ {
                counter.Increment()

                // Quelques lectures pour tester RWMutex
                if j%100 == 0 {
                    value := counter.Value()
                    if value > 0 {
                        // Opération sur la valeur lue
                    }
                }
            }
            fmt.Printf("✅ Incrementer %d terminé\n", id)
        }(i)
    }

    // Test avec quelques goroutines qui décrémentent
    numDecrementers := 3
    decrementsPerGoroutine := 500

    fmt.Printf("🚀 Lancement de %d goroutines de décrémentation (%d décréments chacune)\n",
              numDecrementers, decrementsPerGoroutine)

    for i := 0; i < numDecrementers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 0; j < decrementsPerGoroutine; j++ {
                counter.Decrement()

                // Test de lecture
                if j%50 == 0 {
                    isPositive := counter.IsPositive()
                    if isPositive {
                        // Action basée sur le signe
                    }
                }
            }
            fmt.Printf("✅ Decrementer %d terminé\n", id)
        }(i)
    }

    // Goroutines de lecture intensive
    numReaders := 5

    fmt.Printf("📖 Lancement de %d lecteurs intensifs\n", numReaders)

    for i := 0; i < numReaders; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            readCount := 0
            for j := 0; j < 2000; j++ {
                value := counter.Value()
                if value != 0 {
                    readCount++
                }

                // Test CompareAndSet occasionnel
                if j%200 == 0 {
                    current := counter.Value()
                    if counter.CompareAndSet(current, current+1) {
                        fmt.Printf("📝 Reader %d: CompareAndSet réussi\n", id)
                    }
                }

                time.Sleep(time.Microsecond) // Petite pause
            }
            fmt.Printf("📚 Reader %d terminé (%d lectures non-zéro)\n", id, readCount)
        }(i)
    }

    wg.Wait()
    elapsed := time.Since(start)

    // Calcul de la valeur attendue
    expectedValue := (numIncrementers * incrementsPerGoroutine) -
                    (numDecrementers * decrementsPerGoroutine) +
                    numReaders // CompareAndSet ajoute parfois +1

    finalValue := counter.Value()

    fmt.Printf("\n📊 Résultats du test:\n")
    fmt.Printf("   ⏱️ Temps d'exécution: %v\n", elapsed)
    fmt.Printf("   🎯 Valeur finale: %d\n", finalValue)
    fmt.Printf("   📐 Valeur attendue (environ): %d\n", expectedValue)
    fmt.Printf("   ✅ Test réussi: %t\n", abs(finalValue-expectedValue) < numReaders*2)
    fmt.Printf("   🚀 Opérations/seconde: %.0f\n",
              float64(numIncrementers*incrementsPerGoroutine+numDecrementers*decrementsPerGoroutine)/elapsed.Seconds())
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}
```

### Exercice 2 : Cache avec expiration

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// CacheEntry représente une entrée du cache avec timestamp
type CacheEntry struct {
    Value     interface{}
    ExpiresAt time.Time
}

// IsExpired vérifie si l'entrée a expiré
func (e *CacheEntry) IsExpired() bool {
    return time.Now().After(e.ExpiresAt)
}

// ExpiringCache - Cache thread-safe avec expiration automatique
type ExpiringCache struct {
    mu      sync.RWMutex
    data    map[string]*CacheEntry
    defaultTTL time.Duration
    cleanupInterval time.Duration
    stopCleanup chan bool
    stats   CacheStats
}

// CacheStats statistiques du cache
type CacheStats struct {
    Hits       int64
    Misses     int64
    Expired    int64
    Evictions  int64
    Sets       int64
}

// NewExpiringCache crée un nouveau cache avec expiration
func NewExpiringCache(defaultTTL, cleanupInterval time.Duration) *ExpiringCache {
    cache := &ExpiringCache{
        data:            make(map[string]*CacheEntry),
        defaultTTL:      defaultTTL,
        cleanupInterval: cleanupInterval,
        stopCleanup:     make(chan bool),
    }

    // Démarrer le nettoyage automatique
    go cache.cleanupExpired()

    return cache
}

// Set ajoute une valeur avec TTL par défaut
func (c *ExpiringCache) Set(key string, value interface{}) {
    c.SetWithTTL(key, value, c.defaultTTL)
}

// SetWithTTL ajoute une valeur avec TTL personnalisé
func (c *ExpiringCache) SetWithTTL(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.data[key] = &CacheEntry{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
    c.stats.Sets++

    fmt.Printf("💾 Cache: Set %s (expire dans %v)\n", key, ttl)
}

// Get récupère une valeur du cache
func (c *ExpiringCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()

    entry, exists := c.data[key]
    if !exists {
        c.mu.RUnlock()
        c.mu.Lock()
        c.stats.Misses++
        c.mu.Unlock()
        return nil, false
    }

    if entry.IsExpired() {
        c.mu.RUnlock()
        c.mu.Lock()
        delete(c.data, key)
        c.stats.Expired++
        c.mu.Unlock()
        fmt.Printf("⏰ Cache: %s expiré et supprimé\n", key)
        return nil, false
    }

    value := entry.Value
    c.mu.RUnlock()

    c.mu.Lock()
    c.stats.Hits++
    c.mu.Unlock()

    return value, true
}

// Delete supprime une clé du cache
func (c *ExpiringCache) Delete(key string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()

    if _, exists := c.data[key]; exists {
        delete(c.data, key)
        c.stats.Evictions++
        fmt.Printf("🗑️ Cache: %s supprimé manuellement\n", key)
        return true
    }

    return false
}

// Clear vide tout le cache
func (c *ExpiringCache) Clear() {
    c.mu.Lock()
    defer c.mu.Unlock()

    count := len(c.data)
    c.data = make(map[string]*CacheEntry)
    c.stats.Evictions += int64(count)
    fmt.Printf("🧹 Cache: %d entrées supprimées (Clear)\n", count)
}

// Size retourne le nombre d'entrées dans le cache
func (c *ExpiringCache) Size() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return len(c.data)
}

// Keys retourne toutes les clés non expirées
func (c *ExpiringCache) Keys() []string {
    c.mu.RLock()
    defer c.mu.RUnlock()

    var keys []string
    now := time.Now()

    for key, entry := range c.data {
        if now.Before(entry.ExpiresAt) {
            keys = append(keys, key)
        }
    }

    return keys
}

// GetStats retourne les statistiques du cache
func (c *ExpiringCache) GetStats() CacheStats {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.stats
}

// GetHitRatio calcule le ratio de succès
func (c *ExpiringCache) GetHitRatio() float64 {
    stats := c.GetStats()
    total := stats.Hits + stats.Misses
    if total == 0 {
        return 0.0
    }
    return float64(stats.Hits) / float64(total)
}

// cleanupExpired supprime les entrées expirées (goroutine de fond)
func (c *ExpiringCache) cleanupExpired() {
    ticker := time.NewTicker(c.cleanupInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            c.performCleanup()
        case <-c.stopCleanup:
            return
        }
    }
}

// performCleanup effectue le nettoyage des entrées expirées
func (c *ExpiringCache) performCleanup() {
    c.mu.Lock()
    defer c.mu.Unlock()

    now := time.Now()
    var expiredKeys []string

    // Identifier les clés expirées
    for key, entry := range c.data {
        if entry.IsExpired() {
            expiredKeys = append(expiredKeys, key)
        }
    }

    // Supprimer les entrées expirées
    for _, key := range expiredKeys {
        delete(c.data, key)
        c.stats.Expired++
    }

    if len(expiredKeys) > 0 {
        fmt.Printf("🧹 Nettoyage automatique: %d entrées expirées supprimées\n", len(expiredKeys))
    }
}

// Close arrête le cache et le nettoyage automatique
func (c *ExpiringCache) Close() {
    close(c.stopCleanup)
    c.Clear()
    fmt.Println("🔐 Cache fermé")
}

func testExpiringCache() {
    fmt.Println("🧪 Test du cache avec expiration")

    // Créer un cache avec TTL de 2s et nettoyage toutes les 1s
    cache := NewExpiringCache(2*time.Second, 1*time.Second)
    defer cache.Close()

    var wg sync.WaitGroup

    // Producteurs : ajoutent des données au cache
    fmt.Println("🚀 Démarrage des producteurs")
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go func(producerID int) {
            defer wg.Done()

            for j := 1; j <= 10; j++ {
                key := fmt.Sprintf("producer_%d_item_%d", producerID, j)
                value := fmt.Sprintf("data_from_producer_%d_item_%d", producerID, j)

                // Varier les TTL
                var ttl time.Duration
                switch j % 3 {
                case 0:
                    ttl = 1 * time.Second // Court
                case 1:
                    ttl = 3 * time.Second // Moyen
                case 2:
                    ttl = 5 * time.Second // Long
                }

                cache.SetWithTTL(key, value, ttl)
                time.Sleep(100 * time.Millisecond)
            }
            fmt.Printf("✅ Producteur %d terminé\n", producerID)
        }(i)
    }

    // Consommateurs : lisent du cache
    fmt.Println("📖 Démarrage des consommateurs")
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go func(consumerID int) {
            defer wg.Done()

            hits := 0
            misses := 0

            for j := 1; j <= 50; j++ {
                // Lire des clés aléatoirement
                producerID := (j % 3) + 1
                itemID := (j % 10) + 1
                key := fmt.Sprintf("producer_%d_item_%d", producerID, itemID)

                if value, found := cache.Get(key); found {
                    hits++
                    fmt.Printf("📚 Consommateur %d: trouvé %s = %v\n", consumerID, key, value)
                } else {
                    misses++
                    if j%10 == 0 {
                        fmt.Printf("🔍 Consommateur %d: %s non trouvé (hits=%d, misses=%d)\n",
                                  consumerID, key, hits, misses)
                    }
                }

                time.Sleep(150 * time.Millisecond)
            }

            fmt.Printf("📊 Consommateur %d terminé: %d hits, %d misses\n",
                      consumerID, hits, misses)
        }(i)
    }

    // Moniteur : affiche les stats périodiquement
    wg.Add(1)
    go func() {
        defer wg.Done()

        for i := 0; i < 8; i++ {
            time.Sleep(1 * time.Second)

            stats := cache.GetStats()
            size := cache.Size()
            hitRatio := cache.GetHitRatio()

            fmt.Printf("📊 Stats [%ds]: Size=%d, Hits=%d, Misses=%d, Expired=%d, Hit Ratio=%.2f%%\n",
                      i+1, size, stats.Hits, stats.Misses, stats.Expired, hitRatio*100)
        }
    }()

    // Test des opérations spéciales
    wg.Add(1)
    go func() {
        defer wg.Done()

        time.Sleep(2 * time.Second)

        // Test de suppression manuelle
        cache.Set("temp_key", "temp_value")
        if cache.Delete("temp_key") {
            fmt.Println("🗑️ Suppression manuelle réussie")
        }

        // Test des clés
        keys := cache.Keys()
        fmt.Printf("🔑 Clés actives: %d\n", len(keys))

        time.Sleep(3 * time.Second)

        // Stats finales
        finalStats := cache.GetStats()
        finalSize := cache.Size()

        fmt.Printf("\n🎯 Statistiques finales:\n")
        fmt.Printf("   Taille finale: %d\n", finalSize)
        fmt.Printf("   Total Sets: %d\n", finalStats.Sets)
        fmt.Printf("   Total Hits: %d\n", finalStats.Hits)
        fmt.Printf("   Total Misses: %d\n", finalStats.Misses)
        fmt.Printf("   Total Expired: %d\n", finalStats.Expired)
        fmt.Printf("   Total Evictions: %d\n", finalStats.Evictions)
        fmt.Printf("   Hit Ratio final: %.2f%%\n", cache.GetHitRatio()*100)
    }()

    wg.Wait()
    fmt.Println("🎉 Test du cache terminé")
}

func main() {
    fmt.Println("=== Solutions des exercices Sync ===\n")

    fmt.Println("### Exercice 1: Compteur thread-safe ###")
    testSafeCounter()

    fmt.Println("\n### Exercice 2: Cache avec expiration ###")
    testExpiringCache()
}
```

## Patterns avancés avec sync

### 1. Singleton thread-safe avec Once

```go
type DatabaseConnection struct {
    connectionString string
    isConnected     bool
}

var (
    dbInstance *DatabaseConnection
    dbOnce     sync.Once
)

func GetDBConnection() *DatabaseConnection {
    dbOnce.Do(func() {
        fmt.Println("Initialisation de la connexion DB...")
        dbInstance = &DatabaseConnection{
            connectionString: "postgres://localhost/mydb",
            isConnected:     true,
        }
    })
    return dbInstance
}
```

### 2. Worker pool avec sync

```go
type WorkerPool struct {
    wg       sync.WaitGroup
    jobQueue chan Job
    quit     chan bool
}

func (wp *WorkerPool) Start(numWorkers int) {
    for i := 0; i < numWorkers; i++ {
        wp.wg.Add(1)
        go wp.worker()
    }
}

func (wp *WorkerPool) Stop() {
    close(wp.quit)
    wp.wg.Wait()
}

func (wp *WorkerPool) worker() {
    defer wp.wg.Done()
    for {
        select {
        case job := <-wp.jobQueue:
            job.Process()
        case <-wp.quit:
            return
        }
    }
}
```

### 3. Cache LRU thread-safe

```go
type LRUCache struct {
    mu       sync.RWMutex
    capacity int
    cache    map[string]*Node
    head     *Node
    tail     *Node
}

func (c *LRUCache) Get(key string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if node, exists := c.cache[key]; exists {
        c.moveToHead(node)
        return node.value, true
    }
    return nil, false
}

func (c *LRUCache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    if node, exists := c.cache[key]; exists {
        node.value = value
        c.moveToHead(node)
    } else {
        newNode := &Node{key: key, value: value}
        if len(c.cache) >= c.capacity {
            c.removeTail()
        }
        c.addToHead(newNode)
        c.cache[key] = newNode
    }
}
```

## Performance et optimisations

### 1. Réduire la contention

```go
// ❌ Mauvais : verrou trop large
func badExample(data []int) {
    mu.Lock()
    defer mu.Unlock()

    for i, v := range data {
        processValue(v)        // Traitement long sous verrou
        results[i] = v * 2
    }
}

// ✅ Bon : verrou minimal
func goodExample(data []int) {
    localResults := make([]int, len(data))

    for i, v := range data {
        localResults[i] = processValue(v) * 2  // Traitement hors verrou
    }

    mu.Lock()
    copy(results, localResults)  // Copie rapide sous verrou
    mu.Unlock()
}
```

### 2. Utiliser RWMutex pour les lectures

```go
type Stats struct {
    mu    sync.RWMutex  // Au lieu de sync.Mutex
    data  map[string]int
}

func (s *Stats) Get(key string) int {
    s.mu.RLock()         // Verrou de lecture partagé
    defer s.mu.RUnlock()
    return s.data[key]
}

func (s *Stats) Set(key string, value int) {
    s.mu.Lock()          // Verrou d'écriture exclusif
    defer s.mu.Unlock()
    s.data[key] = value
}
```

### 3. Pool pour éviter les allocations

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func processData(data []byte) {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)

    // Utiliser buf pour le traitement
    copy(buf, data)
    // ... traitement
}
```

## Debugging et monitoring

### 1. Détection de deadlock

```go
// Toujours acquérir les verrous dans le même ordre
func transfer(from, to *Account, amount int) {
    first, second := from, to
    if from.ID > to.ID {
        first, second = to, from
    }

    first.mu.Lock()
    defer first.mu.Unlock()

    second.mu.Lock()
    defer second.mu.Unlock()

    from.balance -= amount
    to.balance += amount
}
```

### 2. Métriques de performance

```go
type MonitoredMutex struct {
    mu           sync.Mutex
    waitTime     time.Duration
    lockCount    int64
    contentions  int64
}

func (m *MonitoredMutex) Lock() {
    start := time.Now()

    // Essayer d'acquérir le verrou sans bloquer
    if !m.mu.TryLock() {
        atomic.AddInt64(&m.contentions, 1)
        m.mu.Lock()  // Bloquer maintenant
    }

    waitTime := time.Since(start)
    atomic.AddInt64(&m.lockCount, 1)

    // Utiliser atomic ou un autre mutex pour protéger waitTime
}
```

## Conclusion

Le package `sync` est essentiel pour :

1. **Protéger les données partagées** avec Mutex/RWMutex
2. **Coordonner les goroutines** avec WaitGroup
3. **Initialiser une seule fois** avec Once
4. **Optimiser les allocations** avec Pool
5. **Synchroniser sur conditions** avec Cond

**Règle d'or** : Utilisez les channels pour communiquer, `sync` pour protéger. Commencez simple avec Mutex, optimisez avec RWMutex quand nécessaire.

**Bonnes pratiques** :
- Toujours utiliser `defer` avec `Unlock()`
- Minimiser la taille des sections critiques
- Éviter les verrous imbriqués
- Préférer RWMutex pour les lectures fréquentes
- Utiliser Once pour les initialisations coûteuses

La maîtrise du package `sync` vous permettra de construire des applications Go concurrentes robustes et performantes !

⏭️
