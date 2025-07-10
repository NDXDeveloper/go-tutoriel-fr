🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13. Programmation réseau en Go

## Introduction

La programmation réseau est l'un des domaines où Go excelle particulièrement. Conçu à l'origine par Google pour développer des services distribués à grande échelle, Go offre des outils puissants et intuitifs pour créer des applications réseau robustes et performantes.

## Pourquoi Go pour la programmation réseau ?

### 1. Performance et simplicité
Go combine la performance des langages compilés avec la simplicité d'utilisation des langages de haut niveau. Sa gestion native de la concurrence via les goroutines permet de traiter facilement des milliers de connexions simultanées.

### 2. Bibliothèque standard riche
La bibliothèque standard de Go inclut des packages complets pour :
- HTTP/HTTPS (`net/http`)
- TCP/UDP (`net`)
- JSON (`encoding/json`)
- URL (`net/url`)
- Templates (`html/template`, `text/template`)

### 3. Gestion native de la concurrence
Les goroutines et channels permettent de créer des serveurs hautement concurrents sans la complexité des threads traditionnels.

## Concepts fondamentaux

### Architecture client-serveur
En programmation réseau, nous distinguons généralement :
- **Client** : Initie la communication et envoie des requêtes
- **Serveur** : Écoute les connexions et traite les requêtes

### Protocoles réseau principaux

#### TCP (Transmission Control Protocol)
- Protocole fiable avec garantie de livraison
- Connexion établie (handshake)
- Idéal pour HTTP, HTTPS, bases de données

#### UDP (User Datagram Protocol)
- Protocole sans connexion, plus rapide
- Pas de garantie de livraison
- Idéal pour jeux, streaming, DNS

#### HTTP/HTTPS
- Protocole de couche application basé sur TCP
- Stateless par nature
- Standard pour les API REST et applications web

## Packages Go essentiels

### `net` - Package de base
```go
import "net"
```
- Fonctions de bas niveau pour TCP/UDP
- Gestion des adresses IP et ports
- Création de listeners et connexions

### `net/http` - HTTP client et serveur
```go
import "net/http"
```
- Serveur HTTP intégré
- Client HTTP avec support des cookies, redirections
- Middleware et routage

### `net/url` - Manipulation d'URLs
```go
import "net/url"
```
- Parsing et construction d'URLs
- Gestion des paramètres de requête
- Encodage/décodage URL

### `encoding/json` - Sérialisation JSON
```go
import "encoding/json"
```
- Marshalling/Unmarshalling
- Tags struct pour personnalisation
- Streaming JSON

## Exemple introductif : Serveur TCP basique

Voici un exemple simple d'un serveur TCP qui illustre les concepts de base :

```go
package main

import (
    "fmt"
    "net"
    "bufio"
    "strings"
)

func main() {
    // Écouter sur le port 8080
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        fmt.Printf("Erreur lors de l'écoute: %v\n", err)
        return
    }
    defer listener.Close()

    fmt.Println("Serveur TCP en écoute sur le port 8080...")

    for {
        // Accepter une connexion
        conn, err := listener.Accept()
        if err != nil {
            fmt.Printf("Erreur lors de l'acceptation: %v\n", err)
            continue
        }

        // Traiter la connexion dans une goroutine séparée
        go handleConnection(conn)
    }
}

func handleConnection(conn net.Conn) {
    defer conn.Close()

    scanner := bufio.NewScanner(conn)
    for scanner.Scan() {
        message := scanner.Text()
        response := fmt.Sprintf("Echo: %s\n", strings.ToUpper(message))
        conn.Write([]byte(response))
    }
}
```

## Gestion des erreurs réseau

La programmation réseau implique de nombreuses sources d'erreurs potentielles :

### Erreurs courantes
- **Timeout** : Connexion trop lente
- **Connection refused** : Port fermé ou service indisponible
- **Network unreachable** : Problème de routage
- **DNS resolution** : Nom d'hôte invalide

### Bonnes pratiques
```go
// Toujours vérifier les erreurs
conn, err := net.Dial("tcp", "example.com:80")
if err != nil {
    // Gérer l'erreur spécifiquement
    if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
        fmt.Println("Timeout de connexion")
    }
    return
}
defer conn.Close()

// Définir des timeouts appropriés
conn.SetDeadline(time.Now().Add(30 * time.Second))
```

## Sécurité réseau

### Considérations importantes
- **Validation d'entrée** : Toujours valider les données reçues
- **Limites de taille** : Éviter les attaques par déni de service
- **Chiffrement** : Utiliser TLS pour les données sensibles
- **Authentification** : Vérifier l'identité des clients

### Exemple de validation basique
```go
func validateInput(input string) error {
    if len(input) > 1024 {
        return fmt.Errorf("entrée trop longue")
    }

    if strings.Contains(input, "<script>") {
        return fmt.Errorf("contenu potentiellement dangereux")
    }

    return nil
}
```

## Concurrence et performances

### Goroutines pour la scalabilité
```go
// Serveur concurrent gérant plusieurs connexions
func startServer() {
    listener, _ := net.Listen("tcp", ":8080")
    defer listener.Close()

    for {
        conn, err := listener.Accept()
        if err != nil {
            continue
        }

        // Chaque connexion dans sa propre goroutine
        go func(c net.Conn) {
            defer c.Close()
            handleClient(c)
        }(conn)
    }
}
```

### Limitation des ressources
```go
// Limiter le nombre de connexions simultanées
const maxConnections = 100
semaphore := make(chan struct{}, maxConnections)

func handleWithLimit(conn net.Conn) {
    semaphore <- struct{}{} // Acquérir
    defer func() { <-semaphore }() // Libérer

    // Traiter la connexion
    handleConnection(conn)
}
```

## Outils de développement

### Testing réseau
```go
// Test d'un serveur HTTP
func TestServer(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(myHandler))
    defer server.Close()

    resp, err := http.Get(server.URL)
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    // Vérifier la réponse
    if resp.StatusCode != http.StatusOK {
        t.Errorf("Expected 200, got %d", resp.StatusCode)
    }
}
```

### Debugging réseau
- **Logs détaillés** : Tracer les requêtes et réponses
- **Monitoring** : Surveiller les connexions actives
- **Profiling** : Analyser les performances réseau

## Prochaines étapes

Dans les sections suivantes, nous explorerons en détail :

1. **HTTP client et serveur** : Création d'applications web robustes
2. **JSON et sérialisation** : Manipulation de données structurées
3. **APIs REST** : Développement d'interfaces de programmation
4. **WebSockets** : Communication bidirectionnelle en temps réel

Chaque section inclura des exemples pratiques et des patterns courants utilisés dans les applications Go modernes.

---

*Cette introduction pose les bases nécessaires pour comprendre la programmation réseau en Go. Les concepts présentés ici seront approfondis dans les sections suivantes avec des exemples concrets et des cas d'usage réels.*

⏭️
