🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12-3 : Génération de code

## Introduction à la génération de code

La génération de code est une technique qui consiste à écrire des programmes qui écrivent d'autres programmes. En Go, cela permet de créer automatiquement du code répétitif, d'optimiser les performances, et de maintenir la cohérence dans une base de code.

## Pourquoi générer du code ?

### Avantages

**Réduction de la répétition** : Éliminer le code boilerplate répétitif.

**Cohérence** : Garantir que des patterns similaires sont implémentés de manière identique.

**Performance** : Générer du code optimisé plutôt que d'utiliser la réflexion.

**Maintien automatique** : Régénérer le code quand les structures changent.

**Sécurité des types** : Le code généré est vérifié à la compilation.

### Cas d'usage courants

- **Getters/Setters** automatiques
- **Méthodes String()** pour les enums
- **Sérialisation/Désérialisation** optimisée
- **Mocks** pour les tests
- **Mappers** entre structures

## Directive `//go:generate`

Go fournit une directive spéciale pour intégrer la génération de code dans le processus de build.

### Syntaxe de base

```go
//go:generate command arguments
```

### Exemple simple

```go
package main

//go:generate echo "Code généré le $(date)" > generated.txt
//go:generate go run script.go

func main() {
    // Votre code ici
}
```

Pour exécuter la génération :
```bash
go generate ./...
```

## Stringer : Premier exemple pratique

L'outil `stringer` génère automatiquement des méthodes `String()` pour les enums.

### Installation

```bash
go install golang.org/x/tools/cmd/stringer@latest
```

### Utilisation

```go
package main

import "fmt"

//go:generate stringer -type=Status
type Status int

const (
    Pending Status = iota
    Running
    Completed
    Failed
)

//go:generate stringer -type=Priority
type Priority int

const (
    Low Priority = iota
    Medium
    High
    Critical
)

func main() {
    s := Running
    p := High

    fmt.Printf("Status: %s\n", s)     // Status: Running
    fmt.Printf("Priority: %s\n", p)   // Priority: High
}
```

Après `go generate`, Go crée automatiquement les fichiers `status_string.go` et `priority_string.go` :

```go
// Code généré automatiquement par stringer
// status_string.go

package main

import "strconv"

func _() {
    // Vérification à la compilation que les constantes sont à jour
    var x [1]struct{}
    _ = x[Pending-0]
    _ = x[Running-1]
    _ = x[Completed-2]
    _ = x[Failed-3]
}

const _Status_name = "PendingRunningCompletedFailed"

var _Status_index = [...]uint8{0, 7, 14, 23, 29}

func (i Status) String() string {
    if i < 0 || i >= Status(len(_Status_index)-1) {
        return "Status(" + strconv.Itoa(int(i)) + ")"
    }
    return _Status_name[_Status_index[i]:_Status_index[i+1]]
}
```

## Créer un générateur simple

Créons notre propre générateur pour créer des getters/setters automatiquement.

### Structure du projet

```
myproject/
├── main.go
├── person.go
├── tools/
│   └── getter-setter-gen/
│       └── main.go
└── generated/
    └── person_accessors.go
```

### Code source (person.go)

```go
package main

//go:generate go run tools/getter-setter-gen/main.go -type=Person -output=generated/person_accessors.go

type Person struct {
    name  string
    age   int
    email string
}

func main() {
    p := &Person{}

    // Utilisation des getters/setters générés
    p.SetName("Alice")
    p.SetAge(30)
    p.SetEmail("alice@example.com")

    fmt.Printf("Nom: %s, Age: %d, Email: %s\n",
        p.GetName(), p.GetAge(), p.GetEmail())
}
```

### Générateur (tools/getter-setter-gen/main.go)

```go
package main

import (
    "flag"
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
    "os"
    "strings"
    "text/template"
    "unicode"
)

type Field struct {
    Name         string
    Type         string
    CapitalName  string
}

type StructInfo struct {
    Name   string
    Fields []Field
}

const accessorTemplate = `// Code généré automatiquement - NE PAS MODIFIER

package main

{{range .Fields}}
// Get{{.CapitalName}} retourne la valeur de {{.Name}}
func (p *{{$.Name}}) Get{{.CapitalName}}() {{.Type}} {
    return p.{{.Name}}
}

// Set{{.CapitalName}} définit la valeur de {{.Name}}
func (p *{{$.Name}}) Set{{.CapitalName}}(value {{.Type}}) {
    p.{{.Name}} = value
}
{{end}}
`

func main() {
    var (
        typeName   = flag.String("type", "", "Nom du type à traiter")
        outputFile = flag.String("output", "", "Fichier de sortie")
    )
    flag.Parse()

    if *typeName == "" || *outputFile == "" {
        fmt.Println("Usage: go run main.go -type=TypeName -output=file.go")
        os.Exit(1)
    }

    // Parser le fichier Go courant
    fset := token.NewFileSet()
    node, err := parser.ParseFile(fset, "person.go", nil, parser.ParseComments)
    if err != nil {
        fmt.Printf("Erreur de parsing: %v\n", err)
        os.Exit(1)
    }

    // Trouver la struct
    structInfo := findStruct(node, *typeName)
    if structInfo == nil {
        fmt.Printf("Struct %s non trouvée\n", *typeName)
        os.Exit(1)
    }

    // Générer le code
    err = generateAccessors(structInfo, *outputFile)
    if err != nil {
        fmt.Printf("Erreur de génération: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Accesseurs générés dans %s\n", *outputFile)
}

func findStruct(node *ast.File, typeName string) *StructInfo {
    var structInfo *StructInfo

    ast.Inspect(node, func(n ast.Node) bool {
        if typeSpec, ok := n.(*ast.TypeSpec); ok {
            if typeSpec.Name.Name == typeName {
                if structType, ok := typeSpec.Type.(*ast.StructType); ok {
                    structInfo = &StructInfo{
                        Name:   typeName,
                        Fields: extractFields(structType),
                    }
                    return false
                }
            }
        }
        return true
    })

    return structInfo
}

func extractFields(structType *ast.StructType) []Field {
    var fields []Field

    for _, field := range structType.Fields.List {
        for _, name := range field.Names {
            // Ne traiter que les champs privés (commencent par une minuscule)
            if unicode.IsLower(rune(name.Name[0])) {
                fieldType := getTypeName(field.Type)
                fields = append(fields, Field{
                    Name:        name.Name,
                    Type:        fieldType,
                    CapitalName: capitalize(name.Name),
                })
            }
        }
    }

    return fields
}

func getTypeName(expr ast.Expr) string {
    switch t := expr.(type) {
    case *ast.Ident:
        return t.Name
    case *ast.StarExpr:
        return "*" + getTypeName(t.X)
    case *ast.ArrayType:
        return "[]" + getTypeName(t.Elt)
    case *ast.MapType:
        return "map[" + getTypeName(t.Key) + "]" + getTypeName(t.Value)
    default:
        return "interface{}"
    }
}

func capitalize(s string) string {
    if s == "" {
        return s
    }
    return strings.ToUpper(s[:1]) + s[1:]
}

func generateAccessors(structInfo *StructInfo, outputFile string) error {
    tmpl, err := template.New("accessors").Parse(accessorTemplate)
    if err != nil {
        return err
    }

    file, err := os.Create(outputFile)
    if err != nil {
        return err
    }
    defer file.Close()

    return tmpl.Execute(file, structInfo)
}
```

### Résultat généré (generated/person_accessors.go)

```go
// Code généré automatiquement - NE PAS MODIFIER

package main

// GetName retourne la valeur de name
func (p *Person) GetName() string {
    return p.name
}

// SetName définit la valeur de name
func (p *Person) SetName(value string) {
    p.name = value
}

// GetAge retourne la valeur de age
func (p *Person) GetAge() int {
    return p.age
}

// SetAge définit la valeur de age
func (p *Person) SetAge(value int) {
    p.age = value
}

// GetEmail retourne la valeur de email
func (p *Person) GetEmail() string {
    return p.email
}

// SetEmail définit la valeur de email
func (p *Person) SetEmail(value string) {
    p.email = value
}
```

## Générateur de validation

Créons un générateur qui produit des fonctions de validation basées sur les tags.

### Code source avec tags

```go
package main

//go:generate go run tools/validator-gen/main.go -input=user.go -output=generated/user_validation.go

type User struct {
    Name  string `validate:"required,min=2,max=50"`
    Email string `validate:"required,email"`
    Age   int    `validate:"required,min=18,max=100"`
}

func main() {
    user := User{
        Name:  "Al",
        Email: "invalid-email",
        Age:   15,
    }

    if errors := ValidateUser(user); len(errors) > 0 {
        for _, err := range errors {
            fmt.Printf("Erreur: %s\n", err)
        }
    } else {
        fmt.Println("Utilisateur valide")
    }
}
```

### Générateur de validation (tools/validator-gen/main.go)

```go
package main

import (
    "flag"
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
    "os"
    "reflect"
    "strconv"
    "strings"
    "text/template"
)

type ValidationRule struct {
    Type  string
    Value string
}

type FieldValidation struct {
    Name  string
    Type  string
    Rules []ValidationRule
}

type StructValidation struct {
    Name   string
    Fields []FieldValidation
}

const validationTemplate = `// Code généré automatiquement - NE PAS MODIFIER

package main

import (
    "fmt"
    "strings"
    "regexp"
)

var emailRegex = regexp.MustCompile(` + "`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$`" + `)

func Validate{{.Name}}(obj {{.Name}}) []string {
    var errors []string

    {{range .Fields}}
    // Validation pour {{.Name}}
    {{range .Rules}}
    {{if eq .Type "required"}}
    if obj.{{$.Name}} == {{getZeroValue $.Type}} {
        errors = append(errors, "{{$.Name}} est requis")
    }
    {{else if eq .Type "min"}}
    {{if eq $.Type "string"}}
    if len(obj.{{$.Name}}) < {{.Value}} {
        errors = append(errors, "{{$.Name}} doit contenir au moins {{.Value}} caractères")
    }
    {{else if eq $.Type "int"}}
    if obj.{{$.Name}} < {{.Value}} {
        errors = append(errors, "{{$.Name}} doit être au moins {{.Value}}")
    }
    {{end}}
    {{else if eq .Type "max"}}
    {{if eq $.Type "string"}}
    if len(obj.{{$.Name}}) > {{.Value}} {
        errors = append(errors, "{{$.Name}} ne peut pas dépasser {{.Value}} caractères")
    }
    {{else if eq $.Type "int"}}
    if obj.{{$.Name}} > {{.Value}} {
        errors = append(errors, "{{$.Name}} ne peut pas dépasser {{.Value}}")
    }
    {{end}}
    {{else if eq .Type "email"}}
    if obj.{{$.Name}} != "" && !emailRegex.MatchString(obj.{{$.Name}}) {
        errors = append(errors, "{{$.Name}} doit être un email valide")
    }
    {{end}}
    {{end}}

    {{end}}
    return errors
}
`

func main() {
    var (
        inputFile  = flag.String("input", "", "Fichier Go d'entrée")
        outputFile = flag.String("output", "", "Fichier de sortie")
    )
    flag.Parse()

    if *inputFile == "" || *outputFile == "" {
        fmt.Println("Usage: go run main.go -input=file.go -output=output.go")
        os.Exit(1)
    }

    structInfo, err := parseValidationStruct(*inputFile)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        os.Exit(1)
    }

    err = generateValidation(structInfo, *outputFile)
    if err != nil {
        fmt.Printf("Erreur de génération: %v\n", err)
        os.Exit(1)
    }

    fmt.Printf("Validation générée dans %s\n", *outputFile)
}

func parseValidationStruct(filename string) (*StructValidation, error) {
    fset := token.NewFileSet()
    node, err := parser.ParseFile(fset, filename, nil, parser.ParseComments)
    if err != nil {
        return nil, err
    }

    var structValidation *StructValidation

    ast.Inspect(node, func(n ast.Node) bool {
        if typeSpec, ok := n.(*ast.TypeSpec); ok {
            if structType, ok := typeSpec.Type.(*ast.StructType); ok {
                fields := extractValidationFields(structType)
                if len(fields) > 0 {
                    structValidation = &StructValidation{
                        Name:   typeSpec.Name.Name,
                        Fields: fields,
                    }
                    return false
                }
            }
        }
        return true
    })

    return structValidation, nil
}

func extractValidationFields(structType *ast.StructType) []FieldValidation {
    var fields []FieldValidation

    for _, field := range structType.Fields.List {
        if field.Tag != nil {
            tagValue := field.Tag.Value
            tagValue = strings.Trim(tagValue, "`")

            if validateTag := extractTag(tagValue, "validate"); validateTag != "" {
                for _, name := range field.Names {
                    fieldType := getTypeName(field.Type)
                    rules := parseValidationRules(validateTag)

                    fields = append(fields, FieldValidation{
                        Name:  name.Name,
                        Type:  fieldType,
                        Rules: rules,
                    })
                }
            }
        }
    }

    return fields
}

func extractTag(tagString, key string) string {
    // Simple extraction de tag (dans un vrai projet, utilisez reflect.StructTag)
    parts := strings.Fields(tagString)
    for _, part := range parts {
        if strings.HasPrefix(part, key+":") {
            value := part[len(key)+1:]
            return strings.Trim(value, `"`)
        }
    }
    return ""
}

func parseValidationRules(validateTag string) []ValidationRule {
    var rules []ValidationRule

    ruleParts := strings.Split(validateTag, ",")
    for _, rulePart := range ruleParts {
        if equalIndex := strings.Index(rulePart, "="); equalIndex != -1 {
            ruleType := rulePart[:equalIndex]
            ruleValue := rulePart[equalIndex+1:]
            rules = append(rules, ValidationRule{
                Type:  ruleType,
                Value: ruleValue,
            })
        } else {
            rules = append(rules, ValidationRule{
                Type:  rulePart,
                Value: "true",
            })
        }
    }

    return rules
}

func getTypeName(expr ast.Expr) string {
    switch t := expr.(type) {
    case *ast.Ident:
        return t.Name
    default:
        return "interface{}"
    }
}

func generateValidation(structInfo *StructValidation, outputFile string) error {
    funcMap := template.FuncMap{
        "getZeroValue": func(typeName string) string {
            switch typeName {
            case "string":
                return `""`
            case "int", "int32", "int64":
                return "0"
            case "bool":
                return "false"
            default:
                return "nil"
            }
        },
    }

    tmpl, err := template.New("validation").Funcs(funcMap).Parse(validationTemplate)
    if err != nil {
        return err
    }

    file, err := os.Create(outputFile)
    if err != nil {
        return err
    }
    defer file.Close()

    return tmpl.Execute(file, structInfo)
}
```

## Templates et text/template

Le package `text/template` est l'outil principal pour générer du code en Go.

### Fonctions utiles dans les templates

```go
package main

import (
    "fmt"
    "strings"
    "text/template"
    "os"
)

type Data struct {
    PackageName string
    Imports     []string
    Functions   []Function
}

type Function struct {
    Name       string
    Parameters []Parameter
    ReturnType string
    Body       string
}

type Parameter struct {
    Name string
    Type string
}

const codeTemplate = `// Code généré automatiquement
package {{.PackageName}}

{{if .Imports}}
import (
{{range .Imports}}    "{{.}}"
{{end}})
{{end}}

{{range .Functions}}
func {{.Name}}({{range $i, $p := .Parameters}}{{if $i}}, {{end}}{{$p.Name}} {{$p.Type}}{{end}}) {{.ReturnType}} {
{{.Body}}
}

{{end}}
`

func main() {
    data := Data{
        PackageName: "generated",
        Imports:     []string{"fmt", "strings"},
        Functions: []Function{
            {
                Name: "HelloWorld",
                Parameters: []Parameter{
                    {Name: "name", Type: "string"},
                },
                ReturnType: "string",
                Body:       `    return fmt.Sprintf("Hello, %s!", name)`,
            },
            {
                Name: "ProcessStrings",
                Parameters: []Parameter{
                    {Name: "items", Type: "[]string"},
                },
                ReturnType: "[]string",
                Body: `    var result []string
    for _, item := range items {
        result = append(result, strings.ToUpper(item))
    }
    return result`,
            },
        },
    }

    tmpl, err := template.New("code").Parse(codeTemplate)
    if err != nil {
        fmt.Printf("Erreur template: %v\n", err)
        return
    }

    err = tmpl.Execute(os.Stdout, data)
    if err != nil {
        fmt.Printf("Erreur exécution: %v\n", err)
    }
}
```

### Templates avec fonctions personnalisées

```go
package main

import (
    "fmt"
    "strings"
    "text/template"
    "os"
)

const advancedTemplate = `// Code généré
package {{.Package}}

{{range .Types}}
type {{.Name}} struct {
{{range .Fields}}    {{.Name | title}} {{.Type}} ` + "`json:\"{{.Name | snake}}\"`" + `
{{end}}}

func New{{.Name}}() *{{.Name}} {
    return &{{.Name}}{}
}

{{range .Fields}}
func ({{$.Name | lower | slice 0 1}} *{{$.Name}}) Get{{.Name | title}}() {{.Type}} {
    return {{$.Name | lower | slice 0 1}}.{{.Name | title}}
}

func ({{$.Name | lower | slice 0 1}} *{{$.Name}}) Set{{.Name | title}}(value {{.Type}}) {
    {{$.Name | lower | slice 0 1}}.{{.Name | title}} = value
}
{{end}}

{{end}}
`

type TemplateData struct {
    Package string
    Types   []TypeInfo
}

type TypeInfo struct {
    Name   string
    Fields []FieldInfo
}

type FieldInfo struct {
    Name string
    Type string
}

func main() {
    funcMap := template.FuncMap{
        "title": strings.Title,
        "lower": strings.ToLower,
        "upper": strings.ToUpper,
        "snake": func(s string) string {
            // Convertir camelCase en snake_case
            var result strings.Builder
            for i, r := range s {
                if i > 0 && r >= 'A' && r <= 'Z' {
                    result.WriteRune('_')
                }
                result.WriteRune(r)
            }
            return strings.ToLower(result.String())
        },
        "slice": func(start, end int, s string) string {
            if start >= len(s) {
                return ""
            }
            if end > len(s) {
                end = len(s)
            }
            return s[start:end]
        },
    }

    data := TemplateData{
        Package: "models",
        Types: []TypeInfo{
            {
                Name: "User",
                Fields: []FieldInfo{
                    {Name: "firstName", Type: "string"},
                    {Name: "lastName", Type: "string"},
                    {Name: "email", Type: "string"},
                    {Name: "age", Type: "int"},
                },
            },
        },
    }

    tmpl, err := template.New("advanced").Funcs(funcMap).Parse(advancedTemplate)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
        return
    }

    err = tmpl.Execute(os.Stdout, data)
    if err != nil {
        fmt.Printf("Erreur: %v\n", err)
    }
}
```

## Intégration avec go generate

### Exemple complet avec Makefile

```makefile
# Makefile
.PHONY: generate build test clean

generate:
	go generate ./...

build: generate
	go build -o bin/myapp ./cmd/myapp

test: generate
	go test ./...

clean:
	rm -rf generated/
	rm -f bin/myapp

install-tools:
	go install golang.org/x/tools/cmd/stringer@latest
	go install github.com/golang/mock/mockgen@latest
```

### Script de génération

```go
// tools/generate.go
//go:build ignore

package main

import (
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
)

func main() {
    if err := runGeneration(); err != nil {
        fmt.Printf("Erreur de génération: %v\n", err)
        os.Exit(1)
    }
}

func runGeneration() error {
    // Créer le dossier generated s'il n'existe pas
    if err := os.MkdirAll("generated", 0755); err != nil {
        return err
    }

    generators := []struct {
        name string
        cmd  []string
    }{
        {
            name: "Enums",
            cmd:  []string{"stringer", "-type=Status,Priority", "-output=generated/enums_string.go"},
        },
        {
            name: "Mocks",
            cmd:  []string{"mockgen", "-source=interfaces.go", "-destination=generated/mocks.go"},
        },
        {
            name: "Accessors",
            cmd:  []string{"go", "run", "tools/accessor-gen/main.go", "-type=User", "-output=generated/user_accessors.go"},
        },
    }

    for _, gen := range generators {
        fmt.Printf("Génération: %s...\n", gen.name)
        cmd := exec.Command(gen.cmd[0], gen.cmd[1:]...)
        if output, err := cmd.CombinedOutput(); err != nil {
            return fmt.Errorf("%s failed: %v\nOutput: %s", gen.name, err, output)
        }
    }

    fmt.Println("Génération terminée avec succès!")
    return nil
}
```

## Bonnes pratiques

### 1. Structure des fichiers générés

```go
// En-tête standard pour les fichiers générés
// Code généré automatiquement par [outil] le [date] - NE PAS MODIFIER
//go:build !ignore_autogenerated

package mypackage
```

### 2. Nommage des fichiers

- `*_generated.go` : fichiers générés
- `*_string.go` : méthodes String() générées
- `tools/` : générateurs personnalisés
- `generated/` : sortie des générateurs

### 3. Tests pour les générateurs

```go
// tools/accessor-gen/main_test.go
package main

import (
    "os"
    "strings"
    "testing"
)

func TestGenerateAccessors(t *testing.T) {
    // Code de test temporaire
    tempFile := "test_output.go"
    defer os.Remove(tempFile)

    structInfo := &StructInfo{
        Name: "TestStruct",
        Fields: []Field{
            {Name: "name", Type: "string", CapitalName: "Name"},
            {Name: "age", Type: "int", CapitalName: "Age"},
        },
    }

    err := generateAccessors(structInfo, tempFile)
    if err != nil {
        t.Fatalf("Génération échouée: %v", err)
    }

    // Vérifier le contenu généré
    content, err := os.ReadFile(tempFile)
    if err != nil {
        t.Fatalf("Lecture échouée: %v", err)
    }

    expected := []string{
        "func (p *TestStruct) GetName() string",
        "func (p *TestStruct) SetName(value string)",
        "func (p *TestStruct) GetAge() int",
        "func (p *TestStruct) SetAge(value int)",
    }

    contentStr := string(content)
    for _, exp := range expected {
        if !strings.Contains(contentStr, exp) {
            t.Errorf("Contenu attendu manquant: %s", exp)
        }
    }
}
```

### 4. Configuration via fichiers

```yaml
# generator-config.yaml
generators:
  - name: "accessors"
    types: ["User", "Product"]
    output: "generated/accessors.go"
    template: "templates/accessor.tmpl"

  - name: "validators"
    types: ["User"]
    output: "generated/validators.go"
    template: "templates/validator.tmpl"
```

## Outils populaires

### 1. Mockgen (Mocks)

```go
//go:generate mockgen -source=user.go -destination=mocks/user_mock.go

type UserService interface {
    GetUser(id int) (*User, error)
    CreateUser(user *User) error
    UpdateUser(user *User) error
    DeleteUser(id int) error
}
```

### 2. Protobuf

```go
//go:generate protoc --go_out=. --go_opt=paths=source_relative user.proto
```

### 3. Wire (Injection de dépendances)

```go
//go:generate wire

// +build wireinject

func InitializeUserService() *UserService {
    wire.Build(NewDatabase, NewUserRepository, NewUserService)
    return &UserService{}
}
```

## Résumé

La génération de code en Go permet de :

- **Automatiser la création** de code répétitif
- **Maintenir la cohérence** dans toute la base de code
- **Optimiser les performances** en évitant la réflexion
- **Réduire les erreurs** grâce à la vérification à la compilation

**Points clés :**
- Utilisez `//go:generate` pour intégrer la génération au build
- Créez des générateurs simples avec `text/template`
- Testez vos générateurs comme du code normal
- Suivez les conventions de nommage pour les fichiers générés
- Intégrez la génération dans votre workflow de développement

La génération de code est un outil puissant qui, utilisé judicieusement, peut considérablement améliorer la productivité et la qualité de votre code Go. Dans la prochaine section, nous verrons des cas d'usage pratiques qui combinent tous ces concepts.

⏭️
