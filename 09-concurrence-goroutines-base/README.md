🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9. Concurrence (Goroutines de base)

## Introduction

La concurrence est l'une des fonctionnalités les plus puissantes et distinctives du langage Go. Contrairement à de nombreux autres langages qui traitent la concurrence comme une fonctionnalité additionnelle complexe, Go l'intègre directement dans sa philosophie et sa syntaxe de base.

## Qu'est-ce que la concurrence ?

La concurrence est la capacité d'un programme à gérer plusieurs tâches en même temps. Il est important de distinguer :

- **Concurrence** : Capacité à structurer un programme pour gérer plusieurs tâches simultanément
- **Parallélisme** : Exécution simultanée de plusieurs tâches sur différents processeurs

La concurrence est une propriété du code, tandis que le parallélisme est une propriété de l'exécution.

## Pourquoi Go excelle-t-il dans la concurrence ?

### 1. **Simplicité syntaxique**
Go rend la programmation concurrente accessible grâce à une syntaxe simple et intuitive. Là où d'autres langages nécessitent des bibliothèques complexes, Go propose des primitives intégrées.

### 2. **Modèle CSP (Communicating Sequential Processes)**
Go s'inspire du modèle CSP développé par Tony Hoare, qui favorise la communication entre processus plutôt que le partage de mémoire. Cette approche réduit considérablement les problèmes de concurrence classiques.

### 3. **Légèreté des goroutines**
Les goroutines sont des threads ultra-légers gérés par le runtime Go. Contrairement aux threads du système d'exploitation, vous pouvez créer des milliers de goroutines sans problème de performance.

### 4. **Channels pour la communication**
Les channels fournissent un mécanisme de communication type-safe entre goroutines, éliminant le besoin de synchronisation manuelle dans de nombreux cas.

## Concepts fondamentaux

### Goroutines
Les goroutines sont des fonctions qui s'exécutent de manière concurrente avec d'autres goroutines. Elles sont :
- Légères (quelques KB de mémoire initialement)
- Gérées par le runtime Go
- Multiplexées sur les threads du système

### Channels
Les channels sont des conduits typés qui permettent aux goroutines de communiquer et de synchroniser leurs exécutions. Ils incarnent la philosophie Go : "Ne communiquez pas en partageant la mémoire, partagez la mémoire en communiquant."

### Scheduler
Le scheduler Go distribue les goroutines sur les threads disponibles, permettant d'exploiter efficacement les ressources système.

## Avantages de la concurrence en Go

### 1. **Performance**
- Exploitation optimale des processeurs multi-cœurs
- Meilleure utilisation des ressources système
- Capacité à gérer des milliers de connexions simultanées

### 2. **Scalabilité**
- Applications capable de gérer une charge élevée
- Facilité d'adaptation aux besoins croissants
- Efficacité dans les architectures distribuées

### 3. **Simplicité de développement**
- Syntaxe intuitive pour les opérations concurrentes
- Moins de code boilerplate que les alternatives
- Debugging facilité par des outils intégrés

### 4. **Robustesse**
- Gestion automatique des goroutines par le runtime
- Détection de race conditions avec l'outil race detector
- Patterns de concurrence éprouvés

## Cas d'usage typiques

### Serveurs web
Gestion simultanée de milliers de requêtes HTTP sans bloquer l'application.

### APIs et microservices
Traitement parallèle des appels externes et des opérations base de données.

### Traitement de données
Pipelines de traitement avec étapes parallèles et séquentielles.

### Applications temps réel
Chat, notifications, streaming de données en temps réel.

## Défis et bonnes pratiques

Bien que Go simplifie la programmation concurrente, certains défis persistent :

### Race conditions
Accès concurrent aux mêmes données sans synchronisation appropriée.

### Deadlocks
Blocage mutuel entre goroutines attendant des ressources.

### Resource leaks
Goroutines qui ne se terminent pas correctement.

### Communication patterns
Choix entre channels, mutexes, et autres primitives de synchronisation.

## Structure de cette section

Cette section couvre les concepts essentiels de la concurrence en Go :

- **9-1 : Introduction aux goroutines** - Création et gestion des goroutines
- **9-2 : Channels basiques** - Communication entre goroutines
- **9-3 : Select statement** - Gestion de multiples channels
- **9-4 : Patterns de concurrence simples** - Patterns courants et bonnes pratiques

## Prérequis

Avant d'aborder cette section, assurez-vous de maîtriser :
- Les fonctions Go (chapitre 4)
- Les structures de données (chapitre 5)
- Les interfaces (chapitre 6)
- La gestion d'erreurs (chapitre 7)

## Citation inspirante

> "Don't communicate by sharing memory, share memory by communicating."
>
> *— Rob Pike, co-créateur de Go*

Cette philosophie guide toute la programmation concurrente en Go et sera le fil conducteur de notre exploration des goroutines et channels.

⏭️
