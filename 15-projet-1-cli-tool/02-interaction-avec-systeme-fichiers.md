🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15-2 : Interaction avec le système de fichiers

## Introduction

L'interaction avec le système de fichiers est au cœur de notre outil CLI GoFiles. Dans cette section, nous allons apprendre à manipuler les fichiers et dossiers de manière sûre et efficace en Go.

## Concepts fondamentaux

### Qu'est-ce que le système de fichiers ?

Le système de fichiers est la façon dont votre ordinateur organise et stocke les données sur le disque dur. Il comprend :

- **Fichiers** : Contenants de données (texte, images, programmes, etc.)
- **Dossiers/Répertoires** : Organisateurs qui contiennent des fichiers et d'autres dossiers
- **Chemins** : Adresses qui indiquent où trouver un fichier ou dossier
- **Permissions** : Règles qui définissent qui peut lire, écrire ou exécuter un fichier

### Types de chemins

```go
// Chemin absolu (depuis la racine du système)
"/home/user/documents/fichier.txt"     // Linux/Mac
"C:\\Users\\user\\Documents\\fichier.txt" // Windows

// Chemin relatif (depuis le répertoire courant)
"./fichier.txt"        // Fichier dans le répertoire courant
"../parent/fichier.txt" // Fichier dans le répertoire parent
"subfolder/fichier.txt" // Fichier dans un sous-dossier
```

## Packages Go pour les fichiers

### 1. Package `os`

Le package `os` fournit les fonctions de base pour interagir avec le système :

```go
import "os"

// Informations sur un fichier
info, err := os.Stat("fichier.txt")
if err != nil {
    // Le fichier n'existe pas ou erreur d'accès
}

// Vérifier si un fichier existe
if _, err := os.Stat("fichier.txt"); err == nil {
    fmt.Println("Le fichier existe")
} else if os.IsNotExist(err) {
    fmt.Println("Le fichier n'existe pas")
}

// Créer un répertoire
err := os.Mkdir("nouveau_dossier", 0755)

// Créer un répertoire avec tous les parents nécessaires
err := os.MkdirAll("chemin/vers/nouveau/dossier", 0755)

// Supprimer un fichier
err := os.Remove("fichier.txt")

// Supprimer un répertoire et tout son contenu
err := os.RemoveAll("dossier")
```

### 2. Package `filepath`

Le package `filepath` aide à manipuler les chemins de fichiers :

```go
import "path/filepath"

// Joindre des parties de chemin
chemin := filepath.Join("home", "user", "documents", "fichier.txt")
// Résultat : "home/user/documents/fichier.txt" (Linux/Mac)
//           "home\\user\\documents\\fichier.txt" (Windows)

// Obtenir le répertoire parent
dir := filepath.Dir("/home/user/documents/fichier.txt")
// Résultat : "/home/user/documents"

// Obtenir le nom du fichier
nom := filepath.Base("/home/user/documents/fichier.txt")
// Résultat : "fichier.txt"

// Obtenir l'extension
ext := filepath.Ext("fichier.txt")
// Résultat : ".txt"

// Parcourir un répertoire récursivement
err := filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path)
    return nil
})
```

### 3. Package `io/ioutil` (déprécié) et `io` + `os`

Pour lire et écrire des fichiers :

```go
import (
    "io"
    "os"
)

// Lire un fichier entier
contenu, err := os.ReadFile("fichier.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(contenu))

// Écrire dans un fichier
err := os.WriteFile("nouveau_fichier.txt", []byte("Contenu"), 0644)
if err != nil {
    log.Fatal(err)
}

// Copier un fichier
source, err := os.Open("source.txt")
if err != nil {
    log.Fatal(err)
}
defer source.Close()

destination, err := os.Create("destination.txt")
if err != nil {
    log.Fatal(err)
}
defer destination.Close()

_, err = io.Copy(destination, source)
if err != nil {
    log.Fatal(err)
}
```

## Implémentation des opérations de fichiers

### Créer un package pour les opérations de fichiers

Créons `internal/fileops/fileops.go` :

```go
package fileops

import (
    "fmt"
    "io"
    "os"
    "path/filepath"
    "strings"
    "time"
)

// FileInfo représente les informations d'un fichier
type FileInfo struct {
    Name        string
    Path        string
    Size        int64
    Mode        os.FileMode
    ModTime     time.Time
    IsDir       bool
    Extension   string
}

// ListOptions définit les options pour lister les fichiers
type ListOptions struct {
    Recursive   bool
    ShowHidden  bool
    Extension   string
    SortBy      string
    Pattern     string
}

// ListFiles liste les fichiers selon les options données
func ListFiles(dir string, options ListOptions) ([]FileInfo, error) {
    var files []FileInfo

    // Vérifier que le répertoire existe
    if _, err := os.Stat(dir); os.IsNotExist(err) {
        return nil, fmt.Errorf("le répertoire '%s' n'existe pas", dir)
    }

    err := filepath.Walk(dir, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Ignorer les fichiers cachés si demandé
        if !options.ShowHidden && strings.HasPrefix(info.Name(), ".") {
            if info.IsDir() {
                return filepath.SkipDir
            }
            return nil
        }

        // Filtrer par extension
        if options.Extension != "" {
            if !strings.HasSuffix(strings.ToLower(info.Name()), "."+strings.ToLower(options.Extension)) {
                return nil
            }
        }

        // Filtrer par pattern (simple)
        if options.Pattern != "" {
            matched, _ := filepath.Match(options.Pattern, info.Name())
            if !matched {
                return nil
            }
        }

        // Ne pas descendre récursivement si pas demandé
        if !options.Recursive && info.IsDir() && path != dir {
            return filepath.SkipDir
        }

        // Créer FileInfo
        fileInfo := FileInfo{
            Name:      info.Name(),
            Path:      path,
            Size:      info.Size(),
            Mode:      info.Mode(),
            ModTime:   info.ModTime(),
            IsDir:     info.IsDir(),
            Extension: filepath.Ext(info.Name()),
        }

        files = append(files, fileInfo)
        return nil
    })

    return files, err
}

// CopyFile copie un fichier de source vers destination
func CopyFile(src, dst string) error {
    // Ouvrir le fichier source
    source, err := os.Open(src)
    if err != nil {
        return fmt.Errorf("impossible d'ouvrir le fichier source '%s': %v", src, err)
    }
    defer source.Close()

    // Créer le fichier destination
    destination, err := os.Create(dst)
    if err != nil {
        return fmt.Errorf("impossible de créer le fichier destination '%s': %v", dst, err)
    }
    defer destination.Close()

    // Copier le contenu
    _, err = io.Copy(destination, source)
    if err != nil {
        return fmt.Errorf("erreur lors de la copie: %v", err)
    }

    // Copier les permissions
    sourceInfo, err := os.Stat(src)
    if err != nil {
        return fmt.Errorf("impossible d'obtenir les informations du fichier source: %v", err)
    }

    err = os.Chmod(dst, sourceInfo.Mode())
    if err != nil {
        return fmt.Errorf("impossible de définir les permissions: %v", err)
    }

    return nil
}

// MoveFile déplace un fichier
func MoveFile(src, dst string) error {
    // Essayer d'abord un rename (plus rapide)
    err := os.Rename(src, dst)
    if err == nil {
        return nil
    }

    // Si le rename échoue, copier puis supprimer
    err = CopyFile(src, dst)
    if err != nil {
        return fmt.Errorf("erreur lors de la copie: %v", err)
    }

    err = os.Remove(src)
    if err != nil {
        return fmt.Errorf("erreur lors de la suppression du fichier source: %v", err)
    }

    return nil
}

// EnsureDir s'assure qu'un répertoire existe
func EnsureDir(path string) error {
    if _, err := os.Stat(path); os.IsNotExist(err) {
        return os.MkdirAll(path, 0755)
    }
    return nil
}

// GetFileSize retourne la taille d'un fichier
func GetFileSize(path string) (int64, error) {
    info, err := os.Stat(path)
    if err != nil {
        return 0, err
    }
    return info.Size(), nil
}

// IsDirectory vérifie si un chemin est un répertoire
func IsDirectory(path string) bool {
    info, err := os.Stat(path)
    if err != nil {
        return false
    }
    return info.IsDir()
}

// SafeRemove supprime un fichier avec confirmation
func SafeRemove(path string, confirm bool) error {
    if confirm {
        fmt.Printf("Voulez-vous vraiment supprimer '%s' ? (y/N): ", path)
        var response string
        fmt.Scanln(&response)
        if strings.ToLower(response) != "y" && strings.ToLower(response) != "yes" {
            return fmt.Errorf("suppression annulée")
        }
    }

    return os.Remove(path)
}
```

### Améliorer la commande `list`

Mettons à jour `cmd/list.go` pour utiliser notre nouveau package :

```go
package cmd

import (
    "fmt"
    "os"
    "sort"
    "strings"
    "time"

    "github.com/spf13/cobra"
    "github.com/votre-username/gofiles/internal/fileops"
)

var (
    extension    string
    recursive    bool
    showHidden   bool
    sortBy       string
    outputFormat string
    pattern      string
)

var listCmd = &cobra.Command{
    Use:   "list [répertoire]",
    Short: "Liste les fichiers d'un répertoire",
    Long: `La commande list affiche le contenu d'un répertoire avec diverses options
de filtrage et de formatage.

Exemples :
  gofiles list                           # Liste le répertoire courant
  gofiles list /home/user/docs           # Liste un répertoire spécifique
  gofiles list --ext=go                  # Liste seulement les fichiers .go
  gofiles list --recursive               # Liste récursivement
  gofiles list --hidden                  # Inclut les fichiers cachés
  gofiles list --pattern="*.txt"         # Liste les fichiers .txt
  gofiles list --sort=size --format=detailed # Liste détaillée triée par taille`,

    Args: cobra.MaximumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        // Déterminer le répertoire
        dir := "."
        if len(args) > 0 {
            dir = args[0]
        }

        // Créer les options
        options := fileops.ListOptions{
            Recursive:  recursive,
            ShowHidden: showHidden,
            Extension:  extension,
            SortBy:     sortBy,
            Pattern:    pattern,
        }

        // Lister les fichiers
        files, err := fileops.ListFiles(dir, options)
        if err != nil {
            fmt.Fprintf(os.Stderr, "Erreur: %v\n", err)
            os.Exit(1)
        }

        // Trier les fichiers
        sortFiles(files, sortBy)

        // Afficher les résultats
        displayFiles(files, outputFormat)
    },
}

func sortFiles(files []fileops.FileInfo, sortBy string) {
    switch sortBy {
    case "size":
        sort.Slice(files, func(i, j int) bool {
            return files[i].Size < files[j].Size
        })
    case "date":
        sort.Slice(files, func(i, j int) bool {
            return files[i].ModTime.Before(files[j].ModTime)
        })
    case "name":
        fallthrough
    default:
        sort.Slice(files, func(i, j int) bool {
            return strings.ToLower(files[i].Name) < strings.ToLower(files[j].Name)
        })
    }
}

func displayFiles(files []fileops.FileInfo, format string) {
    switch format {
    case "detailed":
        displayDetailed(files)
    case "json":
        displayJSON(files)
    default:
        displaySimple(files)
    }
}

func displaySimple(files []fileops.FileInfo) {
    for _, file := range files {
        if file.IsDir {
            fmt.Printf("%s/\n", file.Path)
        } else {
            fmt.Println(file.Path)
        }
    }
}

func displayDetailed(files []fileops.FileInfo) {
    fmt.Printf("%-10s %-12s %-20s %s\n", "PERMISSIONS", "TAILLE", "DATE", "NOM")
    fmt.Println(strings.Repeat("-", 70))

    for _, file := range files {
        permissions := file.Mode.String()
        size := formatSize(file.Size)
        date := file.ModTime.Format("2006-01-02 15:04")
        name := file.Name

        if file.IsDir {
            name += "/"
        }

        fmt.Printf("%-10s %-12s %-20s %s\n", permissions, size, date, name)
    }
}

func displayJSON(files []fileops.FileInfo) {
    fmt.Println("[")
    for i, file := range files {
        fmt.Printf("  {\n")
        fmt.Printf("    \"name\": \"%s\",\n", file.Name)
        fmt.Printf("    \"path\": \"%s\",\n", file.Path)
        fmt.Printf("    \"size\": %d,\n", file.Size)
        fmt.Printf("    \"is_dir\": %t,\n", file.IsDir)
        fmt.Printf("    \"modified\": \"%s\"\n", file.ModTime.Format(time.RFC3339))
        if i < len(files)-1 {
            fmt.Printf("  },\n")
        } else {
            fmt.Printf("  }\n")
        }
    }
    fmt.Println("]")
}

func formatSize(size int64) string {
    if size < 1024 {
        return fmt.Sprintf("%d B", size)
    } else if size < 1024*1024 {
        return fmt.Sprintf("%.1f KB", float64(size)/1024)
    } else if size < 1024*1024*1024 {
        return fmt.Sprintf("%.1f MB", float64(size)/(1024*1024))
    } else {
        return fmt.Sprintf("%.1f GB", float64(size)/(1024*1024*1024))
    }
}

func init() {
    rootCmd.AddCommand(listCmd)

    listCmd.Flags().StringVarP(&extension, "ext", "e", "", "Filtrer par extension")
    listCmd.Flags().BoolVarP(&recursive, "recursive", "r", false, "Listage récursif")
    listCmd.Flags().BoolVarP(&showHidden, "hidden", "H", false, "Inclure les fichiers cachés")
    listCmd.Flags().StringVarP(&sortBy, "sort", "s", "name", "Trier par: name, size, date")
    listCmd.Flags().StringVarP(&outputFormat, "format", "f", "simple", "Format: simple, detailed, json")
    listCmd.Flags().StringVarP(&pattern, "pattern", "p", "", "Pattern de fichier (ex: *.txt)")
}
```

### Ajouter une commande `copy`

Créons `cmd/copy.go` :

```go
package cmd

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "github.com/spf13/cobra"
    "github.com/votre-username/gofiles/internal/fileops"
)

var (
    copyDest       string
    copyRecursive  bool
    copyOverwrite  bool
    copyPattern    string
    copyConfirm    bool
)

var copyCmd = &cobra.Command{
    Use:   "copy [source] --dest [destination]",
    Short: "Copie des fichiers ou répertoires",
    Long: `La commande copy permet de copier des fichiers ou répertoires avec
diverses options de filtrage et de confirmation.

Exemples :
  gofiles copy file.txt --dest ./backup/          # Copie un fichier
  gofiles copy --pattern="*.go" --dest ./backup/  # Copie tous les .go
  gofiles copy ./src --dest ./backup --recursive  # Copie récursive
  gofiles copy file.txt --dest ./backup --confirm # Avec confirmation`,

    Args: cobra.MaximumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        // Vérifier que la destination est fournie
        if copyDest == "" {
            fmt.Fprintf(os.Stderr, "Erreur: --dest est requis\n")
            os.Exit(1)
        }

        // Déterminer la source
        source := "."
        if len(args) > 0 {
            source = args[0]
        }

        // Effectuer la copie
        if err := performCopy(source, copyDest); err != nil {
            fmt.Fprintf(os.Stderr, "Erreur: %v\n", err)
            os.Exit(1)
        }

        fmt.Printf("Copie terminée avec succès vers %s\n", copyDest)
    },
}

func performCopy(source, dest string) error {
    // S'assurer que la destination existe
    if err := fileops.EnsureDir(dest); err != nil {
        return fmt.Errorf("impossible de créer le répertoire destination: %v", err)
    }

    // Vérifier si la source est un fichier unique
    if info, err := os.Stat(source); err == nil && !info.IsDir() {
        return copySingleFile(source, dest)
    }

    // Copie multiple avec pattern
    if copyPattern != "" {
        return copyWithPattern(source, dest, copyPattern)
    }

    // Copie de répertoire
    if copyRecursive {
        return copyDirectory(source, dest)
    }

    return fmt.Errorf("pour copier un répertoire, utilisez --recursive")
}

func copySingleFile(source, dest string) error {
    // Si dest est un répertoire, ajouter le nom du fichier
    if fileops.IsDirectory(dest) {
        dest = filepath.Join(dest, filepath.Base(source))
    }

    // Vérifier si le fichier existe et demander confirmation
    if _, err := os.Stat(dest); err == nil && !copyOverwrite {
        if copyConfirm {
            fmt.Printf("Le fichier '%s' existe. L'écraser ? (y/N): ", dest)
            var response string
            fmt.Scanln(&response)
            if strings.ToLower(response) != "y" && strings.ToLower(response) != "yes" {
                return fmt.Errorf("copie annulée")
            }
        } else {
            return fmt.Errorf("le fichier '%s' existe déjà (utilisez --overwrite)", dest)
        }
    }

    return fileops.CopyFile(source, dest)
}

func copyWithPattern(source, dest, pattern string) error {
    // Lister les fichiers correspondant au pattern
    options := fileops.ListOptions{
        Recursive:  copyRecursive,
        ShowHidden: false,
        Pattern:    pattern,
    }

    files, err := fileops.ListFiles(source, options)
    if err != nil {
        return err
    }

    copiedCount := 0
    for _, file := range files {
        if file.IsDir {
            continue // Ignorer les répertoires pour l'instant
        }

        // Calculer le chemin de destination
        relPath, err := filepath.Rel(source, file.Path)
        if err != nil {
            continue
        }

        destPath := filepath.Join(dest, relPath)

        // Créer le répertoire parent si nécessaire
        if err := fileops.EnsureDir(filepath.Dir(destPath)); err != nil {
            return err
        }

        // Copier le fichier
        if verbose {
            fmt.Printf("Copie: %s -> %s\n", file.Path, destPath)
        }

        if err := fileops.CopyFile(file.Path, destPath); err != nil {
            fmt.Fprintf(os.Stderr, "Erreur lors de la copie de %s: %v\n", file.Path, err)
            continue
        }

        copiedCount++
    }

    fmt.Printf("%d fichier(s) copié(s)\n", copiedCount)
    return nil
}

func copyDirectory(source, dest string) error {
    return filepath.Walk(source, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }

        // Calculer le chemin relatif
        relPath, err := filepath.Rel(source, path)
        if err != nil {
            return err
        }

        destPath := filepath.Join(dest, relPath)

        if info.IsDir() {
            // Créer le répertoire
            return fileops.EnsureDir(destPath)
        } else {
            // Copier le fichier
            if verbose {
                fmt.Printf("Copie: %s -> %s\n", path, destPath)
            }
            return fileops.CopyFile(path, destPath)
        }
    })
}

func init() {
    rootCmd.AddCommand(copyCmd)

    copyCmd.Flags().StringVarP(&copyDest, "dest", "d", "", "Répertoire de destination (requis)")
    copyCmd.Flags().BoolVarP(&copyRecursive, "recursive", "r", false, "Copie récursive")
    copyCmd.Flags().BoolVar(&copyOverwrite, "overwrite", false, "Écraser les fichiers existants")
    copyCmd.Flags().StringVarP(&copyPattern, "pattern", "p", "", "Pattern de fichiers à copier")
    copyCmd.Flags().BoolVar(&copyConfirm, "confirm", false, "Demander confirmation avant écrasement")

    // Marquer --dest comme requis
    copyCmd.MarkFlagRequired("dest")
}
```

## Gestion des erreurs avancée

### Créer un système d'erreurs personnalisé

Créons `internal/errors/errors.go` :

```go
package errors

import (
    "fmt"
)

// FileError représente une erreur liée aux fichiers
type FileError struct {
    Op   string // Opération (copy, move, delete, etc.)
    Path string // Chemin du fichier
    Err  error  // Erreur sous-jacente
}

func (e *FileError) Error() string {
    return fmt.Sprintf("erreur lors de '%s' sur '%s': %v", e.Op, e.Path, e.Err)
}

// NewFileError crée une nouvelle erreur de fichier
func NewFileError(op, path string, err error) *FileError {
    return &FileError{
        Op:   op,
        Path: path,
        Err:  err,
    }
}

// IsPermissionError vérifie si c'est une erreur de permissions
func IsPermissionError(err error) bool {
    if fileErr, ok := err.(*FileError); ok {
        return isPermissionError(fileErr.Err)
    }
    return isPermissionError(err)
}

func isPermissionError(err error) bool {
    return err != nil && (err.Error() == "permission denied" || err.Error() == "access denied")
}
```

## Sécurité et validation

### Validation des chemins

```go
package fileops

import (
    "fmt"
    "path/filepath"
    "strings"
)

// ValidatePath valide un chemin de fichier
func ValidatePath(path string) error {
    // Nettoyer le chemin
    path = filepath.Clean(path)

    // Vérifier les caractères dangereux
    if strings.Contains(path, "..") {
        return fmt.Errorf("chemin dangereux détecté: %s", path)
    }

    // Vérifier la longueur
    if len(path) > 4096 {
        return fmt.Errorf("chemin trop long: %d caractères", len(path))
    }

    return nil
}

// SafePath retourne un chemin sûr
func SafePath(path string) string {
    // Nettoyer et sécuriser le chemin
    path = filepath.Clean(path)

    // Supprimer les éléments dangereux
    parts := strings.Split(path, string(filepath.Separator))
    var safeParts []string

    for _, part := range parts {
        if part != ".." && part != "." && part != "" {
            safeParts = append(safeParts, part)
        }
    }

    return strings.Join(safeParts, string(filepath.Separator))
}
```

## Tests unitaires

### Créer des tests pour nos fonctions

Créons `internal/fileops/fileops_test.go` :

```go
package fileops

import (
    "os"
    "path/filepath"
    "testing"
    "time"
)

func TestCopyFile(t *testing.T) {
    // Créer un fichier temporaire
    tmpDir := t.TempDir()
    sourceFile := filepath.Join(tmpDir, "source.txt")
    destFile := filepath.Join(tmpDir, "dest.txt")

    // Écrire du contenu dans le fichier source
    content := "Hello, World!"
    err := os.WriteFile(sourceFile, []byte(content), 0644)
    if err != nil {
        t.Fatalf("Erreur lors de la création du fichier source: %v", err)
    }

    // Tester la copie
    err = CopyFile(sourceFile, destFile)
    if err != nil {
        t.Fatalf("Erreur lors de la copie: %v", err)
    }

    // Vérifier que le fichier destination existe
    if _, err := os.Stat(destFile); os.IsNotExist(err) {
        t.Fatal("Le fichier destination n'existe pas")
    }

    // Vérifier le contenu
    destContent, err := os.ReadFile(destFile)
    if err != nil {
        t.Fatalf("Erreur lors de la lecture du fichier destination: %v", err)
    }

    if string(destContent) != content {
        t.Fatalf("Contenu incorrect. Attendu: %s, Obtenu: %s", content, string(destContent))
    }
}

func TestListFiles(t *testing.T) {
    // Créer un répertoire temporaire avec des fichiers
    tmpDir := t.TempDir()

    // Créer des fichiers de test
    testFiles := []string{"file1.txt", "file2.go", ".hidden", "subdir/file3.txt"}
    for _, file := range testFiles {
        fullPath := filepath.Join(tmpDir, file)
        err := os.MkdirAll(filepath.Dir(fullPath), 0755)
        if err != nil {
            t.Fatalf("Erreur lors de la création du répertoire: %v", err)
        }

        err = os.WriteFile(fullPath, []byte("test"), 0644)
        if err != nil {
            t.Fatalf("Erreur lors de la création du fichier %s: %v", file, err)
        }
    }

    // Tester le listage
    options := ListOptions{
        Recursive:  true,
        ShowHidden: false,
        Extension:  "",
    }

    files, err := ListFiles(tmpDir, options)
    if err != nil {
        t.Fatalf("Erreur lors du listage: %v", err)
    }

    // Vérifier que nous avons les bons fichiers (sans les cachés)
    expectedCount := 4 // tmpDir + file1.txt + file2.go + subdir + file3.txt
    if len(files) != expectedCount {
        t.Fatalf("Nombre de fichiers incorrect. Attendu: %d, Obtenu: %d", expectedCount, len(files))
    }
}
```

# Exercices pratiques

## Exercice 1 : Commande `move`

Implémentez une commande `move` qui déplace des fichiers d'un endroit à un autre.

### Objectif
Créer une commande qui permet de déplacer des fichiers et répertoires avec les mêmes options que la commande `copy`.

### Spécifications
- Déplacement de fichiers individuels
- Déplacement avec patterns (*.txt, *.go, etc.)
- Déplacement récursif de répertoires
- Confirmation avant écrasement
- Messages informatifs sur les opérations

### Code de départ - `cmd/move.go`

```go
package cmd

import (
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "github.com/spf13/cobra"
    "github.com/votre-username/gofiles/internal/fileops"
)

var (
    moveDest       string
    moveOverwrite  bool
    movePattern    string
    moveConfirm    bool
)

var moveCmd = &cobra.Command{
    Use:   "move [source] --dest [destination]",
    Short: "Déplace des fichiers ou répertoires",
    Long: `La commande move permet de déplacer des fichiers ou répertoires.

Exemples :
  gofiles move file.txt --dest ./new_location/     # Déplace un fichier
  gofiles move --pattern="*.log" --dest ./archive/ # Déplace tous les .log
  gofiles move ./temp --dest ./archive --confirm   # Avec confirmation`,

    Args: cobra.MaximumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        // TODO: Implémenter la logique de déplacement
        fmt.Println("Commande move à implémenter")
    },
}

func init() {
    rootCmd.AddCommand(moveCmd)

    moveCmd.Flags().StringVarP(&moveDest, "dest", "d", "", "Destination (requis)")
    moveCmd.Flags().BoolVar(&moveOverwrite, "overwrite", false, "Écraser les fichiers existants")
    moveCmd.Flags().StringVarP(&movePattern, "pattern", "p", "", "Pattern de fichiers")
    moveCmd.Flags().BoolVar(&moveConfirm, "confirm", false, "Demander confirmation")

    moveCmd.MarkFlagRequired("dest")
}
```

### Questions à résoudre
1. Comment adapter la fonction `performCopy` pour le déplacement ?
2. Quelle est la différence entre déplacer un fichier et le copier puis le supprimer ?
3. Comment gérer les erreurs si la suppression échoue après une copie réussie ?

### Solution proposée

```go
func performMove(source, dest string) error {
    // S'assurer que la destination existe
    if err := fileops.EnsureDir(dest); err != nil {
        return fmt.Errorf("impossible de créer le répertoire destination: %v", err)
    }

    // Vérifier si la source est un fichier unique
    if info, err := os.Stat(source); err == nil && !info.IsDir() {
        return moveSingleFile(source, dest)
    }

    // Déplacement avec pattern
    if movePattern != "" {
        return moveWithPattern(source, dest, movePattern)
    }

    // Déplacement de répertoire entier
    return moveDirectory(source, dest)
}

func moveSingleFile(source, dest string) error {
    // Si dest est un répertoire, ajouter le nom du fichier
    if fileops.IsDirectory(dest) {
        dest = filepath.Join(dest, filepath.Base(source))
    }

    // Vérifier si le fichier existe
    if _, err := os.Stat(dest); err == nil && !moveOverwrite {
        if moveConfirm {
            fmt.Printf("Le fichier '%s' existe. L'écraser ? (y/N): ", dest)
            var response string
            fmt.Scanln(&response)
            if strings.ToLower(response) != "y" && strings.ToLower(response) != "yes" {
                return fmt.Errorf("déplacement annulé")
            }
        } else {
            return fmt.Errorf("le fichier '%s' existe déjà", dest)
        }
    }

    return fileops.MoveFile(source, dest)
}
```

## Exercice 2 : Commande `search`

Créez une commande `search` qui recherche des fichiers par nom ou contenu.

### Objectif
Implémenter une fonctionnalité de recherche avancée qui peut :
- Rechercher par nom de fichier (avec wildcards)
- Rechercher par contenu de fichier
- Rechercher par taille de fichier
- Rechercher par date de modification

### Spécifications
- Support des expressions régulières
- Recherche dans le contenu des fichiers texte
- Filtres par taille (ex: >1MB, <500KB)
- Filtres par date (ex: modifié dans les 7 derniers jours)
- Recherche case-sensitive ou case-insensitive

### Code de départ - `cmd/search.go`

```go
package cmd

import (
    "fmt"
    "os"
    "regexp"
    "strings"
    "time"

    "github.com/spf13/cobra"
)

var (
    searchName      string
    searchContent   string
    searchExtension string
    searchSize      string
    searchDate      string
    searchCaseSens  bool
    searchRegex     bool
)

var searchCmd = &cobra.Command{
    Use:   "search [répertoire]",
    Short: "Recherche des fichiers selon différents critères",
    Long: `La commande search permet de rechercher des fichiers par nom, contenu,
taille ou date de modification.

Exemples :
  gofiles search --name="config"           # Fichiers contenant "config"
  gofiles search --content="TODO"          # Fichiers contenant "TODO"
  gofiles search --size=">1MB"            # Fichiers > 1MB
  gofiles search --date="<7d"             # Modifiés il y a moins de 7 jours
  gofiles search --name=".*\.go$" --regex # Avec regex`,

    Args: cobra.MaximumNArgs(1),
    Run: func(cmd *cobra.Command, args []string) {
        dir := "."
        if len(args) > 0 {
            dir = args[0]
        }

        results, err := performSearch(dir)
        if err != nil {
            fmt.Fprintf(os.Stderr, "Erreur: %v\n", err)
            os.Exit(1)
        }

        displaySearchResults(results)
    },
}

type SearchResult struct {
    Path     string
    Size     int64
    ModTime  time.Time
    Matches  []string // Pour le contenu trouvé
}

func performSearch(dir string) ([]SearchResult, error) {
    // TODO: Implémenter la recherche
    return nil, fmt.Errorf("fonction de recherche à implémenter")
}
```

### Questions à résoudre
1. Comment implémenter la recherche par taille avec des unités (KB, MB, GB) ?
2. Comment rechercher efficacement dans le contenu des fichiers ?
3. Comment gérer les fichiers binaires lors de la recherche de contenu ?

### Solution proposée - Fonctions utilitaires

```go
package fileops

import (
    "bufio"
    "fmt"
    "os"
    "regexp"
    "strconv"
    "strings"
    "time"
)

// SearchCriteria définit les critères de recherche
type SearchCriteria struct {
    NamePattern    string
    ContentPattern string
    Extension      string
    MinSize        int64
    MaxSize        int64
    MinDate        time.Time
    MaxDate        time.Time
    CaseSensitive  bool
    UseRegex       bool
}

// SearchInContent recherche un pattern dans le contenu d'un fichier
func SearchInContent(filePath, pattern string, caseSensitive, useRegex bool) ([]string, error) {
    file, err := os.Open(filePath)
    if err != nil {
        return nil, err
    }
    defer file.Close()

    // Vérifier si c'est un fichier texte
    if !isTextFile(filePath) {
        return nil, nil // Ignorer les fichiers binaires
    }

    var matches []string
    scanner := bufio.NewScanner(file)
    lineNum := 1

    for scanner.Scan() {
        line := scanner.Text()
        searchLine := line
        searchPattern := pattern

        if !caseSensitive {
            searchLine = strings.ToLower(line)
            searchPattern = strings.ToLower(pattern)
        }

        var found bool
        if useRegex {
            matched, err := regexp.MatchString(searchPattern, searchLine)
            if err != nil {
                return nil, err
            }
            found = matched
        } else {
            found = strings.Contains(searchLine, searchPattern)
        }

        if found {
            matches = append(matches, fmt.Sprintf("Ligne %d: %s", lineNum, line))
        }

        lineNum++
    }

    return matches, scanner.Err()
}

// ParseSize parse une taille avec unité (ex: "1MB", "500KB")
func ParseSize(sizeStr string) (int64, error) {
    if sizeStr == "" {
        return 0, nil
    }

    // Retirer les espaces
    sizeStr = strings.TrimSpace(sizeStr)

    // Trouver l'unité
    var multiplier int64 = 1
    var numStr string

    sizeStr = strings.ToUpper(sizeStr)
    if strings.HasSuffix(sizeStr, "GB") {
        multiplier = 1024 * 1024 * 1024
        numStr = strings.TrimSuffix(sizeStr, "GB")
    } else if strings.HasSuffix(sizeStr, "MB") {
        multiplier = 1024 * 1024
        numStr = strings.TrimSuffix(sizeStr, "MB")
    } else if strings.HasSuffix(sizeStr, "KB") {
        multiplier = 1024
        numStr = strings.TrimSuffix(sizeStr, "KB")
    } else if strings.HasSuffix(sizeStr, "B") {
        multiplier = 1
        numStr = strings.TrimSuffix(sizeStr, "B")
    } else {
        numStr = sizeStr
    }

    num, err := strconv.ParseFloat(numStr, 64)
    if err != nil {
        return 0, fmt.Errorf("taille invalide: %s", sizeStr)
    }

    return int64(num * float64(multiplier)), nil
}

// isTextFile vérifie si un fichier est probablement un fichier texte
func isTextFile(filePath string) bool {
    file, err := os.Open(filePath)
    if err != nil {
        return false
    }
    defer file.Close()

    // Lire les premiers 512 octets
    buffer := make([]byte, 512)
    n, err := file.Read(buffer)
    if err != nil {
        return false
    }

    // Vérifier la présence de caractères binaires
    for i := 0; i < n; i++ {
        b := buffer[i]
        // Caractères de contrôle (sauf tab, newline, carriage return)
        if b < 32 && b != 9 && b != 10 && b != 13 {
            return false
        }
    }

    return true
}
```

## Exercice 3 : Gestion des permissions

Ajoutez une vérification des permissions avant d'effectuer des opérations sur les fichiers.

### Objectif
Implémenter un système robuste de vérification des permissions pour éviter les erreurs lors des opérations sur les fichiers.

### Spécifications
- Vérifier les permissions de lecture avant de lire un fichier
- Vérifier les permissions d'écriture avant de copier/déplacer
- Afficher des messages d'erreur clairs
- Proposer des solutions (ex: utiliser sudo)

### Code de départ - `internal/fileops/permissions.go`

```go
package fileops

import (
    "fmt"
    "os"
    "runtime"
)

// PermissionError représente une erreur de permissions
type PermissionError struct {
    Path      string
    Operation string
    Required  string
}

func (e *PermissionError) Error() string {
    return fmt.Sprintf("permission refusée pour %s sur %s (requis: %s)",
        e.Operation, e.Path, e.Required)
}

// CheckReadPermission vérifie si on peut lire un fichier
func CheckReadPermission(path string) error {
    // TODO: Implémenter la vérification de lecture
    return nil
}

// CheckWritePermission vérifie si on peut écrire dans un répertoire
func CheckWritePermission(path string) error {
    // TODO: Implémenter la vérification d'écriture
    return nil
}

// CheckExecutePermission vérifie si on peut exécuter/traverser un répertoire
func CheckExecutePermission(path string) error {
    // TODO: Implémenter la vérification d'exécution
    return nil
}
```

### Questions à résoudre
1. Comment vérifier les permissions de manière portable (Windows/Linux/Mac) ?
2. Comment distinguer les erreurs de permissions des autres erreurs ?
3. Comment proposer des solutions à l'utilisateur ?

### Solution proposée

```go
func CheckReadPermission(path string) error {
    info, err := os.Stat(path)
    if err != nil {
        if os.IsNotExist(err) {
            return fmt.Errorf("le fichier %s n'existe pas", path)
        }
        return err
    }

    // Essayer d'ouvrir le fichier en lecture
    if info.IsDir() {
        // Pour un répertoire, essayer de le lire
        file, err := os.Open(path)
        if err != nil {
            return &PermissionError{
                Path:      path,
                Operation: "lecture",
                Required:  "r--",
            }
        }
        defer file.Close()

        _, err = file.Readdir(1)
        if err != nil && err.Error() != "EOF" {
            return &PermissionError{
                Path:      path,
                Operation: "listage",
                Required:  "r-x",
            }
        }
    } else {
        // Pour un fichier, essayer de l'ouvrir
        file, err := os.Open(path)
        if err != nil {
            return &PermissionError{
                Path:      path,
                Operation: "lecture",
                Required:  "r--",
            }
        }
        defer file.Close()
    }

    return nil
}

func CheckWritePermission(dirPath string) error {
    // Créer un fichier temporaire pour tester l'écriture
    tempFile := filepath.Join(dirPath, ".gofiles_write_test")

    file, err := os.Create(tempFile)
    if err != nil {
        return &PermissionError{
            Path:      dirPath,
            Operation: "écriture",
            Required:  "-w-",
        }
    }
    defer file.Close()
    defer os.Remove(tempFile) // Nettoyer

    return nil
}

// GetPermissionAdvice retourne des conseils pour résoudre un problème de permissions
func GetPermissionAdvice(err error) string {
    if permErr, ok := err.(*PermissionError); ok {
        advice := fmt.Sprintf("Suggestions pour résoudre le problème sur %s:\n", permErr.Path)

        if runtime.GOOS == "windows" {
            advice += "- Lancez le terminal en tant qu'administrateur\n"
            advice += "- Vérifiez les propriétés du fichier dans l'explorateur\n"
        } else {
            advice += "- Utilisez 'sudo' pour les opérations administrateur\n"
            advice += fmt.Sprintf("- Changez les permissions: chmod u+%s %s\n",
                getRequiredPermissionChars(permErr.Required), permErr.Path)
            advice += fmt.Sprintf("- Vérifiez le propriétaire: ls -la %s\n", permErr.Path)
        }

        return advice
    }

    return "Aucun conseil disponible pour cette erreur."
}

func getRequiredPermissionChars(required string) string {
    var chars string
    if strings.Contains(required, "r") {
        chars += "r"
    }
    if strings.Contains(required, "w") {
        chars += "w"
    }
    if strings.Contains(required, "x") {
        chars += "x"
    }
    return chars
}
```

## Exercice 4 : Amélioration de l'interface utilisateur

### Objectif
Améliorer l'expérience utilisateur avec des fonctionnalités avancées.

### Spécifications à implémenter

#### A. Barre de progression
```go
package ui

import (
    "fmt"
    "strings"
    "time"
)

type ProgressBar struct {
    Total     int64
    Current   int64
    Width     int
    StartTime time.Time
}

func NewProgressBar(total int64) *ProgressBar {
    return &ProgressBar{
        Total:     total,
        Width:     50,
        StartTime: time.Now(),
    }
}

func (pb *ProgressBar) Update(current int64) {
    pb.Current = current
    pb.Display()
}

func (pb *ProgressBar) Display() {
    // TODO: Implémenter l'affichage de la barre de progression
    // Format: [████████████████████████████████████████████████████] 100% (1.2MB/s)
}
```

#### B. Coloration de la sortie
```go
package ui

const (
    ColorReset  = "\033[0m"
    ColorRed    = "\033[31m"
    ColorGreen  = "\033[32m"
    ColorYellow = "\033[33m"
    ColorBlue   = "\033[34m"
    ColorPurple = "\033[35m"
    ColorCyan   = "\033[36m"
    ColorWhite  = "\033[37m"
)

func ColorText(text, color string) string {
    return color + text + ColorReset
}

func ErrorText(text string) string {
    return ColorText(text, ColorRed)
}

func SuccessText(text string) string {
    return ColorText(text, ColorGreen)
}

func WarningText(text string) string {
    return ColorText(text, ColorYellow)
}
```

## Exercice 5 : Tests d'intégration

### Objectif
Créer des tests qui vérifient le bon fonctionnement de l'ensemble du CLI.

### Spécifications
- Tests end-to-end des commandes principales
- Tests avec différents scénarios d'erreur
- Tests de performance sur de gros volumes

### Code de départ - `test/integration_test.go`

```go
package test

import (
    "os"
    "os/exec"
    "path/filepath"
    "testing"
)

// TestCLICommands teste les commandes principales du CLI
func TestCLICommands(t *testing.T) {
    // Créer un environnement de test
    testDir := t.TempDir()

    // Compiler le CLI
    binaryPath := filepath.Join(testDir, "gofiles")
    cmd := exec.Command("go", "build", "-o", binaryPath, "../../main.go")
    if err := cmd.Run(); err != nil {
        t.Fatalf("Erreur lors de la compilation: %v", err)
    }

    // Test de la commande list
    t.Run("ListCommand", func(t *testing.T) {
        cmd := exec.Command(binaryPath, "list", "--help")
        output, err := cmd.Output()
        if err != nil {
            t.Fatalf("Erreur lors de l'exécution de 'list --help': %v", err)
        }

        if len(output) == 0 {
            t.Fatal("Aucune sortie pour 'list --help'")
        }
    })

    // TODO: Ajouter plus de tests
}
```

## Résumé des exercices

Ces exercices couvrent les aspects essentiels du développement d'un CLI robuste :

1. **Commande move** : Gestion du déplacement de fichiers
2. **Commande search** : Recherche avancée avec différents critères
3. **Gestion des permissions** : Sécurité et validation
4. **Interface utilisateur** : Amélioration de l'expérience utilisateur
5. **Tests d'intégration** : Validation du fonctionnement global

### Conseils pour la résolution

1. **Commencez simple** : Implémentez d'abord les cas basiques
2. **Testez fréquemment** : Vérifiez chaque fonctionnalité au fur et à mesure
3. **Gérez les erreurs** : Pensez aux cas d'erreur et aux messages utilisateur
4. **Documentez** : Ajoutez des commentaires et des exemples d'utilisation
5. **Optimisez progressivement** : Améliorez les performances après avoir un code fonctionnel

### Prochaines étapes

Une fois ces exercices terminés, vous serez prêt pour la section 15-3 qui couvrira la distribution et le packaging de votre outil CLI, incluant :
- Compilation cross-platform
- Création de packages pour différents OS
- Distribution via GitHub Releases
- Création d'installateurs

⏭️
