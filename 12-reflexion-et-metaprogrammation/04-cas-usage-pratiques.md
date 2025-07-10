🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12-4 : Cas d'usage pratiques

## Introduction

Dans cette section, nous allons explorer des cas d'usage réels où la réflexion et la métaprogrammation sont essentielles. Ces exemples montrent comment ces techniques sont utilisées dans des projets Go du monde réel pour résoudre des problèmes concrets.

## 1. Sérialiseur JSON personnalisé

Créons un sérialiseur JSON qui utilise la réflexion pour gérer des types personnalisés et des transformations spéciales.

### Objectif
- Sérialiser des structures avec des transformations personnalisées
- Gérer des types de dates spéciaux
- Appliquer des masques de sécurité
- Supporter des formats de nommage différents

### Implémentation

```go
package main

import (
    "encoding/json"
    "fmt"
    "reflect"
    "strings"
    "time"
)

// Tags personnalisés pour notre sérialiseur
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name" transform:"title"`
    Email     string    `json:"email" mask:"true"`
    Password  string    `json:"-"`
    CreatedAt time.Time `json:"created_at" format:"date"`
    UpdatedAt time.Time `json:"updated_at" format:"datetime"`
    IsAdmin   bool      `json:"is_admin"`
    Metadata  map[string]interface{} `json:"metadata,omitempty"`
}

type CustomSerializer struct {
    dateFormat     string
    datetimeFormat string
}

func NewCustomSerializer() *CustomSerializer {
    return &CustomSerializer{
        dateFormat:     "2006-01-02",
        datetimeFormat: "2006-01-02 15:04:05",
    }
}

func (cs *CustomSerializer) Serialize(v interface{}) ([]byte, error) {
    transformed := cs.transformValue(reflect.ValueOf(v))
    return json.Marshal(transformed.Interface())
}

func (cs *CustomSerializer) transformValue(v reflect.Value) reflect.Value {
    if !v.IsValid() {
        return v
    }

    switch v.Kind() {
    case reflect.Struct:
        return cs.transformStruct(v)
    case reflect.Slice, reflect.Array:
        return cs.transformSlice(v)
    case reflect.Map:
        return cs.transformMap(v)
    case reflect.Ptr:
        if v.IsNil() {
            return reflect.Zero(v.Type())
        }
        elem := cs.transformValue(v.Elem())
        newPtr := reflect.New(elem.Type())
        newPtr.Elem().Set(elem)
        return newPtr
    default:
        return v
    }
}

func (cs *CustomSerializer) transformStruct(v reflect.Value) reflect.Value {
    t := v.Type()
    result := make(map[string]interface{})

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        // Ignorer les champs non exportés
        if !fieldValue.CanInterface() {
            continue
        }

        // Lire les tags
        jsonTag := field.Tag.Get("json")
        if jsonTag == "-" {
            continue
        }

        transformTag := field.Tag.Get("transform")
        maskTag := field.Tag.Get("mask")
        formatTag := field.Tag.Get("format")

        // Déterminer le nom du champ dans le JSON
        fieldName := cs.getJSONFieldName(field.Name, jsonTag)
        if fieldName == "" {
            continue
        }

        // Transformer la valeur
        transformedValue := cs.transformFieldValue(fieldValue, transformTag, maskTag, formatTag)

        // Gérer omitempty
        if cs.shouldOmit(transformedValue, jsonTag) {
            continue
        }

        result[fieldName] = transformedValue
    }

    return reflect.ValueOf(result)
}

func (cs *CustomSerializer) transformSlice(v reflect.Value) reflect.Value {
    if v.Len() == 0 {
        return v
    }

    result := make([]interface{}, v.Len())
    for i := 0; i < v.Len(); i++ {
        transformed := cs.transformValue(v.Index(i))
        result[i] = transformed.Interface()
    }

    return reflect.ValueOf(result)
}

func (cs *CustomSerializer) transformMap(v reflect.Value) reflect.Value {
    if v.Len() == 0 {
        return v
    }

    result := make(map[string]interface{})
    keys := v.MapKeys()

    for _, key := range keys {
        keyStr := fmt.Sprintf("%v", key.Interface())
        value := cs.transformValue(v.MapIndex(key))
        result[keyStr] = value.Interface()
    }

    return reflect.ValueOf(result)
}

func (cs *CustomSerializer) transformFieldValue(v reflect.Value, transform, mask, format string) interface{} {
    value := v.Interface()

    // Appliquer le masque
    if mask == "true" {
        if str, ok := value.(string); ok {
            return cs.maskString(str)
        }
    }

    // Appliquer le format pour les dates
    if t, ok := value.(time.Time); ok && format != "" {
        switch format {
        case "date":
            return t.Format(cs.dateFormat)
        case "datetime":
            return t.Format(cs.datetimeFormat)
        case "unix":
            return t.Unix()
        }
    }

    // Appliquer les transformations de string
    if str, ok := value.(string); ok && transform != "" {
        switch transform {
        case "upper":
            return strings.ToUpper(str)
        case "lower":
            return strings.ToLower(str)
        case "title":
            return strings.Title(str)
        }
    }

    return value
}

func (cs *CustomSerializer) getJSONFieldName(fieldName, jsonTag string) string {
    if jsonTag == "" {
        return strings.ToLower(fieldName)
    }

    parts := strings.Split(jsonTag, ",")
    if parts[0] == "" {
        return strings.ToLower(fieldName)
    }

    return parts[0]
}

func (cs *CustomSerializer) shouldOmit(value interface{}, jsonTag string) bool {
    if !strings.Contains(jsonTag, "omitempty") {
        return false
    }

    v := reflect.ValueOf(value)
    zero := reflect.Zero(v.Type())
    return reflect.DeepEqual(v.Interface(), zero.Interface())
}

func (cs *CustomSerializer) maskString(s string) string {
    if len(s) <= 4 {
        return strings.Repeat("*", len(s))
    }

    // Montrer les 2 premiers et 2 derniers caractères
    return s[:2] + strings.Repeat("*", len(s)-4) + s[len(s)-2:]
}

func main() {
    user := User{
        ID:        1,
        Name:      "alice dupont",
        Email:     "alice@example.com",
        Password:  "secret123",
        CreatedAt: time.Date(2023, 1, 15, 10, 30, 0, 0, time.UTC),
        UpdatedAt: time.Now(),
        IsAdmin:   false,
        Metadata: map[string]interface{}{
            "theme": "dark",
            "lang":  "fr",
        },
    }

    serializer := NewCustomSerializer()

    jsonData, err := serializer.Serialize(user)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("JSON personnalisé:\n%s\n", string(jsonData))

    // Comparaison avec la sérialisation standard
    standardJSON, _ := json.Marshal(user)
    fmt.Printf("\nJSON standard:\n%s\n", string(standardJSON))
}
```

## 2. ORM simple avec réflexion

Créons un ORM basique qui utilise la réflexion pour mapper les structures Go vers des requêtes SQL.

### Objectif
- Générer automatiquement des requêtes SQL
- Mapper les résultats vers des structures
- Gérer les relations simples
- Supporter les transactions

### Implémentation

```go
package main

import (
    "database/sql"
    "fmt"
    "reflect"
    "strings"
    "time"

    _ "github.com/mattn/go-sqlite3" // Driver SQLite
)

type User struct {
    ID        int       `db:"id,primarykey,autoincrement"`
    Name      string    `db:"name,required"`
    Email     string    `db:"email,unique"`
    CreatedAt time.Time `db:"created_at"`
    UpdatedAt time.Time `db:"updated_at"`
}

type Post struct {
    ID       int       `db:"id,primarykey,autoincrement"`
    Title    string    `db:"title,required"`
    Content  string    `db:"content"`
    UserID   int       `db:"user_id,foreignkey"`
    User     *User     `db:"-" relation:"belongs_to,foreign_key:user_id"`
    Created  time.Time `db:"created_at"`
}

type SimpleORM struct {
    db    *sql.DB
    debug bool
}

type ColumnInfo struct {
    Name         string
    Type         string
    IsPrimaryKey bool
    IsRequired   bool
    IsUnique     bool
    IsForeignKey bool
    AutoInc      bool
}

type TableInfo struct {
    Name    string
    Columns map[string]ColumnInfo
}

func NewSimpleORM(db *sql.DB) *SimpleORM {
    return &SimpleORM{
        db:    db,
        debug: true,
    }
}

func (orm *SimpleORM) AutoMigrate(models ...interface{}) error {
    for _, model := range models {
        if err := orm.createTableIfNotExists(model); err != nil {
            return err
        }
    }
    return nil
}

func (orm *SimpleORM) createTableIfNotExists(model interface{}) error {
    tableInfo := orm.analyzeStruct(model)
    sql := orm.generateCreateTableSQL(tableInfo)

    if orm.debug {
        fmt.Printf("Exécution SQL: %s\n", sql)
    }

    _, err := orm.db.Exec(sql)
    return err
}

func (orm *SimpleORM) analyzeStruct(model interface{}) TableInfo {
    t := reflect.TypeOf(model)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    tableName := orm.getTableName(t.Name())
    columns := make(map[string]ColumnInfo)

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        dbTag := field.Tag.Get("db")

        if dbTag == "-" {
            continue
        }

        columnInfo := orm.parseDBTag(field, dbTag)
        if columnInfo.Name != "" {
            columns[columnInfo.Name] = columnInfo
        }
    }

    return TableInfo{
        Name:    tableName,
        Columns: columns,
    }
}

func (orm *SimpleORM) parseDBTag(field reflect.StructField, dbTag string) ColumnInfo {
    parts := strings.Split(dbTag, ",")
    if len(parts) == 0 || parts[0] == "" {
        return ColumnInfo{}
    }

    columnInfo := ColumnInfo{
        Name: parts[0],
        Type: orm.goTypeToSQL(field.Type),
    }

    for _, part := range parts[1:] {
        switch part {
        case "primarykey":
            columnInfo.IsPrimaryKey = true
        case "required":
            columnInfo.IsRequired = true
        case "unique":
            columnInfo.IsUnique = true
        case "autoincrement":
            columnInfo.AutoInc = true
        case "foreignkey":
            columnInfo.IsForeignKey = true
        }
    }

    return columnInfo
}

func (orm *SimpleORM) goTypeToSQL(t reflect.Type) string {
    switch t.Kind() {
    case reflect.Int, reflect.Int32, reflect.Int64:
        return "INTEGER"
    case reflect.String:
        return "TEXT"
    case reflect.Bool:
        return "BOOLEAN"
    case reflect.Float32, reflect.Float64:
        return "REAL"
    case reflect.Struct:
        if t == reflect.TypeOf(time.Time{}) {
            return "DATETIME"
        }
        return "TEXT"
    default:
        return "TEXT"
    }
}

func (orm *SimpleORM) generateCreateTableSQL(tableInfo TableInfo) string {
    var columns []string
    var constraints []string

    for _, columnInfo := range tableInfo.Columns {
        column := fmt.Sprintf("%s %s", columnInfo.Name, columnInfo.Type)

        if columnInfo.IsPrimaryKey {
            column += " PRIMARY KEY"
        }
        if columnInfo.AutoInc {
            column += " AUTOINCREMENT"
        }
        if columnInfo.IsRequired && !columnInfo.IsPrimaryKey {
            column += " NOT NULL"
        }
        if columnInfo.IsUnique && !columnInfo.IsPrimaryKey {
            column += " UNIQUE"
        }

        columns = append(columns, column)
    }

    sql := fmt.Sprintf("CREATE TABLE IF NOT EXISTS %s (\n  %s",
        tableInfo.Name, strings.Join(columns, ",\n  "))

    if len(constraints) > 0 {
        sql += ",\n  " + strings.Join(constraints, ",\n  ")
    }

    sql += "\n);"
    return sql
}

func (orm *SimpleORM) Create(model interface{}) error {
    v := reflect.ValueOf(model)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("model doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    tableName := orm.getTableName(t.Name())
    var columns []string
    var placeholders []string
    var values []interface{}

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        dbTag := field.Tag.Get("db")
        if dbTag == "-" {
            continue
        }

        columnInfo := orm.parseDBTag(field, dbTag)
        if columnInfo.Name == "" || columnInfo.AutoInc {
            continue
        }

        columns = append(columns, columnInfo.Name)
        placeholders = append(placeholders, "?")

        value := fieldValue.Interface()
        if t, ok := value.(time.Time); ok {
            value = t.Format("2006-01-02 15:04:05")
        }
        values = append(values, value)
    }

    sql := fmt.Sprintf("INSERT INTO %s (%s) VALUES (%s)",
        tableName,
        strings.Join(columns, ", "),
        strings.Join(placeholders, ", "))

    if orm.debug {
        fmt.Printf("Exécution SQL: %s\nValeurs: %v\n", sql, values)
    }

    result, err := orm.db.Exec(sql, values...)
    if err != nil {
        return err
    }

    // Récupérer l'ID généré automatiquement
    if id, err := result.LastInsertId(); err == nil {
        orm.setIDField(v, t, id)
    }

    return nil
}

func (orm *SimpleORM) Find(model interface{}, id interface{}) error {
    v := reflect.ValueOf(model)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("model doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    tableName := orm.getTableName(t.Name())

    sql := fmt.Sprintf("SELECT * FROM %s WHERE id = ?", tableName)

    if orm.debug {
        fmt.Printf("Exécution SQL: %s\nID: %v\n", sql, id)
    }

    row := orm.db.QueryRow(sql, id)
    return orm.scanRowIntoStruct(row, v, t)
}

func (orm *SimpleORM) FindAll(models interface{}) error {
    v := reflect.ValueOf(models)
    if v.Kind() != reflect.Ptr || v.Elem().Kind() != reflect.Slice {
        return fmt.Errorf("models doit être un pointeur vers un slice")
    }

    sliceValue := v.Elem()
    sliceType := sliceValue.Type()
    elemType := sliceType.Elem()

    // Gérer les pointeurs dans le slice
    if elemType.Kind() == reflect.Ptr {
        elemType = elemType.Elem()
    }

    tableName := orm.getTableName(elemType.Name())
    sql := fmt.Sprintf("SELECT * FROM %s", tableName)

    if orm.debug {
        fmt.Printf("Exécution SQL: %s\n", sql)
    }

    rows, err := orm.db.Query(sql)
    if err != nil {
        return err
    }
    defer rows.Close()

    // Créer un nouveau slice
    newSlice := reflect.MakeSlice(sliceType, 0, 0)

    for rows.Next() {
        // Créer une nouvelle instance
        elemValue := reflect.New(elemType).Elem()

        if err := orm.scanRowIntoStruct(rows, elemValue, elemType); err != nil {
            return err
        }

        // Ajouter au slice
        if sliceType.Elem().Kind() == reflect.Ptr {
            elemPtr := reflect.New(elemType)
            elemPtr.Elem().Set(elemValue)
            newSlice = reflect.Append(newSlice, elemPtr)
        } else {
            newSlice = reflect.Append(newSlice, elemValue)
        }
    }

    sliceValue.Set(newSlice)
    return nil
}

func (orm *SimpleORM) scanRowIntoStruct(scanner interface{}, v reflect.Value, t reflect.Type) error {
    var columns []string
    var values []interface{}

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        dbTag := field.Tag.Get("db")
        if dbTag == "-" {
            continue
        }

        columnInfo := orm.parseDBTag(field, dbTag)
        if columnInfo.Name == "" {
            continue
        }

        columns = append(columns, columnInfo.Name)
        values = append(values, fieldValue.Addr().Interface())
    }

    // Scanner les valeurs
    if row, ok := scanner.(*sql.Row); ok {
        return row.Scan(values...)
    } else if rows, ok := scanner.(*sql.Rows); ok {
        return rows.Scan(values...)
    }

    return fmt.Errorf("type de scanner non supporté")
}

func (orm *SimpleORM) setIDField(v reflect.Value, t reflect.Type, id int64) {
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        dbTag := field.Tag.Get("db")

        if strings.Contains(dbTag, "primarykey") && strings.Contains(dbTag, "autoincrement") {
            fieldValue := v.Field(i)
            if fieldValue.CanSet() && fieldValue.Kind() == reflect.Int {
                fieldValue.SetInt(id)
            }
            break
        }
    }
}

func (orm *SimpleORM) getTableName(structName string) string {
    // Convertir CamelCase en snake_case et pluraliser
    var result strings.Builder
    for i, r := range structName {
        if i > 0 && r >= 'A' && r <= 'Z' {
            result.WriteRune('_')
        }
        result.WriteRune(r)
    }

    tableName := strings.ToLower(result.String())

    // Pluralisation simple
    if strings.HasSuffix(tableName, "y") {
        tableName = tableName[:len(tableName)-1] + "ies"
    } else if strings.HasSuffix(tableName, "s") {
        tableName += "es"
    } else {
        tableName += "s"
    }

    return tableName
}

func main() {
    // Créer une base de données en mémoire
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        fmt.Printf("Erreur connexion DB: %v\n", err)
        return
    }
    defer db.Close()

    orm := NewSimpleORM(db)

    // Migration automatique
    err = orm.AutoMigrate(User{}, Post{})
    if err != nil {
        fmt.Printf("Erreur migration: %v\n", err)
        return
    }

    // Créer un utilisateur
    user := &User{
        Name:      "Alice Dupont",
        Email:     "alice@example.com",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    err = orm.Create(user)
    if err != nil {
        fmt.Printf("Erreur création utilisateur: %v\n", err)
        return
    }

    fmt.Printf("Utilisateur créé avec ID: %d\n", user.ID)

    // Créer des posts
    posts := []*Post{
        {
            Title:   "Premier article",
            Content: "Contenu du premier article",
            UserID:  user.ID,
            Created: time.Now(),
        },
        {
            Title:   "Deuxième article",
            Content: "Contenu du deuxième article",
            UserID:  user.ID,
            Created: time.Now(),
        },
    }

    for _, post := range posts {
        err = orm.Create(post)
        if err != nil {
            fmt.Printf("Erreur création post: %v\n", err)
            continue
        }
        fmt.Printf("Post créé avec ID: %d\n", post.ID)
    }

    // Récupérer un utilisateur
    var foundUser User
    err = orm.Find(&foundUser, user.ID)
    if err != nil {
        fmt.Printf("Erreur recherche utilisateur: %v\n", err)
        return
    }

    fmt.Printf("Utilisateur trouvé: %+v\n", foundUser)

    // Récupérer tous les posts
    var allPosts []*Post
    err = orm.FindAll(&allPosts)
    if err != nil {
        fmt.Printf("Erreur recherche posts: %v\n", err)
        return
    }

    fmt.Printf("Posts trouvés: %d\n", len(allPosts))
    for _, post := range allPosts {
        fmt.Printf("  - %s (ID: %d, UserID: %d)\n", post.Title, post.ID, post.UserID)
    }
}
```

## 3. Framework de validation avancé

Créons un framework de validation flexible qui utilise la réflexion et les tags.

### Objectif
- Validation basée sur des tags
- Validation conditionnelle
- Messages d'erreur personnalisés
- Validation de structures imbriquées

### Implémentation

```go
package main

import (
    "fmt"
    "reflect"
    "regexp"
    "strconv"
    "strings"
    "time"
)

type ValidationError struct {
    Field   string
    Value   interface{}
    Tag     string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

type ValidationErrors []ValidationError

func (ve ValidationErrors) Error() string {
    var messages []string
    for _, err := range ve {
        messages = append(messages, err.Error())
    }
    return strings.Join(messages, "; ")
}

type Validator struct {
    customValidators map[string]func(interface{}, string) bool
    customMessages   map[string]string
}

func NewValidator() *Validator {
    return &Validator{
        customValidators: make(map[string]func(interface{}, string) bool),
        customMessages:   make(map[string]string),
    }
}

func (v *Validator) RegisterValidator(tag string, fn func(interface{}, string) bool) {
    v.customValidators[tag] = fn
}

func (v *Validator) RegisterMessage(tag string, message string) {
    v.customMessages[tag] = message
}

func (v *Validator) Validate(s interface{}) ValidationErrors {
    return v.validateValue(reflect.ValueOf(s), "")
}

func (v *Validator) validateValue(rv reflect.Value, prefix string) ValidationErrors {
    var errors ValidationErrors

    if rv.Kind() == reflect.Ptr {
        if rv.IsNil() {
            return errors
        }
        rv = rv.Elem()
    }

    if rv.Kind() != reflect.Struct {
        return errors
    }

    rt := rv.Type()

    for i := 0; i < rt.NumField(); i++ {
        field := rt.Field(i)
        fieldValue := rv.Field(i)

        if !fieldValue.CanInterface() {
            continue
        }

        fieldName := field.Name
        if prefix != "" {
            fieldName = prefix + "." + fieldName
        }

        // Validation des structures imbriquées
        if fieldValue.Kind() == reflect.Struct ||
           (fieldValue.Kind() == reflect.Ptr && fieldValue.Type().Elem().Kind() == reflect.Struct) {
            errors = append(errors, v.validateValue(fieldValue, fieldName)...)
        }

        // Validation des slices de structures
        if fieldValue.Kind() == reflect.Slice {
            for j := 0; j < fieldValue.Len(); j++ {
                elemFieldName := fmt.Sprintf("%s[%d]", fieldName, j)
                errors = append(errors, v.validateValue(fieldValue.Index(j), elemFieldName)...)
            }
        }

        // Validation basée sur les tags
        validateTag := field.Tag.Get("validate")
        if validateTag != "" {
            fieldErrors := v.validateField(fieldValue, fieldName, validateTag, field.Tag)
            errors = append(errors, fieldErrors...)
        }
    }

    return errors
}

func (v *Validator) validateField(fieldValue reflect.Value, fieldName, validateTag string, structTag reflect.StructTag) ValidationErrors {
    var errors ValidationErrors

    rules := strings.Split(validateTag, ",")

    for _, rule := range rules {
        rule = strings.TrimSpace(rule)
        if rule == "" {
            continue
        }

        var param string
        if equalIndex := strings.Index(rule, "="); equalIndex != -1 {
            param = rule[equalIndex+1:]
            rule = rule[:equalIndex]
        }

        // Validation conditionnelle
        if conditionTag := structTag.Get("validate_if"); conditionTag != "" {
            if !v.evaluateCondition(fieldValue, conditionTag) {
                continue
            }
        }

        if !v.validateRule(fieldValue, rule, param) {
            message := v.getErrorMessage(rule, param, fieldName)
            errors = append(errors, ValidationError{
                Field:   fieldName,
                Value:   fieldValue.Interface(),
                Tag:     rule,
                Message: message,
            })
        }
    }

    return errors
}

func (v *Validator) validateRule(fieldValue reflect.Value, rule, param string) bool {
    value := fieldValue.Interface()

    // Validations personnalisées
    if customValidator, exists := v.customValidators[rule]; exists {
        return customValidator(value, param)
    }

    switch rule {
    case "required":
        return !v.isEmpty(fieldValue)

    case "min":
        return v.validateMin(fieldValue, param)

    case "max":
        return v.validateMax(fieldValue, param)

    case "len":
        return v.validateLen(fieldValue, param)

    case "email":
        return v.validateEmail(value)

    case "url":
        return v.validateURL(value)

    case "numeric":
        return v.validateNumeric(value)

    case "alpha":
        return v.validateAlpha(value)

    case "alphanum":
        return v.validateAlphaNum(value)

    case "regex":
        return v.validateRegex(value, param)

    case "oneof":
        return v.validateOneOf(value, param)

    case "datetime":
        return v.validateDateTime(value, param)

    case "future":
        return v.validateFuture(value)

    case "past":
        return v.validatePast(value)

    default:
        return true // Règle inconnue, on passe
    }
}

func (v *Validator) isEmpty(fieldValue reflect.Value) bool {
    switch fieldValue.Kind() {
    case reflect.String:
        return fieldValue.String() == ""
    case reflect.Slice, reflect.Map, reflect.Array:
        return fieldValue.Len() == 0
    case reflect.Ptr:
        return fieldValue.IsNil()
    default:
        zero := reflect.Zero(fieldValue.Type())
        return reflect.DeepEqual(fieldValue.Interface(), zero.Interface())
    }
}

func (v *Validator) validateMin(fieldValue reflect.Value, param string) bool {
    minVal, err := strconv.Atoi(param)
    if err != nil {
        return false
    }

    switch fieldValue.Kind() {
    case reflect.String:
        return len(fieldValue.String()) >= minVal
    case reflect.Slice, reflect.Array:
        return fieldValue.Len() >= minVal
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return fieldValue.Int() >= int64(minVal)
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        return fieldValue.Uint() >= uint64(minVal)
    case reflect.Float32, reflect.Float64:
        return fieldValue.Float() >= float64(minVal)
    default:
        return false
    }
}

func (v *Validator) validateMax(fieldValue reflect.Value, param string) bool {
    maxVal, err := strconv.Atoi(param)
    if err != nil {
        return false
    }

    switch fieldValue.Kind() {
    case reflect.String:
        return len(fieldValue.String()) <= maxVal
    case reflect.Slice, reflect.Array:
        return fieldValue.Len() <= maxVal
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return fieldValue.Int() <= int64(maxVal)
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        return fieldValue.Uint() <= uint64(maxVal)
    case reflect.Float32, reflect.Float64:
        return fieldValue.Float() <= float64(maxVal)
    default:
        return false
    }
}

func (v *Validator) validateLen(fieldValue reflect.Value, param string) bool {
    expectedLen, err := strconv.Atoi(param)
    if err != nil {
        return false
    }

    switch fieldValue.Kind() {
    case reflect.String:
        return len(fieldValue.String()) == expectedLen
    case reflect.Slice, reflect.Array:
        return fieldValue.Len() == expectedLen
    default:
        return false
    }
}

func (v *Validator) validateEmail(value interface{}) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return emailRegex.MatchString(str)
}

func (v *Validator) validateURL(value interface{}) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    urlRegex := regexp.MustCompile(`^https?://[^\s/$.?#].[^\s]*$`)
    return urlRegex.MatchString(str)
}

func (v *Validator) validateNumeric(value interface{}) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    _, err := strconv.ParseFloat(str, 64)
    return err == nil
}

func (v *Validator) validateAlpha(value interface{}) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    alphaRegex := regexp.MustCompile(`^[a-zA-Z]+$`)
    return alphaRegex.MatchString(str)
}

func (v *Validator) validateAlphaNum(value interface{}) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    alphaNumRegex := regexp.MustCompile(`^[a-zA-Z0-9]+$`)
    return alphaNumRegex.MatchString(str)
}

func (v *Validator) validateRegex(value interface{}, pattern string) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    regex, err := regexp.Compile(pattern)
    if err != nil {
        return false
    }

    return regex.MatchString(str)
}

func (v *Validator) validateOneOf(value interface{}, options string) bool {
    str := fmt.Sprintf("%v", value)
    validOptions := strings.Split(options, " ")

    for _, option := range validOptions {
        if str == option {
            return true
        }
    }

    return false
}

func (v *Validator) validateDateTime(value interface{}, format string) bool {
    str, ok := value.(string)
    if !ok {
        return false
    }

    if format == "" {
        format = "2006-01-02 15:04:05"
    }

    _, err := time.Parse(format, str)
    return err == nil
}

func (v *Validator) validateFuture(value interface{}) bool {
    switch t := value.(type) {
    case time.Time:
        return t.After(time.Now())
    case string:
        parsedTime, err := time.Parse("2006-01-02", t)
        if err != nil {
            parsedTime, err = time.Parse("2006-01-02 15:04:05", t)
        }
        if err != nil {
            return false
        }
        return parsedTime.After(time.Now())
    default:
        return false
    }
}

func (v *Validator) validatePast(value interface{}) bool {
    switch t := value.(type) {
    case time.Time:
        return t.Before(time.Now())
    case string:
        parsedTime, err := time.Parse("2006-01-02", t)
        if err != nil {
            parsedTime, err = time.Parse("2006-01-02 15:04:05", t)
        }
        if err != nil {
            return false
        }
        return parsedTime.Before(time.Now())
    default:
        return false
    }
}

func (v *Validator) evaluateCondition(fieldValue reflect.Value, condition string) bool {
    // Pour simplifier, on assume que la condition est au format "field=value"
    parts := strings.Split(condition, "=")
    if len(parts) != 2 {
        return true
    }

    // Dans un vrai framework, on implémenterait un parser plus sophistiqué
    return true
}

func (v *Validator) getErrorMessage(rule, param, fieldName string) string {
    if customMessage, exists := v.customMessages[rule]; exists {
        message := strings.ReplaceAll(customMessage, "{field}", fieldName)
        message = strings.ReplaceAll(message, "{param}", param)
        return message
    }

    switch rule {
    case "required":
        return fmt.Sprintf("%s est requis", fieldName)
    case "min":
        return fmt.Sprintf("%s doit être au minimum %s", fieldName, param)
    case "max":
        return fmt.Sprintf("%s ne peut pas dépasser %s", fieldName, param)
    case "len":
        return fmt.Sprintf("%s doit avoir exactement %s caractères", fieldName, param)
    case "email":
        return fmt.Sprintf("%s doit être un email valide", fieldName)
    case "url":
        return fmt.Sprintf("%s doit être une URL valide", fieldName)
    case "numeric":
        return fmt.Sprintf("%s doit être numérique", fieldName)
    case "alpha":
        return fmt.Sprintf("%s ne peut contenir que des lettres", fieldName)
    case "alphanum":
        return fmt.Sprintf("%s ne peut contenir que des lettres et des chiffres", fieldName)
    case "regex":
        return fmt.Sprintf("%s ne respecte pas le format requis", fieldName)
    case "oneof":
        return fmt.Sprintf("%s doit être l'une de ces valeurs: %s", fieldName, param)
    case "datetime":
        return fmt.Sprintf("%s doit être une date/heure valide", fieldName)
    case "future":
        return fmt.Sprintf("%s doit être une date future", fieldName)
    case "past":
        return fmt.Sprintf("%s doit être une date passée", fieldName)
    default:
        return fmt.Sprintf("%s ne respecte pas la règle %s", fieldName, rule)
    }
}

// Structures d'exemple pour les tests
type Address struct {
    Street   string `validate:"required,min=5"`
    City     string `validate:"required,min=2"`
    ZipCode  string `validate:"required,regex=^[0-9]{5}$"`
    Country  string `validate:"required,oneof=France Belgique Suisse"`
}

type User struct {
    Name        string    `validate:"required,min=2,max=50"`
    Email       string    `validate:"required,email"`
    Age         int       `validate:"required,min=18,max=100"`
    Website     string    `validate:"url"`
    PhoneNumber string    `validate:"regex=^[0-9]{10}$"`
    Address     Address   `validate:"required"`
    BirthDate   time.Time `validate:"past"`
    Tags        []string  `validate:"min=1,max=5"`
    IsActive    bool      `validate:"required"`
}

type Event struct {
    Name      string    `validate:"required,min=3"`
    StartDate time.Time `validate:"required,future"`
    EndDate   time.Time `validate:"required,future"`
    MaxGuests int       `validate:"min=1,max=1000"`
}

func main() {
    validator := NewValidator()

    // Enregistrer des validateurs personnalisés
    validator.RegisterValidator("strong_password", func(value interface{}, param string) bool {
        password, ok := value.(string)
        if !ok {
            return false
        }

        // Au moins 8 caractères, une majuscule, une minuscule, un chiffre
        if len(password) < 8 {
            return false
        }

        hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
        hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
        hasDigit := regexp.MustCompile(`[0-9]`).MatchString(password)

        return hasUpper && hasLower && hasDigit
    })

    // Enregistrer des messages personnalisés
    validator.RegisterMessage("strong_password", "{field} doit contenir au moins 8 caractères, une majuscule, une minuscule et un chiffre")
    validator.RegisterMessage("required", "Le champ {field} est obligatoire")
    validator.RegisterMessage("email", "Veuillez saisir un email valide pour {field}")

    // Test avec des données invalides
    fmt.Println("=== Test avec données invalides ===")
    invalidUser := User{
        Name:        "A",  // Trop court
        Email:       "email-invalide",
        Age:         15,   // Trop jeune
        Website:     "pas-une-url",
        PhoneNumber: "123", // Format invalide
        Address: Address{
            Street:  "123",  // Trop court
            City:    "P",    // Trop court
            ZipCode: "abc",  // Format invalide
            Country: "Espagne", // Pas dans la liste
        },
        BirthDate: time.Now().AddDate(0, 0, 1), // Future (invalide pour birthdate)
        Tags:      []string{}, // Vide
        IsActive:  false,
    }

    errors := validator.Validate(invalidUser)
    if len(errors) > 0 {
        fmt.Printf("Erreurs de validation trouvées (%d):\n", len(errors))
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }

    // Test avec des données valides
    fmt.Println("\n=== Test avec données valides ===")
    validUser := User{
        Name:        "Alice Dupont",
        Email:       "alice@example.com",
        Age:         25,
        Website:     "https://alice.example.com",
        PhoneNumber: "0123456789",
        Address: Address{
            Street:  "123 Rue de la Paix",
            City:    "Paris",
            ZipCode: "75001",
            Country: "France",
        },
        BirthDate: time.Date(1998, 5, 15, 0, 0, 0, 0, time.UTC),
        Tags:      []string{"développeur", "go"},
        IsActive:  true,
    }

    errors = validator.Validate(validUser)
    if len(errors) == 0 {
        fmt.Println("✓ Utilisateur valide")
    } else {
        fmt.Printf("Erreurs de validation:\n")
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }

    // Test de validation d'événement
    fmt.Println("\n=== Test de validation d'événement ===")
    event := Event{
        Name:      "Conférence Go",
        StartDate: time.Now().AddDate(0, 1, 0), // Dans un mois
        EndDate:   time.Now().AddDate(0, 1, 2), // Deux jours plus tard
        MaxGuests: 500,
    }

    errors = validator.Validate(event)
    if len(errors) == 0 {
        fmt.Println("✓ Événement valide")
    } else {
        fmt.Printf("Erreurs de validation:\n")
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }

    // Test de performance simple
    fmt.Println("\n=== Test de performance ===")
    start := time.Now()

    for i := 0; i < 1000; i++ {
        validator.Validate(validUser)
    }

    duration := time.Since(start)
    fmt.Printf("1000 validations en %v (%.2f µs par validation)\n",
        duration, float64(duration.Nanoseconds())/1000.0/1000.0)
}
```

## 4. Système de configuration avec réflexion

Créons un système qui charge automatiquement la configuration depuis plusieurs sources.

```go
package main

import (
    "fmt"
    "os"
    "reflect"
    "strconv"
    "strings"
    "time"
)

type Config struct {
    // Configuration serveur
    ServerPort int    `env:"SERVER_PORT" default:"8080" validate:"min=1,max=65535"`
    ServerHost string `env:"SERVER_HOST" default:"localhost"`

    // Configuration base de données
    DBHost     string `env:"DB_HOST" default:"localhost" validate:"required"`
    DBPort     int    `env:"DB_PORT" default:"5432" validate:"min=1,max=65535"`
    DBName     string `env:"DB_NAME" default:"myapp" validate:"required"`
    DBUser     string `env:"DB_USER" default:"postgres" validate:"required"`
    DBPassword string `env:"DB_PASSWORD" validate:"required"`

    // Configuration cache
    RedisURL      string        `env:"REDIS_URL" default:"redis://localhost:6379"`
    CacheTimeout  time.Duration `env:"CACHE_TIMEOUT" default:"5m"`
    CacheEnabled  bool          `env:"CACHE_ENABLED" default:"true"`

    // Configuration application
    AppName     string   `env:"APP_NAME" default:"MyApp"`
    Debug       bool     `env:"DEBUG" default:"false"`
    AllowedIPs  []string `env:"ALLOWED_IPS" default:"127.0.0.1,::1" separator:","`
    MaxFileSize int64    `env:"MAX_FILE_SIZE" default:"10485760"` // 10MB

    // Configuration avancée
    Features map[string]bool `env:"FEATURES" default:"feature1:true,feature2:false" separator:"," kvseparator:":"`
}

type ConfigLoader struct {
    sources []ConfigSource
}

type ConfigSource interface {
    Load(config interface{}) error
}

// Source d'environnement
type EnvSource struct{}

func (e *EnvSource) Load(config interface{}) error {
    return loadFromEnv(config)
}

// Source de fichier (simulé)
type FileSource struct {
    values map[string]string
}

func NewFileSource() *FileSource {
    // Simuler un fichier de configuration
    return &FileSource{
        values: map[string]string{
            "SERVER_PORT": "9000",
            "DEBUG":       "true",
            "APP_NAME":    "ConfigFromFile",
        },
    }
}

func (f *FileSource) Load(config interface{}) error {
    return loadFromMap(config, f.values)
}

// Source par défaut
type DefaultSource struct{}

func (d *DefaultSource) Load(config interface{}) error {
    return loadDefaults(config)
}

func NewConfigLoader() *ConfigLoader {
    return &ConfigLoader{
        sources: []ConfigSource{
            &DefaultSource{},    // Charger d'abord les valeurs par défaut
            &FileSource{},       // Puis le fichier
            &EnvSource{},        // Enfin les variables d'environnement (priorité la plus haute)
        },
    }
}

func (cl *ConfigLoader) Load(config interface{}) error {
    for _, source := range cl.sources {
        if err := source.Load(config); err != nil {
            return err
        }
    }
    return nil
}

func loadDefaults(config interface{}) error {
    v := reflect.ValueOf(config)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("config doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        if !fieldValue.CanSet() {
            continue
        }

        defaultValue := field.Tag.Get("default")
        if defaultValue != "" {
            if err := setFieldValue(fieldValue, defaultValue, field.Tag); err != nil {
                return fmt.Errorf("erreur pour le champ %s: %v", field.Name, err)
            }
        }
    }

    return nil
}

func loadFromEnv(config interface{}) error {
    v := reflect.ValueOf(config)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("config doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        if !fieldValue.CanSet() {
            continue
        }

        envKey := field.Tag.Get("env")
        if envKey == "" {
            continue
        }

        envValue := os.Getenv(envKey)
        if envValue != "" {
            if err := setFieldValue(fieldValue, envValue, field.Tag); err != nil {
                return fmt.Errorf("erreur pour %s: %v", envKey, err)
            }
        }
    }

    return nil
}

func loadFromMap(config interface{}, values map[string]string) error {
    v := reflect.ValueOf(config)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("config doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        if !fieldValue.CanSet() {
            continue
        }

        envKey := field.Tag.Get("env")
        if envKey == "" {
            continue
        }

        if value, exists := values[envKey]; exists {
            if err := setFieldValue(fieldValue, value, field.Tag); err != nil {
                return fmt.Errorf("erreur pour %s: %v", envKey, err)
            }
        }
    }

    return nil
}

func setFieldValue(fieldValue reflect.Value, stringValue string, tag reflect.StructTag) error {
    separator := tag.Get("separator")
    kvSeparator := tag.Get("kvseparator")

    switch fieldValue.Kind() {
    case reflect.String:
        fieldValue.SetString(stringValue)

    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        if fieldValue.Type() == reflect.TypeOf(time.Duration(0)) {
            // Gérer les durées
            duration, err := time.ParseDuration(stringValue)
            if err != nil {
                return err
            }
            fieldValue.Set(reflect.ValueOf(duration))
        } else {
            intValue, err := strconv.ParseInt(stringValue, 10, 64)
            if err != nil {
                return err
            }
            fieldValue.SetInt(intValue)
        }

    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        uintValue, err := strconv.ParseUint(stringValue, 10, 64)
        if err != nil {
            return err
        }
        fieldValue.SetUint(uintValue)

    case reflect.Float32, reflect.Float64:
        floatValue, err := strconv.ParseFloat(stringValue, 64)
        if err != nil {
            return err
        }
        fieldValue.SetFloat(floatValue)

    case reflect.Bool:
        boolValue, err := strconv.ParseBool(stringValue)
        if err != nil {
            return err
        }
        fieldValue.SetBool(boolValue)

    case reflect.Slice:
        if separator == "" {
            separator = ","
        }
        return setSliceValue(fieldValue, stringValue, separator)

    case reflect.Map:
        if separator == "" {
            separator = ","
        }
        if kvSeparator == "" {
            kvSeparator = ":"
        }
        return setMapValue(fieldValue, stringValue, separator, kvSeparator)

    default:
        return fmt.Errorf("type non supporté: %v", fieldValue.Kind())
    }

    return nil
}

func setSliceValue(fieldValue reflect.Value, stringValue, separator string) error {
    if stringValue == "" {
        return nil
    }

    parts := strings.Split(stringValue, separator)
    elemType := fieldValue.Type().Elem()

    slice := reflect.MakeSlice(fieldValue.Type(), len(parts), len(parts))

    for i, part := range parts {
        part = strings.TrimSpace(part)
        elemValue := slice.Index(i)

        switch elemType.Kind() {
        case reflect.String:
            elemValue.SetString(part)
        case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
            intValue, err := strconv.ParseInt(part, 10, 64)
            if err != nil {
                return err
            }
            elemValue.SetInt(intValue)
        default:
            return fmt.Errorf("type d'élément de slice non supporté: %v", elemType.Kind())
        }
    }

    fieldValue.Set(slice)
    return nil
}

func setMapValue(fieldValue reflect.Value, stringValue, separator, kvSeparator string) error {
    if stringValue == "" {
        return nil
    }

    pairs := strings.Split(stringValue, separator)
    mapType := fieldValue.Type()
    keyType := mapType.Key()
    valueType := mapType.Elem()

    newMap := reflect.MakeMap(mapType)

    for _, pair := range pairs {
        pair = strings.TrimSpace(pair)
        parts := strings.Split(pair, kvSeparator)
        if len(parts) != 2 {
            continue
        }

        key := strings.TrimSpace(parts[0])
        value := strings.TrimSpace(parts[1])

        // Créer la clé
        keyValue := reflect.New(keyType).Elem()
        if keyType.Kind() == reflect.String {
            keyValue.SetString(key)
        } else {
            return fmt.Errorf("type de clé de map non supporté: %v", keyType.Kind())
        }

        // Créer la valeur
        mapValueReflect := reflect.New(valueType).Elem()
        switch valueType.Kind() {
        case reflect.String:
            mapValueReflect.SetString(value)
        case reflect.Bool:
            boolValue, err := strconv.ParseBool(value)
            if err != nil {
                return err
            }
            mapValueReflect.SetBool(boolValue)
        case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
            intValue, err := strconv.ParseInt(value, 10, 64)
            if err != nil {
                return err
            }
            mapValueReflect.SetInt(intValue)
        default:
            return fmt.Errorf("type de valeur de map non supporté: %v", valueType.Kind())
        }

        newMap.SetMapIndex(keyValue, mapValueReflect)
    }

    fieldValue.Set(newMap)
    return nil
}

func printConfig(config interface{}) {
    v := reflect.ValueOf(config)
    if v.Kind() == reflect.Ptr {
        v = v.Elem()
    }
    t := v.Type()

    fmt.Println("Configuration actuelle:")
    fmt.Println(strings.Repeat("=", 50))

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        envKey := field.Tag.Get("env")
        defaultValue := field.Tag.Get("default")

        fmt.Printf("%-15s: %v", field.Name, fieldValue.Interface())

        if envKey != "" {
            fmt.Printf(" (env: %s)", envKey)
        }
        if defaultValue != "" {
            fmt.Printf(" (default: %s)", defaultValue)
        }

        fmt.Println()
    }
}

func main() {
    // Simuler quelques variables d'environnement
    os.Setenv("DB_PASSWORD", "secretpassword")
    os.Setenv("DEBUG", "true")
    os.Setenv("ALLOWED_IPS", "192.168.1.1,192.168.1.2,10.0.0.1")
    os.Setenv("FEATURES", "auth:true,analytics:false,newui:true")

    config := &Config{}
    loader := NewConfigLoader()

    err := loader.Load(config)
    if err != nil {
        fmt.Printf("Erreur de chargement de la configuration: %v\n", err)
        return
    }

    printConfig(config)

    // Test de validation
    fmt.Println("\n" + strings.Repeat("=", 50))
    validator := NewValidator()

    errors := validator.Validate(config)
    if len(errors) == 0 {
        fmt.Println("✓ Configuration valide")
    } else {
        fmt.Printf("Erreurs de validation de la configuration (%d):\n", len(errors))
        for _, err := range errors {
            fmt.Printf("  - %s\n", err.Error())
        }
    }

    // Afficher des informations supplémentaires
    fmt.Println("\nInformations détaillées:")
    fmt.Printf("  - Serveur: %s:%d\n", config.ServerHost, config.ServerPort)
    fmt.Printf("  - Base de données: %s@%s:%d/%s\n",
        config.DBUser, config.DBHost, config.DBPort, config.DBName)
    fmt.Printf("  - Cache: %s (enabled: %t, timeout: %v)\n",
        config.RedisURL, config.CacheEnabled, config.CacheTimeout)
    fmt.Printf("  - IPs autorisées: %v\n", config.AllowedIPs)
    fmt.Printf("  - Fonctionnalités:\n")
    for feature, enabled := range config.Features {
        fmt.Printf("    * %s: %t\n", feature, enabled)
    }
}
```

## Résumé des cas d'usage pratiques

Ces exemples montrent comment la réflexion et la métaprogrammation sont utilisées dans des scenarios réels :

### 1. **Sérialiseur JSON personnalisé**
- **Utilisation** : Transformation de données avant sérialisation
- **Techniques** : Réflexion sur structures, tags personnalisés, transformation conditionnelle
- **Avantages** : Flexibilité, sécurité (masquage), formats personnalisés

### 2. **ORM simple**
- **Utilisation** : Mapping objet-relationnel automatique
- **Techniques** : Analyse de structures, génération SQL, binding de résultats
- **Avantages** : Productivité, cohérence, réduction du code boilerplate

### 3. **Framework de validation**
- **Utilisation** : Validation de données basée sur des règles
- **Techniques** : Tags de validation, règles extensibles, messages personnalisés
- **Avantages** : Flexibilité, réutilisabilité, validation complexe

### 4. **Système de configuration**
- **Utilisation** : Chargement automatique depuis multiple sources
- **Techniques** : Sources multiples, conversion de types, validation
- **Avantages** : Configuration centralisée, validation automatique, flexibilité

### **Points clés à retenir :**

- **Performance** : La réflexion a un coût, utilisez un cache quand possible
- **Sécurité des types** : Ajoutez toujours une validation appropriée
- **Simplicité** : Cachez la complexité derrière des APIs simples
- **Tests** : Testez exhaustivement car les erreurs surviennent à l'exécution
- **Documentation** : Documentez les tags et conventions utilisés

Ces techniques permettent de créer des bibliothèques puissantes et flexibles qui simplifient le développement tout en maintenant la sécurité des types.

## 5. Système d'injection de dépendances

Créons un injecteur de dépendances simple qui utilise la réflexion pour résoudre automatiquement les dépendances.

```go
package main

import (
    "fmt"
    "reflect"
    "strings"
)

// Container pour l'injection de dépendances
type DIContainer struct {
    services   map[reflect.Type]interface{}
    factories  map[reflect.Type]func() interface{}
    singletons map[reflect.Type]interface{}
}

func NewDIContainer() *DIContainer {
    return &DIContainer{
        services:   make(map[reflect.Type]interface{}),
        factories:  make(map[reflect.Type]func() interface{}),
        singletons: make(map[reflect.Type]interface{}),
    }
}

// Enregistrer une instance
func (c *DIContainer) Register(service interface{}) {
    t := reflect.TypeOf(service)
    c.services[t] = service
}

// Enregistrer une factory
func (c *DIContainer) RegisterFactory(serviceType interface{}, factory func() interface{}) {
    t := reflect.TypeOf(serviceType).Elem() // Pour les interfaces
    c.factories[t] = factory
}

// Enregistrer un singleton
func (c *DIContainer) RegisterSingleton(serviceType interface{}, factory func() interface{}) {
    t := reflect.TypeOf(serviceType).Elem()
    c.factories[t] = factory
}

// Résoudre une dépendance
func (c *DIContainer) Resolve(serviceType interface{}) (interface{}, error) {
    t := reflect.TypeOf(serviceType).Elem()
    return c.resolveType(t)
}

func (c *DIContainer) resolveType(t reflect.Type) (interface{}, error) {
    // Vérifier si c'est un singleton déjà créé
    if singleton, exists := c.singletons[t]; exists {
        return singleton, nil
    }

    // Vérifier si c'est un service enregistré
    if service, exists := c.services[t]; exists {
        return service, nil
    }

    // Vérifier si c'est une factory
    if factory, exists := c.factories[t]; exists {
        instance := factory()

        // Si c'est un singleton, le stocker
        c.singletons[t] = instance
        return instance, nil
    }

    // Essayer de créer automatiquement
    if t.Kind() == reflect.Struct {
        return c.createInstance(t)
    }

    return nil, fmt.Errorf("impossible de résoudre le type %v", t)
}

func (c *DIContainer) createInstance(t reflect.Type) (interface{}, error) {
    // Créer une nouvelle instance
    instance := reflect.New(t).Elem()

    // Injecter les dépendances
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := instance.Field(i)

        injectTag := field.Tag.Get("inject")
        if injectTag == "" && !field.Anonymous {
            continue
        }

        if !fieldValue.CanSet() {
            continue
        }

        // Résoudre la dépendance
        dependency, err := c.resolveType(field.Type)
        if err != nil {
            if injectTag == "optional" {
                continue
            }
            return nil, fmt.Errorf("impossible d'injecter %s.%s: %v",
                t.Name(), field.Name, err)
        }

        fieldValue.Set(reflect.ValueOf(dependency))
    }

    return instance.Interface(), nil
}

// Auto-wire un objet existant
func (c *DIContainer) AutoWire(target interface{}) error {
    v := reflect.ValueOf(target)
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("target doit être un pointeur")
    }

    v = v.Elem()
    t := v.Type()

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fieldValue := v.Field(i)

        injectTag := field.Tag.Get("inject")
        if injectTag == "" {
            continue
        }

        if !fieldValue.CanSet() {
            continue
        }

        dependency, err := c.resolveType(field.Type)
        if err != nil {
            if injectTag == "optional" {
                continue
            }
            return fmt.Errorf("impossible d'injecter %s: %v", field.Name, err)
        }

        fieldValue.Set(reflect.ValueOf(dependency))
    }

    return nil
}

// Exemples de services
type Logger interface {
    Log(message string)
}

type ConsoleLogger struct{}

func (cl *ConsoleLogger) Log(message string) {
    fmt.Printf("[LOG] %s\n", message)
}

type FileLogger struct {
    filename string
}

func (fl *FileLogger) Log(message string) {
    fmt.Printf("[FILE:%s] %s\n", fl.filename, message)
}

type Database interface {
    Query(sql string) []map[string]interface{}
}

type PostgresDB struct {
    connectionString string
}

func (db *PostgresDB) Query(sql string) []map[string]interface{} {
    fmt.Printf("[DB] Executing: %s\n", sql)
    return []map[string]interface{}{
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
    }
}

type EmailService interface {
    SendEmail(to, subject, body string) error
}

type SMTPEmailService struct {
    Logger `inject:""`
    host   string
    port   int
}

func (es *SMTPEmailService) SendEmail(to, subject, body string) error {
    es.Logger.Log(fmt.Sprintf("Sending email to %s: %s", to, subject))
    fmt.Printf("[EMAIL] To: %s, Subject: %s\n", to, subject)
    return nil
}

type UserService struct {
    DB     Database     `inject:""`
    Logger Logger       `inject:""`
    Email  EmailService `inject:"optional"`
}

func (us *UserService) CreateUser(name, email string) {
    us.Logger.Log(fmt.Sprintf("Creating user: %s", name))

    // Simuler une requête DB
    us.DB.Query(fmt.Sprintf("INSERT INTO users (name, email) VALUES ('%s', '%s')", name, email))

    // Envoyer un email de bienvenue si le service est disponible
    if us.Email != nil {
        us.Email.SendEmail(email, "Bienvenue!", "Votre compte a été créé avec succès.")
    }

    us.Logger.Log("User created successfully")
}

func (us *UserService) GetUsers() []map[string]interface{} {
    us.Logger.Log("Fetching all users")
    return us.DB.Query("SELECT * FROM users")
}

func main() {
    container := NewDIContainer()

    // Enregistrer les services
    container.Register(&ConsoleLogger{})
    container.Register(&PostgresDB{connectionString: "postgres://localhost/mydb"})

    // Enregistrer l'email service avec une factory
    container.RegisterFactory((*EmailService)(nil), func() interface{} {
        emailService := &SMTPEmailService{
            host: "smtp.example.com",
            port: 587,
        }
        // Auto-wire les dépendances de l'email service
        container.AutoWire(emailService)
        return emailService
    })

    // Résoudre le UserService (création automatique avec injection)
    userServiceInterface, err := container.Resolve((*UserService)(nil))
    if err != nil {
        fmt.Printf("Erreur de résolution: %v\n", err)
        return
    }

    userService := userServiceInterface.(*UserService)

    // Utiliser le service
    fmt.Println("=== Test du UserService ===")
    userService.CreateUser("Alice Dupont", "alice@example.com")

    users := userService.GetUsers()
    fmt.Printf("Utilisateurs trouvés: %d\n", len(users))
    for _, user := range users {
        fmt.Printf("  - %v\n", user)
    }
}
```

## 6. Système de mapping automatique entre structures

Créons un mapper qui convertit automatiquement entre différentes structures.

```go
package main

import (
    "fmt"
    "reflect"
    "strings"
    "time"
)

type Mapper struct {
    customConverters map[string]func(interface{}) interface{}
    namingStrategy   func(string) string
}

func NewMapper() *Mapper {
    return &Mapper{
        customConverters: make(map[string]func(interface{}) interface{}),
        namingStrategy:   func(name string) string { return name }, // Par défaut: identité
    }
}

// Enregistrer un convertisseur personnalisé
func (m *Mapper) RegisterConverter(fromType, toType reflect.Type, converter func(interface{}) interface{}) {
    key := fmt.Sprintf("%s->%s", fromType.String(), toType.String())
    m.customConverters[key] = converter
}

// Définir une stratégie de nommage
func (m *Mapper) SetNamingStrategy(strategy func(string) string) {
    m.namingStrategy = strategy
}

// Mapper une structure vers une autre
func (m *Mapper) Map(source, destination interface{}) error {
    sourceValue := reflect.ValueOf(source)
    destValue := reflect.ValueOf(destination)

    if destValue.Kind() != reflect.Ptr {
        return fmt.Errorf("destination doit être un pointeur")
    }

    destValue = destValue.Elem()

    return m.mapValue(sourceValue, destValue)
}

func (m *Mapper) mapValue(source, dest reflect.Value) error {
    // Gérer les pointeurs
    if source.Kind() == reflect.Ptr {
        if source.IsNil() {
            return nil
        }
        source = source.Elem()
    }

    if dest.Kind() == reflect.Ptr {
        if dest.IsNil() {
            dest.Set(reflect.New(dest.Type().Elem()))
        }
        dest = dest.Elem()
    }

    sourceType := source.Type()
    destType := dest.Type()

    // Vérifier s'il y a un convertisseur personnalisé
    converterKey := fmt.Sprintf("%s->%s", sourceType.String(), destType.String())
    if converter, exists := m.customConverters[converterKey]; exists {
        converted := converter(source.Interface())
        dest.Set(reflect.ValueOf(converted))
        return nil
    }

    // Mapping direct si même type
    if sourceType == destType {
        dest.Set(source)
        return nil
    }

    // Mapping par types
    switch dest.Kind() {
    case reflect.Struct:
        return m.mapStruct(source, dest)
    case reflect.Slice:
        return m.mapSlice(source, dest)
    case reflect.Map:
        return m.mapMap(source, dest)
    default:
        return m.mapPrimitive(source, dest)
    }
}

func (m *Mapper) mapStruct(source, dest reflect.Value) error {
    sourceType := source.Type()
    destType := dest.Type()

    // Créer une map des champs source pour un accès rapide
    sourceFields := make(map[string]reflect.Value)

    if source.Kind() == reflect.Struct {
        for i := 0; i < sourceType.NumField(); i++ {
            field := sourceType.Field(i)
            fieldValue := source.Field(i)

            // Utiliser le tag de mapping s'il existe
            mappingName := field.Tag.Get("map")
            if mappingName == "" {
                mappingName = field.Name
            }

            sourceFields[mappingName] = fieldValue
            sourceFields[m.namingStrategy(field.Name)] = fieldValue
        }
    }

    // Mapper vers les champs de destination
    for i := 0; i < destType.NumField(); i++ {
        destField := destType.Field(i)
        destFieldValue := dest.Field(i)

        if !destFieldValue.CanSet() {
            continue
        }

        // Ignorer les champs avec le tag ignore
        if destField.Tag.Get("map") == "-" {
            continue
        }

        // Déterminer le nom du champ source
        sourceName := destField.Tag.Get("map")
        if sourceName == "" {
            sourceName = destField.Name
        }

        // Chercher le champ source
        sourceFieldValue, found := sourceFields[sourceName]
        if !found {
            sourceFieldValue, found = sourceFields[m.namingStrategy(destField.Name)]
        }

        if !found {
            continue
        }

        // Mapper la valeur
        if err := m.mapValue(sourceFieldValue, destFieldValue); err != nil {
            return fmt.Errorf("erreur mapping %s: %v", destField.Name, err)
        }
    }

    return nil
}

func (m *Mapper) mapSlice(source, dest reflect.Value) error {
    if source.Kind() != reflect.Slice && source.Kind() != reflect.Array {
        return fmt.Errorf("source n'est pas un slice/array")
    }

    sourceLen := source.Len()
    destSlice := reflect.MakeSlice(dest.Type(), sourceLen, sourceLen)

    for i := 0; i < sourceLen; i++ {
        sourceElem := source.Index(i)
        destElem := destSlice.Index(i)

        if err := m.mapValue(sourceElem, destElem); err != nil {
            return fmt.Errorf("erreur mapping élément %d: %v", i, err)
        }
    }

    dest.Set(destSlice)
    return nil
}

func (m *Mapper) mapMap(source, dest reflect.Value) error {
    if source.Kind() != reflect.Map {
        return fmt.Errorf("source n'est pas une map")
    }

    destMap := reflect.MakeMap(dest.Type())
    keys := source.MapKeys()

    for _, key := range keys {
        sourceValue := source.MapIndex(key)

        // Créer une nouvelle valeur pour la destination
        destValue := reflect.New(dest.Type().Elem()).Elem()

        if err := m.mapValue(sourceValue, destValue); err != nil {
            return fmt.Errorf("erreur mapping valeur map: %v", err)
        }

        destMap.SetMapIndex(key, destValue)
    }

    dest.Set(destMap)
    return nil
}

func (m *Mapper) mapPrimitive(source, dest reflect.Value) error {
    sourceKind := source.Kind()
    destKind := dest.Kind()

    // Conversions directes possibles
    if source.Type().ConvertibleTo(dest.Type()) {
        dest.Set(source.Convert(dest.Type()))
        return nil
    }

    // Conversions spéciales
    switch {
    case sourceKind == reflect.String && destKind == reflect.String:
        dest.SetString(source.String())
    case (sourceKind >= reflect.Int && sourceKind <= reflect.Int64) &&
         (destKind >= reflect.Int && destKind <= reflect.Int64):
        dest.SetInt(source.Int())
    case (sourceKind >= reflect.Uint && sourceKind <= reflect.Uint64) &&
         (destKind >= reflect.Uint && destKind <= reflect.Uint64):
        dest.SetUint(source.Uint())
    case (sourceKind == reflect.Float32 || sourceKind == reflect.Float64) &&
         (destKind == reflect.Float32 || destKind == reflect.Float64):
        dest.SetFloat(source.Float())
    case sourceKind == reflect.Bool && destKind == reflect.Bool:
        dest.SetBool(source.Bool())
    default:
        return fmt.Errorf("impossible de convertir %v vers %v", sourceKind, destKind)
    }

    return nil
}

// Structures d'exemple
type UserEntity struct {
    ID        int       `map:"user_id"`
    FirstName string    `map:"first_name"`
    LastName  string    `map:"last_name"`
    Email     string    `map:"email"`
    CreatedAt time.Time `map:"created_at"`
    UpdatedAt time.Time `map:"updated_at"`
    IsActive  bool      `map:"is_active"`
    Profile   Profile   `map:"profile"`
}

type Profile struct {
    Bio     string `map:"biography"`
    Website string `map:"website"`
    Avatar  string `map:"avatar_url"`
}

type UserDTO struct {
    UserID      int       `map:"user_id"`
    FullName    string    `map:"-"` // Sera calculé manuellement
    Email       string    `map:"email"`
    DateCreated time.Time `map:"created_at"`
    Active      bool      `map:"is_active"`
    ProfileInfo ProfileDTO `map:"profile"`
}

type ProfileDTO struct {
    Description string `map:"biography"`
    Website     string `map:"website"`
    ProfilePic  string `map:"avatar_url"`
}

type UserResponse struct {
    ID       int    `json:"id"`
    Name     string `json:"name"`
    Email    string `json:"email"`
    Active   bool   `json:"active"`
    Created  string `json:"created"` // Format string pour JSON
}

func main() {
    mapper := NewMapper()

    // Stratégie de nommage snake_case vers camelCase
    mapper.SetNamingStrategy(func(name string) string {
        parts := strings.Split(strings.ToLower(name), "_")
        if len(parts) <= 1 {
            return name
        }

        result := parts[0]
        for _, part := range parts[1:] {
            if len(part) > 0 {
                result += strings.ToUpper(part[:1]) + part[1:]
            }
        }
        return result
    })

    // Convertisseur personnalisé pour calculer le nom complet
    mapper.RegisterConverter(
        reflect.TypeOf(UserEntity{}),
        reflect.TypeOf(UserDTO{}),
        func(source interface{}) interface{} {
            user := source.(UserEntity)
            dto := UserDTO{}

            // Mapping automatique d'abord
            mapper.Map(user, &dto)

            // Puis calcul du nom complet
            dto.FullName = fmt.Sprintf("%s %s", user.FirstName, user.LastName)

            return dto
        },
    )

    // Convertisseur pour le format de réponse
    mapper.RegisterConverter(
        reflect.TypeOf(UserDTO{}),
        reflect.TypeOf(UserResponse{}),
        func(source interface{}) interface{} {
            dto := source.(UserDTO)
            return UserResponse{
                ID:      dto.UserID,
                Name:    dto.FullName,
                Email:   dto.Email,
                Active:  dto.Active,
                Created: dto.DateCreated.Format("2006-01-02"),
            }
        },
    )

    // Données de test
    user := UserEntity{
        ID:        1,
        FirstName: "Alice",
        LastName:  "Dupont",
        Email:     "alice@example.com",
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
        IsActive:  true,
        Profile: Profile{
            Bio:     "Développeuse Go passionnée",
            Website: "https://alice.dev",
            Avatar:  "https://example.com/avatar.jpg",
        },
    }

    // Test de mapping Entity -> DTO
    fmt.Println("=== Mapping Entity -> DTO ===")
    var userDTO UserDTO
    err := mapper.Map(user, &userDTO)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("DTO: %+v\n", userDTO)
    fmt.Printf("Profile: %+v\n", userDTO.ProfileInfo)

    // Test de mapping DTO -> Response
    fmt.Println("\n=== Mapping DTO -> Response ===")
    var response UserResponse
    err = mapper.Map(userDTO, &response)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("Response: %+v\n", response)

    // Test avec des slices
    fmt.Println("\n=== Mapping avec des slices ===")
    users := []UserEntity{user, user}
    var userDTOs []UserDTO

    err = mapper.Map(users, &userDTOs)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    fmt.Printf("DTOs mappés: %d\n", len(userDTOs))
    for i, dto := range userDTOs {
        fmt.Printf("  [%d] %s (%s)\n", i, dto.FullName, dto.Email)
    }
}
```

## 7. Système de cache avec réflexion

Créons un système de cache intelligent qui utilise la réflexion pour créer des clés automatiquement.

```go
package main

import (
    "crypto/md5"
    "encoding/json"
    "fmt"
    "reflect"
    "strconv"
    "strings"
    "sync"
    "time"
)

type CacheEntry struct {
    Value     interface{}
    ExpiresAt time.Time
    CreatedAt time.Time
}

func (ce *CacheEntry) IsExpired() bool {
    return time.Now().After(ce.ExpiresAt)
}

type Cache struct {
    mu      sync.RWMutex
    entries map[string]*CacheEntry
    ttl     time.Duration
}

func NewCache(ttl time.Duration) *Cache {
    cache := &Cache{
        entries: make(map[string]*CacheEntry),
        ttl:     ttl,
    }

    // Nettoyage périodique
    go cache.cleanup()

    return cache
}

func (c *Cache) cleanup() {
    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        c.mu.Lock()
        for key, entry := range c.entries {
            if entry.IsExpired() {
                delete(c.entries, key)
            }
        }
        c.mu.Unlock()
    }
}

// Générer une clé basée sur la fonction et ses paramètres
func (c *Cache) generateKey(funcName string, args ...interface{}) string {
    var keyParts []string
    keyParts = append(keyParts, funcName)

    for _, arg := range args {
        keyParts = append(keyParts, c.serializeArg(arg))
    }

    key := strings.Join(keyParts, ":")

    // Hacher si trop long
    if len(key) > 100 {
        hash := md5.Sum([]byte(key))
        return fmt.Sprintf("hash:%x", hash)
    }

    return key
}

func (c *Cache) serializeArg(arg interface{}) string {
    if arg == nil {
        return "nil"
    }

    v := reflect.ValueOf(arg)

    switch v.Kind() {
    case reflect.String:
        return v.String()
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return strconv.FormatInt(v.Int(), 10)
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        return strconv.FormatUint(v.Uint(), 10)
    case reflect.Float32, reflect.Float64:
        return strconv.FormatFloat(v.Float(), 'f', -1, 64)
    case reflect.Bool:
        return strconv.FormatBool(v.Bool())
    case reflect.Struct, reflect.Slice, reflect.Map:
        // Sérialiser en JSON pour les types complexes
        if jsonBytes, err := json.Marshal(arg); err == nil {
            return string(jsonBytes)
        }
        return fmt.Sprintf("%+v", arg)
    case reflect.Ptr:
        if v.IsNil() {
            return "nil"
        }
        return c.serializeArg(v.Elem().Interface())
    default:
        return fmt.Sprintf("%+v", arg)
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    entry, exists := c.entries[key]
    if !exists || entry.IsExpired() {
        return nil, false
    }

    return entry.Value, true
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.entries[key] = &CacheEntry{
        Value:     value,
        ExpiresAt: time.Now().Add(c.ttl),
        CreatedAt: time.Now(),
    }
}

// Cache avec décorateur de fonction
type CachedFunction struct {
    cache    *Cache
    funcName string
    fn       reflect.Value
}

func (c *Cache) WrapFunction(name string, fn interface{}) *CachedFunction {
    fnValue := reflect.ValueOf(fn)
    if fnValue.Kind() != reflect.Func {
        panic("fn doit être une fonction")
    }

    return &CachedFunction{
        cache:    c,
        funcName: name,
        fn:       fnValue,
    }
}

func (cf *CachedFunction) Call(args ...interface{}) []interface{} {
    // Générer la clé
    key := cf.cache.generateKey(cf.funcName, args...)

    // Vérifier le cache
    if cached, found := cf.cache.Get(key); found {
        fmt.Printf("[CACHE HIT] %s\n", key)
        return cached.([]interface{})
    }

    fmt.Printf("[CACHE MISS] %s\n", key)

    // Convertir les arguments en reflect.Value
    argValues := make([]reflect.Value, len(args))
    for i, arg := range args {
        argValues[i] = reflect.ValueOf(arg)
    }

    // Appeler la fonction
    results := cf.fn.Call(argValues)

    // Convertir les résultats
    resultInterfaces := make([]interface{}, len(results))
    for i, result := range results {
        resultInterfaces[i] = result.Interface()
    }

    // Mettre en cache
    cf.cache.Set(key, resultInterfaces)

    return resultInterfaces
}

// Décorateur automatique avec réflexion
func (c *Cache) DecorateStruct(target interface{}) interface{} {
    targetValue := reflect.ValueOf(target)
    targetType := reflect.TypeOf(target)

    if targetType.Kind() == reflect.Ptr {
        targetValue = targetValue.Elem()
        targetType = targetType.Elem()
    }

    // Créer une nouvelle structure avec des méthodes cachées
    newStruct := reflect.New(targetType).Elem()
    newStruct.Set(targetValue)

    // Pour chaque méthode, créer une version cachée
    numMethods := targetType.NumMethod()
    for i := 0; i < numMethods; i++ {
        method := targetType.Method(i)

        // Vérifier si la méthode doit être cachée
        if strings.HasPrefix(method.Name, "Cache") {
            continue
        }

        // Créer une version cachée
        c.wrapMethod(newStruct.Addr(), method)
    }

    return newStruct.Interface()
}

func (c *Cache) wrapMethod(target reflect.Value, method reflect.Method) {
    // Cette implémentation est simplifiée
    // Dans un vrai système, on utiliserait des techniques plus avancées
    fmt.Printf("Wrapping method: %s\n", method.Name)
}

// Services d'exemple
type UserService struct {
    db map[int]User
}

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func NewUserService() *UserService {
    return &UserService{
        db: map[int]User{
            1: {ID: 1, Name: "Alice", Email: "alice@example.com"},
            2: {ID: 2, Name: "Bob", Email: "bob@example.com"},
            3: {ID: 3, Name: "Carol", Email: "carol@example.com"},
        },
    }
}

func (us *UserService) GetUser(id int) (User, error) {
    // Simuler une requête lente
    time.Sleep(100 * time.Millisecond)

    if user, exists := us.db[id]; exists {
        return user, nil
    }

    return User{}, fmt.Errorf("utilisateur %d non trouvé", id)
}

func (us *UserService) SearchUsers(query string) ([]User, error) {
    // Simuler une recherche lente
    time.Sleep(200 * time.Millisecond)

    var results []User
    for _, user := range us.db {
        if strings.Contains(strings.ToLower(user.Name), strings.ToLower(query)) ||
           strings.Contains(strings.ToLower(user.Email), strings.ToLower(query)) {
            results = append(results, user)
        }
    }

    return results, nil
}

func (us *UserService) GetUserStats(minID, maxID int) (map[string]int, error) {
    // Simuler un calcul lourd
    time.Sleep(300 * time.Millisecond)

    stats := map[string]int{
        "total":    0,
        "in_range": 0,
    }

    for _, user := range us.db {
        stats["total"]++
        if user.ID >= minID && user.ID <= maxID {
            stats["in_range"]++
        }
    }

    return stats, nil
}

func main() {
    // Créer le cache et le service
    cache := NewCache(5 * time.Second)
    userService := NewUserService()

    // Wrapper les fonctions avec le cache
    getUserCached := cache.WrapFunction("GetUser", userService.GetUser)
    searchUsersCached := cache.WrapFunction("SearchUsers", userService.SearchUsers)
    getUserStatsCached := cache.WrapFunction("GetUserStats", userService.GetUserStats)

    fmt.Println("=== Test du système de cache ===")

    // Test 1: GetUser avec cache
    fmt.Println("\n--- Test GetUser ---")
    start := time.Now()
    results := getUserCached.Call(1)
    fmt.Printf("Premier appel (%.2fms): %v\n", float64(time.Since(start).Nanoseconds())/1e6, results[0])

    start = time.Now()
    results = getUserCached.Call(1) // Doit être en cache
    fmt.Printf("Deuxième appel (%.2fms): %v\n", float64(time.Since(start).Nanoseconds())/1e6, results[0])

    // Test 2: SearchUsers avec cache
    fmt.Println("\n--- Test SearchUsers ---")
    start = time.Now()
    results = searchUsersCached.Call("alice")
    fmt.Printf("Premier appel search (%.2fms): %v résultats\n", float64(time.Since(start).Nanoseconds())/1e6, len(results[0].([]User)))

    start = time.Now()
    results = searchUsersCached.Call("alice") // Doit être en cache
    fmt.Printf("Deuxième appel search (%.2fms): %v résultats\n", float64(time.Since(start).Nanoseconds())/1e6, len(results[0].([]User)))

    // Test 3: GetUserStats avec cache
    fmt.Println("\n--- Test GetUserStats ---")
    start = time.Now()
    results = getUserStatsCached.Call(1, 3)
    fmt.Printf("Premier appel stats (%.2fms): %v\n", float64(time.Since(start).Nanoseconds())/1e6, results[0])

    start = time.Now()
    results = getUserStatsCached.Call(1, 3) // Doit être en cache
    fmt.Printf("Deuxième appel stats (%.2fms): %v\n", float64(time.Since(start).Nanoseconds())/1e6, results[0])

    // Test avec des paramètres différents
    fmt.Println("\n--- Test avec paramètres différents ---")
    start = time.Now()
    results = getUserStatsCached.Call(2, 4) // Nouveaux paramètres, pas en cache
    fmt.Printf("Nouveaux paramètres (%.2fms): %v\n", float64(time.Since(start).Nanoseconds())/1e6, results[0])

    // Statistiques du cache
    fmt.Println("\n--- Statistiques du cache ---")
    cache.mu.RLock()
    fmt.Printf("Entrées en cache: %d\n", len(cache.entries))
    for key, entry := range cache.entries {
        fmt.Printf("  - %s (expire dans %v)\n", key, entry.ExpiresAt.Sub(time.Now()).Round(time.Second))
    }
    cache.mu.RUnlock()
}
```

## 8. Système de monitoring et métriques avec réflexion

Créons un système qui collecte automatiquement des métriques sur les appels de méthodes.

```go
package main

import (
    "fmt"
    "reflect"
    "sync"
    "time"
)

type Metrics struct {
    mu    sync.RWMutex
    stats map[string]*MethodStats
}

type MethodStats struct {
    CallCount    int64
    TotalTime    time.Duration
    AverageTime  time.Duration
    MinTime      time.Duration
    MaxTime      time.Duration
    ErrorCount   int64
    LastCall     time.Time
    LastError    time.Time
}

func NewMetrics() *Metrics {
    return &Metrics{
        stats: make(map[string]*MethodStats),
    }
}

func (m *Metrics) RecordCall(methodName string, duration time.Duration, hasError bool) {
    m.mu.Lock()
    defer m.mu.Unlock()

    stats, exists := m.stats[methodName]
    if !exists {
        stats = &MethodStats{
            MinTime: duration,
            MaxTime: duration,
        }
        m.stats[methodName] = stats
    }

    stats.CallCount++
    stats.TotalTime += duration
    stats.AverageTime = time.Duration(int64(stats.TotalTime) / stats.CallCount)
    stats.LastCall = time.Now()

    if duration < stats.MinTime {
        stats.MinTime = duration
    }
    if duration > stats.MaxTime {
        stats.MaxTime = duration
    }

    if hasError {
        stats.ErrorCount++
        stats.LastError = time.Now()
    }
}

func (m *Metrics) GetStats(methodName string) *MethodStats {
    m.mu.RLock()
    defer m.mu.RUnlock()

    if stats, exists := m.stats[methodName]; exists {
        // Retourner une copie pour éviter les races conditions
        statsCopy := *stats
        return &statsCopy
    }

    return nil
}

func (m *Metrics) GetAllStats() map[string]*MethodStats {
    m.mu.RLock()
    defer m.mu.RUnlock()

    result := make(map[string]*MethodStats)
    for name, stats := range m.stats {
        statsCopy := *stats
        result[name] = &statsCopy
    }

    return result
}

func (m *Metrics) PrintReport() {
    fmt.Println("\n" + "="*60)
    fmt.Println("RAPPORT DE MÉTRIQUES")
    fmt.Println("="*60)

    allStats := m.GetAllStats()

    if len(allStats) == 0 {
        fmt.Println("Aucune métrique disponible")
        return
    }

    for methodName, stats := range allStats {
        fmt.Printf("\nMéthode: %s\n", methodName)
        fmt.Printf("  Appels: %d\n", stats.CallCount)
        fmt.Printf("  Erreurs: %d (%.1f%%)\n",
            stats.ErrorCount,
            float64(stats.ErrorCount)/float64(stats.CallCount)*100)
        fmt.Printf("  Temps total: %v\n", stats.TotalTime.Round(time.Millisecond))
        fmt.Printf("  Temps moyen: %v\n", stats.AverageTime.Round(time.Millisecond))
        fmt.Printf("  Temps min: %v\n", stats.MinTime.Round(time.Millisecond))
        fmt.Printf("  Temps max: %v\n", stats.MaxTime.Round(time.Millisecond))
        fmt.Printf("  Dernier appel: %v\n", stats.LastCall.Format("15:04:05"))
        if !stats.LastError.IsZero() {
            fmt.Printf("  Dernière erreur: %v\n", stats.LastError.Format("15:04:05"))
        }
    }
}

// Décorateur de monitoring
type MonitoredStruct struct {
    target  interface{}
    metrics *Metrics
}

func NewMonitoredStruct(target interface{}, metrics *Metrics) *MonitoredStruct {
    return &MonitoredStruct{
        target:  target,
        metrics: metrics,
    }
}

// Utilise la réflexion pour intercepter les appels de méthodes
func (ms *MonitoredStruct) CallMethod(methodName string, args ...interface{}) ([]interface{}, error) {
    targetValue := reflect.ValueOf(ms.target)
    method := targetValue.MethodByName(methodName)

    if !method.IsValid() {
        return nil, fmt.Errorf("méthode %s non trouvée", methodName)
    }

    // Convertir les arguments
    argValues := make([]reflect.Value, len(args))
    for i, arg := range args {
        argValues[i] = reflect.ValueOf(arg)
    }

    // Mesurer le temps d'exécution
    start := time.Now()
    results := method.Call(argValues)
    duration := time.Since(start)

    // Vérifier s'il y a une erreur
    hasError := false
    resultInterfaces := make([]interface{}, len(results))
    for i, result := range results {
        resultInterfaces[i] = result.Interface()

        // Vérifier si c'est une erreur
        if err, ok := result.Interface().(error); ok && err != nil {
            hasError = true
        }
    }

    // Enregistrer les métriques
    fullMethodName := fmt.Sprintf("%s.%s", reflect.TypeOf(ms.target).Elem().Name(), methodName)
    ms.metrics.RecordCall(fullMethodName, duration, hasError)

    if hasError {
        return resultInterfaces, resultInterfaces[len(resultInterfaces)-1].(error)
    }

    return resultInterfaces, nil
}

// Auto-wrapper utilisant la réflexion
func WrapWithMonitoring(target interface{}, metrics *Metrics) interface{} {
    targetValue := reflect.ValueOf(target)
    targetType := reflect.TypeOf(target)

    if targetType.Kind() == reflect.Ptr {
        targetValue = targetValue.Elem()
        targetType = targetType.Elem()
    }

    // Créer une structure proxy
    proxyType := reflect.StructOf([]reflect.StructField{
        {
            Name: "Target",
            Type: targetType,
        },
        {
            Name: "Metrics",
            Type: reflect.TypeOf((*Metrics)(nil)),
        },
    })

    proxy := reflect.New(proxyType).Elem()
    proxy.Field(0).Set(targetValue)
    proxy.Field(1).Set(reflect.ValueOf(metrics))

    return proxy.Addr().Interface()
}

// Service d'exemple avec différents comportements
type OrderService struct {
    orders map[int]*Order
}

type Order struct {
    ID       int
    UserID   int
    Amount   float64
    Status   string
    Created  time.Time
}

func NewOrderService() *OrderService {
    return &OrderService{
        orders: map[int]*Order{
            1: {ID: 1, UserID: 1, Amount: 99.99, Status: "completed", Created: time.Now().Add(-time.Hour)},
            2: {ID: 2, UserID: 2, Amount: 149.99, Status: "pending", Created: time.Now().Add(-30 * time.Minute)},
            3: {ID: 3, UserID: 1, Amount: 79.99, Status: "cancelled", Created: time.Now().Add(-15 * time.Minute)},
        },
    }
}

func (os *OrderService) GetOrder(id int) (*Order, error) {
    // Simuler une latence variable
    time.Sleep(time.Duration(10+id*5) * time.Millisecond)

    if order, exists := os.orders[id]; exists {
        return order, nil
    }

    return nil, fmt.Errorf("commande %d non trouvée", id)
}

func (os *OrderService) CreateOrder(userID int, amount float64) (*Order, error) {
    // Simuler une opération plus lente
    time.Sleep(50 * time.Millisecond)

    // Simuler une erreur de validation
    if amount <= 0 {
        return nil, fmt.Errorf("montant invalide: %.2f", amount)
    }

    if amount > 1000 {
        return nil, fmt.Errorf("montant trop élevé: %.2f", amount)
    }

    newID := len(os.orders) + 1
    order := &Order{
        ID:      newID,
        UserID:  userID,
        Amount:  amount,
        Status:  "pending",
        Created: time.Now(),
    }

    os.orders[newID] = order
    return order, nil
}

func (os *OrderService) GetOrdersByUser(userID int) ([]*Order, error) {
    // Simuler une requête de base de données
    time.Sleep(30 * time.Millisecond)

    var userOrders []*Order
    for _, order := range os.orders {
        if order.UserID == userID {
            userOrders = append(userOrders, order)
        }
    }

    return userOrders, nil
}

func (os *OrderService) CalculateTotal(orders []*Order) (float64, error) {
    // Simuler un calcul complexe
    time.Sleep(20 * time.Millisecond)

    if len(orders) == 0 {
        return 0, fmt.Errorf("aucune commande fournie")
    }

    var total float64
    for _, order := range orders {
        if order.Status == "completed" {
            total += order.Amount
        }
    }

    return total, nil
}

func main() {
    // Créer les services
    metrics := NewMetrics()
    orderService := NewOrderService()
    monitoredService := NewMonitoredStruct(orderService, metrics)

    fmt.Println("=== Test du système de monitoring ===")

    // Simulation d'utilisation normale
    fmt.Println("\n--- Simulation d'utilisation ---")

    // Test GetOrder (succès et échecs)
    for i := 1; i <= 5; i++ {
        result, err := monitoredService.CallMethod("GetOrder", i)
        if err != nil {
            fmt.Printf("Erreur GetOrder(%d): %v\n", i, err)
        } else {
            order := result[0].(*Order)
            fmt.Printf("GetOrder(%d): Commande #%d - %.2f€\n", i, order.ID, order.Amount)
        }
    }

    // Test CreateOrder (succès et échecs)
    testAmounts := []float64{99.99, -10, 1500, 49.99, 0}
    for i, amount := range testAmounts {
        result, err := monitoredService.CallMethod("CreateOrder", i+10, amount)
        if err != nil {
            fmt.Printf("Erreur CreateOrder(%.2f): %v\n", amount, err)
        } else {
            order := result[0].(*Order)
            fmt.Printf("CreateOrder(%.2f): Nouvelle commande #%d\n", amount, order.ID)
        }
    }

    // Test GetOrdersByUser
    for userID := 1; userID <= 3; userID++ {
        result, err := monitoredService.CallMethod("GetOrdersByUser", userID)
        if err != nil {
            fmt.Printf("Erreur GetOrdersByUser(%d): %v\n", userID, err)
        } else {
            orders := result[0].([]*Order)
            fmt.Printf("GetOrdersByUser(%d): %d commandes trouvées\n", userID, len(orders))

            // Test CalculateTotal
            if len(orders) > 0 {
                result, err := monitoredService.CallMethod("CalculateTotal", orders)
                if err != nil {
                    fmt.Printf("Erreur CalculateTotal: %v\n", err)
                } else {
                    total := result[0].(float64)
                    fmt.Printf("  Total pour utilisateur %d: %.2f€\n", userID, total)
                }
            }
        }
    }

    // Test de performance (appels multiples)
    fmt.Println("\n--- Test de performance ---")
    start := time.Now()
    for i := 0; i < 50; i++ {
        orderID := (i % 3) + 1
        monitoredService.CallMethod("GetOrder", orderID)
    }
    fmt.Printf("50 appels GetOrder en %v\n", time.Since(start).Round(time.Millisecond))

    // Afficher le rapport final
    metrics.PrintReport()

    // Test de seuils d'alerte
    fmt.Println("\n" + "="*60)
    fmt.Println("ANALYSE ET ALERTES")
    fmt.Println("="*60)

    allStats := metrics.GetAllStats()
    for methodName, stats := range allStats {
        // Alertes sur le taux d'erreur
        errorRate := float64(stats.ErrorCount) / float64(stats.CallCount) * 100
        if errorRate > 20 {
            fmt.Printf("🚨 ALERTE: %s a un taux d'erreur élevé (%.1f%%)\n", methodName, errorRate)
        }

        // Alertes sur la performance
        if stats.AverageTime > 100*time.Millisecond {
            fmt.Printf("⚠️  ATTENTION: %s est lent (moyenne: %v)\n", methodName, stats.AverageTime.Round(time.Millisecond))
        }

        // Alertes sur la variabilité
        if stats.MaxTime > 3*stats.AverageTime {
            fmt.Printf("📊 INFO: %s a une latence variable (max: %v, moyenne: %v)\n",
                methodName,
                stats.MaxTime.Round(time.Millisecond),
                stats.AverageTime.Round(time.Millisecond))
        }
    }

    fmt.Println("\n✅ Analyse terminée")
}
```

## 9. Framework de tests automatisés avec réflexion

Créons un framework qui génère automatiquement des tests basés sur la structure des données.

```go
package main

import (
    "fmt"
    "math/rand"
    "reflect"
    "strings"
    "time"
)

type TestFramework struct {
    generators map[reflect.Type]func() interface{}
    validators map[reflect.Type]func(interface{}) bool
    testCount  int
}

func NewTestFramework() *TestFramework {
    tf := &TestFramework{
        generators: make(map[reflect.Type]func() interface{}),
        validators: make(map[reflect.Type]func(interface{}) bool),
        testCount:  10,
    }

    tf.registerDefaultGenerators()
    return tf
}

func (tf *TestFramework) registerDefaultGenerators() {
    // Générateurs pour types primitifs
    tf.generators[reflect.TypeOf(int(0))] = func() interface{} {
        return rand.Intn(1000)
    }

    tf.generators[reflect.TypeOf(string(""))] = func() interface{} {
        chars := "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
        length := rand.Intn(20) + 1
        result := make([]byte, length)
        for i := range result {
            result[i] = chars[rand.Intn(len(chars))]
        }
        return string(result)
    }

    tf.generators[reflect.TypeOf(bool(false))] = func() interface{} {
        return rand.Intn(2) == 1
    }

    tf.generators[reflect.TypeOf(float64(0))] = func() interface{} {
        return rand.Float64() * 1000
    }

    tf.generators[reflect.TypeOf(time.Time{})] = func() interface{} {
        return time.Now().Add(time.Duration(rand.Intn(365*24)) * time.Hour)
    }
}

func (tf *TestFramework) RegisterGenerator(t reflect.Type, generator func() interface{}) {
    tf.generators[t] = generator
}

func (tf *TestFramework) RegisterValidator(t reflect.Type, validator func(interface{}) bool) {
    tf.validators[t] = validator
}

func (tf *TestFramework) SetTestCount(count int) {
    tf.testCount = count
}

// Générer des données de test pour une structure
func (tf *TestFramework) GenerateTestData(structType reflect.Type) interface{} {
    if structType.Kind() == reflect.Ptr {
        structType = structType.Elem()
    }

    if structType.Kind() != reflect.Struct {
        return tf.generateForType(structType)
    }

    // Créer une nouvelle instance
    instance := reflect.New(structType).Elem()

    for i := 0; i < structType.NumField(); i++ {
        field := structType.Field(i)
        fieldValue := instance.Field(i)

        if !fieldValue.CanSet() {
            continue
        }

        // Vérifier les tags de test
        testTag := field.Tag.Get("test")
        if testTag == "skip" {
            continue
        }

        generatedValue := tf.generateForType(field.Type)
        if generatedValue != nil {
            fieldValue.Set(reflect.ValueOf(generatedValue))
        }
    }

    return instance.Interface()
}

func (tf *TestFramework) generateForType(t reflect.Type) interface{} {
    // Générateur personnalisé
    if generator, exists := tf.generators[t]; exists {
        return generator()
    }

    switch t.Kind() {
    case reflect.Slice:
        return tf.generateSlice(t)
    case reflect.Map:
        return tf.generateMap(t)
    case reflect.Ptr:
        elem := tf.generateForType(t.Elem())
        if elem == nil {
            return nil
        }
        ptr := reflect.New(t.Elem())
        ptr.Elem().Set(reflect.ValueOf(elem))
        return ptr.Interface()
    case reflect.Struct:
        return tf.GenerateTestData(t)
    case reflect.Interface:
        // Pour les interfaces, retourner nil
        return nil
    default:
        // Types primitifs par défaut
        zero := reflect.Zero(t)
        return zero.Interface()
    }
}

func (tf *TestFramework) generateSlice(t reflect.Type) interface{} {
    elemType := t.Elem()
    length := rand.Intn(5) + 1

    slice := reflect.MakeSlice(t, length, length)

    for i := 0; i < length; i++ {
        elem := tf.generateForType(elemType)
        if elem != nil {
            slice.Index(i).Set(reflect.ValueOf(elem))
        }
    }

    return slice.Interface()
}

func (tf *TestFramework) generateMap(t reflect.Type) interface{} {
    keyType := t.Key()
    valueType := t.Elem()

    m := reflect.MakeMap(t)
    length := rand.Intn(3) + 1

    for i := 0; i < length; i++ {
        key := tf.generateForType(keyType)
        value := tf.generateForType(valueType)

        if key != nil && value != nil {
            m.SetMapIndex(reflect.ValueOf(key), reflect.ValueOf(value))
        }
    }

    return m.Interface()
}

// Tester une fonction avec des données générées
func (tf *TestFramework) TestFunction(fn interface{}, testName string) {
    fnValue := reflect.ValueOf(fn)
    fnType := reflect.TypeOf(fn)

    if fnType.Kind() != reflect.Func {
        fmt.Printf("❌ %s: n'est pas une fonction\n", testName)
        return
    }

    fmt.Printf("\n🧪 Test: %s\n", testName)
    fmt.Printf("   Fonction: %s\n", fnType.String())

    successCount := 0
    errorCount := 0
    panicCount := 0

    for i := 0; i < tf.testCount; i++ {
        // Générer les arguments
        args := make([]reflect.Value, fnType.NumIn())
        argStrings := make([]string, fnType.NumIn())

        for j := 0; j < fnType.NumIn(); j++ {
            argType := fnType.In(j)
            arg := tf.generateForType(argType)
            args[j] = reflect.ValueOf(arg)
            argStrings[j] = fmt.Sprintf("%v", arg)
        }

        // Appeler la fonction avec gestion des panics
        func() {
            defer func() {
                if r := recover(); r != nil {
                    panicCount++
                    fmt.Printf("   ⚠️  Test %d: PANIC avec args (%s): %v\n",
                        i+1, strings.Join(argStrings, ", "), r)
                }
            }()

            results := fnValue.Call(args)

            // Vérifier s'il y a une erreur en dernière position
            hasError := false
            if len(results) > 0 {
                lastResult := results[len(results)-1]
                if lastResult.Type().Implements(reflect.TypeOf((*error)(nil)).Elem()) {
                    if !lastResult.IsNil() {
                        hasError = true
                        errorCount++
                        fmt.Printf("   ❌ Test %d: Erreur avec args (%s): %v\n",
                            i+1, strings.Join(argStrings, ", "), lastResult.Interface())
                    }
                }
            }

            if !hasError {
                successCount++
                if i < 3 { // Afficher seulement les premiers succès
                    resultStrings := make([]string, len(results))
                    for k, result := range results {
                        resultStrings[k] = fmt.Sprintf("%v", result.Interface())
                    }
                    fmt.Printf("   ✅ Test %d: Succès avec args (%s) → (%s)\n",
                        i+1, strings.Join(argStrings, ", "), strings.Join(resultStrings, ", "))
                }
            }
        }()
    }

    // Résumé
    fmt.Printf("\n   📊 Résultats:\n")
    fmt.Printf("      Succès: %d/%d (%.1f%%)\n", successCount, tf.testCount, float64(successCount)/float64(tf.testCount)*100)
    fmt.Printf("      Erreurs: %d/%d (%.1f%%)\n", errorCount, tf.testCount, float64(errorCount)/float64(tf.testCount)*100)
    fmt.Printf("      Panics: %d/%d (%.1f%%)\n", panicCount, tf.testCount, float64(panicCount)/float64(tf.testCount)*100)

    if panicCount > tf.testCount/4 {
        fmt.Printf("   🚨 ALERTE: Trop de panics détectées!\n")
    }
    if errorCount > tf.testCount/2 {
        fmt.Printf("   ⚠️  ATTENTION: Taux d'erreur élevé!\n")
    }
}

// Structures d'exemple pour les tests
type User struct {
    ID       int       `test:"positive"`
    Name     string    `test:"nonempty"`
    Email    string    `test:"email"`
    Age      int       `test:"range:18-100"`
    IsActive bool
    Created  time.Time
    Tags     []string
    Metadata map[string]interface{} `test:"skip"`
}

type Product struct {
    ID          int     `test:"positive"`
    Name        string  `test:"nonempty"`
    Price       float64 `test:"positive"`
    InStock     bool
    Categories  []string
    Attributes  map[string]string
}

// Fonctions d'exemple à tester
func CreateUser(name string, email string, age int) (*User, error) {
    if name == "" {
        return nil, fmt.Errorf("nom requis")
    }
    if age < 0 || age > 150 {
        return nil, fmt.Errorf("âge invalide: %d", age)
    }
    if !strings.Contains(email, "@") {
        return nil, fmt.Errorf("email invalide: %s", email)
    }

    return &User{
        ID:       rand.Intn(1000) + 1,
        Name:     name,
        Email:    email,
        Age:      age,
        IsActive: true,
        Created:  time.Now(),
    }, nil
}

func CalculateDiscount(price float64, percentage float64) (float64, error) {
    if price < 0 {
        return 0, fmt.Errorf("prix ne peut pas être négatif")
    }
    if percentage < 0 || percentage > 100 {
        return 0, fmt.Errorf("pourcentage doit être entre 0 et 100")
    }

    discount := price * (percentage / 100)
    return price - discount, nil
}

func ProcessTags(tags []string) []string {
    if len(tags) == 0 {
        return []string{}
    }

    var processed []string
    for _, tag := range tags {
        if tag != "" {
            processed = append(processed, strings.ToLower(strings.TrimSpace(tag)))
        }
    }

    return processed
}

func DivideNumbers(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division par zéro")
    }

    return a / b, nil
}

func main() {
    rand.Seed(time.Now().UnixNano())

    framework := NewTestFramework()
    framework.SetTestCount(20)

    // Enregistrer des générateurs personnalisés
    framework.RegisterGenerator(reflect.TypeOf(string("")), func() interface{} {
        emails := []string{"alice@test.com", "bob@example.org", "carol@domain.net", "", "invalid-email", "test@"}
        return emails[rand.Intn(len(emails))]
    })

    framework.RegisterGenerator(reflect.TypeOf(int(0)), func() interface{} {
        // Générer des entiers avec des cas limites
        values := []int{-10, -1, 0, 1, 18, 25, 50, 100, 150, 200}
        return values[rand.Intn(len(values))]
    })

    framework.RegisterGenerator(reflect.TypeOf(float64(0)), func() interface{} {
        // Générer des floats avec des cas limites
        values := []float64{-100, -1, 0, 0.1, 1, 50, 100, 1000}
        return values[rand.Intn(len(values))]
    })

    fmt.Println("=== FRAMEWORK DE TESTS AUTOMATISÉS ===")

    // Test de génération de données
    fmt.Println("\n--- Génération de données de test ---")

    for i := 0; i < 3; i++ {
        user := framework.GenerateTestData(reflect.TypeOf(User{}))
        fmt.Printf("User %d: %+v\n", i+1, user)

        product := framework.GenerateTestData(reflect.TypeOf(Product{}))
        fmt.Printf("Product %d: %+v\n", i+1, product)
    }

    // Tests des fonctions
    fmt.Println("\n" + "="*60)
    fmt.Println("TESTS AUTOMATISÉS DES FONCTIONS")
    fmt.Println("="*60)

    framework.TestFunction(CreateUser, "CreateUser")
    framework.TestFunction(CalculateDiscount, "CalculateDiscount")
    framework.TestFunction(ProcessTags, "ProcessTags")
    framework.TestFunction(DivideNumbers, "DivideNumbers")

    // Test d'une fonction complexe avec structures
    complexFunction := func(user User, product Product) (float64, error) {
        if user.Age < 18 {
            return 0, fmt.Errorf("utilisateur mineur")
        }
        if product.Price <= 0 {
            return 0, fmt.Errorf("prix invalide")
        }
        if !user.IsActive {
            return 0, fmt.Errorf("utilisateur inactif")
        }

        // Calcul avec remise basée sur l'âge
        discount := 0.0
        if user.Age > 65 {
            discount = 0.10 // 10% pour les seniors
        } else if user.Age < 25 {
            discount = 0.05 // 5% pour les jeunes
        }

        return product.Price * (1 - discount), nil
    }

    framework.TestFunction(complexFunction, "ComplexFunction")

    fmt.Println("\n" + "="*60)
    fmt.Println("CONCLUSION")
    fmt.Println("="*60)
    fmt.Println("✅ Tests automatisés terminés")
    fmt.Println("📝 Le framework a généré et testé automatiquement des centaines de cas")
    fmt.Println("🔍 Les fonctions ont été testées avec des données aléatoires et des cas limites")
    fmt.Println("⚡ Ceci démontre la puissance de la réflexion pour l'automatisation des tests")
}
```

## Résumé complet des cas d'usage pratiques

Dans cette section, nous avons exploré **7 cas d'usage pratiques** qui démontrent la puissance de la réflexion et de la métaprogrammation en Go :

### 1. **Sérialiseur JSON personnalisé**
- **Problème résolu** : Transformer automatiquement les données avec des règles personnalisées
- **Techniques utilisées** :
  - Analyse de structures avec réflexion
  - Tags personnalisés pour les transformations
  - Masquage de données sensibles
  - Formats de date/heure personnalisés
- **Bénéfices** : Sécurité, flexibilité, cohérence

### 2. **ORM simple**
- **Problème résolu** : Mapping automatique objet-relationnel
- **Techniques utilisées** :
  - Génération automatique de SQL
  - Analyse de tags pour les contraintes
  - Binding automatique des résultats
  - Gestion des types Go vers SQL
- **Bénéfices** : Productivité, réduction du code boilerplate, cohérence

### 3. **Framework de validation avancé**
- **Problème résolu** : Validation flexible et extensible des données
- **Techniques utilisées** :
  - Règles basées sur des tags
  - Validateurs personnalisés
  - Validation de structures imbriquées
  - Messages d'erreur configurables
- **Bénéfices** : Réutilisabilité, flexibilité, maintenabilité

### 4. **Système de configuration**
- **Problème résolu** : Chargement automatique depuis plusieurs sources
- **Techniques utilisées** :
  - Sources multiples (env, fichier, défauts)
  - Conversion automatique de types
  - Support des types complexes (slices, maps)
  - Validation intégrée
- **Bénéfices** : Configuration centralisée, flexibilité, validation

### 5. **Injection de dépendances**
- **Problème résolu** : Résolution automatique des dépendances
- **Techniques utilisées** :
  - Auto-wiring basé sur les types
  - Factories et singletons
  - Création automatique d'instances
  - Gestion des dépendances optionnelles
- **Bénéfices** : Découplage, testabilité, maintenabilité

### 6. **Système de mapping**
- **Problème résolu** : Conversion automatique entre structures
- **Techniques utilisées** :
  - Mapping par noms de champs
  - Convertisseurs personnalisés
  - Stratégies de nommage
  - Support des types complexes
- **Bénéfices** : Réduction du code, cohérence, flexibilité

### 7. **Cache intelligent**
- **Problème résolu** : Cache automatique basé sur les signatures de fonction
- **Techniques utilisées** :
  - Génération automatique de clés
  - Sérialisation intelligente des paramètres
  - Wrapper transparent de fonctions
  - Gestion de l'expiration
- **Bénéfices** : Performance, transparence, simplicité

### 8. **Système de monitoring**
- **Problème résolu** : Collecte automatique de métriques
- **Techniques utilisées** :
  - Interception transparente d'appels
  - Mesure automatique de performance
  - Détection d'erreurs
  - Génération de rapports
- **Bénéfices** : Observabilité, détection proactive, analyse

### 9. **Framework de tests automatisés**
- **Problème résolu** : Génération automatique de cas de test
- **Techniques utilisées** :
  - Génération de données aléatoires
  - Tests de robustesse automatiques
  - Détection de panics et erreurs
  - Analyse statistique des résultats
- **Bénéfices** : Couverture de test, détection de bugs, automatisation

## Principes clés observés

### **Performance**
- **Utiliser un cache** : Stocker les informations de réflexion coûteuses
- **Éviter la réflexion dans les boucles** : Pré-calculer quand possible
- **Mesurer l'impact** : Profiler les parties critiques

### **Sécurité des types**
- **Validation rigoureuse** : Vérifier tous les types et valeurs
- **Gestion d'erreurs complète** : Anticiper les cas d'échec
- **Tests exhaustifs** : Les erreurs surviennent à l'exécution

### **Simplicité d'utilisation**
- **APIs intuitive** : Cacher la complexité derrière des interfaces simples
- **Conventions claires** : Établir des patterns cohérents
- **Documentation extensive** : Expliquer les comportements et limitations

### **Flexibilité**
- **Extensibilité** : Permettre l'ajout de comportements personnalisés
- **Configuration** : Offrir des options de personnalisation
- **Interopérabilité** : S'intégrer avec l'écosystème existant

## Recommandations d'usage

### ✅ **Utilisez la réflexion quand :**
- Vous créez des bibliothèques génériques
- Vous avez besoin de flexibilité runtime
- Vous voulez réduire le code répétitif
- Les alternatives (generics, interfaces) ne suffisent pas

### ❌ **Évitez la réflexion quand :**
- La performance est critique
- La logique est simple et statique
- Les types sont connus à la compilation
- Une solution plus simple existe

### 🔧 **Bonnes pratiques :**
- Mesurez l'impact sur les performances
- Implémentez un cache pour les opérations coûteuses
- Documentez abondamment le comportement
- Testez exhaustivement tous les cas limites
- Fournissez des alternatives quand possible

La réflexion et la métaprogrammation sont des outils puissants qui, utilisés judicieusement, peuvent considérablement améliorer la productivité et créer des solutions élégantes pour des problèmes complexes. Ces techniques permettent de créer des bibliothèques et frameworks qui simplifient le développement tout en conservant la robustesse et les performances de Go.

⏭️
