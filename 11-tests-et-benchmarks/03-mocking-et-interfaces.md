🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11-3 : Mocking et interfaces

## Qu'est-ce que le mocking ?

Le **mocking** (simulation) est une technique qui permet de remplacer des parties de votre code par des "fausses" versions pendant les tests. Imaginez que votre code envoie des emails - pendant les tests, vous ne voulez pas vraiment envoyer d'emails ! Le mocking vous permet de simuler l'envoi sans qu'il se passe réellement quelque chose.

**Pourquoi utiliser le mocking ?**
- Tester sans dépendances externes (base de données, APIs, fichiers)
- Tests plus rapides et fiables
- Contrôler le comportement des dépendances
- Tester les cas d'erreur facilement

## Le problème : dépendances externes

### Exemple problématique (difficile à tester)

```go
// Service qui envoie des emails - difficile à tester !
type EmailService struct{}

func (e *EmailService) SendEmail(to, subject, body string) error {
    // Code qui envoie vraiment un email via SMTP
    fmt.Printf("Envoi email à %s: %s\n", to, subject)
    return nil // ou une erreur si l'envoi échoue
}

// Service utilisateur qui dépend d'EmailService
type UserService struct {
    emailService *EmailService
}

func (u *UserService) RegisterUser(name, email string) error {
    // Logique d'enregistrement
    fmt.Printf("Enregistrement de %s\n", name)

    // Envoi email de bienvenue
    return u.emailService.SendEmail(email, "Bienvenue !", "Merci de vous être inscrit")
}

// Comment tester ça sans envoyer de vrais emails ? 🤔
```

**Problèmes :**
- Les tests envoient de vrais emails
- Tests lents (connexion réseau)
- Tests fragiles (dépend du service email)
- Difficile de tester les erreurs

## La solution : interfaces et mocking

### Étape 1 : Créer une interface

```go
// Interface qui définit ce qu'on peut faire avec un service email
type EmailSender interface {
    SendEmail(to, subject, body string) error
}
```

### Étape 2 : Implémenter l'interface

```go
// Vraie implémentation pour la production
type RealEmailService struct{}

func (e *RealEmailService) SendEmail(to, subject, body string) error {
    // Code qui envoie vraiment un email
    fmt.Printf("Envoi RÉEL email à %s: %s\n", to, subject)
    return nil
}

// Fausse implémentation pour les tests
type MockEmailService struct {
    SentEmails []Email // Garde une trace des emails "envoyés"
    ShouldFail bool    // Pour simuler des erreurs
}

type Email struct {
    To      string
    Subject string
    Body    string
}

func (m *MockEmailService) SendEmail(to, subject, body string) error {
    if m.ShouldFail {
        return errors.New("erreur simulée d'envoi email")
    }

    // "Envoie" l'email en le stockant
    m.SentEmails = append(m.SentEmails, Email{
        To:      to,
        Subject: subject,
        Body:    body,
    })

    fmt.Printf("Email SIMULÉ envoyé à %s\n", to)
    return nil
}
```

### Étape 3 : Utiliser l'interface dans le service

```go
// UserService utilise maintenant l'interface
type UserService struct {
    emailSender EmailSender // Interface au lieu de struct concrète
}

func NewUserService(emailSender EmailSender) *UserService {
    return &UserService{
        emailSender: emailSender,
    }
}

func (u *UserService) RegisterUser(name, email string) error {
    // Logique d'enregistrement
    fmt.Printf("Enregistrement de %s\n", name)

    // Envoi email via l'interface
    return u.emailSender.SendEmail(email, "Bienvenue !", "Merci de vous être inscrit")
}
```

### Étape 4 : Tester avec le mock

```go
func TestUserService_RegisterUser_Success(t *testing.T) {
    // Créer un mock
    mockEmail := &MockEmailService{}

    // Créer le service avec le mock
    userService := NewUserService(mockEmail)

    // Tester l'enregistrement
    err := userService.RegisterUser("Alice", "alice@example.com")

    // Vérifications
    if err != nil {
        t.Errorf("RegisterUser() erreur inattendue: %v", err)
    }

    // Vérifier qu'un email a été "envoyé"
    if len(mockEmail.SentEmails) != 1 {
        t.Errorf("Attendu 1 email envoyé, reçu %d", len(mockEmail.SentEmails))
    }

    // Vérifier le contenu de l'email
    email := mockEmail.SentEmails[0]
    if email.To != "alice@example.com" {
        t.Errorf("Email To = %s; want alice@example.com", email.To)
    }
    if email.Subject != "Bienvenue !" {
        t.Errorf("Email Subject = %s; want Bienvenue !", email.Subject)
    }
}

func TestUserService_RegisterUser_EmailError(t *testing.T) {
    // Créer un mock qui va échouer
    mockEmail := &MockEmailService{ShouldFail: true}
    userService := NewUserService(mockEmail)

    // Tester l'enregistrement
    err := userService.RegisterUser("Bob", "bob@example.com")

    // Vérifier qu'on reçoit bien l'erreur
    if err == nil {
        t.Error("RegisterUser() devrait retourner une erreur quand l'email échoue")
    }
}
```

## Exemple complet : Service de base de données

### Interface et implémentations

```go
// Interface pour la base de données
type UserRepository interface {
    GetUser(id int) (User, error)
    SaveUser(user User) error
    DeleteUser(id int) error
}

// Modèle User
type User struct {
    ID    int
    Name  string
    Email string
}

// Vraie implémentation (production)
type SQLUserRepository struct {
    db *sql.DB
}

func (r *SQLUserRepository) GetUser(id int) (User, error) {
    // Code SQL réel
    var user User
    err := r.db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id).
        Scan(&user.ID, &user.Name, &user.Email)
    return user, err
}

func (r *SQLUserRepository) SaveUser(user User) error {
    // Code SQL pour sauvegarder
    _, err := r.db.Exec("INSERT INTO users (name, email) VALUES (?, ?)",
        user.Name, user.Email)
    return err
}

func (r *SQLUserRepository) DeleteUser(id int) error {
    // Code SQL pour supprimer
    _, err := r.db.Exec("DELETE FROM users WHERE id = ?", id)
    return err
}

// Mock pour les tests
type MockUserRepository struct {
    Users       map[int]User // Stockage en mémoire
    SaveError   error        // Pour simuler des erreurs de sauvegarde
    DeleteError error        // Pour simuler des erreurs de suppression
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        Users: make(map[int]User),
    }
}

func (m *MockUserRepository) GetUser(id int) (User, error) {
    user, exists := m.Users[id]
    if !exists {
        return User{}, errors.New("utilisateur non trouvé")
    }
    return user, nil
}

func (m *MockUserRepository) SaveUser(user User) error {
    if m.SaveError != nil {
        return m.SaveError
    }

    // Simuler auto-increment de l'ID
    if user.ID == 0 {
        user.ID = len(m.Users) + 1
    }

    m.Users[user.ID] = user
    return nil
}

func (m *MockUserRepository) DeleteUser(id int) error {
    if m.DeleteError != nil {
        return m.DeleteError
    }

    _, exists := m.Users[id]
    if !exists {
        return errors.New("utilisateur non trouvé")
    }

    delete(m.Users, id)
    return nil
}
```

### Service utilisant le repository

```go
// Service qui utilise le repository
type UserManager struct {
    repo UserRepository
}

func NewUserManager(repo UserRepository) *UserManager {
    return &UserManager{repo: repo}
}

func (um *UserManager) CreateUser(name, email string) (User, error) {
    // Validation
    if name == "" {
        return User{}, errors.New("nom requis")
    }
    if email == "" {
        return User{}, errors.New("email requis")
    }

    // Créer l'utilisateur
    user := User{Name: name, Email: email}

    // Sauvegarder via le repository
    err := um.repo.SaveUser(user)
    if err != nil {
        return User{}, fmt.Errorf("erreur de sauvegarde: %w", err)
    }

    return user, nil
}

func (um *UserManager) GetUserProfile(id int) (User, error) {
    return um.repo.GetUser(id)
}

func (um *UserManager) RemoveUser(id int) error {
    // Vérifier que l'utilisateur existe
    _, err := um.repo.GetUser(id)
    if err != nil {
        return fmt.Errorf("utilisateur non trouvé: %w", err)
    }

    // Supprimer l'utilisateur
    return um.repo.DeleteUser(id)
}
```

### Tests avec mocks

```go
func TestUserManager_CreateUser(t *testing.T) {
    tests := []struct {
        name      string
        userName  string
        userEmail string
        saveError error
        wantErr   bool
    }{
        {
            name:      "création réussie",
            userName:  "Alice",
            userEmail: "alice@example.com",
            saveError: nil,
            wantErr:   false,
        },
        {
            name:      "nom vide",
            userName:  "",
            userEmail: "test@example.com",
            saveError: nil,
            wantErr:   true,
        },
        {
            name:      "email vide",
            userName:  "Bob",
            userEmail: "",
            saveError: nil,
            wantErr:   true,
        },
        {
            name:      "erreur de base de données",
            userName:  "Charlie",
            userEmail: "charlie@example.com",
            saveError: errors.New("erreur DB"),
            wantErr:   true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Créer un mock repository
            mockRepo := NewMockUserRepository()
            mockRepo.SaveError = tt.saveError

            // Créer le service avec le mock
            userManager := NewUserManager(mockRepo)

            // Tester la création d'utilisateur
            user, err := userManager.CreateUser(tt.userName, tt.userEmail)

            // Vérifier les erreurs
            if tt.wantErr {
                if err == nil {
                    t.Error("CreateUser() devrait retourner une erreur")
                }
                return
            }

            if err != nil {
                t.Errorf("CreateUser() erreur inattendue: %v", err)
                return
            }

            // Vérifier que l'utilisateur a été créé
            if user.Name != tt.userName {
                t.Errorf("User.Name = %s; want %s", user.Name, tt.userName)
            }
            if user.Email != tt.userEmail {
                t.Errorf("User.Email = %s; want %s", user.Email, tt.userEmail)
            }
        })
    }
}

func TestUserManager_GetUserProfile(t *testing.T) {
    // Setup
    mockRepo := NewMockUserRepository()
    userManager := NewUserManager(mockRepo)

    // Ajouter un utilisateur dans le mock
    testUser := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    mockRepo.Users[1] = testUser

    // Test cas de succès
    t.Run("utilisateur existant", func(t *testing.T) {
        user, err := userManager.GetUserProfile(1)
        if err != nil {
            t.Errorf("GetUserProfile() erreur inattendue: %v", err)
        }
        if user != testUser {
            t.Errorf("GetUserProfile() = %+v; want %+v", user, testUser)
        }
    })

    // Test cas d'échec
    t.Run("utilisateur inexistant", func(t *testing.T) {
        _, err := userManager.GetUserProfile(999)
        if err == nil {
            t.Error("GetUserProfile() devrait retourner une erreur pour un utilisateur inexistant")
        }
    })
}

func TestUserManager_RemoveUser(t *testing.T) {
    mockRepo := NewMockUserRepository()
    userManager := NewUserManager(mockRepo)

    // Ajouter un utilisateur
    testUser := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
    mockRepo.Users[1] = testUser

    t.Run("suppression réussie", func(t *testing.T) {
        err := userManager.RemoveUser(1)
        if err != nil {
            t.Errorf("RemoveUser() erreur inattendue: %v", err)
        }

        // Vérifier que l'utilisateur a été supprimé du mock
        if _, exists := mockRepo.Users[1]; exists {
            t.Error("L'utilisateur devrait avoir été supprimé du repository")
        }
    })

    t.Run("utilisateur inexistant", func(t *testing.T) {
        err := userManager.RemoveUser(999)
        if err == nil {
            t.Error("RemoveUser() devrait retourner une erreur pour un utilisateur inexistant")
        }
    })

    t.Run("erreur de suppression", func(t *testing.T) {
        // Remettre l'utilisateur
        mockRepo.Users[1] = testUser
        // Configurer le mock pour échouer
        mockRepo.DeleteError = errors.New("erreur DB")

        err := userManager.RemoveUser(1)
        if err == nil {
            t.Error("RemoveUser() devrait retourner une erreur quand la DB échoue")
        }
    })
}
```

## Mocking avec des outils externes

### GoMock (génération automatique de mocks)

```go
// Installer GoMock
// go install github.com/golang/mock/mockgen@latest

//go:generate mockgen -source=user.go -destination=mocks/mock_user.go

// Interface à mocker
type UserService interface {
    GetUser(id int) (User, error)
    CreateUser(name, email string) (User, error)
}

// Test avec GoMock
func TestWithGoMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Créer un mock généré automatiquement
    mockService := mocks.NewMockUserService(ctrl)

    // Définir les attentes
    mockService.EXPECT().
        GetUser(1).
        Return(User{ID: 1, Name: "Alice"}, nil).
        Times(1)

    // Utiliser le mock
    user, err := mockService.GetUser(1)

    // Vérifications
    if err != nil {
        t.Errorf("Erreur inattendue: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("Nom = %s; want Alice", user.Name)
    }
}
```

### Testify/mock (framework de test)

```go
// go get github.com/stretchr/testify/mock

type MockUserService struct {
    mock.Mock
}

func (m *MockUserService) GetUser(id int) (User, error) {
    args := m.Called(id)
    return args.Get(0).(User), args.Error(1)
}

func TestWithTestify(t *testing.T) {
    mockService := new(MockUserService)

    // Définir les attentes
    mockService.On("GetUser", 1).Return(User{ID: 1, Name: "Alice"}, nil)

    // Utiliser le mock
    user, err := mockService.GetUser(1)

    // Vérifications
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)

    // Vérifier que toutes les attentes ont été satisfaites
    mockService.AssertExpectations(t)
}
```

## Bonnes pratiques du mocking

### 1. Interfaces minimales

```go
// ✅ Bon : interface spécifique
type EmailSender interface {
    SendEmail(to, subject, body string) error
}

// ❌ Mauvais : interface trop large
type EmailService interface {
    SendEmail(to, subject, body string) error
    ConfigureSMTP(host, port string)
    SetCredentials(user, pass string)
    GetStats() EmailStats
    // ... trop de méthodes
}
```

### 2. Injection de dépendances

```go
// ✅ Bon : injection via constructeur
func NewUserService(emailSender EmailSender) *UserService {
    return &UserService{emailSender: emailSender}
}

// ❌ Mauvais : dépendance codée en dur
func NewUserService() *UserService {
    return &UserService{
        emailSender: &RealEmailService{}, // Impossible à tester
    }
}
```

### 3. Mocks simples et focalisés

```go
// ✅ Bon : mock simple et clair
type MockEmailService struct {
    SentEmails []Email
    ShouldFail bool
}

// ❌ Mauvais : mock trop complexe
type ComplexMockEmailService struct {
    SentEmails []Email
    Config     EmailConfig
    Stats      EmailStats
    Logs       []LogEntry
    // ... trop de responsabilités
}
```

### 4. Vérification des interactions

```go
func TestEmailInteraction(t *testing.T) {
    mockEmail := &MockEmailService{}
    service := NewUserService(mockEmail)

    // Action
    service.RegisterUser("Alice", "alice@example.com")

    // Vérifications
    if len(mockEmail.SentEmails) != 1 {
        t.Error("Un email devrait avoir été envoyé")
    }

    email := mockEmail.SentEmails[0]
    if email.To != "alice@example.com" {
        t.Error("Email envoyé à la mauvaise adresse")
    }
}
```

## Quand utiliser le mocking

### ✅ Utiliser le mocking pour :
- Services externes (APIs, bases de données)
- Opérations lentes (network, fichiers)
- Dépendances difficiles à configurer
- Tester les cas d'erreur
- Isolation des tests unitaires

### ❌ Ne pas utiliser le mocking pour :
- Logique métier pure (fonctions sans dépendances)
- Structures de données simples
- Code que vous possédez et contrôlez complètement
- Tests d'intégration (où vous voulez tester les vraies interactions)

## Exercices pratiques

### Exercice 1 : Mock d'un service de fichiers
Créez une interface `FileStorage` et son mock pour tester ce service :

```go
type DocumentService struct {
    storage FileStorage
}

func (d *DocumentService) SaveDocument(name, content string) error {
    return d.storage.WriteFile(name, content)
}

func (d *DocumentService) GetDocument(name string) (string, error) {
    return d.storage.ReadFile(name)
}
```

### Exercice 2 : Mock d'un service de notification
Créez des mocks pour tester ce service qui envoie des notifications multiples :

```go
type NotificationService struct {
    emailSender EmailSender
    smsSender   SMSSender
    pushSender  PushSender
}

func (n *NotificationService) NotifyUser(userID int, message string) error {
    // Envoie par email, SMS et push
    // Testez différentes combinaisons d'échecs
}
```

## Récapitulatif

Le **mocking avec interfaces** permet de :
- **Isoler** le code à tester des dépendances externes
- **Contrôler** le comportement des dépendances
- **Tester** facilement les cas d'erreur
- **Accélérer** l'exécution des tests

**Points clés à retenir :**
- Définir des interfaces pour les dépendances
- Implémenter des mocks simples pour les tests
- Utiliser l'injection de dépendances
- Vérifier les interactions avec les mocks
- Garder les interfaces petites et focalisées
- Choisir le bon niveau d'abstraction

**Outils disponibles :**
- Mocks manuels (simples et efficaces)
- GoMock (génération automatique)
- Testify (framework complet)

Dans la section suivante, nous découvrirons les **benchmarks et le profiling** pour optimiser les performances de votre code Go !

⏭️

# Solutions des exercices pratiques - Mocking et interfaces

## Exercice 1 : Mock d'un service de fichiers

### Interface et implémentations

```go
package main

import (
    "errors"
    "fmt"
    "io/ioutil"
    "os"
)

// Interface pour le stockage de fichiers
type FileStorage interface {
    WriteFile(name, content string) error
    ReadFile(name string) (string, error)
}

// Service de documents utilisant FileStorage
type DocumentService struct {
    storage FileStorage
}

func NewDocumentService(storage FileStorage) *DocumentService {
    return &DocumentService{storage: storage}
}

func (d *DocumentService) SaveDocument(name, content string) error {
    if name == "" {
        return errors.New("nom de document requis")
    }
    if content == "" {
        return errors.New("contenu de document requis")
    }

    return d.storage.WriteFile(name, content)
}

func (d *DocumentService) GetDocument(name string) (string, error) {
    if name == "" {
        return "", errors.New("nom de document requis")
    }

    return d.storage.ReadFile(name)
}

// Implémentation réelle pour la production
type RealFileStorage struct {
    basePath string
}

func NewRealFileStorage(basePath string) *RealFileStorage {
    return &RealFileStorage{basePath: basePath}
}

func (r *RealFileStorage) WriteFile(name, content string) error {
    filepath := fmt.Sprintf("%s/%s", r.basePath, name)
    return ioutil.WriteFile(filepath, []byte(content), 0644)
}

func (r *RealFileStorage) ReadFile(name string) (string, error) {
    filepath := fmt.Sprintf("%s/%s", r.basePath, name)
    data, err := ioutil.ReadFile(filepath)
    if err != nil {
        return "", err
    }
    return string(data), nil
}

// Mock pour les tests
type MockFileStorage struct {
    Files       map[string]string // Stockage en mémoire
    WriteError  error            // Pour simuler des erreurs d'écriture
    ReadError   error            // Pour simuler des erreurs de lecture
    WriteCalls  []WriteCall      // Trace des appels WriteFile
    ReadCalls   []ReadCall       // Trace des appels ReadFile
}

type WriteCall struct {
    Name    string
    Content string
}

type ReadCall struct {
    Name string
}

func NewMockFileStorage() *MockFileStorage {
    return &MockFileStorage{
        Files:      make(map[string]string),
        WriteCalls: make([]WriteCall, 0),
        ReadCalls:  make([]ReadCall, 0),
    }
}

func (m *MockFileStorage) WriteFile(name, content string) error {
    // Enregistrer l'appel
    m.WriteCalls = append(m.WriteCalls, WriteCall{
        Name:    name,
        Content: content,
    })

    // Simuler une erreur si configuré
    if m.WriteError != nil {
        return m.WriteError
    }

    // Stocker le fichier en mémoire
    m.Files[name] = content
    return nil
}

func (m *MockFileStorage) ReadFile(name string) (string, error) {
    // Enregistrer l'appel
    m.ReadCalls = append(m.ReadCalls, ReadCall{Name: name})

    // Simuler une erreur si configuré
    if m.ReadError != nil {
        return "", m.ReadError
    }

    // Lire depuis le stockage en mémoire
    content, exists := m.Files[name]
    if !exists {
        return "", os.ErrNotExist
    }

    return content, nil
}

// Helper methods pour les tests
func (m *MockFileStorage) GetWriteCallCount() int {
    return len(m.WriteCalls)
}

func (m *MockFileStorage) GetReadCallCount() int {
    return len(m.ReadCalls)
}

func (m *MockFileStorage) WasFileWritten(name string) bool {
    for _, call := range m.WriteCalls {
        if call.Name == name {
            return true
        }
    }
    return false
}

func (m *MockFileStorage) GetWrittenContent(name string) string {
    for _, call := range m.WriteCalls {
        if call.Name == name {
            return call.Content
        }
    }
    return ""
}
```

### Tests complets

```go
package main

import (
    "os"
    "testing"
)

func TestDocumentService_SaveDocument(t *testing.T) {
    tests := []struct {
        name        string
        docName     string
        content     string
        writeError  error
        expectError bool
    }{
        {
            name:        "sauvegarde réussie",
            docName:     "test.txt",
            content:     "Contenu du document",
            writeError:  nil,
            expectError: false,
        },
        {
            name:        "nom vide",
            docName:     "",
            content:     "Contenu",
            writeError:  nil,
            expectError: true,
        },
        {
            name:        "contenu vide",
            docName:     "test.txt",
            content:     "",
            writeError:  nil,
            expectError: true,
        },
        {
            name:        "erreur de stockage",
            docName:     "test.txt",
            content:     "Contenu",
            writeError:  errors.New("erreur disque"),
            expectError: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockStorage := NewMockFileStorage()
            mockStorage.WriteError = tt.writeError
            docService := NewDocumentService(mockStorage)

            // Action
            err := docService.SaveDocument(tt.docName, tt.content)

            // Vérifications d'erreur
            if tt.expectError {
                if err == nil {
                    t.Error("SaveDocument() devrait retourner une erreur")
                }
                return
            }

            if err != nil {
                t.Errorf("SaveDocument() erreur inattendue: %v", err)
                return
            }

            // Vérifier que WriteFile a été appelé correctement
            if mockStorage.GetWriteCallCount() != 1 {
                t.Errorf("WriteFile devrait être appelé 1 fois, appelé %d fois",
                    mockStorage.GetWriteCallCount())
            }

            if !mockStorage.WasFileWritten(tt.docName) {
                t.Errorf("Fichier %s devrait avoir été écrit", tt.docName)
            }

            writtenContent := mockStorage.GetWrittenContent(tt.docName)
            if writtenContent != tt.content {
                t.Errorf("Contenu écrit = %q; want %q", writtenContent, tt.content)
            }

            // Vérifier que le fichier est dans le stockage
            storedContent, exists := mockStorage.Files[tt.docName]
            if !exists {
                t.Error("Fichier devrait être stocké dans le mock")
            }
            if storedContent != tt.content {
                t.Errorf("Contenu stocké = %q; want %q", storedContent, tt.content)
            }
        })
    }
}

func TestDocumentService_GetDocument(t *testing.T) {
    tests := []struct {
        name        string
        docName     string
        setupFile   bool
        fileContent string
        readError   error
        expectError bool
    }{
        {
            name:        "lecture réussie",
            docName:     "test.txt",
            setupFile:   true,
            fileContent: "Contenu du fichier",
            readError:   nil,
            expectError: false,
        },
        {
            name:        "nom vide",
            docName:     "",
            setupFile:   false,
            fileContent: "",
            readError:   nil,
            expectError: true,
        },
        {
            name:        "fichier inexistant",
            docName:     "inexistant.txt",
            setupFile:   false,
            fileContent: "",
            readError:   nil,
            expectError: true,
        },
        {
            name:        "erreur de lecture",
            docName:     "test.txt",
            setupFile:   true,
            fileContent: "Contenu",
            readError:   errors.New("erreur de lecture"),
            expectError: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockStorage := NewMockFileStorage()
            mockStorage.ReadError = tt.readError

            if tt.setupFile {
                mockStorage.Files[tt.docName] = tt.fileContent
            }

            docService := NewDocumentService(mockStorage)

            // Action
            content, err := docService.GetDocument(tt.docName)

            // Vérifications d'erreur
            if tt.expectError {
                if err == nil {
                    t.Error("GetDocument() devrait retourner une erreur")
                }
                return
            }

            if err != nil {
                t.Errorf("GetDocument() erreur inattendue: %v", err)
                return
            }

            // Vérifier le contenu retourné
            if content != tt.fileContent {
                t.Errorf("GetDocument() = %q; want %q", content, tt.fileContent)
            }

            // Vérifier que ReadFile a été appelé
            if mockStorage.GetReadCallCount() != 1 {
                t.Errorf("ReadFile devrait être appelé 1 fois, appelé %d fois",
                    mockStorage.GetReadCallCount())
            }
        })
    }
}

func TestDocumentService_Integration(t *testing.T) {
    // Test d'intégration : sauvegarder puis lire
    mockStorage := NewMockFileStorage()
    docService := NewDocumentService(mockStorage)

    // Sauvegarder un document
    err := docService.SaveDocument("integration.txt", "Contenu d'intégration")
    if err != nil {
        t.Fatalf("SaveDocument() erreur: %v", err)
    }

    // Lire le même document
    content, err := docService.GetDocument("integration.txt")
    if err != nil {
        t.Fatalf("GetDocument() erreur: %v", err)
    }

    if content != "Contenu d'intégration" {
        t.Errorf("Contenu lu = %q; want %q", content, "Contenu d'intégration")
    }

    // Vérifier les interactions
    if mockStorage.GetWriteCallCount() != 1 {
        t.Errorf("WriteFile appelé %d fois; want 1", mockStorage.GetWriteCallCount())
    }
    if mockStorage.GetReadCallCount() != 1 {
        t.Errorf("ReadFile appelé %d fois; want 1", mockStorage.GetReadCallCount())
    }
}
```

## Exercice 2 : Mock d'un service de notification

### Interfaces et implémentations

```go
package main

import (
    "errors"
    "fmt"
)

// Interfaces pour les différents types de notifications
type EmailSender interface {
    SendEmail(userID int, message string) error
}

type SMSSender interface {
    SendSMS(userID int, message string) error
}

type PushSender interface {
    SendPush(userID int, message string) error
}

// Service de notification principal
type NotificationService struct {
    emailSender EmailSender
    smsSender   SMSSender
    pushSender  PushSender
}

func NewNotificationService(email EmailSender, sms SMSSender, push PushSender) *NotificationService {
    return &NotificationService{
        emailSender: email,
        smsSender:   sms,
        pushSender:  push,
    }
}

func (n *NotificationService) NotifyUser(userID int, message string) error {
    if userID <= 0 {
        return errors.New("userID invalide")
    }
    if message == "" {
        return errors.New("message vide")
    }

    var errors []string

    // Envoyer par email
    if err := n.emailSender.SendEmail(userID, message); err != nil {
        errors = append(errors, fmt.Sprintf("email: %v", err))
    }

    // Envoyer par SMS
    if err := n.smsSender.SendSMS(userID, message); err != nil {
        errors = append(errors, fmt.Sprintf("sms: %v", err))
    }

    // Envoyer par push
    if err := n.pushSender.SendPush(userID, message); err != nil {
        errors = append(errors, fmt.Sprintf("push: %v", err))
    }

    // Si toutes les notifications ont échoué
    if len(errors) == 3 {
        return fmt.Errorf("toutes les notifications ont échoué: %v", errors)
    }

    // Si certaines ont échoué, on log mais on ne retourne pas d'erreur
    if len(errors) > 0 {
        fmt.Printf("Certaines notifications ont échoué: %v\n", errors)
    }

    return nil
}

// Implémentations réelles (pour la production)
type RealEmailSender struct{}

func (r *RealEmailSender) SendEmail(userID int, message string) error {
    fmt.Printf("Email envoyé à l'utilisateur %d: %s\n", userID, message)
    return nil
}

type RealSMSSender struct{}

func (r *RealSMSSender) SendSMS(userID int, message string) error {
    fmt.Printf("SMS envoyé à l'utilisateur %d: %s\n", userID, message)
    return nil
}

type RealPushSender struct{}

func (r *RealPushSender) SendPush(userID int, message string) error {
    fmt.Printf("Push envoyé à l'utilisateur %d: %s\n", userID, message)
    return nil
}

// Mocks pour les tests
type MockEmailSender struct {
    SentEmails []EmailNotification
    ShouldFail bool
    FailError  error
}

type EmailNotification struct {
    UserID  int
    Message string
}

func NewMockEmailSender() *MockEmailSender {
    return &MockEmailSender{
        SentEmails: make([]EmailNotification, 0),
    }
}

func (m *MockEmailSender) SendEmail(userID int, message string) error {
    if m.ShouldFail {
        if m.FailError != nil {
            return m.FailError
        }
        return errors.New("erreur email simulée")
    }

    m.SentEmails = append(m.SentEmails, EmailNotification{
        UserID:  userID,
        Message: message,
    })

    return nil
}

func (m *MockEmailSender) GetSentCount() int {
    return len(m.SentEmails)
}

func (m *MockEmailSender) WasSentToUser(userID int) bool {
    for _, email := range m.SentEmails {
        if email.UserID == userID {
            return true
        }
    }
    return false
}

type MockSMSSender struct {
    SentSMS    []SMSNotification
    ShouldFail bool
    FailError  error
}

type SMSNotification struct {
    UserID  int
    Message string
}

func NewMockSMSSender() *MockSMSSender {
    return &MockSMSSender{
        SentSMS: make([]SMSNotification, 0),
    }
}

func (m *MockSMSSender) SendSMS(userID int, message string) error {
    if m.ShouldFail {
        if m.FailError != nil {
            return m.FailError
        }
        return errors.New("erreur SMS simulée")
    }

    m.SentSMS = append(m.SentSMS, SMSNotification{
        UserID:  userID,
        Message: message,
    })

    return nil
}

func (m *MockSMSSender) GetSentCount() int {
    return len(m.SentSMS)
}

func (m *MockSMSSender) WasSentToUser(userID int) bool {
    for _, sms := range m.SentSMS {
        if sms.UserID == userID {
            return true
        }
    }
    return false
}

type MockPushSender struct {
    SentPush   []PushNotification
    ShouldFail bool
    FailError  error
}

type PushNotification struct {
    UserID  int
    Message string
}

func NewMockPushSender() *MockPushSender {
    return &MockPushSender{
        SentPush: make([]PushNotification, 0),
    }
}

func (m *MockPushSender) SendPush(userID int, message string) error {
    if m.ShouldFail {
        if m.FailError != nil {
            return m.FailError
        }
        return errors.New("erreur push simulée")
    }

    m.SentPush = append(m.SentPush, PushNotification{
        UserID:  userID,
        Message: message,
    })

    return nil
}

func (m *MockPushSender) GetSentCount() int {
    return len(m.SentPush)
}

func (m *MockPushSender) WasSentToUser(userID int) bool {
    for _, push := range m.SentPush {
        if push.UserID == userID {
            return true
        }
    }
    return false
}
```

### Tests complets avec différentes combinaisons

```go
package main

import (
    "strings"
    "testing"
)

func TestNotificationService_NotifyUser_Success(t *testing.T) {
    // Setup des mocks
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    service := NewNotificationService(mockEmail, mockSMS, mockPush)

    // Action
    err := service.NotifyUser(123, "Message de test")

    // Vérifications
    if err != nil {
        t.Errorf("NotifyUser() erreur inattendue: %v", err)
    }

    // Vérifier que toutes les notifications ont été envoyées
    if mockEmail.GetSentCount() != 1 {
        t.Errorf("Email: envoyé %d fois; want 1", mockEmail.GetSentCount())
    }
    if mockSMS.GetSentCount() != 1 {
        t.Errorf("SMS: envoyé %d fois; want 1", mockSMS.GetSentCount())
    }
    if mockPush.GetSentCount() != 1 {
        t.Errorf("Push: envoyé %d fois; want 1", mockPush.GetSentCount())
    }

    // Vérifier que les notifications ont été envoyées au bon utilisateur
    if !mockEmail.WasSentToUser(123) {
        t.Error("Email devrait avoir été envoyé à l'utilisateur 123")
    }
    if !mockSMS.WasSentToUser(123) {
        t.Error("SMS devrait avoir été envoyé à l'utilisateur 123")
    }
    if !mockPush.WasSentToUser(123) {
        t.Error("Push devrait avoir été envoyé à l'utilisateur 123")
    }

    // Vérifier le contenu du message
    if len(mockEmail.SentEmails) > 0 && mockEmail.SentEmails[0].Message != "Message de test" {
        t.Errorf("Message email = %q; want %q", mockEmail.SentEmails[0].Message, "Message de test")
    }
}

func TestNotificationService_NotifyUser_ValidationErrors(t *testing.T) {
    tests := []struct {
        name    string
        userID  int
        message string
        wantErr bool
    }{
        {"userID invalide (0)", 0, "Message", true},
        {"userID invalide (négatif)", -1, "Message", true},
        {"message vide", 123, "", true},
        {"paramètres valides", 123, "Message", false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockEmail := NewMockEmailSender()
            mockSMS := NewMockSMSSender()
            mockPush := NewMockPushSender()
            service := NewNotificationService(mockEmail, mockSMS, mockPush)

            // Action
            err := service.NotifyUser(tt.userID, tt.message)

            // Vérifications
            if tt.wantErr {
                if err == nil {
                    t.Error("NotifyUser() devrait retourner une erreur")
                }
                // Vérifier qu'aucune notification n'a été envoyée
                if mockEmail.GetSentCount() != 0 {
                    t.Error("Aucun email ne devrait être envoyé en cas d'erreur de validation")
                }
                return
            }

            if err != nil {
                t.Errorf("NotifyUser() erreur inattendue: %v", err)
            }
        })
    }
}

func TestNotificationService_NotifyUser_PartialFailures(t *testing.T) {
    tests := []struct {
        name        string
        emailFails  bool
        smsFails    bool
        pushFails   bool
        expectError bool
        description string
    }{
        {
            name:        "échec email seulement",
            emailFails:  true,
            smsFails:    false,
            pushFails:   false,
            expectError: false,
            description: "SMS et Push fonctionnent",
        },
        {
            name:        "échec SMS seulement",
            emailFails:  false,
            smsFails:    true,
            pushFails:   false,
            expectError: false,
            description: "Email et Push fonctionnent",
        },
        {
            name:        "échec Push seulement",
            emailFails:  false,
            smsFails:    false,
            pushFails:   true,
            expectError: false,
            description: "Email et SMS fonctionnent",
        },
        {
            name:        "échec Email et SMS",
            emailFails:  true,
            smsFails:    true,
            pushFails:   false,
            expectError: false,
            description: "Push fonctionne",
        },
        {
            name:        "tout échoue",
            emailFails:  true,
            smsFails:    true,
            pushFails:   true,
            expectError: true,
            description: "Toutes les notifications échouent",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup des mocks
            mockEmail := NewMockEmailSender()
            mockSMS := NewMockSMSSender()
            mockPush := NewMockPushSender()

            mockEmail.ShouldFail = tt.emailFails
            mockSMS.ShouldFail = tt.smsFails
            mockPush.ShouldFail = tt.pushFails

            service := NewNotificationService(mockEmail, mockSMS, mockPush)

            // Action
            err := service.NotifyUser(123, "Message de test")

            // Vérifications
            if tt.expectError {
                if err == nil {
                    t.Error("NotifyUser() devrait retourner une erreur quand tout échoue")
                }
                if err != nil && !strings.Contains(err.Error(), "toutes les notifications ont échoué") {
                    t.Errorf("Erreur devrait mentionner que toutes les notifications ont échoué, got: %v", err)
                }
            } else {
                if err != nil {
                    t.Errorf("NotifyUser() ne devrait pas retourner d'erreur en cas d'échec partiel: %v", err)
                }
            }

            // Vérifier le comportement des mocks
            expectedEmailCount := 0
            if !tt.emailFails {
                expectedEmailCount = 1
            }
            if mockEmail.GetSentCount() != expectedEmailCount {
                t.Errorf("Email envoyé %d fois; want %d", mockEmail.GetSentCount(), expectedEmailCount)
            }

            expectedSMSCount := 0
            if !tt.smsFails {
                expectedSMSCount = 1
            }
            if mockSMS.GetSentCount() != expectedSMSCount {
                t.Errorf("SMS envoyé %d fois; want %d", mockSMS.GetSentCount(), expectedSMSCount)
            }

            expectedPushCount := 0
            if !tt.pushFails {
                expectedPushCount = 1
            }
            if mockPush.GetSentCount() != expectedPushCount {
                t.Errorf("Push envoyé %d fois; want %d", mockPush.GetSentCount(), expectedPushCount)
            }
        })
    }
}

func TestNotificationService_NotifyUser_CustomErrors(t *testing.T) {
    // Setup avec des erreurs personnalisées
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    mockEmail.ShouldFail = true
    mockEmail.FailError = errors.New("serveur email indisponible")

    mockSMS.ShouldFail = true
    mockSMS.FailError = errors.New("quota SMS dépassé")

    // Push fonctionne
    mockPush.ShouldFail = false

    service := NewNotificationService(mockEmail, mockSMS, mockPush)

    // Action
    err := service.NotifyUser(123, "Message de test")

    // Vérifications - ne devrait pas retourner d'erreur car Push fonctionne
    if err != nil {
        t.Errorf("NotifyUser() ne devrait pas retourner d'erreur car Push fonctionne: %v", err)
    }

    // Vérifier que Push a été envoyé
    if mockPush.GetSentCount() != 1 {
        t.Errorf("Push devrait être envoyé 1 fois, envoyé %d fois", mockPush.GetSentCount())
    }

    // Vérifier qu'Email et SMS ont échoué (aucun envoi)
    if mockEmail.GetSentCount() != 0 {
        t.Errorf("Email ne devrait pas être envoyé en cas d'échec, envoyé %d fois", mockEmail.GetSentCount())
    }
    if mockSMS.GetSentCount() != 0 {
        t.Errorf("SMS ne devrait pas être envoyé en cas d'échec, envoyé %d fois", mockSMS.GetSentCount())
    }
}

func TestNotificationService_NotifyUser_MultipleUsers(t *testing.T) {
    // Test avec plusieurs utilisateurs
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    service := NewNotificationService(mockEmail, mockSMS, mockPush)

    users := []int{123, 456, 789}
    message := "Message pour tous"

    // Envoyer des notifications à plusieurs utilisateurs
    for _, userID := range users {
        err := service.NotifyUser(userID, message)
        if err != nil {
            t.Errorf("NotifyUser(%d) erreur: %v", userID, err)
        }
    }

    // Vérifier que chaque type de notification a été envoyé 3 fois
    if mockEmail.GetSentCount() != 3 {
        t.Errorf("Email envoyé %d fois; want 3", mockEmail.GetSentCount())
    }
    if mockSMS.GetSentCount() != 3 {
        t.Errorf("SMS envoyé %d fois; want 3", mockSMS.GetSentCount())
    }
    if mockPush.GetSentCount() != 3 {
        t.Errorf("Push envoyé %d fois; want 3", mockPush.GetSentCount())
    }

    // Vérifier que chaque utilisateur a reçu ses notifications
    for _, userID := range users {
        if !mockEmail.WasSentToUser(userID) {
            t.Errorf("Email devrait avoir été envoyé à l'utilisateur %d", userID)
        }
        if !mockSMS.WasSentToUser(userID) {
            t.Errorf("SMS devrait avoir été envoyé à l'utilisateur %d", userID)
        }
        if !mockPush.WasSentToUser(userID) {
            t.Errorf("Push devrait avoir été envoyé à l'utilisateur %d", userID)
        }
    }
}
```

## Tests d'intégration avec les mocks

```go
func TestNotificationService_Integration(t *testing.T) {
    // Test d'intégration simulant un scénario réel
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    service := NewNotificationService(mockEmail, mockSMS, mockPush)

    // Scénario : notification d'urgence où SMS échoue mais autres fonctionnent
    mockSMS.ShouldFail = true
    mockSMS.FailError = errors.New("réseau SMS surchargé")

    err := service.NotifyUser(999, "Alerte urgente")

    // Ne devrait pas retourner d'erreur car 2/3 des notifications fonctionnent
    if err != nil {
        t.Errorf("NotifyUser() ne devrait pas échouer si au moins une notification fonctionne: %v", err)
    }

    // Vérifier que les notifications qui fonctionnent ont été envoyées
    if mockEmail.GetSentCount() != 1 {
        t.Errorf("Email devrait être envoyé 1 fois, envoyé %d fois", mockEmail.GetSentCount())
    }
    if mockPush.GetSentCount() != 1 {
        t.Errorf("Push devrait être envoyé 1 fois, envoyé %d fois", mockPush.GetSentCount())
    }

    // Vérifier que SMS a échoué (pas d'envoi)
    if mockSMS.GetSentCount() != 0 {
        t.Errorf("SMS ne devrait pas être envoyé en cas d'échec, envoyé %d fois", mockSMS.GetSentCount())
    }

    // Vérifier le contenu des notifications envoyées
    if len(mockEmail.SentEmails) > 0 {
        email := mockEmail.SentEmails[0]
        if email.UserID != 999 {
            t.Errorf("Email UserID = %d; want 999", email.UserID)
        }
        if email.Message != "Alerte urgente" {
            t.Errorf("Email Message = %q; want %q", email.Message, "Alerte urgente")
        }
    }

    if len(mockPush.SentPush) > 0 {
        push := mockPush.SentPush[0]
        if push.UserID != 999 {
            t.Errorf("Push UserID = %d; want 999", push.UserID)
        }
        if push.Message != "Alerte urgente" {
            t.Errorf("Push Message = %q; want %q", push.Message, "Alerte urgente")
        }
    }
}

func TestNotificationService_RealWorldScenarios(t *testing.T) {
    scenarios := []struct {
        name        string
        setupMocks  func(*MockEmailSender, *MockSMSSender, *MockPushSender)
        userID      int
        message     string
        expectError bool
        description string
    }{
        {
            name: "service email en maintenance",
            setupMocks: func(email *MockEmailSender, sms *MockSMSSender, push *MockPushSender) {
                email.ShouldFail = true
                email.FailError = errors.New("service email en maintenance")
            },
            userID:      100,
            message:     "Service temporairement indisponible",
            expectError: false,
            description: "SMS et Push compensent l'échec de l'email",
        },
        {
            name: "quota SMS dépassé",
            setupMocks: func(email *MockEmailSender, sms *MockSMSSender, push *MockPushSender) {
                sms.ShouldFail = true
                sms.FailError = errors.New("quota mensuel SMS dépassé")
            },
            userID:      200,
            message:     "Votre commande a été expédiée",
            expectError: false,
            description: "Email et Push fonctionnent normalement",
        },
        {
            name: "service push indisponible",
            setupMocks: func(email *MockEmailSender, sms *MockSMSSender, push *MockPushSender) {
                push.ShouldFail = true
                push.FailError = errors.New("serveur push notifications down")
            },
            userID:      300,
            message:     "Nouveau message reçu",
            expectError: false,
            description: "Email et SMS suffisent pour la notification",
        },
        {
            name: "panne généralisée",
            setupMocks: func(email *MockEmailSender, sms *MockSMSSender, push *MockPushSender) {
                email.ShouldFail = true
                email.FailError = errors.New("serveur email down")
                sms.ShouldFail = true
                sms.FailError = errors.New("passerelle SMS inaccessible")
                push.ShouldFail = true
                push.FailError = errors.New("service push en panne")
            },
            userID:      400,
            message:     "Message critique",
            expectError: true,
            description: "Tous les services sont en panne",
        },
        {
            name: "fonctionnement normal",
            setupMocks: func(email *MockEmailSender, sms *MockSMSSender, push *MockPushSender) {
                // Tous les mocks fonctionnent normalement
            },
            userID:      500,
            message:     "Bienvenue sur notre plateforme",
            expectError: false,
            description: "Toutes les notifications envoyées avec succès",
        },
    }

    for _, scenario := range scenarios {
        t.Run(scenario.name, func(t *testing.T) {
            // Setup des mocks
            mockEmail := NewMockEmailSender()
            mockSMS := NewMockSMSSender()
            mockPush := NewMockPushSender()

            // Configuration spécifique au scénario
            scenario.setupMocks(mockEmail, mockSMS, mockPush)

            service := NewNotificationService(mockEmail, mockSMS, mockPush)

            // Action
            err := service.NotifyUser(scenario.userID, scenario.message)

            // Vérifications
            if scenario.expectError {
                if err == nil {
                    t.Errorf("Scénario '%s': devrait retourner une erreur", scenario.description)
                }
            } else {
                if err != nil {
                    t.Errorf("Scénario '%s': ne devrait pas retourner d'erreur, got: %v", scenario.description, err)
                }
            }

            t.Logf("Scénario '%s': %s", scenario.name, scenario.description)
            t.Logf("  - Emails envoyés: %d", mockEmail.GetSentCount())
            t.Logf("  - SMS envoyés: %d", mockSMS.GetSentCount())
            t.Logf("  - Push envoyés: %d", mockPush.GetSentCount())
        })
    }
}
```

## Tests de performance avec mocks

```go
func TestNotificationService_Performance(t *testing.T) {
    // Test de performance pour vérifier que les appels sont bien parallélisés
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    service := NewNotificationService(mockEmail, mockSMS, mockPush)

    // Simuler une charge importante
    userCount := 100
    message := "Notification de masse"

    start := time.Now()

    for i := 1; i <= userCount; i++ {
        err := service.NotifyUser(i, message)
        if err != nil {
            t.Errorf("NotifyUser(%d) erreur: %v", i, err)
        }
    }

    duration := time.Since(start)

    // Vérifications
    expectedEmails := userCount
    expectedSMS := userCount
    expectedPush := userCount

    if mockEmail.GetSentCount() != expectedEmails {
        t.Errorf("Emails envoyés: %d; want %d", mockEmail.GetSentCount(), expectedEmails)
    }
    if mockSMS.GetSentCount() != expectedSMS {
        t.Errorf("SMS envoyés: %d; want %d", mockSMS.GetSentCount(), expectedSMS)
    }
    if mockPush.GetSentCount() != expectedPush {
        t.Errorf("Push envoyés: %d; want %d", mockPush.GetSentCount(), expectedPush)
    }

    t.Logf("Envoi de %d notifications en %v", userCount*3, duration)

    // Le test avec des mocks devrait être très rapide
    if duration > time.Second {
        t.Errorf("Test trop lent: %v, devrait être < 1s avec des mocks", duration)
    }
}
```

## Tests avec injection de dépendances complexe

```go
// Service plus complexe utilisant le NotificationService
type UserRegistrationService struct {
    notificationService *NotificationService
    userRepository      UserRepository // Interface définie dans l'exercice 1
    logger             Logger
}

type Logger interface {
    Log(level string, message string)
}

type MockLogger struct {
    Logs []LogEntry
}

type LogEntry struct {
    Level   string
    Message string
}

func NewMockLogger() *MockLogger {
    return &MockLogger{
        Logs: make([]LogEntry, 0),
    }
}

func (m *MockLogger) Log(level, message string) {
    m.Logs = append(m.Logs, LogEntry{
        Level:   level,
        Message: message,
    })
}

func (m *MockLogger) GetLogCount() int {
    return len(m.Logs)
}

func (m *MockLogger) GetLogsForLevel(level string) []LogEntry {
    var logs []LogEntry
    for _, log := range m.Logs {
        if log.Level == level {
            logs = append(logs, log)
        }
    }
    return logs
}

func NewUserRegistrationService(
    notification *NotificationService,
    repository UserRepository,
    logger Logger,
) *UserRegistrationService {
    return &UserRegistrationService{
        notificationService: notification,
        userRepository:      repository,
        logger:             logger,
    }
}

func (u *UserRegistrationService) RegisterNewUser(name, email string) error {
    u.logger.Log("INFO", fmt.Sprintf("Début d'enregistrement pour %s", email))

    // Créer l'utilisateur
    user := User{Name: name, Email: email}

    // Sauvegarder en base
    err := u.userRepository.SaveUser(user)
    if err != nil {
        u.logger.Log("ERROR", fmt.Sprintf("Échec sauvegarde utilisateur %s: %v", email, err))
        return fmt.Errorf("erreur de sauvegarde: %w", err)
    }

    u.logger.Log("INFO", fmt.Sprintf("Utilisateur %s sauvegardé avec ID %d", email, user.ID))

    // Envoyer notification de bienvenue
    message := fmt.Sprintf("Bienvenue %s ! Votre compte a été créé avec succès.", name)
    err = u.notificationService.NotifyUser(user.ID, message)
    if err != nil {
        u.logger.Log("WARNING", fmt.Sprintf("Échec envoi notification à %s: %v", email, err))
        // On ne fait pas échouer l'enregistrement pour un problème de notification
    } else {
        u.logger.Log("INFO", fmt.Sprintf("Notifications envoyées à %s", email))
    }

    return nil
}

// Test du service complet avec tous les mocks
func TestUserRegistrationService_RegisterNewUser(t *testing.T) {
    tests := []struct {
        name                  string
        userName              string
        userEmail             string
        repositoryError       error
        notificationError     bool
        expectedLogCount      int
        expectedErrorLogs     int
        expectedWarningLogs   int
        expectRegistrationErr bool
    }{
        {
            name:                  "enregistrement complet réussi",
            userName:              "Alice Dupont",
            userEmail:             "alice@example.com",
            repositoryError:       nil,
            notificationError:     false,
            expectedLogCount:      3, // INFO début, INFO sauvegarde, INFO notification
            expectedErrorLogs:     0,
            expectedWarningLogs:   0,
            expectRegistrationErr: false,
        },
        {
            name:                  "échec de sauvegarde",
            userName:              "Bob Martin",
            userEmail:             "bob@example.com",
            repositoryError:       errors.New("erreur base de données"),
            notificationError:     false,
            expectedLogCount:      2, // INFO début, ERROR sauvegarde
            expectedErrorLogs:     1,
            expectedWarningLogs:   0,
            expectRegistrationErr: true,
        },
        {
            name:                  "sauvegarde OK mais notification échoue",
            userName:              "Charlie Durand",
            userEmail:             "charlie@example.com",
            repositoryError:       nil,
            notificationError:     true,
            expectedLogCount:      3, // INFO début, INFO sauvegarde, WARNING notification
            expectedErrorLogs:     0,
            expectedWarningLogs:   1,
            expectRegistrationErr: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup de tous les mocks
            mockRepo := NewMockUserRepository()
            mockRepo.SaveError = tt.repositoryError

            mockEmail := NewMockEmailSender()
            mockSMS := NewMockSMSSender()
            mockPush := NewMockPushSender()

            if tt.notificationError {
                // Faire échouer toutes les notifications
                mockEmail.ShouldFail = true
                mockSMS.ShouldFail = true
                mockPush.ShouldFail = true
            }

            mockNotificationService := NewNotificationService(mockEmail, mockSMS, mockPush)
            mockLogger := NewMockLogger()

            service := NewUserRegistrationService(mockNotificationService, mockRepo, mockLogger)

            // Action
            err := service.RegisterNewUser(tt.userName, tt.userEmail)

            // Vérifications d'erreur
            if tt.expectRegistrationErr {
                if err == nil {
                    t.Error("RegisterNewUser() devrait retourner une erreur")
                }
            } else {
                if err != nil {
                    t.Errorf("RegisterNewUser() erreur inattendue: %v", err)
                }
            }

            // Vérifications des logs
            if mockLogger.GetLogCount() != tt.expectedLogCount {
                t.Errorf("Nombre de logs = %d; want %d", mockLogger.GetLogCount(), tt.expectedLogCount)
            }

            errorLogs := mockLogger.GetLogsForLevel("ERROR")
            if len(errorLogs) != tt.expectedErrorLogs {
                t.Errorf("Logs ERROR = %d; want %d", len(errorLogs), tt.expectedErrorLogs)
            }

            warningLogs := mockLogger.GetLogsForLevel("WARNING")
            if len(warningLogs) != tt.expectedWarningLogs {
                t.Errorf("Logs WARNING = %d; want %d", len(warningLogs), tt.expectedWarningLogs)
            }

            // Vérifications spécifiques selon le cas
            if !tt.expectRegistrationErr && tt.repositoryError == nil {
                // L'utilisateur devrait être sauvegardé
                if len(mockRepo.Users) != 1 {
                    t.Errorf("Utilisateur devrait être sauvegardé, trouvé %d utilisateurs", len(mockRepo.Users))
                }

                if !tt.notificationError {
                    // Les notifications devraient être envoyées
                    if mockEmail.GetSentCount() == 0 {
                        t.Error("Email devrait être envoyé")
                    }
                    if mockSMS.GetSentCount() == 0 {
                        t.Error("SMS devrait être envoyé")
                    }
                    if mockPush.GetSentCount() == 0 {
                        t.Error("Push devrait être envoyé")
                    }
                }
            }
        })
    }
}
```

## Helpers pour les tests complexes

```go
// Helper pour créer un service de notification avec des mocks préconfigurés
func createTestNotificationService(emailFails, smsFails, pushFails bool) (*NotificationService, *MockEmailSender, *MockSMSSender, *MockPushSender) {
    mockEmail := NewMockEmailSender()
    mockSMS := NewMockSMSSender()
    mockPush := NewMockPushSender()

    mockEmail.ShouldFail = emailFails
    mockSMS.ShouldFail = smsFails
    mockPush.ShouldFail = pushFails

    service := NewNotificationService(mockEmail, mockSMS, mockPush)
    return service, mockEmail, mockSMS, mockPush
}

// Helper pour vérifier qu'une notification a été envoyée à un utilisateur
func assertNotificationSent(t *testing.T, mockEmail *MockEmailSender, mockSMS *MockSMSSender, mockPush *MockPushSender, userID int, message string) {
    t.Helper()

    // Vérifier email
    found := false
    for _, email := range mockEmail.SentEmails {
        if email.UserID == userID && email.Message == message {
            found = true
            break
        }
    }
    if !found && !mockEmail.ShouldFail {
        t.Errorf("Email non trouvé pour utilisateur %d avec message %q", userID, message)
    }

    // Vérifier SMS
    found = false
    for _, sms := range mockSMS.SentSMS {
        if sms.UserID == userID && sms.Message == message {
            found = true
            break
        }
    }
    if !found && !mockSMS.ShouldFail {
        t.Errorf("SMS non trouvé pour utilisateur %d avec message %q", userID, message)
    }

    // Vérifier Push
    found = false
    for _, push := range mockPush.SentPush {
        if push.UserID == userID && push.Message == message {
            found = true
            break
        }
    }
    if !found && !mockPush.ShouldFail {
        t.Errorf("Push non trouvé pour utilisateur %d avec message %q", userID, message)
    }
}

// Test utilisant les helpers
func TestNotificationService_WithHelpers(t *testing.T) {
    // Créer un service où seul l'email échoue
    service, mockEmail, mockSMS, mockPush := createTestNotificationService(true, false, false)

    err := service.NotifyUser(123, "Test avec helpers")
    if err != nil {
        t.Errorf("Ne devrait pas échouer car SMS et Push fonctionnent: %v", err)
    }

    // Utiliser le helper pour vérifier
    assertNotificationSent(t, mockEmail, mockSMS, mockPush, 123, "Test avec helpers")
}
```

## Récapitulatif des solutions

### **Points clés des solutions :**

1. **Structure modulaire** : Interfaces claires séparant les responsabilités
2. **Mocks robustes** : Traçabilité des appels, simulation d'erreurs, états configurables
3. **Tests exhaustifs** : Cas normaux, cas d'erreur, échecs partiels, scénarios réels
4. **Helpers réutilisables** : Fonctions d'assistance pour réduire la duplication
5. **Tests d'intégration** : Validation du comportement complet avec mocks

### **Techniques avancées démontrées :**

- **Injection de dépendances** multiples
- **Simulation d'erreurs** spécifiques et génériques
- **Traçabilité** des interactions avec les mocks
- **Tests de performance** avec mocks
- **Logs et observabilité** dans les tests
- **Helpers personnalisés** pour simplifier les assertions

### **Bonnes pratiques appliquées :**

- Interfaces minimales et focalisées
- Mocks simples avec état contrôlable
- Tests table-driven pour les scénarios multiples
- Vérification des interactions ET des états
- Séparation claire entre setup, action et assertions

Ces solutions montrent comment le mocking permet de tester efficacement du code complexe avec de nombreuses dépendances, tout en gardant les tests rapides, fiables et maintenables !

⏭️
