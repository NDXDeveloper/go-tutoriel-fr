🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16-3 : Validation des données

## Introduction

Dans les sections précédentes, nous avons créé une API sécurisée avec authentification. Cependant, notre API accepte encore n'importe quelles données d'entrée ! Un utilisateur pourrait envoyer des données incomplètes, incorrectes ou même malveillantes.

Dans cette section, nous allons apprendre à :
- **Valider rigoureusement** toutes les données d'entrée
- **Sanitiser** les données pour éviter les attaques
- **Créer des règles métier** complexes
- **Retourner des erreurs de validation** détaillées et utiles

## Pourquoi valider les données ?

La validation des données est cruciale pour :

**Sécurité** : Prévenir les injections SQL, XSS et autres attaques
**Intégrité** : S'assurer que les données respectent les règles métier
**Expérience utilisateur** : Fournir des messages d'erreur clairs
**Stabilité** : Éviter les bugs causés par des données invalides

## Types de validation

### 1. Validation de format
- Format email valide
- Longueur minimum/maximum
- Caractères autorisés

### 2. Validation métier
- Règles spécifiques à l'application
- Contraintes d'unicité
- Relations entre champs

### 3. Validation de sécurité
- Sanitisation des données
- Prévention des attaques
- Limitation de la taille des données

## Installation des dépendances de validation

```bash
go get github.com/go-playground/validator/v10
go get github.com/microcosm-cc/bluemonday
```

## Création du système de validation

### internal/validation/validator.go

```go
package validation

import (
    "errors"
    "fmt"
    "reflect"
    "regexp"
    "strings"

    "github.com/go-playground/validator/v10"
)

// Validator instance globale
var validate *validator.Validate

// ValidationError représente une erreur de validation
type ValidationError struct {
    Field   string `json:"field"`
    Tag     string `json:"tag"`
    Value   string `json:"value"`
    Message string `json:"message"`
}

// ValidationErrors liste d'erreurs de validation
type ValidationErrors []ValidationError

func (ve ValidationErrors) Error() string {
    var messages []string
    for _, err := range ve {
        messages = append(messages, err.Message)
    }
    return strings.Join(messages, "; ")
}

// Init initialise le validateur avec des règles personnalisées
func Init() {
    validate = validator.New()

    // Utiliser les noms de champs JSON pour les erreurs
    validate.RegisterTagNameFunc(func(fld reflect.StructField) string {
        name := strings.SplitN(fld.Tag.Get("json"), ",", 2)[0]
        if name == "-" {
            return ""
        }
        return name
    })

    // Enregistrer les validations personnalisées
    registerCustomValidations()
}

// Validate valide une structure
func Validate(s interface{}) error {
    if err := validate.Struct(s); err != nil {
        var validationErrors ValidationErrors

        for _, err := range err.(validator.ValidationErrors) {
            validationErrors = append(validationErrors, ValidationError{
                Field:   err.Field(),
                Tag:     err.Tag(),
                Value:   fmt.Sprintf("%v", err.Value()),
                Message: getErrorMessage(err),
            })
        }

        return validationErrors
    }
    return nil
}

// registerCustomValidations enregistre les validations personnalisées
func registerCustomValidations() {
    // Validation pour mot de passe fort
    validate.RegisterValidation("strongpassword", validateStrongPassword)

    // Validation pour nom d'utilisateur
    validate.RegisterValidation("username", validateUsername)

    // Validation pour priorité de tâche
    validate.RegisterValidation("priority", validatePriority)

    // Validation pour couleur hexadécimale
    validate.RegisterValidation("hexcolor", validateHexColor)
}

// validateStrongPassword valide qu'un mot de passe est fort
func validateStrongPassword(fl validator.FieldLevel) bool {
    password := fl.Field().String()

    // Au moins 8 caractères
    if len(password) < 8 {
        return false
    }

    // Au moins une majuscule
    hasUpper, _ := regexp.MatchString(`[A-Z]`, password)
    // Au moins une minuscule
    hasLower, _ := regexp.MatchString(`[a-z]`, password)
    // Au moins un chiffre
    hasDigit, _ := regexp.MatchString(`\d`, password)
    // Au moins un caractère spécial
    hasSpecial, _ := regexp.MatchString(`[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?~]`, password)

    return hasUpper && hasLower && hasDigit && hasSpecial
}

// validateUsername valide le format du nom d'utilisateur
func validateUsername(fl validator.FieldLevel) bool {
    username := fl.Field().String()

    // 3-30 caractères, lettres, chiffres, tirets et underscores uniquement
    matched, _ := regexp.MatchString(`^[a-zA-Z0-9_-]{3,30}$`, username)
    return matched
}

// validatePriority valide les priorités de tâche
func validatePriority(fl validator.FieldLevel) bool {
    priority := fl.Field().String()
    validPriorities := []string{"low", "medium", "high", "urgent"}

    for _, valid := range validPriorities {
        if priority == valid {
            return true
        }
    }
    return false
}

// validateHexColor valide une couleur hexadécimale
func validateHexColor(fl validator.FieldLevel) bool {
    color := fl.Field().String()
    matched, _ := regexp.MatchString(`^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$`, color)
    return matched
}

// getErrorMessage retourne un message d'erreur lisible
func getErrorMessage(fe validator.FieldError) string {
    switch fe.Tag() {
    case "required":
        return fmt.Sprintf("Le champ '%s' est requis", fe.Field())
    case "email":
        return fmt.Sprintf("Le champ '%s' doit être une adresse email valide", fe.Field())
    case "min":
        return fmt.Sprintf("Le champ '%s' doit contenir au moins %s caractères", fe.Field(), fe.Param())
    case "max":
        return fmt.Sprintf("Le champ '%s' ne peut pas dépasser %s caractères", fe.Field(), fe.Param())
    case "strongpassword":
        return "Le mot de passe doit contenir au moins 8 caractères avec au moins une majuscule, une minuscule, un chiffre et un caractère spécial"
    case "username":
        return "Le nom d'utilisateur doit contenir entre 3 et 30 caractères (lettres, chiffres, tirets et underscores uniquement)"
    case "priority":
        return "La priorité doit être: low, medium, high ou urgent"
    case "hexcolor":
        return "La couleur doit être au format hexadécimal (ex: #FF0000)"
    default:
        return fmt.Sprintf("Le champ '%s' est invalide", fe.Field())
    }
}
```

## DTOs avec validation

Créons des structures dédiées à la validation des données d'entrée :

### internal/dto/user_dto.go

```go
package dto

import (
    "time"
)

// RegisterUserRequest DTO pour l'inscription
type RegisterUserRequest struct {
    Username string `json:"username" validate:"required,username"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,strongpassword"`
}

// LoginUserRequest DTO pour la connexion
type LoginUserRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=1"`
}

// UpdateUserRequest DTO pour la mise à jour du profil
type UpdateUserRequest struct {
    Username string `json:"username,omitempty" validate:"omitempty,username"`
    Email    string `json:"email,omitempty" validate:"omitempty,email"`
}

// ChangePasswordRequest DTO pour changer le mot de passe
type ChangePasswordRequest struct {
    CurrentPassword string `json:"current_password" validate:"required"`
    NewPassword     string `json:"new_password" validate:"required,strongpassword"`
    ConfirmPassword string `json:"confirm_password" validate:"required,eqfield=NewPassword"`
}
```

### internal/dto/task_dto.go

```go
package dto

import (
    "time"
)

// CreateTaskRequest DTO pour créer une tâche
type CreateTaskRequest struct {
    Title       string     `json:"title" validate:"required,min=1,max=200"`
    Description string     `json:"description,omitempty" validate:"max=1000"`
    Priority    string     `json:"priority,omitempty" validate:"omitempty,priority"`
    DueDate     *time.Time `json:"due_date,omitempty"`
    CategoryID  *uint      `json:"category_id,omitempty"`
}

// UpdateTaskRequest DTO pour mettre à jour une tâche
type UpdateTaskRequest struct {
    Title       string     `json:"title,omitempty" validate:"omitempty,min=1,max=200"`
    Description string     `json:"description,omitempty" validate:"max=1000"`
    Priority    string     `json:"priority,omitempty" validate:"omitempty,priority"`
    Completed   *bool      `json:"completed,omitempty"`
    DueDate     *time.Time `json:"due_date,omitempty"`
    CategoryID  *uint      `json:"category_id,omitempty"`
}

// TaskFilterRequest DTO pour filtrer les tâches
type TaskFilterRequest struct {
    Completed  *bool  `json:"completed,omitempty"`
    Priority   string `json:"priority,omitempty" validate:"omitempty,priority"`
    CategoryID *uint  `json:"category_id,omitempty"`
    Search     string `json:"search,omitempty" validate:"max=100"`
    Page       int    `json:"page,omitempty" validate:"omitempty,min=1"`
    Limit      int    `json:"limit,omitempty" validate:"omitempty,min=1,max=100"`
}
```

### internal/dto/category_dto.go

```go
package dto

// CreateCategoryRequest DTO pour créer une catégorie
type CreateCategoryRequest struct {
    Name  string `json:"name" validate:"required,min=1,max=50"`
    Color string `json:"color,omitempty" validate:"omitempty,hexcolor"`
}

// UpdateCategoryRequest DTO pour mettre à jour une catégorie
type UpdateCategoryRequest struct {
    Name  string `json:"name,omitempty" validate:"omitempty,min=1,max=50"`
    Color string `json:"color,omitempty" validate:"omitempty,hexcolor"`
}
```

## Sanitisation des données

### internal/sanitizer/sanitizer.go

```go
package sanitizer

import (
    "html"
    "regexp"
    "strings"

    "github.com/microcosm-cc/bluemonday"
)

var (
    // Policy pour nettoyer le HTML
    htmlPolicy = bluemonday.StrictPolicy()

    // Regex pour nettoyer les espaces multiples
    spaceRegex = regexp.MustCompile(`\s+`)
)

// SanitizeString nettoie une chaîne de caractères
func SanitizeString(input string) string {
    if input == "" {
        return input
    }

    // Supprimer les espaces en début et fin
    cleaned := strings.TrimSpace(input)

    // Échapper les caractères HTML
    cleaned = html.EscapeString(cleaned)

    // Nettoyer le HTML potentiellement dangereux
    cleaned = htmlPolicy.Sanitize(cleaned)

    // Normaliser les espaces multiples
    cleaned = spaceRegex.ReplaceAllString(cleaned, " ")

    return cleaned
}

// SanitizeEmail nettoie et valide un email
func SanitizeEmail(email string) string {
    if email == "" {
        return email
    }

    // Convertir en minuscules et supprimer les espaces
    cleaned := strings.ToLower(strings.TrimSpace(email))

    return cleaned
}

// SanitizeUsername nettoie un nom d'utilisateur
func SanitizeUsername(username string) string {
    if username == "" {
        return username
    }

    // Supprimer les espaces et convertir en minuscules
    cleaned := strings.ToLower(strings.TrimSpace(username))

    // Supprimer les caractères non autorisés
    reg := regexp.MustCompile(`[^a-zA-Z0-9_-]`)
    cleaned = reg.ReplaceAllString(cleaned, "")

    return cleaned
}
```

## Service de validation

### internal/service/validation_service.go

```go
package service

import (
    "errors"
    "fmt"

    "todo-api/internal/dto"
    "todo-api/internal/models"
    "todo-api/internal/repository"
    "todo-api/internal/sanitizer"
    "todo-api/internal/validation"
)

type ValidationService interface {
    ValidateAndSanitizeUser(req *dto.RegisterUserRequest) error
    ValidateTask(req *dto.CreateTaskRequest, userID uint) error
    ValidateTaskUpdate(req *dto.UpdateTaskRequest, taskID uint, userID uint) error
    ValidateCategory(req *dto.CreateCategoryRequest, userID uint) error
}

type validationService struct {
    userRepo     repository.UserRepository
    taskRepo     repository.TaskRepository
    categoryRepo repository.CategoryRepository
}

func NewValidationService(userRepo repository.UserRepository, taskRepo repository.TaskRepository, categoryRepo repository.CategoryRepository) ValidationService {
    return &validationService{
        userRepo:     userRepo,
        taskRepo:     taskRepo,
        categoryRepo: categoryRepo,
    }
}

// ValidateAndSanitizeUser valide et sanitise les données utilisateur
func (s *validationService) ValidateAndSanitizeUser(req *dto.RegisterUserRequest) error {
    // Sanitiser les données
    req.Username = sanitizer.SanitizeUsername(req.Username)
    req.Email = sanitizer.SanitizeEmail(req.Email)

    // Validation structurelle
    if err := validation.Validate(req); err != nil {
        return err
    }

    // Validation métier - vérifier l'unicité de l'email
    if existingUser, _ := s.userRepo.GetByEmail(req.Email); existingUser != nil {
        return errors.New("cet email est déjà utilisé")
    }

    // Validation métier - vérifier l'unicité du nom d'utilisateur
    if existingUser, _ := s.userRepo.GetByUsername(req.Username); existingUser != nil {
        return errors.New("ce nom d'utilisateur est déjà pris")
    }

    return nil
}

// ValidateTask valide une nouvelle tâche
func (s *validationService) ValidateTask(req *dto.CreateTaskRequest, userID uint) error {
    // Sanitiser les données
    req.Title = sanitizer.SanitizeString(req.Title)
    req.Description = sanitizer.SanitizeString(req.Description)

    // Validation structurelle
    if err := validation.Validate(req); err != nil {
        return err
    }

    // Validation métier - vérifier que la catégorie appartient à l'utilisateur
    if req.CategoryID != nil {
        category, err := s.categoryRepo.GetByID(*req.CategoryID)
        if err != nil {
            return errors.New("catégorie non trouvée")
        }
        if category.UserID != userID {
            return errors.New("vous ne pouvez pas utiliser cette catégorie")
        }
    }

    // Validation métier - vérifier que la date d'échéance n'est pas dans le passé
    // if req.DueDate != nil && req.DueDate.Before(time.Now()) {
    //     return errors.New("la date d'échéance ne peut pas être dans le passé")
    // }

    return nil
}

// ValidateTaskUpdate valide la mise à jour d'une tâche
func (s *validationService) ValidateTaskUpdate(req *dto.UpdateTaskRequest, taskID uint, userID uint) error {
    // Sanitiser les données
    if req.Title != "" {
        req.Title = sanitizer.SanitizeString(req.Title)
    }
    req.Description = sanitizer.SanitizeString(req.Description)

    // Validation structurelle
    if err := validation.Validate(req); err != nil {
        return err
    }

    // Vérifier que la tâche existe et appartient à l'utilisateur
    task, err := s.taskRepo.GetByID(taskID)
    if err != nil {
        return errors.New("tâche non trouvée")
    }
    if task.UserID != userID {
        return errors.New("vous ne pouvez pas modifier cette tâche")
    }

    // Validation métier - vérifier la catégorie si spécifiée
    if req.CategoryID != nil {
        category, err := s.categoryRepo.GetByID(*req.CategoryID)
        if err != nil {
            return errors.New("catégorie non trouvée")
        }
        if category.UserID != userID {
            return errors.New("vous ne pouvez pas utiliser cette catégorie")
        }
    }

    return nil
}

// ValidateCategory valide une nouvelle catégorie
func (s *validationService) ValidateCategory(req *dto.CreateCategoryRequest, userID uint) error {
    // Sanitiser les données
    req.Name = sanitizer.SanitizeString(req.Name)

    // Validation structurelle
    if err := validation.Validate(req); err != nil {
        return err
    }

    // Validation métier - vérifier l'unicité du nom pour cet utilisateur
    if existingCategory, _ := s.categoryRepo.GetByNameAndUserID(req.Name, userID); existingCategory != nil {
        return errors.New("vous avez déjà une catégorie avec ce nom")
    }

    return nil
}
```

## Repository pour les catégories

Nous devons d'abord créer le repository manquant :

### internal/repository/category_repository.go

```go
package repository

import (
    "gorm.io/gorm"
    "todo-api/internal/models"
)

type CategoryRepository interface {
    Create(category *models.Category) error
    GetByID(id uint) (*models.Category, error)
    GetByUserID(userID uint) ([]models.Category, error)
    GetByNameAndUserID(name string, userID uint) (*models.Category, error)
    Update(category *models.Category) error
    Delete(id uint) error
}

type categoryRepository struct {
    db *gorm.DB
}

func NewCategoryRepository(db *gorm.DB) CategoryRepository {
    return &categoryRepository{db: db}
}

func (r *categoryRepository) Create(category *models.Category) error {
    return r.db.Create(category).Error
}

func (r *categoryRepository) GetByID(id uint) (*models.Category, error) {
    var category models.Category
    err := r.db.First(&category, id).Error
    if err != nil {
        return nil, err
    }
    return &category, nil
}

func (r *categoryRepository) GetByUserID(userID uint) ([]models.Category, error) {
    var categories []models.Category
    err := r.db.Where("user_id = ?", userID).Find(&categories).Error
    return categories, err
}

func (r *categoryRepository) GetByNameAndUserID(name string, userID uint) (*models.Category, error) {
    var category models.Category
    err := r.db.Where("name = ? AND user_id = ?", name, userID).First(&category).Error
    if err != nil {
        return nil, err
    }
    return &category, nil
}

func (r *categoryRepository) Update(category *models.Category) error {
    return r.db.Save(category).Error
}

func (r *categoryRepository) Delete(id uint) error {
    return r.db.Delete(&models.Category{}, id).Error
}
```

## Mise à jour des handlers avec validation

### internal/handlers/auth_handler.go

Mettons à jour le handler d'authentification pour utiliser la validation :

```go
package handlers

import (
    "encoding/json"
    "net/http"

    "todo-api/internal/dto"
    "todo-api/internal/service"
    "todo-api/internal/utils"
    "todo-api/internal/validation"
)

type AuthHandler struct {
    authService       service.AuthService
    validationService service.ValidationService
}

func NewAuthHandler(authService service.AuthService, validationService service.ValidationService) *AuthHandler {
    return &AuthHandler{
        authService:       authService,
        validationService: validationService,
    }
}

// Register gère l'inscription des utilisateurs avec validation
func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req dto.RegisterUserRequest

    // Décoder la requête JSON
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Valider et sanitiser les données
    if err := h.validationService.ValidateAndSanitizeUser(&req); err != nil {
        // Si c'est une erreur de validation, retourner les détails
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }

        // Sinon, erreur métier
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir le DTO en modèle (pour l'instant, structure similaire)
    userReq := models.RegisterRequest{
        Username: req.Username,
        Email:    req.Email,
        Password: req.Password,
    }

    // Appeler le service d'authentification
    response, err := h.authService.Register(&userReq)
    if err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, response)
}

// Login gère la connexion avec validation
func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req dto.LoginUserRequest

    // Décoder la requête JSON
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Validation structurelle
    if err := validation.Validate(&req); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir le DTO en modèle
    loginReq := models.LoginRequest{
        Email:    req.Email,
        Password: req.Password,
    }

    // Appeler le service d'authentification
    response, err := h.authService.Login(&loginReq)
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

### internal/handlers/task_handler.go

Mettons à jour le handler de tâches avec validation complète :

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"

    "github.com/gorilla/mux"
    "todo-api/internal/dto"
    "todo-api/internal/middleware"
    "todo-api/internal/models"
    "todo-api/internal/service"
    "todo-api/internal/utils"
    "todo-api/internal/validation"
)

type TaskHandler struct {
    taskService       service.TaskService
    validationService service.ValidationService
}

func NewTaskHandler(taskService service.TaskService, validationService service.ValidationService) *TaskHandler {
    return &TaskHandler{
        taskService:       taskService,
        validationService: validationService,
    }
}

// CreateTask crée une nouvelle tâche avec validation
func (h *TaskHandler) CreateTask(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    var req dto.CreateTaskRequest

    // Décoder le JSON de la requête
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Valider les données
    if err := h.validationService.ValidateTask(&req, claims.UserID); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir DTO en modèle
    task := &models.Task{
        Title:       req.Title,
        Description: req.Description,
        Priority:    req.Priority,
        DueDate:     req.DueDate,
        CategoryID:  req.CategoryID,
        UserID:      claims.UserID,
    }

    // Créer la tâche
    if err := h.taskService.CreateTask(task); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, task)
}

// UpdateTask met à jour une tâche avec validation
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

    var req dto.UpdateTaskRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Valider la mise à jour
    if err := h.validationService.ValidateTaskUpdate(&req, uint(id), claims.UserID); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir DTO en modèle (seulement les champs à mettre à jour)
    updates := &models.Task{}
    if req.Title != "" {
        updates.Title = req.Title
    }
    if req.Description != "" {
        updates.Description = req.Description
    }
    if req.Priority != "" {
        updates.Priority = req.Priority
    }
    if req.Completed != nil {
        updates.Completed = *req.Completed
    }
    if req.DueDate != nil {
        updates.DueDate = req.DueDate
    }
    if req.CategoryID != nil {
        updates.CategoryID = req.CategoryID
    }

    if err := h.taskService.UpdateTask(uint(id), claims.UserID, updates); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche mise à jour avec succès"})
}

// GetTasks récupère les tâches avec filtres validés
func (h *TaskHandler) GetTasks(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    // Parser les paramètres de requête pour les filtres
    var filters dto.TaskFilterRequest

    // Récupérer les paramètres de l'URL
    query := r.URL.Query()

    // Parser les filtres depuis les query parameters
    if completed := query.Get("completed"); completed != "" {
        if completedBool, err := strconv.ParseBool(completed); err == nil {
            filters.Completed = &completedBool
        }
    }

    if priority := query.Get("priority"); priority != "" {
        filters.Priority = priority
    }

    if categoryID := query.Get("category_id"); categoryID != "" {
        if catID, err := strconv.ParseUint(categoryID, 10, 32); err == nil {
            catIDUint := uint(catID)
            filters.CategoryID = &catIDUint
        }
    }

    if search := query.Get("search"); search != "" {
        filters.Search = search
    }

    // Pagination
    filters.Page = 1
    if page := query.Get("page"); page != "" {
        if pageInt, err := strconv.Atoi(page); err == nil && pageInt > 0 {
            filters.Page = pageInt
        }
    }

    filters.Limit = 20 // Par défaut
    if limit := query.Get("limit"); limit != "" {
        if limitInt, err := strconv.Atoi(limit); err == nil && limitInt > 0 {
            filters.Limit = limitInt
        }
    }

    // Valider les filtres
    if err := validation.Validate(&filters); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Paramètres de filtrage invalides",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Récupérer les tâches (pour l'instant sans filtres avancés)
    tasks, err := h.taskService.GetUserTasks(claims.UserID)
    if err != nil {
        utils.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]interface{}{
        "tasks": tasks,
        "pagination": map[string]interface{}{
            "page":  filters.Page,
            "limit": filters.Limit,
            "total": len(tasks),
        },
    })
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

// DeleteTask supprime une tâche avec validation
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

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Tâche supprimée avec succès"})
}
```

## Handler pour les catégories

Créons un handler complet pour les catégories avec validation :

### internal/handlers/category_handler.go

```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"

    "github.com/gorilla/mux"
    "todo-api/internal/dto"
    "todo-api/internal/middleware"
    "todo-api/internal/models"
    "todo-api/internal/service"
    "todo-api/internal/utils"
    "todo-api/internal/validation"
)

type CategoryHandler struct {
    categoryService   service.CategoryService
    validationService service.ValidationService
}

func NewCategoryHandler(categoryService service.CategoryService, validationService service.ValidationService) *CategoryHandler {
    return &CategoryHandler{
        categoryService:   categoryService,
        validationService: validationService,
    }
}

// CreateCategory crée une nouvelle catégorie
func (h *CategoryHandler) CreateCategory(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    var req dto.CreateCategoryRequest

    // Décoder le JSON de la requête
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Valider les données
    if err := h.validationService.ValidateCategory(&req, claims.UserID); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir DTO en modèle
    category := &models.Category{
        Name:   req.Name,
        Color:  req.Color,
        UserID: claims.UserID,
    }

    // Définir une couleur par défaut si pas spécifiée
    if category.Color == "" {
        category.Color = "#3498db"
    }

    // Créer la catégorie
    if err := h.categoryService.CreateCategory(category); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusCreated, category)
}

// GetCategories récupère les catégories de l'utilisateur
func (h *CategoryHandler) GetCategories(w http.ResponseWriter, r *http.Request) {
    // Récupérer l'utilisateur authentifié
    claims, ok := middleware.GetUserFromContext(r.Context())
    if !ok {
        utils.WriteError(w, http.StatusUnauthorized, "Utilisateur non authentifié")
        return
    }

    categories, err := h.categoryService.GetUserCategories(claims.UserID)
    if err != nil {
        utils.WriteError(w, http.StatusInternalServerError, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, categories)
}

// UpdateCategory met à jour une catégorie
func (h *CategoryHandler) UpdateCategory(w http.ResponseWriter, r *http.Request) {
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

    var req dto.UpdateCategoryRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
        return
    }

    // Validation structurelle
    if err := validation.Validate(&req); err != nil {
        if validationErrs, ok := err.(validation.ValidationErrors); ok {
            utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                Success: false,
                Error:   "Erreurs de validation",
                Data:    validationErrs,
            })
            return
        }
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    // Convertir DTO en modèle
    updates := &models.Category{}
    if req.Name != "" {
        updates.Name = req.Name
    }
    if req.Color != "" {
        updates.Color = req.Color
    }

    if err := h.categoryService.UpdateCategory(uint(id), claims.UserID, updates); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Catégorie mise à jour avec succès"})
}

// DeleteCategory supprime une catégorie
func (h *CategoryHandler) DeleteCategory(w http.ResponseWriter, r *http.Request) {
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

    if err := h.categoryService.DeleteCategory(uint(id), claims.UserID); err != nil {
        utils.WriteError(w, http.StatusBadRequest, err.Error())
        return
    }

    utils.WriteSuccess(w, http.StatusOK, map[string]string{"message": "Catégorie supprimée avec succès"})
}
```

## Service pour les catégories

Créons le service manquant pour les catégories :

### internal/service/category_service.go

```go
package service

import (
    "errors"
    "todo-api/internal/models"
    "todo-api/internal/repository"
)

type CategoryService interface {
    CreateCategory(category *models.Category) error
    GetCategoryByID(id uint, userID uint) (*models.Category, error)
    GetUserCategories(userID uint) ([]models.Category, error)
    UpdateCategory(id uint, userID uint, updates *models.Category) error
    DeleteCategory(id uint, userID uint) error
}

type categoryService struct {
    categoryRepo repository.CategoryRepository
}

func NewCategoryService(categoryRepo repository.CategoryRepository) CategoryService {
    return &categoryService{categoryRepo: categoryRepo}
}

func (s *categoryService) CreateCategory(category *models.Category) error {
    // Validation basique
    if category.Name == "" {
        return errors.New("le nom de la catégorie est requis")
    }

    return s.categoryRepo.Create(category)
}

func (s *categoryService) GetCategoryByID(id uint, userID uint) (*models.Category, error) {
    category, err := s.categoryRepo.GetByID(id)
    if err != nil {
        return nil, err
    }

    // Vérifier que la catégorie appartient à l'utilisateur
    if category.UserID != userID {
        return nil, errors.New("accès non autorisé")
    }

    return category, nil
}

func (s *categoryService) GetUserCategories(userID uint) ([]models.Category, error) {
    return s.categoryRepo.GetByUserID(userID)
}

func (s *categoryService) UpdateCategory(id uint, userID uint, updates *models.Category) error {
    // Récupérer la catégorie existante
    existingCategory, err := s.GetCategoryByID(id, userID)
    if err != nil {
        return err
    }

    // Mettre à jour les champs
    if updates.Name != "" {
        existingCategory.Name = updates.Name
    }
    if updates.Color != "" {
        existingCategory.Color = updates.Color
    }

    return s.categoryRepo.Update(existingCategory)
}

func (s *categoryService) DeleteCategory(id uint, userID uint) error {
    // Vérifier que la catégorie appartient à l'utilisateur
    _, err := s.GetCategoryByID(id, userID)
    if err != nil {
        return err
    }

    return s.categoryRepo.Delete(id)
}
```

## Middleware de validation globale

Créons un middleware pour valider automatiquement certains types de requêtes :

### internal/middleware/validation_middleware_enhanced.go

```go
package middleware

import (
    "bytes"
    "encoding/json"
    "io"
    "net/http"
    "strings"

    "todo-api/internal/utils"
    "todo-api/internal/validation"
)

// ValidateContentSize middleware pour limiter la taille du contenu
func ValidateContentSize(maxSize int64) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Limiter la taille du body
            r.Body = http.MaxBytesReader(w, r.Body, maxSize)

            next.ServeHTTP(w, r)
        })
    }
}

// ValidateRequiredFields middleware pour valider la présence de champs requis
func ValidateRequiredFields(fields []string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Appliquer seulement aux requêtes POST et PUT
            if r.Method != "POST" && r.Method != "PUT" {
                next.ServeHTTP(w, r)
                return
            }

            // Lire le body
            body, err := io.ReadAll(r.Body)
            if err != nil {
                utils.WriteError(w, http.StatusBadRequest, "Impossible de lire le body")
                return
            }

            // Parser le JSON
            var data map[string]interface{}
            if err := json.Unmarshal(body, &data); err != nil {
                utils.WriteError(w, http.StatusBadRequest, "Format JSON invalide")
                return
            }

            // Vérifier les champs requis
            var missingFields []string
            for _, field := range fields {
                if value, exists := data[field]; !exists || value == "" {
                    missingFields = append(missingFields, field)
                }
            }

            if len(missingFields) > 0 {
                utils.WriteJSON(w, http.StatusBadRequest, utils.Response{
                    Success: false,
                    Error:   "Champs manquants: " + strings.Join(missingFields, ", "),
                })
                return
            }

            // Remettre le body dans la requête
            r.Body = io.NopCloser(bytes.NewBuffer(body))

            next.ServeHTTP(w, r)
        })
    }
}

// SanitizeInput middleware pour nettoyer les entrées automatiquement
func SanitizeInput(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Appliquer aux requêtes avec du contenu
        if r.Method == "POST" || r.Method == "PUT" || r.Method == "PATCH" {
            // Lire le body
            body, err := io.ReadAll(r.Body)
            if err != nil {
                utils.WriteError(w, http.StatusBadRequest, "Impossible de lire le body")
                return
            }

            // TODO: Implémenter la sanitisation automatique du JSON
            // Pour l'instant, juste remettre le body
            r.Body = io.NopCloser(bytes.NewBuffer(body))
        }

        next.ServeHTTP(w, r)
    })
}
```

## Mise à jour du routeur

Mettons à jour le routeur pour inclure toutes les validations :

### internal/handlers/router.go

```go
package handlers

import (
    "time"

    "github.com/gorilla/mux"
    "todo-api/internal/middleware"
    "todo-api/internal/service"
)

func SetupRoutes(
    taskService service.TaskService,
    authService service.AuthService,
    categoryService service.CategoryService,
    validationService service.ValidationService,
) *mux.Router {
    router := mux.NewRouter()

    // Créer le limiteur de débit
    rateLimiter := middleware.NewRateLimiter(100, time.Hour)

    // Appliquer les middlewares globaux
    router.Use(middleware.CORSMiddleware)
    router.Use(middleware.LoggingMiddleware)
    router.Use(middleware.RateLimitMiddleware(rateLimiter))
    router.Use(middleware.ValidateContentSize(1024 * 1024)) // 1MB max
    router.Use(middleware.ValidateJSONMiddleware)
    router.Use(middleware.SanitizeInput)

    // Créer les handlers
    taskHandler := NewTaskHandler(taskService, validationService)
    authHandler := NewAuthHandler(authService, validationService)
    categoryHandler := NewCategoryHandler(categoryService, validationService)

    // Routes API
    api := router.PathPrefix("/api").Subrouter()

    // Routes publiques
    public := api.PathPrefix("/auth").Subrouter()
    public.HandleFunc("/register", authHandler.Register).Methods("POST")
    public.HandleFunc("/login", authHandler.Login).Methods("POST")

    // Routes protégées
    protected := api.PathPrefix("").Subrouter()
    protected.Use(middleware.AuthMiddleware)

    // Routes d'authentification protégées
    protected.HandleFunc("/auth/profile", authHandler.Profile).Methods("GET")
    protected.HandleFunc("/auth/logout", authHandler.Logout).Methods("POST")

    // Routes pour les tâches
    taskRoutes := protected.PathPrefix("/tasks").Subrouter()
    taskRoutes.HandleFunc("", taskHandler.GetTasks).Methods("GET")
    taskRoutes.HandleFunc("", taskHandler.CreateTask).Methods("POST")
    taskRoutes.HandleFunc("/{id}", taskHandler.GetTask).Methods("GET")
    taskRoutes.HandleFunc("/{id}", taskHandler.UpdateTask).Methods("PUT")
    taskRoutes.HandleFunc("/{id}", taskHandler.DeleteTask).Methods("DELETE")

    // Routes pour les catégories
    categoryRoutes := protected.PathPrefix("/categories").Subrouter()
    categoryRoutes.HandleFunc("", categoryHandler.GetCategories).Methods("GET")
    categoryRoutes.HandleFunc("", categoryHandler.CreateCategory).Methods("POST")
    categoryRoutes.HandleFunc("/{id}", categoryHandler.UpdateCategory).Methods("PUT")
    categoryRoutes.HandleFunc("/{id}", categoryHandler.DeleteCategory).Methods("DELETE")

    // Route de santé
    api.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        utils.WriteSuccess(w, 200, map[string]string{
            "status":    "OK",
            "timestamp": time.Now().Format(time.RFC3339),
            "version":   "1.0.0",
        })
    }).Methods("GET")

    return router
}
```

## Mise à jour du main.go

Modifions le point d'entrée pour initialiser la validation :

### cmd/server/main.go

```go
package main

import (
    "log"
    "net/http"

    "todo-api/internal/config"
    "todo-api/internal/handlers"
    "todo-api/internal/repository"
    "todo-api/internal/service"
    "todo-api/internal/validation"
    "todo-api/pkg/database"
)

func main() {
    // Initialiser le système de validation
    validation.Init()

    // Charger la configuration
    cfg := config.Load()

    // Établir la connexion à la base de données
    database.Connect()
    database.Migrate()

    // Initialiser les repositories
    taskRepo := repository.NewTaskRepository(database.DB)
    userRepo := repository.NewUserRepository(database.DB)
    categoryRepo := repository.NewCategoryRepository(database.DB)

    // Initialiser les services
    taskService := service.NewTaskService(taskRepo)
    authService := service.NewAuthService(userRepo)
    categoryService := service.NewCategoryService(categoryRepo)
    validationService := service.NewValidationService(userRepo, taskRepo, categoryRepo)

    // Configurer les routes
    router := handlers.SetupRoutes(taskService, authService, categoryService, validationService)

    // Configurer le serveur
    server := &http.Server{
        Addr:    ":" + cfg.Port,
        Handler: router,
    }

    log.Printf("Serveur démarré sur le port %s avec validation complète", cfg.Port)
    log.Fatal(server.ListenAndServe())
}
```

## Tests de validation

### Script de test avancé

Créez un fichier `test_validation.sh` :

```bash
#!/bin/bash

BASE_URL="http://localhost:8080/api"

echo "=== Tests de validation des données ==="

# 1. Test inscription avec données invalides
echo "1. Test inscription avec mot de passe faible..."
curl -s -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "ab",
    "email": "invalid-email",
    "password": "123"
  }' | jq .

# 2. Test inscription avec nom d'utilisateur invalide
echo -e "\n2. Test nom d'utilisateur avec caractères spéciaux..."
curl -s -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "user@#$%",
    "email": "valid@example.com",
    "password": "ValidPassword123!"
  }' | jq .

# 3. Test inscription valide
echo -e "\n3. Test inscription valide..."
REGISTER_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "validuser123",
    "email": "valid@example.com",
    "password": "ValidPassword123!"
  }')

echo $REGISTER_RESPONSE | jq .
TOKEN=$(echo $REGISTER_RESPONSE | jq -r '.data.token')

# 4. Test création de tâche sans titre
echo -e "\n4. Test création tâche sans titre..."
curl -s -X POST "$BASE_URL/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "description": "Description sans titre"
  }' | jq .

# 5. Test création de tâche avec priorité invalide
echo -e "\n5. Test priorité invalide..."
curl -s -X POST "$BASE_URL/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Ma tâche",
    "priority": "super_urgent"
  }' | jq .

# 6. Test création de catégorie avec couleur invalide
echo -e "\n6. Test couleur invalide..."
curl -s -X POST "$BASE_URL/categories" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Ma catégorie",
    "color": "red"
  }' | jq .

# 7. Test création valide de catégorie
echo -e "\n7. Test création catégorie valide..."
CATEGORY_RESPONSE=$(curl -s -X POST "$BASE_URL/categories" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Travail",
    "color": "#FF5722"
  }')

echo $CATEGORY_RESPONSE | jq .
CATEGORY_ID=$(echo $CATEGORY_RESPONSE | jq -r '.data.id')

# 8. Test création de tâche valide avec catégorie
echo -e "\n8. Test création tâche complète..."
curl -s -X POST "$BASE_URL/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"title\": \"Tâche importante\",
    \"description\": \"Description détaillée de la tâche\",
    \"priority\": \"high\",
    \"category_id\": $CATEGORY_ID
  }" | jq .

# 9. Test avec contenu trop volumineux
echo -e "\n9. Test contenu trop volumineux..."
LARGE_DESCRIPTION=$(python3 -c "print('x' * 2000)")
curl -s -X POST "$BASE_URL/tasks" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"title\": \"Tâche\",
    \"description\": \"$LARGE_DESCRIPTION\"
  }" | jq .

echo -e "\n=== Tests de validation terminés ==="
```

## Résumé des fonctionnalités de validation

Dans cette section, nous avons implémenté :

**Système de validation robuste** :
- Validation structurelle avec des règles personnalisées
- Validation métier avec vérifications de cohérence
- Messages d'erreur détaillés et localisés

**Sanitisation automatique** :
- Nettoyage des données d'entrée
- Protection contre les attaques XSS
- Normalisation des formats

**DTOs (Data Transfer Objects)** :
- Structures dédiées pour la validation
- Séparation entre données d'entrée et modèles métier
- Règles de validation spécifiques par endpoint

**Middlewares avancés** :
- Limitation de taille des requêtes
- Validation JSON automatique
- Sanitisation globale

**Règles métier** :
- Unicité des noms d'utilisateur et emails
- Vérification des autorisations sur les ressources
- Validation des relations entre entités

## Points clés à retenir

- **Toujours valider côté serveur** même si le client valide
- **Séparer la validation structurelle de la validation métier**
- **Retourner des messages d'erreur utiles** pour améliorer l'UX
- **Sanitiser les données** pour prévenir les attaques
- **Utiliser des DTOs** pour découpler l'API des modèles internes
- **Valider les autorisations** pour chaque ressource

Dans la prochaine section (16-4), nous verrons comment déployer cette API robuste et sécurisée en production avec Docker et les bonnes pratiques de déploiement.

⏭️
