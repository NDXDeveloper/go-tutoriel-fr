🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16-2 : Middleware et authentification

## Introduction

Dans la section précédente, nous avons créé une API fonctionnelle avec les opérations CRUD de base. Cependant, notre API présente un problème majeur : elle n'est pas sécurisée ! N'importe qui peut accéder aux données et les modifier.

Dans cette section, nous allons apprendre à sécuriser notre API en implémentant :
- **L'authentification JWT** pour identifier les utilisateurs
- **Les middlewares** pour gérer la sécurité et les fonctionnalités transversales
- **La gestion des utilisateurs** avec inscription et connexion

## Qu'est-ce qu'un middleware ?

Un **middleware** est une fonction qui s'exécute entre la réception d'une requête HTTP et l'envoi de la réponse. Il permet d'ajouter des fonctionnalités transversales comme :

- **Authentification** : Vérifier si l'utilisateur est connecté
- **Logging** : Enregistrer les requêtes
- **CORS** : Gérer les requêtes cross-origin
- **Rate limiting** : Limiter le nombre de requêtes

### Comment fonctionne un middleware ?

```
Requête → Middleware 1 → Middleware 2 → Handler → Réponse
            ↓             ↓            ↓
         Logging      Authentification  Logique métier
```

## Qu'est-ce que JWT ?

**JWT (JSON Web Token)** est un standard pour transmettre des informations de manière sécurisée entre parties. Un token JWT contient :

- **Header** : Type de token et algorithme de signature
- **Payload** : Données (claims) comme l'ID utilisateur
- **Signature** : Signature cryptographique pour vérifier l'intégrité

### Structure d'un JWT
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
        ↑ Header                    ↑ Payload                                    ↑ Signature
```

## Installation des dépendances

Ajoutons les packages nécessaires pour l'authentification :

```bash
go get github.com/golang-jwt/jwt/v5
go get golang.org/x/crypto/bcrypt
```

## Mise à jour des modèles

### internal/models/user.go

Ajoutons des méthodes utiles pour la gestion des mots de passe :

```go
package models

import (
    "time"
    "golang.org/x/crypto/bcrypt"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `json:"id" gorm:"primaryKey"`
    Username  string         `json:"username" gorm:"unique;not null"`
    Email     string         `json:"email" gorm:"unique;not null"`
    Password  string         `json:"-" gorm:"not null"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `json:"-" gorm:"index"`
    Tasks     []Task         `json:"tasks,omitempty" gorm:"foreignKey:UserID"`
}

func (User) TableName() string {
    return "users"
}

// HashPassword hashe le mot de passe avant sauvegarde
func (u *User) HashPassword() error {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    u.Password = string(hashedPassword)
    return nil
}

// CheckPassword vérifie si le mot de passe fourni correspond au hash
func (u *User) CheckPassword(password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(u.Password), []byte(password))
    return err == nil
}

// BeforeCreate hook GORM pour hasher le mot de passe avant création
func (u *User) BeforeCreate(tx *gorm.DB) error {
    return u.HashPassword()
}
```

### DTOs (Data Transfer Objects)

Créons des structures pour les requêtes d'authentification dans `internal/models/auth.go` :

```go
package models

type RegisterRequest struct {
    Username string `json:"username" validate:"required,min=3,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=6"`
}

type LoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}

type AuthResponse struct {
    User  User   `json:"user"`
    Token string `json:"token"`
}
```

## Utilitaires JWT

Créons les fonctions pour gérer les tokens JWT dans `internal/utils/jwt.go` :

```go
package utils

import (
    "errors"
    "os"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

var jwtSecret = []byte(getJWTSecret())

func getJWTSecret() string {
    secret := os.Getenv("JWT_SECRET")
    if secret == "" {
        // En production, ceci devrait générer une erreur !
        return "ma-cle-secrete-super-sure"
    }
    return secret
}

// GenerateToken génère un token JWT pour un utilisateur
func GenerateToken(userID uint, email string) (string, error) {
    // Définir les claims (revendications)
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)), // 24h
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "todo-api",
        },
    }

    // Créer le token
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

    // Signer le token
    tokenString, err := token.SignedString(jwtSecret)
    if err != nil {
        return "", err
    }

    return tokenString, nil
}

// ValidateToken valide un token JWT et retourne les claims
func ValidateToken(tokenString string) (*Claims, error) {
    // Parser le token
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Vérifier la méthode de signature
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, errors.New("méthode de signature inattendue")
        }
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    // Vérifier si le token est valide
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, errors.New("token invalide")
}
```

## Repository et Service pour les utilisateurs

### internal/repository/user_repository.go

```go
package repository

import (
    "gorm.io/gorm"
    "todo-api/internal/models"
)

type UserRepository interface {
    Create(user *models.User) error
    GetByEmail(email string) (*models.User, error)
    GetByID(id uint) (*models.User, error)
    GetByUsername(username string) (*models.User, error)
}

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) UserRepository {
    return &userRepository{db: db}
}

func (r *userRepository) Create(user *models.User) error {
    return r.db.Create(user).Error
}

func (r *userRepository) GetByEmail(email string) (*models.User, error) {
    var user models.User
    err := r.db.Where("email = ?", email).First(&user).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) GetByID(id uint) (*models.User, error) {
    var user models.User
    err := r.db.First(&user, id).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) GetByUsername(username string) (*models.User, error) {
    var user models.User
    err := r.db.Where("username = ?", username).First(&user).Error
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

### internal/service/auth_service.go

```go
package service

import (
    "errors"
    "todo-api/internal/models"
    "todo-api/internal/repository"
    "todo-api/internal/utils"
)

type AuthService interface {
    Register(req *models.RegisterRequest) (*models.AuthResponse, error)
    Login(req *models.LoginRequest) (*models.AuthResponse, error)
}

type authService struct {
    userRepo repository.UserRepository
}

func NewAuthService(userRepo repository.UserRepository) AuthService {
    return &authService{userRepo: userRepo}
}

func (s *authService) Register(req *models.RegisterRequest) (*models.AuthResponse, error) {
    // Vérifier si l'email existe déjà
    existingUser, _ := s.userRepo.GetByEmail(req.Email)
    if existingUser != nil {
        return nil, errors.New("cet email est déjà utilisé")
    }

    // Vérifier si le nom d'utilisateur existe déjà
    existingUsername, _ := s.userRepo.GetByUsername(req.Username)
    if existingUsername != nil {
        return nil, errors.New("ce nom d'utilisateur est déjà pris")
    }

    // Créer le nouvel utilisateur
    user := &models.User{
        Username: req.Username,
        Email:    req.Email,
        Password: req.Password, // Sera hashé automatiquement par BeforeCreate
    }

    if err := s.userRepo.Create(user); err != nil {
        return nil, errors.New("erreur lors de la création de l'utilisateur")
    }

    // Générer le token JWT
    token, err := utils.GenerateToken(user.ID, user.Email)
    if err != nil {
        return nil, errors.New("erreur lors de la génération du token")
    }

    return &models.AuthResponse{
        User:  *user,
        Token: token,
    }, nil
}

func (s *authService) Login(req *models.LoginRequest) (*models.AuthResponse, error) {
    // Récupérer l'utilisateur par email
    user, err := s.userRepo.GetByEmail(req.Email)
    if err != nil {
        return nil, errors.New("email ou mot de passe incorrect")
    }

    // Vérifier le mot de passe
    if !user.CheckPassword(req.Password) {
        return nil, errors.New("email ou mot de passe incorrect")
    }

    // Générer le token JWT
    token, err := utils.GenerateToken(user.ID, user.Email)
    if err != nil {
        return nil, errors.New("erreur lors de la génération du token")
    }

    return &models.AuthResponse{
        User:  *user,
        Token: token,
    }, nil
}
```

## Création des middlewares

### internal/middleware/auth_middleware.go

```go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "todo-api/internal/utils"
)

type contextKey string

const UserContextKey contextKey = "user"

// AuthMiddleware vérifie l'authentification JWT
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Récupérer le header Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            utils.WriteError(w, http.StatusUnauthorized, "Token d'autorisation manquant")
            return
        }

        // Vérifier le format "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            utils.WriteError(w, http.StatusUnauthorized, "Format du token invalide")
            return
        }

        tokenString := parts[1]

        // Valider le token
        claims, err := utils.ValidateToken(tokenString)
        if err != nil {
            utils.WriteError(w, http.StatusUnauthorized, "Token invalide")
            return
        }

        // Ajouter les informations utilisateur au contexte
        ctx := context.WithValue(r.Context(), UserContextKey, claims)

        // Passer au handler suivant
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// GetUserFromContext récupère les informations utilisateur du contexte
func GetUserFromContext(ctx context.Context) (*utils.Claims, bool) {
    claims, ok := ctx.Value(UserContextKey).(*utils.Claims)
    return claims, ok
}
```

### internal/middleware/cors_middleware.go

```go
package middleware

import (
    "net/http"
)

// CORSMiddleware gère les en-têtes CORS
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Définir les en-têtes CORS
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        // Gérer les requêtes preflight OPTIONS
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        // Passer au handler suivant
        next.ServeHTTP(w, r)
    })
}
```

### internal/middleware/logging_middleware.go

```go
package middleware

import (
    "log"
    "net/http"
    "time"
)

// LoggingMiddleware enregistre les requêtes HTTP
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Créer un ResponseWriter personnalisé pour capturer le status code
        wrappedWriter := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        // Appeler le handler suivant
        next.ServeHTTP(wrappedWriter, r)

        // Enregistrer la requête
        duration := time.Since(start)
        log.Printf(
            "%s %s %d %v",
            r.Method,
            r.URL.Path,
            wrappedWriter.statusCode,
            duration,
        )
    })
}

// responseWriter wrapper pour capturer le status code
type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

## Handlers d'authentification

### internal/handlers/auth_handler.go

```go
package handlers

import (
    "encoding/json"
    "net/http"

    "todo-api/internal/models"
    "todo-api/internal/service"
    "todo-api/internal/utils"
)

type AuthHandler struct {
    authService service.AuthService
}

func NewAuthHandler(authService service.AuthService) *AuthHandler {
    return &AuthHandler{authService: authService}
}

// Register gère l'inscription des utilisateurs
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req models.RegisterRequest

    // Décoder la requête JSON
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Validation basique
    if req.Username == "" || req.Email == "" || req.Password == "" {
        utils.WriteError(w, http.StatusBadRequest, "Tous les champs sont requis")
        return
    }

    if len(req.Password) < 6 {
        utils.WriteError(w, http.StatusBadRequest, "Le mot de passe doit contenir au moins 6 caractères")
        return
    }

    // Appeler le service d'authentification
    response, err := h.authService.Register(&req)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, response)
}

// Login gère la connexion des utilisateurs
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req models.LoginRequest

    // Décoder la requête JSON
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Validation basique
    if req.Email == "" || req.Password == "" {
        utils.WriteError(w, http.StatusBadRequest, "Email et mot de passe requis")
        return
    }

    // Appeler le service d'authentification
    response, err := h.authService.Login(&req)
    if err != nil {
        utils.WriteError(w, http.StatusUnauthorized, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, response)
}

// Profile retourne le profil de l'utilisateur connecté
func (h *AuthHandler) Profile(w http.ResponseWriter, r *http.Request) {
    // Récupérer les informations utilisateur du contexte
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]interface{}{
        "user_id": claims.UserID,
        "email":   claims.Email,
    })
}
```

## Mise à jour des handlers de tâches

Modifions `internal/handlers/task_handler.go` pour utiliser l'authentification :

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"

    "github.com/gorilla/mux"
    "todo-api/internal/middleware"
    "todo-api/internal/models"
    "todo-api/internal/service"
    "todo-api/internal/utils"
)

type TaskHandler struct {
    taskService service.TaskService
}

func NewTaskHandler(taskService service.TaskService) *TaskHandler {
    return &TaskHandler{taskService: taskService}
}

// CreateTask crée une nouvelle tâche
func (h *TaskHandler) CreateTask(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    var task models.Task

    // Décoder le JSON de la requête
    if err := json.NewDecoder(r.Body).Decode(&task); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Assigner la tâche à l'utilisateur authentifié
    task.UserID = claims.UserID

    // Créer la tâche
    if err := h.taskService.CreateTask(&task); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, task)
}

// GetTasks récupère les tâches de l'utilisateur authentifié
func (h *TaskHandler) GetTasks(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    tasks, err := h.taskService.GetUserTasks(claims.UserID)
    if err != nil {
        utils.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, tasks)
}

// GetTask récupère une tâche spécifique
func (h *TaskHandler) GetTask(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    vars := mux.Vars(r)
    id, err := strconv.ParseUint(vars["id"], 10, 32)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    task, err := h.taskService.GetTaskByID(uint(id), claims.UserID)
    if err != nil {
        utils.WriteError(w, http.StatusNotFound, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, task)
}

// UpdateTask met à jour une tâche
func (h *TaskHandler) UpdateTask(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    vars := mux.Vars(r)
    id, err := strconv.ParseUint(vars["id"], 10, 32)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    var updates models.Task
    if err := json.NewDecoder(r.Body).Decode(&updates); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    if err := h.taskService.UpdateTask(uint(id), claims.UserID, &updates); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche mise à jour"})
}

// DeleteTask supprime une tâche
func (h *TaskHandler) DeleteTask(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    vars := mux.Vars(r)
    id, err := strconv.ParseUint(vars["id"], 10, 32)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    if err := h.taskService.DeleteTask(uint(id), claims.UserID); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche supprimée"})
}
```

## Mise à jour du routeur

Modifions `internal/handlers/router.go` pour inclure l'authentification et les middlewares :

```go
package handlers

import (
    "github.com/gorilla/mux"
    "todo-api/internal/middleware"
    "todo-api/internal/service"
)

func SetupRoutes(taskService service.TaskService, authService service.AuthService) *mux.Router {
    router := mux.NewRouter()

    // Appliquer les middlewares globaux
    router.Use(middleware.CORSMiddleware)
    router.Use(middleware.LoggingMiddleware)

    // Créer les handlers
    taskHandler := NewTaskHandler(taskService)
    authHandler := NewAuthHandler(authService)

    // Routes API
    api := router.PathPrefix("/api").Subrouter()

    // Routes publiques (pas d'authentification requise)
    public := api.PathPrefix("/auth").Subrouter()
    public.HandleFunc("/register", authHandler.Register).Methods("POST")
    public.HandleFunc("/login", authHandler.Login).Methods("POST")

    // Routes protégées (authentification requise)
    protected := api.PathPrefix("").Subrouter()
    protected.Use(middleware.AuthMiddleware) // Appliquer le middleware d'auth

    // Routes d'authentification protégées
    protected.HandleFunc("/auth/profile", authHandler.Profile).Methods("GET")

    // Routes pour les tâches (toutes protégées)
    protected.HandleFunc("/tasks", taskHandler.GetTasks).Methods("GET")
    protected.HandleFunc("/tasks", taskHandler.CreateTask).Methods("POST")
    protected.HandleFunc("/tasks/{id}", taskHandler.GetTask).Methods("GET")
    protected.HandleFunc("/tasks/{id}", taskHandler.UpdateTask).Methods("PUT")
    protected.HandleFunc("/tasks/{id}", taskHandler.DeleteTask).Methods("DELETE")

    // Route de santé (publique)
    api.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        utils.WriteSuccess(w, 200, map[string]string{"status": "OK"})
    }).Methods("GET")

    return router
}
```

## Mise à jour du main.go

Modifions `cmd/server/main.go` pour inclure le service d'authentification :

```go
package main

import (
    "log"
    "net/http"

    "todo-api/internal/config"
    "todo-api/internal/handlers"
    "todo-api/internal/repository"
    "todo-api/internal/service"
    "todo-api/pkg/database"
)

func main() {
    // Charger la configuration
    cfg := config.Load()

    // Établir la connexion à la base de données
    database.Connect()
    database.Migrate()

    // Initialiser les repositories
    taskRepo := repository.NewTaskRepository(database.DB)
    userRepo := repository.NewUserRepository(database.DB)

    // Initialiser les services
    taskService := service.NewTaskService(taskRepo)
    authService := service.NewAuthService(userRepo)

    // Configurer les routes
    router := handlers.SetupRoutes(taskService, authService)

    // Configurer le serveur
    server := &http.Server{
        Addr:    ":" + cfg.Port,
        Handler: router,
    }

    log.Printf("Serveur démarré sur le port %s", cfg.Port)
    log.Fatal(server.ListenAndServe())
}
```

## Mise à jour du fichier .env

Ajoutons la clé secrète JWT au fichier `.env` :

```env
PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_NAME=todo_api
JWT_SECRET=ma-cle-super-secrete-de-production
```

## Test de l'authentification

### 1. Inscription d'un utilisateur

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "motdepasse123"
  }'
```

**Réponse attendue :**
```json
{
  "success": true,
  "data": {
    "user": {
      "id": 1,
      "username": "johndoe",
      "email": "john@example.com",
      "created_at": "2024-01-15T10:30:00Z"
    },
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
}
```

### 2. Connexion

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "motdepasse123"
  }'
```

### 3. Accès au profil (authentifié)

```bash
# Récupérer le token de la réponse précédente
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

curl -X GET http://localhost:8080/api/auth/profile \
  -H "Authorization: Bearer $TOKEN"
```

### 4. Créer une tâche (authentifié)

```bash
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Ma tâche sécurisée",
    "description": "Cette tâche nécessite une authentification"
  }'
```

### 5. Tester l'accès non autorisé

```bash
# Essayer d'accéder aux tâches sans token
curl -X GET http://localhost:8080/api/tasks

# Réponse attendue : 401 Unauthorized
{
  "success": false,
  "error": "Token d'autorisation manquant"
}
```

### 6. Récupérer les tâches de l'utilisateur authentifié

```bash
curl -X GET http://localhost:8080/api/tasks \
  -H "Authorization: Bearer $TOKEN"
```

## Gestion avancée des erreurs d'authentification

### internal/utils/errors.go

Créons des utilitaires pour gérer les erreurs d'authentification plus finement :

```go
package utils

import (
    "net/http"
)

// AuthError représente une erreur d'authentification
type AuthError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Type    string `json:"type"`
}

func (e AuthError) Error() string {
    return e.Message
}

// Erreurs d'authentification prédéfinies
var (
    ErrTokenMissing = AuthError{
        Code:    http.StatusUnauthorized,
        Message: "Token d'autorisation manquant",
        Type:    "TOKEN_MISSING",
    }

    ErrTokenInvalid = AuthError{
        Code:    http.StatusUnauthorized,
        Message: "Token invalide ou expiré",
        Type:    "TOKEN_INVALID",
    }

    ErrTokenMalformed = AuthError{
        Code:    http.StatusUnauthorized,
        Message: "Format du token invalide",
        Type:    "TOKEN_MALFORMED",
    }

    ErrUnauthorized = AuthError{
        Code:    http.StatusForbidden,
        Message: "Accès non autorisé à cette ressource",
        Type:    "ACCESS_DENIED",
    }
)

// WriteAuthError écrit une erreur d'authentification
func WriteAuthError(w http.ResponseWriter, authErr AuthError) {
    WriteJSON(w, authErr.Code, Response{
        Success: false,
        Error:   authErr.Message,
    })
}
```

### Mise à jour du middleware d'authentification

Améliorons `internal/middleware/auth_middleware.go` avec une meilleure gestion des erreurs :

```go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "todo-api/internal/utils"
)

type contextKey string

const UserContextKey contextKey = "user"

// AuthMiddleware vérifie l'authentification JWT
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Récupérer le header Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            utils.WriteAuthError(w, utils.ErrTokenMissing)
            return
        }

        // Vérifier le format "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            utils.WriteAuthError(w, utils.ErrTokenMalformed)
            return
        }

        tokenString := parts[1]
        if tokenString == "" {
            utils.WriteAuthError(w, utils.ErrTokenMalformed)
            return
        }

        // Valider le token
        claims, err := utils.ValidateToken(tokenString)
        if err != nil {
            utils.WriteAuthError(w, utils.ErrTokenInvalid)
            return
        }

        // Ajouter les informations utilisateur au contexte
        ctx := context.WithValue(r.Context(), UserContextKey, claims)

        // Passer au handler suivant
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// GetUserFromContext récupère les informations utilisateur du contexte
func GetUserFromContext(ctx context.Context) (*utils.Claims, bool) {
    claims, ok := ctx.Value(UserContextKey).(*utils.Claims)
    return claims, ok
}

// RequireAuth est un helper pour vérifier l'authentification dans les handlers
func RequireAuth(ctx context.Context) (*utils.Claims, error) {
    claims, ok := GetUserFromContext(ctx)
    if !ok {
        return nil, utils.ErrUnauthorized
    }
    return claims, nil
}
```

## Middleware de limitation de débit (Rate Limiting)

Ajoutons un middleware pour limiter le nombre de requêtes par utilisateur :

### internal/middleware/rate_limit_middleware.go

```go
package middleware

import (
    "net/http"
    "sync"
    "time"

    "todo-api/internal/utils"
)

// RateLimiter structure pour gérer les limites de débit
type RateLimiter struct {
    requests map[string]*UserLimit
    mutex    sync.RWMutex
    limit    int
    window   time.Duration
}

// UserLimit garde trace des requêtes par utilisateur
type UserLimit struct {
    count     int
    resetTime time.Time
}

// NewRateLimiter crée un nouveau limiteur de débit
func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
    rl := &RateLimiter{
        requests: make(map[string]*UserLimit),
        limit:    limit,
        window:   window,
    }

    // Nettoyer périodiquement les entrées expirées
    go rl.cleanup()

    return rl
}

// IsAllowed vérifie si la requête est autorisée
func (rl *RateLimiter) IsAllowed(identifier string) bool {
    rl.mutex.Lock()
    defer rl.mutex.Unlock()

    now := time.Now()

    userLimit, exists := rl.requests[identifier]
    if !exists || now.After(userLimit.resetTime) {
        // Nouvelle fenêtre de temps
        rl.requests[identifier] = &UserLimit{
            count:     1,
            resetTime: now.Add(rl.window),
        }
        return true
    }

    if userLimit.count >= rl.limit {
        return false
    }

    userLimit.count++
    return true
}

// cleanup supprime les entrées expirées
func (rl *RateLimiter) cleanup() {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        rl.mutex.Lock()
        now := time.Now()
        for key, userLimit := range rl.requests {
            if now.After(userLimit.resetTime) {
                delete(rl.requests, key)
            }
        }
        rl.mutex.Unlock()
    }
}

// RateLimitMiddleware crée un middleware de limitation de débit
func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Utiliser l'IP comme identifiant (ou l'ID utilisateur si authentifié)
            identifier := r.RemoteAddr

            // Si l'utilisateur est authentifié, utiliser son ID
            if claims, ok := GetUserFromContext(r.Context()); ok {
                identifier = string(rune(claims.UserID))
            }

            if !limiter.IsAllowed(identifier) {
                utils.WriteError(w, http.StatusTooManyRequests, "Trop de requêtes, veuillez réessayer plus tard")
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

## Middleware de validation

Créons un middleware pour valider les données d'entrée :

### internal/middleware/validation_middleware.go

```go
package middleware

import (
    "bytes"
    "encoding/json"
    "io"
    "net/http"
    "strings"

    "todo-api/internal/utils"
)

// ValidateJSONMiddleware valide que le contenu est du JSON valide
func ValidateJSONMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Appliquer seulement aux requêtes avec du contenu
        if r.Method == "POST" || r.Method == "PUT" || r.Method == "PATCH" {
            contentType := r.Header.Get("Content-Type")
            if !strings.Contains(contentType, "application/json") {
                utils.WriteError(w, http.StatusBadRequest, "Content-Type doit être application/json")
                return
            }

            // Lire le body
            body, err := io.ReadAll(r.Body)
            if err != nil {
                utils.WriteError(w, http.StatusBadRequest, "Impossible de lire le body de la requête")
                return
            }

            // Vérifier que c'est du JSON valide
            if len(body) > 0 {
                var js json.RawMessage
                if err := json.Unmarshal(body, &js); err != nil {
                    utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
                    return
                }
            }

            // Remettre le body dans la requête
            r.Body = io.NopCloser(bytes.NewBuffer(body))
        }

        next.ServeHTTP(w, r)
    })
}
```

## Service de gestion des sessions

Créons un service pour gérer les sessions et la révocation de tokens :

### internal/service/session_service.go

```go
package service

import (
    "sync"
    "time"
)

// SessionService gère les sessions actives
type SessionService interface {
    CreateSession(userID uint, tokenID string) error
    IsSessionValid(tokenID string) bool
    RevokeSession(tokenID string) error
    RevokeAllUserSessions(userID uint) error
}

type sessionService struct {
    sessions map[string]*Session
    mutex    sync.RWMutex
}

type Session struct {
    UserID    uint
    TokenID   string
    CreatedAt time.Time
    ExpiresAt time.Time
}

func NewSessionService() SessionService {
    service := &sessionService{
        sessions: make(map[string]*Session),
    }

    // Nettoyer les sessions expirées périodiquement
    go service.cleanup()

    return service
}

func (s *sessionService) CreateSession(userID uint, tokenID string) error {
    s.mutex.Lock()
    defer s.mutex.Unlock()

    s.sessions[tokenID] = &Session{
        UserID:    userID,
        TokenID:   tokenID,
        CreatedAt: time.Now(),
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }

    return nil
}

func (s *sessionService) IsSessionValid(tokenID string) bool {
    s.mutex.RLock()
    defer s.mutex.RUnlock()

    session, exists := s.sessions[tokenID]
    if !exists {
        return false
    }

    return time.Now().Before(session.ExpiresAt)
}

func (s *sessionService) RevokeSession(tokenID string) error {
    s.mutex.Lock()
    defer s.mutex.Unlock()

    delete(s.sessions, tokenID)
    return nil
}

func (s *sessionService) RevokeAllUserSessions(userID uint) error {
    s.mutex.Lock()
    defer s.mutex.Unlock()

    for tokenID, session := range s.sessions {
        if session.UserID == userID {
            delete(s.sessions, tokenID)
        }
    }

    return nil
}

func (s *sessionService) cleanup() {
    ticker := time.NewTicker(time.Hour)
    defer ticker.Stop()

    for range ticker.C {
        s.mutex.Lock()
        now := time.Now()
        for tokenID, session := range s.sessions {
            if now.After(session.ExpiresAt) {
                delete(s.sessions, tokenID)
            }
        }
        s.mutex.Unlock()
    }
}
```

## Amélioration des handlers d'authentification

Mettons à jour `internal/handlers/auth_handler.go` pour inclure la déconnexion :

```go
// Ajout de la méthode Logout dans AuthHandler

// Logout gère la déconnexion des utilisateurs
func (h *AuthHandler) Logout(w http.ResponseWriter, r *http.Request) {
    // Récupérer le token du header
    authHeader := r.Header.Get("Authorization")
    if authHeader == "" {
        utils.WriteError(w, http.StatusBadRequest, "Token manquant")
        return
    }

    parts := strings.Split(authHeader, " ")
    if len(parts) != 2 {
        utils.WriteError(w, http.StatusBadRequest, "Format de token invalide")
        return
    }

    tokenString := parts[1]

    // Valider le token pour récupérer les claims
    claims, err := utils.ValidateToken(tokenString)
    if err != nil {
        utils.WriteError(w, http.StatusUnauthorized, "Token invalide")
        return
    }

    // Révoquer la session (si vous utilisez le service de sessions)
    // h.sessionService.RevokeSession(claims.ID)

    utils.WriteSuccess(w, http.StatusOK, map[string]string{
        "message": "Déconnexion réussie",
    })
}
```

## Mise à jour du routeur avec les nouveaux middlewares

Modifions `internal/handlers/router.go` pour inclure tous nos middlewares :

```go
package handlers

import (
    "time"

    "github.com/gorilla/mux"
    "todo-api/internal/middleware"
    "todo-api/internal/service"
)

func SetupRoutes(taskService service.TaskService, authService service.AuthService) *mux.Router {
    router := mux.NewRouter()

    // Créer le limiteur de débit (100 requêtes par heure par utilisateur)
    rateLimiter := middleware.NewRateLimiter(100, time.Hour)

    // Appliquer les middlewares globaux dans l'ordre
    router.Use(middleware.CORSMiddleware)
    router.Use(middleware.LoggingMiddleware)
    router.Use(middleware.RateLimitMiddleware(rateLimiter))
    router.Use(middleware.ValidateJSONMiddleware)

    // Créer les handlers
    taskHandler := NewTaskHandler(taskService)
    authHandler := NewAuthHandler(authService)

    // Routes API
    api := router.PathPrefix("/api").Subrouter()

    // Routes publiques (pas d'authentification requise)
    public := api.PathPrefix("/auth").Subrouter()
    public.HandleFunc("/register", authHandler.Register).Methods("POST")
    public.HandleFunc("/login", authHandler.Login).Methods("POST")

    // Routes protégées (authentification requise)
    protected := api.PathPrefix("").Subrouter()
    protected.Use(middleware.AuthMiddleware)

    // Routes d'authentification protégées
    protected.HandleFunc("/auth/profile", authHandler.Profile).Methods("GET")
    protected.HandleFunc("/auth/logout", authHandler.Logout).Methods("POST")

    // Routes pour les tâches (toutes protégées)
    protected.HandleFunc("/tasks", taskHandler.GetTasks).Methods("GET")
    protected.HandleFunc("/tasks", taskHandler.CreateTask).Methods("POST")
    protected.HandleFunc("/tasks/{id}", taskHandler.GetTask).Methods("GET")
    protected.HandleFunc("/tasks/{id}", taskHandler.UpdateTask).Methods("PUT")
    protected.HandleFunc("/tasks/{id}", taskHandler.DeleteTask).Methods("DELETE")

    // Route de santé (publique)
    api.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        utils.WriteSuccess(w, 200, map[string]string{
            "status": "OK",
            "timestamp": time.Now().Format(time.RFC3339),
        })
    }).Methods("GET")

    return router
}
```

## Tests complets de l'API sécurisée

### Script de test bash

Créez un fichier `test_api.sh` pour tester toutes les fonctionnalités :

```bash
#!/bin/bash

BASE_URL="http://localhost:8080/api"

echo "=== Test de l'API Todo Sécurisée ==="

# 1. Test de santé
echo "1. Test de santé..."
curl -s "$BASE_URL/health" | jq .

# 2. Inscription
echo -e "\n2. Inscription d'un utilisateur..."
REGISTER_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "password123"
  }')

echo $REGISTER_RESPONSE | jq .

# Extraire le token
TOKEN=$(echo $REGISTER_RESPONSE | jq -r '.data.token')
echo "Token: $TOKEN"

# 3. Connexion
echo -e "\n3. Connexion..."
LOGIN_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "password123"
  }')

echo $LOGIN_RESPONSE | jq .

# 4. Profil utilisateur
echo -e "\n4. Récupération du profil..."
curl -s -X GET "$BASE_URL/auth/profile" \
  -H "Authorization: Bearer $TOKEN" | jq .

# 5. Créer une tâche
echo -e "\n5. Création d'une tâche..."
TASK_RESPONSE=$(curl -s -X POST "$BASE_URL/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Tâche de test",
    "description": "Description de test",
    "priority": "high"
  }')

echo $TASK_RESPONSE | jq .

# Extraire l'ID de la tâche
TASK_ID=$(echo $TASK_RESPONSE | jq -r '.data.id')

# 6. Récupérer toutes les tâches
echo -e "\n6. Récupération des tâches..."
curl -s -X GET "$BASE_URL/tasks" \
  -H "Authorization: Bearer $TOKEN" | jq .

# 7. Récupérer une tâche spécifique
echo -e "\n7. Récupération de la tâche $TASK_ID..."
curl -s -X GET "$BASE_URL/tasks/$TASK_ID" \
  -H "Authorization: Bearer $TOKEN" | jq .

# 8. Mettre à jour la tâche
echo -e "\n8. Mise à jour de la tâche..."
curl -s -X PUT "$BASE_URL/tasks/$TASK_ID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Tâche mise à jour",
    "completed": true
  }' | jq .

# 9. Test d'accès non autorisé
echo -e "\n9. Test d'accès non autorisé..."
curl -s -X GET "$BASE_URL/tasks" | jq .

# 10. Test avec token invalide
echo -e "\n10. Test avec token invalide..."
curl -s -X GET "$BASE_URL/tasks" \
  -H "Authorization: Bearer invalid_token" | jq .

# 11. Déconnexion
echo -e "\n11. Déconnexion..."
curl -s -X POST "$BASE_URL/auth/logout" \
  -H "Authorization: Bearer $TOKEN" | jq .

echo -e "\n=== Tests terminés ==="
```

Rendez le script exécutable et lancez-le :

```bash
chmod +x test_api.sh
./test_api.sh
```

## Résumé des fonctionnalités ajoutées

Dans cette section, nous avons implémenté :

**Authentification JWT complète** :
- Inscription et connexion des utilisateurs
- Génération et validation de tokens JWT
- Hashage sécurisé des mots de passe avec bcrypt

**Middlewares de sécurité** :
- Middleware d'authentification
- Middleware CORS pour les requêtes cross-origin
- Middleware de logging pour tracer les requêtes
- Middleware de limitation de débit (rate limiting)
- Middleware de validation JSON

**Gestion avancée des erreurs** :
- Erreurs d'authentification typées
- Messages d'erreur cohérents
- Codes de statut HTTP appropriés

**Sécurisation complète de l'API** :
- Toutes les routes de tâches sont maintenant protégées
- Isolation des données par utilisateur
- Validation des autorisations

**Architecture robuste** :
- Séparation claire des responsabilités
- Code facilement testable
- Extensibilité pour de nouvelles fonctionnalités

## Points clés à retenir

- **Les middlewares s'exécutent dans l'ordre où ils sont définis**
- **JWT permet une authentification sans état (stateless)**
- **Toujours hasher les mots de passe avant stockage**
- **Valider les autorisations pour chaque ressource**
- **Gérer les erreurs de manière cohérente**
- **Limiter les requêtes pour prévenir les abus**

Dans la prochaine section (16-3), nous ajouterons la validation avancée des données pour rendre notre API encore plus robuste et sécurisée.

⏭️
