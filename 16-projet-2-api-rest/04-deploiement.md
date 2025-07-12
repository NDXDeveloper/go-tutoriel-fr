🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16-4 : Déploiement

## Introduction

Félicitations ! Nous avons créé une API REST complète, sécurisée et robuste. Maintenant, il est temps de la déployer en production. Dans cette section, nous allons apprendre à :

- **Containeriser** notre application avec Docker
- **Orchestrer** les services avec Docker Compose
- **Configurer** l'environnement de production
- **Monitorer** notre application
- **Mettre en place** une pipeline CI/CD simple

## Qu'est-ce que le déploiement ?

Le **déploiement** consiste à rendre votre application accessible aux utilisateurs finaux en la mettant en ligne sur des serveurs. Cela implique :

- **Empaquetage** : Créer une version distribuable de l'application
- **Configuration** : Adapter l'app aux environnements de production
- **Infrastructure** : Mettre en place les serveurs et services nécessaires
- **Monitoring** : Surveiller les performances et la santé

## Containerisation avec Docker

### Pourquoi Docker ?

Docker permet de **containeriser** votre application, c'est-à-dire de l'empaqueter avec toutes ses dépendances dans un conteneur léger et portable qui fonctionne de manière identique partout.

**Avantages** :
- **Portabilité** : Fonctionne sur n'importe quel système
- **Isolation** : Chaque service est isolé
- **Reproductibilité** : Même environnement partout
- **Scalabilité** : Facile de dupliquer des conteneurs

### Dockerfile

Créons un `Dockerfile` à la racine du projet :

```dockerfile
# Utiliser l'image officielle Go comme base
FROM golang:1.21-alpine AS builder

# Installer les certificats SSL et Git
RUN apk add --no-cache ca-certificates git

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers go.mod et go.sum
COPY go.mod go.sum ./

# Télécharger les dépendances
RUN go mod download

# Copier le code source
COPY . .

# Construire l'application
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main cmd/server/main.go

# Image finale minimale
FROM alpine:latest

# Installer les certificats SSL
RUN apk --no-cache add ca-certificates

# Créer un utilisateur non-root
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Définir le répertoire de travail
WORKDIR /app

# Copier l'exécutable depuis l'image de build
COPY --from=builder /app/main .

# Changer le propriétaire des fichiers
RUN chown -R appuser:appgroup /app

# Utiliser l'utilisateur non-root
USER appuser

# Exposer le port
EXPOSE 8080

# Définir les variables d'environnement par défaut
ENV PORT=8080
ENV GIN_MODE=release

# Commande de démarrage
CMD ["./main"]
```

### .dockerignore

Créons un fichier `.dockerignore` pour exclure les fichiers inutiles :

```dockerignore
# Fichiers de développement
.git
.gitignore
README.md
Dockerfile
.dockerignore

# Fichiers de test
*_test.go
test_*.sh

# Fichiers temporaires
*.tmp
*.log

# Dossiers de build
build/
dist/

# Fichiers IDE
.vscode/
.idea/
*.swp
*.swo

# Variables d'environnement locales
.env.local
.env.development

# Dossiers de dépendances
node_modules/
vendor/
```

## Configuration pour la production

### Variables d'environnement

Créons un fichier `.env.production` :

```env
# Base de données
DB_HOST=postgres
DB_PORT=5432
DB_USER=todoapi
DB_PASSWORD=your_secure_password_here
DB_NAME=todo_api_prod

# Application
PORT=8080
GIN_MODE=release
JWT_SECRET=your_very_secure_jwt_secret_key_here

# Monitoring
LOG_LEVEL=info
ENABLE_METRICS=true

# Sécurité
CORS_ORIGINS=https://yourdomain.com,https://www.yourdomain.com
RATE_LIMIT_REQUESTS=1000
RATE_LIMIT_WINDOW=3600

# Base de données de production
POSTGRES_DB=todo_api_prod
POSTGRES_USER=todoapi
POSTGRES_PASSWORD=your_secure_password_here
```

### Configuration de production

Améliorons `internal/config/config.go` pour la production :

```go
package config

import (
    "log"
    "os"
    "strconv"
    "strings"

    "github.com/joho/godotenv"
)

type Config struct {
    // Base de données
    DBHost     string
    DBPort     string
    DBUser     string
    DBPassword string
    DBName     string

    // Application
    Port    string
    GinMode string

    // Sécurité
    JWTSecret          string
    CORSOrigins        []string
    RateLimitRequests  int
    RateLimitWindow    int

    // Monitoring
    LogLevel      string
    EnableMetrics bool
}

func Load() *Config {
    // Charger le fichier .env approprié selon l'environnement
    env := getEnv("GO_ENV", "development")

    var envFile string
    switch env {
    case "production":
        envFile = ".env.production"
    case "staging":
        envFile = ".env.staging"
    default:
        envFile = ".env"
    }

    // Charger le fichier d'environnement
    if err := godotenv.Load(envFile); err != nil {
        log.Printf("Fichier %s non trouvé, utilisation des variables d'environnement système", envFile)
    }

    return &Config{
        // Base de données
        DBHost:     getEnv("DB_HOST", "localhost"),
        DBPort:     getEnv("DB_PORT", "5432"),
        DBUser:     getEnv("DB_USER", "postgres"),
        DBPassword: getEnv("DB_PASSWORD", ""),
        DBName:     getEnv("DB_NAME", "todo_api"),

        // Application
        Port:    getEnv("PORT", "8080"),
        GinMode: getEnv("GIN_MODE", "debug"),

        // Sécurité
        JWTSecret:         getEnv("JWT_SECRET", "default-secret-change-in-production"),
        CORSOrigins:       parseCORSOrigins(getEnv("CORS_ORIGINS", "*")),
        RateLimitRequests: getEnvInt("RATE_LIMIT_REQUESTS", 100),
        RateLimitWindow:   getEnvInt("RATE_LIMIT_WINDOW", 3600),

        // Monitoring
        LogLevel:      getEnv("LOG_LEVEL", "debug"),
        EnableMetrics: getEnvBool("ENABLE_METRICS", false),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        if intValue, err := strconv.Atoi(value); err == nil {
            return intValue
        }
    }
    return defaultValue
}

func getEnvBool(key string, defaultValue bool) bool {
    if value := os.Getenv(key); value != "" {
        if boolValue, err := strconv.ParseBool(value); err == nil {
            return boolValue
        }
    }
    return defaultValue
}

func parseCORSOrigins(origins string) []string {
    if origins == "*" {
        return []string{"*"}
    }
    return strings.Split(origins, ",")
}

// Validate vérifie que la configuration est valide
func (c *Config) Validate() error {
    if c.DBPassword == "" {
        log.Println("ATTENTION: Mot de passe de base de données vide")
    }

    if c.JWTSecret == "default-secret-change-in-production" {
        log.Println("ATTENTION: Utilisez une clé JWT sécurisée en production")
    }

    return nil
}
```

## Docker Compose pour l'orchestration

### docker-compose.yml

Créons un fichier `docker-compose.yml` pour orchestrer tous nos services :

```yaml
version: '3.8'

services:
  # Base de données PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: todo-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-todo_api_prod}
      POSTGRES_USER: ${POSTGRES_USER:-todoapi}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-your_secure_password}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - todo-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-todoapi}"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Application Todo API
  todo-api:
    build: .
    container_name: todo-api
    environment:
      GO_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: ${POSTGRES_USER:-todoapi}
      DB_PASSWORD: ${POSTGRES_PASSWORD:-your_secure_password}
      DB_NAME: ${POSTGRES_DB:-todo_api_prod}
      JWT_SECRET: ${JWT_SECRET:-your_jwt_secret}
      GIN_MODE: release
      PORT: 8080
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - todo-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis pour le cache (optionnel)
  redis:
    image: redis:7-alpine
    container_name: todo-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - todo-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Nginx comme reverse proxy
  nginx:
    image: nginx:alpine
    container_name: todo-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - todo-api
    networks:
      - todo-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  todo-network:
    driver: bridge
```

### docker-compose.prod.yml

Pour la production, créons un fichier séparé avec des optimisations :

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - todo-network
    restart: always
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  todo-api:
    build: .
    environment:
      GO_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_NAME: ${POSTGRES_DB}
      JWT_SECRET: ${JWT_SECRET}
      GIN_MODE: release
    networks:
      - todo-network
    restart: always
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 128M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - todo-api
    networks:
      - todo-network
    restart: always

volumes:
  postgres_data:
    driver: local

networks:
  todo-network:
    driver: overlay
```

## Configuration Nginx

### nginx/nginx.conf

Créons la configuration Nginx pour le reverse proxy :

```nginx
events {
    worker_connections 1024;
}

http {
    # Configuration de base
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Upstream pour l'API
    upstream todo_api {
        server todo-api:8080;
        # Pour plusieurs instances
        # server todo-api-1:8080;
        # server todo-api-2:8080;
    }

    # Redirection HTTP vers HTTPS
    server {
        listen 80;
        server_name your-domain.com www.your-domain.com;
        return 301 https://$server_name$request_uri;
    }

    # Configuration HTTPS
    server {
        listen 443 ssl http2;
        server_name your-domain.com www.your-domain.com;

        # Certificats SSL
        ssl_certificate /etc/nginx/ssl/fullchain.pem;
        ssl_certificate_key /etc/nginx/ssl/privkey.pem;

        # Configuration SSL sécurisée
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # Headers de sécurité
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self'" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # Proxy vers l'API
        location /api/ {
            # Rate limiting
            limit_req zone=api burst=20 nodelay;

            # Proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Proxy vers l'upstream
            proxy_pass http://todo_api;
            proxy_redirect off;

            # Timeouts
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;

            # Headers CORS (si nécessaire)
            add_header Access-Control-Allow-Origin "https://your-frontend.com" always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
        }

        # Health check
        location /health {
            proxy_pass http://todo_api/api/health;
            access_log off;
        }

        # Page d'accueil ou documentation
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```

## Scripts de base de données

### scripts/init.sql

Créons un script d'initialisation pour PostgreSQL :

```sql
-- Script d'initialisation de la base de données Todo API

-- Créer la base de données si elle n'existe pas
CREATE DATABASE todo_api_prod;

-- Se connecter à la base de données
\c todo_api_prod;

-- Créer des extensions utiles
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Créer un utilisateur pour l'application si il n'existe pas
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'todoapi') THEN
        CREATE ROLE todoapi WITH LOGIN PASSWORD 'your_secure_password';
    END IF;
END
$$;

-- Accorder les permissions
GRANT ALL PRIVILEGES ON DATABASE todo_api_prod TO todoapi;
GRANT ALL ON SCHEMA public TO todoapi;

-- Créer des index pour les performances
-- Ces index seront créés automatiquement par GORM, mais on peut les optimiser

-- Index pour les requêtes fréquentes sur les tâches
-- CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tasks_user_id_completed ON tasks(user_id, completed);
-- CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tasks_due_date ON tasks(due_date) WHERE due_date IS NOT NULL;

-- Index pour les catégories
-- CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_categories_user_id_name ON categories(user_id, name);

-- Configuration pour optimiser les performances
ALTER DATABASE todo_api_prod SET log_statement = 'none';
ALTER DATABASE todo_api_prod SET log_min_duration_statement = 1000;
```

## Monitoring et observabilité

### Health checks avancés

Améliorons notre endpoint de santé dans `internal/handlers/health.go` :

```go
package handlers

import (
    "net/http"
    "time"

    "todo-api/internal/utils"
    "todo-api/pkg/database"
)

type HealthStatus struct {
    Status    string            `json:"status"`
    Timestamp string            `json:"timestamp"`
    Version   string            `json:"version"`
    Services  map[string]string `json:"services"`
    Uptime    string            `json:"uptime"`
}

var startTime = time.Now()

func HealthCheck(w http.ResponseWriter, r *http.Request) {
    status := "healthy"
    services := make(map[string]string)

    // Vérifier la base de données
    if err := checkDatabase(); err != nil {
        status = "unhealthy"
        services["database"] = "unhealthy: " + err.Error()
    } else {
        services["database"] = "healthy"
    }

    // Calculer l'uptime
    uptime := time.Since(startTime).String()

    healthStatus := HealthStatus{
        Status:    status,
        Timestamp: time.Now().Format(time.RFC3339),
        Version:   "1.0.0",
        Services:  services,
        Uptime:    uptime,
    }

    statusCode := http.StatusOK
    if status == "unhealthy" {
        statusCode = http.StatusServiceUnavailable
    }

    utils.WriteJSON(w, statusCode, healthStatus)
}

func checkDatabase() error {
    sqlDB, err := database.DB.DB()
    if err != nil {
        return err
    }

    return sqlDB.Ping()
}
```

### Logging structuré

Créons un système de logging pour la production dans `internal/logger/logger.go` :

```go
package logger

import (
    "log/slog"
    "os"
    "strings"
)

var Logger *slog.Logger

func Init(level string) {
    var logLevel slog.Level

    switch strings.ToLower(level) {
    case "debug":
        logLevel = slog.LevelDebug
    case "info":
        logLevel = slog.LevelInfo
    case "warn":
        logLevel = slog.LevelWarn
    case "error":
        logLevel = slog.LevelError
    default:
        logLevel = slog.LevelInfo
    }

    opts := &slog.HandlerOptions{
        Level: logLevel,
    }

    // En production, utiliser JSON pour faciliter l'analyse
    if os.Getenv("GO_ENV") == "production" {
        Logger = slog.New(slog.NewJSONHandler(os.Stdout, opts))
    } else {
        Logger = slog.New(slog.NewTextHandler(os.Stdout, opts))
    }

    // Définir comme logger par défaut
    slog.SetDefault(Logger)
}

// Helper functions
func Info(msg string, args ...any) {
    Logger.Info(msg, args...)
}

func Error(msg string, args ...any) {
    Logger.Error(msg, args...)
}

func Debug(msg string, args ...any) {
    Logger.Debug(msg, args...)
}

func Warn(msg string, args ...any) {
    Logger.Warn(msg, args...)
}
```

## Scripts de déploiement

### deploy.sh

Créons un script de déploiement automatisé :

```bash
#!/bin/bash

# Script de déploiement pour Todo API
set -e

echo "🚀 Début du déploiement de Todo API"

# Variables
ENVIRONMENT=${1:-production}
VERSION=${2:-latest}

echo "📋 Environnement: $ENVIRONMENT"
echo "📋 Version: $VERSION"

# Vérifier que Docker est installé
if ! command -v docker &> /dev/null; then
    echo "❌ Docker n'est pas installé"
    exit 1
fi

if ! command -v docker-compose &> /dev/null; then
    echo "❌ Docker Compose n'est pas installé"
    exit 1
fi

# Créer les répertoires nécessaires
echo "📁 Création des répertoires..."
mkdir -p nginx/ssl
mkdir -p data/postgres
mkdir -p logs

# Vérifier la configuration
echo "🔧 Vérification de la configuration..."
if [ ! -f ".env.$ENVIRONMENT" ]; then
    echo "❌ Fichier .env.$ENVIRONMENT manquant"
    exit 1
fi

# Copier le fichier d'environnement
cp ".env.$ENVIRONMENT" .env

# Construire l'image
echo "🔨 Construction de l'image Docker..."
docker build -t todo-api:$VERSION .

# Arrêter les services existants
echo "🛑 Arrêt des services existants..."
docker-compose down --remove-orphans

# Nettoyer les images inutilisées
echo "🧹 Nettoyage des images inutilisées..."
docker image prune -f

# Démarrer les services
echo "🏃‍♂️ Démarrage des services..."
if [ "$ENVIRONMENT" = "production" ]; then
    docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
else
    docker-compose up -d
fi

# Attendre que les services soient prêts
echo "⏳ Attente du démarrage des services..."
sleep 30

# Vérifier la santé de l'application
echo "🏥 Vérification de la santé de l'application..."
if curl -f http://localhost:8080/api/health > /dev/null 2>&1; then
    echo "✅ L'application est en ligne et fonctionne"
else
    echo "❌ L'application ne répond pas"
    echo "📋 Logs de l'application:"
    docker-compose logs todo-api
    exit 1
fi

# Afficher le statut des services
echo "📊 Statut des services:"
docker-compose ps

echo "🎉 Déploiement terminé avec succès!"
echo "🌐 L'API est accessible sur: http://localhost:8080"
echo "📚 Documentation de l'API: http://localhost:8080/api/health"
```

### Makefile

Créons un Makefile pour simplifier les commandes :

```makefile
# Makefile pour Todo API

.PHONY: help build run test clean deploy logs

# Variables
IMAGE_NAME=todo-api
VERSION?=latest
ENVIRONMENT?=development

help: ## Afficher l'aide
	@echo "Commandes disponibles:"
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

build: ## Construire l'image Docker
	docker build -t $(IMAGE_NAME):$(VERSION) .

run: ## Démarrer l'application en mode développement
	docker-compose up -d

run-prod: ## Démarrer l'application en mode production
	docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

stop: ## Arrêter tous les services
	docker-compose down

restart: stop run ## Redémarrer tous les services

test: ## Exécuter les tests
	go test ./...

test-integration: ## Exécuter les tests d'intégration
	./test_api.sh

clean: ## Nettoyer les conteneurs et images
	docker-compose down --remove-orphans
	docker image prune -f
	docker volume prune -f

deploy: ## Déployer en production
	./deploy.sh production $(VERSION)

logs: ## Afficher les logs
	docker-compose logs -f

logs-api: ## Afficher les logs de l'API uniquement
	docker-compose logs -f todo-api

status: ## Afficher le statut des services
	docker-compose ps

shell: ## Ouvrir un shell dans le conteneur de l'API
	docker-compose exec todo-api sh

db-shell: ## Ouvrir un shell PostgreSQL
	docker-compose exec postgres psql -U todoapi -d todo_api_prod

backup: ## Sauvegarder la base de données
	docker-compose exec postgres pg_dump -U todoapi todo_api_prod > backup_$(shell date +%Y%m%d_%H%M%S).sql

restore: ## Restaurer la base de données (nécessite BACKUP_FILE=fichier.sql)
	@if [ -z "$(BACKUP_FILE)" ]; then echo "Utilisation: make restore BACKUP_FILE=backup.sql"; exit 1; fi
	docker-compose exec -T postgres psql -U todoapi -d todo_api_prod < $(BACKUP_FILE)
```

## CI/CD avec GitHub Actions

### .github/workflows/deploy.yml

Créons une pipeline CI/CD simple avec GitHub Actions :

```yaml
name: Deploy Todo API

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  GO_VERSION: 1.21

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: todo_api_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: ${{ env.GO_VERSION }}

    - name: Cache Go modules
      uses: actions/cache@v3
      with:
        path: ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: |
          ${{ runner.os }}-go-

    - name: Install dependencies
      run: go mod download

    - name: Run tests
      env:
        DB_HOST: localhost
        DB_PORT: 5432
        DB_USER: postgres
        DB_PASSWORD: postgres
        DB_NAME: todo_api_test
        JWT_SECRET: test-secret
      run: go test -v ./...

    - name: Run linter
      uses: golangci/golangci-lint-action@v3
      with:
        version: latest

build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ secrets.DOCKER_USERNAME }}/todo-api:latest
          ${{ secrets.DOCKER_USERNAME }}/todo-api:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Deploy to server
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /opt/todo-api
          docker pull ${{ secrets.DOCKER_USERNAME }}/todo-api:latest
          docker-compose down
          docker-compose up -d
          sleep 30
          curl -f http://localhost:8080/api/health || exit 1
```

## Configuration SSL avec Let's Encrypt

### scripts/setup-ssl.sh

Créons un script pour configurer automatiquement SSL avec Let's Encrypt :

```bash
#!/bin/bash

# Script de configuration SSL avec Let's Encrypt
set -e

DOMAIN=${1:-your-domain.com}
EMAIL=${2:-your-email@example.com}

echo "🔒 Configuration SSL pour $DOMAIN"

# Vérifier que certbot est installé
if ! command -v certbot &> /dev/null; then
    echo "📦 Installation de certbot..."
    apt-get update
    apt-get install -y certbot python3-certbot-nginx
fi

# Arrêter nginx temporairement
echo "🛑 Arrêt temporaire de nginx..."
docker-compose stop nginx

# Obtenir le certificat SSL
echo "📜 Obtention du certificat SSL..."
certbot certonly --standalone \
    --email $EMAIL \
    --agree-tos \
    --no-eff-email \
    -d $DOMAIN \
    -d www.$DOMAIN

# Copier les certificats dans le répertoire nginx
echo "📋 Copie des certificats..."
mkdir -p nginx/ssl
cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem nginx/ssl/
cp /etc/letsencrypt/live/$DOMAIN/privkey.pem nginx/ssl/

# Redémarrer nginx
echo "🏃‍♂️ Redémarrage de nginx..."
docker-compose up -d nginx

# Configurer le renouvellement automatique
echo "🔄 Configuration du renouvellement automatique..."
(crontab -l 2>/dev/null; echo "0 12 * * * /usr/bin/certbot renew --quiet --deploy-hook 'docker-compose restart nginx'") | crontab -

echo "✅ SSL configuré avec succès pour $DOMAIN"
```

## Monitoring avec Prometheus et Grafana

### docker-compose.monitoring.yml

Ajoutons des services de monitoring :

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: todo-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    networks:
      - todo-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: todo-grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
      - ./monitoring/grafana/dashboards:/var/lib/grafana/dashboards
    networks:
      - todo-network
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: todo-node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - todo-network
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### monitoring/prometheus.yml

Configuration de Prometheus :

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'todo-api'
    static_configs:
      - targets: ['todo-api:8080']
    metrics_path: '/api/metrics'
    scrape_interval: 30s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres:5432']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx:80']
```

## Métriques de l'application

### internal/metrics/metrics.go

Ajoutons des métriques personnalisées à notre application :

```go
package metrics

import (
    "net/http"
    "strconv"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Compteurs
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "todo_api_http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    // Histogrammes pour les durées
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "todo_api_http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )

    // Jauges pour les métriques en temps réel
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "todo_api_active_connections",
            Help: "Number of active connections",
        },
    )

    // Métriques métier
    tasksTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "todo_api_tasks_total",
            Help: "Total number of tasks",
        },
        []string{"action", "user_id"},
    )

    usersTotal = prometheus.NewCounter(
        prometheus.CounterOpts{
            Name: "todo_api_users_total",
            Help: "Total number of registered users",
        },
    )
)

// Init initialise les métriques Prometheus
func Init() {
    prometheus.MustRegister(httpRequestsTotal)
    prometheus.MustRegister(httpRequestDuration)
    prometheus.MustRegister(activeConnections)
    prometheus.MustRegister(tasksTotal)
    prometheus.MustRegister(usersTotal)
}

// Handler retourne le handler pour l'endpoint /metrics
func Handler() http.Handler {
    return promhttp.Handler()
}

// RecordHTTPRequest enregistre une requête HTTP
func RecordHTTPRequest(method, endpoint string, statusCode int, duration time.Duration) {
    status := strconv.Itoa(statusCode)
    httpRequestsTotal.WithLabelValues(method, endpoint, status).Inc()
    httpRequestDuration.WithLabelValues(method, endpoint).Observe(duration.Seconds())
}

// RecordTaskAction enregistre une action sur une tâche
func RecordTaskAction(action string, userID uint) {
    tasksTotal.WithLabelValues(action, strconv.FormatUint(uint64(userID), 10)).Inc()
}

// RecordUserRegistration enregistre une inscription d'utilisateur
func RecordUserRegistration() {
    usersTotal.Inc()
}

// SetActiveConnections définit le nombre de connexions actives
func SetActiveConnections(count float64) {
    activeConnections.Set(count)
}
```

### Middleware de métriques

Créons un middleware pour collecter automatiquement les métriques :

```go
// internal/middleware/metrics_middleware.go
package middleware

import (
    "net/http"
    "time"

    "todo-api/internal/metrics"
)

// MetricsMiddleware collecte les métriques HTTP
func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Wrapper pour capturer le status code
        wrappedWriter := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

        // Exécuter le handler
        next.ServeHTTP(wrappedWriter, r)

        // Enregistrer les métriques
        duration := time.Since(start)
        metrics.RecordHTTPRequest(r.Method, r.URL.Path, wrappedWriter.statusCode, duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

## Tests de charge

### scripts/load-test.sh

Créons un script de test de charge avec Apache Bench :

```bash
#!/bin/bash

# Test de charge pour Todo API
set -e

BASE_URL=${1:-http://localhost:8080}
CONCURRENT_USERS=${2:-10}
TOTAL_REQUESTS=${3:-1000}

echo "🚀 Test de charge sur $BASE_URL"
echo "👥 Utilisateurs concurrents: $CONCURRENT_USERS"
echo "📊 Nombre total de requêtes: $TOTAL_REQUESTS"

# Vérifier qu'Apache Bench est installé
if ! command -v ab &> /dev/null; then
    echo "📦 Installation d'Apache Bench..."
    apt-get update && apt-get install -y apache2-utils
fi

# Test du endpoint de santé
echo "🏥 Test du endpoint de santé..."
ab -n $TOTAL_REQUESTS -c $CONCURRENT_USERS "$BASE_URL/api/health"

# Créer un utilisateur pour les tests
echo "👤 Création d'un utilisateur de test..."
USER_RESPONSE=$(curl -s -X POST "$BASE_URL/api/auth/register" \
    -H "Content-Type: application/json" \
    -d '{
        "username": "loadtest",
        "email": "loadtest@example.com",
        "password": "LoadTest123!"
    }')

TOKEN=$(echo $USER_RESPONSE | jq -r '.data.token' 2>/dev/null || echo "")

if [ "$TOKEN" != "" ] && [ "$TOKEN" != "null" ]; then
    echo "✅ Token obtenu: ${TOKEN:0:20}..."

    # Test des tâches avec authentification
    echo "📝 Test de création de tâches..."

    # Créer un fichier temporaire avec les données de tâche
    TASK_DATA='{"title":"Tâche de test","description":"Test de charge","priority":"medium"}'
    echo $TASK_DATA > /tmp/task_data.json

    # Test avec authentification (plus complexe, on utilise curl en boucle)
    echo "🔄 Test avec authentification..."
    for i in $(seq 1 100); do
        curl -s -X POST "$BASE_URL/api/tasks" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $TOKEN" \
            -d @/tmp/task_data.json > /dev/null &

        # Limiter le nombre de processus parallèles
        if (( i % 10 == 0 )); then
            wait
        fi
    done
    wait

    # Nettoyer
    rm /tmp/task_data.json

    echo "✅ Test avec authentification terminé"
else
    echo "❌ Impossible d'obtenir le token d'authentification"
fi

echo "📊 Test de charge terminé"
```

## Documentation de déploiement

### README-DEPLOYMENT.md

Créons une documentation complète de déploiement :

```markdown
# Guide de déploiement - Todo API

## Prérequis

### Serveur
- Ubuntu 20.04+ ou CentOS 8+
- 2+ CPU cores
- 4GB+ RAM
- 20GB+ stockage
- Docker et Docker Compose installés

### Domaine et DNS
- Nom de domaine configuré
- Enregistrements DNS pointant vers votre serveur
- Certificats SSL (Let's Encrypt recommandé)

## Installation rapide

### 1. Cloner le repository

```bash
git clone https://github.com/votre-username/todo-api.git
cd todo-api
```

### 2. Configuration

```bash
# Copier le fichier d'environnement
cp .env.example .env.production

# Éditer la configuration
nano .env.production
```

Variables importantes à configurer :
- `DB_PASSWORD` : Mot de passe sécurisé pour PostgreSQL
- `JWT_SECRET` : Clé secrète pour JWT (32+ caractères)
- `CORS_ORIGINS` : Domaines autorisés pour CORS

### 3. Déploiement

```bash
# Rendre le script exécutable
chmod +x deploy.sh

# Déployer
./deploy.sh production
```

### 4. Configuration SSL

```bash
# Configurer SSL avec Let's Encrypt
chmod +x scripts/setup-ssl.sh
./scripts/setup-ssl.sh votre-domaine.com votre-email@example.com
```

## Commandes utiles

```bash
# Voir les logs
make logs

# Redémarrer les services
make restart

# Sauvegarder la base de données
make backup

# Voir le statut
make status

# Accéder au shell de l'API
make shell
```

## Monitoring

### Accès aux services de monitoring

- **Grafana** : http://votre-domaine.com:3000 (admin/admin)
- **Prometheus** : http://votre-domaine.com:9090
- **API Health** : http://votre-domaine.com/api/health

### Métriques importantes à surveiller

- Temps de réponse des endpoints
- Nombre de requêtes par seconde
- Utilisation CPU et mémoire
- Connexions à la base de données
- Erreurs 4xx/5xx

## Sauvegarde et restauration

### Sauvegarde automatique

```bash
# Configurer une sauvegarde quotidienne
(crontab -l; echo "0 2 * * * cd /opt/todo-api && make backup") | crontab -
```

### Restauration

```bash
# Restaurer depuis une sauvegarde
make restore BACKUP_FILE=backup_20240115_020000.sql
```

## Mise à jour

### Mise à jour manuelle

```bash
git pull origin main
docker-compose down
docker-compose build
docker-compose up -d
```

### Mise à jour avec zéro downtime

```bash
# Utiliser la stratégie blue-green ou rolling update
docker-compose up -d --scale todo-api=2
sleep 30
docker-compose up -d --scale todo-api=1
```

## Dépannage

### Problèmes courants

**L'API ne démarre pas**
```bash
# Vérifier les logs
docker-compose logs todo-api

# Vérifier la configuration
docker-compose config
```

**Base de données inaccessible**
```bash
# Vérifier PostgreSQL
docker-compose logs postgres

# Tester la connexion
make db-shell
```

**Problèmes SSL**
```bash
# Renouveler les certificats
certbot renew

# Redémarrer nginx
docker-compose restart nginx
```

### Contacts

- **Email** : admin@votre-domaine.com
- **Documentation** : https://docs.votre-domaine.com
- **Support** : https://github.com/votre-username/todo-api/issues
```

## Tests de déploiement

### scripts/test-deployment.sh

Script pour tester le déploiement :

```bash
#!/bin/bash

# Test complet du déploiement
set -e

BASE_URL=${1:-https://your-domain.com}

echo "🧪 Test du déploiement sur $BASE_URL"

# Test 1: Health check
echo "1. Test du health check..."
HEALTH_RESPONSE=$(curl -s "$BASE_URL/api/health")
STATUS=$(echo $HEALTH_RESPONSE | jq -r '.status' 2>/dev/null || echo "error")

if [ "$STATUS" = "healthy" ]; then
    echo "✅ Health check OK"
else
    echo "❌ Health check failed: $HEALTH_RESPONSE"
    exit 1
fi

# Test 2: SSL/TLS
echo "2. Test SSL/TLS..."
if curl -s --head "$BASE_URL" | grep -q "HTTP/2 200\|HTTP/1.1 200"; then
    echo "✅ SSL/TLS OK"
else
    echo "❌ SSL/TLS failed"
    exit 1
fi

# Test 3: Inscription utilisateur
echo "3. Test inscription utilisateur..."
REGISTER_RESPONSE=$(curl -s -X POST "$BASE_URL/api/auth/register" \
    -H "Content-Type: application/json" \
    -d '{
        "username": "testdeploy",
        "email": "testdeploy@example.com",
        "password": "TestDeploy123!"
    }')

if echo $REGISTER_RESPONSE | jq -e '.success' > /dev/null 2>&1; then
    echo "✅ Inscription OK"
    TOKEN=$(echo $REGISTER_RESPONSE | jq -r '.data.token')
else
    echo "❌ Inscription failed: $REGISTER_RESPONSE"
    exit 1
fi

# Test 4: Création de tâche
echo "4. Test création de tâche..."
TASK_RESPONSE=$(curl -s -X POST "$BASE_URL/api/tasks" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{
        "title": "Tâche de test déploiement",
        "description": "Test du déploiement",
        "priority": "high"
    }')

if echo $TASK_RESPONSE | jq -e '.success' > /dev/null 2>&1; then
    echo "✅ Création de tâche OK"
else
    echo "❌ Création de tâche failed: $TASK_RESPONSE"
    exit 1
fi

# Test 5: Performance (temps de réponse)
echo "5. Test de performance..."
RESPONSE_TIME=$(curl -o /dev/null -s -w '%{time_total}' "$BASE_URL/api/health")
RESPONSE_TIME_MS=$(echo "$RESPONSE_TIME * 1000" | bc)

if (( $(echo "$RESPONSE_TIME < 2.0" | bc -l) )); then
    echo "✅ Performance OK (${RESPONSE_TIME_MS}ms)"
else
    echo "⚠️  Performance lente (${RESPONSE_TIME_MS}ms)"
fi

# Test 6: Sécurité headers
echo "6. Test des headers de sécurité..."
SECURITY_HEADERS=$(curl -s -I "$BASE_URL" | grep -E "(X-Frame-Options|X-Content-Type-Options|Strict-Transport-Security)")

if [ -n "$SECURITY_HEADERS" ]; then
    echo "✅ Headers de sécurité présents"
else
    echo "⚠️  Headers de sécurité manquants"
fi

echo "🎉 Tests de déploiement terminés avec succès!"
echo "🌐 API disponible sur: $BASE_URL"
echo "📚 Documentation: $BASE_URL/api/health"
```

## Résumé du déploiement

Dans cette section, nous avons appris à :

**Containeriser l'application** :
- Dockerfile optimisé avec multi-stage build
- Images Docker sécurisées et légères
- Configuration pour la production

**Orchestrer les services** :
- Docker Compose pour tous les services
- Configuration de production avec scaling
- Services de monitoring intégrés

**Sécuriser le déploiement** :
- Configuration SSL/TLS automatique
- Headers de sécurité avec Nginx
- Utilisateurs non-root dans les conteneurs

**Automatiser le déploiement** :
- Scripts de déploiement automatisés
- Pipeline CI/CD avec GitHub Actions
- Tests automatisés post-déploiement

**Monitorer l'application** :
- Métriques Prometheus
- Dashboards Grafana
- Health checks complets

**Maintenir en production** :
- Sauvegarde automatique
- Scripts de mise à jour
- Documentation complète

## Points clés à retenir

- **Toujours tester en environnement de staging avant la production**
- **Automatiser les déploiements pour réduire les erreurs**
- **Monitorer les performances et la santé de l'application**
- **Sauvegarder régulièrement les données**
- **Sécuriser tous les accès et communications**
- **Documenter les procédures de déploiement et de maintenance**

🎉 **Félicitations !** Vous avez maintenant une API REST complète, sécurisée, robuste et déployée en production. Cette API respecte toutes les bonnes pratiques modernes de développement et de déploiement.

Votre API Todo est maintenant prête à servir des milliers d'utilisateurs avec confiance !

⏭️
