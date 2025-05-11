# Les fichier .env

Spring Boot ne supporte pas *nativement* la lecture directe des fichiers `.env` de la même manière qu'il lit
`application.properties`. Cependant, il est tout à fait possible d'intégrer cette approche en utilisant une bibliothèque
tierce. La bibliothèque la plus couramment utilisée pour cela en Java est `java-dotenv`.

### Pourquoi utiliser un fichier `.env` avec Spring Boot ?

L'utilisation d'un fichier `.env` présente plusieurs avantages :

1. **Séparation de la Configuration** : Il sépare clairement la configuration sensible ou spécifique à l'environnement
   du code source.
2. **Sécurité** : En ajoutant `.env` au fichier `.gitignore`, on évite de commiter des informations sensibles (clés
   d'API, mots de passe) dans le dépôt de code.
3. **Simplicité pour le Développement Local** : Chaque développeur peut avoir son propre fichier `.env` avec ses
   configurations locales sans affecter les autres.
4. **Cohérence entre Projets** : Si une équipe travaille sur des projets utilisant différentes technologies (Node.js,
   Python, Java), l'utilisation de `.env` peut apporter une certaine cohérence dans la gestion de la configuration
   d'environnement.

## Intégration de `java-dotenv` dans un Projet Spring Boot

Pour utiliser un fichier `.env` dans Spring Boot, il faut ajouter la dépendance `java-dotenv` au projet.

**1. Ajouter la Dépendance**

* **Avec Maven** (dans `pom.xml`) :

```xml

<dependency>
    <groupId>io.github.cdimascio</groupId>
    <artifactId>java-dotenv</artifactId>
    <version>5.2.2</version> <!-- Vérifiez la dernière version -->
</dependency>
```

**2. Charger le Fichier `.env` au Démarrage**

La bibliothèque `java-dotenv` doit charger les variables du fichier `.env` *avant* que Spring Boot ne commence à lire sa
configuration. Le meilleur endroit pour faire cela est au tout début de la méthode `main`.

La bibliothèque charge les variables du fichier `.env` (qui doit se trouver à la racine du projet par défaut) et les
injecte dans les *propriétés système* Java (`System.getProperties()`) ou, selon la configuration, dans les *variables
d'environnement* du processus. Spring Boot lit ensuite ces propriétés système et variables d'environnement comme sources
de configuration valides.

**Exemple de modification de la classe principale :**

```java
package fr.formation.spring;

import io.github.cdimascio.dotenv.Dotenv; // Import nécessaire
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        // Charger les variables du fichier .env
        // Il est crucial de le faire AVANT SpringApplication.run(...)
        // 'load()' les charge dans les propriétés système par défaut.
        Dotenv dotenv = Dotenv.configure()
                .ignoreIfMissing() // Ne pas échouer si .env manque
                .load();

        SpringApplication.run(DemoApplication.class, args);
    }
}
```

**3. Créer le Fichier `.env`**

À la racine du projet, créez un fichier nommé `.env`.

**Exemple de fichier `.env` :**

```dotenv
# Configuration de la base de données
DATABASE_URL=jdbc:postgresql://localhost:5432/mydatabase
DATABASE_USER=mon_utilisateur
DATABASE_PASSWORD=mon_mot_de_passe_secret

# Clé d'API pour un service externe
API_SERVICE_KEY=abcdef1234567890
API_SERVICE_URL=https://api.externalservice.com

# Configuration spécifique à l'application
APP_FEATURE_FLAG_ENABLED=true
APP_DEFAULT_PAGE_SIZE=20
```

**Important :** Ajoutez `.env` à votre fichier `.gitignore` pour ne pas l'inclure dans les commits !

```gitignore
# Fichiers d'environnement locaux
.env
.env.*

# Fichiers générés par l'IDE
# ... autres exclusions ...
```

## Accéder aux Variables d'Environnement dans Spring Boot

Une fois que `java-dotenv` a chargé les variables dans les propriétés système ou l'environnement, Spring Boot peut y
accéder de ses manières habituelles :

### **1. Via `@Value`**

C'est la méthode la plus simple pour injecter une seule propriété.

```java
package fr.formation.spring.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class ApiService {

    // Injection de la valeur depuis .env (via les propriétés système)
    // Le ':' permet de définir une valeur par défaut si la clé n'existe pas
    @Value("${API_SERVICE_KEY:defaultApiKey}")
    private String apiKey;

    @Value("${API_SERVICE_URL}")
    private String serviceUrl;

    public void useApiKey() {
        System.out.println("Utilisation de l'URL : " + serviceUrl);
        System.out.println("Utilisation de la clé d'API : " + apiKey);
        // Logique métier utilisant la clé d'API...
    }
}
```

### **2. Via `@ConfigurationProperties`**

Cette approche est recommandée pour regrouper des propriétés liées. Elle est plus type-safe et mieux organisée.

* **Créer une classe de configuration :**

```java
package fr.formation.spring.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
// Optionnel: validation
// import javax.validation.constraints.NotEmpty; 
// import javax.validation.constraints.NotNull;

// Lie les propriétés préfixées par "database"
@Configuration
// Le préfixe doit correspondre aux clés dans .env (minuscules/kebab-case)
// MAIS java-dotenv charge souvent en majuscules/underscore.
// Spring Boot est flexible et tente de faire correspondre 
// DATABASE_URL ou database.url à database.url
@ConfigurationProperties(prefix = "database")
public class DatabaseConfig {

    // @NotEmpty // Nécessite spring-boot-starter-validation
    private String url; // Correspond à DATABASE_URL

    // @NotEmpty
    private String user; // Correspond à DATABASE_USER

    // @NotEmpty
    private String password; // Correspond à DATABASE_PASSWORD

    // Getters et Setters nécessaires pour @ConfigurationProperties
    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getUser() {
        return user;
    }

    public void setUser(String user) {
        this.user = user;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```

* **Activer `@ConfigurationProperties`**

Il faut s'assurer que Spring Boot scanne et active ces classes. On peut le faire via `@EnableConfigurationProperties`
sur une classe de configuration principale ou sur la classe `Application`.

```java
package fr.formation.spring;

import io.github.cdimascio.dotenv.Dotenv;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

// Import de la classe de configuration
import fr.formation.spring.config.DatabaseConfig;

@SpringBootApplication
@EnableConfigurationProperties(DatabaseConfig.class) // Activer la classe
public class DemoApplication {

    public static void main(String[] args) {
        Dotenv dotenv = Dotenv.configure().ignoreIfMissing().load();
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

* **Utiliser la configuration injectée :**

```java
package fr.formation.spring.component;

import fr.formation.spring.config.DatabaseConfig;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct; // Depuis Java 9+

@Component
public class DatabaseConnector {

    private final DatabaseConfig dbConfig;

    // Injection de la configuration via le constructeur (bonne pratique)
    @Autowired
    public DatabaseConnector(DatabaseConfig dbConfig) {
        this.dbConfig = dbConfig;
    }

    @PostConstruct // Méthode appelée après l'initialisation du bean
    public void connect() {
        System.out.println("Connexion à la base de données...");
        System.out.println("URL: " + dbConfig.getUrl());
        System.out.println("Utilisateur: " + dbConfig.getUser());
        // Ne jamais afficher le mot de passe en production !
        // System.out.println("Mot de passe: " + dbConfig.getPassword());
        System.out.println("Connexion simulée établie.");
    }
}
```

### Bonnes Pratiques

1. **Ne Jamais Commiter `.env`** : C'est la règle d'or. Utilisez `.gitignore`.
2. **Utiliser un Fichier `.env.example`** : Créez un fichier `.env.example` (ou `.env.template`) qui liste toutes les
   variables nécessaires mais avec des valeurs vides ou des exemples. Ce fichier *peut* être commité pour guider les
   autres développeurs.
3. **Convention de Nommage** : Utilisez des majuscules et des underscores (`DATABASE_URL`) dans le fichier `.env`, ce
   qui est une convention courante pour les variables d'environnement. Spring Boot est assez intelligent pour mapper
   `DATABASE_URL` à `database.url` dans `@ConfigurationProperties` ou `${database.url}` dans `@Value`.
4. **Gestion des Environnements Multiples** : Pour différents environnements (dev, staging, prod), n'utilisez pas
   plusieurs fichiers `.env` (`.env.dev`, `.env.prod`) gérés par `java-dotenv` car cela complique le déploiement.
   Préférez les mécanismes standards de Spring Boot (profils, variables d'environnement système fournies par la
   plateforme de déploiement comme Docker, Kubernetes, Cloud Foundry, etc.). Le fichier `.env` est surtout utile pour le
   développement local.
5. **Valeurs par Défaut** : Utilisez les valeurs par défaut de Spring Boot (`@Value("${PROPERTY_KEY:defaultValue}")`) ou
   dans la classe `@ConfigurationProperties` pour rendre l'application plus résiliente si une variable n'est pas
   définie.
6. **Chargement Précoce** : Assurez-vous que `Dotenv.load()` est appelé tout au début de la méthode `main`.

### Cas d'Utilisation Typiques

* **Identifiants de Base de Données** : URL, utilisateur, mot de passe pour la base de données locale.
* **Clés d'API** : Clés pour des services tiers (Stripe, SendGrid, Google Maps, etc.).
* **Secrets d'Application** : Clés secrètes pour JWT, configurations de sécurité.
* **URLs de Services Externes** : Adresses de microservices ou d'API externes spécifiques à l'environnement de
  développement.
* **Feature Flags Locaux** : Activer/désactiver des fonctionnalités pour le développement local.

## Exercices Pratiques

---

### **Exercice 1 : Configuration Simple d'un Service Email**

**Objectif** : Configurer l'adresse d'un serveur SMTP et un port via un fichier `.env` et l'afficher dans un service.

1. **Créez/Modifiez votre fichier `.env`** à la racine du projet avec le contenu suivant :
   ```dotenv
   SMTP_HOST=smtp.local.dev
   SMTP_PORT=1025 
   ```
2. **Assurez-vous que `java-dotenv` est configuré** dans votre classe `DemoApplication` (comme montré précédemment).
3. **Créez un nouveau service** `fr.formation.spring.service.NotificationService` qui utilise `@Value` pour injecter
   `SMTP_HOST` et `SMTP_PORT`.
4. **Ajoutez une méthode** dans ce service (par exemple, `sendTestNotification`) qui affiche les valeurs injectées dans
   la console.
5. **Injectez et appelez** ce service depuis un composant ou un `CommandLineRunner` pour vérifier le résultat au
   démarrage.

#### **Correction de l'Exercice 1 :** {collapsible="true"}

* **Fichier `.env` :** (Comme ci-dessus)

* **`DemoApplication.java` :** (Doit inclure `Dotenv.load()` au début de `main`)

* **`NotificationService.java` :**

```java
package fr.formation.spring.service;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import jakarta.annotation.PostConstruct;

@Service
public class NotificationService {

    // Injection depuis .env via les propriétés système
    @Value("${SMTP_HOST}")
    private String smtpHost;

    // Spring peut convertir en Integer si la valeur est un nombre valide
    @Value("${SMTP_PORT}")
    private int smtpPort;

    @PostConstruct // Appel après injection des dépendances
    public void sendTestNotification() {
        System.out.println("--- Configuration Notification ---");
        System.out.println("Serveur SMTP Host: " + smtpHost);
        System.out.println("Serveur SMTP Port: " + smtpPort);
        System.out.println("Prêt à envoyer des notifications (simulation).");
        System.out.println("---------------------------------");
    }
}
```

* **(Optionnel) `AppRunner.java` pour tester au démarrage :**

```java
package fr.formation.spring.runner;

import fr.formation.spring.service.NotificationService;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class AppRunner implements CommandLineRunner {

    private final NotificationService notificationService;

    // Injection du service
    public AppRunner(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @Override
    public void run(String... args) throws Exception {
        // Le @PostConstruct de NotificationService s'exécute déjà,
        // mais on pourrait appeler d'autres méthodes ici si nécessaire.
        System.out.println("AppRunner: NotificationService initialisé.");
    }
}
```

---

### **Exercice 2 : Configuration Groupée avec `@ConfigurationProperties`**

**Objectif** : Regrouper la configuration d'un service de cache (par exemple, Redis) en utilisant
`@ConfigurationProperties`.

1. **Ajoutez les lignes suivantes** à votre fichier `.env` :
   ```dotenv
   CACHE_HOST=localhost
   CACHE_PORT=6379
   CACHE_ENABLED=true
   ```
2. **Créez une classe de configuration** `fr.formation.spring.config.CacheConfig` annotée avec
   `@ConfigurationProperties(prefix = "cache")` pour mapper ces trois propriétés (`host`, `port`, `enabled`). N'oubliez
   pas les getters et setters.
3. **Activez cette configuration** en ajoutant `CacheConfig.class` à l'annotation `@EnableConfigurationProperties` dans
   votre classe `DemoApplication`.
4. **Créez un composant** `fr.formation.spring.component.CacheManager` qui injecte `CacheConfig`.
5. **Dans une méthode `@PostConstruct`** de `CacheManager`, affichez les valeurs de configuration pour vérifier qu'elles
   sont correctement chargées.

#### **Correction de l'Exercice 2 :** {collapsible="true"}

* **Fichier `.env` :** (Contient les lignes `CACHE_HOST`, `CACHE_PORT`, `CACHE_ENABLED`)

* **`CacheConfig.java` :**

```java
package fr.formation.spring.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;

@Configuration // Rendre la classe découvrable comme bean de configuration
@ConfigurationProperties(prefix = "cache") // Lie les propriétés cache.*
public class CacheConfig {

    private String host; // Correspond à CACHE_HOST
    private int port;    // Correspond à CACHE_PORT
    private boolean enabled; // Correspond à CACHE_ENABLED

    // Getters et Setters requis
    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    public int getPort() {
        return port;
    }

    public void setPort(int port) {
        this.port = port;
    }

    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }
}
```

* **`DemoApplication.java` (mise à jour) :**

```java
package fr.formation.spring;

import io.github.cdimascio.dotenv.Dotenv;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

// Importer les deux classes de configuration
import fr.formation.spring.config.DatabaseConfig;
import fr.formation.spring.config.CacheConfig;

@SpringBootApplication
// Activer les deux configurations
@EnableConfigurationProperties({DatabaseConfig.class, CacheConfig.class})
public class DemoApplication {

    public static void main(String[] args) {
        Dotenv dotenv = Dotenv.configure().ignoreIfMissing().load();
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

* **`CacheManager.java` :**

```java
package fr.formation.spring.component;

import fr.formation.spring.config.CacheConfig;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import jakarta.annotation.PostConstruct;

@Component
public class CacheManager {

    private final CacheConfig cacheConfig;

    @Autowired // Injection via constructeur
    public CacheManager(CacheConfig cacheConfig) {
        this.cacheConfig = cacheConfig;
    }

    @PostConstruct
    public void displayCacheConfiguration() {
        System.out.println("--- Configuration Cache ---");
        System.out.println("Cache Hôte: " + cacheConfig.getHost());
        System.out.println("Cache Port: " + cacheConfig.getPort());
        System.out.println("Cache Activé: " + cacheConfig.isEnabled());
        if (cacheConfig.isEnabled()) {
            System.out.println("Initialisation du cache (simulation)...");
        } else {
            System.out.println("Le cache est désactivé.");
        }
        System.out.println("-------------------------");
    }
}
```

---

### Conseils Supplémentaires

* **Priorité des Sources de Configuration** : Rappelez-vous l'ordre de priorité de Spring Boot. Les variables
  d'environnement système et les propriétés système (où `java-dotenv` charge les valeurs) ont une priorité plus élevée
  que `application.properties`. Cela signifie qu'une variable définie dans `.env` écrasera la même variable définie dans
  `application.properties`.
* **Alternatives** : Pour des environnements de production complexes, considérez des solutions plus robustes comme
  Spring Cloud Config Server, HashiCorp Vault, ou les systèmes de gestion de secrets intégrés aux plateformes cloud (AWS
  Secrets Manager, Azure Key Vault, Kubernetes Secrets). Le fichier `.env` reste excellent pour le développement local
  et les configurations simples.
* **Validation** : Pour les classes `@ConfigurationProperties`, ajoutez la dépendance `spring-boot-starter-validation`
  et utilisez les annotations de validation JSR-303 (comme `@NotEmpty`, `@Min`, `@Max`, `@Pattern`) sur les champs pour
  vous assurer que les configurations sont valides au démarrage.

En résumé, bien que Spring Boot n'intègre pas nativement le support des fichiers `.env`, l'utilisation de bibliothèques
comme `java-dotenv` permet d'adopter facilement ce pattern de configuration populaire, particulièrement utile pour la
gestion des secrets et la configuration locale en développement, tout en s'intégrant harmonieusement avec les mécanismes
de configuration standard de Spring Boot comme `@Value` et `@ConfigurationProperties`.