🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Projet 3 : Microservice

## Introduction

Bienvenue dans ce troisième projet pratique où nous allons construire un microservice complet en Go. Ce projet représente l'aboutissement de vos connaissances acquises jusqu'à présent et vous permettra d'appliquer les concepts avancés dans un contexte réel de développement moderne.

## Objectifs du projet

À la fin de ce projet, vous serez capable de :

- Concevoir et implémenter une architecture microservices robuste
- Mettre en place des mécanismes de communication inter-services efficaces
- Intégrer des solutions d'observabilité (logs, métriques, tracing)
- Containeriser votre application avec Docker
- Déployer et gérer des microservices en production

## Contexte du projet

Nous allons développer un système de gestion de commandes pour une plateforme e-commerce. Ce système sera composé de plusieurs microservices :

- **Service Produits** : Gestion du catalogue produits
- **Service Commandes** : Traitement des commandes
- **Service Utilisateurs** : Authentification et gestion des utilisateurs
- **Service Notifications** : Envoi de notifications (email, SMS)
- **API Gateway** : Point d'entrée unique pour les clients

## Technologies utilisées

Pour ce projet, nous utiliserons les technologies suivantes :

- **Go 1.21+** : Langage principal
- **Gin** : Framework HTTP léger et performant
- **PostgreSQL** : Base de données relationnelle
- **Redis** : Cache et message broker
- **Docker** : Containerisation
- **Docker Compose** : Orchestration locale
- **Prometheus** : Métriques
- **Grafana** : Visualisation des métriques
- **Jaeger** : Tracing distribué
- **logrus** : Logging structuré

## Prérequis

Avant de commencer ce projet, assurez-vous d'avoir :

### Connaissances requises
- Maîtrise des concepts Go fondamentaux (variables, fonctions, structs, interfaces)
- Compréhension des goroutines et channels
- Expérience avec les APIs REST et HTTP
- Connaissance des bases de données SQL
- Familiarité avec les concepts de concurrence

### Outils à installer
- Go 1.21 ou version supérieure
- Docker et Docker Compose
- PostgreSQL (ou via Docker)
- Redis (ou via Docker)
- Git pour le versioning
- Un éditeur de code (VS Code, GoLand, etc.)

## Structure du projet

Voici l'organisation générale que nous suivrons :

```
microservice-ecommerce/
├── cmd/                          # Points d'entrée des applications
│   ├── api-gateway/
│   ├── product-service/
│   ├── order-service/
│   ├── user-service/
│   └── notification-service/
├── internal/                     # Code interne non exportable
│   ├── config/                   # Configuration
│   ├── domain/                   # Modèles métier
│   ├── repository/               # Couche d'accès aux données
│   ├── service/                  # Logique métier
│   ├── handler/                  # Handlers HTTP
│   ├── middleware/               # Middlewares
│   └── observability/            # Monitoring et logging
├── pkg/                          # Packages réutilisables
│   ├── database/
│   ├── cache/
│   ├── messaging/
│   └── logger/
├── api/                          # Définitions API (OpenAPI/Swagger)
├── deployments/                  # Fichiers de déploiement
│   ├── docker/
│   └── k8s/
├── scripts/                      # Scripts d'automatisation
├── docs/                         # Documentation
└── docker-compose.yml           # Configuration locale
```

## Approche de développement

Nous suivrons une approche itérative :

1. **Phase 1** : Conception de l'architecture globale
2. **Phase 2** : Développement des services de base
3. **Phase 3** : Implémentation de la communication inter-services
4. **Phase 4** : Intégration de l'observabilité
5. **Phase 5** : Containerisation et déploiement

## Principes de développement

Tout au long de ce projet, nous respecterons les principes suivants :

### Clean Architecture
- Séparation claire des responsabilités
- Indépendance des frameworks
- Testabilité maximale

### Twelve-Factor App
- Configuration via variables d'environnement
- Séparation des préoccupations
- Stateless design

### Patterns microservices
- Single Responsibility Principle
- Database per service
- API First design
- Circuit Breaker pattern

## Métriques de succès

À la fin de ce projet, votre système devra :

- Gérer 1000+ requêtes par seconde
- Avoir un temps de réponse moyen < 100ms
- Atteindre 99% de disponibilité
- Avoir une couverture de tests > 80%
- Supporter la montée en charge horizontale

## Défis à relever

Ce projet vous confrontera à des défis réels :

- **Complexité distribuée** : Gestion de la cohérence des données
- **Résilience** : Tolérance aux pannes et recovery
- **Observabilité** : Monitoring et debugging distribué
- **Sécurité** : Authentification et autorisation inter-services
- **Performance** : Optimisation des communications réseau

## Ressources utiles

- [Go Microservices Blog](https://go.dev/blog/)
- [Microservices Patterns](https://microservices.io/patterns/)
- [12 Factor App](https://12factor.net/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)

## Prochaines étapes

Dans les sections suivantes, nous aborderons :
- Les principes d'architecture microservices
- L'implémentation pas à pas de chaque service
- Les patterns de communication inter-services
- L'intégration complète de l'observabilité
- La containerisation et le déploiement

Préparez-vous à construire un système robuste et scalable qui reflète les meilleures pratiques de l'industrie !

⏭️
