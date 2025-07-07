🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2-4 : Commentaires et conventions de nommage

## Introduction

Écrire du code qui fonctionne n'est que la moitié du travail d'un développeur. L'autre moitié consiste à écrire du code **lisible**, **compréhensible** et **maintenable**. Les commentaires et les conventions de nommage sont vos outils pour transformer du code technique en une histoire claire que tout développeur peut comprendre.

## Pourquoi les commentaires et conventions sont-ils importants ?

### Analogie du livre

Imaginez deux livres :
- **Livre A** : Pas de titre de chapitre, pas de paragraphes, mots inventés
- **Livre B** : Titres clairs, paragraphes organisés, mots du dictionnaire

Lequel préférez-vous lire ? C'est pareil pour le code !

### Réalité du développement

**80% du temps** d'un développeur est consacré à **lire** du code existant, pas à en écrire du nouveau. Un code bien commenté et bien nommé vous fera gagner énormément de temps.

## Les commentaires en Go

### Types de commentaires

Go propose deux types de commentaires :

#### 1. Commentaires de ligne (`//`)

```go
// Ceci est un commentaire de ligne
var age int = 25  // Commentaire en fin de ligne
```

#### 2. Commentaires de bloc (`/* */`)

```go
/*
Ceci est un commentaire de bloc
qui peut s'étendre sur
plusieurs lignes
*/
var nom string = "Alice"
```

**Recommandation :** Utilisez principalement `//` en Go, c'est la convention.

### Quand et comment commenter

#### ✅ Bonnes pratiques

**1. Expliquer le POURQUOI, pas le QUOI**

```go
// ❌ Mauvais : explique ce que fait le code (évident)
age++ // Incrémente age de 1

// ✅ Bon : explique pourquoi on le fait
age++ // Compenser l'année bissextile dans le calcul
```

**2. Commenter les algorithmes complexes**

```go
// Algorithme de validation Luhn pour les numéros de carte bancaire
// Ref: https://en.wikipedia.org/wiki/Luhn_algorithm
func validerNumeroCarte(numero string) bool {
    somme := 0
    estPair := false

    // Parcourir de droite à gauche
    for i := len(numero) - 1; i >= 0; i-- {
        chiffre := int(numero[i] - '0')

        if estPair {
            chiffre *= 2
            if chiffre > 9 {
                chiffre -= 9
            }
        }

        somme += chiffre
        estPair = !estPair
    }

    return somme%10 == 0
}
```

**3. Documenter les fonctions publiques**

```go
// CalculerTVA calcule la TVA pour un prix donné.
// Le taux doit être exprimé en pourcentage (ex: 20 pour 20%).
// Retourne le montant de TVA et le prix TTC.
func CalculerTVA(prixHT float64, taux float64) (tva float64, prixTTC float64) {
    tva = prixHT * taux / 100
    prixTTC = prixHT + tva
    return
}
```

**4. Expliquer les décisions de conception**

```go
// Utilisation d'une map au lieu d'un slice pour des recherches O(1)
// Le projet peut avoir jusqu'à 100k utilisateurs
var utilisateurs = make(map[string]*Utilisateur)

// Timeout de 30 secondes pour éviter les blocages réseau
// mais assez long pour les connexions lentes
const TIMEOUT_RESEAU = 30 * time.Second
```

#### ❌ Commentaires à éviter

**1. Commentaires évidents**

```go
// ❌ Inutile
var nom string = "Alice"  // Déclare une variable nom de type string

// ❌ Redondant
i++  // Augmente i de 1
```

**2. Commentaires obsolètes**

```go
// ❌ Commentaire faux/obsolète
// Calcule la TVA à 19.6% (ancien taux français)
tva := prix * 0.20  // Le code fait 20%, pas 19.6% !
```

**3. Commentaires insultants ou négatifs**

```go
// ❌ Peu professionnel
// Ce code est horrible mais ça marche
// Code écrit par un stagiaire, attention aux bugs
```

### Documentation automatique avec godoc

Go intègre un système de documentation automatique. Les commentaires suivant certaines règles deviennent de la documentation.

#### Règles de godoc

**1. Commenter les packages**

```go
// Package calculatrice fournit des fonctions mathématiques de base
// pour les calculs commerciaux avec gestion de la TVA et des remises.
//
// Exemple d'utilisation :
//     tva, ttc := calculatrice.CalculerTVA(100.0, 20.0)
//     fmt.Printf("TVA: %.2f, TTC: %.2f", tva, ttc)
package calculatrice
```

**2. Commenter les fonctions exportées**

```go
// CalculerRemise calcule le montant de la remise selon le type de client.
//
// Les types de clients supportés sont :
//   - "particulier" : 0% de remise
//   - "professionnel" : 10% de remise
//   - "vip" : 20% de remise
//
// Retourne une erreur si le type de client n'est pas reconnu.
func CalculerRemise(montant float64, typeClient string) (float64, error) {
    // ...
}
```

**3. Commenter les types exportés**

```go
// Utilisateur représente un utilisateur de l'application.
type Utilisateur struct {
    // ID est l'identifiant unique de l'utilisateur
    ID int

    // Nom est le nom de famille (obligatoire)
    Nom string

    // Email doit être une adresse email valide
    Email string

    // DateInscription est automatiquement définie à la création
    DateInscription time.Time
}
```

**4. Commenter les constantes importantes**

```go
// Codes d'erreur HTTP personnalisés
const (
    // ErreurUtilisateurInexistant indique que l'utilisateur demandé n'existe pas
    ErreurUtilisateurInexistant = 4001

    // ErreurMotDePasseIncorrect indique une tentative de connexion avec un mauvais mot de passe
    ErreurMotDePasseIncorrect = 4002
)
```

### Voir la documentation

```bash
# Voir la documentation d'un package
go doc fmt.Println

# Générer et voir la documentation de votre projet
godoc -http=:6060
# Puis aller sur http://localhost:6060
```

## Conventions de nommage

### Philosophie Go

Go privilégie la **simplicité** et la **clarté**. Les noms doivent être :
- **Courts** mais **descriptifs**
- **Cohérents** dans tout le projet
- **Prononçables** et **mémorisables**

### Règles générales

#### 1. CamelCase vs snake_case

**Go utilise exclusivement le camelCase :**

```go
// ✅ Correct (Go style)
var nomUtilisateur string
var dateNaissance time.Time
var estActif bool

// ❌ Incorrect (pas du Go)
var nom_utilisateur string
var date_naissance time.Time
var est_actif bool
```

#### 2. Visibilité : Majuscule vs Minuscule

**La première lettre détermine la visibilité :**

```go
// ✅ Exporté (public) - accessible depuis d'autres packages
var NomPublic string
func FonctionPublique() {}
type StructPublique struct {}

// ✅ Non exporté (privé) - accessible uniquement dans le même package
var nomPrive string
func fonctionPrivee() {}
type structPrivee struct {}
```

### Conventions par type d'élément

#### Variables

**Variables locales : courts et précis**

```go
// ✅ Bon pour les variables temporaires
for i := 0; i < 10; i++ {
    // i est clair dans ce contexte
}

var u *User  // u pour user dans un contexte évident

// ✅ Bon pour les variables importantes
var utilisateurConnecte *User
var messageErreur string
var compteurTentatives int
```

**Variables globales : descriptives**

```go
// ✅ Claires et descriptives
var ConfigurationServeur *Config
var BaseDeDonnees *sql.DB
var LoggerPrincipal *log.Logger
```

#### Constantes

```go
// ✅ SCREAMING_SNAKE_CASE pour les constantes importantes
const (
    VERSION_APPLICATION = "1.0.0"
    TIMEOUT_CONNEXION   = 30 * time.Second
    TAILLE_BUFFER_MAX   = 1024
)

// ✅ camelCase pour les constantes locales
const (
    tailleDefaut = 10
    couleurDefaut = "bleu"
)
```

#### Fonctions

```go
// ✅ Verbes d'action
func creerUtilisateur() {}
func validerEmail() {}
func obtenirConfiguration() {}
func mettreAJourStatut() {}

// ✅ Questions pour les fonctions booléennes
func estValide() bool {}
func peutAcceder() bool {}
func aPermission() bool {}
```

#### Types (structs, interfaces)

```go
// ✅ Noms (pas de verbes)
type Utilisateur struct {}
type Configuration struct {}
type GestionnaireEmail struct {}

// ✅ Interfaces : souvent terminées par -er ou -able
type Lecteur interface {
    Lire() ([]byte, error)
}

type Validable interface {
    Valider() error
}

type Convertisseur interface {
    Convertir() string
}
```

#### Packages

```go
// ✅ Court, descriptif, pas de majuscules
package user
package email
package config
package http

// ❌ À éviter
package User           // Majuscule
package gestionUser    // Trop long
package utils          // Trop vague
```

### Acronymes et abréviations

**Règle spéciale : les acronymes gardent leur casse**

```go
// ✅ Correct
var URLSite string      // pas urlSite
var IDUtilisateur int   // pas idUtilisateur
var HTTPClient *http.Client  // pas httpClient

type APIResponse struct {}   // pas apiResponse
func ParseXML() {}          // pas parseXml
```

**Abréviations courantes acceptées :**

```go
// ✅ Acceptées dans Go
var config *Config  // au lieu de configuration
var doc *Document   // au lieu de document
var ctx context.Context  // au lieu de contexte
var req *Request    // au lieu de requete
var resp *Response  // au lieu de reponse
```

### Exemples pratiques de bon nommage

#### Exemple 1 : Application e-commerce

```go
package ecommerce

import "time"

// Produit représente un article en vente
type Produit struct {
    ID          int       `json:"id"`
    Nom         string    `json:"nom"`
    Prix        float64   `json:"prix"`
    Description string    `json:"description"`
    Stock       int       `json:"stock"`
    DateAjout   time.Time `json:"date_ajout"`
}

// Panier représente le panier d'un utilisateur
type Panier struct {
    ID           int
    UtilisateurID int
    Articles     []ArticlePanier
    DateCreation time.Time
}

// ArticlePanier représente un produit dans un panier
type ArticlePanier struct {
    ProduitID int
    Quantite  int
    PrixUnitaire float64
}

// CreerProduit ajoute un nouveau produit au catalogue
func CreerProduit(nom string, prix float64, stock int) (*Produit, error) {
    // Implementation...
}

// AjouterAuPanier ajoute un produit au panier de l'utilisateur
func (p *Panier) AjouterArticle(produitID int, quantite int) error {
    // Implementation...
}

// CalculerTotal calcule le montant total du panier
func (p *Panier) CalculerTotal() float64 {
    var total float64
    for _, article := range p.Articles {
        total += article.PrixUnitaire * float64(article.Quantite)
    }
    return total
}

// EstDisponible vérifie si un produit est en stock
func (p *Produit) EstDisponible(quantiteDemandee int) bool {
    return p.Stock >= quantiteDemandee
}
```

#### Exemple 2 : Gestionnaire d'utilisateurs

```go
package user

import (
    "errors"
    "time"
)

// Erreurs courantes
var (
    ErrUtilisateurInexistant = errors.New("utilisateur inexistant")
    ErrEmailDejaUtilise      = errors.New("email déjà utilisé")
    ErrMotDePasseTropCourt   = errors.New("mot de passe trop court")
)

// Constantes de validation
const (
    TailleMotDePasseMin = 8
    TailleNomMax        = 100
    FormatEmail         = `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
)

// Utilisateur représente un utilisateur de l'application
type Utilisateur struct {
    id              int       // privé
    Nom             string    `json:"nom"`
    Email           string    `json:"email"`
    MotDePasseHash  string    `json:"-"`  // Exclu du JSON
    DateInscription time.Time `json:"date_inscription"`
    EstActif        bool      `json:"est_actif"`
}

// Service de gestion des utilisateurs
type ServiceUtilisateur struct {
    repo Repository  // Interface pour la base de données
}

// Repository définit les opérations de base de données
type Repository interface {
    Creer(utilisateur *Utilisateur) error
    ObtenirParID(id int) (*Utilisateur, error)
    ObtenirParEmail(email string) (*Utilisateur, error)
    MettreAJour(utilisateur *Utilisateur) error
    Supprimer(id int) error
}

// NouvelUtilisateur crée une nouvelle instance d'utilisateur
func NouvelUtilisateur(nom, email, motDePasse string) (*Utilisateur, error) {
    if err := validerEmail(email); err != nil {
        return nil, err
    }

    if err := validerMotDePasse(motDePasse); err != nil {
        return nil, err
    }

    return &Utilisateur{
        Nom:             nom,
        Email:           email,
        MotDePasseHash:  hasherMotDePasse(motDePasse),
        DateInscription: time.Now(),
        EstActif:        true,
    }, nil
}

// ObtenirID retourne l'ID de l'utilisateur
func (u *Utilisateur) ObtenirID() int {
    return u.id
}

// ChangerMotDePasse met à jour le mot de passe de l'utilisateur
func (u *Utilisateur) ChangerMotDePasse(nouveauMotDePasse string) error {
    if err := validerMotDePasse(nouveauMotDePasse); err != nil {
        return err
    }

    u.MotDePasseHash = hasherMotDePasse(nouveauMotDePasse)
    return nil
}

// VerifierMotDePasse vérifie si le mot de passe est correct
func (u *Utilisateur) VerifierMotDePasse(motDePasse string) bool {
    return verifierHash(motDePasse, u.MotDePasseHash)
}

// Fonctions utilitaires privées
func validerEmail(email string) error {
    // Implementation de validation
    return nil
}

func validerMotDePasse(motDePasse string) error {
    if len(motDePasse) < TailleMotDePasseMin {
        return ErrMotDePasseTropCourt
    }
    return nil
}

func hasherMotDePasse(motDePasse string) string {
    // Implementation du hashage
    return ""
}

func verifierHash(motDePasse, hash string) bool {
    // Implementation de vérification
    return true
}
```

### Outils pour vérifier les conventions

#### 1. go fmt

**Formate automatiquement votre code selon les standards Go :**

```bash
# Formater un fichier
go fmt main.go

# Formater tout le projet
go fmt ./...

# Voir les changements sans les appliquer
gofmt -d main.go
```

#### 2. go vet

**Détecte les problèmes courants :**

```bash
# Vérifier un fichier
go vet main.go

# Vérifier tout le projet
go vet ./...
```

#### 3. golint (outil externe)

**Vérifie les conventions de style :**

```bash
# Installer golint
go install golang.org/x/lint/golint@latest

# Utiliser golint
golint main.go
golint ./...
```

### Configuration VS Code

**Automatiser le formatage dans VS Code :**

```json
{
    "go.formatTool": "gofmt",
    "editor.formatOnSave": true,
    "go.lintOnSave": "package",
    "go.vetOnSave": "package",
    "go.buildOnSave": "off",
    "go.useLanguageServer": true
}
```

## Erreurs courantes de nommage

### 1. Noms trop courts ou cryptiques

```go
// ❌ Trop cryptique
var u *User
var cfg *Config
var mgr *Manager

// ✅ Plus clair (selon le contexte)
var utilisateurConnecte *User
var configurationServeur *Config
var gestionnaireEmail *Manager
```

### 2. Noms trop longs

```go
// ❌ Trop verbeux
var gestionnaireDeConnexionBaseDeDonneesUtilisateur *DatabaseManager

// ✅ Plus concis
var gestionnaireBD *DatabaseManager
var dbManager *DatabaseManager
```

### 3. Incohérence dans le projet

```go
// ❌ Incohérent
func creerUtilisateur() {}
func deleteUser() {}        // Mélange français/anglais
func updateUtilisateur() {} // Incohérent

// ✅ Cohérent
func creerUtilisateur() {}
func supprimerUtilisateur() {}
func mettreAJourUtilisateur() {}
```

### 4. Mauvaise visibilité

```go
// ❌ Fonction utilitaire exportée inutilement
func FormatString(s string) string {}  // Devrait être privée

// ✅ Visibilité appropriée
func formatString(s string) string {}  // Privée

// Fonction publique de l'API
func FormatUserName(nom, prenom string) string {}
```

## Exemple complet : Application TODO

```go
// Package todo fournit les fonctionnalités de gestion de tâches.
//
// Cette bibliothèque permet de créer, modifier et organiser des tâches
// avec support des priorités et des échéances.
//
// Exemple d'utilisation :
//     gestionnaire := todo.NouveauGestionnaire()
//     tache, err := gestionnaire.CreerTache("Acheter du lait", todo.PrioriteHaute)
//     if err != nil {
//         log.Fatal(err)
//     }
//     gestionnaire.MarquerComplete(tache.ID)
package todo

import (
    "errors"
    "time"
)

// Constantes pour les priorités
const (
    PrioriteBasse  = 1
    PrioriteMoyenne = 2
    PrioriteHaute  = 3
)

// Erreurs prédéfinies
var (
    ErrTacheInexistante = errors.New("tâche inexistante")
    ErrTitreVide       = errors.New("le titre ne peut pas être vide")
    ErrPrioriteInvalide = errors.New("priorité invalide")
)

// Tache représente une tâche à accomplir
type Tache struct {
    ID          int       `json:"id"`
    Titre       string    `json:"titre"`
    Description string    `json:"description,omitempty"`
    Priorite    int       `json:"priorite"`
    EstComplete bool      `json:"est_complete"`
    DateCreation time.Time `json:"date_creation"`
    DateEcheance *time.Time `json:"date_echeance,omitempty"`
}

// GestionnaireTaches gère une collection de tâches
type GestionnaireTaches struct {
    taches     map[int]*Tache
    prochainID int
}

// NouveauGestionnaire crée une nouvelle instance de gestionnaire de tâches
func NouveauGestionnaire() *GestionnaireTaches {
    return &GestionnaireTaches{
        taches:     make(map[int]*Tache),
        prochainID: 1,
    }
}

// CreerTache ajoute une nouvelle tâche avec le titre et la priorité spécifiés.
// Retourne la tâche créée ou une erreur si les paramètres sont invalides.
func (g *GestionnaireTaches) CreerTache(titre string, priorite int) (*Tache, error) {
    // Validation des paramètres
    if titre == "" {
        return nil, ErrTitreVide
    }

    if priorite < PrioriteBasse || priorite > PrioriteHaute {
        return nil, ErrPrioriteInvalide
    }

    // Création de la tâche
    tache := &Tache{
        ID:           g.prochainID,
        Titre:        titre,
        Priorite:     priorite,
        EstComplete:  false,
        DateCreation: time.Now(),
    }

    // Ajout au gestionnaire
    g.taches[tache.ID] = tache
    g.prochainID++

    return tache, nil
}

// ObtenirTache retourne la tâche avec l'ID spécifié.
// Retourne nil et ErrTacheInexistante si la tâche n'existe pas.
func (g *GestionnaireTaches) ObtenirTache(id int) (*Tache, error) {
    tache, existe := g.taches[id]
    if !existe {
        return nil, ErrTacheInexistante
    }
    return tache, nil
}

// ListerTaches retourne toutes les tâches, optionnellement filtrées par statut.
// Si seulementEnCours est true, ne retourne que les tâches non complétées.
func (g *GestionnaireTaches) ListerTaches(seulementEnCours bool) []*Tache {
    var resultat []*Tache

    for _, tache := range g.taches {
        if seulementEnCours && tache.EstComplete {
            continue // Ignorer les tâches complétées
        }
        resultat = append(resultat, tache)
    }

    return resultat
}

// MarquerComplete marque une tâche comme terminée.
// Retourne une erreur si la tâche n'existe pas.
func (g *GestionnaireTaches) MarquerComplete(id int) error {
    tache, existe := g.taches[id]
    if !existe {
        return ErrTacheInexistante
    }

    tache.EstComplete = true
    return nil
}

// DefinirEcheance définit une date d'échéance pour une tâche.
// Retourne une erreur si la tâche n'existe pas.
func (g *GestionnaireTaches) DefinirEcheance(id int, echeance time.Time) error {
    tache, existe := g.taches[id]
    if !existe {
        return ErrTacheInexistante
    }

    tache.DateEcheance = &echeance
    return nil
}

// EstEnRetard vérifie si la tâche a dépassé sa date d'échéance.
// Retourne false si aucune échéance n'est définie.
func (t *Tache) EstEnRetard() bool {
    if t.DateEcheance == nil || t.EstComplete {
        return false
    }
    return time.Now().After(*t.DateEcheance)
}

// ObtenirPrioriteTexte retourne la priorité sous forme de texte lisible.
func (t *Tache) ObtenirPrioriteTexte() string {
    switch t.Priorite {
    case PrioriteBasse:
        return "Basse"
    case PrioriteMoyenne:
        return "Moyenne"
    case PrioriteHaute:
        return "Haute"
    default:
        return "Inconnue"
    }
}

// CompterTaches retourne le nombre total de tâches et le nombre de tâches complétées.
func (g *GestionnaireTaches) CompterTaches() (total int, completees int) {
    total = len(g.taches)

    for _, tache := range g.taches {
        if tache.EstComplete {
            completees++
        }
    }

    return total, completees
}
```

## Exercices pratiques

### Exercice 1 : Améliorer le code

Refactorisez ce code en appliquant les bonnes pratiques :

```go
package main

import "fmt"

var u string = "Alice"
var a int = 25
var s float64 = 2500.50

func f1() {
    fmt.Println(u, a, s)
}

func f2(x float64, y float64) float64 {
    return x * y
}
```

### Exercice 2 : Créer une bibliothèque

Créez un package `calculatrice` avec :
- Fonctions pour les 4 opérations de base
- Types pour représenter des fractions
- Documentation complète
- Gestion d'erreurs appropriée

### Exercice 3 : Commenter du code existant

Ajoutez des commentaires appropriés à ce code :

```go
package main

import (
    "fmt"
    "math"
)

func main() {
    r := 5.0
    result := math.Pi * r * r
    fmt.Printf("%.2f\n", result)
}
```

### Solutions

**Exercice 1 :**
```go
// Package main démonte l'utilisation des conventions de nommage Go.
package main

import "fmt"

// Informations de l'utilisateur
var nomUtilisateur string = "Alice"
var ageUtilisateur int = 25
var salaireAnnuel float64 = 2500.50

// AfficherProfilUtilisateur affiche les informations de l'utilisateur connecté
func AfficherProfilUtilisateur() {
    fmt.Printf("Utilisateur: %s, Age: %d ans, Salaire: %.2f€\n",
               nomUtilisateur, ageUtilisateur, salaireAnnuel)
}

// CalculerSalaireMensuel calcule le salaire mensuel à partir du salaire annuel
func CalculerSalaireMensuel(salaireAnnuel float64, nbMois float64) float64 {
    return salaireAnnuel / nbMois
}

func main() {
    AfficherProfilUtilisateur()

    salaireMensuel := CalculerSalaireMensuel(salaireAnnuel, 12)
    fmt.Printf("Salaire mensuel: %.2f€\n", salaireMensuel)
}
```

**Exercice 2 :**
```go
// Package calculatrice fournit des fonctions mathématiques de base
// et des opérations sur les fractions.
//
// Exemple d'utilisation :
//     resultat, err := calculatrice.Diviser(10, 3)
//     if err != nil {
//         log.Fatal(err)
//     }
//     fmt.Printf("Résultat: %.2f\n", resultat)
package calculatrice

import "errors"

// Erreurs prédéfinies
var (
    ErrDivisionParZero = errors.New("division par zéro impossible")
    ErrDenominateurZero = errors.New("le dénominateur ne peut pas être zéro")
)

// Fraction représente une fraction mathématique
type Fraction struct {
    Numerateur   int `json:"numerateur"`
    Denominateur int `json:"denominateur"`
}

// NouvelleFraction crée une nouvelle fraction en vérifiant que le dénominateur n'est pas zéro.
// Retourne une erreur si le dénominateur est zéro.
func NouvelleFraction(numerateur, denominateur int) (*Fraction, error) {
    if denominateur == 0 {
        return nil, ErrDenominateurZero
    }

    return &Fraction{
        Numerateur:   numerateur,
        Denominateur: denominateur,
    }, nil
}

// Additionner retourne la somme de deux nombres.
func Additionner(a, b float64) float64 {
    return a + b
}

// Soustraire retourne la différence entre deux nombres.
func Soustraire(a, b float64) float64 {
    return a - b
}

// Multiplier retourne le produit de deux nombres.
func Multiplier(a, b float64) float64 {
    return a * b
}

// Diviser retourne le quotient de deux nombres.
// Retourne une erreur si le diviseur est zéro.
func Diviser(dividende, diviseur float64) (float64, error) {
    if diviseur == 0 {
        return 0, ErrDivisionParZero
    }
    return dividende / diviseur, nil
}

// Additionner additionne deux fractions et retourne le résultat sous forme de nouvelle fraction.
// Le résultat n'est pas automatiquement simplifié.
func (f *Fraction) Additionner(autre *Fraction) *Fraction {
    // Pour additionner a/b + c/d = (a*d + c*b) / (b*d)
    nouveauNumerateur := f.Numerateur*autre.Denominateur + autre.Numerateur*f.Denominateur
    nouveauDenominateur := f.Denominateur * autre.Denominateur

    // Pas de vérification d'erreur car on sait que les dénominateurs sont non-zéro
    resultat, _ := NouvelleFraction(nouveauNumerateur, nouveauDenominateur)
    return resultat
}

// Multiplier multiplie deux fractions et retourne le résultat sous forme de nouvelle fraction.
func (f *Fraction) Multiplier(autre *Fraction) *Fraction {
    // Pour multiplier a/b * c/d = (a*c) / (b*d)
    nouveauNumerateur := f.Numerateur * autre.Numerateur
    nouveauDenominateur := f.Denominateur * autre.Denominateur

    resultat, _ := NouvelleFraction(nouveauNumerateur, nouveauDenominateur)
    return resultat
}

// VersDecimal convertit la fraction en nombre décimal.
func (f *Fraction) VersDecimal() float64 {
    return float64(f.Numerateur) / float64(f.Denominateur)
}

// Simplifier retourne une nouvelle fraction simplifiée en divisant par le PGCD.
func (f *Fraction) Simplifier() *Fraction {
    pgcd := calculerPGCD(abs(f.Numerateur), abs(f.Denominateur))

    resultat, _ := NouvelleFraction(f.Numerateur/pgcd, f.Denominateur/pgcd)
    return resultat
}

// Fonctions utilitaires privées

// calculerPGCD calcule le Plus Grand Commun Diviseur de deux entiers
// en utilisant l'algorithme d'Euclide.
func calculerPGCD(a, b int) int {
    for b != 0 {
        a, b = b, a%b
    }
    return a
}

// abs retourne la valeur absolue d'un entier.
func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}
```

**Exercice 3 :**
```go
// Package main calcule et affiche la surface d'un cercle.
package main

import (
    "fmt"
    "math"
)

func main() {
    // Rayon du cercle en unités (mètres, centimètres, etc.)
    rayon := 5.0

    // Calcul de la surface en utilisant la formule π × r²
    surface := math.Pi * rayon * rayon

    // Affichage du résultat avec 2 décimales
    fmt.Printf("Surface du cercle (rayon %.1f): %.2f unités²\n", rayon, surface)
}
```

## Outils et automatisation

### Intégration dans l'éditeur

#### Configuration VS Code avancée

```json
{
    "go.formatTool": "gofmt",
    "go.lintTool": "golint",
    "go.vetOnSave": "package",
    "go.lintOnSave": "package",
    "go.buildOnSave": "off",
    "go.useLanguageServer": true,
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.organizeImports": true
    },
    "go.docsTool": "godoc",
    "go.generateTestsFlags": [
        "-exported"
    ]
}
```

#### Raccourcis utiles

| Raccourci | Action |
|-----------|--------|
| `Ctrl+Shift+P` → "Go: Add Tags" | Ajouter des tags JSON/XML |
| `Ctrl+Shift+P` → "Go: Generate Tests" | Générer des tests |
| `F12` | Aller à la définition |
| `Shift+F12` | Trouver toutes les références |
| `Ctrl+K Ctrl+D` | Voir la documentation |

### Scripts de vérification

#### Makefile pour automatiser les vérifications

```makefile
# Makefile pour projet Go
.PHONY: fmt vet lint test doc clean

# Formater le code
fmt:
	go fmt ./...

# Vérifier avec go vet
vet:
	go vet ./...

# Linter (nécessite golint)
lint:
	golint ./...

# Exécuter tous les tests
test:
	go test -v ./...

# Générer la documentation
doc:
	godoc -http=:6060

# Vérification complète avant commit
check: fmt vet lint test
	@echo "Toutes les vérifications sont passées !"

# Nettoyer les fichiers générés
clean:
	go clean ./...
	rm -f *.out *.test
```

**Utilisation :**
```bash
make fmt    # Formater le code
make check  # Vérification complète
make doc    # Lancer la documentation
```

#### Script de pre-commit

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "Vérification du code avant commit..."

# Formater le code
echo "Formatage du code..."
go fmt ./...

# Vérifier avec go vet
echo "Vérification avec go vet..."
if ! go vet ./...; then
    echo "❌ go vet a détecté des problèmes"
    exit 1
fi

# Linter
echo "Vérification du style..."
if command -v golint &> /dev/null; then
    if ! golint ./...; then
        echo "⚠️  golint a détecté des problèmes de style"
        # Ne pas bloquer le commit pour le style
    fi
fi

# Tests
echo "Exécution des tests..."
if ! go test ./...; then
    echo "❌ Des tests échouent"
    exit 1
fi

echo "✅ Toutes les vérifications sont passées !"
```

### Documentation automatique

#### Générer la documentation

```bash
# Documentation locale
godoc -http=:6060

# Générer des fichiers HTML
godoc -html . > documentation.html

# Documentation d'un package spécifique
go doc package/sous-package
```

#### Exemple de documentation complète

```go
// Package weather fournit des fonctionnalités de récupération et d'analyse
// de données météorologiques.
//
// Ce package permet d'obtenir des informations météo depuis différentes sources
// et de les analyser pour fournir des prédictions et des alertes.
//
// Utilisation basique :
//
//     client := weather.NewClient("votre-api-key")
//     meteo, err := client.ObtenirMeteoActuelle("Paris")
//     if err != nil {
//         log.Fatal(err)
//     }
//     fmt.Printf("Température: %.1f°C\n", meteo.Temperature)
//
// Utilisation avancée avec prédictions :
//
//     predictions, err := client.ObtenirPredictions("Lyon", 7) // 7 jours
//     if err != nil {
//         log.Fatal(err)
//     }
//
//     for _, pred := range predictions {
//         fmt.Printf("%s: %s, %.1f°C\n",
//                    pred.Date.Format("2006-01-02"),
//                    pred.Description,
//                    pred.TemperatureMoyenne)
//     }
//
// Configuration :
//
// Le package peut être configuré via des variables d'environnement :
//   - WEATHER_API_KEY : Clé API pour le service météo
//   - WEATHER_TIMEOUT : Timeout des requêtes en secondes (défaut: 30)
//   - WEATHER_CACHE_TTL : Durée de vie du cache en minutes (défaut: 10)
package weather

import (
    "context"
    "errors"
    "time"
)

// Constantes de configuration par défaut
const (
    // TimeoutParDefaut est le timeout par défaut pour les requêtes HTTP
    TimeoutParDefaut = 30 * time.Second

    // CacheTTLParDefaut est la durée de vie par défaut du cache
    CacheTTLParDefaut = 10 * time.Minute

    // URLAPIParDefaut est l'URL de base de l'API météo
    URLAPIParDefaut = "https://api.meteo.fr/v1"
)

// Erreurs prédéfinies du package
var (
    // ErrCleAPIManquante indique qu'aucune clé API n'a été fournie
    ErrCleAPIManquante = errors.New("clé API manquante")

    // ErrVilleInexistante indique que la ville demandée n'a pas été trouvée
    ErrVilleInexistante = errors.New("ville inexistante")

    // ErrServiceIndisponible indique que le service météo est temporairement indisponible
    ErrServiceIndisponible = errors.New("service météo indisponible")

    // ErrDonneesInvalides indique que les données reçues de l'API sont invalides
    ErrDonneesInvalides = errors.New("données météo invalides")
)

// ConditionMeteo représente les conditions météorologiques possibles
type ConditionMeteo int

const (
    // ConditionInconnue indique que la condition météo n'est pas déterminée
    ConditionInconnue ConditionMeteo = iota

    // Ensoleille indique un temps ensoleillé
    Ensoleille

    // NuageuxPartiel indique un temps partiellement nuageux
    NuageuxPartiel

    // Nuageux indique un temps nuageux
    Nuageux

    // Pluvieux indique de la pluie
    Pluvieux

    // Orageux indique des orages
    Orageux

    // Neigeux indique de la neige
    Neigeux
)

// String retourne la représentation textuelle de la condition météo
func (c ConditionMeteo) String() string {
    switch c {
    case Ensoleille:
        return "Ensoleillé"
    case NuageuxPartiel:
        return "Partiellement nuageux"
    case Nuageux:
        return "Nuageux"
    case Pluvieux:
        return "Pluvieux"
    case Orageux:
        return "Orageux"
    case Neigeux:
        return "Neigeux"
    default:
        return "Inconnu"
    }
}

// MeteoActuelle représente les conditions météorologiques actuelles pour une ville
type MeteoActuelle struct {
    // Ville est le nom de la ville
    Ville string `json:"ville"`

    // Pays est le code pays (ISO 3166-1 alpha-2)
    Pays string `json:"pays"`

    // Temperature est la température actuelle en degrés Celsius
    Temperature float64 `json:"temperature"`

    // TemperatureRessentie est la température ressentie en degrés Celsius
    TemperatureRessentie float64 `json:"temperature_ressentie"`

    // Humidite est le pourcentage d'humidité (0-100)
    Humidite int `json:"humidite"`

    // Pression est la pression atmosphérique en hPa
    Pression float64 `json:"pression"`

    // VitesseVent est la vitesse du vent en km/h
    VitesseVent float64 `json:"vitesse_vent"`

    // DirectionVent est la direction du vent en degrés (0-360)
    DirectionVent int `json:"direction_vent"`

    // Condition décrit les conditions météorologiques actuelles
    Condition ConditionMeteo `json:"condition"`

    // Description est une description textuelle des conditions
    Description string `json:"description"`

    // DateMiseAJour est la date de dernière mise à jour des données
    DateMiseAJour time.Time `json:"date_mise_a_jour"`
}

// EstChaud retourne true si la température est considérée comme chaude (>= 25°C)
func (m *MeteoActuelle) EstChaud() bool {
    return m.Temperature >= 25.0
}

// EstFroid retourne true si la température est considérée comme froide (<= 5°C)
func (m *MeteoActuelle) EstFroid() bool {
    return m.Temperature <= 5.0
}

// EstVenteux retourne true si le vent est fort (>= 50 km/h)
func (m *MeteoActuelle) EstVenteux() bool {
    return m.VitesseVent >= 50.0
}

// Configuration contient les paramètres de configuration du client météo
type Configuration struct {
    // CleAPI est la clé d'authentification pour l'API météo
    CleAPI string

    // URLAPI est l'URL de base de l'API météo
    URLAPI string

    // Timeout est le timeout pour les requêtes HTTP
    Timeout time.Duration

    // CacheTTL est la durée de vie des données en cache
    CacheTTL time.Duration

    // ActiverCache indique si le cache doit être utilisé
    ActiverCache bool
}

// ConfigurationParDefaut retourne une configuration avec les valeurs par défaut
func ConfigurationParDefaut() *Configuration {
    return &Configuration{
        URLAPI:       URLAPIParDefaut,
        Timeout:      TimeoutParDefaut,
        CacheTTL:     CacheTTLParDefaut,
        ActiverCache: true,
    }
}

// Client représente un client pour l'API météo
type Client struct {
    config *Configuration
    cache  map[string]*entreeCache  // Cache simple en mémoire
}

// entreeCache représente une entrée du cache avec sa date d'expiration
type entreeCache struct {
    donnees    *MeteoActuelle
    expiration time.Time
}

// NouveauClient crée un nouveau client météo avec la clé API fournie.
// Utilise la configuration par défaut.
func NouveauClient(cleAPI string) (*Client, error) {
    if cleAPI == "" {
        return nil, ErrCleAPIManquante
    }

    config := ConfigurationParDefaut()
    config.CleAPI = cleAPI

    return NouveauClientAvecConfig(config)
}

// NouveauClientAvecConfig crée un nouveau client météo avec une configuration personnalisée.
func NouveauClientAvecConfig(config *Configuration) (*Client, error) {
    if config.CleAPI == "" {
        return nil, ErrCleAPIManquante
    }

    return &Client{
        config: config,
        cache:  make(map[string]*entreeCache),
    }, nil
}

// ObtenirMeteoActuelle récupère les conditions météorologiques actuelles pour la ville spécifiée.
// Utilise le cache si disponible et non expiré.
//
// Paramètres :
//   ville : nom de la ville (ex: "Paris", "Lyon", "Marseille")
//
// Retourne :
//   *MeteoActuelle : les conditions météorologiques actuelles
//   error : erreur en cas de problème de récupération
//
// Erreurs possibles :
//   - ErrVilleInexistante si la ville n'est pas trouvée
//   - ErrServiceIndisponible si l'API est indisponible
//   - ErrDonneesInvalides si les données reçues sont corrompues
func (c *Client) ObtenirMeteoActuelle(ville string) (*MeteoActuelle, error) {
    return c.ObtenirMeteoActuelleAvecContexte(context.Background(), ville)
}

// ObtenirMeteoActuelleAvecContexte récupère les conditions météorologiques actuelles
// avec support d'annulation via le contexte.
func (c *Client) ObtenirMeteoActuelleAvecContexte(ctx context.Context, ville string) (*MeteoActuelle, error) {
    // Vérifier le cache en premier
    if c.config.ActiverCache {
        if meteo := c.obtenirDuCache(ville); meteo != nil {
            return meteo, nil
        }
    }

    // Récupérer depuis l'API
    meteo, err := c.recupererDepuisAPI(ctx, ville)
    if err != nil {
        return nil, err
    }

    // Mettre en cache
    if c.config.ActiverCache {
        c.mettreEnCache(ville, meteo)
    }

    return meteo, nil
}

// ViderCache supprime toutes les entrées du cache
func (c *Client) ViderCache() {
    c.cache = make(map[string]*entreeCache)
}

// StatistiquesCache retourne des statistiques sur l'utilisation du cache
func (c *Client) StatistiquesCache() (entrees int, expirees int) {
    maintenant := time.Now()

    for _, entree := range c.cache {
        entrees++
        if maintenant.After(entree.expiration) {
            expirees++
        }
    }

    return entrees, expirees
}

// Fonctions privées pour l'implémentation

func (c *Client) obtenirDuCache(ville string) *MeteoActuelle {
    entree, existe := c.cache[ville]
    if !existe {
        return nil
    }

    // Vérifier l'expiration
    if time.Now().After(entree.expiration) {
        delete(c.cache, ville)
        return nil
    }

    return entree.donnees
}

func (c *Client) mettreEnCache(ville string, meteo *MeteoActuelle) {
    c.cache[ville] = &entreeCache{
        donnees:    meteo,
        expiration: time.Now().Add(c.config.CacheTTL),
    }
}

func (c *Client) recupererDepuisAPI(ctx context.Context, ville string) (*MeteoActuelle, error) {
    // Implémentation de la récupération depuis l'API
    // Cette fonction ferait un appel HTTP réel en production

    // Simulation pour l'exemple
    return &MeteoActuelle{
        Ville:                ville,
        Pays:                 "FR",
        Temperature:          22.5,
        TemperatureRessentie: 24.0,
        Humidite:            65,
        Pression:            1013.25,
        VitesseVent:         15.0,
        DirectionVent:       230,
        Condition:           Ensoleille,
        Description:         "Ensoleillé avec quelques nuages",
        DateMiseAJour:       time.Now(),
    }, nil
}
```

## Récapitulatif final

**Ce que vous avez appris :**
- ✅ Types de commentaires en Go (// vs /* */)
- ✅ Quand et comment commenter efficacement
- ✅ Documentation automatique avec godoc
- ✅ Conventions de nommage Go (camelCase, visibilité)
- ✅ Outils d'automatisation (go fmt, go vet, golint)
- ✅ Intégration dans l'éditeur et workflows

**Règles d'or des commentaires :**
1. **Expliquer le POURQUOI**, pas le QUOI
2. **Documenter les fonctions publiques** obligatoirement
3. **Éviter les commentaires évidents** ou redondants
4. **Maintenir les commentaires à jour** avec le code
5. **Utiliser godoc** pour la documentation officielle

**Règles d'or du nommage :**
1. **CamelCase** toujours (jamais snake_case)
2. **Majuscule = public**, minuscule = privé
3. **Noms descriptifs** mais pas trop longs
4. **Cohérence** dans tout le projet
5. **Acronymes gardent leur casse** (HTTP, URL, ID)

**Outils essentiels :**
```bash
go fmt ./...     # Formater le code
go vet ./...     # Détecter les problèmes
golint ./...     # Vérifier le style
godoc -http=:6060 # Documentation locale
```

**Workflow recommandé :**
1. Écrire le code
2. Ajouter les commentaires godoc
3. Formater avec `go fmt`
4. Vérifier avec `go vet`
5. Contrôler le style avec `golint`
6. Tester la documentation avec `godoc`

---

*Félicitations ! Vous maîtrisez maintenant les bases de la syntaxe Go. Votre code est maintenant lisible, bien structuré et suit les conventions de la communauté. Dans la prochaine section, nous aborderons les structures de contrôle qui permettront de rendre vos programmes interactifs et dynamiques !*

⏭️
