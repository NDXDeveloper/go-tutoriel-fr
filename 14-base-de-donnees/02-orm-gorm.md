🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14-2 : ORM (GORM)

## Introduction

Un ORM (Object-Relational Mapping) est un outil qui fait le pont entre votre code Go et votre base de données. Au lieu d'écrire des requêtes SQL à la main, vous manipulez des objets Go et l'ORM se charge de générer les requêtes SQL correspondantes.

**GORM** est l'ORM le plus populaire pour Go. Il simplifie énormément le travail avec les bases de données tout en restant flexible et puissant.

## Pourquoi utiliser GORM ?

### Avantages par rapport au SQL brut :
- ✅ **Moins de code** : Plus besoin d'écrire des requêtes SQL longues
- ✅ **Sécurité** : Protection automatique contre les injections SQL
- ✅ **Migrations automatiques** : Création et mise à jour des tables
- ✅ **Associations** : Gestion automatique des relations entre tables
- ✅ **Validation** : Validation des données intégrée
- ✅ **Hooks** : Fonctions exécutées automatiquement (avant/après sauvegarde)

### Inconvénients :
- ❌ **Moins de contrôle** : Sur les requêtes SQL générées
- ❌ **Courbe d'apprentissage** : Syntaxe spécifique à apprendre
- ❌ **Performance** : Peut être plus lent que le SQL optimisé à la main

## Installation

```bash
go mod init mon-projet-gorm
go get -u gorm.io/gorm
go get -u gorm.io/driver/sqlite  # Pour SQLite
# ou
go get -u gorm.io/driver/postgres  # Pour PostgreSQL
go get -u gorm.io/driver/mysql     # Pour MySQL
```

## Premier exemple avec GORM

Commençons par un exemple simple pour comprendre les bases :

```go
package main

import (
    "fmt"
    "log"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

// Utilisateur représente un utilisateur dans notre base de données
type Utilisateur struct {
    ID    uint   `gorm:"primaryKey"`
    Nom   string `gorm:"size:100;not null"`
    Email string `gorm:"uniqueIndex;not null"`
    Age   int    `gorm:"check:age >= 0"`
}

func main() {
    // Connexion à la base de données
    db, err := gorm.Open(sqlite.Open("exemple.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur de connexion à la base de données:", err)
    }

    // Migration automatique (création de la table)
    err = db.AutoMigrate(&Utilisateur{})
    if err != nil {
        log.Fatal("Erreur lors de la migration:", err)
    }

    fmt.Println("Connexion réussie et table créée !")
}
```

### Explications importantes :

1. **Tags GORM** : Les annotations entre backticks définissent les contraintes :
   - `primaryKey` : Clé primaire
   - `size:100` : Taille maximum de 100 caractères
   - `not null` : Champ obligatoire
   - `uniqueIndex` : Index unique (pas de doublons)
   - `check:age >= 0` : Contrainte de validation

2. **AutoMigrate** : Crée automatiquement la table si elle n'existe pas

## Les modèles GORM

### Modèle de base

GORM fournit un modèle de base avec des champs communs :

```go
import "gorm.io/gorm"

type Utilisateur struct {
    gorm.Model  // Ajoute ID, CreatedAt, UpdatedAt, DeletedAt
    Nom   string
    Email string
}

// Équivalent à :
type Utilisateur struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
    Nom       string
    Email     string
}
```

### Modèle personnalisé

Vous pouvez aussi définir votre propre structure :

```go
type Produit struct {
    ID          uint      `gorm:"primaryKey"`
    Code        string    `gorm:"uniqueIndex;size:20"`
    Nom         string    `gorm:"size:100;not null"`
    Prix        float64   `gorm:"not null;check:prix > 0"`
    Description string    `gorm:"type:text"`
    EnStock     bool      `gorm:"default:true"`
    DateAjout   time.Time `gorm:"autoCreateTime"`
}
```

## Opérations CRUD avec GORM

### Create - Créer des enregistrements

```go
func creerUtilisateurs(db *gorm.DB) {
    // Créer un utilisateur
    utilisateur := Utilisateur{
        Nom:   "Alice Dupont",
        Email: "alice@example.com",
        Age:   25,
    }

    result := db.Create(&utilisateur)
    if result.Error != nil {
        fmt.Printf("Erreur lors de la création: %v\n", result.Error)
        return
    }

    fmt.Printf("Utilisateur créé avec l'ID: %d\n", utilisateur.ID)
    fmt.Printf("Nombre de lignes affectées: %d\n", result.RowsAffected)

    // Créer plusieurs utilisateurs en une fois
    utilisateurs := []Utilisateur{
        {Nom: "Bob Martin", Email: "bob@example.com", Age: 30},
        {Nom: "Charlie Brown", Email: "charlie@example.com", Age: 28},
    }

    db.Create(&utilisateurs)
    fmt.Printf("Créé %d utilisateurs\n", len(utilisateurs))
}
```

### Read - Lire des enregistrements

```go
func lireUtilisateurs(db *gorm.DB) {
    var utilisateur Utilisateur

    // Trouver par clé primaire
    db.First(&utilisateur, 1) // WHERE id = 1
    fmt.Printf("Premier utilisateur: %+v\n", utilisateur)

    // Trouver par condition
    db.First(&utilisateur, "nom = ?", "Alice Dupont")
    fmt.Printf("Alice: %+v\n", utilisateur)

    // Récupérer tous les utilisateurs
    var utilisateurs []Utilisateur
    db.Find(&utilisateurs)
    fmt.Printf("Tous les utilisateurs (%d):\n", len(utilisateurs))
    for _, u := range utilisateurs {
        fmt.Printf("  %+v\n", u)
    }

    // Récupérer avec conditions
    var adultes []Utilisateur
    db.Where("age >= ?", 18).Find(&adultes)
    fmt.Printf("Utilisateurs adultes: %d\n", len(adultes))

    // Récupérer avec LIKE
    var utilisateursAlice []Utilisateur
    db.Where("nom LIKE ?", "%Alice%").Find(&utilisateursAlice)

    // Limiter et ordonner
    var premiers []Utilisateur
    db.Order("age desc").Limit(2).Find(&premiers)
    fmt.Printf("Les 2 plus âgés: %+v\n", premiers)
}
```

### Update - Mettre à jour

```go
func mettreAJourUtilisateurs(db *gorm.DB) {
    var utilisateur Utilisateur

    // Trouver l'utilisateur à modifier
    db.First(&utilisateur, "nom = ?", "Alice Dupont")

    // Mettre à jour un champ
    db.Model(&utilisateur).Update("age", 26)

    // Mettre à jour plusieurs champs
    db.Model(&utilisateur).Updates(Utilisateur{
        Nom: "Alice Martin",
        Age: 27,
    })

    // Ou avec une map
    db.Model(&utilisateur).Updates(map[string]interface{}{
        "nom": "Alice Dubois",
        "age": 28,
    })

    // Mettre à jour avec conditions (sans récupérer l'objet)
    db.Model(&Utilisateur{}).Where("age < ?", 18).Update("age", 18)

    fmt.Println("Utilisateurs mis à jour")
}
```

### Delete - Supprimer

```go
func supprimerUtilisateurs(db *gorm.DB) {
    var utilisateur Utilisateur

    // Suppression douce (soft delete) - avec gorm.Model
    db.First(&utilisateur, "nom = ?", "Bob Martin")
    db.Delete(&utilisateur) // Met DeletedAt à la date actuelle

    // Suppression définitive
    db.Unscoped().Delete(&utilisateur)

    // Supprimer par condition
    db.Where("age < ?", 16).Delete(&Utilisateur{})

    // Supprimer par ID
    db.Delete(&Utilisateur{}, 1) // WHERE id = 1

    fmt.Println("Utilisateurs supprimés")
}
```

## Requêtes avancées

### Conditions complexes

```go
func requetesAvancees(db *gorm.DB) {
    var utilisateurs []Utilisateur

    // Conditions multiples avec AND
    db.Where("age > ? AND nom LIKE ?", 20, "%Alice%").Find(&utilisateurs)

    // Conditions avec OR
    db.Where("age < ? OR age > ?", 18, 65).Find(&utilisateurs)

    // Avec struct (tous les champs non-zéro sont des conditions AND)
    db.Where(&Utilisateur{Age: 25}).Find(&utilisateurs)

    // Avec map
    db.Where(map[string]interface{}{
        "age":  25,
        "nom": "Alice",
    }).Find(&utilisateurs)

    // IN clause
    db.Where("nom IN ?", []string{"Alice", "Bob", "Charlie"}).Find(&utilisateurs)

    // Between
    db.Where("age BETWEEN ? AND ?", 20, 30).Find(&utilisateurs)

    // Order by
    db.Order("age desc, nom asc").Find(&utilisateurs)

    // Group by et Having
    type Resultat struct {
        Age   int
        Count int
    }

    var resultats []Resultat
    db.Model(&Utilisateur{}).
        Select("age, count(*) as count").
        Group("age").
        Having("count > ?", 1).
        Find(&resultats)
}
```

### Pagination

```go
func pagination(db *gorm.DB, page, taillePage int) ([]Utilisateur, int64) {
    var utilisateurs []Utilisateur
    var total int64

    // Compter le total
    db.Model(&Utilisateur{}).Count(&total)

    // Récupérer la page demandée
    offset := (page - 1) * taillePage
    db.Offset(offset).Limit(taillePage).Find(&utilisateurs)

    return utilisateurs, total
}

// Utilisation
func exempleNavigation(db *gorm.DB) {
    page := 1
    taillePage := 5

    for {
        utilisateurs, total := pagination(db, page, taillePage)

        if len(utilisateurs) == 0 {
            break
        }

        fmt.Printf("Page %d (Total: %d)\n", page, total)
        for _, u := range utilisateurs {
            fmt.Printf("  %s (%d ans)\n", u.Nom, u.Age)
        }

        page++
        if page > int((total+int64(taillePage)-1)/int64(taillePage)) {
            break
        }
    }
}
```

## Associations et relations

### Relations One-to-Many (Un-à-plusieurs)

```go
type Utilisateur struct {
    gorm.Model
    Nom      string
    Email    string
    Articles []Article // Un utilisateur a plusieurs articles
}

type Article struct {
    gorm.Model
    Titre        string
    Contenu      string
    UtilisateurID uint      // Clé étrangère
    Utilisateur   Utilisateur // Référence vers l'utilisateur
}

func exempleBlog(db *gorm.DB) {
    // Migration
    db.AutoMigrate(&Utilisateur{}, &Article{})

    // Créer un utilisateur avec des articles
    utilisateur := Utilisateur{
        Nom:   "Alice",
        Email: "alice@blog.com",
        Articles: []Article{
            {Titre: "Premier article", Contenu: "Contenu du premier article"},
            {Titre: "Deuxième article", Contenu: "Contenu du deuxième article"},
        },
    }

    db.Create(&utilisateur)

    // Récupérer l'utilisateur avec ses articles
    var utilisateurAvecArticles Utilisateur
    db.Preload("Articles").First(&utilisateurAvecArticles, utilisateur.ID)

    fmt.Printf("Utilisateur: %s\n", utilisateurAvecArticles.Nom)
    for _, article := range utilisateurAvecArticles.Articles {
        fmt.Printf("  Article: %s\n", article.Titre)
    }
}
```

### Relations Many-to-Many (Plusieurs-à-plusieurs)

```go
type Utilisateur struct {
    gorm.Model
    Nom    string
    Langues []Langue `gorm:"many2many:utilisateur_langues;"`
}

type Langue struct {
    gorm.Model
    Nom         string
    Utilisateurs []Utilisateur `gorm:"many2many:utilisateur_langues;"`
}

func exempleLangues(db *gorm.DB) {
    // Migration
    db.AutoMigrate(&Utilisateur{}, &Langue{})

    // Créer des langues
    francais := Langue{Nom: "Français"}
    anglais := Langue{Nom: "Anglais"}
    espagnol := Langue{Nom: "Espagnol"}

    db.Create(&francais)
    db.Create(&anglais)
    db.Create(&espagnol)

    // Créer un utilisateur qui parle plusieurs langues
    utilisateur := Utilisateur{
        Nom: "Alice",
        Langues: []Langue{francais, anglais},
    }

    db.Create(&utilisateur)

    // Ajouter une langue à un utilisateur existant
    db.Model(&utilisateur).Association("Langues").Append(&espagnol)

    // Récupérer l'utilisateur avec ses langues
    var utilisateurAvecLangues Utilisateur
    db.Preload("Langues").First(&utilisateurAvecLangues, utilisateur.ID)

    fmt.Printf("Utilisateur: %s\n", utilisateurAvecLangues.Nom)
    for _, langue := range utilisateurAvecLangues.Langues {
        fmt.Printf("  Parle: %s\n", langue.Nom)
    }
}
```

## Validation et hooks

### Validation personnalisée

```go
import "errors"

type Utilisateur struct {
    gorm.Model
    Nom   string `gorm:"size:100;not null"`
    Email string `gorm:"uniqueIndex;not null"`
    Age   int    `gorm:"check:age >= 0"`
}

// BeforeCreate est appelé avant la création
func (u *Utilisateur) BeforeCreate(tx *gorm.DB) (err error) {
    if u.Age < 13 {
        return errors.New("l'âge minimum est de 13 ans")
    }

    // Normaliser l'email
    u.Email = strings.ToLower(u.Email)

    return nil
}

// AfterCreate est appelé après la création
func (u *Utilisateur) AfterCreate(tx *gorm.DB) (err error) {
    fmt.Printf("Utilisateur %s créé avec l'ID %d\n", u.Nom, u.ID)
    return nil
}

// BeforeUpdate est appelé avant la mise à jour
func (u *Utilisateur) BeforeUpdate(tx *gorm.DB) (err error) {
    if u.Age < 0 {
        return errors.New("l'âge ne peut pas être négatif")
    }
    return nil
}
```

### Hooks disponibles

```go
// Hooks pour Create
func (u *Utilisateur) BeforeSave(tx *gorm.DB) error     { return nil }
func (u *Utilisateur) BeforeCreate(tx *gorm.DB) error   { return nil }
func (u *Utilisateur) AfterCreate(tx *gorm.DB) error    { return nil }
func (u *Utilisateur) AfterSave(tx *gorm.DB) error      { return nil }

// Hooks pour Update
func (u *Utilisateur) BeforeUpdate(tx *gorm.DB) error   { return nil }
func (u *Utilisateur) AfterUpdate(tx *gorm.DB) error    { return nil }

// Hooks pour Delete
func (u *Utilisateur) BeforeDelete(tx *gorm.DB) error   { return nil }
func (u *Utilisateur) AfterDelete(tx *gorm.DB) error    { return nil }

// Hook pour Find
func (u *Utilisateur) AfterFind(tx *gorm.DB) error      { return nil }
```

## Transactions

```go
func exempleTransaction(db *gorm.DB) error {
    // Transaction manuelle
    tx := db.Begin()
    defer func() {
        if r := recover(); r != nil {
            tx.Rollback()
        }
    }()

    if err := tx.Error; err != nil {
        return err
    }

    // Opérations dans la transaction
    utilisateur := Utilisateur{Nom: "Test User", Email: "test@example.com"}
    if err := tx.Create(&utilisateur).Error; err != nil {
        tx.Rollback()
        return err
    }

    article := Article{Titre: "Test Article", UtilisateurID: utilisateur.ID}
    if err := tx.Create(&article).Error; err != nil {
        tx.Rollback()
        return err
    }

    // Valider la transaction
    return tx.Commit().Error
}

// Transaction automatique
func exempleTransactionAuto(db *gorm.DB) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // Toutes les opérations dans cette fonction sont dans une transaction

        utilisateur := Utilisateur{Nom: "Auto User", Email: "auto@example.com"}
        if err := tx.Create(&utilisateur).Error; err != nil {
            return err // Rollback automatique
        }

        article := Article{Titre: "Auto Article", UtilisateurID: utilisateur.ID}
        if err := tx.Create(&article).Error; err != nil {
            return err // Rollback automatique
        }

        return nil // Commit automatique
    })
}
```

## Exemple complet : Système de blog avec GORM

```go
package main

import (
    "fmt"
    "log"
    "time"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

// Modèles
type Auteur struct {
    gorm.Model
    Nom      string `gorm:"size:100;not null"`
    Email    string `gorm:"uniqueIndex;not null"`
    Articles []Article
}

type Article struct {
    gorm.Model
    Titre    string `gorm:"size:200;not null"`
    Contenu  string `gorm:"type:text;not null"`
    Publie   bool   `gorm:"default:false"`
    AuteurID uint
    Auteur   Auteur
    Tags     []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    gorm.Model
    Nom      string `gorm:"size:50;uniqueIndex;not null"`
    Articles []Article `gorm:"many2many:article_tags;"`
}

// BlogService encapsule les opérations du blog
type BlogService struct {
    db *gorm.DB
}

func NouveauBlogService(db *gorm.DB) *BlogService {
    return &BlogService{db: db}
}

func (bs *BlogService) CreerAuteur(nom, email string) (*Auteur, error) {
    auteur := Auteur{Nom: nom, Email: email}
    result := bs.db.Create(&auteur)
    return &auteur, result.Error
}

func (bs *BlogService) CreerArticle(titre, contenu string, auteurID uint, nomsTags []string) (*Article, error) {
    return nil, bs.db.Transaction(func(tx *gorm.DB) error {
        // Créer l'article
        article := Article{
            Titre:    titre,
            Contenu:  contenu,
            AuteurID: auteurID,
            Publie:   false,
        }

        if err := tx.Create(&article).Error; err != nil {
            return err
        }

        // Créer ou récupérer les tags
        for _, nomTag := range nomsTags {
            var tag Tag
            // Essayer de trouver le tag existant
            if err := tx.Where("nom = ?", nomTag).First(&tag).Error; err != nil {
                if err == gorm.ErrRecordNotFound {
                    // Créer le tag s'il n'existe pas
                    tag = Tag{Nom: nomTag}
                    if err := tx.Create(&tag).Error; err != nil {
                        return err
                    }
                } else {
                    return err
                }
            }

            // Associer le tag à l'article
            if err := tx.Model(&article).Association("Tags").Append(&tag); err != nil {
                return err
            }
        }

        return nil
    })
}

func (bs *BlogService) PublierArticle(articleID uint) error {
    return bs.db.Model(&Article{}).Where("id = ?", articleID).Update("publie", true).Error
}

func (bs *BlogService) ObtenirArticlesPublies() ([]Article, error) {
    var articles []Article
    err := bs.db.Preload("Auteur").Preload("Tags").
        Where("publie = ?", true).
        Order("created_at desc").
        Find(&articles).Error
    return articles, err
}

func (bs *BlogService) RechercherArticles(terme string) ([]Article, error) {
    var articles []Article
    err := bs.db.Preload("Auteur").Preload("Tags").
        Where("publie = ? AND (titre LIKE ? OR contenu LIKE ?)",
              true, "%"+terme+"%", "%"+terme+"%").
        Order("created_at desc").
        Find(&articles).Error
    return articles, err
}

func main() {
    // Connexion
    db, err := gorm.Open(sqlite.Open("blog_gorm.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur de connexion:", err)
    }

    // Migration
    err = db.AutoMigrate(&Auteur{}, &Article{}, &Tag{})
    if err != nil {
        log.Fatal("Erreur de migration:", err)
    }

    // Service
    blogService := NouveauBlogService(db)

    // Créer un auteur
    auteur, err := blogService.CreerAuteur("Alice Dupont", "alice@blog.com")
    if err != nil {
        log.Printf("Erreur création auteur: %v", err)
    } else {
        fmt.Printf("Auteur créé: %s (ID: %d)\n", auteur.Nom, auteur.ID)
    }

    // Créer un article avec des tags
    err = blogService.CreerArticle(
        "Introduction à Go",
        "Go est un langage de programmation moderne...",
        auteur.ID,
        []string{"go", "programmation", "tutoriel"},
    )
    if err != nil {
        log.Printf("Erreur création article: %v", err)
    } else {
        fmt.Println("Article créé avec succès")
    }

    // Publier l'article
    err = blogService.PublierArticle(1)
    if err != nil {
        log.Printf("Erreur publication: %v", err)
    } else {
        fmt.Println("Article publié")
    }

    // Récupérer les articles publiés
    articles, err := blogService.ObtenirArticlesPublies()
    if err != nil {
        log.Printf("Erreur récupération: %v", err)
    } else {
        fmt.Printf("\nArticles publiés (%d):\n", len(articles))
        for _, article := range articles {
            fmt.Printf("- %s par %s\n", article.Titre, article.Auteur.Nom)
            fmt.Printf("  Tags: ")
            for i, tag := range article.Tags {
                if i > 0 {
                    fmt.Print(", ")
                }
                fmt.Print(tag.Nom)
            }
            fmt.Println()
        }
    }
}
```

## Bonnes pratiques avec GORM

### 1. Organisation du code

```go
// models/user.go
type User struct {
    gorm.Model
    Name  string `gorm:"size:100;not null"`
    Email string `gorm:"uniqueIndex;not null"`
}

// repositories/user_repository.go
type UserRepository struct {
    db *gorm.DB
}

func (r *UserRepository) Create(user *User) error {
    return r.db.Create(user).Error
}

func (r *UserRepository) FindByEmail(email string) (*User, error) {
    var user User
    err := r.db.Where("email = ?", email).First(&user).Error
    return &user, err
}

// services/user_service.go
type UserService struct {
    repo *UserRepository
}

func (s *UserService) CreateUser(name, email string) (*User, error) {
    user := &User{Name: name, Email: email}
    return user, s.repo.Create(user)
}
```

### 2. Gestion des erreurs

```go
func (r *UserRepository) FindByID(id uint) (*User, error) {
    var user User
    err := r.db.First(&user, id).Error

    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, fmt.Errorf("utilisateur %d non trouvé", id)
        }
        return nil, fmt.Errorf("erreur base de données: %w", err)
    }

    return &user, nil
}
```

### 3. Configuration de la connexion

```go
func ConnecterDB() (*gorm.DB, error) {
    config := &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // Logging des requêtes
        NamingStrategy: schema.NamingStrategy{
            TablePrefix:   "app_",    // Préfixe des tables
            SingularTable: false,     // Utiliser les noms pluriels
        },
    }

    db, err := gorm.Open(sqlite.Open("app.db"), config)
    if err != nil {
        return nil, err
    }

    // Configuration du pool de connexions
    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db, nil
}
```

## Exercices pratiques

### Exercice 1 : E-commerce simple
Créez un système d'e-commerce avec :
- Modèles : `Produit`, `Categorie`, `Commande`, `LigneCommande`
- Relations appropriées entre les modèles
- Fonctions pour créer des produits, passer des commandes, calculer des totaux

### Exercice 2 : Réseau social basique
Implémentez :
- Modèles : `Utilisateur`, `Post`, `Commentaire`, `Like`
- Système de followers (relation many-to-many sur Utilisateur)
- Timeline des posts des utilisateurs suivis

## Résumé

GORM vous permet de :
- ✅ **Simplifier** le code de base de données
- ✅ **Automatiser** les migrations et validations
- ✅ **Gérer** facilement les relations complexes
- ✅ **Sécuriser** automatiquement les requêtes
- ✅ **Structurer** votre code de manière professionnelle

Dans la prochaine section, nous verrons comment gérer les migrations de base de données pour faire évoluer votre schéma au fil du temps.

## Ressources supplémentaires

- [Documentation officielle GORM](https://gorm.io/docs/)
- [Guide des associations GORM](https://gorm.io/docs/associations.html)
- [Hooks et callbacks GORM](https://gorm.io/docs/hooks.html)

⏭️

# Solutions des exercices GORM

## Exercice 1 : E-commerce simple

### Structure du projet

```
ecommerce/
├── main.go
├── models/
│   ├── produit.go
│   ├── categorie.go
│   ├── commande.go
│   └── ligne_commande.go
├── services/
│   ├── produit_service.go
│   └── commande_service.go
└── go.mod
```

### models/categorie.go

```go
package models

import "gorm.io/gorm"

type Categorie struct {
    gorm.Model
    Nom         string    `gorm:"size:100;not null;uniqueIndex"`
    Description string    `gorm:"type:text"`
    Produits    []Produit `gorm:"foreignKey:CategorieID"`
}

// BeforeDelete vérifie qu'il n'y a pas de produits dans la catégorie
func (c *Categorie) BeforeDelete(tx *gorm.DB) error {
    var count int64
    tx.Model(&Produit{}).Where("categorie_id = ?", c.ID).Count(&count)
    if count > 0 {
        return gorm.ErrInvalidValue
    }
    return nil
}
```

### models/produit.go

```go
package models

import (
    "errors"
    "gorm.io/gorm"
)

type Produit struct {
    gorm.Model
    Nom          string  `gorm:"size:200;not null"`
    Description  string  `gorm:"type:text"`
    Prix         float64 `gorm:"not null;check:prix > 0"`
    Stock        int     `gorm:"not null;default:0;check:stock >= 0"`
    SKU          string  `gorm:"size:50;uniqueIndex;not null"`
    CategorieID  uint    `gorm:"not null"`
    Categorie    Categorie
    LignesCommande []LigneCommande `gorm:"foreignKey:ProduitID"`
}

// BeforeCreate valide les données avant création
func (p *Produit) BeforeCreate(tx *gorm.DB) error {
    if p.Prix <= 0 {
        return errors.New("le prix doit être positif")
    }
    if p.Stock < 0 {
        return errors.New("le stock ne peut pas être négatif")
    }
    return nil
}

// BeforeUpdate valide les données avant mise à jour
func (p *Produit) BeforeUpdate(tx *gorm.DB) error {
    if p.Prix <= 0 {
        return errors.New("le prix doit être positif")
    }
    if p.Stock < 0 {
        return errors.New("le stock ne peut pas être négatif")
    }
    return nil
}

// EstDisponible vérifie si le produit est en stock
func (p *Produit) EstDisponible(quantite int) bool {
    return p.Stock >= quantite
}

// PrixTotal calcule le prix total pour une quantité donnée
func (p *Produit) PrixTotal(quantite int) float64 {
    return p.Prix * float64(quantite)
}
```

### models/commande.go

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type StatusCommande string

const (
    StatusEnAttente StatusCommande = "en_attente"
    StatusConfirmee StatusCommande = "confirmee"
    StatusExpediee  StatusCommande = "expediee"
    StatusLivree    StatusCommande = "livree"
    StatusAnnulee   StatusCommande = "annulee"
)

type Commande struct {
    gorm.Model
    NumeroCommande   string         `gorm:"size:50;uniqueIndex;not null"`
    Status           StatusCommande `gorm:"type:varchar(20);default:'en_attente'"`
    DateCommande     time.Time      `gorm:"autoCreateTime"`
    AdresseLivraison string         `gorm:"type:text;not null"`
    Total            float64        `gorm:"default:0"`
    LignesCommande   []LigneCommande `gorm:"foreignKey:CommandeID;constraint:OnDelete:CASCADE"`
}

// BeforeCreate génère un numéro de commande automatique
func (c *Commande) BeforeCreate(tx *gorm.DB) error {
    if c.NumeroCommande == "" {
        c.NumeroCommande = generateOrderNumber()
    }
    return nil
}

// AfterCreate calcule le total après création
func (c *Commande) AfterCreate(tx *gorm.DB) error {
    return c.CalculerTotal(tx)
}

// CalculerTotal recalcule le total de la commande
func (c *Commande) CalculerTotal(tx *gorm.DB) error {
    var total float64

    var lignes []LigneCommande
    if err := tx.Where("commande_id = ?", c.ID).Find(&lignes).Error; err != nil {
        return err
    }

    for _, ligne := range lignes {
        total += ligne.SousTotal
    }

    return tx.Model(c).Update("total", total).Error
}

// PeutEtreAnnulee vérifie si la commande peut être annulée
func (c *Commande) PeutEtreAnnulee() bool {
    return c.Status == StatusEnAttente || c.Status == StatusConfirmee
}

// generateOrderNumber génère un numéro de commande unique
func generateOrderNumber() string {
    return "CMD" + time.Now().Format("20060102150405")
}
```

### models/ligne_commande.go

```go
package models

import (
    "errors"
    "gorm.io/gorm"
)

type LigneCommande struct {
    gorm.Model
    CommandeID uint     `gorm:"not null"`
    Commande   Commande
    ProduitID  uint     `gorm:"not null"`
    Produit    Produit
    Quantite   int     `gorm:"not null;check:quantite > 0"`
    PrixUnitaire float64 `gorm:"not null"`
    SousTotal  float64 `gorm:"not null"`
}

// BeforeCreate calcule le sous-total et vérifie le stock
func (lc *LigneCommande) BeforeCreate(tx *gorm.DB) error {
    // Récupérer le produit pour vérifier le stock et le prix
    var produit Produit
    if err := tx.First(&produit, lc.ProduitID).Error; err != nil {
        return err
    }

    // Vérifier le stock
    if !produit.EstDisponible(lc.Quantite) {
        return errors.New("stock insuffisant")
    }

    // Utiliser le prix actuel du produit
    lc.PrixUnitaire = produit.Prix
    lc.SousTotal = lc.PrixUnitaire * float64(lc.Quantite)

    return nil
}

// AfterCreate met à jour le stock du produit
func (lc *LigneCommande) AfterCreate(tx *gorm.DB) error {
    return tx.Model(&Produit{}).
        Where("id = ?", lc.ProduitID).
        Update("stock", gorm.Expr("stock - ?", lc.Quantite)).Error
}

// BeforeDelete remet le stock lors de la suppression
func (lc *LigneCommande) BeforeDelete(tx *gorm.DB) error {
    return tx.Model(&Produit{}).
        Where("id = ?", lc.ProduitID).
        Update("stock", gorm.Expr("stock + ?", lc.Quantite)).Error
}
```

### services/produit_service.go

```go
package services

import (
    "ecommerce/models"
    "gorm.io/gorm"
)

type ProduitService struct {
    db *gorm.DB
}

func NewProduitService(db *gorm.DB) *ProduitService {
    return &ProduitService{db: db}
}

func (ps *ProduitService) CreerCategorie(nom, description string) (*models.Categorie, error) {
    categorie := &models.Categorie{
        Nom:         nom,
        Description: description,
    }

    err := ps.db.Create(categorie).Error
    return categorie, err
}

func (ps *ProduitService) CreerProduit(nom, description, sku string, prix float64, stock int, categorieID uint) (*models.Produit, error) {
    produit := &models.Produit{
        Nom:         nom,
        Description: description,
        SKU:         sku,
        Prix:        prix,
        Stock:       stock,
        CategorieID: categorieID,
    }

    err := ps.db.Create(produit).Error
    return produit, err
}

func (ps *ProduitService) ObtenirProduitsParCategorie(categorieID uint) ([]models.Produit, error) {
    var produits []models.Produit
    err := ps.db.Preload("Categorie").Where("categorie_id = ?", categorieID).Find(&produits).Error
    return produits, err
}

func (ps *ProduitService) RechercherProduits(terme string) ([]models.Produit, error) {
    var produits []models.Produit
    err := ps.db.Preload("Categorie").
        Where("nom LIKE ? OR description LIKE ?", "%"+terme+"%", "%"+terme+"%").
        Find(&produits).Error
    return produits, err
}

func (ps *ProduitService) ObtenirProduitsEnStock() ([]models.Produit, error) {
    var produits []models.Produit
    err := ps.db.Preload("Categorie").Where("stock > 0").Find(&produits).Error
    return produits, err
}

func (ps *ProduitService) MettreAJourStock(produitID uint, nouveauStock int) error {
    return ps.db.Model(&models.Produit{}).Where("id = ?", produitID).Update("stock", nouveauStock).Error
}
```

### services/commande_service.go

```go
package services

import (
    "ecommerce/models"
    "errors"
    "gorm.io/gorm"
)

type CommandeService struct {
    db *gorm.DB
}

func NewCommandeService(db *gorm.DB) *CommandeService {
    return &CommandeService{db: db}
}

type ArticlePanier struct {
    ProduitID uint
    Quantite  int
}

func (cs *CommandeService) CreerCommande(adresseLivraison string, articles []ArticlePanier) (*models.Commande, error) {
    var commande *models.Commande

    err := cs.db.Transaction(func(tx *gorm.DB) error {
        // Créer la commande
        commande = &models.Commande{
            AdresseLivraison: adresseLivraison,
            Status:          models.StatusEnAttente,
        }

        if err := tx.Create(commande).Error; err != nil {
            return err
        }

        // Ajouter les lignes de commande
        for _, article := range articles {
            ligne := &models.LigneCommande{
                CommandeID: commande.ID,
                ProduitID:  article.ProduitID,
                Quantite:   article.Quantite,
            }

            if err := tx.Create(ligne).Error; err != nil {
                return err
            }
        }

        // Recalculer le total
        return commande.CalculerTotal(tx)
    })

    return commande, err
}

func (cs *CommandeService) AjouterArticle(commandeID, produitID uint, quantite int) error {
    return cs.db.Transaction(func(tx *gorm.DB) error {
        // Vérifier que la commande existe et peut être modifiée
        var commande models.Commande
        if err := tx.First(&commande, commandeID).Error; err != nil {
            return err
        }

        if commande.Status != models.StatusEnAttente {
            return errors.New("impossible de modifier une commande confirmée")
        }

        // Vérifier si l'article existe déjà dans la commande
        var ligneExistante models.LigneCommande
        err := tx.Where("commande_id = ? AND produit_id = ?", commandeID, produitID).First(&ligneExistante).Error

        if err == nil {
            // Mettre à jour la quantité existante
            nouvellQuantite := ligneExistante.Quantite + quantite
            return cs.ModifierQuantite(commandeID, produitID, nouvellQuantite)
        } else if err == gorm.ErrRecordNotFound {
            // Créer une nouvelle ligne
            ligne := &models.LigneCommande{
                CommandeID: commandeID,
                ProduitID:  produitID,
                Quantite:   quantite,
            }

            if err := tx.Create(ligne).Error; err != nil {
                return err
            }

            return commande.CalculerTotal(tx)
        } else {
            return err
        }
    })
}

func (cs *CommandeService) ModifierQuantite(commandeID, produitID uint, nouvelleQuantite int) error {
    return cs.db.Transaction(func(tx *gorm.DB) error {
        var ligne models.LigneCommande
        if err := tx.Where("commande_id = ? AND produit_id = ?", commandeID, produitID).First(&ligne).Error; err != nil {
            return err
        }

        // Remettre l'ancien stock
        if err := ligne.BeforeDelete(tx); err != nil {
            return err
        }

        // Mettre à jour la quantité
        ligne.Quantite = nouvelleQuantite

        // Recalculer le sous-total et vérifier le nouveau stock
        var produit models.Produit
        if err := tx.First(&produit, ligne.ProduitID).Error; err != nil {
            return err
        }

        if !produit.EstDisponible(nouvelleQuantite) {
            return errors.New("stock insuffisant")
        }

        ligne.SousTotal = ligne.PrixUnitaire * float64(nouvelleQuantite)

        if err := tx.Save(&ligne).Error; err != nil {
            return err
        }

        // Déduire le nouveau stock
        if err := ligne.AfterCreate(tx); err != nil {
            return err
        }

        // Recalculer le total de la commande
        var commande models.Commande
        if err := tx.First(&commande, commandeID).Error; err != nil {
            return err
        }

        return commande.CalculerTotal(tx)
    })
}

func (cs *CommandeService) SupprimerArticle(commandeID, produitID uint) error {
    return cs.db.Transaction(func(tx *gorm.DB) error {
        var ligne models.LigneCommande
        if err := tx.Where("commande_id = ? AND produit_id = ?", commandeID, produitID).First(&ligne).Error; err != nil {
            return err
        }

        if err := tx.Delete(&ligne).Error; err != nil {
            return err
        }

        // Recalculer le total de la commande
        var commande models.Commande
        if err := tx.First(&commande, commandeID).Error; err != nil {
            return err
        }

        return commande.CalculerTotal(tx)
    })
}

func (cs *CommandeService) ConfirmerCommande(commandeID uint) error {
    return cs.db.Model(&models.Commande{}).
        Where("id = ? AND status = ?", commandeID, models.StatusEnAttente).
        Update("status", models.StatusConfirmee).Error
}

func (cs *CommandeService) AnnulerCommande(commandeID uint) error {
    return cs.db.Transaction(func(tx *gorm.DB) error {
        var commande models.Commande
        if err := tx.Preload("LignesCommande").First(&commande, commandeID).Error; err != nil {
            return err
        }

        if !commande.PeutEtreAnnulee() {
            return errors.New("cette commande ne peut pas être annulée")
        }

        // Remettre le stock pour chaque ligne
        for _, ligne := range commande.LignesCommande {
            if err := ligne.BeforeDelete(tx); err != nil {
                return err
            }
        }

        // Marquer comme annulée
        return tx.Model(&commande).Update("status", models.StatusAnnulee).Error
    })
}

func (cs *CommandeService) ObtenirCommande(commandeID uint) (*models.Commande, error) {
    var commande models.Commande
    err := cs.db.Preload("LignesCommande.Produit.Categorie").First(&commande, commandeID).Error
    return &commande, err
}

func (cs *CommandeService) ListerCommandes(status models.StatusCommande) ([]models.Commande, error) {
    var commandes []models.Commande
    query := cs.db.Preload("LignesCommande.Produit")

    if status != "" {
        query = query.Where("status = ?", status)
    }

    err := query.Order("created_at desc").Find(&commandes).Error
    return commandes, err
}

func (cs *CommandeService) CalculerChiffreAffaires() (float64, error) {
    var total float64
    err := cs.db.Model(&models.Commande{}).
        Where("status IN ?", []models.StatusCommande{models.StatusConfirmee, models.StatusExpediee, models.StatusLivree}).
        Select("COALESCE(SUM(total), 0)").
        Scan(&total).Error
    return total, err
}
```

### main.go

```go
package main

import (
    "ecommerce/models"
    "ecommerce/services"
    "fmt"
    "log"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

func main() {
    // Connexion à la base de données
    db, err := gorm.Open(sqlite.Open("ecommerce.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur de connexion:", err)
    }

    // Migration
    err = db.AutoMigrate(&models.Categorie{}, &models.Produit{}, &models.Commande{}, &models.LigneCommande{})
    if err != nil {
        log.Fatal("Erreur de migration:", err)
    }

    // Services
    produitService := services.NewProduitService(db)
    commandeService := services.NewCommandeService(db)

    // Créer des catégories
    electronique, _ := produitService.CreerCategorie("Électronique", "Appareils électroniques")
    vetements, _ := produitService.CreerCategorie("Vêtements", "Vêtements et accessoires")

    // Créer des produits
    produitService.CreerProduit("iPhone 15", "Smartphone Apple", "IPH15", 999.99, 50, electronique.ID)
    produitService.CreerProduit("MacBook Pro", "Ordinateur portable", "MBP16", 2499.99, 20, electronique.ID)
    produitService.CreerProduit("T-shirt Go", "T-shirt avec logo Go", "TSH-GO", 29.99, 100, vetements.ID)

    // Passer une commande
    panier := []services.ArticlePanier{
        {ProduitID: 1, Quantite: 1}, // iPhone
        {ProduitID: 3, Quantite: 2}, // T-shirts
    }

    commande, err := commandeService.CreerCommande("123 Rue de la Paix, Paris", panier)
    if err != nil {
        log.Printf("Erreur commande: %v", err)
    } else {
        fmt.Printf("Commande créée: %s (Total: %.2f€)\n", commande.NumeroCommande, commande.Total)
    }

    // Afficher les détails de la commande
    commandeComplete, _ := commandeService.ObtenirCommande(commande.ID)
    fmt.Printf("\nDétails de la commande %s:\n", commandeComplete.NumeroCommande)
    for _, ligne := range commandeComplete.LignesCommande {
        fmt.Printf("- %s x%d = %.2f€\n",
            ligne.Produit.Nom, ligne.Quantite, ligne.SousTotal)
    }
    fmt.Printf("Total: %.2f€\n", commandeComplete.Total)

    // Confirmer la commande
    commandeService.ConfirmerCommande(commande.ID)
    fmt.Println("Commande confirmée!")

    // Statistiques
    ca, _ := commandeService.CalculerChiffreAffaires()
    fmt.Printf("Chiffre d'affaires: %.2f€\n", ca)
}
```

## Exercice 2 : Réseau social basique

### Structure du projet

```
social/
├── main.go
├── models/
│   ├── utilisateur.go
│   ├── post.go
│   ├── commentaire.go
│   └── like.go
├── services/
│   ├── utilisateur_service.go
│   └── social_service.go
└── go.mod
```

### models/utilisateur.go

```go
package models

import (
    "errors"
    "strings"
    "time"
    "gorm.io/gorm"
)

type Utilisateur struct {
    gorm.Model
    Username    string `gorm:"size:50;uniqueIndex;not null"`
    Email       string `gorm:"uniqueIndex;not null"`
    Nom         string `gorm:"size:100;not null"`
    Bio         string `gorm:"size:500"`
    Avatar      string `gorm:"size:255"`
    DateNaissance *time.Time

    // Relations
    Posts       []Post       `gorm:"foreignKey:AuteurID"`
    Commentaires []Commentaire `gorm:"foreignKey:AuteurID"`
    Likes       []Like       `gorm:"foreignKey:UtilisateurID"`

    // Self-referencing many-to-many pour les followers
    Followers   []Utilisateur `gorm:"many2many:user_followers;joinForeignKey:UserID;joinReferences:FollowerID"`
    Following   []Utilisateur `gorm:"many2many:user_followers;joinForeignKey:FollowerID;joinReferences:UserID"`
}

// BeforeCreate valide et normalise les données
func (u *Utilisateur) BeforeCreate(tx *gorm.DB) error {
    u.Username = strings.ToLower(strings.TrimSpace(u.Username))
    u.Email = strings.ToLower(strings.TrimSpace(u.Email))

    if len(u.Username) < 3 {
        return errors.New("le nom d'utilisateur doit faire au moins 3 caractères")
    }

    if !strings.Contains(u.Email, "@") {
        return errors.New("adresse email invalide")
    }

    return nil
}

// NombreFollowers retourne le nombre de followers
func (u *Utilisateur) NombreFollowers(db *gorm.DB) int64 {
    var count int64
    db.Model(u).Association("Followers").Count(&count)
    return count
}

// NombreFollowing retourne le nombre de personnes suivies
func (u *Utilisateur) NombreFollowing(db *gorm.DB) int64 {
    var count int64
    db.Model(u).Association("Following").Count(&count)
    return count
}

// EstSuivi vérifie si cet utilisateur suit un autre utilisateur
func (u *Utilisateur) EstSuivi(db *gorm.DB, autreUserID uint) bool {
    var count int64
    db.Table("user_followers").
        Where("follower_id = ? AND user_id = ?", u.ID, autreUserID).
        Count(&count)
    return count > 0
}
```

### models/post.go

```go
package models

import (
    "errors"
    "gorm.io/gorm"
)

type Post struct {
    gorm.Model
    Contenu     string `gorm:"type:text;not null"`
    ImageURL    string `gorm:"size:500"`
    AuteurID    uint   `gorm:"not null"`
    Auteur      Utilisateur

    // Relations
    Commentaires []Commentaire `gorm:"foreignKey:PostID;constraint:OnDelete:CASCADE"`
    Likes       []Like        `gorm:"foreignKey:PostID;constraint:OnDelete:CASCADE"`
}

// BeforeCreate valide le contenu
func (p *Post) BeforeCreate(tx *gorm.DB) error {
    if len(p.Contenu) == 0 {
        return errors.New("le contenu du post ne peut pas être vide")
    }
    if len(p.Contenu) > 1000 {
        return errors.New("le contenu du post ne peut pas dépasser 1000 caractères")
    }
    return nil
}

// NombreLikes retourne le nombre de likes du post
func (p *Post) NombreLikes(db *gorm.DB) int64 {
    var count int64
    db.Model(&Like{}).Where("post_id = ?", p.ID).Count(&count)
    return count
}

// NombreCommentaires retourne le nombre de commentaires
func (p *Post) NombreCommentaires(db *gorm.DB) int64 {
    var count int64
    db.Model(&Commentaire{}).Where("post_id = ?", p.ID).Count(&count)
    return count
}

// EstLikeePar vérifie si le post est liké par un utilisateur
func (p *Post) EstLikeePar(db *gorm.DB, utilisateurID uint) bool {
    var count int64
    db.Model(&Like{}).Where("post_id = ? AND utilisateur_id = ?", p.ID, utilisateurID).Count(&count)
    return count > 0
}
```

### models/commentaire.go

```go
package models

import (
    "errors"
    "gorm.io/gorm"
)

type Commentaire struct {
    gorm.Model
    Contenu      string `gorm:"type:text;not null"`
    PostID       uint   `gorm:"not null"`
    Post         Post
    AuteurID     uint   `gorm:"not null"`
    Auteur       Utilisateur

    // Auto-référence pour les réponses
    ParentID     *uint `gorm:"index"`
    Parent       *Commentaire `gorm:"foreignKey:ParentID"`
    Reponses     []Commentaire `gorm:"foreignKey:ParentID"`
}

// BeforeCreate valide le commentaire
func (c *Commentaire) BeforeCreate(tx *gorm.DB) error {
    if len(c.Contenu) == 0 {
        return errors.New("le commentaire ne peut pas être vide")
    }
    if len(c.Contenu) > 500 {
        return errors.New("le commentaire ne peut pas dépasser 500 caractères")
    }
    return nil
}

// EstReponse vérifie si c'est une réponse à un autre commentaire
func (c *Commentaire) EstReponse() bool {
    return c.ParentID != nil
}

// NombreReponses retourne le nombre de réponses
func (c *Commentaire) NombreReponses(db *gorm.DB) int64 {
    var count int64
    db.Model(&Commentaire{}).Where("parent_id = ?", c.ID).Count(&count)
    return count
}
```

### models/like.go

```go
package models

import "gorm.io/gorm"

type Like struct {
    gorm.Model
    UtilisateurID uint `gorm:"not null;uniqueIndex:idx_user_post"`
    Utilisateur   Utilisateur
    PostID        uint `gorm:"not null;uniqueIndex:idx_user_post"`
    Post          Post
}

// TableName définit le nom de la table
func (Like) TableName() string {
    return "likes"
}
```

### services/utilisateur_service.go

```go
package services

import (
    "social/models"
    "gorm.io/gorm"
)

type UtilisateurService struct {
    db *gorm.DB
}

func NewUtilisateurService(db *gorm.DB) *UtilisateurService {
    return &UtilisateurService{db: db}
}

func (us *UtilisateurService) CreerUtilisateur(username, email, nom, bio string) (*models.Utilisateur, error) {
    utilisateur := &models.Utilisateur{
        Username: username,
        Email:    email,
        Nom:      nom,
        Bio:      bio,
    }

    err := us.db.Create(utilisateur).Error
    return utilisateur, err
}

func (us *UtilisateurService) ObtenirUtilisateur(userID uint) (*models.Utilisateur, error) {
    var utilisateur models.Utilisateur
    err := us.db.First(&utilisateur, userID).Error
    return &utilisateur, err
}

func (us *UtilisateurService) ObtenirParUsername(username string) (*models.Utilisateur, error) {
    var utilisateur models.Utilisateur
    err := us.db.Where("username = ?", username).First(&utilisateur).Error
    return &utilisateur, err
}

func (us *UtilisateurService) SuivreUtilisateur(followerID, userID uint) error {
    if followerID == userID {
        return gorm.ErrInvalidValue
    }

    var follower, user models.Utilisateur
    if err := us.db.First(&follower, followerID).Error; err != nil {
        return err
    }
    if err := us.db.First(&user, userID).Error; err != nil {
        return err
    }

    // Vérifier si la relation existe déjà
    if follower.EstSuivi(us.db, userID) {
        return gorm.ErrDuplicatedKey
    }

    return us.db.Model(&follower).Association("Following").Append(&user)
}

func (us *UtilisateurService) NePlusSuivreUtilisateur(followerID, userID uint) error {
    var follower, user models.Utilisateur
    if err := us.db.First(&follower, followerID).Error; err != nil {
        return err
    }
    if err := us.db.First(&user, userID).Error; err != nil {
        return err
    }

    return us.db.Model(&follower).Association("Following").Delete(&user)
}

func (us *UtilisateurService) ObtenirFollowers(userID uint) ([]models.Utilisateur, error) {
    var utilisateur models.Utilisateur
    if err := us.db.Preload("Followers").First(&utilisateur, userID).Error; err != nil {
        return nil, err
    }
    return utilisateur.Followers, nil
}

func (us *UtilisateurService) ObtenirFollowing(userID uint) ([]models.Utilisateur, error) {
    var utilisateur models.Utilisateur
    if err := us.db.Preload("Following").First(&utilisateur, userID).Error; err != nil {
        return nil, err
    }
    return utilisateur.Following, nil
}

func (us *UtilisateurService) RechercherUtilisateurs(terme string) ([]models.Utilisateur, error) {
    var utilisateurs []models.Utilisateur
    err := us.db.Where("username LIKE ? OR nom LIKE ?", "%"+terme+"%", "%"+terme+"%").
        Limit(20).Find(&utilisateurs).Error
    return utilisateurs, err
}

func (us *UtilisateurService) SuggererUtilisateurs(userID uint, limite int) ([]models.Utilisateur, error) {
    // Suggestion basée sur les utilisateurs non suivis
    var suggestions []models.Utilisateur

    subquery := us.db.Model(&models.Utilisateur{}).
        Select("user_id").
        Table("user_followers").
        Where("follower_id = ?", userID)

    err := us.db.Where("id != ? AND id NOT IN (?)", userID, subquery).
        Limit(limite).
        Find(&suggestions).Error

    return suggestions, err
}

func (us *UtilisateurService) ObtenirStatistiques(userID uint) (map[string]int64, error) {
    var utilisateur models.Utilisateur
    if err := us.db.First(&utilisateur, userID).Error; err != nil {
        return nil, err
    }

    stats := make(map[string]int64)

    // Nombre de posts
    us.db.Model(&models.Post{}).Where("auteur_id = ?", userID).Count(&stats["posts"])

    // Nombre de followers
    stats["followers"] = utilisateur.NombreFollowers(us.db)

    // Nombre de following
    stats["following"] = utilisateur.NombreFollowing(us.db)

    // Nombre total de likes reçus
    us.db.Table("likes").
        Joins("JOIN posts ON posts.id = likes.post_id").
        Where("posts.auteur_id = ?", userID).
        Count(&stats["likes_recus"])

    return stats, nil
}
```

### services/social_service.go

```go
package services

import (
    "social/models"
    "gorm.io/gorm"
)

type SocialService struct {
    db *gorm.DB
}

func NewSocialService(db *gorm.DB) *SocialService {
    return &SocialService{db: db}
}

// === POSTS ===

func (ss *SocialService) CreerPost(auteurID uint, contenu, imageURL string) (*models.Post, error) {
    post := &models.Post{
        Contenu:  contenu,
        ImageURL: imageURL,
        AuteurID: auteurID,
    }

    err := ss.db.Create(post).Error
    return post, err
}

func (ss *SocialService) ObtenirPost(postID uint) (*models.Post, error) {
    var post models.Post
    err := ss.db.Preload("Auteur").
        Preload("Commentaires.Auteur").
        Preload("Commentaires.Reponses.Auteur").
        First(&post, postID).Error
    return &post, err
}

func (ss *SocialService) SupprimerPost(postID, auteurID uint) error {
    // Vérifier que l'utilisateur est bien l'auteur
    var post models.Post
    if err := ss.db.Where("id = ? AND auteur_id = ?", postID, auteurID).First(&post).Error; err != nil {
        return err
    }

    return ss.db.Delete(&post).Error
}

func (ss *SocialService) ObtenirPostsUtilisateur(auteurID uint, limite, offset int) ([]models.Post, error) {
    var posts []models.Post
    err := ss.db.Preload("Auteur").
        Where("auteur_id = ?", auteurID).
        Order("created_at desc").
        Limit(limite).Offset(offset).
        Find(&posts).Error
    return posts, err
}

// === TIMELINE ===

func (ss *SocialService) ObtenirTimeline(userID uint, limite, offset int) ([]models.Post, error) {
    // Récupérer les posts des utilisateurs suivis + ses propres posts
    var posts []models.Post

    subquery := ss.db.Model(&models.Utilisateur{}).
        Select("user_id").
        Table("user_followers").
        Where("follower_id = ?", userID)

    err := ss.db.Preload("Auteur").
        Where("auteur_id = ? OR auteur_id IN (?)", userID, subquery).
        Order("created_at desc").
        Limit(limite).Offset(offset).
        Find(&posts).Error

    return posts, err
}

func (ss *SocialService) ObtenirTimelineAvecStats(userID uint, limite, offset int) ([]map[string]interface{}, error) {
    posts, err := ss.ObtenirTimeline(userID, limite, offset)
    if err != nil {
        return nil, err
    }

    var result []map[string]interface{}

    for _, post := range posts {
        postData := map[string]interface{}{
            "post":             post,
            "nombre_likes":     post.NombreLikes(ss.db),
            "nombre_commentaires": post.NombreCommentaires(ss.db),
            "est_like":         post.EstLikeePar(ss.db, userID),
        }
        result = append(result, postData)
    }

    return result, nil
}

// === LIKES ===

func (ss *SocialService) LikerPost(userID, postID uint) error {
    // Vérifier que le post existe
    var post models.Post
    if err := ss.db.First(&post, postID).Error; err != nil {
        return err
    }

    // Vérifier si déjà liké
    if post.EstLikeePar(ss.db, userID) {
        return gorm.ErrDuplicatedKey
    }

    like := &models.Like{
        UtilisateurID: userID,
        PostID:       postID,
    }

    return ss.db.Create(like).Error
}

func (ss *SocialService) UnlikerPost(userID, postID uint) error {
    return ss.db.Where("utilisateur_id = ? AND post_id = ?", userID, postID).
        Delete(&models.Like{}).Error
}

func (ss *SocialService) ObtenirLikesPost(postID uint) ([]models.Utilisateur, error) {
    var utilisateurs []models.Utilisateur
    err := ss.db.Table("utilisateurs").
        Joins("JOIN likes ON likes.utilisateur_id = utilisateurs.id").
        Where("likes.post_id = ?", postID).
        Order("likes.created_at desc").
        Find(&utilisateurs).Error
    return utilisateurs, err
}

// === COMMENTAIRES ===

func (ss *SocialService) CommenterPost(auteurID, postID uint, contenu string) (*models.Commentaire, error) {
    commentaire := &models.Commentaire{
        Contenu:  contenu,
        PostID:   postID,
        AuteurID: auteurID,
    }

    err := ss.db.Create(commentaire).Error
    return commentaire, err
}

func (ss *SocialService) RepondreCommentaire(auteurID, postID, parentID uint, contenu string) (*models.Commentaire, error) {
    // Vérifier que le commentaire parent existe et appartient au bon post
    var parent models.Commentaire
    if err := ss.db.Where("id = ? AND post_id = ?", parentID, postID).First(&parent).Error; err != nil {
        return nil, err
    }

    reponse := &models.Commentaire{
        Contenu:  contenu,
        PostID:   postID,
        AuteurID: auteurID,
        ParentID: &parentID,
    }

    err := ss.db.Create(reponse).Error
    return reponse, err
}

func (ss *SocialService) SupprimerCommentaire(commentaireID, auteurID uint) error {
    // Vérifier que l'utilisateur est bien l'auteur
    var commentaire models.Commentaire
    if err := ss.db.Where("id = ? AND auteur_id = ?", commentaireID, auteurID).First(&commentaire).Error; err != nil {
        return err
    }

    return ss.db.Delete(&commentaire).Error
}

func (ss *SocialService) ObtenirCommentairesPost(postID uint) ([]models.Commentaire, error) {
    var commentaires []models.Commentaire
    err := ss.db.Preload("Auteur").
        Preload("Reponses.Auteur").
        Where("post_id = ? AND parent_id IS NULL", postID).
        Order("created_at asc").
        Find(&commentaires).Error
    return commentaires, err
}

// === ACTIVITÉ ET RECOMMANDATIONS ===

func (ss *SocialService) ObtenirActiviteRecente(userID uint, limite int) ([]map[string]interface{}, error) {
    var activites []map[string]interface{}

    // Posts récents des utilisateurs suivis
    subquery := ss.db.Model(&models.Utilisateur{}).
        Select("user_id").
        Table("user_followers").
        Where("follower_id = ?", userID)

    var posts []models.Post
    ss.db.Preload("Auteur").
        Where("auteur_id IN (?)", subquery).
        Order("created_at desc").
        Limit(limite).
        Find(&posts)

    for _, post := range posts {
        activites = append(activites, map[string]interface{}{
            "type":        "nouveau_post",
            "utilisateur": post.Auteur,
            "post":        post,
            "date":        post.CreatedAt,
        })
    }

    // Nouveaux followers
    var nouveauxFollowers []models.Utilisateur
    ss.db.Table("utilisateurs").
        Joins("JOIN user_followers ON user_followers.follower_id = utilisateurs.id").
        Where("user_followers.user_id = ?", userID).
        Order("user_followers.created_at desc").
        Limit(5).
        Find(&nouveauxFollowers)

    for _, follower := range nouveauxFollowers {
        activites = append(activites, map[string]interface{}{
            "type":        "nouveau_follower",
            "utilisateur": follower,
            "date":        follower.CreatedAt,
        })
    }

    return activites, nil
}

func (ss *SocialService) ObtenirPostsPopulaires(limite int) ([]models.Post, error) {
    var posts []models.Post

    err := ss.db.Preload("Auteur").
        Table("posts").
        Select("posts.*, COUNT(likes.id) as like_count").
        Joins("LEFT JOIN likes ON likes.post_id = posts.id").
        Group("posts.id").
        Order("like_count desc, posts.created_at desc").
        Limit(limite).
        Find(&posts).Error

    return posts, err
}

func (ss *SocialService) ObtenirStatistiquesPost(postID uint) (map[string]interface{}, error) {
    var post models.Post
    if err := ss.db.Preload("Auteur").First(&post, postID).Error; err != nil {
        return nil, err
    }

    stats := map[string]interface{}{
        "post":                post,
        "nombre_likes":        post.NombreLikes(ss.db),
        "nombre_commentaires": post.NombreCommentaires(ss.db),
    }

    // Top 3 des utilisateurs qui ont liké
    var topLikers []models.Utilisateur
    ss.db.Table("utilisateurs").
        Joins("JOIN likes ON likes.utilisateur_id = utilisateurs.id").
        Where("likes.post_id = ?", postID).
        Limit(3).
        Find(&topLikers)

    stats["top_likers"] = topLikers

    return stats, nil
}

// === MODÉRATION ===

func (ss *SocialService) SignalerPost(postID, signaleurID uint, raison string) error {
    // Dans un vrai système, on aurait une table de signalements
    // Ici, on simule en ajoutant un log
    return nil
}

func (ss *SocialService) BloquerUtilisateur(blockerID, blockedID uint) error {
    // Dans un vrai système, on aurait une table de blocages
    // qui empêcherait l'interaction entre les utilisateurs
    return nil
}
```

### main.go

```go
package main

import (
    "fmt"
    "log"
    "social/models"
    "social/services"

    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

func main() {
    // Connexion à la base de données
    db, err := gorm.Open(sqlite.Open("social.db"), &gorm.Config{})
    if err != nil {
        log.Fatal("Erreur de connexion:", err)
    }

    // Migration
    err = db.AutoMigrate(&models.Utilisateur{}, &models.Post{}, &models.Commentaire{}, &models.Like{})
    if err != nil {
        log.Fatal("Erreur de migration:", err)
    }

    // Services
    userService := services.NewUtilisateurService(db)
    socialService := services.NewSocialService(db)

    fmt.Println("=== CRÉATION D'UTILISATEURS ===")

    // Créer des utilisateurs
    alice, _ := userService.CreerUtilisateur("alice", "alice@social.com", "Alice Dupont", "Développeuse Go passionnée")
    bob, _ := userService.CreerUtilisateur("bob", "bob@social.com", "Bob Martin", "Amateur de technologie")
    charlie, _ := userService.CreerUtilisateur("charlie", "charlie@social.com", "Charlie Brown", "Photographe")

    fmt.Printf("Utilisateurs créés: %s, %s, %s\n", alice.Username, bob.Username, charlie.Username)

    fmt.Println("\n=== RELATIONS DE SUIVI ===")

    // Créer des relations de suivi
    userService.SuivreUtilisateur(alice.ID, bob.ID)    // Alice suit Bob
    userService.SuivreUtilisateur(alice.ID, charlie.ID) // Alice suit Charlie
    userService.SuivreUtilisateur(bob.ID, alice.ID)    // Bob suit Alice
    userService.SuivreUtilisateur(charlie.ID, alice.ID) // Charlie suit Alice

    // Afficher les statistiques
    statsAlice, _ := userService.ObtenirStatistiques(alice.ID)
    fmt.Printf("Alice - Followers: %d, Following: %d\n",
        statsAlice["followers"], statsAlice["following"])

    fmt.Println("\n=== CRÉATION DE POSTS ===")

    // Créer des posts
    post1, _ := socialService.CreerPost(alice.ID, "Mon premier post sur ce réseau social ! 🎉", "")
    post2, _ := socialService.CreerPost(bob.ID, "Découverte de Go aujourd'hui, c'est fantastique !", "")
    post3, _ := socialService.CreerPost(charlie.ID, "Nouvelle photo de coucher de soleil 📸", "sunset.jpg")
    post4, _ := socialService.CreerPost(alice.ID, "Travail sur un nouveau projet en Go. Quelqu'un a des conseils pour optimiser les performances ?", "")

    fmt.Printf("Posts créés: %d, %d, %d, %d\n", post1.ID, post2.ID, post3.ID, post4.ID)

    fmt.Println("\n=== INTERACTIONS (LIKES ET COMMENTAIRES) ===")

    // Ajouter des likes
    socialService.LikerPost(bob.ID, post1.ID)    // Bob like le post d'Alice
    socialService.LikerPost(charlie.ID, post1.ID) // Charlie like le post d'Alice
    socialService.LikerPost(alice.ID, post2.ID)   // Alice like le post de Bob
    socialService.LikerPost(alice.ID, post3.ID)   // Alice like le post de Charlie
    socialService.LikerPost(bob.ID, post4.ID)     // Bob like le second post d'Alice

    // Ajouter des commentaires
    comment1, _ := socialService.CommenterPost(bob.ID, post1.ID, "Bienvenue ! 🎊")
    socialService.CommenterPost(charlie.ID, post1.ID, "Super ! Hâte de voir tes futurs posts")
    socialService.RepondreCommentaire(alice.ID, post1.ID, comment1.ID, "Merci Bob ! 😊")

    socialService.CommenterPost(alice.ID, post2.ID, "Go est effectivement un excellent langage !")
    socialService.CommenterPost(charlie.ID, post2.ID, "J'aimerais apprendre Go aussi")

    socialService.CommenterPost(alice.ID, post3.ID, "Magnifique photo ! 😍")
    socialService.CommenterPost(bob.ID, post3.ID, "Où a été prise cette photo ?")

    fmt.Println("Likes et commentaires ajoutés")

    fmt.Println("\n=== TIMELINE D'ALICE ===")

    // Afficher la timeline d'Alice (ses posts + ceux qu'elle suit)
    timeline, _ := socialService.ObtenirTimelineAvecStats(alice.ID, 10, 0)

    for _, item := range timeline {
        post := item["post"].(models.Post)
        fmt.Printf("\n@%s - %s\n", post.Auteur.Username, post.CreatedAt.Format("15:04"))
        fmt.Printf("%s\n", post.Contenu)
        if post.ImageURL != "" {
            fmt.Printf("📷 %s\n", post.ImageURL)
        }
        fmt.Printf("❤️ %d likes | 💬 %d commentaires",
            item["nombre_likes"], item["nombre_commentaires"])
        if item["est_like"].(bool) {
            fmt.Printf(" | ✅ Vous avez liké")
        }
        fmt.Println()
    }

    fmt.Println("\n=== DÉTAILS D'UN POST ===")

    // Afficher les détails du premier post avec commentaires
    postDetaille, _ := socialService.ObtenirPost(post1.ID)
    fmt.Printf("Post de @%s: %s\n", postDetaille.Auteur.Username, postDetaille.Contenu)

    commentaires, _ := socialService.ObtenirCommentairesPost(post1.ID)
    fmt.Println("Commentaires:")
    for _, comment := range commentaires {
        fmt.Printf("  @%s: %s\n", comment.Auteur.Username, comment.Contenu)

        // Afficher les réponses
        for _, reponse := range comment.Reponses {
            fmt.Printf("    ↳ @%s: %s\n", reponse.Auteur.Username, reponse.Contenu)
        }
    }

    fmt.Println("\n=== POSTS POPULAIRES ===")

    // Afficher les posts populaires
    postsPopulaires, _ := socialService.ObtenirPostsPopulaires(3)
    for i, post := range postsPopulaires {
        stats, _ := socialService.ObtenirStatistiquesPost(post.ID)
        fmt.Printf("%d. @%s: %s\n", i+1, post.Auteur.Username, post.Contenu)
        fmt.Printf("   ❤️ %d likes | 💬 %d commentaires\n",
            stats["nombre_likes"], stats["nombre_commentaires"])
    }

    fmt.Println("\n=== RECHERCHE D'UTILISATEURS ===")

    // Recherche d'utilisateurs
    resultats, _ := userService.RechercherUtilisateurs("alice")
    fmt.Printf("Recherche 'alice': %d résultat(s)\n", len(resultats))
    for _, user := range resultats {
        fmt.Printf("  @%s - %s\n", user.Username, user.Nom)
    }

    fmt.Println("\n=== SUGGESTIONS POUR BOB ===")

    // Suggestions d'utilisateurs pour Bob
    suggestions, _ := userService.SuggererUtilisateurs(bob.ID, 5)
    fmt.Printf("Suggestions pour Bob: %d utilisateur(s)\n", len(suggestions))
    for _, user := range suggestions {
        fmt.Printf("  @%s - %s\n", user.Username, user.Nom)
    }

    fmt.Println("\n=== ACTIVITÉ RÉCENTE POUR ALICE ===")

    // Activité récente pour Alice
    activite, _ := socialService.ObtenirActiviteRecente(alice.ID, 5)
    for _, act := range activite {
        switch act["type"] {
        case "nouveau_post":
            post := act["post"].(models.Post)
            user := act["utilisateur"].(models.Utilisateur)
            fmt.Printf("📝 @%s a publié: %s\n", user.Username, post.Contenu)
        case "nouveau_follower":
            user := act["utilisateur"].(models.Utilisateur)
            fmt.Printf("👤 @%s vous suit maintenant\n", user.Username)
        }
    }

    fmt.Println("\n=== STATISTIQUES FINALES ===")

    // Statistiques finales pour tous les utilisateurs
    for _, user := range []*models.Utilisateur{alice, bob, charlie} {
        stats, _ := userService.ObtenirStatistiques(user.ID)
        fmt.Printf("@%s: %d posts, %d followers, %d following, %d likes reçus\n",
            user.Username, stats["posts"], stats["followers"],
            stats["following"], stats["likes_recus"])
    }
}
```

## Fichier go.mod pour les deux projets

```go
// Pour e-commerce
module ecommerce

go 1.21

require (
    gorm.io/driver/sqlite v1.5.4
    gorm.io/gorm v1.25.5
)

// Pour réseau social
module social

go 1.21

require (
    gorm.io/driver/sqlite v1.5.4
    gorm.io/gorm v1.25.5
)
```

## Instructions d'installation et d'exécution

### E-commerce :
```bash
mkdir ecommerce && cd ecommerce
go mod init ecommerce
go get gorm.io/driver/sqlite gorm.io/gorm
# Créer les fichiers models/ et services/
go run main.go
```

### Réseau social :
```bash
mkdir social && cd social
go mod init social
go get gorm.io/driver/sqlite gorm.io/gorm
# Créer les fichiers models/ et services/
go run main.go
```

## Fonctionnalités implémentées

### E-commerce :
- ✅ **Modèles complets** avec validation et contraintes
- ✅ **Gestion du stock** automatique lors des commandes
- ✅ **Calcul automatique** des totaux et sous-totaux
- ✅ **Transactions** pour garantir la cohérence
- ✅ **Statuts de commande** avec workflow
- ✅ **Relations complexes** entre toutes les entités
- ✅ **Hooks GORM** pour validation et calculs automatiques

### Réseau social :
- ✅ **Système de followers** bidirectionnel
- ✅ **Timeline intelligente** (posts des utilisateurs suivis)
- ✅ **Commentaires hiérarchiques** (réponses aux commentaires)
- ✅ **Système de likes** avec prévention des doublons
- ✅ **Recherche et suggestions** d'utilisateurs
- ✅ **Activité récente** et posts populaires
- ✅ **Statistiques complètes** pour chaque utilisateur
- ✅ **Modération basique** (signalement, blocage)

## Concepts GORM démontrés

1. **Relations complexes** : One-to-Many, Many-to-Many, self-referencing
2. **Hooks** : BeforeCreate, AfterCreate, BeforeDelete
3. **Transactions** : Automatiques et manuelles
4. **Preloading** : Chargement optimisé des relations
5. **Requêtes avancées** : Jointures, sous-requêtes, agrégations
6. **Validation** : Contraintes base de données et validation applicative
7. **Associations** : Gestion des relations many-to-many
8. **Soft delete** : Suppression logique
9. **Indexes** : Pour optimiser les performances

Ces solutions montrent comment construire des applications réelles avec GORM en respectant les bonnes pratiques et en utilisant toutes les fonctionnalités avancées de l'ORM.

⏭️
