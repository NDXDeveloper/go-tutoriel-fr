🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10-1 : Channels buffered et unbuffered

## Introduction

Les channels sont au cœur de la concurrence en Go. Ils permettent aux goroutines de communiquer entre elles de manière sûre. Il existe deux types principaux de channels : les **unbuffered** (non-bufferisés) et les **buffered** (bufferisés). Comprendre leurs différences est essentiel pour écrire du code concurrent efficace.

## Channels unbuffered (non-bufferisés)

### Qu'est-ce qu'un channel unbuffered ?

Un channel unbuffered est un canal de communication **synchrone** entre goroutines. Cela signifie que l'envoi et la réception doivent se faire en même temps.

### Création d'un channel unbuffered

```go
ch := make(chan int)  // Channel unbuffered pour des int
```

### Comportement

- **Envoi bloquant** : Quand une goroutine envoie une valeur, elle se bloque jusqu'à ce qu'une autre goroutine reçoive cette valeur
- **Réception bloquante** : Quand une goroutine tente de recevoir, elle se bloque jusqu'à ce qu'une valeur soit envoyée

### Exemple simple

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string)

    // Lancement d'une goroutine qui envoie
    go func() {
        fmt.Println("Goroutine: Je vais envoyer un message...")
        ch <- "Hello World"
        fmt.Println("Goroutine: Message envoyé !")
    }()

    // Attendre un peu pour voir l'ordre d'exécution
    time.Sleep(100 * time.Millisecond)

    fmt.Println("Main: Je vais recevoir le message...")
    message := <-ch
    fmt.Println("Main: Reçu:", message)
}
```

**Sortie :**
```
Goroutine: Je vais envoyer un message...
Main: Je vais recevoir le message...
Main: Reçu: Hello World
Goroutine: Message envoyé !
```

### Analogie du monde réel

Imaginez deux personnes qui se passent un objet **directement de main à main**. L'une ne peut pas lâcher l'objet tant que l'autre ne l'a pas saisi. C'est exactement comme un channel unbuffered.

## Channels buffered (bufferisés)

### Qu'est-ce qu'un channel buffered ?

Un channel buffered possède une **capacité** définie. Il peut stocker plusieurs valeurs en attente sans bloquer l'envoi, tant que le buffer n'est pas plein.

### Création d'un channel buffered

```go
ch := make(chan int, 3)  // Channel avec un buffer de 3 éléments
```

### Comportement

- **Envoi non-bloquant** : Tant que le buffer n'est pas plein, l'envoi ne bloque pas
- **Réception non-bloquante** : Tant que le buffer n'est pas vide, la réception ne bloque pas
- **Blocage** : L'envoi bloque quand le buffer est plein, la réception bloque quand le buffer est vide

### Exemple simple

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string, 2)  // Buffer de 2 éléments

    // Envoi de plusieurs messages sans bloquer
    fmt.Println("Envoi du premier message...")
    ch <- "Message 1"
    fmt.Println("Premier message envoyé !")

    fmt.Println("Envoi du deuxième message...")
    ch <- "Message 2"
    fmt.Println("Deuxième message envoyé !")

    // Le troisième message bloquerait car le buffer est plein

    // Lancement d'une goroutine pour recevoir
    go func() {
        time.Sleep(100 * time.Millisecond)
        msg1 := <-ch
        fmt.Println("Goroutine reçu:", msg1)

        time.Sleep(100 * time.Millisecond)
        msg2 := <-ch
        fmt.Println("Goroutine reçu:", msg2)
    }()

    // Attendre que la goroutine termine
    time.Sleep(300 * time.Millisecond)
    fmt.Println("Programme terminé")
}
```

### Analogie du monde réel

Imaginez une **boîte aux lettres** avec plusieurs compartiments. Vous pouvez y déposer plusieurs lettres sans attendre que quelqu'un les récupère, tant qu'il y a de la place. C'est comme un channel buffered.

## Comparaison pratique

### Exemple avec unbuffered

```go
func exempleUnbuffered() {
    ch := make(chan int)

    go func() {
        fmt.Println("Goroutine: avant l'envoi")
        ch <- 42
        fmt.Println("Goroutine: après l'envoi")
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: avant la réception")
    val := <-ch
    fmt.Println("Main: reçu", val)
}
```

### Exemple avec buffered

```go
func exempleBuffered() {
    ch := make(chan int, 1)

    go func() {
        fmt.Println("Goroutine: avant l'envoi")
        ch <- 42
        fmt.Println("Goroutine: après l'envoi")  // S'exécute immédiatement
    }()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: avant la réception")
    val := <-ch
    fmt.Println("Main: reçu", val)
}
```

## Quand utiliser chaque type ?

### Utilisez un channel unbuffered quand :

- Vous voulez une **synchronisation stricte** entre goroutines
- Vous implementez un **handshake** (confirmation mutuelle)
- Vous voulez garantir qu'un message est **immédiatement traité**

### Utilisez un channel buffered quand :

- Vous voulez **découpler** l'envoi de la réception
- Vous implementez un **système de queue** (file d'attente)
- Vous voulez **améliorer les performances** en évitant des blocages
- Vous connaissez le **nombre maximum** d'éléments en attente

## Cas d'usage concrets

### 1. Notification d'événement (unbuffered)

```go
func notificationEvenement() {
    done := make(chan bool)

    go func() {
        fmt.Println("Traitement en cours...")
        time.Sleep(2 * time.Second)
        fmt.Println("Traitement terminé")
        done <- true  // Signal de fin
    }()

    fmt.Println("Attente de la fin du traitement...")
    <-done  // Attendre la notification
    fmt.Println("Tout est terminé !")
}
```

### 2. Queue de travail (buffered)

```go
func queueDeTravail() {
    jobs := make(chan int, 5)  // Queue de 5 jobs maximum

    // Producteur
    go func() {
        for i := 1; i <= 10; i++ {
            fmt.Printf("Ajout du job %d\n", i)
            jobs <- i
        }
        close(jobs)
    }()

    // Consommateur
    go func() {
        for job := range jobs {
            fmt.Printf("Traitement du job %d\n", job)
            time.Sleep(500 * time.Millisecond)
        }
    }()

    time.Sleep(6 * time.Second)
}
```

## Propriétés importantes

### Vérifier la capacité et la longueur

```go
func infoChannel() {
    ch := make(chan int, 5)

    ch <- 1
    ch <- 2
    ch <- 3

    fmt.Printf("Capacité: %d\n", cap(ch))  // 5
    fmt.Printf("Longueur: %d\n", len(ch))  // 3
    fmt.Printf("Libre: %d\n", cap(ch) - len(ch))  // 2
}
```

### Channel fermé

```go
func channelFerme() {
    ch := make(chan int, 2)

    ch <- 1
    ch <- 2
    close(ch)  // Fermer le channel

    // On peut encore lire les valeurs en attente
    fmt.Println(<-ch)  // 1
    fmt.Println(<-ch)  // 2
    fmt.Println(<-ch)  // 0 (valeur zéro)

    // Vérifier si le channel est fermé
    val, ok := <-ch
    fmt.Printf("Valeur: %d, Channel ouvert: %t\n", val, ok)
}
```

## Pièges à éviter

### 1. Deadlock avec unbuffered

```go
// ❌ ERREUR : Deadlock !
func deadlockExample() {
    ch := make(chan int)
    ch <- 42  // Bloque indéfiniment car personne ne lit
    fmt.Println(<-ch)
}

// ✅ CORRECT : Utiliser une goroutine
func correctExample() {
    ch := make(chan int)
    go func() {
        ch <- 42
    }()
    fmt.Println(<-ch)
}
```

### 2. Buffer trop petit

```go
// ❌ Peut causer des blocages inattendus
func bufferTropPetit() {
    ch := make(chan int, 1)

    ch <- 1
    ch <- 2  // Bloque car le buffer est plein !
}

// ✅ Adapter la taille du buffer aux besoins
func bufferAdapte() {
    ch := make(chan int, 3)  // Taille appropriée

    ch <- 1
    ch <- 2
    ch <- 3

    fmt.Println(<-ch, <-ch, <-ch)
}
```

## Exercices pratiques

### Exercice 1 : Ping-Pong (unbuffered)

Créez deux goroutines qui s'envoient un message en ping-pong 5 fois.

```go
func pingPong() {
    ping := make(chan string)
    pong := make(chan string)

    // Implémentez les goroutines ping et pong
    // Indication : une goroutine envoie "ping" et reçoit "pong"
    //              l'autre fait l'inverse
}
```

### Exercice 2 : Producteur-Consommateur (buffered)

Créez un système où un producteur génère des nombres de 1 à 10, et un consommateur les traite avec un délai.

```go
func producteurConsommateur() {
    buffer := make(chan int, 3)

    // Implémentez le producteur et le consommateur
    // Le producteur doit pouvoir envoyer plusieurs valeurs
    // même si le consommateur est lent
}
```

## Résumé

- **Unbuffered** : Communication synchrone, blocage mutuel, parfait pour la synchronisation
- **Buffered** : Communication asynchrone, découplage, parfait pour les queues
- **Capacité** : Définit combien d'éléments peuvent être stockés
- **Longueur** : Nombre actuel d'éléments dans le buffer
- **Fermeture** : `close(ch)` indique qu'aucune nouvelle valeur ne sera envoyée

La maîtrise des channels buffered et unbuffered est fondamentale pour créer des applications concurrentes robustes et performantes en Go.

# Solutions des exercices pratiques

## Exercice 1 : Ping-Pong (unbuffered)

### Solution complète

```go
package main

import (
    "fmt"
    "time"
)

func pingPong() {
    ping := make(chan string)
    pong := make(chan string)

    // Goroutine PING
    go func() {
        for i := 1; i <= 5; i++ {
            // Envoie "ping" et attend "pong"
            ping <- fmt.Sprintf("ping %d", i)
            response := <-pong
            fmt.Printf("Ping reçu: %s\n", response)
        }
        // Signal de fin
        ping <- "stop"
    }()

    // Goroutine PONG
    go func() {
        for {
            // Attend "ping"
            message := <-ping
            if message == "stop" {
                break
            }
            fmt.Printf("Pong reçu: %s\n", message)

            // Répond avec "pong"
            pong <- fmt.Sprintf("pong %d", len(message))
        }
    }()

    // Attendre que les goroutines terminent
    time.Sleep(1 * time.Second)
    fmt.Println("Ping-Pong terminé !")
}

func main() {
    pingPong()
}
```

### Sortie attendue

```
Pong reçu: ping 1
Ping reçu: pong 6
Pong reçu: ping 2
Ping reçu: pong 6
Pong reçu: ping 3
Ping reçu: pong 6
Pong reçu: ping 4
Ping reçu: pong 6
Pong reçu: ping 5
Ping reçu: pong 6
Ping-Pong terminé !
```

### Solution alternative (plus simple)

```go
func pingPongSimple() {
    ch := make(chan string)

    // Goroutine 1 : Commence par "ping"
    go func() {
        for i := 1; i <= 5; i++ {
            ch <- "ping"
            fmt.Printf("Envoyé: ping %d\n", i)

            msg := <-ch
            fmt.Printf("Reçu: %s %d\n", msg, i)
        }
    }()

    // Goroutine 2 : Répond avec "pong"
    go func() {
        for i := 1; i <= 5; i++ {
            msg := <-ch
            fmt.Printf("Reçu: %s %d\n", msg, i)

            ch <- "pong"
            fmt.Printf("Envoyé: pong %d\n", i)
        }
    }()

    time.Sleep(1 * time.Second)
}
```

### Explication

- **Channels unbuffered** : Chaque envoi bloque jusqu'à ce qu'il soit reçu
- **Synchronisation** : Les goroutines alternent naturellement
- **Signal d'arrêt** : Nécessaire pour terminer proprement les goroutines
- **Blocage mutuel** : Assure que les messages sont traités dans l'ordre

---

## Exercice 2 : Producteur-Consommateur (buffered)

### Solution complète

```go
package main

import (
    "fmt"
    "time"
)

func producteurConsommateur() {
    buffer := make(chan int, 3)
    done := make(chan bool)

    // Producteur
    go func() {
        fmt.Println("🏭 Producteur: Démarrage")
        for i := 1; i <= 10; i++ {
            fmt.Printf("🏭 Producteur: Génération du nombre %d\n", i)
            buffer <- i
            fmt.Printf("🏭 Producteur: Nombre %d envoyé (buffer: %d/%d)\n",
                      i, len(buffer), cap(buffer))

            // Petit délai pour visualiser le buffer
            time.Sleep(200 * time.Millisecond)
        }
        close(buffer)
        fmt.Println("🏭 Producteur: Terminé (buffer fermé)")
    }()

    // Consommateur
    go func() {
        fmt.Println("🛒 Consommateur: Démarrage")
        for number := range buffer {
            fmt.Printf("🛒 Consommateur: Traitement du nombre %d...\n", number)

            // Simulation d'un traitement lent
            time.Sleep(600 * time.Millisecond)

            fmt.Printf("🛒 Consommateur: Nombre %d traité !\n", number)
        }
        fmt.Println("🛒 Consommateur: Terminé")
        done <- true
    }()

    // Attendre la fin du consommateur
    <-done
    fmt.Println("✅ Système producteur-consommateur terminé")
}

func main() {
    producteurConsommateur()
}
```

### Sortie attendue (partielle)

```
🏭 Producteur: Démarrage
🛒 Consommateur: Démarrage
🏭 Producteur: Génération du nombre 1
🏭 Producteur: Nombre 1 envoyé (buffer: 1/3)
🛒 Consommateur: Traitement du nombre 1...
🏭 Producteur: Génération du nombre 2
🏭 Producteur: Nombre 2 envoyé (buffer: 2/3)
🏭 Producteur: Génération du nombre 3
🏭 Producteur: Nombre 3 envoyé (buffer: 3/3)
🏭 Producteur: Génération du nombre 4
🛒 Consommateur: Nombre 1 traité !
🛒 Consommateur: Traitement du nombre 2...
🏭 Producteur: Nombre 4 envoyé (buffer: 3/3)
...
```

### Solution avec plusieurs consommateurs

```go
func producteurMultiConsommateur() {
    buffer := make(chan int, 5)
    done := make(chan bool)

    // Producteur
    go func() {
        for i := 1; i <= 20; i++ {
            buffer <- i
            fmt.Printf("📦 Produit: %d\n", i)
            time.Sleep(100 * time.Millisecond)
        }
        close(buffer)
    }()

    // Plusieurs consommateurs
    numConsommateurs := 3
    for id := 1; id <= numConsommateurs; id++ {
        go func(consommateurID int) {
            for number := range buffer {
                fmt.Printf("👷 Consommateur %d traite: %d\n", consommateurID, number)
                time.Sleep(500 * time.Millisecond)
                fmt.Printf("✅ Consommateur %d terminé: %d\n", consommateurID, number)
            }
            done <- true
        }(id)
    }

    // Attendre tous les consommateurs
    for i := 0; i < numConsommateurs; i++ {
        <-done
    }
    fmt.Println("🎉 Tous les consommateurs ont terminé")
}
```

### Solution avec monitoring

```go
func producteurConsommateurAvecMonitoring() {
    buffer := make(chan int, 3)
    done := make(chan bool)
    stats := make(chan string, 10)

    // Moniteur du buffer
    go func() {
        ticker := time.NewTicker(300 * time.Millisecond)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                fmt.Printf("📊 Buffer: %d/%d éléments\n", len(buffer), cap(buffer))
            case stat := <-stats:
                fmt.Printf("📈 %s\n", stat)
            case <-done:
                return
            }
        }
    }()

    // Producteur avec statistiques
    go func() {
        for i := 1; i <= 10; i++ {
            buffer <- i
            stats <- fmt.Sprintf("Produit: %d", i)
            time.Sleep(200 * time.Millisecond)
        }
        close(buffer)
    }()

    // Consommateur avec statistiques
    go func() {
        count := 0
        for number := range buffer {
            count++
            stats <- fmt.Sprintf("Consommation #%d: %d", count, number)
            time.Sleep(600 * time.Millisecond)
        }
        stats <- fmt.Sprintf("Total consommé: %d éléments", count)
        done <- true
    }()

    <-done
    time.Sleep(100 * time.Millisecond) // Laisser le temps au monitoring
}
```

### Explication

- **Buffer de taille 3** : Permet au producteur d'avancer même si le consommateur est lent
- **Découplage** : Le producteur et le consommateur travaillent à leur propre rythme
- **Channel fermé** : `close(buffer)` indique la fin de production
- **Range sur channel** : `for number := range buffer` lit jusqu'à la fermeture
- **Synchronisation de fin** : Le channel `done` signale la fin du consommateur

### Avantages du buffering

1. **Performance** : Le producteur ne bloque pas à chaque envoi
2. **Flexibilité** : Gestion des pics de production
3. **Découplage** : Producteur et consommateur indépendants
4. **Scalabilité** : Facile d'ajouter plusieurs consommateurs

### Points importants

- **Taille du buffer** : Doit être adaptée au cas d'usage
- **Fermeture** : Toujours fermer le channel côté producteur
- **Gestion d'erreurs** : Dans un vrai système, ajouter la gestion d'erreurs
- **Graceful shutdown** : Utiliser des signaux pour arrêter proprement

Ces solutions montrent comment les channels buffered permettent de créer des systèmes robustes et performants pour le traitement asynchrone des données.

⏭️
