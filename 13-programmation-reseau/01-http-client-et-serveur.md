🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13-1 : HTTP Client et Serveur en Go

## Introduction

HTTP (HyperText Transfer Protocol) est le protocole de communication le plus utilisé sur Internet. Il permet aux clients (comme les navigateurs web) de communiquer avec les serveurs pour échanger des données. Go offre des outils puissants et simples pour créer des clients et serveurs HTTP.

## Qu'est-ce que HTTP ?

### Concept de base
HTTP fonctionne selon le modèle **requête-réponse** :
1. Le **client** envoie une requête HTTP au serveur
2. Le **serveur** traite la requête et renvoie une réponse HTTP
3. La connexion se ferme (HTTP est "stateless")

### Méthodes HTTP principales
- **GET** : Récupérer des données
- **POST** : Envoyer des données (création)
- **PUT** : Mettre à jour des données
- **DELETE** : Supprimer des données
- **HEAD** : Comme GET mais sans le corps de la réponse

### Codes de statut HTTP
- **2xx** : Succès (200 OK, 201 Created)
- **3xx** : Redirection (301 Moved, 302 Found)
- **4xx** : Erreur client (400 Bad Request, 404 Not Found)
- **5xx** : Erreur serveur (500 Internal Server Error)

## Package `net/http`

Go fournit le package `net/http` qui contient tout ce dont vous avez besoin pour créer des clients et serveurs HTTP.

```go
import "net/http"
```

## Créer un serveur HTTP

### Serveur basique

Voici le plus simple serveur HTTP possible :

```go
package main

import (
    "fmt"
    "net/http"
    "log"
)

func main() {
    // Définir une fonction pour gérer les requêtes
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Bonjour, vous avez visité : %s", r.URL.Path)
    })

    // Démarrer le serveur sur le port 8080
    fmt.Println("Serveur démarré sur http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Explication :**
- `http.HandleFunc("/", ...)` : Définit une fonction pour gérer les requêtes vers "/"
- `w http.ResponseWriter` : Permet d'écrire la réponse
- `r *http.Request` : Contient les informations de la requête
- `http.ListenAndServe(":8080", nil)` : Démarre le serveur sur le port 8080

### Serveur avec plusieurs routes

```go
package main

import (
    "fmt"
    "net/http"
    "log"
)

func homePage(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Bienvenue sur la page d'accueil !")
}

func aboutPage(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "À propos de notre site")
}

func contactPage(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Contactez-nous à : contact@example.com")
}

func main() {
    // Définir plusieurs routes
    http.HandleFunc("/", homePage)
    http.HandleFunc("/about", aboutPage)
    http.HandleFunc("/contact", contactPage)

    fmt.Println("Serveur démarré sur http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Serveur avec gestion des méthodes HTTP

```go
package main

import (
    "fmt"
    "net/http"
    "log"
)

func userHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        fmt.Fprintf(w, "Récupérer les informations utilisateur")
    case "POST":
        fmt.Fprintf(w, "Créer un nouvel utilisateur")
    case "PUT":
        fmt.Fprintf(w, "Mettre à jour un utilisateur")
    case "DELETE":
        fmt.Fprintf(w, "Supprimer un utilisateur")
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func main() {
    http.HandleFunc("/user", userHandler)

    fmt.Println("Serveur démarré sur http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Serveur avec gestion des paramètres

```go
package main

import (
    "fmt"
    "net/http"
    "log"
)

func greetHandler(w http.ResponseWriter, r *http.Request) {
    // Récupérer les paramètres de l'URL
    name := r.URL.Query().Get("name")
    age := r.URL.Query().Get("age")

    if name == "" {
        name = "Visiteur"
    }

    if age == "" {
        fmt.Fprintf(w, "Bonjour %s !", name)
    } else {
        fmt.Fprintf(w, "Bonjour %s, vous avez %s ans !", name, age)
    }
}

func main() {
    http.HandleFunc("/greet", greetHandler)

    fmt.Println("Serveur démarré sur http://localhost:8080")
    fmt.Println("Testez avec : http://localhost:8080/greet?name=Alice&age=25")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Créer un client HTTP

### Client GET basique

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "log"
)

func main() {
    // Effectuer une requête GET
    resp, err := http.Get("https://httpbin.org/get")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close() // Important : fermer le Body !

    // Lire la réponse
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Code de statut : %d\n", resp.StatusCode)
    fmt.Printf("Réponse : %s\n", string(body))
}
```

### Client avec gestion des headers

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "log"
)

func main() {
    // Créer une requête personnalisée
    req, err := http.NewRequest("GET", "https://httpbin.org/headers", nil)
    if err != nil {
        log.Fatal(err)
    }

    // Ajouter des headers personnalisés
    req.Header.Set("User-Agent", "Mon-Client-Go/1.0")
    req.Header.Set("Accept", "application/json")

    // Créer un client HTTP
    client := &http.Client{}

    // Envoyer la requête
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Lire la réponse
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Réponse : %s\n", string(body))
}
```

### Client POST avec données

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "net/http"
    "log"
)

func main() {
    // Données à envoyer
    data := []byte(`{"nom": "Alice", "age": 30}`)

    // Créer une requête POST
    req, err := http.NewRequest("POST", "https://httpbin.org/post", bytes.NewBuffer(data))
    if err != nil {
        log.Fatal(err)
    }

    // Définir le type de contenu
    req.Header.Set("Content-Type", "application/json")

    // Envoyer la requête
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    // Lire la réponse
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Code de statut : %d\n", resp.StatusCode)
    fmt.Printf("Réponse : %s\n", string(body))
}
```

## Exemple pratique : API simple

Créons une API simple pour gérer une liste d'utilisateurs :

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "log"
    "strconv"
    "strings"
)

// Structure pour représenter un utilisateur
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

// Base de données en mémoire
var users = []User{
    {ID: 1, Name: "Alice", Age: 30},
    {ID: 2, Name: "Bob", Age: 25},
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        getAllUsers(w, r)
    case "POST":
        createUser(w, r)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func getAllUsers(w http.ResponseWriter, r *http.Request) {
    // Définir le type de contenu
    w.Header().Set("Content-Type", "application/json")

    // Encoder les utilisateurs en JSON
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var newUser User

    // Décoder le JSON de la requête
    err := json.NewDecoder(r.Body).Decode(&newUser)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Erreur lors du décodage JSON : %v", err)
        return
    }

    // Générer un ID
    newUser.ID = len(users) + 1

    // Ajouter l'utilisateur à la liste
    users = append(users, newUser)

    // Répondre avec l'utilisateur créé
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(newUser)
}

func userHandler(w http.ResponseWriter, r *http.Request) {
    // Extraire l'ID de l'URL
    path := strings.TrimPrefix(r.URL.Path, "/user/")
    id, err := strconv.Atoi(path)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "ID utilisateur invalide")
        return
    }

    switch r.Method {
    case "GET":
        getUser(w, r, id)
    case "DELETE":
        deleteUser(w, r, id)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func getUser(w http.ResponseWriter, r *http.Request, id int) {
    // Chercher l'utilisateur
    for _, user := range users {
        if user.ID == id {
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(user)
            return
        }
    }

    // Utilisateur non trouvé
    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "Utilisateur non trouvé")
}

func deleteUser(w http.ResponseWriter, r *http.Request, id int) {
    // Chercher et supprimer l'utilisateur
    for i, user := range users {
        if user.ID == id {
            users = append(users[:i], users[i+1:]...)
            w.WriteHeader(http.StatusNoContent)
            return
        }
    }

    // Utilisateur non trouvé
    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "Utilisateur non trouvé")
}

func main() {
    // Définir les routes
    http.HandleFunc("/users", usersHandler)
    http.HandleFunc("/user/", userHandler)

    fmt.Println("API démarrée sur http://localhost:8080")
    fmt.Println("Endpoints disponibles :")
    fmt.Println("  GET /users - Récupérer tous les utilisateurs")
    fmt.Println("  POST /users - Créer un utilisateur")
    fmt.Println("  GET /user/{id} - Récupérer un utilisateur")
    fmt.Println("  DELETE /user/{id} - Supprimer un utilisateur")

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Tester l'API

### Avec curl (ligne de commande)

```bash
# Récupérer tous les utilisateurs
curl http://localhost:8080/users

# Créer un utilisateur
curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Charlie", "age": 35}'

# Récupérer un utilisateur spécifique
curl http://localhost:8080/user/1

# Supprimer un utilisateur
curl -X DELETE http://localhost:8080/user/2
```

### Avec un client Go

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "log"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

func main() {
    // Test GET tous les utilisateurs
    fmt.Println("=== Récupération de tous les utilisateurs ===")
    testGetAllUsers()

    // Test POST création d'utilisateur
    fmt.Println("\n=== Création d'un utilisateur ===")
    testCreateUser()

    // Test GET utilisateur spécifique
    fmt.Println("\n=== Récupération d'un utilisateur spécifique ===")
    testGetUser(1)
}

func testGetAllUsers() {
    resp, err := http.Get("http://localhost:8080/users")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    var users []User
    json.NewDecoder(resp.Body).Decode(&users)

    fmt.Printf("Utilisateurs récupérés : %+v\n", users)
}

func testCreateUser() {
    user := User{Name: "Test User", Age: 28}
    data, _ := json.Marshal(user)

    resp, err := http.Post("http://localhost:8080/users", "application/json", bytes.NewBuffer(data))
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    var createdUser User
    json.NewDecoder(resp.Body).Decode(&createdUser)

    fmt.Printf("Utilisateur créé : %+v\n", createdUser)
}

func testGetUser(id int) {
    resp, err := http.Get(fmt.Sprintf("http://localhost:8080/user/%d", id))
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusNotFound {
        fmt.Println("Utilisateur non trouvé")
        return
    }

    var user User
    json.NewDecoder(resp.Body).Decode(&user)

    fmt.Printf("Utilisateur récupéré : %+v\n", user)
}
```

## Bonnes pratiques

### 1. Toujours fermer les réponses
```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close() // Important !
```

### 2. Gérer les codes d'erreur
```go
resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("erreur HTTP : %d", resp.StatusCode)
}
```

### 3. Définir des timeouts
```go
client := &http.Client{
    Timeout: 30 * time.Second,
}
```

### 4. Utiliser des contextes
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```

## Exercices pratiques

### Exercice 1 : Serveur de fichiers
Créez un serveur qui sert des fichiers statiques depuis un dossier.

**Solution complète :**
```go
package main

import (
    "fmt"
    "net/http"
    "log"
    "os"
)

func main() {
    // Créer le dossier static s'il n'existe pas
    if _, err := os.Stat("static"); os.IsNotExist(err) {
        os.Mkdir("static", 0755)

        // Créer un fichier HTML d'exemple
        htmlContent := `<!DOCTYPE html>
<html>
<head>
    <title>Mon site statique</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        h1 { color: #333; }
    </style>
</head>
<body>
    <h1>Bienvenue sur mon site !</h1>
    <p>Ceci est un fichier statique servi par Go.</p>
    <a href="/about.html">À propos</a>
</body>
</html>`

        f, _ := os.Create("static/index.html")
        f.WriteString(htmlContent)
        f.Close()

        // Créer une page "À propos"
        aboutContent := `<!DOCTYPE html>
<html>
<head>
    <title>À propos</title>
</head>
<body>
    <h1>À propos</h1>
    <p>Cette page est servie statiquement par Go !</p>
    <a href="/">Retour à l'accueil</a>
</body>
</html>`

        f2, _ := os.Create("static/about.html")
        f2.WriteString(aboutContent)
        f2.Close()

        fmt.Println("Dossier 'static' créé avec des fichiers d'exemple")
    }

    // Servir les fichiers du dossier "static"
    fs := http.FileServer(http.Dir("static/"))
    http.Handle("/", fs)

    // Ajouter un handler personnalisé pour afficher les logs
    http.HandleFunc("/logs", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Fichier demandé : %s\n", r.URL.Path)
        fmt.Fprintf(w, "Méthode : %s\n", r.Method)
        fmt.Fprintf(w, "User-Agent : %s\n", r.Header.Get("User-Agent"))
    })

    fmt.Println("Serveur de fichiers démarré sur http://localhost:8080")
    fmt.Println("Visitez http://localhost:8080 pour voir les fichiers")
    fmt.Println("Visitez http://localhost:8080/logs pour voir les informations de requête")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Exercice 2 : Client météo
Créez un client qui récupère la météo depuis une API publique.

**Solution avec API OpenWeatherMap :**
```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "log"
    "os"
)

// Structures pour décoder la réponse JSON
type WeatherResponse struct {
    Name    string `json:"name"`
    Main    Main   `json:"main"`
    Weather []Weather `json:"weather"`
    Wind    Wind   `json:"wind"`
}

type Main struct {
    Temp      float64 `json:"temp"`
    FeelsLike float64 `json:"feels_like"`
    Humidity  int     `json:"humidity"`
    Pressure  int     `json:"pressure"`
}

type Weather struct {
    Main        string `json:"main"`
    Description string `json:"description"`
}

type Wind struct {
    Speed float64 `json:"speed"`
}

func getWeather(city string, apiKey string) (*WeatherResponse, error) {
    // Construire l'URL de l'API
    url := fmt.Sprintf("http://api.openweathermap.org/data/2.5/weather?q=%s&appid=%s&units=metric&lang=fr", city, apiKey)

    // Faire la requête HTTP
    resp, err := http.Get(url)
    if err != nil {
        return nil, fmt.Errorf("erreur lors de la requête : %v", err)
    }
    defer resp.Body.Close()

    // Vérifier le code de statut
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("erreur API : code %d", resp.StatusCode)
    }

    // Décoder la réponse JSON
    var weather WeatherResponse
    if err := json.NewDecoder(resp.Body).Decode(&weather); err != nil {
        return nil, fmt.Errorf("erreur lors du décodage JSON : %v", err)
    }

    return &weather, nil
}

func displayWeather(weather *WeatherResponse) {
    fmt.Printf("\n🌤️  Météo pour %s\n", weather.Name)
    fmt.Printf("═══════════════════════════════\n")
    fmt.Printf("🌡️  Température : %.1f°C\n", weather.Main.Temp)
    fmt.Printf("🌡️  Ressenti : %.1f°C\n", weather.Main.FeelsLike)

    if len(weather.Weather) > 0 {
        fmt.Printf("☁️  Conditions : %s (%s)\n", weather.Weather[0].Main, weather.Weather[0].Description)
    }

    fmt.Printf("💧 Humidité : %d%%\n", weather.Main.Humidity)
    fmt.Printf("🔽 Pression : %d hPa\n", weather.Main.Pressure)
    fmt.Printf("💨 Vent : %.1f m/s\n", weather.Wind.Speed)
}

func main() {
    // Récupérer la clé API depuis les variables d'environnement
    apiKey := os.Getenv("OPENWEATHER_API_KEY")
    if apiKey == "" {
        fmt.Println("⚠️  Veuillez définir la variable d'environnement OPENWEATHER_API_KEY")
        fmt.Println("Inscrivez-vous sur https://openweathermap.org/api pour obtenir une clé gratuite")

        // Pour la démonstration, utiliser une API gratuite alternative
        fmt.Println("\n🔄 Utilisation de l'API wttr.in comme alternative...")
        getWeatherAlternative("Paris")
        return
    }

    // Liste de villes à tester
    cities := []string{"Paris", "Lyon", "Marseille", "Tokyo", "New York"}

    for _, city := range cities {
        weather, err := getWeather(city, apiKey)
        if err != nil {
            fmt.Printf("❌ Erreur pour %s : %v\n", city, err)
            continue
        }

        displayWeather(weather)
        fmt.Println()
    }
}

// Alternative gratuite sans clé API
func getWeatherAlternative(city string) {
    url := fmt.Sprintf("http://wttr.in/%s?format=j1", city)

    resp, err := http.Get(url)
    if err != nil {
        log.Printf("Erreur lors de la requête : %v", err)
        return
    }
    defer resp.Body.Close()

    // Pour wttr.in, on peut aussi utiliser un format simple
    urlSimple := fmt.Sprintf("http://wttr.in/%s?format=3", city)
    respSimple, err := http.Get(urlSimple)
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer respSimple.Body.Close()

    // Lire la réponse simple
    body := make([]byte, 1024)
    n, _ := respSimple.Body.Read(body)

    fmt.Printf("🌤️  Météo pour %s : %s\n", city, string(body[:n]))
}
```

**Pour utiliser avec OpenWeatherMap :**
```bash
# Obtenir une clé API gratuite sur https://openweathermap.org/api
export OPENWEATHER_API_KEY="votre_cle_api_ici"
go run weather_client.go
```

### Exercice 3 : API de blog
Étendez l'API utilisateurs pour gérer des articles de blog avec titre, contenu et auteur.

**Solution complète :**
```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "log"
    "strconv"
    "strings"
    "time"
)

// Structures de données
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

type Article struct {
    ID        int       `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    AuthorID  int       `json:"author_id"`
    Author    *User     `json:"author,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}

// Base de données en mémoire
var users = []User{
    {ID: 1, Name: "Alice", Age: 30},
    {ID: 2, Name: "Bob", Age: 25},
}

var articles = []Article{
    {
        ID: 1,
        Title: "Mon premier article",
        Content: "Ceci est le contenu de mon premier article de blog.",
        AuthorID: 1,
        CreatedAt: time.Now().Add(-24 * time.Hour),
        UpdatedAt: time.Now().Add(-24 * time.Hour),
    },
    {
        ID: 2,
        Title: "Go est fantastique",
        Content: "Dans cet article, je vais expliquer pourquoi Go est un excellent langage de programmation.",
        AuthorID: 2,
        CreatedAt: time.Now().Add(-12 * time.Hour),
        UpdatedAt: time.Now().Add(-12 * time.Hour),
    },
}

// Fonctions utilitaires
func findUserByID(id int) *User {
    for _, user := range users {
        if user.ID == id {
            return &user
        }
    }
    return nil
}

func findArticleByID(id int) *Article {
    for i, article := range articles {
        if article.ID == id {
            return &articles[i]
        }
    }
    return nil
}

func populateArticleAuthor(article *Article) {
    if author := findUserByID(article.AuthorID); author != nil {
        article.Author = author
    }
}

// Handlers pour les utilisateurs (identiques à l'exemple précédent)
func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        getAllUsers(w, r)
    case "POST":
        createUser(w, r)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func getAllUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var newUser User

    if err := json.NewDecoder(r.Body).Decode(&newUser); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Erreur lors du décodage JSON : %v", err)
        return
    }

    newUser.ID = len(users) + 1
    users = append(users, newUser)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(newUser)
}

// Handlers pour les articles
func articlesHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case "GET":
        getAllArticles(w, r)
    case "POST":
        createArticle(w, r)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func getAllArticles(w http.ResponseWriter, r *http.Request) {
    // Récupérer les paramètres de requête
    authorIDStr := r.URL.Query().Get("author_id")
    includeAuthor := r.URL.Query().Get("include_author") == "true"

    var filteredArticles []Article

    // Filtrer par auteur si spécifié
    if authorIDStr != "" {
        authorID, err := strconv.Atoi(authorIDStr)
        if err != nil {
            w.WriteHeader(http.StatusBadRequest)
            fmt.Fprintf(w, "ID auteur invalide")
            return
        }

        for _, article := range articles {
            if article.AuthorID == authorID {
                articleCopy := article
                if includeAuthor {
                    populateArticleAuthor(&articleCopy)
                }
                filteredArticles = append(filteredArticles, articleCopy)
            }
        }
    } else {
        // Tous les articles
        filteredArticles = make([]Article, len(articles))
        copy(filteredArticles, articles)

        if includeAuthor {
            for i := range filteredArticles {
                populateArticleAuthor(&filteredArticles[i])
            }
        }
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(filteredArticles)
}

func createArticle(w http.ResponseWriter, r *http.Request) {
    var newArticle Article

    if err := json.NewDecoder(r.Body).Decode(&newArticle); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Erreur lors du décodage JSON : %v", err)
        return
    }

    // Vérifier que l'auteur existe
    if findUserByID(newArticle.AuthorID) == nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Auteur inexistant")
        return
    }

    // Validation des champs requis
    if newArticle.Title == "" || newArticle.Content == "" {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Titre et contenu requis")
        return
    }

    // Générer un ID et définir les timestamps
    newArticle.ID = len(articles) + 1
    newArticle.CreatedAt = time.Now()
    newArticle.UpdatedAt = time.Now()

    articles = append(articles, newArticle)

    // Inclure l'auteur dans la réponse
    populateArticleAuthor(&newArticle)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(newArticle)
}

func articleHandler(w http.ResponseWriter, r *http.Request) {
    // Extraire l'ID de l'URL
    path := strings.TrimPrefix(r.URL.Path, "/article/")
    id, err := strconv.Atoi(path)
    if err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "ID article invalide")
        return
    }

    switch r.Method {
    case "GET":
        getArticle(w, r, id)
    case "PUT":
        updateArticle(w, r, id)
    case "DELETE":
        deleteArticle(w, r, id)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
        fmt.Fprintf(w, "Méthode non autorisée")
    }
}

func getArticle(w http.ResponseWriter, r *http.Request, id int) {
    article := findArticleByID(id)
    if article == nil {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "Article non trouvé")
        return
    }

    // Inclure l'auteur par défaut pour un article individuel
    articleCopy := *article
    populateArticleAuthor(&articleCopy)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(articleCopy)
}

func updateArticle(w http.ResponseWriter, r *http.Request, id int) {
    article := findArticleByID(id)
    if article == nil {
        w.WriteHeader(http.StatusNotFound)
        fmt.Fprintf(w, "Article non trouvé")
        return
    }

    var updatedArticle Article
    if err := json.NewDecoder(r.Body).Decode(&updatedArticle); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Erreur lors du décodage JSON : %v", err)
        return
    }

    // Mettre à jour les champs modifiables
    if updatedArticle.Title != "" {
        article.Title = updatedArticle.Title
    }
    if updatedArticle.Content != "" {
        article.Content = updatedArticle.Content
    }
    article.UpdatedAt = time.Now()

    populateArticleAuthor(article)

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(article)
}

func deleteArticle(w http.ResponseWriter, r *http.Request, id int) {
    for i, article := range articles {
        if article.ID == id {
            articles = append(articles[:i], articles[i+1:]...)
            w.WriteHeader(http.StatusNoContent)
            return
        }
    }

    w.WriteHeader(http.StatusNotFound)
    fmt.Fprintf(w, "Article non trouvé")
}

// Handler pour les statistiques
func statsHandler(w http.ResponseWriter, r *http.Request) {
    stats := map[string]interface{}{
        "total_users":    len(users),
        "total_articles": len(articles),
        "articles_by_author": func() map[string]int {
            authorStats := make(map[string]int)
            for _, article := range articles {
                if author := findUserByID(article.AuthorID); author != nil {
                    authorStats[author.Name]++
                }
            }
            return authorStats
        }(),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}

func main() {
    // Routes pour les utilisateurs
    http.HandleFunc("/users", usersHandler)

    // Routes pour les articles
    http.HandleFunc("/articles", articlesHandler)
    http.HandleFunc("/article/", articleHandler)

    // Route pour les statistiques
    http.HandleFunc("/stats", statsHandler)

    // Page d'accueil avec documentation
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/" {
            w.WriteHeader(http.StatusNotFound)
            fmt.Fprintf(w, "Page non trouvée")
            return
        }

        fmt.Fprintf(w, `
🚀 API Blog - Documentation

Endpoints disponibles :

📝 Articles :
  GET /articles - Récupérer tous les articles
    ?author_id=X - Filtrer par auteur
    ?include_author=true - Inclure les infos auteur
  POST /articles - Créer un article
  GET /article/{id} - Récupérer un article
  PUT /article/{id} - Mettre à jour un article
  DELETE /article/{id} - Supprimer un article

👥 Utilisateurs :
  GET /users - Récupérer tous les utilisateurs
  POST /users - Créer un utilisateur

📊 Statistiques :
  GET /stats - Statistiques du blog

Exemples :
  curl http://localhost:8080/articles
  curl http://localhost:8080/articles?include_author=true
  curl -X POST http://localhost:8080/articles \
    -H "Content-Type: application/json" \
    -d '{"title":"Mon article","content":"Contenu...","author_id":1}'
`)
    })

    fmt.Println("🚀 API Blog démarrée sur http://localhost:8080")
    fmt.Println("📖 Visitez http://localhost:8080 pour la documentation")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Client de test pour l'API Blog :**
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "log"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Age  int    `json:"age"`
}

type Article struct {
    ID       int    `json:"id"`
    Title    string `json:"title"`
    Content  string `json:"content"`
    AuthorID int    `json:"author_id"`
    Author   *User  `json:"author,omitempty"`
}

func main() {
    baseURL := "http://localhost:8080"

    fmt.Println("🧪 Test de l'API Blog")
    fmt.Println("=====================")

    // Test 1: Récupérer tous les articles
    fmt.Println("\n📖 1. Récupération de tous les articles")
    testGetAllArticles(baseURL)

    // Test 2: Créer un nouvel utilisateur
    fmt.Println("\n👤 2. Création d'un utilisateur")
    testCreateUser(baseURL)

    // Test 3: Créer un nouvel article
    fmt.Println("\n📝 3. Création d'un article")
    testCreateArticle(baseURL)

    // Test 4: Récupérer les articles avec auteurs
    fmt.Println("\n📚 4. Articles avec informations auteur")
    testGetArticlesWithAuthors(baseURL)

    // Test 5: Statistiques
    fmt.Println("\n📊 5. Statistiques du blog")
    testGetStats(baseURL)
}

func testGetAllArticles(baseURL string) {
    resp, err := http.Get(baseURL + "/articles")
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer resp.Body.Close()

    var articles []Article
    json.NewDecoder(resp.Body).Decode(&articles)

    fmt.Printf("📄 %d articles trouvés\n", len(articles))
    for _, article := range articles {
        fmt.Printf("  - %s (ID: %d)\n", article.Title, article.ID)
    }
}

func testCreateUser(baseURL string) {
    user := User{Name: "Charlie", Age: 28}
    data, _ := json.Marshal(user)

    resp, err := http.Post(baseURL+"/users", "application/json", bytes.NewBuffer(data))
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer resp.Body.Close()

    var createdUser User
    json.NewDecoder(resp.Body).Decode(&createdUser)

    fmt.Printf("✅ Utilisateur créé : %s (ID: %d)\n", createdUser.Name, createdUser.ID)
}

func testCreateArticle(baseURL string) {
    article := Article{
        Title:    "Test automatisé",
        Content:  "Cet article a été créé par un test automatisé en Go.",
        AuthorID: 1,
    }
    data, _ := json.Marshal(article)

    resp, err := http.Post(baseURL+"/articles", "application/json", bytes.NewBuffer(data))
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer resp.Body.Close()

    var createdArticle Article
    json.NewDecoder(resp.Body).Decode(&createdArticle)

    fmt.Printf("✅ Article créé : %s (ID: %d)\n", createdArticle.Title, createdArticle.ID)
    if createdArticle.Author != nil {
        fmt.Printf("   Auteur : %s\n", createdArticle.Author.Name)
    }
}

func testGetArticlesWithAuthors(baseURL string) {
    resp, err := http.Get(baseURL + "/articles?include_author=true")
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer resp.Body.Close()

    var articles []Article
    json.NewDecoder(resp.Body).Decode(&articles)

    for _, article := range articles {
        authorName := "Inconnu"
        if article.Author != nil {
            authorName = article.Author.Name
        }
        fmt.Printf("📖 %s par %s\n", article.Title, authorName)
    }
}

func testGetStats(baseURL string) {
    resp, err := http.Get(baseURL + "/stats")
    if err != nil {
        log.Printf("Erreur : %v", err)
        return
    }
    defer resp.Body.Close()

    var stats map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&stats)

    fmt.Printf("📊 Statistiques :\n")
    for key, value := range stats {
        fmt.Printf("   %s: %v\n", key, value)
    }
}
```

## Résumé

Dans cette section, vous avez appris :

- **Concepts HTTP** : Méthodes, codes de statut, requête-réponse
- **Serveurs HTTP** : Créer des serveurs avec routes et gestion des méthodes
- **Clients HTTP** : Effectuer des requêtes GET, POST avec headers personnalisés
- **API REST** : Créer une API simple avec JSON
- **Bonnes pratiques** : Gestion des erreurs, timeouts, fermeture des ressources

Le package `net/http` de Go offre tout ce dont vous avez besoin pour créer des applications web robustes et performantes. Dans la section suivante, nous approfondirons le travail avec JSON et la sérialisation de données.

---

*Félicitations ! Vous maîtrisez maintenant les bases de HTTP en Go. Ces concepts sont essentiels pour développer des applications web modernes.*

⏭️
