🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17-4 : Containerisation avec Docker

## Introduction à la containerisation

### Qu'est-ce que la containerisation ?

Imaginez que vous déménagiez et que vous vouliez emballer vos affaires. Vous pourriez :
1. **Tout mettre en vrac dans un camion** → Difficile à organiser et à transporter
2. **Utiliser des cartons standardisés** → Facile à empiler, transporter et organiser

La containerisation, c'est comme utiliser des cartons standardisés pour vos applications !

**Sans containers** :
```
Serveur 1: App A + App B + App C
- Conflits de dépendances
- "Ça marche sur mon machine"
- Difficile à déployer
```

**Avec containers** :
```
Serveur 1: [Container A] [Container B] [Container C]
- Isolation complète
- Portable partout
- Déploiement simple
```

### Pourquoi Docker avec les microservices ?

#### Avantages
- **Isolation** : Chaque service dans son propre container
- **Portabilité** : "Build once, run anywhere"
- **Scalabilité** : Facile d'ajouter/supprimer des instances
- **Consistance** : Même environnement partout (dev, test, prod)
- **Efficacité** : Partage des ressources système

#### Analogie simple
Docker c'est comme un conteneur de transport maritime :
- **Standardisé** : Même taille partout dans le monde
- **Portable** : Peut aller sur camion, train, bateau
- **Isolé** : Le contenu ne se mélange pas
- **Efficace** : Optimise l'espace de transport

## Concepts fondamentaux de Docker

### Images vs Containers

```
Image Docker = Recette de cuisine
Container = Plat préparé avec cette recette
```

**Image** :
- Template en lecture seule
- Contient l'application + dépendances
- Peut être partagée et versionnée

**Container** :
- Instance en cours d'exécution d'une image
- Peut être démarré, arrêté, supprimé
- Peut avoir plusieurs containers de la même image

### Architecture Docker

```
┌─────────────────────────────────────┐
│           Applications              │
├─────────────────────────────────────┤
│            Containers               │
├─────────────────────────────────────┤
│           Docker Engine             │
├─────────────────────────────────────┤
│         Système d'exploitation      │
└─────────────────────────────────────┘
```

## Création d'un Dockerfile pour un microservice Go

### Dockerfile de base

```dockerfile
# Dockerfile basique pour un service Go
FROM golang:1.21-alpine AS builder

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de dépendances
COPY go.mod go.sum ./

# Télécharger les dépendances
RUN go mod download

# Copier le code source
COPY . .

# Compiler l'application
RUN go build -o main ./cmd/server

# Image finale
FROM alpine:latest

# Installer les certificats SSL pour les appels HTTPS
RUN apk --no-cache add ca-certificates

# Créer un utilisateur non-root pour la sécurité
RUN adduser -D -s /bin/sh appuser

WORKDIR /home/appuser

# Copier l'exécutable depuis l'étape de build
COPY --from=builder /app/main .

# Changer le propriétaire du fichier
RUN chown appuser:appuser main

# Utiliser l'utilisateur non-root
USER appuser

# Exposer le port
EXPOSE 8080

# Commande de démarrage
CMD ["./main"]
```

### Dockerfile multi-stage optimisé

```dockerfile
# Dockerfile optimisé pour la production
FROM golang:1.21-alpine AS builder

# Métadonnées
LABEL maintainer="votre-email@company.com"
LABEL description="User Service for E-commerce Platform"

# Variables d'environnement pour le build
ENV CGO_ENABLED=0
ENV GOOS=linux
ENV GOARCH=amd64

# Installer git pour les dépendances privées
RUN apk add --no-cache git

# Créer le répertoire de travail
WORKDIR /build

# Copier et télécharger les dépendances (mise en cache)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copier le code source
COPY . .

# Compiler avec optimisations
RUN go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -X main.version=$(git describe --tags --always)" \
    -o app \
    ./cmd/server

# Image finale ultra-légère
FROM scratch

# Copier les certificats SSL
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copier l'exécutable
COPY --from=builder /build/app /app

# Exposer le port
EXPOSE 8080

# Point d'entrée
ENTRYPOINT ["/app"]
```

### Dockerfile avec configuration flexible

```dockerfile
# Dockerfile avec support de configuration
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copier les dépendances
COPY go.mod go.sum ./
RUN go mod download

# Copier le code source
COPY . .

# Arguments de build
ARG BUILD_VERSION=dev
ARG BUILD_DATE
ARG GIT_COMMIT

# Compiler avec métadonnées
RUN go build \
    -ldflags="-X main.version=${BUILD_VERSION} -X main.buildDate=${BUILD_DATE} -X main.gitCommit=${GIT_COMMIT}" \
    -o main \
    ./cmd/server

# Image finale
FROM alpine:3.18

# Installer les outils nécessaires
RUN apk --no-cache add \
    ca-certificates \
    tzdata \
    && update-ca-certificates

# Créer un utilisateur pour l'application
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Créer les répertoires nécessaires
RUN mkdir -p /app/config /app/logs && \
    chown -R appuser:appgroup /app

WORKDIR /app

# Copier l'exécutable
COPY --from=builder /app/main .
COPY --from=builder /app/config/default.yaml ./config/

# Changer les permissions
RUN chmod +x main && \
    chown appuser:appgroup main

# Utiliser l'utilisateur non-root
USER appuser

# Variables d'environnement par défaut
ENV PORT=8080
ENV ENV=production
ENV LOG_LEVEL=info

# Exposer le port
EXPOSE $PORT

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:$PORT/health || exit 1

# Commande de démarrage
CMD ["./main"]
```

## Optimisations Docker

### Utilisation du .dockerignore

```bash
# .dockerignore
# Exclure les fichiers inutiles pour réduire la taille du context

# Fichiers de développement
.git/
.gitignore
README.md
Dockerfile*
docker-compose*.yml

# Fichiers de build locaux
vendor/
*.exe
*.dll
*.so
*.dylib

# Logs et données temporaires
logs/
*.log
tmp/
temp/

# Fichiers d'IDE
.vscode/
.idea/
*.swp
*.swo

# Fichiers de test
*_test.go
test/
coverage.out

# Documentation
docs/
*.md

# Fichiers de configuration locale
.env
.env.local
config/local.yaml
```

### Build multi-architecture

```dockerfile
# Dockerfile avec support multi-architecture
FROM --platform=$BUILDPLATFORM golang:1.21-alpine AS builder

# Arguments automatiques de Docker
ARG TARGETOS
ARG TARGETARCH
ARG BUILDPLATFORM

WORKDIR /app

# Copier les dépendances
COPY go.mod go.sum ./
RUN go mod download

# Copier le code source
COPY . .

# Compiler pour l'architecture cible
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build -a -installsuffix cgo -o main ./cmd/server

# Image finale
FROM alpine:latest

RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .
CMD ["./main"]
```

### Script de build optimisé

```bash
#!/bin/bash
# scripts/build-docker.sh

set -e

# Configuration
SERVICE_NAME=${1:-user-service}
VERSION=${2:-$(git describe --tags --always)}
REGISTRY=${DOCKER_REGISTRY:-localhost:5000}

# Couleurs pour l'affichage
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}🐳 Building Docker image for $SERVICE_NAME${NC}"

# Vérifier que Docker est disponible
if ! command -v docker &> /dev/null; then
    echo -e "${RED}❌ Docker is not installed${NC}"
    exit 1
fi

# Variables de build
BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ')
GIT_COMMIT=$(git rev-parse --short HEAD)

# Tags
IMAGE_NAME="$REGISTRY/$SERVICE_NAME"
TAGS=(
    "$IMAGE_NAME:$VERSION"
    "$IMAGE_NAME:latest"
)

echo -e "${YELLOW}📋 Build information:${NC}"
echo "  Service: $SERVICE_NAME"
echo "  Version: $VERSION"
echo "  Commit: $GIT_COMMIT"
echo "  Build Date: $BUILD_DATE"
echo ""

# Build de l'image
echo -e "${GREEN}🔨 Building image...${NC}"
docker build \
    --build-arg BUILD_VERSION="$VERSION" \
    --build-arg BUILD_DATE="$BUILD_DATE" \
    --build-arg GIT_COMMIT="$GIT_COMMIT" \
    --file "services/$SERVICE_NAME/Dockerfile" \
    --tag "${TAGS[0]}" \
    --tag "${TAGS[1]}" \
    "services/$SERVICE_NAME"

# Afficher la taille de l'image
echo -e "${GREEN}📦 Image size:${NC}"
docker images "$IMAGE_NAME" --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Security scan (optionnel)
if command -v trivy &> /dev/null; then
    echo -e "${YELLOW}🔒 Running security scan...${NC}"
    trivy image --severity HIGH,CRITICAL "${TAGS[0]}"
fi

echo -e "${GREEN}✅ Build completed successfully!${NC}"
echo ""
echo -e "${YELLOW}🚀 To run the container:${NC}"
echo "  docker run -p 8080:8080 ${TAGS[0]}"
echo ""
echo -e "${YELLOW}📤 To push to registry:${NC}"
echo "  docker push ${TAGS[0]}"
echo "  docker push ${TAGS[1]}"
```

## Docker Compose pour le développement

### docker-compose.yml de base

```yaml
version: '3.8'

services:
  # Service utilisateurs
  user-service:
    build:
      context: ./services/user-service
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    environment:
      - PORT=8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=users
      - DB_USER=postgres
      - DB_PASSWORD=password
      - REDIS_URL=redis:6379
    depends_on:
      - postgres
      - redis
    volumes:
      - ./services/user-service/config:/app/config
    networks:
      - microservices
    restart: unless-stopped

  # Service produits
  product-service:
    build:
      context: ./services/product-service
      dockerfile: Dockerfile
    ports:
      - "8082:8080"
    environment:
      - PORT=8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=products
      - DB_USER=postgres
      - DB_PASSWORD=password
    depends_on:
      - postgres
    networks:
      - microservices
    restart: unless-stopped

  # Service commandes
  order-service:
    build:
      context: ./services/order-service
      dockerfile: Dockerfile
    ports:
      - "8083:8080"
    environment:
      - PORT=8080
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=orders
      - DB_USER=postgres
      - DB_PASSWORD=password
      - USER_SERVICE_URL=http://user-service:8080
      - PRODUCT_SERVICE_URL=http://product-service:8080
    depends_on:
      - postgres
      - user-service
      - product-service
    networks:
      - microservices
    restart: unless-stopped

  # Base de données PostgreSQL
  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=ecommerce
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks:
      - microservices
    restart: unless-stopped

  # Redis pour le cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - microservices
    restart: unless-stopped
    command: redis-server --appendonly yes

  # API Gateway (Nginx)
  api-gateway:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./gateway/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - user-service
      - product-service
      - order-service
    networks:
      - microservices
    restart: unless-stopped

networks:
  microservices:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

### docker-compose.override.yml pour le développement

```yaml
version: '3.8'

# Override pour le développement
services:
  user-service:
    build:
      target: development  # Utiliser un stage de dev si défini
    volumes:
      - ./services/user-service:/app  # Live reload
    environment:
      - ENV=development
      - LOG_LEVEL=debug
      - HOT_RELOAD=true
    ports:
      - "9091:9090"  # Port pour les métriques

  product-service:
    volumes:
      - ./services/product-service:/app
    environment:
      - ENV=development
      - LOG_LEVEL=debug
    ports:
      - "9092:9090"

  order-service:
    volumes:
      - ./services/order-service:/app
    environment:
      - ENV=development
      - LOG_LEVEL=debug
    ports:
      - "9093:9090"

  # Outils de développement
  postgres-admin:
    image: adminer
    ports:
      - "8080:8080"
    environment:
      - ADMINER_DEFAULT_SERVER=postgres
    networks:
      - microservices

  redis-commander:
    image: rediscommander/redis-commander:latest
    ports:
      - "8081:8081"
    environment:
      - REDIS_HOSTS=local:redis:6379
    networks:
      - microservices
```

### docker-compose.prod.yml pour la production

```yaml
version: '3.8'

services:
  user-service:
    image: ${REGISTRY}/user-service:${VERSION}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 128M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - ENV=production
      - LOG_LEVEL=info
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  product-service:
    image: ${REGISTRY}/product-service:${VERSION}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    environment:
      - ENV=production
      - LOG_LEVEL=info
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  order-service:
    image: ${REGISTRY}/order-service:${VERSION}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
    environment:
      - ENV=production
      - LOG_LEVEL=info

  postgres:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 2G
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/postgres_password
    secrets:
      - postgres_password

secrets:
  postgres_password:
    external: true
```

## Configuration avancée

### Health checks et monitoring

```dockerfile
# Dockerfile avec health check avancé
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o main ./cmd/server

FROM alpine:latest
RUN apk --no-cache add ca-certificates wget curl
WORKDIR /root/
COPY --from=builder /app/main .

# Script de health check personnalisé
COPY healthcheck.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/healthcheck.sh

# Health check avec script personnalisé
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /usr/local/bin/healthcheck.sh

EXPOSE 8080
CMD ["./main"]
```

```bash
#!/bin/sh
# healthcheck.sh

# Vérifier que le service répond
if ! wget --no-verbose --tries=1 --spider --timeout=5 http://localhost:8080/health; then
    echo "Health check endpoint failed"
    exit 1
fi

# Vérifier la connectivité à la base de données
if ! wget --no-verbose --tries=1 --spider --timeout=5 http://localhost:8080/health/db; then
    echo "Database health check failed"
    exit 1
fi

# Vérifier la mémoire disponible
MEMORY_USAGE=$(cat /proc/meminfo | grep MemAvailable | awk '{print $2}')
if [ "$MEMORY_USAGE" -lt 50000 ]; then
    echo "Low memory warning: ${MEMORY_USAGE}KB available"
    exit 1
fi

echo "All health checks passed"
exit 0
```

### Configuration avec des secrets

```yaml
# docker-compose avec gestion des secrets
version: '3.8'

services:
  user-service:
    image: user-service:latest
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
    secrets:
      - db_password
      - jwt_secret
    networks:
      - microservices

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt

networks:
  microservices:
    driver: bridge

volumes:
  postgres_data:
```

```go
// Configuration pour lire les secrets dans Go
package config

import (
    "fmt"
    "io/ioutil"
    "os"
    "strings"
)

func GetSecretOrEnv(secretFile, envVar, defaultValue string) string {
    // Essayer de lire depuis un fichier secret
    if secretFile != "" {
        if data, err := ioutil.ReadFile(secretFile); err == nil {
            return strings.TrimSpace(string(data))
        }
    }

    // Fallback sur variable d'environnement
    if value := os.Getenv(envVar); value != "" {
        return value
    }

    return defaultValue
}

type DatabaseConfig struct {
    Host     string
    Port     string
    Name     string
    User     string
    Password string
}

func LoadDatabaseConfig() *DatabaseConfig {
    return &DatabaseConfig{
        Host: GetSecretOrEnv("", "DB_HOST", "localhost"),
        Port: GetSecretOrEnv("", "DB_PORT", "5432"),
        Name: GetSecretOrEnv("", "DB_NAME", "app"),
        User: GetSecretOrEnv("", "DB_USER", "app"),
        Password: GetSecretOrEnv(
            os.Getenv("DB_PASSWORD_FILE"),
            "DB_PASSWORD",
            "default_password",
        ),
    }
}
```

## Scripts d'automatisation

### Script de déploiement complet

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

# Configuration
ENVIRONMENT=${1:-development}
VERSION=${2:-latest}
COMPOSE_FILE="docker-compose.yml"

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}🚀 Deploying microservices stack${NC}"
echo "Environment: $ENVIRONMENT"
echo "Version: $VERSION"
echo ""

# Valider l'environnement
case $ENVIRONMENT in
    development)
        COMPOSE_FILE="docker-compose.yml"
        ;;
    staging)
        COMPOSE_FILE="docker-compose.staging.yml"
        ;;
    production)
        COMPOSE_FILE="docker-compose.prod.yml"
        ;;
    *)
        echo -e "${RED}❌ Invalid environment: $ENVIRONMENT${NC}"
        echo "Valid environments: development, staging, production"
        exit 1
        ;;
esac

# Fonction pour attendre qu'un service soit prêt
wait_for_service() {
    local service_url=$1
    local service_name=$2
    local timeout=${3:-60}

    echo -e "${YELLOW}⏳ Waiting for $service_name to be ready...${NC}"

    for i in $(seq 1 $timeout); do
        if curl -f "$service_url" >/dev/null 2>&1; then
            echo -e "${GREEN}✅ $service_name is ready${NC}"
            return 0
        fi
        sleep 1
    done

    echo -e "${RED}❌ $service_name failed to start within ${timeout}s${NC}"
    return 1
}

# Arrêter les services existants
echo -e "${YELLOW}🛑 Stopping existing services...${NC}"
docker-compose -f "$COMPOSE_FILE" down

# Nettoyer les images inutilisées (production seulement)
if [ "$ENVIRONMENT" = "production" ]; then
    echo -e "${YELLOW}🧹 Cleaning up unused images...${NC}"
    docker image prune -f
fi

# Démarrer les services
echo -e "${YELLOW}🚀 Starting services...${NC}"
export VERSION=$VERSION
docker-compose -f "$COMPOSE_FILE" up -d

# Attendre que les services soient prêts
echo -e "${YELLOW}⏳ Waiting for services to be ready...${NC}"
sleep 10

# Vérifier la santé des services
services=(
    "http://localhost:8081/health:User Service"
    "http://localhost:8082/health:Product Service"
    "http://localhost:8083/health:Order Service"
)

all_healthy=true
for service in "${services[@]}"; do
    IFS=':' read -r url name <<< "$service"
    if ! wait_for_service "$url" "$name" 30; then
        all_healthy=false
    fi
done

# Résumé du déploiement
echo ""
echo -e "${GREEN}📋 Deployment Summary${NC}"
echo "=================================="

if $all_healthy; then
    echo -e "${GREEN}✅ All services are healthy${NC}"
    echo ""
    echo -e "${YELLOW}🌐 Service endpoints:${NC}"
    echo "User Service: http://localhost:8081"
    echo "Product Service: http://localhost:8082"
    echo "Order Service: http://localhost:8083"
    echo "API Gateway: http://localhost:80"
    echo ""
    echo -e "${YELLOW}📊 Management tools:${NC}"
    if [ "$ENVIRONMENT" = "development" ]; then
        echo "Database Admin: http://localhost:8080"
        echo "Redis Commander: http://localhost:8081"
    fi
else
    echo -e "${RED}❌ Some services failed to start${NC}"
    echo ""
    echo -e "${YELLOW}🔍 Check logs with:${NC}"
    echo "docker-compose -f $COMPOSE_FILE logs [service-name]"
    exit 1
fi
```

### Script de surveillance des containers

```bash
#!/bin/bash
# scripts/monitor-containers.sh

# Monitoring en temps réel des containers

while true; do
    clear
    echo "=== Docker Containers Status ==="
    echo "$(date)"
    echo ""

    # Status des containers
    docker-compose ps
    echo ""

    # Utilisation des ressources
    echo "=== Resource Usage ==="
    docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"
    echo ""

    # Health checks
    echo "=== Health Status ==="
    for container in $(docker-compose ps -q); do
        name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
        health=$(docker inspect --format '{{.State.Health.Status}}' $container 2>/dev/null || echo "no healthcheck")
        echo "$name: $health"
    done

    echo ""
    echo "Press Ctrl+C to exit"
    sleep 10
done
```

### Script de backup automatique

```bash
#!/bin/bash
# scripts/backup.sh

set -e

BACKUP_DIR="/backup/microservices/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "🗄️ Creating backup in $BACKUP_DIR"

# Backup des volumes Docker
echo "📦 Backing up Docker volumes..."
for volume in $(docker volume ls -q | grep -E "(postgres|redis)"); do
    echo "  - Backing up volume: $volume"
    docker run --rm \
        -v "$volume":/data \
        -v "$BACKUP_DIR":/backup \
        alpine tar czf "/backup/${volume}.tar.gz" -C /data .
done

# Backup des configurations
echo "⚙️ Backing up configurations..."
tar czf "$BACKUP_DIR/configs.tar.gz" \
    docker-compose*.yml \
    services/ \
    gateway/ \
    scripts/ \
    .env* 2>/dev/null || true

# Backup des logs
echo "📝 Backing up logs..."
mkdir -p "$BACKUP_DIR/logs"
for container in $(docker-compose ps -q); do
    name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
    docker logs "$container" > "$BACKUP_DIR/logs/${name}.log" 2>&1
done

# Informations sur le backup
cat > "$BACKUP_DIR/backup-info.txt" << EOF
Backup created: $(date)
Docker version: $(docker --version)
Docker Compose version: $(docker-compose --version)
Running containers: $(docker-compose ps --services | tr '\n' ' ')
Backup size: $(du -sh "$BACKUP_DIR" | cut -f1)
EOF

echo "✅ Backup completed: $BACKUP_DIR"
echo "Backup size: $(du -sh "$BACKUP_DIR" | cut -f1)"
```

## Bonnes pratiques de sécurité

### Dockerfile sécurisé

```dockerfile
# Dockerfile avec bonnes pratiques de sécurité
FROM golang:1.21-alpine AS builder

# Installer uniquement les outils nécessaires
RUN apk add --no-cache git ca-certificates

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download && go mod verify

COPY . .

# Compiler avec optimisations de sécurité
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o app ./cmd/server

# Image finale minimale
FROM scratch

# Copier uniquement les certificats nécessaires
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copier l'exécutable
COPY --from=builder /app/app /app

# Utiliser un port non-privilégié
EXPOSE 8080

# Pas de shell = moins de surface d'attaque
ENTRYPOINT ["/app"]
```

## Scanner de sécurité

```bash
#!/bin/bash
# scripts/security-scan.sh

set -e

IMAGE_NAME=${1:-user-service:latest}

echo "🔒 Running security scan on $IMAGE_NAME"

# Vérifier que Trivy est installé
if ! command -v trivy &> /dev/null; then
    echo "Installing Trivy..."
    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
fi

# Scan des vulnérabilités
echo "🔍 Scanning for vulnerabilities..."
trivy image --severity HIGH,CRITICAL --format table "$IMAGE_NAME"

# Scan détaillé en JSON pour l'automatisation
echo ""
echo "📄 Generating detailed report..."
trivy image --severity HIGH,CRITICAL --format json --output "security-report-$(date +%Y%m%d).json" "$IMAGE_NAME"

# Scan des secrets
echo ""
echo "🔐 Scanning for secrets..."
trivy image --scanners secret "$IMAGE_NAME"

# Scan de configuration
echo ""
echo "⚙️ Scanning configuration..."
trivy image --scanners config "$IMAGE_NAME"

# Vérifier si l'image a des vulnérabilités critiques
CRITICAL_COUNT=$(trivy image --severity CRITICAL --format json "$IMAGE_NAME" | jq '.Results[].Vulnerabilities | length')

if [ "$CRITICAL_COUNT" -gt 0 ]; then
    echo ""
    echo "❌ CRITICAL: Found $CRITICAL_COUNT critical vulnerabilities"
    echo "🚫 Image should not be deployed to production"
    exit 1
else
    echo ""
    echo "✅ No critical vulnerabilities found"
    exit 0
fi
```

### Audit de sécurité automatisé

```bash
#!/bin/bash
# scripts/docker-security-audit.sh

set -e

echo "🔐 Docker Security Audit"
echo "======================="

# Vérifier la configuration Docker
echo "🐳 Checking Docker daemon configuration..."

# Vérifier que Docker daemon ne tourne pas en mode debug
if docker version --format '{{.Server.Version}}' | grep -q "debug"; then
    echo "⚠️  WARNING: Docker daemon is running in debug mode"
fi

# Vérifier les containers privilégiés
echo ""
echo "🔍 Checking for privileged containers..."
PRIVILEGED=$(docker ps --format "table {{.Names}}\t{{.Status}}" --filter "label=privileged=true")
if [ -n "$PRIVILEGED" ]; then
    echo "⚠️  WARNING: Found privileged containers:"
    echo "$PRIVILEGED"
fi

# Vérifier les containers avec des capabilities ajoutées
echo ""
echo "🔍 Checking for containers with added capabilities..."
for container in $(docker ps -q); do
    caps=$(docker inspect $container --format '{{.HostConfig.CapAdd}}')
    if [ "$caps" != "[]" ] && [ "$caps" != "<no value>" ]; then
        name=$(docker inspect $container --format '{{.Name}}')
        echo "⚠️  WARNING: Container $name has added capabilities: $caps"
    fi
done

# Vérifier les volumes avec montage en écriture sur /
echo ""
echo "🔍 Checking for dangerous volume mounts..."
for container in $(docker ps -q); do
    mounts=$(docker inspect $container --format '{{range .Mounts}}{{.Source}}:{{.Destination}}:{{.Mode}} {{end}}')
    if echo "$mounts" | grep -q ":/.*:.*rw"; then
        name=$(docker inspect $container --format '{{.Name}}')
        echo "⚠️  WARNING: Container $name has potentially dangerous mounts"
        echo "  $mounts"
    fi
done

# Vérifier les ports exposés
echo ""
echo "🔍 Checking exposed ports..."
for container in $(docker ps -q); do
    name=$(docker inspect $container --format '{{.Name}}')
    ports=$(docker port $container 2>/dev/null | grep '0.0.0.0' || true)
    if [ -n "$ports" ]; then
        echo "ℹ️  Container $name exposes ports:"
        echo "$ports"
    fi
done

# Vérifier les images avec des utilisateurs root
echo ""
echo "🔍 Checking for containers running as root..."
for container in $(docker ps -q); do
    user=$(docker inspect $container --format '{{.Config.User}}')
    if [ -z "$user" ] || [ "$user" = "root" ] || [ "$user" = "0" ]; then
        name=$(docker inspect $container --format '{{.Name}}')
        echo "⚠️  WARNING: Container $name is running as root"
    fi
done

echo ""
echo "✅ Security audit completed"
```

## Optimisations pour la production

### Configuration Docker production

```bash
# /etc/docker/daemon.json - Configuration production
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true,
  "seccomp-profile": "/etc/docker/seccomp.json",
  "default-ulimits": {
    "nofile": {
      "hard": 65536,
      "soft": 65536
    }
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": [
    "https://your-registry-mirror.com"
  ],
  "insecure-registries": [],
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5
}
```

### Profil Seccomp personnalisé

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "bind",
        "brk",
        "chdir",
        "chmod",
        "chown",
        "clone",
        "close",
        "connect",
        "dup",
        "dup2",
        "execve",
        "exit",
        "exit_group",
        "fcntl",
        "fstat",
        "futex",
        "getcwd",
        "getdents",
        "getgid",
        "getpid",
        "getppid",
        "getuid",
        "listen",
        "lstat",
        "mmap",
        "mprotect",
        "munmap",
        "open",
        "openat",
        "read",
        "readlink",
        "rt_sigaction",
        "rt_sigprocmask",
        "rt_sigreturn",
        "sched_yield",
        "select",
        "socket",
        "stat",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

### Dockerfile production optimisé

```dockerfile
# Dockerfile production avec toutes les optimisations
FROM golang:1.21-alpine AS builder

# Arguments de build
ARG BUILD_VERSION=unknown
ARG BUILD_DATE=unknown
ARG GIT_COMMIT=unknown

# Optimisations build
ENV CGO_ENABLED=0
ENV GOOS=linux
ENV GOARCH=amd64

# Installer uniquement les dépendances nécessaires
RUN apk add --no-cache git ca-certificates tzdata

# Créer un utilisateur non-root pour le build
RUN adduser -D -s /bin/sh builder
USER builder

WORKDIR /home/builder/app

# Cache des dépendances
COPY --chown=builder:builder go.mod go.sum ./
RUN go mod download && go mod verify

# Build de l'application
COPY --chown=builder:builder . .
RUN go build \
    -a \
    -installsuffix cgo \
    -ldflags="-w -s -X main.version=${BUILD_VERSION} -X main.buildDate=${BUILD_DATE} -X main.gitCommit=${GIT_COMMIT}" \
    -tags netgo \
    -o app \
    ./cmd/server

# Image finale ultra-minimale
FROM scratch

# Copier les certificats et timezone
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Copier l'exécutable
COPY --from=builder /home/builder/app/app /app

# Labels pour la métadonnée
LABEL maintainer="team@company.com"
LABEL version="${BUILD_VERSION}"
LABEL description="Production-ready microservice"

# Port non-privilégié
EXPOSE 8080

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["/app", "--health-check"]

# Utiliser l'exécutable directement (pas de shell)
ENTRYPOINT ["/app"]
```

### Docker Compose production avec haute disponibilité

```yaml
version: '3.8'

x-common-variables: &common-variables
  ENVIRONMENT: production
  LOG_LEVEL: info
  METRICS_ENABLED: "true"

x-resource-limits: &resource-limits
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 1G
      reservations:
        cpus: '0.1'
        memory: 128M

services:
  # Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - user-service
      - product-service
      - order-service
    deploy:
      replicas: 2
      <<: *resource-limits
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - frontend
      - backend

  # Microservices
  user-service:
    image: ${REGISTRY}/user-service:${VERSION}
    environment:
      <<: *common-variables
      SERVICE_NAME: user-service
      DB_HOST: postgres-primary
      REDIS_URL: redis-cluster
    deploy:
      replicas: 3
      <<: *resource-limits
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      rollback_config:
        parallelism: 1
        delay: 5s
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - backend
    depends_on:
      - postgres-primary
      - redis-cluster
    secrets:
      - db_password
      - jwt_secret

  product-service:
    image: ${REGISTRY}/product-service:${VERSION}
    environment:
      <<: *common-variables
      SERVICE_NAME: product-service
    deploy:
      replicas: 2
      <<: *resource-limits
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - backend
    secrets:
      - db_password

  order-service:
    image: ${REGISTRY}/order-service:${VERSION}
    environment:
      <<: *common-variables
      SERVICE_NAME: order-service
      USER_SERVICE_URL: http://user-service:8080
      PRODUCT_SERVICE_URL: http://product-service:8080
    deploy:
      replicas: 2
      <<: *resource-limits
    healthcheck:
      test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - backend
    secrets:
      - db_password

  # Database cluster
  postgres-primary:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: ecommerce
      POSTGRES_REPLICATION_MODE: master
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD_FILE: /run/secrets/replication_password
    volumes:
      - postgres_primary_data:/var/lib/postgresql/data
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./postgres/pg_hba.conf:/etc/postgresql/pg_hba.conf
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G
    networks:
      - backend
    secrets:
      - db_password
      - replication_password

  postgres-replica:
    image: postgres:15-alpine
    environment:
      POSTGRES_MASTER_SERVICE: postgres-primary
      POSTGRES_REPLICATION_MODE: slave
      POSTGRES_REPLICATION_USER: replicator
      POSTGRES_REPLICATION_PASSWORD_FILE: /run/secrets/replication_password
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
    networks:
      - backend
    depends_on:
      - postgres-primary
    secrets:
      - replication_password

  # Redis Cluster
  redis-cluster:
    image: redis:7-alpine
    command: redis-server /etc/redis/redis.conf --cluster-enabled yes
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/etc/redis/redis.conf
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    networks:
      - backend

# Réseaux séparés pour la sécurité
networks:
  frontend:
    driver: overlay
    external: true
  backend:
    driver: overlay
    internal: true

# Volumes persistants
volumes:
  postgres_primary_data:
    driver: local
  postgres_replica_data:
    driver: local
  redis_data:
    driver: local

# Secrets sécurisés
secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
  replication_password:
    external: true
```

## Monitoring et observabilité Docker

### Métriques Docker avec Prometheus

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # Exporteur de métriques Docker
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg
    networks:
      - monitoring

  # Exporteur de métriques système
  node-exporter:
    image: prom/node-exporter:v1.6.1
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
      - monitoring

  # Prometheus pour collecter les métriques
  prometheus:
    image: prom/prometheus:v2.47.0
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
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  # Grafana pour la visualisation
  grafana:
    image: grafana/grafana:10.1.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

### Configuration Prometheus pour Docker

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Métriques des containers Docker
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    scrape_interval: 10s

  # Métriques du système
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # Métriques des microservices
  - job_name: 'microservices'
    static_configs:
      - targets:
          - 'user-service:9090'
          - 'product-service:9090'
          - 'order-service:9090'
    scrape_interval: 5s

  # Métriques Docker Engine (si activé)
  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
```

### Dashboard Grafana pour Docker

```json
{
  "dashboard": {
    "title": "Docker Containers Monitoring",
    "panels": [
      {
        "title": "Container CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{name=~\".+\"}[5m]) * 100",
            "legendFormat": "{{name}}"
          }
        ],
        "yAxes": [{"label": "CPU %"}]
      },
      {
        "title": "Container Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{name=~\".+\"} / 1024 / 1024",
            "legendFormat": "{{name}}"
          }
        ],
        "yAxes": [{"label": "Memory (MB)"}]
      },
      {
        "title": "Container Network I/O",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_network_receive_bytes_total{name=~\".+\"}[5m])",
            "legendFormat": "{{name}} - RX"
          },
          {
            "expr": "rate(container_network_transmit_bytes_total{name=~\".+\"}[5m])",
            "legendFormat": "{{name}} - TX"
          }
        ],
        "yAxes": [{"label": "Bytes/sec"}]
      },
      {
        "title": "Container Disk I/O",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_fs_reads_bytes_total{name=~\".+\"}[5m])",
            "legendFormat": "{{name}} - Read"
          },
          {
            "expr": "rate(container_fs_writes_bytes_total{name=~\".+\"}[5m])",
            "legendFormat": "{{name}} - Write"
          }
        ],
        "yAxes": [{"label": "Bytes/sec"}]
      }
    ]
  }
}
```

## CI/CD avec Docker

### Pipeline GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - security
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  REGISTRY: $CI_REGISTRY
  IMAGE_TAG: $CI_COMMIT_SHA

# Tests unitaires
test:
  stage: test
  image: golang:1.21-alpine
  script:
    - go mod download
    - go test ./... -coverprofile=coverage.out
    - go tool cover -html=coverage.out -o coverage.html
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.html
  only:
    - merge_requests
    - main

# Build des images Docker
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
    - |
      for service in user-service product-service order-service; do
        echo "Building $service..."
        docker build \
          --build-arg BUILD_VERSION=$CI_COMMIT_TAG \
          --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
          --build-arg GIT_COMMIT=$CI_COMMIT_SHA \
          --tag $REGISTRY/$service:$IMAGE_TAG \
          --tag $REGISTRY/$service:latest \
          services/$service/

        docker push $REGISTRY/$service:$IMAGE_TAG
        docker push $REGISTRY/$service:latest
      done
  only:
    - main
    - tags

# Scan de sécurité
security:
  stage: security
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
  script:
    - |
      for service in user-service product-service order-service; do
        echo "Scanning $service for vulnerabilities..."
        trivy image --exit-code 1 --severity HIGH,CRITICAL $REGISTRY/$service:$IMAGE_TAG
      done
  allow_failure: false
  only:
    - main
    - tags

# Déploiement en staging
deploy_staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache docker-compose
  script:
    - export VERSION=$IMAGE_TAG
    - docker-compose -f docker-compose.staging.yml up -d
    - ./scripts/wait-for-services.sh
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

# Déploiement en production (manuel)
deploy_production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache docker-compose
  script:
    - export VERSION=$CI_COMMIT_TAG
    - docker-compose -f docker-compose.prod.yml up -d
    - ./scripts/wait-for-services.sh
    - ./scripts/smoke-tests.sh
  environment:
    name: production
    url: https://api.example.com
  when: manual
  only:
    - tags
```

### GitHub Actions

```yaml
# .github/workflows/docker.yml
name: Docker Build and Deploy

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: 1.21

      - name: Run tests
        run: |
          go mod download
          go test ./... -coverprofile=coverage.out

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.out

  build:
    needs: test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, product-service, order-service]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ matrix.service }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            BUILD_VERSION=${{ github.ref_name }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  security:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'

    steps:
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/user-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  deploy:
    needs: [build, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying to staging environment"
          # Commandes de déploiement ici
```

## Troubleshooting et debugging

### Scripts de diagnostic

```bash
#!/bin/bash
# scripts/docker-diagnostics.sh

echo "🔍 Docker Diagnostics Report"
echo "============================"
echo "Generated: $(date)"
echo ""

# Informations système
echo "📊 System Information"
echo "--------------------"
echo "Docker Version: $(docker --version)"
echo "Docker Compose Version: $(docker-compose --version)"
echo "OS: $(uname -a)"
echo "Available Memory: $(free -h | grep ^Mem | awk '{print $7}')"
echo "Available Disk: $(df -h / | tail -1 | awk '{print $4}')"
echo ""

# État des containers
echo "🐳 Container Status"
echo "------------------"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Image}}"
echo ""

# Utilisation des ressources
echo "📈 Resource Usage"
echo "----------------"
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"
echo ""

# Logs des containers ayant des erreurs
echo "🚨 Container Errors"
echo "------------------"
for container in $(docker ps -aq); do
    name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')

    # Vérifier si le container a redémarré récemment
    restart_count=$(docker inspect --format '{{.RestartCount}}' $container)
    if [ "$restart_count" -gt 0 ]; then
        echo "⚠️  $name has restarted $restart_count times"
        echo "Recent logs:"
        docker logs --tail 10 $container 2>&1 | sed 's/^/  /'
        echo ""
    fi

    # Vérifier si le container est en état d'erreur
    state=$(docker inspect --format '{{.State.Status}}' $container)
    if [ "$state" != "running" ]; then
        echo "❌ $name is in state: $state"
        echo "Recent logs:"
        docker logs --tail 10 $container 2>&1 | sed 's/^/  /'
        echo ""
    fi
done

# Volumes et leur utilisation
echo "💾 Volume Usage"
echo "--------------"
docker system df -v
echo ""

# Images et leur taille
echo "🖼️  Image Information"
echo "-------------------"
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}\t{{.CreatedSince}}"
echo ""

# Réseaux
echo "🌐 Network Information"
echo "---------------------"
docker network ls
echo ""

# Health checks
echo "🏥 Health Checks"
echo "---------------"
for container in $(docker ps -q); do
    name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
    health=$(docker inspect --format '{{.State.Health.Status}}' $container 2>/dev/null || echo "no healthcheck")

    if [ "$health" = "unhealthy" ]; then
        echo "❌ $name: $health"
        echo "Health check logs:"
        docker inspect --format '{{range .State.Health.Log}}{{.Output}}{{end}}' $container | tail -3 | sed 's/^/  /'
    elif [ "$health" = "healthy" ]; then
        echo "✅ $name: $health"
    else
        echo "⚪ $name: $health"
    fi
done
echo ""

# Connectivité réseau entre containers
echo "🔗 Network Connectivity"
echo "----------------------"
for container in $(docker ps --format '{{.Names}}'); do
    echo "Testing connectivity from $container:"

    # Test de connectivité vers les autres services
    for target in user-service product-service order-service postgres redis; do
        if [ "$container" != "$target" ]; then
            result=$(docker exec $container sh -c "nc -z $target 8080 2>/dev/null && echo 'OK' || echo 'FAIL'" 2>/dev/null || echo "N/A")
            echo "  $container -> $target: $result"
        fi
    done
    echo ""
done

# Recommandations de nettoyage
echo "🧹 Cleanup Recommendations"
echo "-------------------------"
unused_images=$(docker images -f "dangling=true" -q | wc -l)
if [ "$unused_images" -gt 0 ]; then
    echo "⚠️  Found $unused_images unused images. Run: docker image prune"
fi

unused_volumes=$(docker volume ls -f "dangling=true" -q | wc -l)
if [ "$unused_volumes" -gt 0 ]; then
    echo "⚠️  Found $unused_volumes unused volumes. Run: docker volume prune"
fi

stopped_containers=$(docker ps -f "status=exited" -q | wc -l)
if [ "$stopped_containers" -gt 0 ]; then
    echo "⚠️  Found $stopped_containers stopped containers. Run: docker container prune"
fi

echo ""
echo "✅ Diagnostics completed"
```

### Script de dépannage automatique

```bash
#!/bin/bash
# scripts/docker-auto-fix.sh

set -e

echo "🔧 Docker Auto-Fix Utility"
echo "=========================="

# Fonction pour redémarrer un service avec retry
restart_service_with_retry() {
    local service=$1
    local max_attempts=3
    local attempt=1

    while [ $attempt -le $max_attempts ]; do
        echo "Attempt $attempt/$max_attempts: Restarting $service..."

        if docker-compose restart $service; then
            echo "✅ $service restarted successfully"

            # Attendre que le service soit prêt
            sleep 10
            if docker-compose ps $service | grep -q "Up"; then
                echo "✅ $service is running"
                return 0
            fi
        fi

        attempt=$((attempt + 1))
        sleep 5
    done

    echo "❌ Failed to restart $service after $max_attempts attempts"
    return 1
}

# Vérifier l'espace disque
check_disk_space() {
    echo "💾 Checking disk space..."

    disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

    if [ "$disk_usage" -gt 85 ]; then
        echo "⚠️  Disk usage is high: ${disk_usage}%"
        echo "🧹 Cleaning up Docker resources..."

        # Nettoyer les containers arrêtés
        docker container prune -f

        # Nettoyer les images inutilisées
        docker image prune -f

        # Nettoyer les volumes inutilisés
        docker volume prune -f

        # Nettoyer les réseaux inutilisés
        docker network prune -f

        new_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        echo "✅ Disk usage after cleanup: ${new_usage}%"
    else
        echo "✅ Disk usage is OK: ${disk_usage}%"
    fi
}

# Vérifier la mémoire
check_memory() {
    echo "🧠 Checking memory usage..."

    available_mem=$(free | grep ^Mem | awk '{print $7}')
    total_mem=$(free | grep ^Mem | awk '{print $2}')
    mem_percent=$((100 - (available_mem * 100 / total_mem)))

    if [ "$mem_percent" -gt 90 ]; then
        echo "⚠️  Memory usage is high: ${mem_percent}%"
        echo "🔄 Restarting memory-intensive services..."

        # Redémarrer les services gourmands en mémoire
        for service in order-service product-service; do
            if docker-compose ps $service | grep -q "Up"; then
                restart_service_with_retry $service
            fi
        done
    else
        echo "✅ Memory usage is OK: ${mem_percent}%"
    fi
}

# Vérifier les containers en échec
check_failed_containers() {
    echo "🚨 Checking for failed containers..."

    failed_containers=$(docker ps -f "status=exited" --format "{{.Names}}")

    if [ -n "$failed_containers" ]; then
        echo "Found failed containers: $failed_containers"

        for container in $failed_containers; do
            echo "📋 Checking logs for $container..."
            docker logs --tail 20 $container

            echo "🔄 Attempting to restart $container..."
            restart_service_with_retry $container
        done
    else
        echo "✅ No failed containers found"
    fi
}

# Vérifier la connectivité réseau
check_network_connectivity() {
    echo "🌐 Checking network connectivity..."

    # Tester la connectivité entre services critiques
    services=("user-service" "product-service" "order-service")

    for service in "${services[@]}"; do
        if docker-compose ps $service | grep -q "Up"; then
            # Test de connectivité vers la base de données
            if ! docker-compose exec -T $service sh -c "nc -z postgres 5432" 2>/dev/null; then
                echo "❌ $service cannot connect to database"
                echo "🔄 Restarting $service..."
                restart_service_with_retry $service
            else
                echo "✅ $service database connectivity OK"
            fi

            # Test de connectivité vers Redis
            if ! docker-compose exec -T $service sh -c "nc -z redis 6379" 2>/dev/null; then
                echo "❌ $service cannot connect to Redis"
                echo "🔄 Restarting Redis..."
                restart_service_with_retry redis
            else
                echo "✅ $service Redis connectivity OK"
            fi
        fi
    done
}

# Vérifier les health checks
check_health_status() {
    echo "🏥 Checking health status..."

    for container in $(docker ps -q); do
        name=$(docker inspect --format '{{.Name}}' $container | sed 's/\///')
        health=$(docker inspect --format '{{.State.Health.Status}}' $container 2>/dev/null || echo "no healthcheck")

        if [ "$health" = "unhealthy" ]; then
            echo "❌ $name is unhealthy"
            echo "🔄 Restarting $name..."
            restart_service_with_retry $name
        fi
    done
}

# Exécuter toutes les vérifications
main() {
    echo "🚀 Starting auto-fix procedures..."
    echo ""

    check_disk_space
    echo ""

    check_memory
    echo ""

    check_failed_containers
    echo ""

    check_network_connectivity
    echo ""

    check_health_status
    echo ""

    echo "✅ Auto-fix completed"
    echo ""
    echo "📊 Final system status:"
    docker-compose ps
}

# Exécuter le script principal
main
```

## Performance et optimisation

### Optimisation des images Docker

```bash
#!/bin/bash
# scripts/optimize-images.sh

echo "🚀 Docker Image Optimization"
echo "============================"

# Analyse de la taille des images
echo "📊 Current image sizes:"
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
echo ""

# Fonction pour analyser les layers d'une image
analyze_image_layers() {
    local image=$1
    echo "🔍 Analyzing layers for $image:"

    # Utiliser dive si disponible
    if command -v dive &> /dev/null; then
        dive $image --ci
    else
        echo "Install 'dive' tool for detailed layer analysis"
        docker history $image --format "table {{.CreatedBy}}\t{{.Size}}"
    fi
    echo ""
}

# Optimiser une image spécifique
optimize_image() {
    local service=$1
    local dockerfile_path="services/$service/Dockerfile"

    echo "🔧 Optimizing $service image..."

    # Créer un Dockerfile optimisé
    cat > "${dockerfile_path}.optimized" << 'EOF'
# Multi-stage build optimisé
FROM golang:1.21-alpine AS builder

# Installer uniquement les dépendances de build nécessaires
RUN apk add --no-cache git ca-certificates

WORKDIR /build

# Cache des dépendances Go
COPY go.mod go.sum ./
RUN go mod download

# Build avec optimisations
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -a -installsuffix cgo \
    -ldflags="-w -s" \
    -o app ./cmd/server

# Image finale ultra-légère
FROM scratch

# Copier uniquement le nécessaire
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/app /app

EXPOSE 8080
ENTRYPOINT ["/app"]
EOF

    # Build de l'image optimisée
    echo "Building optimized image for $service..."
    docker build -f "${dockerfile_path}.optimized" -t "${service}:optimized" "services/$service/"

    # Comparer les tailles
    original_size=$(docker images --format "{{.Size}}" "${service}:latest" 2>/dev/null || echo "N/A")
    optimized_size=$(docker images --format "{{.Size}}" "${service}:optimized")

    echo "Original size: $original_size"
    echo "Optimized size: $optimized_size"
    echo ""
}

# Optimiser toutes les images
services=("user-service" "product-service" "order-service")

for service in "${services[@]}"; do
    if [ -f "services/$service/Dockerfile" ]; then
        analyze_image_layers "$service:latest"
        optimize_image "$service"
    fi
done

echo "✅ Optimization completed"
echo ""
echo "🏃 To use optimized images, update your docker-compose.yml:"
echo "  image: service-name:optimized"
```

### Monitoring des performances

```go
// Performance monitoring pour les containers
package monitoring

import (
    "context"
    "fmt"
    "time"

    "github.com/docker/docker/api/types"
    "github.com/docker/docker/client"
)

type ContainerMonitor struct {
    client *client.Client
}

func NewContainerMonitor() (*ContainerMonitor, error) {
    cli, err := client.NewClientWithOpts(client.FromEnv)
    if err != nil {
        return nil, err
    }

    return &ContainerMonitor{client: cli}, nil
}

type ContainerStats struct {
    ID          string    `json:"id"`
    Name        string    `json:"name"`
    CPUPercent  float64   `json:"cpu_percent"`
    MemoryUsage uint64    `json:"memory_usage"`
    MemoryLimit uint64    `json:"memory_limit"`
    MemoryPercent float64 `json:"memory_percent"`
    NetworkRx   uint64    `json:"network_rx"`
    NetworkTx   uint64    `json:"network_tx"`
    Timestamp   time.Time `json:"timestamp"`
}

func (cm *ContainerMonitor) GetContainerStats(ctx context.Context, containerID string) (*ContainerStats, error) {
    stats, err := cm.client.ContainerStats(ctx, containerID, false)
    if err != nil {
        return nil, err
    }
    defer stats.Body.Close()

    var v types.StatsJSON
    if err := json.NewDecoder(stats.Body).Decode(&v); err != nil {
        return nil, err
    }

    // Calculer le pourcentage CPU
    cpuPercent := calculateCPUPercent(&v)

    // Calculer le pourcentage mémoire
    memPercent := float64(v.MemoryStats.Usage) / float64(v.MemoryStats.Limit) * 100

    // Calculer le réseau
    var networkRx, networkTx uint64
    for _, network := range v.Networks {
        networkRx += network.RxBytes
        networkTx += network.TxBytes
    }

    return &ContainerStats{
        ID:            containerID,
        Name:          v.Name,
        CPUPercent:    cpuPercent,
        MemoryUsage:   v.MemoryStats.Usage,
        MemoryLimit:   v.MemoryStats.Limit,
        MemoryPercent: memPercent,
        NetworkRx:     networkRx,
        NetworkTx:     networkTx,
        Timestamp:     time.Now(),
    }, nil
}

func calculateCPUPercent(stats *types.StatsJSON) float64 {
    cpuUsage := stats.CPUStats.CPUUsage.TotalUsage
    prevCPUUsage := stats.PreCPUStats.CPUUsage.TotalUsage

    systemUsage := stats.CPUStats.SystemUsage
    prevSystemUsage := stats.PreCPUStats.SystemUsage

    cpuDelta := float64(cpuUsage - prevCPUUsage)
    systemDelta := float64(systemUsage - prevSystemUsage)

    if systemDelta > 0 && cpuDelta > 0 {
        return (cpuDelta / systemDelta) * float64(len(stats.CPUStats.CPUUsage.PercpuUsage)) * 100
    }

    return 0
}

// API endpoint pour exposer les métriques
func (cm *ContainerMonitor) MonitorAllContainers(ctx context.Context) ([]*ContainerStats, error) {
    containers, err := cm.client.ContainerList(ctx, types.ContainerListOptions{})
    if err != nil {
        return nil, err
    }

    var stats []*ContainerStats
    for _, container := range containers {
        containerStats, err := cm.GetContainerStats(ctx, container.ID)
        if err != nil {
            continue // Ignorer les erreurs pour les containers individuels
        }
        stats = append(stats, containerStats)
    }

    return stats, nil
}
```

## Sécurité avancée

### Configuration AppArmor

```bash
# /etc/apparmor.d/docker-microservice
#include <tunables/global>

profile docker-microservice flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # Autoriser seulement les syscalls nécessaires
  capability,
  network inet stream,
  network inet dgram,

  # Accès aux fichiers
  /app ix,
  /etc/ssl/certs/** r,
  /tmp/** rw,
  /var/log/** w,

  # Interdire l'accès au système de fichiers sensible
  deny /proc/sys/** w,
  deny /sys/** w,
  deny /etc/passwd r,
  deny /etc/shadow r,
  deny /etc/group r,

  # Autoriser uniquement les ports nécessaires
  network inet stream bind port 8080,
  network inet stream bind port 9090,
}
```

### Configuration SELinux

```bash
# Politique SELinux personnalisée pour les microservices
module microservice_policy 1.0;

require {
    type container_t;
    type container_file_t;
    type http_port_t;
    class tcp_socket { bind listen };
    class file { read write create };
}

# Autoriser les containers à se lier aux ports HTTP
allow container_t http_port_t:tcp_socket { bind listen };

# Autoriser l'accès aux fichiers de configuration
allow container_t container_file_t:file { read write create };
```

### Scanner de conformité

```bash
#!/bin/bash
# scripts/compliance-check.sh

echo "🔒 Docker Compliance Check"
echo "========================="

# CIS Docker Benchmark checks
echo "📋 Running CIS Docker Benchmark checks..."

# 1. Vérifier que Docker daemon n'est pas en mode swarm
if docker info 2>/dev/null | grep -q "Swarm: active"; then
    echo "⚠️  WARNING: Docker is running in swarm mode"
else
    echo "✅ Docker swarm mode check passed"
fi

# 2. Vérifier que les containers ne tournent pas en mode privilégié
echo ""
echo "🔍 Checking for privileged containers..."
privileged_containers=$(docker ps --quiet --filter "label=privileged=true")
if [ -n "$privileged_containers" ]; then
    echo "❌ FAIL: Privileged containers found"
    docker ps --format "table {{.Names}}\t{{.Image}}" --filter "label=privileged=true"
else
    echo "✅ No privileged containers found"
fi

# 3. Vérifier que les containers ne montent pas /var/run/docker.sock
echo ""
echo "🔍 Checking for Docker socket mounts..."
for container in $(docker ps -q); do
    mounts=$(docker inspect $container --format '{{range .Mounts}}{{.Source}}{{end}}')
    if echo "$mounts" | grep -q "/var/run/docker.sock"; then
        name=$(docker inspect $container --format '{{.Name}}')
        echo "❌ FAIL: Container $name mounts Docker socket"
    fi
done

# 4. Vérifier que les containers utilisent des utilisateurs non-root
echo ""
echo "🔍 Checking for root users in containers..."
for container in $(docker ps -q); do
    user=$(docker inspect $container --format '{{.Config.User}}')
    name=$(docker inspect $container --format '{{.Name}}')

    if [ -z "$user" ] || [ "$user" = "root" ] || [ "$user" = "0" ]; then
        echo "⚠️  WARNING: Container $name is running as root"
    else
        echo "✅ Container $name is using non-root user: $user"
    fi
done

# 5. Vérifier que les images ont des tags spécifiques (pas latest)
echo ""
echo "🔍 Checking for latest tags..."
for container in $(docker ps -q); do
    image=$(docker inspect $container --format '{{.Config.Image}}')
    name=$(docker inspect $container --format '{{.Name}}')

    if echo "$image" | grep -q ":latest$"; then
        echo "⚠️  WARNING: Container $name uses 'latest' tag: $image"
    else
        echo "✅ Container $name uses specific tag: $image"
    fi
done

# 6. Vérifier les limites de ressources
echo ""
echo "🔍 Checking resource limits..."
for container in $(docker ps -q); do
    name=$(docker inspect $container --format '{{.Name}}')
    memory=$(docker inspect $container --format '{{.HostConfig.Memory}}')
    cpus=$(docker inspect $container --format '{{.HostConfig.CpuQuota}}')

    if [ "$memory" = "0" ]; then
        echo "⚠️  WARNING: Container $name has no memory limit"
    else
        memory_mb=$((memory / 1024 / 1024))
        echo "✅ Container $name has memory limit: ${memory_mb}MB"
    fi

    if [ "$cpus" = "0" ] || [ "$cpus" = "-1" ]; then
        echo "⚠️  WARNING: Container $name has no CPU limit"
    else
        echo "✅ Container $name has CPU limit"
    fi
done

echo ""
echo "✅ Compliance check completed"
```

## Récapitulatif et bonnes pratiques

### Checklist Docker pour la production

```markdown
# Checklist Docker Production

## Sécurité
- [ ] Images basées sur des distributions minimales (Alpine, Distroless)
- [ ] Utilisateur non-root dans tous les containers
- [ ] Pas de secrets en variables d'environnement
- [ ] Scan de sécurité automatisé (Trivy, Snyk)
- [ ] Signature des images Docker
- [ ] Mise à jour régulière des images de base

## Performance
- [ ] Multi-stage builds pour réduire la taille
- [ ] .dockerignore optimisé
- [ ] Limites de ressources définies (CPU, mémoire)
- [ ] Health checks configurés
- [ ] Logs avec rotation automatique
- [ ] Monitoring des métriques containers

## Fiabilité
- [ ] Images versionnées (pas de :latest en production)
- [ ] Health checks pour tous les services
- [ ] Restart policies configurées
- [ ] Backup des volumes critiques
- [ ] Tests d'intégration avec Docker
- [ ] Rollback automatique en cas d'échec

## Observabilité
- [ ] Logging centralisé configuré
- [ ] Métriques exposées (/metrics)
- [ ] Tracing distribué activé
- [ ] Dashboards de monitoring
- [ ] Alertes configurées
- [ ] Documentation des runbooks
```

### Commandes Docker essentielles

```bash
# Nettoyage et maintenance
docker system prune -a                 # Nettoyer tout
docker image prune -f                  # Supprimer images inutilisées
docker volume prune -f                 # Supprimer volumes inutilisés
docker container prune -f              # Supprimer containers arrêtés

# Monitoring et debugging
docker stats                           # Utilisation des ressources
docker logs -f container-name          # Suivre les logs en temps réel
docker exec -it container-name sh      # Shell dans un container
docker inspect container-name          # Informations détaillées

# Performance et diagnostic
docker system df                       # Utilisation de l'espace disque
docker history image-name              # Historique des layers
docker top container-name              # Processus dans le container

# Sauvegarde et restauration
docker save image-name > image.tar     # Sauvegarder une image
docker load < image.tar                # Restaurer une image
docker export container > backup.tar   # Exporter un container
docker import backup.tar new-image     # Importer un container

# Sécurité
docker scan image-name                  # Scanner les vulnérabilités
docker trust inspect image-name        # Vérifier la signature
docker secret ls                       # Lister les secrets
```

### Patterns Docker recommandés

```dockerfile
# Pattern 1: Build optimisé avec cache
FROM golang:1.21-alpine AS dependencies
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

FROM dependencies AS builder
COPY . .
RUN go build -o app ./cmd/server

FROM alpine:latest AS runtime
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/app /app
ENTRYPOINT ["/app"]

# Pattern 2: Configuration flexible
FROM alpine:latest
ENV PORT=8080 \
    LOG_LEVEL=info \
    ENV=production
EXPOSE $PORT
CMD ["./app", "--port", "$PORT"]

# Pattern 3: Health check intelligent
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ./app --health-check || exit 1

# Pattern 4: Utilisateur non-root
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup
USER appuser
```

## Conclusion

La containerisation avec Docker est essentielle pour les microservices modernes. Elle apporte :

### **Avantages clés**
- **Isolation** : Chaque service dans son environnement
- **Portabilité** : "Build once, run anywhere"
- **Scalabilité** : Facilite la montée en charge
- **Consistance** : Même environnement partout

### **Points d'attention**
- **Sécurité** : Utiliser des images minimales et des utilisateurs non-root
- **Performance** : Optimiser la taille des images et les ressources
- **Monitoring** : Surveiller les containers en production
- **Maintenance** : Automatiser les mises à jour et le nettoyage

### **Étapes suivantes**
1. Commencer par containeriser un service simple
2. Mettre en place le monitoring
3. Automatiser le build et le déploiement
4. Ajouter la sécurité et la conformité
5. Optimiser les performances

Docker transforme la façon de développer, déployer et maintenir les microservices. Maîtriser ces concepts vous permettra de construire des systèmes robustes et scalables !

⏭️



⏭️
