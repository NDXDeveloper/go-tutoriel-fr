🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8-2 : Visibilité (public/private)

## Qu'est-ce que la visibilité ?

La **visibilité** détermine si un élément de votre code (fonction, variable, struct, etc.) peut être utilisé depuis un autre package. En Go, cette visibilité est contrôlée par une règle très simple : **la première lettre du nom**.

## La règle d'or de Go

### Public (Exporté)
Si le nom commence par une **lettre MAJUSCULE**, l'élément est **public** et peut être utilisé depuis d'autres packages.

```go
func Add(a, b int) int {     // ✅ Public - accessible depuis d'autres packages
    return a + b
}

var GlobalConfig string      // ✅ Public - accessible depuis d'autres packages

type User struct {           // ✅ Public - accessible depuis d'autres packages
    Name string              // ✅ Public - champ accessible
}
```

### Private (Non-exporté)
Si le nom commence par une **lettre minuscule**, l'élément est **privé** et ne peut être utilisé que dans le même package.

```go
func add(a, b int) int {     // ❌ Privé - accessible uniquement dans ce package
    return a + b
}

var internalConfig string    // ❌ Privé - accessible uniquement dans ce package

type user struct {           // ❌ Privé - accessible uniquement dans ce package
    name string              // ❌ Privé - champ accessible uniquement dans ce package
}
```

## Exemple pratique : Gestion d'utilisateurs

Créons un package pour gérer des utilisateurs et voir la visibilité en action.

### Structure du projet
```
user-management/
├── go.mod
├── main.go
└── user/
    ├── user.go
    └── validation.go
```

### 1. Fichier user/user.go
```go
package user

import "fmt"

// User représente un utilisateur (PUBLIC)
type User struct {
    Name    string // PUBLIC - accessible depuis l'extérieur
    Email   string // PUBLIC - accessible depuis l'extérieur
    age     int    // PRIVÉ - accessible uniquement dans le package user
    password string // PRIVÉ - accessible uniquement dans le package user
}

// NewUser crée un nouvel utilisateur (PUBLIC)
func NewUser(name, email string, age int) *User {
    return &User{
        Name:  name,
        Email: email,
        age:   age,
    }
}

// GetAge retourne l'âge de l'utilisateur (PUBLIC)
func (u *User) GetAge() int {
    return u.age
}

// SetAge modifie l'âge de l'utilisateur (PUBLIC)
func (u *User) SetAge(newAge int) {
    if isValidAge(newAge) { // Utilise une fonction privée
        u.age = newAge
    }
}

// setPassword modifie le mot de passe (PRIVÉ)
func (u *User) setPassword(pwd string) {
    u.password = pwd
}

// GetInfo retourne les informations de l'utilisateur (PUBLIC)
func (u *User) GetInfo() string {
    return fmt.Sprintf("Nom: %s, Email: %s, Âge: %d", u.Name, u.Email, u.age)
}

// helper function privée
func isValidAge(age int) bool {
    return age >= 0 && age <= 150
}
```

### 2. Fichier user/validation.go
```go
package user

import "strings"

// IsValidEmail vérifie si un email est valide (PUBLIC)
func IsValidEmail(email string) bool {
    return strings.Contains(email, "@") && strings.Contains(email, ".")
}

// validateName vérifie si un nom est valide (PRIVÉ)
func validateName(name string) bool {
    return len(name) >= 2 && len(name) <= 50
}

// CreateUserSafe crée un utilisateur avec validation (PUBLIC)
func CreateUserSafe(name, email string, age int) (*User, error) {
    if !validateName(name) {
        return nil, fmt.Errorf("nom invalide: %s", name)
    }

    if !IsValidEmail(email) {
        return nil, fmt.Errorf("email invalide: %s", email)
    }

    if !isValidAge(age) {
        return nil, fmt.Errorf("âge invalide: %d", age)
    }

    return NewUser(name, email, age), nil
}
```

### 3. Fichier main.go
```go
package main

import (
    "fmt"
    "user-management/user"
)

func main() {
    // ✅ FONCTIONS PUBLIQUES - Accessibles

    // Création d'un utilisateur
    u := user.NewUser("Alice", "alice@example.com", 25)
    fmt.Println(u.GetInfo())

    // Accès aux champs publics
    fmt.Println("Nom:", u.Name)
    fmt.Println("Email:", u.Email)

    // Utilisation des méthodes publiques
    fmt.Println("Âge:", u.GetAge())
    u.SetAge(26)
    fmt.Println("Nouvel âge:", u.GetAge())

    // Validation avec fonction publique
    if user.IsValidEmail("test@example.com") {
        fmt.Println("Email valide")
    }

    // Création sécurisée
    safeUser, err := user.CreateUserSafe("Bob", "bob@example.com", 30)
    if err != nil {
        fmt.Println("Erreur:", err)
    } else {
        fmt.Println("Utilisateur créé:", safeUser.GetInfo())
    }

    // ❌ ERREURS - Éléments privés non accessibles

    // fmt.Println(u.age)        // ❌ Erreur : age est privé
    // fmt.Println(u.password)   // ❌ Erreur : password est privé
    // u.setPassword("secret")   // ❌ Erreur : setPassword est privé
    // validateName("test")      // ❌ Erreur : validateName est privé
}
```

## Visibilité des structs et de leurs champs

### Struct publique avec champs mixtes
```go
type Product struct {
    Name        string  // PUBLIC - accessible depuis l'extérieur
    Price       float64 // PUBLIC - accessible depuis l'extérieur
    description string  // PRIVÉ - accessible uniquement dans ce package
    stock       int     // PRIVÉ - accessible uniquement dans ce package
}
```

### Utilisation depuis un autre package
```go
// Dans un autre package
p := shop.Product{
    Name:  "Laptop",     // ✅ OK - champ public
    Price: 999.99,       // ✅ OK - champ public
    // description: "...", // ❌ Erreur - champ privé
    // stock: 10,          // ❌ Erreur - champ privé
}
```

## Méthodes avec visibilité

### Exemple : Compte bancaire
```go
package bank

type Account struct {
    Owner   string  // PUBLIC
    balance float64 // PRIVÉ - sécurité importante !
}

// GetBalance retourne le solde (PUBLIC)
func (a *Account) GetBalance() float64 {
    return a.balance
}

// Deposit ajoute de l'argent (PUBLIC)
func (a *Account) Deposit(amount float64) {
    if amount > 0 {
        a.balance += amount
    }
}

// Withdraw retire de l'argent (PUBLIC)
func (a *Account) Withdraw(amount float64) bool {
    if canWithdraw(a.balance, amount) {
        a.balance -= amount
        return true
    }
    return false
}

// canWithdraw vérifie si le retrait est possible (PRIVÉ)
func canWithdraw(balance, amount float64) bool {
    return balance >= amount && amount > 0
}

// resetBalance remet le solde à zéro (PRIVÉ)
func (a *Account) resetBalance() {
    a.balance = 0
}
```

## Interfaces et visibilité

### Interface publique avec méthodes mixtes
```go
package shapes

// Shape interface publique
type Shape interface {
    Area() float64      // Méthode publique requise
    Perimeter() float64 // Méthode publique requise
}

// Rectangle implémente Shape
type Rectangle struct {
    Width  float64 // PUBLIC
    Height float64 // PUBLIC
}

// Area calcule l'aire (PUBLIC - requis par l'interface)
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Perimeter calcule le périmètre (PUBLIC - requis par l'interface)
func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}

// validate vérifie les dimensions (PRIVÉ)
func (r Rectangle) validate() bool {
    return r.Width > 0 && r.Height > 0
}
```

## Bonnes pratiques

### 1. Principe du minimum de visibilité
Commencez toujours par rendre les éléments privés, puis rendez-les publics seulement si nécessaire.

```go
// ✅ Bon : API minimaliste
type Database struct {
    connection string // privé
}

func (db *Database) Connect() error { /* ... */ }    // public
func (db *Database) Query(sql string) { /* ... */ }  // public
func (db *Database) close() { /* ... */ }            // privé
```

### 2. Exposer des méthodes plutôt que des champs
```go
// ❌ Évitez : exposition directe des champs
type User struct {
    Balance float64 // N'importe qui peut modifier le solde !
}

// ✅ Préférez : contrôle via des méthodes
type User struct {
    balance float64 // privé
}

func (u *User) GetBalance() float64 { return u.balance }
func (u *User) AddToBalance(amount float64) {
    if amount > 0 {
        u.balance += amount
    }
}
```

### 3. Constructeurs pour l'initialisation
```go
type Car struct {
    Brand string
    model string // privé
    year  int    // privé
}

// NewCar constructeur public
func NewCar(brand, model string, year int) *Car {
    return &Car{
        Brand: brand,
        model: model,
        year:  year,
    }
}

// GetModel getter public
func (c *Car) GetModel() string {
    return c.model
}
```

## Erreurs communes

### 1. Oublier la majuscule
```go
// ❌ Erreur : fonction non accessible depuis l'extérieur
func calculateTax(amount float64) float64 {
    return amount * 0.2
}

// ✅ Correct : fonction accessible depuis l'extérieur
func CalculateTax(amount float64) float64 {
    return amount * 0.2
}
```

### 2. Exposer trop d'éléments internes
```go
// ❌ Trop d'exposition
type Server struct {
    Port          int
    Host          string
    InternalState map[string]interface{} // Dangereux !
    DebugMode     bool
}

// ✅ Exposition contrôlée
type Server struct {
    port          int
    host          string
    internalState map[string]interface{}
    debugMode     bool
}

func (s *Server) GetPort() int { return s.port }
func (s *Server) GetHost() string { return s.host }
```

## Exercice pratique

Créez un package `calculator` qui implémente une calculatrice avec :

**Éléments publics :**
- Struct `Calculator`
- Méthodes : `Add()`, `Subtract()`, `Multiply()`, `Divide()`
- Fonction `NewCalculator()`
- Méthode `GetResult()`

**Éléments privés :**
- Champ `result` dans la struct
- Fonction `isValidNumber()`
- Méthode `reset()`

**Test dans main.go :**
```go
calc := calculator.NewCalculator()
calc.Add(10)
calc.Multiply(2)
fmt.Println(calc.GetResult()) // Doit afficher 20
```

## Résumé

La visibilité en Go est simple mais puissante :

**📋 Règles :**
- **Majuscule** = Public (exporté)
- **Minuscule** = Privé (non-exporté)

**✅ Avantages :**
- Contrôle de l'accès aux données
- APIs plus propres et sécurisées
- Encapsulation naturelle
- Maintenance facilitée

**🎯 Bonnes pratiques :**
- Commencez par rendre tout privé
- Exposez seulement ce qui est nécessaire
- Utilisez des méthodes pour contrôler l'accès
- Créez des constructeurs pour l'initialisation

Dans la section suivante, nous découvrirons les modules Go et la gestion des dépendances !

⏭️
