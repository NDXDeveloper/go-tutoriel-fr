🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17-1 : Architecture microservices

## Qu'est-ce qu'une architecture microservices ?

### Définition simple

Imaginez une grande entreprise. Au lieu d'avoir une seule personne qui fait tout (comptabilité, ventes, marketing, etc.), l'entreprise divise le travail en départements spécialisés. Chaque département :
- A sa propre responsabilité
- Communique avec les autres départements
- Peut fonctionner indépendamment
- Peut être géré par des équipes différentes

C'est exactement le principe des microservices ! Au lieu d'avoir une seule application monolithique qui fait tout, nous divisons l'application en plusieurs petits services indépendants.

### Comparaison : Monolithe vs Microservices

#### Application Monolithique
```
┌─────────────────────────────────────┐
│         Application E-commerce      │
│  ┌─────────────────────────────────┐│
│  │  Gestion des utilisateurs       ││
│  │  Gestion des produits           ││
│  │  Gestion des commandes          ││
│  │  Gestion des paiements          ││
│  │  Gestion des notifications      ││
│  └─────────────────────────────────┘│
│         Base de données unique       │
└─────────────────────────────────────┘
```

#### Architecture Microservices
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Service    │    │  Service    │    │  Service    │
│Utilisateurs │    │  Produits   │    │ Commandes   │
│     +       │    │     +       │    │     +       │
│    DB       │    │    DB       │    │    DB       │
└─────────────┘    └─────────────┘    └─────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                          │
                ┌─────────────┐
                │ API Gateway │
                │(Point entry)│
                └─────────────┘
```

## Avantages des microservices

### 1. Indépendance
- Chaque service peut être développé par une équipe différente
- Choix de technologies spécifiques pour chaque service
- Déploiement indépendant (pas besoin d'arrêter tout le système)

### 2. Scalabilité
- Vous pouvez faire grandir seulement les services qui en ont besoin
- Exemple : Si beaucoup de personnes consultent les produits, on peut ajouter plus d'instances du service produits

### 3. Résilience
- Si un service tombe, les autres continuent à fonctionner
- Isolation des pannes

### 4. Maintenabilité
- Code plus petit et plus facile à comprendre
- Tests plus ciblés
- Moins de risque de casser autre chose en modifiant un service

## Inconvénients à connaître

### 1. Complexité
- Plus de services = plus de choses à gérer
- Communication réseau entre services
- Monitoring plus complexe

### 2. Cohérence des données
- Difficile de maintenir la cohérence entre plusieurs bases de données
- Transactions distribuées complexes

### 3. Latence réseau
- Appels réseau plus lents que appels en mémoire
- Gestion des pannes réseau

## Principes fondamentaux

### 1. Single Responsibility Principle (SRP)
**Principe** : Chaque service doit avoir une seule responsabilité bien définie.

**Exemple** :
```go
// ✅ BON - Service spécialisé
type ProductService struct {
    repository ProductRepository
}

func (s *ProductService) GetProduct(id string) (*Product, error) {
    // Logique uniquement liée aux produits
}

// ❌ MAUVAIS - Service qui fait trop de choses
type EverythingService struct {
    userRepo    UserRepository
    productRepo ProductRepository
    orderRepo   OrderRepository
}
```

### 2. Database per Service
**Principe** : Chaque service a sa propre base de données.

```
Service Utilisateurs  →  DB Utilisateurs
Service Produits     →  DB Produits
Service Commandes    →  DB Commandes
```

**Pourquoi ?**
- Évite les dépendances entre services
- Chaque service peut choisir la technologie de base de données qui lui convient
- Pas de conflit sur les modifications de schéma

### 3. API First Design
**Principe** : Définir l'API avant d'implémenter le service.

**Exemple d'API REST** :
```
GET    /api/v1/products        # Lister les produits
GET    /api/v1/products/{id}   # Obtenir un produit
POST   /api/v1/products        # Créer un produit
PUT    /api/v1/products/{id}   # Modifier un produit
DELETE /api/v1/products/{id}   # Supprimer un produit
```

## Patterns de communication

### 1. Communication Synchrone (Request-Response)

**Cas d'usage** : Quand vous avez besoin d'une réponse immédiate.

```go
// Appel HTTP vers un autre service
func (s *OrderService) CreateOrder(order *Order) error {
    // Vérifier que le produit existe
    product, err := s.productClient.GetProduct(order.ProductID)
    if err != nil {
        return err
    }

    // Créer la commande
    return s.repository.Save(order)
}
```

### 2. Communication Asynchrone (Message Queue)

**Cas d'usage** : Quand vous pouvez traiter la demande plus tard.

```go
// Publier un message dans une queue
func (s *OrderService) ProcessOrder(order *Order) error {
    // Sauvegarder la commande
    err := s.repository.Save(order)
    if err != nil {
        return err
    }

    // Publier un événement (notification asynchrone)
    event := OrderCreatedEvent{
        OrderID:    order.ID,
        CustomerID: order.CustomerID,
        Amount:     order.Amount,
    }

    return s.eventPublisher.Publish("order.created", event)
}
```

## Architecture de notre projet e-commerce

### Vue d'ensemble
```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Point entry)  │
                    └─────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Service    │    │  Service    │    │  Service    │
│Utilisateurs │    │  Produits   │    │ Commandes   │
│             │    │             │    │             │
│ PostgreSQL  │    │ PostgreSQL  │    │ PostgreSQL  │
└─────────────┘    └─────────────┘    └─────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌─────────────┐
                    │   Service   │
                    │Notifications│
                    │             │
                    │    Redis    │
                    └─────────────┘
```

### Responsabilités de chaque service

#### API Gateway
**Rôle** : Point d'entrée unique pour tous les clients
**Responsabilités** :
- Routage des requêtes vers les bons services
- Authentification et autorisation
- Rate limiting
- Logging et monitoring centralisé

#### Service Utilisateurs
**Rôle** : Gestion des utilisateurs et authentification
**Responsabilités** :
- Inscription et connexion
- Gestion des profils
- Génération et validation des tokens JWT
- Gestion des rôles et permissions

#### Service Produits
**Rôle** : Gestion du catalogue produits
**Responsabilités** :
- CRUD des produits
- Gestion des stocks
- Recherche et filtrage
- Gestion des catégories

#### Service Commandes
**Rôle** : Traitement des commandes
**Responsabilités** :
- Création et suivi des commandes
- Calcul des prix et taxes
- Gestion des statuts de commande
- Historique des commandes

#### Service Notifications
**Rôle** : Envoi de notifications
**Responsabilités** :
- Notifications email
- Notifications push
- Gestion des templates
- Historique des notifications

## Exemple pratique : Flux de création d'une commande

Voyons comment les services collaborent pour créer une commande :

```
1. Client → API Gateway : POST /orders
2. API Gateway → Service Utilisateurs : Vérifier le token
3. Service Utilisateurs → API Gateway : Token valide
4. API Gateway → Service Commandes : Créer la commande
5. Service Commandes → Service Produits : Vérifier le stock
6. Service Produits → Service Commandes : Stock disponible
7. Service Commandes → DB : Sauvegarder la commande
8. Service Commandes → Message Queue : Publier "order.created"
9. Service Notifications : Recevoir l'événement
10. Service Notifications → Client : Envoyer email de confirmation
```

## Bonnes pratiques

### 1. Taille des services
- **Règle empirique** : Une équipe de 2-8 personnes peut maintenir un service
- **Complexité** : Un service ne doit pas dépasser 1000-2000 lignes de code
- **Domaine** : Un service par domaine métier

### 2. Versioning des APIs
```go
// Version dans l'URL
GET /api/v1/products
GET /api/v2/products

// Version dans les headers
GET /api/products
Accept: application/vnd.myapi.v1+json
```

### 3. Gestion des erreurs
```go
// Retourner des erreurs standardisées
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

// Exemple d'utilisation
func (h *ProductHandler) GetProduct(c *gin.Context) {
    id := c.Param("id")

    product, err := h.service.GetProduct(id)
    if err != nil {
        if errors.Is(err, ErrProductNotFound) {
            c.JSON(http.StatusNotFound, APIError{
                Code:    "PRODUCT_NOT_FOUND",
                Message: "Product not found",
                Details: fmt.Sprintf("Product with ID %s does not exist", id),
            })
            return
        }

        c.JSON(http.StatusInternalServerError, APIError{
            Code:    "INTERNAL_ERROR",
            Message: "Internal server error",
        })
        return
    }

    c.JSON(http.StatusOK, product)
}
```

## Outils et technologies

### Communication
- **HTTP/REST** : Pour les appels synchrones
- **gRPC** : Pour les appels synchrones haute performance
- **Message Queues** : Redis, RabbitMQ, Apache Kafka

### Monitoring
- **Logs** : logrus, zap
- **Métriques** : Prometheus
- **Tracing** : Jaeger, Zipkin

### Orchestration
- **Développement** : Docker Compose
- **Production** : Kubernetes, Docker Swarm

## Récapitulatif

Les microservices sont une approche architecturale qui :
- Divise l'application en services indépendants
- Améliore la scalabilité et la maintenabilité
- Introduit de la complexité qu'il faut gérer
- Nécessite de bonnes pratiques et outils

Dans les prochaines sections, nous implémenterons concrètement cette architecture en créant chaque service étape par étape.

## Exercices pratiques

1. **Réflexion** : Identifiez 3 services que vous pourriez extraire d'une application de blog
2. **Design** : Dessinez l'architecture d'un système de réservation d'hôtel avec des microservices
3. **API** : Définissez les endpoints REST pour un service de gestion de panier d'achat

## Prochaines étapes

Dans la section suivante, nous aborderons la **communication inter-services** et verrons comment faire collaborer efficacement nos microservices.

⏭️

# Solutions des exercices pratiques - Architecture microservices

## Exercice 1 : Réflexion - Services d'une application de blog

### Services identifiés

#### 1. Service Articles/Contenu
**Responsabilité** : Gestion des articles et du contenu
**Fonctionnalités** :
- Créer, modifier, supprimer des articles
- Gestion des brouillons et publications
- Recherche et filtrage d'articles
- Gestion des catégories et tags
- Gestion des médias (images, vidéos)

#### 2. Service Utilisateurs/Authentification
**Responsabilité** : Gestion des utilisateurs et authentification
**Fonctionnalités** :
- Inscription et connexion
- Gestion des profils utilisateurs
- Gestion des rôles (lecteur, auteur, éditeur, admin)
- Authentification JWT
- Réinitialisation de mot de passe

#### 3. Service Commentaires/Interactions
**Responsabilité** : Gestion des interactions sociales
**Fonctionnalités** :
- Système de commentaires
- Modération des commentaires
- Système de likes/votes
- Notifications d'interactions
- Gestion du spam

### Services additionnels possibles

#### 4. Service Notifications
**Responsabilité** : Envoi de notifications
**Fonctionnalités** :
- Notifications email
- Notifications push
- Notifications d'abonnements
- Templates de notifications

#### 5. Service Analytics
**Responsabilité** : Statistiques et analytics
**Fonctionnalités** :
- Suivi des vues d'articles
- Statistiques d'engagement
- Rapports pour les auteurs
- Métriques de performance

---

## Exercice 2 : Design - Architecture système de réservation d'hôtel

### Architecture générale

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Load Balancer)│
                    └─────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Service    │    │  Service    │    │  Service    │
│Utilisateurs │    │   Hôtels    │    │Réservations │
│             │    │             │    │             │
│ PostgreSQL  │    │ PostgreSQL  │    │ PostgreSQL  │
└─────────────┘    └─────────────┘    └─────────────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Service    │    │  Service    │    │  Service    │
│ Paiements   │    │   Search    │    │Notifications│
│             │    │             │    │             │
│ PostgreSQL  │    │ Elasticsearch│    │    Redis    │
└─────────────┘    └─────────────┘    └─────────────┘
```

### Détail des services

#### 1. Service Utilisateurs
**Base de données** : PostgreSQL
**Responsabilités** :
- Gestion des comptes clients
- Authentification et autorisation
- Profils utilisateurs
- Historique des réservations

#### 2. Service Hôtels
**Base de données** : PostgreSQL
**Responsabilités** :
- Catalogue des hôtels
- Gestion des chambres et tarifs
- Disponibilités
- Équipements et services
- Photos et descriptions

#### 3. Service Réservations
**Base de données** : PostgreSQL
**Responsabilités** :
- Création et gestion des réservations
- Vérification des disponibilités
- Calcul des prix
- Gestion des statuts de réservation
- Annulations et modifications

#### 4. Service Paiements
**Base de données** : PostgreSQL
**Responsabilités** :
- Traitement des paiements
- Gestion des remboursements
- Intégration avec providers (Stripe, PayPal)
- Facturation
- Gestion des devises

#### 5. Service Search/Recherche
**Base de données** : Elasticsearch
**Responsabilités** :
- Recherche d'hôtels par critères
- Filtrage avancé
- Suggestions et recommandations
- Indexation des données hôtels
- Géolocalisation

#### 6. Service Notifications
**Base de données** : Redis
**Responsabilités** :
- Confirmations de réservation
- Rappels de voyage
- Notifications push
- Emails transactionnels
- SMS

### Flux de réservation

```
1. Client → API Gateway : GET /search?city=Paris&dates=...
2. API Gateway → Service Search : Rechercher hôtels
3. Service Search → Service Hôtels : Obtenir détails
4. Client → API Gateway : POST /reservations
5. API Gateway → Service Utilisateurs : Vérifier authentification
6. API Gateway → Service Réservations : Créer réservation
7. Service Réservations → Service Hôtels : Vérifier disponibilité
8. Service Réservations → Service Paiements : Traiter paiement
9. Service Paiements → Service Réservations : Confirmation paiement
10. Service Réservations → Queue : Publier "booking.confirmed"
11. Service Notifications : Envoyer confirmation client
```

---

## Exercice 3 : API - Endpoints REST pour service de panier d'achat

### Endpoints principaux

#### 1. Gestion du panier

```http
# Obtenir le panier d'un utilisateur
GET /api/v1/carts/{userId}
Response: 200 OK
{
  "id": "cart-123",
  "userId": "user-456",
  "items": [
    {
      "id": "item-1",
      "productId": "product-789",
      "productName": "Laptop Dell",
      "quantity": 1,
      "unitPrice": 999.99,
      "totalPrice": 999.99
    }
  ],
  "totalAmount": 999.99,
  "currency": "EUR",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:35:00Z"
}

# Créer un nouveau panier (optionnel si auto-créé)
POST /api/v1/carts
Request Body:
{
  "userId": "user-456"
}
Response: 201 Created
{
  "id": "cart-123",
  "userId": "user-456",
  "items": [],
  "totalAmount": 0,
  "currency": "EUR",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

#### 2. Gestion des articles

```http
# Ajouter un article au panier
POST /api/v1/carts/{cartId}/items
Request Body:
{
  "productId": "product-789",
  "quantity": 2
}
Response: 201 Created
{
  "id": "item-1",
  "productId": "product-789",
  "productName": "Laptop Dell",
  "quantity": 2,
  "unitPrice": 999.99,
  "totalPrice": 1999.98
}

# Modifier la quantité d'un article
PUT /api/v1/carts/{cartId}/items/{itemId}
Request Body:
{
  "quantity": 3
}
Response: 200 OK
{
  "id": "item-1",
  "productId": "product-789",
  "productName": "Laptop Dell",
  "quantity": 3,
  "unitPrice": 999.99,
  "totalPrice": 2999.97
}

# Supprimer un article du panier
DELETE /api/v1/carts/{cartId}/items/{itemId}
Response: 204 No Content
```

#### 3. Opérations sur le panier

```http
# Vider le panier
DELETE /api/v1/carts/{cartId}/items
Response: 204 No Content

# Appliquer un code promo
POST /api/v1/carts/{cartId}/promocode
Request Body:
{
  "code": "SUMMER2024"
}
Response: 200 OK
{
  "discount": {
    "code": "SUMMER2024",
    "type": "percentage",
    "value": 10,
    "amount": 99.99
  },
  "totalAmount": 899.99
}

# Calculer les frais de livraison
POST /api/v1/carts/{cartId}/shipping
Request Body:
{
  "address": {
    "country": "FR",
    "city": "Paris",
    "zipCode": "75001"
  }
}
Response: 200 OK
{
  "shippingOptions": [
    {
      "id": "standard",
      "name": "Livraison standard",
      "price": 5.99,
      "deliveryTime": "3-5 jours"
    },
    {
      "id": "express",
      "name": "Livraison express",
      "price": 12.99,
      "deliveryTime": "1-2 jours"
    }
  ]
}
```

### Codes de réponse et gestion d'erreurs

```http
# Succès
200 OK - Opération réussie
201 Created - Ressource créée
204 No Content - Suppression réussie

# Erreurs client
400 Bad Request - Données invalides
{
  "error": {
    "code": "INVALID_QUANTITY",
    "message": "Quantity must be greater than 0",
    "field": "quantity"
  }
}

404 Not Found - Ressource non trouvée
{
  "error": {
    "code": "CART_NOT_FOUND",
    "message": "Cart not found",
    "cartId": "cart-123"
  }
}

409 Conflict - Conflit (stock insuffisant)
{
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Not enough stock available",
    "availableStock": 2,
    "requestedQuantity": 5
  }
}

# Erreurs serveur
500 Internal Server Error - Erreur serveur
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An internal error occurred"
  }
}
```

### Implémentation Go - Exemple de handler

```go
package handlers

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

type CartHandler struct {
    cartService CartService
}

type AddItemRequest struct {
    ProductID string `json:"productId" binding:"required"`
    Quantity  int    `json:"quantity" binding:"required,min=1"`
}

func (h *CartHandler) AddItem(c *gin.Context) {
    cartID := c.Param("cartId")

    var req AddItemRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error": map[string]interface{}{
                "code":    "INVALID_REQUEST",
                "message": err.Error(),
            },
        })
        return
    }

    item, err := h.cartService.AddItem(cartID, req.ProductID, req.Quantity)
    if err != nil {
        switch err {
        case ErrCartNotFound:
            c.JSON(http.StatusNotFound, gin.H{
                "error": map[string]interface{}{
                    "code":    "CART_NOT_FOUND",
                    "message": "Cart not found",
                    "cartId":  cartID,
                },
            })
        case ErrInsufficientStock:
            c.JSON(http.StatusConflict, gin.H{
                "error": map[string]interface{}{
                    "code":    "INSUFFICIENT_STOCK",
                    "message": "Not enough stock available",
                },
            })
        default:
            c.JSON(http.StatusInternalServerError, gin.H{
                "error": map[string]interface{}{
                    "code":    "INTERNAL_ERROR",
                    "message": "An internal error occurred",
                },
            })
        }
        return
    }

    c.JSON(http.StatusCreated, item)
}

func (h *CartHandler) GetCart(c *gin.Context) {
    userID := c.Param("userId")

    cart, err := h.cartService.GetCartByUserID(userID)
    if err != nil {
        if err == ErrCartNotFound {
            c.JSON(http.StatusNotFound, gin.H{
                "error": map[string]interface{}{
                    "code":    "CART_NOT_FOUND",
                    "message": "Cart not found for user",
                    "userId":  userID,
                },
            })
            return
        }

        c.JSON(http.StatusInternalServerError, gin.H{
            "error": map[string]interface{}{
                "code":    "INTERNAL_ERROR",
                "message": "An internal error occurred",
            },
        })
        return
    }

    c.JSON(http.StatusOK, cart)
}
```

### Sécurité et bonnes pratiques

#### 1. Authentification
```go
// Middleware d'authentification
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")

        userID, err := validateToken(token)
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{
                "error": map[string]interface{}{
                    "code":    "UNAUTHORIZED",
                    "message": "Invalid or missing token",
                },
            })
            c.Abort()
            return
        }

        c.Set("userID", userID)
        c.Next()
    }
}
```

#### 2. Validation des données
```go
type CartItem struct {
    ID          string  `json:"id"`
    ProductID   string  `json:"productId" binding:"required"`
    Quantity    int     `json:"quantity" binding:"required,min=1,max=100"`
    UnitPrice   float64 `json:"unitPrice" binding:"required,min=0"`
    TotalPrice  float64 `json:"totalPrice"`
}
```

#### 3. Rate limiting
```go
// Limiter les requêtes par utilisateur
func RateLimitMiddleware() gin.HandlerFunc {
    // Implémentation avec Redis ou en mémoire
    return gin.HandlerFunc(func(c *gin.Context) {
        // Vérifier les limites
        // Si dépassé, retourner 429 Too Many Requests
    })
}
```

## Résumé des solutions

Ces exercices illustrent :
- **Décomposition fonctionnelle** : Séparer les responsabilités métier
- **Design d'architecture** : Penser aux interactions entre services
- **Conception d'API** : Respecter les conventions REST et HTTP
- **Gestion d'erreurs** : Codes de réponse appropriés et messages clairs
- **Sécurité** : Authentification, validation, rate limiting

Ces solutions servent de base pour implémenter des microservices robustes et maintenables.

⏭️
