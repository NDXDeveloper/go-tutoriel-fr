🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9-2 : Channels basiques

## Qu'est-ce qu'un channel ?

Un channel (canal) est un conduit qui permet aux goroutines de communiquer entre elles et de synchroniser leur exécution. C'est comme un tuyau par lequel vous pouvez envoyer et recevoir des données.

**Analogie simple** : Imaginez un facteur qui livre des lettres. Le channel est comme la boîte aux lettres :
- Une personne peut **déposer** une lettre (envoyer des données)
- Une autre personne peut **récupérer** la lettre (recevoir des données)
- Si la boîte est vide, la personne qui veut récupérer une lettre doit **attendre**

## Pourquoi utiliser des channels ?

Dans la section précédente, nous avons utilisé `time.Sleep()` pour attendre que les goroutines se terminent. C'est pratique pour apprendre, mais ce n'est pas optimal car :

- ❌ Nous ne savons pas quand les goroutines se terminent réellement
- ❌ Nous ne pouvons pas récupérer de résultats depuis les goroutines
- ❌ Nous risquons d'attendre trop ou pas assez longtemps

Les channels résolvent ces problèmes !

## Syntaxe de base

### Déclaration d'un channel

```go
// Déclaration d'un channel qui transporte des entiers
var ch chan int

// Création d'un channel avec make
ch = make(chan int)

// Déclaration et création en une ligne
ch := make(chan int)
```

### Envoi de données

```go
ch <- 42  // Envoie la valeur 42 dans le channel
```

### Réception de données

```go
valeur := <-ch  // Reçoit une valeur depuis le channel
```

## Votre premier exemple avec channel

```go
package main

import "fmt"

func envoyerMessage(ch chan string) {
    ch <- "Bonjour depuis une goroutine !"
}

func main() {
    // Création d'un channel pour les strings
    ch := make(chan string)

    // Lancement de la goroutine
    go envoyerMessage(ch)

    // Réception du message (le programme attend ici)
    message := <-ch
    fmt.Println("Message reçu:", message)

    fmt.Println("Fin du programme")
}
```

**Sortie :**
```
Message reçu: Bonjour depuis une goroutine !
Fin du programme
```

### Pourquoi ça marche sans `time.Sleep()` ?

La ligne `message := <-ch` **bloque** le programme principal jusqu'à ce qu'une valeur soit disponible dans le channel. C'est une synchronisation automatique !

## Channels et synchronisation

### Exemple : Calculatrice concurrente

```go
package main

import "fmt"

func additionner(a, b int, resultat chan int) {
    somme := a + b
    resultat <- somme  // Envoie le résultat
}

func multiplier(a, b int, resultat chan int) {
    produit := a * b
    resultat <- produit  // Envoie le résultat
}

func main() {
    // Création de channels pour les résultats
    chAddition := make(chan int)
    chMultiplication := make(chan int)

    // Lancement des calculs en parallèle
    go additionner(10, 5, chAddition)
    go multiplier(10, 5, chMultiplication)

    // Réception des résultats
    somme := <-chAddition
    produit := <-chMultiplication

    fmt.Printf("10 + 5 = %d\n", somme)
    fmt.Printf("10 × 5 = %d\n", produit)
}
```

## Channels avec différents types

Les channels sont **typés**. Vous devez spécifier le type de données qu'ils transportent :

```go
package main

import "fmt"

func main() {
    // Channel pour des entiers
    chInt := make(chan int)

    // Channel pour des strings
    chString := make(chan string)

    // Channel pour des booleans
    chBool := make(chan bool)

    // Envoi en goroutines
    go func() {
        chInt <- 42
        chString <- "Hello"
        chBool <- true
    }()

    // Réception
    nombre := <-chInt
    texte := <-chString
    drapeau := <-chBool

    fmt.Printf("Nombre: %d, Texte: %s, Drapeau: %t\n", nombre, texte, drapeau)
}
```

## Exemple pratique : Téléchargeur avec feedback

Reprenons l'exemple du téléchargeur, mais cette fois avec des channels pour savoir quand c'est terminé :

```go
package main

import (
    "fmt"
    "time"
)

func telecharger(fichier string, termine chan string) {
    fmt.Printf("Début téléchargement: %s\n", fichier)

    // Simulation du téléchargement
    time.Sleep(time.Duration(len(fichier)*100) * time.Millisecond)

    // Notification de fin
    termine <- fmt.Sprintf("Téléchargement de %s terminé", fichier)
}

func main() {
    fmt.Println("Démarrage des téléchargements")

    // Channel pour recevoir les notifications
    termine := make(chan string)

    fichiers := []string{"document.pdf", "image.jpg", "video.mp4"}

    // Lancement des téléchargements
    for _, fichier := range fichiers {
        go telecharger(fichier, termine)
    }

    // Attendre que tous les téléchargements se terminent
    for i := 0; i < len(fichiers); i++ {
        message := <-termine
        fmt.Println(message)
    }

    fmt.Println("Tous les téléchargements terminés !")
}
```

**Sortie :**
```
Démarrage des téléchargements
Téléchargement de image.jpg terminé
Téléchargement de document.pdf terminé
Téléchargement de video.mp4 terminé
Tous les téléchargements terminés !
```

## Channels uni-directionnels

Vous pouvez créer des channels qui ne peuvent que **envoyer** ou que **recevoir** :

```go
package main

import "fmt"

// Cette fonction ne peut qu'envoyer dans le channel
func producteur(ch chan<- int) {
    for i := 1; i <= 3; i++ {
        ch <- i
    }
    close(ch)  // Ferme le channel quand terminé
}

// Cette fonction ne peut que recevoir depuis le channel
func consommateur(ch <-chan int) {
    for valeur := range ch {
        fmt.Printf("Reçu: %d\n", valeur)
    }
}

func main() {
    ch := make(chan int)

    go producteur(ch)
    consommateur(ch)
}
```

### Syntaxe des channels uni-directionnels

- `chan<- int` : channel qui ne peut qu'**envoyer** des entiers
- `<-chan int` : channel qui ne peut que **recevoir** des entiers
- `chan int` : channel bi-directionnel (peut envoyer et recevoir)

## Fermeture des channels

### Pourquoi fermer un channel ?

Fermer un channel indique qu'aucune valeur ne sera plus envoyée. C'est utile pour signaler la fin d'une séquence de données.

```go
package main

import "fmt"

func envoyerNombres(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch)  // Indique qu'on a fini d'envoyer
}

func main() {
    ch := make(chan int)

    go envoyerNombres(ch)

    // Réception jusqu'à fermeture du channel
    for {
        valeur, ok := <-ch
        if !ok {
            fmt.Println("Channel fermé")
            break
        }
        fmt.Printf("Reçu: %d\n", valeur)
    }
}
```

### Utilisation de `range` avec channels

Plus simple avec `range` :

```go
func main() {
    ch := make(chan int)

    go envoyerNombres(ch)

    // range s'arrête automatiquement quand le channel est fermé
    for valeur := range ch {
        fmt.Printf("Reçu: %d\n", valeur)
    }

    fmt.Println("Terminé")
}
```

## ⚠️ Erreurs courantes

### 1. Deadlock - Envoi sans récepteur

```go
func main() {
    ch := make(chan int)
    ch <- 42  // ❌ Bloque pour toujours !
    // Il n'y a personne pour recevoir
}
```

**Erreur :** `fatal error: all goroutines are asleep - deadlock!`

### 2. Deadlock - Réception sans envoyeur

```go
func main() {
    ch := make(chan int)
    valeur := <-ch  // ❌ Bloque pour toujours !
    // Il n'y a personne pour envoyer
}
```

### 3. Envoi dans un channel fermé

```go
func main() {
    ch := make(chan int)
    close(ch)
    ch <- 42  // ❌ Panic !
}
```

**Erreur :** `panic: send on closed channel`

## Patterns courants

### Pattern 1 : Signal de terminaison

```go
func worker(done chan bool) {
    fmt.Println("Travail en cours...")
    time.Sleep(1 * time.Second)
    fmt.Println("Travail terminé")
    done <- true  // Signal de fin
}

func main() {
    done := make(chan bool)
    go worker(done)

    <-done  // Attendre le signal
    fmt.Println("Programme terminé")
}
```

### Pattern 2 : Pipeline simple

```go
func generer(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch)
}

func doubler(entree <-chan int, sortie chan<- int) {
    for nombre := range entree {
        sortie <- nombre * 2
    }
    close(sortie)
}

func main() {
    ch1 := make(chan int)
    ch2 := make(chan int)

    go generer(ch1)
    go doubler(ch1, ch2)

    for resultat := range ch2 {
        fmt.Printf("Résultat: %d\n", resultat)
    }
}
```

## Résumé

Les channels permettent aux goroutines de communiquer de manière sûre :

- **Création** : `ch := make(chan type)`
- **Envoi** : `ch <- valeur`
- **Réception** : `valeur := <-ch`
- **Fermeture** : `close(ch)`
- **Synchronisation automatique** : les opérations bloquent jusqu'à ce qu'il y ait un correspondant

**Règle d'or** : "Ne communiquez pas en partageant la mémoire, partagez la mémoire en communiquant."

**Prochaine étape** : Apprendre le `select` statement pour gérer plusieurs channels simultanément.

## Exercices pratiques

### Exercice 1 : Calculatrice simple
Créez deux goroutines : une qui calcule la somme de 1 à 100, l'autre qui calcule le produit de 1 à 10. Utilisez des channels pour récupérer les résultats.

### Exercice 2 : Producteur-Consommateur
Créez une goroutine qui produit les nombres de 1 à 10 et une autre qui les affiche en ajoutant "Nombre: " devant chaque valeur.

### Exercice 3 : Workers parallèles
Créez 3 workers qui reçoivent des tâches depuis un channel commun et signalent leur fin via un autre channel.

# Solutions des exercices - Channels basiques

## Exercice 1 : Calculatrice simple

**Énoncé** : Créez deux goroutines : une qui calcule la somme de 1 à 100, l'autre qui calcule le produit de 1 à 10. Utilisez des channels pour récupérer les résultats.

### Solution

```go
package main

import "fmt"

func calculerSomme(debut, fin int, resultat chan int) {
    somme := 0
    for i := debut; i <= fin; i++ {
        somme += i
    }
    resultat <- somme
}

func calculerProduit(debut, fin int, resultat chan int) {
    produit := 1
    for i := debut; i <= fin; i++ {
        produit *= i
    }
    resultat <- produit
}

func main() {
    fmt.Println("Démarrage des calculs parallèles")

    // Création des channels pour les résultats
    chSomme := make(chan int)
    chProduit := make(chan int)

    // Lancement des calculs en parallèle
    go calculerSomme(1, 100, chSomme)
    go calculerProduit(1, 10, chProduit)

    // Réception des résultats
    somme := <-chSomme
    produit := <-chProduit

    // Affichage des résultats
    fmt.Printf("Somme de 1 à 100: %d\n", somme)
    fmt.Printf("Produit de 1 à 10: %d\n", produit)

    fmt.Println("Calculs terminés")
}
```

### Sortie attendue
```
Démarrage des calculs parallèles
Somme de 1 à 100: 5050
Produit de 1 à 10: 3628800
Calculs terminés
```

### Solution alternative avec un seul channel

```go
package main

import "fmt"

type Resultat struct {
    Type   string
    Valeur int
}

func calculerSomme(debut, fin int, resultat chan Resultat) {
    somme := 0
    for i := debut; i <= fin; i++ {
        somme += i
    }
    resultat <- Resultat{Type: "Somme", Valeur: somme}
}

func calculerProduit(debut, fin int, resultat chan Resultat) {
    produit := 1
    for i := debut; i <= fin; i++ {
        produit *= i
    }
    resultat <- Resultat{Type: "Produit", Valeur: produit}
}

func main() {
    fmt.Println("Démarrage des calculs parallèles")

    // Un seul channel pour les deux résultats
    resultat := make(chan Resultat)

    // Lancement des calculs
    go calculerSomme(1, 100, resultat)
    go calculerProduit(1, 10, resultat)

    // Réception des 2 résultats
    for i := 0; i < 2; i++ {
        res := <-resultat
        fmt.Printf("%s: %d\n", res.Type, res.Valeur)
    }

    fmt.Println("Calculs terminés")
}
```

### Explication
- Deux goroutines calculent en parallèle
- Chaque goroutine envoie son résultat via un channel
- Le programme principal attend les deux résultats
- Les calculs se font simultanément, pas séquentiellement

---

## Exercice 2 : Producteur-Consommateur

**Énoncé** : Créez une goroutine qui produit les nombres de 1 à 10 et une autre qui les affiche en ajoutant "Nombre: " devant chaque valeur.

### Solution

```go
package main

import "fmt"

func producteur(ch chan int) {
    fmt.Println("Producteur: Début de production")

    for i := 1; i <= 10; i++ {
        fmt.Printf("Producteur: Envoi de %d\n", i)
        ch <- i
    }

    close(ch) // Important: fermer le channel quand terminé
    fmt.Println("Producteur: Production terminée")
}

func consommateur(ch chan int) {
    fmt.Println("Consommateur: Prêt à recevoir")

    for nombre := range ch {
        fmt.Printf("Consommateur: Nombre: %d\n", nombre)
    }

    fmt.Println("Consommateur: Consommation terminée")
}

func main() {
    fmt.Println("Démarrage du système producteur-consommateur")

    // Channel pour transporter les nombres
    ch := make(chan int)

    // Lancement du producteur et consommateur
    go producteur(ch)
    go consommateur(ch)

    // Attendre que le consommateur termine
    // (il se termine automatiquement quand le channel est fermé)
    done := make(chan bool)

    go func() {
        consommateur(ch)
        done <- true
    }()

    <-done
    fmt.Println("Système terminé")
}
```

### Solution simplifiée

```go
package main

import "fmt"

func producteur(ch chan int) {
    for i := 1; i <= 10; i++ {
        ch <- i
    }
    close(ch)
}

func main() {
    ch := make(chan int)

    // Lancement du producteur
    go producteur(ch)

    // Consommation dans le main
    for nombre := range ch {
        fmt.Printf("Nombre: %d\n", nombre)
    }

    fmt.Println("Terminé")
}
```

### Sortie attendue
```
Nombre: 1
Nombre: 2
Nombre: 3
Nombre: 4
Nombre: 5
Nombre: 6
Nombre: 7
Nombre: 8
Nombre: 9
Nombre: 10
Terminé
```

### Explication
- Le producteur génère les nombres de 1 à 10
- Le consommateur reçoit et affiche chaque nombre
- `close(ch)` indique la fin de production
- `range ch` s'arrête automatiquement quand le channel est fermé

---

## Exercice 3 : Workers parallèles

**Énoncé** : Créez 3 workers qui reçoivent des tâches depuis un channel commun et signalent leur fin via un autre channel.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

type Tache struct {
    ID       int
    Donnees  string
}

type Resultat struct {
    TacheID  int
    WorkerID int
    Message  string
}

func worker(id int, taches <-chan Tache, resultats chan<- Resultat) {
    fmt.Printf("Worker %d: Démarré\n", id)

    for tache := range taches {
        fmt.Printf("Worker %d: Traitement tâche %d (%s)\n", id, tache.ID, tache.Donnees)

        // Simulation du travail
        time.Sleep(500 * time.Millisecond)

        // Envoi du résultat
        resultats <- Resultat{
            TacheID:  tache.ID,
            WorkerID: id,
            Message:  fmt.Sprintf("Tâche %d terminée par worker %d", tache.ID, id),
        }
    }

    fmt.Printf("Worker %d: Terminé\n", id)
}

func main() {
    fmt.Println("Démarrage du système de workers")

    // Channels
    taches := make(chan Tache, 10)      // Buffer pour éviter les blocages
    resultats := make(chan Resultat, 10)

    // Lancement de 3 workers
    for i := 1; i <= 3; i++ {
        go worker(i, taches, resultats)
    }

    // Envoi des tâches
    tachesAEnvoyer := []Tache{
        {ID: 1, Donnees: "Traiter fichier A"},
        {ID: 2, Donnees: "Calculer statistiques"},
        {ID: 3, Donnees: "Envoyer email"},
        {ID: 4, Donnees: "Sauvegarder données"},
        {ID: 5, Donnees: "Générer rapport"},
        {ID: 6, Donnees: "Nettoyer cache"},
    }

    fmt.Printf("Envoi de %d tâches\n", len(tachesAEnvoyer))
    for _, tache := range tachesAEnvoyer {
        taches <- tache
    }
    close(taches) // Signaler qu'il n'y a plus de tâches

    // Collecte des résultats
    for i := 0; i < len(tachesAEnvoyer); i++ {
        resultat := <-resultats
        fmt.Printf("Résultat: %s\n", resultat.Message)
    }

    fmt.Println("Toutes les tâches terminées")
}
```

### Solution simplifiée avec signal de fin

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, taches <-chan string, fini chan<- string) {
    for tache := range taches {
        fmt.Printf("Worker %d: %s\n", id, tache)
        time.Sleep(300 * time.Millisecond) // Simulation
    }
    fini <- fmt.Sprintf("Worker %d terminé", id)
}

func main() {
    taches := make(chan string, 5)
    fini := make(chan string)

    // 3 workers
    for i := 1; i <= 3; i++ {
        go worker(i, taches, fini)
    }

    // Envoi des tâches
    lesTaches := []string{
        "Tâche A", "Tâche B", "Tâche C",
        "Tâche D", "Tâche E", "Tâche F",
    }

    for _, tache := range lesTaches {
        taches <- tache
    }
    close(taches)

    // Attendre que tous les workers terminent
    for i := 0; i < 3; i++ {
        message := <-fini
        fmt.Println(message)
    }

    fmt.Println("Tous les workers terminés")
}
```

### Sortie possible
```
Worker 1: Démarré
Worker 2: Démarré
Worker 3: Démarré
Envoi de 6 tâches
Worker 1: Traitement tâche 1 (Traiter fichier A)
Worker 2: Traitement tâche 2 (Calculer statistiques)
Worker 3: Traitement tâche 3 (Envoyer email)
Worker 1: Traitement tâche 4 (Sauvegarder données)
Worker 2: Traitement tâche 5 (Générer rapport)
Worker 3: Traitement tâche 6 (Nettoyer cache)
Résultat: Tâche 1 terminée par worker 1
Résultat: Tâche 2 terminée par worker 2
Résultat: Tâche 3 terminée par worker 3
Résultat: Tâche 4 terminée par worker 1
Résultat: Tâche 5 terminée par worker 2
Résultat: Tâche 6 terminée par worker 3
Worker 1: Terminé
Worker 2: Terminé
Worker 3: Terminé
Toutes les tâches terminées
```

### Explication
- **Pool de workers** : 3 goroutines qui traitent les tâches
- **Channel commun** : Toutes les tâches passent par le même channel
- **Distribution automatique** : Les workers prennent les tâches disponibles
- **Synchronisation** : Le channel de résultats permet d'attendre la fin

---

## Comparaison des performances

### Version séquentielle
```go
func main() {
    debut := time.Now()

    for i := 1; i <= 6; i++ {
        fmt.Printf("Traitement tâche %d\n", i)
        time.Sleep(500 * time.Millisecond)
    }

    fmt.Printf("Temps total: %v\n", time.Since(debut))
    // Temps: ~3 secondes
}
```

### Version avec workers
```go
// Avec 3 workers parallèles
// Temps: ~1 seconde (3x plus rapide)
```

## Points importants à retenir

1. **Channels typés** : Chaque channel transporte un type spécifique
2. **Fermeture** : `close(ch)` indique qu'aucune valeur ne sera plus envoyée
3. **Range sur channels** : S'arrête automatiquement à la fermeture
4. **Channels buffered** : `make(chan type, taille)` permet d'éviter certains blocages
5. **Patterns courants** : Producteur-consommateur, pool de workers

## Prochaine étape

Ces exercices montrent la puissance des channels pour :
- Récupérer des résultats de calculs parallèles
- Implémenter des pipelines de données
- Créer des pools de workers

Dans la section suivante (9-3 : Select statement), nous apprendrons à gérer plusieurs channels simultanément et à implémenter des timeouts et des opérations non-bloquantes.

⏭️
