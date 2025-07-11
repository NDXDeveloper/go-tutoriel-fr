🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15-1 : Parsing des arguments

## Introduction

Le parsing des arguments est la première étape cruciale dans la création d'un outil CLI. C'est ce qui permet à votre programme de comprendre ce que l'utilisateur veut faire quand il tape une commande comme `gofiles list --ext=go` ou `gofiles copy --dest=./backup`.

## Qu'est-ce que le parsing d'arguments ?

Quand un utilisateur tape une commande dans le terminal, le système d'exploitation divise cette commande en plusieurs parties :

```bash
gofiles list --ext=go --verbose ./src
```

Se décompose en :
- `gofiles` : le nom du programme
- `list` : la sous-commande
- `--ext=go` : une option avec sa valeur
- `--verbose` : un flag (option booléenne)
- `./src` : un argument positionnel

Notre travail est de "parser" (analyser) ces éléments pour comprendre ce que l'utilisateur veut faire.

## Approches pour le parsing d'arguments

### 1. Package `flag` (standard Go)

Le package `flag` est inclus dans Go, mais il est limité :

```go
package main

import (
    "flag"
    "fmt"
)

func main() {
    // Définir les options
    var verbose = flag.Bool("verbose", false, "Mode verbeux")
    var ext = flag.String("ext", "", "Extension de fichier")

    // Parser les arguments
    flag.Parse()

    // Utiliser les valeurs
    fmt.Printf("Verbose: %v\n", *verbose)
    fmt.Printf("Extension: %s\n", *ext)
}
```

**Problèmes avec `flag` :**
- Pas de support natif pour les sous-commandes
- Interface utilisateur basique
- Pas de génération automatique d'aide
- Difficile à maintenir pour des CLI complexes

### 2. Package `cobra` (recommandé)

Cobra est le standard de facto pour les CLI en Go. Il est utilisé par des projets majeurs comme Docker, Kubernetes, et Hugo.

## Installation de Cobra

```bash
go get github.com/spf13/cobra@latest
```

## Concepts de base de Cobra

### Commands (Commandes)

Une commande représente une action que votre CLI peut effectuer :

```go
var rootCmd = &cobra.Command{
    Use:   "gofiles",
    Short: "Un gestionnaire de fichiers en ligne de commande",
    Long:  "GoFiles est un outil CLI pour gérer vos fichiers efficacement.",
}
```

### Flags (Options)

Les flags sont des options que l'utilisateur peut passer :

```go
// Flag persistant (disponible pour toutes les sous-commandes)
rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "Mode verbeux")

// Flag local (seulement pour cette commande)
listCmd.Flags().StringVarP(&extension, "ext", "e", "", "Filtrer par extension")
```

### Arguments

Les arguments sont les valeurs passées après les flags :

```go
// Dans la fonction Run de votre commande
Run: func(cmd *cobra.Command, args []string) {
    // args contient les arguments positionnels
    fmt.Printf("Arguments reçus: %v\n", args)
}
```

## Création de notre CLI GoFiles

### Étape 1 : Structure du projet

Créons d'abord la structure de base :

```
gofiles/
├── main.go
├── cmd/
│   ├── root.go
│   └── list.go
├── go.mod
└── go.sum
```

### Étape 2 : Le fichier main.go

```go
package main

import (
    "fmt"
    "os"

    "github.com/votre-username/gofiles/cmd"
)

func main() {
    if err := cmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### Étape 3 : La commande racine (cmd/root.go)

```go
package cmd

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

var (
    // Variables globales pour les flags persistants
    verbose bool
    cfgFile string
)

// rootCmd représente la commande de base quand appelée sans sous-commandes
var rootCmd = &cobra.Command{
    Use:   "gofiles",
    Short: "Un gestionnaire de fichiers CLI puissant",
    Long: `GoFiles est un outil en ligne de commande qui vous permet de gérer
vos fichiers efficacement. Il offre des fonctionnalités comme la liste,
la copie, la recherche et la compression de fichiers.

Exemples d'utilisation :
  gofiles list --ext=go
  gofiles copy --pattern="*.txt" --dest=./backup
  gofiles search --name="*config*"`,

    // Cette fonction s'exécute quand la commande racine est appelée
    Run: func(cmd *cobra.Command, args []string) {
        fmt.Println("Bienvenue dans GoFiles!")
        fmt.Println("Utilisez 'gofiles --help' pour voir les commandes disponibles.")
    },
}

// Execute ajoute toutes les commandes enfant à la commande racine et définit les flags
func Execute() error {
    return rootCmd.Execute()
}

func init() {
    // Flags persistants (disponibles pour toutes les sous-commandes)
    rootCmd.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "Affichage verbeux")
    rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "fichier de configuration (défaut: $HOME/.gofiles.yaml)")

    // Flags locaux (seulement pour la commande racine)
    rootCmd.Flags().BoolP("version", "V", false, "Afficher la version")
}
```

### Étape 4 : Une sous-commande (cmd/list.go)

```go
package cmd

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "github.com/spf13/cobra"
)

var (
    // Variables pour les flags de la commande list
    extension    string
    recursive    bool
    showHidden   bool
    sortBy       string
    outputFormat string
)

// listCmd représente la commande list
var listCmd = &cobra.Command{
    Use:   "list [répertoire]",
    Short: "Liste les fichiers d'un répertoire",
    Long: `La commande list affiche le contenu d'un répertoire avec diverses options
de filtrage et de formatage.

Exemples :
  gofiles list                    # Liste le répertoire courant
  gofiles list /home/user/docs    # Liste un répertoire spécifique
  gofiles list --ext=go           # Liste seulement les fichiers .go
  gofiles list --recursive        # Liste récursivement
  gofiles list --hidden           # Inclut les fichiers cachés`,

    // Validation des arguments
    Args: cobra.MaximumNArgs(1), // Maximum 1 argument (le répertoire)

    // Fonction principale de la commande
    Run: func(cmd *cobra.Command, args []string) {
        // Déterminer le répertoire à lister
        dir := "."
        if len(args) > 0 {
            dir = args[0]
        }

        // Vérifier que le répertoire existe
        if _, err := os.Stat(dir); os.IsNotExist(err) {
            fmt.Fprintf(os.Stderr, "Erreur: Le répertoire '%s' n'existe pas\n", dir)
            os.Exit(1)
        }

        // Appeler la fonction de listage
        if err := listFiles(dir); err != nil {
            fmt.Fprintf(os.Stderr, "Erreur lors du listage: %v\n", err)
            os.Exit(1)
        }
    },
}

func listFiles(dir string) error {
    if verbose {
        fmt.Printf("Listage du répertoire: %s\n", dir)
        fmt.Printf("Extension filtrée: %s\n", extension)
        fmt.Printf("Récursif: %v\n", recursive)
        fmt.Printf("Fichiers cachés: %v\n", showHidden)
    }

    // Fonction de listage simple
    return filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Ignorer les fichiers cachés si demandé
        if !showHidden && strings.HasPrefix(info.Name(), ".") {
            if info.IsDir() {
                return filepath.SkipDir
            }
            return nil
        }

        // Filtrer par extension si spécifiée
        if extension != "" && !strings.HasSuffix(info.Name(), "."+extension) {
            return nil
        }

        // Ne pas descendre récursivement si pas demandé
        if !recursive && info.IsDir() && path != dir {
            return filepath.SkipDir
        }

        // Afficher le fichier
        if outputFormat == "detailed" {
            fmt.Printf("%s\t%d\t%s\n", info.Mode(), info.Size(), path)
        } else {
            fmt.Println(path)
        }

        return nil
    })
}

func init() {
    // Ajouter la commande list à la commande racine
    rootCmd.AddCommand(listCmd)

    // Flags pour la commande list
    listCmd.Flags().StringVarP(&extension, "ext", "e", "", "Filtrer par extension (ex: go, txt)")
    listCmd.Flags().BoolVarP(&recursive, "recursive", "r", false, "Listage récursif")
    listCmd.Flags().BoolVarP(&showHidden, "hidden", "H", false, "Inclure les fichiers cachés")
    listCmd.Flags().StringVarP(&sortBy, "sort", "s", "name", "Trier par: name, size, date")
    listCmd.Flags().StringVarP(&outputFormat, "format", "f", "simple", "Format de sortie: simple, detailed")

    // Marquer certains flags comme requis (optionnel)
    // listCmd.MarkFlagRequired("ext")
}
```

## Test de notre CLI

### Compilation et test

```bash
# Compiler
go build -o gofiles main.go

# Tester la commande racine
./gofiles

# Tester l'aide
./gofiles --help

# Tester la sous-commande list
./gofiles list --help
./gofiles list --ext=go --verbose
./gofiles list --recursive --hidden
```

## Fonctionnalités avancées de Cobra

### 1. Validation des arguments

```go
var listCmd = &cobra.Command{
    Use:   "list [répertoire]",
    Short: "Liste les fichiers",
    Args:  cobra.ExactArgs(1), // Exactement 1 argument requis
    // Autres options :
    // Args: cobra.NoArgs,           // Aucun argument
    // Args: cobra.MinimumNArgs(1),  // Au moins 1 argument
    // Args: cobra.MaximumNArgs(2),  // Au maximum 2 arguments
    // Args: cobra.RangeArgs(1, 3),  // Entre 1 et 3 arguments
    Run: func(cmd *cobra.Command, args []string) {
        // ...
    },
}
```

### 2. Validation personnalisée

```go
var listCmd = &cobra.Command{
    Use:   "list [répertoire]",
    Short: "Liste les fichiers",
    Args: func(cmd *cobra.Command, args []string) error {
        if len(args) < 1 {
            return fmt.Errorf("au moins un répertoire est requis")
        }

        // Vérifier que le répertoire existe
        if _, err := os.Stat(args[0]); os.IsNotExist(err) {
            return fmt.Errorf("le répertoire '%s' n'existe pas", args[0])
        }

        return nil
    },
    Run: func(cmd *cobra.Command, args []string) {
        // ...
    },
}
```

### 3. Hooks (fonctions avant/après)

```go
var listCmd = &cobra.Command{
    Use:   "list",
    Short: "Liste les fichiers",

    // Exécuté avant la commande
    PreRun: func(cmd *cobra.Command, args []string) {
        fmt.Println("Préparation du listage...")
    },

    // Fonction principale
    Run: func(cmd *cobra.Command, args []string) {
        // Logique principale
    },

    // Exécuté après la commande
    PostRun: func(cmd *cobra.Command, args []string) {
        fmt.Println("Listage terminé.")
    },
}
```

## Gestion des erreurs

```go
Run: func(cmd *cobra.Command, args []string) {
    if err := listFiles(args[0]); err != nil {
        // Écrire l'erreur sur stderr
        fmt.Fprintf(os.Stderr, "Erreur: %v\n", err)

        // Sortir avec un code d'erreur
        os.Exit(1)
    }
},
```

## Meilleures pratiques

### 1. Organisation du code

- Une commande par fichier dans le dossier `cmd/`
- Variables globales pour les flags persistants
- Fonctions séparées pour la logique métier

### 2. Messages d'aide utiles

```go
var listCmd = &cobra.Command{
    Use:   "list [répertoire]",
    Short: "Liste les fichiers d'un répertoire",
    Long: `Description détaillée avec exemples d'utilisation.

Cette commande permet de lister les fichiers avec de nombreuses options.`,

    Example: `  gofiles list
  gofiles list /home/user
  gofiles list --ext=go --recursive`,
}
```

### 3. Validation et gestion d'erreurs

- Toujours valider les arguments
- Utiliser des messages d'erreur clairs
- Retourner des codes d'erreur appropriés

### 4. Flags bien nommés

```go
// Bon : court et long
listCmd.Flags().StringVarP(&extension, "ext", "e", "", "Extension de fichier")

// Éviter : seulement court ou peu clair
listCmd.Flags().StringVarP(&extension, "x", "", "", "Ext")
```

## Exercices pratiques

### Exercice 1 : Ajouter une commande `version`

Créez une commande qui affiche la version de votre outil.

### Exercice 2 : Améliorer la validation

Ajoutez une validation pour s'assurer que l'extension fournie ne contient pas de caractères invalides.

### Exercice 3 : Flags dépendants

Implémentez une logique où le flag `--sort` n'est disponible que si `--format=detailed` est utilisé.

## Résumé

Dans cette section, nous avons appris :

- **Les bases du parsing d'arguments** et pourquoi c'est important
- **L'utilisation de Cobra** pour créer des CLI professionnels
- **La structure d'un projet CLI** avec commandes et sous-commandes
- **Les bonnes pratiques** pour la validation et la gestion d'erreurs
- **L'implémentation concrète** d'une commande `list` complète

Notre CLI commence à prendre forme ! Dans la prochaine section, nous approfondirons l'interaction avec le système de fichiers pour implémenter des fonctionnalités plus avancées.

## Prochaine étape

La section suivante (15-2) couvrira l'interaction avec le système de fichiers, où nous implémenterons des opérations comme la copie, le déplacement et la recherche de fichiers.

⏭️
