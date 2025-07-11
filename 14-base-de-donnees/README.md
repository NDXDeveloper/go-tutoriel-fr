🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14. Base de données

## Introduction

La gestion des bases de données est un aspect crucial du développement d'applications modernes. Go offre un excellent support pour travailler avec différents types de bases de données grâce à sa bibliothèque standard robuste et à un écosystème riche de packages tiers.

## Pourquoi les bases de données en Go ?

Go se distingue par plusieurs avantages pour le développement d'applications orientées données :

- **Performance native** : Les applications Go compilées offrent d'excellentes performances pour les opérations de base de données
- **Concurrence intégrée** : Les goroutines permettent de gérer efficacement les connexions multiples
- **Typage fort** : La vérification des types à la compilation réduit les erreurs liées aux requêtes
- **Écosystème mature** : De nombreux drivers et ORMs disponibles pour tous les types de bases de données

## Types de bases de données supportées

### Bases de données relationnelles (SQL)
- **PostgreSQL** : Base de données open-source robuste et riche en fonctionnalités
- **MySQL/MariaDB** : Très populaire pour les applications web
- **SQLite** : Idéale pour les applications embarquées et les tests
- **SQL Server** : Solution Microsoft pour l'entreprise
- **Oracle** : Base de données enterprise

### Bases de données NoSQL
- **MongoDB** : Base de données orientée documents
- **Redis** : Store clé-valeur en mémoire
- **Cassandra** : Base de données distribuée
- **CouchDB** : Base de données orientée documents

## Approches de développement

### 1. SQL brut avec database/sql
L'approche la plus directe utilisant la bibliothèque standard de Go. Offre un contrôle total mais demande plus de code.

**Avantages :**
- Contrôle total sur les requêtes
- Performance optimale
- Pas de dépendances externes
- Apprentissage des concepts SQL

**Inconvénients :**
- Plus de code à écrire
- Gestion manuelle des migrations
- Pas de validation automatique

### 2. Query Builders
Des outils comme Squirrel qui génèrent du SQL de manière programmatique.

**Avantages :**
- Requêtes typées
- Réutilisabilité
- Moins d'erreurs de syntaxe SQL

**Inconvénients :**
- Courbe d'apprentissage
- Abstraction supplémentaire

### 3. ORM (Object-Relational Mapping)
Des solutions comme GORM qui mappent automatiquement les structs Go aux tables de base de données.

**Avantages :**
- Développement rapide
- Migrations automatiques
- Validation intégrée
- Associations automatiques

**Inconvénients :**
- Moins de contrôle sur les requêtes
- Possibles problèmes de performance
- Courbe d'apprentissage

## Concepts clés à maîtriser

### Connection Pooling
Go gère automatiquement un pool de connexions pour optimiser les performances :

```go
db.SetMaxOpenConns(25)    // Nombre maximum de connexions ouvertes
db.SetMaxIdleConns(25)    // Nombre maximum de connexions inactives
db.SetConnMaxLifetime(5 * time.Minute)  // Durée de vie maximum d'une connexion
```

### Gestion des transactions
Les transactions garantissent la cohérence des données :

```go
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback()

// Opérations en transaction
// ...

return tx.Commit()
```

### Sécurité
- **Requêtes préparées** : Protection contre les injections SQL
- **Validation des données** : Validation côté application
- **Chiffrement** : Connexions sécurisées (TLS/SSL)

## Bonnes pratiques

### 1. Séparation des couches
Structurez votre application en couches distinctes :
- **Repository** : Accès aux données
- **Service** : Logique métier
- **Handler** : Interface web/API

### 2. Gestion des erreurs
Toujours vérifier et gérer les erreurs de base de données :

```go
if err != nil {
    if err == sql.ErrNoRows {
        // Gérer le cas "aucun résultat"
    } else {
        // Gérer les autres erreurs
    }
}
```

### 3. Context pour les timeouts
Utilisez le package context pour gérer les timeouts :

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

rows, err := db.QueryContext(ctx, query, args...)
```

### 4. Tests
Testez votre code de base de données avec :
- **Bases de données de test** : SQLite ou conteneurs Docker
- **Mocks** : Interfaces pour les tests unitaires
- **Fixtures** : Données de test cohérentes

## Structure recommandée

Pour une application Go utilisant des bases de données, voici une structure recommandée :

```
myapp/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   ├── database/
│   │   ├── connection.go
│   │   └── migrations/
│   ├── models/
│   ├── repositories/
│   ├── services/
│   └── handlers/
├── pkg/
├── migrations/
└── tests/
```

## Prérequis pour cette section

Avant de commencer, assurez-vous de maîtriser :
- Les structs et interfaces Go
- La gestion des erreurs
- Le package context
- Les goroutines et channels (pour la concurrence)
- Les tests unitaires

## Ce que vous apprendrez

Dans les prochaines sections, vous découvrirez :

1. **SQL avec database/sql** : Utilisation de la bibliothèque standard pour les opérations CRUD
2. **ORM avec GORM** : Développement rapide avec un ORM complet
3. **Migrations** : Gestion de l'évolution du schéma de base de données
4. **Connexions et pools** : Optimisation des performances et de la scalabilité

Chaque approche sera illustrée avec des exemples pratiques et des cas d'usage réels pour vous permettre de choisir la meilleure solution selon vos besoins.

---

*Prêt à plonger dans le monde des bases de données avec Go ? Commençons par explorer l'approche fondamentale avec le package database/sql.*

⏭️
