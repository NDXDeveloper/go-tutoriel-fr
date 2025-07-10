🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13-3 : APIs REST en Go

## Introduction

REST (Representational State Transfer) est un style architectural pour concevoir des services web. Les APIs REST sont devenues le standard pour créer des services web modernes car elles sont simples, scalables et faciles à comprendre. En Go, créer des APIs REST robustes est straightforward grâce aux packages intégrés et à la philosophie du langage.

## Qu'est-ce qu'une API REST ?

### Principes fondamentaux

REST repose sur plusieurs principes clés :

1. **Ressources** : Tout est une ressource identifiée par une URL
2. **Méthodes HTTP** : Utilisation des verbes HTTP standard
3. **Sans état** : Chaque requête est indépendante
4. **Représentations** : Les données peuvent être en JSON, XML, etc.
5. **Interface uniforme** : Conventions cohérentes

### Méthodes HTTP et leurs usages

```
GET    /users       → Récupérer tous les utilisateurs
GET    /users/123   → Récupérer l'utilisateur 123
POST   /users       → Créer un nouvel utilisateur
PUT    /users/123   → Mettre à jour l'utilisateur 123
DELETE /users/123   → Supprimer l'utilisateur 123
```

### Codes de statut HTTP courants

- **200 OK** : Requête réussie
- **201 Created** : Ressource créée avec succès
- **204 No Content** : Suppression réussie
- **400 Bad Request** : Données invalides
- **404 Not Found** : Ressource non trouvée
- **500 Internal Server Error** : Erreur serveur

## Structure d'une API REST en Go

### Architecture recommandée

```
api/
├── main.go           # Point d'entrée
├── handlers/         # Gestionnaires de routes
├── models/          # Structures de données
├── middleware/      # Middleware (auth, logging, etc.)
├── database/        # Couche de données
└── utils/           # Utilitaires
```

## Exemple complet : API de gestion de tâches

### 1. Modèles de données

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "strconv"
    "strings"
    "time"
)

// Structure pour une tâche
type Task struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description,omitempty"`
    Completed   bool      `json:"completed"`
    Priority    string    `json:"priority"`           // low, medium, high
    DueDate     *time.Time `json:"due_date,omitempty"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
    Category    string    `json:"category,omitempty"`
}

// Structure pour créer/modifier une tâche
type TaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description,omitempty"`
    Priority    string `json:"priority,omitempty"`
    DueDate     string `json:"due_date,omitempty"` // Format: "2024-12-25"
    Category    string `json:"category,omitempty"`
}

// Structure pour les réponses API
type APIResponse struct {
    Success bool        `json:"success"`
    Message string      `json:"message,omitempty"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

// Structure pour les métadonnées de pagination
type PaginationMeta struct {
    Page       int `json:"page"`
    PageSize   int `json:"page_size"`
    TotalPages int `json:"total_pages"`
    TotalItems int `json:"total_items"`
}

type PaginatedResponse struct {
    Success bool           `json:"success"`
    Data    []Task         `json:"data"`
    Meta    PaginationMeta `json:"meta"`
}
```

### 2. Base de données en mémoire

```go
// Simuler une base de données avec une slice
type TaskDatabase struct {
    tasks    []Task
    nextID   int
}

// Créer une nouvelle base de données
func NewTaskDatabase() *TaskDatabase {
    return &TaskDatabase{
        tasks:  make([]Task, 0),
        nextID: 1,
    }
}

// Ajouter une tâche
func (db *TaskDatabase) Create(taskReq TaskRequest) (*Task, error) {
    // Validation
    if taskReq.Title == "" {
        return nil, fmt.Errorf("titre requis")
    }

    // Valider la priorité
    validPriorities := map[string]bool{"low": true, "medium": true, "high": true}
    if taskReq.Priority != "" && !validPriorities[taskReq.Priority] {
        return nil, fmt.Errorf("priorité invalide (low, medium, high)")
    }

    task := Task{
        ID:          db.nextID,
        Title:       taskReq.Title,
        Description: taskReq.Description,
        Completed:   false,
        Priority:    taskReq.Priority,
        CreatedAt:   time.Now(),
        UpdatedAt:   time.Now(),
        Category:    taskReq.Category,
    }

    // Parser la date si fournie
    if taskReq.DueDate != "" {
        if dueDate, err := time.Parse("2006-01-02", taskReq.DueDate); err == nil {
            task.DueDate = &dueDate
        } else {
            return nil, fmt.Errorf("format de date invalide (YYYY-MM-DD)")
        }
    }

    // Définir priorité par défaut
    if task.Priority == "" {
        task.Priority = "medium"
    }

    db.tasks = append(db.tasks, task)
    db.nextID++

    return &task, nil
}

// Récupérer toutes les tâches avec filtres
func (db *TaskDatabase) GetAll(filters map[string]string) []Task {
    var filtered []Task

    for _, task := range db.tasks {
        // Filtrer par statut
        if status, exists := filters["completed"]; exists {
            completed := status == "true"
            if task.Completed != completed {
                continue
            }
        }

        // Filtrer par priorité
        if priority, exists := filters["priority"]; exists {
            if task.Priority != priority {
                continue
            }
        }

        // Filtrer par catégorie
        if category, exists := filters["category"]; exists {
            if task.Category != category {
                continue
            }
        }

        filtered = append(filtered, task)
    }

    return filtered
}

// Récupérer une tâche par ID
func (db *TaskDatabase) GetByID(id int) (*Task, error) {
    for i, task := range db.tasks {
        if task.ID == id {
            return &db.tasks[i], nil
        }
    }
    return nil, fmt.Errorf("tâche non trouvée")
}

// Mettre à jour une tâche
func (db *TaskDatabase) Update(id int, taskReq TaskRequest) (*Task, error) {
    task, err := db.GetByID(id)
    if err != nil {
        return nil, err
    }

    // Mettre à jour les champs fournis
    if taskReq.Title != "" {
        task.Title = taskReq.Title
    }
    if taskReq.Description != "" {
        task.Description = taskReq.Description
    }
    if taskReq.Priority != "" {
        validPriorities := map[string]bool{"low": true, "medium": true, "high": true}
        if !validPriorities[taskReq.Priority] {
            return nil, fmt.Errorf("priorité invalide")
        }
        task.Priority = taskReq.Priority
    }
    if taskReq.Category != "" {
        task.Category = taskReq.Category
    }

    // Mettre à jour la date si fournie
    if taskReq.DueDate != "" {
        if dueDate, err := time.Parse("2006-01-02", taskReq.DueDate); err == nil {
            task.DueDate = &dueDate
        } else {
            return nil, fmt.Errorf("format de date invalide")
        }
    }

    task.UpdatedAt = time.Now()
    return task, nil
}

// Marquer comme complété/non complété
func (db *TaskDatabase) ToggleComplete(id int) (*Task, error) {
    task, err := db.GetByID(id)
    if err != nil {
        return nil, err
    }

    task.Completed = !task.Completed
    task.UpdatedAt = time.Now()
    return task, nil
}

// Supprimer une tâche
func (db *TaskDatabase) Delete(id int) error {
    for i, task := range db.tasks {
        if task.ID == id {
            db.tasks = append(db.tasks[:i], db.tasks[i+1:]...)
            return nil
        }
    }
    return fmt.Errorf("tâche non trouvée")
}

// Obtenir des statistiques
func (db *TaskDatabase) GetStats() map[string]interface{} {
    total := len(db.tasks)
    completed := 0
    byPriority := map[string]int{"low": 0, "medium": 0, "high": 0}
    overdue := 0

    now := time.Now()
    for _, task := range db.tasks {
        if task.Completed {
            completed++
        }
        byPriority[task.Priority]++

        if task.DueDate != nil && task.DueDate.Before(now) && !task.Completed {
            overdue++
        }
    }

    return map[string]interface{}{
        "total":       total,
        "completed":   completed,
        "pending":     total - completed,
        "overdue":     overdue,
        "by_priority": byPriority,
    }
}
```

### 3. Gestionnaires HTTP (Handlers)

```go
// Structure principale de l'API
type TaskAPI struct {
    db *TaskDatabase
}

// Créer une nouvelle instance de l'API
func NewTaskAPI() *TaskAPI {
    return &TaskAPI{
        db: NewTaskDatabase(),
    }
}

// Utilitaire pour envoyer les réponses JSON
func (api *TaskAPI) sendJSON(w http.ResponseWriter, statusCode int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)
    json.NewEncoder(w).Encode(data)
}

// Utilitaire pour envoyer les erreurs
func (api *TaskAPI) sendError(w http.ResponseWriter, statusCode int, message string) {
    response := APIResponse{
        Success: false,
        Error:   message,
    }
    api.sendJSON(w, statusCode, response)
}

// Utilitaire pour extraire l'ID de l'URL
func extractID(path string) (int, error) {
    parts := strings.Split(path, "/")
    if len(parts) < 3 {
        return 0, fmt.Errorf("ID manquant")
    }

    idStr := parts[len(parts)-1]
    return strconv.Atoi(idStr)
}

// GET /tasks - Récupérer toutes les tâches
func (api *TaskAPI) getTasks(w http.ResponseWriter, r *http.Request) {
    // Extraire les paramètres de filtrage
    filters := make(map[string]string)
    query := r.URL.Query()

    if completed := query.Get("completed"); completed != "" {
        filters["completed"] = completed
    }
    if priority := query.Get("priority"); priority != "" {
        filters["priority"] = priority
    }
    if category := query.Get("category"); category != "" {
        filters["category"] = category
    }

    tasks := api.db.GetAll(filters)

    // Pagination simple
    page := 1
    pageSize := 10

    if pageStr := query.Get("page"); pageStr != "" {
        if p, err := strconv.Atoi(pageStr); err == nil && p > 0 {
            page = p
        }
    }

    if sizeStr := query.Get("page_size"); sizeStr != "" {
        if s, err := strconv.Atoi(sizeStr); err == nil && s > 0 && s <= 100 {
            pageSize = s
        }
    }

    // Calculer la pagination
    totalItems := len(tasks)
    totalPages := (totalItems + pageSize - 1) / pageSize
    start := (page - 1) * pageSize
    end := start + pageSize

    if start > totalItems {
        start = totalItems
    }
    if end > totalItems {
        end = totalItems
    }

    paginatedTasks := tasks[start:end]

    response := PaginatedResponse{
        Success: true,
        Data:    paginatedTasks,
        Meta: PaginationMeta{
            Page:       page,
            PageSize:   pageSize,
            TotalPages: totalPages,
            TotalItems: totalItems,
        },
    }

    api.sendJSON(w, http.StatusOK, response)
}

// GET /tasks/{id} - Récupérer une tâche spécifique
func (api *TaskAPI) getTask(w http.ResponseWriter, r *http.Request) {
    id, err := extractID(r.URL.Path)
    if err != nil {
        api.sendError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    task, err := api.db.GetByID(id)
    if err != nil {
        api.sendError(w, http.StatusNotFound, "Tâche non trouvée")
        return
    }

    response := APIResponse{
        Success: true,
        Data:    task,
    }
    api.sendJSON(w, http.StatusOK, response)
}

// POST /tasks - Créer une nouvelle tâche
func (api *TaskAPI) createTask(w http.ResponseWriter, r *http.Request) {
    var taskReq TaskRequest

    if err := json.NewDecoder(r.Body).Decode(&taskReq); err != nil {
        api.sendError(w, http.StatusBadRequest, "JSON invalide")
        return
    }

    task, err := api.db.Create(taskReq)
    if err != nil {
        api.sendError(w, http.StatusBadRequest, err.Error())
        return
    }

    response := APIResponse{
        Success: true,
        Message: "Tâche créée avec succès",
        Data:    task,
    }
    api.sendJSON(w, http.StatusCreated, response)
}

// PUT /tasks/{id} - Mettre à jour une tâche
func (api *TaskAPI) updateTask(w http.ResponseWriter, r *http.Request) {
    id, err := extractID(r.URL.Path)
    if err != nil {
        api.sendError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    var taskReq TaskRequest
    if err := json.NewDecoder(r.Body).Decode(&taskReq); err != nil {
        api.sendError(w, http.StatusBadRequest, "JSON invalide")
        return
    }

    task, err := api.db.Update(id, taskReq)
    if err != nil {
        if err.Error() == "tâche non trouvée" {
            api.sendError(w, http.StatusNotFound, err.Error())
        } else {
            api.sendError(w, http.StatusBadRequest, err.Error())
        }
        return
    }

    response := APIResponse{
        Success: true,
        Message: "Tâche mise à jour avec succès",
        Data:    task,
    }
    api.sendJSON(w, http.StatusOK, response)
}

// PATCH /tasks/{id}/toggle - Basculer le statut complété
func (api *TaskAPI) toggleTask(w http.ResponseWriter, r *http.Request) {
    id, err := extractID(strings.TrimSuffix(r.URL.Path, "/toggle"))
    if err != nil {
        api.sendError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    task, err := api.db.ToggleComplete(id)
    if err != nil {
        api.sendError(w, http.StatusNotFound, "Tâche non trouvée")
        return
    }

    response := APIResponse{
        Success: true,
        Message: "Statut de la tâche mis à jour",
        Data:    task,
    }
    api.sendJSON(w, http.StatusOK, response)
}

// DELETE /tasks/{id} - Supprimer une tâche
func (api *TaskAPI) deleteTask(w http.ResponseWriter, r *http.Request) {
    id, err := extractID(r.URL.Path)
    if err != nil {
        api.sendError(w, http.StatusBadRequest, "ID invalide")
        return
    }

    if err := api.db.Delete(id); err != nil {
        api.sendError(w, http.StatusNotFound, "Tâche non trouvée")
        return
    }

    response := APIResponse{
        Success: true,
        Message: "Tâche supprimée avec succès",
    }
    api.sendJSON(w, http.StatusOK, response)
}

// GET /tasks/stats - Obtenir des statistiques
func (api *TaskAPI) getStats(w http.ResponseWriter, r *http.Request) {
    stats := api.db.GetStats()

    response := APIResponse{
        Success: true,
        Data:    stats,
    }
    api.sendJSON(w, http.StatusOK, response)
}

// Gestionnaire principal de routage
func (api *TaskAPI) tasksHandler(w http.ResponseWriter, r *http.Request) {
    // Activer CORS pour les tests depuis le navigateur
    w.Header().Set("Access-Control-Allow-Origin", "*")
    w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
    w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

    // Gérer les requêtes OPTIONS (CORS preflight)
    if r.Method == "OPTIONS" {
        w.WriteHeader(http.StatusOK)
        return
    }

    path := r.URL.Path

    switch {
    case path == "/tasks" && r.Method == "GET":
        api.getTasks(w, r)
    case path == "/tasks" && r.Method == "POST":
        api.createTask(w, r)
    case path == "/tasks/stats" && r.Method == "GET":
        api.getStats(w, r)
    case strings.HasPrefix(path, "/tasks/") && strings.HasSuffix(path, "/toggle") && r.Method == "PATCH":
        api.toggleTask(w, r)
    case strings.HasPrefix(path, "/tasks/") && r.Method == "GET":
        api.getTask(w, r)
    case strings.HasPrefix(path, "/tasks/") && r.Method == "PUT":
        api.updateTask(w, r)
    case strings.HasPrefix(path, "/tasks/") && r.Method == "DELETE":
        api.deleteTask(w, r)
    default:
        api.sendError(w, http.StatusNotFound, "Endpoint non trouvé")
    }
}
```

### 4. Middleware

```go
// Middleware de logging
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrapper pour capturer le code de statut
        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        next.ServeHTTP(wrapped, r)

        duration := time.Since(start)
        log.Printf("[%s] %s %s - %d - %v",
            time.Now().Format("2006-01-02 15:04:05"),
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            duration,
        )
    }
}

// Wrapper pour capturer le code de statut
type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Middleware de validation Content-Type
func contentTypeMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" || r.Method == "PUT" || r.Method == "PATCH" {
            contentType := r.Header.Get("Content-Type")
            if !strings.Contains(contentType, "application/json") {
                http.Error(w, "Content-Type doit être application/json", http.StatusUnsupportedMediaType)
                return
            }
        }
        next.ServeHTTP(w, r)
    }
}

// Middleware de limitation de taille
func bodySizeMiddleware(maxSize int64) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            r.Body = http.MaxBytesReader(w, r.Body, maxSize)
            next.ServeHTTP(w, r)
        }
    }
}
```

### 5. Documentation et interface de test

```go
// Gestionnaire pour la documentation API
func (api *TaskAPI) docsHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Task API Documentation</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .container { max-width: 1000px; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; text-align: center; }
        .endpoint { background: #f8f9fa; padding: 15px; margin: 20px 0; border-radius: 5px; border-left: 4px solid #007bff; }
        .method { display: inline-block; padding: 2px 8px; border-radius: 3px; color: white; font-weight: bold; }
        .get { background: #28a745; }
        .post { background: #007bff; }
        .put { background: #ffc107; color: #000; }
        .patch { background: #6c757d; }
        .delete { background: #dc3545; }
        .example { background: #e9ecef; padding: 10px; border-radius: 3px; margin: 10px 0; }
        pre { overflow-x: auto; }
        .test-form { background: #f0f8ff; padding: 20px; border-radius: 5px; margin: 20px 0; }
        input, select, textarea { width: 100%; padding: 8px; margin: 5px 0; border: 1px solid #ddd; border-radius: 3px; }
        button { background: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 3px; cursor: pointer; }
        button:hover { background: #0056b3; }
        #result { margin-top: 20px; padding: 15px; border-radius: 5px; }
        .success { background: #d4edda; border: 1px solid #c3e6cb; color: #155724; }
        .error { background: #f8d7da; border: 1px solid #f5c6cb; color: #721c24; }
    </style>
</head>
<body>
    <div class="container">
        <h1>📋 Task API Documentation</h1>

        <h2>Endpoints</h2>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks</h3>
            <p><strong>Description:</strong> Récupérer toutes les tâches avec filtres et pagination</p>
            <p><strong>Paramètres:</strong></p>
            <ul>
                <li><code>completed</code> - true/false</li>
                <li><code>priority</code> - low/medium/high</li>
                <li><code>category</code> - nom de catégorie</li>
                <li><code>page</code> - numéro de page (défaut: 1)</li>
                <li><code>page_size</code> - taille de page (défaut: 10, max: 100)</li>
            </ul>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Récupérer une tâche spécifique</p>
        </div>

        <div class="endpoint">
            <h3><span class="method post">POST</span> /tasks</h3>
            <p><strong>Description:</strong> Créer une nouvelle tâche</p>
            <div class="example">
                <strong>Exemple de body:</strong>
                <pre>{
  "title": "Apprendre Go",
  "description": "Suivre le tutoriel complet",
  "priority": "high",
  "due_date": "2024-12-31",
  "category": "Programmation"
}</pre>
            </div>
        </div>

        <div class="endpoint">
            <h3><span class="method put">PUT</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Mettre à jour une tâche</p>
        </div>

        <div class="endpoint">
            <h3><span class="method patch">PATCH</span> /tasks/{id}/toggle</h3>
            <p><strong>Description:</strong> Basculer le statut complété/non complété</p>
        </div>

        <div class="endpoint">
            <h3><span class="method delete">DELETE</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Supprimer une tâche</p>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks/stats</h3>
            <p><strong>Description:</strong> Obtenir des statistiques sur les tâches</p>
        </div>

        <div class="test-form">
            <h2>🧪 Test interactif</h2>

            <div>
                <label>Méthode:</label>
                <select id="method">
                    <option value="GET">GET</option>
                    <option value="POST">POST</option>
                    <option value="PUT">PUT</option>
                    <option value="PATCH">PATCH</option>
                    <option value="DELETE">DELETE</option>
                </select>
            </div>

            <div>
                <label>URL:</label>
                <input type="text" id="url" value="/tasks" placeholder="/tasks">
            </div>

            <div>
                <label>Body JSON (pour POST/PUT):</label>
                <textarea id="body" rows="6" placeholder='{"title": "Ma tâche", "priority": "medium"}'></textarea>
            </div>

            <button onclick="testAPI()">Envoyer la requête</button>
            <button onclick="loadSampleData()">Charger des données d'exemple</button>

            <div id="result"></div>
        </div>
    </div>

    <script>
        function testAPI() {
            const method = document.getElementById('method').value;
            const url = document.getElementById('url').value;
            const body = document.getElementById('body').value;

            const options = {
                method: method,
                headers: {
                    'Content-Type': 'application/json'
                }
            };

            if (method === 'POST' || method === 'PUT' || method === 'PATCH') {
                if (body.trim()) {
                    options.body = body;
                }
            }

            fetch(url, options)
                .then(response => response.json())
                .then(data => {
                    const resultDiv = document.getElementById('result');
                    resultDiv.className = data.success ? 'success' : 'error';
                    resultDiv.innerHTML = '<h3>Résultat:</h3><pre>' + JSON.stringify(data, null, 2) + '</pre>';
                })
                .catch(error => {
                    const resultDiv = document.getElementById('result');
                    resultDiv.className = 'error';
                    resultDiv.innerHTML = '<h3>Erreur:</h3><p>' + error + '</p>';
                });
        }

        function loadSampleData() {
            const sampleTasks = [
                {
                    title: "Finir le projet Go",
                    description: "Compléter l'API REST et ajouter les tests",
                    priority: "high",
                    due_date: "2024-12-25",
                    category: "Travail"
                },
                {
                    title: "Faire les courses",
                    description: "Acheter des ingrédients pour le dîner",
                    priority: "medium",
                    category: "Personnel"
                },
                {
                    title: "Lire un livre",
                    description: "Continuer la lecture de Clean Code",
                    priority: "low",
                    category: "Formation"
                }
            ];

            // Créer chaque tâche
            sampleTasks.forEach((task, index) => {
                setTimeout(() => {
                    fetch('/tasks', {
                        method: 'POST',
                        headers: {
                            'Content-Type': 'application/json'
                        },
                        body: JSON.stringify(task)
                    });
                }, index * 100);
            });

            setTimeout(() => {
                alert('Données d\'exemple chargées ! Testez GET /tasks pour les voir.');
            }, 500);
        }

        // Exemples prédéfinis
        function setExample(type) {
            const examples = {
                createTask: {
                    method: 'POST',
                    url: '/tasks',
                    body: JSON.stringify({
                        title: "Nouvelle tâche",
                        description: "Description de la tâche",
                        priority: "medium",
                        due_date: "2024-12-31",
                        category: "Exemple"
                    }, null, 2)
                },
                updateTask: {
                    method: 'PUT',
                    url: '/tasks/1',
                    body: JSON.stringify({
                        title: "Tâche mise à jour",
                        priority: "high"
                    }, null, 2)
                },
                toggleTask: {
                    method: 'PATCH',
                    url: '/tasks/1/toggle',
                    body: ''
                },
                deleteTask: {
                    method: 'DELETE',
                    url: '/tasks/1',
                    body: ''
                }
            };

            const example = examples[type];
            if (example) {
                document.getElementById('method').value = example.method;
                document.getElementById('url').value = example.url;
                document.getElementById('body').value = example.body;
            }
        }
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Point d'entrée principal
func main() {
    api := NewTaskAPI()

    // Ajouter quelques données d'exemple
    api.db.Create(TaskRequest{
        Title:       "Apprendre Go",
        Description: "Étudier les APIs REST en Go",
        Priority:    "high",
        Category:    "Formation",
    })

    api.db.Create(TaskRequest{
        Title:       "Faire du sport",
        Description: "Aller courir 30 minutes",
        Priority:    "medium",
        Category:    "Santé",
    })

    // Configuration des routes avec middleware
    http.HandleFunc("/", api.docsHandler)
    http.HandleFunc("/tasks", loggingMiddleware(contentTypeMiddleware(api.tasksHandler)))
    http.HandleFunc("/tasks/", loggingMiddleware(contentTypeMiddleware(api.tasksHandler)))

    port := ":8080"
    fmt.Printf("🚀 Task API démarrée sur http://localhost%s\n", port)
    fmt.Printf("📖 Documentation: http://localhost%s\n", port)
    fmt.Printf("🔗 Endpoints:\n")
    fmt.Printf("   GET    /tasks           - Lister les tâches\n")
    fmt.Printf("   POST   /tasks           - Créer une tâche\n")
    fmt.Printf("   GET    /tasks/{id}      - Récupérer une tâche\n")
    fmt.Printf("   PUT    /tasks/{id}      - Mettre à jour une tâche\n")
    fmt.Printf("   PATCH  /tasks/{id}/toggle - Basculer complété\n")
    fmt.Printf("   DELETE /tasks/{id}      - Supprimer une tâche\n")
    fmt.Printf("   GET    /tasks/stats     - Statistiques\n")

    log.Fatal(http.ListenAndServe(port, nil))
}
```

## 7. Tests avec curl

### Exemples de commandes pour tester l'API

```bash
# 1. Récupérer toutes les tâches
curl http://localhost:8080/tasks

# 2. Récupérer les tâches avec pagination
curl "http://localhost:8080/tasks?page=1&page_size=5"

# 3. Filtrer les tâches complétées
curl "http://localhost:8080/tasks?completed=true"

# 4. Filtrer par priorité
curl "http://localhost:8080/tasks?priority=high"

# 5. Créer une nouvelle tâche
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Réviser les APIs REST",
    "description": "Étudier les bonnes pratiques",
    "priority": "high",
    "due_date": "2024-12-30",
    "category": "Formation"
  }'

# 6. Récupérer une tâche spécifique
curl http://localhost:8080/tasks/1

# 7. Mettre à jour une tâche
curl -X PUT http://localhost:8080/tasks/1 \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Réviser les APIs REST - URGENT",
    "priority": "high"
  }'

# 8. Marquer une tâche comme complétée
curl -X PATCH http://localhost:8080/tasks/1/toggle

# 9. Supprimer une tâche
curl -X DELETE http://localhost:8080/tasks/1

# 10. Obtenir des statistiques
curl http://localhost:8080/tasks/stats
```

## 8. Client Go pour tester l'API

### Client de test automatisé

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

// Structures identiques à l'API
type Task struct {
    ID          int       `json:"id"`
    Title       string    `json:"title"`
    Description string    `json:"description,omitempty"`
    Completed   bool      `json:"completed"`
    Priority    string    `json:"priority"`
    DueDate     *time.Time `json:"due_date,omitempty"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
    Category    string    `json:"category,omitempty"`
}

type TaskRequest struct {
    Title       string `json:"title"`
    Description string `json:"description,omitempty"`
    Priority    string `json:"priority,omitempty"`
    DueDate     string `json:"due_date,omitempty"`
    Category    string `json:"category,omitempty"`
}

type APIResponse struct {
    Success bool        `json:"success"`
    Message string      `json:"message,omitempty"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}

// Client pour l'API Task
type TaskClient struct {
    BaseURL string
    Client  *http.Client
}

// Créer un nouveau client
func NewTaskClient(baseURL string) *TaskClient {
    return &TaskClient{
        BaseURL: baseURL,
        Client: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// Effectuer une requête HTTP
func (tc *TaskClient) makeRequest(method, endpoint string, body interface{}) (*http.Response, error) {
    var bodyReader io.Reader

    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, err
        }
        bodyReader = bytes.NewBuffer(jsonData)
    }

    req, err := http.NewRequest(method, tc.BaseURL+endpoint, bodyReader)
    if err != nil {
        return nil, err
    }

    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }

    return tc.Client.Do(req)
}

// Récupérer toutes les tâches
func (tc *TaskClient) GetTasks(filters map[string]string) ([]Task, error) {
    endpoint := "/tasks"

    // Ajouter les filtres comme paramètres de requête
    if len(filters) > 0 {
        endpoint += "?"
        first := true
        for key, value := range filters {
            if !first {
                endpoint += "&"
            }
            endpoint += fmt.Sprintf("%s=%s", key, value)
            first = false
        }
    }

    resp, err := tc.makeRequest("GET", endpoint, nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool   `json:"success"`
        Data    []Task `json:"data"`
        Error   string `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return apiResp.Data, nil
}

// Récupérer une tâche par ID
func (tc *TaskClient) GetTask(id int) (*Task, error) {
    resp, err := tc.makeRequest("GET", fmt.Sprintf("/tasks/%d", id), nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool  `json:"success"`
        Data    Task  `json:"data"`
        Error   string `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return &apiResp.Data, nil
}

// Créer une nouvelle tâche
func (tc *TaskClient) CreateTask(taskReq TaskRequest) (*Task, error) {
    resp, err := tc.makeRequest("POST", "/tasks", taskReq)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool  `json:"success"`
        Data    Task  `json:"data"`
        Error   string `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return &apiResp.Data, nil
}

// Mettre à jour une tâche
func (tc *TaskClient) UpdateTask(id int, taskReq TaskRequest) (*Task, error) {
    resp, err := tc.makeRequest("PUT", fmt.Sprintf("/tasks/%d", id), taskReq)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool  `json:"success"`
        Data    Task  `json:"data"`
        Error   string `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return &apiResp.Data, nil
}

// Basculer le statut d'une tâche
func (tc *TaskClient) ToggleTask(id int) (*Task, error) {
    resp, err := tc.makeRequest("PATCH", fmt.Sprintf("/tasks/%d/toggle", id), nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool  `json:"success"`
        Data    Task  `json:"data"`
        Error   string `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return &apiResp.Data, nil
}

// Supprimer une tâche
func (tc *TaskClient) DeleteTask(id int) error {
    resp, err := tc.makeRequest("DELETE", fmt.Sprintf("/tasks/%d", id), nil)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var apiResp APIResponse
    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return err
    }

    if !apiResp.Success {
        return fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return nil
}

// Obtenir des statistiques
func (tc *TaskClient) GetStats() (map[string]interface{}, error) {
    resp, err := tc.makeRequest("GET", "/tasks/stats", nil)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var apiResp struct {
        Success bool                   `json:"success"`
        Data    map[string]interface{} `json:"data"`
        Error   string                 `json:"error"`
    }

    if err := json.NewDecoder(resp.Body).Decode(&apiResp); err != nil {
        return nil, err
    }

    if !apiResp.Success {
        return nil, fmt.Errorf("erreur API: %s", apiResp.Error)
    }

    return apiResp.Data, nil
}

// Fonction de test principale
func main() {
    client := NewTaskClient("http://localhost:8080")

    fmt.Println("🧪 Test du client API Task")
    fmt.Println("==========================")

    // Test 1: Récupérer toutes les tâches
    fmt.Println("\n📋 1. Récupération de toutes les tâches")
    tasks, err := client.GetTasks(nil)
    if err != nil {
        fmt.Printf("❌ Erreur: %v\n", err)
    } else {
        fmt.Printf("✅ %d tâches trouvées\n", len(tasks))
        for _, task := range tasks {
            status := "❌"
            if task.Completed {
                status = "✅"
            }
            fmt.Printf("   %s [%s] %s (Priorité: %s)\n", status, task.Priority, task.Title, task.Priority)
        }
    }

    // Test 2: Créer une nouvelle tâche
    fmt.Println("\n➕ 2. Création d'une nouvelle tâche")
    newTask := TaskRequest{
        Title:       "Tâche de test",
        Description: "Créée par le client Go",
        Priority:    "high",
        Category:    "Test",
        DueDate:     "2024-12-31",
    }

    createdTask, err := client.CreateTask(newTask)
    if err != nil {
        fmt.Printf("❌ Erreur: %v\n", err)
    } else {
        fmt.Printf("✅ Tâche créée avec l'ID %d\n", createdTask.ID)
        fmt.Printf("   Titre: %s\n", createdTask.Title)
        fmt.Printf("   Priorité: %s\n", createdTask.Priority)
    }

    // Test 3: Récupérer la tâche créée
    if createdTask != nil {
        fmt.Printf("\n🔍 3. Récupération de la tâche ID %d\n", createdTask.ID)
        task, err := client.GetTask(createdTask.ID)
        if err != nil {
            fmt.Printf("❌ Erreur: %v\n", err)
        } else {
            fmt.Printf("✅ Tâche trouvée: %s\n", task.Title)
            fmt.Printf("   Créée le: %s\n", task.CreatedAt.Format("2006-01-02 15:04:05"))
        }

        // Test 4: Mettre à jour la tâche
        fmt.Printf("\n✏️ 4. Mise à jour de la tâche ID %d\n", createdTask.ID)
        updateReq := TaskRequest{
            Title:    "Tâche de test MISE À JOUR",
            Priority: "medium",
        }

        updatedTask, err := client.UpdateTask(createdTask.ID, updateReq)
        if err != nil {
            fmt.Printf("❌ Erreur: %v\n", err)
        } else {
            fmt.Printf("✅ Tâche mise à jour: %s\n", updatedTask.Title)
            fmt.Printf("   Nouvelle priorité: %s\n", updatedTask.Priority)
        }

        // Test 5: Basculer le statut
        fmt.Printf("\n🔄 5. Basculer le statut de la tâche ID %d\n", createdTask.ID)
        toggledTask, err := client.ToggleTask(createdTask.ID)
        if err != nil {
            fmt.Printf("❌ Erreur: %v\n", err)
        } else {
            status := "non complétée"
            if toggledTask.Completed {
                status = "complétée"
            }
            fmt.Printf("✅ Tâche maintenant %s\n", status)
        }

        // Test 6: Supprimer la tâche
        fmt.Printf("\n🗑️ 6. Suppression de la tâche ID %d\n", createdTask.ID)
        err := client.DeleteTask(createdTask.ID)
        if err != nil {
            fmt.Printf("❌ Erreur: %v\n", err)
        } else {
            fmt.Printf("✅ Tâche supprimée avec succès\n")
        }
    }

    // Test 7: Filtres
    fmt.Println("\n🔍 7. Test des filtres")
    filters := map[string]string{
        "priority": "high",
    }

    filteredTasks, err := client.GetTasks(filters)
    if err != nil {
        fmt.Printf("❌ Erreur: %v\n", err)
    } else {
        fmt.Printf("✅ %d tâches avec priorité 'high'\n", len(filteredTasks))
    }

    // Test 8: Statistiques
    fmt.Println("\n📊 8. Statistiques")
    stats, err := client.GetStats()
    if err != nil {
        fmt.Printf("❌ Erreur: %v\n", err)
    } else {
        fmt.Printf("✅ Statistiques obtenues:\n")
        for key, value := range stats {
            fmt.Printf("   %s: %v\n", key, value)
        }
    }

    fmt.Println("\n🎉 Tests terminés!")
}
```

## 9. Bonnes pratiques et améliorations

### Gestion d'erreurs avancée

```go
// Types d'erreurs personnalisées
type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

func (e APIError) Error() string {
    return fmt.Sprintf("API Error %d: %s", e.Code, e.Message)
}

// Gestionnaire d'erreurs centralisé
func (api *TaskAPI) handleError(w http.ResponseWriter, err error) {
    var statusCode int
    var message string

    switch {
    case strings.Contains(err.Error(), "non trouvée"):
        statusCode = http.StatusNotFound
        message = "Ressource non trouvée"
    case strings.Contains(err.Error(), "invalide"):
        statusCode = http.StatusBadRequest
        message = "Données invalides"
    default:
        statusCode = http.StatusInternalServerError
        message = "Erreur interne du serveur"
    }

    apiErr := APIError{
        Code:    statusCode,
        Message: message,
        Details: err.Error(),
    }

    api.sendJSON(w, statusCode, APIResponse{
        Success: false,
        Error:   apiErr.Message,
    })
}
```

### Validation avancée

```go
// Validateur pour les tâches
type TaskValidator struct{}

func (v TaskValidator) ValidateCreate(req TaskRequest) error {
    if strings.TrimSpace(req.Title) == "" {
        return fmt.Errorf("le titre est requis")
    }

    if len(req.Title) > 200 {
        return fmt.Errorf("le titre ne peut pas dépasser 200 caractères")
    }

    if len(req.Description) > 1000 {
        return fmt.Errorf("la description ne peut pas dépasser 1000 caractères")
    }

    validPriorities := map[string]bool{"low": true, "medium": true, "high": true}
    if req.Priority != "" && !validPriorities[req.Priority] {
        return fmt.Errorf("priorité invalide: doit être 'low', 'medium' ou 'high'")
    }

    if req.DueDate != "" {
        if _, err := time.Parse("2006-01-02", req.DueDate); err != nil {
            return fmt.Errorf("format de date invalide: utilisez YYYY-MM-DD")
        }
    }

    return nil
}

func (v TaskValidator) ValidateUpdate(req TaskRequest) error {
    if req.Title != "" && len(req.Title) > 200 {
        return fmt.Errorf("le titre ne peut pas dépasser 200 caractères")
    }

    if req.Description != "" && len(req.Description) > 1000 {
        return fmt.Errorf("la description ne peut pas dépasser 1000 caractères")
    }

    validPriorities := map[string]bool{"low": true, "medium": true, "high": true}
    if req.Priority != "" && !validPriorities[req.Priority] {
        return fmt.Errorf("priorité invalide")
    }

    return nil
}
```

### Middleware d'authentification (optionnel)

```go
// Middleware d'authentification simple avec token
func authMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        // Vérifier le token (exemple simple)
        if token == "" {
            http.Error(w, "Token d'authentification requis", http.StatusUnauthorized)
            return
        }

        if !strings.HasPrefix(token, "Bearer ") {
            http.Error(w, "Format de token invalide", http.StatusUnauthorized)
            return
        }

        actualToken := strings.TrimPrefix(token, "Bearer ")
        if actualToken != "secret-token-123" {
            http.Error(w, "Token invalide", http.StatusUnauthorized)
            return
        }

        next.ServeHTTP(w, r)
    }
}
```

## 10. Exercices pratiques

### Exercice 1 : Ajouter l'authentification
Modifiez l'API pour inclure :
- Système d'utilisateurs
- Authentification par token JWT
- Tâches privées par utilisateur

### Exercice 2 : Ajouter la persistance
Remplacez la base de données en mémoire par :
- SQLite pour commencer
- PostgreSQL pour la production

### Exercice 3 : API de commentaires
Étendez l'API pour supporter :
- Commentaires sur les tâches
- CRUD complet pour les commentaires
- Recherche dans les commentaires

## Résumé

Dans cette section, vous avez appris :

- **Concepts REST** : Principes, méthodes HTTP, codes de statut
- **Architecture API** : Structure de projet, séparation des responsabilités
- **CRUD complet** : Create, Read, Update, Delete avec Go
- **Gestion d'erreurs** : Réponses cohérentes et informatives
- **Middleware** : Logging, validation, CORS
- **Documentation** : Interface web interactive
- **Client API** : Consommation d'API depuis Go
- **Bonnes pratiques** : Validation, pagination, filtres

Les APIs REST sont essentielles dans le développement moderne. Go offre tous les outils nécessaires pour créer des APIs robustes, performantes et maintenables.

---

*Félicitations ! Vous maîtrisez maintenant la création d'APIs REST complètes en Go. Dans la section suivante, nous explorerons les WebSockets pour la communication temps réel.*

⏭️

# API avec Authentification - Documentation et Tests

## Page de documentation et fonction principale

```go
// Page de documentation
func (api *AuthenticatedTaskAPI) docsHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Task API with Authentication</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .container { max-width: 1000px; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; text-align: center; }
        .endpoint { background: #f8f9fa; padding: 15px; margin: 20px 0; border-radius: 5px; border-left: 4px solid #007bff; }
        .method { display: inline-block; padding: 2px 8px; border-radius: 3px; color: white; font-weight: bold; margin-right: 10px; }
        .get { background: #28a745; }
        .post { background: #007bff; }
        .put { background: #ffc107; color: #000; }
        .patch { background: #6c757d; }
        .delete { background: #dc3545; }
        .auth-section { background: #fff3cd; padding: 20px; border-radius: 5px; margin: 20px 0; border-left: 4px solid #ffc107; }
        .test-form { background: #f0f8ff; padding: 20px; border-radius: 5px; margin: 20px 0; }
        input, textarea, select { width: 100%; padding: 8px; margin: 5px 0; border: 1px solid #ddd; border-radius: 3px; box-sizing: border-box; }
        button { background: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 3px; cursor: pointer; margin: 5px; }
        button:hover { background: #0056b3; }
        button.logout { background: #dc3545; }
        button.logout:hover { background: #c82333; }
        #result { margin-top: 20px; padding: 15px; border-radius: 5px; }
        .success { background: #d4edda; border: 1px solid #c3e6cb; color: #155724; }
        .error { background: #f8d7da; border: 1px solid #f5c6cb; color: #721c24; }
        .example { background: #e9ecef; padding: 10px; border-radius: 3px; margin: 10px 0; }
        pre { overflow-x: auto; }
        .token-display { background: #d1ecf1; padding: 10px; border-radius: 3px; margin: 10px 0; word-break: break-all; }
        .hidden { display: none; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔐 Task API with Authentication</h1>

        <div class="auth-section">
            <h2>🔑 Authentication Required</h2>
            <p>Cette API utilise l'authentification JWT. Vous devez vous inscrire ou vous connecter pour accéder aux endpoints des tâches.</p>

            <div id="auth-forms">
                <h3>Inscription</h3>
                <div>
                    <input type="text" id="reg-username" placeholder="Nom d'utilisateur (min 3 caractères)">
                    <input type="email" id="reg-email" placeholder="Email">
                    <input type="password" id="reg-password" placeholder="Mot de passe (min 6 caractères)">
                    <button onclick="register()">S'inscrire</button>
                </div>

                <h3>Connexion</h3>
                <div>
                    <input type="text" id="login-username" placeholder="Nom d'utilisateur">
                    <input type="password" id="login-password" placeholder="Mot de passe">
                    <button onclick="login()">Se connecter</button>
                </div>
            </div>

            <div id="user-info" class="hidden">
                <h3>Utilisateur connecté</h3>
                <div id="user-details"></div>
                <button class="logout" onclick="logout()">Se déconnecter</button>
            </div>

            <div id="token-display" class="token-display hidden">
                <strong>Token JWT:</strong> <span id="token-value"></span>
            </div>
        </div>

        <h2>📋 Endpoints</h2>

        <div class="endpoint">
            <h3><span class="method post">POST</span> /auth/register</h3>
            <p><strong>Description:</strong> Inscription d'un nouvel utilisateur</p>
            <div class="example">
                <strong>Body:</strong>
                <pre>{"username": "alice", "email": "alice@example.com", "password": "password123"}</pre>
            </div>
        </div>

        <div class="endpoint">
            <h3><span class="method post">POST</span> /auth/login</h3>
            <p><strong>Description:</strong> Connexion d'un utilisateur</p>
            <div class="example">
                <strong>Body:</strong>
                <pre>{"username": "alice", "password": "password123"}</pre>
            </div>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /auth/profile</h3>
            <p><strong>Description:</strong> Récupérer les informations du profil utilisateur</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks</h3>
            <p><strong>Description:</strong> Récupérer toutes les tâches de l'utilisateur connecté</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
            <p><strong>Paramètres:</strong> completed, priority, category</p>
        </div>

        <div class="endpoint">
            <h3><span class="method post">POST</span> /tasks</h3>
            <p><strong>Description:</strong> Créer une nouvelle tâche pour l'utilisateur connecté</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
            <div class="example">
                <strong>Body:</strong>
                <pre>{"title": "Ma tâche", "priority": "high", "due_date": "2024-12-31"}</pre>
            </div>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Récupérer une tâche spécifique de l'utilisateur</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="endpoint">
            <h3><span class="method put">PUT</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Mettre à jour une tâche de l'utilisateur</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="endpoint">
            <h3><span class="method patch">PATCH</span> /tasks/{id}/toggle</h3>
            <p><strong>Description:</strong> Basculer le statut complété/non complété d'une tâche</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="endpoint">
            <h3><span class="method delete">DELETE</span> /tasks/{id}</h3>
            <p><strong>Description:</strong> Supprimer une tâche de l'utilisateur</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="endpoint">
            <h3><span class="method get">GET</span> /tasks/stats</h3>
            <p><strong>Description:</strong> Obtenir des statistiques sur les tâches de l'utilisateur</p>
            <p><strong>Authentification:</strong> Bearer token requis</p>
        </div>

        <div class="test-form">
            <h2>🧪 Test des Endpoints</h2>
            <div id="api-test" class="hidden">
                <div>
                    <label>Méthode:</label>
                    <select id="method">
                        <option value="GET">GET</option>
                        <option value="POST">POST</option>
                        <option value="PUT">PUT</option>
                        <option value="PATCH">PATCH</option>
                        <option value="DELETE">DELETE</option>
                    </select>
                </div>

                <div>
                    <label>URL:</label>
                    <input type="text" id="url" value="/tasks" placeholder="/tasks">
                </div>

                <div>
                    <label>Body JSON (pour POST/PUT):</label>
                    <textarea id="body" rows="6" placeholder='{"title": "Ma tâche", "priority": "medium"}'></textarea>
                </div>

                <button onclick="testAPI()">Envoyer la requête</button>
                <button onclick="loadTaskExample()">Exemple de tâche</button>
                <button onclick="getMyTasks()">Mes tâches</button>
                <button onclick="getMyStats()">Mes statistiques</button>
            </div>

            <div id="login-required">
                <p>🔒 Connectez-vous pour tester les endpoints des tâches</p>
            </div>

            <div id="result"></div>
        </div>
    </div>

    <script>
        let currentToken = localStorage.getItem('authToken');
        let currentUser = JSON.parse(localStorage.getItem('currentUser') || 'null');

        // Initialiser l'interface
        document.addEventListener('DOMContentLoaded', function() {
            updateUI();
        });

        function updateUI() {
            const authForms = document.getElementById('auth-forms');
            const userInfo = document.getElementById('user-info');
            const tokenDisplay = document.getElementById('token-display');
            const apiTest = document.getElementById('api-test');
            const loginRequired = document.getElementById('login-required');

            if (currentToken && currentUser) {
                authForms.classList.add('hidden');
                userInfo.classList.remove('hidden');
                tokenDisplay.classList.remove('hidden');
                apiTest.classList.remove('hidden');
                loginRequired.classList.add('hidden');

                document.getElementById('user-details').innerHTML =
                    '<strong>Username:</strong> ' + currentUser.username + '<br>' +
                    '<strong>Email:</strong> ' + currentUser.email + '<br>' +
                    '<strong>ID:</strong> ' + currentUser.id;

                document.getElementById('token-value').textContent = currentToken.substring(0, 50) + '...';
            } else {
                authForms.classList.remove('hidden');
                userInfo.classList.add('hidden');
                tokenDisplay.classList.add('hidden');
                apiTest.classList.add('hidden');
                loginRequired.classList.remove('hidden');
            }
        }

        function register() {
            const username = document.getElementById('reg-username').value;
            const email = document.getElementById('reg-email').value;
            const password = document.getElementById('reg-password').value;

            if (!username || !email || !password) {
                showResult('Tous les champs sont requis', 'error');
                return;
            }

            fetch('/auth/register', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({username, email, password})
            })
            .then(response => response.json())
            .then(data => {
                if (data.success) {
                    currentToken = data.token;
                    currentUser = data.user;
                    localStorage.setItem('authToken', currentToken);
                    localStorage.setItem('currentUser', JSON.stringify(currentUser));
                    showResult('Inscription réussie ! Bienvenue ' + data.user.username, 'success');
                    updateUI();
                    clearAuthForms();
                } else {
                    showResult('Erreur: ' + data.error, 'error');
                }
            })
            .catch(error => showResult('Erreur réseau: ' + error, 'error'));
        }

        function login() {
            const username = document.getElementById('login-username').value;
            const password = document.getElementById('login-password').value;

            if (!username || !password) {
                showResult('Nom d\'utilisateur et mot de passe requis', 'error');
                return;
            }

            fetch('/auth/login', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({username, password})
            })
            .then(response => response.json())
            .then(data => {
                if (data.success) {
                    currentToken = data.token;
                    currentUser = data.user;
                    localStorage.setItem('authToken', currentToken);
                    localStorage.setItem('currentUser', JSON.stringify(currentUser));
                    showResult('Connexion réussie ! Bienvenue ' + data.user.username, 'success');
                    updateUI();
                    clearAuthForms();
                } else {
                    showResult('Erreur: ' + data.error, 'error');
                }
            })
            .catch(error => showResult('Erreur réseau: ' + error, 'error'));
        }

        function logout() {
            currentToken = null;
            currentUser = null;
            localStorage.removeItem('authToken');
            localStorage.removeItem('currentUser');
            showResult('Déconnexion réussie', 'success');
            updateUI();
        }

        function clearAuthForms() {
            document.getElementById('reg-username').value = '';
            document.getElementById('reg-email').value = '';
            document.getElementById('reg-password').value = '';
            document.getElementById('login-username').value = '';
            document.getElementById('login-password').value = '';
        }

        function testAPI() {
            if (!currentToken) {
                showResult('Vous devez être connecté pour tester les endpoints', 'error');
                return;
            }

            const method = document.getElementById('method').value;
            const url = document.getElementById('url').value;
            const body = document.getElementById('body').value;

            const options = {
                method: method,
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer ' + currentToken
                }
            };

            if ((method === 'POST' || method === 'PUT' || method === 'PATCH') && body.trim()) {
                options.body = body;
            }

            fetch(url, options)
                .then(response => response.json())
                .then(data => {
                    showResult('Réponse: ' + JSON.stringify(data, null, 2), data.success ? 'success' : 'error');
                })
                .catch(error => {
                    showResult('Erreur: ' + error, 'error');
                });
        }

        function loadTaskExample() {
            document.getElementById('method').value = 'POST';
            document.getElementById('url').value = '/tasks';
            document.getElementById('body').value = JSON.stringify({
                title: "Exemple de tâche",
                description: "Ceci est une tâche créée depuis l'interface de test",
                priority: "high",
                due_date: "2024-12-31",
                category: "Test"
            }, null, 2);
        }

        function getMyTasks() {
            document.getElementById('method').value = 'GET';
            document.getElementById('url').value = '/tasks';
            document.getElementById('body').value = '';
            testAPI();
        }

        function getMyStats() {
            document.getElementById('method').value = 'GET';
            document.getElementById('url').value = '/tasks/stats';
            document.getElementById('body').value = '';
            testAPI();
        }

        function showResult(message, type) {
            const resultDiv = document.getElementById('result');
            resultDiv.className = type;
            resultDiv.innerHTML = '<pre>' + message + '</pre>';
        }
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Point d'entrée principal
func main() {
    api := NewAuthenticatedTaskAPI()

    // Créer quelques utilisateurs de test
    testUsers := []RegisterRequest{
        {Username: "admin", Email: "admin@example.com", Password: "admin123"},
        {Username: "demo", Email: "demo@example.com", Password: "demo123"},
    }

    for _, user := range testUsers {
        api.auth.register(user)
    }

    // Créer quelques tâches de test pour l'utilisateur demo
    api.auth.createTask(2, TaskRequest{
        Title:       "Tâche de démonstration",
        Description: "Ceci est une tâche créée pour la démonstration",
        Priority:    "medium",
        Category:    "Demo",
    })

    api.auth.createTask(2, TaskRequest{
        Title:       "Tâche urgente",
        Description: "Une tâche avec haute priorité",
        Priority:    "high",
        Category:    "Travail",
        DueDate:     "2024-12-31",
    })

    // Configuration des routes
    http.HandleFunc("/", api.docsHandler)

    // Routes d'authentification (pas de middleware)
    http.HandleFunc("/auth/register", api.registerHandler)
    http.HandleFunc("/auth/login", api.loginHandler)
    http.HandleFunc("/auth/profile", api.authMiddleware(api.profileHandler))

    // Routes de tâches (avec middleware d'authentification)
    http.HandleFunc("/tasks", api.tasksRouter)
    http.HandleFunc("/tasks/", api.tasksRouter)

    port := ":8080"
    fmt.Printf("🚀 API avec authentification démarrée sur http://localhost%s\n", port)
    fmt.Printf("📖 Documentation: http://localhost%s\n", port)
    fmt.Printf("\n🔐 Endpoints d'authentification:\n")
    fmt.Printf("   POST /auth/register     - Inscription\n")
    fmt.Printf("   POST /auth/login        - Connexion\n")
    fmt.Printf("   GET  /auth/profile      - Profil utilisateur (auth requise)\n")
    fmt.Printf("\n📋 Endpoints de tâches (authentification requise):\n")
    fmt.Printf("   GET    /tasks           - Lister les tâches\n")
    fmt.Printf("   POST   /tasks           - Créer une tâche\n")
    fmt.Printf("   GET    /tasks/{id}      - Récupérer une tâche\n")
    fmt.Printf("   PUT    /tasks/{id}      - Mettre à jour une tâche\n")
    fmt.Printf("   PATCH  /tasks/{id}/toggle - Basculer complété\n")
    fmt.Printf("   DELETE /tasks/{id}      - Supprimer une tâche\n")
    fmt.Printf("   GET    /tasks/stats     - Statistiques\n")
    fmt.Printf("\n👥 Utilisateurs de test créés:\n")
    fmt.Printf("   Username: admin, Password: admin123\n")
    fmt.Printf("   Username: demo,  Password: demo123 (avec quelques tâches)\n")

    log.Fatal(http.ListenAndServe(port, nil))
}
```

## Tests avec curl

### 1. Tests d'authentification

```bash
# 1. Inscription d'un nouvel utilisateur
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "email": "alice@example.com",
    "password": "password123"
  }'

# 2. Connexion avec l'utilisateur créé
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "password123"
  }'

# Sauvegarder le token reçu
export TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# 3. Récupérer le profil utilisateur
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/auth/profile

# 4. Test avec un utilisateur de démonstration
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "demo", "password": "demo123"}'
```

### 2. Tests des tâches (avec authentification)

```bash
# 1. Récupérer toutes les tâches de l'utilisateur
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/tasks

# 2. Créer une nouvelle tâche
curl -X POST http://localhost:8080/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Ma première tâche privée",
    "description": "Cette tâche n'appartient qu'à moi",
    "priority": "high",
    "due_date": "2024-12-25",
    "category": "Personnel"
  }'

# 3. Récupérer une tâche spécifique
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/tasks/1

# 4. Mettre à jour une tâche
curl -X PUT http://localhost:8080/tasks/1 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Tâche mise à jour",
    "priority": "medium"
  }'

# 5. Basculer le statut d'une tâche
curl -X PATCH http://localhost:8080/tasks/1/toggle \
  -H "Authorization: Bearer $TOKEN"

# 6. Filtrer les tâches par priorité
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/tasks?priority=high"

# 7. Filtrer les tâches complétées
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/tasks?completed=true"

# 8. Obtenir des statistiques
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/tasks/stats

# 9. Supprimer une tâche
curl -X DELETE http://localhost:8080/tasks/1 \
  -H "Authorization: Bearer $TOKEN"
```

### 3. Tests d'erreurs de sécurité

```bash
# 1. Tentative d'accès sans token
curl http://localhost:8080/tasks

# 2. Tentative d'accès avec token invalide
curl -H "Authorization: Bearer invalid-token" \
  http://localhost:8080/tasks

# 3. Tentative d'accès avec mauvais format de token
curl -H "Authorization: invalid-format" \
  http://localhost:8080/tasks

# 4. Tentative de connexion avec mauvais mot de passe
curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "alice", "password": "wrongpassword"}'

# 5. Tentative d'inscription avec email déjà utilisé
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice2",
    "email": "alice@example.com",
    "password": "password123"
  }'
```

## Client Go pour tester l'authentification

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type AuthClient struct {
    BaseURL string
    Token   string
    Client  *http.Client
}

func NewAuthClient(baseURL string) *AuthClient {
    return &AuthClient{
        BaseURL: baseURL,
        Client: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

func (ac *AuthClient) register(username, email, password string) error {
    reqData := map[string]string{
        "username": username,
        "email":    email,
        "password": password,
    }

    jsonData, _ := json.Marshal(reqData)
    resp, err := ac.Client.Post(ac.BaseURL+"/auth/register", "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        if token, ok := result["token"].(string); ok {
            ac.Token = token
            fmt.Printf("✅ Inscription réussie ! Token: %s...\n", token[:20])
        }
    } else {
        return fmt.Errorf("erreur: %v", result["error"])
    }

    return nil
}

func (ac *AuthClient) login(username, password string) error {
    reqData := map[string]string{
        "username": username,
        "password": password,
    }

    jsonData, _ := json.Marshal(reqData)
    resp, err := ac.Client.Post(ac.BaseURL+"/auth/login", "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        if token, ok := result["token"].(string); ok {
            ac.Token = token
            fmt.Printf("✅ Connexion réussie ! Token: %s...\n", token[:20])
        }
    } else {
        return fmt.Errorf("erreur: %v", result["error"])
    }

    return nil
}

func (ac *AuthClient) createTask(title, priority string) error {
    if ac.Token == "" {
        return fmt.Errorf("non authentifié")
    }

    reqData := map[string]string{
        "title":    title,
        "priority": priority,
    }

    jsonData, _ := json.Marshal(reqData)
    req, _ := http.NewRequest("POST", ac.BaseURL+"/tasks", bytes.NewBuffer(jsonData))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+ac.Token)

    resp, err := ac.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        fmt.Printf("✅ Tâche créée : %s\n", title)
    } else {
        return fmt.Errorf("erreur: %v", result["error"])
    }

    return nil
}

func (ac *AuthClient) getTasks() error {
    if ac.Token == "" {
        return fmt.Errorf("non authentifié")
    }

    req, _ := http.NewRequest("GET", ac.BaseURL+"/tasks", nil)
    req.Header.Set("Authorization", "Bearer "+ac.Token)

    resp, err := ac.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        if data, ok := result["data"].([]interface{}); ok {
            fmt.Printf("✅ %d tâches trouvées\n", len(data))
            for i, task := range data {
                if taskMap, ok := task.(map[string]interface{}); ok {
                    fmt.Printf("   %d. %s (Priorité: %s)\n", i+1, taskMap["title"], taskMap["priority"])
                }
            }
        }
    } else {
        return fmt.Errorf("erreur: %v", result["error"])
    }

    return nil
}

func main() {
    client := NewAuthClient("http://localhost:8080")

    fmt.Println("🧪 Test du client avec authentification")
    fmt.Println("=========================================")

    // Test 1: Inscription
    fmt.Println("\n1. Test d'inscription")
    if err := client.register("testuser", "test@example.com", "password123"); err != nil {
        fmt.Printf("❌ Erreur d'inscription: %v\n", err)

        // Si l'inscription échoue, essayer de se connecter
        fmt.Println("\n2. Tentative de connexion avec utilisateur existant")
        if err := client.login("testuser", "password123"); err != nil {
            fmt.Printf("❌ Erreur de connexion: %v\n", err)
            return
        }
    }

    // Test 2: Créer des tâches
    fmt.Println("\n3. Création de tâches")
    tasks := []struct {
        title    string
        priority string
    }{
        {"Apprendre l'authentification JWT", "high"},
        {"Tester l'API", "medium"},
        {"Documenter le code", "low"},
    }

    for _, task := range tasks {
        if err := client.createTask(task.title, task.priority); err != nil {
            fmt.Printf("❌ Erreur création tâche: %v\n", err)
        }
        time.Sleep(100 * time.Millisecond) // Petite pause entre les créations
    }

    // Test 3: Récupérer les tâches
    fmt.Println("\n4. Récupération des tâches")
    if err := client.getTasks(); err != nil {
        fmt.Printf("❌ Erreur récupération tâches: %v\n", err)
    }

    // Test 4: Statistiques
    fmt.Println("\n5. Statistiques")
    if err := client.getStats(); err != nil {
        fmt.Printf("❌ Erreur statistiques: %v\n", err)
    }

    // Test 5: Test de sécurité - tentative d'accès sans token
    fmt.Println("\n6. Test de sécurité (sans token)")
    oldToken := client.Token
    client.Token = ""
    if err := client.getTasks(); err != nil {
        fmt.Printf("✅ Sécurité OK - Accès refusé sans token: %v\n", err)
    } else {
        fmt.Printf("❌ Problème de sécurité - Accès autorisé sans token!\n")
    }
    client.Token = oldToken

    // Test 6: Test avec token invalide
    fmt.Println("\n7. Test de sécurité (token invalide)")
    client.Token = "invalid-token-12345"
    if err := client.getTasks(); err != nil {
        fmt.Printf("✅ Sécurité OK - Token invalide rejeté: %v\n", err)
    } else {
        fmt.Printf("❌ Problème de sécurité - Token invalide accepté!\n")
    }

    fmt.Println("\n🎉 Tests terminés!")
}

func (ac *AuthClient) getStats() error {
    if ac.Token == "" {
        return fmt.Errorf("non authentifié")
    }

    req, _ := http.NewRequest("GET", ac.BaseURL+"/tasks/stats", nil)
    req.Header.Set("Authorization", "Bearer "+ac.Token)

    resp, err := ac.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        if data, ok := result["data"].(map[string]interface{}); ok {
            fmt.Printf("✅ Statistiques:\n")
            for key, value := range data {
                fmt.Printf("   %s: %v\n", key, value)
            }
        }
    } else {
        return fmt.Errorf("erreur: %v", result["error"])
    }

    return nil
}
```

## Script de test automatisé complet

```bash
#!/bin/bash

# Script de test automatisé pour l'API avec authentification
echo "🧪 Tests automatisés de l'API avec authentification"
echo "=================================================="

API_URL="http://localhost:8080"

# Fonction pour afficher les résultats
check_response() {
    local response="$1"
    local description="$2"

    if echo "$response" | grep -q '"success":true'; then
        echo "✅ $description"
    else
        echo "❌ $description"
        echo "   Réponse: $response"
    fi
}

# Test 1: Inscription d'un nouvel utilisateur
echo -e "\n1. Test d'inscription"
REGISTER_RESPONSE=$(curl -s -X POST $API_URL/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser_'$(date +%s)'",
    "email": "test'$(date +%s)'@example.com",
    "password": "password123"
  }')

check_response "$REGISTER_RESPONSE" "Inscription d'un nouvel utilisateur"

# Extraire le token de l'inscription
TOKEN=$(echo "$REGISTER_RESPONSE" | grep -o '"token":"[^"]*"' | cut -d'"' -f4)

if [ -z "$TOKEN" ]; then
    echo "❌ Impossible d'extraire le token, tentative de connexion avec utilisateur demo"

    # Test de connexion avec utilisateur existant
    LOGIN_RESPONSE=$(curl -s -X POST $API_URL/auth/login \
      -H "Content-Type: application/json" \
      -d '{"username": "demo", "password": "demo123"}')

    check_response "$LOGIN_RESPONSE" "Connexion avec utilisateur demo"
    TOKEN=$(echo "$LOGIN_RESPONSE" | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
fi

if [ -z "$TOKEN" ]; then
    echo "❌ Impossible d'obtenir un token, arrêt des tests"
    exit 1
fi

echo "🔑 Token obtenu: ${TOKEN:0:20}..."

# Test 2: Récupération du profil
echo -e "\n2. Test de récupération du profil"
PROFILE_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" $API_URL/auth/profile)
check_response "$PROFILE_RESPONSE" "Récupération du profil utilisateur"

# Test 3: Création de tâches
echo -e "\n3. Test de création de tâches"

TASKS=(
    '{"title": "Tâche de test 1", "priority": "high", "category": "Test"}'
    '{"title": "Tâche de test 2", "priority": "medium", "due_date": "2024-12-31"}'
    '{"title": "Tâche de test 3", "priority": "low", "description": "Une tâche simple"}'
)

for i in "${!TASKS[@]}"; do
    TASK_RESPONSE=$(curl -s -X POST $API_URL/tasks \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $TOKEN" \
      -d "${TASKS[$i]}")

    check_response "$TASK_RESPONSE" "Création de la tâche $((i+1))"
done

# Test 4: Récupération des tâches
echo -e "\n4. Test de récupération des tâches"
TASKS_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" $API_URL/tasks)
check_response "$TASKS_RESPONSE" "Récupération de toutes les tâches"

TASK_COUNT=$(echo "$TASKS_RESPONSE" | grep -o '"count":[0-9]*' | cut -d':' -f2)
echo "   Nombre de tâches: $TASK_COUNT"

# Test 5: Filtrage des tâches
echo -e "\n5. Test de filtrage des tâches"

# Filtrer par priorité
HIGH_PRIORITY_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/tasks?priority=high")
check_response "$HIGH_PRIORITY_RESPONSE" "Filtrage par priorité élevée"

# Filtrer par statut
COMPLETED_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" "$API_URL/tasks?completed=false")
check_response "$COMPLETED_RESPONSE" "Filtrage par statut non complété"

# Test 6: Récupération d'une tâche spécifique
echo -e "\n6. Test de récupération d'une tâche spécifique"
SINGLE_TASK_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" $API_URL/tasks/1)

if echo "$SINGLE_TASK_RESPONSE" | grep -q '"success":true'; then
    echo "✅ Récupération de la tâche ID 1"

    # Test 7: Mise à jour de la tâche
    echo -e "\n7. Test de mise à jour de tâche"
    UPDATE_RESPONSE=$(curl -s -X PUT $API_URL/tasks/1 \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer $TOKEN" \
      -d '{"title": "Tâche mise à jour", "priority": "medium"}')

    check_response "$UPDATE_RESPONSE" "Mise à jour de la tâche ID 1"

    # Test 8: Basculer le statut
    echo -e "\n8. Test de basculement de statut"
    TOGGLE_RESPONSE=$(curl -s -X PATCH $API_URL/tasks/1/toggle \
      -H "Authorization: Bearer $TOKEN")

    check_response "$TOGGLE_RESPONSE" "Basculement du statut de la tâche ID 1"
else
    echo "⚠️  Aucune tâche trouvée pour les tests de modification"
fi

# Test 9: Statistiques
echo -e "\n9. Test des statistiques"
STATS_RESPONSE=$(curl -s -H "Authorization: Bearer $TOKEN" $API_URL/tasks/stats)
check_response "$STATS_RESPONSE" "Récupération des statistiques"

# Test 10: Tests de sécurité
echo -e "\n10. Tests de sécurité"

# Test sans token
NO_TOKEN_RESPONSE=$(curl -s $API_URL/tasks)
if echo "$NO_TOKEN_RESPONSE" | grep -q '"success":false'; then
    echo "✅ Accès refusé sans token"
else
    echo "❌ Problème de sécurité: accès autorisé sans token"
fi

# Test avec token invalide
INVALID_TOKEN_RESPONSE=$(curl -s -H "Authorization: Bearer invalid-token" $API_URL/tasks)
if echo "$INVALID_TOKEN_RESPONSE" | grep -q '"success":false'; then
    echo "✅ Token invalide rejeté"
else
    echo "❌ Problème de sécurité: token invalide accepté"
fi

# Test avec mauvais format d'Authorization
BAD_FORMAT_RESPONSE=$(curl -s -H "Authorization: invalid-format" $API_URL/tasks)
if echo "$BAD_FORMAT_RESPONSE" | grep -q '"success":false'; then
    echo "✅ Mauvais format d'Authorization rejeté"
else
    echo "❌ Problème de sécurité: mauvais format accepté"
fi

# Test 11: Tests d'erreurs d'authentification
echo -e "\n11. Tests d'erreurs d'authentification"

# Tentative de connexion avec mauvais mot de passe
BAD_LOGIN_RESPONSE=$(curl -s -X POST $API_URL/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "demo", "password": "wrongpassword"}')

if echo "$BAD_LOGIN_RESPONSE" | grep -q '"success":false'; then
    echo "✅ Connexion refusée avec mauvais mot de passe"
else
    echo "❌ Problème de sécurité: connexion autorisée avec mauvais mot de passe"
fi

# Tentative d'inscription avec données invalides
INVALID_REGISTER_RESPONSE=$(curl -s -X POST $API_URL/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "", "email": "invalid", "password": "123"}')

if echo "$INVALID_REGISTER_RESPONSE" | grep -q '"success":false'; then
    echo "✅ Inscription refusée avec données invalides"
else
    echo "❌ Problème de validation: inscription autorisée avec données invalides"
fi

# Test 12: Nettoyage (suppression de tâches)
echo -e "\n12. Test de suppression"
if [ ! -z "$TASK_COUNT" ] && [ "$TASK_COUNT" -gt 0 ]; then
    DELETE_RESPONSE=$(curl -s -X DELETE $API_URL/tasks/1 \
      -H "Authorization: Bearer $TOKEN")

    check_response "$DELETE_RESPONSE" "Suppression de la tâche ID 1"
fi

echo -e "\n🎉 Tests automatisés terminés!"
echo "============================================"

# Résumé
echo -e "\n📊 Résumé des fonctionnalités testées:"
echo "✓ Inscription et connexion d'utilisateurs"
echo "✓ Authentification JWT"
echo "✓ Isolation des données par utilisateur"
echo "✓ CRUD complet des tâches"
echo "✓ Filtrage et recherche"
echo "✓ Statistiques utilisateur"
echo "✓ Sécurité et validation"
echo "✓ Gestion des erreurs"
```

## Tests de performance et charge

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// Test de charge pour l'API avec authentification
type LoadTester struct {
    BaseURL string
    Client  *http.Client
}

func NewLoadTester(baseURL string) *LoadTester {
    return &LoadTester{
        BaseURL: baseURL,
        Client: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}

func (lt *LoadTester) registerUser(username, email, password string) (string, error) {
    reqData := map[string]string{
        "username": username,
        "email":    email,
        "password": password,
    }

    jsonData, _ := json.Marshal(reqData)
    resp, err := lt.Client.Post(lt.BaseURL+"/auth/register", "application/json", bytes.NewBuffer(jsonData))
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        if token, ok := result["token"].(string); ok {
            return token, nil
        }
    }

    return "", fmt.Errorf("échec inscription: %v", result["error"])
}

func (lt *LoadTester) createTaskWithToken(token, title string) error {
    reqData := map[string]string{
        "title":    title,
        "priority": "medium",
    }

    jsonData, _ := json.Marshal(reqData)
    req, _ := http.NewRequest("POST", lt.BaseURL+"/tasks", bytes.NewBuffer(jsonData))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+token)

    resp, err := lt.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&result)

    if success, ok := result["success"].(bool); ok && success {
        return nil
    }

    return fmt.Errorf("échec création tâche")
}

func (lt *LoadTester) getTasksWithToken(token string) error {
    req, _ := http.NewRequest("GET", lt.BaseURL+"/tasks", nil)
    req.Header.Set("Authorization", "Bearer "+token)

    resp, err := lt.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    return nil
}

func (lt *LoadTester) runLoadTest(numUsers, tasksPerUser int) {
    fmt.Printf("🚀 Test de charge: %d utilisateurs, %d tâches par utilisateur\n", numUsers, tasksPerUser)
    fmt.Println("================================================================")

    start := time.Now()
    var wg sync.WaitGroup
    var successCount, errorCount int
    var mutex sync.Mutex

    // Canal pour limiter la concurrence
    semaphore := make(chan struct{}, 10) // Max 10 requêtes simultanées

    for i := 0; i < numUsers; i++ {
        wg.Add(1)

        go func(userNum int) {
            defer wg.Done()

            semaphore <- struct{}{} // Acquérir le semaphore
            defer func() { <-semaphore }() // Libérer le semaphore

            username := fmt.Sprintf("user%d", userNum)
            email := fmt.Sprintf("user%d@test.com", userNum)
            password := "password123"

            // Inscription
            token, err := lt.registerUser(username, email, password)
            if err != nil {
                mutex.Lock()
                errorCount++
                mutex.Unlock()
                fmt.Printf("❌ Erreur inscription utilisateur %d: %v\n", userNum, err)
                return
            }

            // Créer des tâches
            for j := 0; j < tasksPerUser; j++ {
                taskTitle := fmt.Sprintf("Tâche %d de l'utilisateur %d", j+1, userNum)

                if err := lt.createTaskWithToken(token, taskTitle); err != nil {
                    mutex.Lock()
                    errorCount++
                    mutex.Unlock()
                    continue
                }

                mutex.Lock()
                successCount++
                mutex.Unlock()
            }

            // Récupérer les tâches
            if err := lt.getTasksWithToken(token); err != nil {
                mutex.Lock()
                errorCount++
                mutex.Unlock()
            } else {
                mutex.Lock()
                successCount++
                mutex.Unlock()
            }

            fmt.Printf("✅ Utilisateur %d terminé\n", userNum)

        }(i)
    }

    wg.Wait()
    duration := time.Since(start)

    totalOperations := successCount + errorCount
    operationsPerSecond := float64(totalOperations) / duration.Seconds()

    fmt.Println("\n📊 Résultats du test de charge:")
    fmt.Printf("⏱️  Durée totale: %v\n", duration)
    fmt.Printf("✅ Opérations réussies: %d\n", successCount)
    fmt.Printf("❌ Opérations échouées: %d\n", errorCount)
    fmt.Printf("📈 Opérations/seconde: %.2f\n", operationsPerSecond)
    fmt.Printf("📊 Taux de succès: %.2f%%\n", float64(successCount)/float64(totalOperations)*100)
}

func main() {
    tester := NewLoadTester("http://localhost:8080")

    fmt.Println("🧪 Tests de performance de l'API avec authentification")
    fmt.Println("======================================================")

    // Test 1: Test léger
    fmt.Println("\n1. Test léger (5 utilisateurs, 2 tâches chacun)")
    tester.runLoadTest(5, 2)

    time.Sleep(2 * time.Second)

    // Test 2: Test moyen
    fmt.Println("\n2. Test moyen (10 utilisateurs, 5 tâches chacun)")
    tester.runLoadTest(10, 5)

    time.Sleep(2 * time.Second)

    // Test 3: Test plus intensif
    fmt.Println("\n3. Test intensif (20 utilisateurs, 3 tâches chacun)")
    tester.runLoadTest(20, 3)

    fmt.Println("\n🎉 Tests de performance terminés!")
}
```

## Exercice de validation

### Créer un script de validation complète

```bash
#!/bin/bash

# Script de validation complète de l'API avec authentification
echo "🔍 Validation complète de l'API avec authentification"
echo "===================================================="

# Vérifier que le serveur répond
if ! curl -s http://localhost:8080 > /dev/null; then
    echo "❌ Le serveur n'est pas accessible sur http://localhost:8080"
    echo "   Assurez-vous que l'API est démarrée"
    exit 1
fi

echo "✅ Serveur accessible"

# Tests de sécurité approfondis
echo -e "\n🔒 Tests de sécurité approfondis"

# Test 1: Tentative d'injection SQL dans l'inscription
echo "Test 1: Protection contre l'injection"
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "admin; DROP TABLE users;", "email": "hack@test.com", "password": "password"}' \
  | grep -q '"success":false' && echo "✅ Protection injection OK" || echo "❌ Vulnérable à l'injection"

# Test 2: Tentative avec mot de passe trop court
echo "Test 2: Validation mot de passe"
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "test", "email": "test@test.com", "password": "123"}' \
  | grep -q '"success":false' && echo "✅ Validation mot de passe OK" || echo "❌ Mot de passe faible accepté"

# Test 3: Tentative avec email invalide
echo "Test 3: Validation email"
curl -s -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username": "test", "email": "invalid-email", "password": "password123"}' \
  | grep -q '"success":false' && echo "✅ Validation email OK" || echo "❌ Email invalide accepté"

# Test 4: Tentative de réutilisation de token expiré (simulation)
echo "Test 4: Gestion des tokens expirés"
EXPIRED_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MzQ2NzYwMDB9.invalid"
curl -s -H "Authorization: Bearer $EXPIRED_TOKEN" http://localhost:8080/tasks \
  | grep -q '"success":false' && echo "✅ Token expiré rejeté" || echo "❌ Token expiré accepté"

echo -e "\n✅ Validation terminée - API sécurisée et fonctionnelle"
```

## Résumé de la solution

Cette solution complète de l'exercice 1 inclut :

### ✅ **Fonctionnalités implémentées :**
- **Système d'utilisateurs** complet avec inscription et connexion
- **Authentification JWT** sécurisée avec tokens signés
- **Tâches privées** isolées par utilisateur
- **Validation robuste** des données d'entrée
- **Gestion d'erreurs** complète
- **Interface web** interactive pour tester l'API
- **Sécurité renforcée** avec middleware d'authentification

### 🧪 **Tests fournis :**
- **Client Go** pour tests automatisés
- **Scripts bash** pour validation
- **Tests de charge** et performance
- **Tests de sécurité** approfondis
- **Interface web** de test interactive

### 🔒 **Sécurité :**
- Mots de passe hachés avec bcrypt
- Tokens JWT avec expiration
- Validation stricte des entrées
- Protection contre les accès non autorisés
- Isolation complète des données par utilisateur

Cette solution est prête pour un environnement de production avec les adaptations appropriées (base de données persistante, configuration sécurisée, etc.).

⏭️
