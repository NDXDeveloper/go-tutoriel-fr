🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19-3 : Dependency injection

## Qu'est-ce que la Dependency Injection ?

La Dependency Injection (DI) est une technique qui consiste à **fournir les dépendances à un objet plutôt que de les créer à l'intérieur de l'objet**.

### Analogie simple
Imaginez que vous voulez faire du café :

**❌ Sans DI** : Vous construisez la cafetière, achetez les grains, moulez le café, etc.
```go
type CoffeeMaker struct{}

func (cm *CoffeeMaker) MakeCoffee() {
    grinder := NewGrinder()     // Je crée moi-même
    beans := NewBeans()         // Je crée moi-même
    water := NewWater()         // Je crée moi-même
    // ... faire le café
}
```

**✅ Avec DI** : Quelqu'un vous donne une cafetière prête, du café moulu, etc.
```go
type CoffeeMaker struct {
    grinder Grinder
    beans   Beans
    water   Water
}

func NewCoffeeMaker(grinder Grinder, beans Beans, water Water) *CoffeeMaker {
    return &CoffeeMaker{
        grinder: grinder,
        beans:   beans,
        water:   water,
    }
}
```

## Pourquoi utiliser la Dependency Injection ?

### 1. **Facilite les tests**
Vous pouvez remplacer les vraies dépendances par des mocks.

### 2. **Rend le code flexible**
Vous pouvez changer d'implémentation sans modifier le code.

### 3. **Améliore la lisibilité**
Les dépendances sont explicites et visibles.

### 4. **Évite les couplages forts**
Les objets ne dépendent pas d'implémentations concrètes.

## Les 3 types de Dependency Injection

### 1. Constructor Injection (Recommandé en Go)

Les dépendances sont passées lors de la création de l'objet.

```go
// Interface
type Database interface {
    Save(data string) error
    Load(id string) (string, error)
}

// Service qui dépend de Database
type UserService struct {
    db Database
}

// ✅ Constructor Injection
func NewUserService(db Database) *UserService {
    return &UserService{
        db: db,
    }
}

func (us *UserService) CreateUser(name string) error {
    return us.db.Save(name)
}
```

### 2. Setter Injection (Moins courant en Go)

Les dépendances sont injectées via des méthodes.

```go
type UserService struct {
    db Database
}

// Setter Injection
func (us *UserService) SetDatabase(db Database) {
    us.db = db
}
```

### 3. Interface Injection (Rare en Go)

L'objet implémente une interface pour recevoir ses dépendances.

## Exemple pratique : Système de notifications

### Étape 1 : Définir les interfaces

```go
// Interfaces pour les dépendances
type EmailSender interface {
    SendEmail(to, subject, body string) error
}

type Logger interface {
    Log(message string)
}

type UserRepository interface {
    GetUserEmail(userID int) (string, error)
}
```

### Étape 2 : Créer le service principal

```go
// Service de notification avec ses dépendances
type NotificationService struct {
    emailSender EmailSender
    logger      Logger
    userRepo    UserRepository
}

// Constructor avec injection des dépendances
func NewNotificationService(
    emailSender EmailSender,
    logger Logger,
    userRepo UserRepository,
) *NotificationService {
    return &NotificationService{
        emailSender: emailSender,
        logger:      logger,
        userRepo:    userRepo,
    }
}

// Méthode métier
func (ns *NotificationService) SendWelcomeEmail(userID int) error {
    // Utiliser le repository injecté
    email, err := ns.userRepo.GetUserEmail(userID)
    if err != nil {
        ns.logger.Log(fmt.Sprintf("Erreur récupération email: %v", err))
        return fmt.Errorf("impossible de récupérer l'email: %w", err)
    }

    // Utiliser l'email sender injecté
    subject := "Bienvenue !"
    body := "Merci de vous être inscrit sur notre plateforme."

    err = ns.emailSender.SendEmail(email, subject, body)
    if err != nil {
        ns.logger.Log(fmt.Sprintf("Erreur envoi email: %v", err))
        return fmt.Errorf("impossible d'envoyer l'email: %w", err)
    }

    // Utiliser le logger injecté
    ns.logger.Log(fmt.Sprintf("Email de bienvenue envoyé à %s", email))
    return nil
}
```

### Étape 3 : Implémentations concrètes

#### Email Sender

```go
// Implémentation réelle
type SMTPEmailSender struct {
    host     string
    port     int
    username string
    password string
}

func NewSMTPEmailSender(host string, port int, username, password string) *SMTPEmailSender {
    return &SMTPEmailSender{
        host:     host,
        port:     port,
        username: username,
        password: password,
    }
}

func (s *SMTPEmailSender) SendEmail(to, subject, body string) error {
    // Logique d'envoi d'email via SMTP
    fmt.Printf("Envoi email SMTP à %s: %s\n", to, subject)
    return nil
}

// Implémentation pour les tests
type MockEmailSender struct {
    SentEmails []string
}

func (m *MockEmailSender) SendEmail(to, subject, body string) error {
    m.SentEmails = append(m.SentEmails, to)
    return nil
}
```

#### Logger

```go
// Implémentation console
type ConsoleLogger struct{}

func (cl *ConsoleLogger) Log(message string) {
    fmt.Printf("[LOG] %s\n", message)
}

// Implémentation fichier
type FileLogger struct {
    filename string
}

func NewFileLogger(filename string) *FileLogger {
    return &FileLogger{filename: filename}
}

func (fl *FileLogger) Log(message string) {
    // Écrire dans un fichier
    fmt.Printf("[FILE] %s: %s\n", fl.filename, message)
}

// Implémentation pour les tests
type MockLogger struct {
    Messages []string
}

func (ml *MockLogger) Log(message string) {
    ml.Messages = append(ml.Messages, message)
}
```

#### User Repository

```go
// Implémentation base de données
type DatabaseUserRepository struct {
    db *sql.DB
}

func NewDatabaseUserRepository(db *sql.DB) *DatabaseUserRepository {
    return &DatabaseUserRepository{db: db}
}

func (dur *DatabaseUserRepository) GetUserEmail(userID int) (string, error) {
    var email string
    query := "SELECT email FROM users WHERE id = $1"
    err := dur.db.QueryRow(query, userID).Scan(&email)
    if err != nil {
        return "", err
    }
    return email, nil
}

// Implémentation pour les tests
type MockUserRepository struct {
    Users map[int]string
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        Users: map[int]string{
            1: "john@example.com",
            2: "jane@example.com",
        },
    }
}

func (mur *MockUserRepository) GetUserEmail(userID int) (string, error) {
    email, exists := mur.Users[userID]
    if !exists {
        return "", errors.New("utilisateur non trouvé")
    }
    return email, nil
}
```

### Étape 4 : Assemblage (Wire up)

```go
// Fonction main avec assemblage manuel
func main() {
    // Création des dépendances
    emailSender := NewSMTPEmailSender("smtp.example.com", 587, "user", "pass")
    logger := &ConsoleLogger{}

    // Connexion base de données
    db, err := sql.Open("postgres", "connection_string")
    if err != nil {
        log.Fatal(err)
    }
    userRepo := NewDatabaseUserRepository(db)

    // Injection des dépendances
    notificationService := NewNotificationService(emailSender, logger, userRepo)

    // Utilisation
    err = notificationService.SendWelcomeEmail(1)
    if err != nil {
        log.Printf("Erreur: %v", err)
    }
}
```

## Tests avec Dependency Injection

L'un des grands avantages de la DI est la facilité des tests.

```go
func TestNotificationService_SendWelcomeEmail(t *testing.T) {
    // Arrange : Créer des mocks
    mockEmailSender := &MockEmailSender{}
    mockLogger := &MockLogger{}
    mockUserRepo := NewMockUserRepository()

    // Injecter les mocks
    service := NewNotificationService(mockEmailSender, mockLogger, mockUserRepo)

    // Act : Exécuter la méthode
    err := service.SendWelcomeEmail(1)

    // Assert : Vérifier les résultats
    if err != nil {
        t.Errorf("Erreur inattendue: %v", err)
    }

    // Vérifier que l'email a été envoyé
    if len(mockEmailSender.SentEmails) != 1 {
        t.Errorf("Attendu 1 email envoyé, reçu %d", len(mockEmailSender.SentEmails))
    }

    if mockEmailSender.SentEmails[0] != "john@example.com" {
        t.Errorf("Email envoyé à la mauvaise adresse: %s", mockEmailSender.SentEmails[0])
    }

    // Vérifier que le log a été écrit
    if len(mockLogger.Messages) == 0 {
        t.Error("Aucun message de log trouvé")
    }
}

func TestNotificationService_SendWelcomeEmail_UserNotFound(t *testing.T) {
    // Arrange
    mockEmailSender := &MockEmailSender{}
    mockLogger := &MockLogger{}
    mockUserRepo := NewMockUserRepository()

    service := NewNotificationService(mockEmailSender, mockLogger, mockUserRepo)

    // Act : Utilisateur inexistant
    err := service.SendWelcomeEmail(999)

    // Assert : Doit retourner une erreur
    if err == nil {
        t.Error("Une erreur était attendue pour un utilisateur inexistant")
    }

    // Aucun email ne doit être envoyé
    if len(mockEmailSender.SentEmails) != 0 {
        t.Errorf("Aucun email ne devrait être envoyé, mais %d ont été envoyés",
                 len(mockEmailSender.SentEmails))
    }
}
```

## Container DI simple en Go

Pour des projets plus complexes, vous pouvez créer un container simple :

```go
// Container simple pour gérer les dépendances
type Container struct {
    services map[string]interface{}
}

func NewContainer() *Container {
    return &Container{
        services: make(map[string]interface{}),
    }
}

// Enregistrer un service
func (c *Container) Register(name string, service interface{}) {
    c.services[name] = service
}

// Récupérer un service
func (c *Container) Get(name string) interface{} {
    return c.services[name]
}

// Méthodes typées pour éviter les cast
func (c *Container) GetEmailSender() EmailSender {
    return c.services["emailSender"].(EmailSender)
}

func (c *Container) GetLogger() Logger {
    return c.services["logger"].(Logger)
}

func (c *Container) GetUserRepository() UserRepository {
    return c.services["userRepository"].(UserRepository)
}

// Configuration du container
func SetupContainer() *Container {
    container := NewContainer()

    // Enregistrer les services
    container.Register("emailSender", NewSMTPEmailSender("smtp.example.com", 587, "user", "pass"))
    container.Register("logger", &ConsoleLogger{})

    db, _ := sql.Open("postgres", "connection_string")
    container.Register("userRepository", NewDatabaseUserRepository(db))

    return container
}

// Utilisation
func main() {
    container := SetupContainer()

    notificationService := NewNotificationService(
        container.GetEmailSender(),
        container.GetLogger(),
        container.GetUserRepository(),
    )

    notificationService.SendWelcomeEmail(1)
}
```

## Wire : Générateur de code DI

Go a un outil officiel appelé Wire qui génère du code DI automatiquement.

### Installation

```bash
go install github.com/google/wire/cmd/wire@latest
```

### Configuration Wire

```go
// wire.go
//go:build wireinject
// +build wireinject

package main

import (
    "github.com/google/wire"
)

// Providers (fonctions qui créent des dépendances)
func ProvideEmailSender() EmailSender {
    return NewSMTPEmailSender("smtp.example.com", 587, "user", "pass")
}

func ProvideLogger() Logger {
    return &ConsoleLogger{}
}

func ProvideUserRepository() UserRepository {
    db, _ := sql.Open("postgres", "connection_string")
    return NewDatabaseUserRepository(db)
}

// Wire set
var ServiceSet = wire.NewSet(
    ProvideEmailSender,
    ProvideLogger,
    ProvideUserRepository,
    NewNotificationService,
)

// Fonction à générer
func InitializeNotificationService() *NotificationService {
    wire.Build(ServiceSet)
    return nil
}
```

### Génération du code

```bash
wire
```

Wire génère automatiquement :

```go
// wire_gen.go (généré automatiquement)
func InitializeNotificationService() *NotificationService {
    emailSender := ProvideEmailSender()
    logger := ProvideLogger()
    userRepository := ProvideUserRepository()
    notificationService := NewNotificationService(emailSender, logger, userRepository)
    return notificationService
}
```

## Pattern Options avec DI

Vous pouvez combiner DI avec le pattern Options :

```go
// Configuration avec options
type ServiceConfig struct {
    EmailSender EmailSender
    Logger      Logger
    UserRepo    UserRepository
}

type ServiceOption func(*ServiceConfig)

func WithEmailSender(emailSender EmailSender) ServiceOption {
    return func(config *ServiceConfig) {
        config.EmailSender = emailSender
    }
}

func WithLogger(logger Logger) ServiceOption {
    return func(config *ServiceConfig) {
        config.Logger = logger
    }
}

func WithUserRepository(userRepo UserRepository) ServiceOption {
    return func(config *ServiceConfig) {
        config.UserRepo = userRepo
    }
}

// Constructeur avec options
func NewNotificationServiceWithOptions(options ...ServiceOption) *NotificationService {
    config := &ServiceConfig{
        // Valeurs par défaut
        EmailSender: &MockEmailSender{},
        Logger:      &ConsoleLogger{},
        UserRepo:    NewMockUserRepository(),
    }

    for _, option := range options {
        option(config)
    }

    return &NotificationService{
        emailSender: config.EmailSender,
        logger:      config.Logger,
        userRepo:    config.UserRepo,
    }
}

// Utilisation
func main() {
    service := NewNotificationServiceWithOptions(
        WithEmailSender(NewSMTPEmailSender("smtp.example.com", 587, "user", "pass")),
        WithLogger(NewFileLogger("app.log")),
    )

    service.SendWelcomeEmail(1)
}
```

## Bonnes pratiques

### 1. **Utilisez des interfaces pour les dépendances**
```go
// ✅ Bon : Dépendre d'une interface
type UserService struct {
    repo UserRepository // interface
}

// ❌ Éviter : Dépendre d'une implémentation
type UserService struct {
    repo *DatabaseUserRepository // struct concrète
}
```

### 2. **Gardez les constructeurs simples**
```go
// ✅ Bon : Constructeur simple
func NewUserService(repo UserRepository, logger Logger) *UserService {
    return &UserService{
        repo:   repo,
        logger: logger,
    }
}

// ❌ Éviter : Logique complexe dans le constructeur
func NewUserService(repo UserRepository, logger Logger) *UserService {
    service := &UserService{repo: repo, logger: logger}
    service.initialize()  // ❌ Logique métier dans le constructeur
    service.loadConfig()  // ❌ Trop de responsabilités
    return service
}
```

### 3. **Validez les dépendances**
```go
func NewUserService(repo UserRepository, logger Logger) *UserService {
    if repo == nil {
        panic("repository cannot be nil")
    }
    if logger == nil {
        panic("logger cannot be nil")
    }

    return &UserService{
        repo:   repo,
        logger: logger,
    }
}
```

### 4. **Organisez l'assemblage dans main()**
```go
func main() {
    // Toute la configuration DI ici
    emailSender := NewSMTPEmailSender(...)
    logger := &ConsoleLogger{}
    userRepo := NewDatabaseUserRepository(...)

    notificationService := NewNotificationService(emailSender, logger, userRepo)
    userService := NewUserService(userRepo, logger)

    // Démarrer l'application
    startServer(notificationService, userService)
}
```

### 5. **Évitez les anti-patterns**

```go
// ❌ Service Locator (anti-pattern)
type UserService struct{}

func (us *UserService) CreateUser(name string) {
    repo := ServiceLocator.Get("userRepository") // ❌ Dépendance cachée
    repo.Save(name)
}

// ✅ Dependency Injection (bon pattern)
type UserService struct {
    repo UserRepository // ✅ Dépendance explicite
}

func (us *UserService) CreateUser(name string) {
    us.repo.Save(name)
}
```

## Quand utiliser la DI ?

### ✅ **Utilisez la DI quand** :
- Votre objet a des dépendances externes
- Vous voulez tester votre code facilement
- Vous voulez rendre votre code flexible
- Vous avez plusieurs implémentations possibles

### ❌ **N'utilisez PAS la DI quand** :
- L'objet n'a pas de dépendances
- La dépendance est stable et ne changera jamais
- Cela rend le code plus complexe sans bénéfice

## Résumé

La Dependency Injection en Go :

**Avantages** :
- Code plus testable
- Flexibilité et évolutivité
- Dépendances explicites
- Découplage des composants

**Méthodes** :
- Constructor Injection (recommandé)
- Container DI simple
- Wire pour les gros projets
- Pattern Options pour la flexibilité

**Points clés** :
- Injectez des interfaces, pas des structs
- Assemblez tout dans main()
- Gardez les constructeurs simples
- Validez les dépendances

La DI peut sembler complexe au début, mais elle rend votre code Go beaucoup plus maintenable et testable !

⏭️
