🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1-2 : Installation et configuration de l'environnement Go

## Vue d'ensemble

Dans cette section, nous allons installer Go sur votre système et configurer votre environnement de développement. Go est disponible sur tous les systèmes d'exploitation majeurs : Windows, macOS et Linux.

## Installation de Go

### Méthode 1 : Installation officielle (Recommandée)

#### Windows

**Étape 1 : Télécharger Go**
1. Rendez-vous sur [golang.org/dl](https://golang.org/dl/)
2. Téléchargez le fichier `.msi` pour Windows (ex: `go1.21.5.windows-amd64.msi`)
3. Exécutez le fichier téléchargé

**Étape 2 : Installation**
1. Suivez l'assistant d'installation
2. Go sera installé dans `C:\Program Files\Go` par défaut
3. L'installateur ajoutera automatiquement Go au PATH

**Étape 3 : Vérification**
Ouvrez l'invite de commande (cmd) et tapez :
```cmd
go version
```
Vous devriez voir quelque chose comme :
```
go version go1.21.5 windows/amd64
```

#### macOS

**Option A : Avec l'installateur officiel**
1. Téléchargez le fichier `.pkg` depuis [golang.org/dl](https://golang.org/dl/)
2. Double-cliquez sur le fichier téléchargé
3. Suivez les instructions d'installation

**Option B : Avec Homebrew (si installé)**
```bash
brew install go
```

**Vérification :**
```bash
go version
```

#### Linux (Ubuntu/Debian)

**Option A : Avec le gestionnaire de paquets**
```bash
sudo apt update
sudo apt install golang-go
```

**Option B : Installation manuelle (version plus récente)**
```bash
# Télécharger la dernière version
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz

# Extraire dans /usr/local
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz

# Ajouter au PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

**Vérification :**
```bash
go version
```

### Méthode 2 : Installation avec des gestionnaires de versions

#### Avec g (gestionnaire de versions Go)
```bash
# Installer g
curl -sSL https://git.io/g-install | sh -s

# Installer la dernière version de Go
g install latest
```

## Configuration de l'environnement

### Variables d'environnement importantes

#### GOROOT
- **Définition** : Répertoire où Go est installé
- **Valeur par défaut** : Définie automatiquement lors de l'installation
- **À modifier ?** : Généralement non, sauf installation personnalisée

#### GOPATH (moins important depuis Go 1.11)
- **Définition** : Répertoire de travail pour les projets Go
- **Utilisation moderne** : Remplacé par les modules Go
- **Configuration** : Optionnelle pour les nouveaux projets

#### Configuration recommandée

**Windows :**
```cmd
# Vérifier les variables (PowerShell)
$env:GOROOT
$env:GOPATH
go env GOROOT
go env GOPATH
```

**macOS/Linux :**
```bash
# Vérifier les variables
echo $GOROOT
echo $GOPATH
go env GOROOT
go env GOPATH
```

### Structure de répertoires recommandée

```
~/projets-go/
├── mon-premier-projet/
│   ├── go.mod
│   └── main.go
├── api-rest/
│   ├── go.mod
│   └── main.go
└── outils/
    ├── go.mod
    └── main.go
```

## Choix de l'éditeur de code

### Éditeurs recommandés pour débutants

#### 1. Visual Studio Code (Gratuit, Recommandé)

**Installation :**
1. Téléchargez VS Code depuis [code.visualstudio.com](https://code.visualstudio.com/)
2. Installez l'extension Go officielle

**Configuration de l'extension Go :**
1. Ouvrez VS Code
2. Allez dans Extensions (Ctrl+Shift+X)
3. Recherchez "Go" et installez l'extension officielle de Google
4. Redémarrez VS Code

**Fonctionnalités disponibles :**
- Coloration syntaxique
- Auto-complétion intelligente
- Débogage intégré
- Formatage automatique
- Détection d'erreurs en temps réel

#### 2. GoLand (Payant, Professionnel)

**Avantages :**
- IDE complet spécialisé pour Go
- Débogage avancé
- Outils de refactoring puissants
- Intégration Git excellente

**Inconvénients :**
- Payant (30 jours d'essai gratuit)
- Plus lourd que VS Code

#### 3. Autres alternatives

**Vim/Neovim :** Pour les utilisateurs expérimentés
**Sublime Text :** Léger avec plugin Go
**Atom :** Gratuit avec extensions Go

### Configuration de VS Code pour Go

#### Installation des outils Go

Après avoir installé l'extension Go, VS Code vous proposera d'installer les outils Go nécessaires :

1. Ouvrez VS Code
2. Créez un fichier `test.go`
3. Une notification apparaîtra pour installer les outils Go
4. Cliquez sur "Install All"

**Outils installés automatiquement :**
- `gopls` : Serveur de langage Go
- `dlv` : Débogueur Go
- `goimports` : Formatage et organisation des imports
- `staticcheck` : Analyse statique du code

#### Configuration personnalisée

Créez un fichier `.vscode/settings.json` dans votre projet :

```json
{
    "go.formatTool": "goimports",
    "go.useLanguageServer": true,
    "go.lintOnSave": "package",
    "go.vetOnSave": "package",
    "editor.formatOnSave": true,
    "go.toolsEnvVars": {
        "GO111MODULE": "on"
    }
}
```

## Test de l'installation

### Créer votre premier programme

**Étape 1 : Créer un répertoire de projet**
```bash
mkdir mon-premier-go
cd mon-premier-go
```

**Étape 2 : Initialiser un module Go**
```bash
go mod init mon-premier-go
```

**Étape 3 : Créer le fichier main.go**
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
    fmt.Println("Go est correctement installé !")
}
```

**Étape 4 : Exécuter le programme**
```bash
go run main.go
```

**Résultat attendu :**
```
Hello, World!
Go est correctement installé !
```

### Vérification complète de l'installation

```bash
# Version de Go
go version

# Configuration de l'environnement
go env

# Vérifier que les outils fonctionnent
go fmt main.go    # Formatage
go vet main.go    # Vérification
go build main.go  # Compilation
```

## Outils de développement intégrés

### Commandes Go essentielles

| Commande | Description | Exemple |
|----------|-------------|---------|
| `go run` | Exécute un programme Go | `go run main.go` |
| `go build` | Compile un programme | `go build main.go` |
| `go fmt` | Formate le code | `go fmt .` |
| `go vet` | Vérifie le code | `go vet .` |
| `go test` | Exécute les tests | `go test .` |
| `go mod` | Gestion des modules | `go mod init myproject` |
| `go get` | Télécharge des dépendances | `go get github.com/gin-gonic/gin` |

### Formatage automatique

Go intègre un formateur de code officiel :

```bash
# Formater tous les fichiers Go du répertoire
go fmt .

# Formater un fichier spécifique
go fmt main.go

# Formater avec goimports (organise aussi les imports)
goimports -w .
```

**Avantage :** Tous les développeurs Go utilisent le même style de formatage !

## Résolution des problèmes courants

### Go n'est pas reconnu dans le terminal

**Windows :**
1. Vérifiez que `C:\Program Files\Go\bin` est dans votre PATH
2. Redémarrez votre terminal
3. Ou ajoutez manuellement à votre PATH

**macOS/Linux :**
```bash
# Ajouter à ~/.bashrc ou ~/.zshrc
export PATH=$PATH:/usr/local/go/bin

# Recharger la configuration
source ~/.bashrc
```

### Erreur "go: cannot find main module"

**Solution :**
```bash
# Initialiser un module Go dans votre répertoire
go mod init nom-du-projet
```

### VS Code ne reconnaît pas Go

1. Vérifiez que l'extension Go est installée
2. Rechargez VS Code (Ctrl+Shift+P → "Developer: Reload Window")
3. Vérifiez que `gopls` est installé : `gopls version`

## Prochaines étapes

Félicitations ! Vous avez maintenant :
- ✅ Go installé sur votre système
- ✅ Un éditeur de code configuré
- ✅ Votre premier programme Go qui fonctionne
- ✅ Les outils de développement prêts

**Récapitulatif des commandes importantes :**
```bash
go version          # Vérifier la version
go mod init projet  # Initialiser un nouveau projet
go run main.go      # Exécuter un programme
go build main.go    # Compiler un programme
go fmt .            # Formater le code
```

---

*Dans la prochaine section, nous analyserons en détail votre premier programme "Hello World" et découvrirons la structure d'un programme Go !*

⏭️
