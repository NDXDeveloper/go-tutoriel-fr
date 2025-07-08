🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7-3 : Panic et recover

## Introduction

Dans les sections précédentes, nous avons vu comment Go gère les erreurs de manière explicite avec des valeurs d'erreur. Cependant, Go propose aussi deux mécanismes pour des situations exceptionnelles : `panic` et `recover`. Ces mécanismes ressemblent aux exceptions d'autres langages, mais sont à utiliser avec parcimonie.

## Qu'est-ce qu'un panic ?

Un **panic** est un mécanisme qui arrête brutalement l'exécution normale d'un programme. C'est l'équivalent d'une "exception non gérée" dans d'autres langages.

### Quand se produit un panic ?

Un panic peut se produire dans deux situations :

1. **Automatiquement** par Go lors d'erreurs graves :
   - Division par zéro
   - Accès à un index invalide d'un slice/array
   - Accès à une clé inexistante d'une map (avec assertion)
   - Déréférencement d'un pointeur nil

2. **Manuellement** avec la fonction `panic()` :
   - Quand vous détectez une situation critique
   - Quand votre programme ne peut pas continuer

## Exemple de panic automatique

```go
package main

import "fmt"

func main() {
    // Panic automatique - accès à un index invalide
    nombres := []int{1, 2, 3}
    fmt.Println(nombres[10]) // ❌ Panic : index out of range
}
```

**Sortie :**
```
panic: runtime error: index out of range [10] with length 3
```

## Fonction panic()

Vous pouvez déclencher un panic manuellement avec la fonction `panic()` :

```go
package main

import "fmt"

func diviser(a, b float64) float64 {
    if b == 0 {
        panic("Division par zéro interdite!")
    }
    return a / b
}

func main() {
    fmt.Println("Début du programme")

    resultat := diviser(10, 2)
    fmt.Println("Résultat:", resultat)

    // Ceci va déclencher un panic
    resultat2 := diviser(10, 0)
    fmt.Println("Résultat 2:", resultat2) // Cette ligne ne s'exécutera jamais

    fmt.Println("Fin du programme") // Cette ligne ne s'exécutera jamais
}
```

**Sortie :**
```
Début du programme
Résultat: 5
panic: Division par zéro interdite!
```

## Qu'est-ce que recover ?

`recover()` est une fonction qui permet de "capturer" un panic et de reprendre le contrôle du programme. Elle ne fonctionne que dans une fonction `defer`.

### Syntaxe de base

```go
func main() {
    // defer doit être déclaré AVANT le panic
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Panic récupéré:", r)
        }
    }()

    // Code qui peut paniquer
    panic("Something went wrong!")

    fmt.Println("Cette ligne ne s'exécutera jamais")
}
```

## Exemple pratique avec recover

```go
package main

import "fmt"

func operationRisquee(nombre int) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Panic récupéré dans operationRisquee: %v\n", r)
        }
    }()

    if nombre < 0 {
        panic("Nombre négatif non autorisé")
    }

    // Simulation d'une opération qui peut paniquer
    slice := make([]int, nombre)
    fmt.Println("Accès à l'index:", slice[nombre-1])
}

func main() {
    fmt.Println("=== Test avec nombre valide ===")
    operationRisquee(5)

    fmt.Println("\n=== Test avec nombre négatif ===")
    operationRisquee(-1)

    fmt.Println("\n=== Test avec zéro ===")
    operationRisquee(0) // Ceci va aussi paniquer (index -1)

    fmt.Println("\n=== Programme terminé normalement ===")
}
```

**Sortie :**
```
=== Test avec nombre valide ===
Accès à l'index: 0

=== Test avec nombre négatif ===
Panic récupéré dans operationRisquee: Nombre négatif non autorisé

=== Test avec zéro ===
Panic récupéré dans operationRisquee: runtime error: index out of range [-1]

=== Programme terminé normalement ===
```

## Exemple avancé : Serveur web robuste

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

// Middleware pour récupérer les panics
func recoverMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic récupéré: %v", err)
                http.Error(w, "Erreur interne du serveur", http.StatusInternalServerError)
            }
        }()
        next(w, r)
    }
}

// Handler qui peut paniquer
func handlerRisque(w http.ResponseWriter, r *http.Request) {
    // Simulation d'une condition qui cause un panic
    if r.URL.Query().Get("crash") == "true" {
        panic("Crash simulé!")
    }

    fmt.Fprintf(w, "Hello, World!")
}

func main() {
    http.HandleFunc("/", recoverMiddleware(handlerRisque))

    fmt.Println("Serveur démarré sur :8080")
    fmt.Println("Testez:")
    fmt.Println("- http://localhost:8080/ (normal)")
    fmt.Println("- http://localhost:8080/?crash=true (panic)")

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Cas d'usage appropriés

### ✅ Quand utiliser panic

1. **Erreurs de programmation** (bugs dans le code)
```go
func traiterTableau(arr []int, index int) {
    if index < 0 || index >= len(arr) {
        panic(fmt.Sprintf("Index %d hors limites pour tableau de taille %d", index, len(arr)))
    }
    // Traitement...
}
```

2. **Initialisation critique échouée**
```go
func init() {
    fichierConfig, err := os.ReadFile("config.yaml")
    if err != nil {
        panic("Configuration obligatoire introuvable: " + err.Error())
    }
    // Traitement config...
}
```

3. **Conditions impossibles** (ne devrait jamais arriver)
```go
func traiterType(valeur interface{}) {
    switch v := valeur.(type) {
    case string:
        // Traitement string
    case int:
        // Traitement int
    default:
        panic(fmt.Sprintf("Type non supporté: %T", v))
    }
}
```

### ✅ Quand utiliser recover

1. **Serveurs web** (éviter qu'une requête fasse planter tout le serveur)
2. **Workers/goroutines** (éviter qu'une goroutine fasse planter le programme)
3. **Librairies** (protéger l'utilisateur de la librairie)

## Exemple pratique : Worker Pool sécurisé

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Fonction worker qui peut paniquer
func worker(id int, jobs <-chan int, results chan<- string, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        // Protection contre les panics
        func() {
            defer func() {
                if r := recover(); r != nil {
                    results <- fmt.Sprintf("Worker %d - Panic sur job %d: %v", id, job, r)
                }
            }()

            // Simulation d'un travail qui peut paniquer
            if job == 13 { // Numéro malchanceux
                panic("Numéro 13 interdit!")
            }

            if job%2 == 0 {
                time.Sleep(100 * time.Millisecond) // Simulation travail
                results <- fmt.Sprintf("Worker %d - Job %d terminé", id, job)
            } else {
                // Simulation d'une division par zéro
                _ = 10 / (job - job) // Ceci va paniquer si job == job
            }
        }()
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 15

    jobs := make(chan int, numJobs)
    results := make(chan string, numJobs)

    var wg sync.WaitGroup

    // Démarrage des workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Envoi des jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // Fermeture du canal results quand tous les workers ont terminé
    go func() {
        wg.Wait()
        close(results)
    }()

    // Lecture des résultats
    for result := range results {
        fmt.Println(result)
    }
}
```

## Erreurs communes à éviter

### ❌ Utiliser panic pour une gestion d'erreur normale

```go
// ❌ Mauvais - panic pour une erreur normale
func lireFichier(nom string) []byte {
    data, err := os.ReadFile(nom)
    if err != nil {
        panic(err) // Mauvais!
    }
    return data
}

// ✅ Bon - retourner une erreur
func lireFichier(nom string) ([]byte, error) {
    return os.ReadFile(nom)
}
```

### ❌ Recover dans la mauvaise fonction

```go
// ❌ Mauvais - recover ne fonctionne pas ici
func mauvaisRecover() {
    if r := recover(); r != nil { // Ne fonctionnera jamais
        fmt.Println("Panic récupéré:", r)
    }
    panic("Test panic")
}

// ✅ Bon - recover dans defer
func bonRecover() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Panic récupéré:", r)
        }
    }()
    panic("Test panic")
}
```

### ❌ Ignorer les panics récupérés

```go
// ❌ Mauvais - ignorer silencieusement
defer func() {
    recover() // Ignore le panic sans rien faire
}()

// ✅ Bon - logger ou traiter le panic
defer func() {
    if r := recover(); r != nil {
        log.Printf("Panic récupéré: %v", r)
        // Traitement approprié
    }
}()
```

## Bonnes pratiques

### 1. Préférer les erreurs aux panics

```go
// ✅ Bon - utiliser des erreurs pour les cas normaux
func diviser(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division par zéro")
    }
    return a / b, nil
}

// ❌ Moins bon - panic pour un cas prévisible
func diviser(a, b float64) float64 {
    if b == 0 {
        panic("division par zéro")
    }
    return a / b
}
```

### 2. Logger les panics récupérés

```go
defer func() {
    if r := recover(); r != nil {
        log.Printf("Panic récupéré dans %s: %v", "nomFonction", r)
        // Traitement approprié
    }
}()
```

### 3. Documenter les fonctions qui peuvent paniquer

```go
// CalculerRacine calcule la racine carrée d'un nombre.
// Panic si le nombre est négatif.
func CalculerRacine(n float64) float64 {
    if n < 0 {
        panic("racine carrée d'un nombre négatif")
    }
    return math.Sqrt(n)
}
```

## Points clés à retenir

1. **Panic** : arrête brutalement l'exécution (équivalent d'une exception)
2. **Recover** : permet de capturer un panic (uniquement dans `defer`)
3. **Usage** : panic pour des erreurs critiques, recover pour protéger les points d'entrée
4. **Préférer les erreurs** : utiliser panic/recover uniquement pour des cas exceptionnels
5. **Toujours logger** : ne jamais ignorer silencieusement un panic récupéré

Panic et recover sont des outils puissants mais à utiliser avec parcimonie. La philosophie Go privilégie la gestion explicite des erreurs avec des valeurs d'erreur plutôt que des exceptions.

⏭️
