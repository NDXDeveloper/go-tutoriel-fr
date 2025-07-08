🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6-4 : Composition vs héritage

## Introduction

Go ne supporte pas l'**héritage** au sens traditionnel des langages orientés objet. Au lieu de cela, Go privilégie la **composition** pour créer des relations entre types. Cette approche suit le principe "Composition over Inheritance" et rend le code plus flexible et maintenable.

Dans cette section, nous allons comprendre pourquoi Go a fait ce choix et comment utiliser efficacement la composition.

## Qu'est-ce que l'héritage (pour comparaison) ?

Dans les langages orientés objet traditionnels, l'héritage permet à une classe d'hériter des propriétés et méthodes d'une classe parent :

```
// Pseudo-code (pas du Go!)
class Animal {
    nom string
    age int

    manger() { ... }
    dormir() { ... }
}

class Chien extends Animal {  // Héritage
    race string

    aboyer() { ... }
}
```

## La composition en Go

En Go, nous utilisons la **composition** : nous incluons un type dans un autre type.

### Composition explicite

```go
package main

import "fmt"

// Type de base
type Animal struct {
    Nom string
    Age int
}

func (a Animal) Manger() {
    fmt.Printf("%s mange\n", a.Nom)
}

func (a Animal) Dormir() {
    fmt.Printf("%s dort\n", a.Nom)
}

// Composition explicite
type Chien struct {
    Animal Animal  // Champ explicite
    Race   string
}

func (c Chien) Aboyer() {
    fmt.Printf("%s aboie: Woof!\n", c.Animal.Nom)
}

func main() {
    chien := Chien{
        Animal: Animal{Nom: "Rex", Age: 3},
        Race:   "Labrador",
    }

    // Accès explicite aux méthodes de Animal
    chien.Animal.Manger()
    chien.Animal.Dormir()
    chien.Aboyer()

    fmt.Printf("Race: %s\n", chien.Race)
}
```

### Composition par embedding (recommandée)

Go permet l'**embedding** (incorporation) qui rend l'utilisation plus naturelle :

```go
package main

import "fmt"

type Animal struct {
    Nom string
    Age int
}

func (a Animal) Manger() {
    fmt.Printf("%s mange\n", a.Nom)
}

func (a Animal) Dormir() {
    fmt.Printf("%s dort\n", a.Nom)
}

func (a Animal) Presenter() {
    fmt.Printf("Je suis %s, j'ai %d ans\n", a.Nom, a.Age)
}

// Embedding - pas de nom de champ
type Chien struct {
    Animal        // Type embedded
    Race   string
}

func (c Chien) Aboyer() {
    fmt.Printf("%s aboie: Woof!\n", c.Nom) // Accès direct à Nom
}

// Redéfinition de méthode
func (c Chien) Presenter() {
    fmt.Printf("Je suis %s, un %s de %d ans\n", c.Nom, c.Race, c.Age)
}

func main() {
    chien := Chien{
        Animal: Animal{Nom: "Rex", Age: 3},
        Race:   "Labrador",
    }

    // Accès direct aux méthodes et champs de Animal
    chien.Manger()  // Méthode de Animal
    chien.Dormir()  // Méthode de Animal
    chien.Aboyer()  // Méthode de Chien

    // La méthode de Chien "masque" celle de Animal
    chien.Presenter() // Version de Chien

    // Accès explicite à la version de Animal si nécessaire
    chien.Animal.Presenter() // Version de Animal
}
```

**Sortie :**
```
Rex mange
Rex dort
Rex aboie: Woof!
Je suis Rex, un Labrador de 3 ans
Je suis Rex, j'ai 3 ans
```

## Exemple complet : Système d'employés

```go
package main

import "fmt"

// Type de base : Personne
type Personne struct {
    Nom    string
    Age    int
    Email  string
}

func (p Personne) SePresenter() {
    fmt.Printf("Bonjour, je suis %s, %d ans\n", p.Nom, p.Age)
}

func (p Personne) Contacter() {
    fmt.Printf("Contactez-moi à: %s\n", p.Email)
}

// Employé compose Personne
type Employe struct {
    Personne
    Poste    string
    Salaire  float64
    Identifiant string
}

func (e Employe) Travailler() {
    fmt.Printf("%s travaille en tant que %s\n", e.Nom, e.Poste)
}

func (e Employe) AfficherSalaire() {
    fmt.Printf("Salaire de %s: %.2f€\n", e.Nom, e.Salaire)
}

// Redéfinition de la présentation pour les employés
func (e Employe) SePresenter() {
    fmt.Printf("Bonjour, je suis %s, %s dans l'entreprise (ID: %s)\n",
               e.Nom, e.Poste, e.Identifiant)
}

// Manager compose Employé (composition à plusieurs niveaux)
type Manager struct {
    Employe
    Equipe []string
    Budget float64
}

func (m Manager) GererEquipe() {
    fmt.Printf("%s gère une équipe de %d personnes: %v\n",
               m.Nom, len(m.Equipe), m.Equipe)
}

func (m Manager) AllouerBudget(montant float64) {
    if montant <= m.Budget {
        m.Budget -= montant
        fmt.Printf("%s a alloué %.2f€, budget restant: %.2f€\n",
                   m.Nom, montant, m.Budget)
    } else {
        fmt.Printf("Budget insuffisant pour %s\n", m.Nom)
    }
}

// Redéfinition pour les managers
func (m Manager) SePresenter() {
    fmt.Printf("Bonjour, je suis %s, %s et manager (équipe de %d)\n",
               m.Nom, m.Poste, len(m.Equipe))
}

func main() {
    // Création d'une personne
    personne := Personne{
        Nom:   "Alice Martin",
        Age:   25,
        Email: "alice@example.com",
    }

    fmt.Println("=== Personne ===")
    personne.SePresenter()
    personne.Contacter()

    // Création d'un employé
    employe := Employe{
        Personne: Personne{
            Nom:   "Bob Dupont",
            Age:   30,
            Email: "bob@company.com",
        },
        Poste:       "Développeur",
        Salaire:     45000,
        Identifiant: "EMP001",
    }

    fmt.Println("\n=== Employé ===")
    employe.SePresenter()    // Version Employe
    employe.Contacter()      // Héritée de Personne
    employe.Travailler()
    employe.AfficherSalaire()

    // Création d'un manager
    manager := Manager{
        Employe: Employe{
            Personne: Personne{
                Nom:   "Claire Durand",
                Age:   35,
                Email: "claire@company.com",
            },
            Poste:       "Chef de projet",
            Salaire:     65000,
            Identifiant: "MGR001",
        },
        Equipe: []string{"Bob", "David", "Emma"},
        Budget: 100000,
    }

    fmt.Println("\n=== Manager ===")
    manager.SePresenter()     // Version Manager
    manager.Contacter()       // De Personne
    manager.Travailler()      // De Employe
    manager.GererEquipe()     // Spécifique au Manager
    manager.AllouerBudget(25000)

    // Accès aux différents niveaux
    fmt.Println("\n=== Accès multi-niveaux ===")
    fmt.Printf("Nom (direct): %s\n", manager.Nom)
    fmt.Printf("Email (via Personne): %s\n", manager.Personne.Email)
    fmt.Printf("Salaire (via Employe): %.2f€\n", manager.Employe.Salaire)
}
```

## Composition avec interfaces

La vraie puissance de la composition apparaît avec les interfaces :

```go
package main

import "fmt"

// Interfaces
type Lecteur interface {
    Lire() string
}

type Ecrivain interface {
    Ecrire(data string) error
}

// Types de base
type FichierTexte struct {
    nom     string
    contenu string
}

func (f *FichierTexte) Lire() string {
    return f.contenu
}

func (f *FichierTexte) Ecrire(data string) error {
    f.contenu = data
    return nil
}

// Composant de compression
type Compresseur struct {
    active bool
}

func (c *Compresseur) Comprimer(data string) string {
    if !c.active {
        return data
    }
    return fmt.Sprintf("COMPRESSED[%s]", data)
}

func (c *Compresseur) Decompresser(data string) string {
    if !c.active {
        return data
    }
    // Simulation de décompression
    if len(data) > 12 && data[:11] == "COMPRESSED[" && data[len(data)-1] == ']' {
        return data[11 : len(data)-1]
    }
    return data
}

// Composant de chiffrement
type Chiffreur struct {
    active bool
    cle    string
}

func (c *Chiffreur) Chiffrer(data string) string {
    if !c.active {
        return data
    }
    return fmt.Sprintf("ENCRYPTED[%s]", data)
}

func (c *Chiffreur) Dechiffrer(data string) string {
    if !c.active {
        return data
    }
    // Simulation de déchiffrement
    if len(data) > 12 && data[:10] == "ENCRYPTED[" && data[len(data)-1] == ']' {
        return data[10 : len(data)-1]
    }
    return data
}

// Fichier sécurisé qui compose plusieurs fonctionnalités
type FichierSecurise struct {
    FichierTexte        // Embedding du fichier de base
    Compresseur         // Embedding du compresseur
    Chiffreur           // Embedding du chiffreur
}

// Redéfinition des méthodes pour ajouter sécurité
func (fs *FichierSecurise) Lire() string {
    // Lecture avec déchiffrement et décompression
    contenu := fs.FichierTexte.Lire()
    contenu = fs.Chiffreur.Dechiffrer(contenu)
    contenu = fs.Compresseur.Decompresser(contenu)
    return contenu
}

func (fs *FichierSecurise) Ecrire(data string) error {
    // Écriture avec compression et chiffrement
    contenu := fs.Compresseur.Comprimer(data)
    contenu = fs.Chiffreur.Chiffrer(contenu)
    return fs.FichierTexte.Ecrire(contenu)
}

func main() {
    // Fichier simple
    fichierSimple := &FichierTexte{nom: "test.txt"}

    fmt.Println("=== Fichier simple ===")
    fichierSimple.Ecrire("Contenu secret")
    fmt.Printf("Contenu lu: %s\n", fichierSimple.Lire())

    // Fichier sécurisé
    fichierSecurise := &FichierSecurise{
        FichierTexte: FichierTexte{nom: "secure.txt"},
        Compresseur:  Compresseur{active: true},
        Chiffreur:    Chiffreur{active: true, cle: "secret123"},
    }

    fmt.Println("\n=== Fichier sécurisé ===")
    fichierSecurise.Ecrire("Données confidentielles")
    fmt.Printf("Contenu brut: %s\n", fichierSecurise.FichierTexte.Lire())
    fmt.Printf("Contenu déchiffré: %s\n", fichierSecurise.Lire())

    // Utilisation polymorphe
    fmt.Println("\n=== Utilisation polymorphe ===")
    var fichiers []Lecteur = []Lecteur{fichierSimple, fichierSecurise}

    for i, fichier := range fichiers {
        fmt.Printf("Fichier %d: %s\n", i+1, fichier.Lire())
    }
}
```

## Composition vs Héritage : Avantages

### 1. Flexibilité
```go
// Avec la composition, on peut facilement combiner différentes capacités
type RobotVolant struct {
    Moteur
    Ailes
    IA
}

type RobotTerrestre struct {
    Moteur
    Roues
    IA
}

type RobotAquatique struct {
    Moteur
    Helices
    IA
    Etancheite
}
```

### 2. Éviter le problème du diamant
L'héritage multiple peut créer des ambiguïtés. La composition les évite :

```go
// Pas de problème d'ambiguïté avec la composition
type SystemeComplet struct {
    Audio    // a une méthode Volume()
    Video    // a aussi une méthode Volume()
    Reseau
}

func (s SystemeComplet) ReglerVolumeAudio(niveau int) {
    s.Audio.ReglerVolume(niveau)
}

func (s SystemeComplet) ReglerVolumeVideo(niveau int) {
    s.Video.ReglerVolume(niveau)
}
```

## Patterns de composition courants

### 1. Decorator Pattern
```go
package main

import "fmt"

// Interface de base
type Boisson interface {
    Cout() float64
    Description() string
}

// Implémentation de base
type Cafe struct{}

func (c Cafe) Cout() float64 {
    return 2.0
}

func (c Cafe) Description() string {
    return "Café"
}

// Décorateurs
type AvecLait struct {
    Boisson
}

func (a AvecLait) Cout() float64 {
    return a.Boisson.Cout() + 0.5
}

func (a AvecLait) Description() string {
    return a.Boisson.Description() + " + Lait"
}

type AvecSucre struct {
    Boisson
}

func (a AvecSucre) Cout() float64 {
    return a.Boisson.Cout() + 0.2
}

func (a AvecSucre) Description() string {
    return a.Boisson.Description() + " + Sucre"
}

func main() {
    // Composition de décorateurs
    cafe := Cafe{}
    cafeAvecLait := AvecLait{cafe}
    cafeComplet := AvecSucre{cafeAvecLait}

    fmt.Printf("%s - %.2f€\n", cafeComplet.Description(), cafeComplet.Cout())
}
```

### 2. Strategy Pattern
```go
package main

import "fmt"

// Interface pour les stratégies
type StrategieCalcul interface {
    Calculer(a, b float64) float64
}

// Différentes stratégies
type Addition struct{}
func (Addition) Calculer(a, b float64) float64 { return a + b }

type Multiplication struct{}
func (Multiplication) Calculer(a, b float64) float64 { return a * b }

// Contexte qui utilise la composition
type Calculatrice struct {
    strategie StrategieCalcul
}

func (c *Calculatrice) DefinirStrategie(s StrategieCalcul) {
    c.strategie = s
}

func (c *Calculatrice) Executer(a, b float64) float64 {
    return c.strategie.Calculer(a, b)
}

func main() {
    calc := &Calculatrice{}

    calc.DefinirStrategie(Addition{})
    fmt.Printf("5 + 3 = %.2f\n", calc.Executer(5, 3))

    calc.DefinirStrategie(Multiplication{})
    fmt.Printf("5 * 3 = %.2f\n", calc.Executer(5, 3))
}
```

## Exemple pratique : Serveur HTTP modulaire

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// Composants de base
type Logger struct {
    enabled bool
}

func (l *Logger) Log(message string) {
    if l.enabled {
        fmt.Printf("[%s] %s\n", time.Now().Format("15:04:05"), message)
    }
}

type Authenticator struct {
    required bool
    users    map[string]string
}

func (a *Authenticator) IsValid(username, password string) bool {
    if !a.required {
        return true
    }
    validPassword, exists := a.users[username]
    return exists && validPassword == password
}

type RateLimiter struct {
    enabled   bool
    requests  map[string]int
    limit     int
    resetTime time.Time
}

func (r *RateLimiter) Allow(clientIP string) bool {
    if !r.enabled {
        return true
    }

    // Reset simple basé sur le temps
    if time.Now().After(r.resetTime) {
        r.requests = make(map[string]int)
        r.resetTime = time.Now().Add(time.Minute)
    }

    r.requests[clientIP]++
    return r.requests[clientIP] <= r.limit
}

// Serveur composé
type ServeurHTTP struct {
    Logger
    Authenticator
    RateLimiter
    port string
}

func NouveauServeur(port string) *ServeurHTTP {
    return &ServeurHTTP{
        Logger: Logger{enabled: true},
        Authenticator: Authenticator{
            required: true,
            users:    map[string]string{"admin": "secret", "user": "password"},
        },
        RateLimiter: RateLimiter{
            enabled:   true,
            requests:  make(map[string]int),
            limit:     10,
            resetTime: time.Now().Add(time.Minute),
        },
        port: port,
    }
}

func (s *ServeurHTTP) HandleRequest(w http.ResponseWriter, r *http.Request) {
    clientIP := r.RemoteAddr
    s.Log(fmt.Sprintf("Requête de %s vers %s", clientIP, r.URL.Path))

    // Vérification rate limiting
    if !s.RateLimiter.Allow(clientIP) {
        s.Log(fmt.Sprintf("Rate limit dépassé pour %s", clientIP))
        http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
        return
    }

    // Vérification authentification
    username, password, hasAuth := r.BasicAuth()
    if !s.Authenticator.IsValid(username, password) {
        s.Log(fmt.Sprintf("Authentification échouée pour %s", username))
        w.Header().Set("WWW-Authenticate", "Basic realm='Restricted'")
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    s.Log(fmt.Sprintf("Requête autorisée pour %s", username))

    // Traitement de la requête
    switch r.URL.Path {
    case "/":
        fmt.Fprintf(w, "Bienvenue %s!", username)
    case "/status":
        fmt.Fprintf(w, "Serveur actif - %s", time.Now().Format("15:04:05"))
    default:
        http.NotFound(w, r)
    }
}

func (s *ServeurHTTP) Demarrer() {
    s.Log(fmt.Sprintf("Démarrage du serveur sur le port %s", s.port))

    http.HandleFunc("/", s.HandleRequest)

    fmt.Printf("Serveur démarré sur http://localhost%s\n", s.port)
    fmt.Println("Utilisez: curl -u admin:secret http://localhost" + s.port + "/")

    log.Fatal(http.ListenAndServe(s.port, nil))
}

func main() {
    serveur := NouveauServeur(":8080")

    // Configuration optionnelle
    serveur.Logger.enabled = true
    serveur.RateLimiter.limit = 5

    // Démarrage (commenté pour éviter de bloquer l'exemple)
    // serveur.Demarrer()

    fmt.Println("Serveur configuré avec composition !")
}
```

## Bonnes pratiques

1. **Préférez la composition à l'héritage** : Plus flexible et évite les hiérarchies complexes
2. **Utilisez l'embedding pour la simplicité** : Quand vous voulez "hériter" du comportement
3. **Composez des interfaces** : Créez des interfaces spécialisées et composez-les
4. **Gardez les composants indépendants** : Chaque composant doit avoir une responsabilité claire
5. **Documentez les relations** : Expliquez pourquoi certains types sont composés

## Quand utiliser quoi ?

- **Embedding** : Quand vous voulez étendre un type avec des fonctionnalités supplémentaires
- **Composition explicite** : Quand vous voulez contrôler précisément l'accès aux composants
- **Interfaces** : Pour définir des contrats et permettre le polymorphisme
- **Composition multiple** : Pour combiner différentes capacités de manière flexible

## Résumé

La composition en Go offre plusieurs avantages par rapport à l'héritage traditionnel :

- **Flexibilité** : Combinaison libre de fonctionnalités
- **Simplicité** : Pas de hiérarchies complexes
- **Testabilité** : Chaque composant peut être testé indépendamment
- **Maintenabilité** : Modifications localisées sans impact sur l'ensemble

Go encourage la composition pour créer des systèmes modulaires et flexibles. Cette approche, combinée aux interfaces, permet de créer du code robuste et évolutif.

Dans la prochaine section, nous aborderons la gestion des erreurs, un aspect crucial de la programmation Go.

⏭️
