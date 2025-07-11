🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Projet 1 : CLI Tool

## Introduction

Dans ce projet pratique, nous allons créer un outil en ligne de commande (CLI) complet en Go. Ce projet vous permettra d'appliquer les connaissances acquises dans les parties précédentes tout en découvrant les spécificités du développement d'applications CLI.

## Objectifs du projet

À la fin de ce projet, vous saurez :
- Créer une application CLI robuste et professionnelle
- Gérer les arguments et options de ligne de commande
- Interagir avec le système de fichiers
- Structurer un projet Go pour la distribution
- Implémenter des fonctionnalités courantes des outils CLI

## Qu'est-ce qu'un CLI Tool ?

Un CLI Tool (Command Line Interface Tool) est un programme qui s'exécute dans le terminal et permet à l'utilisateur d'effectuer des tâches via des commandes textuelles. Les outils CLI sont particulièrement populaires chez les développeurs et les administrateurs système car ils :

- Sont facilement automatisables dans des scripts
- Consomment moins de ressources que les interfaces graphiques
- Peuvent être intégrés dans des pipelines CI/CD
- Offrent une grande flexibilité d'utilisation

## Exemples d'outils CLI populaires

- **Git** : Gestion de version (`git commit`, `git push`, etc.)
- **Docker** : Containerisation (`docker build`, `docker run`, etc.)
- **kubectl** : Gestion de Kubernetes
- **aws-cli** : Interface AWS
- **hugo** : Générateur de sites statiques (écrit en Go !)

## Notre projet : "GoFiles" - Un gestionnaire de fichiers CLI

Nous allons créer un outil appelé **GoFiles** qui permettra de :

1. **Lister** le contenu d'un répertoire avec des options de filtrage
2. **Copier** et **déplacer** des fichiers avec confirmation
3. **Rechercher** des fichiers par nom ou extension
4. **Compresser** et **décompresser** des archives
5. **Afficher** des statistiques sur les fichiers et répertoires

### Exemple d'utilisation finale

```bash
# Lister tous les fichiers .go dans le répertoire courant
gofiles list --ext=go

# Copier tous les fichiers .txt vers un répertoire de sauvegarde
gofiles copy --pattern="*.txt" --dest=./backup

# Rechercher tous les fichiers contenant "config" dans le nom
gofiles search --name="*config*"

# Compresser un répertoire
gofiles compress --source=./project --output=project.tar.gz

# Afficher les statistiques d'un répertoire
gofiles stats --path=./src --recursive
```

## Architecture du projet

Notre CLI sera structuré de la façon suivante :

```
gofiles/
├── main.go              # Point d'entrée de l'application
├── cmd/                 # Commandes CLI
│   ├── root.go          # Commande racine
│   ├── list.go          # Commande list
│   ├── copy.go          # Commande copy
│   ├── search.go        # Commande search
│   ├── compress.go      # Commande compress
│   └── stats.go         # Commande stats
├── internal/            # Logique métier privée
│   ├── fileops/         # Opérations sur les fichiers
│   ├── utils/           # Utilitaires
│   └── config/          # Configuration
├── pkg/                 # Packages réutilisables
│   └── archive/         # Gestion des archives
├── go.mod
├── go.sum
├── README.md
└── Makefile             # Automatisation du build
```

## Fonctionnalités clés à implémenter

### 1. Interface utilisateur intuitive
- Commandes et sous-commandes claires
- Messages d'aide détaillés
- Gestion des erreurs avec messages explicites
- Options courtes et longues (`-h` et `--help`)

### 2. Gestion robuste des fichiers
- Vérification des permissions
- Gestion des chemins absolus et relatifs
- Support des patterns de fichiers (wildcards)
- Opérations sécurisées avec confirmations

### 3. Configuration flexible
- Fichier de configuration optionnel
- Variables d'environnement
- Options par défaut personnalisables
- Profils d'utilisation

### 4. Performances optimisées
- Traitement concurrent pour les opérations sur de gros volumes
- Barre de progression pour les opérations longues
- Mise en cache intelligente
- Gestion efficace de la mémoire

## Technologies et packages utilisés

### Packages standard Go
- `os` et `os/exec` : Interaction avec le système
- `path/filepath` : Manipulation des chemins
- `archive/tar` et `compress/gzip` : Gestion des archives
- `flag` : Parsing basique des arguments (puis remplacement)
- `fmt` et `log` : Affichage et logging

### Packages externes
- **cobra** : Framework CLI moderne et puissant
- **viper** : Gestion de configuration avancée
- **color** : Couleurs dans le terminal
- **progressbar** : Barres de progression

## Bonnes pratiques CLI

Notre outil respectera les conventions Unix/Linux :
- Code de sortie 0 pour succès, non-zéro pour erreur
- Sortie standard (stdout) pour les données, erreur (stderr) pour les messages d'erreur
- Support des pipes et redirections
- Respect des signaux système (SIGINT, SIGTERM)
- Documentation intégrée avec `--help`

## Prérequis

Avant de commencer, assurez-vous de maîtriser :
- Les bases de Go (variables, fonctions, structures)
- La gestion des erreurs en Go
- Les packages et modules Go
- Les opérations sur les fichiers
- Les concepts de base de la ligne de commande

## Structure du tutoriel

Ce projet sera décomposé en plusieurs parties :

1. **Parsing des arguments** - Mise en place de l'interface CLI
2. **Interaction avec le système de fichiers** - Opérations sur les fichiers
3. **Distribution et packaging** - Compilation et distribution

Chaque partie comprendra :
- Explication des concepts
- Implémentation étape par étape
- Tests unitaires
- Exemples d'utilisation
- Exercices pratiques

## Préparation de l'environnement

Créez un nouveau répertoire pour le projet :

```bash
mkdir gofiles
cd gofiles
go mod init github.com/votre-username/gofiles
```

## Prochaine étape

Dans la section suivante, nous commencerons par implémenter le parsing des arguments avec le package `cobra`, qui nous permettra de créer une interface CLI professionnelle et extensible.

---

*Ce projet vous donnera une expérience pratique complète du développement d'applications CLI en Go, depuis la conception jusqu'à la distribution.*

⏭️
