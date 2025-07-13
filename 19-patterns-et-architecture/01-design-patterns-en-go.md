🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19-1 : Design patterns en Go

## Introduction aux design patterns

Les design patterns (ou patterns de conception) sont des solutions éprouvées à des problèmes récurrents en programmation. Ils représentent des bonnes pratiques qui ont été formalisées pour être réutilisées.

**Important** : En Go, nous n'utilisons pas tous les patterns classiques. Le langage privilégie la simplicité, donc certains patterns complexes ne sont pas nécessaires ou même contre-productifs.

## Principe général en Go

Go suit cette philosophie : **"Accept interfaces, return structs"**

```go
// ✅ Bon : accepte une interface, retourne une struct
func ProcessData(reader io.Reader) *Result {
    // ...
}

// ❌ Éviter : accepte une struct, retourne une interface
func ProcessData(file *os.File) io.Reader {
    // ...
}
```

## 1. Strategy Pattern (Stratégie)

Le pattern Strategy permet de choisir un algorithme à l'exécution.

### Problème résolu
Vous avez plusieurs façons de faire la même chose et vous voulez pouvoir changer de méthode facilement.

### Exemple concret : Calculatrice de prix

```go
// Définition de l'interface
type PriceCalculator interface {
    Calculate(price float64) float64
}

// Stratégie 1 : Remise fixe
type FixedDiscount struct {
    Amount float64
}

func (f FixedDiscount) Calculate(price float64) float64 {
    return price - f.Amount
}

// Stratégie 2 : Remise en pourcentage
type PercentageDiscount struct {
    Percentage float64
}

func (p PercentageDiscount) Calculate(price float64) float64 {
    return price * (1 - p.Percentage/100)
}

// Utilisation
type Product struct {
    Name       string
    BasePrice  float64
    Calculator PriceCalculator
}

func (p Product) FinalPrice() float64 {
    return p.Calculator.Calculate(p.BasePrice)
}

// Exemple d'utilisation
func main() {
    // Produit avec remise fixe
    product1 := Product{
        Name:       "Laptop",
        BasePrice:  1000,
        Calculator: FixedDiscount{Amount: 100},
    }

    // Produit avec remise en pourcentage
    product2 := Product{
        Name:       "Phone",
        BasePrice:  500,
        Calculator: PercentageDiscount{Percentage: 10},
    }

    fmt.Printf("%s: %.2f€\n", product1.Name, product1.FinalPrice()) // Laptop: 900.00€
    fmt.Printf("%s: %.2f€\n", product2.Name, product2.FinalPrice()) // Phone: 450.00€
}
```

## 2. Factory Pattern (Fabrique)

Le Factory pattern crée des objets sans spécifier leur classe exacte.

### Problème résolu
Vous voulez créer différents types d'objets selon le contexte, sans que le code appelant connaisse les détails.

### Exemple concret : Création de loggers

```go
// Interface commune
type Logger interface {
    Log(message string)
}

// Implémentation pour fichier
type FileLogger struct {
    filename string
}

func (f FileLogger) Log(message string) {
    // Écrire dans un fichier
    fmt.Printf("[FILE] %s: %s\n", f.filename, message)
}

// Implémentation pour console
type ConsoleLogger struct{}

func (c ConsoleLogger) Log(message string) {
    fmt.Printf("[CONSOLE] %s\n", message)
}

// Factory function
func NewLogger(logType string) Logger {
    switch logType {
    case "file":
        return FileLogger{filename: "app.log"}
    case "console":
        return ConsoleLogger{}
    default:
        return ConsoleLogger{} // valeur par défaut
    }
}

// Utilisation
func main() {
    // Le code appelant ne sait pas quel type de logger il utilise
    logger := NewLogger("file")
    logger.Log("Application started")

    logger2 := NewLogger("console")
    logger2.Log("Debug message")
}
```

## 3. Observer Pattern (Observateur)

Le pattern Observer permet à un objet de notifier plusieurs autres objets des changements.

### Problème résolu
Vous voulez que plusieurs parties de votre code soient informées quand quelque chose se passe.

### Exemple concret : Système de notifications

```go
// Interface pour les observateurs
type Observer interface {
    Notify(message string)
}

// Observateur par email
type EmailNotifier struct {
    Email string
}

func (e EmailNotifier) Notify(message string) {
    fmt.Printf("Email to %s: %s\n", e.Email, message)
}

// Observateur par SMS
type SMSNotifier struct {
    Phone string
}

func (s SMSNotifier) Notify(message string) {
    fmt.Printf("SMS to %s: %s\n", s.Phone, message)
}

// Sujet observé
type OrderSystem struct {
    observers []Observer
}

func (o *OrderSystem) AddObserver(observer Observer) {
    o.observers = append(o.observers, observer)
}

func (o *OrderSystem) NotifyAll(message string) {
    for _, observer := range o.observers {
        observer.Notify(message)
    }
}

func (o *OrderSystem) CreateOrder(productName string) {
    // Logique de création de commande
    fmt.Printf("Order created for: %s\n", productName)

    // Notifier tous les observateurs
    o.NotifyAll(fmt.Sprintf("New order: %s", productName))
}

// Utilisation
func main() {
    orderSystem := &OrderSystem{}

    // Ajouter des observateurs
    orderSystem.AddObserver(EmailNotifier{Email: "admin@example.com"})
    orderSystem.AddObserver(SMSNotifier{Phone: "+33123456789"})

    // Créer une commande
    orderSystem.CreateOrder("Laptop")
    // Sortie:
    // Order created for: Laptop
    // Email to admin@example.com: New order: Laptop
    // SMS to +33123456789: New order: Laptop
}
```

## 4. Builder Pattern (Constructeur)

Le Builder pattern construit des objets complexes étape par étape.

### Problème résolu
Vous avez des objets avec beaucoup de paramètres optionnels et vous voulez une API claire pour les créer.

### Exemple concret : Configuration d'un serveur HTTP

```go
// Structure à construire
type ServerConfig struct {
    Host     string
    Port     int
    Timeout  time.Duration
    MaxConns int
    UseHTTPS bool
}

// Builder
type ServerBuilder struct {
    config ServerConfig
}

// Méthodes de construction (fluent interface)
func (b *ServerBuilder) Host(host string) *ServerBuilder {
    b.config.Host = host
    return b
}

func (b *ServerBuilder) Port(port int) *ServerBuilder {
    b.config.Port = port
    return b
}

func (b *ServerBuilder) Timeout(timeout time.Duration) *ServerBuilder {
    b.config.Timeout = timeout
    return b
}

func (b *ServerBuilder) MaxConnections(max int) *ServerBuilder {
    b.config.MaxConns = max
    return b
}

func (b *ServerBuilder) EnableHTTPS() *ServerBuilder {
    b.config.UseHTTPS = true
    return b
}

func (b *ServerBuilder) Build() ServerConfig {
    // Valeurs par défaut
    if b.config.Host == "" {
        b.config.Host = "localhost"
    }
    if b.config.Port == 0 {
        b.config.Port = 8080
    }
    if b.config.Timeout == 0 {
        b.config.Timeout = 30 * time.Second
    }
    if b.config.MaxConns == 0 {
        b.config.MaxConns = 100
    }

    return b.config
}

// Factory function pour créer un builder
func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{}
}

// Utilisation
func main() {
    // Configuration simple
    config1 := NewServerBuilder().
        Host("example.com").
        Port(9090).
        Build()

    fmt.Printf("Server 1: %+v\n", config1)

    // Configuration complète
    config2 := NewServerBuilder().
        Host("api.example.com").
        Port(443).
        Timeout(60 * time.Second).
        MaxConnections(500).
        EnableHTTPS().
        Build()

    fmt.Printf("Server 2: %+v\n", config2)
}
```

## 5. Singleton Pattern (Singleton)

Le Singleton assure qu'une classe n'a qu'une seule instance.

### Problème résolu
Vous voulez être sûr qu'il n'y a qu'une seule instance d'un objet dans votre application.

### Exemple concret : Configuration globale

```go
package config

import (
    "sync"
)

// Structure de configuration
type Config struct {
    DatabaseURL string
    APIKey      string
    Debug       bool
}

// Instance unique
var instance *Config
var once sync.Once

// Fonction pour obtenir l'instance
func GetConfig() *Config {
    once.Do(func() {
        instance = &Config{
            DatabaseURL: "localhost:5432",
            APIKey:      "default-key",
            Debug:       false,
        }
    })
    return instance
}

// Utilisation
func main() {
    // Même instance partout
    config1 := GetConfig()
    config2 := GetConfig()

    fmt.Printf("Same instance: %t\n", config1 == config2) // true

    // Modification partagée
    config1.Debug = true
    fmt.Printf("Config2 debug: %t\n", config2.Debug) // true
}
```

## 6. Decorator Pattern (Décorateur)

Le Decorator ajoute des fonctionnalités à un objet sans modifier sa structure.

### Problème résolu
Vous voulez ajouter des comportements à un objet de manière flexible.

### Exemple concret : Middleware HTTP

```go
// Type de handler HTTP
type HandlerFunc func(http.ResponseWriter, *http.Request)

// Décorateur pour logging
func LoggingDecorator(handler HandlerFunc) HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        fmt.Printf("Request: %s %s\n", r.Method, r.URL.Path)

        handler(w, r)

        fmt.Printf("Completed in: %v\n", time.Since(start))
    }
}

// Décorateur pour authentification
func AuthDecorator(handler HandlerFunc) HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token != "Bearer valid-token" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        handler(w, r)
    }
}

// Handler de base
func HelloHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello, World!"))
}

// Utilisation
func main() {
    // Composer les décorateurs
    decoratedHandler := LoggingDecorator(
        AuthDecorator(HelloHandler),
    )

    http.HandleFunc("/hello", decoratedHandler)
    http.ListenAndServe(":8080", nil)
}
```

## 7. Template Method Pattern (Méthode template)

Le Template Method définit le squelette d'un algorithme, laissant les détails aux sous-classes.

### Problème résolu
Vous avez plusieurs algorithmes similaires qui diffèrent seulement dans certaines étapes.

### Exemple concret : Processus de traitement de données

```go
// Interface définissant les étapes variables
type DataProcessor interface {
    Validate(data []byte) error
    Transform(data []byte) []byte
    Save(data []byte) error
}

// Implémentation pour JSON
type JSONProcessor struct {
    OutputPath string
}

func (j JSONProcessor) Validate(data []byte) error {
    // Validation JSON
    var temp interface{}
    return json.Unmarshal(data, &temp)
}

func (j JSONProcessor) Transform(data []byte) []byte {
    // Transformation JSON (exemple : indentation)
    var temp interface{}
    json.Unmarshal(data, &temp)
    result, _ := json.MarshalIndent(temp, "", "  ")
    return result
}

func (j JSONProcessor) Save(data []byte) error {
    // Sauvegarde dans un fichier
    return os.WriteFile(j.OutputPath, data, 0644)
}

// Implémentation pour CSV
type CSVProcessor struct {
    OutputPath string
}

func (c CSVProcessor) Validate(data []byte) error {
    // Validation CSV basique
    lines := strings.Split(string(data), "\n")
    if len(lines) < 2 {
        return errors.New("CSV must have at least 2 lines")
    }
    return nil
}

func (c CSVProcessor) Transform(data []byte) []byte {
    // Transformation CSV (exemple : tri)
    lines := strings.Split(string(data), "\n")
    sort.Strings(lines[1:]) // Trier sans l'en-tête
    return []byte(strings.Join(lines, "\n"))
}

func (c CSVProcessor) Save(data []byte) error {
    return os.WriteFile(c.OutputPath, data, 0644)
}

// Fonction template (algorithme principal)
func ProcessData(processor DataProcessor, inputData []byte) error {
    // Étape 1 : Validation
    if err := processor.Validate(inputData); err != nil {
        return fmt.Errorf("validation failed: %w", err)
    }

    // Étape 2 : Transformation
    transformedData := processor.Transform(inputData)

    // Étape 3 : Sauvegarde
    if err := processor.Save(transformedData); err != nil {
        return fmt.Errorf("save failed: %w", err)
    }

    return nil
}

// Utilisation
func main() {
    jsonData := []byte(`{"name": "John", "age": 30}`)
    csvData := []byte("name,age\nJohn,30\nJane,25")

    // Traiter JSON
    jsonProcessor := JSONProcessor{OutputPath: "output.json"}
    ProcessData(jsonProcessor, jsonData)

    // Traiter CSV
    csvProcessor := CSVProcessor{OutputPath: "output.csv"}
    ProcessData(csvProcessor, csvData)
}
```

## Patterns spécifiques à Go

### 8. Options Pattern

Ce pattern est très populaire en Go pour gérer les paramètres optionnels.

```go
// Configuration d'un client HTTP
type HTTPClient struct {
    timeout time.Duration
    retries int
    headers map[string]string
}

// Type d'option
type ClientOption func(*HTTPClient)

// Options disponibles
func WithTimeout(timeout time.Duration) ClientOption {
    return func(c *HTTPClient) {
        c.timeout = timeout
    }
}

func WithRetries(retries int) ClientOption {
    return func(c *HTTPClient) {
        c.retries = retries
    }
}

func WithHeader(key, value string) ClientOption {
    return func(c *HTTPClient) {
        if c.headers == nil {
            c.headers = make(map[string]string)
        }
        c.headers[key] = value
    }
}

// Constructeur avec options
func NewHTTPClient(options ...ClientOption) *HTTPClient {
    client := &HTTPClient{
        timeout: 30 * time.Second, // valeur par défaut
        retries: 3,                // valeur par défaut
        headers: make(map[string]string),
    }

    for _, option := range options {
        option(client)
    }

    return client
}

// Utilisation
func main() {
    // Client avec configuration par défaut
    client1 := NewHTTPClient()

    // Client avec options personnalisées
    client2 := NewHTTPClient(
        WithTimeout(60 * time.Second),
        WithRetries(5),
        WithHeader("User-Agent", "MyApp/1.0"),
        WithHeader("Accept", "application/json"),
    )

    fmt.Printf("Client 1: %+v\n", client1)
    fmt.Printf("Client 2: %+v\n", client2)
}
```

## Bonnes pratiques

### 1. Gardez les interfaces petites
```go
// ✅ Bon : interface spécifique
type Writer interface {
    Write([]byte) (int, error)
}

// ❌ Éviter : interface trop large
type FileManager interface {
    Open(string) error
    Close() error
    Read([]byte) (int, error)
    Write([]byte) (int, error)
    Seek(int64, int) (int64, error)
    // ... trop de méthodes
}
```

### 2. Utilisez la composition
```go
// ✅ Bon : composition
type ReadWriter struct {
    io.Reader
    io.Writer
}

// ❌ Éviter en Go : héritage (n'existe pas)
```

### 3. Retournez des erreurs explicites
```go
// ✅ Bon
func CreateUser(name string) (*User, error) {
    if name == "" {
        return nil, errors.New("name cannot be empty")
    }
    return &User{Name: name}, nil
}

// ❌ Éviter : panic pour les erreurs prévues
func CreateUser(name string) *User {
    if name == "" {
        panic("name cannot be empty")
    }
    return &User{Name: name}
}
```

## Résumé

Les design patterns en Go sont adaptés à la philosophie du langage :
- **Simplicité** : Pas de sur-ingénierie
- **Interfaces** : Petites et spécifiques
- **Composition** : Plutôt que l'héritage
- **Pragmatisme** : Utilisez seulement ce qui est nécessaire

Les patterns les plus utiles en Go sont :
- **Strategy** : Pour les algorithmes interchangeables
- **Factory** : Pour la création d'objets
- **Observer** : Pour les notifications
- **Builder** : Pour les objets complexes
- **Options** : Pour les paramètres optionnels (spécifique à Go)

N'oubliez pas : un bon code Go est simple, lisible et idiomatique. Les patterns ne doivent pas compliquer votre code, mais le rendre plus maintenable.

⏭️
