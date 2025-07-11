🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14-1 : SQL avec database/sql

## Introduction

Le package `database/sql` est la bibliothèque standard de Go pour interagir avec les bases de données SQL. Il fournit une interface générique qui fonctionne avec différents drivers de bases de données. C'est l'approche la plus fondamentale et elle vous donnera une compréhension solide des concepts de base.

## Qu'est-ce que database/sql ?

`database/sql` est une **interface** qui définit comment Go doit communiquer avec les bases de données SQL. Il ne contient pas de driver spécifique, mais fournit les outils pour :
- Exécuter des requêtes SQL
- Gérer les connexions
- Traiter les résultats
- Gérer les transactions

## Installation d'un driver

Pour utiliser `database/sql`, vous devez installer un driver spécifique à votre base de données. Voici les plus populaires :

### SQLite (recommandé pour débuter)
```bash
go mod init mon-projet
go get github.com/mattn/go-sqlite3
```

### PostgreSQL
```bash
go get github.com/lib/pq
```

### MySQL
```bash
go get github.com/go-sql-driver/mysql
```

## Premier exemple : Connexion à une base de données

Commençons par un exemple simple avec SQLite :

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/mattn/go-sqlite3"  // Driver SQLite
)

func main() {
    // Ouvrir une connexion à la base de données
    db, err := sql.Open("sqlite3", "exemple.db")
    if err != nil {
        log.Fatal("Erreur d'ouverture de la base de données:", err)
    }
    defer db.Close()  // Fermer la connexion à la fin

    // Tester la connexion
    if err := db.Ping(); err != nil {
        log.Fatal("Impossible de se connecter à la base de données:", err)
    }

    fmt.Println("Connexion à la base de données réussie !")
}
```

### Explications ligne par ligne :

1. `_ "github.com/mattn/go-sqlite3"` : Import du driver (le `_` signifie qu'on utilise seulement son code d'initialisation)
2. `sql.Open("sqlite3", "exemple.db")` : Ouvre une connexion avec le driver "sqlite3" vers le fichier "exemple.db"
3. `defer db.Close()` : Assure que la connexion sera fermée à la fin de la fonction
4. `db.Ping()` : Teste si la connexion fonctionne vraiment

## Création d'une table

Avant d'insérer des données, créons une table :

```go
func creerTable(db *sql.DB) error {
    requete := `
    CREATE TABLE IF NOT EXISTS utilisateurs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nom TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        age INTEGER
    );`

    _, err := db.Exec(requete)
    if err != nil {
        return fmt.Errorf("erreur lors de la création de la table: %v", err)
    }

    fmt.Println("Table 'utilisateurs' créée avec succès")
    return nil
}
```

### Points importants :
- `db.Exec()` : Utilisé pour les requêtes qui ne retournent pas de résultats (CREATE, INSERT, UPDATE, DELETE)
- `IF NOT EXISTS` : Évite les erreurs si la table existe déjà
- Le SQL est écrit dans une chaîne de caractères multiligne avec les backticks

## Opérations CRUD

CRUD signifie **Create** (Créer), **Read** (Lire), **Update** (Mettre à jour), **Delete** (Supprimer). Voyons chaque opération :

### Create - Insérer des données

```go
func ajouterUtilisateur(db *sql.DB, nom, email string, age int) error {
    requete := `INSERT INTO utilisateurs (nom, email, age) VALUES (?, ?, ?)`

    result, err := db.Exec(requete, nom, email, age)
    if err != nil {
        return fmt.Errorf("erreur lors de l'insertion: %v", err)
    }

    // Récupérer l'ID de l'utilisateur créé
    id, err := result.LastInsertId()
    if err != nil {
        return fmt.Errorf("erreur lors de la récupération de l'ID: %v", err)
    }

    fmt.Printf("Utilisateur ajouté avec l'ID: %d\n", id)
    return nil
}
```

**Points clés :**
- `?` : Placeholders pour éviter les injections SQL
- `db.Exec()` : Retourne un `Result` qui contient des informations sur l'opération
- `LastInsertId()` : Récupère l'ID auto-généré

### Read - Lire les données

#### Lire un seul utilisateur :

```go
type Utilisateur struct {
    ID    int
    Nom   string
    Email string
    Age   int
}

func obtenirUtilisateur(db *sql.DB, id int) (*Utilisateur, error) {
    requete := `SELECT id, nom, email, age FROM utilisateurs WHERE id = ?`

    row := db.QueryRow(requete, id)

    var u Utilisateur
    err := row.Scan(&u.ID, &u.Nom, &u.Email, &u.Age)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("utilisateur avec l'ID %d non trouvé", id)
        }
        return nil, fmt.Errorf("erreur lors de la lecture: %v", err)
    }

    return &u, nil
}
```

#### Lire plusieurs utilisateurs :

```go
func obtenirTousUtilisateurs(db *sql.DB) ([]Utilisateur, error) {
    requete := `SELECT id, nom, email, age FROM utilisateurs`

    rows, err := db.Query(requete)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la requête: %v", err)
    }
    defer rows.Close()  // Important : fermer les rows

    var utilisateurs []Utilisateur
    for rows.Next() {
        var u Utilisateur
        err := rows.Scan(&u.ID, &u.Nom, &u.Email, &u.Age)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du scan: %v", err)
        }
        utilisateurs = append(utilisateurs, u)
    }

    // Vérifier s'il y a eu une erreur pendant l'itération
    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("erreur lors de l'itération: %v", err)
    }

    return utilisateurs, nil
}
```

**Différences importantes :**
- `QueryRow()` : Pour récupérer une seule ligne
- `Query()` : Pour récupérer plusieurs lignes
- `Scan()` : Copie les valeurs des colonnes dans des variables Go
- `defer rows.Close()` : Toujours fermer les rows après une Query()

### Update - Mettre à jour

```go
func mettreAJourUtilisateur(db *sql.DB, id int, nom, email string, age int) error {
    requete := `UPDATE utilisateurs SET nom = ?, email = ?, age = ? WHERE id = ?`

    result, err := db.Exec(requete, nom, email, age, id)
    if err != nil {
        return fmt.Errorf("erreur lors de la mise à jour: %v", err)
    }

    // Vérifier si une ligne a été modifiée
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("erreur lors de la vérification: %v", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("aucun utilisateur trouvé avec l'ID %d", id)
    }

    fmt.Printf("Utilisateur %d mis à jour avec succès\n", id)
    return nil
}
```

### Delete - Supprimer

```go
func supprimerUtilisateur(db *sql.DB, id int) error {
    requete := `DELETE FROM utilisateurs WHERE id = ?`

    result, err := db.Exec(requete, id)
    if err != nil {
        return fmt.Errorf("erreur lors de la suppression: %v", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("erreur lors de la vérification: %v", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("aucun utilisateur trouvé avec l'ID %d", id)
    }

    fmt.Printf("Utilisateur %d supprimé avec succès\n", id)
    return nil
}
```

## Gestion des erreurs communes

### sql.ErrNoRows
Cette erreur survient quand une requête `QueryRow()` ne retourne aucun résultat :

```go
user, err := obtenirUtilisateur(db, 999)
if err != nil {
    if err == sql.ErrNoRows {
        fmt.Println("Utilisateur non trouvé")
    } else {
        fmt.Printf("Erreur: %v\n", err)
    }
}
```

### Gestion des valeurs NULL

Si votre base de données peut contenir des valeurs NULL, utilisez les types sql.NullString, sql.NullInt64, etc. :

```go
type Utilisateur struct {
    ID    int
    Nom   string
    Email string
    Age   sql.NullInt64  // Peut être NULL
}

func obtenirUtilisateurAvecNull(db *sql.DB, id int) (*Utilisateur, error) {
    requete := `SELECT id, nom, email, age FROM utilisateurs WHERE id = ?`

    row := db.QueryRow(requete, id)

    var u Utilisateur
    err := row.Scan(&u.ID, &u.Nom, &u.Email, &u.Age)
    if err != nil {
        return nil, err
    }

    // Vérifier si l'âge est NULL
    if u.Age.Valid {
        fmt.Printf("Âge: %d\n", u.Age.Int64)
    } else {
        fmt.Println("Âge: non spécifié")
    }

    return &u, nil
}
```

## Exemple complet

Voici un exemple complet qui combine tout ce que nous avons vu :

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/mattn/go-sqlite3"
)

type Utilisateur struct {
    ID    int
    Nom   string
    Email string
    Age   int
}

func main() {
    // Connexion à la base de données
    db, err := sql.Open("sqlite3", "exemple.db")
    if err != nil {
        log.Fatal("Erreur d'ouverture:", err)
    }
    defer db.Close()

    // Créer la table
    if err := creerTable(db); err != nil {
        log.Fatal("Erreur création table:", err)
    }

    // Ajouter des utilisateurs
    if err := ajouterUtilisateur(db, "Alice", "alice@email.com", 25); err != nil {
        log.Fatal("Erreur ajout utilisateur:", err)
    }

    if err := ajouterUtilisateur(db, "Bob", "bob@email.com", 30); err != nil {
        log.Fatal("Erreur ajout utilisateur:", err)
    }

    // Lire tous les utilisateurs
    utilisateurs, err := obtenirTousUtilisateurs(db)
    if err != nil {
        log.Fatal("Erreur lecture utilisateurs:", err)
    }

    fmt.Println("Tous les utilisateurs:")
    for _, u := range utilisateurs {
        fmt.Printf("ID: %d, Nom: %s, Email: %s, Age: %d\n",
            u.ID, u.Nom, u.Email, u.Age)
    }

    // Mettre à jour un utilisateur
    if err := mettreAJourUtilisateur(db, 1, "Alice Dupont", "alice.dupont@email.com", 26); err != nil {
        log.Fatal("Erreur mise à jour:", err)
    }

    // Lire un utilisateur spécifique
    user, err := obtenirUtilisateur(db, 1)
    if err != nil {
        log.Fatal("Erreur lecture utilisateur:", err)
    }

    fmt.Printf("Utilisateur mis à jour: %+v\n", user)
}

// Fonctions précédemment définies (creerTable, ajouterUtilisateur, etc.)
```

## Bonnes pratiques

### 1. Toujours fermer les connexions
```go
defer db.Close()
defer rows.Close()
```

### 2. Utiliser des requêtes préparées
```go
// Éviter cela (injection SQL possible)
requete := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", nom)

// Préférer cela
requete := "SELECT * FROM users WHERE name = ?"
rows, err := db.Query(requete, nom)
```

### 3. Gérer les erreurs appropriément
```go
if err != nil {
    if err == sql.ErrNoRows {
        // Cas spécifique : pas de résultat
    } else {
        // Autres erreurs
    }
}
```

### 4. Utiliser des transactions pour les opérations multiples
```go
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback()

// Plusieurs opérations...

return tx.Commit()
```

## Exercices pratiques

### Exercice 1 : Carnet d'adresses
Créez une application qui gère un carnet d'adresses avec les fonctionnalités suivantes :
- Ajouter un contact (nom, téléphone, email)
- Rechercher un contact par nom
- Lister tous les contacts
- Supprimer un contact

### Exercice 2 : Système de blog simple
Créez un système de blog avec :
- Une table `articles` (id, titre, contenu, date_creation)
- Fonctions pour créer, lire, mettre à jour et supprimer des articles
- Fonction pour lister les articles par date

## Résumé

Dans cette section, vous avez appris :
- ✅ Comment utiliser le package `database/sql`
- ✅ Les opérations CRUD de base
- ✅ La gestion des erreurs communes
- ✅ Les bonnes pratiques de sécurité
- ✅ Comment structurer le code avec des bases de données

La prochaine section vous montrera comment utiliser un ORM (GORM) pour simplifier ces opérations, mais maîtriser `database/sql` vous donne une base solide pour comprendre ce qui se passe sous le capot.

## Ressources supplémentaires

- [Documentation officielle database/sql](https://pkg.go.dev/database/sql)
- [Guide des drivers Go](https://github.com/golang/go/wiki/SQLDrivers)
- [Tutoriel SQLite](https://www.sqlite.org/lang.html)

⏭️

# Solutions des exercices pratiques - Base de données Go

## Exercice 1 : Carnet d'adresses

### Structure du projet

```
carnet-adresses/
├── main.go
├── models.go
├── database.go
└── go.mod
```

### models.go

```go
package main

import "fmt"

// Contact représente un contact dans le carnet d'adresses
type Contact struct {
    ID        int
    Nom       string
    Telephone string
    Email     string
}

// String retourne une représentation string du contact
func (c Contact) String() string {
    return fmt.Sprintf("ID: %d | Nom: %s | Tél: %s | Email: %s",
        c.ID, c.Nom, c.Telephone, c.Email)
}

// Valider vérifie que les champs obligatoires sont remplis
func (c Contact) Valider() error {
    if c.Nom == "" {
        return fmt.Errorf("le nom est obligatoire")
    }
    if c.Telephone == "" && c.Email == "" {
        return fmt.Errorf("au moins un numéro de téléphone ou un email est requis")
    }
    return nil
}
```

### database.go

```go
package main

import (
    "database/sql"
    "fmt"
    "strings"

    _ "github.com/mattn/go-sqlite3"
)

// CarnetAdresses gère la base de données des contacts
type CarnetAdresses struct {
    db *sql.DB
}

// NouveauCarnetAdresses crée une nouvelle instance du carnet d'adresses
func NouveauCarnetAdresses(fichierDB string) (*CarnetAdresses, error) {
    db, err := sql.Open("sqlite3", fichierDB)
    if err != nil {
        return nil, fmt.Errorf("erreur d'ouverture de la base de données: %v", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("impossible de se connecter à la base de données: %v", err)
    }

    carnet := &CarnetAdresses{db: db}

    if err := carnet.creerTable(); err != nil {
        return nil, fmt.Errorf("erreur lors de la création de la table: %v", err)
    }

    return carnet, nil
}

// Fermer ferme la connexion à la base de données
func (ca *CarnetAdresses) Fermer() error {
    return ca.db.Close()
}

// creerTable crée la table des contacts si elle n'existe pas
func (ca *CarnetAdresses) creerTable() error {
    requete := `
    CREATE TABLE IF NOT EXISTS contacts (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nom TEXT NOT NULL,
        telephone TEXT,
        email TEXT,
        UNIQUE(nom, telephone, email)
    );`

    _, err := ca.db.Exec(requete)
    return err
}

// AjouterContact ajoute un nouveau contact
func (ca *CarnetAdresses) AjouterContact(contact Contact) (*Contact, error) {
    // Valider le contact
    if err := contact.Valider(); err != nil {
        return nil, err
    }

    requete := `INSERT INTO contacts (nom, telephone, email) VALUES (?, ?, ?)`

    result, err := ca.db.Exec(requete, contact.Nom, contact.Telephone, contact.Email)
    if err != nil {
        // Vérifier si c'est une erreur de doublon
        if strings.Contains(err.Error(), "UNIQUE constraint failed") {
            return nil, fmt.Errorf("ce contact existe déjà")
        }
        return nil, fmt.Errorf("erreur lors de l'ajout du contact: %v", err)
    }

    id, err := result.LastInsertId()
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération de l'ID: %v", err)
    }

    contact.ID = int(id)
    return &contact, nil
}

// RechercherParNom recherche des contacts par nom (recherche partielle)
func (ca *CarnetAdresses) RechercherParNom(nom string) ([]Contact, error) {
    requete := `SELECT id, nom, telephone, email FROM contacts WHERE nom LIKE ? ORDER BY nom`

    rows, err := ca.db.Query(requete, "%"+nom+"%")
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la recherche: %v", err)
    }
    defer rows.Close()

    var contacts []Contact
    for rows.Next() {
        var c Contact
        err := rows.Scan(&c.ID, &c.Nom, &c.Telephone, &c.Email)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du scan: %v", err)
        }
        contacts = append(contacts, c)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("erreur lors de l'itération: %v", err)
    }

    return contacts, nil
}

// ObtenirTousLesContacts retourne tous les contacts
func (ca *CarnetAdresses) ObtenirTousLesContacts() ([]Contact, error) {
    requete := `SELECT id, nom, telephone, email FROM contacts ORDER BY nom`

    rows, err := ca.db.Query(requete)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération: %v", err)
    }
    defer rows.Close()

    var contacts []Contact
    for rows.Next() {
        var c Contact
        err := rows.Scan(&c.ID, &c.Nom, &c.Telephone, &c.Email)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du scan: %v", err)
        }
        contacts = append(contacts, c)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("erreur lors de l'itération: %v", err)
    }

    return contacts, nil
}

// ObtenirContactParID retourne un contact par son ID
func (ca *CarnetAdresses) ObtenirContactParID(id int) (*Contact, error) {
    requete := `SELECT id, nom, telephone, email FROM contacts WHERE id = ?`

    row := ca.db.QueryRow(requete, id)

    var c Contact
    err := row.Scan(&c.ID, &c.Nom, &c.Telephone, &c.Email)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("contact avec l'ID %d non trouvé", id)
        }
        return nil, fmt.Errorf("erreur lors de la lecture: %v", err)
    }

    return &c, nil
}

// SupprimerContact supprime un contact par son ID
func (ca *CarnetAdresses) SupprimerContact(id int) error {
    requete := `DELETE FROM contacts WHERE id = ?`

    result, err := ca.db.Exec(requete, id)
    if err != nil {
        return fmt.Errorf("erreur lors de la suppression: %v", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("erreur lors de la vérification: %v", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("aucun contact trouvé avec l'ID %d", id)
    }

    return nil
}

// CompterContacts retourne le nombre total de contacts
func (ca *CarnetAdresses) CompterContacts() (int, error) {
    requete := `SELECT COUNT(*) FROM contacts`

    var count int
    err := ca.db.QueryRow(requete).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("erreur lors du comptage: %v", err)
    }

    return count, nil
}
```

### main.go

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
    "strconv"
    "strings"
)

func main() {
    // Créer le carnet d'adresses
    carnet, err := NouveauCarnetAdresses("carnet.db")
    if err != nil {
        log.Fatal("Erreur lors de la création du carnet:", err)
    }
    defer carnet.Fermer()

    scanner := bufio.NewScanner(os.Stdin)

    fmt.Println("=== CARNET D'ADRESSES ===")
    fmt.Println("Tapez 'aide' pour voir les commandes disponibles")

    for {
        fmt.Print("\n> ")
        if !scanner.Scan() {
            break
        }

        commande := strings.TrimSpace(scanner.Text())
        if commande == "" {
            continue
        }

        switch commande {
        case "aide":
            afficherAide()
        case "ajouter":
            ajouterContact(carnet, scanner)
        case "rechercher":
            rechercherContact(carnet, scanner)
        case "lister":
            listerContacts(carnet)
        case "supprimer":
            supprimerContact(carnet, scanner)
        case "compter":
            compterContacts(carnet)
        case "quitter":
            fmt.Println("Au revoir!")
            return
        default:
            fmt.Println("Commande inconnue. Tapez 'aide' pour voir les commandes disponibles.")
        }
    }
}

func afficherAide() {
    fmt.Println("\nCommandes disponibles:")
    fmt.Println("  ajouter    - Ajouter un nouveau contact")
    fmt.Println("  rechercher - Rechercher un contact par nom")
    fmt.Println("  lister     - Lister tous les contacts")
    fmt.Println("  supprimer  - Supprimer un contact")
    fmt.Println("  compter    - Compter le nombre de contacts")
    fmt.Println("  aide       - Afficher cette aide")
    fmt.Println("  quitter    - Quitter l'application")
}

func ajouterContact(carnet *CarnetAdresses, scanner *bufio.Scanner) {
    fmt.Print("Nom: ")
    scanner.Scan()
    nom := strings.TrimSpace(scanner.Text())

    fmt.Print("Téléphone: ")
    scanner.Scan()
    telephone := strings.TrimSpace(scanner.Text())

    fmt.Print("Email: ")
    scanner.Scan()
    email := strings.TrimSpace(scanner.Text())

    contact := Contact{
        Nom:       nom,
        Telephone: telephone,
        Email:     email,
    }

    contactAjoute, err := carnet.AjouterContact(contact)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Contact ajouté avec succès!\n%s\n", contactAjoute)
}

func rechercherContact(carnet *CarnetAdresses, scanner *bufio.Scanner) {
    fmt.Print("Nom à rechercher: ")
    scanner.Scan()
    nom := strings.TrimSpace(scanner.Text())

    contacts, err := carnet.RechercherParNom(nom)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    if len(contacts) == 0 {
        fmt.Println("Aucun contact trouvé.")
        return
    }

    fmt.Printf("\n%d contact(s) trouvé(s):\n", len(contacts))
    for _, contact := range contacts {
        fmt.Printf("  %s\n", contact)
    }
}

func listerContacts(carnet *CarnetAdresses) {
    contacts, err := carnet.ObtenirTousLesContacts()
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    if len(contacts) == 0 {
        fmt.Println("Aucun contact dans le carnet.")
        return
    }

    fmt.Printf("\nTous les contacts (%d):\n", len(contacts))
    for _, contact := range contacts {
        fmt.Printf("  %s\n", contact)
    }
}

func supprimerContact(carnet *CarnetAdresses, scanner *bufio.Scanner) {
    fmt.Print("ID du contact à supprimer: ")
    scanner.Scan()
    idStr := strings.TrimSpace(scanner.Text())

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("ID invalide.")
        return
    }

    // Afficher le contact avant suppression
    contact, err := carnet.ObtenirContactParID(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Contact à supprimer: %s\n", contact)
    fmt.Print("Êtes-vous sûr(e) ? (oui/non): ")
    scanner.Scan()
    confirmation := strings.ToLower(strings.TrimSpace(scanner.Text()))

    if confirmation != "oui" && confirmation != "o" {
        fmt.Println("Suppression annulée.")
        return
    }

    err = carnet.SupprimerContact(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Println("Contact supprimé avec succès!")
}

func compterContacts(carnet *CarnetAdresses) {
    count, err := carnet.CompterContacts()
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Nombre total de contacts: %d\n", count)
}
```

## Exercice 2 : Système de blog simple

### Structure du projet

```
blog-simple/
├── main.go
├── models.go
├── database.go
└── go.mod
```

### models.go

```go
package main

import (
    "fmt"
    "time"
)

// Article représente un article de blog
type Article struct {
    ID            int
    Titre         string
    Contenu       string
    DateCreation  time.Time
}

// String retourne une représentation string de l'article
func (a Article) String() string {
    return fmt.Sprintf("ID: %d | Titre: %s | Date: %s",
        a.ID, a.Titre, a.DateCreation.Format("2006-01-02 15:04:05"))
}

// StringDetaille retourne une représentation détaillée de l'article
func (a Article) StringDetaille() string {
    return fmt.Sprintf(`
=== Article #%d ===
Titre: %s
Date: %s
Contenu:
%s
================`,
        a.ID, a.Titre, a.DateCreation.Format("2006-01-02 15:04:05"), a.Contenu)
}

// Valider vérifie que les champs obligatoires sont remplis
func (a Article) Valider() error {
    if a.Titre == "" {
        return fmt.Errorf("le titre est obligatoire")
    }
    if a.Contenu == "" {
        return fmt.Errorf("le contenu est obligatoire")
    }
    return nil
}

// Extrait retourne les premiers n caractères du contenu
func (a Article) Extrait(n int) string {
    if len(a.Contenu) <= n {
        return a.Contenu
    }
    return a.Contenu[:n] + "..."
}
```

### database.go

```go
package main

import (
    "database/sql"
    "fmt"
    "time"

    _ "github.com/mattn/go-sqlite3"
)

// Blog gère la base de données des articles
type Blog struct {
    db *sql.DB
}

// NouveauBlog crée une nouvelle instance du blog
func NouveauBlog(fichierDB string) (*Blog, error) {
    db, err := sql.Open("sqlite3", fichierDB)
    if err != nil {
        return nil, fmt.Errorf("erreur d'ouverture de la base de données: %v", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("impossible de se connecter à la base de données: %v", err)
    }

    blog := &Blog{db: db}

    if err := blog.creerTable(); err != nil {
        return nil, fmt.Errorf("erreur lors de la création de la table: %v", err)
    }

    return blog, nil
}

// Fermer ferme la connexion à la base de données
func (b *Blog) Fermer() error {
    return b.db.Close()
}

// creerTable crée la table des articles si elle n'existe pas
func (b *Blog) creerTable() error {
    requete := `
    CREATE TABLE IF NOT EXISTS articles (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        titre TEXT NOT NULL,
        contenu TEXT NOT NULL,
        date_creation DATETIME DEFAULT CURRENT_TIMESTAMP
    );`

    _, err := b.db.Exec(requete)
    return err
}

// CreerArticle ajoute un nouvel article
func (b *Blog) CreerArticle(article Article) (*Article, error) {
    // Valider l'article
    if err := article.Valider(); err != nil {
        return nil, err
    }

    // Définir la date de création si elle n'est pas spécifiée
    if article.DateCreation.IsZero() {
        article.DateCreation = time.Now()
    }

    requete := `INSERT INTO articles (titre, contenu, date_creation) VALUES (?, ?, ?)`

    result, err := b.db.Exec(requete, article.Titre, article.Contenu, article.DateCreation)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la création de l'article: %v", err)
    }

    id, err := result.LastInsertId()
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération de l'ID: %v", err)
    }

    article.ID = int(id)
    return &article, nil
}

// LireArticle lit un article par son ID
func (b *Blog) LireArticle(id int) (*Article, error) {
    requete := `SELECT id, titre, contenu, date_creation FROM articles WHERE id = ?`

    row := b.db.QueryRow(requete, id)

    var a Article
    var dateStr string
    err := row.Scan(&a.ID, &a.Titre, &a.Contenu, &dateStr)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("article avec l'ID %d non trouvé", id)
        }
        return nil, fmt.Errorf("erreur lors de la lecture: %v", err)
    }

    // Parser la date
    a.DateCreation, err = time.Parse("2006-01-02 15:04:05", dateStr)
    if err != nil {
        return nil, fmt.Errorf("erreur lors du parsing de la date: %v", err)
    }

    return &a, nil
}

// MettreAJourArticle met à jour un article existant
func (b *Blog) MettreAJourArticle(id int, titre, contenu string) error {
    article := Article{Titre: titre, Contenu: contenu}
    if err := article.Valider(); err != nil {
        return err
    }

    requete := `UPDATE articles SET titre = ?, contenu = ? WHERE id = ?`

    result, err := b.db.Exec(requete, titre, contenu, id)
    if err != nil {
        return fmt.Errorf("erreur lors de la mise à jour: %v", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("erreur lors de la vérification: %v", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("aucun article trouvé avec l'ID %d", id)
    }

    return nil
}

// SupprimerArticle supprime un article par son ID
func (b *Blog) SupprimerArticle(id int) error {
    requete := `DELETE FROM articles WHERE id = ?`

    result, err := b.db.Exec(requete, id)
    if err != nil {
        return fmt.Errorf("erreur lors de la suppression: %v", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("erreur lors de la vérification: %v", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("aucun article trouvé avec l'ID %d", id)
    }

    return nil
}

// ListerArticlesParDate liste tous les articles triés par date (du plus récent au plus ancien)
func (b *Blog) ListerArticlesParDate(limite int) ([]Article, error) {
    requete := `SELECT id, titre, contenu, date_creation FROM articles ORDER BY date_creation DESC`

    if limite > 0 {
        requete += fmt.Sprintf(" LIMIT %d", limite)
    }

    rows, err := b.db.Query(requete)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération: %v", err)
    }
    defer rows.Close()

    return b.scanArticles(rows)
}

// ListerTousLesArticles liste tous les articles
func (b *Blog) ListerTousLesArticles() ([]Article, error) {
    requete := `SELECT id, titre, contenu, date_creation FROM articles ORDER BY date_creation DESC`

    rows, err := b.db.Query(requete)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération: %v", err)
    }
    defer rows.Close()

    return b.scanArticles(rows)
}

// RechercherArticles recherche des articles par titre ou contenu
func (b *Blog) RechercherArticles(terme string) ([]Article, error) {
    requete := `SELECT id, titre, contenu, date_creation FROM articles
                WHERE titre LIKE ? OR contenu LIKE ?
                ORDER BY date_creation DESC`

    rows, err := b.db.Query(requete, "%"+terme+"%", "%"+terme+"%")
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la recherche: %v", err)
    }
    defer rows.Close()

    return b.scanArticles(rows)
}

// scanArticles aide à scanner les résultats de requête
func (b *Blog) scanArticles(rows *sql.Rows) ([]Article, error) {
    var articles []Article
    for rows.Next() {
        var a Article
        var dateStr string
        err := rows.Scan(&a.ID, &a.Titre, &a.Contenu, &dateStr)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du scan: %v", err)
        }

        // Parser la date
        a.DateCreation, err = time.Parse("2006-01-02 15:04:05", dateStr)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du parsing de la date: %v", err)
        }

        articles = append(articles, a)
    }

    if err := rows.Err(); err != nil {
        return nil, fmt.Errorf("erreur lors de l'itération: %v", err)
    }

    return articles, nil
}

// CompterArticles retourne le nombre total d'articles
func (b *Blog) CompterArticles() (int, error) {
    requete := `SELECT COUNT(*) FROM articles`

    var count int
    err := b.db.QueryRow(requete).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("erreur lors du comptage: %v", err)
    }

    return count, nil
}

// StatistiquesParMois retourne le nombre d'articles par mois
func (b *Blog) StatistiquesParMois() (map[string]int, error) {
    requete := `SELECT strftime('%Y-%m', date_creation) as mois, COUNT(*) as nb_articles
                FROM articles
                GROUP BY mois
                ORDER BY mois DESC`

    rows, err := b.db.Query(requete)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la récupération des statistiques: %v", err)
    }
    defer rows.Close()

    stats := make(map[string]int)
    for rows.Next() {
        var mois string
        var count int
        err := rows.Scan(&mois, &count)
        if err != nil {
            return nil, fmt.Errorf("erreur lors du scan: %v", err)
        }
        stats[mois] = count
    }

    return stats, nil
}
```

### main.go

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
    "strconv"
    "strings"
)

func main() {
    // Créer le blog
    blog, err := NouveauBlog("blog.db")
    if err != nil {
        log.Fatal("Erreur lors de la création du blog:", err)
    }
    defer blog.Fermer()

    scanner := bufio.NewScanner(os.Stdin)

    fmt.Println("=== BLOG SIMPLE ===")
    fmt.Println("Tapez 'aide' pour voir les commandes disponibles")

    for {
        fmt.Print("\n> ")
        if !scanner.Scan() {
            break
        }

        commande := strings.TrimSpace(scanner.Text())
        if commande == "" {
            continue
        }

        switch commande {
        case "aide":
            afficherAide()
        case "creer":
            creerArticle(blog, scanner)
        case "lire":
            lireArticle(blog, scanner)
        case "modifier":
            modifierArticle(blog, scanner)
        case "supprimer":
            supprimerArticle(blog, scanner)
        case "lister":
            listerArticles(blog, scanner)
        case "rechercher":
            rechercherArticles(blog, scanner)
        case "stats":
            afficherStatistiques(blog)
        case "quitter":
            fmt.Println("Au revoir!")
            return
        default:
            fmt.Println("Commande inconnue. Tapez 'aide' pour voir les commandes disponibles.")
        }
    }
}

func afficherAide() {
    fmt.Println("\nCommandes disponibles:")
    fmt.Println("  creer      - Créer un nouvel article")
    fmt.Println("  lire       - Lire un article par ID")
    fmt.Println("  modifier   - Modifier un article existant")
    fmt.Println("  supprimer  - Supprimer un article")
    fmt.Println("  lister     - Lister les articles par date")
    fmt.Println("  rechercher - Rechercher des articles")
    fmt.Println("  stats      - Afficher les statistiques")
    fmt.Println("  aide       - Afficher cette aide")
    fmt.Println("  quitter    - Quitter l'application")
}

func creerArticle(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("Titre: ")
    scanner.Scan()
    titre := strings.TrimSpace(scanner.Text())

    fmt.Println("Contenu (tapez 'FIN' sur une ligne vide pour terminer):")
    var contenu strings.Builder
    for {
        scanner.Scan()
        ligne := scanner.Text()
        if ligne == "FIN" {
            break
        }
        contenu.WriteString(ligne)
        contenu.WriteString("\n")
    }

    article := Article{
        Titre:   titre,
        Contenu: strings.TrimSpace(contenu.String()),
    }

    articleCree, err := blog.CreerArticle(article)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Article créé avec succès!\n%s\n", articleCree)
}

func lireArticle(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("ID de l'article: ")
    scanner.Scan()
    idStr := strings.TrimSpace(scanner.Text())

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("ID invalide.")
        return
    }

    article, err := blog.LireArticle(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Println(article.StringDetaille())
}

func modifierArticle(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("ID de l'article à modifier: ")
    scanner.Scan()
    idStr := strings.TrimSpace(scanner.Text())

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("ID invalide.")
        return
    }

    // Afficher l'article actuel
    article, err := blog.LireArticle(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Article actuel:\n%s\n", article.StringDetaille())

    fmt.Printf("Nouveau titre (actuel: %s): ", article.Titre)
    scanner.Scan()
    nouveauTitre := strings.TrimSpace(scanner.Text())
    if nouveauTitre == "" {
        nouveauTitre = article.Titre
    }

    fmt.Println("Nouveau contenu (tapez 'FIN' sur une ligne vide pour terminer, 'GARDER' pour conserver l'actuel):")
    var contenu strings.Builder
    for {
        scanner.Scan()
        ligne := scanner.Text()
        if ligne == "FIN" {
            break
        }
        if ligne == "GARDER" {
            contenu.WriteString(article.Contenu)
            break
        }
        contenu.WriteString(ligne)
        contenu.WriteString("\n")
    }

    nouveauContenu := strings.TrimSpace(contenu.String())
    if nouveauContenu == "" {
        nouveauContenu = article.Contenu
    }

    err = blog.MettreAJourArticle(id, nouveauTitre, nouveauContenu)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Println("Article modifié avec succès!")
}

func supprimerArticle(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("ID de l'article à supprimer: ")
    scanner.Scan()
    idStr := strings.TrimSpace(scanner.Text())

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("ID invalide.")
        return
    }

    // Afficher l'article avant suppression
    article, err := blog.LireArticle(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Article à supprimer:\n%s\n", article.StringDetaille())
    fmt.Print("Êtes-vous sûr(e) de vouloir supprimer cet article ? (oui/non): ")
    scanner.Scan()
    confirmation := strings.ToLower(strings.TrimSpace(scanner.Text()))

    if confirmation != "oui" && confirmation != "o" {
        fmt.Println("Suppression annulée.")
        return
    }

    err = blog.SupprimerArticle(id)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Println("Article supprimé avec succès!")
}

func listerArticles(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("Nombre d'articles à afficher (0 pour tous): ")
    scanner.Scan()
    limiteStr := strings.TrimSpace(scanner.Text())

    limite := 0
    if limiteStr != "" && limiteStr != "0" {
        var err error
        limite, err = strconv.Atoi(limiteStr)
        if err != nil {
            fmt.Println("Nombre invalide, affichage de tous les articles.")
            limite = 0
        }
    }

    articles, err := blog.ListerArticlesParDate(limite)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    if len(articles) == 0 {
        fmt.Println("Aucun article trouvé.")
        return
    }

    fmt.Printf("\n=== Articles (%d) ===\n", len(articles))
    for _, article := range articles {
        fmt.Printf("\n%s\n", article)
        fmt.Printf("Extrait: %s\n", article.Extrait(100))
        fmt.Println(strings.Repeat("-", 50))
    }

    fmt.Print("\nVoulez-vous lire un article en détail ? (ID ou 'non'): ")
    scanner.Scan()
    choix := strings.TrimSpace(scanner.Text())

    if choix != "non" && choix != "n" && choix != "" {
        id, err := strconv.Atoi(choix)
        if err != nil {
            fmt.Println("ID invalide.")
            return
        }

        article, err := blog.LireArticle(id)
        if err != nil {
            fmt.Printf("Erreur: %v\n", err)
            return
        }

        fmt.Println(article.StringDetaille())
    }
}

func rechercherArticles(blog *Blog, scanner *bufio.Scanner) {
    fmt.Print("Terme à rechercher: ")
    scanner.Scan()
    terme := strings.TrimSpace(scanner.Text())

    if terme == "" {
        fmt.Println("Terme de recherche vide.")
        return
    }

    articles, err := blog.RechercherArticles(terme)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    if len(articles) == 0 {
        fmt.Printf("Aucun article trouvé pour '%s'.\n", terme)
        return
    }

    fmt.Printf("\n=== Résultats de recherche pour '%s' (%d) ===\n", terme, len(articles))
    for _, article := range articles {
        fmt.Printf("\n%s\n", article)
        fmt.Printf("Extrait: %s\n", article.Extrait(100))
        fmt.Println(strings.Repeat("-", 50))
    }

    fmt.Print("\nVoulez-vous lire un article en détail ? (ID ou 'non'): ")
    scanner.Scan()
    choix := strings.TrimSpace(scanner.Text())

    if choix != "non" && choix != "n" && choix != "" {
        id, err := strconv.Atoi(choix)
        if err != nil {
            fmt.Println("ID invalide.")
            return
        }

        article, err := blog.LireArticle(id)
        if err != nil {
            fmt.Printf("Erreur: %v\n", err)
            return
        }

        fmt.Println(article.StringDetaille())
    }
}

func afficherStatistiques(blog *Blog) {
    // Nombre total d'articles
    total, err := blog.CompterArticles()
    if err != nil {
        fmt.Printf("Erreur lors du comptage: %v\n", err)
        return
    }

    fmt.Printf("\n=== Statistiques du blog ===\n")
    fmt.Printf("Nombre total d'articles: %d\n", total)

    if total == 0 {
        fmt.Println("Aucun article dans le blog.")
        return
    }

    // Statistiques par mois
    stats, err := blog.StatistiquesParMois()
    if err != nil {
        fmt.Printf("Erreur lors de la récupération des statistiques: %v\n", err)
        return
    }

    fmt.Println("\nArticles par mois:")
    for mois, count := range stats {
        fmt.Printf("  %s: %d article(s)\n", mois, count)
    }

    // Article le plus récent
    articles, err := blog.ListerArticlesParDate(1)
    if err != nil {
        fmt.Printf("Erreur lors de la récupération du dernier article: %v\n", err)
        return
    }

    if len(articles) > 0 {
        fmt.Printf("\nDernier article publié:\n")
        fmt.Printf("  %s\n", articles[0])
    }

    // Tous les articles (aperçu)
    tousArticles, err := blog.ListerTousLesArticles()
    if err != nil {
        fmt.Printf("Erreur lors de la récupération des articles: %v\n", err)
        return
    }

    if len(tousArticles) > 0 {
        fmt.Printf("\nAperçu de tous les articles:\n")
        for i, article := range tousArticles {
            if i >= 5 { // Limiter à 5 articles pour l'aperçu
                fmt.Printf("  ... et %d autre(s)\n", len(tousArticles)-i)
                break
            }
            fmt.Printf("  %s\n", article)
        }
    }

    fmt.Println(strings.Repeat("=", 30))
}
```

## Fichier go.mod pour les deux projets

```go
module blog-simple

go 1.21

require github.com/mattn/go-sqlite3 v1.14.17
```

## Instructions d'utilisation

### Pour le carnet d'adresses :

1. **Installation :**
   ```bash
   mkdir carnet-adresses
   cd carnet-adresses
   go mod init carnet-adresses
   go get github.com/mattn/go-sqlite3
   ```

2. **Créer les fichiers** models.go, database.go et main.go avec le code fourni

3. **Exécution :**
   ```bash
   go run .
   ```

4. **Commandes disponibles :**
   - `ajouter` : Ajouter un nouveau contact
   - `rechercher` : Rechercher par nom
   - `lister` : Voir tous les contacts
   - `supprimer` : Supprimer un contact
   - `compter` : Nombre total de contacts

### Pour le blog :

1. **Installation :**
   ```bash
   mkdir blog-simple
   cd blog-simple
   go mod init blog-simple
   go get github.com/mattn/go-sqlite3
   ```

2. **Créer les fichiers** models.go, database.go et main.go avec le code fourni

3. **Exécution :**
   ```bash
   go run .
   ```

4. **Commandes disponibles :**
   - `creer` : Créer un nouvel article
   - `lire` : Lire un article par ID
   - `modifier` : Modifier un article existant
   - `supprimer` : Supprimer un article
   - `lister` : Lister les articles par date
   - `rechercher` : Rechercher dans les articles
   - `stats` : Afficher les statistiques

## Fonctionnalités bonus implémentées

### Carnet d'adresses :
- ✅ Validation des données d'entrée
- ✅ Recherche partielle par nom (LIKE)
- ✅ Confirmation avant suppression
- ✅ Comptage des contacts
- ✅ Gestion des doublons
- ✅ Interface utilisateur conviviale

### Blog :
- ✅ Gestion complète des articles (CRUD)
- ✅ Tri par date automatique
- ✅ Recherche dans titre et contenu
- ✅ Statistiques par mois
- ✅ Extraits d'articles
- ✅ Confirmation avant suppression
- ✅ Modification avec conservation optionnelle
- ✅ Interface interactive

## Points d'apprentissage

Ces exercices vous ont permis de pratiquer :

1. **Architecture en couches** : Séparation models/database/interface
2. **Gestion des erreurs** : Validation et gestion appropriée
3. **Requêtes SQL avancées** : LIKE, ORDER BY, GROUP BY, COUNT
4. **Interface utilisateur** : Interaction en ligne de commande
5. **Bonnes pratiques** : Validation, confirmation, gestion des ressources
6. **Types de données** : Gestion des dates, strings, entiers
7. **Patterns Go** : Structs, méthodes, interfaces, defer

Ces solutions sont des exemples complets et fonctionnels que vous pouvez étendre selon vos besoins !

⏭️
