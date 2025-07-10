🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12-1 : Package reflect

## Introduction au package reflect

Le package `reflect` est le cœur de la réflexion en Go. Il fournit deux types principaux qui permettent d'examiner et de manipuler les types et valeurs à l'exécution :

- `reflect.Type` : représente un type Go
- `reflect.Value` : représente une valeur Go et permet de la manipuler

## Concepts de base

### Obtenir le Type et la Value

Pour commencer à utiliser la réflexion, vous devez d'abord obtenir les informations sur un objet :

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x int = 42

    // Obtenir le type
    t := reflect.TypeOf(x)
    fmt.Printf("Type: %v\n", t)        // Type: int
    fmt.Printf("Kind: %v\n", t.Kind()) // Kind: int

    // Obtenir la valeur
    v := reflect.ValueOf(x)
    fmt.Printf("Value: %v\n", v)          // Value: 42
    fmt.Printf("Type from Value: %v\n", v.Type()) // Type from Value: int
}
```

### Différence entre Type et Kind

- **Type** : le type exact (par exemple `MyInt` même s'il est basé sur `int`)
- **Kind** : le type sous-jacent (par exemple `int`, `string`, `struct`, etc.)

```go
package main

import (
    "fmt"
    "reflect"
)

type MyInt int

func main() {
    var x MyInt = 42

    t := reflect.TypeOf(x)
    fmt.Printf("Type: %v\n", t)        // Type: main.MyInt
    fmt.Printf("Kind: %v\n", t.Kind()) // Kind: int
}
```

## Travailler avec les types primitifs

### Examiner les types de base

```go
package main

import (
    "fmt"
    "reflect"
)

func examineType(x interface{}) {
    t := reflect.TypeOf(x)
    v := reflect.ValueOf(x)

    fmt.Printf("=== Examen de %v ===\n", x)
    fmt.Printf("Type: %v\n", t)
    fmt.Printf("Kind: %v\n", t.Kind())
    fmt.Printf("Value: %v\n", v)
    fmt.Printf("Can Set: %v\n", v.CanSet())
    fmt.Println()
}

func main() {
    examineType(42)
    examineType("hello")
    examineType(3.14)
    examineType(true)
}
```

### Convertir les valeurs

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    v := reflect.ValueOf(42)

    // Récupérer la valeur originale
    originalValue := v.Interface()
    fmt.Printf("Valeur originale: %v (type: %T)\n", originalValue, originalValue)

    // Convertir en int (type assertion)
    intValue := v.Interface().(int)
    fmt.Printf("Valeur int: %v\n", intValue)

    // Vérifier le type avant conversion
    if v.Kind() == reflect.Int {
        fmt.Printf("C'est bien un int: %d\n", v.Int())
    }
}
```

## Travailler avec les pointeurs

La réflexion avec les pointeurs nécessite une attention particulière :

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := 42
    p := &x

    // Réflexion sur le pointeur
    vPtr := reflect.ValueOf(p)
    fmt.Printf("Type du pointeur: %v\n", vPtr.Type())     // *int
    fmt.Printf("Kind du pointeur: %v\n", vPtr.Kind())     // ptr

    // Déréférencer le pointeur
    vElem := vPtr.Elem()
    fmt.Printf("Type de l'élément: %v\n", vElem.Type())   // int
    fmt.Printf("Valeur de l'élément: %v\n", vElem.Int())  // 42

    // Vérifier si c'est un pointeur
    if vPtr.Kind() == reflect.Ptr {
        fmt.Println("C'est un pointeur!")
        if !vPtr.IsNil() {
            fmt.Printf("Valeur pointée: %v\n", vPtr.Elem().Interface())
        }
    }
}
```

## Modifier les valeurs (Settable values)

Pour modifier une valeur via la réflexion, elle doit être "settable" :

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := 42

    // Ceci ne fonctionne PAS
    v1 := reflect.ValueOf(x)
    fmt.Printf("v1 CanSet: %v\n", v1.CanSet()) // false

    // Ceci fonctionne
    v2 := reflect.ValueOf(&x).Elem()
    fmt.Printf("v2 CanSet: %v\n", v2.CanSet()) // true

    if v2.CanSet() {
        v2.SetInt(100)
        fmt.Printf("Nouvelle valeur de x: %v\n", x) // 100
    }
}
```

### Exemple pratique : fonction générique pour modifier des valeurs

```go
package main

import (
    "fmt"
    "reflect"
)

func setValue(ptr interface{}, newValue interface{}) error {
    v := reflect.ValueOf(ptr)

    // Vérifier que c'est un pointeur
    if v.Kind() != reflect.Ptr {
        return fmt.Errorf("pas un pointeur")
    }

    // Déréférencer
    elem := v.Elem()
    if !elem.CanSet() {
        return fmt.Errorf("valeur non modifiable")
    }

    // Créer une valeur du bon type
    newV := reflect.ValueOf(newValue)

    // Vérifier la compatibilité des types
    if elem.Type() != newV.Type() {
        return fmt.Errorf("types incompatibles: %v vs %v", elem.Type(), newV.Type())
    }

    // Modifier la valeur
    elem.Set(newV)
    return nil
}

func main() {
    var x int = 42
    var s string = "hello"

    fmt.Printf("Avant: x=%d, s=%s\n", x, s)

    setValue(&x, 100)
    setValue(&s, "world")

    fmt.Printf("Après: x=%d, s=%s\n", x, s)
}
```

## Travailler avec les slices et arrays

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    v := reflect.ValueOf(numbers)

    fmt.Printf("Type: %v\n", v.Type())           // []int
    fmt.Printf("Kind: %v\n", v.Kind())           // slice
    fmt.Printf("Length: %d\n", v.Len())          // 5
    fmt.Printf("Capacity: %d\n", v.Cap())        // 5

    // Parcourir les éléments
    fmt.Print("Éléments: ")
    for i := 0; i < v.Len(); i++ {
        element := v.Index(i)
        fmt.Printf("%d ", element.Int())
    }
    fmt.Println()

    // Modifier un élément (si le slice est modifiable)
    if v.Index(0).CanSet() {
        v.Index(0).SetInt(10)
        fmt.Printf("Premier élément modifié: %v\n", numbers)
    }
}
```

### Créer des slices dynamiquement

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // Créer un slice d'int vide
    sliceType := reflect.TypeOf([]int{})
    newSlice := reflect.MakeSlice(sliceType, 0, 10)

    // Ajouter des éléments
    for i := 1; i <= 5; i++ {
        newSlice = reflect.Append(newSlice, reflect.ValueOf(i))
    }

    // Convertir en slice Go normal
    result := newSlice.Interface().([]int)
    fmt.Printf("Slice créé: %v\n", result)
}
```

## Travailler avec les maps

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    m := map[string]int{
        "alice": 25,
        "bob":   30,
        "carol": 35,
    }

    v := reflect.ValueOf(m)

    fmt.Printf("Type: %v\n", v.Type())  // map[string]int
    fmt.Printf("Kind: %v\n", v.Kind())  // map
    fmt.Printf("Length: %d\n", v.Len()) // 3

    // Parcourir les clés
    keys := v.MapKeys()
    fmt.Println("Clés et valeurs:")
    for _, key := range keys {
        value := v.MapIndex(key)
        fmt.Printf("  %v: %v\n", key.Interface(), value.Interface())
    }

    // Ajouter une nouvelle entrée
    newKey := reflect.ValueOf("david")
    newValue := reflect.ValueOf(40)
    v.SetMapIndex(newKey, newValue)

    fmt.Printf("Map après ajout: %v\n", m)
}
```

### Créer des maps dynamiquement

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    // Créer un type map[string]int
    mapType := reflect.TypeOf(map[string]int{})
    newMap := reflect.MakeMap(mapType)

    // Ajouter des éléments
    entries := map[string]int{
        "un":    1,
        "deux":  2,
        "trois": 3,
    }

    for k, v := range entries {
        key := reflect.ValueOf(k)
        value := reflect.ValueOf(v)
        newMap.SetMapIndex(key, value)
    }

    // Convertir en map Go normale
    result := newMap.Interface().(map[string]int)
    fmt.Printf("Map créée: %v\n", result)
}
```

## Exemple pratique : Fonction d'affichage générique

Créons une fonction qui peut afficher n'importe quel type de données :

```go
package main

import (
    "fmt"
    "reflect"
)

func printValue(x interface{}) {
    v := reflect.ValueOf(x)
    printValueHelper(v, 0)
}

func printValueHelper(v reflect.Value, indent int) {
    spaces := ""
    for i := 0; i < indent; i++ {
        spaces += "  "
    }

    switch v.Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        fmt.Printf("%s%d\n", spaces, v.Int())
    case reflect.String:
        fmt.Printf("%s\"%s\"\n", spaces, v.String())
    case reflect.Bool:
        fmt.Printf("%s%t\n", spaces, v.Bool())
    case reflect.Float32, reflect.Float64:
        fmt.Printf("%s%f\n", spaces, v.Float())
    case reflect.Slice, reflect.Array:
        fmt.Printf("%s[\n", spaces)
        for i := 0; i < v.Len(); i++ {
            printValueHelper(v.Index(i), indent+1)
        }
        fmt.Printf("%s]\n", spaces)
    case reflect.Map:
        fmt.Printf("%s{\n", spaces)
        keys := v.MapKeys()
        for _, key := range keys {
            fmt.Printf("%s  %v: ", spaces, key.Interface())
            printValueHelper(v.MapIndex(key), 0)
        }
        fmt.Printf("%s}\n", spaces)
    case reflect.Ptr:
        if v.IsNil() {
            fmt.Printf("%s<nil>\n", spaces)
        } else {
            printValueHelper(v.Elem(), indent)
        }
    default:
        fmt.Printf("%s%v (type: %s)\n", spaces, v.Interface(), v.Type())
    }
}

func main() {
    // Tester avec différents types
    printValue(42)
    printValue("hello")
    printValue([]int{1, 2, 3})
    printValue(map[string]int{"a": 1, "b": 2})

    x := 100
    printValue(&x)
}
```

## Vérifications et sécurité

Lors de l'utilisation de la réflexion, il est important de faire des vérifications :

```go
package main

import (
    "fmt"
    "reflect"
)

func safeGetInt(x interface{}) (int, error) {
    v := reflect.ValueOf(x)

    // Vérifier le type
    switch v.Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return int(v.Int()), nil
    case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
        return int(v.Uint()), nil
    case reflect.Float32, reflect.Float64:
        return int(v.Float()), nil
    case reflect.String:
        // Pourrait essayer de parser la string
        return 0, fmt.Errorf("conversion string vers int non implémentée")
    default:
        return 0, fmt.Errorf("impossible de convertir %v en int", v.Kind())
    }
}

func main() {
    values := []interface{}{
        42,
        int32(42),
        uint(42),
        3.14,
        "42",
        true,
    }

    for _, val := range values {
        result, err := safeGetInt(val)
        if err != nil {
            fmt.Printf("Erreur avec %v (%T): %v\n", val, val, err)
        } else {
            fmt.Printf("Succès avec %v (%T): %d\n", val, val, result)
        }
    }
}
```

## Bonnes pratiques

### 1. Toujours vérifier les types et valeurs

```go
func safeOperation(x interface{}) {
    v := reflect.ValueOf(x)

    // Vérifier si c'est nil
    if !v.IsValid() {
        fmt.Println("Valeur invalide")
        return
    }

    // Vérifier le type avant utilisation
    if v.Kind() != reflect.Int {
        fmt.Printf("Type attendu: int, reçu: %v\n", v.Kind())
        return
    }

    // Maintenant c'est sûr
    fmt.Printf("Valeur: %d\n", v.Int())
}
```

### 2. Gérer les cas d'erreur

```go
func getValue(x interface{}) interface{} {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Erreur de réflexion: %v\n", r)
        }
    }()

    v := reflect.ValueOf(x)
    return v.Interface()
}
```

### 3. Utiliser des fonctions utilitaires

```go
func isNumeric(v reflect.Value) bool {
    switch v.Kind() {
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64,
         reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64,
         reflect.Float32, reflect.Float64:
        return true
    default:
        return false
    }
}

func canConvertToString(v reflect.Value) bool {
    switch v.Kind() {
    case reflect.String:
        return true
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        return true
    case reflect.Bool:
        return true
    default:
        return false
    }
}
```

## Résumé

Le package `reflect` vous permet de :

- **Examiner les types** avec `reflect.TypeOf()` et `reflect.ValueOf()`
- **Distinguer Type et Kind** pour une analyse précise
- **Modifier les valeurs** via des pointeurs et `CanSet()`
- **Travailler avec les collections** (slices, maps, arrays)
- **Créer des structures dynamiquement** avec `MakeSlice`, `MakeMap`, etc.
- **Implémenter des fonctions génériques** qui fonctionnent avec différents types

**Points importants à retenir :**
- La réflexion est puissante mais coûteuse en performance
- Toujours vérifier les types et gérer les erreurs
- Une valeur doit être "settable" pour être modifiée
- Utilisez `recover()` pour gérer les panics potentielles
- La réflexion sacrifie la sécurité de type au profit de la flexibilité

Dans la prochaine section, nous verrons comment utiliser les tags de struct pour ajouter des métadonnées à vos structures et les exploiter via la réflexion.

⏭️
