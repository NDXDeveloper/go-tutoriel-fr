🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16-1 : Architecture d'une API

## Introduction

L'architecture d'une API est la fondation sur laquelle repose toute votre application. Une bonne architecture facilite la maintenance, les tests, et l'évolutivité de votre code. Dans cette section, nous allons apprendre à structurer notre API REST de manière professionnelle.

## Qu'est-ce que l'architecture d'une API ?

L'architecture d'une API définit comment organiser et structurer votre code pour créer une application maintenable et évolutive. Elle détermine :

- **Comment les différentes parties de votre code interagissent**
- **Où placer chaque type de fonctionnalité**
- **Comment séparer les responsabilités**
- **Comment organiser vos fichiers et dossiers**

## Les couches d'une API

Une API bien conçue est organisée en **couches** distinctes. Chaque couche a une responsabilité spécifique :

### 1. Couche de présentation (Handlers)
**Responsabilité** : Recevoir les requêtes HTTP et renvoyer les réponses
- Parse les paramètres de la requête
- Appelle la couche service
- Formate la réponse JSON

### 2. Couche de logique métier (Services)
**Responsabilité** : Contient la logique de l'application
- Règles métier
- Validation des données
- Orchestration des opérations

### 3. Couche de données (Repository)
**Responsabilité** : Accès aux données
- Requêtes en base de données
- Gestion des connexions
- Mapping des données

### 4. Couche de modèles (Models)
**Responsabilité** : Définir les structures de données
- Structures Go
- Validation des champs
- Relations entre entités

## Mise en pratique : Setup du projet

### Étape 1 : Initialisation du projet

Créons la structure de base de notre projet :

```bash
mkdir todo-api
cd todo-api
go mod init todo-api
```

### Étape 2 : Structure des dossiers

Créons l'architecture de dossiers :

```
todo-api/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   ├── handlers/
│   ├── middleware/
│   ├── models/
│   ├── repository/
│   ├── service/
│   └── utils/
├── pkg/
│   └── database/
└── go.mod
```

**Explication des dossiers :**

- `cmd/server/` : Point d'entrée de l'application
- `internal/` : Code privé de l'application (ne peut pas être importé par d'autres projets)
- `pkg/` : Code réutilisable (peut être importé par d'autres projets)

### Étape 3 : Installation des dépendances

```bash
go get github.com/gorilla/mux
go get gorm.io/gorm
go get gorm.io/driver/postgres
go get github.com/joho/godotenv
```

## Création des modèles

Commençons par définir nos structures de données dans `internal/models/`.

### internal/models/user.go

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type User struct {
    ID        uint           `json:"id" gorm:"primaryKey"`
    Username  string         `json:"username" gorm:"unique;not null"`
    Email     string         `json:"email" gorm:"unique;not null"`
    Password  string         `json:"-" gorm:"not null"` // Le tag "-" cache le mot de passe dans le JSON
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `json:"-" gorm:"index"` // Soft delete
    Tasks     []Task         `json:"tasks,omitempty" gorm:"foreignKey:UserID"`
}

// TableName spécifie le nom de la table
func (User) TableName() string {
    return "users"
}
```

### internal/models/task.go

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type Task struct {
    ID          uint           `json:"id" gorm:"primaryKey"`
    Title       string         `json:"title" gorm:"not null"`
    Description string         `json:"description"`
    Completed   bool           `json:"completed" gorm:"default:false"`
    Priority    string         `json:"priority" gorm:"default:'medium'"`
    DueDate     *time.Time     `json:"due_date,omitempty"`
    UserID      uint           `json:"user_id" gorm:"not null"`
    CategoryID  *uint          `json:"category_id,omitempty"`
    CreatedAt   time.Time      `json:"created_at"`
    UpdatedAt   time.Time      `json:"updated_at"`
    DeletedAt   gorm.DeletedAt `json:"-" gorm:"index"`

    // Relations
    User     User      `json:"user,omitempty" gorm:"foreignKey:UserID"`
    Category *Category `json:"category,omitempty" gorm:"foreignKey:CategoryID"`
}

func (Task) TableName() string {
    return "tasks"
}
```

### internal/models/category.go

```go
package models

import (
    "time"
    "gorm.io/gorm"
)

type Category struct {
    ID        uint           `json:"id" gorm:"primaryKey"`
    Name      string         `json:"name" gorm:"not null"`
    Color     string         `json:"color" gorm:"default:'#3498db'"`
    UserID    uint           `json:"user_id" gorm:"not null"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `json:"-" gorm:"index"`
    Tasks     []Task         `json:"tasks,omitempty" gorm:"foreignKey:CategoryID"`
}

func (Category) TableName() string {
    return "categories"
}
```

## Configuration de la base de données

### pkg/database/postgres.go

```go
package database

import (
    "fmt"
    "log"
    "os"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"

    "todo-api/internal/models"
)

var DB *gorm.DB

// Connect établit la connexion à la base de données
func Connect() {
    var err error

    // Configuration de la connexion
    dsn := fmt.Sprintf(
        "host=%s user=%s password=%s dbname=%s port=%s sslmode=disable",
        os.Getenv("DB_HOST"),
        os.Getenv("DB_USER"),
        os.Getenv("DB_PASSWORD"),
        os.Getenv("DB_NAME"),
        os.Getenv("DB_PORT"),
    )

    // Connexion à la base de données
    DB, err = gorm.Open(postgres.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })

    if err != nil {
        log.Fatal("Erreur de connexion à la base de données:", err)
    }

    log.Println("Connexion à la base de données établie")
}

// Migrate effectue les migrations automatiques
func Migrate() {
    DB.AutoMigrate(&models.User{}, &models.Category{}, &models.Task{})
    log.Println("Migrations effectuées")
}
```

## Configuration de l'application

### internal/config/config.go

```go
package config

import (
    "log"
    "os"

    "github.com/joho/godotenv"
)

type Config struct {
    Port   string
    DBHost string
    DBPort string
    DBUser string
    DBPassword string
    DBName string
}

func Load() *Config {
    // Charger le fichier .env
    err := godotenv.Load()
    if err != nil {
        log.Println("Fichier .env non trouvé, utilisation des variables d'environnement")
    }

    return &Config{
        Port:       getEnv("PORT", "8080"),
        DBHost:     getEnv("DB_HOST", "localhost"),
        DBPort:     getEnv("DB_PORT", "5432"),
        DBUser:     getEnv("DB_USER", "postgres"),
        DBPassword: getEnv("DB_PASSWORD", ""),
        DBName:     getEnv("DB_NAME", "todo_api"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

## Couche Repository

La couche repository gère l'accès aux données. Elle isole la logique métier des détails de la base de données.

### internal/repository/task_repository.go

```go
package repository

import (
    "gorm.io/gorm"
    "todo-api/internal/models"
)

type TaskRepository interface {
    Create(task *models.Task) error
    GetByID(id uint) (*models.Task, error)
    GetByUserID(userID uint) ([]models.Task, error)
    Update(task *models.Task) error
    Delete(id uint) error
}

type taskRepository struct {
    db *gorm.DB
}

func NewTaskRepository(db *gorm.DB) TaskRepository {
    return &taskRepository{db: db}
}

func (r *taskRepository) Create(task *models.Task) error {
    return r.db.Create(task).Error
}

func (r *taskRepository) GetByID(id uint) (*models.Task, error) {
    var task models.Task
    err := r.db.Preload("User").Preload("Category").First(&task, id).Error
    if err != nil {
        return nil, err
    }
    return &task, nil
}

func (r *taskRepository) GetByUserID(userID uint) ([]models.Task, error) {
    var tasks []models.Task
    err := r.db.Where("user_id = ?", userID).
        Preload("Category").
        Find(&tasks).Error
    return tasks, err
}

func (r *taskRepository) Update(task *models.Task) error {
    return r.db.Save(task).Error
}

func (r *taskRepository) Delete(id uint) error {
    return r.db.Delete(&models.Task{}, id).Error
}
```

## Couche Service

La couche service contient la logique métier de l'application.

### internal/service/task_service.go

```go
package service

import (
    "errors"
    "todo-api/internal/models"
    "todo-api/internal/repository"
)

type TaskService interface {
    CreateTask(task *models.Task) error
    GetTaskByID(id uint, userID uint) (*models.Task, error)
    GetUserTasks(userID uint) ([]models.Task, error)
    UpdateTask(id uint, userID uint, updates *models.Task) error
    DeleteTask(id uint, userID uint) error
}

type taskService struct {
    taskRepo repository.TaskRepository
}

func NewTaskService(taskRepo repository.TaskRepository) TaskService {
    return &taskService{taskRepo: taskRepo}
}

func (s *taskService) CreateTask(task *models.Task) error {
    // Validation basique
    if task.Title == "" {
        return errors.New("le titre est requis")
    }

    // Valeurs par défaut
    if task.Priority == "" {
        task.Priority = "medium"
    }

    return s.taskRepo.Create(task)
}

func (s *taskService) GetTaskByID(id uint, userID uint) (*models.Task, error) {
    task, err := s.taskRepo.GetByID(id)
    if err != nil {
        return nil, err
    }

    // Vérifier que la tâche appartient à l'utilisateur
    if task.UserID != userID {
        return nil, errors.New("accès non autorisé")
    }

    return task, nil
}

func (s *taskService) GetUserTasks(userID uint) ([]models.Task, error) {
    return s.taskRepo.GetByUserID(userID)
}

func (s *taskService) UpdateTask(id uint, userID uint, updates *models.Task) error {
    // Récupérer la tâche existante
    existingTask, err := s.GetTaskByID(id, userID)
    if err != nil {
        return err
    }

    // Mettre à jour les champs
    if updates.Title != "" {
        existingTask.Title = updates.Title
    }
    if updates.Description != "" {
        existingTask.Description = updates.Description
    }
    existingTask.Completed = updates.Completed
    if updates.Priority != "" {
        existingTask.Priority = updates.Priority
    }
    if updates.DueDate != nil {
        existingTask.DueDate = updates.DueDate
    }
    if updates.CategoryID != nil {
        existingTask.CategoryID = updates.CategoryID
    }

    return s.taskRepo.Update(existingTask)
}

func (s *taskService) DeleteTask(id uint, userID uint) error {
    // Vérifier que la tâche appartient à l'utilisateur
    _, err := s.GetTaskByID(id, userID)
    if err != nil {
        return err
    }

    return s.taskRepo.Delete(id)
}
```

## Couche Handlers

Les handlers gèrent les requêtes HTTP et les réponses.

### internal/utils/response.go

D'abord, créons des utilitaires pour les réponses JSON :

```go
package utils

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    Success bool        `json:"success"`
    Message string      `json:"message,omitempty"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

func WriteJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func WriteSuccess(w http.ResponseWriter, status int, data interface{}) {
    WriteJSON(w, status, Response{
        Success: true,
        Data:    data,
    })
}

func WriteError(w http.ResponseWriter, status int, message string) {
    WriteJSON(w, status, Response{
        Success: false,
        Error:   message,
    })
}
```

### internal/handlers/task_handler.go

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"

    "github.com/gorilla/mux"
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
    var task models.Task

    // Décoder le JSON de la requête
    if err := json.NewDecoder(r.Body).Decode(&task); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // TODO: Récupérer l'ID utilisateur depuis le contexte d'authentification
    // Pour le moment, on utilise un ID fictif
    task.UserID = 1

    // Créer la tâche
    if err := h.taskService.CreateTask(&task); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, task)
}

// GetTasks récupère les tâches de l'utilisateur
func (h *TaskHandler) GetTasks(w http.ResponseWriter, r *http.Request) {
    // TODO: Récupérer l'ID utilisateur depuis le contexte d'authentification
    userID := uint(1)

    tasks, err := h.taskService.GetUserTasks(userID)
    if err != nil {
        utils.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, tasks)
}

// GetTask récupère une tâche spécifique
func (h *TaskHandler) GetTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.ParseUint(vars["id"], 10, 32)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    // TODO: Récupérer l'ID utilisateur depuis le contexte d'authentification
    userID := uint(1)

    task, err := h.taskService.GetTaskByID(uint(id), userID)
    if err != nil {
        utils.WriteError(w, http.StatusNotFound, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, task)
}

// UpdateTask met à jour une tâche
func (h *TaskHandler) UpdateTask(w http.ResponseWriter, r *http.Request) {
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

    // TODO: Récupérer l'ID utilisateur depuis le contexte d'authentification
    userID := uint(1)

    if err := h.taskService.UpdateTask(uint(id), userID, &updates); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche mise à jour"})
}

// DeleteTask supprime une tâche
func (h *TaskHandler) DeleteTask(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.ParseUint(vars["id"], 10, 32)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    // TODO: Récupérer l'ID utilisateur depuis le contexte d'authentification
    userID := uint(1)

    if err := h.taskService.DeleteTask(uint(id), userID); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche supprimée"})
}
```

## Configuration des routes

### internal/handlers/router.go

```go
package handlers

import (
    "github.com/gorilla/mux"
    "todo-api/internal/service"
)

func SetupRoutes(taskService service.TaskService) *mux.Router {
    router := mux.NewRouter()

    // Créer les handlers
    taskHandler := NewTaskHandler(taskService)

    // Définir les routes API
    api := router.PathPrefix("/api").Subrouter()

    // Routes pour les tâches
    api.HandleFunc("/tasks", taskHandler.GetTasks).Methods("GET")
    api.HandleFunc("/tasks", taskHandler.CreateTask).Methods("POST")
    api.HandleFunc("/tasks/{id}", taskHandler.GetTask).Methods("GET")
    api.HandleFunc("/tasks/{id}", taskHandler.UpdateTask).Methods("PUT")
    api.HandleFunc("/tasks/{id}", taskHandler.DeleteTask).Methods("DELETE")

    // Route de santé
    api.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        utils.WriteSuccess(w, 200, map[string]string{"status": "OK"})
    }).Methods("GET")

    return router
}
```

## Point d'entrée principal

### cmd/server/main.go

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"

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

    // Initialiser les couches
    taskRepo := repository.NewTaskRepository(database.DB)
    taskService := service.NewTaskService(taskRepo)

    // Configurer les routes
    router := handlers.SetupRoutes(taskService)

    // Configurer le serveur
    server := &http.Server{
        Addr:    ":" + cfg.Port,
        Handler: router,
    }

    log.Printf("Serveur démarré sur le port %s", cfg.Port)
    log.Fatal(server.ListenAndServe())
}
```

## Fichier de configuration environnement

Créez un fichier `.env` à la racine du projet :

```env
PORT=8080
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=yourpassword
DB_NAME=todo_api
```

## Test de l'API

### Démarrage du serveur

```bash
go run cmd/server/main.go
```

### Tests avec curl

```bash
# Créer une tâche
curl -X POST http://localhost:8080/api/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Ma première tâche", "description": "Description de test"}'

# Récupérer toutes les tâches
curl http://localhost:8080/api/tasks

# Récupérer une tâche spécifique
curl http://localhost:8080/api/tasks/1
```

## Résumé

Dans cette section, nous avons appris :

**Architecture en couches** : Séparation claire des responsabilités entre handlers, services, et repositories

**Modèles de données** : Définition des structures avec GORM pour la persistance

**Dependency Injection** : Injection des dépendances pour une architecture testable

**API REST basique** : Implémentation des opérations CRUD pour les tâches

**Configuration** : Gestion de la configuration avec des variables d'environnement

**Structure projet** : Organisation professionnelle du code Go

Dans la prochaine section (16-2), nous ajouterons les middlewares et l'authentification pour sécuriser notre API et la rendre plus robuste.

## Points clés à retenir

- **Une bonne architecture facilite la maintenance et l'évolution**
- **Chaque couche a une responsabilité spécifique**
- **L'injection de dépendances améliore la testabilité**
- **La séparation des préoccupations rend le code plus lisible**
- **Une structure de projet cohérente améliore la collaboration en équipe**

⏭️
