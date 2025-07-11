🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14-3 : Migrations

## Introduction

Les **migrations** sont des scripts qui permettent de faire évoluer la structure de votre base de données de manière contrôlée et versionnée. Imaginez-les comme un "historique des modifications" de votre base de données que vous pouvez appliquer ou annuler.

### Pourquoi les migrations sont-elles importantes ?

- 🔄 **Évolution contrôlée** : Modifier la structure sans perdre de données
- 👥 **Travail en équipe** : Synchroniser les changements entre développeurs
- 🚀 **Déploiement sûr** : Appliquer les changements en production sans risque
- ⏪ **Rollback** : Revenir en arrière en cas de problème
- 📝 **Traçabilité** : Historique de tous les changements

## Qu'est-ce qu'une migration ?

Une migration est un fichier qui contient :
- **UP** : Les instructions pour appliquer un changement (ajouter une table, une colonne, etc.)
- **DOWN** : Les instructions pour annuler ce changement (supprimer la table, la colonne, etc.)

### Exemple simple :
```sql
-- UP: Ajouter une table utilisateurs
CREATE TABLE utilisateurs (
    id SERIAL PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);

-- DOWN: Supprimer la table utilisateurs
DROP TABLE utilisateurs;
```

## Migrations en Go

En Go, il existe plusieurs outils pour gérer les migrations :

### 1. GORM AutoMigrate (simple mais limité)
### 2. golang-migrate (plus puissant et flexible)
### 3. Migrations manuelles avec GORM
### 4. Outils spécialisés (Atlas, Goose, etc.)

## 1. GORM AutoMigrate

### Principe de base

GORM peut automatiquement créer et modifier les tables basées sur vos structs :

```go
package main

import (
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Utilisateur struct {
    gorm.Model
    Nom   string `gorm:"size:100;not null"`
    Email string `gorm:"uniqueIndex;not null"`
    Age   int    `gorm:"check:age >= 0"`
}

func main() {
    db, err := gorm.Open(sqlite.Open("exemple.db"), &gorm.Config{})
    if err != nil {
        panic("Erreur de connexion à la base de données")
    }

    // Migration automatique
    err = db.AutoMigrate(&Utilisateur{})
    if err != nil {
        panic("Erreur lors de la migration")
    }

    println("Migration terminée !")
}
```

### Évolution d'un modèle

Modifions notre modèle pour ajouter un champ :

```go
type Utilisateur struct {
    gorm.Model
    Nom       string `gorm:"size:100;not null"`
    Email     string `gorm:"uniqueIndex;not null"`
    Age       int    `gorm:"check:age >= 0"`
    Telephone string `gorm:"size:20"` // Nouveau champ
}

func main() {
    db, err := gorm.Open(sqlite.Open("exemple.db"), &gorm.Config{})
    if err != nil {
        panic("Erreur de connexion")
    }

    // GORM va automatiquement ajouter la colonne telephone
    err = db.AutoMigrate(&Utilisateur{})
    if err != nil {
        panic("Erreur lors de la migration")
    }

    println("Nouvelle colonne ajoutée !")
}
```

### Limitations d'AutoMigrate

⚠️ **AutoMigrate ne peut pas :**
- Supprimer des colonnes
- Modifier le type d'une colonne existante
- Renommer des colonnes
- Créer des indexes complexes
- Gérer les données lors des changements

## 2. golang-migrate (Recommandé)

### Installation

```bash
# Installer l'outil CLI
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Ou avec Homebrew sur macOS
brew install golang-migrate

# Ajouter le package à votre projet
go get -u github.com/golang-migrate/migrate/v4
go get -u github.com/golang-migrate/migrate/v4/database/sqlite3
go get -u github.com/golang-migrate/migrate/v4/source/file
```

### Structure d'un projet avec migrations

```
mon-projet/
├── main.go
├── models/
│   └── user.go
├── migrations/
│   ├── 000001_create_users_table.up.sql
│   ├── 000001_create_users_table.down.sql
│   ├── 000002_add_phone_to_users.up.sql
│   ├── 000002_add_phone_to_users.down.sql
│   └── 000003_create_posts_table.up.sql
│   └── 000003_create_posts_table.down.sql
└── go.mod
```

### Créer votre première migration

```bash
# Se placer dans le dossier du projet
cd mon-projet

# Créer une migration pour la table utilisateurs
migrate create -ext sql -dir migrations -seq create_users_table
```

Cela crée deux fichiers :
- `000001_create_users_table.up.sql`
- `000001_create_users_table.down.sql`

### Contenu des fichiers de migration

**000001_create_users_table.up.sql** :
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at DATETIME,
    updated_at DATETIME,
    deleted_at DATETIME,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    age INTEGER CHECK(age >= 0),
    INDEX idx_users_deleted_at (deleted_at)
);
```

**000001_create_users_table.down.sql** :
```sql
DROP TABLE users;
```

### Ajouter une nouvelle migration

```bash
# Créer une migration pour ajouter le téléphone
migrate create -ext sql -dir migrations -seq add_phone_to_users
```

**000002_add_phone_to_users.up.sql** :
```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
```

**000002_add_phone_to_users.down.sql** :
```sql
ALTER TABLE users DROP COLUMN phone;
```

### Intégration dans le code Go

```go
package main

import (
    "database/sql"
    "log"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/sqlite3"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    _ "github.com/mattn/go-sqlite3"
)

func runMigrations(databaseURL string) error {
    // Ouvrir la connexion à la base de données
    db, err := sql.Open("sqlite3", databaseURL)
    if err != nil {
        return err
    }
    defer db.Close()

    // Créer l'instance de migration
    driver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
    if err != nil {
        return err
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations", // Chemin vers les fichiers de migration
        "sqlite3",           // Type de base de données
        driver,
    )
    if err != nil {
        return err
    }

    // Appliquer toutes les migrations
    err = m.Up()
    if err != nil && err != migrate.ErrNoChange {
        return err
    }

    log.Println("Migrations appliquées avec succès")
    return nil
}

func main() {
    // Appliquer les migrations au démarrage
    err := runMigrations("./app.db")
    if err != nil {
        log.Fatal("Erreur lors des migrations:", err)
    }

    // Reste de votre application...
    log.Println("Application démarrée")
}
```

### Commandes CLI utiles

```bash
# Appliquer toutes les migrations
migrate -path migrations -database "sqlite3://app.db" up

# Appliquer jusqu'à une version spécifique
migrate -path migrations -database "sqlite3://app.db" goto 2

# Annuler la dernière migration
migrate -path migrations -database "sqlite3://app.db" down 1

# Voir la version actuelle
migrate -path migrations -database "sqlite3://app.db" version

# Forcer une version (en cas de problème)
migrate -path migrations -database "sqlite3://app.db" force 1
```

## 3. Migrations manuelles avec GORM

Pour des cas plus complexes, vous pouvez écrire des migrations personnalisées :

```go
package main

import (
    "fmt"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type Migration struct {
    ID      string
    Applied bool
}

type MigrationRunner struct {
    db *gorm.DB
}

func NewMigrationRunner(db *gorm.DB) *MigrationRunner {
    // Créer la table des migrations si elle n'existe pas
    db.AutoMigrate(&Migration{})
    return &MigrationRunner{db: db}
}

func (mr *MigrationRunner) Run(id string, upFunc func(*gorm.DB) error, downFunc func(*gorm.DB) error) error {
    var migration Migration

    // Vérifier si la migration a déjà été appliquée
    result := mr.db.Where("id = ?", id).First(&migration)

    if result.Error == gorm.ErrRecordNotFound {
        // Appliquer la migration
        err := upFunc(mr.db)
        if err != nil {
            return fmt.Errorf("erreur lors de l'application de la migration %s: %v", id, err)
        }

        // Marquer comme appliquée
        migration = Migration{ID: id, Applied: true}
        mr.db.Create(&migration)

        fmt.Printf("Migration %s appliquée\n", id)
    } else {
        fmt.Printf("Migration %s déjà appliquée\n", id)
    }

    return nil
}

// Exemple d'utilisation
func main() {
    db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{})
    if err != nil {
        panic("Erreur de connexion")
    }

    runner := NewMigrationRunner(db)

    // Migration 1: Créer la table utilisateurs
    runner.Run("001_create_users", func(db *gorm.DB) error {
        return db.Exec(`
            CREATE TABLE users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                created_at DATETIME,
                updated_at DATETIME,
                deleted_at DATETIME,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL,
                age INTEGER CHECK(age >= 0)
            )
        `).Error
    }, func(db *gorm.DB) error {
        return db.Exec("DROP TABLE users").Error
    })

    // Migration 2: Ajouter la colonne téléphone
    runner.Run("002_add_phone", func(db *gorm.DB) error {
        return db.Exec("ALTER TABLE users ADD COLUMN phone VARCHAR(20)").Error
    }, func(db *gorm.DB) error {
        return db.Exec("ALTER TABLE users DROP COLUMN phone").Error
    })

    // Migration 3: Créer des données par défaut
    runner.Run("003_seed_data", func(db *gorm.DB) error {
        return db.Exec(`
            INSERT INTO users (name, email, age) VALUES
            ('Admin', 'admin@example.com', 30),
            ('Test User', 'test@example.com', 25)
        `).Error
    }, func(db *gorm.DB) error {
        return db.Exec("DELETE FROM users WHERE email IN ('admin@example.com', 'test@example.com')").Error
    })
}
```

## Migrations de données complexes

### Exemple : Restructurer des données

Supposons que nous voulions séparer le nom complet en prénom et nom :

**000004_split_name_field.up.sql** :
```sql
-- Ajouter les nouvelles colonnes
ALTER TABLE users ADD COLUMN first_name VARCHAR(50);
ALTER TABLE users ADD COLUMN last_name VARCHAR(50);

-- Migrer les données (exemple simple)
UPDATE users SET
    first_name = SUBSTR(name, 1, INSTR(name, ' ') - 1),
    last_name = SUBSTR(name, INSTR(name, ' ') + 1)
WHERE INSTR(name, ' ') > 0;

-- Pour les noms sans espace
UPDATE users SET
    first_name = name,
    last_name = ''
WHERE INSTR(name, ' ') = 0;
```

**000004_split_name_field.down.sql** :
```sql
-- Reconstruire le nom complet
UPDATE users SET name = first_name || ' ' || last_name
WHERE last_name != '';

UPDATE users SET name = first_name
WHERE last_name = '';

-- Supprimer les colonnes
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

## Migration avec GORM et golang-migrate

Vous pouvez combiner les deux approches :

```go
package main

import (
    "log"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/sqlite3"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    FirstName string `gorm:"size:50"`
    LastName  string `gorm:"size:50"`
    Email     string `gorm:"uniqueIndex"`
    Phone     string `gorm:"size:20"`
}

func main() {
    // 1. Appliquer les migrations de structure avec golang-migrate
    if err := runStructureMigrations(); err != nil {
        log.Fatal("Erreur migrations structure:", err)
    }

    // 2. Utiliser GORM pour le reste
    db, err := gorm.Open(sqlite.Open("app.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur connexion:", err)
    }

    // 3. Synchroniser avec les modèles (pour les indexes, etc.)
    if err := db.AutoMigrate(&User{}); err != nil {
        log.Fatal("Erreur AutoMigrate:", err)
    }

    log.Println("Base de données prête !")
}

func runStructureMigrations() error {
    db, err := sql.Open("sqlite3", "app.db")
    if err != nil {
        return err
    }
    defer db.Close()

    driver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
    if err != nil {
        return err
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations",
        "sqlite3",
        driver,
    )
    if err != nil {
        return err
    }

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    return nil
}
```

## Bonnes pratiques pour les migrations

### 1. Nommage des fichiers
```
YYYYMMDDHHMMSS_description.up.sql
YYYYMMDDHHMMSS_description.down.sql
```

Exemple :
```
20240315143000_create_users_table.up.sql
20240315143000_create_users_table.down.sql
```

### 2. Une migration = un changement
```bash
# Bon
migrate create -ext sql -dir migrations -seq add_user_phone
migrate create -ext sql -dir migrations -seq create_posts_table

# Mauvais (trop de changements dans une migration)
migrate create -ext sql -dir migrations -seq add_phone_and_create_posts_and_modify_users
```

### 3. Toujours tester le rollback
```sql
-- up.sql
ALTER TABLE users ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- down.sql - TESTER que cette commande fonctionne !
ALTER TABLE users DROP COLUMN status;
```

### 4. Sauvegardes avant migration
```go
func runMigrationsWithBackup(dbPath string) error {
    // Créer une sauvegarde
    backupPath := dbPath + ".backup." + time.Now().Format("20060102150405")
    if err := copyFile(dbPath, backupPath); err != nil {
        return fmt.Errorf("erreur sauvegarde: %v", err)
    }

    // Appliquer les migrations
    if err := runMigrations(dbPath); err != nil {
        log.Printf("Erreur migration, restauration depuis %s", backupPath)
        return copyFile(backupPath, dbPath)
    }

    return nil
}
```

### 5. Gestion des données sensibles
```sql
-- Attention aux données sensibles lors des migrations
ALTER TABLE users ADD COLUMN encrypted_data TEXT;

-- Utiliser des scripts séparés pour migrer les données sensibles
-- Ne jamais inclure de données sensibles dans les fichiers de migration
```

## Exemple complet : Blog avec migrations

### Structure du projet
```
blog/
├── main.go
├── models/
│   ├── user.go
│   └── post.go
├── migrations/
│   ├── 000001_create_users.up.sql
│   ├── 000001_create_users.down.sql
│   ├── 000002_create_posts.up.sql
│   ├── 000002_create_posts.down.sql
│   ├── 000003_add_user_bio.up.sql
│   └── 000003_add_user_bio.down.sql
└── migrate.go
```

### migrations/000001_create_users.up.sql
```sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL
);

CREATE INDEX idx_users_deleted_at ON users(deleted_at);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

### migrations/000002_create_posts.up.sql
```sql
CREATE TABLE posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    deleted_at DATETIME,
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    published BOOLEAN DEFAULT FALSE,
    user_id INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_posts_deleted_at ON posts(deleted_at);
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_published ON posts(published);
```

### migrations/000003_add_user_bio.up.sql
```sql
ALTER TABLE users ADD COLUMN bio TEXT;
```

### migrate.go
```go
package main

import (
    "database/sql"
    "log"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/sqlite3"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    _ "github.com/mattn/go-sqlite3"
)

func RunMigrations(databaseURL string) error {
    db, err := sql.Open("sqlite3", databaseURL)
    if err != nil {
        return err
    }
    defer db.Close()

    driver, err := sqlite3.WithInstance(db, &sqlite3.Config{})
    if err != nil {
        return err
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations",
        "sqlite3",
        driver,
    )
    if err != nil {
        return err
    }

    version, dirty, err := m.Version()
    if err == nil {
        log.Printf("Version actuelle: %d, dirty: %v", version, dirty)
    }

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    log.Println("Migrations terminées")
    return nil
}
```

### main.go
```go
package main

import (
    "log"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Username     string `gorm:"size:50;uniqueIndex;not null"`
    Email        string `gorm:"size:255;uniqueIndex;not null"`
    PasswordHash string `gorm:"size:255;not null"`
    Bio          string `gorm:"type:text"`
    Posts        []Post `gorm:"foreignKey:UserID"`
}

type Post struct {
    gorm.Model
    Title     string `gorm:"size:200;not null"`
    Content   string `gorm:"type:text;not null"`
    Published bool   `gorm:"default:false"`
    UserID    uint   `gorm:"not null"`
    User      User
}

func main() {
    // 1. Appliquer les migrations
    if err := RunMigrations("blog.db"); err != nil {
        log.Fatal("Erreur migrations:", err)
    }

    // 2. Initialiser GORM
    db, err := gorm.Open(sqlite.Open("blog.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur connexion:", err)
    }

    // 3. Créer des données de test
    user := User{
        Username:     "alice",
        Email:        "alice@blog.com",
        PasswordHash: "hashed_password",
        Bio:          "Développeuse passionnée",
    }

    db.Create(&user)

    post := Post{
        Title:     "Mon premier article",
        Content:   "Contenu de l'article...",
        Published: true,
        UserID:    user.ID,
    }

    db.Create(&post)

    log.Println("Blog initialisé avec succès !")
}
```

## Migrations en environnement de production

### Script de déploiement
```bash
#!/bin/bash
# deploy.sh

set -e

echo "Déploiement en cours..."

# Sauvegarder la base de données
cp production.db production.db.backup.$(date +%Y%m%d_%H%M%S)

# Appliquer les migrations
./migrate -path migrations -database "sqlite3://production.db" up

# Démarrer l'application
./mon-app

echo "Déploiement terminé"
```

### Surveillance des migrations
```go
func MonitorMigrations(db *gorm.DB) {
    // Vérifier l'état des migrations régulièrement
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()

    for range ticker.C {
        var count int64
        if err := db.Raw("SELECT COUNT(*) FROM schema_migrations").Scan(&count).Error; err != nil {
            log.Printf("Erreur vérification migrations: %v", err)
        } else {
            log.Printf("Migrations appliquées: %d", count)
        }
    }
}
```

## Outils et ressources

### Outils recommandés
- **golang-migrate** : Le plus populaire et stable
- **Atlas** : Moderne avec UI web
- **Goose** : Simple et efficace
- **GORM AutoMigrate** : Pour le développement rapide

### Commandes utiles
```bash
# Créer une nouvelle migration
migrate create -ext sql -dir migrations -seq description

# État actuel
migrate -path migrations -database "sqlite3://app.db" version

# Appliquer toutes les migrations
migrate -path migrations -database "sqlite3://app.db" up

# Rollback d'une migration
migrate -path migrations -database "sqlite3://app.db" down 1

# Aller à une version spécifique
migrate -path migrations -database "sqlite3://app.db" goto 5
```

## Résumé

Les migrations vous permettent de :
- ✅ **Évolution contrôlée** de votre base de données
- ✅ **Collaboration** efficace en équipe
- ✅ **Déploiements** sûrs et reproductibles
- ✅ **Rollback** en cas de problème
- ✅ **Historique** complet des changements

### Prochaine étape
Dans la section suivante, nous verrons comment optimiser les **connexions et pools** pour améliorer les performances de votre application.

## Ressources supplémentaires

- [Documentation golang-migrate](https://github.com/golang-migrate/migrate)
- [Guide des migrations GORM](https://gorm.io/docs/migration.html)
- [Bonnes pratiques de migration](https://flywaydb.org/documentation/concepts/migrations)

⏭️
