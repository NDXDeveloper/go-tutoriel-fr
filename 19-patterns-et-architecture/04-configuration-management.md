🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19-4 : Configuration management

## Qu'est-ce que la Configuration Management ?

La Configuration Management consiste à **gérer les paramètres de votre application** de manière propre et flexible. Ces paramètres incluent les connexions base de données, les clés API, les ports d'écoute, etc.

### Analogie simple
Imaginez que vous déménagez dans plusieurs villes :

**❌ Sans configuration** : Vous codez votre adresse en dur dans tous vos documents
```go
func main() {
    address := "123 Rue de Paris, 75001 Paris" // ❌ Codé en dur
    // Problème : comment changer pour Lyon ?
}
```

**✅ Avec configuration** : Vous avez un carnet d'adresses que vous mettez à jour
```go
func main() {
    config := LoadConfig()
    address := config.Address // ✅ Flexible
    // Facile de changer selon la ville !
}
```

## Pourquoi gérer la configuration ?

### 1. **Environnements multiples**
Développement, test, production ont des paramètres différents.

### 2. **Sécurité**
Les mots de passe ne doivent pas être dans le code.

### 3. **Flexibilité**
Changer des paramètres sans recompiler.

### 4. **Équipe**
Chaque développeur peut avoir sa propre configuration.

## Les sources de configuration

### 1. **Variables d'environnement**
```bash
export DB_HOST=localhost
export DB_PORT=5432
export API_KEY=secret123
```

### 2. **Fichiers de configuration**
```yaml
# config.yaml
database:
  host: localhost
  port: 5432
server:
  port: 8080
```

### 3. **Arguments de ligne de commande**
```bash
./myapp --port=8080 --db-host=localhost
```

### 4. **Valeurs par défaut**
Directement dans le code Go.

## Structure de configuration recommandée

```go
// Structure de configuration principale
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Redis    RedisConfig    `yaml:"redis"`
    Email    EmailConfig    `yaml:"email"`
    Log      LogConfig      `yaml:"log"`
}

type ServerConfig struct {
    Port    int    `yaml:"port" env:"SERVER_PORT" default:"8080"`
    Host    string `yaml:"host" env:"SERVER_HOST" default:"localhost"`
    Timeout int    `yaml:"timeout" env:"SERVER_TIMEOUT" default:"30"`
}

type DatabaseConfig struct {
    Host     string `yaml:"host" env:"DB_HOST" default:"localhost"`
    Port     int    `yaml:"port" env:"DB_PORT" default:"5432"`
    User     string `yaml:"user" env:"DB_USER" default:"postgres"`
    Password string `yaml:"password" env:"DB_PASSWORD"`
    Name     string `yaml:"name" env:"DB_NAME" default:"myapp"`
    SSLMode  string `yaml:"ssl_mode" env:"DB_SSL_MODE" default:"disable"`
}

type RedisConfig struct {
    Host     string `yaml:"host" env:"REDIS_HOST" default:"localhost"`
    Port     int    `yaml:"port" env:"REDIS_PORT" default:"6379"`
    Password string `yaml:"password" env:"REDIS_PASSWORD"`
}

type EmailConfig struct {
    SMTPHost     string `yaml:"smtp_host" env:"EMAIL_SMTP_HOST"`
    SMTPPort     int    `yaml:"smtp_port" env:"EMAIL_SMTP_PORT" default:"587"`
    SMTPUser     string `yaml:"smtp_user" env:"EMAIL_SMTP_USER"`
    SMTPPassword string `yaml:"smtp_password" env:"EMAIL_SMTP_PASSWORD"`
}

type LogConfig struct {
    Level  string `yaml:"level" env:"LOG_LEVEL" default:"info"`
    Format string `yaml:"format" env:"LOG_FORMAT" default:"json"`
}
```

## Méthode 1 : Configuration simple avec variables d'environnement

### Lecture des variables d'environnement

```go
package config

import (
    "os"
    "strconv"
    "fmt"
)

// Fonction pour lire une variable d'environnement string
func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

// Fonction pour lire une variable d'environnement int
func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}

// Fonction pour lire une variable d'environnement bool
func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if boolValue, err := strconv.ParseBool(value); err == nil {
            return boolValue
        }
    }
    return defaultValue
}

// Chargement de la configuration
func LoadConfig() *Config {
    return &Config{
        Server: ServerConfig{
            Port:    getEnvInt("SERVER_PORT", 8080),
            Host:    getEnv("SERVER_HOST", "localhost"),
            Timeout: getEnvInt("SERVER_TIMEOUT", 30),
        },
        Database: DatabaseConfig{
            Host:     getEnv("DB_HOST", "localhost"),
            Port:     getEnvInt("DB_PORT", 5432),
            User:     getEnv("DB_USER", "postgres"),
            Password: getEnv("DB_PASSWORD", ""),
            Name:     getEnv("DB_NAME", "myapp"),
            SSLMode:  getEnv("DB_SSL_MODE", "disable"),
        },
        Redis: RedisConfig{
            Host:     getEnv("REDIS_HOST", "localhost"),
            Port:     getEnvInt("REDIS_PORT", 6379),
            Password: getEnv("REDIS_PASSWORD", ""),
        },
        Email: EmailConfig{
            SMTPHost:     getEnv("EMAIL_SMTP_HOST", ""),
            SMTPPort:     getEnvInt("EMAIL_SMTP_PORT", 587),
            SMTPUser:     getEnv("EMAIL_SMTP_USER", ""),
            SMTPPassword: getEnv("EMAIL_SMTP_PASSWORD", ""),
        },
        Log: LogConfig{
            Level:  getEnv("LOG_LEVEL", "info"),
            Format: getEnv("LOG_FORMAT", "json"),
        },
    }
}

// Validation de la configuration
func (c *Config) Validate() error {
    if c.Database.Password == "" {
        return fmt.Errorf("DB_PASSWORD est requis")
    }

    if c.Server.Port < 1 || c.Server.Port > 65535 {
        return fmt.Errorf("port serveur invalide: %d", c.Server.Port)
    }

    if c.Log.Level != "debug" && c.Log.Level != "info" &&
       c.Log.Level != "warn" && c.Log.Level != "error" {
        return fmt.Errorf("niveau de log invalide: %s", c.Log.Level)
    }

    return nil
}

// Utilisation
func main() {
    config := LoadConfig()

    if err := config.Validate(); err != nil {
        log.Fatalf("Configuration invalide: %v", err)
    }

    fmt.Printf("Serveur démarré sur %s:%d\n", config.Server.Host, config.Server.Port)
}
```

### Fichier .env pour le développement

```bash
# .env (fichier de développement)
SERVER_PORT=8080
SERVER_HOST=localhost

DB_HOST=localhost
DB_PORT=5432
DB_USER=myuser
DB_PASSWORD=mypassword
DB_NAME=myapp_dev

REDIS_HOST=localhost
REDIS_PORT=6379

EMAIL_SMTP_HOST=smtp.mailtrap.io
EMAIL_SMTP_PORT=587
EMAIL_SMTP_USER=testuser
EMAIL_SMTP_PASSWORD=testpass

LOG_LEVEL=debug
LOG_FORMAT=text
```

## Méthode 2 : Configuration avec fichiers YAML

### Installation des dépendances

```bash
go get gopkg.in/yaml.v3
```

### Fichier de configuration

```yaml
# config.yaml
server:
  port: 8080
  host: "localhost"
  timeout: 30

database:
  host: "localhost"
  port: 5432
  user: "postgres"
  password: "password"
  name: "myapp"
  ssl_mode: "disable"

redis:
  host: "localhost"
  port: 6379
  password: ""

email:
  smtp_host: "smtp.example.com"
  smtp_port: 587
  smtp_user: "user@example.com"
  smtp_password: "password"

log:
  level: "info"
  format: "json"
```

### Chargement des fichiers YAML

```go
package config

import (
    "fmt"
    "os"
    "path/filepath"

    "gopkg.in/yaml.v3"
)

// Chargement depuis un fichier YAML
func LoadFromFile(filename string) (*Config, error) {
    // Vérifier que le fichier existe
    if _, err := os.Stat(filename); os.IsNotExist(err) {
        return nil, fmt.Errorf("fichier de configuration non trouvé: %s", filename)
    }

    // Lire le fichier
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("erreur lecture fichier: %w", err)
    }

    // Parser le YAML
    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("erreur parsing YAML: %w", err)
    }

    return &config, nil
}

// Chargement automatique selon l'environnement
func LoadConfigAuto() (*Config, error) {
    env := getEnv("GO_ENV", "development")

    // Chercher le fichier de config approprié
    configFiles := []string{
        fmt.Sprintf("config.%s.yaml", env),
        "config.yaml",
        "config.yml",
    }

    for _, filename := range configFiles {
        if _, err := os.Stat(filename); err == nil {
            fmt.Printf("Chargement de la configuration: %s\n", filename)
            return LoadFromFile(filename)
        }
    }

    return nil, fmt.Errorf("aucun fichier de configuration trouvé")
}

// Sauvegarde d'une configuration
func (c *Config) SaveToFile(filename string) error {
    data, err := yaml.Marshal(c)
    if err != nil {
        return fmt.Errorf("erreur marshaling YAML: %w", err)
    }

    // Créer le répertoire si nécessaire
    if err := os.MkdirAll(filepath.Dir(filename), 0755); err != nil {
        return fmt.Errorf("erreur création répertoire: %w", err)
    }

    // Écrire le fichier
    if err := os.WriteFile(filename, data, 0644); err != nil {
        return fmt.Errorf("erreur écriture fichier: %w", err)
    }

    return nil
}
```

## Méthode 3 : Configuration hybride (recommandée)

Cette méthode combine fichiers + variables d'environnement :

```go
package config

import (
    "fmt"
    "os"
    "reflect"
    "strconv"
    "strings"

    "gopkg.in/yaml.v3"
)

// Chargement hybride : fichier + env variables
func LoadConfigHybrid() (*Config, error) {
    // 1. Commencer par les valeurs par défaut
    config := getDefaultConfig()

    // 2. Surcharger avec le fichier si disponible
    if configFromFile, err := LoadConfigAuto(); err == nil {
        mergeConfigs(config, configFromFile)
    }

    // 3. Surcharger avec les variables d'environnement
    overrideWithEnv(config)

    return config, nil
}

// Configuration par défaut
func getDefaultConfig() *Config {
    return &Config{
        Server: ServerConfig{
            Port:    8080,
            Host:    "localhost",
            Timeout: 30,
        },
        Database: DatabaseConfig{
            Host:    "localhost",
            Port:    5432,
            User:    "postgres",
            Name:    "myapp",
            SSLMode: "disable",
        },
        Redis: RedisConfig{
            Host: "localhost",
            Port: 6379,
        },
        Email: EmailConfig{
            SMTPPort: 587,
        },
        Log: LogConfig{
            Level:  "info",
            Format: "json",
        },
    }
}

// Fusion de deux configurations
func mergeConfigs(base, override *Config) {
    if override.Server.Port != 0 {
        base.Server.Port = override.Server.Port
    }
    if override.Server.Host != "" {
        base.Server.Host = override.Server.Host
    }
    if override.Server.Timeout != 0 {
        base.Server.Timeout = override.Server.Timeout
    }

    // ... continuer pour tous les champs
}

// Surcharge avec les variables d'environnement
func overrideWithEnv(config *Config) {
    // Utiliser la réflexion pour automatiser
    overrideStructWithEnv(reflect.ValueOf(&config.Server).Elem(), "SERVER_")
    overrideStructWithEnv(reflect.ValueOf(&config.Database).Elem(), "DB_")
    overrideStructWithEnv(reflect.ValueOf(&config.Redis).Elem(), "REDIS_")
    overrideStructWithEnv(reflect.ValueOf(&config.Email).Elem(), "EMAIL_")
    overrideStructWithEnv(reflect.ValueOf(&config.Log).Elem(), "LOG_")
}

// Surcharge automatique avec réflexion
func overrideStructWithEnv(v reflect.Value, prefix string) {
    t := v.Type()

    for i := 0; i < v.NumField(); i++ {
        field := v.Field(i)
        fieldType := t.Field(i)

        // Générer le nom de la variable d'environnement
        envName := prefix + strings.ToUpper(fieldType.Name)

        // Vérifier si la variable existe
        if envValue := os.Getenv(envName); envValue != "" {
            setFieldFromString(field, envValue)
        }
    }
}

// Définir la valeur d'un champ depuis une string
func setFieldFromString(field reflect.Value, value string) {
    if !field.CanSet() {
        return
    }

    switch field.Kind() {
    case reflect.String:
        field.SetString(value)
    case reflect.Int:
        if intValue, err := strconv.Atoi(value); err == nil {
            field.SetInt(int64(intValue))
        }
    case reflect.Bool:
        if boolValue, err := strconv.ParseBool(value); err == nil {
            field.SetBool(boolValue)
        }
    }
}
```

## Méthode 4 : Configuration avec Viper (bibliothèque populaire)

### Installation

```bash
go get github.com/spf13/viper
```

### Configuration avec Viper

```go
package config

import (
    "fmt"
    "strings"

    "github.com/spf13/viper"
)

func LoadConfigViper() (*Config, error) {
    // Configuration de Viper
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AddConfigPath("./config")
    viper.AddConfigPath("/etc/myapp")

    // Variables d'environnement
    viper.AutomaticEnv()
    viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

    // Valeurs par défaut
    setDefaults()

    // Lire le fichier de configuration
    if err := viper.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("erreur lecture config: %w", err)
        }
        // Pas grave si le fichier n'existe pas, on utilise les défauts + env
    }

    // Mapper vers notre struct
    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, fmt.Errorf("erreur unmarshaling: %w", err)
    }

    return &config, nil
}

func setDefaults() {
    // Serveur
    viper.SetDefault("server.port", 8080)
    viper.SetDefault("server.host", "localhost")
    viper.SetDefault("server.timeout", 30)

    // Base de données
    viper.SetDefault("database.host", "localhost")
    viper.SetDefault("database.port", 5432)
    viper.SetDefault("database.user", "postgres")
    viper.SetDefault("database.name", "myapp")
    viper.SetDefault("database.ssl_mode", "disable")

    // Redis
    viper.SetDefault("redis.host", "localhost")
    viper.SetDefault("redis.port", 6379)

    // Email
    viper.SetDefault("email.smtp_port", 587)

    // Log
    viper.SetDefault("log.level", "info")
    viper.SetDefault("log.format", "json")
}

// Variables d'environnement avec Viper
// SERVER_PORT -> server.port
// DB_HOST -> database.host
// REDIS_PASSWORD -> redis.password
```

## Gestion des secrets

### 1. **Jamais dans le code**
```go
// ❌ JAMAIS faire ça
const APIKey = "secret123"

// ✅ Toujours depuis l'environnement
apiKey := os.Getenv("API_KEY")
```

### 2. **Fichiers .env ignorés par Git**
```bash
# .gitignore
.env
.env.local
*.secret
config.production.yaml
```

### 3. **Validation des secrets requis**
```go
func (c *Config) ValidateSecrets() error {
    required := []struct {
        value string
        name  string
    }{
        {c.Database.Password, "Database Password"},
        {c.Email.SMTPPassword, "SMTP Password"},
    }

    for _, req := range required {
        if strings.TrimSpace(req.value) == "" {
            return fmt.Errorf("%s est requis", req.name)
        }
    }

    return nil
}
```

## Configuration par environnement

### Structure des fichiers

```
config/
├── config.yaml              # Configuration commune
├── config.development.yaml  # Surcharge pour développement
├── config.testing.yaml     # Surcharge pour tests
├── config.production.yaml  # Surcharge pour production
└── .env.example            # Exemple de variables d'env
```

### Exemple de surcharge

```yaml
# config.yaml (base)
server:
  timeout: 30
log:
  format: "json"

# config.development.yaml
server:
  timeout: 60  # Plus long en dev pour débugger
log:
  level: "debug"
  format: "text"  # Plus lisible en dev

# config.production.yaml
server:
  timeout: 15  # Plus strict en prod
log:
  level: "info"
```

### Chargement automatique

```go
func LoadConfigForEnv() (*Config, error) {
    env := getEnv("GO_ENV", "development")

    // Charger la config de base
    config, err := LoadFromFile("config/config.yaml")
    if err != nil {
        return nil, err
    }

    // Surcharger avec l'environnement spécifique
    envConfigFile := fmt.Sprintf("config/config.%s.yaml", env)
    if _, err := os.Stat(envConfigFile); err == nil {
        envConfig, err := LoadFromFile(envConfigFile)
        if err != nil {
            return nil, err
        }
        mergeConfigs(config, envConfig)
    }

    // Surcharger avec les variables d'environnement
    overrideWithEnv(config)

    return config, nil
}
```

## Patterns avancés

### 1. **Configuration hot-reload**

```go
type ConfigManager struct {
    config     *Config
    configPath string
    mu         sync.RWMutex
    callbacks  []func(*Config)
}

func NewConfigManager(configPath string) *ConfigManager {
    return &ConfigManager{
        configPath: configPath,
        callbacks:  make([]func(*Config), 0),
    }
}

func (cm *ConfigManager) Load() error {
    config, err := LoadFromFile(cm.configPath)
    if err != nil {
        return err
    }

    cm.mu.Lock()
    oldConfig := cm.config
    cm.config = config
    cm.mu.Unlock()

    // Notifier les callbacks si la config a changé
    if oldConfig != nil {
        for _, callback := range cm.callbacks {
            callback(config)
        }
    }

    return nil
}

func (cm *ConfigManager) Get() *Config {
    cm.mu.RLock()
    defer cm.mu.RUnlock()
    return cm.config
}

func (cm *ConfigManager) OnConfigChange(callback func(*Config)) {
    cm.callbacks = append(cm.callbacks, callback)
}

// Surveillance du fichier
func (cm *ConfigManager) Watch() {
    // Utiliser fsnotify pour surveiller les changements
    // Recharger automatiquement la config
}
```

### 2. **Configuration typée avec validation**

```go
type Port int

func (p Port) Validate() error {
    if p < 1 || p > 65535 {
        return fmt.Errorf("port invalide: %d", p)
    }
    return nil
}

type DatabaseURL string

func (d DatabaseURL) Validate() error {
    if !strings.HasPrefix(string(d), "postgres://") {
        return fmt.Errorf("URL de base de données invalide")
    }
    return nil
}

// Configuration avec types personnalisés
type TypedConfig struct {
    ServerPort Port        `yaml:"server_port"`
    DatabaseURL DatabaseURL `yaml:"database_url"`
}

func (tc *TypedConfig) Validate() error {
    if err := tc.ServerPort.Validate(); err != nil {
        return err
    }
    return tc.DatabaseURL.Validate()
}
```

## Exemple complet d'utilisation

```go
// main.go
package main

import (
    "log"
    "myapp/config"
    "myapp/server"
)

func main() {
    // Charger la configuration
    cfg, err := config.LoadConfigHybrid()
    if err != nil {
        log.Fatalf("Erreur chargement config: %v", err)
    }

    // Valider la configuration
    if err := cfg.Validate(); err != nil {
        log.Fatalf("Configuration invalide: %v", err)
    }

    // Afficher la config (sans les secrets)
    log.Printf("Configuration chargée pour l'environnement: %s",
               getEnv("GO_ENV", "development"))
    log.Printf("Serveur: %s:%d", cfg.Server.Host, cfg.Server.Port)
    log.Printf("Base de données: %s:%d/%s",
               cfg.Database.Host, cfg.Database.Port, cfg.Database.Name)

    // Démarrer l'application
    srv := server.New(cfg)
    if err := srv.Start(); err != nil {
        log.Fatalf("Erreur serveur: %v", err)
    }
}
```

## Bonnes pratiques

### 1. **Hiérarchie de configuration**
```
1. Valeurs par défaut (dans le code)
2. Fichier de configuration
3. Variables d'environnement
4. Arguments de ligne de commande
```

### 2. **Validation stricte**
```go
func (c *Config) Validate() error {
    validators := []func() error{
        c.validateServer,
        c.validateDatabase,
        c.validateEmail,
        c.validateSecrets,
    }

    for _, validate := range validators {
        if err := validate(); err != nil {
            return err
        }
    }

    return nil
}
```

### 3. **Documentation de la configuration**
```go
type Config struct {
    // Port d'écoute du serveur HTTP (1-65535)
    // Variable d'environnement: SERVER_PORT
    // Défaut: 8080
    ServerPort int `yaml:"server_port" env:"SERVER_PORT" default:"8080"`

    // Mot de passe de la base de données
    // Variable d'environnement: DB_PASSWORD (requis)
    // Aucune valeur par défaut pour sécurité
    DatabasePassword string `yaml:"database_password" env:"DB_PASSWORD"`
}
```

### 4. **Configuration immutable**
```go
// Une fois chargée, la config ne doit pas changer
type Config struct {
    server ServerConfig
    // ... autres champs
}

// Pas de setters, uniquement des getters
func (c *Config) Server() ServerConfig {
    return c.server
}
```

## Résumé

La Configuration Management en Go :

**Sources de configuration** :
- Variables d'environnement (flexible)
- Fichiers YAML/JSON (structuré)
- Arguments de ligne de commande (ponctuel)
- Valeurs par défaut (sécurité)

**Bonnes pratiques** :
- Hiérarchie claire des sources
- Validation stricte des valeurs
- Séparation des secrets
- Configuration par environnement

**Outils recommandés** :
- Viper pour les gros projets
- Configuration manuelle pour les petits projets
- Variables d'environnement pour les secrets

**Points clés** :
- Jamais de secrets dans le code
- Validation obligatoire au démarrage
- Documentation claire des options
- Configuration immutable une fois chargée

Une bonne gestion de configuration rend votre application flexible, sécurisée et facile à déployer !

⏭️
