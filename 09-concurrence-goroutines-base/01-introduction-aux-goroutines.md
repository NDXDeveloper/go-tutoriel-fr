🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9-1 : Introduction aux goroutines

## Qu'est-ce qu'une goroutine ?

Une goroutine est une fonction qui s'exécute de manière concurrente avec d'autres goroutines dans le même espace d'adressage. C'est la façon dont Go gère la concurrence.

**Analogie simple** : Imaginez que vous préparez le dîner. Au lieu de faire chaque tâche l'une après l'autre (couper les légumes, puis faire cuire la viande, puis préparer la sauce), vous pouvez faire plusieurs choses en même temps. Les goroutines permettent à votre programme de faire de même.

## Différence avec les threads traditionnels

| Aspect | Threads OS | Goroutines |
|--------|------------|------------|
| **Taille mémoire** | ~2MB | ~2KB (initial) |
| **Création** | Coûteuse | Très rapide |
| **Gestion** | Par l'OS | Par le runtime Go |
| **Nombre recommandé** | Dizaines | Milliers |

## Votre première goroutine

### Syntaxe de base

Pour créer une goroutine, il suffit d'ajouter le mot-clé `go` devant un appel de fonction :

```go
package main

import (
    "fmt"
    "time"
)

func direBonjour() {
    fmt.Println("Bonjour depuis une goroutine !")
}

func main() {
    // Exécution normale
    direBonjour()

    // Exécution en goroutine
    go direBonjour()

    // Pause pour laisser le temps à la goroutine de s'exécuter
    time.Sleep(1 * time.Second)

    fmt.Println("Fin du programme")
}
```

**Sortie attendue :**
```
Bonjour depuis une goroutine !
Bonjour depuis une goroutine !
Fin du programme
```

### Explication pas à pas

1. **Ligne 1** : `direBonjour()` s'exécute normalement (de manière synchrone)
2. **Ligne 2** : `go direBonjour()` lance une goroutine (de manière asynchrone)
3. **Ligne 3** : `time.Sleep()` donne le temps à la goroutine de s'exécuter
4. **Ligne 4** : Le programme principal continue et se termine

## Exemple plus détaillé

```go
package main

import (
    "fmt"
    "time"
)

func compter(nom string) {
    for i := 1; i <= 5; i++ {
        fmt.Printf("%s: %d\n", nom, i)
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    fmt.Println("Démarrage du programme")

    // Lancement de deux goroutines
    go compter("Goroutine 1")
    go compter("Goroutine 2")

    // Exécution normale (sans goroutine)
    compter("Main")

    fmt.Println("Fin du programme")
}
```

**Sortie possible :**
```
Démarrage du programme
Main: 1
Goroutine 1: 1
Goroutine 2: 1
Main: 2
Goroutine 1: 2
Goroutine 2: 2
Main: 3
Goroutine 1: 3
Goroutine 2: 3
Main: 4
Goroutine 1: 4
Goroutine 2: 4
Main: 5
Goroutine 1: 5
Goroutine 2: 5
Fin du programme
```

## ⚠️ Piège courant : Le programme se termine trop tôt

```go
package main

import "fmt"

func direBonjour() {
    fmt.Println("Bonjour depuis une goroutine !")
}

func main() {
    go direBonjour()
    fmt.Println("Fin du programme")
    // Le programme se termine avant que la goroutine ait le temps de s'exécuter !
}
```

**Problème** : Le programme principal se termine avant que la goroutine ait le temps de s'exécuter.

**Solution temporaire** : Ajouter `time.Sleep()` (nous verrons de meilleures solutions plus tard).

## Goroutines avec des fonctions anonymes

Vous pouvez aussi créer des goroutines avec des fonctions anonymes :

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    fmt.Println("Démarrage")

    // Goroutine avec fonction anonyme
    go func() {
        fmt.Println("Goroutine anonyme exécutée !")
    }()

    // Goroutine avec paramètres
    go func(nom string, nombre int) {
        fmt.Printf("Salut %s, voici le nombre %d\n", nom, nombre)
    }("Alice", 42)

    time.Sleep(1 * time.Second)
    fmt.Println("Fin")
}
```

## Goroutines avec des valeurs de retour

⚠️ **Important** : Les goroutines ne peuvent pas retourner de valeurs directement.

```go
// ❌ Ceci ne fonctionne PAS
func calculer() int {
    return 42
}

func main() {
    resultat := go calculer() // Erreur de compilation !
}
```

**Pourquoi ?** Parce que la goroutine s'exécute de manière asynchrone. Le programme principal ne peut pas attendre le résultat sans mécanisme de synchronisation.

**Solution** : Utiliser des channels (que nous verrons dans la section suivante).

## Exemple pratique : Téléchargement concurrent

```go
package main

import (
    "fmt"
    "time"
)

func telechargerFichier(nomFichier string) {
    fmt.Printf("Début téléchargement de %s\n", nomFichier)

    // Simulation d'un téléchargement
    time.Sleep(2 * time.Second)

    fmt.Printf("Téléchargement de %s terminé\n", nomFichier)
}

func main() {
    fmt.Println("Démarrage des téléchargements")

    // Téléchargements séquentiels (lent)
    // telechargerFichier("fichier1.txt")
    // telechargerFichier("fichier2.txt")
    // telechargerFichier("fichier3.txt")

    // Téléchargements concurrents (rapide)
    go telechargerFichier("fichier1.txt")
    go telechargerFichier("fichier2.txt")
    go telechargerFichier("fichier3.txt")

    // Attendre que tous les téléchargements se terminent
    time.Sleep(3 * time.Second)

    fmt.Println("Tous les téléchargements terminés")
}
```

## Bonnes pratiques pour les débutants

### 1. **Utilisez `time.Sleep()` pour vos premiers tests**
C'est une solution temporaire mais pratique pour comprendre les goroutines.

### 2. **Évitez de partager des variables**
```go
// ❌ Évitez ceci au début
var compteur int

func incrementer() {
    compteur++ // Accès concurrent dangereux
}

func main() {
    go incrementer()
    go incrementer()
    // Résultat imprévisible !
}
```

### 3. **Commencez par des opérations indépendantes**
Les goroutines fonctionnent mieux quand elles n'ont pas besoin de partager des données.

### 4. **Une goroutine par tâche**
Créez une goroutine pour chaque tâche indépendante que vous voulez exécuter en parallèle.

## Résumé

Les goroutines sont des fonctions qui s'exécutent de manière concurrente. Elles permettent à votre programme de faire plusieurs choses en même temps :

- **Syntaxe simple** : `go maFonction()`
- **Très légères** : vous pouvez en créer des milliers
- **Exécution asynchrone** : ne bloquent pas le programme principal
- **Pas de valeur de retour directe** : utilisez des channels pour la communication

**Prochaine étape** : Apprendre les channels pour faire communiquer les goroutines entre elles de manière sûre et efficace.

## Exercices pratiques

### Exercice 1 : Première goroutine
Créez un programme qui affiche "Bonjour" depuis une goroutine et "Au revoir" depuis le programme principal.

### Exercice 2 : Plusieurs goroutines
Créez 3 goroutines qui affichent chacune leur nom et un compteur de 1 à 3.

### Exercice 3 : Simulation de tâches
Créez un programme qui simule 5 tâches s'exécutant en parallèle, chacune prenant un temps différent à se terminer.

**Note** : Les solutions utiliseront `time.Sleep()` pour l'instant. Nous verrons de meilleures façons de synchroniser les goroutines avec les channels.

# Solutions des exercices - Goroutines

## Exercice 1 : Première goroutine

**Énoncé** : Créez un programme qui affiche "Bonjour" depuis une goroutine et "Au revoir" depuis le programme principal.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

func direBonjour() {
    fmt.Println("Bonjour")
}

func main() {
    // Lancement de la goroutine
    go direBonjour()

    // Affichage depuis le programme principal
    fmt.Println("Au revoir")

    // Attendre que la goroutine se termine
    time.Sleep(100 * time.Millisecond)
}
```

### Sortie attendue
```
Au revoir
Bonjour
```
ou
```
Bonjour
Au revoir
```

### Explication
- La goroutine et le programme principal s'exécutent en parallèle
- L'ordre d'affichage peut varier selon l'ordonnancement
- `time.Sleep()` assure que la goroutine a le temps de s'exécuter avant la fin du programme

---

## Exercice 2 : Plusieurs goroutines

**Énoncé** : Créez 3 goroutines qui affichent chacune leur nom et un compteur de 1 à 3.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

func compter(nom string) {
    for i := 1; i <= 3; i++ {
        fmt.Printf("%s: %d\n", nom, i)
        time.Sleep(300 * time.Millisecond)
    }
}

func main() {
    fmt.Println("Démarrage des goroutines")

    // Lancement de 3 goroutines
    go compter("Goroutine-A")
    go compter("Goroutine-B")
    go compter("Goroutine-C")

    // Attendre que toutes les goroutines se terminent
    // 3 compteurs * 300ms = 900ms + marge de sécurité
    time.Sleep(1 * time.Second)

    fmt.Println("Toutes les goroutines terminées")
}
```

### Sortie possible
```
Démarrage des goroutines
Goroutine-A: 1
Goroutine-B: 1
Goroutine-C: 1
Goroutine-A: 2
Goroutine-B: 2
Goroutine-C: 2
Goroutine-A: 3
Goroutine-B: 3
Goroutine-C: 3
Toutes les goroutines terminées
```

### Explication
- Les 3 goroutines s'exécutent en parallèle
- Chaque goroutine affiche son nom et son compteur
- L'ordre peut varier mais le pattern reste cohérent
- Le `time.Sleep()` final assure que toutes les goroutines se terminent

---

## Exercice 3 : Simulation de tâches

**Énoncé** : Créez un programme qui simule 5 tâches s'exécutant en parallèle, chacune prenant un temps différent à se terminer.

### Solution

```go
package main

import (
    "fmt"
    "time"
)

func executerTache(nom string, duree time.Duration) {
    fmt.Printf("Début de la tâche %s (durée: %v)\n", nom, duree)

    // Simulation du travail
    time.Sleep(duree)

    fmt.Printf("Tâche %s terminée !\n", nom)
}

func main() {
    fmt.Println("Démarrage de 5 tâches en parallèle")

    // Définition des tâches avec différentes durées
    taches := map[string]time.Duration{
        "Tâche-1": 1 * time.Second,
        "Tâche-2": 500 * time.Millisecond,
        "Tâche-3": 1500 * time.Millisecond,
        "Tâche-4": 800 * time.Millisecond,
        "Tâche-5": 1200 * time.Millisecond,
    }

    // Lancement de toutes les tâches en parallèle
    for nom, duree := range taches {
        go executerTache(nom, duree)
    }

    // Attendre que toutes les tâches se terminent
    // La plus longue prend 1500ms + marge de sécurité
    time.Sleep(2 * time.Second)

    fmt.Println("Toutes les tâches terminées")
}
```

### Sortie possible
```
Démarrage de 5 tâches en parallèle
Début de la tâche Tâche-1 (durée: 1s)
Début de la tâche Tâche-2 (durée: 500ms)
Début de la tâche Tâche-3 (durée: 1.5s)
Début de la tâche Tâche-4 (durée: 800ms)
Début de la tâche Tâche-5 (durée: 1.2s)
Tâche Tâche-2 terminée !
Tâche Tâche-4 terminée !
Tâche Tâche-1 terminée !
Tâche Tâche-5 terminée !
Tâche Tâche-3 terminée !
Toutes les tâches terminées
```

### Solution alternative avec slice

```go
package main

import (
    "fmt"
    "time"
)

func executerTache(id int, duree time.Duration) {
    fmt.Printf("Début de la tâche %d (durée: %v)\n", id, duree)

    time.Sleep(duree)

    fmt.Printf("Tâche %d terminée !\n", id)
}

func main() {
    fmt.Println("Démarrage de 5 tâches en parallèle")

    // Durées différentes pour chaque tâche
    durees := []time.Duration{
        1 * time.Second,
        500 * time.Millisecond,
        1500 * time.Millisecond,
        800 * time.Millisecond,
        1200 * time.Millisecond,
    }

    // Lancement des tâches
    for i, duree := range durees {
        go executerTache(i+1, duree)
    }

    // Attendre la fin de toutes les tâches
    time.Sleep(2 * time.Second)

    fmt.Println("Toutes les tâches terminées")
}
```

### Explication
- 5 tâches s'exécutent en parallèle avec des durées différentes
- Les tâches se terminent dans l'ordre de leur durée (la plus courte en premier)
- Le temps total d'exécution est celui de la tâche la plus longue
- Sans goroutines, le temps total serait la somme de toutes les durées

---

## Comparaison : Avec et sans goroutines

### Version séquentielle (sans goroutines)

```go
func main() {
    fmt.Println("Exécution séquentielle")
    debut := time.Now()

    executerTache("Tâche-1", 1*time.Second)
    executerTache("Tâche-2", 500*time.Millisecond)
    executerTache("Tâche-3", 1500*time.Millisecond)
    executerTache("Tâche-4", 800*time.Millisecond)
    executerTache("Tâche-5", 1200*time.Millisecond)

    fmt.Printf("Temps total: %v\n", time.Since(debut))
    // Temps total: ~5 secondes
}
```

### Version concurrente (avec goroutines)

```go
func main() {
    fmt.Println("Exécution concurrente")
    debut := time.Now()

    go executerTache("Tâche-1", 1*time.Second)
    go executerTache("Tâche-2", 500*time.Millisecond)
    go executerTache("Tâche-3", 1500*time.Millisecond)
    go executerTache("Tâche-4", 800*time.Millisecond)
    go executerTache("Tâche-5", 1200*time.Millisecond)

    time.Sleep(2*time.Second)
    fmt.Printf("Temps total: %v\n", time.Since(debut))
    // Temps total: ~2 secondes
}
```

## Points importants à retenir

1. **Ordre d'exécution** : Les goroutines peuvent s'exécuter dans n'importe quel ordre
2. **Gain de performance** : Les tâches parallèles sont beaucoup plus rapides
3. **Synchronisation** : `time.Sleep()` est une solution temporaire pour attendre les goroutines
4. **Indépendance** : Ces exemples fonctionnent car les tâches sont indépendantes

## Prochaine étape

Ces exercices montrent la puissance des goroutines pour l'exécution parallèle. Cependant, utiliser `time.Sleep()` n'est pas une solution pratique dans le monde réel.

Dans la section suivante (9-2 : Channels basiques), nous apprendrons comment :
- Faire communiquer les goroutines
- Synchroniser leur exécution sans `time.Sleep()`
- Récupérer des résultats depuis les goroutines

⏭️
