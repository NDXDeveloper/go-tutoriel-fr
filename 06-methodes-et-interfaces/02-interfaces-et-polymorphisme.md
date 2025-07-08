🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6-2 : Interfaces et polymorphisme

## Introduction

En Go, une **interface** est un type qui définit un ensemble de signatures de méthodes. Contrairement aux langages orientés objet traditionnels, Go utilise une approche différente : les interfaces sont **implémentées implicitement**. Cela signifie qu'un type implémente automatiquement une interface s'il possède toutes les méthodes définies par cette interface.

Le **polymorphisme** permet à différents types de se comporter de la même manière à travers une interface commune.

## Qu'est-ce qu'une interface ?

Une interface est un "contrat" qui spécifie quelles méthodes un type doit avoir. Voici la syntaxe :

```go
type NomInterface interface {
    Methode1() TypeRetour
    Methode2(param Type) TypeRetour
    // ... autres méthodes
}
```

## Exemple simple : Formes géométriques

Commençons par un exemple concret avec des formes géométriques :

```go
package main

import (
    "fmt"
    "math"
)

// Définition de l'interface Forme
type Forme interface {
    Aire() float64
    Perimetre() float64
}

// Structure Rectangle
type Rectangle struct {
    largeur  float64
    hauteur  float64
}

// Méthodes pour Rectangle
func (r Rectangle) Aire() float64 {
    return r.largeur * r.hauteur
}

func (r Rectangle) Perimetre() float64 {
    return 2 * (r.largeur + r.hauteur)
}

// Structure Cercle
type Cercle struct {
    rayon float64
}

// Méthodes pour Cercle
func (c Cercle) Aire() float64 {
    return math.Pi * c.rayon * c.rayon
}

func (c Cercle) Perimetre() float64 {
    return 2 * math.Pi * c.rayon
}

// Fonction qui accepte n'importe quelle forme
func AfficherInfosForme(f Forme) {
    fmt.Printf("Aire: %.2f\n", f.Aire())
    fmt.Printf("Périmètre: %.2f\n", f.Perimetre())
}

func main() {
    // Création des formes
    rect := Rectangle{largeur: 5.0, hauteur: 3.0}
    cercle := Cercle{rayon: 2.5}

    // Polymorphisme : même fonction, types différents
    fmt.Println("=== Rectangle ===")
    AfficherInfosForme(rect)

    fmt.Println("\n=== Cercle ===")
    AfficherInfosForme(cercle)
}
```

**Sortie :**
```
=== Rectangle ===
Aire: 15.00
Périmètre: 16.00

=== Cercle ===
Aire: 19.63
Périmètre: 15.71
```

## Implémentation implicite

La beauté des interfaces Go réside dans leur implémentation implicite. Vous n'avez pas besoin de déclarer explicitement qu'un type implémente une interface :

```go
// Pas besoin de dire "Rectangle implements Forme"
// Go le détecte automatiquement car Rectangle a les méthodes Aire() et Perimetre()
```

## Exemple pratique : Système de notification

```go
package main

import "fmt"

// Interface pour les notifications
type Notificateur interface {
    Envoyer(message string) error
}

// Implémentation Email
type Email struct {
    destinataire string
}

func (e Email) Envoyer(message string) error {
    fmt.Printf("📧 Email envoyé à %s: %s\n", e.destinataire, message)
    return nil
}

// Implémentation SMS
type SMS struct {
    numero string
}

func (s SMS) Envoyer(message string) error {
    fmt.Printf("📱 SMS envoyé au %s: %s\n", s.numero, message)
    return nil
}

// Implémentation Push
type NotificationPush struct {
    deviceId string
}

func (n NotificationPush) Envoyer(message string) error {
    fmt.Printf("🔔 Push envoyée à %s: %s\n", n.deviceId, message)
    return nil
}

// Service de notification qui accepte n'importe quel notificateur
type ServiceNotification struct {
    notificateurs []Notificateur
}

func (s *ServiceNotification) AjouterNotificateur(n Notificateur) {
    s.notificateurs = append(s.notificateurs, n)
}

func (s *ServiceNotification) EnvoyerATous(message string) {
    for _, notificateur := range s.notificateurs {
        notificateur.Envoyer(message)
    }
}

func main() {
    // Création du service
    service := &ServiceNotification{}

    // Ajout de différents types de notificateurs
    service.AjouterNotificateur(Email{destinataire: "alice@example.com"})
    service.AjouterNotificateur(SMS{numero: "0123456789"})
    service.AjouterNotificateur(NotificationPush{deviceId: "device-123"})

    // Envoi d'une notification à tous
    service.EnvoyerATous("Votre commande a été expédiée!")
}
```

## Slice d'interfaces

Vous pouvez créer des slices d'interfaces pour traiter différents types de manière uniforme :

```go
package main

import "fmt"

// Interface pour les animaux
type Animal interface {
    Cri() string
    Nom() string
}

// Implémentations
type Chien struct {
    nom string
}

func (c Chien) Cri() string {
    return "Woof!"
}

func (c Chien) Nom() string {
    return c.nom
}

type Chat struct {
    nom string
}

func (c Chat) Cri() string {
    return "Miaou!"
}

func (c Chat) Nom() string {
    return c.nom
}

type Vache struct {
    nom string
}

func (v Vache) Cri() string {
    return "Meuh!"
}

func (v Vache) Nom() string {
    return v.nom
}

func main() {
    // Slice d'interfaces
    animaux := []Animal{
        Chien{nom: "Rex"},
        Chat{nom: "Whiskers"},
        Vache{nom: "Marguerite"},
    }

    // Polymorphisme : même code, comportements différents
    fmt.Println("=== Concert d'animaux ===")
    for _, animal := range animaux {
        fmt.Printf("%s dit: %s\n", animal.Nom(), animal.Cri())
    }
}
```

## Interface vide

L'interface vide `interface{}` peut contenir n'importe quel type :

```go
package main

import "fmt"

// Fonction qui accepte n'importe quel type
func AfficherType(v interface{}) {
    fmt.Printf("Valeur: %v, Type: %T\n", v, v)
}

func main() {
    AfficherType(42)
    AfficherType("Hello")
    AfficherType(3.14)
    AfficherType([]int{1, 2, 3})
}
```

**Note :** En Go 1.18+, on utilise plutôt `any` qui est un alias pour `interface{}`.

## Composition d'interfaces

Vous pouvez composer des interfaces à partir d'autres interfaces :

```go
package main

import "fmt"

// Interfaces de base
type Lecteur interface {
    Lire() string
}

type Ecrivain interface {
    Ecrire(data string) error
}

// Interface composée
type LecteurEcrivain interface {
    Lecteur
    Ecrivain
}

// Implémentation
type Fichier struct {
    nom     string
    contenu string
}

func (f *Fichier) Lire() string {
    return f.contenu
}

func (f *Fichier) Ecrire(data string) error {
    f.contenu = data
    return nil
}

func main() {
    fichier := &Fichier{nom: "test.txt"}

    // Utilisation comme LecteurEcrivain
    var rw LecteurEcrivain = fichier

    rw.Ecrire("Contenu du fichier")
    contenu := rw.Lire()
    fmt.Println("Contenu lu:", contenu)
}
```

## Exemple complet : Système de paiement

```go
package main

import (
    "fmt"
    "errors"
)

// Interface pour les processeurs de paiement
type ProcesseurPaiement interface {
    Payer(montant float64) error
    Verifier() bool
}

// Implémentation Carte de crédit
type CarteCredit struct {
    numero string
    solde  float64
}

func (c *CarteCredit) Payer(montant float64) error {
    if !c.Verifier() {
        return errors.New("carte invalide")
    }
    if c.solde < montant {
        return errors.New("solde insuffisant")
    }
    c.solde -= montant
    fmt.Printf("💳 Paiement de %.2f€ effectué par carte (solde restant: %.2f€)\n", montant, c.solde)
    return nil
}

func (c *CarteCredit) Verifier() bool {
    return len(c.numero) == 16 && c.solde >= 0
}

// Implémentation PayPal
type PayPal struct {
    email string
    solde float64
}

func (p *PayPal) Payer(montant float64) error {
    if !p.Verifier() {
        return errors.New("compte PayPal invalide")
    }
    if p.solde < montant {
        return errors.New("solde PayPal insuffisant")
    }
    p.solde -= montant
    fmt.Printf("🅿️ Paiement de %.2f€ effectué via PayPal (solde restant: %.2f€)\n", montant, p.solde)
    return nil
}

func (p *PayPal) Verifier() bool {
    return len(p.email) > 0 && p.solde >= 0
}

// Implémentation Cryptomonnaie
type Crypto struct {
    portefeuille string
    solde        float64
}

func (c *Crypto) Payer(montant float64) error {
    if !c.Verifier() {
        return errors.New("portefeuille crypto invalide")
    }
    if c.solde < montant {
        return errors.New("solde crypto insuffisant")
    }
    c.solde -= montant
    fmt.Printf("₿ Paiement de %.2f€ effectué en crypto (solde restant: %.2f€)\n", montant, c.solde)
    return nil
}

func (c *Crypto) Verifier() bool {
    return len(c.portefeuille) > 0 && c.solde >= 0
}

// Système de commande
type Commande struct {
    id       int
    montant  float64
    produits []string
}

// Processeur de commande
type ProcesseurCommande struct{}

func (pc *ProcesseurCommande) TraiterCommande(commande *Commande, processeur ProcesseurPaiement) error {
    fmt.Printf("\n=== Traitement de la commande #%d ===\n", commande.id)
    fmt.Printf("Produits: %v\n", commande.produits)
    fmt.Printf("Montant: %.2f€\n", commande.montant)

    if !processeur.Verifier() {
        return errors.New("méthode de paiement invalide")
    }

    err := processeur.Payer(commande.montant)
    if err != nil {
        return fmt.Errorf("erreur de paiement: %w", err)
    }

    fmt.Println("✅ Commande traitée avec succès!")
    return nil
}

func main() {
    // Création des méthodes de paiement
    carte := &CarteCredit{numero: "1234567890123456", solde: 1000.0}
    paypal := &PayPal{email: "user@example.com", solde: 500.0}
    crypto := &Crypto{portefeuille: "1A2B3C4D5E6F", solde: 200.0}

    // Création des commandes
    commande1 := &Commande{id: 1, montant: 150.0, produits: []string{"Laptop", "Souris"}}
    commande2 := &Commande{id: 2, montant: 75.0, produits: []string{"Clavier", "Écran"}}
    commande3 := &Commande{id: 3, montant: 120.0, produits: []string{"Tablette"}}

    // Processeur de commande
    processeur := &ProcesseurCommande{}

    // Traitement avec différentes méthodes de paiement
    processeur.TraiterCommande(commande1, carte)
    processeur.TraiterCommande(commande2, paypal)
    processeur.TraiterCommande(commande3, crypto)

    // Exemple d'erreur
    commandeErreur := &Commande{id: 4, montant: 2000.0, produits: []string{"Serveur"}}
    err := processeur.TraiterCommande(commandeErreur, carte)
    if err != nil {
        fmt.Printf("❌ Erreur: %v\n", err)
    }
}
```

## Avantages des interfaces

1. **Flexibilité** : Permet de changer d'implémentation sans modifier le code client
2. **Testabilité** : Facilite la création de mocks pour les tests
3. **Découplage** : Réduit les dépendances entre les composants
4. **Extensibilité** : Permet d'ajouter de nouveaux types facilement

## Bonnes pratiques

1. **Interfaces petites** : Préférez des interfaces avec peu de méthodes (souvent une seule)
2. **Définir côté consommateur** : Définissez les interfaces là où elles sont utilisées, pas là où elles sont implémentées
3. **Noms expressifs** : Utilisez des noms qui décrivent le comportement (`Lecteur`, `Écrivain`, `Calculateur`)
4. **Convention -er** : Beaucoup d'interfaces en Go se terminent par `-er` (ex: `Reader`, `Writer`, `Stringer`)

## Interfaces courantes du standard library

```go
// io.Reader - pour lire des données
type Reader interface {
    Read([]byte) (int, error)
}

// io.Writer - pour écrire des données
type Writer interface {
    Write([]byte) (int, error)
}

// fmt.Stringer - pour la représentation string
type Stringer interface {
    String() string
}

// sort.Interface - pour le tri
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

## Résumé

Les interfaces en Go sont un outil puissant pour créer du code flexible et modulaire :

- **Implémentation implicite** : Pas besoin de déclarer explicitement l'implémentation
- **Polymorphisme** : Permet à différents types de se comporter de manière uniforme
- **Composition** : Les interfaces peuvent être composées d'autres interfaces
- **Découplage** : Réduit les dépendances entre les composants

Les interfaces sont la clé pour écrire du code Go idiomatique et maintenable. Dans la prochaine section, nous explorerons l'interface vide et les assertions de type plus en détail.

⏭️
