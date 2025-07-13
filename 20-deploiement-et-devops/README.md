🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20. Déploiement et DevOps

## Introduction

Le déploiement et les pratiques DevOps sont des aspects cruciaux du développement d'applications Go en production. Cette section vous guidera à travers les meilleures pratiques pour déployer, monitorer et maintenir des applications Go dans des environnements de production réels.

## Pourquoi les pratiques DevOps sont-elles importantes pour Go ?

Go a été conçu dès le départ pour être un langage adapté aux systèmes distribués et aux environnements cloud. Ses caractéristiques naturelles en font un excellent choix pour les pratiques DevOps :

### Avantages de Go pour le DevOps

**Binaires statiques :**
- Compilation en binaire unique sans dépendances externes
- Déploiement simplifié sur différents environnements
- Images Docker ultra-légères (scratch, distroless)

**Performance et efficacité :**
- Faible consommation mémoire et CPU
- Temps de démarrage rapide
- Excellent pour les microservices et conteneurs

**Concurrence native :**
- Goroutines permettent de gérer efficacement les charges
- Adapté aux architectures distribuées
- Scaling horizontal facilité

**Écosystème Cloud Native :**
- Kubernetes est écrit en Go
- Nombreux outils DevOps développés en Go (Docker, Prometheus, etc.)
- Intégration naturelle avec les technologies cloud

## Défis du déploiement Go

Bien que Go facilite le déploiement, certains défis subsistent :

### Gestion de la configuration
```go
// Exemple : Configuration d'environnement
type Config struct {
    Port     string `env:"PORT" envDefault:"8080"`
    DBHost   string `env:"DB_HOST" envDefault:"localhost"`
    LogLevel string `env:"LOG_LEVEL" envDefault:"info"`
}
```

### Gestion des secrets
- Variables d'environnement vs fichiers de configuration
- Intégration avec des gestionnaires de secrets (Vault, AWS Secrets Manager)
- Rotation des clés et certificats

### Monitoring et observabilité
- Métriques applicatives (Prometheus)
- Logging structuré
- Tracing distribué
- Health checks

## Architecture de déploiement moderne

### Approche traditionnelle vs Cloud Native

**Déploiement traditionnel :**
```
Application → Serveur physique/VM → Load Balancer → Utilisateurs
```

**Déploiement Cloud Native :**
```
Code → Container → Orchestrateur (K8s) → Service Mesh → Utilisateurs
```

### Patterns de déploiement

**Blue-Green Deployment :**
- Deux environnements identiques
- Basculement instantané
- Rollback rapide en cas de problème

**Canary Deployment :**
- Déploiement progressif
- Test sur un sous-ensemble d'utilisateurs
- Réduction des risques

**Rolling Update :**
- Mise à jour progressive des instances
- Zéro downtime
- Rollback automatique en cas d'échec

## Containerisation avec Go

### Dockerfile optimisé pour Go

```dockerfile
# Multi-stage build pour optimiser la taille
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Image finale ultra-légère
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/main .
CMD ["./main"]
```

### Bonnes pratiques de containerisation

**Sécurité :**
- Utiliser des images de base minimales
- Scanner les vulnérabilités
- Exécuter avec un utilisateur non-root

**Performance :**
- Optimiser les layers Docker
- Utiliser le cache de build
- Minimiser la taille des images

**Observabilité :**
- Logs structurés vers stdout
- Métriques exposées
- Health checks intégrés

## Infrastructure as Code (IaC)

### Outils populaires

**Terraform :**
```hcl
resource "aws_ecs_service" "go_app" {
  name            = "go-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.go_app.arn
  desired_count   = 3
}
```

**Helm Charts pour Kubernetes :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.app.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      containers:
      - name: {{ .Values.app.name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: {{ .Values.app.port }}
```

## Environnements de déploiement

### Stratégie d'environnements

**Development :**
- Environnement local
- Docker Compose pour les dépendances
- Hot reload pour le développement

**Staging :**
- Réplique de production
- Tests d'intégration
- Validation des déploiements

**Production :**
- Haute disponibilité
- Monitoring avancé
- Backup et disaster recovery

### Configuration par environnement

```go
type Environment string

const (
    Development Environment = "development"
    Staging     Environment = "staging"
    Production  Environment = "production"
)

func LoadConfig(env Environment) *Config {
    switch env {
    case Production:
        return &Config{
            LogLevel: "warn",
            Debug:    false,
            Timeout:  30 * time.Second,
        }
    case Staging:
        return &Config{
            LogLevel: "info",
            Debug:    true,
            Timeout:  10 * time.Second,
        }
    default:
        return &Config{
            LogLevel: "debug",
            Debug:    true,
            Timeout:  5 * time.Second,
        }
    }
}
```

## Prérequis pour cette section

Avant de continuer avec les sections suivantes, assurez-vous de maîtriser :

1. **Concepts Docker de base**
   - Création et gestion d'images
   - Docker Compose
   - Registries Docker

2. **Notions de cloud computing**
   - Services cloud (AWS, GCP, Azure)
   - Concepts de réseau
   - Stockage et bases de données cloud

3. **Bases de Kubernetes (optionnel mais recommandé)**
   - Pods, Services, Deployments
   - ConfigMaps et Secrets
   - Ingress Controllers

4. **Outils de ligne de commande**
   - Git pour le versioning
   - Make pour l'automatisation
   - cURL pour les tests API

## Objectifs de cette section

À la fin de cette section, vous serez capable de :

- Mettre en place un pipeline CI/CD complet pour une application Go
- Implémenter un monitoring et une observabilité efficaces
- Sécuriser vos déploiements Go
- Appliquer les bonnes pratiques DevOps en production

## Structure des sous-sections

Cette section est organisée en quatre parties principales :

1. **CI/CD avec Go** - Automatisation du build, test et déploiement
2. **Monitoring et observabilité** - Métriques, logs et tracing
3. **Sécurité** - Pratiques de sécurité pour les applications Go
4. **Bonnes pratiques en production** - Optimisations et maintenance

Chaque sous-section comprendra des exemples pratiques, des configurations réelles et des cas d'usage concrets pour vous permettre d'appliquer immédiatement ces concepts dans vos projets.

⏭️
