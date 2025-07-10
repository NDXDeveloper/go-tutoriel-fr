🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12-2 : Tags de struct

## Introduction aux tags de struct

Les tags de struct sont des métadonnées que vous pouvez attacher aux champs d'une structure. Ces tags ne sont pas visibles pendant l'exécution normale du programme, mais peuvent être lus via la réflexion. Ils sont largement utilisés par les bibliothèques Go pour configurer le comportement sans modifier le code.

## Syntaxe des tags

Un tag est une chaîne de caractères placée après le type d'un champ, entre des backticks (`` ` ``).

```go
type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}
```

### Format des tags

Les tags suivent généralement ce format : `key:"value"` ou `key:"value,option1,option2"`

```go
type Product struct {
    ID          int     `json:"id" db:"product_id" validate:"required"`
    Name        string  `json:"name" db:"name" validate:"required,min=1,max=100"`
    Price       float64 `json:"price" db:"price" validate:"required,gt=0"`
    Description string  `json:"description,omitempty" db:"description"`
    IsActive    bool    `json:"is_active" db:"is_active" validate:"required"`
}
```

## Lire les tags avec la réflexion

### Exemple de base

```go
package main

import (
    "fmt"
    "reflect"
)

type Person struct {
    Name string `json:"name" db:"full_name" validate:"required"`
    Age  int    `json:"age" db:"age" validate:"min=0,max=150"`
}

func main() {
    t := reflect.TypeOf(Person{})

    // Parcourir tous les champs
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)

        fmt.Printf("Champ: %s\n", field.Name)
        fmt.Printf("  Tag complet: %s\n", field.Tag)
        fmt.Printf("  Tag JSON: %s\n", field.Tag.Get("json"))
        fmt.Printf("  Tag DB: %s\n", field.Tag.Get("db"))
        fmt.Printf("  Tag Validate: %s\n", field.Tag.Get("validate"))
        fmt.Println()
    }
}
```

### Fonction utilitaire pour examiner les tags

```go
package main

import (
    "fmt"
    "reflect"
    "strings"
)

func inspectStruct(s interface{}) {
    t := reflect.TypeOf(s)

    // Si c'est un pointeur, obtenir le type sous-jacent
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    if t.Kind() != reflect.Struct {
        fmt.Println("Pas une structure")
        return
    }

    fmt.Printf("=== Inspection de %s ===\n", t.Name())

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("\nChamp: %s (type: %s)\n", field.Name, field.Type)

        if field.Tag != "" {
            fmt.Printf("  Tags:\n")

            // Méthode simple pour parser les tags
            tagStr := string(field.Tag)
            parts := strings.Fields(tagStr)

            for _, part := range parts {
                if colonIndex := strings.Index(part, ":"); colonIndex != -1 {
                    key := part[:colonIndex]
                    value := strings.Trim(part[colonIndex+1:], `"`)
                    fmt.Printf("    %s: %s\n", key, value)
                }
            }
        } else {
            fmt.Printf("  Aucun tag\n")
        }
    }
}

type Book struct {
    ISBN   string `json:"isbn" db:"isbn" validate:"required,isbn"`
    Title  string `json:"title" db:"title" validate:"required,min=1"`
    Author string `json:"author" db:"author_name" validate:"required"`
    Pages  int    `json:"pages" db:"page_count" validate:"min=1"`
}

func main() {
    book := Book{}
    inspectStruct(book)
}
```

## Cas d'usage courants

### 1. Sérialisation JSON

Les tags JSON sont probablement les plus courants en Go :

```go
package main

import (
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"-"`                    // Jamais sérialisé
    Bio      string `json:"bio,omitempty"`        // Omis si vide
    IsAdmin  bool   `json:"is_admin,omitempty"`   // Omis si false
}

func main() {
    user := User{
        ID:       1,
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
        Bio:      "",
        IsAdmin:  false,
    }

    jsonData, err := json.Marshal(user)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("JSON: %s\n", jsonData)
    // Résultat: {"id":1,"name":"Alice","email":"alice@example.com"}
    // Noter que Password, Bio et IsAdmin sont omis
}
```

### 2. Mapping base de données

```go
package main

import (
    "fmt"
    "reflect"
    "strings"
)

type Article struct {
    ID        int    `db:"id,primarykey,autoincrement"`
    Title     string `db:"title,not null"`
    Content   string `db:"content"`
    AuthorID  int    `db:"author_id,foreignkey"`
    CreatedAt string `db:"created_at,timestamp"`
}

// Simuler la génération d'une requête SQL
func generateCreateTable(s interface{}) string {
    t := reflect.TypeOf(s)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    var fields []string
    tableName := strings.ToLower(t.Name()) + "s"

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        dbTag := field.Tag.Get("db")

        if dbTag == "" || dbTag == "-" {
            continue
        }

        parts := strings.Split(dbTag, ",")
        columnName := parts[0]

        // Mapper les types Go vers SQL
        sqlType := mapGoTypeToSQL(field.Type)

        fieldDef := columnName + " " + sqlType

        // Ajouter les contraintes
        for _, part := range parts[1:] {
            switch part {
            case "primarykey":
                fieldDef += " PRIMARY KEY"
            case "autoincrement":
                fieldDef += " AUTOINCREMENT"
            case "not null":
                fieldDef += " NOT NULL"
            }
        }

        fields = append(fields, fieldDef)
    }

    return fmt.Sprintf("CREATE TABLE %s (\n  %s\n);",
        tableName, strings.Join(fields, ",\n  "))
}

func mapGoTypeToSQL(t reflect.Type) string {
    switch t.Kind() {
    case reflect.Int, reflect.Int32, reflect.Int64:
        return "INTEGER"
    case reflect.String:
        return "TEXT"
    case reflect.Bool:
        return "BOOLEAN"
    case reflect.Float32, reflect.Float64:
        return "REAL"
    default:
        return "TEXT"
    }
}

func main() {
    article := Article{}
    sql := generateCreateTable(article)
    fmt.Println(sql)
}
```

### 3. Validation de données

```go
package main

import (
    "fmt"
    "reflect"
    "strconv"
    "strings"
)

type Customer struct {
    Name  string `validate:"required,min=2,max=50"`
    Email string `validate:"required,email"`
    Age   int    `validate:"required,min=18,max=100"`
    Phone string `validate:"required,phone"`
}

type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Validateur simple basé sur les tags
func validate(s interface{}) []ValidationError {
    var errors []ValidationError

    v := reflect.ValueOf(s)
    t := reflect.TypeOf(s)

    if t.Kind() == reflect.Ptr {
        v = v.Elem()
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := v.Field(i)

        validateTag := field.Tag.Get("validate")
        if validateTag == "" {
            continue
        }

        rules := strings.Split(validateTag, ",")
        for _, rule := range rules {
            if err := validateRule(field.Name, value, rule); err != nil {
                errors = append(errors, *err)
            }
        }
    }

    return errors
}

func validateRule(fieldName string, value reflect.Value, rule string) *ValidationError {
    parts := strings.Split(rule, "=")
    ruleName := parts[0]

    switch ruleName {
    case "required":
        if isZeroValue(value) {
            return &ValidationError{fieldName, "champ obligatoire"}
        }
    case "min":
        if len(parts) < 2 {
            return nil
        }
        min, err := strconv.Atoi(parts[1])
        if err != nil {
            return nil
        }

        switch value.Kind() {
        case reflect.String:
            if len(value.String()) < min {
                return &ValidationError{fieldName, fmt.Sprintf("longueur minimale: %d", min)}
            }
        case reflect.Int, reflect.Int32, reflect.Int64:
            if value.Int() < int64(min) {
                return &ValidationError{fieldName, fmt.Sprintf("valeur minimale: %d", min)}
            }
        }
    case "max":
        if len(parts) < 2 {
            return nil
        }
        max, err := strconv.Atoi(parts[1])
        if err != nil {
            return nil
        }

        switch value.Kind() {
        case reflect.String:
            if len(value.String()) > max {
                return &ValidationError{fieldName, fmt.Sprintf("longueur maximale: %d", max)}
            }
        case reflect.Int, reflect.Int32, reflect.Int64:
            if value.Int() > int64(max) {
                return &ValidationError{fieldName, fmt.Sprintf("valeur maximale: %d", max)}
            }
        }
    case "email":
        if value.Kind() == reflect.String && !isValidEmail(value.String()) {
            return &ValidationError{fieldName, "format email invalide"}
        }
    case "phone":
        if value.Kind() == reflect.String && !isValidPhone(value.String()) {
            return &ValidationError{fieldName, "format téléphone invalide"}
        }
    }

    return nil
}

func isZeroValue(v reflect.Value) bool {
    zero := reflect.Zero(v.Type())
    return reflect.DeepEqual(v.Interface(), zero.Interface())
}

func isValidEmail(email string) bool {
    // Validation très simple (dans la vraie vie, utilisez une regex appropriée)
    return strings.Contains(email, "@") && strings.Contains(email, ".")
}

func isValidPhone(phone string) bool {
    // Validation très simple
    return len(phone) >= 10
}

func main() {
    // Test avec des données valides
    validCustomer := Customer{
        Name:  "Alice Dupont",
        Email: "alice@example.com",
        Age:   25,
        Phone: "0123456789",
    }

    errors := validate(validCustomer)
    if len(errors) == 0 {
        fmt.Println("✓ Client valide")
    } else {
        fmt.Println("✗ Erreurs de validation:")
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }

    // Test avec des données invalides
    fmt.Println()
    invalidCustomer := Customer{
        Name:  "A",                    // Trop court
        Email: "email-invalide",       // Pas d'email valide
        Age:   15,                     // Trop jeune
        Phone: "123",                  // Trop court
    }

    errors = validate(invalidCustomer)
    if len(errors) == 0 {
        fmt.Println("✓ Client valide")
    } else {
        fmt.Println("✗ Erreurs de validation:")
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }
}
```

## Tags personnalisés

Vous pouvez créer vos propres tags pour vos besoins spécifiques :

```go
package main

import (
    "fmt"
    "reflect"
    "strings"
)

type Config struct {
    DatabaseURL string `env:"DATABASE_URL" default:"localhost:5432" required:"true"`
    Port        int    `env:"PORT" default:"8080"`
    Debug       bool   `env:"DEBUG" default:"false"`
    ApiKey      string `env:"API_KEY" required:"true"`
}

// Simuler la lecture de variables d'environnement
var mockEnv = map[string]string{
    "DATABASE_URL": "prod-db:5432",
    "PORT":         "3000",
    "DEBUG":        "true",
    // API_KEY est manquant intentionnellement
}

func loadConfig(config interface{}) error {
    v := reflect.ValueOf(config)
    t := reflect.TypeOf(config)

    if v.Kind() != reflect.Ptr || v.Elem().Kind() != reflect.Struct {
        return fmt.Errorf("config doit être un pointeur vers une struct")
    }

    v = v.Elem()
    t = t.Elem()

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := v.Field(i)

        if !value.CanSet() {
            continue
        }

        envKey := field.Tag.Get("env")
        defaultValue := field.Tag.Get("default")
        required := field.Tag.Get("required") == "true"

        // Chercher dans l'environnement (simulé)
        envValue, exists := mockEnv[envKey]

        if !exists {
            if required && defaultValue == "" {
                return fmt.Errorf("variable d'environnement requise manquante: %s", envKey)
            }
            envValue = defaultValue
        }

        if envValue == "" && required {
            return fmt.Errorf("valeur vide pour la variable requise: %s", envKey)
        }

        // Convertir la valeur selon le type
        if err := setFieldValue(value, envValue); err != nil {
            return fmt.Errorf("erreur pour %s: %v", envKey, err)
        }
    }

    return nil
}

func setFieldValue(field reflect.Value, stringValue string) error {
    if stringValue == "" {
        return nil // Garder la valeur zéro
    }

    switch field.Kind() {
    case reflect.String:
        field.SetString(stringValue)
    case reflect.Int, reflect.Int32, reflect.Int64:
        // Pour simplifier, on assume que c'est un entier valide
        if stringValue == "8080" {
            field.SetInt(8080)
        } else if stringValue == "3000" {
            field.SetInt(3000)
        }
    case reflect.Bool:
        field.SetBool(stringValue == "true")
    default:
        return fmt.Errorf("type non supporté: %v", field.Kind())
    }

    return nil
}

func main() {
    config := &Config{}

    err := loadConfig(config)
    if err != nil {
        fmt.Printf("Erreur de configuration: %v\n", err)
        return
    }

    fmt.Printf("Configuration chargée:\n")
    fmt.Printf("  DatabaseURL: %s\n", config.DatabaseURL)
    fmt.Printf("  Port: %d\n", config.Port)
    fmt.Printf("  Debug: %t\n", config.Debug)
    fmt.Printf("  ApiKey: %s\n", config.ApiKey)
}
```

## Outils et techniques avancées

### 1. Parser les tags complexes

```go
package main

import (
    "fmt"
    "reflect"
    "strconv"
    "strings"
)

type FieldInfo struct {
    Name         string
    Type         reflect.Type
    JSONName     string
    JSONOmitEmpty bool
    JSONIgnore   bool
    DBColumn     string
    DBConstraints []string
    Validations  map[string]string
}

func parseStructInfo(s interface{}) []FieldInfo {
    var fields []FieldInfo

    t := reflect.TypeOf(s)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        info := FieldInfo{
            Name:        field.Name,
            Type:        field.Type,
            Validations: make(map[string]string),
        }

        // Parser le tag JSON
        jsonTag := field.Tag.Get("json")
        if jsonTag != "" {
            info.JSONName, info.JSONOmitEmpty, info.JSONIgnore = parseJSONTag(jsonTag)
        } else {
            info.JSONName = strings.ToLower(field.Name)
        }

        // Parser le tag DB
        dbTag := field.Tag.Get("db")
        if dbTag != "" {
            parts := strings.Split(dbTag, ",")
            info.DBColumn = parts[0]
            info.DBConstraints = parts[1:]
        }

        // Parser les validations
        validateTag := field.Tag.Get("validate")
        if validateTag != "" {
            info.Validations = parseValidationTag(validateTag)
        }

        fields = append(fields, info)
    }

    return fields
}

func parseJSONTag(tag string) (name string, omitEmpty bool, ignore bool) {
    if tag == "-" {
        return "", false, true
    }

    parts := strings.Split(tag, ",")
    name = parts[0]

    for _, part := range parts[1:] {
        if part == "omitempty" {
            omitEmpty = true
        }
    }

    return name, omitEmpty, false
}

func parseValidationTag(tag string) map[string]string {
    validations := make(map[string]string)

    rules := strings.Split(tag, ",")
    for _, rule := range rules {
        if equalIndex := strings.Index(rule, "="); equalIndex != -1 {
            key := rule[:equalIndex]
            value := rule[equalIndex+1:]
            validations[key] = value
        } else {
            validations[rule] = "true"
        }
    }

    return validations
}

type Product struct {
    ID          int     `json:"id" db:"id,primarykey" validate:"required"`
    Name        string  `json:"name" db:"name,not null" validate:"required,min=1,max=100"`
    Price       float64 `json:"price" db:"price" validate:"required,min=0.01"`
    Description string  `json:"description,omitempty" db:"description"`
    Hidden      string  `json:"-" db:"internal_notes"`
}

func main() {
    product := Product{}
    fields := parseStructInfo(product)

    fmt.Println("Analyse de la structure Product:")
    fmt.Println(strings.Repeat("=", 50))

    for _, field := range fields {
        fmt.Printf("\nChamp: %s (%s)\n", field.Name, field.Type)
        fmt.Printf("  JSON: %s", field.JSONName)
        if field.JSONOmitEmpty {
            fmt.Printf(" (omitempty)")
        }
        if field.JSONIgnore {
            fmt.Printf(" (ignoré)")
        }
        fmt.Println()

        if field.DBColumn != "" {
            fmt.Printf("  DB: %s", field.DBColumn)
            if len(field.DBConstraints) > 0 {
                fmt.Printf(" [%s]", strings.Join(field.DBConstraints, ", "))
            }
            fmt.Println()
        }

        if len(field.Validations) > 0 {
            fmt.Printf("  Validations: ")
            var validations []string
            for key, value := range field.Validations {
                if value == "true" {
                    validations = append(validations, key)
                } else {
                    validations = append(validations, fmt.Sprintf("%s=%s", key, value))
                }
            }
            fmt.Printf("%s\n", strings.Join(validations, ", "))
        }
    }
}
```

### 2. Cache des informations de réflexion

Pour améliorer les performances, il est courant de mettre en cache les informations de réflexion :

```go
package main

import (
    "fmt"
    "reflect"
    "sync"
)

type StructCache struct {
    mu     sync.RWMutex
    cache  map[reflect.Type][]FieldInfo
}

func NewStructCache() *StructCache {
    return &StructCache{
        cache: make(map[reflect.Type][]FieldInfo),
    }
}

func (sc *StructCache) GetFields(s interface{}) []FieldInfo {
    t := reflect.TypeOf(s)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    // Lecture avec verrou partagé
    sc.mu.RLock()
    if fields, exists := sc.cache[t]; exists {
        sc.mu.RUnlock()
        return fields
    }
    sc.mu.RUnlock()

    // Calcul avec verrou exclusif
    sc.mu.Lock()
    defer sc.mu.Unlock()

    // Vérifier à nouveau (double-checked locking)
    if fields, exists := sc.cache[t]; exists {
        return fields
    }

    // Calculer les informations
    fields := parseStructInfo(s)
    sc.cache[t] = fields

    return fields
}

// Instance globale du cache
var globalCache = NewStructCache()

func GetCachedFields(s interface{}) []FieldInfo {
    return globalCache.GetFields(s)
}

func main() {
    type User struct {
        Name  string `json:"name" validate:"required"`
        Email string `json:"email" validate:"required,email"`
    }

    user := User{}

    // Premier appel - calcule et met en cache
    fmt.Println("Premier appel:")
    fields1 := GetCachedFields(user)
    for _, field := range fields1 {
        fmt.Printf("  %s -> %s\n", field.Name, field.JSONName)
    }

    // Deuxième appel - utilise le cache
    fmt.Println("\nDeuxième appel (depuis le cache):")
    fields2 := GetCachedFields(user)
    for _, field := range fields2 {
        fmt.Printf("  %s -> %s\n", field.Name, field.JSONName)
    }

    fmt.Printf("\nCache size: %d\n", len(globalCache.cache))
}
```

## Bonnes pratiques

### 1. Conventions de nommage

```go
// ✓ Bien - noms cohérents et clairs
type User struct {
    ID        int    `json:"id" db:"user_id"`
    FirstName string `json:"first_name" db:"first_name"`
    LastName  string `json:"last_name" db:"last_name"`
    Email     string `json:"email" db:"email"`
}

// ✗ Éviter - incohérent
type BadUser struct {
    ID   int    `json:"userId" db:"id"`        // Camel vs snake case
    Name string `json:"name" db:"user_name"`   // Incohérent avec le champ
}
```

### 2. Documentation des tags

```go
// Config représente la configuration de l'application
type Config struct {
    // Port d'écoute du serveur
    // Environnement: SERVER_PORT
    // Défaut: 8080
    Port int `env:"SERVER_PORT" default:"8080" validate:"min=1,max=65535"`

    // URL de connexion à la base de données
    // Environnement: DATABASE_URL
    // Requis: oui
    DatabaseURL string `env:"DATABASE_URL" validate:"required,url"`
}
```

### 3. Validation des tags

```go
func validateTags(s interface{}) error {
    t := reflect.TypeOf(s)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)

        // Vérifier que les tags JSON sont bien formés
        jsonTag := field.Tag.Get("json")
        if jsonTag != "" && jsonTag != "-" {
            parts := strings.Split(jsonTag, ",")
            if parts[0] == "" {
                return fmt.Errorf("tag json vide pour le champ %s", field.Name)
            }
        }

        // Vérifier les validations
        validateTag := field.Tag.Get("validate")
        if validateTag != "" {
            rules := strings.Split(validateTag, ",")
            for _, rule := range rules {
                if !isValidRule(rule) {
                    return fmt.Errorf("règle de validation invalide '%s' pour le champ %s", rule, field.Name)
                }
            }
        }
    }

    return nil
}

func isValidRule(rule string) bool {
    validRules := []string{"required", "email", "phone", "url"}
    validPrefixes := []string{"min=", "max=", "len="}

    for _, valid := range validRules {
        if rule == valid {
            return true
        }
    }

    for _, prefix := range validPrefixes {
        if strings.HasPrefix(rule, prefix) {
            return true
        }
    }

    return false
}
```

## Résumé

Les tags de struct sont un mécanisme puissant pour :

- **Configurer la sérialisation** (JSON, XML, etc.)
- **Mapper les champs** vers des colonnes de base de données
- **Définir des règles de validation** sur les données
- **Créer des métadonnées personnalisées** pour vos besoins

**Points clés à retenir :**
- Les tags sont des chaînes entre backticks après le type de champ
- Ils sont accessibles uniquement via la réflexion
- Le format standard est `key:"value,option1,option2"`
- Utilisez un cache pour améliorer les performances
- Documentez vos tags personnalisés
- Validez la syntaxe des tags dans vos tests

Les tags permettent de séparer la logique métier de la configuration, rendant votre code plus propre et plus maintenable. Dans la prochaine section, nous verrons comment utiliser la génération de code pour créer automatiquement du code Go basé sur ces métadonnées.

⏭️
