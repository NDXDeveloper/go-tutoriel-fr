🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17-3 : Observabilité (logs, métriques)

## Introduction à l'observabilité

### Qu'est-ce que l'observabilité ?

Imaginez que vous conduisez une voiture. Pour bien conduire, vous avez besoin :
- **Du tableau de bord** (vitesse, essence, température) → **Métriques**
- **Du rétroviseur** (ce qui s'est passé) → **Logs**
- **Du GPS avec historique** (où vous êtes, d'où vous venez) → **Traces**

L'observabilité dans les microservices, c'est pareil ! C'est votre capacité à comprendre ce qui se passe dans votre système en observant ses sorties.

### Pourquoi l'observabilité est cruciale ?

Dans une architecture monolithique, déboguer est comme chercher un problème dans une seule maison. Avec les microservices, c'est comme chercher un problème dans tout un quartier !

**Sans observabilité** :
```
Client : "Le site ne marche pas !"
Vous : "Euh... Je ne sais pas quel service a un problème..."
```

**Avec observabilité** :
```
Client : "Le site ne marche pas !"
Vous : "Je vois que le Service Commandes a un taux d'erreur de 25%
       depuis 10 minutes à cause d'un timeout avec la base de données."
```

## Les trois piliers de l'observabilité

### 1. Logs (Journaux)
**Ce que c'est** : Messages textuels décrivant ce qui s'est passé
**Analogie** : Le journal de bord d'un capitaine de navire

### 2. Métriques
**Ce que c'est** : Données numériques agrégées dans le temps
**Analogie** : Les indicateurs du tableau de bord de votre voiture

### 3. Traces distribuées
**Ce que c'est** : Suivi d'une requête à travers tous les services
**Analogie** : GPS avec historique complet du trajet

## Implémentation des Logs

### Configuration de base avec Logrus

```go
package logger

import (
    "os"
    "github.com/sirupsen/logrus"
)

// Configuration du logger global
func InitLogger() *logrus.Logger {
    log := logrus.New()

    // Format JSON pour une meilleure structure
    log.SetFormatter(&logrus.JSONFormatter{
        TimestampFormat: "2006-01-02 15:04:05",
        FieldMap: logrus.FieldMap{
            logrus.FieldKeyTime:  "timestamp",
            logrus.FieldKeyLevel: "level",
            logrus.FieldKeyMsg:   "message",
        },
    })

    // Niveau de log basé sur l'environnement
    env := os.Getenv("ENVIRONMENT")
    switch env {
    case "production":
        log.SetLevel(logrus.InfoLevel)
    case "development":
        log.SetLevel(logrus.DebugLevel)
    default:
        log.SetLevel(logrus.InfoLevel)
    }

    return log
}

// Logger avec contexte
type ContextLogger struct {
    logger   *logrus.Logger
    service  string
    version  string
}

func NewContextLogger(service, version string) *ContextLogger {
    return &ContextLogger{
        logger:  InitLogger(),
        service: service,
        version: version,
    }
}

// Méthodes de logging avec contexte automatique
func (cl *ContextLogger) Info(message string, fields map[string]interface{}) {
    entry := cl.logger.WithFields(logrus.Fields{
        "service": cl.service,
        "version": cl.version,
    })

    if fields != nil {
        entry = entry.WithFields(fields)
    }

    entry.Info(message)
}

func (cl *ContextLogger) Error(message string, err error, fields map[string]interface{}) {
    entry := cl.logger.WithFields(logrus.Fields{
        "service": cl.service,
        "version": cl.version,
        "error":   err.Error(),
    })

    if fields != nil {
        entry = entry.WithFields(fields)
    }

    entry.Error(message)
}

func (cl *ContextLogger) Debug(message string, fields map[string]interface{}) {
    entry := cl.logger.WithFields(logrus.Fields{
        "service": cl.service,
        "version": cl.version,
    })

    if fields != nil {
        entry = entry.WithFields(fields)
    }

    entry.Debug(message)
}
```

### Logging structuré dans les handlers HTTP

```go
package handlers

import (
    "time"
    "github.com/gin-gonic/gin"
    "myapp/logger"
)

type OrderHandler struct {
    service OrderService
    logger  *logger.ContextLogger
}

func NewOrderHandler(service OrderService) *OrderHandler {
    return &OrderHandler{
        service: service,
        logger:  logger.NewContextLogger("order-service", "1.0.0"),
    }
}

func (h *OrderHandler) CreateOrder(c *gin.Context) {
    startTime := time.Now()
    userID := c.GetString("userID") // Extrait du middleware auth

    // Log du début de la requête
    h.logger.Info("Creating order", map[string]interface{}{
        "user_id":    userID,
        "request_id": c.GetString("requestID"),
        "endpoint":   c.FullPath(),
        "method":     c.Request.Method,
    })

    var orderReq CreateOrderRequest
    if err := c.ShouldBindJSON(&orderReq); err != nil {
        // Log d'erreur de validation
        h.logger.Error("Invalid request body", err, map[string]interface{}{
            "user_id":    userID,
            "request_id": c.GetString("requestID"),
            "error_type": "validation",
        })

        c.JSON(400, gin.H{"error": "Invalid request body"})
        return
    }

    // Créer la commande
    order, err := h.service.CreateOrder(userID, &orderReq)
    if err != nil {
        // Log d'erreur métier
        h.logger.Error("Failed to create order", err, map[string]interface{}{
            "user_id":    userID,
            "request_id": c.GetString("requestID"),
            "order_data": orderReq,
            "error_type": "business_logic",
        })

        c.JSON(500, gin.H{"error": "Failed to create order"})
        return
    }

    duration := time.Since(startTime)

    // Log de succès avec métriques
    h.logger.Info("Order created successfully", map[string]interface{}{
        "user_id":     userID,
        "request_id":  c.GetString("requestID"),
        "order_id":    order.ID,
        "order_total": order.TotalAmount,
        "duration_ms": duration.Milliseconds(),
        "status":      "success",
    })

    c.JSON(201, order)
}
```

### Middleware de logging

```go
package middleware

import (
    "time"
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
    "myapp/logger"
)

func LoggingMiddleware(contextLogger *logger.ContextLogger) gin.HandlerFunc {
    return func(c *gin.Context) {
        startTime := time.Now()

        // Générer un ID de requête unique
        requestID := uuid.New().String()
        c.Set("requestID", requestID)

        // Log de début de requête
        contextLogger.Info("Request started", map[string]interface{}{
            "request_id": requestID,
            "method":     c.Request.Method,
            "path":       c.Request.URL.Path,
            "user_agent": c.Request.UserAgent(),
            "ip":         c.ClientIP(),
        })

        // Traiter la requête
        c.Next()

        duration := time.Since(startTime)
        statusCode := c.Writer.Status()

        // Déterminer le niveau de log basé sur le status
        logLevel := "info"
        if statusCode >= 400 && statusCode < 500 {
            logLevel = "warn"
        } else if statusCode >= 500 {
            logLevel = "error"
        }

        logData := map[string]interface{}{
            "request_id":  requestID,
            "method":      c.Request.Method,
            "path":        c.Request.URL.Path,
            "status_code": statusCode,
            "duration_ms": duration.Milliseconds(),
            "response_size": c.Writer.Size(),
        }

        message := "Request completed"
        switch logLevel {
        case "error":
            contextLogger.Error(message, nil, logData)
        case "warn":
            contextLogger.Info(message + " with warning", logData)
        default:
            contextLogger.Info(message, logData)
        }
    }
}
```

## Implémentation des Métriques avec Prometheus

### Configuration de base

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "time"
)

// Métriques globales de l'application
var (
    // Counter : nombre total de requêtes HTTP
    HTTPRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status_code", "service"},
    )

    // Histogram : durée des requêtes HTTP
    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "Duration of HTTP requests in seconds",
            Buckets: []float64{0.001, 0.01, 0.1, 0.5, 1.0, 2.5, 5.0, 10.0},
        },
        []string{"method", "endpoint", "service"},
    )

    // Gauge : nombre de connexions actives
    ActiveConnections = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
        []string{"service"},
    )

    // Counter : erreurs business
    BusinessErrorsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "business_errors_total",
            Help: "Total number of business errors",
        },
        []string{"service", "error_type", "operation"},
    )

    // Gauge : santé de la base de données
    DatabaseHealth = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "database_health",
            Help: "Database health status (1 = healthy, 0 = unhealthy)",
        },
        []string{"service", "database_name"},
    )
)

// Service de métriques
type MetricsService struct {
    serviceName string
}

func NewMetricsService(serviceName string) *MetricsService {
    return &MetricsService{
        serviceName: serviceName,
    }
}

// Enregistrer une requête HTTP
func (m *MetricsService) RecordHTTPRequest(method, endpoint, statusCode string, duration time.Duration) {
    HTTPRequestsTotal.WithLabelValues(method, endpoint, statusCode, m.serviceName).Inc()
    HTTPRequestDuration.WithLabelValues(method, endpoint, m.serviceName).Observe(duration.Seconds())
}

// Enregistrer une erreur business
func (m *MetricsService) RecordBusinessError(errorType, operation string) {
    BusinessErrorsTotal.WithLabelValues(m.serviceName, errorType, operation).Inc()
}

// Mettre à jour les connexions actives
func (m *MetricsService) UpdateActiveConnections(count int) {
    ActiveConnections.WithLabelValues(m.serviceName).Set(float64(count))
}

// Mettre à jour la santé de la base de données
func (m *MetricsService) UpdateDatabaseHealth(dbName string, healthy bool) {
    value := 0.0
    if healthy {
        value = 1.0
    }
    DatabaseHealth.WithLabelValues(m.serviceName, dbName).Set(value)
}
```

### Middleware de métriques

```go
package middleware

import (
    "strconv"
    "time"
    "github.com/gin-gonic/gin"
    "myapp/metrics"
)

func MetricsMiddleware(metricsService *metrics.MetricsService) gin.HandlerFunc {
    return func(c *gin.Context) {
        startTime := time.Now()

        // Traiter la requête
        c.Next()

        // Calculer la durée
        duration := time.Since(startTime)

        // Enregistrer les métriques
        method := c.Request.Method
        endpoint := c.FullPath()
        statusCode := strconv.Itoa(c.Writer.Status())

        metricsService.RecordHTTPRequest(method, endpoint, statusCode, duration)
    }
}
```

### Métriques métier personnalisées

```go
package service

import (
    "context"
    "time"
    "myapp/metrics"
    "myapp/logger"
)

type OrderService struct {
    repository      OrderRepository
    metricsService  *metrics.MetricsService
    logger          *logger.ContextLogger
}

// Métriques spécifiques au domaine métier
var (
    OrdersCreatedTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
        []string{"service", "payment_method"},
    )

    OrderValue = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "order_value_euros",
            Help: "Value of orders in euros",
            Buckets: []float64{10, 50, 100, 250, 500, 1000, 2500, 5000},
        },
        []string{"service"},
    )

    DatabaseOperationDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "database_operation_duration_seconds",
            Help: "Duration of database operations",
            Buckets: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0},
        },
        []string{"service", "operation", "table"},
    )
)

func (s *OrderService) CreateOrder(ctx context.Context, userID string, orderData *CreateOrderRequest) (*Order, error) {
    startTime := time.Now()

    s.logger.Info("Creating order", map[string]interface{}{
        "user_id": userID,
        "amount":  orderData.Amount,
        "items_count": len(orderData.Items),
    })

    // Créer la commande
    order := &Order{
        ID:            generateOrderID(),
        UserID:        userID,
        Items:         orderData.Items,
        TotalAmount:   orderData.Amount,
        PaymentMethod: orderData.PaymentMethod,
        Status:        "pending",
        CreatedAt:     time.Now(),
    }

    // Mesurer le temps de la base de données
    dbStartTime := time.Now()
    err := s.repository.Save(ctx, order)
    dbDuration := time.Since(dbStartTime)

    // Enregistrer les métriques de base de données
    DatabaseOperationDuration.WithLabelValues("order-service", "insert", "orders").Observe(dbDuration.Seconds())

    if err != nil {
        s.metricsService.RecordBusinessError("database_error", "create_order")
        s.logger.Error("Failed to save order", err, map[string]interface{}{
            "user_id": userID,
            "order_id": order.ID,
        })
        return nil, err
    }

    // Enregistrer les métriques métier
    OrdersCreatedTotal.WithLabelValues("order-service", order.PaymentMethod).Inc()
    OrderValue.WithLabelValues("order-service").Observe(order.TotalAmount)

    duration := time.Since(startTime)
    s.logger.Info("Order created successfully", map[string]interface{}{
        "user_id":     userID,
        "order_id":    order.ID,
        "duration_ms": duration.Milliseconds(),
        "amount":      order.TotalAmount,
    })

    return order, nil
}
```

### Health checks avec métriques

```go
package health

import (
    "context"
    "database/sql"
    "time"
    "myapp/metrics"
)

type HealthChecker struct {
    db             *sql.DB
    metricsService *metrics.MetricsService
}

func NewHealthChecker(db *sql.DB, metricsService *metrics.MetricsService) *HealthChecker {
    return &HealthChecker{
        db:             db,
        metricsService: metricsService,
    }
}

type HealthStatus struct {
    Status    string            `json:"status"`
    Timestamp time.Time         `json:"timestamp"`
    Checks    map[string]Check  `json:"checks"`
}

type Check struct {
    Status   string        `json:"status"`
    Duration time.Duration `json:"duration"`
    Message  string        `json:"message,omitempty"`
}

func (hc *HealthChecker) CheckHealth(ctx context.Context) *HealthStatus {
    status := &HealthStatus{
        Status:    "healthy",
        Timestamp: time.Now(),
        Checks:    make(map[string]Check),
    }

    // Vérifier la base de données
    dbCheck := hc.checkDatabase(ctx)
    status.Checks["database"] = dbCheck

    // Mettre à jour les métriques
    hc.metricsService.UpdateDatabaseHealth("postgres", dbCheck.Status == "healthy")

    // Déterminer le statut global
    if dbCheck.Status != "healthy" {
        status.Status = "unhealthy"
    }

    return status
}

func (hc *HealthChecker) checkDatabase(ctx context.Context) Check {
    startTime := time.Now()

    err := hc.db.PingContext(ctx)
    duration := time.Since(startTime)

    if err != nil {
        return Check{
            Status:   "unhealthy",
            Duration: duration,
            Message:  err.Error(),
        }
    }

    return Check{
        Status:   "healthy",
        Duration: duration,
    }
}
```

## Endpoint de métriques

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "myapp/health"
    "myapp/metrics"
)

func setupMetricsRoutes(r *gin.Engine, healthChecker *health.HealthChecker) {
    // Endpoint Prometheus (format standard)
    r.GET("/metrics", gin.WrapH(promhttp.Handler()))

    // Endpoint de santé (format JSON)
    r.GET("/health", func(c *gin.Context) {
        ctx := c.Request.Context()
        health := healthChecker.CheckHealth(ctx)

        statusCode := http.StatusOK
        if health.Status != "healthy" {
            statusCode = http.StatusServiceUnavailable
        }

        c.JSON(statusCode, health)
    })

    // Endpoint de santé simple (pour load balancer)
    r.GET("/health/live", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "alive"})
    })

    // Endpoint de readiness (prêt à recevoir du trafic)
    r.GET("/health/ready", func(c *gin.Context) {
        ctx := c.Request.Context()
        health := healthChecker.CheckHealth(ctx)

        if health.Status == "healthy" {
            c.JSON(http.StatusOK, gin.H{"status": "ready"})
        } else {
            c.JSON(http.StatusServiceUnavailable, gin.H{"status": "not ready"})
        }
    })
}
```

## Configuration Docker et Prometheus

### Dockerfile optimisé pour l'observabilité

```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o main ./cmd/server

FROM alpine:latest

# Installer ca-certificates pour HTTPS
RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/main .

# Exposer les ports
EXPOSE 8080 8090

# Variables d'environnement pour l'observabilité
ENV GIN_MODE=release
ENV LOG_LEVEL=info
ENV METRICS_PORT=8090

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1

CMD ["./main"]
```

### Configuration Prometheus (prometheus.yml)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Scraper pour nos microservices
  - job_name: 'microservices'
    static_configs:
      - targets:
          - 'user-service:8090'
          - 'order-service:8090'
          - 'product-service:8090'
          - 'cart-service:8090'
    scrape_interval: 10s
    metrics_path: /metrics

  # Scraper pour Prometheus lui-même
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

# Règles d'alerte
rule_files:
  - "alert_rules.yml"

# Configuration d'Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093'
```

### Règles d'alerte (alert_rules.yml)

```yaml
groups:
  - name: microservices
    rules:
      # Taux d'erreur élevé
      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Service {{ $labels.service }} has error rate > 10% for 2 minutes"

      # Latence élevée
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "Service {{ $labels.service }} has 95th percentile latency > 1s"

      # Service down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "Service {{ $labels.service }} has been down for more than 1 minute"

      # Base de données inaccessible
      - alert: DatabaseUnhealthy
        expr: database_health == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Database is unhealthy"
          description: "Database {{ $labels.database_name }} in service {{ $labels.service }} is unhealthy"
```

### Docker Compose avec monitoring

```yaml
version: '3.8'

services:
  # Nos microservices
  user-service:
    build: ./user-service
    ports:
      - "8081:8080"
      - "8091:8090"  # Port métriques
    environment:
      - SERVICE_NAME=user-service
      - METRICS_PORT=8090
    networks:
      - microservices

  order-service:
    build: ./order-service
    ports:
      - "8082:8080"
      - "8092:8090"
    environment:
      - SERVICE_NAME=order-service
      - METRICS_PORT=8090
    networks:
      - microservices

  # Monitoring stack
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/alert_rules.yml:/etc/prometheus/alert_rules.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    networks:
      - microservices

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - microservices

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    networks:
      - microservices

networks:
  microservices:
    driver: bridge

volumes:
  grafana-storage:
```

## Dashboards Grafana

### Configuration des datasources Grafana

```yaml
# monitoring/grafana/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### Dashboard JSON pour Grafana

```json
{
  "dashboard": {
    "id": null,
    "title": "Microservices Overview",
    "tags": ["microservices", "overview"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "HTTP Requests Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{service}} - {{method}} {{endpoint}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec"
          }
        ]
      },
      {
        "id": 2,
        "title": "HTTP Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"4..|5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "{{service}} error rate"
          }
        ],
        "yAxes": [
          {
            "label": "Error Rate",
            "max": 1,
            "min": 0
          }
        ]
      },
      {
        "id": 3,
        "title": "Response Time 95th Percentile",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{service}} 95th percentile"
          }
        ],
        "yAxes": [
          {
            "label": "Seconds"
          }
        ]
      },
      {
        "id": 4,
        "title": "Active Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "active_connections",
            "legendFormat": "{{service}} active connections"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

## Alerting avec Alertmanager

### Configuration Alertmanager

```yaml
# monitoring/alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@yourcompany.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    email_configs:
      - to: 'admin@yourcompany.com'
        subject: 'Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Service: {{ .Labels.service }}
          Severity: {{ .Labels.severity }}
          {{ end }}

    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Service:* {{ .Labels.service }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'service']
```

## Bonnes pratiques de l'observabilité

### 1. Structure des logs

```go
// ✅ BON - Log structuré
logger.Info("Order created", map[string]interface{}{
    "order_id":       "order-123",
    "user_id":        "user-456",
    "amount":         99.99,
    "payment_method": "card",
    "items_count":    3,
    "processing_time_ms": 150,
})

// ❌ MAUVAIS - Log non structuré
logger.Info("Order order-123 created for user user-456 with amount 99.99 using card")
```

### 2. Niveaux de logs appropriés

```go
// DEBUG - Informations de développement
logger.Debug("Cache hit", map[string]interface{}{
    "cache_key": "user:123",
    "hit_ratio": 0.85,
})

// INFO - Événements normaux important pour l'audit
logger.Info("User logged in", map[string]interface{}{
    "user_id": "user-123",
    "ip":      "192.168.1.100",
    "method":  "oauth",
})

// WARN - Situations anormales mais pas d'erreur
logger.Warn("High memory usage", map[string]interface{}{
    "memory_usage_percent": 85,
    "threshold":           80,
})

// ERROR - Erreurs qui affectent l'opération
logger.Error("Failed to process payment", err, map[string]interface{}{
    "order_id":       "order-123",
    "payment_method": "card",
    "error_code":     "insufficient_funds",
})

// FATAL - Erreurs qui empêchent le service de fonctionner
logger.Fatal("Cannot connect to database", err, map[string]interface{}{
    "database_url": "postgres://...",
    "retry_count":  5,
})
```

### 3. Corrélation des logs avec Request ID

```go
package middleware

import (
    "context"
    "github.com/gin-gonic/gin"
    "github.com/google/uuid"
)

func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }

        // Ajouter au contexte
        ctx := context.WithValue(c.Request.Context(), "request_id", requestID)
        c.Request = c.Request.WithContext(ctx)

        // Ajouter au header de réponse
        c.Header("X-Request-ID", requestID)

        // Rendre disponible pour les handlers
        c.Set("request_id", requestID)

        c.Next()
    }
}

// Logger qui utilise automatiquement le Request ID
func LogWithContext(ctx context.Context, logger *logrus.Logger, level string, message string, fields map[string]interface{}) {
    if fields == nil {
        fields = make(map[string]interface{})
    }

    // Ajouter automatiquement le request ID
    if requestID := ctx.Value("request_id"); requestID != nil {
        fields["request_id"] = requestID
    }

    entry := logger.WithFields(fields)

    switch level {
    case "debug":
        entry.Debug(message)
    case "info":
        entry.Info(message)
    case "warn":
        entry.Warn(message)
    case "error":
        entry.Error(message)
    }
}
```

### 4. Métriques Golden Signals

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

// Les 4 Golden Signals du monitoring SRE
var (
    // 1. LATENCY - Combien de temps pour répondre
    RequestLatency = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "request_latency_seconds",
            Help: "Request latency in seconds",
            Buckets: []float64{0.001, 0.01, 0.1, 0.5, 1.0, 2.5, 5.0, 10.0},
        },
        []string{"service", "method", "endpoint"},
    )

    // 2. TRAFFIC - Combien de demandes
    RequestRate = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "requests_total",
            Help: "Total number of requests",
        },
        []string{"service", "method", "endpoint", "status"},
    )

    // 3. ERRORS - Taux d'erreur
    ErrorRate = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "errors_total",
            Help: "Total number of errors",
        },
        []string{"service", "error_type", "operation"},
    )

    // 4. SATURATION - Utilisation des ressources
    ResourceUtilization = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "resource_utilization",
            Help: "Resource utilization percentage",
        },
        []string{"service", "resource_type"},
    )
)

// Service pour collecter les métriques système
type SystemMetrics struct {
    serviceName string
}

func NewSystemMetrics(serviceName string) *SystemMetrics {
    return &SystemMetrics{serviceName: serviceName}
}

func (sm *SystemMetrics) CollectSystemMetrics() {
    // CPU utilization
    cpuUsage := getCPUUsage() // Fonction à implémenter
    ResourceUtilization.WithLabelValues(sm.serviceName, "cpu").Set(cpuUsage)

    // Memory utilization
    memUsage := getMemoryUsage() // Fonction à implémenter
    ResourceUtilization.WithLabelValues(sm.serviceName, "memory").Set(memUsage)

    // Disk utilization
    diskUsage := getDiskUsage() // Fonction à implémenter
    ResourceUtilization.WithLabelValues(sm.serviceName, "disk").Set(diskUsage)
}
```

### 5. Sampling intelligent des logs

```go
package logger

import (
    "hash/fnv"
    "math/rand"
    "time"
)

type SamplingLogger struct {
    logger     *logrus.Logger
    sampleRate float64 // 0.0 to 1.0
}

func NewSamplingLogger(logger *logrus.Logger, sampleRate float64) *SamplingLogger {
    return &SamplingLogger{
        logger:     logger,
        sampleRate: sampleRate,
    }
}

func (sl *SamplingLogger) ShouldLog(level string, key string) bool {
    // Toujours logger les erreurs
    if level == "error" || level == "fatal" {
        return true
    }

    // Sampling basé sur la clé pour la consistance
    hash := fnv.New32a()
    hash.Write([]byte(key))
    hashValue := float64(hash.Sum32()) / float64(^uint32(0))

    return hashValue < sl.sampleRate
}

func (sl *SamplingLogger) InfoSampled(key, message string, fields map[string]interface{}) {
    if sl.ShouldLog("info", key) {
        if fields == nil {
            fields = make(map[string]interface{})
        }
        fields["sampled"] = true
        sl.logger.WithFields(fields).Info(message)
    }
}

// Exemple d'utilisation pour des logs haute fréquence
func (h *ProductHandler) GetProduct(c *gin.Context) {
    productID := c.Param("id")

    // Log samplé pour éviter le spam sur les requêtes fréquentes
    h.samplingLogger.InfoSampled(
        productID, // Clé pour sampling consistant
        "Product requested",
        map[string]interface{}{
            "product_id": productID,
            "user_id":    c.GetString("user_id"),
        },
    )

    // ... logique métier
}
```

## Traces distribuées avec Jaeger

### Configuration de base

```go
package tracing

import (
    "context"
    "io"
    "github.com/opentracing/opentracing-go"
    "github.com/uber/jaeger-client-go"
    "github.com/uber/jaeger-client-go/config"
)

func InitTracing(serviceName string) (opentracing.Tracer, io.Closer, error) {
    cfg := config.Configuration{
        ServiceName: serviceName,
        Sampler: &config.SamplerConfig{
            Type:  jaeger.SamplerTypeConst,
            Param: 1, // Sample 100% des traces (ajuster en production)
        },
        Reporter: &config.ReporterConfig{
            LogSpans:            true,
            BufferFlushInterval: 1 * time.Second,
            LocalAgentHostPort:  "jaeger:6831", // Agent Jaeger
        },
    }

    tracer, closer, err := cfg.NewTracer()
    if err != nil {
        return nil, nil, err
    }

    opentracing.SetGlobalTracer(tracer)
    return tracer, closer, nil
}

// Middleware pour tracer les requêtes HTTP
func TracingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        span := opentracing.StartSpan(
            fmt.Sprintf("%s %s", c.Request.Method, c.FullPath()),
        )
        defer span.Finish()

        // Ajouter des tags
        span.SetTag("http.method", c.Request.Method)
        span.SetTag("http.url", c.Request.URL.String())
        span.SetTag("user.id", c.GetString("user_id"))

        // Injecter le span dans le contexte
        ctx := opentracing.ContextWithSpan(c.Request.Context(), span)
        c.Request = c.Request.WithContext(ctx)

        c.Next()

        // Ajouter le status code après traitement
        span.SetTag("http.status_code", c.Writer.Status())

        if c.Writer.Status() >= 400 {
            span.SetTag("error", true)
        }
    }
}
```

### Tracer les appels entre services

```go
package client

import (
    "context"
    "net/http"
    "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

type TracedHTTPClient struct {
    client *http.Client
}

func NewTracedHTTPClient() *TracedHTTPClient {
    return &TracedHTTPClient{
        client: &http.Client{Timeout: 10 * time.Second},
    }
}

func (tc *TracedHTTPClient) Get(ctx context.Context, url string) (*http.Response, error) {
    // Créer un span pour cet appel HTTP
    span, ctx := opentracing.StartSpanFromContext(ctx, "http.client.get")
    defer span.Finish()

    // Ajouter des tags
    span.SetTag("http.method", "GET")
    span.SetTag("http.url", url)
    ext.SpanKindRPCClient.Set(span)

    // Créer la requête
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        ext.Error.Set(span, true)
        span.LogFields(
            log.String("error", err.Error()),
        )
        return nil, err
    }

    // Injecter le contexte de tracing dans les headers HTTP
    err = opentracing.GlobalTracer().Inject(
        span.Context(),
        opentracing.HTTPHeaders,
        opentracing.HTTPHeadersCarrier(req.Header),
    )
    if err != nil {
        span.LogFields(
            log.String("inject.error", err.Error()),
        )
    }

    // Faire l'appel
    resp, err := tc.client.Do(req)
    if err != nil {
        ext.Error.Set(span, true)
        span.LogFields(
            log.String("error", err.Error()),
        )
        return nil, err
    }

    // Ajouter le status code
    span.SetTag("http.status_code", resp.StatusCode)

    if resp.StatusCode >= 400 {
        ext.Error.Set(span, true)
    }

    return resp, nil
}
```

### Extraire le contexte de tracing côté serveur

```go
package middleware

import (
    "github.com/gin-gonic/gin"
    "github.com/opentracing/opentracing-go"
    "github.com/opentracing/opentracing-go/ext"
)

func ExtractTracingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Extraire le contexte de tracing des headers
        spanCtx, _ := opentracing.GlobalTracer().Extract(
            opentracing.HTTPHeaders,
            opentracing.HTTPHeadersCarrier(c.Request.Header),
        )

        // Créer un nouveau span enfant
        span := opentracing.StartSpan(
            fmt.Sprintf("%s %s", c.Request.Method, c.FullPath()),
            ext.RPCServerOption(spanCtx),
        )
        defer span.Finish()

        // Marquer comme span serveur
        ext.SpanKindRPCServer.Set(span)

        // Ajouter au contexte
        ctx := opentracing.ContextWithSpan(c.Request.Context(), span)
        c.Request = c.Request.WithContext(ctx)

        c.Next()

        // Finaliser avec le status
        span.SetTag("http.status_code", c.Writer.Status())
        if c.Writer.Status() >= 400 {
            ext.Error.Set(span, true)
        }
    }
}
```

## Monitoring avancé

### SLI/SLO (Service Level Indicators/Objectives)

```go
package sli

import (
    "time"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

// SLI: Métriques de niveau de service
var (
    // Availability SLI: % de requêtes réussies
    AvailabilitySLI = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "sli_availability_total",
            Help: "Total requests for availability SLI",
        },
        []string{"service", "status"}, // status: "success" ou "failure"
    )

    // Latency SLI: % de requêtes sous un seuil
    LatencySLI = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "sli_latency_seconds",
            Help: "Request latency for SLI calculation",
            Buckets: []float64{0.1, 0.25, 0.5, 1.0, 2.5, 5.0}, // SLO thresholds
        },
        []string{"service", "endpoint"},
    )

    // Quality SLI: % de requêtes sans erreur business
    QualitySLI = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "sli_quality_total",
            Help: "Total requests for quality SLI",
        },
        []string{"service", "operation", "quality"}, // quality: "good" ou "bad"
    )
)

type SLIRecorder struct {
    serviceName string
}

func NewSLIRecorder(serviceName string) *SLIRecorder {
    return &SLIRecorder{serviceName: serviceName}
}

func (s *SLIRecorder) RecordAvailability(success bool) {
    status := "failure"
    if success {
        status = "success"
    }
    AvailabilitySLI.WithLabelValues(s.serviceName, status).Inc()
}

func (s *SLIRecorder) RecordLatency(endpoint string, duration time.Duration) {
    LatencySLI.WithLabelValues(s.serviceName, endpoint).Observe(duration.Seconds())
}

func (s *SLIRecorder) RecordQuality(operation string, isGoodQuality bool) {
    quality := "bad"
    if isGoodQuality {
        quality = "good"
    }
    QualitySLI.WithLabelValues(s.serviceName, operation, quality).Inc()
}
```

### Error Budget et alertes SLO

```yaml
# Règles Prometheus pour SLO
groups:
  - name: slo_rules
    interval: 30s
    rules:
      # Availability SLO: 99.9% (error budget: 0.1%)
      - record: slo:availability_rate
        expr: |
          (
            rate(sli_availability_total{status="success"}[5m]) /
            rate(sli_availability_total[5m])
          )

      # Latency SLO: 95% des requêtes < 500ms
      - record: slo:latency_rate
        expr: |
          (
            rate(sli_latency_seconds_bucket{le="0.5"}[5m]) /
            rate(sli_latency_seconds_count[5m])
          )

      # Error Budget Burn Rate
      - record: slo:error_budget_burn_rate
        expr: |
          (1 - slo:availability_rate) / 0.001  # 0.1% error budget

  - name: slo_alerts
    rules:
      # Fast burn: consomme 5% du budget en 1h
      - alert: ErrorBudgetBurnRateFast
        expr: slo:error_budget_burn_rate > 14.4  # 5% en 1h
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error budget burn rate"
          description: "Service {{ $labels.service }} is burning error budget at {{ $value }}x the acceptable rate"

      # Slow burn: consomme 10% du budget en 6h
      - alert: ErrorBudgetBurnRateSlow
        expr: slo:error_budget_burn_rate > 2.4   # 10% en 6h
        for: 15m
        labels:
          severity: warning
```

### Monitoring de la saturation

```go
package saturation

import (
    "runtime"
    "time"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Métriques de saturation système
    GoroutineCount = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "goroutines_count",
            Help: "Number of active goroutines",
        },
        []string{"service"},
    )

    MemoryUsage = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "memory_usage_bytes",
            Help: "Memory usage in bytes",
        },
        []string{"service", "type"}, // type: heap, stack, etc.
    )

    GCDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "gc_duration_seconds",
            Help: "Garbage collection duration",
            Buckets: []float64{0.001, 0.01, 0.1, 0.5, 1.0},
        },
        []string{"service"},
    )
)

type SaturationMonitor struct {
    serviceName string
    ticker      *time.Ticker
    stopCh      chan struct{}
}

func NewSaturationMonitor(serviceName string) *SaturationMonitor {
    return &SaturationMonitor{
        serviceName: serviceName,
        ticker:      time.NewTicker(30 * time.Second),
        stopCh:      make(chan struct{}),
    }
}

func (sm *SaturationMonitor) Start() {
    go func() {
        for {
            select {
            case <-sm.ticker.C:
                sm.collectMetrics()
            case <-sm.stopCh:
                return
            }
        }
    }()
}

func (sm *SaturationMonitor) Stop() {
    sm.ticker.Stop()
    close(sm.stopCh)
}

func (sm *SaturationMonitor) collectMetrics() {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)

    // Goroutines
    GoroutineCount.WithLabelValues(sm.serviceName).Set(float64(runtime.NumGoroutine()))

    // Memory usage
    MemoryUsage.WithLabelValues(sm.serviceName, "heap_alloc").Set(float64(m.HeapAlloc))
    MemoryUsage.WithLabelValues(sm.serviceName, "heap_sys").Set(float64(m.HeapSys))
    MemoryUsage.WithLabelValues(sm.serviceName, "stack_inuse").Set(float64(m.StackInuse))

    // GC stats
    GCDuration.WithLabelValues(sm.serviceName).Observe(float64(m.PauseTotalNs) / 1e9)
}
```

## Tests de l'observabilité

### Tests des métriques

```go
package metrics

import (
    "testing"
    "time"
    "github.com/prometheus/client_golang/prometheus/testutil"
    "github.com/stretchr/testify/assert"
)

func TestMetricsRecording(t *testing.T) {
    // Reset des métriques pour les tests
    HTTPRequestsTotal.Reset()
    HTTPRequestDuration.Reset()

    service := NewMetricsService("test-service")

    // Enregistrer quelques métriques
    service.RecordHTTPRequest("GET", "/api/users", "200", 100*time.Millisecond)
    service.RecordHTTPRequest("POST", "/api/users", "201", 200*time.Millisecond)
    service.RecordHTTPRequest("GET", "/api/users", "500", 50*time.Millisecond)

    // Vérifier les counters
    assert.Equal(t, 3.0, testutil.ToFloat64(HTTPRequestsTotal))

    // Vérifier les labels
    getRequests := testutil.ToFloat64(HTTPRequestsTotal.WithLabelValues("GET", "/api/users", "200", "test-service"))
    assert.Equal(t, 1.0, getRequests)

    postRequests := testutil.ToFloat64(HTTPRequestsTotal.WithLabelValues("POST", "/api/users", "201", "test-service"))
    assert.Equal(t, 1.0, postRequests)

    errorRequests := testutil.ToFloat64(HTTPRequestsTotal.WithLabelValues("GET", "/api/users", "500", "test-service"))
    assert.Equal(t, 1.0, errorRequests)
}

func TestSLIRecording(t *testing.T) {
    AvailabilitySLI.Reset()

    recorder := NewSLIRecorder("test-service")

    // Simuler des requêtes
    recorder.RecordAvailability(true)   // success
    recorder.RecordAvailability(true)   // success
    recorder.RecordAvailability(false)  // failure

    // Vérifier les métriques
    totalSuccess := testutil.ToFloat64(AvailabilitySLI.WithLabelValues("test-service", "success"))
    totalFailure := testutil.ToFloat64(AvailabilitySLI.WithLabelValues("test-service", "failure"))

    assert.Equal(t, 2.0, totalSuccess)
    assert.Equal(t, 1.0, totalFailure)
}
```

### Tests des logs

```go
package logger

import (
    "bytes"
    "encoding/json"
    "testing"
    "github.com/sirupsen/logrus"
    "github.com/stretchr/testify/assert"
)

func TestStructuredLogging(t *testing.T) {
    // Capturer les logs dans un buffer
    var buf bytes.Buffer

    logger := logrus.New()
    logger.SetOutput(&buf)
    logger.SetFormatter(&logrus.JSONFormatter{})

    contextLogger := &ContextLogger{
        logger:  logger,
        service: "test-service",
        version: "1.0.0",
    }

    // Log un message
    contextLogger.Info("Test message", map[string]interface{}{
        "user_id": "user-123",
        "action":  "test",
    })

    // Parser le JSON
    var logEntry map[string]interface{}
    err := json.Unmarshal(buf.Bytes(), &logEntry)
    assert.NoError(t, err)

    // Vérifier les champs
    assert.Equal(t, "Test message", logEntry["message"])
    assert.Equal(t, "test-service", logEntry["service"])
    assert.Equal(t, "1.0.0", logEntry["version"])
    assert.Equal(t, "user-123", logEntry["user_id"])
    assert.Equal(t, "test", logEntry["action"])
    assert.Equal(t, "info", logEntry["level"])
}

func TestLogSampling(t *testing.T) {
    logger := logrus.New()
    logger.SetOutput(&bytes.Buffer{}) // Discard output

    samplingLogger := NewSamplingLogger(logger, 0.5) // 50% sample rate

    // Tester avec la même clé (doit être consistant)
    key := "test-key"
    result1 := samplingLogger.ShouldLog("info", key)
    result2 := samplingLogger.ShouldLog("info", key)

    assert.Equal(t, result1, result2, "Sampling should be consistent for the same key")

    // Tester que les erreurs sont toujours loggées
    assert.True(t, samplingLogger.ShouldLog("error", key))
    assert.True(t, samplingLogger.ShouldLog("fatal", key))
}
```

## Configuration en production

### Variables d'environnement

```go
package config

import (
    "os"
    "strconv"
    "time"
)

type ObservabilityConfig struct {
    // Logging
    LogLevel        string
    LogFormat       string // "json" ou "text"
    LogSampling     float64

    // Metrics
    MetricsEnabled  bool
    MetricsPort     int
    MetricsPath     string

    // Tracing
    TracingEnabled  bool
    TracingSampler  float64
    JaegerEndpoint  string

    // Health checks
    HealthPath      string
    ReadinessPath   string
    LivenessPath    string
}

func LoadObservabilityConfig() *ObservabilityConfig {
    return &ObservabilityConfig{
        LogLevel:       getEnvString("LOG_LEVEL", "info"),
        LogFormat:      getEnvString("LOG_FORMAT", "json"),
        LogSampling:    getEnvFloat("LOG_SAMPLING", 1.0),

        MetricsEnabled: getEnvBool("METRICS_ENABLED", true),
        MetricsPort:    getEnvInt("METRICS_PORT", 8090),
        MetricsPath:    getEnvString("METRICS_PATH", "/metrics"),

        TracingEnabled: getEnvBool("TRACING_ENABLED", true),
        TracingSampler: getEnvFloat("TRACING_SAMPLER", 0.1), // 10% en prod
        JaegerEndpoint: getEnvString("JAEGER_ENDPOINT", "jaeger:6831"),

        HealthPath:     getEnvString("HEALTH_PATH", "/health"),
        ReadinessPath:  getEnvString("READINESS_PATH", "/health/ready"),
        LivenessPath:   getEnvString("LIVENESS_PATH", "/health/live"),
    }
}

func getEnvString(key, defaultValue string) string {
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

func getEnvFloat(key string, defaultValue float64) float64 {
    if value := os.Getenv(key); value != "" {
        if floatValue, err := strconv.ParseFloat(value, 64); err == nil {
            return floatValue
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
```

## Configuration Docker Compose pour environnement complet

```yaml
version: '3.8'

services:
  # Application services
  user-service:
    build: ./user-service
    environment:
      - LOG_LEVEL=info
      - LOG_FORMAT=json
      - METRICS_ENABLED=true
      - METRICS_PORT=8090
      - TRACING_ENABLED=true
      - TRACING_SAMPLER=0.1
      - JAEGER_ENDPOINT=jaeger:6831
      - SERVICE_NAME=user-service
      - SERVICE_VERSION=1.0.0
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=8090"
      - "prometheus.path=/metrics"
    ports:
      - "8081:8080"
      - "8091:8090"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - observability
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  order-service:
    build: ./order-service
    environment:
      - LOG_LEVEL=info
      - LOG_FORMAT=json
      - METRICS_ENABLED=true
      - METRICS_PORT=8090
      - TRACING_ENABLED=true
      - TRACING_SAMPLER=0.1
      - JAEGER_ENDPOINT=jaeger:6831
      - SERVICE_NAME=order-service
      - SERVICE_VERSION=1.0.0
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=8090"
      - "prometheus.path=/metrics"
    ports:
      - "8082:8080"
      - "8092:8090"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - observability
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  product-service:
    build: ./product-service
    environment:
      - LOG_LEVEL=info
      - LOG_FORMAT=json
      - METRICS_ENABLED=true
      - METRICS_PORT=8090
      - TRACING_ENABLED=true
      - TRACING_SAMPLER=0.1
      - JAEGER_ENDPOINT=jaeger:6831
      - SERVICE_NAME=product-service
      - SERVICE_VERSION=1.0.0
    labels:
      - "prometheus.scrape=true"
      - "prometheus.port=8090"
      - "prometheus.path=/metrics"
    ports:
      - "8083:8080"
      - "8093:8090"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - observability
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Monitoring stack
  jaeger:
    image: jaegertracing/all-in-one:1.50
    ports:
      - "16686:16686"  # Jaeger UI
      - "6831:6831/udp"  # Jaeger agent
      - "14268:14268"   # Jaeger collector
    environment:
      - COLLECTOR_ZIPKIN_HTTP_PORT=9411
      - SPAN_STORAGE_TYPE=elasticsearch
      - ES_SERVER_URLS=http://elasticsearch:9200
      - ES_TAGS_AS_FIELDS_ALL=true
    depends_on:
      - elasticsearch
    networks:
      - observability
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:v2.47.0
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/rules:/etc/prometheus/rules
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - observability
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.1.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_SECURITY_ADMIN_USER=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-worldmap-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
      - ./monitoring/grafana/config:/etc/grafana/provisioning/notifiers
    depends_on:
      - prometheus
    networks:
      - observability
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.26.0
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://localhost:9093'
    networks:
      - observability
    restart: unless-stopped

  # ELK Stack pour les logs
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - observability
    restart: unless-stopped

  logstash:
    image: docker.elastic.co/logstash/logstash:8.10.0
    volumes:
      - ./monitoring/logstash/pipeline:/usr/share/logstash/pipeline
      - ./monitoring/logstash/config:/usr/share/logstash/config
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - "LS_JAVA_OPTS=-Xmx256m -Xms256m"
    depends_on:
      - elasticsearch
    networks:
      - observability
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - ELASTICSEARCH_USERNAME=elastic
      - ELASTICSEARCH_PASSWORD=changeme
    depends_on:
      - elasticsearch
    networks:
      - observability
    restart: unless-stopped

  # Filebeat pour collecter les logs des conteneurs
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.10.0
    user: root
    volumes:
      - ./monitoring/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash
    networks:
      - observability
    restart: unless-stopped

  # Node Exporter pour les métriques système
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
      - observability
    restart: unless-stopped

  # cAdvisor pour les métriques des conteneurs
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
      - observability
    restart: unless-stopped

networks:
  observability:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
  elasticsearch-data:
```

## Configuration Prometheus détaillée

### monitoring/prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'local'
    region: 'local'

rule_files:
  - "rules/*.yml"

scrape_configs:
  # Microservices
  - job_name: 'microservices'
    static_configs:
      - targets:
          - 'user-service:8090'
          - 'order-service:8090'
          - 'product-service:8090'
    scrape_interval: 10s
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
      - source_labels: [__address__]
        regex: '([^:]+):(.*)'
        target_label: service
        replacement: '${1}'

  # Infrastructure monitoring
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Monitoring stack
  - job_name: 'grafana'
    static_configs:
      - targets: ['grafana:3000']
    metrics_path: /metrics

  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093'
```

### monitoring/rules/microservices.yml

```yaml
groups:
  - name: microservices.rules
    interval: 30s
    rules:
      # SLI/SLO Rules
      - record: sli:http_requests:rate5m
        expr: |
          sum(rate(http_requests_total[5m])) by (service, method, endpoint)

      - record: sli:http_requests_errors:rate5m
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service, method, endpoint)

      - record: sli:http_requests_availability:ratio5m
        expr: |
          1 - (
            sli:http_requests_errors:rate5m /
            sli:http_requests:rate5m
          )

      - record: sli:http_request_latency:p99_5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, endpoint, le)
          )

      - record: sli:http_request_latency:p95_5m
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, endpoint, le)
          )

      - record: sli:http_request_latency:p50_5m
        expr: |
          histogram_quantile(0.50,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, endpoint, le)
          )

  - name: microservices.alerts
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service) /
            sum(rate(http_requests_total[5m])) by (service)
          ) * 100 > 5
        for: 2m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} on service {{ $labels.service }}"
          runbook_url: "https://runbooks.company.com/high-error-rate"

      # Very High Error Rate
      - alert: VeryHighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service) /
            sum(rate(http_requests_total[5m])) by (service)
          ) * 100 > 15
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Very high error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} on service {{ $labels.service }}"
          runbook_url: "https://runbooks.company.com/very-high-error-rate"

      # High Latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          ) > 1
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High latency on {{ $labels.service }}"
          description: "95th percentile latency is {{ $value }}s on service {{ $labels.service }}"

      # Service Down
      - alert: ServiceDown
        expr: up{job="microservices"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.service }} is down"
          description: "Service {{ $labels.service }} has been down for more than 1 minute"

      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          (
            memory_usage_bytes{type="heap_alloc"} /
            memory_usage_bytes{type="heap_sys"}
          ) * 100 > 80
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "High memory usage on {{ $labels.service }}"
          description: "Memory usage is {{ $value | humanizePercentage }} on service {{ $labels.service }}"

      # Too Many Goroutines
      - alert: TooManyGoroutines
        expr: goroutines_count > 1000
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Too many goroutines on {{ $labels.service }}"
          description: "{{ $labels.service }} has {{ $value }} goroutines"

      # Database Health
      - alert: DatabaseUnhealthy
        expr: database_health == 0
        for: 30s
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Database unhealthy"
          description: "Database {{ $labels.database_name }} is unhealthy on service {{ $labels.service }}"
```

## Configuration Grafana

### monitoring/grafana/datasources/prometheus.yml

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: "5s"
      queryTimeout: "60s"
      httpMethod: "GET"

  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686
    editable: true

  - name: Elasticsearch
    type: elasticsearch
    access: proxy
    url: http://elasticsearch:9200
    database: "filebeat-*"
    jsonData:
      timeField: "@timestamp"
      esVersion: "8.0.0"
      logMessageField: "message"
      logLevelField: "level"
```

### monitoring/grafana/dashboards/dashboard.yml

```yaml
apiVersion: 1

providers:
  - name: 'microservices'
    orgId: 1
    folder: 'Microservices'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards/microservices
```

### Dashboard JSON complet pour microservices

```json
{
  "dashboard": {
    "id": null,
    "title": "Microservices Monitoring",
    "description": "Complete monitoring dashboard for microservices",
    "tags": ["microservices", "monitoring"],
    "timezone": "browser",
    "refresh": "5s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Service Health Overview",
        "type": "stat",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "up{job=\"microservices\"}",
            "legendFormat": "{{service}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "green", "value": 1}
              ]
            },
            "mappings": [
              {"options": {"0": {"text": "DOWN"}}, "type": "value"},
              {"options": {"1": {"text": "UP"}}, "type": "value"}
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "HTTP Requests Rate",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec",
            "min": 0
          }
        ],
        "legend": {
          "show": true,
          "values": true,
          "current": true,
          "avg": true
        }
      },
      {
        "id": 3,
        "title": "HTTP Error Rate",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"4..|5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
            "legendFormat": "{{service}}"
          }
        ],
        "yAxes": [
          {
            "label": "Error Rate (%)",
            "min": 0,
            "max": 100
          }
        ],
        "thresholds": [
          {
            "value": 5,
            "colorMode": "critical",
            "op": "gt"
          },
          {
            "value": 1,
            "colorMode": "warning",
            "op": "gt"
          }
        ]
      },
      {
        "id": 4,
        "title": "Response Time Percentiles",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 16},
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le))",
            "legendFormat": "{{service}} 50th"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le))",
            "legendFormat": "{{service}} 95th"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le))",
            "legendFormat": "{{service}} 99th"
          }
        ],
        "yAxes": [
          {
            "label": "Response Time (s)",
            "min": 0
          }
        ]
      },
      {
        "id": 5,
        "title": "Memory Usage",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 16},
        "targets": [
          {
            "expr": "memory_usage_bytes{type=\"heap_alloc\"} / 1024 / 1024",
            "legendFormat": "{{service}} Heap Alloc (MB)"
          },
          {
            "expr": "memory_usage_bytes{type=\"heap_sys\"} / 1024 / 1024",
            "legendFormat": "{{service}} Heap Sys (MB)"
          }
        ],
        "yAxes": [
          {
            "label": "Memory (MB)",
            "min": 0
          }
        ]
      },
      {
        "id": 6,
        "title": "Active Goroutines",
        "type": "graph",
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 24},
        "targets": [
          {
            "expr": "goroutines_count",
            "legendFormat": "{{service}}"
          }
        ],
        "yAxes": [
          {
            "label": "Goroutines",
            "min": 0
          }
        ]
      },
      {
        "id": 7,
        "title": "Database Health",
        "type": "stat",
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 24},
        "targets": [
          {
            "expr": "database_health",
            "legendFormat": "{{service}} - {{database_name}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "green", "value": 1}
              ]
            },
            "mappings": [
              {"options": {"0": {"text": "UNHEALTHY"}}, "type": "value"},
              {"options": {"1": {"text": "HEALTHY"}}, "type": "value"}
            ]
          }
        }
      }
    ]
  }
}
```

## Configuration ELK Stack

### monitoring/logstash/pipeline/logstash.conf

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON logs from containers
  if [container][name] =~ /-service$/ {
    json {
      source => "message"
      target => "app"
    }

    # Extract service name from container name
    grok {
      match => { "[container][name]" => "/?(?<service_name>[^-]+)-service" }
    }

    # Add standard fields
    mutate {
      add_field => { "log_type" => "microservice" }
      add_field => { "environment" => "development" }
    }

    # Parse timestamp if present
    if [app][timestamp] {
      date {
        match => [ "[app][timestamp]", "ISO8601" ]
        target => "@timestamp"
      }
    }

    # Normalize log levels
    if [app][level] {
      mutate {
        uppercase => [ "[app][level]" ]
      }
    }
  }

  # Add Docker metadata
  mutate {
    add_field => { "docker_image" => "%{[container][image][name]}" }
    add_field => { "docker_container_id" => "%{[container][id]}" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "microservices-logs-%{+YYYY.MM.dd}"
    template_name => "microservices"
    template => "/usr/share/logstash/templates/microservices.json"
    template_overwrite => true
  }

  # Debug output
  stdout {
    codec => rubydebug
  }
}
```

### monitoring/logstash/templates/microservices.json

```json
{
  "index_patterns": ["microservices-logs-*"],
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "index.refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "service_name": {"type": "keyword"},
      "log_type": {"type": "keyword"},
      "environment": {"type": "keyword"},
      "app": {
        "properties": {
          "level": {"type": "keyword"},
          "message": {"type": "text"},
          "service": {"type": "keyword"},
          "version": {"type": "keyword"},
          "request_id": {"type": "keyword"},
          "user_id": {"type": "keyword"},
          "order_id": {"type": "keyword"},
          "duration_ms": {"type": "long"}
        }
      },
      "container": {
        "properties": {
          "name": {"type": "keyword"},
          "id": {"type": "keyword"},
          "image": {
            "properties": {
              "name": {"type": "keyword"}
            }
          }
        }
      }
    }
  }
}
```

### monitoring/filebeat/filebeat.yml

```yaml
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  processors:
    - add_docker_metadata:
        host: "unix:///var/run/docker.sock"
    - decode_json_fields:
        fields: ["message"]
        target: ""
        overwrite_keys: true

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_docker_metadata: ~
```

## Configuration Alertmanager

### monitoring/alertmanager.yml

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'monitoring@yourcompany.com'
  smtp_auth_username: 'monitoring@yourcompany.com'
  smtp_auth_password: 'your-app-password'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty'
      repeat_interval: 5m

    # Warning alerts go to Slack
    - match:
        severity: warning
      receiver: 'slack'
      repeat_interval: 30m

    # Team-specific routing
    - match:
        team: platform
      receiver: 'platform-team'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team-lead@yourcompany.com'
        subject: '[ALERT] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Severity: {{ .Labels.severity }}
          Service: {{ .Labels.service }}
          Time: {{ .StartsAt }}
          {{ if .Annotations.runbook_url }}
          Runbook: {{ .Annotations.runbook_url }}
          {{ end }}
          {{ end }}

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#monitoring'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Service:* {{ .Labels.service }}
          {{ if .Annotations.runbook_url }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}
          {{ end }}
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_INTEGRATION_KEY'
        description: '{{ .GroupLabels.alertname }} - {{ .GroupLabels.service }}'
        details:
          severity: '{{ .GroupLabels.severity }}'
          service: '{{ .GroupLabels.service }}'
          alert_count: '{{ len .Alerts }}'
          cluster: '{{ .GroupLabels.cluster }}'

  - name: 'platform-team'
    email_configs:
      - to: 'platform-team@yourcompany.com'
        subject: '[PLATFORM] {{ .GroupLabels.alertname }}'
        body: |
          Platform Team Alert

          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Service: {{ .Labels.service }}
          Severity: {{ .Labels.severity }}
          Started: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          {{ if .Annotations.runbook_url }}
          Runbook: {{ .Annotations.runbook_url }}
          {{ end }}

          Labels:
          {{ range .Labels.SortedPairs }}  {{ .Name }}: {{ .Value }}
          {{ end }}
          {{ end }}
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/PLATFORM/WEBHOOK'
        channel: '#platform-alerts'
        title: '[PLATFORM] {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          :warning: *{{ .Annotations.summary }}*
          *Description:* {{ .Annotations.description }}
          *Service:* {{ .Labels.service }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}

  - name: 'webhook'
    webhook_configs:
      - url: 'http://your-webhook-service:8080/alerts'
        send_resolved: true
        http_config:
          basic_auth:
            username: 'webhook_user'
            password: 'webhook_password'
        max_alerts: 10

inhibit_rules:
  # Inhibit warning alerts when critical alerts are firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'service']

  # Inhibit duplicate alerts
  - source_match:
      alertname: 'ServiceDown'
    target_match:
      alertname: 'HighErrorRate'
    equal: ['service']
```

## Templates d'alertes personnalisées

### monitoring/alertmanager/templates/custom.tmpl

```go
{{ define "slack.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
{{ .GroupLabels.SortedPairs.Values | join " " }}
{{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
{{ if .Annotations.summary }}*Summary:* {{ .Annotations.summary }}{{ end }}
{{ if .Annotations.description }}*Description:* {{ .Annotations.description }}{{ end }}
*Severity:* {{ .Labels.severity | title }}
*Service:* {{ .Labels.service }}
*Instance:* {{ .Labels.instance }}
{{ if .Annotations.runbook_url }}*Runbook:* {{ .Annotations.runbook_url }}{{ end }}
*Started:* {{ .StartsAt.Format "Mon Jan 2 15:04:05 MST 2006" }}
{{ if .EndsAt }}*Ended:* {{ .EndsAt.Format "Mon Jan 2 15:04:05 MST 2006" }}{{ end }}
---
{{ end }}
{{ end }}

{{ define "email.subject" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }} on {{ .GroupLabels.service }}
{{ end }}

{{ define "email.html" }}
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .alert { border: 1px solid #ddd; margin: 10px 0; padding: 15px; border-radius: 5px; }
        .critical { border-left: 5px solid #d32f2f; background-color: #ffebee; }
        .warning { border-left: 5px solid #f57c00; background-color: #fff3e0; }
        .info { border-left: 5px solid #1976d2; background-color: #e3f2fd; }
        .resolved { border-left: 5px solid #388e3c; background-color: #e8f5e8; }
        .label { background-color: #e0e0e0; padding: 2px 8px; border-radius: 3px; margin: 2px; }
    </style>
</head>
<body>
    <h2>Alert: {{ .GroupLabels.alertname }}</h2>
    <p><strong>Status:</strong> {{ .Status | title }}</p>
    <p><strong>Service:</strong> {{ .GroupLabels.service }}</p>

    {{ range .Alerts }}
    <div class="alert {{ .Labels.severity }}">
        <h3>{{ .Annotations.summary }}</h3>
        <p>{{ .Annotations.description }}</p>

        <p><strong>Started:</strong> {{ .StartsAt.Format "2006-01-02 15:04:05 MST" }}</p>
        {{ if .EndsAt }}<p><strong>Ended:</strong> {{ .EndsAt.Format "2006-01-02 15:04:05 MST" }}</p>{{ end }}

        <h4>Labels:</h4>
        {{ range .Labels.SortedPairs }}
        <span class="label">{{ .Name }}: {{ .Value }}</span>
        {{ end }}

        {{ if .Annotations.runbook_url }}
        <p><a href="{{ .Annotations.runbook_url }}">📖 View Runbook</a></p>
        {{ end }}
    </div>
    {{ end }}
</body>
</html>
{{ end }}
```

## Scripts de gestion et automation

### scripts/monitoring-setup.sh

```bash
#!/bin/bash

# Script de configuration automatique du monitoring

set -e

echo "🚀 Setting up monitoring stack..."

# Créer les répertoires nécessaires
mkdir -p monitoring/{prometheus,grafana/{dashboards,datasources},alertmanager,logstash/{config,pipeline},filebeat}
mkdir -p data/{prometheus,grafana,elasticsearch,alertmanager}

# Permissions pour Grafana
sudo chown -R 472:472 data/grafana

# Permissions pour Elasticsearch
sudo chown -R 1000:1000 data/elasticsearch

echo "📊 Configuring Prometheus..."
# Valider la configuration Prometheus
docker run --rm -v $(pwd)/monitoring/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus:latest promtool check config /etc/prometheus/prometheus.yml

echo "🔔 Configuring Alertmanager..."
# Valider la configuration Alertmanager
docker run --rm -v $(pwd)/monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml prom/alertmanager:latest amtool check-config /etc/alertmanager/alertmanager.yml

echo "📈 Starting monitoring stack..."
docker-compose up -d

echo "⏳ Waiting for services to be ready..."
sleep 30

echo "🔍 Checking service health..."
services=("prometheus:9090" "grafana:3000" "alertmanager:9093" "jaeger:16686")

for service in "${services[@]}"; do
    IFS=':' read -r name port <<< "$service"
    if curl -s "http://localhost:${port}" > /dev/null; then
        echo "✅ $name is ready"
    else
        echo "❌ $name is not responding"
    fi
done

echo "🎉 Monitoring stack is ready!"
echo ""
echo "📊 Prometheus: http://localhost:9090"
echo "📈 Grafana: http://localhost:3000 (admin/admin123)"
echo "🔔 Alertmanager: http://localhost:9093"
echo "🔍 Jaeger: http://localhost:16686"
echo "📝 Kibana: http://localhost:5601"
```

### scripts/backup-monitoring.sh

```bash
#!/bin/bash

# Script de sauvegarde des données de monitoring

set -e

BACKUP_DIR="/backup/monitoring/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "📦 Creating backup in $BACKUP_DIR..."

# Backup Prometheus data
echo "📊 Backing up Prometheus data..."
docker run --rm -v monitoring_prometheus-data:/data -v "$BACKUP_DIR":/backup alpine tar czf /backup/prometheus-data.tar.gz -C /data .

# Backup Grafana data
echo "📈 Backing up Grafana data..."
docker run --rm -v monitoring_grafana-data:/data -v "$BACKUP_DIR":/backup alpine tar czf /backup/grafana-data.tar.gz -C /data .

# Backup Elasticsearch data
echo "🔍 Backing up Elasticsearch data..."
docker run --rm -v monitoring_elasticsearch-data:/data -v "$BACKUP_DIR":/backup alpine tar czf /backup/elasticsearch-data.tar.gz -C /data .

# Backup configurations
echo "⚙️ Backing up configurations..."
tar czf "$BACKUP_DIR/configs.tar.gz" monitoring/

# Create backup info
cat > "$BACKUP_DIR/backup-info.txt" << EOF
Backup created: $(date)
Hostname: $(hostname)
Docker version: $(docker --version)
Backup size: $(du -sh "$BACKUP_DIR" | cut -f1)
EOF

echo "✅ Backup completed: $BACKUP_DIR"
```

### scripts/health-check.sh

```bash
#!/bin/bash

# Script de vérification de santé du monitoring

set -e

echo "🏥 Health Check - Monitoring Stack"
echo "=================================="

# Function to check HTTP endpoint
check_endpoint() {
    local name=$1
    local url=$2
    local expected_code=${3:-200}

    if response=$(curl -s -o /dev/null -w "%{http_code}" "$url" 2>/dev/null); then
        if [ "$response" -eq "$expected_code" ]; then
            echo "✅ $name: OK (HTTP $response)"
            return 0
        else
            echo "⚠️  $name: Unexpected response (HTTP $response)"
            return 1
        fi
    else
        echo "❌ $name: Not reachable"
        return 1
    fi
}

# Function to check Docker service
check_docker_service() {
    local service=$1

    if docker-compose ps "$service" | grep -q "Up"; then
        echo "✅ Docker service $service: Running"
        return 0
    else
        echo "❌ Docker service $service: Not running"
        return 1
    fi
}

# Check Docker services
echo "🐳 Checking Docker services..."
services=("prometheus" "grafana" "alertmanager" "jaeger" "elasticsearch" "kibana" "logstash")
docker_issues=0

for service in "${services[@]}"; do
    if ! check_docker_service "$service"; then
        ((docker_issues++))
    fi
done

# Check HTTP endpoints
echo ""
echo "🌐 Checking HTTP endpoints..."
endpoints=(
    "Prometheus:http://localhost:9090/-/healthy"
    "Grafana:http://localhost:3000/api/health"
    "Alertmanager:http://localhost:9093/-/healthy"
    "Jaeger:http://localhost:16686/"
    "Elasticsearch:http://localhost:9200/_cluster/health"
    "Kibana:http://localhost:5601/api/status"
)

http_issues=0
for endpoint in "${endpoints[@]}"; do
    IFS=':' read -r name url <<< "$endpoint"
    if ! check_endpoint "$name" "$url"; then
        ((http_issues++))
    fi
done

# Check data volumes
echo ""
echo "💾 Checking data volumes..."
volumes=("prometheus-data" "grafana-data" "elasticsearch-data" "alertmanager-data")
volume_issues=0

for volume in "${volumes[@]}"; do
    if docker volume inspect "monitoring_$volume" > /dev/null 2>&1; then
        echo "✅ Volume $volume: Exists"
    else
        echo "❌ Volume $volume: Missing"
        ((volume_issues++))
    fi
done

# Check disk space
echo ""
echo "💽 Checking disk space..."
disk_usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$disk_usage" -lt 80 ]; then
    echo "✅ Disk space: OK ($disk_usage% used)"
else
    echo "⚠️  Disk space: Warning ($disk_usage% used)"
fi

# Summary
echo ""
echo "📋 Health Check Summary"
echo "======================"
echo "Docker services issues: $docker_issues"
echo "HTTP endpoints issues: $http_issues"
echo "Volume issues: $volume_issues"

total_issues=$((docker_issues + http_issues + volume_issues))
if [ "$total_issues" -eq 0 ]; then
    echo "🎉 All systems operational!"
    exit 0
else
    echo "⚠️  $total_issues issues found"
    exit 1
fi
```

## Runbooks et documentation

### docs/runbooks/high-error-rate.md

```markdown
# Runbook: High Error Rate

## Alert Description
This alert fires when the error rate for a service exceeds 5% over a 2-minute period.

## Severity: Warning

## Immediate Actions

### 1. Check Service Health
```bash
# Check if service is responding
curl -f http://service:8080/health

# Check recent logs
docker logs service-name --tail 100

# Check metrics
curl http://service:8080/metrics | grep http_requests_total
```

### 2. Identify Error Types
```bash
# Check error logs in Kibana
# Filter by: service_name AND level:ERROR

# Check Grafana dashboard
# Look at "HTTP Status Codes" panel
```

### 3. Common Causes & Solutions

#### Database Connection Issues
```bash
# Check database health
curl http://service:8080/health | jq '.checks.database'

# Check database logs
docker logs postgres-container --tail 50

# Restart database connection pool
curl -X POST http://service:8080/admin/db/reconnect
```

#### Downstream Service Failures
```bash
# Check dependent services
kubectl get pods -l app=dependent-service

# Check circuit breaker status
curl http://service:8080/metrics | grep circuit_breaker_state
```

#### Resource Exhaustion
```bash
# Check memory usage
docker stats service-container

# Check goroutines
curl http://service:8080/metrics | grep goroutines_count
```

## Escalation

If error rate exceeds 15% or persists for more than 10 minutes:
1. Page on-call engineer
2. Consider rolling back recent deployments
3. Scale service horizontally if resource constrained

## Prevention

- Implement proper retry logic with exponential backoff
- Add circuit breakers for downstream calls
- Monitor resource usage and set up auto-scaling
- Implement health checks for all dependencies
```

### docs/runbooks/service-down.md

```markdown
# Runbook: Service Down

## Alert Description
This alert fires when a service instance is not responding to health checks.

## Severity: Critical

## Immediate Actions

### 1. Verify Service Status
```bash
# Check Docker container status
docker ps | grep service-name

# Check container logs
docker logs service-name --tail 50

# Try to restart service
docker-compose restart service-name
```

### 2. Check Infrastructure
```bash
# Check available resources
free -h
df -h
docker system df

# Check network connectivity
ping service-host
telnet service-host 8080
```

### 3. Emergency Procedures

#### Quick Recovery
```bash
# Scale up replicas (if using orchestrator)
kubectl scale deployment service-name --replicas=3

# Manual restart
docker-compose up -d service-name

# Check load balancer configuration
curl -I http://load-balancer/service-endpoint
```

#### Data Recovery
```bash
# Check data volumes
docker volume ls | grep service

# Restore from backup if needed
./scripts/restore-service-data.sh latest
```

## Root Cause Analysis

After service recovery, investigate:
1. Review application logs for panic/crashes
2. Check system resources at time of failure
3. Review recent deployments or configuration changes
4. Check database connectivity and queries
5. Review monitoring dashboards for trends

## Prevention

- Implement proper graceful shutdown
- Add resource limits and health checks
- Set up auto-restart policies
- Monitor key metrics and set up proactive alerts
- Regular chaos engineering tests
```

## Métriques SRE et SLOs

### monitoring/slo-config.yml

```yaml
# Service Level Objectives Configuration
slos:
  user-service:
    availability:
      target: 99.9  # 99.9% availability
      window: 30d   # 30-day window
      error_budget: 0.1  # 0.1% error budget

    latency:
      target: 95    # 95% of requests
      threshold: 500ms  # under 500ms
      window: 30d

    quality:
      target: 99    # 99% of requests without business errors
      window: 30d

  order-service:
    availability:
      target: 99.95 # Higher SLO for critical service
      window: 30d
      error_budget: 0.05

    latency:
      target: 90
      threshold: 1s
      window: 30d

# Error Budget Policies
error_budget_policies:
  - name: "Fast Burn"
    condition: "5% budget consumed in 1 hour"
    action: "Page on-call engineer immediately"

  - name: "Medium Burn"
    condition: "10% budget consumed in 6 hours"
    action: "Create incident, notify team"

  - name: "Slow Burn"
    condition: "50% budget consumed in 3 days"
    action: "Review and plan improvements"
```

### Calcul automatique des SLOs

```go
package slo

import (
    "context"
    "fmt"
    "time"

    "github.com/prometheus/client_golang/api"
    v1 "github.com/prometheus/client_golang/api/prometheus/v1"
)

type SLOCalculator struct {
    client v1.API
}

type SLOResult struct {
    Service        string    `json:"service"`
    SLOType        string    `json:"slo_type"`
    Target         float64   `json:"target"`
    Current        float64   `json:"current"`
    ErrorBudget    float64   `json:"error_budget"`
    BudgetRemaining float64  `json:"budget_remaining"`
    Window         string    `json:"window"`
    CalculatedAt   time.Time `json:"calculated_at"`
}

func NewSLOCalculator(prometheusURL string) (*SLOCalculator, error) {
    client, err := api.NewClient(api.Config{
        Address: prometheusURL,
    })
    if err != nil {
        return nil, err
    }

    return &SLOCalculator{
        client: v1.NewAPI(client),
    }, nil
}

func (s *SLOCalculator) CalculateAvailabilitySLO(ctx context.Context, service string, window time.Duration, target float64) (*SLOResult, error) {
    // Query pour calculer la disponibilité
    query := fmt.Sprintf(`
        sum(rate(sli_availability_total{service="%s",status="success"}[%s])) /
        sum(rate(sli_availability_total{service="%s"}[%s]))
    `, service, formatDuration(window), service, formatDuration(window))

    result, _, err := s.client.Query(ctx, query, time.Now())
    if err != nil {
        return nil, err
    }

    // Extraire la valeur
    availability := extractValue(result)
    errorBudget := 1 - target
    budgetUsed := 1 - availability
    budgetRemaining := errorBudget - budgetUsed

    return &SLOResult{
        Service:         service,
        SLOType:         "availability",
        Target:          target,
        Current:         availability,
        ErrorBudget:     errorBudget,
        BudgetRemaining: budgetRemaining,
        Window:          window.String(),
        CalculatedAt:    time.Now(),
    }, nil
}

func (s *SLOCalculator) CalculateLatencySLO(ctx context.Context, service string, threshold time.Duration, percentile float64, window time.Duration) (*SLOResult, error) {
    // Query pour calculer le pourcentage de requêtes sous le seuil
    query := fmt.Sprintf(`
        sum(rate(sli_latency_seconds_bucket{service="%s",le="%g"}[%s])) /
        sum(rate(sli_latency_seconds_count{service="%s"}[%s]))
    `, service, threshold.Seconds(), formatDuration(window), service, formatDuration(window))

    result, _, err := s.client.Query(ctx, query, time.Now())
    if err != nil {
        return nil, err
    }

    current := extractValue(result)
    target := percentile / 100
    errorBudget := 1 - target
    budgetUsed := 1 - current
    budgetRemaining := errorBudget - budgetUsed

    return &SLOResult{
        Service:         service,
        SLOType:         "latency",
        Target:          target,
        Current:         current,
        ErrorBudget:     errorBudget,
        BudgetRemaining: budgetRemaining,
        Window:          window.String(),
        CalculatedAt:    time.Now(),
    }, nil
}

func formatDuration(d time.Duration) string {
    if d >= 24*time.Hour {
        return fmt.Sprintf("%dd", int(d.Hours()/24))
    }
    if d >= time.Hour {
        return fmt.Sprintf("%dh", int(d.Hours()))
    }
    return fmt.Sprintf("%dm", int(d.Minutes()))
}

// API endpoint pour exposer les SLOs
func (h *SLOHandler) GetSLOs(c *gin.Context) {
    service := c.Query("service")
    if service == "" {
        c.JSON(400, gin.H{"error": "service parameter required"})
        return
    }

    ctx := c.Request.Context()
    window := 30 * 24 * time.Hour // 30 jours

    // Calculer availability SLO
    availabilitySLO, err := h.calculator.CalculateAvailabilitySLO(ctx, service, window, 0.999)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    // Calculer latency SLO
    latencySLO, err := h.calculator.CalculateLatencySLO(ctx, service, 500*time.Millisecond, 95, window)
    if err != nil {
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }

    c.JSON(200, gin.H{
        "service": service,
        "slos": []SLOResult{*availabilitySLO, *latencySLO},
        "calculated_at": time.Now(),
    })
}
```

## Tests de chaos et résilience

### scripts/chaos-testing.sh

```bash
#!/bin/bash

# Script de tests de chaos pour valider la résilience

set -e

echo "🔥 Starting Chaos Engineering Tests"
echo "=================================="

# Function to simulate high CPU load
simulate_cpu_load() {
    local service=$1
    echo "💥 Simulating high CPU load on $service..."

    docker exec "$service" sh -c "
        for i in {1..4}; do
            yes > /dev/null &
        done
        sleep 30
        killall yes
    " &

    sleep 35
    echo "✅ CPU load test completed for $service"
}

# Function to simulate memory pressure
simulate_memory_pressure() {
    local service=$1
    echo "💥 Simulating memory pressure on $service..."

    docker exec "$service" sh -c "
        # Allocate 100MB of memory
        python3 -c 'a = [0] * (100 * 1024 * 1024); import time; time.sleep(30)'
    " &

    sleep 35
    echo "✅ Memory pressure test completed for $service"
}

# Function to simulate network latency
simulate_network_latency() {
    local service=$1
    echo "💥 Simulating network latency for $service..."

    # Add 100ms latency to all traffic
    docker exec "$service" tc qdisc add dev eth0 root handle 1: prio
    docker exec "$service" tc qdisc add dev eth0 parent 1:3 handle 30: netem delay 100ms
    docker exec "$service" tc filter add dev eth0 protocol ip parent 1:0 prio 3 u32 match ip dst 0.0.0.0/0 flowid 1:3

    sleep 30

    # Remove latency
    docker exec "$service" tc qdisc del dev eth0 root
    echo "✅ Network latency test completed for $service"
}

# Function to simulate service failure
simulate_service_failure() {
    local service=$1
    echo "💥 Simulating service failure for $service..."

    docker-compose stop "$service"
    sleep 30
    docker-compose start "$service"

    # Wait for service to be ready
    sleep 10
    echo "✅ Service failure test completed for $service"
}

# Function to check system health during tests
check_health() {
    echo "🏥 Checking system health..."

    # Check if other services are still responding
    services=("user-service:8081" "order-service:8082" "product-service:8083")

    for service in "${services[@]}"; do
        IFS=':' read -r name port <<< "$service"
        if curl -f "http://localhost:${port}/health" > /dev/null 2>&1; then
            echo "✅ $name is healthy"
        else
            echo "❌ $name is not responding"
        fi
    done

    # Check error rates in Prometheus
    error_rate=$(curl -s 'http://localhost:9090/api/v1/query?query=rate(http_requests_total{status_code=~"5.."}[1m])' | jq -r '.data.result[0].value[1] // "0"')
    echo "📊 Current error rate: $error_rate/sec"
}

# Run chaos tests
services=("user-service" "order-service")

for service in "${services[@]}"; do
    echo ""
    echo "🎯 Testing resilience of $service"
    echo "================================"

    # Baseline health check
    check_health

    # CPU load test
    simulate_cpu_load "$service"
    check_health

    # Memory pressure test
    simulate_memory_pressure "$service"
    check_health

    # Network latency test (commented out as it requires special privileges)
    # simulate_network_latency "$service"
    # check_health

    # Service failure test
    simulate_service_failure "$service"
    check_health

    echo "✅ Chaos tests completed for $service"
done

echo ""
echo "🎉 All chaos tests completed!"
echo "Check Grafana dashboards for detailed metrics during the tests."
```

## Récapitulatif et bonnes pratiques

### Checklist de mise en production

```markdown
# Checklist - Observabilité en Production

## Logging
- [ ] Logs structurés en JSON
- [ ] Niveaux de logs appropriés
- [ ] Request ID pour tracer les requêtes
- [ ] Rotation et rétention des logs configurées
- [ ] Logs centralisés (ELK/Loki)
- [ ] Sampling pour les logs haute fréquence

## Métriques
- [ ] Golden Signals implémentées (Latency, Traffic, Errors, Saturation)
- [ ] Métriques métier définies
- [ ] SLI/SLO configurés
- [ ] Dashboards Grafana créés
- [ ] Alertes critiques configurées
- [ ] Health checks pour tous les services

## Tracing
- [ ] Tracing distribué activé
- [ ] Sampling configuré (10% en production)
- [ ] Spans avec tags appropriés
- [ ] Corrélation avec logs et métriques

## Alerting
- [ ] Alertes basées sur SLO
- [ ] Runbooks pour chaque alerte
- [ ] Escalation configurée
- [ ] Tests des notifications
- [ ] Inhibition des alertes redondantes

## Sécurité
- [ ] Authentification pour les dashboards
- [ ] Pas de données sensibles dans les logs
- [ ] Chiffrement des communications
- [ ] Accès restreint aux métriques
- [ ] Audit des accès aux outils de monitoring

## Performance
- [ ] Overhead de monitoring < 5%
- [ ] Rétention des données optimisée
- [ ] Compression activée
- [ ] Indices Elasticsearch optimisés
- [ ] Queries Prometheus optimisées

## Backup et DR
- [ ] Sauvegarde des configurations
- [ ] Sauvegarde des dashboards
- [ ] Sauvegarde des données historiques
- [ ] Procédure de restauration testée
- [ ] Monitoring multi-région si applicable
```

### Guide de troubleshooting

```markdown
# Guide de Troubleshooting - Observabilité

## Problèmes courants et solutions

### 1. Métriques manquantes

**Symptômes:**
- Dashboards vides
- Alertes ne se déclenchent pas
- Données manquantes dans Prometheus

**Diagnostic:**
```bash
# Vérifier que Prometheus scrape les services
curl http://localhost:9090/api/v1/targets

# Vérifier les métriques exposées par le service
curl http://service:8090/metrics

# Vérifier les logs de Prometheus
docker logs prometheus-container
```

**Solutions:**
- Vérifier la configuration de scraping dans prometheus.yml
- S'assurer que les services exposent /metrics sur le bon port
- Vérifier la connectivité réseau entre Prometheus et les services
- Valider les labels dans les métriques

### 2. Alertes qui ne se déclenchent pas

**Diagnostic:**
```bash
# Vérifier les règles d'alerte
curl http://localhost:9090/api/v1/rules

# Tester une requête d'alerte manuellement
curl 'http://localhost:9090/api/v1/query?query=up==0'

# Vérifier Alertmanager
curl http://localhost:9093/api/v1/alerts
```

**Solutions:**
- Valider la syntaxe des règles d'alerte
- Vérifier que les métriques utilisées existent
- S'assurer qu'Alertmanager reçoit les alertes
- Vérifier la configuration de routage dans Alertmanager

### 3. Logs non visibles dans Kibana

**Diagnostic:**
```bash
# Vérifier Filebeat
docker logs filebeat-container

# Vérifier Logstash
docker logs logstash-container

# Vérifier les indices Elasticsearch
curl http://localhost:9200/_cat/indices
```

**Solutions:**
- Vérifier la configuration Filebeat
- S'assurer que Logstash parse correctement les logs
- Créer les index patterns dans Kibana
- Vérifier les permissions sur les fichiers de logs
```

## Optimisations de performance

### Configuration Prometheus optimisée

```yaml
# prometheus.yml optimisé pour la production
global:
  scrape_interval: 15s          # Pas trop fréquent
  evaluation_interval: 15s      # Aligné avec scrape_interval

  # Réduire la mémoire utilisée
  query:
    timeout: 2m
    max_concurrency: 20
    max_samples: 50000000

# Optimisations de rétention
rule_files:
  - "recording_rules/*.yml"     # Utiliser des recording rules

# Configuration de stockage
storage:
  tsdb:
    retention.time: 15d         # Garder 15 jours de données brutes
    retention.size: 50GB        # Limite de taille
    min-block-duration: 2h      # Blocks plus grands = moins d'overhead
    max-block-duration: 25h
```

### Recording rules pour optimiser les queries

```yaml
# monitoring/rules/recording_rules.yml
groups:
  - name: instance_down
    interval: 30s
    rules:
      # Pre-calculer les métriques couramment utilisées
      - record: instance:up
        expr: up

      - record: job:up:avg
        expr: avg by (job)(up)

  - name: http_requests
    interval: 30s
    rules:
      # Taux de requêtes par service (5min)
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job, service)

      # Taux d'erreur par service (5min)
      - record: job:http_requests_errors:rate5m
        expr: sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (job, service)

      # Disponibilité par service
      - record: job:availability:ratio5m
        expr: |
          (
            job:http_requests:rate5m - job:http_requests_errors:rate5m
          ) / job:http_requests:rate5m

  - name: latency
    interval: 30s
    rules:
      # Percentiles de latence pré-calculés
      - record: job:http_request_duration:p50
        expr: histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, service, le))

      - record: job:http_request_duration:p95
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, service, le))

      - record: job:http_request_duration:p99
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, service, le))
```

### Configuration Elasticsearch optimisée

```yaml
# elasticsearch.yml
cluster.name: microservices-logs
node.name: es-node-1

# Optimisations mémoire
bootstrap.memory_lock: true
indices.memory.index_buffer_size: 30%

# Optimisations de performance
index.refresh_interval: 30s
index.number_of_replicas: 0  # Pas de réplication en dev
index.merge.policy.max_merged_segment: 5gb

# Template pour les indices de logs
PUT _index_template/microservices-logs
{
  "index_patterns": ["microservices-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "30s",
      "codec": "best_compression",
      "lifecycle": {
        "name": "microservices-logs-policy",
        "rollover_alias": "microservices-logs"
      }
    },
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "service": {"type": "keyword"},
        "message": {
          "type": "text",
          "fields": {
            "keyword": {"type": "keyword", "ignore_above": 256}
          }
        }
      }
    }
  }
}
```

## Monitoring multi-environnement

### Configuration par environnement

```go
package config

type Environment string

const (
    Development Environment = "development"
    Staging    Environment = "staging"
    Production Environment = "production"
)

type ObservabilityConfig struct {
    Environment Environment

    // Logging
    LogLevel    string
    LogSampling float64

    // Metrics
    MetricsRetention string
    ScrapeInterval   string

    // Tracing
    TracingSampler float64

    // Alerting
    AlertsEnabled bool
}

func GetObservabilityConfig(env Environment) *ObservabilityConfig {
    switch env {
    case Development:
        return &ObservabilityConfig{
            Environment:      Development,
            LogLevel:         "debug",
            LogSampling:      1.0,    // 100% sampling en dev
            MetricsRetention: "7d",
            ScrapeInterval:   "5s",
            TracingSampler:   1.0,    // 100% tracing en dev
            AlertsEnabled:    false,  // Pas d'alertes en dev
        }
    case Staging:
        return &ObservabilityConfig{
            Environment:      Staging,
            LogLevel:         "info",
            LogSampling:      0.5,    // 50% sampling
            MetricsRetention: "15d",
            ScrapeInterval:   "10s",
            TracingSampler:   0.5,    // 50% tracing
            AlertsEnabled:    true,   // Alertes activées mais moins sensibles
        }
    case Production:
        return &ObservabilityConfig{
            Environment:      Production,
            LogLevel:         "info",
            LogSampling:      0.1,    // 10% sampling pour éviter le spam
            MetricsRetention: "30d",
            ScrapeInterval:   "15s",
            TracingSampler:   0.1,    // 10% tracing pour minimiser l'overhead
            AlertsEnabled:    true,
        }
    default:
        return GetObservabilityConfig(Production) // Default sécurisé
    }
}
```

### Script de déploiement multi-env

```bash
#!/bin/bash

# deploy-monitoring.sh
set -e

ENVIRONMENT=${1:-development}
CONFIG_DIR="monitoring/${ENVIRONMENT}"

echo "🚀 Deploying monitoring stack for environment: $ENVIRONMENT"

# Validation de l'environnement
case $ENVIRONMENT in
    development|staging|production)
        echo "✅ Valid environment: $ENVIRONMENT"
        ;;
    *)
        echo "❌ Invalid environment. Use: development, staging, or production"
        exit 1
        ;;
esac

# Créer les répertoires si nécessaire
mkdir -p "$CONFIG_DIR"/{prometheus,grafana,alertmanager}

# Copier les configurations de base
cp monitoring/templates/* "$CONFIG_DIR/"

# Appliquer les configurations spécifiques à l'environnement
case $ENVIRONMENT in
    development)
        # Configurations de développement
        sed -i 's/scrape_interval: 15s/scrape_interval: 5s/' "$CONFIG_DIR/prometheus.yml"
        sed -i 's/retention.time: 15d/retention.time: 7d/' "$CONFIG_DIR/prometheus.yml"

        # Désactiver les alertes critiques en dev
        echo "# Alerts disabled in development" > "$CONFIG_DIR/alertmanager.yml"
        ;;

    staging)
        # Configurations de staging
        sed -i 's/scrape_interval: 15s/scrape_interval: 10s/' "$CONFIG_DIR/prometheus.yml"

        # Alertes moins sensibles
        sed -i 's/for: 2m/for: 5m/' "$CONFIG_DIR/alert_rules.yml"
        ;;

    production)
        # Configurations de production
        echo "✅ Using production defaults"

        # Valider les configurations critiques
        if ! docker run --rm -v "$(pwd)/$CONFIG_DIR/prometheus.yml":/etc/prometheus/prometheus.yml prom/prometheus:latest promtool check config /etc/prometheus/prometheus.yml; then
            echo "❌ Invalid Prometheus configuration"
            exit 1
        fi
        ;;
esac

# Déployer avec Docker Compose
export MONITORING_ENV=$ENVIRONMENT
docker-compose -f docker-compose.monitoring.yml --env-file "environments/${ENVIRONMENT}.env" up -d

echo "✅ Monitoring stack deployed for $ENVIRONMENT"

# Afficher les endpoints
echo ""
echo "📊 Monitoring endpoints:"
echo "Prometheus: http://localhost:9090"
echo "Grafana: http://localhost:3000"
echo "Alertmanager: http://localhost:9093"
echo "Jaeger: http://localhost:16686"
```

## Tests d'intégration pour l'observabilité

### Tests automatisés

```go
package observability_test

import (
    "context"
    "fmt"
    "net/http"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestObservabilityStack(t *testing.T) {
    // Test que tous les services de monitoring sont disponibles
    endpoints := map[string]string{
        "prometheus":   "http://localhost:9090/-/healthy",
        "grafana":      "http://localhost:3000/api/health",
        "alertmanager": "http://localhost:9093/-/healthy",
        "jaeger":       "http://localhost:16686/",
    }

    for service, url := range endpoints {
        t.Run(fmt.Sprintf("test_%s_health", service), func(t *testing.T) {
            resp, err := http.Get(url)
            require.NoError(t, err)
            defer resp.Body.Close()

            assert.Equal(t, http.StatusOK, resp.StatusCode,
                "Service %s should be healthy", service)
        })
    }
}

func TestMetricsCollection(t *testing.T) {
    // Attendre que les métriques soient collectées
    time.Sleep(30 * time.Second)

    // Tester que Prometheus collecte les métriques
    resp, err := http.Get("http://localhost:9090/api/v1/query?query=up")
    require.NoError(t, err)
    defer resp.Body.Close()

    assert.Equal(t, http.StatusOK, resp.StatusCode)

    // TODO: Parser la réponse JSON et vérifier les métriques
}

func TestAlertsConfiguration(t *testing.T) {
    // Tester que les règles d'alerte sont chargées
    resp, err := http.Get("http://localhost:9090/api/v1/rules")
    require.NoError(t, err)
    defer resp.Body.Close()

    assert.Equal(t, http.StatusOK, resp.StatusCode)

    // TODO: Vérifier que les règles critiques sont présentes
}

func TestLogIngestion(t *testing.T) {
    // Vérifier que les logs sont ingérés dans Elasticsearch
    time.Sleep(60 * time.Second) // Attendre l'ingestion

    resp, err := http.Get("http://localhost:9200/_cat/indices/microservices-logs-*")
    require.NoError(t, err)
    defer resp.Body.Close()

    assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

### Tests de charge pour l'observabilité

```go
package loadtest

import (
    "context"
    "fmt"
    "sync"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
)

func TestObservabilityUnderLoad(t *testing.T) {
    const (
        numGoroutines = 100
        duration     = 5 * time.Minute
        requestsPerSecond = 1000
    )

    ctx, cancel := context.WithTimeout(context.Background(), duration)
    defer cancel()

    var wg sync.WaitGroup
    errorCount := int64(0)

    // Générer de la charge sur les services
    for i := 0; i < numGoroutines; i++ {
        wg.Add(1)
        go func(goroutineID int) {
            defer wg.Done()

            ticker := time.NewTicker(time.Second / requestsPerSecond * numGoroutines)
            defer ticker.Stop()

            for {
                select {
                case <-ctx.Done():
                    return
                case <-ticker.C:
                    // Faire une requête à un service
                    if err := makeTestRequest(); err != nil {
                        atomic.AddInt64(&errorCount, 1)
                    }
                }
            }
        }(i)
    }

    // Monitorer les performances pendant le test
    go func() {
        ticker := time.NewTicker(30 * time.Second)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                checkObservabilityPerformance(t)
            }
        }
    }()

    wg.Wait()

    // Vérifier que l'observabilité a tenu le coup
    assert.Less(t, errorCount, int64(100), "Too many errors during load test")

    // Vérifier que Prometheus est toujours responsive
    resp, err := http.Get("http://localhost:9090/-/healthy")
    assert.NoError(t, err)
    if resp != nil {
        resp.Body.Close()
        assert.Equal(t, http.StatusOK, resp.StatusCode)
    }
}

func checkObservabilityPerformance(t *testing.T) {
    // Vérifier la latence des queries Prometheus
    start := time.Now()
    resp, err := http.Get("http://localhost:9090/api/v1/query?query=up")
    if err == nil && resp != nil {
        resp.Body.Close()
        latency := time.Since(start)
        assert.Less(t, latency, 2*time.Second, "Prometheus query too slow")
    }

    // Vérifier que Grafana répond
    resp, err = http.Get("http://localhost:3000/api/health")
    if err == nil && resp != nil {
        resp.Body.Close()
        assert.Equal(t, http.StatusOK, resp.StatusCode)
    }
}
```

## Documentation et formation

### Guide de démarrage rapide

```markdown
# Guide de démarrage rapide - Observabilité

## 🚀 Démarrage en 5 minutes

### 1. Cloner et configurer
```bash
git clone https://github.com/company/microservices-observability
cd microservices-observability
cp .env.example .env
```

### 2. Démarrer la stack
```bash
./scripts/monitoring-setup.sh
docker-compose up -d
```

### 3. Accéder aux interfaces
- **Grafana**: http://localhost:3000 (admin/admin123)
- **Prometheus**: http://localhost:9090
- **Jaeger**: http://localhost:16686
- **Kibana**: http://localhost:5601

### 4. Vérifier le fonctionnement
```bash
./scripts/health-check.sh
```

## 📊 Premier dashboard

1. Ouvrir Grafana
2. Aller dans "Dashboards" > "Import"
3. Importer le dashboard ID: 12345 (Microservices Overview)
4. Configurer la source de données Prometheus

## 🔔 Première alerte

1. Vérifier dans Prometheus que les métriques arrivent
2. Aller dans Alertmanager: http://localhost:9093
3. Les alertes de test devraient apparaître sous peu

## 🔍 Premier trace

1. Faire quelques requêtes à vos services
2. Ouvrir Jaeger: http://localhost:16686
3. Chercher les traces par service name

## ❓ Besoin d'aide?

- Documentation complète: [docs/](./docs/)
- Runbooks: [docs/runbooks/](./docs/runbooks/)
- Slack: #monitoring-help
```

### Formation équipe

```markdown
# Formation Observabilité - Équipe de développement

## 📚 Parcours de formation (4 heures)

### Module 1: Concepts (30 min)
- Les 3 piliers de l'observabilité
- Différence monitoring vs observabilité
- SLI/SLO/Error Budget

### Module 2: Logging (45 min)
- Logs structurés en Go
- Niveaux de logs appropriés
- Corrélation avec Request ID
- **Exercice**: Ajouter des logs à un service existant

### Module 3: Métriques (60 min)
- Golden Signals
- Types de métriques Prometheus
- Création de métriques métier
- **Exercice**: Implémenter des métriques custom

### Module 4: Dashboards (45 min)
- Navigation dans Grafana
- Création de dashboards
- Best practices de visualisation
- **Exercice**: Créer un dashboard pour son équipe

### Module 5: Alerting (45 min)
- Quand alerter vs quand monitorer
- Écriture de bonnes alertes
- Runbooks efficaces
- **Exercice**: Configurer une alerte critique

### Module 6: Troubleshooting (30 min)
- Méthodologie de debug
- Utilisation du tracing distribué
- Corrélation logs/métriques/traces
- **Exercice**: Résoudre un incident simulé

### Module 7: Production (15 min)
- Checklist go-live
- Bonnes pratiques
- Optimisations performance

## 🎯 Objectifs d'apprentissage

À la fin de cette formation, vous devriez être capable de:
- [ ] Ajouter de l'observabilité à un nouveau service
- [ ] Créer des dashboards pertinents
- [ ] Configurer des alertes appropriées
- [ ] Debugger un problème en production
- [ ] Optimiser les performances d'observabilité

## 📋 Certification

Quiz final de 20 questions couvrant tous les modules.
Score minimum: 80% pour la certification.
```

## Évolution et roadmap

### Roadmap observabilité

```markdown
# Roadmap Observabilité 2024-2025

## Q4 2024 - Fondations ✅
- [x] Stack monitoring de base (Prometheus, Grafana, Jaeger)
- [x] Logging centralisé (ELK)
- [x] Alerting critique
- [x] Dashboards service-level

## Q1 2025 - Maturité 🔄
- [ ] SLO/Error Budget automatisés
- [ ] Alerting intelligent (ML-based)
- [ ] Tracing performance optimisé
- [ ] Dashboards business-level

## Q2 2025 - Scale 📈
- [ ] Multi-cluster monitoring
- [ ] Fédération Prometheus
- [ ] Long-term storage (Thanos)
- [ ] Monitoring infrastructure (Kubernetes)

## Q3 2025 - Intelligence 🧠
- [ ] Détection d'anomalies automatique
- [ ] Root cause analysis automatisée
- [ ] Prédiction de pannes
- [ ] Auto-remediation pour problèmes courants

## Q4 2025 - Excellence ⭐
- [ ] Observabilité dans CI/CD
- [ ] Chaos engineering intégré
- [ ] Observabilité security (SIEM)
- [ ] Cost optimization automatique
```

### Métriques d'adoption

```go
package adoption

// Métriques pour mesurer l'adoption de l'observabilité
var (
    ServicesWithObservability = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "observability_adoption_services_total",
            Help: "Number of services with observability implemented",
        },
        []string{"team", "observability_type"}, // logs, metrics, traces
    )

    DashboardUsage = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "observability_dashboard_views_total",
            Help: "Number of dashboard views",
        },
        []string{"dashboard", "user", "team"},
    )

    AlertsActionable = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "observability_alerts_actionable_ratio",
            Help: "Ratio of alerts that led to action",
        },
        []string{"team", "severity"},
    )

    MTTR = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "observability_mttr_seconds",
            Help: "Mean Time To Resolution for incidents",
            Buckets: []float64{60, 300, 900, 1800, 3600, 7200}, // 1min à 2h
        },
        []string{"team", "severity"},
    )
)

// Tableau de bord pour mesurer le ROI de l'observabilité
type ObservabilityROI struct {
    // Temps économisé en debugging
    DebuggingTimeSaved time.Duration

    // Incidents évités grâce à l'alerting proactif
    IncidentsAvoided int

    // Réduction du MTTR
    MTTRImprovement time.Duration

    // Satisfaction développeur (1-10)
    DeveloperSatisfaction float64

    // Coût de l'infrastructure monitoring
    InfrastructureCost float64
}
```

Ce guide complet de l'observabilité vous donne toutes les clés pour implémenter, maintenir et faire évoluer un système de monitoring robuste pour vos microservices. L'observabilité n'est pas juste un outil technique, c'est un changement de culture qui permet de construire des systèmes plus fiables et de réagir plus rapidement aux problèmes.

## Récapitulatif des concepts clés

1. **Les 3 piliers** : Logs, Métriques, Traces
2. **Golden Signals** : Latency, Traffic, Errors, Saturation
3. **SLI/SLO** : Mesurer et garantir la qualité de service
4. **Alerting intelligent** : Alerter sur ce qui compte vraiment
5. **Automation** : Réduire le travail manuel et les erreurs humaines

La prochaine étape est la mise en pratique ! Commencez par implémenter l'observabilité sur un service pilote, puis étendez progressivement à tous vos services.

⏭️
