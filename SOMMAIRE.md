# Tutoriel Go de débutant à avancé

## **Partie I : Fondamentaux (Débutant)**

### 1. [Introduction à Go](./01-introduction-a-go/)
- 1-1 : [Histoire et philosophie du langage](./01-introduction-a-go/01-histoire-et-philosophie-du-langage.md)
- 1-2 : [Installation et configuration de l'environnement](./01-introduction-a-go/02-installation-et-configuration-environnement.md)
- 1-3 : [Premier programme "Hello World"](./01-introduction-a-go/03-premier-programme-hello-world.md)
- 1-4 : [Structure d'un projet Go](./01-introduction-a-go/04-structure-projet-go.md)

### 2. [Syntaxe de base](./02-syntaxe-de-base/)
- 2-1 : [Variables et constantes](./02-syntaxe-de-base/01-variables-et-constantes.md)
- 2-2 : [Types de données primitifs](./02-syntaxe-de-base/02-types-donnees-primitifs.md)
- 2-3 : [Opérateurs arithmétiques et logiques](./02-syntaxe-de-base/03-operateurs-arithmetiques-et-logiques.md)
- 2-4 : [Commentaires et conventions de nommage](./02-syntaxe-de-base/04-commentaires-et-conventions-nommage.md)

### 3. [Structures de contrôle](./03-structures-de-controle/)
- 3-1 : [Conditions (if/else, switch)](./03-structures-de-controle/01-conditions-if-else-switch.md)
- 3-2 : [Boucles (for, range)](./03-structures-de-controle/02-boucles-for-range.md)
- 3-3 : [Gestion des erreurs basique](./03-structures-de-controle/03-gestion-erreurs-basique.md)

### 4. [Fonctions](./04-fonctions/)
- 4-1 : [Déclaration et appel de fonctions](./04-fonctions/01-declaration-et-appel-fonctions.md)
- 4-2 : [Paramètres et valeurs de retour](./04-fonctions/02-parametres-et-valeurs-retour.md)
- 4-3 : [Fonctions variadiques](./04-fonctions/03-fonctions-variadiques.md)
- 4-4 : [Fonctions anonymes et closures](./04-fonctions/04-fonctions-anonymes-et-closures.md)

### 5. [Structures de données](./05-structures-de-donnees/)
- 5-1 : [Arrays et slices](./05-structures-de-donnees/01-arrays-et-slices.md)
- 5-2 : [Maps](./05-structures-de-donnees/02-maps.md)
- 5-3 : [Structs](./05-structures-de-donnees/03-structs.md)
- 5-4 : [Pointers](./05-structures-de-donnees/04-pointers.md)

## **Partie II : Programmation intermédiaire**

### 6. [Méthodes et interfaces](./06-methodes-et-interfaces/)
- 6-1 : [Définition de méthodes](./06-methodes-et-interfaces/01-definition-methodes.md)
- 6-2 : [Interfaces et polymorphisme](./06-methodes-et-interfaces/02-interfaces-et-polymorphisme.md)
- 6-3 : [Interface vide et assertions de type](./06-methodes-et-interfaces/03-interface-vide-et-assertions-type.md)
- 6-4 : [Composition vs héritage](./06-methodes-et-interfaces/04-composition-vs-heritage.md)

### 7. [Gestion des erreurs](./07-gestion-des-erreurs/)
- 7-1 : [Conventions d'erreur en Go](./07-gestion-des-erreurs/01-conventions-erreur-en-go.md)
- 7-2 : [Création d'erreurs personnalisées](./07-gestion-des-erreurs/02-creation-erreurs-personnalisees.md)
- 7-3 : [Panic et recover](./07-gestion-des-erreurs/03-panic-et-recover.md)
- 7-4 : [Bonnes pratiques](./07-gestion-des-erreurs/04-bonnes-pratiques.md)

### 8. [Packages et modules](./08-packages-et-modules/)
- 8-1 : [Organisation du code en packages](./08-packages-et-modules/01-organisation-code-en-packages.md)
- 8-2 : [Visibilité (public/private)](./08-packages-et-modules/02-visibilite-public-private.md)
- 8-3 : [Go modules et gestion des dépendances](./08-packages-et-modules/03-go-modules-et-gestion-dependances.md)
- 8-4 : [Documentation avec godoc](./08-packages-et-modules/04-documentation-avec-godoc.md)

### 9. [Concurrence (Goroutines de base)](./09-concurrence-goroutines-base/)
- 9-1 : [Introduction aux goroutines](./09-concurrence-goroutines-base/01-introduction-aux-goroutines.md)
- 9-2 : [Channels basiques](./09-concurrence-goroutines-base/02-channels-basiques.md)
- 9-3 : [Select statement](./09-concurrence-goroutines-base/03-select-statement.md)
- 9-4 : [Patterns de concurrence simples](./09-concurrence-goroutines-base/04-patterns-concurrence-simples.md)

## **Partie III : Développement avancé**

### 10. [Concurrence avancée](./10-concurrence-avancee/)
- 10-1 : [Channels buffered et unbuffered](./10-concurrence-avancee/01-channels-buffered-et-unbuffered.md)
- 10-2 : [Worker pools](./10-concurrence-avancee/02-worker-pools.md)
- 10-3 : [Context package](./10-concurrence-avancee/03-context-package.md)
- 10-4 : [Sync package (WaitGroup, Mutex, etc.)](./10-concurrence-avancee/04-sync-package-waitgroup-mutex.md)

### 11. [Tests et benchmarks](./11-tests-et-benchmarks/)
- 11-1 : [Tests unitaires avec testing](./11-tests-et-benchmarks/01-tests-unitaires-avec-testing.md)
- 11-2 : [Table-driven tests](./11-tests-et-benchmarks/02-table-driven-tests.md)
- 11-3 : [Mocking et interfaces](./11-tests-et-benchmarks/03-mocking-et-interfaces.md)
- 11-4 : [Benchmarks et profiling](./11-tests-et-benchmarks/04-benchmarks-et-profiling.md)

### 12. [Réflexion et métaprogrammation](./12-reflexion-et-metaprogrammation/)
- 12-1 : [Package reflect](./12-reflexion-et-metaprogrammation/01-package-reflect.md)
- 12-2 : [Tags de struct](./12-reflexion-et-metaprogrammation/02-tags-de-struct.md)
- 12-3 : [Génération de code](./12-reflexion-et-metaprogrammation/03-generation-de-code.md)
- 12-4 : [Cas d'usage pratiques](./12-reflexion-et-metaprogrammation/04-cas-usage-pratiques.md)

### 13. [Programmation réseau](./13-programmation-reseau/)
- 13-1 : [HTTP client et serveur](./13-programmation-reseau/01-http-client-et-serveur.md)
- 13-2 : [JSON et sérialisation](./13-programmation-reseau/02-json-et-serialisation.md)
- 13-3 : [APIs REST](./13-programmation-reseau/03-apis-rest.md)
- 13-4 : [WebSockets](./13-programmation-reseau/04-websockets.md)

### 14. [Base de données](./14-base-de-donnees/)
- 14-1 : [SQL avec database/sql](./14-base-de-donnees/01-sql-avec-database-sql.md)
- 14-2 : [ORM (GORM)](./14-base-de-donnees/02-orm-gorm.md)
- 14-3 : [Migrations](./14-base-de-donnees/03-migrations.md)
- 14-4 : [Connexions et pools](./14-base-de-donnees/04-connexions-et-pools.md)

## **Partie IV : Projets pratiques**

### 15. [Projet 1 : CLI Tool](./15-projet-1-cli-tool/)
- 15-1 : [Parsing des arguments](./15-projet-1-cli-tool/01-parsing-des-arguments.md)
- 15-2 : [Interaction avec le système de fichiers](./15-projet-1-cli-tool/02-interaction-avec-systeme-fichiers.md)
- 15-3 : [Distribution et packaging](./15-projet-1-cli-tool/03-distribution-et-packaging.md)

### 16. [Projet 2 : API REST](./16-projet-2-api-rest/)
- 16-1 : [Architecture d'une API](./16-projet-2-api-rest/01-architecture-api.md)
- 16-2 : [Middleware et authentification](./16-projet-2-api-rest/02-middleware-et-authentification.md)
- 16-3 : [Validation des données](./16-projet-2-api-rest/03-validation-des-donnees.md)
- 16-4 : [Déploiement](./16-projet-2-api-rest/04-deploiement.md)

### 17. [Projet 3 : Microservice](./17-projet-3-microservice/)
- 17-1 : [Architecture microservices](./17-projet-3-microservice/01-architecture-microservices.md)
- 17-2 : [Communication inter-services](./17-projet-3-microservice/02-communication-inter-services.md)
- 17-3 : [Observabilité (logs, métriques)](./17-projet-3-microservice/03-observabilite-logs-metriques.md)
- 17-4 : [Containerisation avec Docker](./17-projet-3-microservice/04-containerisation-avec-docker.md)

## **Partie V : Optimisation et production**

### 18. [Performance et optimisation](./18-performance-et-optimisation/)
- 18-1 : [Profiling mémoire et CPU](./18-performance-et-optimisation/01-profiling-memoire-et-cpu.md)
- 18-2 : [Optimisations courantes](./18-performance-et-optimisation/02-optimisations-courantes.md)
- 18-3 : [Garbage collector](./18-performance-et-optimisation/03-garbage-collector.md)
- 18-4 : [Memory leaks](./18-performance-et-optimisation/04-memory-leaks.md)

### 19. [Patterns et architecture](./19-patterns-et-architecture/)
- 19-1 : [Design patterns en Go](./19-patterns-et-architecture/01-design-patterns-en-go.md)
- 19-2 : [Clean architecture](./19-patterns-et-architecture/02-clean-architecture.md)
- 19-3 : [Dependency injection](./19-patterns-et-architecture/03-dependency-injection.md)
- 19-4 : [Configuration management](./19-patterns-et-architecture/04-configuration-management.md)

### 20. [Déploiement et DevOps](./20-deploiement-et-devops/)
- 20-1 : [CI/CD avec Go](./20-deploiement-et-devops/01-ci-cd-avec-go.md)
- 20-2 : [Monitoring et observabilité](./20-deploiement-et-devops/02-monitoring-et-observabilite.md)
- 20-3 : [Sécurité](./20-deploiement-et-devops/03-securite.md)
- 20-4 : [Bonnes pratiques en production](./20-deploiement-et-devops/04-bonnes-pratiques-en-production.md)
