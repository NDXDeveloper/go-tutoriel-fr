🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Projet 2 : API REST

## Introduction

Après avoir maîtrisé les concepts fondamentaux de Go et développé un outil en ligne de commande, nous allons maintenant nous attaquer à un projet plus complexe : la création d'une API REST complète. Ce projet vous permettra d'appliquer vos connaissances en programmation réseau, gestion des données, et architecture logicielle.

## Objectifs du projet

À la fin de ce projet, vous serez capable de :

- Concevoir et implémenter une API REST robuste en Go
- Gérer l'authentification et l'autorisation
- Valider et traiter les données d'entrée
- Structurer votre code selon les bonnes pratiques
- Déployer votre API en production

## Qu'est-ce qu'une API REST ?

REST (Representational State Transfer) est un style architectural pour les services web qui définit des conventions pour créer des APIs web. Une API REST utilise les méthodes HTTP standard (GET, POST, PUT, DELETE) pour effectuer des opérations sur des ressources identifiées par des URLs.

### Principes REST

**Sans état (Stateless)** : Chaque requête doit contenir toutes les informations nécessaires pour être traitée. Le serveur ne conserve pas d'état entre les requêtes.

**Interface uniforme** : Utilisation des méthodes HTTP standard et des codes de statut appropriés.

**Cacheable** : Les réponses peuvent être mises en cache pour améliorer les performances.

**Architecture en couches** : Séparation claire entre les différentes couches de l'application.

## Le projet : Gestionnaire de tâches

Nous allons développer une API pour un gestionnaire de tâches (Todo List) avec les fonctionnalités suivantes :

### Fonctionnalités principales

- **Gestion des utilisateurs** : Inscription, connexion, profils
- **Gestion des tâches** : Créer, lire, modifier, supprimer des tâches
- **Catégories** : Organiser les tâches par catégories
- **Authentification** : Système d'authentification JWT
- **Validation** : Validation des données d'entrée
- **Pagination** : Pagination des résultats pour les listes

### Technologies utilisées

- **Gorilla Mux** : Router HTTP puissant et flexible
- **GORM** : ORM pour la gestion de base de données
- **JWT-Go** : Gestion des tokens d'authentification
- **PostgreSQL** : Base de données relationnelle
- **Testify** : Framework de tests
- **Docker** : Conteneurisation pour le déploiement

## Structure des données

### Modèle User
```go
type User struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    Username  string    `json:"username" gorm:"unique;not null"`
    Email     string    `json:"email" gorm:"unique;not null"`
    Password  string    `json:"-" gorm:"not null"` // Le tag "-" cache le mot de passe
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Tasks     []Task    `json:"tasks,omitempty" gorm:"foreignKey:UserID"`
}
```

### Modèle Task
```go
type Task struct {
    ID          uint      `json:"id" gorm:"primaryKey"`
    Title       string    `json:"title" gorm:"not null"`
    Description string    `json:"description"`
    Completed   bool      `json:"completed" gorm:"default:false"`
    Priority    string    `json:"priority" gorm:"default:'medium'"`
    DueDate     *time.Time `json:"due_date,omitempty"`
    UserID      uint      `json:"user_id" gorm:"not null"`
    CategoryID  *uint     `json:"category_id,omitempty"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
    User        User      `json:"user,omitempty" gorm:"foreignKey:UserID"`
    Category    *Category `json:"category,omitempty" gorm:"foreignKey:CategoryID"`
}
```

### Modèle Category
```go
type Category struct {
    ID        uint      `json:"id" gorm:"primaryKey"`
    Name      string    `json:"name" gorm:"not null"`
    Color     string    `json:"color" gorm:"default:'#3498db'"`
    UserID    uint      `json:"user_id" gorm:"not null"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    Tasks     []Task    `json:"tasks,omitempty" gorm:"foreignKey:CategoryID"`
}
```

## Endpoints de l'API

### Authentification
- `POST /api/auth/register` : Inscription
- `POST /api/auth/login` : Connexion
- `POST /api/auth/logout` : Déconnexion
- `GET /api/auth/profile` : Profil utilisateur

### Gestion des tâches
- `GET /api/tasks` : Liste des tâches (avec pagination et filtres)
- `POST /api/tasks` : Créer une tâche
- `GET /api/tasks/{id}` : Détail d'une tâche
- `PUT /api/tasks/{id}` : Modifier une tâche
- `DELETE /api/tasks/{id}` : Supprimer une tâche
- `PATCH /api/tasks/{id}/complete` : Marquer comme terminée

### Gestion des catégories
- `GET /api/categories` : Liste des catégories
- `POST /api/categories` : Créer une catégorie
- `PUT /api/categories/{id}` : Modifier une catégorie
- `DELETE /api/categories/{id}` : Supprimer une catégorie

## Structure du projet

```
todo-api/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handlers/
│   │   ├── auth.go
│   │   ├── tasks.go
│   │   └── categories.go
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── cors.go
│   │   └── logging.go
│   ├── models/
│   │   ├── user.go
│   │   ├── task.go
│   │   └── category.go
│   ├── repository/
│   │   ├── user.go
│   │   ├── task.go
│   │   └── category.go
│   ├── service/
│   │   ├── auth.go
│   │   ├── task.go
│   │   └── category.go
│   └── utils/
│       ├── response.go
│       ├── validation.go
│       └── jwt.go
├── pkg/
│   └── database/
│       └── postgres.go
├── tests/
│   ├── handlers/
│   ├── services/
│   └── integration/
├── docs/
│   └── api.md
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── README.md
```

## Prérequis

Avant de commencer ce projet, assurez-vous d'avoir :

- Go 1.19 ou plus récent installé
- PostgreSQL installé et configuré
- Docker et Docker Compose (optionnel, pour l'environnement de développement)
- Un éditeur de code ou IDE (VS Code, GoLand, etc.)
- Connaissance des concepts REST et HTTP
- Familiarité avec SQL et les bases de données relationnelles

## Méthodologie de développement

Nous suivrons une approche de développement incrémentale :

1. **Setup initial** : Configuration de l'environnement et des dépendances
2. **Modèles de données** : Définition des structures et de la base de données
3. **Authentification** : Implémentation du système d'auth JWT
4. **Endpoints basiques** : CRUD pour les tâches et catégories
5. **Validation et sécurité** : Validation des données et sécurisation
6. **Tests** : Tests unitaires et d'intégration
7. **Documentation** : Documentation de l'API
8. **Déploiement** : Containerisation et déploiement

## Bonnes pratiques appliquées

- **Separation of Concerns** : Séparation claire entre les couches
- **Dependency Injection** : Injection de dépendances pour la testabilité
- **Error Handling** : Gestion d'erreurs robuste et cohérente
- **Logging** : Logging structuré pour le monitoring
- **Validation** : Validation rigoureuse des données d'entrée
- **Security** : Authentification, autorisation et protection CSRF
- **Testing** : Tests unitaires et d'intégration complets

---

Dans les sections suivantes, nous allons détailler chaque aspect de la construction de cette API, en commençant par l'architecture générale et la mise en place de l'environnement de développement.

⏭️
