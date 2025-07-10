🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13-2 : JSON et sérialisation en Go

## Introduction

JSON (JavaScript Object Notation) est le format d'échange de données le plus populaire sur le web. Il est simple, lisible par les humains et supporté par presque tous les langages de programmation. En Go, le package `encoding/json` offre des outils puissants pour convertir des données Go en JSON et vice versa.

## Qu'est-ce que JSON ?

### Format JSON de base
JSON utilise une syntaxe simple avec des paires clé-valeur :

```json
{
  "nom": "Alice",
  "age": 30,
  "actif": true,
  "adresse": {
    "rue": "123 Rue Principale",
    "ville": "Paris"
  },
  "hobbies": ["lecture", "natation", "programmation"]
}
```

### Types de données JSON
- **String** : `"texte"`
- **Number** : `42`, `3.14`
- **Boolean** : `true`, `false`
- **Array** : `[1, 2, 3]`
- **Object** : `{"clé": "valeur"}`
- **Null** : `null`

## Package `encoding/json`

```go
import "encoding/json"
```

Le package JSON de Go fournit deux opérations principales :
- **Marshal** : Convertir Go → JSON (sérialisation)
- **Unmarshal** : Convertir JSON → Go (désérialisation)

## Sérialisation (Go vers JSON)

### Exemple basique avec `json.Marshal`

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Personne struct {
    Nom   string `json:"nom"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}

func main() {
    // Créer une structure Go
    p := Personne{
        Nom:   "Alice",
        Age:   30,
        Email: "alice@example.com",
    }

    // Convertir en JSON
    jsonData, err := json.Marshal(p)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("JSON : %s\n", string(jsonData))
    // Résultat : {"nom":"Alice","age":30,"email":"alice@example.com"}
}
```

### Sérialisation avec indentation

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Personne struct {
    Nom     string   `json:"nom"`
    Age     int      `json:"age"`
    Email   string   `json:"email"`
    Hobbies []string `json:"hobbies"`
}

func main() {
    p := Personne{
        Nom:     "Bob",
        Age:     25,
        Email:   "bob@example.com",
        Hobbies: []string{"football", "lecture", "cuisine"},
    }

    // JSON indenté (plus lisible)
    jsonData, err := json.MarshalIndent(p, "", "  ")
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("JSON indenté :\n%s\n", string(jsonData))
}
```

**Résultat :**
```json
{
  "nom": "Bob",
  "age": 25,
  "email": "bob@example.com",
  "hobbies": [
    "football",
    "lecture",
    "cuisine"
  ]
}
```

## Désérialisation (JSON vers Go)

### Exemple basique avec `json.Unmarshal`

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Personne struct {
    Nom   string `json:"nom"`
    Age   int    `json:"age"`
    Email string `json:"email"`
}

func main() {
    // JSON en string
    jsonString := `{"nom":"Charlie","age":35,"email":"charlie@example.com"}`

    // Convertir JSON en structure Go
    var p Personne
    err := json.Unmarshal([]byte(jsonString), &p)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Personne : %+v\n", p)
    // Résultat : Personne : {Nom:Charlie Age:35 Email:charlie@example.com}

    // Accéder aux champs individuellement
    fmt.Printf("Nom : %s\n", p.Nom)
    fmt.Printf("Age : %d ans\n", p.Age)
}
```

### Désérialisation avec validation

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Produit struct {
    ID          int     `json:"id"`
    Nom         string  `json:"nom"`
    Prix        float64 `json:"prix"`
    EnStock     bool    `json:"en_stock"`
    Categories  []string `json:"categories"`
}

func main() {
    jsonString := `{
        "id": 1,
        "nom": "Ordinateur portable",
        "prix": 999.99,
        "en_stock": true,
        "categories": ["électronique", "informatique"]
    }`

    var produit Produit
    err := json.Unmarshal([]byte(jsonString), &produit)
    if err != nil {
        log.Fatal("Erreur de désérialisation :", err)
    }

    // Valider les données
    if produit.Prix <= 0 {
        log.Fatal("Prix invalide")
    }

    if produit.Nom == "" {
        log.Fatal("Nom de produit requis")
    }

    fmt.Printf("Produit valide : %+v\n", produit)
    fmt.Printf("Prix formaté : %.2f€\n", produit.Prix)
}
```

## Tags JSON

Les tags JSON permettent de personnaliser la sérialisation/désérialisation.

### Tags de base

```go
type Utilisateur struct {
    ID       int    `json:"id"`                    // Nom personnalisé
    Nom      string `json:"name"`                  // Nom différent
    MotPasse string `json:"-"`                     // Ignorer ce champ
    Email    string `json:"email,omitempty"`       // Omettre si vide
    Age      int    `json:"age,string"`            // Convertir en string
    Actif    bool   `json:"is_active"`             // Nom personnalisé
}
```

### Exemple avec différents tags

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Compte struct {
    ID          int     `json:"id"`
    Nom         string  `json:"username"`
    MotPasse    string  `json:"-"`                    // Jamais sérialisé
    Email       string  `json:"email,omitempty"`      // Omis si vide
    Telephone   string  `json:"phone,omitempty"`      // Omis si vide
    Solde       float64 `json:"balance,string"`       // Converti en string
    Actif       bool    `json:"is_active"`
    Description string  `json:"description,omitempty"`
}

func main() {
    compte := Compte{
        ID:       1,
        Nom:      "alice_doe",
        MotPasse: "secret123",  // Ne sera pas dans le JSON
        Email:    "alice@example.com",
        // Telephone vide - sera omis
        Solde: 1250.75,
        Actif: true,
        // Description vide - sera omise
    }

    jsonData, err := json.MarshalIndent(compte, "", "  ")
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("JSON du compte :\n%s\n", string(jsonData))
}
```

**Résultat :**
```json
{
  "id": 1,
  "username": "alice_doe",
  "email": "alice@example.com",
  "balance": "1250.75",
  "is_active": true
}
```

## Structures imbriquées

### Exemple avec objets complexes

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "time"
)

type Adresse struct {
    Rue       string `json:"rue"`
    Ville     string `json:"ville"`
    CodePostal string `json:"code_postal"`
    Pays      string `json:"pays"`
}

type Personne struct {
    ID           int       `json:"id"`
    Nom          string    `json:"nom"`
    Prenom       string    `json:"prenom"`
    DateNaissance time.Time `json:"date_naissance"`
    Adresse      Adresse   `json:"adresse"`
    Telephones   []string  `json:"telephones,omitempty"`
}

func main() {
    personne := Personne{
        ID:     1,
        Nom:    "Dupont",
        Prenom: "Marie",
        DateNaissance: time.Date(1990, 5, 15, 0, 0, 0, 0, time.UTC),
        Adresse: Adresse{
            Rue:       "123 Avenue de la Paix",
            Ville:     "Lyon",
            CodePostal: "69000",
            Pays:      "France",
        },
        Telephones: []string{"0123456789", "0987654321"},
    }

    // Sérialisation
    jsonData, err := json.MarshalIndent(personne, "", "  ")
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("=== Sérialisation ===")
    fmt.Printf("%s\n\n", string(jsonData))

    // Désérialisation
    var nouvellePersonne Personne
    err = json.Unmarshal(jsonData, &nouvellePersonne)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("=== Désérialisation ===")
    fmt.Printf("Nom complet : %s %s\n", nouvellePersonne.Prenom, nouvellePersonne.Nom)
    fmt.Printf("Ville : %s\n", nouvellePersonne.Adresse.Ville)
    fmt.Printf("Téléphones : %v\n", nouvellePersonne.Telephones)
}
```

## Gestion des dates

### Format de date personnalisé

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "time"
)

// Type personnalisé pour les dates
type DateCustom time.Time

const formatDate = "2006-01-02" // Format Go pour YYYY-MM-DD

// Implémentation de json.Marshaler
func (d DateCustom) MarshalJSON() ([]byte, error) {
    return json.Marshal(time.Time(d).Format(formatDate))
}

// Implémentation de json.Unmarshaler
func (d *DateCustom) UnmarshalJSON(data []byte) error {
    var dateStr string
    if err := json.Unmarshal(data, &dateStr); err != nil {
        return err
    }

    parsedTime, err := time.Parse(formatDate, dateStr)
    if err != nil {
        return err
    }

    *d = DateCustom(parsedTime)
    return nil
}

type Evenement struct {
    ID          int        `json:"id"`
    Titre       string     `json:"titre"`
    Date        DateCustom `json:"date"`
    Description string     `json:"description,omitempty"`
}

func main() {
    evenement := Evenement{
        ID:          1,
        Titre:       "Conférence Go",
        Date:        DateCustom(time.Date(2024, 12, 15, 0, 0, 0, 0, time.UTC)),
        Description: "Une conférence sur le langage Go",
    }

    // Sérialisation avec format de date personnalisé
    jsonData, err := json.MarshalIndent(evenement, "", "  ")
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("JSON avec date formatée :\n%s\n", string(jsonData))

    // Désérialisation
    jsonString := `{
        "id": 2,
        "titre": "Workshop JSON",
        "date": "2024-12-20",
        "description": "Apprendre JSON en Go"
    }`

    var nouvelEvenement Evenement
    err = json.Unmarshal([]byte(jsonString), &nouvelEvenement)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("\nÉvénement désérialisé : %+v\n", nouvelEvenement)
    fmt.Printf("Date convertie : %s\n", time.Time(nouvelEvenement.Date).Format("02/01/2006"))
}
```

## Interface vide et types dynamiques

### Utilisation de `interface{}`

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

func main() {
    // JSON avec structure inconnue
    jsonString := `{
        "nom": "Produit X",
        "prix": 99.99,
        "disponible": true,
        "tags": ["nouveau", "populaire"],
        "details": {
            "couleur": "rouge",
            "taille": "M"
        }
    }`

    // Utiliser interface{} pour une structure flexible
    var data interface{}
    err := json.Unmarshal([]byte(jsonString), &data)
    if err != nil {
        log.Fatal(err)
    }

    // Accéder aux données avec assertion de type
    if dataMap, ok := data.(map[string]interface{}); ok {
        fmt.Printf("Nom : %s\n", dataMap["nom"].(string))
        fmt.Printf("Prix : %.2f\n", dataMap["prix"].(float64))
        fmt.Printf("Disponible : %t\n", dataMap["disponible"].(bool))

        // Accéder aux tableaux
        if tags, ok := dataMap["tags"].([]interface{}); ok {
            fmt.Print("Tags : ")
            for i, tag := range tags {
                if i > 0 {
                    fmt.Print(", ")
                }
                fmt.Print(tag.(string))
            }
            fmt.Println()
        }

        // Accéder aux objets imbriqués
        if details, ok := dataMap["details"].(map[string]interface{}); ok {
            fmt.Printf("Couleur : %s\n", details["couleur"].(string))
            fmt.Printf("Taille : %s\n", details["taille"].(string))
        }
    }
}
```

### Fonction utilitaire pour navigation JSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

// Fonction utilitaire pour extraire des valeurs JSON
func getJSONValue(data interface{}, path ...string) interface{} {
    current := data

    for _, key := range path {
        if currentMap, ok := current.(map[string]interface{}); ok {
            current = currentMap[key]
        } else {
            return nil
        }
    }

    return current
}

// Fonction pour obtenir une string avec valeur par défaut
func getJSONString(data interface{}, defaultValue string, path ...string) string {
    if value := getJSONValue(data, path...); value != nil {
        if str, ok := value.(string); ok {
            return str
        }
    }
    return defaultValue
}

// Fonction pour obtenir un float64 avec valeur par défaut
func getJSONFloat(data interface{}, defaultValue float64, path ...string) float64 {
    if value := getJSONValue(data, path...); value != nil {
        if num, ok := value.(float64); ok {
            return num
        }
    }
    return defaultValue
}

func main() {
    jsonString := `{
        "produit": {
            "info": {
                "nom": "SuperWidget",
                "prix": 49.99
            },
            "stock": {
                "quantite": 100
            }
        }
    }`

    var data interface{}
    json.Unmarshal([]byte(jsonString), &data)

    // Utilisation des fonctions utilitaires
    nom := getJSONString(data, "Produit inconnu", "produit", "info", "nom")
    prix := getJSONFloat(data, 0.0, "produit", "info", "prix")
    quantite := getJSONFloat(data, 0, "produit", "stock", "quantite")

    fmt.Printf("Produit : %s\n", nom)
    fmt.Printf("Prix : %.2f€\n", prix)
    fmt.Printf("Quantité : %.0f\n", quantite)
}
```

## Streaming JSON

### Pour de gros fichiers JSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "strings"
)

type Message struct {
    ID      int    `json:"id"`
    Auteur  string `json:"auteur"`
    Contenu string `json:"contenu"`
}

func main() {
    // Simuler un gros fichier JSON avec plusieurs objets
    jsonStream := `
    {"id": 1, "auteur": "Alice", "contenu": "Premier message"}
    {"id": 2, "auteur": "Bob", "contenu": "Deuxième message"}
    {"id": 3, "auteur": "Charlie", "contenu": "Troisième message"}
    `

    // Créer un decoder pour le streaming
    decoder := json.NewDecoder(strings.NewReader(jsonStream))

    fmt.Println("=== Lecture en streaming ===")

    // Lire ligne par ligne
    for decoder.More() {
        var message Message
        if err := decoder.Decode(&message); err != nil {
            log.Printf("Erreur de décodage : %v", err)
            continue
        }

        fmt.Printf("Message %d de %s : %s\n",
            message.ID, message.Auteur, message.Contenu)
    }
}
```

### Encoder en streaming

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Utilisateur struct {
    ID   int    `json:"id"`
    Nom  string `json:"nom"`
    Role string `json:"role"`
}

func main() {
    // Créer un encoder vers stdout
    encoder := json.NewEncoder(os.Stdout)
    encoder.SetIndent("", "  ") // Indentation pour la lisibilité

    fmt.Println("=== Encodage en streaming ===")

    // Encoder plusieurs objets
    utilisateurs := []Utilisateur{
        {ID: 1, Nom: "Admin", Role: "administrateur"},
        {ID: 2, Nom: "User1", Role: "utilisateur"},
        {ID: 3, Nom: "Moderator", Role: "modérateur"},
    }

    for _, user := range utilisateurs {
        if err := encoder.Encode(user); err != nil {
            log.Printf("Erreur d'encodage : %v", err)
        }
    }
}
```

## Exemple pratique : API de configuration

Créons un système de configuration utilisant JSON :

```go
package main

import (
    "encoding/json"
    "fmt"
    "io/ioutil"
    "log"
    "os"
    "time"
)

// Configuration de l'application
type Config struct {
    Server   ServerConfig   `json:"server"`
    Database DatabaseConfig `json:"database"`
    Logging  LoggingConfig  `json:"logging"`
    Features FeatureFlags   `json:"features"`
}

type ServerConfig struct {
    Host         string        `json:"host"`
    Port         int           `json:"port"`
    ReadTimeout  time.Duration `json:"read_timeout"`
    WriteTimeout time.Duration `json:"write_timeout"`
    HTTPS        bool          `json:"https"`
}

type DatabaseConfig struct {
    Host         string `json:"host"`
    Port         int    `json:"port"`
    Username     string `json:"username"`
    Password     string `json:"-"`              // Ne pas sérialiser
    DatabaseName string `json:"database_name"`
    MaxPoolSize  int    `json:"max_pool_size,omitempty"`
}

type LoggingConfig struct {
    Level    string `json:"level"`
    FilePath string `json:"file_path,omitempty"`
    MaxSize  int    `json:"max_size_mb,omitempty"`
}

type FeatureFlags struct {
    EnableCache     bool `json:"enable_cache"`
    EnableMetrics   bool `json:"enable_metrics"`
    EnableRateLimit bool `json:"enable_rate_limit"`
}

// Charger la configuration depuis un fichier JSON
func LoadConfig(filename string) (*Config, error) {
    data, err := ioutil.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("erreur lecture fichier : %v", err)
    }

    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("erreur décodage JSON : %v", err)
    }

    return &config, nil
}

// Sauvegarder la configuration dans un fichier JSON
func (c *Config) Save(filename string) error {
    data, err := json.MarshalIndent(c, "", "  ")
    if err != nil {
        return fmt.Errorf("erreur encodage JSON : %v", err)
    }

    return ioutil.WriteFile(filename, data, 0644)
}

// Valider la configuration
func (c *Config) Validate() error {
    if c.Server.Port <= 0 || c.Server.Port > 65535 {
        return fmt.Errorf("port serveur invalide : %d", c.Server.Port)
    }

    if c.Database.Host == "" {
        return fmt.Errorf("host de base de données requis")
    }

    validLevels := map[string]bool{
        "debug": true, "info": true, "warn": true, "error": true,
    }
    if !validLevels[c.Logging.Level] {
        return fmt.Errorf("niveau de log invalide : %s", c.Logging.Level)
    }

    return nil
}

// Afficher la configuration (sans mots de passe)
func (c *Config) Display() {
    fmt.Println("=== Configuration de l'application ===")
    fmt.Printf("Serveur : %s:%d (HTTPS: %t)\n",
        c.Server.Host, c.Server.Port, c.Server.HTTPS)
    fmt.Printf("Base de données : %s@%s:%d/%s\n",
        c.Database.Username, c.Database.Host, c.Database.Port, c.Database.DatabaseName)
    fmt.Printf("Logging : niveau %s\n", c.Logging.Level)

    fmt.Println("Fonctionnalités activées :")
    if c.Features.EnableCache {
        fmt.Println("  ✓ Cache")
    }
    if c.Features.EnableMetrics {
        fmt.Println("  ✓ Métriques")
    }
    if c.Features.EnableRateLimit {
        fmt.Println("  ✓ Limitation de débit")
    }
}

func main() {
    configFile := "app_config.json"

    // Créer une configuration par défaut si le fichier n'existe pas
    if _, err := os.Stat(configFile); os.IsNotExist(err) {
        fmt.Println("Création de la configuration par défaut...")

        defaultConfig := &Config{
            Server: ServerConfig{
                Host:         "localhost",
                Port:         8080,
                ReadTimeout:  10 * time.Second,
                WriteTimeout: 10 * time.Second,
                HTTPS:        false,
            },
            Database: DatabaseConfig{
                Host:         "localhost",
                Port:         5432,
                Username:     "app_user",
                Password:     "secret123",
                DatabaseName: "myapp",
                MaxPoolSize:  20,
            },
            Logging: LoggingConfig{
                Level:    "info",
                FilePath: "/var/log/myapp.log",
                MaxSize:  100,
            },
            Features: FeatureFlags{
                EnableCache:     true,
                EnableMetrics:   true,
                EnableRateLimit: false,
            },
        }

        if err := defaultConfig.Save(configFile); err != nil {
            log.Fatal("Erreur sauvegarde config :", err)
        }

        fmt.Printf("Configuration sauvegardée dans %s\n\n", configFile)
    }

    // Charger la configuration
    config, err := LoadConfig(configFile)
    if err != nil {
        log.Fatal("Erreur chargement config :", err)
    }

    // Valider la configuration
    if err := config.Validate(); err != nil {
        log.Fatal("Configuration invalide :", err)
    }

    // Afficher la configuration
    config.Display()

    // Exemple de modification de configuration
    fmt.Println("\n=== Modification de la configuration ===")
    config.Features.EnableRateLimit = true
    config.Logging.Level = "debug"

    // Sauvegarder les modifications
    if err := config.Save(configFile); err != nil {
        log.Fatal("Erreur sauvegarde :", err)
    }

    fmt.Println("Configuration mise à jour et sauvegardée.")
}
```

## Bonnes pratiques

### 1. Gérer les erreurs de sérialisation
```go
func safeJSONMarshal(v interface{}) ([]byte, error) {
    data, err := json.Marshal(v)
    if err != nil {
        return nil, fmt.Errorf("erreur de sérialisation : %v", err)
    }
    return data, nil
}
```

### 2. Valider les données après désérialisation
```go
func parseAndValidateUser(jsonData []byte) (*User, error) {
    var user User
    if err := json.Unmarshal(jsonData, &user); err != nil {
        return nil, err
    }

    if user.Email == "" {
        return nil, fmt.Errorf("email requis")
    }

    if user.Age < 0 || user.Age > 150 {
        return nil, fmt.Errorf("âge invalide : %d", user.Age)
    }

    return &user, nil
}
```

### 3. Utiliser des pointeurs pour les champs optionnels
```go
type Profil struct {
    Nom         string  `json:"nom"`
    Age         *int    `json:"age,omitempty"`         // Optionnel
    Telephone   *string `json:"telephone,omitempty"`   // Optionnel
    Newsletter  *bool   `json:"newsletter,omitempty"`  // Optionnel
}
```

### 4. Créer des types personnalisés pour la validation
```go
type Email string

func (e Email) MarshalJSON() ([]byte, error) {
    // Valider l'email avant sérialisation
    if !strings.Contains(string(e), "@") {
        return nil, fmt.Errorf("email invalide : %s", e)
    }
    return json.Marshal(string(e))
}
```

## Exercices pratiques

### Exercice 1 : Carnet d'adresses JSON
Créez un programme qui gère un carnet d'adresses en JSON avec :
- Ajout de contacts
- Recherche par nom
- Sauvegarde/chargement depuis fichier

### Exercice 2 : Analyseur de logs JSON
Créez un programme qui lit des logs au format JSON et produit des statistiques.

### Exercice 3 : API de conversion
Créez une API qui convertit entre différents formats (JSON ↔ XML ↔ CSV).

## Résumé

Dans cette section, vous avez appris :

- **Concepts JSON** : Syntaxe, types de données, structure
- **Sérialisation** : `json.Marshal` et `json.MarshalIndent`
- **Désérialisation** : `json.Unmarshal` avec validation
- **Tags JSON** : Personnalisation des noms de champs, omission, conversion
- **Structures complexes** : Objets imbriqués, dates personnalisées
- **Types dynamiques** : `interface{}` pour JSON flexible
- **Streaming** : Traitement de gros fichiers JSON
- **Exemple pratique** : Système de configuration complet

Le package `encoding/json` de Go est très puissant et permet de travailler efficacement avec les données JSON. Ces compétences sont essentielles pour développer des APIs REST modernes.

---

*Félicitations ! Vous maîtrisez maintenant la sérialisation JSON en Go. Dans la section suivante, nous verrons comment créer des APIs REST complètes.*

⏭️

## Exercices pratiques

### Exercice 1 : Carnet d'adresses JSON
Créez un programme qui gère un carnet d'adresses en JSON avec :
- Ajout de contacts
- Recherche par nom
- Sauvegarde/chargement depuis fichier

**Solution complète :**
```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "io/ioutil"
    "log"
    "os"
    "strconv"
    "strings"
    "time"
)

// Structure pour un contact
type Contact struct {
    ID          int       `json:"id"`
    Nom         string    `json:"nom"`
    Prenom      string    `json:"prenom"`
    Email       string    `json:"email,omitempty"`
    Telephone   string    `json:"telephone,omitempty"`
    Adresse     Adresse   `json:"adresse,omitempty"`
    DateAjout   time.Time `json:"date_ajout"`
    Entreprise  string    `json:"entreprise,omitempty"`
    Notes       string    `json:"notes,omitempty"`
}

type Adresse struct {
    Rue        string `json:"rue,omitempty"`
    Ville      string `json:"ville,omitempty"`
    CodePostal string `json:"code_postal,omitempty"`
    Pays       string `json:"pays,omitempty"`
}

// Structure pour le carnet d'adresses
type CarnetAdresses struct {
    Contacts    []Contact `json:"contacts"`
    DernierID   int       `json:"dernier_id"`
    DateCreation time.Time `json:"date_creation"`
    fichier     string    // Non exporté, pas dans JSON
}

// Créer un nouveau carnet d'adresses
func NouveauCarnet(fichier string) *CarnetAdresses {
    return &CarnetAdresses{
        Contacts:     make([]Contact, 0),
        DernierID:    0,
        DateCreation: time.Now(),
        fichier:      fichier,
    }
}

// Charger le carnet depuis un fichier JSON
func ChargerCarnet(fichier string) (*CarnetAdresses, error) {
    if _, err := os.Stat(fichier); os.IsNotExist(err) {
        fmt.Printf("Fichier %s non trouvé, création d'un nouveau carnet.\n", fichier)
        return NouveauCarnet(fichier), nil
    }

    data, err := ioutil.ReadFile(fichier)
    if err != nil {
        return nil, fmt.Errorf("erreur lecture fichier : %v", err)
    }

    var carnet CarnetAdresses
    if err := json.Unmarshal(data, &carnet); err != nil {
        return nil, fmt.Errorf("erreur décodage JSON : %v", err)
    }

    carnet.fichier = fichier
    return &carnet, nil
}

// Sauvegarder le carnet dans un fichier JSON
func (c *CarnetAdresses) Sauvegarder() error {
    data, err := json.MarshalIndent(c, "", "  ")
    if err != nil {
        return fmt.Errorf("erreur encodage JSON : %v", err)
    }

    return ioutil.WriteFile(c.fichier, data, 0644)
}

// Ajouter un contact
func (c *CarnetAdresses) AjouterContact(contact Contact) {
    c.DernierID++
    contact.ID = c.DernierID
    contact.DateAjout = time.Now()
    c.Contacts = append(c.Contacts, contact)
    fmt.Printf("✅ Contact %s %s ajouté avec l'ID %d\n", contact.Prenom, contact.Nom, contact.ID)
}

// Rechercher des contacts par nom ou prénom
func (c *CarnetAdresses) RechercherContacts(terme string) []Contact {
    var resultats []Contact
    terme = strings.ToLower(terme)

    for _, contact := range c.Contacts {
        nom := strings.ToLower(contact.Nom)
        prenom := strings.ToLower(contact.Prenom)

        if strings.Contains(nom, terme) || strings.Contains(prenom, terme) {
            resultats = append(resultats, contact)
        }
    }

    return resultats
}

// Rechercher un contact par ID
func (c *CarnetAdresses) RechercherParID(id int) *Contact {
    for i, contact := range c.Contacts {
        if contact.ID == id {
            return &c.Contacts[i]
        }
    }
    return nil
}

// Supprimer un contact par ID
func (c *CarnetAdresses) SupprimerContact(id int) bool {
    for i, contact := range c.Contacts {
        if contact.ID == id {
            c.Contacts = append(c.Contacts[:i], c.Contacts[i+1:]...)
            fmt.Printf("✅ Contact %s %s supprimé\n", contact.Prenom, contact.Nom)
            return true
        }
    }
    return false
}

// Afficher un contact
func (c *Contact) Afficher() {
    fmt.Printf("\n📋 Contact ID: %d\n", c.ID)
    fmt.Printf("👤 Nom: %s %s\n", c.Prenom, c.Nom)

    if c.Email != "" {
        fmt.Printf("📧 Email: %s\n", c.Email)
    }
    if c.Telephone != "" {
        fmt.Printf("📞 Téléphone: %s\n", c.Telephone)
    }
    if c.Entreprise != "" {
        fmt.Printf("🏢 Entreprise: %s\n", c.Entreprise)
    }

    // Afficher l'adresse si elle existe
    if c.Adresse.Rue != "" || c.Adresse.Ville != "" {
        fmt.Printf("🏠 Adresse: ")
        if c.Adresse.Rue != "" {
            fmt.Printf("%s, ", c.Adresse.Rue)
        }
        if c.Adresse.Ville != "" {
            fmt.Printf("%s", c.Adresse.Ville)
        }
        if c.Adresse.CodePostal != "" {
            fmt.Printf(" %s", c.Adresse.CodePostal)
        }
        if c.Adresse.Pays != "" {
            fmt.Printf(", %s", c.Adresse.Pays)
        }
        fmt.Println()
    }

    if c.Notes != "" {
        fmt.Printf("📝 Notes: %s\n", c.Notes)
    }

    fmt.Printf("📅 Ajouté le: %s\n", c.DateAjout.Format("02/01/2006 15:04"))
}

// Afficher les statistiques du carnet
func (c *CarnetAdresses) AfficherStatistiques() {
    fmt.Printf("\n📊 Statistiques du carnet d'adresses\n")
    fmt.Printf("════════════════════════════════════\n")
    fmt.Printf("📋 Nombre total de contacts: %d\n", len(c.Contacts))
    fmt.Printf("📅 Carnet créé le: %s\n", c.DateCreation.Format("02/01/2006 15:04"))

    // Compter les contacts avec email
    contactsAvecEmail := 0
    for _, contact := range c.Contacts {
        if contact.Email != "" {
            contactsAvecEmail++
        }
    }
    fmt.Printf("📧 Contacts avec email: %d\n", contactsAvecEmail)

    // Compter par entreprise
    entreprises := make(map[string]int)
    for _, contact := range c.Contacts {
        if contact.Entreprise != "" {
            entreprises[contact.Entreprise]++
        }
    }

    if len(entreprises) > 0 {
        fmt.Printf("\n🏢 Répartition par entreprise:\n")
        for entreprise, count := range entreprises {
            fmt.Printf("   %s: %d contact(s)\n", entreprise, count)
        }
    }
}

// Interface utilisateur en ligne de commande
func main() {
    fmt.Println("📇 Carnet d'Adresses JSON")
    fmt.Println("========================")

    // Charger le carnet existant ou en créer un nouveau
    carnet, err := ChargerCarnet("carnet_adresses.json")
    if err != nil {
        log.Fatal("Erreur lors du chargement:", err)
    }

    reader := bufio.NewReader(os.Stdin)

    for {
        fmt.Println("\n📋 Menu:")
        fmt.Println("1. Ajouter un contact")
        fmt.Println("2. Rechercher des contacts")
        fmt.Println("3. Afficher tous les contacts")
        fmt.Println("4. Modifier un contact")
        fmt.Println("5. Supprimer un contact")
        fmt.Println("6. Afficher les statistiques")
        fmt.Println("7. Sauvegarder et quitter")
        fmt.Print("\nChoisissez une option (1-7): ")

        choix, _ := reader.ReadString('\n')
        choix = strings.TrimSpace(choix)

        switch choix {
        case "1":
            ajouterContactInteractif(carnet, reader)
        case "2":
            rechercherContactsInteractif(carnet, reader)
        case "3":
            afficherTousLesContacts(carnet)
        case "4":
            modifierContactInteractif(carnet, reader)
        case "5":
            supprimerContactInteractif(carnet, reader)
        case "6":
            carnet.AfficherStatistiques()
        case "7":
            if err := carnet.Sauvegarder(); err != nil {
                fmt.Printf("❌ Erreur lors de la sauvegarde: %v\n", err)
            } else {
                fmt.Println("✅ Carnet sauvegardé avec succès!")
            }
            fmt.Println("👋 Au revoir!")
            return
        default:
            fmt.Println("❌ Option invalide. Veuillez choisir entre 1 et 7.")
        }
    }
}

func ajouterContactInteractif(carnet *CarnetAdresses, reader *bufio.Reader) {
    fmt.Println("\n➕ Ajouter un nouveau contact")
    fmt.Println("=============================")

    var contact Contact

    fmt.Print("Prénom: ")
    contact.Prenom, _ = reader.ReadString('\n')
    contact.Prenom = strings.TrimSpace(contact.Prenom)

    fmt.Print("Nom: ")
    contact.Nom, _ = reader.ReadString('\n')
    contact.Nom = strings.TrimSpace(contact.Nom)

    if contact.Prenom == "" && contact.Nom == "" {
        fmt.Println("❌ Au moins un nom ou prénom est requis.")
        return
    }

    fmt.Print("Email (optionnel): ")
    contact.Email, _ = reader.ReadString('\n')
    contact.Email = strings.TrimSpace(contact.Email)

    fmt.Print("Téléphone (optionnel): ")
    contact.Telephone, _ = reader.ReadString('\n')
    contact.Telephone = strings.TrimSpace(contact.Telephone)

    fmt.Print("Entreprise (optionnel): ")
    contact.Entreprise, _ = reader.ReadString('\n')
    contact.Entreprise = strings.TrimSpace(contact.Entreprise)

    // Adresse optionnelle
    fmt.Print("Adresse - Rue (optionnel): ")
    contact.Adresse.Rue, _ = reader.ReadString('\n')
    contact.Adresse.Rue = strings.TrimSpace(contact.Adresse.Rue)

    if contact.Adresse.Rue != "" {
        fmt.Print("Ville: ")
        contact.Adresse.Ville, _ = reader.ReadString('\n')
        contact.Adresse.Ville = strings.TrimSpace(contact.Adresse.Ville)

        fmt.Print("Code postal: ")
        contact.Adresse.CodePostal, _ = reader.ReadString('\n')
        contact.Adresse.CodePostal = strings.TrimSpace(contact.Adresse.CodePostal)

        fmt.Print("Pays: ")
        contact.Adresse.Pays, _ = reader.ReadString('\n')
        contact.Adresse.Pays = strings.TrimSpace(contact.Adresse.Pays)
    }

    fmt.Print("Notes (optionnel): ")
    contact.Notes, _ = reader.ReadString('\n')
    contact.Notes = strings.TrimSpace(contact.Notes)

    carnet.AjouterContact(contact)
}

func rechercherContactsInteractif(carnet *CarnetAdresses, reader *bufio.Reader) {
    fmt.Print("\n🔍 Rechercher des contacts par nom: ")
    terme, _ := reader.ReadString('\n')
    terme = strings.TrimSpace(terme)

    if terme == "" {
        fmt.Println("❌ Veuillez entrer un terme de recherche.")
        return
    }

    resultats := carnet.RechercherContacts(terme)

    if len(resultats) == 0 {
        fmt.Printf("❌ Aucun contact trouvé pour '%s'.\n", terme)
        return
    }

    fmt.Printf("\n✅ %d contact(s) trouvé(s) pour '%s':\n", len(resultats), terme)
    for _, contact := range resultats {
        contact.Afficher()
    }
}

func afficherTousLesContacts(carnet *CarnetAdresses) {
    if len(carnet.Contacts) == 0 {
        fmt.Println("\n📭 Aucun contact dans le carnet.")
        return
    }

    fmt.Printf("\n📋 Tous les contacts (%d):\n", len(carnet.Contacts))
    for _, contact := range carnet.Contacts {
        contact.Afficher()
    }
}

func modifierContactInteractif(carnet *CarnetAdresses, reader *bufio.Reader) {
    fmt.Print("\n✏️ ID du contact à modifier: ")
    idStr, _ := reader.ReadString('\n')
    idStr = strings.TrimSpace(idStr)

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("❌ ID invalide.")
        return
    }

    contact := carnet.RechercherParID(id)
    if contact == nil {
        fmt.Printf("❌ Contact avec l'ID %d non trouvé.\n", id)
        return
    }

    fmt.Println("\nContact actuel:")
    contact.Afficher()

    fmt.Println("\nLaissez vide pour conserver la valeur actuelle.")

    fmt.Printf("Nouveau prénom [%s]: ", contact.Prenom)
    prenom, _ := reader.ReadString('\n')
    prenom = strings.TrimSpace(prenom)
    if prenom != "" {
        contact.Prenom = prenom
    }

    fmt.Printf("Nouveau nom [%s]: ", contact.Nom)
    nom, _ := reader.ReadString('\n')
    nom = strings.TrimSpace(nom)
    if nom != "" {
        contact.Nom = nom
    }

    fmt.Printf("Nouvel email [%s]: ", contact.Email)
    email, _ := reader.ReadString('\n')
    email = strings.TrimSpace(email)
    if email != "" {
        contact.Email = email
    }

    fmt.Println("✅ Contact modifié avec succès.")
}

func supprimerContactInteractif(carnet *CarnetAdresses, reader *bufio.Reader) {
    fmt.Print("\n🗑️ ID du contact à supprimer: ")
    idStr, _ := reader.ReadString('\n')
    idStr = strings.TrimSpace(idStr)

    id, err := strconv.Atoi(idStr)
    if err != nil {
        fmt.Println("❌ ID invalide.")
        return
    }

    contact := carnet.RechercherParID(id)
    if contact == nil {
        fmt.Printf("❌ Contact avec l'ID %d non trouvé.\n", id)
        return
    }

    fmt.Println("\nContact à supprimer:")
    contact.Afficher()

    fmt.Print("\nÊtes-vous sûr de vouloir supprimer ce contact? (oui/non): ")
    confirmation, _ := reader.ReadString('\n')
    confirmation = strings.ToLower(strings.TrimSpace(confirmation))

    if confirmation == "oui" || confirmation == "o" {
        if carnet.SupprimerContact(id) {
            fmt.Println("✅ Contact supprimé avec succès.")
        } else {
            fmt.Println("❌ Erreur lors de la suppression.")
        }
    } else {
        fmt.Println("❌ Suppression annulée.")
    }
}
```

### Exercice 2 : Analyseur de logs JSON
Créez un programme qui lit des logs au format JSON et produit des statistiques.

**Solution complète :**
```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "io"
    "log"
    "os"
    "sort"
    "strings"
    "time"
)

// Structure pour une entrée de log
type LogEntry struct {
    Timestamp   time.Time `json:"timestamp"`
    Level       string    `json:"level"`
    Message     string    `json:"message"`
    Service     string    `json:"service,omitempty"`
    UserID      string    `json:"user_id,omitempty"`
    IP          string    `json:"ip,omitempty"`
    StatusCode  int       `json:"status_code,omitempty"`
    ResponseTime int      `json:"response_time_ms,omitempty"`
    URL         string    `json:"url,omitempty"`
    Method      string    `json:"method,omitempty"`
    Error       string    `json:"error,omitempty"`
}

// Structure pour les statistiques
type LogStats struct {
    TotalEntries    int                    `json:"total_entries"`
    LevelCounts     map[string]int         `json:"level_counts"`
    ServiceCounts   map[string]int         `json:"service_counts"`
    StatusCodes     map[int]int            `json:"status_codes"`
    ErrorMessages   map[string]int         `json:"error_messages"`
    TopUsers        map[string]int         `json:"top_users"`
    TopIPs          map[string]int         `json:"top_ips"`
    TopURLs         map[string]int         `json:"top_urls"`
    HourlyActivity  map[int]int            `json:"hourly_activity"`
    AvgResponseTime float64                `json:"avg_response_time_ms"`
    TimeRange       struct {
        Start time.Time `json:"start"`
        End   time.Time `json:"end"`
    } `json:"time_range"`
}

// Analyseur de logs
type LogAnalyzer struct {
    stats LogStats
}

// Créer un nouvel analyseur
func NewLogAnalyzer() *LogAnalyzer {
    return &LogAnalyzer{
        stats: LogStats{
            LevelCounts:    make(map[string]int),
            ServiceCounts:  make(map[string]int),
            StatusCodes:    make(map[int]int),
            ErrorMessages:  make(map[string]int),
            TopUsers:       make(map[string]int),
            TopIPs:         make(map[string]int),
            TopURLs:        make(map[string]int),
            HourlyActivity: make(map[int]int),
        },
    }
}

// Analyser un fichier de logs
func (la *LogAnalyzer) AnalyzeFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return fmt.Errorf("erreur ouverture fichier : %v", err)
    }
    defer file.Close()

    return la.AnalyzeReader(file)
}

// Analyser depuis un Reader
func (la *LogAnalyzer) AnalyzeReader(reader io.Reader) error {
    scanner := bufio.NewScanner(reader)
    totalResponseTime := 0
    responseTimeCount := 0

    for scanner.Scan() {
        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }

        var entry LogEntry
        if err := json.Unmarshal([]byte(line), &entry); err != nil {
            fmt.Printf("⚠️ Ligne ignorée (JSON invalide) : %s\n", line)
            continue
        }

        la.processEntry(entry)

        // Calculer la moyenne des temps de réponse
        if entry.ResponseTime > 0 {
            totalResponseTime += entry.ResponseTime
            responseTimeCount++
        }
    }

    // Calculer la moyenne des temps de réponse
    if responseTimeCount > 0 {
        la.stats.AvgResponseTime = float64(totalResponseTime) / float64(responseTimeCount)
    }

    return scanner.Err()
}

// Traiter une entrée de log
func (la *LogAnalyzer) processEntry(entry LogEntry) {
    la.stats.TotalEntries++

    // Définir la plage de temps
    if la.stats.TimeRange.Start.IsZero() || entry.Timestamp.Before(la.stats.TimeRange.Start) {
        la.stats.TimeRange.Start = entry.Timestamp
    }
    if la.stats.TimeRange.End.IsZero() || entry.Timestamp.After(la.stats.TimeRange.End) {
        la.stats.TimeRange.End = entry.Timestamp
    }

    // Compter par niveau
    la.stats.LevelCounts[entry.Level]++

    // Compter par service
    if entry.Service != "" {
        la.stats.ServiceCounts[entry.Service]++
    }

    // Compter les codes de statut
    if entry.StatusCode > 0 {
        la.stats.StatusCodes[entry.StatusCode]++
    }

    // Compter les erreurs
    if entry.Error != "" {
        la.stats.ErrorMessages[entry.Error]++
    }

    // Compter les utilisateurs
    if entry.UserID != "" {
        la.stats.TopUsers[entry.UserID]++
    }

    // Compter les IPs
    if entry.IP != "" {
        la.stats.TopIPs[entry.IP]++
    }

    // Compter les URLs
    if entry.URL != "" {
        la.stats.TopURLs[entry.URL]++
    }

    // Activité par heure
    hour := entry.Timestamp.Hour()
    la.stats.HourlyActivity[hour]++
}

// Afficher les statistiques
func (la *LogAnalyzer) DisplayStats() {
    fmt.Printf("\n📊 ANALYSE DES LOGS\n")
    fmt.Printf("═══════════════════\n\n")

    // Informations générales
    fmt.Printf("📋 Entrées totales : %d\n", la.stats.TotalEntries)
    if !la.stats.TimeRange.Start.IsZero() {
        fmt.Printf("📅 Période : %s → %s\n",
            la.stats.TimeRange.Start.Format("2006-01-02 15:04:05"),
            la.stats.TimeRange.End.Format("2006-01-02 15:04:05"))

        duration := la.stats.TimeRange.End.Sub(la.stats.TimeRange.Start)
        fmt.Printf("⏱️ Durée : %v\n", duration.Round(time.Second))
    }

    if la.stats.AvgResponseTime > 0 {
        fmt.Printf("⚡ Temps de réponse moyen : %.2f ms\n", la.stats.AvgResponseTime)
    }

    // Niveaux de log
    fmt.Printf("\n🎯 Répartition par niveau :\n")
    la.displayTopItems(la.stats.LevelCounts, 10)

    // Services
    if len(la.stats.ServiceCounts) > 0 {
        fmt.Printf("\n🔧 Répartition par service :\n")
        la.displayTopItems(la.stats.ServiceCounts, 10)
    }

    // Codes de statut HTTP
    if len(la.stats.StatusCodes) > 0 {
        fmt.Printf("\n🌐 Codes de statut HTTP :\n")
        statusMap := make(map[string]int)
        for code, count := range la.stats.StatusCodes {
            statusMap[fmt.Sprintf("%d", code)] = count
        }
        la.displayTopItems(statusMap, 10)
    }

    // Top erreurs
    if len(la.stats.ErrorMessages) > 0 {
        fmt.Printf("\n❌ Erreurs les plus fréquentes :\n")
        la.displayTopItems(la.stats.ErrorMessages, 5)
    }

    // Top utilisateurs
    if len(la.stats.TopUsers) > 0 {
        fmt.Printf("\n👥 Utilisateurs les plus actifs :\n")
        la.displayTopItems(la.stats.TopUsers, 10)
    }

    // Top IPs
    if len(la.stats.TopIPs) > 0 {
        fmt.Printf("\n🌍 Adresses IP les plus actives :\n")
        la.displayTopItems(la.stats.TopIPs, 10)
    }

    // Top URLs
    if len(la.stats.TopURLs) > 0 {
        fmt.Printf("\n🔗 URLs les plus demandées :\n")
        la.displayTopItems(la.stats.TopURLs, 10)
    }

    // Activité par heure
    if len(la.stats.HourlyActivity) > 0 {
        fmt.Printf("\n🕐 Activité par heure :\n")
        la.displayHourlyActivity()
    }
}

// Afficher les éléments les plus fréquents
func (la *LogAnalyzer) displayTopItems(items map[string]int, limit int) {
    type item struct {
        key   string
        count int
    }

    var sorted []item
    for k, v := range items {
        sorted = append(sorted, item{k, v})
    }

    sort.Slice(sorted, func(i, j int) bool {
        return sorted[i].count > sorted[j].count
    })

    for i, item := range sorted {
        if i >= limit {
            break
        }
        percentage := float64(item.count) / float64(la.stats.TotalEntries) * 100
        fmt.Printf("   %s : %d (%.1f%%)\n", item.key, item.count, percentage)
    }
}

// Afficher l'activité par heure sous forme de graphique ASCII
func (la *LogAnalyzer) displayHourlyActivity() {
    maxCount := 0
    for _, count := range la.stats.HourlyActivity {
        if count > maxCount {
            maxCount = count
        }
    }

    for hour := 0; hour < 24; hour++ {
        count := la.stats.HourlyActivity[hour]
        percentage := float64(count) / float64(maxCount)
        barLength := int(percentage * 50) // Barre de 50 caractères max

        bar := strings.Repeat("█", barLength)
        fmt.Printf("   %02d:00 │%s %d\n", hour,
            fmt.Sprintf("%-50s", bar), count)
    }
}

// Sauvegarder les statistiques en JSON
func (la *LogAnalyzer) SaveStats(filename string) error {
    data, err := json.MarshalIndent(la.stats, "", "  ")
    if err != nil {
        return err
    }

    return os.WriteFile(filename, data, 0644)
}

// Générer des logs d'exemple
func generateSampleLogs(filename string, count int) error {
    file, err := os.Create(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    services := []string{"web", "api", "auth", "database", "cache"}
    levels := []string{"INFO", "DEBUG", "WARN", "ERROR"}
    methods := []string{"GET", "POST", "PUT", "DELETE"}
    urls := []string{"/api/users", "/api/products", "/login", "/dashboard", "/api/orders"}
    ips := []string{"192.168.1.1", "10.0.0.5", "172.16.0.10", "203.0.113.45"}

    for i := 0; i < count; i++ {
        entry := LogEntry{
            Timestamp:    time.Now().Add(-time.Duration(count-i) * time.Minute),
            Level:        levels[i%len(levels)],
            Service:      services[i%len(services)],
            Message:      fmt.Sprintf("Message de log numéro %d", i+1),
            UserID:       fmt.Sprintf("user_%d", (i%10)+1),
            IP:           ips[i%len(ips)],
            Method:       methods[i%len(methods)],
            URL:          urls[i%len(urls)],
            StatusCode:   200 + (i%4)*100, // 200, 300, 400, 500
            ResponseTime: 50 + (i % 200),
        }

        if entry.Level == "ERROR" {
            entry.Error = fmt.Sprintf("Erreur example %d", (i%3)+1)
        }

        jsonData, _ := json.Marshal(entry)
        file.Write(jsonData)
        file.WriteString("\n")
    }

    return nil
}

func main() {
    if len(os.Args) < 2 {
        fmt.Println("📊 Analyseur de Logs JSON")
        fmt.Println("Usage: go run log_analyzer.go <fichier_logs.json>")
        fmt.Println("\nOptions:")
        fmt.Println("  --generate <nombre>  Générer des logs d'exemple")
        fmt.Println("  --save-stats <fichier>  Sauvegarder les stats en JSON")

        // Générer des logs d'exemple si aucun fichier n'est fourni
        fmt.Print("\nVoulez-vous générer des logs d'exemple? (oui/non): ")
        var response string
        fmt.Scanln(&response)

        if strings.ToLower(response) == "oui" || strings.ToLower(response) == "o" {
            fmt.Println("Génération de 1000 logs d'exemple...")
            if err := generateSampleLogs("sample_logs.json", 1000); err != nil {
                log.Fatal("Erreur génération logs:", err)
            }
            fmt.Println("✅ Logs générés dans sample_logs.json")
            os.Args = append(os.Args, "sample_logs.json")
        } else {
            return
        }
    }

    filename := os.Args[1]

    // Vérifier les options
    saveStatsFile := ""
    for i, arg := range os.Args {
        if arg == "--save-stats" && i+1 < len(os.Args) {
            saveStatsFile = os.Args[i+1]
        }
    }

    fmt.Printf("📊 Analyse du fichier: %s\n", filename)

    analyzer := NewLogAnalyzer()

    start := time.Now()
    if err := analyzer.AnalyzeFile(filename); err != nil {
        log.Fatal("Erreur analyse:", err)
    }
    analysisTime := time.Since(start)

    analyzer.DisplayStats()

    fmt.Printf("\n⏱️ Temps d'analyse : %v\n", analysisTime)

    // Sauvegarder les statistiques si demandé
    if saveStatsFile != "" {
        if err := analyzer.SaveStats(saveStatsFile); err != nil {
            fmt.Printf("❌ Erreur sauvegarde stats: %v\n", err)
        } else {
            fmt.Printf("✅ Statistiques sauvegardées dans %s\n", saveStatsFile)
        }
    }
}
```

### Exercice 3 : API de conversion
Créez une API qui convertit entre différents formats (JSON ↔ XML ↔ CSV).

**Solution complète :**
```go
package main

import (
    "encoding/csv"
    "encoding/json"
    "encoding/xml"
    "fmt"
    "io"
    "net/http"
    "reflect"
    "strconv"
    "strings"
    "time"
)

// Structure générique pour les données
type DataRecord map[string]interface{}
type DataSet []DataRecord

// Structure pour la réponse API
type ConversionResponse struct {
    Success     bool        `json:"success" xml:"success"`
    Message     string      `json:"message,omitempty" xml:"message,omitempty"`
    Data        interface{} `json:"data,omitempty" xml:"data,omitempty"`
    SourceFormat string     `json:"source_format" xml:"source_format"`
    TargetFormat string     `json:"target_format" xml:"target_format"`
    RecordCount  int        `json:"record_count" xml:"record_count"`
    Timestamp    time.Time  `json:"timestamp" xml:"timestamp"`
}

// Structure XML wrapper pour les datasets
type XMLDataSet struct {
    XMLName xml.Name    `xml:"dataset"`
    Records []XMLRecord `xml:"record"`
}

type XMLRecord struct {
    XMLName xml.Name      `xml:"record"`
    Fields  []XMLField    `xml:"field"`
}

type XMLField struct {
    XMLName xml.Name `xml:"field"`
    Name    string   `xml:"name,attr"`
    Value   string   `xml:",chardata"`
}

// Convertisseur principal
type FormatConverter struct{}

// Convertir JSON vers CSV
func (fc *FormatConverter) JSONToCSV(jsonData []byte) (string, error) {
    var data DataSet
    if err := json.Unmarshal(jsonData, &data); err != nil {
        return "", fmt.Errorf("erreur parsing JSON: %v", err)
    }

    if len(data) == 0 {
        return "", fmt.Errorf("données vides")
    }

    // Extraire toutes les clés uniques
    allKeys := make(map[string]bool)
    for _, record := range data {
        for key := range record {
            allKeys[key] = true
        }
    }

    // Créer la liste des headers
    var headers []string
    for key := range allKeys {
        headers = append(headers, key)
    }

    // Construire le CSV
    var csvBuilder strings.Builder

    // Headers
    csvBuilder.WriteString(strings.Join(headers, ","))
    csvBuilder.WriteString("\n")

    // Données
    for _, record := range data {
        var values []string
        for _, header := range headers {
            value := ""
            if val, exists := record[header]; exists && val != nil {
                value = fmt.Sprintf("%v", val)
                // Échapper les guillemets et virgules
                if strings.Contains(value, ",") || strings.Contains(value, "\"") {
                    value = "\"" + strings.ReplaceAll(value, "\"", "\"\"") + "\""
                }
            }
            values = append(values, value)
        }
        csvBuilder.WriteString(strings.Join(values, ","))
        csvBuilder.WriteString("\n")
    }

    return csvBuilder.String(), nil
}

// Convertir CSV vers JSON
func (fc *FormatConverter) CSVToJSON(csvData string) ([]byte, error) {
    reader := csv.NewReader(strings.NewReader(csvData))
    records, err := reader.ReadAll()
    if err != nil {
        return nil, fmt.Errorf("erreur parsing CSV: %v", err)
    }

    if len(records) < 2 {
        return nil, fmt.Errorf("CSV doit avoir au moins 2 lignes (headers + données)")
    }

    headers := records[0]
    var data DataSet

    for i := 1; i < len(records); i++ {
        record := make(DataRecord)
        for j, value := range records[i] {
            if j < len(headers) {
                // Tenter de convertir en nombre si possible
                if numValue, err := strconv.ParseFloat(value, 64); err == nil {
                    record[headers[j]] = numValue
                } else if boolValue, err := strconv.ParseBool(value); err == nil {
                    record[headers[j]] = boolValue
                } else {
                    record[headers[j]] = value
                }
            }
        }
        data = append(data, record)
    }

    return json.MarshalIndent(data, "", "  ")
}

// Convertir JSON vers XML
func (fc *FormatConverter) JSONToXML(jsonData []byte) ([]byte, error) {
    var data DataSet
    if err := json.Unmarshal(jsonData, &data); err != nil {
        return nil, fmt.Errorf("erreur parsing JSON: %v", err)
    }

    xmlData := XMLDataSet{}

    for _, record := range data {
        xmlRecord := XMLRecord{}
        for key, value := range record {
            field := XMLField{
                Name:  key,
                Value: fmt.Sprintf("%v", value),
            }
            xmlRecord.Fields = append(xmlRecord.Fields, field)
        }
        xmlData.Records = append(xmlData.Records, xmlRecord)
    }

    output, err := xml.MarshalIndent(xmlData, "", "  ")
    if err != nil {
        return nil, fmt.Errorf("erreur génération XML: %v", err)
    }

    return append([]byte(xml.Header), output...), nil
}

// Convertir XML vers JSON
func (fc *FormatConverter) XMLToJSON(xmlData []byte) ([]byte, error) {
    var xmlDataSet XMLDataSet
    if err := xml.Unmarshal(xmlData, &xmlDataSet); err != nil {
        return nil, fmt.Errorf("erreur parsing XML: %v", err)
    }

    var data DataSet

    for _, xmlRecord := range xmlDataSet.Records {
        record := make(DataRecord)
        for _, field := range xmlRecord.Fields {
            // Tenter de convertir les types
            if numValue, err := strconv.ParseFloat(field.Value, 64); err == nil {
                record[field.Name] = numValue
            } else if boolValue, err := strconv.ParseBool(field.Value); err == nil {
                record[field.Name] = boolValue
            } else {
                record[field.Name] = field.Value
            }
        }
        data = append(data, record)
    }

    return json.MarshalIndent(data, "", "  ")
}

// Convertir XML vers CSV
func (fc *FormatConverter) XMLToCSV(xmlData []byte) (string, error) {
    jsonData, err := fc.XMLToJSON(xmlData)
    if err != nil {
        return "", err
    }
    return fc.JSONToCSV(jsonData)
}

// Convertir CSV vers XML
func (fc *FormatConverter) CSVToXML(csvData string) ([]byte, error) {
    jsonData, err := fc.CSVToJSON(csvData)
    if err != nil {
        return nil, err
    }
    return fc.JSONToXML(jsonData)
}

// Détecter le format automatiquement
func (fc *FormatConverter) DetectFormat(data []byte) string {
    dataStr := strings.TrimSpace(string(data))

    // Vérifier XML
    if strings.HasPrefix(dataStr, "<?xml") || strings.HasPrefix(dataStr, "<") {
        return "xml"
    }

    // Vérifier JSON
    if (strings.HasPrefix(dataStr, "{") && strings.HasSuffix(dataStr, "}")) ||
       (strings.HasPrefix(dataStr, "[") && strings.HasSuffix(dataStr, "]")) {
        return "json"
    }

    // Par défaut, considérer comme CSV
    return "csv"
}

// Handler principal de conversion
func (fc *FormatConverter) convertHandler(w http.ResponseWriter, r *http.Request) {
    if r.Method != "POST" {
        http.Error(w, "Méthode non autorisée", http.StatusMethodNotAllowed)
        return
    }

    // Lire les données
    body, err := io.ReadAll(r.Body)
    if err != nil {
        fc.sendErrorResponse(w, "Erreur lecture données", err)
        return
    }

    // Paramètres de conversion
    sourceFormat := r.Header.Get("Source-Format")
    targetFormat := r.Header.Get("Target-Format")

    // Détecter automatiquement le format source si non spécifié
    if sourceFormat == "" {
        sourceFormat = fc.DetectFormat(body)
    }

    // Format cible par défaut
    if targetFormat == "" {
        targetFormat = "json"
    }

    // Effectuer la conversion
    var result interface{}
    var resultStr string
    var resultBytes []byte

    switch sourceFormat + "_to_" + targetFormat {
    case "json_to_csv":
        resultStr, err = fc.JSONToCSV(body)
        result = resultStr
    case "csv_to_json":
        resultBytes, err = fc.CSVToJSON(string(body))
        result = json.RawMessage(resultBytes)
    case "json_to_xml":
        resultBytes, err = fc.JSONToXML(body)
        result = string(resultBytes)
    case "xml_to_json":
        resultBytes, err = fc.XMLToJSON(body)
        result = json.RawMessage(resultBytes)
    case "xml_to_csv":
        resultStr, err = fc.XMLToCSV(body)
        result = resultStr
    case "csv_to_xml":
        resultBytes, err = fc.CSVToXML(string(body))
        result = string(resultBytes)
    case "json_to_json", "csv_to_csv", "xml_to_xml":
        result = string(body)
    default:
        fc.sendErrorResponse(w, fmt.Sprintf("Conversion non supportée: %s vers %s", sourceFormat, targetFormat), nil)
        return
    }

    if err != nil {
        fc.sendErrorResponse(w, "Erreur de conversion", err)
        return
    }

    // Compter les enregistrements
    recordCount := fc.countRecords(result, targetFormat)

    // Réponse de succès
    response := ConversionResponse{
        Success:      true,
        Data:         result,
        SourceFormat: sourceFormat,
        TargetFormat: targetFormat,
        RecordCount:  recordCount,
        Timestamp:    time.Now(),
    }

    // Définir le Content-Type selon le format cible
    switch targetFormat {
    case "json":
        w.Header().Set("Content-Type", "application/json")
    case "xml":
        w.Header().Set("Content-Type", "application/xml")
    case "csv":
        w.Header().Set("Content-Type", "text/csv")
    }

    json.NewEncoder(w).Encode(response)
}

// Compter les enregistrements
func (fc *FormatConverter) countRecords(data interface{}, format string) int {
    switch format {
    case "json":
        if rawMsg, ok := data.(json.RawMessage); ok {
            var dataset []interface{}
            if err := json.Unmarshal(rawMsg, &dataset); err == nil {
                return len(dataset)
            }
        }
    case "csv":
        if csvStr, ok := data.(string); ok {
            lines := strings.Split(strings.TrimSpace(csvStr), "\n")
            if len(lines) > 1 {
                return len(lines) - 1 // Exclure les headers
            }
        }
    case "xml":
        if xmlStr, ok := data.(string); ok {
            return strings.Count(xmlStr, "<record>")
        }
    }
    return 0
}

// Envoyer une réponse d'erreur
func (fc *FormatConverter) sendErrorResponse(w http.ResponseWriter, message string, err error) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusBadRequest)

    errorMsg := message
    if err != nil {
        errorMsg += ": " + err.Error()
    }

    response := ConversionResponse{
        Success:   false,
        Message:   errorMsg,
        Timestamp: time.Now(),
    }

    json.NewEncoder(w).Encode(response)
}

// Page de documentation
func (fc *FormatConverter) docsHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>API de Conversion de Formats</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
        .container { max-width: 800px; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #333; text-align: center; }
        .endpoint { background: #f8f9fa; padding: 15px; margin: 20px 0; border-radius: 5px; border-left: 4px solid #007bff; }
        .example { background: #e9ecef; padding: 10px; border-radius: 3px; margin: 10px 0; }
        pre { overflow-x: auto; }
        .format { display: inline-block; background: #28a745; color: white; padding: 2px 8px; border-radius: 3px; margin: 2px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔄 API de Conversion de Formats</h1>

        <h2>Formats supportés</h2>
        <p>
            <span class="format">JSON</span>
            <span class="format">XML</span>
            <span class="format">CSV</span>
        </p>

        <div class="endpoint">
            <h3>POST /convert</h3>
            <p><strong>Description:</strong> Convertir des données entre différents formats</p>

            <h4>Headers:</h4>
            <ul>
                <li><code>Source-Format</code>: json, xml, ou csv (auto-détection si omis)</li>
                <li><code>Target-Format</code>: json, xml, ou csv (json par défaut)</li>
                <li><code>Content-Type</code>: application/json, application/xml, ou text/csv</li>
            </ul>

            <h4>Exemple avec curl:</h4>
            <div class="example">
                <pre># JSON vers CSV
curl -X POST http://localhost:8080/convert \
  -H "Source-Format: json" \
  -H "Target-Format: csv" \
  -H "Content-Type: application/json" \
  -d '[{"nom":"Alice","age":30},{"nom":"Bob","age":25}]'</pre>
            </div>
        </div>

        <div class="endpoint">
            <h3>GET /example/{format}</h3>
            <p><strong>Description:</strong> Obtenir des données d'exemple dans le format spécifié</p>
            <p><strong>Formats:</strong> json, xml, csv</p>
        </div>

        <h2>Conversions supportées</h2>
        <ul>
            <li>JSON ↔ CSV</li>
            <li>JSON ↔ XML</li>
            <li>XML ↔ CSV</li>
        </ul>

        <h2>Test interactif</h2>
        <form id="convertForm">
            <div style="margin: 10px 0;">
                <label>Format source:</label>
                <select id="sourceFormat">
                    <option value="">Auto-détection</option>
                    <option value="json">JSON</option>
                    <option value="xml">XML</option>
                    <option value="csv">CSV</option>
                </select>
            </div>

            <div style="margin: 10px 0;">
                <label>Format cible:</label>
                <select id="targetFormat">
                    <option value="json">JSON</option>
                    <option value="xml">XML</option>
                    <option value="csv">CSV</option>
                </select>
            </div>

            <div style="margin: 10px 0;">
                <label>Données à convertir:</label><br>
                <textarea id="inputData" rows="10" cols="80" placeholder="Entrez vos données ici..."></textarea>
            </div>

            <button type="button" onclick="convertData()">Convertir</button>
            <button type="button" onclick="loadExample()">Charger exemple JSON</button>
        </form>

        <div id="result" style="margin-top: 20px;"></div>
    </div>

    <script>
        function loadExample() {
            document.getElementById('inputData').value = JSON.stringify([
                {"nom": "Alice", "age": 30, "ville": "Paris"},
                {"nom": "Bob", "age": 25, "ville": "Lyon"},
                {"nom": "Charlie", "age": 35, "ville": "Marseille"}
            ], null, 2);
        }

        function convertData() {
            const sourceFormat = document.getElementById('sourceFormat').value;
            const targetFormat = document.getElementById('targetFormat').value;
            const inputData = document.getElementById('inputData').value;

            if (!inputData.trim()) {
                alert('Veuillez entrer des données à convertir');
                return;
            }

            const headers = {
                'Content-Type': 'application/json'
            };

            if (sourceFormat) headers['Source-Format'] = sourceFormat;
            if (targetFormat) headers['Target-Format'] = targetFormat;

            fetch('/convert', {
                method: 'POST',
                headers: headers,
                body: inputData
            })
            .then(response => response.json())
            .then(data => {
                const resultDiv = document.getElementById('result');
                if (data.success) {
                    resultDiv.innerHTML = '<h3>✅ Conversion réussie</h3>' +
                        '<p>Format: ' + data.source_format + ' → ' + data.target_format + '</p>' +
                        '<p>Enregistrements: ' + data.record_count + '</p>' +
                        '<pre>' + JSON.stringify(data.data, null, 2) + '</pre>';
                } else {
                    resultDiv.innerHTML = '<h3>❌ Erreur</h3><p>' + data.message + '</p>';
                }
            })
            .catch(error => {
                document.getElementById('result').innerHTML = '<h3>❌ Erreur</h3><p>' + error + '</p>';
            });
        }
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Générer des exemples de données
func (fc *FormatConverter) exampleHandler(w http.ResponseWriter, r *http.Request) {
    format := strings.TrimPrefix(r.URL.Path, "/example/")

    // Données d'exemple
    sampleData := []map[string]interface{}{
        {"id": 1, "nom": "Alice", "age": 30, "ville": "Paris", "actif": true},
        {"id": 2, "nom": "Bob", "age": 25, "ville": "Lyon", "actif": false},
        {"id": 3, "nom": "Charlie", "age": 35, "ville": "Marseille", "actif": true},
    }

    switch format {
    case "json":
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(sampleData)

    case "csv":
        csvData, _ := fc.JSONToCSV(func() []byte {
            data, _ := json.Marshal(sampleData)
            return data
        }())
        w.Header().Set("Content-Type", "text/csv")
        w.Write([]byte(csvData))

    case "xml":
        jsonData, _ := json.Marshal(sampleData)
        xmlData, _ := fc.JSONToXML(jsonData)
        w.Header().Set("Content-Type", "application/xml")
        w.Write(xmlData)

    default:
        http.Error(w, "Format non supporté. Utilisez: json, xml, ou csv", http.StatusBadRequest)
    }
}

func main() {
    converter := &FormatConverter{}

    // Routes
    http.HandleFunc("/", converter.docsHandler)
    http.HandleFunc("/convert", converter.convertHandler)
    http.HandleFunc("/example/", converter.exampleHandler)

    fmt.Println("🚀 API de Conversion de Formats démarrée")
    fmt.Println("📖 Documentation: http://localhost:8080")
    fmt.Println("🔄 Endpoint de conversion: http://localhost:8080/convert")
    fmt.Println("📝 Exemples: http://localhost:8080/example/{json|xml|csv}")

    fmt.Println("\n✨ Exemples de test:")
    fmt.Println("curl -X POST http://localhost:8080/convert \\")
    fmt.Println("  -H 'Target-Format: csv' \\")
    fmt.Println("  -d '[{\"nom\":\"Test\",\"age\":25}]'")

    if err := http.ListenAndServe(":8080", nil); err != nil {
        fmt.Printf("Erreur serveur: %v\n", err)
    }
}

```


⏭️
