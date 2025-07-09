🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9-3 : Select statement

## Qu'est-ce que le select statement ?

Le `select` statement est comme un `switch` pour les channels. Il permet d'attendre sur plusieurs opérations de channels simultanément et d'exécuter le premier cas qui devient prêt.

**Analogie simple** : Imaginez que vous attendez plusieurs appels téléphoniques importants. Au lieu d'attendre un appel spécifique, vous restez près du téléphone et répondez au premier qui sonne. Le `select` fait la même chose avec les channels.

## Pourquoi utiliser select ?

Dans les exemples précédents, nous recevions des channels un par un :

```go
// ❌ Réception séquentielle
resultat1 := <-ch1  // Attend ch1, puis...
resultat2 := <-ch2  // Attend ch2
```

Mais que faire si :
- Nous voulons traiter les résultats dans l'ordre où ils arrivent ?
- Nous voulons implémenter un timeout ?
- Nous voulons vérifier si un channel est prêt sans bloquer ?

Le `select` résout tous ces problèmes !

## Syntaxe de base

```go
select {
case valeur := <-ch1:
    // Exécuté si ch1 reçoit une valeur
    fmt.Println("Reçu de ch1:", valeur)

case ch2 <- 42:
    // Exécuté si on peut envoyer dans ch2
    fmt.Println("Envoyé dans ch2")

default:
    // Exécuté si aucun case n'est prêt (optionnel)
    fmt.Println("Aucun channel prêt")
}
```

## Votre premier exemple avec select

```go
package main

import (
    "fmt"
    "time"
)

func lent(ch chan string) {
    time.Sleep(2 * time.Second)
    ch <- "Tâche lente terminée"
}

func rapide(ch chan string) {
    time.Sleep(500 * time.Millisecond)
    ch <- "Tâche rapide terminée"
}

func main() {
    chLent := make(chan string)
    chRapide := make(chan string)

    // Lancement des deux tâches
    go lent(chLent)
    go rapide(chRapide)

    // Traitement du premier qui arrive
    select {
    case msg := <-chLent:
        fmt.Println("Premier:", msg)
    case msg := <-chRapide:
        fmt.Println("Premier:", msg)
    }

    fmt.Println("Programme terminé")
}
```

**Sortie :**
```
Premier: Tâche rapide terminée
Programme terminé
```

Le `select` a choisi le channel qui était prêt en premier (le rapide) !

## Select avec timeout

Un des usages les plus courants : implémenter un timeout.

```go
package main

import (
    "fmt"
    "time"
)

func tacheLongue(ch chan string) {
    time.Sleep(3 * time.Second)
    ch <- "Tâche terminée"
}

func main() {
    ch := make(chan string)
    go tacheLongue(ch)

    select {
    case resultat := <-ch:
        fmt.Println("Résultat:", resultat)

    case <-time.After(1 * time.Second):
        fmt.Println("Timeout ! La tâche prend trop de temps")
    }
}
```

**Sortie :**
```
Timeout ! La tâche prend trop de temps
```

### Explication de `time.After()`

`time.After(duree)` retourne un channel qui recevra une valeur après la durée spécifiée. C'est parfait pour les timeouts !

## Select avec default (non-bloquant)

Le cas `default` s'exécute immédiatement si aucun channel n'est prêt :

```go
package main

import "fmt"

func main() {
    ch := make(chan string)

    select {
    case msg := <-ch:
        fmt.Println("Message reçu:", msg)
    default:
        fmt.Println("Aucun message disponible")
    }
}
```

**Sortie :**
```
Aucun message disponible
```

### Vérification non-bloquante

Utilise pour vérifier si un channel a des données sans attendre :

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch := make(chan string, 1)  // Channel avec buffer

    // Envoi d'un message
    ch <- "Hello"

    // Vérification immédiate
    select {
    case msg := <-ch:
        fmt.Println("Message trouvé:", msg)
    default:
        fmt.Println("Pas de message")
    }

    // Deuxième vérification (channel maintenant vide)
    select {
    case msg := <-ch:
        fmt.Println("Message trouvé:", msg)
    default:
        fmt.Println("Pas de message")
    }
}
```

**Sortie :**
```
Message trouvé: Hello
Pas de message
```

## Exemple pratique : Serveur avec multiple sources

```go
package main

import (
    "fmt"
    "time"
)

func sourceA(ch chan string) {
    for i := 1; i <= 3; i++ {
        time.Sleep(1 * time.Second)
        ch <- fmt.Sprintf("Source A - Message %d", i)
    }
}

func sourceB(ch chan string) {
    for i := 1; i <= 3; i++ {
        time.Sleep(1500 * time.Millisecond)
        ch <- fmt.Sprintf("Source B - Message %d", i)
    }
}

func main() {
    chA := make(chan string)
    chB := make(chan string)
    quit := make(chan bool)

    go sourceA(chA)
    go sourceB(chB)

    // Simuler un signal d'arrêt après 4 secondes
    go func() {
        time.Sleep(4 * time.Second)
        quit <- true
    }()

    fmt.Println("Serveur démarré, en attente de messages...")

    for {
        select {
        case msgA := <-chA:
            fmt.Println("Reçu:", msgA)

        case msgB := <-chB:
            fmt.Println("Reçu:", msgB)

        case <-quit:
            fmt.Println("Signal d'arrêt reçu")
            return

        case <-time.After(3 * time.Second):
            fmt.Println("Pas de message depuis 3 secondes")
        }
    }
}
```

**Sortie possible :**
```
Serveur démarré, en attente de messages...
Reçu: Source A - Message 1
Reçu: Source B - Message 1
Reçu: Source A - Message 2
Reçu: Source A - Message 3
Signal d'arrêt reçu
```

## Select dans une boucle

Pattern très courant : utiliser `select` dans une boucle infinie pour créer un serveur ou un dispatcher :

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, taches <-chan string, resultats chan<- string) {
    for tache := range taches {
        time.Sleep(500 * time.Millisecond)
        resultats <- fmt.Sprintf("Worker %d a terminé: %s", id, tache)
    }
}

func main() {
    taches := make(chan string)
    resultats := make(chan string)
    done := make(chan bool)

    // Lancement de 2 workers
    go worker(1, taches, resultats)
    go worker(2, taches, resultats)

    // Envoi de tâches
    go func() {
        taches <- "Tâche A"
        taches <- "Tâche B"
        taches <- "Tâche C"
        taches <- "Tâche D"
        close(taches)
    }()

    // Signal de fin après 3 secondes
    go func() {
        time.Sleep(3 * time.Second)
        done <- true
    }()

    // Dispatcher principal
    compteur := 0
    for {
        select {
        case resultat := <-resultats:
            fmt.Println(resultat)
            compteur++
            if compteur == 4 {
                fmt.Println("Toutes les tâches terminées")
                return
            }

        case <-done:
            fmt.Println("Timeout - arrêt forcé")
            return
        }
    }
}
```

## Randomisation des cases

Si plusieurs cases sont prêts simultanément, Go choisit un case **aléatoirement** :

```go
package main

import "fmt"

func main() {
    ch1 := make(chan string, 1)
    ch2 := make(chan string, 1)

    // Préparation des deux channels
    ch1 <- "Message de ch1"
    ch2 <- "Message de ch2"

    // Les deux cases sont prêts - Go choisit aléatoirement
    for i := 0; i < 5; i++ {
        // Remplir à nouveau les channels
        ch1 <- fmt.Sprintf("ch1-%d", i)
        ch2 <- fmt.Sprintf("ch2-%d", i)

        select {
        case msg := <-ch1:
            fmt.Println("Choisi ch1:", msg)
        case msg := <-ch2:
            fmt.Println("Choisi ch2:", msg)
        }
    }
}
```

**Sortie possible :**
```
Choisi ch1: ch1-0
Choisi ch2: ch2-1
Choisi ch1: ch1-2
Choisi ch1: ch1-3
Choisi ch2: ch2-4
```

## Patterns courants avec select

### Pattern 1 : Fan-in (Combiner plusieurs sources)

```go
func fanIn(ch1, ch2 <-chan string) <-chan string {
    sortie := make(chan string)

    go func() {
        for {
            select {
            case msg := <-ch1:
                sortie <- msg
            case msg := <-ch2:
                sortie <- msg
            }
        }
    }()

    return sortie
}
```

### Pattern 2 : Worker avec timeout

```go
func workerAvecTimeout(taches <-chan string, timeout time.Duration) {
    for {
        select {
        case tache := <-taches:
            fmt.Println("Traitement:", tache)

        case <-time.After(timeout):
            fmt.Println("Pas de tâche reçue, arrêt du worker")
            return
        }
    }
}
```

### Pattern 3 : Heartbeat

```go
func heartbeat() {
    ticker := time.NewTicker(1 * time.Second)
    quit := make(chan bool)

    go func() {
        time.Sleep(5 * time.Second)
        quit <- true
    }()

    for {
        select {
        case <-ticker.C:
            fmt.Println("Heartbeat:", time.Now().Format("15:04:05"))

        case <-quit:
            ticker.Stop()
            fmt.Println("Heartbeat arrêté")
            return
        }
    }
}
```

## ⚠️ Pièges courants

### 1. Select sans default peut bloquer

```go
select {
case msg := <-ch:
    fmt.Println(msg)
// ❌ Bloque si ch n'a pas de données
}
```

**Solution :** Ajouter un `default` ou un timeout.

### 2. Goroutine leak avec select infini

```go
go func() {
    for {
        select {
        case msg := <-ch:
            process(msg)
        }
    }
}() // ❌ Cette goroutine ne se termine jamais !
```

**Solution :** Ajouter un cas pour l'arrêt :

```go
go func() {
    for {
        select {
        case msg := <-ch:
            process(msg)
        case <-quit:
            return  // ✅ Sortie propre
        }
    }
}()
```

### 3. Channels fermés dans select

```go
select {
case msg := <-ch:
    if msg == "" {
        // ❌ msg peut être la valeur zéro, pas forcément fermé
    }
}
```

**Solution :** Utiliser la syntaxe à deux valeurs :

```go
select {
case msg, ok := <-ch:
    if !ok {
        // ✅ Channel fermé
        return
    }
    // Traiter msg
}
```

## Exercice guidé : Chat simple

Créons un petit système de chat qui illustre l'utilisation du select :

```go
package main

import (
    "fmt"
    "time"
)

func utilisateur(nom string, messages chan<- string) {
    msgs := []string{
        nom + ": Salut tout le monde !",
        nom + ": Comment ça va ?",
        nom + ": À bientôt !",
    }

    for _, msg := range msgs {
        time.Sleep(1 * time.Second)
        messages <- msg
    }
}

func serveurChat() {
    messages := make(chan string)
    quit := make(chan bool)

    // Utilisateurs qui discutent
    go utilisateur("Alice", messages)
    go utilisateur("Bob", messages)

    // Arrêt après 5 secondes
    go func() {
        time.Sleep(5 * time.Second)
        quit <- true
    }()

    fmt.Println("=== Chat démarré ===")

    for {
        select {
        case msg := <-messages:
            fmt.Printf("[%s] %s\n",
                time.Now().Format("15:04:05"), msg)

        case <-quit:
            fmt.Println("=== Chat fermé ===")
            return

        case <-time.After(2 * time.Second):
            fmt.Println("--- Silence dans le chat ---")
        }
    }
}

func main() {
    serveurChat()
}
```

## Résumé

Le `select` statement est essentiel pour la programmation concurrente avancée :

- **Attente multiple** : Attend sur plusieurs channels simultanément
- **Non-bloquant** : Avec `default`, vérifie sans attendre
- **Timeouts** : Avec `time.After()`, implémente des timeouts
- **Randomisation** : Choisit aléatoirement si plusieurs cases sont prêts
- **Patterns puissants** : Fan-in, worker pools, heartbeats

**Règles importantes :**
- Un seul cas s'exécute par passage dans le select
- Si aucun cas n'est prêt et qu'il n'y a pas de `default`, le select bloque
- Les channels fermés sont toujours "prêts" (retournent la valeur zéro)

**Prochaine étape :** Patterns de concurrence plus avancés qui combinent goroutines, channels et select.

## Exercices pratiques

### Exercice 1 : Course de vitesse
Créez 3 goroutines qui font des calculs de durée différente. Utilisez select pour afficher le gagnant.

### Exercice 2 : Worker avec timeout
Créez un worker qui traite des tâches, mais s'arrête s'il n'y a pas de nouvelle tâche pendant 2 secondes.

### Exercice 3 : Multiplexeur de messages
Créez un système qui reçoit des messages de 3 sources différentes et les affiche avec un timestamp, plus un système d'arrêt propre.

# Solutions des exercices - Select statement

## Exercice 1 : Course de vitesse

**Énoncé** : Créez 3 goroutines qui font des calculs de durée différente. Utilisez select pour afficher le gagnant.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

func coureur(nom string, duree time.Duration, arrivee chan string) {
    fmt.Printf("%s: Début de la course\n", nom)

    // Simulation du calcul/travail
    time.Sleep(duree)

    arrivee <- fmt.Sprintf("%s terminé en %v", nom, duree)
}

func main() {
    fmt.Println("=== Course de vitesse ===")
    fmt.Println("3 coureurs s'affrontent !")

    // Channel pour recevoir les résultats
    arrivee := make(chan string)

    // Lancement des 3 coureurs avec des durées différentes
    go coureur("Alice", 800*time.Millisecond, arrivee)
    go coureur("Bob", 1200*time.Millisecond, arrivee)
    go coureur("Charlie", 500*time.Millisecond, arrivee)

    // Course : le premier qui arrive gagne !
    select {
    case gagnant := <-arrivee:
        fmt.Printf("🏆 GAGNANT: %s\n", gagnant)
    }

    // Optionnel : afficher les autres arrivées
    fmt.Println("\nAutres arrivées:")
    for i := 0; i < 2; i++ {
        select {
        case autre := <-arrivee:
            fmt.Printf("   %s\n", autre)
        case <-time.After(2 * time.Second):
            fmt.Println("   Timeout pour les retardataires")
            break
        }
    }

    fmt.Println("Course terminée !")
}
```

### Sortie attendue
```
=== Course de vitesse ===
3 coureurs s'affrontent !
Alice: Début de la course
Bob: Début de la course
Charlie: Début de la course
🏆 GAGNANT: Charlie terminé en 500ms

Autres arrivées:
   Alice terminé en 800ms
   Bob terminé en 1.2s
Course terminée !
```

### Solution alternative avec classement complet

```go
package main

import (
    "fmt"
    "time"
)

type Resultat struct {
    Nom   string
    Duree time.Duration
    Ordre int
}

func coureur(nom string, duree time.Duration, resultats chan Resultat) {
    debut := time.Now()
    fmt.Printf("%s: Début de la course\n", nom)

    // Simulation du calcul
    time.Sleep(duree)

    resultats <- Resultat{
        Nom:   nom,
        Duree: duree,
        Ordre: 0, // Sera défini par l'ordre d'arrivée
    }
}

func main() {
    fmt.Println("=== Course de vitesse avec classement ===")

    resultats := make(chan Resultat)
    coureurs := []struct {
        nom   string
        duree time.Duration
    }{
        {"Alice", 800 * time.Millisecond},
        {"Bob", 1200 * time.Millisecond},
        {"Charlie", 500 * time.Millisecond},
    }

    // Lancement de tous les coureurs
    for _, coureur := range coureurs {
        go coureur(coureur.nom, coureur.duree, resultats)
    }

    // Réception des résultats dans l'ordre d'arrivée
    fmt.Println("\n🏁 Classement final:")
    for position := 1; position <= len(coureurs); position++ {
        select {
        case resultat := <-resultats:
            emoji := "🥇"
            if position == 2 {
                emoji = "🥈"
            } else if position == 3 {
                emoji = "🥉"
            }
            fmt.Printf("%s %d. %s (%v)\n", emoji, position, resultat.Nom, resultat.Duree)

        case <-time.After(2 * time.Second):
            fmt.Printf("⏰ Position %d: Timeout\n", position)
        }
    }
}
```

---

## Exercice 2 : Worker avec timeout

**Énoncé** : Créez un worker qui traite des tâches, mais s'arrête s'il n'y a pas de nouvelle tâche pendant 2 secondes.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, taches <-chan string) {
    fmt.Printf("Worker %d: Démarré\n", id)
    tachesTraitees := 0

    for {
        select {
        case tache, ok := <-taches:
            if !ok {
                fmt.Printf("Worker %d: Channel fermé, arrêt\n", id)
                return
            }

            fmt.Printf("Worker %d: Traitement de '%s'\n", id, tache)
            time.Sleep(500 * time.Millisecond) // Simulation du travail
            tachesTraitees++
            fmt.Printf("Worker %d: '%s' terminée\n", id, tache)

        case <-time.After(2 * time.Second):
            fmt.Printf("Worker %d: Timeout ! Pas de tâche depuis 2s\n", id)
            fmt.Printf("Worker %d: Total traité: %d tâches\n", id, tachesTraitees)
            return
        }
    }
}

func main() {
    fmt.Println("=== Worker avec timeout ===")

    taches := make(chan string)

    // Lancement du worker
    go worker(1, taches)

    // Envoi de quelques tâches avec des pauses
    go func() {
        fmt.Println("Envoi de tâches...")

        taches <- "Tâche A"
        time.Sleep(1 * time.Second)

        taches <- "Tâche B"
        time.Sleep(1 * time.Second)

        taches <- "Tâche C"

        // Pause longue - va déclencher le timeout
        fmt.Println("Pause longue... (3 secondes)")
        time.Sleep(3 * time.Second)

        // Cette tâche ne sera pas traitée car le worker s'est arrêté
        taches <- "Tâche D"
    }()

    // Laisser le temps au système de fonctionner
    time.Sleep(8 * time.Second)
    fmt.Println("Programme principal terminé")
}
```

### Solution avec multiple workers

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func workerAvecTimeout(id int, taches <-chan string, wg *sync.WaitGroup) {
    defer wg.Done()

    fmt.Printf("Worker %d: Démarré\n", id)
    tachesTraitees := 0

    for {
        select {
        case tache, ok := <-taches:
            if !ok {
                fmt.Printf("Worker %d: Channel fermé (%d tâches traitées)\n", id, tachesTraitees)
                return
            }

            fmt.Printf("Worker %d: Traitement '%s'\n", id, tache)
            time.Sleep(300 * time.Millisecond)
            tachesTraitees++

        case <-time.After(2 * time.Second):
            fmt.Printf("Worker %d: Timeout - arrêt (%d tâches traitées)\n", id, tachesTraitees)
            return
        }
    }
}

func main() {
    fmt.Println("=== Plusieurs workers avec timeout ===")

    taches := make(chan string, 5)
    var wg sync.WaitGroup

    // Lancement de 3 workers
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go workerAvecTimeout(i, taches, &wg)
    }

    // Producteur de tâches
    go func() {
        for i := 1; i <= 8; i++ {
            taches <- fmt.Sprintf("Tâche-%d", i)
            time.Sleep(500 * time.Millisecond)
        }

        fmt.Println("Toutes les tâches envoyées, pause...")
        // Les workers vont timeout après 2 secondes
    }()

    wg.Wait()
    fmt.Println("Tous les workers terminés")
}
```

### Sortie attendue
```
=== Worker avec timeout ===
Worker 1: Démarré
Envoi de tâches...
Worker 1: Traitement de 'Tâche A'
Worker 1: 'Tâche A' terminée
Worker 1: Traitement de 'Tâche B'
Worker 1: 'Tâche B' terminée
Worker 1: Traitement de 'Tâche C'
Worker 1: 'Tâche C' terminée
Pause longue... (3 secondes)
Worker 1: Timeout ! Pas de tâche depuis 2s
Worker 1: Total traité: 3 tâches
Programme principal terminé
```

---

## Exercice 3 : Multiplexeur de messages

**Énoncé** : Créez un système qui reçoit des messages de 3 sources différentes et les affiche avec un timestamp, plus un système d'arrêt propre.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

type Message struct {
    Source  string
    Contenu string
    Heure   time.Time
}

func source(nom string, messages chan<- Message, intervalle time.Duration) {
    compteur := 1
    ticker := time.NewTicker(intervalle)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            msg := Message{
                Source:  nom,
                Contenu: fmt.Sprintf("Message %d de %s", compteur, nom),
                Heure:   time.Now(),
            }

            select {
            case messages <- msg:
                compteur++
            default:
                // Channel plein, on ignore ce message
                fmt.Printf("⚠️ Buffer plein, message de %s ignoré\n", nom)
            }
        }
    }
}

func multiplexeur() {
    fmt.Println("=== Multiplexeur de messages ===")

    // Channels
    messages := make(chan Message, 10) // Buffer pour éviter les blocages
    shutdown := make(chan bool)

    // Lancement des 3 sources avec des fréquences différentes
    go source("SourceA", messages, 1*time.Second)   // Rapide
    go source("SourceB", messages, 1500*time.Millisecond) // Moyen
    go source("SourceC", messages, 2*time.Second)   // Lent

    // Signal d'arrêt après 10 secondes
    go func() {
        time.Sleep(10 * time.Second)
        fmt.Println("\n🛑 Signal d'arrêt envoyé")
        shutdown <- true
    }()

    fmt.Println("Multiplexeur démarré, en attente de messages...\n")

    // Boucle principale du multiplexeur
    messagesRecus := 0
    for {
        select {
        case msg := <-messages:
            messagesRecus++
            fmt.Printf("[%s] 📨 %s: %s\n",
                msg.Heure.Format("15:04:05.000"),
                msg.Source,
                msg.Contenu)

        case <-shutdown:
            fmt.Printf("\n✅ Arrêt du multiplexeur\n")
            fmt.Printf("📊 Total de messages reçus: %d\n", messagesRecus)
            return

        case <-time.After(3 * time.Second):
            fmt.Println("⏰ Pas de message depuis 3 secondes...")
        }
    }
}

func main() {
    multiplexeur()
}
```

### Solution avancée avec statistiques

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type Message struct {
    Source    string
    Contenu   string
    Timestamp time.Time
    ID        int
}

type Statistiques struct {
    TotalMessages     int
    MessagesParSource map[string]int
    PremierMessage    time.Time
    DernierMessage    time.Time
}

func source(nom string, messages chan<- Message, intervalle time.Duration, quit <-chan bool) {
    ticker := time.NewTicker(intervalle)
    defer ticker.Stop()

    id := 1
    for {
        select {
        case <-quit:
            fmt.Printf("🔴 Source %s arrêtée\n", nom)
            return

        case <-ticker.C:
            msg := Message{
                Source:    nom,
                Contenu:   fmt.Sprintf("Message %d", id),
                Timestamp: time.Now(),
                ID:        id,
            }

            select {
            case messages <- msg:
                id++
            case <-time.After(100 * time.Millisecond):
                fmt.Printf("⚠️ Timeout envoi pour %s\n", nom)
            }
        }
    }
}

func multiplexeurAvance() {
    fmt.Println("=== Multiplexeur avancé ===")
    fmt.Println("Appuyez sur Ctrl+C pour arrêter proprement\n")

    // Channels
    messages := make(chan Message, 20)
    quit := make(chan bool)

    // Gestion des signaux système (Ctrl+C)
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

    // Statistiques
    stats := Statistiques{
        MessagesParSource: make(map[string]int),
    }

    // Lancement des sources
    sources := []struct {
        nom       string
        intervalle time.Duration
    }{
        {"API-Server", 800 * time.Millisecond},
        {"Database", 1200 * time.Millisecond},
        {"Cache", 2000 * time.Millisecond},
    }

    for _, src := range sources {
        go source(src.nom, messages, src.intervalle, quit)
    }

    fmt.Println("📡 Multiplexeur en marche...\n")

    // Boucle principale
    for {
        select {
        case msg := <-messages:
            // Mise à jour des statistiques
            stats.TotalMessages++
            stats.MessagesParSource[msg.Source]++
            stats.DernierMessage = msg.Timestamp

            if stats.TotalMessages == 1 {
                stats.PremierMessage = msg.Timestamp
            }

            // Affichage du message
            fmt.Printf("[%s] 📨 %s: %s (ID: %d)\n",
                msg.Timestamp.Format("15:04:05.000"),
                msg.Source,
                msg.Contenu,
                msg.ID)

        case <-sigChan:
            fmt.Println("\n🛑 Signal d'interruption reçu")
            goto shutdown

        case <-time.After(5 * time.Second):
            fmt.Println("⏰ Silence depuis 5 secondes...")
            fmt.Printf("📊 Messages reçus jusqu'à présent: %d\n", stats.TotalMessages)
        }
    }

shutdown:
    fmt.Println("🔄 Arrêt en cours...")

    // Signal d'arrêt aux sources
    close(quit)

    // Collecte des derniers messages
    fmt.Println("📥 Collecte des derniers messages...")
    timeout := time.After(2 * time.Second)

collectLoop:
    for {
        select {
        case msg := <-messages:
            stats.TotalMessages++
            stats.MessagesParSource[msg.Source]++
            fmt.Printf("[%s] 📨 %s: %s (final)\n",
                msg.Timestamp.Format("15:04:05.000"),
                msg.Source,
                msg.Contenu)

        case <-timeout:
            break collectLoop
        }
    }

    // Affichage des statistiques finales
    fmt.Println("\n📊 === STATISTIQUES FINALES ===")
    fmt.Printf("Total de messages: %d\n", stats.TotalMessages)
    fmt.Printf("Durée de fonctionnement: %v\n",
        stats.DernierMessage.Sub(stats.PremierMessage).Round(time.Second))

    fmt.Println("\nMessages par source:")
    for source, count := range stats.MessagesParSource {
        pourcentage := float64(count) * 100.0 / float64(stats.TotalMessages)
        fmt.Printf("  %s: %d (%.1f%%)\n", source, count, pourcentage)
    }

    fmt.Println("\n✅ Multiplexeur arrêté proprement")
}

func main() {
    multiplexeurAvance()
}
```

### Sortie attendue
```
=== Multiplexeur de messages ===
Multiplexeur démarré, en attente de messages...

[14:25:01.123] 📨 SourceA: Message 1 de SourceA
[14:25:01.623] 📨 SourceB: Message 1 de SourceB
[14:25:02.124] 📨 SourceA: Message 2 de SourceA
[14:25:02.124] 📨 SourceC: Message 1 de SourceC
[14:25:03.125] 📨 SourceA: Message 3 de SourceA
[14:25:03.125] 📨 SourceB: Message 2 de SourceB
[14:25:04.126] 📨 SourceA: Message 4 de SourceA
[14:25:04.126] 📨 SourceC: Message 2 de SourceC

🛑 Signal d'arrêt envoyé

✅ Arrêt du multiplexeur
📊 Total de messages reçus: 8
```

## Points importants à retenir

### 1. **Select avec priorités**
Si vous voulez donner la priorité à certains channels, vous pouvez imbriquer les select :

```go
select {
case msg := <-channelPrioritaire:
    // Traitement prioritaire
default:
    select {
    case msg := <-channelNormal:
        // Traitement normal
    case <-timeout:
        // Timeout
    }
}
```

### 2. **Gestion propre des ressources**
Toujours fermer les channels et arrêter les goroutines proprement :

```go
defer close(messages)
defer ticker.Stop()
```

### 3. **Buffers pour éviter les blocages**
Utilisez des channels avec buffer pour les systèmes à haute fréquence :

```go
messages := make(chan Message, 100) // Buffer de 100
```

### 4. **Timeouts adaptatifs**
Ajustez les timeouts selon le contexte :

```go
case <-time.After(getTimeoutDuration()):
```

## Patterns démontrés

1. **Fan-in** : Plusieurs sources vers un destinataire
2. **Timeout dynamique** : Workers qui s'arrêtent automatiquement
3. **Graceful shutdown** : Arrêt propre avec signal système
4. **Statistiques en temps réel** : Collecte de métriques
5. **Gestion des erreurs** : Timeouts et channels pleins

Ces exercices montrent comment le `select` statement est essentiel pour créer des systèmes concurrents robustes et réactifs en Go.

⏭️
