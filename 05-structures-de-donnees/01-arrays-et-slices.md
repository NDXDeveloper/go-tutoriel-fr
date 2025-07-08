🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5-1 : Arrays et Slices

## Introduction

Les arrays (tableaux) et slices (tranches) sont des structures de données fondamentales en Go pour stocker des collections d'éléments du même type. Bien qu'ils soient liés, ils ont des caractéristiques très différentes qu'il est important de comprendre.

## Arrays (Tableaux)

### Qu'est-ce qu'un Array ?

Un array est une collection d'éléments de même type avec une **taille fixe** définie à la compilation. Une fois créé, vous ne pouvez pas changer sa taille.

### Déclaration d'Arrays

```go
// Déclaration avec initialisation à zéro
var nombres [5]int // [0, 0, 0, 0, 0]

// Déclaration avec valeurs initiales
var fruits [3]string = [3]string{"pomme", "banane", "orange"}

// Syntaxe courte
couleurs := [4]string{"rouge", "bleu", "vert", "jaune"}

// Laisser Go calculer la taille automatiquement
animaux := [...]string{"chat", "chien", "oiseau"} // taille = 3
```

### Accès aux éléments

```go
nombres := [5]int{10, 20, 30, 40, 50}

// Accès en lecture
fmt.Println(nombres[0]) // 10
fmt.Println(nombres[2]) // 30

// Modification
nombres[1] = 25
fmt.Println(nombres) // [10, 25, 30, 40, 50]

// Obtenir la longueur
fmt.Println(len(nombres)) // 5
```

### Parcourir un Array

```go
scores := [4]int{85, 92, 78, 96}

// Méthode 1 : boucle for classique
for i := 0; i < len(scores); i++ {
    fmt.Printf("Score %d: %d\n", i, scores[i])
}

// Méthode 2 : range (recommandée)
for index, score := range scores {
    fmt.Printf("Score %d: %d\n", index, score)
}

// Si vous n'avez pas besoin de l'index
for _, score := range scores {
    fmt.Printf("Score: %d\n", score)
}
```

### Limitations des Arrays

```go
func main() {
    // Problème : les arrays ont une taille fixe
    var petitArray [3]int = [3]int{1, 2, 3}
    var grandArray [5]int = [5]int{1, 2, 3, 4, 5}

    // Ceci ne compile pas ! Types différents
    // petitArray = grandArray // Erreur !

    // La taille fait partie du type
    fmt.Printf("Type de petitArray: %T\n", petitArray) // [3]int
    fmt.Printf("Type de grandArray: %T\n", grandArray) // [5]int
}
```

## Slices (Tranches)

### Qu'est-ce qu'un Slice ?

Un slice est une **vue dynamique** sur un array. Il peut grandir et rétrécir selon les besoins. C'est la structure de données la plus utilisée en Go pour les collections.

### Déclaration de Slices

```go
// Déclaration d'un slice vide
var nombres []int // slice vide (nil)

// Déclaration avec valeurs initiales
fruits := []string{"pomme", "banane", "orange"}

// Création avec make()
scores := make([]int, 5)      // slice de 5 éléments (tous à 0)
buffer := make([]int, 3, 10)  // longueur 3, capacité 10
```

### Différences clés : Array vs Slice

```go
// Array - taille fixe
var array [5]int = [5]int{1, 2, 3, 4, 5}

// Slice - taille dynamique
var slice []int = []int{1, 2, 3, 4, 5}

fmt.Printf("Array: %T\n", array) // [5]int
fmt.Printf("Slice: %T\n", slice) // []int
```

### Longueur et Capacité

```go
slice := make([]int, 3, 5)

fmt.Println("Longueur:", len(slice))   // 3
fmt.Println("Capacité:", cap(slice))   // 5
fmt.Println("Slice:", slice)           // [0 0 0]
```

**Explication :**
- **Longueur** : nombre d'éléments actuellement dans le slice
- **Capacité** : nombre maximum d'éléments que le slice peut contenir avant réallocation

### Ajouter des éléments avec append()

```go
var nombres []int // slice vide

// Ajouter des éléments
nombres = append(nombres, 10)
nombres = append(nombres, 20, 30, 40)

fmt.Println(nombres) // [10 20 30 40]

// Ajouter un autre slice
autresNombres := []int{50, 60}
nombres = append(nombres, autresNombres...)
fmt.Println(nombres) // [10 20 30 40 50 60]
```

### Créer des Slices à partir d'Arrays

```go
array := [6]int{1, 2, 3, 4, 5, 6}

// Créer des slices à partir de l'array
slice1 := array[1:4]   // éléments 1, 2, 3 (indices 1 à 3)
slice2 := array[:3]    // éléments 1, 2, 3 (début à indice 2)
slice3 := array[2:]    // éléments 3, 4, 5, 6 (indice 2 à la fin)
slice4 := array[:]     // tous les éléments

fmt.Println("Array:", array)      // [1 2 3 4 5 6]
fmt.Println("slice1:", slice1)    // [2 3 4]
fmt.Println("slice2:", slice2)    // [1 2 3]
fmt.Println("slice3:", slice3)    // [3 4 5 6]
fmt.Println("slice4:", slice4)    // [1 2 3 4 5 6]
```

### Attention : Slices partagent la mémoire !

```go
array := [5]int{1, 2, 3, 4, 5}
slice1 := array[1:4]  // [2 3 4]
slice2 := array[2:5]  // [3 4 5]

fmt.Println("Avant modification:")
fmt.Println("slice1:", slice1) // [2 3 4]
fmt.Println("slice2:", slice2) // [3 4 5]

// Modifier slice1 affecte l'array sous-jacent
slice1[1] = 99

fmt.Println("Après modification:")
fmt.Println("array:", array)   // [1 2 99 4 5]
fmt.Println("slice1:", slice1) // [2 99 4]
fmt.Println("slice2:", slice2) // [99 4 5] <- affecté aussi !
```

### Copier des Slices

```go
original := []int{1, 2, 3, 4, 5}

// Mauvaise façon (référence)
mauvaiseCopie := original
mauvaiseCopie[0] = 99
fmt.Println("Original:", original) // [99 2 3 4 5] - modifié !

// Bonne façon (copie indépendante)
original = []int{1, 2, 3, 4, 5} // reset
bonneCopie := make([]int, len(original))
copy(bonneCopie, original)
bonneCopie[0] = 99
fmt.Println("Original:", original)   // [1 2 3 4 5] - intact
fmt.Println("Copie:", bonneCopie)    // [99 2 3 4 5]
```

### Parcourir des Slices

```go
fruits := []string{"pomme", "banane", "orange", "kiwi"}

// Méthode range (recommandée)
for index, fruit := range fruits {
    fmt.Printf("%d: %s\n", index, fruit)
}

// Seulement les valeurs
for _, fruit := range fruits {
    fmt.Printf("Fruit: %s\n", fruit)
}

// Seulement les indices
for index := range fruits {
    fmt.Printf("Index: %d\n", index)
}
```

### Supprimer des éléments

```go
nombres := []int{1, 2, 3, 4, 5}

// Supprimer l'élément à l'index 2 (valeur 3)
index := 2
nombres = append(nombres[:index], nombres[index+1:]...)

fmt.Println(nombres) // [1 2 4 5]
```

### Exemples pratiques

#### Exemple 1 : Calculer la moyenne

```go
func moyenne(nombres []float64) float64 {
    if len(nombres) == 0 {
        return 0
    }

    var somme float64
    for _, nombre := range nombres {
        somme += nombre
    }

    return somme / float64(len(nombres))
}

func main() {
    notes := []float64{15.5, 18.0, 12.5, 16.0, 14.5}
    moy := moyenne(notes)
    fmt.Printf("Moyenne: %.2f\n", moy) // Moyenne: 15.30
}
```

#### Exemple 2 : Filtrer des éléments

```go
func filtrerPositifs(nombres []int) []int {
    var resultat []int

    for _, nombre := range nombres {
        if nombre > 0 {
            resultat = append(resultat, nombre)
        }
    }

    return resultat
}

func main() {
    valeurs := []int{-2, 5, -1, 8, 0, 3, -4}
    positifs := filtrerPositifs(valeurs)
    fmt.Println("Positifs:", positifs) // Positifs: [5 8 3]
}
```

#### Exemple 3 : Inverser un slice

```go
func inverser(slice []int) {
    for i, j := 0, len(slice)-1; i < j; i, j = i+1, j-1 {
        slice[i], slice[j] = slice[j], slice[i]
    }
}

func main() {
    nombres := []int{1, 2, 3, 4, 5}
    fmt.Println("Avant:", nombres)     // Avant: [1 2 3 4 5]

    inverser(nombres)
    fmt.Println("Après:", nombres)     // Après: [5 4 3 2 1]
}
```

## Bonnes pratiques

### 1. Préférez les Slices aux Arrays

```go
// ❌ Évitez (sauf cas spécifique)
func traiterArray(arr [5]int) {
    // Limité à exactement 5 éléments
}

// ✅ Préférez
func traiterSlice(slice []int) {
    // Fonctionne avec n'importe quelle taille
}
```

### 2. Utilisez make() pour les slices de taille connue

```go
// ❌ Moins efficace
var slice []int
for i := 0; i < 1000; i++ {
    slice = append(slice, i) // Réallocations multiples
}

// ✅ Plus efficace
slice := make([]int, 0, 1000) // Capacité pré-allouée
for i := 0; i < 1000; i++ {
    slice = append(slice, i) // Pas de réallocation
}
```

### 3. Vérifiez la longueur avant l'accès

```go
func premierElement(slice []int) int {
    if len(slice) == 0 {
        return 0 // ou gérer l'erreur
    }
    return slice[0]
}
```

### 4. Attention aux fuites mémoire

```go
// ❌ Peut causer une fuite mémoire
func extraireDebut(slice []int) []int {
    return slice[:3] // Garde une référence vers le slice original
}

// ✅ Créer une copie indépendante
func extraireDebut(slice []int) []int {
    if len(slice) < 3 {
        return slice
    }
    copie := make([]int, 3)
    copy(copie, slice[:3])
    return copie
}
```

## Exercices pratiques

### Exercice 1 : Basique
Créez un programme qui :
1. Déclare un slice de strings avec 5 prénoms
2. Affiche chaque prénom avec son index
3. Ajoute deux nouveaux prénoms
4. Affiche la longueur finale

### Exercice 2 : Intermédiaire
Écrivez une fonction qui prend un slice d'entiers et retourne :
- Le plus petit élément
- Le plus grand élément
- La somme de tous les éléments

### Exercice 3 : Avancé
Implémentez une fonction qui :
1. Prend un slice de strings
2. Retourne un nouveau slice sans les doublons
3. Préserve l'ordre de la première occurrence

```go
package main

import (
	"fmt"
)

// ==========================================
// EXERCICE 1 : BASIQUE
// ==========================================

func exercice1() {
	fmt.Println("=== EXERCICE 1 : BASIQUE ===")

	// 1. Déclarer un slice de strings avec 5 prénoms
	prenoms := []string{"Alice", "Bob", "Charlie", "Diana", "Eve"}

	// 2. Afficher chaque prénom avec son index
	fmt.Println("Prénoms initiaux :")
	for index, prenom := range prenoms {
		fmt.Printf("Index %d: %s\n", index, prenom)
	}

	// 3. Ajouter deux nouveaux prénoms
	prenoms = append(prenoms, "Frank", "Grace")

	// 4. Afficher la longueur finale
	fmt.Printf("\nAprès ajout de 2 prénoms:\n")
	fmt.Printf("Longueur finale: %d\n", len(prenoms))
	fmt.Printf("Tous les prénoms: %v\n", prenoms)

	fmt.Println()
}

// ==========================================
// EXERCICE 2 : INTERMÉDIAIRE
// ==========================================

// Fonction qui analyse un slice d'entiers
func analyserSlice(nombres []int) (min int, max int, somme int) {
	// Vérifier si le slice est vide
	if len(nombres) == 0 {
		return 0, 0, 0
	}

	// Initialiser avec le premier élément
	min = nombres[0]
	max = nombres[0]
	somme = 0

	// Parcourir tous les éléments
	for _, nombre := range nombres {
		// Calculer la somme
		somme += nombre

		// Trouver le minimum
		if nombre < min {
			min = nombre
		}

		// Trouver le maximum
		if nombre > max {
			max = nombre
		}
	}

	return min, max, somme
}

func exercice2() {
	fmt.Println("=== EXERCICE 2 : INTERMÉDIAIRE ===")

	// Test avec différents slices
	testCases := [][]int{
		{5, 2, 8, 1, 9, 3},
		{-5, -2, -8, -1},
		{42},
		{}, // slice vide
		{10, 10, 10, 10},
	}

	for i, nombres := range testCases {
		fmt.Printf("Test %d: %v\n", i+1, nombres)

		if len(nombres) == 0 {
			fmt.Println("  Slice vide !")
		} else {
			min, max, somme := analyserSlice(nombres)
			fmt.Printf("  Minimum: %d\n", min)
			fmt.Printf("  Maximum: %d\n", max)
			fmt.Printf("  Somme: %d\n", somme)
		}
		fmt.Println()
	}
}

// ==========================================
// EXERCICE 3 : AVANCÉ
// ==========================================

// Fonction pour supprimer les doublons tout en préservant l'ordre
func supprimerDoublons(slice []string) []string {
	// Utiliser une map pour suivre les éléments déjà vus
	vus := make(map[string]bool)
	resultat := []string{}

	// Parcourir le slice original
	for _, element := range slice {
		// Si l'élément n'a pas encore été vu
		if !vus[element] {
			// L'ajouter au résultat
			resultat = append(resultat, element)
			// Le marquer comme vu
			vus[element] = true
		}
	}

	return resultat
}

// Version alternative avec une approche différente
func supprimerDoublonsV2(slice []string) []string {
	if len(slice) == 0 {
		return []string{}
	}

	resultat := []string{}

	// Pour chaque élément du slice
	for _, element := range slice {
		// Vérifier s'il existe déjà dans le résultat
		existe := false
		for _, existing := range resultat {
			if existing == element {
				existe = true
				break
			}
		}

		// S'il n'existe pas, l'ajouter
		if !existe {
			resultat = append(resultat, element)
		}
	}

	return resultat
}

func exercice3() {
	fmt.Println("=== EXERCICE 3 : AVANCÉ ===")

	// Test avec différents slices
	testCases := [][]string{
		{"pomme", "banane", "pomme", "orange", "banane", "kiwi"},
		{"a", "b", "c", "a", "b", "c", "d"},
		{"test", "test", "test"},
		{"unique"},
		{}, // slice vide
		{"", "vide", "", "encore", "vide"},
	}

	for i, slice := range testCases {
		fmt.Printf("Test %d:\n", i+1)
		fmt.Printf("  Original: %v\n", slice)

		// Tester la première version
		sansDoublons := supprimerDoublons(slice)
		fmt.Printf("  Sans doublons (v1): %v\n", sansDoublons)

		// Tester la deuxième version
		sansDoublonsV2 := supprimerDoublonsV2(slice)
		fmt.Printf("  Sans doublons (v2): %v\n", sansDoublonsV2)

		fmt.Println()
	}
}

// ==========================================
// FONCTIONS UTILITAIRES BONUS
// ==========================================

// Fonction pour afficher des statistiques sur un slice
func afficherStats(slice []int) {
	if len(slice) == 0 {
		fmt.Println("Slice vide")
		return
	}

	min, max, somme := analyserSlice(slice)
	moyenne := float64(somme) / float64(len(slice))

	fmt.Printf("Statistiques pour %v:\n", slice)
	fmt.Printf("  Longueur: %d\n", len(slice))
	fmt.Printf("  Minimum: %d\n", min)
	fmt.Printf("  Maximum: %d\n", max)
	fmt.Printf("  Somme: %d\n", somme)
	fmt.Printf("  Moyenne: %.2f\n", moyenne)
}

// Fonction pour démontrer les concepts avancés
func demonstrationAvancee() {
	fmt.Println("=== DÉMONSTRATION AVANCÉE ===")

	// Montrer la différence entre append et copy
	original := []int{1, 2, 3}
	fmt.Printf("Original: %v\n", original)

	// Méthode 1: append (peut réallouer)
	copie1 := append([]int{}, original...)
	copie1[0] = 99
	fmt.Printf("Après modification copie1: original=%v, copie1=%v\n", original, copie1)

	// Méthode 2: copy (plus explicite)
	copie2 := make([]int, len(original))
	copy(copie2, original)
	copie2[1] = 88
	fmt.Printf("Après modification copie2: original=%v, copie2=%v\n", original, copie2)

	// Montrer la capacité
	slice := make([]int, 3, 10)
	fmt.Printf("Slice avec longueur 3, capacité 10: %v\n", slice)
	fmt.Printf("Longueur: %d, Capacité: %d\n", len(slice), cap(slice))

	// Ajouter des éléments
	for i := 0; i < 8; i++ {
		slice = append(slice, i)
		fmt.Printf("Après ajout de %d: len=%d, cap=%d\n", i, len(slice), cap(slice))
	}
}

// ==========================================
// FONCTION PRINCIPALE
// ==========================================

func main() {
	fmt.Println("SOLUTIONS DES EXERCICES - ARRAYS ET SLICES")
	fmt.Println("===========================================")

	exercice1()
	exercice2()
	exercice3()

	// Bonus: démonstration avancée
	demonstrationAvancee()

	// Bonus: utilisation de la fonction utilitaire
	fmt.Println("\n=== BONUS: STATISTIQUES ===")
	testSlice := []int{15, 8, 23, 4, 16, 42, 11, 7}
	afficherStats(testSlice)
}
```

## Résumé

Les arrays et slices sont des structures fondamentales en Go :

- **Arrays** : Taille fixe, passés par valeur, rarement utilisés directement
- **Slices** : Taille dynamique, passés par référence, très utilisés

**Points clés à retenir :**
- Les slices sont plus flexibles que les arrays
- Utilisez `append()` pour ajouter des éléments
- Attention au partage de mémoire entre slices
- Utilisez `make()` pour pré-allouer la capacité quand possible
- Toujours vérifier la longueur avant l'accès aux éléments

Les slices sont l'une des structures de données les plus importantes en Go. Maîtriser leur utilisation vous permettra d'écrire du code plus efficace et plus idiomatique.

⏭️
