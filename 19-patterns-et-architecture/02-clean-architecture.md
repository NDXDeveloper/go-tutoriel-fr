🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19-2 : Clean architecture

## Qu'est-ce que la Clean Architecture ?

La Clean Architecture est une façon d'organiser votre code pour qu'il soit **facile à tester**, **facile à maintenir** et **indépendant des détails techniques**.

### Analogie simple
Imaginez une maison :
- **Le cœur** (les règles métier) : C'est ce que fait votre application
- **Les murs** (les interfaces) : Ils séparent les pièces
- **Les branchements** (la base de données, l'API) : Ce sont les détails techniques

Dans une maison bien conçue, vous pouvez changer la plomberie sans démolir toute la maison !

## Les principes de base

### 1. Séparation des responsabilités
Chaque partie de votre code a un rôle précis :
- **Entities** : Les règles métier importantes
- **Use Cases** : Ce que fait votre application
- **Interface Adapters** : La traduction entre les couches
- **Frameworks & Drivers** : Les détails techniques

### 2. Règle de dépendance
**Important** : Les couches internes ne doivent JAMAIS dépendre des couches externes.

```
External ─→ Interface ─→ Use Cases ─→ Entities
        (dépendance)
```

## Structure d'un projet Clean Architecture

Voici comment organiser votre projet Go :

```
myapp/
├── cmd/
│   └── api/
│       └── main.go           # Point d'entrée
├── internal/
│   ├── entity/              # Entités métier
│   │   └── user.go
│   ├── usecase/             # Cas d'usage
│   │   └── user_usecase.go
│   ├── adapter/             # Adaptateurs
│   │   ├── repository/      # Accès aux données
│   │   │   └── user_repo.go
│   │   └── handler/         # Handlers HTTP
│   │       └── user_handler.go
│   └── infrastructure/      # Détails techniques
│       ├── database/
│       │   └── postgres.go
│       └── server/
│           └── server.go
├── pkg/                     # Packages réutilisables
└── go.mod
```

## Exemple concret : Système de gestion d'utilisateurs

### Étape 1 : Entities (Entités)

Les entités représentent les concepts métier de votre application.

```go
// internal/entity/user.go
package entity

import (
    "errors"
    "time"
    "strings"
)

// User représente un utilisateur dans notre système
type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Règles métier : validation
func (u *User) Validate() error {
    if strings.TrimSpace(u.Name) == "" {
        return errors.New("le nom ne peut pas être vide")
    }

    if !strings.Contains(u.Email, "@") {
        return errors.New("email invalide")
    }

    return nil
}

// Règles métier : mise à jour
func (u *User) UpdateName(newName string) error {
    if strings.TrimSpace(newName) == "" {
        return errors.New("le nom ne peut pas être vide")
    }

    u.Name = newName
    u.UpdatedAt = time.Now()
    return nil
}

// Règles métier : création
func NewUser(name, email string) (*User, error) {
    user := &User{
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    if err := user.Validate(); err != nil {
        return nil, err
    }

    return user, nil
}
```

### Étape 2 : Use Cases (Cas d'usage)

Les use cases définissent ce que votre application peut faire.

```go
// internal/usecase/user_usecase.go
package usecase

import (
    "context"
    "fmt"
    "myapp/internal/entity"
)

// Interface pour accéder aux données (définie dans la couche use case)
type UserRepository interface {
    Create(ctx context.Context, user *entity.User) error
    GetByID(ctx context.Context, id int) (*entity.User, error)
    GetByEmail(ctx context.Context, email string) (*entity.User, error)
    Update(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id int) error
}

// Use case pour gérer les utilisateurs
type UserUseCase struct {
    userRepo UserRepository
}

func NewUserUseCase(userRepo UserRepository) *UserUseCase {
    return &UserUseCase{
        userRepo: userRepo,
    }
}

// Cas d'usage : créer un utilisateur
func (uc *UserUseCase) CreateUser(ctx context.Context, name, email string) (*entity.User, error) {
    // Vérifier si l'email existe déjà
    existingUser, err := uc.userRepo.GetByEmail(ctx, email)
    if err == nil && existingUser != nil {
        return nil, fmt.Errorf("un utilisateur avec cet email existe déjà")
    }

    // Créer le nouvel utilisateur (règles métier)
    user, err := entity.NewUser(name, email)
    if err != nil {
        return nil, fmt.Errorf("erreur de création : %w", err)
    }

    // Sauvegarder
    if err := uc.userRepo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("erreur de sauvegarde : %w", err)
    }

    return user, nil
}

// Cas d'usage : récupérer un utilisateur
func (uc *UserUseCase) GetUser(ctx context.Context, id int) (*entity.User, error) {
    if id <= 0 {
        return nil, fmt.Errorf("ID invalide")
    }

    user, err := uc.userRepo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("utilisateur non trouvé : %w", err)
    }

    return user, nil
}

// Cas d'usage : mettre à jour un utilisateur
func (uc *UserUseCase) UpdateUser(ctx context.Context, id int, name string) (*entity.User, error) {
    // Récupérer l'utilisateur existant
    user, err := uc.userRepo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("utilisateur non trouvé : %w", err)
    }

    // Appliquer les règles métier
    if err := user.UpdateName(name); err != nil {
        return nil, fmt.Errorf("erreur de mise à jour : %w", err)
    }

    // Sauvegarder
    if err := uc.userRepo.Update(ctx, user); err != nil {
        return nil, fmt.Errorf("erreur de sauvegarde : %w", err)
    }

    return user, nil
}

// Cas d'usage : supprimer un utilisateur
func (uc *UserUseCase) DeleteUser(ctx context.Context, id int) error {
    if id <= 0 {
        return fmt.Errorf("ID invalide")
    }

    // Vérifier que l'utilisateur existe
    _, err := uc.userRepo.GetByID(ctx, id)
    if err != nil {
        return fmt.Errorf("utilisateur non trouvé : %w", err)
    }

    // Supprimer
    if err := uc.userRepo.Delete(ctx, id); err != nil {
        return fmt.Errorf("erreur de suppression : %w", err)
    }

    return nil
}
```

### Étape 3 : Interface Adapters (Adaptateurs)

#### Repository (Accès aux données)

```go
// internal/adapter/repository/user_repo.go
package repository

import (
    "context"
    "database/sql"
    "fmt"
    "myapp/internal/entity"
    "time"
)

// Implémentation concrète du repository
type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(ctx context.Context, user *entity.User) error {
    query := `
        INSERT INTO users (name, email, created_at, updated_at)
        VALUES ($1, $2, $3, $4)
        RETURNING id
    `

    err := r.db.QueryRowContext(
        ctx, query,
        user.Name, user.Email, user.CreatedAt, user.UpdatedAt,
    ).Scan(&user.ID)

    if err != nil {
        return fmt.Errorf("erreur lors de la création : %w", err)
    }

    return nil
}

func (r *UserRepository) GetByID(ctx context.Context, id int) (*entity.User, error) {
    query := `
        SELECT id, name, email, created_at, updated_at
        FROM users
        WHERE id = $1
    `

    user := &entity.User{}
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID, &user.Name, &user.Email, &user.CreatedAt, &user.UpdatedAt,
    )

    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("utilisateur non trouvé")
        }
        return nil, fmt.Errorf("erreur de récupération : %w", err)
    }

    return user, nil
}

func (r *UserRepository) GetByEmail(ctx context.Context, email string) (*entity.User, error) {
    query := `
        SELECT id, name, email, created_at, updated_at
        FROM users
        WHERE email = $1
    `

    user := &entity.User{}
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID, &user.Name, &user.Email, &user.CreatedAt, &user.UpdatedAt,
    )

    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("utilisateur non trouvé")
        }
        return nil, fmt.Errorf("erreur de récupération : %w", err)
    }

    return user, nil
}

func (r *UserRepository) Update(ctx context.Context, user *entity.User) error {
    query := `
        UPDATE users
        SET name = $1, email = $2, updated_at = $3
        WHERE id = $4
    `

    _, err := r.db.ExecContext(
        ctx, query,
        user.Name, user.Email, user.UpdatedAt, user.ID,
    )

    if err != nil {
        return fmt.Errorf("erreur de mise à jour : %w", err)
    }

    return nil
}

func (r *UserRepository) Delete(ctx context.Context, id int) error {
    query := `DELETE FROM users WHERE id = $1`

    _, err := r.db.ExecContext(ctx, query, id)
    if err != nil {
        return fmt.Errorf("erreur de suppression : %w", err)
    }

    return nil
}
```

#### Handler HTTP

```go
// internal/adapter/handler/user_handler.go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    "myapp/internal/usecase"

    "github.com/gorilla/mux"
)

// Structures pour les requêtes/réponses
type CreateUserRequest struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

type UpdateUserRequest struct {
    Name string `json:"name"`
}

type UserResponse struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email"`
    CreatedAt string `json:"created_at"`
    UpdatedAt string `json:"updated_at"`
}

type ErrorResponse struct {
    Error string `json:"error"`
}

// Handler pour les utilisateurs
type UserHandler struct {
    userUseCase *usecase.UserUseCase
}

func NewUserHandler(userUseCase *usecase.UserUseCase) *UserHandler {
    return &UserHandler{
        userUseCase: userUseCase,
    }
}

// POST /users
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.sendError(w, "Format JSON invalide", http.StatusBadRequest)
        return
    }

    user, err := h.userUseCase.CreateUser(r.Context(), req.Name, req.Email)
    if err != nil {
        h.sendError(w, err.Error(), http.StatusBadRequest)
        return
    }

    response := UserResponse{
        ID:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Format("2006-01-02T15:04:05Z"),
        UpdatedAt: user.UpdatedAt.Format("2006-01-02T15:04:05Z"),
    }

    h.sendJSON(w, response, http.StatusCreated)
}

// GET /users/{id}
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.sendError(w, "ID invalide", http.StatusBadRequest)
        return
    }

    user, err := h.userUseCase.GetUser(r.Context(), id)
    if err != nil {
        h.sendError(w, err.Error(), http.StatusNotFound)
        return
    }

    response := UserResponse{
        ID:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Format("2006-01-02T15:04:05Z"),
        UpdatedAt: user.UpdatedAt.Format("2006-01-02T15:04:05Z"),
    }

    h.sendJSON(w, response, http.StatusOK)
}

// PUT /users/{id}
func (h *UserHandler) UpdateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.sendError(w, "ID invalide", http.StatusBadRequest)
        return
    }

    var req UpdateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.sendError(w, "Format JSON invalide", http.StatusBadRequest)
        return
    }

    user, err := h.userUseCase.UpdateUser(r.Context(), id, req.Name)
    if err != nil {
        h.sendError(w, err.Error(), http.StatusBadRequest)
        return
    }

    response := UserResponse{
        ID:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Format("2006-01-02T15:04:05Z"),
        UpdatedAt: user.UpdatedAt.Format("2006-01-02T15:04:05Z"),
    }

    h.sendJSON(w, response, http.StatusOK)
}

// DELETE /users/{id}
func (h *UserHandler) DeleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        h.sendError(w, "ID invalide", http.StatusBadRequest)
        return
    }

    if err := h.userUseCase.DeleteUser(r.Context(), id); err != nil {
        h.sendError(w, err.Error(), http.StatusNotFound)
        return
    }

    w.WriteHeader(http.StatusNoContent)
}

// Méthodes utilitaires
func (h *UserHandler) sendJSON(w http.ResponseWriter, data interface{}, status int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func (h *UserHandler) sendError(w http.ResponseWriter, message string, status int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(ErrorResponse{Error: message})
}
```

### Étape 4 : Infrastructure (Détails techniques)

#### Configuration de la base de données

```go
// internal/infrastructure/database/postgres.go
package database

import (
    "database/sql"
    "fmt"
    _ "github.com/lib/pq"
)

type Config struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
}

func Connect(cfg Config) (*sql.DB, error) {
    dsn := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=disable",
        cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName,
    )

    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, fmt.Errorf("erreur de connexion : %w", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("impossible de contacter la base : %w", err)
    }

    return db, nil
}

func CreateUserTable(db *sql.DB) error {
    query := `
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            email VARCHAR(100) UNIQUE NOT NULL,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    `

    _, err := db.Exec(query)
    if err != nil {
        return fmt.Errorf("erreur création table : %w", err)
    }

    return nil
}
```

#### Serveur HTTP

```go
// internal/infrastructure/server/server.go
package server

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "time"

    "myapp/internal/adapter/handler"
    "github.com/gorilla/mux"
)

type Server struct {
    router      *mux.Router
    userHandler *handler.UserHandler
}

func NewServer(userHandler *handler.UserHandler) *Server {
    return &Server{
        router:      mux.NewRouter(),
        userHandler: userHandler,
    }
}

func (s *Server) SetupRoutes() {
    // Routes pour les utilisateurs
    s.router.HandleFunc("/users", s.userHandler.CreateUser).Methods("POST")
    s.router.HandleFunc("/users/{id}", s.userHandler.GetUser).Methods("GET")
    s.router.HandleFunc("/users/{id}", s.userHandler.UpdateUser).Methods("PUT")
    s.router.HandleFunc("/users/{id}", s.userHandler.DeleteUser).Methods("DELETE")

    // Middleware
    s.router.Use(s.loggingMiddleware)
    s.router.Use(s.corsMiddleware)
}

func (s *Server) Start(port int) error {
    s.SetupRoutes()

    srv := &http.Server{
        Addr:         fmt.Sprintf(":%d", port),
        Handler:      s.router,
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    log.Printf("Serveur démarré sur le port %d", port)
    return srv.ListenAndServe()
}

// Middleware de logging
func (s *Server) loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Middleware CORS
func (s *Server) corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### Étape 5 : Point d'entrée principal

```go
// cmd/api/main.go
package main

import (
    "log"
    "myapp/internal/adapter/handler"
    "myapp/internal/adapter/repository"
    "myapp/internal/infrastructure/database"
    "myapp/internal/infrastructure/server"
    "myapp/internal/usecase"
)

func main() {
    // Configuration de la base de données
    dbConfig := database.Config{
        Host:     "localhost",
        Port:     5432,
        User:     "postgres",
        Password: "password",
        DBName:   "myapp",
    }

    // Connexion à la base de données
    db, err := database.Connect(dbConfig)
    if err != nil {
        log.Fatal("Erreur de connexion à la base :", err)
    }
    defer db.Close()

    // Création de la table
    if err := database.CreateUserTable(db); err != nil {
        log.Fatal("Erreur de création de table :", err)
    }

    // Création des couches (injection de dépendances)
    userRepo := repository.NewUserRepository(db)
    userUseCase := usecase.NewUserUseCase(userRepo)
    userHandler := handler.NewUserHandler(userUseCase)

    // Création et démarrage du serveur
    srv := server.NewServer(userHandler)

    log.Println("Démarrage de l'application...")
    if err := srv.Start(8080); err != nil {
        log.Fatal("Erreur du serveur :", err)
    }
}
```

## Tests avec Clean Architecture

L'un des grands avantages de Clean Architecture est la facilité des tests.

### Test du Use Case

```go
// internal/usecase/user_usecase_test.go
package usecase

import (
    "context"
    "testing"
    "myapp/internal/entity"
)

// Mock repository pour les tests
type MockUserRepository struct {
    users map[int]*entity.User
    nextID int
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users: make(map[int]*entity.User),
        nextID: 1,
    }
}

func (m *MockUserRepository) Create(ctx context.Context, user *entity.User) error {
    user.ID = m.nextID
    m.users[m.nextID] = user
    m.nextID++
    return nil
}

func (m *MockUserRepository) GetByID(ctx context.Context, id int) (*entity.User, error) {
    user, exists := m.users[id]
    if !exists {
        return nil, errors.New("utilisateur non trouvé")
    }
    return user, nil
}

func (m *MockUserRepository) GetByEmail(ctx context.Context, email string) (*entity.User, error) {
    for _, user := range m.users {
        if user.Email == email {
            return user, nil
        }
    }
    return nil, errors.New("utilisateur non trouvé")
}

func (m *MockUserRepository) Update(ctx context.Context, user *entity.User) error {
    m.users[user.ID] = user
    return nil
}

func (m *MockUserRepository) Delete(ctx context.Context, id int) error {
    delete(m.users, id)
    return nil
}

// Test de création d'utilisateur
func TestCreateUser(t *testing.T) {
    // Arrange
    repo := NewMockUserRepository()
    useCase := NewUserUseCase(repo)

    // Act
    user, err := useCase.CreateUser(context.Background(), "John Doe", "john@example.com")

    // Assert
    if err != nil {
        t.Errorf("Erreur inattendue : %v", err)
    }

    if user.Name != "John Doe" {
        t.Errorf("Nom attendu : John Doe, reçu : %s", user.Name)
    }

    if user.Email != "john@example.com" {
        t.Errorf("Email attendu : john@example.com, reçu : %s", user.Email)
    }
}

// Test de création d'utilisateur avec email dupliqué
func TestCreateUserDuplicateEmail(t *testing.T) {
    // Arrange
    repo := NewMockUserRepository()
    useCase := NewUserUseCase(repo)

    // Créer un premier utilisateur
    useCase.CreateUser(context.Background(), "John Doe", "john@example.com")

    // Act : essayer de créer un deuxième utilisateur avec le même email
    _, err := useCase.CreateUser(context.Background(), "Jane Doe", "john@example.com")

    // Assert
    if err == nil {
        t.Error("Une erreur était attendue pour un email dupliqué")
    }
}
```

## Avantages de Clean Architecture

### 1. **Facilité de test**
- Chaque couche peut être testée isolément
- Les mocks sont simples à créer
- Les tests sont rapides (pas de base de données)

### 2. **Flexibilité**
- Changer de base de données sans impacter le métier
- Changer d'API (REST vers GraphQL) facilement
- Ajouter de nouvelles fonctionnalités sans casser l'existant

### 3. **Maintenabilité**
- Code organisé et prévisible
- Séparation claire des responsabilités
- Facile à comprendre pour les nouveaux développeurs

### 4. **Indépendance**
- Le métier ne dépend pas de la technologie
- Possibilité de développer en parallèle
- Réutilisation du code métier

## Bonnes pratiques

### 1. **Définir les interfaces dans les couches internes**
```go
// ✅ Bon : Interface définie dans le use case
type UserRepository interface {
    Create(ctx context.Context, user *entity.User) error
}

// ❌ Éviter : Interface définie dans l'infrastructure
```

### 2. **Garder les entités simples**
```go
// ✅ Bon : Entité avec règles métier
type User struct {
    ID   int
    Name string
}

func (u *User) Validate() error {
    // règles métier
}

// ❌ Éviter : Entité avec dépendances externes
type User struct {
    ID   int
    Name string
    db   *sql.DB // ❌ Pas de dépendance technique
}
```

### 3. **Utiliser des contextes**
```go
// ✅ Bon : Toujours passer un context
func (uc *UserUseCase) CreateUser(ctx context.Context, name string) error {
    return uc.repo.Create(ctx, user)
}
```

### 4. **Gérer les erreurs proprement**
```go
// ✅ Bon : Erreurs spécifiques
func (uc *UserUseCase) GetUser(ctx context.Context, id int) (*entity.User, error) {
    user, err := uc.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("utilisateur non trouvé : %w", err)
    }
    return user, nil
}
```

## Résumé

Clean Architecture en Go vous permet de :

1. **Séparer les responsabilités** : Chaque couche a un rôle précis
2. **Faciliter les tests** : Mocks simples et tests rapides
3. **Rendre le code flexible** : Changements faciles sans impact
4. **Améliorer la maintenabilité** : Code organisé et prévisible

**Points clés à retenir** :
- Les entités contiennent les règles métier
- Les use cases orchestrent les actions
- Les interfaces sont définies dans les couches internes
- L'infrastructure est remplaçable
- Les tests sont simples grâce aux mocks

Cette architecture peut sembler complexe au début, mais elle devient très naturelle avec la pratique et vous fera gagner énormément de temps sur le long terme !

⏭️
