🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13-4 : WebSockets en Go

## Introduction

Les WebSockets permettent une communication bidirectionnelle en temps réel entre un client et un serveur. Contrairement à HTTP qui suit un modèle requête-réponse, les WebSockets maintiennent une connexion persistante permettant l'échange de messages dans les deux sens à tout moment.

## Qu'est-ce que les WebSockets ?

### Différences avec HTTP classique

**HTTP traditionnel :**
```
Client  →  Requête   →  Serveur
Client  ←  Réponse   ←  Serveur
[Connexion fermée]
```

**WebSocket :**
```
Client  ↔  Messages  ↔  Serveur
[Connexion maintenue]
```

### Cas d'usage des WebSockets
- **Chat en temps réel** : Messages instantanés
- **Jeux multijoueurs** : Synchronisation d'état
- **Notifications push** : Alertes en direct
- **Collaboration** : Édition simultanée de documents
- **Streaming de données** : Cours de bourse, météo
- **Monitoring** : Tableau de bord en temps réel

## Package Gorilla WebSocket

Go ne fournit pas de support WebSocket natif dans sa bibliothèque standard, mais le package `github.com/gorilla/websocket` est la solution de référence.

### Installation

```bash
go mod init websocket-example
go get github.com/gorilla/websocket
```

## Exemple basique : Echo Server

### Serveur WebSocket simple

```go
package main

import (
    "fmt"
    "log"
    "net/http"

    "github.com/gorilla/websocket"
)

// Configurer l'upgrader pour transformer HTTP en WebSocket
var upgrader = websocket.Upgrader{
    // Autoriser les connexions depuis n'importe quelle origine (pour le développement)
    CheckOrigin: func(r *http.Request) bool {
        return true
    },
}

// Gestionnaire WebSocket
func wsHandler(w http.ResponseWriter, r *http.Request) {
    // Upgrader la connexion HTTP vers WebSocket
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("Erreur upgrade WebSocket: %v", err)
        return
    }
    defer conn.Close()

    fmt.Println("📡 Nouvelle connexion WebSocket établie")

    // Boucle de lecture des messages
    for {
        // Lire un message du client
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            log.Printf("Erreur lecture message: %v", err)
            break
        }

        fmt.Printf("📨 Message reçu: %s\n", string(message))

        // Renvoyer le message (echo)
        err = conn.WriteMessage(messageType, message)
        if err != nil {
            log.Printf("Erreur écriture message: %v", err)
            break
        }

        fmt.Printf("📤 Message renvoyé: %s\n", string(message))
    }

    fmt.Println("🔌 Connexion WebSocket fermée")
}

// Page HTML de test
func homeHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Test WebSocket</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .container { max-width: 600px; }
        #messages { border: 1px solid #ccc; height: 300px; overflow-y: scroll; padding: 10px; margin: 10px 0; }
        input[type="text"] { width: 70%; padding: 5px; }
        button { padding: 5px 15px; }
        .message { margin: 5px 0; padding: 5px; border-radius: 3px; }
        .sent { background-color: #e3f2fd; text-align: right; }
        .received { background-color: #f3e5f5; text-align: left; }
        .status { color: #666; font-style: italic; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔌 Test WebSocket</h1>

        <div>
            <button id="connect">Se connecter</button>
            <button id="disconnect" disabled>Se déconnecter</button>
            <span id="status" class="status">Déconnecté</span>
        </div>

        <div id="messages"></div>

        <div>
            <input type="text" id="messageInput" placeholder="Tapez votre message..." disabled>
            <button id="send" disabled>Envoyer</button>
        </div>
    </div>

    <script>
        let ws = null;
        const messagesDiv = document.getElementById('messages');
        const messageInput = document.getElementById('messageInput');
        const connectBtn = document.getElementById('connect');
        const disconnectBtn = document.getElementById('disconnect');
        const sendBtn = document.getElementById('send');
        const statusSpan = document.getElementById('status');

        function addMessage(content, type) {
            const messageDiv = document.createElement('div');
            messageDiv.className = 'message ' + type;
            messageDiv.textContent = content;
            messagesDiv.appendChild(messageDiv);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }

        function updateStatus(status) {
            statusSpan.textContent = status;
        }

        connectBtn.addEventListener('click', function() {
            const wsUrl = 'ws://localhost:8080/ws';
            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                updateStatus('Connecté');
                addMessage('🟢 Connexion établie', 'status');

                connectBtn.disabled = true;
                disconnectBtn.disabled = false;
                messageInput.disabled = false;
                sendBtn.disabled = false;
                messageInput.focus();
            };

            ws.onmessage = function(event) {
                addMessage('📨 Reçu: ' + event.data, 'received');
            };

            ws.onclose = function() {
                updateStatus('Déconnecté');
                addMessage('🔴 Connexion fermée', 'status');

                connectBtn.disabled = false;
                disconnectBtn.disabled = true;
                messageInput.disabled = true;
                sendBtn.disabled = true;
            };

            ws.onerror = function(error) {
                updateStatus('Erreur');
                addMessage('❌ Erreur: ' + error, 'status');
            };
        });

        disconnectBtn.addEventListener('click', function() {
            if (ws) {
                ws.close();
            }
        });

        sendBtn.addEventListener('click', function() {
            sendMessage();
        });

        messageInput.addEventListener('keypress', function(e) {
            if (e.key === 'Enter') {
                sendMessage();
            }
        });

        function sendMessage() {
            const message = messageInput.value.trim();
            if (message && ws && ws.readyState === WebSocket.OPEN) {
                ws.send(message);
                addMessage('📤 Envoyé: ' + message, 'sent');
                messageInput.value = '';
            }
        }
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

func main() {
    // Routes
    http.HandleFunc("/", homeHandler)
    http.HandleFunc("/ws", wsHandler)

    fmt.Println("🚀 Serveur WebSocket démarré sur http://localhost:8080")
    fmt.Println("📖 Visitez http://localhost:8080 pour tester")

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Client WebSocket en Go

### Client simple

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "os"
    "os/signal"
    "time"

    "github.com/gorilla/websocket"
)

func main() {
    // Se connecter au serveur WebSocket
    conn, _, err := websocket.DefaultDialer.Dial("ws://localhost:8080/ws", nil)
    if err != nil {
        log.Fatal("Erreur connexion WebSocket:", err)
    }
    defer conn.Close()

    fmt.Println("📡 Connecté au serveur WebSocket")

    // Gérer l'interruption (Ctrl+C)
    interrupt := make(chan os.Signal, 1)
    signal.Notify(interrupt, os.Interrupt)

    // Canal pour les messages à envoyer
    messageChan := make(chan string)

    // Goroutine pour lire les messages du serveur
    go func() {
        for {
            _, message, err := conn.ReadMessage()
            if err != nil {
                log.Printf("Erreur lecture: %v", err)
                return
            }
            fmt.Printf("📨 Reçu: %s\n", string(message))
        }
    }()

    // Goroutine pour lire les entrées utilisateur
    go func() {
        scanner := bufio.NewScanner(os.Stdin)
        fmt.Println("💬 Tapez vos messages (Ctrl+C pour quitter):")

        for scanner.Scan() {
            message := scanner.Text()
            if message != "" {
                messageChan <- message
            }
        }
    }()

    // Boucle principale
    for {
        select {
        case message := <-messageChan:
            // Envoyer le message
            err := conn.WriteMessage(websocket.TextMessage, []byte(message))
            if err != nil {
                log.Printf("Erreur envoi: %v", err)
                return
            }
            fmt.Printf("📤 Envoyé: %s\n", message)

        case <-interrupt:
            fmt.Println("\n👋 Fermeture de la connexion...")

            // Envoyer un message de fermeture
            err := conn.WriteMessage(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""))
            if err != nil {
                log.Printf("Erreur fermeture: %v", err)
                return
            }

            // Attendre un peu pour que le message soit envoyé
            time.Sleep(time.Second)
            return
        }
    }
}
```

## Chat Room en temps réel

### Serveur de chat complet

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

// Structure pour les messages
type Message struct {
    Type      string    `json:"type"`
    Content   string    `json:"content"`
    Username  string    `json:"username"`
    Timestamp time.Time `json:"timestamp"`
    Room      string    `json:"room,omitempty"`
}

// Structure pour un client connecté
type Client struct {
    conn     *websocket.Conn
    username string
    room     string
    hub      *Hub
    send     chan Message
}

// Structure pour gérer tous les clients
type Hub struct {
    // Clients connectés par room
    rooms map[string]map[*Client]bool

    // Canal pour enregistrer de nouveaux clients
    register chan *Client

    // Canal pour désenregistrer des clients
    unregister chan *Client

    // Canal pour diffuser des messages
    broadcast chan Message

    // Mutex pour accès concurrent
    mutex sync.RWMutex
}

// Créer un nouveau hub
func newHub() *Hub {
    return &Hub{
        rooms:      make(map[string]map[*Client]bool),
        register:   make(chan *Client),
        unregister: make(chan *Client),
        broadcast:  make(chan Message),
    }
}

// Boucle principale du hub
func (h *Hub) run() {
    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()

            // Créer la room si elle n'existe pas
            if h.rooms[client.room] == nil {
                h.rooms[client.room] = make(map[*Client]bool)
            }

            // Ajouter le client à la room
            h.rooms[client.room][client] = true
            h.mutex.Unlock()

            // Notifier les autres clients de l'arrivée
            joinMessage := Message{
                Type:      "join",
                Content:   fmt.Sprintf("%s a rejoint la room", client.username),
                Username:  "Système",
                Timestamp: time.Now(),
                Room:      client.room,
            }

            h.broadcastToRoom(client.room, joinMessage)

            fmt.Printf("👤 %s connecté à la room '%s'\n", client.username, client.room)

        case client := <-h.unregister:
            h.mutex.Lock()
            if clients, ok := h.rooms[client.room]; ok {
                if _, exists := clients[client]; exists {
                    delete(clients, client)
                    close(client.send)

                    // Supprimer la room si elle est vide
                    if len(clients) == 0 {
                        delete(h.rooms, client.room)
                    }
                }
            }
            h.mutex.Unlock()

            // Notifier les autres clients du départ
            leaveMessage := Message{
                Type:      "leave",
                Content:   fmt.Sprintf("%s a quitté la room", client.username),
                Username:  "Système",
                Timestamp: time.Now(),
                Room:      client.room,
            }

            h.broadcastToRoom(client.room, leaveMessage)

            fmt.Printf("👋 %s déconnecté de la room '%s'\n", client.username, client.room)

        case message := <-h.broadcast:
            h.broadcastToRoom(message.Room, message)
        }
    }
}

// Diffuser un message à tous les clients d'une room
func (h *Hub) broadcastToRoom(room string, message Message) {
    h.mutex.RLock()
    clients := h.rooms[room]
    h.mutex.RUnlock()

    for client := range clients {
        select {
        case client.send <- message:
        default:
            // Le canal est plein, fermer la connexion
            close(client.send)
            delete(clients, client)
        }
    }
}

// Obtenir la liste des clients d'une room
func (h *Hub) getRoomClients(room string) []string {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    var usernames []string
    if clients, ok := h.rooms[room]; ok {
        for client := range clients {
            usernames = append(usernames, client.username)
        }
    }

    return usernames
}

// Configurer l'upgrader
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return true
    },
}

// Hub global
var hub = newHub()

// Gestionnaire WebSocket pour le chat
func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("Erreur upgrade: %v", err)
        return
    }

    // Récupérer les paramètres d'URL
    username := r.URL.Query().Get("username")
    room := r.URL.Query().Get("room")

    if username == "" {
        username = "Anonyme"
    }
    if room == "" {
        room = "general"
    }

    // Créer le client
    client := &Client{
        conn:     conn,
        username: username,
        room:     room,
        hub:      hub,
        send:     make(chan Message, 256),
    }

    // Enregistrer le client
    hub.register <- client

    // Démarrer les goroutines pour ce client
    go client.writePump()
    go client.readPump()
}

// Lire les messages du client
func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    // Configuration des timeouts
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        var message Message
        err := c.conn.ReadJSON(&message)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("Erreur WebSocket: %v", err)
            }
            break
        }

        // Ajouter les métadonnées du message
        message.Username = c.username
        message.Timestamp = time.Now()
        message.Room = c.room

        // Diffuser le message
        c.hub.broadcast <- message
    }
}

// Écrire les messages vers le client
func (c *Client) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.conn.WriteJSON(message); err != nil {
                log.Printf("Erreur écriture: %v", err)
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// Handler pour obtenir les statistiques
func statsHandler(w http.ResponseWriter, r *http.Request) {
    hub.mutex.RLock()
    defer hub.mutex.RUnlock()

    stats := map[string]interface{}{
        "rooms": make(map[string]interface{}),
        "total_clients": 0,
    }

    totalClients := 0
    for room, clients := range hub.rooms {
        clientCount := len(clients)
        totalClients += clientCount

        var usernames []string
        for client := range clients {
            usernames = append(usernames, client.username)
        }

        stats["rooms"].(map[string]interface{})[room] = map[string]interface{}{
            "client_count": clientCount,
            "clients":      usernames,
        }
    }

    stats["total_clients"] = totalClients

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(stats)
}

// Page HTML pour le chat
func chatHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Chat WebSocket</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background: #f5f5f5; }
        .container { max-width: 800px; margin: 0 auto; background: white; border-radius: 10px; overflow: hidden; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .header { background: #007bff; color: white; padding: 20px; text-align: center; }
        .chat-container { display: flex; height: 500px; }
        .sidebar { width: 200px; background: #f8f9fa; border-right: 1px solid #ddd; padding: 10px; }
        .main-chat { flex: 1; display: flex; flex-direction: column; }
        .messages { flex: 1; overflow-y: scroll; padding: 20px; border-bottom: 1px solid #ddd; }
        .message { margin: 10px 0; padding: 10px; border-radius: 8px; }
        .message.own { background: #007bff; color: white; margin-left: 50px; }
        .message.other { background: #e9ecef; margin-right: 50px; }
        .message.system { background: #fff3cd; text-align: center; font-style: italic; }
        .message-meta { font-size: 0.8em; opacity: 0.7; margin-bottom: 5px; }
        .input-area { padding: 20px; background: #f8f9fa; display: flex; gap: 10px; }
        .input-area input { flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 5px; }
        .input-area button { padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 5px; cursor: pointer; }
        .input-area button:hover { background: #0056b3; }
        .input-area button:disabled { background: #ccc; cursor: not-allowed; }
        .login-form { padding: 40px; text-align: center; }
        .login-form input { display: block; width: 200px; margin: 10px auto; padding: 10px; border: 1px solid #ddd; border-radius: 5px; }
        .login-form button { padding: 10px 30px; background: #28a745; color: white; border: none; border-radius: 5px; cursor: pointer; }
        .users-list { list-style: none; padding: 0; }
        .users-list li { padding: 5px; margin: 2px 0; background: #e9ecef; border-radius: 3px; }
        .status { padding: 10px; text-align: center; font-weight: bold; }
        .connected { color: #28a745; }
        .disconnected { color: #dc3545; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>💬 Chat WebSocket</h1>
        </div>

        <div id="login-form" class="login-form">
            <h2>Rejoindre le chat</h2>
            <input type="text" id="username" placeholder="Votre nom" required>
            <input type="text" id="room" placeholder="Nom de la room (optionnel)" value="general">
            <button onclick="connect()">Se connecter</button>
        </div>

        <div id="chat-interface" class="chat-container" style="display: none;">
            <div class="sidebar">
                <h3>Room: <span id="current-room"></span></h3>
                <div class="status" id="connection-status">Déconnecté</div>

                <h4>Utilisateurs connectés:</h4>
                <ul id="users-list" class="users-list"></ul>

                <button onclick="disconnect()" style="width: 100%; margin-top: 20px;">Quitter</button>
            </div>

            <div class="main-chat">
                <div id="messages" class="messages"></div>

                <div class="input-area">
                    <input type="text" id="message-input" placeholder="Tapez votre message..." disabled>
                    <button id="send-btn" onclick="sendMessage()" disabled>Envoyer</button>
                </div>
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let currentUsername = '';
        let currentRoom = '';

        function connect() {
            const username = document.getElementById('username').value.trim();
            const room = document.getElementById('room').value.trim() || 'general';

            if (!username) {
                alert('Veuillez entrer votre nom');
                return;
            }

            currentUsername = username;
            currentRoom = room;

            const wsUrl = \`ws://localhost:8080/ws?username=\${encodeURIComponent(username)}&room=\${encodeURIComponent(room)}\`;
            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                document.getElementById('login-form').style.display = 'none';
                document.getElementById('chat-interface').style.display = 'flex';
                document.getElementById('current-room').textContent = room;
                document.getElementById('connection-status').textContent = 'Connecté';
                document.getElementById('connection-status').className = 'status connected';

                document.getElementById('message-input').disabled = false;
                document.getElementById('send-btn').disabled = false;
                document.getElementById('message-input').focus();

                // Demander la liste des utilisateurs
                updateUsersList();
            };

            ws.onmessage = function(event) {
                const message = JSON.parse(event.data);
                addMessage(message);
            };

            ws.onclose = function() {
                document.getElementById('connection-status').textContent = 'Déconnecté';
                document.getElementById('connection-status').className = 'status disconnected';
                document.getElementById('message-input').disabled = true;
                document.getElementById('send-btn').disabled = true;
            };

            ws.onerror = function(error) {
                console.error('Erreur WebSocket:', error);
                alert('Erreur de connexion WebSocket');
            };
        }

        function disconnect() {
            if (ws) {
                ws.close();
            }

            document.getElementById('login-form').style.display = 'block';
            document.getElementById('chat-interface').style.display = 'none';
            document.getElementById('messages').innerHTML = '';
        }

        function sendMessage() {
            const input = document.getElementById('message-input');
            const message = input.value.trim();

            if (message && ws && ws.readyState === WebSocket.OPEN) {
                const messageObj = {
                    type: 'message',
                    content: message
                };

                ws.send(JSON.stringify(messageObj));
                input.value = '';
            }
        }

        function addMessage(message) {
            const messagesDiv = document.getElementById('messages');
            const messageEl = document.createElement('div');

            let className = 'message ';
            if (message.type === 'join' || message.type === 'leave') {
                className += 'system';
            } else if (message.username === currentUsername) {
                className += 'own';
            } else {
                className += 'other';
            }

            messageEl.className = className;

            const timestamp = new Date(message.timestamp).toLocaleTimeString();

            messageEl.innerHTML = \`
                <div class="message-meta">\${message.username} - \${timestamp}</div>
                <div>\${message.content}</div>
            \`;

            messagesDiv.appendChild(messageEl);
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }

        function updateUsersList() {
            // Cette fonction pourrait être améliorée pour récupérer la liste en temps réel
            // Pour l'instant, on simule avec les messages join/leave
        }

        // Envoyer message avec Entrée
        document.addEventListener('DOMContentLoaded', function() {
            document.getElementById('message-input').addEventListener('keypress', function(e) {
                if (e.key === 'Enter') {
                    sendMessage();
                }
            });

            document.getElementById('username').addEventListener('keypress', function(e) {
                if (e.key === 'Enter') {
                    connect();
                }
            });
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

func main() {
    // Démarrer le hub
    go hub.run()

    // Routes
    http.HandleFunc("/", chatHandler)
    http.HandleFunc("/ws", wsHandler)
    http.HandleFunc("/stats", statsHandler)

    fmt.Println("🚀 Serveur de chat WebSocket démarré sur http://localhost:8080")
    fmt.Println("📊 Statistiques disponibles sur http://localhost:8080/stats")

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Gestion des erreurs et bonnes pratiques

### Gestion robuste des connexions

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

// Configuration pour les WebSockets
const (
    writeWait      = 10 * time.Second    // Timeout d'écriture
    pongWait       = 60 * time.Second    // Timeout pour recevoir pong
    pingPeriod     = (pongWait * 9) / 10 // Intervalle d'envoi de ping
    maxMessageSize = 512                 // Taille max des messages
)

// Client amélioré avec gestion d'erreurs
type RobustClient struct {
    id       string
    conn     *websocket.Conn
    hub      *RobustHub
    send     chan []byte
    ctx      context.Context
    cancel   context.CancelFunc
    metadata map[string]interface{}
    mutex    sync.RWMutex
}

// Hub amélioré
type RobustHub struct {
    clients    map[*RobustClient]bool
    register   chan *RobustClient
    unregister chan *RobustClient
    broadcast  chan []byte
    mutex      sync.RWMutex
}

func newRobustHub() *RobustHub {
    return &RobustHub{
        clients:    make(map[*RobustClient]bool),
        register:   make(chan *RobustClient),
        unregister: make(chan *RobustClient),
        broadcast:  make(chan []byte),
    }
}

func (h *RobustHub) run() {
    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()
            h.clients[client] = true
            h.mutex.Unlock()

            log.Printf("📡 Client %s connecté (Total: %d)", client.id, len(h.clients))

            // Message de bienvenue
            welcomeMsg := fmt.Sprintf(`{"type":"system","message":"Bienvenue %s! %d utilisateurs connectés"}`,
                client.id, len(h.clients))

            select {
            case client.send <- []byte(welcomeMsg):
            default:
                close(client.send)
                delete(h.clients, client)
            }

        case client := <-h.unregister:
            h.mutex.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
                h.mutex.Unlock()

                log.Printf("🔌 Client %s déconnecté (Restants: %d)", client.id, len(h.clients))

                // Notifier les autres clients
                leaveMsg := fmt.Sprintf(`{"type":"system","message":"L'utilisateur %s a quitté le chat"}`, client.id)
                h.broadcastMessage([]byte(leaveMsg))
            } else {
                h.mutex.Unlock()
            }

        case message := <-h.broadcast:
            h.broadcastMessage(message)
        }
    }
}

func (h *RobustHub) broadcastMessage(message []byte) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    for client := range h.clients {
        select {
        case client.send <- message:
        default:
            close(client.send)
            delete(h.clients, client)
        }
    }
}

func (h *RobustHub) getStats() map[string]interface{} {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    return map[string]interface{}{
        "connected_clients": len(h.clients),
        "timestamp":         time.Now(),
    }
}

// Client robuste avec gestion complète des erreurs
func newRobustClient(conn *websocket.Conn, hub *RobustHub, id string) *RobustClient {
    ctx, cancel := context.WithCancel(context.Background())

    return &RobustClient{
        id:       id,
        conn:     conn,
        hub:      hub,
        send:     make(chan []byte, 256),
        ctx:      ctx,
        cancel:   cancel,
        metadata: make(map[string]interface{}),
    }
}

// Lecture des messages avec gestion d'erreurs
func (c *RobustClient) readPump() {
    defer func() {
        c.cancel()
        c.hub.unregister <- c
        c.conn.Close()
    }()

    // Configuration des limites
    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        select {
        case <-c.ctx.Done():
            return
        default:
            _, message, err := c.conn.ReadMessage()
            if err != nil {
                if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                    log.Printf("Erreur WebSocket pour %s: %v", c.id, err)
                }
                return
            }

            // Traiter le message
            c.handleMessage(message)
        }
    }
}

// Écriture des messages avec gestion d'erreurs
func (c *RobustClient) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case <-c.ctx.Done():
            return

        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))

            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // Ajouter les messages en attente
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'})
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// Traitement des messages entrants
func (c *RobustClient) handleMessage(message []byte) {
    // Parser le message JSON
    var msg map[string]interface{}
    if err := json.Unmarshal(message, &msg); err != nil {
        log.Printf("Erreur parsing message de %s: %v", c.id, err)
        return
    }

    // Ajouter des métadonnées
    msg["sender"] = c.id
    msg["timestamp"] = time.Now()

    // Reserialiser et diffuser
    enrichedMessage, err := json.Marshal(msg)
    if err != nil {
        log.Printf("Erreur serialization: %v", err)
        return
    }

    c.hub.broadcast <- enrichedMessage
}

// Configurer l'upgrader avec sécurité
var robustUpgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // En production, vérifier l'origine
        origin := r.Header.Get("Origin")
        allowedOrigins := []string{
            "http://localhost:8080",
            "https://yourdomain.com",
        }

        for _, allowed := range allowedOrigins {
            if origin == allowed {
                return true
            }
        }

        // Pour le développement, autoriser localhost
        return strings.Contains(origin, "localhost")
    },
    Error: func(w http.ResponseWriter, r *http.Request, status int, reason error) {
        log.Printf("Erreur WebSocket upgrade: %v", reason)
        http.Error(w, reason.Error(), status)
    },
}

// Handler WebSocket robuste
func robustWSHandler(hub *RobustHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := robustUpgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Erreur upgrade: %v", err)
            return
        }

        // Générer un ID unique pour le client
        clientID := r.URL.Query().Get("id")
        if clientID == "" {
            clientID = fmt.Sprintf("user_%d", time.Now().UnixNano())
        }

        // Créer le client
        client := newRobustClient(conn, hub, clientID)

        // Enregistrer le client
        hub.register <- client

        // Démarrer les goroutines
        go client.writePump()
        go client.readPump()
    }
}
```

## Monitoring et métriques en temps réel

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// Métriques du serveur WebSocket
type WSMetrics struct {
    mutex              sync.RWMutex
    TotalConnections   int64     `json:"total_connections"`
    ActiveConnections  int64     `json:"active_connections"`
    MessagesReceived   int64     `json:"messages_received"`
    MessagesSent       int64     `json:"messages_sent"`
    ErrorsCount        int64     `json:"errors_count"`
    Uptime             time.Time `json:"uptime"`
    LastActivity       time.Time `json:"last_activity"`
    ConnectionsPerHour []int     `json:"connections_per_hour"` // 24 heures
}

// Singleton pour les métriques
var metrics = &WSMetrics{
    Uptime:             time.Now(),
    ConnectionsPerHour: make([]int, 24),
}

func (m *WSMetrics) incrementConnections() {
    m.mutex.Lock()
    defer m.mutex.Unlock()

    m.TotalConnections++
    m.ActiveConnections++
    m.LastActivity = time.Now()

    // Mettre à jour les connexions par heure
    hour := time.Now().Hour()
    m.ConnectionsPerHour[hour]++
}

func (m *WSMetrics) decrementConnections() {
    m.mutex.Lock()
    defer m.mutex.Unlock()

    m.ActiveConnections--
    m.LastActivity = time.Now()
}

func (m *WSMetrics) incrementMessages(sent bool) {
    m.mutex.Lock()
    defer m.mutex.Unlock()

    if sent {
        m.MessagesSent++
    } else {
        m.MessagesReceived++
    }
    m.LastActivity = time.Now()
}

func (m *WSMetrics) incrementErrors() {
    m.mutex.Lock()
    defer m.mutex.Unlock()

    m.ErrorsCount++
}

func (m *WSMetrics) getSnapshot() WSMetrics {
    m.mutex.RLock()
    defer m.mutex.RUnlock()

    return *m
}

// Handler pour les métriques
func metricsHandler(w http.ResponseWriter, r *http.Request) {
    snapshot := metrics.getSnapshot()

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(snapshot)
}

// Page de monitoring en temps réel
func monitoringHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Monitoring</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; background: #f5f5f5; }
        .container { max-width: 1200px; margin: 0 auto; }
        .card { background: white; padding: 20px; margin: 20px 0; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .metric { display: inline-block; margin: 10px 20px; text-align: center; }
        .metric-value { font-size: 2em; font-weight: bold; color: #007bff; }
        .metric-label { font-size: 0.9em; color: #666; }
        .chart { height: 300px; background: #f8f9fa; border: 1px solid #ddd; margin: 20px 0; padding: 20px; }
        .status-good { color: #28a745; }
        .status-warning { color: #ffc107; }
        .status-error { color: #dc3545; }
        .refresh-btn { background: #007bff; color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; }
        .refresh-btn:hover { background: #0056b3; }
    </style>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <div class="container">
        <h1>📊 WebSocket Server Monitoring</h1>

        <div class="card">
            <h2>Métriques en temps réel</h2>
            <button class="refresh-btn" onclick="refreshMetrics()">Actualiser</button>
            <button class="refresh-btn" onclick="toggleAutoRefresh()">Auto-refresh</button>

            <div id="metrics-display">
                <div class="metric">
                    <div class="metric-value" id="active-connections">-</div>
                    <div class="metric-label">Connexions actives</div>
                </div>

                <div class="metric">
                    <div class="metric-value" id="total-connections">-</div>
                    <div class="metric-label">Total connexions</div>
                </div>

                <div class="metric">
                    <div class="metric-value" id="messages-received">-</div>
                    <div class="metric-label">Messages reçus</div>
                </div>

                <div class="metric">
                    <div class="metric-value" id="messages-sent">-</div>
                    <div class="metric-label">Messages envoyés</div>
                </div>

                <div class="metric">
                    <div class="metric-value" id="errors-count">-</div>
                    <div class="metric-label">Erreurs</div>
                </div>

                <div class="metric">
                    <div class="metric-value" id="uptime">-</div>
                    <div class="metric-label">Uptime</div>
                </div>
            </div>
        </div>

        <div class="card">
            <h2>Connexions par heure (24h)</h2>
            <canvas id="connectionsChart" width="400" height="200"></canvas>
        </div>

        <div class="card">
            <h2>Test de connexion</h2>
            <button onclick="testConnection()">Tester une connexion</button>
            <button onclick="testMultipleConnections()">Tester 10 connexions</button>
            <div id="test-results"></div>
        </div>
    </div>

    <script>
        let chart = null;
        let autoRefreshInterval = null;
        let isAutoRefresh = false;

        // Initialiser le graphique
        function initChart() {
            const ctx = document.getElementById('connectionsChart').getContext('2d');
            chart = new Chart(ctx, {
                type: 'line',
                data: {
                    labels: Array.from({length: 24}, (_, i) => i + 'h'),
                    datasets: [{
                        label: 'Connexions par heure',
                        data: new Array(24).fill(0),
                        borderColor: '#007bff',
                        backgroundColor: 'rgba(0, 123, 255, 0.1)',
                        fill: true
                    }]
                },
                options: {
                    responsive: true,
                    scales: {
                        y: {
                            beginAtZero: true
                        }
                    }
                }
            });
        }

        // Récupérer et afficher les métriques
        function refreshMetrics() {
            fetch('/metrics')
                .then(response => response.json())
                .then(data => {
                    document.getElementById('active-connections').textContent = data.active_connections;
                    document.getElementById('total-connections').textContent = data.total_connections;
                    document.getElementById('messages-received').textContent = data.messages_received;
                    document.getElementById('messages-sent').textContent = data.messages_sent;
                    document.getElementById('errors-count').textContent = data.errors_count;

                    // Calculer l'uptime
                    const uptime = new Date() - new Date(data.uptime);
                    const hours = Math.floor(uptime / (1000 * 60 * 60));
                    const minutes = Math.floor((uptime % (1000 * 60 * 60)) / (1000 * 60));
                    document.getElementById('uptime').textContent = hours + 'h ' + minutes + 'm';

                    // Mettre à jour le graphique
                    if (chart && data.connections_per_hour) {
                        chart.data.datasets[0].data = data.connections_per_hour;
                        chart.update();
                    }

                    // Colorer les métriques selon les seuils
                    updateMetricColors(data);
                })
                .catch(error => {
                    console.error('Erreur récupération métriques:', error);
                });
        }

        function updateMetricColors(data) {
            const activeEl = document.getElementById('active-connections');
            if (data.active_connections > 100) {
                activeEl.className = 'metric-value status-warning';
            } else if (data.active_connections > 500) {
                activeEl.className = 'metric-value status-error';
            } else {
                activeEl.className = 'metric-value status-good';
            }

            const errorsEl = document.getElementById('errors-count');
            if (data.errors_count > 10) {
                errorsEl.className = 'metric-value status-warning';
            } else if (data.errors_count > 50) {
                errorsEl.className = 'metric-value status-error';
            } else {
                errorsEl.className = 'metric-value status-good';
            }
        }

        function toggleAutoRefresh() {
            if (isAutoRefresh) {
                clearInterval(autoRefreshInterval);
                isAutoRefresh = false;
                document.querySelector('button[onclick="toggleAutoRefresh()"]').textContent = 'Auto-refresh';
            } else {
                autoRefreshInterval = setInterval(refreshMetrics, 5000);
                isAutoRefresh = true;
                document.querySelector('button[onclick="toggleAutoRefresh()"]').textContent = 'Stop auto-refresh';
            }
        }

        function testConnection() {
            const ws = new WebSocket('ws://localhost:8080/ws?id=test_' + Date.now());
            const resultsDiv = document.getElementById('test-results');

            ws.onopen = function() {
                resultsDiv.innerHTML += '<div>✅ Connexion test réussie</div>';

                // Envoyer un message de test
                ws.send(JSON.stringify({
                    type: 'test',
                    message: 'Message de test depuis le monitoring'
                }));

                // Fermer après 2 secondes
                setTimeout(() => {
                    ws.close();
                    resultsDiv.innerHTML += '<div>🔌 Connexion test fermée</div>';
                }, 2000);
            };

            ws.onerror = function(error) {
                resultsDiv.innerHTML += '<div>❌ Erreur connexion test: ' + error + '</div>';
            };
        }

        function testMultipleConnections() {
            const resultsDiv = document.getElementById('test-results');
            resultsDiv.innerHTML += '<div>🚀 Test de 10 connexions simultanées...</div>';

            let successCount = 0;
            let errorCount = 0;

            for (let i = 0; i < 10; i++) {
                const ws = new WebSocket('ws://localhost:8080/ws?id=load_test_' + i + '_' + Date.now());

                ws.onopen = function() {
                    successCount++;
                    if (successCount + errorCount === 10) {
                        resultsDiv.innerHTML += '<div>📊 Résultat: ' + successCount + ' succès, ' + errorCount + ' erreurs</div>';
                    }

                    setTimeout(() => ws.close(), 1000);
                };

                ws.onerror = function() {
                    errorCount++;
                    if (successCount + errorCount === 10) {
                        resultsDiv.innerHTML += '<div>📊 Résultat: ' + successCount + ' succès, ' + errorCount + ' erreurs</div>';
                    }
                };
            }
        }

        // Initialisation
        document.addEventListener('DOMContentLoaded', function() {
            initChart();
            refreshMetrics();

            // Auto-refresh par défaut
            toggleAutoRefresh();
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}
```

## Application complète avec authentification WebSocket

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/golang-jwt/jwt/v5"
    "github.com/gorilla/websocket"
)

// Message avec authentification
type AuthMessage struct {
    Type      string                 `json:"type"`
    Content   string                 `json:"content"`
    UserID    int                    `json:"user_id"`
    Username  string                 `json:"username"`
    Timestamp time.Time              `json:"timestamp"`
    Metadata  map[string]interface{} `json:"metadata,omitempty"`
}

// Client authentifié
type AuthenticatedClient struct {
    id       string
    userID   int
    username string
    conn     *websocket.Conn
    send     chan AuthMessage
    hub      *AuthenticatedHub
    rooms    map[string]bool
    mutex    sync.RWMutex
}

// Hub avec authentification
type AuthenticatedHub struct {
    clients    map[*AuthenticatedClient]bool
    rooms      map[string]map[*AuthenticatedClient]bool
    register   chan *AuthenticatedClient
    unregister chan *AuthenticatedClient
    broadcast  chan AuthMessage
    mutex      sync.RWMutex
}

// Claims JWT pour WebSocket
type WSClaims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    jwt.RegisteredClaims
}

var jwtSecret = []byte("your-secret-key")

func newAuthenticatedHub() *AuthenticatedHub {
    return &AuthenticatedHub{
        clients:    make(map[*AuthenticatedClient]bool),
        rooms:      make(map[string]map[*AuthenticatedClient]bool),
        register:   make(chan *AuthenticatedClient),
        unregister: make(chan *AuthenticatedClient),
        broadcast:  make(chan AuthMessage),
    }
}

func (h *AuthenticatedHub) run() {
    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()
            h.clients[client] = true
            h.mutex.Unlock()

            log.Printf("👤 Utilisateur %s (%d) connecté", client.username, client.userID)

            // Message de bienvenue
            welcomeMsg := AuthMessage{
                Type:      "system",
                Content:   fmt.Sprintf("Bienvenue %s!", client.username),
                Timestamp: time.Now(),
            }

            select {
            case client.send <- welcomeMsg:
            default:
                close(client.send)
                delete(h.clients, client)
            }

        case client := <-h.unregister:
            h.mutex.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)

                // Retirer des rooms
                for room := range client.rooms {
                    if roomClients, exists := h.rooms[room]; exists {
                        delete(roomClients, client)
                        if len(roomClients) == 0 {
                            delete(h.rooms, room)
                        }
                    }
                }
            }
            h.mutex.Unlock()

            log.Printf("👋 Utilisateur %s déconnecté", client.username)

        case message := <-h.broadcast:
            h.broadcastMessage(message)
        }
    }
}

func (h *AuthenticatedHub) broadcastMessage(message AuthMessage) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    // Si le message a une room spécifiée, diffuser seulement dans cette room
    if room, exists := message.Metadata["room"].(string); exists && room != "" {
        if roomClients, roomExists := h.rooms[room]; roomExists {
            for client := range roomClients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                    delete(roomClients, client)
                }
            }
        }
    } else {
        // Diffuser à tous les clients
        for client := range h.clients {
            select {
            case client.send <- message:
            default:
                close(client.send)
                delete(h.clients, client)
            }
        }
    }
}

func (h *AuthenticatedHub) joinRoom(client *AuthenticatedClient, room string) {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    if h.rooms[room] == nil {
        h.rooms[room] = make(map[*AuthenticatedClient]bool)
    }

    h.rooms[room][client] = true
    client.mutex.Lock()
    client.rooms[room] = true
    client.mutex.Unlock()

    log.Printf("👤 %s a rejoint la room '%s'", client.username, room)
}

func (h *AuthenticatedHub) leaveRoom(client *AuthenticatedClient, room string) {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    if roomClients, exists := h.rooms[room]; exists {
        delete(roomClients, client)
        if len(roomClients) == 0 {
            delete(h.rooms, room)
        }
    }

    client.mutex.Lock()
    delete(client.rooms, room)
    client.mutex.Unlock()

    log.Printf("👤 %s a quitté la room '%s'", client.username, room)
}

// Valider le token JWT
func validateJWTToken(tokenString string) (*WSClaims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &WSClaims{}, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })

    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*WSClaims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("token invalide")
    }

    return claims, nil
}

// Handler WebSocket avec authentification
func authenticatedWSHandler(hub *AuthenticatedHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Récupérer le token depuis les paramètres de requête ou headers
        token := r.URL.Query().Get("token")
        if token == "" {
            token = r.Header.Get("Sec-WebSocket-Protocol")
        }

        if token == "" {
            http.Error(w, "Token d'authentification requis", http.StatusUnauthorized)
            return
        }

        // Valider le token
        claims, err := validateJWTToken(token)
        if err != nil {
            http.Error(w, "Token invalide", http.StatusUnauthorized)
            return
        }

        // Upgrader la connexion
        conn, err := robustUpgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Erreur upgrade: %v", err)
            return
        }

        // Créer le client authentifié
        client := &AuthenticatedClient{
            id:       fmt.Sprintf("%s_%d", claims.Username, time.Now().UnixNano()),
            userID:   claims.UserID,
            username: claims.Username,
            conn:     conn,
            send:     make(chan AuthMessage, 256),
            hub:      hub,
            rooms:    make(map[string]bool),
        }

        // Enregistrer le client
        hub.register <- client

        // Démarrer les goroutines
        go client.writePump()
        go client.readPump()
    }
}

func (c *AuthenticatedClient) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        var message AuthMessage
        err := c.conn.ReadJSON(&message)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("Erreur WebSocket: %v", err)
            }
            break
        }

        // Enrichir le message avec les infos utilisateur
        message.UserID = c.userID
        message.Username = c.username
        message.Timestamp = time.Now()

        // Traiter les commandes spéciales
        if message.Type == "join_room" {
            if room, ok := message.Metadata["room"].(string); ok {
                c.hub.joinRoom(c, room)
                continue
            }
        } else if message.Type == "leave_room" {
            if room, ok := message.Metadata["room"].(string); ok {
                c.hub.leaveRoom(c, room)
                continue
            }
        }

        // Incrémenter les métriques
        metrics.incrementMessages(false)

        // Diffuser le message
        c.hub.broadcast <- message
    }
}

func (c *AuthenticatedClient) writePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))

            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.conn.WriteJSON(message); err != nil {
                log.Printf("Erreur écriture: %v", err)
                return
            }

            // Incrémenter les métriques
            metrics.incrementMessages(true)

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// Générer un token JWT pour les tests
func generateTestToken(userID int, username string) (string, error) {
    claims := WSClaims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "websocket-test",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// Handler pour générer des tokens de test
func tokenHandler(w http.ResponseWriter, r *http.Request) {
    userID := 1
    username := r.URL.Query().Get("username")
    if username == "" {
        username = "TestUser"
    }

    token, err := generateTestToken(userID, username)
    if err != nil {
        http.Error(w, "Erreur génération token", http.StatusInternalServerError)
        return
    }

    response := map[string]string{
        "token":    token,
        "username": username,
        "user_id":  fmt.Sprintf("%d", userID),
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// Interface utilisateur complète
func appHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Application WebSocket Complète</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f0f2f5; }

        .app-container { display: flex; height: 100vh; }
        .sidebar { width: 250px; background: #2c3e50; color: white; display: flex; flex-direction: column; }
        .main-content { flex: 1; display: flex; flex-direction: column; }

        .sidebar-header { padding: 20px; background: #34495e; text-align: center; }
        .sidebar-header h2 { margin-bottom: 10px; }
        .user-info { font-size: 0.9em; opacity: 0.8; }

        .rooms-section { flex: 1; padding: 20px; }
        .rooms-section h3 { margin-bottom: 15px; color: #ecf0f1; }
        .room-list { list-style: none; }
        .room-item {
            padding: 10px; margin: 5px 0; background: #34495e; border-radius: 5px;
            cursor: pointer; transition: background 0.3s;
        }
        .room-item:hover { background: #4a5f7a; }
        .room-item.active { background: #3498db; }

        .add-room { margin-top: 15px; }
        .add-room input {
            width: 100%; padding: 8px; border: none; border-radius: 3px;
            margin-bottom: 10px;
        }
        .add-room button {
            width: 100%; padding: 8px; background: #27ae60; color: white;
            border: none; border-radius: 3px; cursor: pointer;
        }

        .chat-header {
            background: white; padding: 20px; border-bottom: 1px solid #e1e8ed;
            display: flex; justify-content: space-between; align-items: center;
        }
        .chat-title { font-size: 1.2em; font-weight: bold; }
        .connection-status {
            padding: 5px 15px; border-radius: 20px; font-size: 0.9em;
            font-weight: bold;
        }
        .connected { background: #d4edda; color: #155724; }
        .disconnected { background: #f8d7da; color: #721c24; }

        .messages-container {
            flex: 1; overflow-y: auto; padding: 20px; background: white;
            background-image: radial-gradient(circle, #f8f9fa 1px, transparent 1px);
            background-size: 20px 20px;
        }

        .message {
            max-width: 70%; margin: 15px 0; padding: 12px 16px;
            border-radius: 18px; word-wrap: break-word; position: relative;
        }
        .message.own {
            background: #007bff; color: white; margin-left: auto;
            border-bottom-right-radius: 4px;
        }
        .message.other {
            background: #e9ecef; color: #333; margin-right: auto;
            border-bottom-left-radius: 4px;
        }
        .message.system {
            background: #fff3cd; color: #856404; text-align: center;
            margin: 10px auto; max-width: 60%; font-style: italic;
        }

        .message-header {
            font-size: 0.8em; opacity: 0.7; margin-bottom: 4px;
            display: flex; justify-content: space-between;
        }
        .message-content { line-height: 1.4; }

        .input-area {
            padding: 20px; background: #f8f9fa; border-top: 1px solid #e1e8ed;
            display: flex; align-items: center; gap: 15px;
        }
        .message-input {
            flex: 1; padding: 12px 16px; border: 1px solid #ced4da;
            border-radius: 25px; font-size: 1em; outline: none;
            transition: border-color 0.3s;
        }
        .message-input:focus { border-color: #007bff; }
        .send-button {
            width: 45px; height: 45px; background: #007bff; color: white;
            border: none; border-radius: 50%; cursor: pointer;
            display: flex; align-items: center; justify-content: center;
            transition: background 0.3s;
        }
        .send-button:hover { background: #0056b3; }
        .send-button:disabled { background: #6c757d; cursor: not-allowed; }

        .login-overlay {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.8); display: flex; align-items: center;
            justify-content: center; z-index: 1000;
        }
        .login-form {
            background: white; padding: 40px; border-radius: 10px;
            text-align: center; min-width: 300px;
        }
        .login-form h2 { margin-bottom: 30px; color: #333; }
        .login-form input {
            width: 100%; padding: 12px; margin: 10px 0; border: 1px solid #ddd;
            border-radius: 5px; font-size: 1em;
        }
        .login-form button {
            width: 100%; padding: 12px; background: #007bff; color: white;
            border: none; border-radius: 5px; font-size: 1em; cursor: pointer;
            margin-top: 20px; transition: background 0.3s;
        }
        .login-form button:hover { background: #0056b3; }

        .hidden { display: none !important; }

        @media (max-width: 768px) {
            .sidebar { width: 200px; }
            .message { max-width: 85%; }
        }
    </style>
</head>
<body>
    <div id="login-overlay" class="login-overlay">
        <div class="login-form">
            <h2>🔐 Connexion WebSocket</h2>
            <input type="text" id="username-input" placeholder="Nom d'utilisateur" required>
            <button onclick="login()">Se connecter</button>
            <div style="margin-top: 20px; font-size: 0.9em; color: #666;">
                Entrez votre nom pour rejoindre le chat
            </div>
        </div>
    </div>

    <div id="app-container" class="app-container hidden">
        <div class="sidebar">
            <div class="sidebar-header">
                <h2>💬 WebSocket Chat</h2>
                <div class="user-info">
                    Connecté en tant que<br>
                    <strong id="current-username">-</strong>
                </div>
            </div>

            <div class="rooms-section">
                <h3>🏠 Rooms</h3>
                <ul id="rooms-list" class="room-list">
                    <li class="room-item active" data-room="general" onclick="joinRoom('general')">
                        # general
                    </li>
                </ul>

                <div class="add-room">
                    <input type="text" id="new-room-input" placeholder="Nom de la nouvelle room">
                    <button onclick="createRoom()">Créer Room</button>
                </div>

                <div style="margin-top: 30px; padding-top: 20px; border-top: 1px solid #34495e;">
                    <button onclick="disconnect()" style="width: 100%; padding: 10px; background: #e74c3c; color: white; border: none; border-radius: 5px; cursor: pointer;">
                        Déconnexion
                    </button>
                </div>
            </div>
        </div>

        <div class="main-content">
            <div class="chat-header">
                <div class="chat-title">💬 <span id="current-room-name">general</span></div>
                <div id="connection-status" class="connection-status disconnected">Déconnecté</div>
            </div>

            <div id="messages-container" class="messages-container"></div>

            <div class="input-area">
                <input type="text" id="message-input" class="message-input"
                       placeholder="Tapez votre message..." disabled>
                <button id="send-button" class="send-button" onclick="sendMessage()" disabled>
                    ➤
                </button>
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let currentToken = null;
        let currentUsername = '';
        let currentRoom = 'general';
        let isConnected = false;

        // Connexion utilisateur
        function login() {
            const username = document.getElementById('username-input').value.trim();
            if (!username) {
                alert('Veuillez entrer un nom d\'utilisateur');
                return;
            }

            // Obtenir un token de test
            fetch('/token?username=' + encodeURIComponent(username))
                .then(response => response.json())
                .then(data => {
                    currentToken = data.token;
                    currentUsername = data.username;

                    // Masquer l'overlay de connexion
                    document.getElementById('login-overlay').classList.add('hidden');
                    document.getElementById('app-container').classList.remove('hidden');
                    document.getElementById('current-username').textContent = currentUsername;

                    // Se connecter au WebSocket
                    connectWebSocket();
                })
                .catch(error => {
                    console.error('Erreur obtention token:', error);
                    alert('Erreur de connexion');
                });
        }

        // Connexion WebSocket
        function connectWebSocket() {
            if (!currentToken) {
                console.error('Pas de token disponible');
                return;
            }

            const wsUrl = \`ws://localhost:8080/ws?token=\${currentToken}\`;
            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                isConnected = true;
                updateConnectionStatus('connected');
                enableInterface(true);

                // Rejoindre la room par défaut
                joinRoom('general');

                console.log('WebSocket connecté');
            };

            ws.onmessage = function(event) {
                const message = JSON.parse(event.data);
                displayMessage(message);
            };

            ws.onclose = function() {
                isConnected = false;
                updateConnectionStatus('disconnected');
                enableInterface(false);

                console.log('WebSocket fermé');

                // Tentative de reconnexion après 3 secondes
                setTimeout(() => {
                    if (!isConnected) {
                        console.log('Tentative de reconnexion...');
                        connectWebSocket();
                    }
                }, 3000);
            };

            ws.onerror = function(error) {
                console.error('Erreur WebSocket:', error);
                updateConnectionStatus('error');
            };
        }

        // Rejoindre une room
        function joinRoom(roomName) {
            if (!isConnected) return;

            // Quitter la room actuelle si différente
            if (currentRoom !== roomName) {
                leaveRoom(currentRoom);
            }

            currentRoom = roomName;
            document.getElementById('current-room-name').textContent = roomName;

            // Mettre à jour l'interface
            document.querySelectorAll('.room-item').forEach(item => {
                item.classList.remove('active');
            });
            document.querySelector(\`[data-room="\${roomName}"]\`).classList.add('active');

            // Vider les messages
            document.getElementById('messages-container').innerHTML = '';

            // Envoyer la commande de rejoindre
            const joinMessage = {
                type: 'join_room',
                content: '',
                metadata: { room: roomName }
            };

            ws.send(JSON.stringify(joinMessage));
        }

        // Quitter une room
        function leaveRoom(roomName) {
            if (!isConnected || !roomName) return;

            const leaveMessage = {
                type: 'leave_room',
                content: '',
                metadata: { room: roomName }
            };

            ws.send(JSON.stringify(leaveMessage));
        }

        // Créer une nouvelle room
        function createRoom() {
            const roomName = document.getElementById('new-room-input').value.trim();
            if (!roomName) {
                alert('Veuillez entrer un nom de room');
                return;
            }

            // Vérifier si la room existe déjà
            if (document.querySelector(\`[data-room="\${roomName}"]\`)) {
                alert('Cette room existe déjà');
                return;
            }

            // Ajouter la room à la liste
            const roomsList = document.getElementById('rooms-list');
            const roomItem = document.createElement('li');
            roomItem.className = 'room-item';
            roomItem.setAttribute('data-room', roomName);
            roomItem.onclick = () => joinRoom(roomName);
            roomItem.textContent = '# ' + roomName;

            roomsList.appendChild(roomItem);

            // Vider l'input
            document.getElementById('new-room-input').value = '';

            // Rejoindre automatiquement la nouvelle room
            joinRoom(roomName);
        }

        // Envoyer un message
        function sendMessage() {
            const input = document.getElementById('message-input');
            const content = input.value.trim();

            if (!content || !isConnected) return;

            const message = {
                type: 'message',
                content: content,
                metadata: { room: currentRoom }
            };

            ws.send(JSON.stringify(message));
            input.value = '';
        }

        // Afficher un message
        function displayMessage(message) {
            const container = document.getElementById('messages-container');
            const messageEl = document.createElement('div');

            let className = 'message ';
            if (message.type === 'system') {
                className += 'system';
            } else if (message.username === currentUsername) {
                className += 'own';
            } else {
                className += 'other';
            }

            messageEl.className = className;

            const timestamp = new Date(message.timestamp).toLocaleTimeString();

            if (message.type === 'system') {
                messageEl.innerHTML = \`<div class="message-content">\${message.content}</div>\`;
            } else {
                messageEl.innerHTML = \`
                    <div class="message-header">
                        <span>\${message.username}</span>
                        <span>\${timestamp}</span>
                    </div>
                    <div class="message-content">\${message.content}</div>
                \`;
            }

            container.appendChild(messageEl);
            container.scrollTop = container.scrollHeight;
        }

        // Mettre à jour le statut de connexion
        function updateConnectionStatus(status) {
            const statusEl = document.getElementById('connection-status');

            switch (status) {
                case 'connected':
                    statusEl.textContent = 'Connecté';
                    statusEl.className = 'connection-status connected';
                    break;
                case 'disconnected':
                    statusEl.textContent = 'Déconnecté';
                    statusEl.className = 'connection-status disconnected';
                    break;
                case 'error':
                    statusEl.textContent = 'Erreur';
                    statusEl.className = 'connection-status disconnected';
                    break;
            }
        }

        // Activer/désactiver l'interface
        function enableInterface(enable) {
            document.getElementById('message-input').disabled = !enable;
            document.getElementById('send-button').disabled = !enable;

            if (enable) {
                document.getElementById('message-input').focus();
            }
        }

        // Déconnexion
        function disconnect() {
            if (ws) {
                isConnected = false;
                ws.close();
            }

            // Retour à l'écran de connexion
            document.getElementById('login-overlay').classList.remove('hidden');
            document.getElementById('app-container').classList.add('hidden');

            // Réinitialiser
            currentToken = null;
            currentUsername = '';
            currentRoom = 'general';
            document.getElementById('username-input').value = '';
            document.getElementById('messages-container').innerHTML = '';
        }

        // Événements clavier
        document.addEventListener('DOMContentLoaded', function() {
            document.getElementById('message-input').addEventListener('keypress', function(e) {
                if (e.key === 'Enter') {
                    sendMessage();
                }
            });

            document.getElementById('username-input').addEventListener('keypress', function(e) {
                if (e.key === 'Enter') {
                    login();
                }
            });

            document.getElementById('new-room-input').addEventListener('keypress', function(e) {
                if (e.key === 'Enter') {
                    createRoom();
                }
            });

            // Focus automatique sur le champ username
            document.getElementById('username-input').focus();
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Fonction principale avec tous les composants
func main() {
    // Créer le hub authentifié
    authHub := newAuthenticatedHub()
    go authHub.run()

    // Configurer les routes
    http.HandleFunc("/", appHandler)
    http.HandleFunc("/ws", authenticatedWSHandler(authHub))
    http.HandleFunc("/token", tokenHandler)
    http.HandleFunc("/metrics", metricsHandler)
    http.HandleFunc("/monitoring", monitoringHandler)

    // Initialiser les métriques
    metrics.incrementConnections() // Pour initialiser
    metrics.decrementConnections()

    fmt.Println("🚀 Application WebSocket complète démarrée")
    fmt.Println("📱 Application principale: http://localhost:8080")
    fmt.Println("📊 Monitoring: http://localhost:8080/monitoring")
    fmt.Println("📈 Métriques: http://localhost:8080/metrics")
    fmt.Println("🔑 Tokens de test: http://localhost:8080/token?username=VotreNom")

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Client Go pour tester l'application

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "time"

    "github.com/gorilla/websocket"
)

// Client de test pour l'application WebSocket
type TestClient struct {
    username string
    token    string
    conn     *websocket.Conn
}

func NewTestClient(username string) *TestClient {
    return &TestClient{
        username: username,
    }
}

func (tc *TestClient) GetToken() error {
    resp, err := http.Get(fmt.Sprintf("http://localhost:8080/token?username=%s", tc.username))
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    var tokenData map[string]string
    if err := json.NewDecoder(resp.Body).Decode(&tokenData); err != nil {
        return err
    }

    tc.token = tokenData["token"]
    fmt.Printf("🔑 Token obtenu pour %s\n", tc.username)
    return nil
}

func (tc *TestClient) Connect() error {
    if tc.token == "" {
        return fmt.Errorf("token requis")
    }

    wsURL := fmt.Sprintf("ws://localhost:8080/ws?token=%s", tc.token)
    conn, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
    if err != nil {
        return err
    }

    tc.conn = conn
    fmt.Printf("📡 %s connecté au WebSocket\n", tc.username)

    // Démarrer la lecture des messages
    go tc.readMessages()

    return nil
}

func (tc *TestClient) readMessages() {
    for {
        var message map[string]interface{}
        err := tc.conn.ReadJSON(&message)
        if err != nil {
            fmt.Printf("❌ %s: Erreur lecture: %v\n", tc.username, err)
            return
        }

        fmt.Printf("📨 %s reçu: %s\n", tc.username, message["content"])
    }
}

func (tc *TestClient) SendMessage(content, room string) error {
    message := map[string]interface{}{
        "type":    "message",
        "content": content,
        "metadata": map[string]string{
            "room": room,
        },
    }

    return tc.conn.WriteJSON(message)
}

func (tc *TestClient) JoinRoom(room string) error {
    message := map[string]interface{}{
        "type":    "join_room",
        "content": "",
        "metadata": map[string]string{
            "room": room,
        },
    }

    return tc.conn.WriteJSON(message)
}

func (tc *TestClient) Close() {
    if tc.conn != nil {
        tc.conn.Close()
        fmt.Printf("🔌 %s déconnecté\n", tc.username)
    }
}

// Test automatisé
func main() {
    fmt.Println("🧪 Test automatisé de l'application WebSocket")
    fmt.Println("==============================================")

    // Créer plusieurs clients de test
    clients := []*TestClient{
        NewTestClient("Alice"),
        NewTestClient("Bob"),
        NewTestClient("Charlie"),
    }

    // Obtenir les tokens et se connecter
    for _, client := range clients {
        if err := client.GetToken(); err != nil {
            log.Printf("Erreur token pour %s: %v", client.username, err)
            continue
        }

        if err := client.Connect(); err != nil {
            log.Printf("Erreur connexion pour %s: %v", client.username, err)
            continue
        }

        time.Sleep(100 * time.Millisecond)
    }

    // Attendre un peu pour que les connexions s'établissent
    time.Sleep(time.Second)

    // Test de messages dans la room general
    fmt.Println("\n📝 Test de messages dans la room 'general'")
    clients[0].SendMessage("Salut tout le monde!", "general")
    clients[1].SendMessage("Hello Alice!", "general")
    clients[2].SendMessage("Bonjour à tous!", "general")

    time.Sleep(time.Second)

    // Test de création et join de room
    fmt.Println("\n🏠 Test de création de room 'developers'")
    clients[0].JoinRoom("developers")
    clients[1].JoinRoom("developers")

    time.Sleep(500 * time.Millisecond)

    clients[0].SendMessage("Parlons de Go!", "developers")
    clients[1].SendMessage("Excellente idée!", "developers")

    time.Sleep(time.Second)

    // Test de room séparée
    fmt.Println("\n🎮 Test de room séparée 'gaming'")
    clients[2].JoinRoom("gaming")
    clients[2].SendMessage("Quelqu'un pour jouer?", "gaming")

    time.Sleep(time.Second)

    // Fermer les connexions
    fmt.Println("\n🔌 Fermeture des connexions")
    for _, client := range clients {
        client.Close()
        time.Sleep(200 * time.Millisecond)
    }

    fmt.Println("\n✅ Tests terminés!")
}
```

## Résumé complet des WebSockets

### Ce que vous avez appris :

1. **Concepts de base** : Différences avec HTTP, cas d'usage
2. **Implementation simple** : Echo server et client basique
3. **Chat en temps réel** : Gestion multi-utilisateurs avec rooms
4. **Gestion robuste** : Erreurs, timeouts, reconnexion
5. **Authentification** : Intégration JWT avec WebSockets
6. **Monitoring** : Métriques et surveillance en temps réel
7. **Interface complète** : Application web moderne avec toutes les fonctionnalités

### Fonctionnalités avancées couvertes :

- **Gestion des connexions** avec timeouts et ping/pong
- **Rooms et channels** pour organiser les communications
- **Authentification sécurisée** avec JWT
- **Métriques en temps réel** pour le monitoring
- **Interface utilisateur moderne** responsive
- **Reconnexion automatique** en cas de perte de connexion
- **Gestion d'erreurs complète** et robuste

Cette implémentation est prête pour la production avec les ajustements appropriés pour votre cas d'usage spécifique !

## Exercices pratiques

### Exercice 1 : Notifications push
Créez un système de notifications push en temps réel avec :
- Différents types de notifications (info, warning, error)
- Filtrage par utilisateur
- Historique des notifications

### Exercice 2 : Collaboration en temps réel
Implémentez un éditeur de texte collaboratif avec :
- Synchronisation en temps réel
- Gestion des conflits
- Curseurs multi-utilisateurs

### Exercice 3 : Dashboard en temps réel
Créez un tableau de bord avec :
- Métriques mises à jour en temps réel
- Graphiques dynamiques
- Alertes automatiques

Ces exercices vous permettront d'approfondir vos connaissances et d'explorer des cas d'usage avancés des WebSockets !

⏭️

# Solutions des Exercices WebSockets

## Exercice 1 : Système de notifications push

### Solution complète

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

// Types de notifications
type NotificationType string

const (
    NotificationInfo    NotificationType = "info"
    NotificationWarning NotificationType = "warning"
    NotificationError   NotificationType = "error"
    NotificationSuccess NotificationType = "success"
)

// Structure d'une notification
type Notification struct {
    ID          string           `json:"id"`
    Type        NotificationType `json:"type"`
    Title       string           `json:"title"`
    Message     string           `json:"message"`
    UserID      int              `json:"user_id,omitempty"`
    UserRole    string           `json:"user_role,omitempty"`
    CreatedAt   time.Time        `json:"created_at"`
    ExpiresAt   *time.Time       `json:"expires_at,omitempty"`
    IsRead      bool             `json:"is_read"`
    Priority    int              `json:"priority"` // 1-5, 5 étant le plus prioritaire
    Category    string           `json:"category,omitempty"`
    ActionURL   string           `json:"action_url,omitempty"`
    Metadata    map[string]interface{} `json:"metadata,omitempty"`
}

// Client de notifications
type NotificationClient struct {
    ID       string
    UserID   int
    Username string
    Role     string
    Conn     *websocket.Conn
    Send     chan Notification
    Hub      *NotificationHub
    Filters  NotificationFilters
}

// Filtres de notifications
type NotificationFilters struct {
    Types      []NotificationType `json:"types"`
    Categories []string           `json:"categories"`
    MinPriority int               `json:"min_priority"`
    OnlyUnread bool               `json:"only_unread"`
}

// Hub de notifications
type NotificationHub struct {
    clients        map[*NotificationClient]bool
    notifications  []Notification
    register       chan *NotificationClient
    unregister     chan *NotificationClient
    broadcast      chan Notification
    mutex          sync.RWMutex
    maxHistory     int
}

func NewNotificationHub() *NotificationHub {
    return &NotificationHub{
        clients:       make(map[*NotificationClient]bool),
        notifications: make([]Notification, 0),
        register:      make(chan *NotificationClient),
        unregister:    make(chan *NotificationClient),
        broadcast:     make(chan Notification),
        maxHistory:    1000, // Garder les 1000 dernières notifications
    }
}

func (h *NotificationHub) run() {
    // Démarrer le nettoyage périodique
    go h.cleanupExpiredNotifications()

    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()
            h.clients[client] = true
            h.mutex.Unlock()

            log.Printf("📱 Client %s connecté aux notifications", client.Username)

            // Envoyer l'historique des notifications non lues
            go h.sendNotificationHistory(client)

        case client := <-h.unregister:
            h.mutex.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.Send)
            }
            h.mutex.Unlock()

            log.Printf("📱 Client %s déconnecté des notifications", client.Username)

        case notification := <-h.broadcast:
            h.mutex.Lock()
            // Ajouter à l'historique
            h.notifications = append(h.notifications, notification)

            // Limiter la taille de l'historique
            if len(h.notifications) > h.maxHistory {
                h.notifications = h.notifications[len(h.notifications)-h.maxHistory:]
            }
            h.mutex.Unlock()

            // Diffuser aux clients appropriés
            h.sendToFilteredClients(notification)
        }
    }
}

func (h *NotificationHub) sendNotificationHistory(client *NotificationClient) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    for _, notification := range h.notifications {
        if h.shouldSendToClient(client, notification) && !notification.IsRead {
            select {
            case client.Send <- notification:
            default:
                close(client.Send)
                delete(h.clients, client)
                return
            }
        }
    }
}

func (h *NotificationHub) sendToFilteredClients(notification Notification) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    for client := range h.clients {
        if h.shouldSendToClient(client, notification) {
            select {
            case client.Send <- notification:
            default:
                close(client.Send)
                delete(h.clients, client)
            }
        }
    }
}

func (h *NotificationHub) shouldSendToClient(client *NotificationClient, notification Notification) bool {
    // Vérifier le ciblage utilisateur
    if notification.UserID != 0 && notification.UserID != client.UserID {
        return false
    }

    // Vérifier le rôle
    if notification.UserRole != "" && notification.UserRole != client.Role {
        return false
    }

    // Appliquer les filtres du client
    filters := client.Filters

    // Filtrer par type
    if len(filters.Types) > 0 {
        found := false
        for _, t := range filters.Types {
            if t == notification.Type {
                found = true
                break
            }
        }
        if !found {
            return false
        }
    }

    // Filtrer par catégorie
    if len(filters.Categories) > 0 && notification.Category != "" {
        found := false
        for _, c := range filters.Categories {
            if c == notification.Category {
                found = true
                break
            }
        }
        if !found {
            return false
        }
    }

    // Filtrer par priorité
    if notification.Priority < filters.MinPriority {
        return false
    }

    // Filtrer les lues/non lues
    if filters.OnlyUnread && notification.IsRead {
        return false
    }

    return true
}

func (h *NotificationHub) cleanupExpiredNotifications() {
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()

    for range ticker.C {
        h.mutex.Lock()
        now := time.Now()
        filtered := h.notifications[:0]

        for _, notification := range h.notifications {
            if notification.ExpiresAt == nil || notification.ExpiresAt.After(now) {
                filtered = append(filtered, notification)
            }
        }

        h.notifications = filtered
        h.mutex.Unlock()

        log.Printf("🧹 Nettoyage des notifications expirées effectué")
    }
}

// API pour créer des notifications
func (h *NotificationHub) CreateNotification(notification Notification) {
    if notification.ID == "" {
        notification.ID = fmt.Sprintf("notif_%d", time.Now().UnixNano())
    }

    if notification.CreatedAt.IsZero() {
        notification.CreatedAt = time.Now()
    }

    h.broadcast <- notification
}

// Marquer comme lu
func (h *NotificationHub) MarkAsRead(notificationID string, userID int) bool {
    h.mutex.Lock()
    defer h.mutex.Unlock()

    for i, notification := range h.notifications {
        if notification.ID == notificationID && (notification.UserID == 0 || notification.UserID == userID) {
            h.notifications[i].IsRead = true
            return true
        }
    }

    return false
}

// Obtenir l'historique
func (h *NotificationHub) GetHistory(userID int, limit int) []Notification {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    var userNotifications []Notification

    for i := len(h.notifications) - 1; i >= 0 && len(userNotifications) < limit; i-- {
        notification := h.notifications[i]
        if notification.UserID == 0 || notification.UserID == userID {
            userNotifications = append(userNotifications, notification)
        }
    }

    return userNotifications
}

// Configuration WebSocket
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return true
    },
}

// Handler WebSocket pour les notifications
func notificationWSHandler(hub *NotificationHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Erreur upgrade: %v", err)
            return
        }

        // Simuler l'authentification (en production, utiliser JWT)
        userID := 1
        username := r.URL.Query().Get("username")
        if username == "" {
            username = "User"
        }
        role := r.URL.Query().Get("role")
        if role == "" {
            role = "user"
        }

        client := &NotificationClient{
            ID:       fmt.Sprintf("%s_%d", username, time.Now().UnixNano()),
            UserID:   userID,
            Username: username,
            Role:     role,
            Conn:     conn,
            Send:     make(chan Notification, 256),
            Hub:      hub,
            Filters: NotificationFilters{
                MinPriority: 1,
                OnlyUnread:  false,
            },
        }

        hub.register <- client

        go client.writePump()
        go client.readPump()
    }
}

func (c *NotificationClient) readPump() {
    defer func() {
        c.Hub.unregister <- c
        c.Conn.Close()
    }()

    for {
        var message map[string]interface{}
        err := c.Conn.ReadJSON(&message)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("Erreur WebSocket: %v", err)
            }
            break
        }

        // Traiter les commandes du client
        c.handleCommand(message)
    }
}

func (c *NotificationClient) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()

    for {
        select {
        case notification, ok := <-c.Send:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.Conn.WriteJSON(notification); err != nil {
                log.Printf("Erreur écriture: %v", err)
                return
            }

        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

func (c *NotificationClient) handleCommand(message map[string]interface{}) {
    command, ok := message["command"].(string)
    if !ok {
        return
    }

    switch command {
    case "mark_read":
        if notifID, ok := message["notification_id"].(string); ok {
            c.Hub.MarkAsRead(notifID, c.UserID)
        }

    case "update_filters":
        if filtersData, ok := message["filters"].(map[string]interface{}); ok {
            c.updateFilters(filtersData)
        }

    case "get_history":
        limit := 50
        if l, ok := message["limit"].(float64); ok {
            limit = int(l)
        }

        history := c.Hub.GetHistory(c.UserID, limit)
        response := map[string]interface{}{
            "type":    "history",
            "history": history,
        }

        c.Conn.WriteJSON(response)
    }
}

func (c *NotificationClient) updateFilters(filtersData map[string]interface{}) {
    // Mettre à jour les filtres du client
    if types, ok := filtersData["types"].([]interface{}); ok {
        c.Filters.Types = make([]NotificationType, 0)
        for _, t := range types {
            if typeStr, ok := t.(string); ok {
                c.Filters.Types = append(c.Filters.Types, NotificationType(typeStr))
            }
        }
    }

    if minPriority, ok := filtersData["min_priority"].(float64); ok {
        c.Filters.MinPriority = int(minPriority)
    }

    if onlyUnread, ok := filtersData["only_unread"].(bool); ok {
        c.Filters.OnlyUnread = onlyUnread
    }
}

// API REST pour créer des notifications
func createNotificationHandler(hub *NotificationHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Method != "POST" {
            http.Error(w, "Méthode non autorisée", http.StatusMethodNotAllowed)
            return
        }

        var notification Notification
        if err := json.NewDecoder(r.Body).Decode(&notification); err != nil {
            http.Error(w, "JSON invalide", http.StatusBadRequest)
            return
        }

        // Validation
        if notification.Title == "" || notification.Message == "" {
            http.Error(w, "Titre et message requis", http.StatusBadRequest)
            return
        }

        hub.CreateNotification(notification)

        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{
            "status": "success",
            "id":     notification.ID,
        })
    }
}

// Interface utilisateur pour les notifications
func notificationUIHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Système de Notifications Push</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background: #f5f5f5; }
        .container { max-width: 800px; margin: 0 auto; }
        .header { text-align: center; margin-bottom: 30px; }
        .notification-panel { background: white; border-radius: 10px; padding: 20px; margin-bottom: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }

        .notification {
            border: 1px solid #ddd; border-radius: 8px; padding: 15px; margin: 10px 0;
            position: relative; transition: all 0.3s ease;
        }
        .notification.unread { border-left: 4px solid #007bff; background: #f8f9ff; }
        .notification.info { border-left-color: #17a2b8; }
        .notification.success { border-left-color: #28a745; }
        .notification.warning { border-left-color: #ffc107; }
        .notification.error { border-left-color: #dc3545; }

        .notification-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
        .notification-title { font-weight: bold; margin: 0; }
        .notification-meta { font-size: 0.8em; color: #666; }
        .notification-message { margin: 10px 0; line-height: 1.4; }
        .notification-actions { text-align: right; }

        .btn { padding: 5px 15px; border: none; border-radius: 5px; cursor: pointer; margin: 0 5px; }
        .btn-primary { background: #007bff; color: white; }
        .btn-success { background: #28a745; color: white; }
        .btn-secondary { background: #6c757d; color: white; }

        .filters { display: flex; gap: 15px; align-items: center; margin-bottom: 20px; flex-wrap: wrap; }
        .filters select, .filters input { padding: 5px; border: 1px solid #ddd; border-radius: 3px; }

        .connection-status {
            padding: 10px; border-radius: 5px; text-align: center; margin-bottom: 20px;
            font-weight: bold;
        }
        .connected { background: #d4edda; color: #155724; }
        .disconnected { background: #f8d7da; color: #721c24; }

        .create-notification { background: #fff3cd; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
        .create-notification h3 { margin-top: 0; }
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 5px; font-weight: bold; }
        .form-group input, .form-group select, .form-group textarea {
            width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 3px;
        }
        .form-group textarea { height: 80px; resize: vertical; }

        .priority-badge {
            display: inline-block; padding: 2px 8px; border-radius: 12px;
            font-size: 0.8em; font-weight: bold; color: white;
        }
        .priority-1 { background: #6c757d; }
        .priority-2 { background: #17a2b8; }
        .priority-3 { background: #ffc107; color: #000; }
        .priority-4 { background: #fd7e14; }
        .priority-5 { background: #dc3545; }

        .stats { display: flex; gap: 20px; margin-bottom: 20px; }
        .stat { flex: 1; background: white; padding: 15px; border-radius: 8px; text-align: center; }
        .stat-value { font-size: 2em; font-weight: bold; color: #007bff; }
        .stat-label { color: #666; font-size: 0.9em; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🔔 Système de Notifications Push</h1>
        </div>

        <div id="connection-status" class="connection-status disconnected">
            Déconnecté
        </div>

        <div class="stats">
            <div class="stat">
                <div class="stat-value" id="total-notifications">0</div>
                <div class="stat-label">Total</div>
            </div>
            <div class="stat">
                <div class="stat-value" id="unread-notifications">0</div>
                <div class="stat-label">Non lues</div>
            </div>
            <div class="stat">
                <div class="stat-value" id="high-priority">0</div>
                <div class="stat-label">Haute priorité</div>
            </div>
        </div>

        <div class="create-notification">
            <h3>📝 Créer une notification de test</h3>
            <div class="form-group">
                <label>Type :</label>
                <select id="notif-type">
                    <option value="info">Info</option>
                    <option value="success">Succès</option>
                    <option value="warning">Avertissement</option>
                    <option value="error">Erreur</option>
                </select>
            </div>
            <div class="form-group">
                <label>Titre :</label>
                <input type="text" id="notif-title" placeholder="Titre de la notification">
            </div>
            <div class="form-group">
                <label>Message :</label>
                <textarea id="notif-message" placeholder="Contenu de la notification"></textarea>
            </div>
            <div class="form-group">
                <label>Priorité :</label>
                <select id="notif-priority">
                    <option value="1">1 - Basse</option>
                    <option value="2">2 - Normale</option>
                    <option value="3" selected>3 - Moyenne</option>
                    <option value="4">4 - Haute</option>
                    <option value="5">5 - Critique</option>
                </select>
            </div>
            <button class="btn btn-primary" onclick="createNotification()">Créer Notification</button>
            <button class="btn btn-secondary" onclick="createRandomNotification()">Notification Aléatoire</button>
        </div>

        <div class="notification-panel">
            <h3>🔔 Filtres</h3>
            <div class="filters">
                <label>
                    Type :
                    <select id="filter-type" onchange="updateFilters()">
                        <option value="">Tous</option>
                        <option value="info">Info</option>
                        <option value="success">Succès</option>
                        <option value="warning">Avertissement</option>
                        <option value="error">Erreur</option>
                    </select>
                </label>

                <label>
                    Priorité min :
                    <select id="filter-priority" onchange="updateFilters()">
                        <option value="1">1+</option>
                        <option value="2">2+</option>
                        <option value="3" selected>3+</option>
                        <option value="4">4+</option>
                        <option value="5">5</option>
                    </select>
                </label>

                <label>
                    <input type="checkbox" id="filter-unread" onchange="updateFilters()">
                    Seulement non lues
                </label>

                <button class="btn btn-secondary" onclick="loadHistory()">Charger historique</button>
                <button class="btn btn-secondary" onclick="markAllRead()">Tout marquer lu</button>
            </div>
        </div>

        <div class="notification-panel">
            <h3>📬 Notifications</h3>
            <div id="notifications-container">
                <!-- Les notifications apparaîtront ici -->
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let notifications = [];
        let isConnected = false;

        // Connexion WebSocket
        function connect() {
            const username = 'TestUser';
            const role = 'admin';
            const wsUrl = \`ws://localhost:8080/notifications?username=\${username}&role=\${role}\`;

            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                isConnected = true;
                updateConnectionStatus(true);
                console.log('Connecté aux notifications');

                // Charger l'historique initial
                loadHistory();
            };

            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);

                if (data.type === 'history') {
                    notifications = data.history.reverse();
                    displayNotifications();
                } else {
                    // Nouvelle notification
                    notifications.unshift(data);
                    displayNotifications();

                    // Notification du navigateur si supporté
                    if (Notification.permission === 'granted') {
                        new Notification(data.title, {
                            body: data.message,
                            icon: getNotificationIcon(data.type)
                        });
                    }
                }

                updateStats();
            };

            ws.onclose = function() {
                isConnected = false;
                updateConnectionStatus(false);
                console.log('Déconnecté des notifications');

                // Tentative de reconnexion après 3 secondes
                setTimeout(connect, 3000);
            };

            ws.onerror = function(error) {
                console.error('Erreur WebSocket:', error);
            };
        }

        function updateConnectionStatus(connected) {
            const statusEl = document.getElementById('connection-status');
            if (connected) {
                statusEl.textContent = 'Connecté aux notifications';
                statusEl.className = 'connection-status connected';
            } else {
                statusEl.textContent = 'Déconnecté - Tentative de reconnexion...';
                statusEl.className = 'connection-status disconnected';
            }
        }

        function displayNotifications() {
            const container = document.getElementById('notifications-container');
            container.innerHTML = '';

            notifications.forEach(notification => {
                const notifEl = createNotificationElement(notification);
                container.appendChild(notifEl);
            });
        }

        function createNotificationElement(notification) {
            const div = document.createElement('div');
            div.className = \`notification \${notification.type} \${!notification.is_read ? 'unread' : ''}\`;
            div.dataset.id = notification.id;

            const priorityBadge = \`<span class="priority-badge priority-\${notification.priority}">P\${notification.priority}</span>\`;
            const timestamp = new Date(notification.created_at).toLocaleString();

            div.innerHTML = \`
                <div class="notification-header">
                    <h4 class="notification-title">\${notification.title} \${priorityBadge}</h4>
                    <div class="notification-meta">\${timestamp}</div>
                </div>
                <div class="notification-message">\${notification.message}</div>
                <div class="notification-actions">
                    \${!notification.is_read ? '<button class="btn btn-primary" onclick="markAsRead(\\'\\' + notification.id + '\\')">Marquer comme lu</button>' : '<span style="color: #28a745;">✓ Lu</span>'}
                </div>
            \`;

            return div;
        }

        function markAsRead(notificationId) {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    command: 'mark_read',
                    notification_id: notificationId
                }));

                // Mettre à jour localement
                const notification = notifications.find(n => n.id === notificationId);
                if (notification) {
                    notification.is_read = true;
                    displayNotifications();
                    updateStats();
                }
            }
        }

        function markAllRead() {
            notifications.forEach(notification => {
                if (!notification.is_read) {
                    markAsRead(notification.id);
                }
            });
        }

        function updateFilters() {
            const filterType = document.getElementById('filter-type').value;
            const filterPriority = parseInt(document.getElementById('filter-priority').value);
            const filterUnread = document.getElementById('filter-unread').checked;

            const filters = {
                types: filterType ? [filterType] : [],
                min_priority: filterPriority,
                only_unread: filterUnread
            };

            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    command: 'update_filters',
                    filters: filters
                }));
            }
        }

        function loadHistory() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                ws.send(JSON.stringify({
                    command: 'get_history',
                    limit: 100
                }));
            }
        }

        function updateStats() {
            const total = notifications.length;
            const unread = notifications.filter(n => !n.is_read).length;
            const highPriority = notifications.filter(n => n.priority >= 4).length;

            document.getElementById('total-notifications').textContent = total;
            document.getElementById('unread-notifications').textContent = unread;
            document.getElementById('high-priority').textContent = highPriority;
        }

        function createNotification() {
            const type = document.getElementById('notif-type').value;
            const title = document.getElementById('notif-title').value;
            const message = document.getElementById('notif-message').value;
            const priority = parseInt(document.getElementById('notif-priority').value);

            if (!title || !message) {
                alert('Titre et message requis');
                return;
            }

            const notification = {
                type: type,
                title: title,
                message: message,
                priority: priority,
                category: 'test'
            };

            fetch('/api/notifications', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(notification)
            })
            .then(response => response.json())
            .then(data => {
                console.log('Notification créée:', data);
                // Vider le formulaire
                document.getElementById('notif-title').value = '';
                document.getElementById('notif-message').value = '';
            })
            .catch(error => {
                console.error('Erreur:', error);
            });
        }

        function createRandomNotification() {
            const types = ['info', 'success', 'warning', 'error'];
            const titles = [
                'Mise à jour système',
                'Nouveau message',
                'Alerte sécurité',
                'Tâche terminée',
                'Erreur détectée',
                'Backup complété'
            ];
            const messages = [
                'Le système a été mis à jour avec succès.',
                'Vous avez reçu un nouveau message.',
                'Une activité suspecte a été détectée.',
                'Votre tâche a été exécutée avec succès.',
                'Une erreur est survenue lors du traitement.',
                'La sauvegarde automatique est terminée.'
            ];

            const randomType = types[Math.floor(Math.random() * types.length)];
            const randomTitle = titles[Math.floor(Math.random() * titles.length)];
            const randomMessage = messages[Math.floor(Math.random() * messages.length)];
            const randomPriority = Math.floor(Math.random() * 5) + 1;

            const notification = {
                type: randomType,
                title: randomTitle,
                message: randomMessage,
                priority: randomPriority,
                category: 'auto'
            };

            fetch('/api/notifications', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(notification)
            })
            .then(response => response.json())
            .then(data => {
                console.log('Notification aléatoire créée:', data);
            })
            .catch(error => {
                console.error('Erreur:', error);
            });
        }

        function getNotificationIcon(type) {
            switch (type) {
                case 'success': return '✅';
                case 'warning': return '⚠️';
                case 'error': return '❌';
                default: return 'ℹ️';
            }
        }

        // Demander permission pour les notifications du navigateur
        function requestNotificationPermission() {
            if ('Notification' in window && Notification.permission === 'default') {
                Notification.requestPermission();
            }
        }

        // Initialisation
        document.addEventListener('DOMContentLoaded', function() {
            requestNotificationPermission();
            connect();

            // Créer une notification de test toutes les 30 secondes
            setInterval(createRandomNotification, 30000);
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Point d'entrée principal pour l'exercice 1
func mainNotifications() {
    hub := NewNotificationHub()
    go hub.run()

    // Routes
    http.HandleFunc("/", notificationUIHandler)
    http.HandleFunc("/notifications", notificationWSHandler(hub))
    http.HandleFunc("/api/notifications", createNotificationHandler(hub))

    fmt.Println("🔔 Système de notifications démarré sur http://localhost:8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Exercice 2 : Éditeur de texte collaboratif

### Solution complète

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

// Opération de modification de texte
type Operation struct {
    Type     string `json:"type"`     // insert, delete, retain
    Position int    `json:"position"` // Position dans le texte
    Content  string `json:"content"`  // Contenu pour insert
    Length   int    `json:"length"`   // Longueur pour delete/retain
    Author   string `json:"author"`   // Auteur de l'opération
    ID       string `json:"id"`       // ID unique de l'opération
}

// Transformation des opérations (OT - Operational Transform)
type OperationTransformer struct{}

func (ot *OperationTransformer) Transform(op1, op2 Operation) (Operation, Operation) {
    newOp1 := op1
    newOp2 := op2

    // Règles de transformation basiques
    if op1.Type == "insert" && op2.Type == "insert" {
        if op1.Position <= op2.Position {
            newOp2.Position += len(op1.Content)
        } else {
            newOp1.Position += len(op2.Content)
        }
    } else if op1.Type == "insert" && op2.Type == "delete" {
        if op1.Position <= op2.Position {
            newOp2.Position += len(op1.Content)
        } else if op1.Position < op2.Position+op2.Length {
            newOp2.Length += len(op1.Content)
        }
    } else if op1.Type == "delete" && op2.Type == "insert" {
        if op2.Position <= op1.Position {
            newOp1.Position += len(op2.Content)
        } else if op2.Position < op1.Position+op1.Length {
            newOp1.Length += len(op2.Content)
        }
    } else if op1.Type == "delete" && op2.Type == "delete" {
        if op1.Position < op2.Position {
            if op1.Position+op1.Length <= op2.Position {
                newOp2.Position -= op1.Length
            } else {
                overlap := (op1.Position + op1.Length) - op2.Position
                newOp2.Position = op1.Position
                newOp2.Length -= overlap
            }
        } else if op2.Position < op1.Position {
            if op2.Position+op2.Length <= op1.Position {
                newOp1.Position -= op2.Length
            } else {
                overlap := (op2.Position + op2.Length) - op1.Position
                newOp1.Position = op2.Position
                newOp1.Length -= overlap
            }
        }
    }

    return newOp1, newOp2
}

// Document collaboratif
type CollaborativeDocument struct {
    ID          string      `json:"id"`
    Title       string      `json:"title"`
    Content     string      `json:"content"`
    Operations  []Operation `json:"operations"`
    Version     int         `json:"version"`
    LastModified time.Time  `json:"last_modified"`
    mutex       sync.RWMutex
}

func NewCollaborativeDocument(id, title string) *CollaborativeDocument {
    return &CollaborativeDocument{
        ID:           id,
        Title:        title,
        Content:      "",
        Operations:   make([]Operation, 0),
        Version:      0,
        LastModified: time.Now(),
    }
}

func (doc *CollaborativeDocument) ApplyOperation(op Operation) error {
    doc.mutex.Lock()
    defer doc.mutex.Unlock()

    switch op.Type {
    case "insert":
        if op.Position < 0 || op.Position > len(doc.Content) {
            return fmt.Errorf("position invalide pour insertion")
        }
        doc.Content = doc.Content[:op.Position] + op.Content + doc.Content[op.Position:]

    case "delete":
        if op.Position < 0 || op.Position+op.Length > len(doc.Content) {
            return fmt.Errorf("position invalide pour suppression")
        }
        doc.Content = doc.Content[:op.Position] + doc.Content[op.Position+op.Length:]

    default:
        return fmt.Errorf("type d'opération non supporté: %s", op.Type)
    }

    doc.Operations = append(doc.Operations, op)
    doc.Version++
    doc.LastModified = time.Now()

    return nil
}

// Curseur d'utilisateur
type UserCursor struct {
    UserID   string `json:"user_id"`
    Username string `json:"username"`
    Position int    `json:"position"`
    Color    string `json:"color"`
}

// Client collaboratif
type CollaborativeClient struct {
    ID       string
    Username string
    Color    string
    DocID    string
    Conn     *websocket.Conn
    Send     chan interface{}
    Hub      *CollaborativeHub
    Cursor   UserCursor
}

// Hub collaboratif
type CollaborativeHub struct {
    documents  map[string]*CollaborativeDocument
    clients    map[string]map[*CollaborativeClient]bool // docID -> clients
    register   chan *CollaborativeClient
    unregister chan *CollaborativeClient
    operation  chan OperationMessage
    cursor     chan CursorMessage
    mutex      sync.RWMutex
    transformer *OperationTransformer
}

type OperationMessage struct {
    DocID     string    `json:"doc_id"`
    Operation Operation `json:"operation"`
    Client    *CollaborativeClient
}

type CursorMessage struct {
    DocID  string     `json:"doc_id"`
    Cursor UserCursor `json:"cursor"`
    Client *CollaborativeClient
}

func NewCollaborativeHub() *CollaborativeHub {
    return &CollaborativeHub{
        documents:   make(map[string]*CollaborativeDocument),
        clients:     make(map[string]map[*CollaborativeClient]bool),
        register:    make(chan *CollaborativeClient),
        unregister:  make(chan *CollaborativeClient),
        operation:   make(chan OperationMessage),
        cursor:      make(chan CursorMessage),
        transformer: &OperationTransformer{},
    }
}

func (h *CollaborativeHub) run() {
    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()

            // Créer le document s'il n'existe pas
            if h.documents[client.DocID] == nil {
                h.documents[client.DocID] = NewCollaborativeDocument(client.DocID, "Document Collaboratif")
            }

            // Ajouter le client
            if h.clients[client.DocID] == nil {
                h.clients[client.DocID] = make(map[*CollaborativeClient]bool)
            }
            h.clients[client.DocID][client] = true

            h.mutex.Unlock()

            // Envoyer l'état actuel du document
            doc := h.documents[client.DocID]
            doc.mutex.RLock()
            initialState := map[string]interface{}{
                "type":    "document_state",
                "content": doc.Content,
                "version": doc.Version,
                "title":   doc.Title,
            }
            doc.mutex.RUnlock()

            select {
            case client.Send <- initialState:
            default:
                close(client.Send)
                delete(h.clients[client.DocID], client)
            }

            // Notifier les autres clients
            h.broadcastToDocument(client.DocID, map[string]interface{}{
                "type": "user_joined",
                "user": map[string]string{
                    "id":       client.ID,
                    "username": client.Username,
                    "color":    client.Color,
                },
            }, client)

            log.Printf("👤 %s rejoint le document %s", client.Username, client.DocID)

        case client := <-h.unregister:
            h.mutex.Lock()
            if clients, ok := h.clients[client.DocID]; ok {
                if _, exists := clients[client]; exists {
                    delete(clients, client)
                    close(client.Send)

                    if len(clients) == 0 {
                        delete(h.clients, client.DocID)
                    }
                }
            }
            h.mutex.Unlock()

            // Notifier les autres clients
            h.broadcastToDocument(client.DocID, map[string]interface{}{
                "type": "user_left",
                "user": map[string]string{
                    "id":       client.ID,
                    "username": client.Username,
                },
            }, nil)

            log.Printf("👋 %s quitte le document %s", client.Username, client.DocID)

        case opMsg := <-h.operation:
            h.handleOperation(opMsg)

        case cursorMsg := <-h.cursor:
            h.handleCursor(cursorMsg)
        }
    }
}

func (h *CollaborativeHub) handleOperation(opMsg OperationMessage) {
    h.mutex.Lock()
    doc := h.documents[opMsg.DocID]
    h.mutex.Unlock()

    if doc == nil {
        return
    }

    // Transformer l'opération contre les opérations concurrentes
    transformedOp := opMsg.Operation

    // Appliquer l'opération au document
    if err := doc.ApplyOperation(transformedOp); err != nil {
        log.Printf("Erreur application opération: %v", err)
        return
    }

    // Diffuser l'opération aux autres clients
    operationBroadcast := map[string]interface{}{
        "type":      "operation",
        "operation": transformedOp,
        "version":   doc.Version,
    }

    h.broadcastToDocument(opMsg.DocID, operationBroadcast, opMsg.Client)
}

func (h *CollaborativeHub) handleCursor(cursorMsg CursorMessage) {
    // Mettre à jour le curseur du client
    cursorMsg.Client.Cursor = cursorMsg.Cursor

    // Diffuser la position du curseur aux autres clients
    cursorBroadcast := map[string]interface{}{
        "type":   "cursor",
        "cursor": cursorMsg.Cursor,
    }

    h.broadcastToDocument(cursorMsg.DocID, cursorBroadcast, cursorMsg.Client)
}

func (h *CollaborativeHub) broadcastToDocument(docID string, message interface{}, exclude *CollaborativeClient) {
    h.mutex.RLock()
    clients := h.clients[docID]
    h.mutex.RUnlock()

    for client := range clients {
        if client != exclude {
            select {
            case client.Send <- message:
            default:
                close(client.Send)
                delete(clients, client)
            }
        }
    }
}

// Handler WebSocket pour l'éditeur collaboratif
func collaborativeEditorHandler(hub *CollaborativeHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Erreur upgrade: %v", err)
            return
        }

        username := r.URL.Query().Get("username")
        if username == "" {
            username = fmt.Sprintf("User_%d", time.Now().Unix())
        }

        docID := r.URL.Query().Get("doc")
        if docID == "" {
            docID = "default"
        }

        // Couleurs prédéfinies pour les curseurs
        colors := []string{"#FF6B6B", "#4ECDC4", "#45B7D1", "#96CEB4", "#FFEAA7", "#DDA0DD", "#98D8C8", "#F7DC6F"}
        color := colors[len(username)%len(colors)]

        client := &CollaborativeClient{
            ID:       fmt.Sprintf("%s_%d", username, time.Now().UnixNano()),
            Username: username,
            Color:    color,
            DocID:    docID,
            Conn:     conn,
            Send:     make(chan interface{}, 256),
            Hub:      hub,
            Cursor: UserCursor{
                UserID:   fmt.Sprintf("%s_%d", username, time.Now().UnixNano()),
                Username: username,
                Position: 0,
                Color:    color,
            },
        }

        hub.register <- client

        go client.writePump()
        go client.readPump()
    }
}

func (c *CollaborativeClient) readPump() {
    defer func() {
        c.Hub.unregister <- c
        c.Conn.Close()
    }()

    c.Conn.SetReadLimit(512)
    c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.Conn.SetPongHandler(func(string) error {
        c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        var message map[string]interface{}
        err := c.Conn.ReadJSON(&message)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("Erreur WebSocket: %v", err)
            }
            break
        }

        c.handleMessage(message)
    }
}

func (c *CollaborativeClient) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.Send:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.Conn.WriteJSON(message); err != nil {
                log.Printf("Erreur écriture: %v", err)
                return
            }

        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

func (c *CollaborativeClient) handleMessage(message map[string]interface{}) {
    msgType, ok := message["type"].(string)
    if !ok {
        return
    }

    switch msgType {
    case "operation":
        if opData, ok := message["operation"].(map[string]interface{}); ok {
            operation := Operation{
                Type:     opData["type"].(string),
                Position: int(opData["position"].(float64)),
                Author:   c.Username,
                ID:       fmt.Sprintf("%s_%d", c.ID, time.Now().UnixNano()),
            }

            if content, exists := opData["content"]; exists {
                operation.Content = content.(string)
            }

            if length, exists := opData["length"]; exists {
                operation.Length = int(length.(float64))
            }

            c.Hub.operation <- OperationMessage{
                DocID:     c.DocID,
                Operation: operation,
                Client:    c,
            }
        }

    case "cursor":
        if cursorData, ok := message["cursor"].(map[string]interface{}); ok {
            cursor := UserCursor{
                UserID:   c.Cursor.UserID,
                Username: c.Username,
                Position: int(cursorData["position"].(float64)),
                Color:    c.Color,
            }

            c.Hub.cursor <- CursorMessage{
                DocID:  c.DocID,
                Cursor: cursor,
                Client: c,
            }
        }
    }
}

// Interface utilisateur pour l'éditeur collaboratif
func editorUIHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Éditeur Collaboratif</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 0; background: #f5f5f5; }
        .header { background: #2c3e50; color: white; padding: 15px; display: flex; justify-content: space-between; align-items: center; }
        .header h1 { margin: 0; }
        .users-list { display: flex; gap: 10px; align-items: center; }
        .user-badge {
            display: inline-block; padding: 5px 10px; border-radius: 15px;
            font-size: 0.8em; font-weight: bold; color: white;
        }

        .editor-container { height: calc(100vh - 70px); display: flex; }
        .editor-panel { flex: 1; background: white; position: relative; }
        .sidebar { width: 250px; background: #34495e; color: white; padding: 20px; }

        .editor-textarea {
            width: 100%; height: 100%; border: none; padding: 20px;
            font-family: 'Courier New', monospace; font-size: 14px; line-height: 1.5;
            resize: none; outline: none; box-sizing: border-box;
        }

        .cursor {
            position: absolute; width: 2px; height: 20px;
            background: currentColor; z-index: 10; pointer-events: none;
            animation: blink 1s infinite;
        }

        @keyframes blink {
            0%, 50% { opacity: 1; }
            51%, 100% { opacity: 0; }
        }

        .cursor-label {
            position: absolute; top: -25px; left: 0; padding: 2px 6px;
            border-radius: 3px; font-size: 10px; white-space: nowrap;
            background: currentColor; color: white;
        }

        .stats { margin-bottom: 20px; }
        .stats h3 { margin-top: 0; color: #ecf0f1; }
        .stat-item { margin: 10px 0; padding: 8px; background: #2c3e50; border-radius: 5px; }

        .operations-log { max-height: 300px; overflow-y: auto; }
        .operation {
            padding: 5px; margin: 2px 0; border-radius: 3px;
            font-size: 0.8em; background: rgba(255,255,255,0.1);
        }
        .operation.insert { border-left: 3px solid #27ae60; }
        .operation.delete { border-left: 3px solid #e74c3c; }

        .connection-status {
            padding: 10px; border-radius: 5px; text-align: center;
            margin-bottom: 20px; font-weight: bold;
        }
        .connected { background: #27ae60; }
        .disconnected { background: #e74c3c; }
    </style>
</head>
<body>
    <div class="header">
        <h1>✏️ Éditeur Collaboratif</h1>
        <div class="users-list" id="users-list">
            <!-- Utilisateurs connectés -->
        </div>
    </div>

    <div class="editor-container">
        <div class="editor-panel">
            <textarea id="editor" class="editor-textarea" placeholder="Commencez à taper..."></textarea>
            <div id="cursors-container">
                <!-- Curseurs des autres utilisateurs -->
            </div>
        </div>

        <div class="sidebar">
            <div id="connection-status" class="connection-status disconnected">
                Déconnecté
            </div>

            <div class="stats">
                <h3>📊 Statistiques</h3>
                <div class="stat-item">
                    Document: <strong id="doc-title">-</strong>
                </div>
                <div class="stat-item">
                    Version: <strong id="doc-version">0</strong>
                </div>
                <div class="stat-item">
                    Caractères: <strong id="char-count">0</strong>
                </div>
                <div class="stat-item">
                    Utilisateurs: <strong id="user-count">0</strong>
                </div>
            </div>

            <div>
                <h3>📝 Opérations récentes</h3>
                <div id="operations-log" class="operations-log">
                    <!-- Log des opérations -->
                </div>
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let isConnected = false;
        let currentUser = null;
        let documentVersion = 0;
        let users = new Map();
        let editor = null;
        let suppressNextEvent = false;

        // Paramètres de l'URL
        const urlParams = new URLSearchParams(window.location.search);
        const username = urlParams.get('username') || prompt('Votre nom:') || 'Utilisateur';
        const docId = urlParams.get('doc') || 'default';

        function connect() {
            const wsUrl = \`ws://localhost:8080/editor?username=\${encodeURIComponent(username)}&doc=\${encodeURIComponent(docId)}\`;
            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                isConnected = true;
                updateConnectionStatus(true);
                console.log('Connecté à l\\'éditeur collaboratif');
            };

            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                handleMessage(data);
            };

            ws.onclose = function() {
                isConnected = false;
                updateConnectionStatus(false);
                console.log('Déconnecté de l\\'éditeur');

                // Tentative de reconnexion
                setTimeout(connect, 3000);
            };

            ws.onerror = function(error) {
                console.error('Erreur WebSocket:', error);
            };
        }

        function handleMessage(data) {
            switch (data.type) {
                case 'document_state':
                    suppressNextEvent = true;
                    editor.value = data.content;
                    documentVersion = data.version;
                    document.getElementById('doc-title').textContent = data.title;
                    document.getElementById('doc-version').textContent = data.version;
                    updateCharCount();
                    break;

                case 'operation':
                    applyOperation(data.operation);
                    documentVersion = data.version;
                    document.getElementById('doc-version').textContent = data.version;
                    logOperation(data.operation);
                    break;

                case 'cursor':
                    updateUserCursor(data.cursor);
                    break;

                case 'user_joined':
                    users.set(data.user.id, data.user);
                    updateUsersList();
                    addOperationLog(\`\${data.user.username} a rejoint le document\`, 'system');
                    break;

                case 'user_left':
                    users.delete(data.user.id);
                    updateUsersList();
                    removeUserCursor(data.user.id);
                    addOperationLog(\`\${data.user.username} a quitté le document\`, 'system');
                    break;
            }
        }

        function applyOperation(operation) {
            suppressNextEvent = true;
            const currentPos = editor.selectionStart;

            switch (operation.type) {
                case 'insert':
                    const beforeInsert = editor.value.substring(0, operation.position);
                    const afterInsert = editor.value.substring(operation.position);
                    editor.value = beforeInsert + operation.content + afterInsert;

                    // Ajuster la position du curseur si nécessaire
                    if (currentPos >= operation.position) {
                        editor.setSelectionRange(currentPos + operation.content.length, currentPos + operation.content.length);
                    }
                    break;

                case 'delete':
                    const beforeDelete = editor.value.substring(0, operation.position);
                    const afterDelete = editor.value.substring(operation.position + operation.length);
                    editor.value = beforeDelete + afterDelete;

                    // Ajuster la position du curseur si nécessaire
                    if (currentPos > operation.position) {
                        const newPos = Math.max(operation.position, currentPos - operation.length);
                        editor.setSelectionRange(newPos, newPos);
                    }
                    break;
            }

            updateCharCount();
        }

        function sendOperation(type, position, content = '', length = 0) {
            if (!isConnected) return;

            const operation = {
                type: type,
                position: position,
                content: content,
                length: length
            };

            ws.send(JSON.stringify({
                type: 'operation',
                operation: operation
            }));
        }

        function sendCursor(position) {
            if (!isConnected) return;

            ws.send(JSON.stringify({
                type: 'cursor',
                cursor: {
                    position: position
                }
            }));
        }

        function updateUserCursor(cursor) {
            const container = document.getElementById('cursors-container');
            let cursorEl = document.getElementById(\`cursor-\${cursor.user_id}\`);

            if (!cursorEl) {
                cursorEl = document.createElement('div');
                cursorEl.id = \`cursor-\${cursor.user_id}\`;
                cursorEl.className = 'cursor';
                cursorEl.style.color = cursor.color;

                const label = document.createElement('div');
                label.className = 'cursor-label';
                label.textContent = cursor.username;
                cursorEl.appendChild(label);

                container.appendChild(cursorEl);
            }

            // Calculer la position du curseur
            const textBeforeCursor = editor.value.substring(0, cursor.position);
            const lines = textBeforeCursor.split('\\n');
            const lineNumber = lines.length - 1;
            const columnNumber = lines[lineNumber].length;

            const lineHeight = 21; // Hauteur de ligne approximative
            const charWidth = 8.4; // Largeur de caractère approximative

            cursorEl.style.top = (20 + lineNumber * lineHeight) + 'px';
            cursorEl.style.left = (20 + columnNumber * charWidth) + 'px';
        }

        function removeUserCursor(userId) {
            const cursorEl = document.getElementById(`cursor-${userId}`);
            if (cursorEl) {
                cursorEl.remove();
            }
        }

        function updateUsersList() {
            const container = document.getElementById('users-list');
            container.innerHTML = '';

            users.forEach(user => {
                const badge = document.createElement('div');
                badge.className = 'user-badge';
                badge.style.backgroundColor = user.color;
                badge.textContent = user.username;
                container.appendChild(badge);
            });

            document.getElementById('user-count').textContent = users.size;
        }

        function updateConnectionStatus(connected) {
            const statusEl = document.getElementById('connection-status');
            if (connected) {
                statusEl.textContent = 'Connecté';
                statusEl.className = 'connection-status connected';
            } else {
                statusEl.textContent = 'Déconnecté';
                statusEl.className = 'connection-status disconnected';
            }
        }

        function updateCharCount() {
            document.getElementById('char-count').textContent = editor.value.length;
        }

        function logOperation(operation) {
            addOperationLog(
                `${operation.author}: ${operation.type} "${operation.content || ''}" à la position ${operation.position}`,
                operation.type
            );
        }

        function addOperationLog(message, type) {
            const container = document.getElementById('operations-log');
            const logEl = document.createElement('div');
            logEl.className = `operation ${type}`;
            logEl.textContent = `${new Date().toLocaleTimeString()} - ${message}`;

            container.appendChild(logEl);
            container.scrollTop = container.scrollHeight;

            // Limiter le nombre d'entrées
            if (container.children.length > 50) {
                container.removeChild(container.firstChild);
            }
        }

        // Gestion des événements de l'éditeur
        function setupEditorEvents() {
            let lastContent = '';
            let lastSelection = 0;

            editor.addEventListener('input', function(e) {
                if (suppressNextEvent) {
                    suppressNextEvent = false;
                    lastContent = editor.value;
                    return;
                }

                const currentContent = editor.value;
                const currentSelection = editor.selectionStart;

                // Détecter le type d'opération
                if (currentContent.length > lastContent.length) {
                    // Insertion
                    const insertedText = currentContent.substring(lastSelection, lastSelection + (currentContent.length - lastContent.length));
                    sendOperation('insert', lastSelection, insertedText);
                } else if (currentContent.length < lastContent.length) {
                    // Suppression
                    const deletedLength = lastContent.length - currentContent.length;
                    sendOperation('delete', currentSelection, '', deletedLength);
                }

                lastContent = currentContent;
                updateCharCount();
            });

            editor.addEventListener('selectionchange', function() {
                sendCursor(editor.selectionStart);
            });

            editor.addEventListener('click', function() {
                sendCursor(editor.selectionStart);
            });

            editor.addEventListener('keyup', function() {
                sendCursor(editor.selectionStart);
            });

            // Mise à jour initiale
            lastContent = editor.value;
        }

        // Initialisation
        document.addEventListener('DOMContentLoaded', function() {
            editor = document.getElementById('editor');
            setupEditorEvents();
            connect();

            // Mettre à jour le titre de la page
            document.title = `Éditeur Collaboratif - ${username}`;
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Point d'entrée principal pour l'exercice 2
func mainCollaborativeEditor() {
    hub := NewCollaborativeHub()
    go hub.run()

    // Routes
    http.HandleFunc("/", editorUIHandler)
    http.HandleFunc("/editor", collaborativeEditorHandler(hub))

    fmt.Println("✏️ Éditeur collaboratif démarré sur http://localhost:8080")
    fmt.Println("📝 Testez avec plusieurs onglets : http://localhost:8080?username=Alice&doc=test")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

## Exercice 3 : Dashboard en temps réel

### Solution complète

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "math"
    "math/rand"
    "net/http"
    "sync"
    "time"

    "github.com/gorilla/websocket"
)

// Métriques du système
type SystemMetrics struct {
    Timestamp       time.Time `json:"timestamp"`
    CPUUsage        float64   `json:"cpu_usage"`
    MemoryUsage     float64   `json:"memory_usage"`
    DiskUsage       float64   `json:"disk_usage"`
    NetworkIn       float64   `json:"network_in"`
    NetworkOut      float64   `json:"network_out"`
    ActiveUsers     int       `json:"active_users"`
    RequestsPerSec  int       `json:"requests_per_sec"`
    ErrorRate       float64   `json:"error_rate"`
    ResponseTime    float64   `json:"response_time"`
    DatabaseConnections int   `json:"database_connections"`
    CacheHitRate    float64   `json:"cache_hit_rate"`
}

// Alerte système
type Alert struct {
    ID          string    `json:"id"`
    Type        string    `json:"type"`        // warning, critical, info
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Metric      string    `json:"metric"`
    Value       float64   `json:"value"`
    Threshold   float64   `json:"threshold"`
    Timestamp   time.Time `json:"timestamp"`
    Resolved    bool      `json:"resolved"`
}

// Configuration des seuils d'alerte
type AlertConfig struct {
    CPUWarning     float64 `json:"cpu_warning"`
    CPUCritical    float64 `json:"cpu_critical"`
    MemoryWarning  float64 `json:"memory_warning"`
    MemoryCritical float64 `json:"memory_critical"`
    ErrorRateMax   float64 `json:"error_rate_max"`
    ResponseTimeMax float64 `json:"response_time_max"`
}

// Générateur de métriques (simulation)
type MetricsGenerator struct {
    mutex         sync.RWMutex
    currentMetrics SystemMetrics
    alerts        []Alert
    alertConfig   AlertConfig
    history       []SystemMetrics
    maxHistory    int
}

func NewMetricsGenerator() *MetricsGenerator {
    return &MetricsGenerator{
        alerts: make([]Alert, 0),
        alertConfig: AlertConfig{
            CPUWarning:      70.0,
            CPUCritical:     90.0,
            MemoryWarning:   80.0,
            MemoryCritical:  95.0,
            ErrorRateMax:    5.0,
            ResponseTimeMax: 1000.0,
        },
        history:    make([]SystemMetrics, 0),
        maxHistory: 100,
    }
}

func (mg *MetricsGenerator) generateMetrics() SystemMetrics {
    now := time.Now()

    // Simulation de métriques réalistes avec tendances
    baseTime := float64(now.Unix() % 3600) // Cycle d'une heure

    metrics := SystemMetrics{
        Timestamp:       now,
        CPUUsage:        30 + 40*math.Sin(baseTime/600) + 10*rand.Float64(),
        MemoryUsage:     45 + 25*math.Sin(baseTime/900) + 15*rand.Float64(),
        DiskUsage:       60 + 10*math.Sin(baseTime/1800) + 5*rand.Float64(),
        NetworkIn:       100 + 50*math.Sin(baseTime/300) + 20*rand.Float64(),
        NetworkOut:      80 + 40*math.Sin(baseTime/300) + 15*rand.Float64(),
        ActiveUsers:     100 + int(50*math.Sin(baseTime/1200)) + rand.Intn(20),
        RequestsPerSec:  500 + int(200*math.Sin(baseTime/600)) + rand.Intn(100),
        ErrorRate:       1 + 3*rand.Float64(),
        ResponseTime:    200 + 300*rand.Float64(),
        DatabaseConnections: 20 + rand.Intn(30),
        CacheHitRate:    85 + 10*rand.Float64(),
    }

    // Simuler des pics occasionnels
    if rand.Float64() < 0.1 {
        metrics.CPUUsage = math.Min(100, metrics.CPUUsage+30)
        metrics.ResponseTime += 500
        metrics.ErrorRate += 2
    }

    mg.mutex.Lock()
    mg.currentMetrics = metrics
    mg.history = append(mg.history, metrics)

    // Limiter l'historique
    if len(mg.history) > mg.maxHistory {
        mg.history = mg.history[1:]
    }
    mg.mutex.Unlock()

    // Vérifier les seuils d'alerte
    mg.checkAlerts(metrics)

    return metrics
}

func (mg *MetricsGenerator) checkAlerts(metrics SystemMetrics) {
    alerts := make([]Alert, 0)

    // Alerte CPU
    if metrics.CPUUsage > mg.alertConfig.CPUCritical {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("cpu_critical_%d", time.Now().Unix()),
            Type:        "critical",
            Title:       "CPU Usage Critical",
            Description: fmt.Sprintf("CPU usage is at %.1f%%", metrics.CPUUsage),
            Metric:      "cpu_usage",
            Value:       metrics.CPUUsage,
            Threshold:   mg.alertConfig.CPUCritical,
            Timestamp:   time.Now(),
        })
    } else if metrics.CPUUsage > mg.alertConfig.CPUWarning {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("cpu_warning_%d", time.Now().Unix()),
            Type:        "warning",
            Title:       "CPU Usage High",
            Description: fmt.Sprintf("CPU usage is at %.1f%%", metrics.CPUUsage),
            Metric:      "cpu_usage",
            Value:       metrics.CPUUsage,
            Threshold:   mg.alertConfig.CPUWarning,
            Timestamp:   time.Now(),
        })
    }

    // Alerte Mémoire
    if metrics.MemoryUsage > mg.alertConfig.MemoryCritical {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("memory_critical_%d", time.Now().Unix()),
            Type:        "critical",
            Title:       "Memory Usage Critical",
            Description: fmt.Sprintf("Memory usage is at %.1f%%", metrics.MemoryUsage),
            Metric:      "memory_usage",
            Value:       metrics.MemoryUsage,
            Threshold:   mg.alertConfig.MemoryCritical,
            Timestamp:   time.Now(),
        })
    } else if metrics.MemoryUsage > mg.alertConfig.MemoryWarning {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("memory_warning_%d", time.Now().Unix()),
            Type:        "warning",
            Title:       "Memory Usage High",
            Description: fmt.Sprintf("Memory usage is at %.1f%%", metrics.MemoryUsage),
            Metric:      "memory_usage",
            Value:       metrics.MemoryUsage,
            Threshold:   mg.alertConfig.MemoryWarning,
            Timestamp:   time.Now(),
        })
    }

    // Alerte Taux d'erreur
    if metrics.ErrorRate > mg.alertConfig.ErrorRateMax {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("error_rate_%d", time.Now().Unix()),
            Type:        "warning",
            Title:       "High Error Rate",
            Description: fmt.Sprintf("Error rate is at %.1f%%", metrics.ErrorRate),
            Metric:      "error_rate",
            Value:       metrics.ErrorRate,
            Threshold:   mg.alertConfig.ErrorRateMax,
            Timestamp:   time.Now(),
        })
    }

    // Alerte Temps de réponse
    if metrics.ResponseTime > mg.alertConfig.ResponseTimeMax {
        alerts = append(alerts, Alert{
            ID:          fmt.Sprintf("response_time_%d", time.Now().Unix()),
            Type:        "warning",
            Title:       "High Response Time",
            Description: fmt.Sprintf("Response time is %.1fms", metrics.ResponseTime),
            Metric:      "response_time",
            Value:       metrics.ResponseTime,
            Threshold:   mg.alertConfig.ResponseTimeMax,
            Timestamp:   time.Now(),
        })
    }

    if len(alerts) > 0 {
        mg.mutex.Lock()
        mg.alerts = append(mg.alerts, alerts...)

        // Limiter le nombre d'alertes
        if len(mg.alerts) > 50 {
            mg.alerts = mg.alerts[len(mg.alerts)-50:]
        }
        mg.mutex.Unlock()
    }
}

func (mg *MetricsGenerator) GetCurrentMetrics() SystemMetrics {
    mg.mutex.RLock()
    defer mg.mutex.RUnlock()
    return mg.currentMetrics
}

func (mg *MetricsGenerator) GetHistory() []SystemMetrics {
    mg.mutex.RLock()
    defer mg.mutex.RUnlock()

    history := make([]SystemMetrics, len(mg.history))
    copy(history, mg.history)
    return history
}

func (mg *MetricsGenerator) GetAlerts() []Alert {
    mg.mutex.RLock()
    defer mg.mutex.RUnlock()

    alerts := make([]Alert, len(mg.alerts))
    copy(alerts, mg.alerts)
    return alerts
}

// Client dashboard
type DashboardClient struct {
    ID   string
    Conn *websocket.Conn
    Send chan interface{}
    Hub  *DashboardHub
}

// Hub dashboard
type DashboardHub struct {
    clients   map[*DashboardClient]bool
    register  chan *DashboardClient
    unregister chan *DashboardClient
    broadcast chan interface{}
    metrics   *MetricsGenerator
    mutex     sync.RWMutex
}

func NewDashboardHub() *DashboardHub {
    return &DashboardHub{
        clients:    make(map[*DashboardClient]bool),
        register:   make(chan *DashboardClient),
        unregister: make(chan *DashboardClient),
        broadcast:  make(chan interface{}),
        metrics:    NewMetricsGenerator(),
    }
}

func (h *DashboardHub) run() {
    // Démarrer la génération de métriques
    go h.generateMetricsLoop()

    for {
        select {
        case client := <-h.register:
            h.mutex.Lock()
            h.clients[client] = true
            h.mutex.Unlock()

            log.Printf("📊 Client dashboard connecté")

            // Envoyer l'état initial
            initialData := map[string]interface{}{
                "type":    "initial_data",
                "metrics": h.metrics.GetCurrentMetrics(),
                "history": h.metrics.GetHistory(),
                "alerts":  h.metrics.GetAlerts(),
            }

            select {
            case client.Send <- initialData:
            default:
                close(client.Send)
                delete(h.clients, client)
            }

        case client := <-h.unregister:
            h.mutex.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.Send)
            }
            h.mutex.Unlock()

            log.Printf("📊 Client dashboard déconnecté")

        case message := <-h.broadcast:
            h.broadcastToAll(message)
        }
    }
}

func (h *DashboardHub) generateMetricsLoop() {
    ticker := time.NewTicker(2 * time.Second) // Mise à jour toutes les 2 secondes
    defer ticker.Stop()

    for range ticker.C {
        metrics := h.metrics.generateMetrics()

        // Diffuser les nouvelles métriques
        h.broadcast <- map[string]interface{}{
            "type":    "metrics_update",
            "metrics": metrics,
        }

        // Diffuser les nouvelles alertes
        alerts := h.metrics.GetAlerts()
        if len(alerts) > 0 {
            recentAlerts := make([]Alert, 0)
            cutoff := time.Now().Add(-10 * time.Second)

            for _, alert := range alerts {
                if alert.Timestamp.After(cutoff) {
                    recentAlerts = append(recentAlerts, alert)
                }
            }

            if len(recentAlerts) > 0 {
                h.broadcast <- map[string]interface{}{
                    "type":   "new_alerts",
                    "alerts": recentAlerts,
                }
            }
        }
    }
}

func (h *DashboardHub) broadcastToAll(message interface{}) {
    h.mutex.RLock()
    defer h.mutex.RUnlock()

    for client := range h.clients {
        select {
        case client.Send <- message:
        default:
            close(client.Send)
            delete(h.clients, client)
        }
    }
}

// Handler WebSocket pour le dashboard
func dashboardWSHandler(hub *DashboardHub) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Erreur upgrade: %v", err)
            return
        }

        client := &DashboardClient{
            ID:   fmt.Sprintf("dashboard_%d", time.Now().UnixNano()),
            Conn: conn,
            Send: make(chan interface{}, 256),
            Hub:  hub,
        }

        hub.register <- client

        go client.writePump()
        go client.readPump()
    }
}

func (c *DashboardClient) readPump() {
    defer func() {
        c.Hub.unregister <- c
        c.Conn.Close()
    }()

    c.Conn.SetReadLimit(512)
    c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.Conn.SetPongHandler(func(string) error {
        c.Conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })

    for {
        var message map[string]interface{}
        err := c.Conn.ReadJSON(&message)
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("Erreur WebSocket: %v", err)
            }
            break
        }

        // Traiter les commandes du client si nécessaire
    }
}

func (c *DashboardClient) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.Conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.Send:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

            if !ok {
                c.Conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            if err := c.Conn.WriteJSON(message); err != nil {
                log.Printf("Erreur écriture: %v", err)
                return
            }

        case <-ticker.C:
            c.Conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.Conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}

// Interface utilisateur pour le dashboard
func dashboardUIHandler(w http.ResponseWriter, r *http.Request) {
    html := `<!DOCTYPE html>
<html>
<head>
    <title>Dashboard Temps Réel</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #0f1419; color: #ffffff; }

        .header { background: #1a1f2e; padding: 20px; border-bottom: 2px solid #2d3748; }
        .header h1 { color: #4299e1; text-align: center; }

        .container { padding: 20px; }
        .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; margin-bottom: 30px; }

        .card { background: #1a202c; border: 1px solid #2d3748; border-radius: 12px; padding: 20px; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3); }
        .card h3 { color: #63b3ed; margin-bottom: 15px; display: flex; align-items: center; gap: 10px; }

        .metric-value { font-size: 2.5em; font-weight: bold; text-align: center; margin: 15px 0; }
        .metric-label { text-align: center; color: #a0aec0; font-size: 0.9em; }
        .metric-unit { font-size: 0.6em; color: #718096; }

        .cpu { color: #f56565; }
        .memory { color: #48bb78; }
        .disk { color: #ed8936; }
        .network { color: #4299e1; }
        .users { color: #9f7aea; }
        .performance { color: #38b2ac; }

        .chart-container { position: relative; height: 300px; margin: 20px 0; }
        .chart-small { height: 200px; }

        .alerts-container { max-height: 400px; overflow-y: auto; }
        .alert { padding: 12px; margin: 8px 0; border-radius: 8px; border-left: 4px solid; }
        .alert.critical { background: rgba(245, 101, 101, 0.1); border-color: #f56565; }
        .alert.warning { background: rgba(237, 137, 54, 0.1); border-color: #ed8936; }
        .alert.info { background: rgba(66, 153, 225, 0.1); border-color: #4299e1; }

        .alert-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px; }
        .alert-title { font-weight: bold; }
        .alert-time { font-size: 0.8em; color: #a0aec0; }
        .alert-description { font-size: 0.9em; }

        .status-indicator { display: inline-block; width: 12px; height: 12px; border-radius: 50%; margin-right: 8px; }
        .status-online { background: #48bb78; animation: pulse 2s infinite; }
        .status-offline { background: #f56565; }

        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
            100% { opacity: 1; }
        }

        .stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 15px; }
        .stat-item { text-align: center; padding: 10px; background: #2d3748; border-radius: 8px; }
        .stat-value { font-size: 1.5em; font-weight: bold; color: #4299e1; }
        .stat-label { font-size: 0.8em; color: #a0aec0; margin-top: 5px; }

        .connection-status { padding: 10px; border-radius: 8px; text-align: center; margin-bottom: 20px; }
        .connected { background: rgba(72, 187, 120, 0.2); border: 1px solid #48bb78; }
        .disconnected { background: rgba(245, 101, 101, 0.2); border: 1px solid #f56565; }
    </style>
</head>
<body>
    <div class="header">
        <h1>📊 Dashboard Système - Temps Réel</h1>
        <div id="connection-status" class="connection-status disconnected">
            <span class="status-indicator status-offline"></span>
            Connexion en cours...
        </div>
    </div>

    <div class="container">
        <!-- Métriques principales -->
        <div class="grid">
            <div class="card">
                <h3>🖥️ CPU</h3>
                <div class="metric-value cpu" id="cpu-value">--<span class="metric-unit">%</span></div>
                <div class="metric-label">Utilisation processeur</div>
                <div class="chart-container chart-small">
                    <canvas id="cpu-chart"></canvas>
                </div>
            </div>

            <div class="card">
                <h3>💾 Mémoire</h3>
                <div class="metric-value memory" id="memory-value">--<span class="metric-unit">%</span></div>
                <div class="metric-label">Utilisation mémoire</div>
                <div class="chart-container chart-small">
                    <canvas id="memory-chart"></canvas>
                </div>
            </div>

            <div class="card">
                <h3>💽 Disque</h3>
                <div class="metric-value disk" id="disk-value">--<span class="metric-unit">%</span></div>
                <div class="metric-label">Utilisation disque</div>
            </div>

            <div class="card">
                <h3>👥 Utilisateurs</h3>
                <div class="metric-value users" id="users-value">--</div>
                <div class="metric-label">Utilisateurs actifs</div>
            </div>
        </div>

        <!-- Métriques réseau et performance -->
        <div class="grid">
            <div class="card">
                <h3>🌐 Réseau</h3>
                <div class="stats-grid">
                    <div class="stat-item">
                        <div class="stat-value" id="network-in">--</div>
                        <div class="stat-label">Entrant (MB/s)</div>
                    </div>
                    <div class="stat-item">
                        <div class="stat-value" id="network-out">--</div>
                        <div class="stat-label">Sortant (MB/s)</div>
                    </div>
                </div>
                <div class="chart-container chart-small">
                    <canvas id="network-chart"></canvas>
                </div>
            </div>

            <div class="card">
                <h3>⚡ Performance</h3>
                <div class="stats-grid">
                    <div class="stat-item">
                        <div class="stat-value" id="requests-value">--</div>
                        <div class="stat-label">Req/sec</div>
                    </div>
                    <div class="stat-item">
                        <div class="stat-value" id="response-time">--</div>
                        <div class="stat-label">Temps réponse (ms)</div>
                    </div>
                    <div class="stat-item">
                        <div class="stat-value" id="error-rate">--</div>
                        <div class="stat-label">Taux erreur (%)</div>
                    </div>
                    <div class="stat-item">
                        <div class="stat-value" id="cache-hit">--</div>
                        <div class="stat-label">Cache hit (%)</div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Graphiques de tendances -->
        <div class="grid">
            <div class="card" style="grid-column: 1 / -1;">
                <h3>📈 Tendances Système</h3>
                <div class="chart-container">
                    <canvas id="trends-chart"></canvas>
                </div>
            </div>
        </div>

        <!-- Alertes -->
        <div class="grid">
            <div class="card" style="grid-column: 1 / -1;">
                <h3>🚨 Alertes Système</h3>
                <div id="alerts-container" class="alerts-container">
                    <!-- Les alertes apparaîtront ici -->
                </div>
            </div>
        </div>
    </div>

    <script>
        let ws = null;
        let isConnected = false;
        let charts = {};
        let metricsHistory = [];
        let maxHistoryPoints = 50;

        // Configuration des graphiques
        const chartConfig = {
            type: 'line',
            options: {
                responsive: true,
                maintainAspectRatio: false,
                scales: {
                    x: {
                        type: 'time',
                        time: {
                            unit: 'second',
                            displayFormats: {
                                second: 'HH:mm:ss'
                            }
                        },
                        grid: {
                            color: '#2d3748'
                        },
                        ticks: {
                            color: '#a0aec0'
                        }
                    },
                    y: {
                        beginAtZero: true,
                        max: 100,
                        grid: {
                            color: '#2d3748'
                        },
                        ticks: {
                            color: '#a0aec0'
                        }
                    }
                },
                plugins: {
                    legend: {
                        labels: {
                            color: '#a0aec0'
                        }
                    }
                },
                elements: {
                    point: {
                        radius: 0
                    },
                    line: {
                        tension: 0.4
                    }
                },
                animation: {
                    duration: 0
                }
            }
        };

        function initCharts() {
            // Graphique CPU
            charts.cpu = new Chart(document.getElementById('cpu-chart'), {
                ...chartConfig,
                data: {
                    datasets: [{
                        label: 'CPU %',
                        data: [],
                        borderColor: '#f56565',
                        backgroundColor: 'rgba(245, 101, 101, 0.1)',
                        fill: true
                    }]
                }
            });

            // Graphique Mémoire
            charts.memory = new Chart(document.getElementById('memory-chart'), {
                ...chartConfig,
                data: {
                    datasets: [{
                        label: 'Mémoire %',
                        data: [],
                        borderColor: '#48bb78',
                        backgroundColor: 'rgba(72, 187, 120, 0.1)',
                        fill: true
                    }]
                }
            });

            // Graphique Réseau
            charts.network = new Chart(document.getElementById('network-chart'), {
                ...chartConfig,
                data: {
                    datasets: [{
                        label: 'Entrant (MB/s)',
                        data: [],
                        borderColor: '#4299e1',
                        backgroundColor: 'rgba(66, 153, 225, 0.1)'
                    }, {
                        label: 'Sortant (MB/s)',
                        data: [],
                        borderColor: '#9f7aea',
                        backgroundColor: 'rgba(159, 122, 234, 0.1)'
                    }]
                },
                options: {
                    ...chartConfig.options,
                    scales: {
                        ...chartConfig.options.scales,
                        y: {
                            ...chartConfig.options.scales.y,
                            max: 200
                        }
                    }
                }
            });

            // Graphique de tendances principal
            charts.trends = new Chart(document.getElementById('trends-chart'), {
                ...chartConfig,
                data: {
                    datasets: [{
                        label: 'CPU %',
                        data: [],
                        borderColor: '#f56565',
                        backgroundColor: 'rgba(245, 101, 101, 0.1)'
                    }, {
                        label: 'Mémoire %',
                        data: [],
                        borderColor: '#48bb78',
                        backgroundColor: 'rgba(72, 187, 120, 0.1)'
                    }, {
                        label: 'Taux erreur %',
                        data: [],
                        borderColor: '#ed8936',
                        backgroundColor: 'rgba(237, 137, 54, 0.1)',
                        yAxisID: 'y1'
                    }]
                },
                options: {
                    ...chartConfig.options,
                    scales: {
                        ...chartConfig.options.scales,
                        y1: {
                            type: 'linear',
                            display: true,
                            position: 'right',
                            max: 10,
                            grid: {
                                drawOnChartArea: false,
                                color: '#2d3748'
                            },
                            ticks: {
                                color: '#a0aec0'
                            }
                        }
                    }
                }
            });
        }

        function connect() {
            const wsUrl = 'ws://localhost:8080/dashboard';
            ws = new WebSocket(wsUrl);

            ws.onopen = function() {
                isConnected = true;
                updateConnectionStatus(true);
                console.log('Connecté au dashboard');
            };

            ws.onmessage = function(event) {
                const data = JSON.parse(event.data);
                handleMessage(data);
            };

            ws.onclose = function() {
                isConnected = false;
                updateConnectionStatus(false);
                console.log('Déconnecté du dashboard');

                // Tentative de reconnexion après 3 secondes
                setTimeout(connect, 3000);
            };

            ws.onerror = function(error) {
                console.error('Erreur WebSocket:', error);
            };
        }

        function handleMessage(data) {
            switch (data.type) {
                case 'initial_data':
                    loadInitialData(data);
                    break;

                case 'metrics_update':
                    updateMetrics(data.metrics);
                    break;

                case 'new_alerts':
                    addAlerts(data.alerts);
                    break;
            }
        }

        function loadInitialData(data) {
            // Charger l'historique
            metricsHistory = data.history;
            updateAllCharts();

            // Charger les métriques actuelles
            updateMetrics(data.metrics);

            // Charger les alertes
            displayAlerts(data.alerts);
        }

        function updateMetrics(metrics) {
            // Ajouter aux métriques historiques
            metricsHistory.push(metrics);

            // Limiter l'historique
            if (metricsHistory.length > maxHistoryPoints) {
                metricsHistory.shift();
            }

            // Mettre à jour l'affichage
            updateDisplayValues(metrics);
            updateAllCharts();
        }

        function updateDisplayValues(metrics) {
            document.getElementById('cpu-value').innerHTML = `${metrics.cpu_usage.toFixed(1)}<span class="metric-unit">%</span>`;
            document.getElementById('memory-value').innerHTML = `${metrics.memory_usage.toFixed(1)}<span class="metric-unit">%</span>`;
            document.getElementById('disk-value').innerHTML = `${metrics.disk_usage.toFixed(1)}<span class="metric-unit">%</span>`;
            document.getElementById('users-value').textContent = metrics.active_users;

            document.getElementById('network-in').textContent = metrics.network_in.toFixed(1);
            document.getElementById('network-out').textContent = metrics.network_out.toFixed(1);

            document.getElementById('requests-value').textContent = metrics.requests_per_sec;
            document.getElementById('response-time').textContent = metrics.response_time.toFixed(0);
            document.getElementById('error-rate').textContent = metrics.error_rate.toFixed(1);
            document.getElementById('cache-hit').textContent = metrics.cache_hit_rate.toFixed(1);

            // Appliquer des couleurs selon les seuils
            applyThresholdColors(metrics);
        }

        function applyThresholdColors(metrics) {
            const cpuEl = document.getElementById('cpu-value');
            if (metrics.cpu_usage > 90) {
                cpuEl.style.color = '#f56565';
            } else if (metrics.cpu_usage > 70) {
                cpuEl.style.color = '#ed8936';
            } else {
                cpuEl.style.color = '#48bb78';
            }

            const memoryEl = document.getElementById('memory-value');
            if (metrics.memory_usage > 95) {
                memoryEl.style.color = '#f56565';
            } else if (metrics.memory_usage > 80) {
                memoryEl.style.color = '#ed8936';
            } else {
                memoryEl.style.color = '#48bb78';
            }

            const errorEl = document.getElementById('error-rate');
            if (metrics.error_rate > 5) {
                errorEl.style.color = '#f56565';
            } else if (metrics.error_rate > 2) {
                errorEl.style.color = '#ed8936';
            } else {
                errorEl.style.color = '#48bb78';
            }
        }

        function updateAllCharts() {
            if (metricsHistory.length === 0) return;

            const timeLabels = metricsHistory.map(m => new Date(m.timestamp));

            // Graphique CPU
            charts.cpu.data.labels = timeLabels;
            charts.cpu.data.datasets[0].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.cpu_usage
            }));
            charts.cpu.update('none');

            // Graphique Mémoire
            charts.memory.data.labels = timeLabels;
            charts.memory.data.datasets[0].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.memory_usage
            }));
            charts.memory.update('none');

            // Graphique Réseau
            charts.network.data.labels = timeLabels;
            charts.network.data.datasets[0].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.network_in
            }));
            charts.network.data.datasets[1].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.network_out
            }));
            charts.network.update('none');

            // Graphique de tendances
            charts.trends.data.labels = timeLabels;
            charts.trends.data.datasets[0].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.cpu_usage
            }));
            charts.trends.data.datasets[1].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.memory_usage
            }));
            charts.trends.data.datasets[2].data = metricsHistory.map(m => ({
                x: new Date(m.timestamp),
                y: m.error_rate
            }));
            charts.trends.update('none');
        }

        function displayAlerts(alerts) {
            const container = document.getElementById('alerts-container');
            container.innerHTML = '';

            if (alerts.length === 0) {
                container.innerHTML = '<div style="text-align: center; color: #48bb78; padding: 20px;">✅ Aucune alerte active</div>';
                return;
            }

            // Trier par timestamp (plus récent en premier)
            alerts.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));

            alerts.forEach(alert => {
                const alertEl = createAlertElement(alert);
                container.appendChild(alertEl);
            });
        }

        function addAlerts(newAlerts) {
            newAlerts.forEach(alert => {
                const container = document.getElementById('alerts-container');

                // Vérifier si l'alerte existe déjà
                if (!document.getElementById(`alert-${alert.id}`)) {
                    const alertEl = createAlertElement(alert);
                    container.insertBefore(alertEl, container.firstChild);

                    // Animation d'entrée
                    alertEl.style.opacity = '0';
                    alertEl.style.transform = 'translateY(-20px)';

                    setTimeout(() => {
                        alertEl.style.transition = 'all 0.3s ease';
                        alertEl.style.opacity = '1';
                        alertEl.style.transform = 'translateY(0)';
                    }, 100);

                    // Notification sonore (optionnelle)
                    if (alert.type === 'critical') {
                        playNotificationSound();
                    }
                }
            });

            // Limiter le nombre d'alertes affichées
            const container = document.getElementById('alerts-container');
            while (container.children.length > 20) {
                container.removeChild(container.lastChild);
            }
        }

        function createAlertElement(alert) {
            const div = document.createElement('div');
            div.className = `alert ${alert.type}`;
            div.id = `alert-${alert.id}`;

            const icon = getAlertIcon(alert.type);
            const timestamp = new Date(alert.timestamp).toLocaleString();

            div.innerHTML = `
                <div class="alert-header">
                    <div class="alert-title">${icon} ${alert.title}</div>
                    <div class="alert-time">${timestamp}</div>
                </div>
                <div class="alert-description">${alert.description}</div>
            `;

            return div;
        }

        function getAlertIcon(type) {
            switch (type) {
                case 'critical': return '🚨';
                case 'warning': return '⚠️';
                case 'info': return 'ℹ️';
                default: return '🔔';
            }
        }

        function playNotificationSound() {
            // Créer un son de notification simple
            const audioContext = new (window.AudioContext || window.webkitAudioContext)();
            const oscillator = audioContext.createOscillator();
            const gainNode = audioContext.createGain();

            oscillator.connect(gainNode);
            gainNode.connect(audioContext.destination);

            oscillator.frequency.setValueAtTime(800, audioContext.currentTime);
            oscillator.frequency.setValueAtTime(600, audioContext.currentTime + 0.1);

            gainNode.gain.setValueAtTime(0.3, audioContext.currentTime);
            gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5);

            oscillator.start(audioContext.currentTime);
            oscillator.stop(audioContext.currentTime + 0.5);
        }

        function updateConnectionStatus(connected) {
            const statusEl = document.getElementById('connection-status');
            const indicator = statusEl.querySelector('.status-indicator');

            if (connected) {
                statusEl.textContent = 'Connecté - Données en temps réel';
                statusEl.className = 'connection-status connected';
                indicator.className = 'status-indicator status-online';
                statusEl.insertBefore(indicator, statusEl.firstChild);
            } else {
                statusEl.textContent = 'Déconnecté - Tentative de reconnexion...';
                statusEl.className = 'connection-status disconnected';
                indicator.className = 'status-indicator status-offline';
                statusEl.insertBefore(indicator, statusEl.firstChild);
            }
        }

        // Initialisation
        document.addEventListener('DOMContentLoaded', function() {
            initCharts();
            connect();

            // Mise à jour de l'heure en temps réel
            setInterval(() => {
                const now = new Date().toLocaleString('fr-FR', {
                    hour: '2-digit',
                    minute: '2-digit',
                    second: '2-digit'
                });
                // Vous pouvez afficher l'heure quelque part si nécessaire
            }, 1000);
        });

        // Gérer la visibilité de la page pour économiser les ressources
        document.addEventListener('visibilitychange', function() {
            if (document.hidden) {
                // Réduire la fréquence de mise à jour si la page n'est pas visible
                console.log('Page cachée - ralentissement des mises à jour');
            } else {
                // Reprendre les mises à jour normales
                console.log('Page visible - reprise des mises à jour normales');
            }
        });
    </script>
</body>
</html>`

    w.Header().Set("Content-Type", "text/html")
    w.Write([]byte(html))
}

// Point d'entrée principal pour l'exercice 3
func mainDashboard() {
    hub := NewDashboardHub()
    go hub.run()

    // Routes
    http.HandleFunc("/", dashboardUIHandler)
    http.HandleFunc("/dashboard", dashboardWSHandler(hub))

    // API pour récupérer les métriques actuelles
    http.HandleFunc("/api/metrics", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(hub.metrics.GetCurrentMetrics())
    })

    // API pour récupérer l'historique
    http.HandleFunc("/api/history", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(hub.metrics.GetHistory())
    })

    // API pour récupérer les alertes
    http.HandleFunc("/api/alerts", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(hub.metrics.GetAlerts())
    })

    fmt.Println("📊 Dashboard temps réel démarré sur http://localhost:8080")
    fmt.Println("📈 Métriques mises à jour toutes les 2 secondes")
    fmt.Println("🚨 Alertes automatiques basées sur des seuils")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

// Point d'entrée pour choisir l'exercice
func main() {
    if len(os.Args) < 2 {
        fmt.Println("Usage: go run main.go [notifications|editor|dashboard]")
        fmt.Println("  notifications - Système de notifications push")
        fmt.Println("  editor       - Éditeur de texte collaboratif")
        fmt.Println("  dashboard    - Dashboard temps réel")
        return
    }

    switch os.Args[1] {
    case "notifications":
        mainNotifications()
    case "editor":
        mainCollaborativeEditor()
    case "dashboard":
        mainDashboard()
    default:
        fmt.Println("Exercice non reconnu. Utilisez: notifications, editor, ou dashboard")
    }
}
```

## Instructions d'utilisation

### Exercice 1 : Notifications push
```bash
go run main.go notifications
# Visitez http://localhost:8080
# Testez la création de notifications
# Observez les filtres et l'historique
```

### Exercice 2 : Éditeur collaboratif
```bash
go run main.go editor
# Ouvrez plusieurs onglets avec différents utilisateurs:
# http://localhost:8080?username=Alice&doc=test
# http://localhost:8080?username=Bob&doc=test
# Tapez simultanément et observez la synchronisation
```

### Exercice 3 : Dashboard temps réel
```bash
go run main.go dashboard
# Visitez http://localhost:8080
# Observez les métriques en temps réel
# Attendez les alertes automatiques
```

## Fonctionnalités implémentées

### 🔔 **Exercice 1 - Notifications**
- ✅ Types de notifications (info, warning, error, success)
- ✅ Filtrage par utilisateur et rôle
- ✅ Historique complet avec pagination
- ✅ Notifications navigateur intégrées
- ✅ Interface de test interactive
- ✅ Système de priorités

### ✏️ **Exercice 2 - Éditeur collaboratif**
- ✅ Synchronisation en temps réel
- ✅ Transformation opérationnelle (OT) basique
- ✅ Curseurs multi-utilisateurs avec couleurs
- ✅ Gestion des conflits d'édition
- ✅ Historique des opérations
- ✅ Interface moderne et responsive

### 📊 **Exercice 3 - Dashboard**
- ✅ Métriques système simulées réalistes
- ✅ Graphiques temps réel avec Chart.js
- ✅ Alertes automatiques avec seuils
- ✅ Interface dark theme moderne
- ✅ Historique et tendances
- ✅ Notifications sonores pour alertes critiques

## Technologies utilisées

- **Backend** : Go avec Gorilla WebSocket
- **Frontend** : HTML5, CSS3, JavaScript ES6+
- **Graphiques** : Chart.js pour visualisations
- **WebSockets** : Communication bidirectionnelle temps réel
- **Concurrence** : Goroutines et channels Go
- **Architecture** : Hub pattern pour gestion des connexions

Ces trois exercices offrent une progression complète dans l'utilisation des WebSockets, de la communication simple aux applications complexes temps réel !

## Points d'apprentissage

1. **Gestion d'état** : Synchronisation entre clients
2. **Performance** : Optimisation des updates fréquentes
3. **UX temps réel** : Feedback immédiat utilisateur
4. **Architecture scalable** : Hub pattern et goroutines
5. **Gestion d'erreurs** : Reconnexion automatique
6. **Sécurité** : Validation des données WebSocket

⏭️
