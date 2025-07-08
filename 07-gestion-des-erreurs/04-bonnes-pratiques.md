🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7-4 : Bonnes pratiques

## Introduction

Maintenant que nous avons vu les conventions d'erreur, les erreurs personnalisées et les mécanismes panic/recover, cette section synthétise les bonnes pratiques pour une gestion d'erreur efficace et professionnelle en Go.

## Principe fondamental : Fail Fast, Fail Clear

La philosophie Go pour les erreurs suit le principe **"Fail Fast, Fail Clear"** :
- **Fail Fast** : détecter les erreurs le plus tôt possible
- **Fail Clear** : fournir des messages d'erreur clairs et actionnables

```go
// ✅ Bon - détection rapide avec message clair
func traiterCommande(cmd *Commande) error {
    if cmd == nil {
        return errors.New("commande ne peut pas être nil")
    }

    if cmd.ID == "" {
        return errors.New("ID de commande requis")
    }

    if cmd.Montant <= 0 {
        return fmt.Errorf("montant invalide: %.2f (doit être > 0)", cmd.Montant)
    }

    // Traitement...
    return nil
}
```

## 1. Toujours vérifier les erreurs

### ✅ La règle d'or

```go
// ✅ Toujours vérifier immédiatement
fichier, err := os.Open("config.txt")
if err != nil {
    return fmt.Errorf("impossible d'ouvrir le fichier de configuration: %w", err)
}
defer fichier.Close()

// ❌ Ne JAMAIS ignorer une erreur
fichier, _ := os.Open("config.txt") // Dangereux!
```

### Gestion du cas "erreur ignorée intentionnellement"

```go
// Si vous devez vraiment ignorer une erreur, commentez pourquoi
_, err := fmt.Fprintf(os.Stderr, "Message de debug: %s\n", msg)
_ = err // Ignoré intentionnellement - erreur non critique pour stderr
```

## 2. Messages d'erreur efficaces

### Structure des messages

Les bons messages d'erreur suivent cette structure :
**[Contexte] [Action] [Cause] [Solution optionnelle]**

```go
// ✅ Excellent message d'erreur
return fmt.Errorf("connexion à la base de données 'users' échouée sur %s:%d: %w (vérifiez que le service est démarré)",
    host, port, err)

// ❌ Message trop vague
return errors.New("erreur de connexion")
```

### Conventions de formatage

```go
// ✅ Bonnes pratiques
return errors.New("fichier de configuration introuvable")           // minuscules
return errors.New("timeout de connexion dépassé")                   // pas de ponctuation
return fmt.Errorf("utilisateur %q non trouvé", username)           // guillemets pour les valeurs
return fmt.Errorf("age invalide: %d (doit être entre 0 et 150)", age) // contexte des limites

// ❌ À éviter
return errors.New("Fichier de configuration introuvable.")          // majuscule + point
return errors.New("ERREUR!")                                        // trop dramatique
return errors.New("ça ne marche pas")                              // trop vague
```

## 3. Gestion des erreurs en cascade

### Enrobage d'erreurs (Wrapping)

```go
// ✅ Bon enrobage avec contexte
func lireConfiguration() (*Config, error) {
    data, err := os.ReadFile("app.config")
    if err != nil {
        return nil, fmt.Errorf("lecture de la configuration: %w", err)
    }

    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("parsing de la configuration: %w", err)
    }

    return &config, nil
}

// Utilisation avec unwrapping
func main() {
    _, err := lireConfiguration()
    if err != nil {
        // Vérifier l'erreur racine
        if errors.Is(err, os.ErrNotExist) {
            log.Println("Fichier de config manquant, utilisation des valeurs par défaut")
            return
        }
        log.Fatal("Erreur critique:", err)
    }
}
```

### Éviter la sur-enrobement

```go
// ❌ Sur-enrobage - messages répétitifs
func niveau1() error {
    if err := niveau2(); err != nil {
        return fmt.Errorf("erreur niveau1: %w", err)
    }
    return nil
}

func niveau2() error {
    if err := niveau3(); err != nil {
        return fmt.Errorf("erreur niveau2: %w", err)
    }
    return nil
}

// ✅ Enrobage sélectif avec contexte utile
func niveau1() error {
    if err := niveau2(); err != nil {
        return fmt.Errorf("échec de l'initialisation du service: %w", err)
    }
    return nil
}

func niveau2() error {
    return niveau3() // Passer directement si pas de contexte à ajouter
}
```

## 4. Stratégies de gestion d'erreur

### Pattern : Early Return

```go
// ✅ Excellent - early return pour une lisibilité maximale
func traiterUtilisateur(id string) (*Utilisateur, error) {
    if id == "" {
        return nil, errors.New("ID utilisateur requis")
    }

    utilisateur, err := db.TrouverUtilisateur(id)
    if err != nil {
        return nil, fmt.Errorf("recherche utilisateur %q: %w", id, err)
    }

    if utilisateur.Actif == false {
        return nil, fmt.Errorf("utilisateur %q est désactivé", id)
    }

    if err := validerUtilisateur(utilisateur); err != nil {
        return nil, fmt.Errorf("validation utilisateur %q: %w", id, err)
    }

    return utilisateur, nil
}

// ❌ Difficile à lire - imbrication excessive
func traiterUtilisateurMauvais(id string) (*Utilisateur, error) {
    if id != "" {
        utilisateur, err := db.TrouverUtilisateur(id)
        if err == nil {
            if utilisateur.Actif {
                if err := validerUtilisateur(utilisateur); err == nil {
                    return utilisateur, nil
                } else {
                    return nil, err
                }
            } else {
                return nil, fmt.Errorf("utilisateur désactivé")
            }
        } else {
            return nil, err
        }
    } else {
        return nil, errors.New("ID requis")
    }
}
```

### Pattern : Error Accumulation

```go
// Pour collecter plusieurs erreurs
type ErrorList []error

func (e ErrorList) Error() string {
    var messages []string
    for _, err := range e {
        messages = append(messages, err.Error())
    }
    return strings.Join(messages, "; ")
}

func validerFormulaire(form *Formulaire) error {
    var erreurs ErrorList

    if form.Nom == "" {
        erreurs = append(erreurs, errors.New("nom requis"))
    }

    if form.Email == "" {
        erreurs = append(erreurs, errors.New("email requis"))
    } else if !strings.Contains(form.Email, "@") {
        erreurs = append(erreurs, errors.New("format email invalide"))
    }

    if form.Age < 0 || form.Age > 150 {
        erreurs = append(erreurs, fmt.Errorf("âge invalide: %d", form.Age))
    }

    if len(erreurs) > 0 {
        return erreurs
    }

    return nil
}
```

## 5. Tests et erreurs

### Tester les cas d'erreur

```go
func TestDivision(t *testing.T) {
    tests := []struct {
        name        string
        a, b        float64
        expected    float64
        expectError bool
        errorMsg    string
    }{
        {
            name:        "division normale",
            a:           10, b: 2,
            expected:    5,
            expectError: false,
        },
        {
            name:        "division par zéro",
            a:           10, b: 0,
            expectError: true,
            errorMsg:    "division par zéro",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := diviser(tt.a, tt.b)

            if tt.expectError {
                if err == nil {
                    t.Errorf("attendait une erreur mais n'en a pas reçu")
                }
                if !strings.Contains(err.Error(), tt.errorMsg) {
                    t.Errorf("message d'erreur incorrect. Attendu: %q, Reçu: %q",
                        tt.errorMsg, err.Error())
                }
            } else {
                if err != nil {
                    t.Errorf("erreur inattendue: %v", err)
                }
                if result != tt.expected {
                    t.Errorf("résultat incorrect. Attendu: %f, Reçu: %f",
                        tt.expected, result)
                }
            }
        })
    }
}
```

## 6. Logging et monitoring

### Niveaux de logging appropriés

```go
import "log/slog"

func traiterRequete(req *http.Request) error {
    // Debug - informations de développement
    slog.Debug("traitement de la requête", "path", req.URL.Path, "method", req.Method)

    utilisateur, err := authentifier(req)
    if err != nil {
        // Warn - erreur récupérable
        slog.Warn("échec d'authentification", "error", err, "ip", req.RemoteAddr)
        return fmt.Errorf("authentification échouée: %w", err)
    }

    if err := autoriser(utilisateur, req.URL.Path); err != nil {
        // Warn - tentative d'accès non autorisé
        slog.Warn("accès refusé", "user", utilisateur.ID, "path", req.URL.Path)
        return fmt.Errorf("accès non autorisé: %w", err)
    }

    if err := traiterLogique(req); err != nil {
        // Error - erreur métier importante
        slog.Error("erreur de traitement", "error", err, "user", utilisateur.ID)
        return fmt.Errorf("traitement échoué: %w", err)
    }

    // Info - opération réussie
    slog.Info("requête traitée avec succès", "user", utilisateur.ID, "path", req.URL.Path)
    return nil
}
```

### Métriques d'erreur

```go
// Structure pour tracker les métriques d'erreur
type ErrorMetrics struct {
    mu     sync.RWMutex
    counts map[string]int
}

func (em *ErrorMetrics) Record(errorType string) {
    em.mu.Lock()
    defer em.mu.Unlock()
    em.counts[errorType]++
}

func (em *ErrorMetrics) Get() map[string]int {
    em.mu.RLock()
    defer em.mu.RUnlock()

    result := make(map[string]int)
    for k, v := range em.counts {
        result[k] = v
    }
    return result
}

// Usage dans l'application
var metrics = &ErrorMetrics{counts: make(map[string]int)}

func operationMetier() error {
    if err := quelqueChose(); err != nil {
        metrics.Record("operation_failed")
        return fmt.Errorf("opération échouée: %w", err)
    }
    return nil
}
```

## 7. Documentation des erreurs

### Documenter les erreurs possibles

```go
// TrouverUtilisateur recherche un utilisateur par son ID.
//
// Erreurs possibles:
//   - ErrUtilisateurIntrouvable si l'ID n'existe pas
//   - ErrBaseDeDonneesIndisponible si la connexion échoue
//   - errors.New si l'ID est vide ou invalide
func TrouverUtilisateur(id string) (*Utilisateur, error) {
    if id == "" {
        return nil, errors.New("ID utilisateur requis")
    }

    // Implementation...
}
```

### Erreurs exportées pour les packages

```go
package users

import "errors"

// Erreurs publiques que les utilisateurs du package peuvent tester
var (
    ErrUtilisateurIntrouvable      = errors.New("utilisateur introuvable")
    ErrMotDePasseIncorrect         = errors.New("mot de passe incorrect")
    ErrUtilisateurDejaPresentExist = errors.New("utilisateur déjà existant")
)

// Utilisation
func main() {
    _, err := users.TrouverUtilisateur("123")
    if errors.Is(err, users.ErrUtilisateurIntrouvable) {
        // Traitement spécifique
    }
}
```

## 8. Patterns avancés

### Retry Pattern avec backoff

```go
import (
    "context"
    "time"
    "math/rand"
)

func avecRetry(ctx context.Context, maxRetries int, operation func() error) error {
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        lastErr = operation()
        if lastErr == nil {
            return nil // Succès
        }

        // Vérifier si c'est une erreur récupérable
        if !estErreurRecuperable(lastErr) {
            return fmt.Errorf("erreur non récupérable: %w", lastErr)
        }

        if attempt < maxRetries {
            // Backoff exponentiel avec jitter
            delay := time.Duration(attempt) * time.Second
            jitter := time.Duration(rand.Intn(1000)) * time.Millisecond

            select {
            case <-time.After(delay + jitter):
                // Continue to next attempt
            case <-ctx.Done():
                return fmt.Errorf("opération annulée après %d tentatives: %w", attempt+1, ctx.Err())
            }
        }
    }

    return fmt.Errorf("échec après %d tentatives, dernière erreur: %w", maxRetries+1, lastErr)
}

func estErreurRecuperable(err error) bool {
    // Logique pour déterminer si l'erreur peut être réessayée
    return strings.Contains(err.Error(), "timeout") ||
           strings.Contains(err.Error(), "connection refused")
}
```

### Circuit Breaker Pattern

```go
type CircuitBreaker struct {
    maxFailures int
    resetTime   time.Duration
    failures    int
    lastFailure time.Time
    state       string // "closed", "open", "half-open"
    mu          sync.RWMutex
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.resetTime {
            cb.state = "half-open"
            cb.failures = 0
        } else {
            return errors.New("circuit breaker ouvert")
        }
    }

    err := fn()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()

        if cb.failures >= cb.maxFailures {
            cb.state = "open"
        }
        return fmt.Errorf("opération échouée: %w", err)
    }

    cb.failures = 0
    cb.state = "closed"
    return nil
}
```

## Checklist des bonnes pratiques

### ✅ À faire

- [ ] Toujours vérifier les erreurs immédiatement
- [ ] Utiliser des messages d'erreur descriptifs et actionnables
- [ ] Enrober les erreurs avec du contexte (`fmt.Errorf` avec `%w`)
- [ ] Préférer les erreurs aux panics pour les cas normaux
- [ ] Tester les cas d'erreur dans vos tests unitaires
- [ ] Documenter les erreurs possibles dans vos fonctions publiques
- [ ] Utiliser early return pour éviter l'imbrication
- [ ] Logger les erreurs avec le niveau approprié
- [ ] Créer des erreurs prédéfinies pour les cas courants

### ❌ À éviter

- [ ] Ignorer les erreurs avec `_`
- [ ] Messages d'erreur vagues ("ça ne marche pas")
- [ ] Sur-enrobage d'erreurs sans valeur ajoutée
- [ ] Panic pour des erreurs récupérables
- [ ] Comparaisons de strings pour identifier les erreurs
- [ ] Erreurs sans contexte
- [ ] Imbrication excessive (if dans if dans if...)
- [ ] Logging excessif des mêmes erreurs

## Points clés à retenir

1. **Clarté avant tout** : des messages d'erreur clairs économisent des heures de débogage
2. **Contexte précieux** : chaque niveau doit ajouter du contexte utile
3. **Fail fast** : détecter et signaler les erreurs le plus tôt possible
4. **Testabilité** : toujours tester les chemins d'erreur
5. **Documentation** : documenter les erreurs possibles dans vos APIs
6. **Monitoring** : logger et mesurer les erreurs pour améliorer la robustesse

Une bonne gestion d'erreur est la marque d'un code professionnel et maintenable. Elle facilite le débogage, améliore l'expérience utilisateur et rend le code plus robuste en production.

⏭️
