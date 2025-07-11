🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 15-3 : Distribution et packaging

## Introduction

Créer un outil CLI fonctionnel n'est que la moitié du travail. Pour que votre outil soit vraiment utile, il faut que les utilisateurs puissent l'installer et l'utiliser facilement. Cette section couvre tout ce qu'il faut savoir sur la distribution et le packaging d'applications Go.

## Qu'est-ce que la distribution ?

La distribution consiste à rendre votre application disponible pour les utilisateurs finaux. Cela inclut :

- **Compilation** pour différents systèmes d'exploitation
- **Packaging** dans des formats appropriés
- **Distribution** via des canaux accessibles
- **Installation** simple pour l'utilisateur final

## Compilation Go : les bases

### Compilation simple

```bash
# Compiler pour votre système actuel
go build -o gofiles main.go

# Compiler avec optimisations
go build -ldflags="-s -w" -o gofiles main.go
```

Les flags `-ldflags="-s -w"` :
- `-s` : Supprime la table des symboles
- `-w` : Supprime les informations de débogage
- Résultat : Binaire plus petit

### Compilation cross-platform

Go permet de compiler facilement pour différents systèmes :

```bash
# Variables d'environnement importantes
# GOOS : Système d'exploitation cible
# GOARCH : Architecture cible

# Linux 64-bit
GOOS=linux GOARCH=amd64 go build -o gofiles-linux-amd64 main.go

# Windows 64-bit
GOOS=windows GOARCH=amd64 go build -o gofiles-windows-amd64.exe main.go

# macOS 64-bit
GOOS=darwin GOARCH=amd64 go build -o gofiles-darwin-amd64 main.go

# macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o gofiles-darwin-arm64 main.go
```

### Systèmes et architectures supportés

```bash
# Voir toutes les combinaisons possibles
go tool dist list

# Quelques exemples populaires :
# linux/amd64    - Linux 64-bit
# linux/arm64    - Linux ARM64 (Raspberry Pi, etc.)
# windows/amd64  - Windows 64-bit
# darwin/amd64   - macOS Intel
# darwin/arm64   - macOS Apple Silicon
# freebsd/amd64  - FreeBSD 64-bit
```

## Automatisation avec Makefile

Créons un `Makefile` pour automatiser la compilation :

```makefile
# Variables
BINARY_NAME=gofiles
VERSION=1.0.0
BUILD_DIR=build
LDFLAGS=-ldflags="-s -w -X main.version=${VERSION}"

# Commande par défaut
.PHONY: all
all: clean build

# Nettoyer les builds précédents
.PHONY: clean
clean:
	rm -rf ${BUILD_DIR}
	mkdir -p ${BUILD_DIR}

# Build pour le système local
.PHONY: build
build:
	go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME} main.go

# Build pour tous les systèmes
.PHONY: build-all
build-all: clean
	# Linux
	GOOS=linux GOARCH=amd64 go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME}-linux-amd64 main.go
	GOOS=linux GOARCH=arm64 go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME}-linux-arm64 main.go

	# Windows
	GOOS=windows GOARCH=amd64 go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME}-windows-amd64.exe main.go

	# macOS
	GOOS=darwin GOARCH=amd64 go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME}-darwin-amd64 main.go
	GOOS=darwin GOARCH=arm64 go build ${LDFLAGS} -o ${BUILD_DIR}/${BINARY_NAME}-darwin-arm64 main.go

# Créer des archives
.PHONY: package
package: build-all
	cd ${BUILD_DIR} && \
	tar -czf ${BINARY_NAME}-linux-amd64.tar.gz ${BINARY_NAME}-linux-amd64 && \
	tar -czf ${BINARY_NAME}-linux-arm64.tar.gz ${BINARY_NAME}-linux-arm64 && \
	tar -czf ${BINARY_NAME}-darwin-amd64.tar.gz ${BINARY_NAME}-darwin-amd64 && \
	tar -czf ${BINARY_NAME}-darwin-arm64.tar.gz ${BINARY_NAME}-darwin-arm64 && \
	zip ${BINARY_NAME}-windows-amd64.zip ${BINARY_NAME}-windows-amd64.exe

# Installer localement
.PHONY: install
install: build
	cp ${BUILD_DIR}/${BINARY_NAME} /usr/local/bin/

# Tests
.PHONY: test
test:
	go test ./...

# Afficher l'aide
.PHONY: help
help:
	@echo "Commandes disponibles :"
	@echo "  make build      - Compiler pour le système local"
	@echo "  make build-all  - Compiler pour tous les systèmes"
	@echo "  make package    - Créer les archives"
	@echo "  make install    - Installer localement"
	@echo "  make clean      - Nettoyer les builds"
	@echo "  make test       - Lancer les tests"
```

Usage du Makefile :

```bash
# Compiler pour le système local
make build

# Compiler pour tous les systèmes
make build-all

# Créer les packages
make package

# Installer localement
make install
```

## Gestion des versions

### Intégrer la version dans le binaire

Modifions `main.go` pour inclure la version :

```go
package main

import (
    "fmt"
    "os"

    "github.com/votre-username/gofiles/cmd"
)

// Variables injectées à la compilation
var (
    version   = "dev"
    buildTime = "unknown"
    gitCommit = "unknown"
)

func main() {
    // Définir les informations de version pour cobra
    cmd.SetVersionInfo(version, buildTime, gitCommit)

    if err := cmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

Mettons à jour `cmd/root.go` :

```go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
)

var (
    version   string
    buildTime string
    gitCommit string
)

var rootCmd = &cobra.Command{
    Use:     "gofiles",
    Short:   "Un gestionnaire de fichiers CLI puissant",
    Version: version,
}

// SetVersionInfo définit les informations de version
func SetVersionInfo(v, bt, gc string) {
    version = v
    buildTime = bt
    gitCommit = gc

    // Personnaliser l'affichage de la version
    rootCmd.SetVersionTemplate(`{{printf "%s version %s\n" .Name .Version}}{{printf "Build time: %s\n" "` + buildTime + `"}}{{printf "Git commit: %s\n" "` + gitCommit + `"}}`)
}

func Execute() error {
    return rootCmd.Execute()
}
```

### Injection de variables à la compilation

```bash
# Obtenir des informations de build
VERSION=$(git describe --tags --always --dirty)
BUILD_TIME=$(date +%Y-%m-%dT%H:%M:%S%z)
GIT_COMMIT=$(git rev-parse HEAD)

# Compiler avec ces informations
go build -ldflags="-X main.version=${VERSION} -X main.buildTime=${BUILD_TIME} -X main.gitCommit=${GIT_COMMIT}" -o gofiles main.go
```

Mettons à jour le Makefile :

```makefile
# Variables dynamiques
VERSION=$(shell git describe --tags --always --dirty)
BUILD_TIME=$(shell date +%Y-%m-%dT%H:%M:%S%z)
GIT_COMMIT=$(shell git rev-parse HEAD)
LDFLAGS=-ldflags="-s -w -X main.version=${VERSION} -X main.buildTime=${BUILD_TIME} -X main.gitCommit=${GIT_COMMIT}"
```

## Packaging pour différents systèmes

### 1. Archives TAR/ZIP (Universal)

```bash
# Créer une structure de package
mkdir -p package/gofiles-1.0.0
cp build/gofiles-linux-amd64 package/gofiles-1.0.0/gofiles
cp README.md package/gofiles-1.0.0/
cp LICENSE package/gofiles-1.0.0/

# Créer l'archive
cd package
tar -czf gofiles-1.0.0-linux-amd64.tar.gz gofiles-1.0.0/
```

### 2. Packages DEB (Debian/Ubuntu)

Créons une structure pour un package Debian :

```
debian-package/
├── DEBIAN/
│   ├── control
│   ├── postinst
│   └── prerm
└── usr/
    ├── local/
    │   └── bin/
    │       └── gofiles
    └── share/
        └── doc/
            └── gofiles/
                ├── README.md
                └── changelog
```

Fichier `debian-package/DEBIAN/control` :

```
Package: gofiles
Version: 1.0.0
Section: utils
Priority: optional
Architecture: amd64
Depends: libc6
Maintainer: Votre Nom <votre.email@example.com>
Description: Gestionnaire de fichiers CLI
 GoFiles est un outil en ligne de commande puissant pour gérer
 vos fichiers efficacement. Il offre des fonctionnalités comme
 la liste, la copie, la recherche et la compression de fichiers.
```

Fichier `debian-package/DEBIAN/postinst` :

```bash
#!/bin/bash
echo "GoFiles installé avec succès!"
echo "Utilisez 'gofiles --help' pour commencer."
```

Création du package :

```bash
# Copier le binaire
cp build/gofiles-linux-amd64 debian-package/usr/local/bin/gofiles
chmod +x debian-package/usr/local/bin/gofiles

# Définir les permissions
chmod 755 debian-package/DEBIAN/postinst

# Créer le package
dpkg-deb --build debian-package gofiles-1.0.0-amd64.deb
```

### 3. Packages RPM (Red Hat/CentOS/Fedora)

Créons un fichier spec `gofiles.spec` :

```spec
Name:           gofiles
Version:        1.0.0
Release:        1%{?dist}
Summary:        Gestionnaire de fichiers CLI

License:        MIT
URL:            https://github.com/votre-username/gofiles
Source0:        %{name}-%{version}.tar.gz

BuildRequires:  golang >= 1.19
Requires:       glibc

%description
GoFiles est un outil en ligne de commande puissant pour gérer
vos fichiers efficacement.

%prep
%setup -q

%build
go build -ldflags="-s -w" -o %{name} main.go

%install
mkdir -p %{buildroot}%{_bindir}
install -m 755 %{name} %{buildroot}%{_bindir}/%{name}

%files
%{_bindir}/%{name}
%doc README.md
%license LICENSE

%changelog
* Mon Jan 01 2024 Votre Nom <votre.email@example.com> - 1.0.0-1
- Version initiale
```

Construction du RPM :

```bash
# Préparer l'environnement
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}

# Copier le spec
cp gofiles.spec ~/rpmbuild/SPECS/

# Créer l'archive source
tar -czf ~/rpmbuild/SOURCES/gofiles-1.0.0.tar.gz .

# Construire le RPM
rpmbuild -ba ~/rpmbuild/SPECS/gofiles.spec
```

## Distribution via GitHub

### 1. GitHub Releases

Créons un script pour automatiser les releases :

```bash
#!/bin/bash
# scripts/release.sh

set -e

VERSION=$1
if [ -z "$VERSION" ]; then
    echo "Usage: $0 <version>"
    echo "Example: $0 v1.0.0"
    exit 1
fi

echo "Creating release $VERSION..."

# Créer le tag
git tag -a $VERSION -m "Release $VERSION"
git push origin $VERSION

# Compiler pour toutes les plateformes
make build-all
make package

# Créer la release GitHub (nécessite gh CLI)
gh release create $VERSION \
    --title "Release $VERSION" \
    --notes "See CHANGELOG.md for details" \
    build/*.tar.gz \
    build/*.zip
```

### 2. GitHub Actions pour CI/CD

Créons `.github/workflows/release.yml` :

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3

    - name: Set up Go
      uses: actions/setup-go@v3
      with:
        go-version: 1.19

    - name: Get version
      id: version
      run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

    - name: Build all platforms
      run: |
        make build-all
        make package

    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        files: |
          build/*.tar.gz
          build/*.zip
        body: |
          ## Changes in ${{ steps.version.outputs.VERSION }}

          See [CHANGELOG.md](CHANGELOG.md) for detailed changes.

          ## Installation

          Download the appropriate binary for your system and add it to your PATH.
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Gestionnaires de packages

### 1. Homebrew (macOS/Linux)

Créons une formule Homebrew `Formula/gofiles.rb` :

```ruby
class Gofiles < Formula
  desc "Gestionnaire de fichiers CLI puissant"
  homepage "https://github.com/votre-username/gofiles"
  url "https://github.com/votre-username/gofiles/archive/v1.0.0.tar.gz"
  sha256 "sha256_du_tarball"
  license "MIT"

  depends_on "go" => :build

  def install
    system "go", "build", *std_go_args(ldflags: "-s -w"), "./main.go"
  end

  test do
    assert_match "gofiles version", shell_output("#{bin}/gofiles --version")
  end
end
```

### 2. Snap (Linux)

Créons un `snapcraft.yaml` :

```yaml
name: gofiles
version: '1.0.0'
summary: Gestionnaire de fichiers CLI
description: |
  GoFiles est un outil en ligne de commande puissant pour gérer
  vos fichiers efficacement.

grade: stable
confinement: strict

parts:
  gofiles:
    plugin: go
    source: .
    build-snaps: [go]

apps:
  gofiles:
    command: bin/gofiles
    plugs: [home, removable-media]
```

Construction du snap :

```bash
# Installer snapcraft
sudo apt install snapcraft

# Construire le snap
snapcraft

# Installer localement pour tester
sudo snap install gofiles_1.0.0_amd64.snap --dangerous
```

### 3. Chocolatey (Windows)

Créons un package Chocolatey `tools/chocolateyinstall.ps1` :

```powershell
$ErrorActionPreference = 'Stop'

$packageName = 'gofiles'
$url64 = 'https://github.com/votre-username/gofiles/releases/download/v1.0.0/gofiles-windows-amd64.zip'
$checksum64 = 'SHA256_CHECKSUM'

$packageArgs = @{
  packageName   = $packageName
  url64bit      = $url64
  checksum64    = $checksum64
  checksumType64= 'sha256'
  unzipLocation = "$(Split-Path -parent $MyInvocation.MyCommand.Definition)"
}

Install-ChocolateyZipPackage @packageArgs
```

## Scripts d'installation

### Installation automatique (Unix)

Créons `install.sh` :

```bash
#!/bin/bash

set -e

# Configuration
REPO="votre-username/gofiles"
BINARY_NAME="gofiles"
INSTALL_DIR="/usr/local/bin"

# Détecter l'OS et l'architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

case $ARCH in
    x86_64) ARCH="amd64" ;;
    arm64|aarch64) ARCH="arm64" ;;
    *) echo "Architecture non supportée: $ARCH"; exit 1 ;;
esac

# Obtenir la dernière version
VERSION=$(curl -s "https://api.github.com/repos/$REPO/releases/latest" | grep '"tag_name"' | cut -d'"' -f4)

# Construire l'URL de téléchargement
DOWNLOAD_URL="https://github.com/$REPO/releases/download/$VERSION/${BINARY_NAME}-${OS}-${ARCH}.tar.gz"

echo "Installation de $BINARY_NAME $VERSION pour $OS-$ARCH..."

# Télécharger et installer
TMP_DIR=$(mktemp -d)
cd $TMP_DIR

echo "Téléchargement depuis $DOWNLOAD_URL..."
curl -L -o "${BINARY_NAME}.tar.gz" "$DOWNLOAD_URL"

echo "Extraction..."
tar -xzf "${BINARY_NAME}.tar.gz"

echo "Installation dans $INSTALL_DIR..."
sudo mv "${BINARY_NAME}-${OS}-${ARCH}" "$INSTALL_DIR/$BINARY_NAME"
sudo chmod +x "$INSTALL_DIR/$BINARY_NAME"

# Nettoyage
cd /
rm -rf $TMP_DIR

echo "$BINARY_NAME installé avec succès!"
echo "Utilisez '$BINARY_NAME --help' pour commencer."
```

### Installation Windows (PowerShell)

Créons `install.ps1` :

```powershell
$ErrorActionPreference = 'Stop'

# Configuration
$Repo = "votre-username/gofiles"
$BinaryName = "gofiles"
$InstallDir = "$env:USERPROFILE\bin"

# Créer le répertoire d'installation
if (!(Test-Path $InstallDir)) {
    New-Item -ItemType Directory -Path $InstallDir | Out-Null
}

# Obtenir la dernière version
$LatestRelease = Invoke-RestMethod -Uri "https://api.github.com/repos/$Repo/releases/latest"
$Version = $LatestRelease.tag_name

# Construire l'URL de téléchargement
$DownloadUrl = "https://github.com/$Repo/releases/download/$Version/$BinaryName-windows-amd64.zip"

Write-Host "Installation de $BinaryName $Version..."

# Télécharger
$TempFile = [System.IO.Path]::GetTempFileName() + ".zip"
Write-Host "Téléchargement depuis $DownloadUrl..."
Invoke-WebRequest -Uri $DownloadUrl -OutFile $TempFile

# Extraire
Write-Host "Extraction..."
Expand-Archive -Path $TempFile -DestinationPath $InstallDir -Force

# Nettoyer
Remove-Item $TempFile

# Ajouter au PATH si nécessaire
$CurrentPath = [Environment]::GetEnvironmentVariable("PATH", "User")
if ($CurrentPath -notlike "*$InstallDir*") {
    Write-Host "Ajout de $InstallDir au PATH..."
    [Environment]::SetEnvironmentVariable("PATH", "$CurrentPath;$InstallDir", "User")
    Write-Host "Redémarrez votre terminal pour que les changements prennent effet."
}

Write-Host "$BinaryName installé avec succès!"
Write-Host "Utilisez '$BinaryName --help' pour commencer."
```

## Documentation pour les utilisateurs

### README.md complet

```markdown
# GoFiles

Un gestionnaire de fichiers CLI puissant et facile à utiliser.

## Installation

### Installation automatique (Linux/macOS)

```bash
curl -fsSL https://raw.githubusercontent.com/votre-username/gofiles/main/install.sh | bash
```

### Installation automatique (Windows)

```powershell
iwr -useb https://raw.githubusercontent.com/votre-username/gofiles/main/install.ps1 | iex
```

### Installation manuelle

1. Téléchargez la dernière version depuis [Releases](https://github.com/votre-username/gofiles/releases)
2. Extrayez l'archive
3. Déplacez le binaire dans votre PATH

### Gestionnaires de packages

#### Homebrew (macOS/Linux)
```bash
brew install votre-username/tap/gofiles
```

#### Snap (Linux)
```bash
sudo snap install gofiles
```

#### Chocolatey (Windows)
```bash
choco install gofiles
```

## Utilisation rapide

```bash
# Lister les fichiers
gofiles list --ext=go --recursive

# Copier des fichiers
gofiles copy --pattern="*.txt" --dest=./backup

# Rechercher des fichiers
gofiles search --name="config" --content="password"
```

## Documentation complète

Voir le [Wiki](https://github.com/votre-username/gofiles/wiki) pour la documentation complète.
```

## Tests de distribution

### Script de test automatisé

Créons `scripts/test-distribution.sh` :

```bash
#!/bin/bash

set -e

echo "Test de distribution de GoFiles..."

# Test 1: Compilation pour toutes les plateformes
echo "Test 1: Compilation cross-platform..."
make build-all

# Vérifier que tous les binaires existent
for binary in build/gofiles-*; do
    if [ ! -f "$binary" ]; then
        echo "Erreur: $binary n'existe pas"
        exit 1
    fi
    echo "✓ $binary créé"
done

# Test 2: Packages
echo "Test 2: Création des packages..."
make package

# Test 3: Installation locale
echo "Test 3: Test d'installation locale..."
make install
which gofiles > /dev/null || { echo "Erreur: gofiles non trouvé dans PATH"; exit 1; }
echo "✓ Installation locale réussie"

# Test 4: Fonctionnalité de base
echo "Test 4: Test fonctionnel..."
gofiles --version > /dev/null || { echo "Erreur: gofiles --version échoue"; exit 1; }
gofiles list --help > /dev/null || { echo "Erreur: gofiles list --help échoue"; exit 1; }
echo "✓ Tests fonctionnels réussis"

echo "Tous les tests de distribution sont passés!"
```

## Meilleures pratiques

### 1. Versioning sémantique

Utilisez [Semantic Versioning](https://semver.org/) :
- `MAJOR.MINOR.PATCH`
- `1.0.0` : Version stable initiale
- `1.1.0` : Nouvelles fonctionnalités compatibles
- `1.0.1` : Corrections de bugs
- `2.0.0` : Changements incompatibles

### 2. Changelog

Maintenez un `CHANGELOG.md` :

```markdown
# Changelog

## [1.1.0] - 2024-01-15

### Added
- Nouvelle commande `search` avec support regex
- Support des archives ZIP

### Changed
- Amélioration des performances de la commande `list`

### Fixed
- Correction du bug de permissions sur Windows

## [1.0.0] - 2024-01-01

### Added
- Version initiale avec commandes `list` et `copy`
```

### 3. Sécurité

- Signez vos binaires quand possible
- Fournissez des checksums (SHA256)
- Utilisez HTTPS pour tous les téléchargements

### 4. Documentation

- README clair avec exemples
- Documentation d'installation pour chaque plateforme
- Documentation d'utilisation complète
- FAQ pour les problèmes courants

## Résumé

Dans cette section, nous avons couvert :

- **Compilation cross-platform** avec Go
- **Automatisation** avec Makefile et scripts
- **Packaging** pour différents systèmes (DEB, RPM, Snap)
- **Distribution** via GitHub Releases et gestionnaires de packages
- **Installation automatisée** avec scripts
- **Tests de distribution** et bonnes pratiques

Votre outil CLI GoFiles est maintenant prêt à être distribué professionnellement ! Les utilisateurs peuvent l'installer facilement sur leur système et commencer à l'utiliser immédiatement.

## Prochaines étapes

Maintenant que vous maîtrisez la distribution, vous pourriez vouloir explorer :
- Monitoring et télémétrie d'usage
- Système de mise à jour automatique
- Plugins et extensions
- Interface web complémentaire

⏭️
