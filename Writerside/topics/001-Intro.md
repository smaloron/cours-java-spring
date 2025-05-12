# Intro

Spring est un Jakarta EE simplifié, modulaire et extensible.
Il est à JEE ce que Symfony est à PHP. Ces deux frameworks offrent :

- Un conteneur d’injection de dépendances

- Un système de configuration via annotations ou YAML/XML

- Une architecture modulaire basée sur des composants

- Des controllers qui traitent les requêtes web

- Un ORM (Doctrine ↔ Spring Data JPA)

- Une sécurité configurable (SecurityBundle ↔ Spring Security)

### Les principaux modules de Spring

![spring-architecture.png](spring-architecture.png)

| Module                   | Rôle principal                          | Exemples d'applications                              |
|--------------------------|-----------------------------------------|------------------------------------------------------|
| spring-core              | Conteneur IoC, injection de dépendances | Toute application Spring                             |
| spring-beans             | Gestion des beans Java                  | App de gestion de stock                              |
| spring-context           | Contexte applicatif, événements, i18n   | App multilingue                                      |
| spring-expression (SpEL) | Expressions dynamiques dans les configs | Seuils d’alerte dynamiques                           |
| spring-web               | Communication HTTP, support servlet     | API REST, upload de fichiers                         |
| spring-webmvc            | Implémentation MVC                      | Site e-commerce                                      |
| spring-webflux           | Programmation réactive (non bloquante)  | Streaming, chat en temps réel                        |
| spring-websocket         | Communication WebSocket                 | Notifications temps réel                             |
| spring-jdbc              | Accès base de données via JDBC          | Reporting, app legacy                                |
| spring-tx                | Gestion des transactions                | App bancaire (virements)                             |
| spring-orm               | Intégration ORM (Hibernate, JPA)        | Inscription universitaire                            |
| spring-data              | Abstraction des DAO                     | Blog, forum, app CRUD                                |
| spring-oxm               | Mapping XML ↔ Objet                     | App B2B avec échanges XML                            |
| spring-jms               | Intégration avec JMS                    | Commandes en file d’attente                          |
| spring-security          | Sécurité Web et méthode                 | Portail sécurisé avec rôles                          |
| spring-security-oauth2   | Authentification OAuth2/OpenID          | Connexion via Google/Facebook                        |
| spring-messaging         | Communication asynchrone                | Alertes en temps réel                                |
| spring-integration       | Intégration de systèmes par EIP         | Pipeline de traitement de documents                  |
| spring-batch             | Traitement par lots                     | Génération de factures ou fiches de paie             |
| spring-aop               | Programmation orientée aspect           | Logging, sécurité transverse                         |
| spring-aspects           | Intégration AspectJ                     | Audit réglementaire                                  |
| spring-cloud-config      | Configuration centralisée               | Microservices avec config Git                        |
| spring-cloud-netflix     | Découverte, tolérance de panne          | Architecture microservices (Eureka, Ribbon, Hystrix) |
| spring-cloud-gateway     | Gateway réactive API                    | Unification des microservices                        |
| spring-cloud-stream      | Messaging orienté événements            | Analytics, tracking utilisateurs                     |
| spring-test              | Tests unitaires et d’intégration        | Test de services, REST, base H2                      |



#### Core Container
##### spring-core
**Rôle :** Fournit le cœur du framework, notamment le conteneur IoC (Inversion of Control).

**Fonctionnalités :** Injection de dépendances, gestion du cycle de vie des beans.

**Exemples d'applications :**

Toute application Spring ! C’est la base nécessaire pour faire fonctionner les autres modules.

##### spring-beans
**Rôle :** Manipulation des objets Java (beans) déclarés dans le conteneur Spring.

**Fonctionnalités :** Création, configuration, initialisation, destruction des beans.

**Exemple :**

Une application de gestion de stock où ProductService, ProductRepository, etc. sont gérés comme beans Spring.

##### spring-context
**Rôle :** Fournit un contexte riche pour l'application (ApplicationContext).

**Fonctionnalités :** Gestion de ressources, internationalisation, événements d’application.

**Exemples :**

Une application Web multilingue avec des messages personnalisés selon la langue.

##### spring-expression (SpEL)
**Rôle :** Expression Language utilisé dans les fichiers de configuration ou annotations.

**Fonctionnalités :** Manipulation dynamique des valeurs (ex : ${...}, #{...}).

**Exemple :**

Configuration dynamique des seuils d’alerte dans un système de monitoring.

#### Web
##### spring-web
**Rôle :** Base pour le développement web (servlets, REST, etc.).

**Fonctionnalités :** Gestion des requêtes HTTP, multipart, cookies.

**Exemples :**

API REST de consultation météo ou service de téléchargement de fichiers.

##### spring-webmvc
**Rôle :** Implémentation du design pattern MVC.

**Fonctionnalités :** Contrôleurs, vues (JSP, Thymeleaf), routes, validation.

**Exemples :**

Application e-commerce avec catalogue produit, panier, et page de paiement.

##### spring-webflux
**Rôle :** Développement réactif non-bloquant avec Project Reactor.

**Fonctionnalités :** Web MVC asynchrone pour grande scalabilité.

**Exemples :**

Application de streaming vidéo/audio ou système de chat en temps réel.

##### spring-websocket
**Rôle :** Support WebSocket pour communication bidirectionnelle en temps réel.

**Exemples :**

Dashboard temps réel de données financières ou notifications push pour une app mobile.

#### Data Access / Integration

##### spring-jdbc
**Rôle :** Accès à la base via JDBC sans boilerplate.

**Exemples :**

Application de reporting se connectant à une base PostgreSQL.

##### spring-tx
**Rôle :** Gestion déclarative ou programmatique des transactions.

**Exemples :**

Application bancaire assurant la cohérence entre virement et réception.

##### spring-orm
**Rôle :** Intégration avec ORM comme Hibernate ou JPA.

**Exemples :**

Système de gestion d’inscriptions universitaires utilisant des entités Student, Course.

##### spring-data
**Rôle :** Abstraction des opérations CRUD avec Spring Data JPA, Mongo, etc.

**Exemples :**

Application de blog où les articles et commentaires sont sauvegardés en base.

##### spring-oxm
**Rôle :** Mapping objet ↔ XML.

**Exemples :**

Application de gestion de commandes B2B échangeant des documents XML via SOAP.

##### spring-jms
**Rôle :** Intégration avec les brokers de messages (ActiveMQ, Artemis…).

**Exemples :**

Application de traitement de commandes en file d’attente (asynchrone).

#### Security

##### spring-security
**Rôle :** Gestion complète de la sécurité.

**Fonctionnalités :** Authentification, autorisation, CSRF, OAuth2, etc.

**Exemples :**

Application d'administration avec rôles ADMIN, USER.

##### spring-security-oauth2
**Rôle :** Intégration OAuth2 / OpenID Connect.

**Exemples :**

Connexion via Google/Facebook dans une application de réseau social.

#### Messaging & Events

##### spring-messaging
**Rôle :** Abstraction pour la communication message-based (WebSocket, STOMP).

**Exemples :**

Notification temps réel de nouvelles offres dans une app d'annonces.

##### spring-integration
**Rôle :** Architecture orientée messages, intégration entre systèmes (EIP).

**Exemples :**

Pipeline de traitement de documents entre systèmes internes.

##### spring-batch
**Rôle :** Traitement de données par lots.

**Fonctionnalités :** Étapes, redémarrage, log, parallélisme.

**Exemples :**

Traitement nocturne de fichiers de paie ou génération de factures PDF.

#### AOP

#### spring-aop
**Rôle :** Ajout d’aspects (cross-cutting concerns).

**Exemples :**

Journalisation, mesures de performance, ou sécurité appliquée à plusieurs méthodes.

##### spring-aspects
**Rôle :** Intégration avec AspectJ.

**Exemples :**

Audit détaillé des appels de méthodes dans une application réglementée.

#### Spring Cloud (microservices)

##### spring-cloud-config
**Rôle :** Configuration centralisée pour microservices.

**Exemples :**

Système distribué où la configuration est poussée depuis Git.

##### spring-cloud-netflix
**Rôle :** Intégration Netflix OSS (Eureka, Ribbon, Hystrix).

**Exemples :**

Architecture microservices avec service discovery et tolérance de panne.

##### spring-cloud-gateway
**Rôle :** API Gateway réactive.

**Exemples :**

Microservices REST derrière une seule URL publique.

##### spring-cloud-stream
**Rôle :** Abstraction pour Kafka/RabbitMQ.

**Exemples :**

Traitement d’événements utilisateur (clicks, logs, analytics).

#### Test

##### spring-test
**Rôle :** Support pour les tests unitaires/integration.

**Exemples :**

Tests de contrôleurs REST, DAO avec base en mémoire (H2), etc.

## Création d'un projet

Spring propose un site qui facilite le démarrage d'un projet.
On rentre quelques paramètres, on sélectionne les dépendances
et on obtient une très bonne base pour commencer à travailler.

Il suffit de télécharger et décompresser le fichier `.zip` puis de l'ouvrir dans un IDE.

[https://start.spring.io](https://start.spring.io)

![CleanShot 2025-04-20 at 16.25.38@2x.png](CleanShot 2025-04-20 at 16.25.38@2x.png)

### Intégration dans IntelliJ

Avec IntelliJ les choses sont encore plus simples puisque tout est intégré à l'IDE. Il suffit de créer un nouveau 
projet et de choisir Spring Boot.

![CleanShot 2025-04-20 at 16.43.44@2x.png](CleanShot 2025-04-20 at 16.43.44@2x.png)

La fenêtre suivante propose le choix des dépendances

![CleanShot 2025-04-20 at 16.48.00@2x.png](CleanShot 2025-04-20 at 16.48.00@2x.png)

### Un premier contrôleur et une première vue

#### Le contrôleur

```java
package fr.formation.spring.demo.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

// Indique que ce contrôleur sert des vues (pas REST)
@Controller
public class HelloController {

    // Définition de la route
    @GetMapping("/hello")
    // L'argument model est injecté par Spring MVC,
    // il sert à passer des données du contrôleur vers la vue
    public String hello(Model model) {
        // Ajoute une variable à la vue via le model
        model.addAttribute("message", "Bonjour Spring !");
        // Retourne le nom de la vue (hello.html ou hello.jsp)
        return "hello";
    }
}
```

#### La vue
```html
<!DOCTYPE html>
<!-- chargement du name space Thymeleaf dans ce document html -->
<html xmlns:th="http://www.thymeleaf.org">
<head>
  <title>Hello Page</title>
  <meta charset="UTF-8"/>
</head>
<body>
<!-- Affichage de la variable passée à la vue -->
<h1 th:text="${message}">Message par défaut</h1>
</body>
</html>
```

#### L'exécution

```shell
./mvnw spring-boot:run
```

Tester dans un navigateur 

[http://localhost:8080](http://localhost:8080)