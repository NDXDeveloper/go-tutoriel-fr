🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20-3 : Sécurité

## Introduction à la sécurité en Go

### Pourquoi la sécurité est-elle cruciale ?

La sécurité n'est pas une fonctionnalité qu'on ajoute à la fin du développement, c'est un aspect fondamental qui doit être intégré dès le début. En Go, comme dans tout langage, les applications peuvent être vulnérables à diverses attaques si elles ne sont pas correctement sécurisées.

### Les principes de base de la sécurité

```mermaid
graph TD
    A[Sécurité] --> B[Authentification]
    A --> C[Autorisation]
    A --> D[Chiffrement]
    A --> E[Validation des données]
    A --> F[Logging sécurisé]

    B --> G[Qui êtes-vous ?]
    C --> H[Que pouvez-vous faire ?]
    D --> I[Protection des données]
    E --> J[Prévention des attaques]
    F --> K[Traçabilité]
```

**Les 5 piliers de la sécurité :**
1. **Authentification** : Vérifier l'identité des utilisateurs
2. **Autorisation** : Contrôler l'accès aux ressources
3. **Chiffrement** : Protéger les données sensibles
4. **Validation** : S'assurer de l'intégrité des données
5. **Logging** : Tracer les événements de sécurité

## 1. Authentification

### JWT (JSON Web Tokens) - Implémentation basique

```go
// auth.go - Authentification avec JWT
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "golang.org/x/crypto/bcrypt"
)

// Structure pour les utilisateurs
type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Password string `json:"-"` // Ne jamais renvoyer le mot de passe
}

// Claims JWT personnalisés
type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    jwt.RegisteredClaims
}

// Clé secrète pour signer les tokens (à stocker en sécurité !)
var jwtSecret = []byte("votre-cle-secrete-super-longue-et-complexe")

// Base de données simulée des utilisateurs
var users = []User{
    {ID: 1, Username: "alice", Email: "alice@example.com", Password: "$2a$10$..."}, // Mot de passe hashé
    {ID: 2, Username: "bob", Email: "bob@example.com", Password: "$2a$10$..."},
}

func main() {
    http.HandleFunc("/register", registerHandler)
    http.HandleFunc("/login", loginHandler)
    http.HandleFunc("/profile", authenticateMiddleware(profileHandler))
    http.HandleFunc("/admin", authenticateMiddleware(adminHandler))

    fmt.Println("🔐 Serveur sécurisé démarré sur :8080")
    fmt.Println("📝 Endpoints disponibles:")
    fmt.Println("  POST /register - Inscription")
    fmt.Println("  POST /login - Connexion")
    fmt.Println("  GET /profile - Profil (authentifié)")
    fmt.Println("  GET /admin - Admin (authentifié)")

    http.ListenAndServe(":8080", nil)
}

// Hachage sécurisé des mots de passe
func hashPassword(password string) (string, error) {
    // Coût de 12 pour un bon équilibre sécurité/performance
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 12)
    if err != nil {
        return "", err
    }
    return string(hash), nil
}

// Vérification du mot de passe
func checkPassword(hashedPassword, password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password))
    return err == nil
}

// Génération d'un token JWT
func generateToken(user User) (string, error) {
    expirationTime := time.Now().Add(24 * time.Hour)

    claims := &Claims{
        UserID:   user.ID,
        Username: user.Username,
        Email:    user.Email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(expirationTime),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "monapp",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// Validation d'un token JWT
func validateToken(tokenString string) (*Claims, error) {
    claims := &Claims{}

    token, err := jwt.ParseWithClaims(tokenString, claims, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("méthode de signature inattendue: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    if !token.Valid {
        return nil, fmt.Errorf("token invalide")
    }

    return claims, nil
}

// Handler d'inscription
func registerHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Méthode non autorisée", http.StatusMethodNotAllowed)
        return
    }

    var newUser struct {
        Username string `json:"username"`
        Email    string `json:"email"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&newUser); err != nil {
        http.Error(w, "Données invalides", http.StatusBadRequest)
        return
    }

    // Validation des données
    if len(newUser.Username) < 3 {
        http.Error(w, "Le nom d'utilisateur doit faire au moins 3 caractères", http.StatusBadRequest)
        return
    }

    if len(newUser.Password) < 8 {
        http.Error(w, "Le mot de passe doit faire au moins 8 caractères", http.StatusBadRequest)
        return
    }

    // Vérifier si l'utilisateur existe déjà
    for _, user := range users {
        if user.Username == newUser.Username || user.Email == newUser.Email {
            http.Error(w, "Utilisateur déjà existant", http.StatusConflict)
            return
        }
    }

    // Hacher le mot de passe
    hashedPassword, err := hashPassword(newUser.Password)
    if err != nil {
        http.Error(w, "Erreur lors du hachage du mot de passe", http.StatusInternalServerError)
        return
    }

    // Créer le nouvel utilisateur
    user := User{
        ID:       len(users) + 1,
        Username: newUser.Username,
        Email:    newUser.Email,
        Password: hashedPassword,
    }
    users = append(users, user)

    // Générer un token
    token, err := generateToken(user)
    if err != nil {
        http.Error(w, "Erreur lors de la génération du token", http.StatusInternalServerError)
        return
    }

    response := map[string]interface{}{
        "message": "Inscription réussie",
        "token":   token,
        "user": map[string]interface{}{
            "id":       user.ID,
            "username": user.Username,
            "email":    user.Email,
        },
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// Handler de connexion
func loginHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Méthode non autorisée", http.StatusMethodNotAllowed)
        return
    }

    var credentials struct {
        Username string `json:"username"`
        Password string `json:"password"`
    }

    if err := json.NewDecoder(r.Body).Decode(&credentials); err != nil {
        http.Error(w, "Données invalides", http.StatusBadRequest)
        return
    }

    // Rechercher l'utilisateur
    var foundUser *User
    for _, user := range users {
        if user.Username == credentials.Username || user.Email == credentials.Username {
            foundUser = &user
            break
        }
    }

    if foundUser == nil {
        http.Error(w, "Utilisateur non trouvé", http.StatusUnauthorized)
        return
    }

    // Vérifier le mot de passe
    if !checkPassword(foundUser.Password, credentials.Password) {
        http.Error(w, "Mot de passe incorrect", http.StatusUnauthorized)
        return
    }

    // Générer un token
    token, err := generateToken(*foundUser)
    if err != nil {
        http.Error(w, "Erreur lors de la génération du token", http.StatusInternalServerError)
        return
    }

    response := map[string]interface{}{
        "message": "Connexion réussie",
        "token":   token,
        "user": map[string]interface{}{
            "id":       foundUser.ID,
            "username": foundUser.Username,
            "email":    foundUser.Email,
        },
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// Middleware d'authentification
func authenticateMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Récupérer le token depuis l'en-tête Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Token d'authentification requis", http.StatusUnauthorized)
            return
        }

        // Format: "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Format de token invalide", http.StatusUnauthorized)
            return
        }

        token := parts[1]

        // Valider le token
        claims, err := validateToken(token)
        if err != nil {
            http.Error(w, "Token invalide: "+err.Error(), http.StatusUnauthorized)
            return
        }

        // Ajouter les informations utilisateur au contexte
        r.Header.Set("X-User-ID", fmt.Sprintf("%d", claims.UserID))
        r.Header.Set("X-Username", claims.Username)

        next(w, r)
    }
}

// Handler protégé - profil utilisateur
func profileHandler(w http.ResponseWriter, r *http.Request) {
    userID := r.Header.Get("X-User-ID")
    username := r.Header.Get("X-Username")

    response := map[string]interface{}{
        "message":  "Profil utilisateur",
        "user_id":  userID,
        "username": username,
        "data":     "Données sensibles du profil",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// Handler protégé - administration
func adminHandler(w http.ResponseWriter, r *http.Request) {
    username := r.Header.Get("X-Username")

    // Vérification des droits admin (simulation)
    if username != "alice" {
        http.Error(w, "Accès interdit - Droits administrateur requis", http.StatusForbidden)
        return
    }

    response := map[string]interface{}{
        "message": "Panneau d'administration",
        "admin":   username,
        "users":   len(users),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

### Test de l'authentification

```bash
# 1. Inscription
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","email":"test@example.com","password":"motdepasse123"}'

# 2. Connexion
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"motdepasse123"}'

# 3. Accès au profil (remplacez TOKEN par le token reçu)
curl -H "Authorization: Bearer TOKEN" http://localhost:8080/profile

# 4. Tentative d'accès admin
curl -H "Authorization: Bearer TOKEN" http://localhost:8080/admin
```

## 2. Validation et sanitization des données

### Protection contre les injections

```go
// validation.go - Validation et sanitization
package main

import (
    "fmt"
    "html"
    "net/http"
    "regexp"
    "strconv"
    "strings"
    "unicode"
)

// Validateur de données
type Validator struct {
    errors map[string]string
}

func NewValidator() *Validator {
    return &Validator{
        errors: make(map[string]string),
    }
}

func (v *Validator) AddError(field, message string) {
    v.errors[field] = message
}

func (v *Validator) IsValid() bool {
    return len(v.errors) == 0
}

func (v *Validator) GetErrors() map[string]string {
    return v.errors
}

// Validation d'email
func (v *Validator) ValidateEmail(field, email string) {
    emailRegex := regexp.MustCompile(`^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$`)
    if !emailRegex.MatchString(strings.ToLower(email)) {
        v.AddError(field, "Format d'email invalide")
    }
}

// Validation de mot de passe fort
func (v *Validator) ValidatePassword(field, password string) {
    if len(password) < 8 {
        v.AddError(field, "Le mot de passe doit faire au moins 8 caractères")
        return
    }

    var hasUpper, hasLower, hasNumber, hasSpecial bool

    for _, char := range password {
        switch {
        case unicode.IsUpper(char):
            hasUpper = true
        case unicode.IsLower(char):
            hasLower = true
        case unicode.IsNumber(char):
            hasNumber = true
        case unicode.IsPunct(char) || unicode.IsSymbol(char):
            hasSpecial = true
        }
    }

    if !hasUpper {
        v.AddError(field, "Le mot de passe doit contenir au moins une majuscule")
    }
    if !hasLower {
        v.AddError(field, "Le mot de passe doit contenir au moins une minuscule")
    }
    if !hasNumber {
        v.AddError(field, "Le mot de passe doit contenir au moins un chiffre")
    }
    if !hasSpecial {
        v.AddError(field, "Le mot de passe doit contenir au moins un caractère spécial")
    }
}

// Validation de nom d'utilisateur
func (v *Validator) ValidateUsername(field, username string) {
    if len(username) < 3 || len(username) > 20 {
        v.AddError(field, "Le nom d'utilisateur doit faire entre 3 et 20 caractères")
        return
    }

    usernameRegex := regexp.MustCompile(`^[a-zA-Z0-9_]+$`)
    if !usernameRegex.MatchString(username) {
        v.AddError(field, "Le nom d'utilisateur ne peut contenir que des lettres, chiffres et underscores")
    }
}

// Sanitization HTML
func SanitizeHTML(input string) string {
    return html.EscapeString(input)
}

// Sanitization SQL (base)
func SanitizeSQL(input string) string {
    // Supprimer les caractères dangereux pour SQL
    dangerous := []string{"'", "\"", ";", "--", "/*", "*/", "xp_", "sp_"}

    for _, danger := range dangerous {
        input = strings.ReplaceAll(input, danger, "")
    }

    return strings.TrimSpace(input)
}

// Validation de requête sécurisée
func secureSearchHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Méthode non autorisée", http.StatusMethodNotAllowed)
        return
    }

    // Récupération et validation des paramètres
    query := r.URL.Query().Get("q")
    limitStr := r.URL.Query().Get("limit")

    validator := NewValidator()

    // Validation de la requête de recherche
    if query == "" {
        validator.AddError("q", "Paramètre de recherche requis")
    } else if len(query) > 100 {
        validator.AddError("q", "La requête de recherche est trop longue (max 100 caractères)")
    }

    // Validation de la limite
    var limit int = 10 // valeur par défaut
    if limitStr != "" {
        var err error
        limit, err = strconv.Atoi(limitStr)
        if err != nil || limit < 1 || limit > 100 {
            validator.AddError("limit", "La limite doit être un nombre entre 1 et 100")
        }
    }

    if !validator.IsValid() {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusBadRequest)
        response := map[string]interface{}{
            "error":  "Données invalides",
            "errors": validator.GetErrors(),
        }
        json.NewEncoder(w).Encode(response)
        return
    }

    // Sanitization de la requête
    safeQuery := SanitizeHTML(SanitizeSQL(query))

    // Simulation de recherche sécurisée
    results := performSecureSearch(safeQuery, limit)

    response := map[string]interface{}{
        "query":   safeQuery,
        "limit":   limit,
        "results": results,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func performSecureSearch(query string, limit int) []string {
    // Simulation de résultats de recherche
    allResults := []string{
        "Résultat 1 pour " + query,
        "Résultat 2 pour " + query,
        "Résultat 3 pour " + query,
        "Résultat 4 pour " + query,
        "Résultat 5 pour " + query,
    }

    if limit > len(allResults) {
        limit = len(allResults)
    }

    return allResults[:limit]
}
```

## 3. Protection CSRF et XSS

### Middleware anti-CSRF

```go
// csrf.go - Protection CSRF
package main

import (
    "crypto/rand"
    "encoding/hex"
    "fmt"
    "net/http"
    "strings"
    "time"
)

// Stockage des tokens CSRF (en production, utilisez Redis ou une base de données)
var csrfTokens = make(map[string]time.Time)

// Génération d'un token CSRF
func generateCSRFToken() (string, error) {
    bytes := make([]byte, 32)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }

    token := hex.EncodeToString(bytes)
    csrfTokens[token] = time.Now().Add(1 * time.Hour) // Expire dans 1 heure

    return token, nil
}

// Validation d'un token CSRF
func validateCSRFToken(token string) bool {
    expiryTime, exists := csrfTokens[token]
    if !exists {
        return false
    }

    if time.Now().After(expiryTime) {
        delete(csrfTokens, token)
        return false
    }

    return true
}

// Middleware de protection CSRF
func csrfProtection(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Générer un token pour les requêtes GET
        if r.Method == http.MethodGet {
            token, err := generateCSRFToken()
            if err != nil {
                http.Error(w, "Erreur serveur", http.StatusInternalServerError)
                return
            }

            // Ajouter le token aux en-têtes et cookies
            w.Header().Set("X-CSRF-Token", token)
            http.SetCookie(w, &http.Cookie{
                Name:     "csrf_token",
                Value:    token,
                HttpOnly: true,
                Secure:   true, // En production avec HTTPS
                SameSite: http.SameSiteStrictMode,
                Path:     "/",
            })
        }

        // Valider le token pour les requêtes modifiantes
        if r.Method == http.MethodPost || r.Method == http.MethodPut ||
           r.Method == http.MethodDelete || r.Method == http.MethodPatch {

            token := r.Header.Get("X-CSRF-Token")
            if token == "" {
                // Essayer de récupérer depuis le formulaire
                token = r.FormValue("csrf_token")
            }

            if token == "" || !validateCSRFToken(token) {
                http.Error(w, "Token CSRF invalide ou manquant", http.StatusForbidden)
                return
            }

            // Supprimer le token après utilisation
            delete(csrfTokens, token)
        }

        next(w, r)
    }
}

// Nettoyage périodique des tokens expirés
func cleanupExpiredTokens() {
    ticker := time.NewTicker(10 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            now := time.Now()
            for token, expiry := range csrfTokens {
                if now.After(expiry) {
                    delete(csrfTokens, token)
                }
            }
        }
    }
}

// Handler sécurisé pour formulaire
func secureFormHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodGet {
        // Afficher le formulaire avec le token CSRF
        token := w.Header().Get("X-CSRF-Token")

        html := fmt.Sprintf(`
<!DOCTYPE html>
<html>
<head>
    <title>Formulaire Sécurisé</title>
</head>
<body>
    <h1>Formulaire Protégé CSRF</h1>
    <form method="POST" action="/secure-form">
        <input type="hidden" name="csrf_token" value="%s">
        <label>Nom: <input type="text" name="name" required></label><br><br>
        <label>Email: <input type="email" name="email" required></label><br><br>
        <label>Message: <textarea name="message" required></textarea></label><br><br>
        <input type="submit" value="Envoyer">
    </form>
</body>
</html>`, token)

        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(html))
        return
    }

    if r.Method == http.MethodPost {
        // Traitement sécurisé du formulaire
        name := SanitizeHTML(r.FormValue("name"))
        email := SanitizeHTML(r.FormValue("email"))
        message := SanitizeHTML(r.FormValue("message"))

        validator := NewValidator()
        validator.ValidateEmail("email", email)

        if name == "" {
            validator.AddError("name", "Le nom est requis")
        }

        if message == "" {
            validator.AddError("message", "Le message est requis")
        }

        if !validator.IsValid() {
            w.WriteHeader(http.StatusBadRequest)
            w.Write([]byte("Données invalides"))
            return
        }

        // Traitement réussi
        response := fmt.Sprintf(`
<!DOCTYPE html>
<html>
<head>
    <title>Succès</title>
</head>
<body>
    <h1>Message envoyé avec succès !</h1>
    <p>Nom: %s</p>
    <p>Email: %s</p>
    <p>Message: %s</p>
    <a href="/secure-form">Retour au formulaire</a>
</body>
</html>`, name, email, message)

        w.Header().Set("Content-Type", "text/html")
        w.Write([]byte(response))
    }
}
```

## 4. Sécurisation des en-têtes HTTP

### Middleware de sécurité des en-têtes

```go
// headers.go - Sécurisation des en-têtes HTTP
package main

import (
    "net/http"
)

// Middleware pour sécuriser les en-têtes HTTP
func securityHeaders(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Protection XSS
        w.Header().Set("X-XSS-Protection", "1; mode=block")

        // Prévention du sniffing de type MIME
        w.Header().Set("X-Content-Type-Options", "nosniff")

        // Protection contre le clickjacking
        w.Header().Set("X-Frame-Options", "DENY")

        // Force HTTPS (si applicable)
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

        // Politique de contenu sécurisée (CSP)
        csp := "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "font-src 'self'; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none'"
        w.Header().Set("Content-Security-Policy", csp)

        // Contrôle du referrer
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")

        // Permissions par défaut restrictives
        w.Header().Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")

        // Supprimer les en-têtes révélant des informations sur le serveur
        w.Header().Del("Server")
        w.Header().Del("X-Powered-By")

        next(w, r)
    }
}

// Configuration CORS sécurisée
func corsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Domaines autorisés (à configurer selon vos besoins)
        allowedOrigins := []string{
            "https://monapp.com",
            "https://www.monapp.com",
        }

        origin := r.Header.Get("Origin")

        // Vérifier si l'origine est autorisée
        for _, allowedOrigin := range allowedOrigins {
            if origin == allowedOrigin {
                w.Header().Set("Access-Control-Allow-Origin", origin)
                break
            }
        }

        // Méthodes autorisées
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")

        // En-têtes autorisés
        w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type, Authorization, X-CSRF-Token")

        // Permettre les cookies
        w.Header().Set("Access-Control-Allow-Credentials", "true")

        // Durée de cache pour les requêtes preflight
        w.Header().Set("Access-Control-Max-Age", "86400")

        // Traiter les requêtes OPTIONS (preflight)
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusOK)
            return
        }

        next(w, r)
    }
}

// Middleware de limitation de taux (rate limiting)
func rateLimitMiddleware(next http.HandlerFunc) http.HandlerFunc {
    // Stockage simple en mémoire (en production, utilisez Redis)
    requests := make(map[string][]time.Time)
    var mu sync.RWMutex

    return func(w http.ResponseWriter, r *http.Request) {
        clientIP := r.RemoteAddr
        now := time.Now()

        mu.Lock()
        defer mu.Unlock()

        // Nettoyer les anciennes requêtes (plus de 1 minute)
        var recentRequests []time.Time
        for _, requestTime := range requests[clientIP] {
            if now.Sub(requestTime) < time.Minute {
                recentRequests = append(recentRequests, requestTime)
            }
        }

        // Vérifier la limite (60 requêtes par minute)
        if len(recentRequests) >= 60 {
            w.Header().Set("Retry-After", "60")
            http.Error(w, "Trop de requêtes. Réessayez dans une minute.", http.StatusTooManyRequests)
            return
        }

        // Ajouter la requête actuelle
        recentRequests = append(recentRequests, now)
        requests[clientIP] = recentRequests

        // Ajouter des en-têtes informatifs
        w.Header().Set("X-RateLimit-Limit", "60")
        w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", 60-len(recentRequests)))
        w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", now.Add(time.Minute).Unix()))

        next(w, r)
    }
}

// Nettoyage périodique de la mémoire
func cleanupRateLimit() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // Nettoyage des données anciennes
            // En production, implémentez ceci avec Redis et TTL
        }
    }
}
```

## 5. Gestion sécurisée des secrets

### Configuration des secrets d'environnement

```go
// secrets.go - Gestion sécurisée des secrets
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "errors"
    "fmt"
    "io"
    "os"
    "strings"
)

// Structure pour la configuration des secrets
type SecretConfig struct {
    JWTSecret      string
    DatabaseURL    string
    APIKey         string
    EncryptionKey  string
}

// Chargement sécurisé de la configuration
func LoadSecretConfig() (*SecretConfig, error) {
    config := &SecretConfig{
        JWTSecret:     getEnvOrPanic("JWT_SECRET"),
        DatabaseURL:   getEnvOrPanic("DATABASE_URL"),
        APIKey:        getEnvOrPanic("API_KEY"),
        EncryptionKey: getEnvOrPanic("ENCRYPTION_KEY"),
    }

    // Validation des secrets
    if err := validateSecrets(config); err != nil {
        return nil, err
    }

    return config, nil
}

func getEnvOrPanic(key string) string {
    value := os.Getenv(key)
    if value == "" {
        panic(fmt.Sprintf("Variable d'environnement requise non définie: %s", key))
    }
    return value
}

func validateSecrets(config *SecretConfig) error {
    // Validation de la clé JWT
    if len(config.JWTSecret) < 32 {
        return errors.New("JWT_SECRET doit faire au moins 32 caractères")
    }

    // Validation de l'URL de base de données
    if !strings.HasPrefix(config.DatabaseURL, "postgres://") &&
       !strings.HasPrefix(config.DatabaseURL, "mysql://") {
        return errors.New("DATABASE_URL doit être une URL de base de données valide")
    }

    // Validation de la clé de chiffrement
    if len(config.EncryptionKey) != 32 {
        return errors.New("ENCRYPTION_KEY doit faire exactement 32 caractères")
    }

    return nil
}

// Chiffrement AES-256-GCM
func EncryptData(plaintext, key string) (string, error) {
    keyBytes := []byte(key)
    if len(keyBytes) != 32 {
        return "", errors.New("la clé doit faire 32 bytes")
    }

    block, err := aes.NewCipher(keyBytes)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", err
    }

    ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// Déchiffrement AES-256-GCM
func DecryptData(ciphertext, key string) (string, error) {
    keyBytes := []byte(key)
    if len(keyBytes) != 32 {
        return "", errors.New("la clé doit faire 32 bytes")
    }

    data, err := base64.StdEncoding.DecodeString(ciphertext)
    if err != nil {
        return "", err
    }

    block, err := aes.NewCipher(keyBytes)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    if len(data) < gcm.NonceSize() {
        return "", errors.New("données chiffrées trop courtes")
    }

    nonce := data[:gcm.NonceSize()]
    ciphertext_bytes := data[gcm.NonceSize():]

    plaintext, err := gcm.Open(nil, nonce, ciphertext_bytes, nil)
    if err != nil {
        return "", err
    }

    return string(plaintext), nil
}

// Stockage sécurisé de données sensibles
func secureDataHandler(w http.ResponseWriter, r *http.Request) {
    config, err := LoadSecretConfig()
    if err != nil {
        http.Error(w, "Erreur de configuration", http.StatusInternalServerError)
        return
    }

    if r.Method == http.MethodPost {
        // Chiffrer des données sensibles
        sensitiveData := r.FormValue("data")
        if sensitiveData == "" {
            http.Error(w, "Données requises", http.StatusBadRequest)
            return
        }

        encryptedData, err := EncryptData(sensitiveData, config.EncryptionKey)
        if err != nil {
            http.Error(w, "Erreur de chiffrement", http.StatusInternalServerError)
            return
        }

        response := map[string]interface{}{
            "message":        "Données chiffrées avec succès",
            "encrypted_data": encryptedData,
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(response)
        return
    }

    if r.Method == http.MethodGet {
        // Déchiffrer des données
        encryptedData := r.URL.Query().Get("data")
        if encryptedData == "" {
            http.Error(w, "Données chiffrées requises", http.StatusBadRequest)
            return
        }

        decryptedData, err := DecryptData(encryptedData, config.EncryptionKey)
        if err != nil {
            http.Error(w, "Erreur de déchiffrement", http.StatusBadRequest)
            return
        }

        response := map[string]interface{}{
            "message":        "Données déchiffrées avec succès",
            "decrypted_data": decryptedData,
        }

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(response)
    }
}
```

### Fichier .env d'exemple

```bash
# .env - Variables d'environnement (NE JAMAIS COMMITER CE FICHIER !)
JWT_SECRET=votre-cle-jwt-super-secrete-de-plus-de-32-caracteres
DATABASE_URL=postgres://user:password@localhost:5432/myapp
API_KEY=sk-1234567890abcdef
ENCRYPTION_KEY=abcdefghijklmnopqrstuvwxyz123456
```

## 6. Logging sécurisé

### Logger sécurisé avec filtrage des données sensibles

```go
// secure_logging.go - Logging sécurisé
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "regexp"
    "strings"
    "time"

    "github.com/sirupsen/logrus"
)

// Configuration du logger sécurisé
var secureLogger = logrus.New()

func init() {
    secureLogger.SetFormatter(&SecureJSONFormatter{})
    secureLogger.SetLevel(logrus.InfoLevel)
}

// Formatter personnalisé qui masque les données sensibles
type SecureJSONFormatter struct{}

func (f *SecureJSONFormatter) Format(entry *logrus.Entry) ([]byte, error) {
    // Copier les données pour éviter les modifications
    data := make(map[string]interface{})
    for k, v := range entry.Data {
        data[k] = f.sanitizeField(k, v)
    }

    // Ajouter les champs standard
    data["time"] = entry.Time.Format(time.RFC3339)
    data["level"] = entry.Level.String()
    data["message"] = f.sanitizeMessage(entry.Message)

    serialized, err := json.Marshal(data)
    if err != nil {
        return nil, fmt.Errorf("erreur de sérialisation JSON: %v", err)
    }

    return append(serialized, '\n'), nil
}

// Sanitisation des champs sensibles
func (f *SecureJSONFormatter) sanitizeField(key string, value interface{}) interface{} {
    keyLower := strings.ToLower(key)

    // Champs à masquer complètement
    sensitiveFields := []string{
        "password", "passwd", "secret", "token", "key", "authorization",
        "credit_card", "ssn", "social_security", "api_key",
    }

    for _, sensitive := range sensitiveFields {
        if strings.Contains(keyLower, sensitive) {
            return "[REDACTED]"
        }
    }

    // Masquer partiellement les emails et IPs
    if strings.Contains(keyLower, "email") {
        if email, ok := value.(string); ok {
            return f.maskEmail(email)
        }
    }

    if strings.Contains(keyLower, "ip") || strings.Contains(keyLower, "addr") {
        if ip, ok := value.(string); ok {
            return f.maskIP(ip)
        }
    }

    return value
}

// Sanitisation du message principal
func (f *SecureJSONFormatter) sanitizeMessage(message string) string {
    // Regex pour détecter des patterns sensibles
    patterns := map[string]string{
        `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b`: "[EMAIL]",     // Email
        `\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b`:          "[CARD]",      // Carte de crédit
        `\b\d{3}-\d{2}-\d{4}\b`:                               "[SSN]",       // SSN américain
        `password[=:]\s*\S+`:                                  "password=[REDACTED]", // Password
        `token[=:]\s*\S+`:                                     "token=[REDACTED]",    // Token
    }

    result := message
    for pattern, replacement := range patterns {
        re := regexp.MustCompile(pattern)
        result = re.ReplaceAllString(result, replacement)
    }

    return result
}

// Masquage partiel d'email
func (f *SecureJSONFormatter) maskEmail(email string) string {
    parts := strings.Split(email, "@")
    if len(parts) != 2 {
        return "[INVALID_EMAIL]"
    }

    user := parts[0]
    domain := parts[1]

    if len(user) <= 2 {
        return "*@" + domain
    }

    masked := string(user[0]) + strings.Repeat("*", len(user)-2) + string(user[len(user)-1])
    return masked + "@" + domain
}

// Masquage partiel d'IP
func (f *SecureJSONFormatter) maskIP(ip string) string {
    parts := strings.Split(ip, ".")
    if len(parts) == 4 {
        // IPv4
        return fmt.Sprintf("%s.%s.*.%s", parts[0], parts[1], parts[3])
    }

    // IPv6 ou autre format
    if len(ip) > 8 {
        return ip[:4] + "****" + ip[len(ip)-4:]
    }

    return "[IP]"
}

// Middleware de logging sécurisé
func secureLoggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrapper pour capturer la réponse
        ww := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        // Log de la requête (avec données sanitisées)
        requestLogger := secureLogger.WithFields(logrus.Fields{
            "method":      r.Method,
            "path":        r.URL.Path,
            "user_agent":  r.UserAgent(),
            "remote_ip":   r.RemoteAddr,
            "request_id":  generateRequestID(),
        })

        // Ne pas logger les paramètres sensibles
        if r.URL.RawQuery != "" && !containsSensitiveParams(r.URL.RawQuery) {
            requestLogger = requestLogger.WithField("query", r.URL.RawQuery)
        }

        requestLogger.Info("Requête reçue")

        // Traitement de la requête
        next(ww, r)

        // Log de la réponse
        duration := time.Since(start)
        responseLogger := requestLogger.WithFields(logrus.Fields{
            "status_code": ww.statusCode,
            "duration_ms": duration.Milliseconds(),
        })

        // Log différencié selon le statut
        switch {
        case ww.statusCode >= 500:
            responseLogger.Error("Erreur serveur")
        case ww.statusCode >= 400:
            responseLogger.Warn("Erreur client")
        case ww.statusCode >= 300:
            responseLogger.Info("Redirection")
        default:
            responseLogger.Info("Requête réussie")
        }
    }
}

// Vérification de paramètres sensibles
func containsSensitiveParams(query string) bool {
    sensitiveParams := []string{
        "password", "token", "secret", "key", "auth", "credential"
    }

    queryLower := strings.ToLower(query)
    for _, param := range sensitiveParams {
        if strings.Contains(queryLower, param) {
            return true
        }
    }

    return false
}

// Logger d'événements de sécurité
func logSecurityEvent(eventType, description string, r *http.Request, extra map[string]interface{}) {
    fields := logrus.Fields{
        "event_type":   eventType,
        "description":  description,
        "remote_ip":    r.RemoteAddr,
        "user_agent":   r.UserAgent(),
        "path":         r.URL.Path,
        "method":       r.Method,
        "timestamp":    time.Now().Unix(),
    }

    // Ajouter les champs supplémentaires
    for k, v := range extra {
        fields[k] = v
    }

    secureLogger.WithFields(fields).Warn("Événement de sécurité détecté")
}

// Exemple d'utilisation du logging sécurisé
func sensitiveDataHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method == http.MethodPost {
        // Log d'une tentative d'accès aux données sensibles
        logSecurityEvent("sensitive_data_access", "Tentative d'accès aux données sensibles", r, map[string]interface{}{
            "user_id": r.Header.Get("X-User-ID"),
        })

        // Traitement...
        secureLogger.WithFields(logrus.Fields{
            "user_id": r.Header.Get("X-User-ID"),
            "action":  "data_retrieved",
        }).Info("Données sensibles récupérées")

        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Données traitées"))
    }
}
```

## 7. Configuration Docker sécurisée

### Dockerfile sécurisé

```dockerfile
# Dockerfile sécurisé
FROM golang:1.21-alpine AS builder

# Installer les certificats CA
RUN apk --no-cache add ca-certificates git

# Créer un utilisateur non-root
RUN adduser -D -g '' appuser

WORKDIR /app

# Copier les fichiers de dépendances
COPY go.mod go.sum ./
RUN go mod download

# Copier le code source
COPY . .

# Compiler avec optimisations de sécurité
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags '-extldflags "-static" -s -w' \
    -o main .

# Image finale ultra-sécurisée
FROM scratch

# Copier les certificats CA
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copier le fichier passwd pour l'utilisateur
COPY --from=builder /etc/passwd /etc/passwd

# Copier l'exécutable
COPY --from=builder /app/main /main

# Utiliser l'utilisateur non-root
USER appuser

# Exposer le port (information seulement)
EXPOSE 8080

# Commande d'exécution
ENTRYPOINT ["/main"]
```

### Docker Compose sécurisé

```yaml
# docker-compose.yml sécurisé
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
      - DB_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - jwt_secret
      - db_password
    networks:
      - app_network
    restart: unless-stopped
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
    user: "1000:1000"

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_network
    restart: unless-stopped

secrets:
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  db_password:
    file: ./secrets/db_password.txt

volumes:
  db_data:
    driver: local

networks:
  app_network:
    driver: bridge
    internal: false
```

## 8. Tests de sécurité

### Tests automatisés de sécurité

```go
// security_test.go - Tests de sécurité
package main

import (
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"
)

func TestXSSProtection(t *testing.T) {
    // Test de protection XSS
    maliciousInput := "<script>alert('XSS')</script>"

    req, err := http.NewRequest("POST", "/secure-form", strings.NewReader("name="+maliciousInput))
    if err != nil {
        t.Fatal(err)
    }

    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(secureFormHandler)

    handler.ServeHTTP(rr, req)

    // Vérifier que le script est échappé
    if strings.Contains(rr.Body.String(), "<script>") {
        t.Error("Vulnérabilité XSS détectée: script non échappé")
    }
}

func TestCSRFProtection(t *testing.T) {
    // Test de protection CSRF
    req, err := http.NewRequest("POST", "/secure-form", strings.NewReader("name=test"))
    if err != nil {
        t.Fatal(err)
    }

    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")

    rr := httptest.NewRecorder()
    handler := csrfProtection(secureFormHandler)

    handler.ServeHTTP(rr, req)

    // Vérifier que la requête sans token CSRF est rejetée
    if rr.Code != http.StatusForbidden {
        t.Errorf("Protection CSRF échouée: attendu %d, reçu %d", http.StatusForbidden, rr.Code)
    }
}

func TestRateLimit(t *testing.T) {
    handler := rateLimitMiddleware(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    // Faire 61 requêtes rapidement
    for i := 0; i < 61; i++ {
        req, _ := http.NewRequest("GET", "/", nil)
        req.RemoteAddr = "127.0.0.1:12345"

        rr := httptest.NewRecorder()
        handler.ServeHTTP(rr, req)

        // La 61ème requête devrait être bloquée
        if i == 60 && rr.Code != http.StatusTooManyRequests {
            t.Errorf("Rate limiting échoué: attendu %d, reçu %d", http.StatusTooManyRequests, rr.Code)
        }
    }
}

func TestSecurityHeaders(t *testing.T) {
    req, err := http.NewRequest("GET", "/", nil)
    if err != nil {
        t.Fatal(err)
    }

    rr := httptest.NewRecorder()
    handler := securityHeaders(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    handler.ServeHTTP(rr, req)

    // Vérifier les en-têtes de sécurité
    securityHeadersToCheck := map[string]string{
        "X-XSS-Protection":           "1; mode=block",
        "X-Content-Type-Options":     "nosniff",
        "X-Frame-Options":            "DENY",
        "Strict-Transport-Security":  "max-age=31536000; includeSubDomains",
    }

    for header, expectedValue := range securityHeadersToCheck {
        if value := rr.Header().Get(header); value != expectedValue {
            t.Errorf("En-tête %s manquant ou incorrect: attendu '%s', reçu '%s'",
                header, expectedValue, value)
        }
    }
}

func TestPasswordHashing(t *testing.T) {
    password := "motdepasse123"

    // Tester le hachage
    hash, err := hashPassword(password)
    if err != nil {
        t.Fatal("Erreur lors du hachage:", err)
    }

    // Vérifier que le hash n'est pas le mot de passe en clair
    if hash == password {
        t.Error("Le mot de passe n'a pas été haché")
    }

    // Tester la vérification
    if !checkPassword(hash, password) {
        t.Error("Vérification du mot de passe échouée")
    }

    // Tester avec un mauvais mot de passe
    if checkPassword(hash, "mauvais_mot_de_passe") {
        t.Error("Vérification du mot de passe accepte un mauvais mot de passe")
    }
}

func BenchmarkPasswordHashing(b *testing.B) {
    password := "motdepasse123"

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        hashPassword(password)
    }
}
```

### 9. Autorisation et contrôle d'accès

```go
// rbac.go - Role-Based Access Control
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strings"
)

// Système de rôles et permissions
type Role string

const (
    RoleUser  Role = "user"
    RoleAdmin Role = "admin"
    RoleMod   Role = "moderator"
)

type Permission string

const (
    PermReadUser   Permission = "read:user"
    PermWriteUser  Permission = "write:user"
    PermDeleteUser Permission = "delete:user"
    PermAdminPanel Permission = "admin:panel"
)

// Mapping des rôles vers les permissions
var rolePermissions = map[Role][]Permission{
    RoleUser: {
        PermReadUser,
    },
    RoleMod: {
        PermReadUser,
        PermWriteUser,
    },
    RoleAdmin: {
        PermReadUser,
        PermWriteUser,
        PermDeleteUser,
        PermAdminPanel,
    },
}

// Vérifier si un rôle a une permission
func hasPermission(role Role, permission Permission) bool {
    permissions, exists := rolePermissions[role]
    if !exists {
        return false
    }

    for _, perm := range permissions {
        if perm == permission {
            return true
        }
    }
    return false
}

// Middleware de contrôle d'accès
func requirePermission(permission Permission) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            // Récupérer le rôle depuis le token (ajouté par le middleware d'auth)
            roleStr := r.Header.Get("X-User-Role")
            if roleStr == "" {
                http.Error(w, "Rôle utilisateur non trouvé", http.StatusForbidden)
                return
            }

            role := Role(roleStr)

            if !hasPermission(role, permission) {
                http.Error(w, "Permission insuffisante", http.StatusForbidden)
                return
            }

            next(w, r)
        }
    }
}

// Exemple d'utilisation
func adminOnlyHandler(w http.ResponseWriter, r *http.Request) {
    response := map[string]interface{}{
        "message": "Accès administrateur autorisé",
        "data":    "Données sensibles admin",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// Dans votre main :
// http.HandleFunc("/admin", requirePermission(PermAdminPanel)(adminOnlyHandler))
```

### 10. Outils d'analyse de sécurité

```bash
#!/bin/bash
# security_tools.sh - Installation et utilisation des outils de sécurité Go

echo "🔧 Installation des outils de sécurité Go"

# 1. gosec - Analyseur de sécurité statique
echo "📦 Installation de gosec..."
go install github.com/securecodewarrior/gosec/v2/cmd/gosec@latest

# 2. govulncheck - Détection de vulnérabilités
echo "📦 Installation de govulncheck..."
go install golang.org/x/vuln/cmd/govulncheck@latest

# 3. staticcheck - Analyseur statique
echo "📦 Installation de staticcheck..."
go install honnef.co/go/tools/cmd/staticcheck@latest

echo "✅ Outils installés avec succès"

echo "🔍 Exécution des analyses de sécurité..."

# Analyse avec gosec
echo "🔒 Analyse gosec..."
gosec ./...

# Vérification des vulnérabilités
echo "🛡️ Vérification des vulnérabilités..."
govulncheck ./...

# Analyse statique
echo "🔬 Analyse statique..."
staticcheck ./...

echo "✅ Analyses terminées"
```

### 11. Configuration CI/CD sécurisée

```yaml
# .github/workflows/security.yml
name: Security Checks

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup Go
      uses: actions/setup-go@v4
      with:
        go-version: 1.21

    - name: Install security tools
      run: |
        go install github.com/securecodewarrior/gosec/v2/cmd/gosec@latest
        go install golang.org/x/vuln/cmd/govulncheck@latest

    - name: Run gosec
      run: gosec ./...

    - name: Run govulncheck
      run: govulncheck ./...

    - name: Docker security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:latest'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload security results
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
```

### 12. Gestion des sessions sécurisées

```go
// sessions.go - Gestion des sessions sécurisées
package main

import (
    "crypto/rand"
    "encoding/hex"
    "net/http"
    "sync"
    "time"
)

type Session struct {
    ID        string
    UserID    int
    CreatedAt time.Time
    ExpiresAt time.Time
    Data      map[string]interface{}
}

type SessionManager struct {
    sessions map[string]*Session
    mutex    sync.RWMutex
}

func NewSessionManager() *SessionManager {
    sm := &SessionManager{
        sessions: make(map[string]*Session),
    }

    // Nettoyage périodique des sessions expirées
    go sm.cleanupExpiredSessions()

    return sm
}

func (sm *SessionManager) CreateSession(userID int) (*Session, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return nil, err
    }

    session := &Session{
        ID:        sessionID,
        UserID:    userID,
        CreatedAt: time.Now(),
        ExpiresAt: time.Now().Add(24 * time.Hour),
        Data:      make(map[string]interface{}),
    }

    sm.mutex.Lock()
    sm.sessions[sessionID] = session
    sm.mutex.Unlock()

    return session, nil
}

func (sm *SessionManager) GetSession(sessionID string) (*Session, bool) {
    sm.mutex.RLock()
    defer sm.mutex.RUnlock()

    session, exists := sm.sessions[sessionID]
    if !exists {
        return nil, false
    }

    if time.Now().After(session.ExpiresAt) {
        delete(sm.sessions, sessionID)
        return nil, false
    }

    return session, true
}

func (sm *SessionManager) DeleteSession(sessionID string) {
    sm.mutex.Lock()
    defer sm.mutex.Unlock()

    delete(sm.sessions, sessionID)
}

func generateSessionID() (string, error) {
    bytes := make([]byte, 32)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return hex.EncodeToString(bytes), nil
}

func (sm *SessionManager) cleanupExpiredSessions() {
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            sm.mutex.Lock()
            now := time.Now()
            for id, session := range sm.sessions {
                if now.After(session.ExpiresAt) {
                    delete(sm.sessions, id)
                }
            }
            sm.mutex.Unlock()
        }
    }
}

// Middleware de session
func (sm *SessionManager) SessionMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        cookie, err := r.Cookie("session_id")
        if err != nil {
            // Pas de session, continuer sans
            next(w, r)
            return
        }

        session, exists := sm.GetSession(cookie.Value)
        if !exists {
            // Session invalide, supprimer le cookie
            http.SetCookie(w, &http.Cookie{
                Name:   "session_id",
                Value:  "",
                MaxAge: -1,
            })
            next(w, r)
            return
        }

        // Ajouter les informations de session à la requête
        r.Header.Set("X-Session-User-ID", fmt.Sprintf("%d", session.UserID))

        next(w, r)
    }
}
```

### 13. Protection contre les attaques par déni de service

```go
// dos_protection.go - Protection contre les attaques DoS
package main

import (
    "context"
    "net/http"
    "sync"
    "time"
)

// Protection contre les requêtes lentes (Slowloris)
func slowlorisProtection(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Timeout pour les requêtes lentes
        ctx, cancel := context.WithTimeout(r.Context(), 30*time.Second)
        defer cancel()

        r = r.WithContext(ctx)

        done := make(chan bool, 1)
        go func() {
            next(w, r)
            done <- true
        }()

        select {
        case <-done:
            // Requête terminée normalement
        case <-ctx.Done():
            // Timeout atteint
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
        }
    }
}

// Protection contre les gros uploads
func maxBodySizeMiddleware(maxSize int64) func(http.HandlerFunc) http.HandlerFunc {
    return func(next http.HandlerFunc) http.HandlerFunc {
        return func(w http.ResponseWriter, r *http.Request) {
            if r.ContentLength > maxSize {
                http.Error(w, "Request entity too large", http.StatusRequestEntityTooLarge)
                return
            }

            r.Body = http.MaxBytesReader(w, r.Body, maxSize)
            next(w, r)
        }
    }
}

// Protection avancée contre le rate limiting par IP
type RateLimiter struct {
    visitors map[string]*Visitor
    mutex    sync.RWMutex
}

type Visitor struct {
    lastSeen time.Time
    requests int
    blocked  bool
    blockUntil time.Time
}

func NewRateLimiter() *RateLimiter {
    rl := &RateLimiter{
        visitors: make(map[string]*Visitor),
    }

    go rl.cleanup()
    return rl
}

func (rl *RateLimiter) Allow(ip string) bool {
    rl.mutex.Lock()
    defer rl.mutex.Unlock()

    now := time.Now()
    visitor, exists := rl.visitors[ip]

    if !exists {
        rl.visitors[ip] = &Visitor{
            lastSeen: now,
            requests: 1,
        }
        return true
    }

    // Si bloqué, vérifier si le blocage est expiré
    if visitor.blocked {
        if now.After(visitor.blockUntil) {
            visitor.blocked = false
            visitor.requests = 1
            visitor.lastSeen = now
            return true
        }
        return false
    }

    // Réinitialiser le compteur si plus d'une minute s'est écoulée
    if now.Sub(visitor.lastSeen) > time.Minute {
        visitor.requests = 1
        visitor.lastSeen = now
        return true
    }

    visitor.requests++
    visitor.lastSeen = now

    // Bloquer si trop de requêtes
    if visitor.requests > 60 { // 60 requêtes par minute
        visitor.blocked = true
        visitor.blockUntil = now.Add(15 * time.Minute) // Bloquer 15 minutes
        return false
    }

    return true
}

func (rl *RateLimiter) cleanup() {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            rl.mutex.Lock()
            now := time.Now()
            for ip, visitor := range rl.visitors {
                if now.Sub(visitor.lastSeen) > time.Hour {
                    delete(rl.visitors, ip)
                }
            }
            rl.mutex.Unlock()
        }
    }
}

func (rl *RateLimiter) Middleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr

        if !rl.Allow(ip) {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        next(w, r)
    }
}
```

### 14. Monitoring de sécurité

```go
// security_monitoring.go - Monitoring des événements de sécurité
package main

import (
    "encoding/json"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    securityEvents = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "security_events_total",
            Help: "Total number of security events",
        },
        []string{"event_type", "severity"},
    )

    authAttempts = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "auth_attempts_total",
            Help: "Total authentication attempts",
        },
        []string{"result", "method"},
    )
)

type SecurityEvent struct {
    Type        string    `json:"type"`
    Severity    string    `json:"severity"`
    IP          string    `json:"ip"`
    UserAgent   string    `json:"user_agent"`
    Timestamp   time.Time `json:"timestamp"`
    Description string    `json:"description"`
    Data        map[string]interface{} `json:"data,omitempty"`
}

func recordSecurityEvent(eventType, severity, description string, r *http.Request, data map[string]interface{}) {
    event := SecurityEvent{
        Type:        eventType,
        Severity:    severity,
        IP:          r.RemoteAddr,
        UserAgent:   r.UserAgent(),
        Timestamp:   time.Now(),
        Description: description,
        Data:        data,
    }

    // Incrémenter les métriques Prometheus
    securityEvents.WithLabelValues(eventType, severity).Inc()

    // Logger l'événement
    eventJSON, _ := json.Marshal(event)
    secureLogger.WithField("security_event", string(eventJSON)).Warn("Security event recorded")

    // En production, vous pourriez aussi :
    // - Envoyer à un SIEM
    // - Déclencher des alertes
    // - Bloquer automatiquement l'IP si nécessaire
}

// Middleware de détection d'intrusion basique
func intrusionDetection(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Détecter les tentatives d'injection SQL
        if detectSQLInjection(r) {
            recordSecurityEvent("sql_injection_attempt", "high", "SQL injection detected in request", r, map[string]interface{}{
                "url": r.URL.String(),
                "method": r.Method,
            })
        }

        // Détecter les tentatives XSS
        if detectXSS(r) {
            recordSecurityEvent("xss_attempt", "medium", "XSS attempt detected", r, nil)
        }

        // Détecter les User-Agents suspects
        if detectSuspiciousUserAgent(r) {
            recordSecurityEvent("suspicious_user_agent", "low", "Suspicious user agent detected", r, map[string]interface{}{
                "user_agent": r.UserAgent(),
            })
        }

        next(w, r)
    }
}

func detectSQLInjection(r *http.Request) bool {
    patterns := []string{
        "' OR '1'='1",
        "'; DROP TABLE",
        "UNION SELECT",
        "' OR 1=1--",
    }

    query := r.URL.RawQuery
    for _, pattern := range patterns {
        if strings.Contains(strings.ToUpper(query), strings.ToUpper(pattern)) {
            return true
        }
    }

    return false
}

func detectXSS(r *http.Request) bool {
    patterns := []string{
        "<script",
        "javascript:",
        "onerror=",
        "onload=",
    }

    query := r.URL.RawQuery
    for _, pattern := range patterns {
        if strings.Contains(strings.ToLower(query), strings.ToLower(pattern)) {
            return true
        }
    }

    return false
}

func detectSuspiciousUserAgent(r *http.Request) bool {
    userAgent := strings.ToLower(r.UserAgent())
    suspiciousPatterns := []string{
        "sqlmap",
        "nikto",
        "nmap",
        "masscan",
        "burp",
        "bot",
        "crawler",
    }

    for _, pattern := range suspiciousPatterns {
        if strings.Contains(userAgent, pattern) {
            return true
        }
    }

    return false
}
```


## 15. Checklist de sécurité

### Checklist avant déploiement

```markdown
# 🔒 Checklist de Sécurité Go
## ✅ Authentification et Autorisation
- [ ] Hachage sécurisé des mots de passe (bcrypt, coût ≥ 12)
- [ ] Validation forte des mots de passe
- [ ] Tokens JWT avec expiration
- [ ] Validation des tokens côté serveur
- [ ] Gestion des sessions sécurisée
- [ ] Logout approprié (invalidation des tokens)
- [ ] **Contrôle d'accès basé sur les rôles (RBAC)**
- [ ] **Protection contre les attaques par force brute**
- [ ] **Authentification multi-facteurs (MFA) si applicable**
- [ ] **Verrouillage de compte après échecs répétés**

## ✅ Validation des Données
- [ ] Validation de tous les inputs utilisateur
- [ ] Sanitisation HTML pour prévenir XSS
- [ ] Protection contre l'injection SQL
- [ ] Validation des types de fichiers uploadés
- [ ] Limitation de taille des requêtes
- [ ] **Validation des formats de données (JSON, XML)**
- [ ] **Protection contre l'injection de commandes**
- [ ] **Validation des URLs et redirections**
- [ ] **Filtrage des caractères spéciaux**

## ✅ Protection CSRF/XSS
- [ ] Tokens CSRF pour les formulaires
- [ ] En-têtes CSP (Content Security Policy)
- [ ] Échappement HTML automatique
- [ ] Validation des origins pour les requêtes AJAX
- [ ] **Protection contre le clickjacking**
- [ ] **Validation des en-têtes Referer**
- [ ] **SameSite cookies configurés**

## ✅ En-têtes de Sécurité
- [ ] Strict-Transport-Security (HTTPS)
- [ ] X-Frame-Options (clickjacking)
- [ ] X-Content-Type-Options (MIME sniffing)
- [ ] X-XSS-Protection
- [ ] Referrer-Policy
- [ ] **Permissions-Policy configuré**
- [ ] **Cache-Control pour données sensibles**
- [ ] **Suppression des en-têtes révélant la stack technique**

## ✅ Gestion des Secrets
- [ ] Variables d'environnement pour les secrets
- [ ] Pas de secrets dans le code source
- [ ] Chiffrement des données sensibles
- [ ] Rotation régulière des clés
- [ ] **Gestionnaire de secrets (Vault, AWS Secrets Manager)**
- [ ] **Chiffrement des secrets au repos**
- [ ] **Accès aux secrets basé sur les permissions**
- [ ] **Audit des accès aux secrets**

## ✅ Logging et Monitoring
- [ ] Logs sécurisés (pas de données sensibles)
- [ ] Monitoring des tentatives de connexion
- [ ] Alertes sur les événements suspects
- [ ] Conservation appropriée des logs
- [ ] **Détection d'intrusion automatisée**
- [ ] **Corrélation des événements de sécurité**
- [ ] **Backup sécurisé des logs**
- [ ] **Compliance avec les réglementations (RGPD, etc.)**

## ✅ Rate Limiting
- [ ] Limitation des requêtes par IP
- [ ] Protection contre le brute force
- [ ] Throttling des APIs sensibles
- [ ] **Rate limiting adaptatif selon le comportement**
- [ ] **Protection contre les attaques DDoS**
- [ ] **Limitation par utilisateur authentifié**

## ✅ Configuration Docker
- [ ] Utilisateur non-root
- [ ] Image minimale (scratch/distroless)
- [ ] Pas de secrets dans les layers Docker
- [ ] Scan de vulnérabilités des images
- [ ] **Read-only filesystem**
- [ ] **Capabilities Linux minimales**
- [ ] **No-new-privileges flag**
- [ ] **Image signée et vérifiée**

## ✅ HTTPS et TLS
- [ ] HTTPS obligatoire en production
- [ ] Certificats valides et à jour
- [ ] Configuration TLS sécurisée
- [ ] Redirection HTTP vers HTTPS
- [ ] **TLS 1.2+ minimum**
- [ ] **Ciphers suites sécurisées uniquement**
- [ ] **HSTS avec preload**
- [ ] **Certificate pinning si applicable**

## ✅ **Base de Données**
- [ ] **Connexions chiffrées (SSL/TLS)**
- [ ] **Principe du moindre privilège pour les comptes DB**
- [ ] **Requêtes préparées pour éviter l'injection SQL**
- [ ] **Backup chiffrés**
- [ ] **Audit des accès à la base de données**
- [ ] **Masquage des données sensibles en dev/test**

## ✅ **Réseau et Infrastructure**
- [ ] **Firewall configuré (ports minimum nécessaires)**
- [ ] **VPN ou bastion host pour l'accès admin**
- [ ] **Segmentation réseau appropriée**
- [ ] **Monitoring du trafic réseau**
- [ ] **Protection contre les scan de ports**
- [ ] **Géolocalisation des connexions suspectes**

## ✅ **Tests et Analyse**
- [ ] **Tests d'intrusion réguliers**
- [ ] **Analyse statique automatisée (gosec)**
- [ ] **Scan de vulnérabilités (govulncheck)**
- [ ] **Tests de sécurité dans le CI/CD**
- [ ] **Code review axé sécurité**
- [ ] **Tests de charge pour DoS**

## ✅ **Conformité et Gouvernance**
- [ ] **Politique de sécurité documentée**
- [ ] **Formation sécurité de l'équipe**
- [ ] **Plan de réponse aux incidents**
- [ ] **Conformité réglementaire (RGPD, SOX, etc.)**
- [ ] **Audit de sécurité périodique**
- [ ] **Documentation des vulnérabilités connues**

## ✅ **Sauvegarde et Récupération**
- [ ] **Plan de continuité d'activité**
- [ ] **Sauvegrades chiffrées et testées**
- [ ] **Procédure de récupération documentée**
- [ ] **Tests de restauration réguliers**
- [ ] **RTO/RPO définis et validés**

## ✅ **Développement Sécurisé**
- [ ] **Dépendances à jour et sans vulnérabilités**
- [ ] **Gestion des versions des dépendances**
- [ ] **Supply chain security**
- [ ] **Code signing pour les releases**
- [ ] **Environnements de dev isolés**
```

## Exercices pratiques

### Exercice 1 : Authentification JWT
Implémentez un système d'authentification complet avec :
1. Inscription sécurisée avec validation
2. Connexion avec génération de JWT
3. Middleware de protection des routes
4. Endpoint de profil protégé

### Exercice 2 : Protection XSS et CSRF
Créez un formulaire sécurisé qui :
1. Génère et valide des tokens CSRF
2. Échappe automatiquement les données HTML
3. Valide tous les inputs utilisateur
4. Affiche des messages d'erreur sécurisés

### Exercice 3 : Rate Limiting
Implémentez un système de limitation qui :
1. Limite à 100 requêtes par minute par IP
2. Retourne des en-têtes informatifs
3. Nettoie automatiquement les anciennes données
4. Applique des limites différentes par endpoint

### Exercice 4 : Audit de sécurité
Analysez une application Go existante et :
1. Identifiez les vulnérabilités potentielles
2. Implémentez les corrections nécessaires
3. Ajoutez des tests de sécurité
4. Créez un rapport d'audit

## Points clés à retenir

1. **Sécurité par défaut** : Configurez votre application pour être sécurisée dès le départ
2. **Validation rigoureuse** : Ne jamais faire confiance aux données utilisateur
3. **Principe du moindre privilège** : Accordez uniquement les permissions nécessaires
4. **Défense en profondeur** : Multipliez les couches de sécurité
5. **Tests réguliers** : Automatisez les tests de sécurité dans votre CI/CD

La sécurité est un processus continu, pas une destination.

⏭️
