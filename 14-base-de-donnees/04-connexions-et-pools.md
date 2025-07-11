🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14-4 : Connexions et pools

## Introduction

Imaginez un restaurant où il n'y aurait qu'un seul serveur pour tous les clients. Ce serait très lent ! Les **pools de connexions** fonctionnent comme une équipe de serveurs : plusieurs connexions sont prêtes à servir simultanément vos requêtes de base de données.

### Qu'est-ce qu'une connexion de base de données ?

Une **connexion** est un "canal de communication" entre votre application Go et votre base de données. C'est comme une ligne téléphonique qui permet d'envoyer des requêtes SQL et de recevoir les résultats.

### Qu'est-ce qu'un pool de connexions ?

Un **pool de connexions** est un ensemble de connexions pré-établies et réutilisables. Au lieu de créer une nouvelle connexion pour chaque requête (coûteux), on emprunte une connexion du pool, on l'utilise, puis on la remet dans le pool.

## Pourquoi les pools sont-ils importants ?

### Sans pool (mauvaise approche) :
```go
// ❌ Mauvais : créer une nouvelle connexion à chaque fois
func obtenirUtilisateur(id int) (*Utilisateur, error) {
    // Nouvelle connexion (coûteuse !)
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        return nil, err
    }
    defer db.Close() // Ferme la connexion après usage

    // Faire la requête...
    return user, nil
}
```

### Avec pool (bonne approche) :
```go
// ✅ Bon : réutiliser les connexions d'un pool
var db *sql.DB // Pool de connexions global

func init() {
    // Créer le pool une seule fois au démarrage
    db, _ = sql.Open("postgres", "connection_string")
    configurePool(db)
}

func obtenirUtilisateur(id int) (*Utilisateur, error) {
    // Emprunter une connexion du pool
    // Pas besoin de db.Close() !
    return user, nil
}
```

## Avantages des pools de connexions

- 🚀 **Performance** : Pas de délai de création/destruction
- 💰 **Économie de ressources** : Réutilisation des connexions
- 🔒 **Contrôle de charge** : Limite le nombre de connexions simultanées
- 🛡️ **Stabilité** : Évite la surcharge de la base de données
- ⚡ **Rapidité** : Connexions déjà établies et prêtes

## Configuration de base avec database/sql

### Paramètres essentiels

```go
package main

import (
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq" // Driver PostgreSQL
)

func configurerPool(db *sql.DB) {
    // Nombre maximum de connexions ouvertes
    db.SetMaxOpenConns(25)

    // Nombre maximum de connexions inactives dans le pool
    db.SetMaxIdleConns(25)

    // Durée de vie maximum d'une connexion
    db.SetConnMaxLifetime(5 * time.Minute)

    // Durée maximum d'inactivité avant fermeture
    db.SetConnMaxIdleTime(1 * time.Minute)
}

func main() {
    // Créer le pool de connexions
    db, err := sql.Open("postgres",
        "host=localhost user=myuser password=mypass dbname=mydb sslmode=disable")
    if err != nil {
        panic(err)
    }
    defer db.Close()

    // Configurer le pool
    configurerPool(db)

    // Tester la connexion
    if err := db.Ping(); err != nil {
        panic("Impossible de se connecter à la base de données")
    }

    fmt.Println("Pool de connexions configuré et testé !")

    // Utiliser la base de données...
    exempleUtilisation(db)
}

func exempleUtilisation(db *sql.DB) {
    // Cette requête utilise automatiquement le pool
    rows, err := db.Query("SELECT id, nom FROM utilisateurs LIMIT 5")
    if err != nil {
        fmt.Printf("Erreur requête: %v\n", err)
        return
    }
    defer rows.Close()

    fmt.Println("Utilisateurs:")
    for rows.Next() {
        var id int
        var nom string
        rows.Scan(&id, &nom)
        fmt.Printf("- %d: %s\n", id, nom)
    }
}
```

### Explication des paramètres

#### SetMaxOpenConns(n)
```go
db.SetMaxOpenConns(25)
```
- **Définition** : Nombre maximum de connexions simultanées
- **Par défaut** : Illimité (peut être dangereux !)
- **Recommandation** : 10-30 pour la plupart des applications

#### SetMaxIdleConns(n)
```go
db.SetMaxIdleConns(10)
```
- **Définition** : Connexions gardées ouvertes en "standby"
- **Par défaut** : 2
- **Recommandation** : Environ la moitié de MaxOpenConns

#### SetConnMaxLifetime(duration)
```go
db.SetConnMaxLifetime(5 * time.Minute)
```
- **Définition** : Durée de vie maximum d'une connexion
- **Pourquoi** : Éviter les connexions "zombies"
- **Recommandation** : 5-30 minutes

#### SetConnMaxIdleTime(duration)
```go
db.SetConnMaxIdleTime(30 * time.Second)
```
- **Définition** : Temps d'inactivité avant fermeture
- **Pourquoi** : Libérer les ressources inutilisées
- **Recommandation** : 30 secondes à 2 minutes

## Configuration avec GORM

GORM utilise `database/sql` en arrière-plan, donc les mêmes principes s'appliquent :

```go
package main

import (
    "log"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type Utilisateur struct {
    gorm.Model
    Nom   string
    Email string
}

func configurerGORM() *gorm.DB {
    // Configuration de la connexion
    dsn := "host=localhost user=myuser password=mypass dbname=mydb port=5432 sslmode=disable"

    // Configuration GORM avec logger
    config := &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    }

    // Ouvrir la connexion
    db, err := gorm.Open(postgres.Open(dsn), config)
    if err != nil {
        log.Fatal("Erreur connexion GORM:", err)
    }

    // Récupérer l'instance SQL sous-jacente
    sqlDB, err := db.DB()
    if err != nil {
        log.Fatal("Erreur récupération SQL DB:", err)
    }

    // Configurer le pool
    sqlDB.SetMaxOpenConns(20)
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetConnMaxLifetime(5 * time.Minute)
    sqlDB.SetConnMaxIdleTime(1 * time.Minute)

    log.Println("Pool GORM configuré")
    return db
}

func main() {
    db := configurerGORM()

    // Migration automatique
    db.AutoMigrate(&Utilisateur{})

    // Test d'utilisation
    user := Utilisateur{Nom: "Alice", Email: "alice@example.com"}
    db.Create(&user)

    var users []Utilisateur
    db.Find(&users)

    log.Printf("Trouvé %d utilisateurs", len(users))
}
```

## Monitoring et statistiques

### Surveiller l'état du pool

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"
)

type PoolMonitor struct {
    db *sql.DB
}

func NewPoolMonitor(db *sql.DB) *PoolMonitor {
    return &PoolMonitor{db: db}
}

func (pm *PoolMonitor) AfficherStats() {
    stats := pm.db.Stats()

    fmt.Println("=== STATISTIQUES DU POOL ===")
    fmt.Printf("Connexions ouvertes: %d\n", stats.OpenConnections)
    fmt.Printf("Connexions en cours d'utilisation: %d\n", stats.InUse)
    fmt.Printf("Connexions inactives: %d\n", stats.Idle)
    fmt.Printf("Attentes de connexion: %d\n", stats.WaitCount)
    fmt.Printf("Durée d'attente totale: %v\n", stats.WaitDuration)
    fmt.Printf("Connexions fermées (max lifetime): %d\n", stats.MaxLifetimeClosed)
    fmt.Printf("Connexions fermées (max idle): %d\n", stats.MaxIdleClosed)
    fmt.Printf("Connexions fermées (max idle time): %d\n", stats.MaxIdleTimeClosed)
    fmt.Println("===============================")
}

func (pm *PoolMonitor) SurveillanceEnContinu() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        stats := pm.db.Stats()

        // Alertes simples
        if stats.WaitCount > 100 {
            log.Printf("⚠️  ALERTE: Trop d'attentes de connexion (%d)", stats.WaitCount)
        }

        if stats.OpenConnections >= 20 { // Supposons max = 25
            log.Printf("⚠️  ALERTE: Pool presque plein (%d/25)", stats.OpenConnections)
        }

        // Log périodique
        log.Printf("Pool: %d ouvertes, %d utilisées, %d inactives",
            stats.OpenConnections, stats.InUse, stats.Idle)
    }
}

// Exemple d'utilisation
func exempleMonitoring() {
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Configuration du pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)

    // Créer le moniteur
    monitor := NewPoolMonitor(db)

    // Surveillance en arrière-plan
    go monitor.SurveillanceEnContinu()

    // Affichage manuel
    monitor.AfficherStats()

    // Simulation de charge
    for i := 0; i < 100; i++ {
        go func(id int) {
            rows, err := db.Query("SELECT 1")
            if err != nil {
                log.Printf("Erreur requête %d: %v", id, err)
                return
            }
            rows.Close()
        }(i)
    }

    time.Sleep(10 * time.Second)
    monitor.AfficherStats()
}
```

## Test de charge et optimisation

### Simulateur de charge

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "sync"
    "time"
)

type LoadTester struct {
    db *sql.DB
}

func NewLoadTester(db *sql.DB) *LoadTester {
    return &LoadTester{db: db}
}

func (lt *LoadTester) TestConcurrency(nbRoutines, nbRequetesParRoutine int) {
    fmt.Printf("Test de charge: %d goroutines × %d requêtes\n",
        nbRoutines, nbRequetesParRoutine)

    var wg sync.WaitGroup
    start := time.Now()

    erreurs := make(chan error, nbRoutines*nbRequetesParRoutine)

    for i := 0; i < nbRoutines; i++ {
        wg.Add(1)
        go func(routineID int) {
            defer wg.Done()

            for j := 0; j < nbRequetesParRoutine; j++ {
                if err := lt.requeteSimple(); err != nil {
                    erreurs <- fmt.Errorf("routine %d, requête %d: %v",
                        routineID, j, err)
                }
            }
        }(i)
    }

    wg.Wait()
    close(erreurs)

    duration := time.Since(start)
    totalRequetes := nbRoutines * nbRequetesParRoutine

    // Compter les erreurs
    nbErreurs := 0
    for err := range erreurs {
        log.Println("Erreur:", err)
        nbErreurs++
    }

    fmt.Printf("Résultats du test:\n")
    fmt.Printf("- Durée totale: %v\n", duration)
    fmt.Printf("- Requêtes totales: %d\n", totalRequetes)
    fmt.Printf("- Requêtes réussies: %d\n", totalRequetes-nbErreurs)
    fmt.Printf("- Erreurs: %d\n", nbErreurs)
    fmt.Printf("- Requêtes/seconde: %.2f\n",
        float64(totalRequetes)/duration.Seconds())
}

func (lt *LoadTester) requeteSimple() error {
    rows, err := lt.db.Query("SELECT 1")
    if err != nil {
        return err
    }
    defer rows.Close()

    for rows.Next() {
        var dummy int
        if err := rows.Scan(&dummy); err != nil {
            return err
        }
    }

    return rows.Err()
}

// Benchmarker différentes configurations
func benchmarkConfigurations() {
    configs := []struct {
        nom           string
        maxOpen       int
        maxIdle       int
        maxLifetime   time.Duration
    }{
        {"Conservative", 5, 2, 1 * time.Minute},
        {"Balanced", 10, 5, 5 * time.Minute},
        {"Aggressive", 25, 10, 10 * time.Minute},
        {"High-Performance", 50, 25, 30 * time.Minute},
    }

    for _, config := range configs {
        fmt.Printf("\n=== Test configuration: %s ===\n", config.nom)

        db, err := sql.Open("postgres", "connection_string")
        if err != nil {
            log.Printf("Erreur connexion pour %s: %v", config.nom, err)
            continue
        }

        db.SetMaxOpenConns(config.maxOpen)
        db.SetMaxIdleConns(config.maxIdle)
        db.SetConnMaxLifetime(config.maxLifetime)

        tester := NewLoadTester(db)
        tester.TestConcurrency(10, 50) // 10 goroutines × 50 requêtes

        db.Close()
    }
}
```

## Gestion des timeouts

### Configuration des timeouts avec Context

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"
)

type DatabaseManager struct {
    db *sql.DB
}

func NewDatabaseManager(connectionString string) (*DatabaseManager, error) {
    db, err := sql.Open("postgres", connectionString)
    if err != nil {
        return nil, err
    }

    // Configuration optimisée avec timeouts
    db.SetMaxOpenConns(20)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(30 * time.Second)

    return &DatabaseManager{db: db}, nil
}

func (dm *DatabaseManager) RequeteAvecTimeout(query string, timeout time.Duration, args ...interface{}) (*sql.Rows, error) {
    // Créer un context avec timeout
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    // Exécuter la requête avec timeout
    return dm.db.QueryContext(ctx, query, args...)
}

func (dm *DatabaseManager) ExempleUtilisation() {
    // Requête rapide (timeout court)
    rows, err := dm.RequeteAvecTimeout(
        "SELECT id, nom FROM utilisateurs LIMIT 10",
        2*time.Second)
    if err != nil {
        fmt.Printf("Erreur requête rapide: %v\n", err)
        return
    }
    rows.Close()

    // Requête complexe (timeout plus long)
    rows, err = dm.RequeteAvecTimeout(`
        SELECT u.nom, COUNT(p.id) as nb_posts
        FROM utilisateurs u
        LEFT JOIN posts p ON u.id = p.user_id
        GROUP BY u.id, u.nom
        ORDER BY nb_posts DESC`,
        10*time.Second)
    if err != nil {
        fmt.Printf("Erreur requête complexe: %v\n", err)
        return
    }
    rows.Close()

    fmt.Println("Requêtes exécutées avec succès")
}

// Context avec deadline absolue
func (dm *DatabaseManager) RequeteAvecDeadline(query string, deadline time.Time, args ...interface{}) (*sql.Rows, error) {
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()

    return dm.db.QueryContext(ctx, query, args...)
}

// Context annulable manuellement
func (dm *DatabaseManager) RequeteAnnulable() {
    ctx, cancel := context.WithCancel(context.Background())

    // Annuler après 5 secondes
    go func() {
        time.Sleep(5 * time.Second)
        cancel()
    }()

    rows, err := dm.db.QueryContext(ctx, "SELECT pg_sleep(10)") // Requête longue
    if err != nil {
        fmt.Printf("Requête annulée: %v\n", err)
        return
    }
    rows.Close()
}
```

## Configuration selon l'environnement

### Configuration adaptive

```go
package main

import (
    "database/sql"
    "fmt"
    "os"
    "strconv"
    "time"
)

type EnvironmentConfig struct {
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration
}

func getConfigFromEnv() EnvironmentConfig {
    env := os.Getenv("APP_ENV")

    switch env {
    case "production":
        return EnvironmentConfig{
            MaxOpenConns:    50,
            MaxIdleConns:    25,
            ConnMaxLifetime: 30 * time.Minute,
            ConnMaxIdleTime: 5 * time.Minute,
        }
    case "staging":
        return EnvironmentConfig{
            MaxOpenConns:    20,
            MaxIdleConns:    10,
            ConnMaxLifetime: 10 * time.Minute,
            ConnMaxIdleTime: 2 * time.Minute,
        }
    case "development":
        return EnvironmentConfig{
            MaxOpenConns:    5,
            MaxIdleConns:    2,
            ConnMaxLifetime: 5 * time.Minute,
            ConnMaxIdleTime: 1 * time.Minute,
        }
    default:
        // Configuration par défaut
        return EnvironmentConfig{
            MaxOpenConns:    10,
            MaxIdleConns:    5,
            ConnMaxLifetime: 5 * time.Minute,
            ConnMaxIdleTime: 1 * time.Minute,
        }
    }
}

func getConfigFromEnvVars() EnvironmentConfig {
    config := EnvironmentConfig{}

    // Lire depuis les variables d'environnement avec valeurs par défaut
    if val := os.Getenv("DB_MAX_OPEN_CONNS"); val != "" {
        if parsed, err := strconv.Atoi(val); err == nil {
            config.MaxOpenConns = parsed
        }
    } else {
        config.MaxOpenConns = 25
    }

    if val := os.Getenv("DB_MAX_IDLE_CONNS"); val != "" {
        if parsed, err := strconv.Atoi(val); err == nil {
            config.MaxIdleConns = parsed
        }
    } else {
        config.MaxIdleConns = 10
    }

    if val := os.Getenv("DB_CONN_MAX_LIFETIME"); val != "" {
        if parsed, err := time.ParseDuration(val); err == nil {
            config.ConnMaxLifetime = parsed
        }
    } else {
        config.ConnMaxLifetime = 5 * time.Minute
    }

    if val := os.Getenv("DB_CONN_MAX_IDLE_TIME"); val != "" {
        if parsed, err := time.ParseDuration(val); err == nil {
            config.ConnMaxIdleTime = parsed
        }
    } else {
        config.ConnMaxIdleTime = 1 * time.Minute
    }

    return config
}

func configureDatabase(connectionString string) (*sql.DB, error) {
    db, err := sql.Open("postgres", connectionString)
    if err != nil {
        return nil, err
    }

    // Choisir la méthode de configuration
    config := getConfigFromEnv() // ou getConfigFromEnvVars()

    // Appliquer la configuration
    db.SetMaxOpenConns(config.MaxOpenConns)
    db.SetMaxIdleConns(config.MaxIdleConns)
    db.SetConnMaxLifetime(config.ConnMaxLifetime)
    db.SetConnMaxIdleTime(config.ConnMaxIdleTime)

    fmt.Printf("Base de données configurée pour %s:\n", os.Getenv("APP_ENV"))
    fmt.Printf("- MaxOpenConns: %d\n", config.MaxOpenConns)
    fmt.Printf("- MaxIdleConns: %d\n", config.MaxIdleConns)
    fmt.Printf("- ConnMaxLifetime: %v\n", config.ConnMaxLifetime)
    fmt.Printf("- ConnMaxIdleTime: %v\n", config.ConnMaxIdleTime)

    return db, nil
}
```

## Patterns avancés

### Pool Manager avec retry automatique

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"
)

type PoolManager struct {
    db     *sql.DB
    config EnvironmentConfig
}

func NewPoolManager(connectionString string) (*PoolManager, error) {
    pm := &PoolManager{
        config: getConfigFromEnv(),
    }

    if err := pm.connect(connectionString); err != nil {
        return nil, err
    }

    // Démarrer la surveillance
    go pm.healthCheck()

    return pm, nil
}

func (pm *PoolManager) connect(connectionString string) error {
    db, err := sql.Open("postgres", connectionString)
    if err != nil {
        return err
    }

    // Configuration du pool
    db.SetMaxOpenConns(pm.config.MaxOpenConns)
    db.SetMaxIdleConns(pm.config.MaxIdleConns)
    db.SetConnMaxLifetime(pm.config.ConnMaxLifetime)
    db.SetConnMaxIdleTime(pm.config.ConnMaxIdleTime)

    // Test de connexion
    if err := db.Ping(); err != nil {
        db.Close()
        return err
    }

    pm.db = db
    return nil
}

func (pm *PoolManager) healthCheck() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        if err := pm.db.Ping(); err != nil {
            log.Printf("❌ Health check failed: %v", err)
            pm.handleConnectionLoss()
        } else {
            stats := pm.db.Stats()
            if stats.WaitCount > 0 {
                log.Printf("⚠️  Pool pressure detected: %d waits", stats.WaitCount)
            }
        }
    }
}

func (pm *PoolManager) handleConnectionLoss() {
    log.Println("🔄 Attempting to reconnect...")

    // Fermer la connexion actuelle
    if pm.db != nil {
        pm.db.Close()
    }

    // Retry avec backoff exponentiel
    for attempt := 1; attempt <= 5; attempt++ {
        time.Sleep(time.Duration(attempt*attempt) * time.Second)

        if err := pm.connect("connection_string"); err != nil {
            log.Printf("Reconnection attempt %d failed: %v", attempt, err)
        } else {
            log.Printf("✅ Reconnected successfully after %d attempts", attempt)
            return
        }
    }

    log.Fatal("💥 Failed to reconnect after 5 attempts")
}

func (pm *PoolManager) GetDB() *sql.DB {
    return pm.db
}

func (pm *PoolManager) Close() error {
    return pm.db.Close()
}
```

### Circuit Breaker pour la base de données

```go
package main

import (
    "database/sql"
    "errors"
    "sync"
    "time"
)

type CircuitBreakerState int

const (
    StateClosed CircuitBreakerState = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu          sync.RWMutex
    state       CircuitBreakerState
    failureCount int
    lastFailure time.Time
    threshold   int
    timeout     time.Duration
}

type DatabaseWithCircuitBreaker struct {
    db      *sql.DB
    breaker *CircuitBreaker
}

func NewDatabaseWithCircuitBreaker(db *sql.DB) *DatabaseWithCircuitBreaker {
    return &DatabaseWithCircuitBreaker{
        db: db,
        breaker: &CircuitBreaker{
            threshold: 5,                // 5 échecs consécutifs
            timeout:   30 * time.Second, // Attendre 30s avant de réessayer
        },
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateOpen:
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = StateHalfOpen
            cb.failureCount = 0
        } else {
            return errors.New("circuit breaker is open")
        }
    }

    err := fn()

    if err != nil {
        cb.failureCount++
        cb.lastFailure = time.Now()

        if cb.failureCount >= cb.threshold {
            cb.state = StateOpen
        }
        return err
    }

    // Succès
    cb.failureCount = 0
    cb.state = StateClosed
    return nil
}

func (dwcb *DatabaseWithCircuitBreaker) Query(query string, args ...interface{}) (*sql.Rows, error) {
    var rows *sql.Rows
    var err error

    circuitErr := dwcb.breaker.Call(func() error {
        rows, err = dwcb.db.Query(query, args...)
        return err
    })

    if circuitErr != nil {
        return nil, circuitErr
    }

    return rows, err
}
```

## Recommandations par cas d'usage

### Application web classique
```go
func configWebApp(db *sql.DB) {
    db.SetMaxOpenConns(25)              // Trafic modéré
    db.SetMaxIdleConns(10)              // Garde quelques connexions
    db.SetConnMaxLifetime(5 * time.Minute)  // Renouvellement régulier
    db.SetConnMaxIdleTime(1 * time.Minute)  // Libération rapide
}
```

### API haute performance
```go
func configHighPerformanceAPI(db *sql.DB) {
    db.SetMaxOpenConns(100)             // Beaucoup de trafic concurrent
    db.SetMaxIdleConns(50)              // Garde beaucoup de connexions prêtes
    db.SetConnMaxLifetime(30 * time.Minute) // Durée de vie plus longue
    db.SetConnMaxIdleTime(5 * time.Minute)  // Patience plus longue
}
```

### Application batch/ETL
```go
func configBatchApp(db *sql.DB) {
    db.SetMaxOpenConns(5)               // Peu de concurrence
    db.SetMaxIdleConns(2)               // Minimum de connexions
    db.SetConnMaxLifetime(1 * time.Hour)   // Connexions longue durée
    db.SetConnMaxIdleTime(10 * time.Minute) // Patience pour les pauses
}
```

### Microservice
```go
func configMicroservice(db *sql.DB) {
    db.SetMaxOpenConns(10)              // Charge légère
    db.SetMaxIdleConns(3)               // Peu de connexions idle
    db.SetConnMaxLifetime(10 * time.Minute) // Renouvellement modéré
    db.SetConnMaxIdleTime(2 * time.Minute)  // Libération rapide
}
```

## Debugging et troubleshooting

### Problèmes courants et solutions

#### 1. "too many connections"
```go
// Problème: MaxOpenConns trop élevé
db.SetMaxOpenConns(1000) // ❌ Trop !

// Solution: Réduire et surveiller
db.SetMaxOpenConns(25)   // ✅ Plus raisonnable

// Diagnostic
stats := db.Stats()
if stats.OpenConnections >= maxOpen * 0.8 {
    log.Printf("⚠️  Pool utilisation: %d/%d", stats.OpenConnections, maxOpen)
}
```

#### 2. Connexions qui "traînent" (connection leaks)
```go
// ❌ Problème: Ne pas fermer les rows
func mauvaisFonction(db *sql.DB) {
    rows, err := db.Query("SELECT * FROM users")
    if err != nil {
        return
    }
    // OOPS! Pas de rows.Close() - fuite de connexion !

    for rows.Next() {
        // traitement...
    }
}

// ✅ Solution: Toujours fermer avec defer
func bonneFonction(db *sql.DB) {
    rows, err := db.Query("SELECT * FROM users")
    if err != nil {
        return
    }
    defer rows.Close() // ✅ Connexion libérée automatiquement

    for rows.Next() {
        // traitement...
    }
}

// 🔍 Diagnostic automatique
func detecterFuites(db *sql.DB) {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()

    var dernierInUse int

    for range ticker.C {
        stats := db.Stats()

        if stats.InUse > dernierInUse {
            log.Printf("📈 Connexions en cours: %d (était %d)",
                stats.InUse, dernierInUse)
        } else if stats.InUse > 0 && stats.InUse == dernierInUse {
            log.Printf("⚠️  Connexions bloquées? En cours: %d", stats.InUse)
        }

        dernierInUse = stats.InUse
    }
}
```

#### 3. Timeouts et requêtes lentes
```go
// ❌ Problème: Pas de timeout sur les requêtes lentes
func requeteSansTimeout(db *sql.DB) {
    // Cette requête peut bloquer indéfiniment
    rows, err := db.Query("SELECT * FROM huge_table ORDER BY complex_calculation")
    if err != nil {
        return
    }
    defer rows.Close()
}

// ✅ Solution: Utiliser Context avec timeout
func requeteAvecTimeout(db *sql.DB) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx, "SELECT * FROM huge_table ORDER BY complex_calculation")
    if err != nil {
        if err == context.DeadlineExceeded {
            log.Println("⏰ Requête annulée - timeout")
        }
        return
    }
    defer rows.Close()
}

// 🔍 Surveillance des requêtes lentes
type SlowQueryDetector struct {
    db        *sql.DB
    threshold time.Duration
}

func (sqd *SlowQueryDetector) QueryWithLogging(query string, args ...interface{}) (*sql.Rows, error) {
    start := time.Now()

    rows, err := sqd.db.Query(query, args...)

    duration := time.Since(start)
    if duration > sqd.threshold {
        log.Printf("🐌 Requête lente détectée: %v\nQuery: %s", duration, query)
    }

    return rows, err
}
```

#### 4. Pool mal dimensionné
```go
// 🔍 Analyseur de charge pour dimensionner le pool
type PoolAnalyzer struct {
    db           *sql.DB
    maxObserved  int
    samples      []int
    startTime    time.Time
}

func NewPoolAnalyzer(db *sql.DB) *PoolAnalyzer {
    return &PoolAnalyzer{
        db:        db,
        samples:   make([]int, 0),
        startTime: time.Now(),
    }
}

func (pa *PoolAnalyzer) StartAnalysis() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        stats := pa.db.Stats()
        concurrent := stats.InUse

        pa.samples = append(pa.samples, concurrent)

        if concurrent > pa.maxObserved {
            pa.maxObserved = concurrent
            log.Printf("📊 Nouveau pic de concurrence: %d", concurrent)
        }

        // Rapport toutes les minutes
        if len(pa.samples) >= 12 { // 12 échantillons = 1 minute
            pa.generateReport()
            pa.samples = pa.samples[:0] // Reset
        }
    }
}

func (pa *PoolAnalyzer) generateReport() {
    if len(pa.samples) == 0 {
        return
    }

    // Calculer moyenne et médiane
    sum := 0
    for _, sample := range pa.samples {
        sum += sample
    }
    moyenne := float64(sum) / float64(len(pa.samples))

    // Trier pour la médiane
    sorted := make([]int, len(pa.samples))
    copy(sorted, pa.samples)

    // Tri simple
    for i := 0; i < len(sorted); i++ {
        for j := i + 1; j < len(sorted); j++ {
            if sorted[i] > sorted[j] {
                sorted[i], sorted[j] = sorted[j], sorted[i]
            }
        }
    }

    mediane := sorted[len(sorted)/2]

    log.Printf("📈 Rapport de charge (dernière minute):")
    log.Printf("   Moyenne: %.1f connexions", moyenne)
    log.Printf("   Médiane: %d connexions", mediane)
    log.Printf("   Maximum: %d connexions", pa.maxObserved)
    log.Printf("   Recommandation MaxOpenConns: %d", pa.maxObserved*2)
}
```

#### 5. Gestion des erreurs de connexion
```go
type RobustDB struct {
    db           *sql.DB
    retryCount   int
    retryDelay   time.Duration
    circuitOpen  bool
    lastError    time.Time
}

func NewRobustDB(connectionString string) (*RobustDB, error) {
    db, err := sql.Open("postgres", connectionString)
    if err != nil {
        return nil, err
    }

    return &RobustDB{
        db:         db,
        retryCount: 3,
        retryDelay: 1 * time.Second,
    }, nil
}

func (rdb *RobustDB) QueryWithRetry(query string, args ...interface{}) (*sql.Rows, error) {
    // Vérifier le circuit breaker
    if rdb.circuitOpen && time.Since(rdb.lastError) < 30*time.Second {
        return nil, errors.New("circuit breaker open - too many recent failures")
    }

    var lastErr error

    for attempt := 0; attempt <= rdb.retryCount; attempt++ {
        if attempt > 0 {
            log.Printf("🔄 Tentative %d/%d pour la requête", attempt+1, rdb.retryCount+1)
            time.Sleep(rdb.retryDelay * time.Duration(attempt))
        }

        rows, err := rdb.db.Query(query, args...)
        if err == nil {
            // Succès - réinitialiser le circuit breaker
            rdb.circuitOpen = false
            return rows, nil
        }

        lastErr = err

        // Analyser le type d'erreur
        if isConnectionError(err) {
            log.Printf("❌ Erreur de connexion: %v", err)
            continue // Retry
        } else {
            // Erreur SQL - ne pas retry
            break
        }
    }

    // Toutes les tentatives ont échoué
    rdb.circuitOpen = true
    rdb.lastError = time.Now()
    return nil, fmt.Errorf("failed after %d attempts: %v", rdb.retryCount+1, lastErr)
}

func isConnectionError(err error) bool {
    errStr := err.Error()
    connectionErrors := []string{
        "connection refused",
        "connection reset",
        "broken pipe",
        "network is unreachable",
        "timeout",
    }

    for _, connErr := range connectionErrors {
        if strings.Contains(errStr, connErr) {
            return true
        }
    }
    return false
}
```

## Monitoring avancé

### Dashboard de monitoring en temps réel
```go
package main

import (
    "encoding/json"
    "fmt"
    "html/template"
    "net/http"
    "time"
)

type DatabaseMonitor struct {
    db    *sql.DB
    stats []PoolStats
}

type PoolStats struct {
    Timestamp       time.Time `json:"timestamp"`
    OpenConnections int       `json:"openConnections"`
    InUse           int       `json:"inUse"`
    Idle            int       `json:"idle"`
    WaitCount       int64     `json:"waitCount"`
    WaitDuration    string    `json:"waitDuration"`
}

func NewDatabaseMonitor(db *sql.DB) *DatabaseMonitor {
    dm := &DatabaseMonitor{
        db:    db,
        stats: make([]PoolStats, 0),
    }

    // Collecter les stats toutes les 10 secondes
    go dm.collectStats()

    return dm
}

func (dm *DatabaseMonitor) collectStats() {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        stats := dm.db.Stats()

        poolStats := PoolStats{
            Timestamp:       time.Now(),
            OpenConnections: stats.OpenConnections,
            InUse:           stats.InUse,
            Idle:            stats.Idle,
            WaitCount:       stats.WaitCount,
            WaitDuration:    stats.WaitDuration.String(),
        }

        dm.stats = append(dm.stats, poolStats)

        // Garder seulement les 100 derniers points (16 minutes)
        if len(dm.stats) > 100 {
            dm.stats = dm.stats[1:]
        }
    }
}

func (dm *DatabaseMonitor) ServeHTTP() {
    http.HandleFunc("/", dm.dashboardHandler)
    http.HandleFunc("/api/stats", dm.statsHandler)

    fmt.Println("Dashboard disponible sur http://localhost:8080")
    http.ListenAndServe(":8080", nil)
}

func (dm *DatabaseMonitor) dashboardHandler(w http.ResponseWriter, r *http.Request) {
    tmpl := `
<!DOCTYPE html>
<html>
<head>
    <title>Database Pool Monitor</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .container { max-width: 1200px; margin: 0 auto; }
        .stats { display: flex; gap: 20px; margin-bottom: 20px; }
        .stat-card {
            background: #f5f5f5;
            padding: 15px;
            border-radius: 5px;
            text-align: center;
            flex: 1;
        }
        .chart-container { width: 100%; height: 400px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Database Pool Monitor</h1>

        <div class="stats">
            <div class="stat-card">
                <h3>Connexions ouvertes</h3>
                <div id="openConns">-</div>
            </div>
            <div class="stat-card">
                <h3>En utilisation</h3>
                <div id="inUse">-</div>
            </div>
            <div class="stat-card">
                <h3>Inactives</h3>
                <div id="idle">-</div>
            </div>
            <div class="stat-card">
                <h3>Total attentes</h3>
                <div id="waitCount">-</div>
            </div>
        </div>

        <div class="chart-container">
            <canvas id="chart"></canvas>
        </div>
    </div>

    <script>
        const ctx = document.getElementById('chart').getContext('2d');
        const chart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: [],
                datasets: [{
                    label: 'Connexions ouvertes',
                    data: [],
                    borderColor: 'rgb(75, 192, 192)',
                    tension: 0.1
                }, {
                    label: 'En utilisation',
                    data: [],
                    borderColor: 'rgb(255, 99, 132)',
                    tension: 0.1
                }, {
                    label: 'Inactives',
                    data: [],
                    borderColor: 'rgb(54, 162, 235)',
                    tension: 0.1
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    y: {
                        beginAtZero: true
                    }
                }
            }
        });

        function updateStats() {
            fetch('/api/stats')
                .then(response => response.json())
                .then(data => {
                    if (data.length > 0) {
                        const latest = data[data.length - 1];
                        document.getElementById('openConns').textContent = latest.openConnections;
                        document.getElementById('inUse').textContent = latest.inUse;
                        document.getElementById('idle').textContent = latest.idle;
                        document.getElementById('waitCount').textContent = latest.waitCount;
                    }

                    // Mettre à jour le graphique
                    chart.data.labels = data.map(d => new Date(d.timestamp).toLocaleTimeString());
                    chart.data.datasets[0].data = data.map(d => d.openConnections);
                    chart.data.datasets[1].data = data.map(d => d.inUse);
                    chart.data.datasets[2].data = data.map(d => d.idle);
                    chart.update();
                });
        }

        // Mettre à jour toutes les 5 secondes
        setInterval(updateStats, 5000);
        updateStats(); // Premier chargement
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    fmt.Fprint(w, tmpl)
}

func (dm *DatabaseMonitor) statsHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(dm.stats)
}

// Exemple d'utilisation
func exempleMonitoring() {
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Configuration du pool
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(5 * time.Minute)

    // Démarrer le monitoring
    monitor := NewDatabaseMonitor(db)
    monitor.ServeHTTP() // Bloquant
}
```

## Exemples concrets d'applications

### 1. API REST avec pool optimisé
```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "time"

    "github.com/gorilla/mux"
)

type APIServer struct {
    db *sql.DB
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func NewAPIServer() *APIServer {
    // Configuration spécifique API REST
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        log.Fatal("Erreur connexion:", err)
    }

    // Configuration pour API REST (beaucoup de requêtes courtes)
    db.SetMaxOpenConns(50)  // Haute concurrence
    db.SetMaxIdleConns(25)  // Beaucoup de connexions prêtes
    db.SetConnMaxLifetime(10 * time.Minute)
    db.SetConnMaxIdleTime(2 * time.Minute)

    return &APIServer{db: db}
}

func (api *APIServer) getUserHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "ID invalide", http.StatusBadRequest)
        return
    }

    // Timeout court pour API responsive
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    var user User
    err = api.db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", userID).
        Scan(&user.ID, &user.Name, &user.Email)

    if err != nil {
        if err == sql.ErrNoRows {
            http.Error(w, "Utilisateur non trouvé", http.StatusNotFound)
        } else {
            log.Printf("Erreur BDD: %v", err)
            http.Error(w, "Erreur serveur", http.StatusInternalServerError)
        }
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (api *APIServer) getUsersHandler(w http.ResponseWriter, r *http.Request) {
    // Timeout plus long pour les listes
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    rows, err := api.db.QueryContext(ctx,
        "SELECT id, name, email FROM users ORDER BY id LIMIT 100")
    if err != nil {
        log.Printf("Erreur requête: %v", err)
        http.Error(w, "Erreur serveur", http.StatusInternalServerError)
        return
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var user User
        if err := rows.Scan(&user.ID, &user.Name, &user.Email); err != nil {
            log.Printf("Erreur scan: %v", err)
            continue
        }
        users = append(users, user)
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func (api *APIServer) Start() {
    r := mux.NewRouter()
    r.HandleFunc("/users/{id}", api.getUserHandler).Methods("GET")
    r.HandleFunc("/users", api.getUsersHandler).Methods("GET")

    log.Println("API démarrée sur :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

### 2. Worker pour traitement batch
```go
package main

import (
    "context"
    "database/sql"
    "log"
    "sync"
    "time"
)

type BatchProcessor struct {
    db          *sql.DB
    workerCount int
    batchSize   int
}

func NewBatchProcessor() *BatchProcessor {
    // Configuration pour traitement batch
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        log.Fatal("Erreur connexion:", err)
    }

    // Configuration pour batch (longues transactions)
    db.SetMaxOpenConns(10)   // Moins de concurrence
    db.SetMaxIdleConns(5)    // Économie de ressources
    db.SetConnMaxLifetime(30 * time.Minute) // Connexions longue durée
    db.SetConnMaxIdleTime(5 * time.Minute)  // Patience entre batches

    return &BatchProcessor{
        db:          db,
        workerCount: 5,
        batchSize:   1000,
    }
}

func (bp *BatchProcessor) ProcessAllUsers() {
    // Canal pour distribuer le travail
    userIDs := make(chan int, bp.batchSize)

    // Lancer les workers
    var wg sync.WaitGroup
    for i := 0; i < bp.workerCount; i++ {
        wg.Add(1)
        go bp.worker(i, userIDs, &wg)
    }

    // Alimenter le canal avec les IDs à traiter
    go func() {
        defer close(userIDs)

        // Timeout long pour les gros datasets
        ctx, cancel := context.WithTimeout(context.Background(), 1*time.Hour)
        defer cancel()

        rows, err := bp.db.QueryContext(ctx,
            "SELECT id FROM users WHERE status = 'pending' ORDER BY id")
        if err != nil {
            log.Printf("Erreur requête users: %v", err)
            return
        }
        defer rows.Close()

        for rows.Next() {
            var userID int
            if err := rows.Scan(&userID); err != nil {
                log.Printf("Erreur scan: %v", err)
                continue
            }

            select {
            case userIDs <- userID:
            case <-ctx.Done():
                log.Println("Timeout pendant la lecture des users")
                return
            }
        }
    }()

    // Attendre tous les workers
    wg.Wait()
    log.Println("Traitement batch terminé")
}

func (bp *BatchProcessor) worker(id int, userIDs <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()

    log.Printf("Worker %d démarré", id)

    for userID := range userIDs {
        if err := bp.processUser(userID); err != nil {
            log.Printf("Worker %d - Erreur traitement user %d: %v", id, userID, err)
        }
    }

    log.Printf("Worker %d terminé", id)
}

func (bp *BatchProcessor) processUser(userID int) error {
    // Traitement avec transaction
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    tx, err := bp.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Simuler un traitement complexe
    _, err = tx.ExecContext(ctx,
        "UPDATE users SET status = 'processing', updated_at = NOW() WHERE id = $1",
        userID)
    if err != nil {
        return err
    }

    // Simulated work
    time.Sleep(100 * time.Millisecond)

    _, err = tx.ExecContext(ctx,
        "UPDATE users SET status = 'completed', updated_at = NOW() WHERE id = $1",
        userID)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

## Configuration finale recommandée

### Template de configuration universelle
```go
package config

import (
    "database/sql"
    "fmt"
    "os"
    "strconv"
    "time"
)

type DBConfig struct {
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
    ConnMaxIdleTime time.Duration

    // Paramètres de surveillance
    EnableMonitoring bool
    HealthCheckInterval time.Duration

    // Paramètres de robustesse
    RetryAttempts int
    RetryDelay    time.Duration
}

func LoadDBConfig() DBConfig {
    return DBConfig{
        MaxOpenConns:        getEnvAsInt("DB_MAX_OPEN_CONNS", 25),
        MaxIdleConns:        getEnvAsInt("DB_MAX_IDLE_CONNS", 10),
        ConnMaxLifetime:     getEnvAsDuration("DB_CONN_MAX_LIFETIME", 5*time.Minute),
        ConnMaxIdleTime:     getEnvAsDuration("DB_CONN_MAX_IDLE_TIME", 1*time.Minute),
        EnableMonitoring:    getEnvAsBool("DB_ENABLE_MONITORING", true),
        HealthCheckInterval: getEnvAsDuration("DB_HEALTH_CHECK_INTERVAL", 30*time.Second),
        RetryAttempts:       getEnvAsInt("DB_RETRY_ATTEMPTS", 3),
        RetryDelay:          getEnvAsDuration("DB_RETRY_DELAY", 1*time.Second),
    }
}

func (config DBConfig) Apply(db *sql.DB) {
    db.SetMaxOpenConns(config.MaxOpenConns)
    db.SetMaxIdleConns(config.MaxIdleConns)
    db.SetConnMaxLifetime(config.ConnMaxLifetime)
    db.SetConnMaxIdleTime(config.ConnMaxIdleTime)

    if config.EnableMonitoring {
        go monitorPool(db, config.HealthCheckInterval)
    }
}

func getEnvAsInt(name string, defaultVal int) int {
    if val := os.Getenv(name); val != "" {
        if parsed, err := strconv.Atoi(val); err == nil {
            return parsed
        }
    }
    return defaultVal
}

func getEnvAsDuration(name string, defaultVal time.Duration) time.Duration {
    if val := os.Getenv(name); val != "" {
        if parsed, err := time.ParseDuration(val); err == nil {
            return parsed
        }
    }
    return defaultVal
}

func getEnvAsBool(name string, defaultVal bool) bool {
    if val := os.Getenv(name); val != "" {
        if parsed, err := strconv.ParseBool(val); err == nil {
            return parsed
        }
    }
    return defaultVal
}

func monitorPool(db *sql.DB, interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for range ticker.C {
        stats := db.Stats()

        // Logs de monitoring
        if stats.WaitCount > 0 {
            fmt.Printf("⚠️  Pool pressure: %d waits, duration: %v\n",
                stats.WaitCount, stats.WaitDuration)
        }

        if stats.OpenConnections > stats.MaxOpenConnections*80/100 {
            fmt.Printf("⚠️  Pool utilization high: %d/%d\n",
                stats.OpenConnections, stats.MaxOpenConnections)
        }
    }
}
```

## Résumé et bonnes pratiques

### ✅ À faire
- **Configurer le pool** selon votre type d'application
- **Monitorer** les statistiques régulièrement
- **Utiliser des timeouts** avec Context
- **Fermer les ressources** avec defer
- **Tester sous charge** avant la production
- **Adapter la configuration** selon l'environnement

### ❌ À éviter
- Pool trop petit (goulot d'étranglement)
- Pool trop grand (surcharge de la BDD)
- Oublier de fermer les rows/statements
- Pas de timeout sur les requêtes
- Configuration identique pour tous les environnements

### 📊 Métriques importantes à surveiller
- `OpenConnections` : Nombre de connexions actives
- `InUse` : Connexions en cours d'utilisation
- `WaitCount` : Nombre d'attentes de connexions
- `WaitDuration` : Temps d'attente total
- `MaxLifetimeClosed` : Connexions fermées par expiration

### 🎯 Valeurs recommandées par défaut
```go
// Configuration universelle
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(1 * time.Minute)
```

Les pools de connexions sont essentiels pour des applications performantes et stables. Une bonne configuration peut faire la différence entre une application qui s'effondre sous la charge et une qui reste réactive même avec beaucoup d'utilisateurs simultanés.

Dans la prochaine section, nous explorerons les **patterns avancés** et l'**architecture** pour construire des applications Go robustes avec des bases de données.

⏭️
