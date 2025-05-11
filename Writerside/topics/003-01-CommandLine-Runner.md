# CommandLine Runner

Dans l'écosystème Spring Boot, `CommandLineRunner` est une interface fonctionnelle simple mais puissante. Elle permet
d'exécuter du code une fois que le `ApplicationContext` de Spring est complètement chargé, mais juste avant que la
méthode `run()` de `SpringApplication` ne se termine. Ceci est particulièrement utile pour des tâches d'initialisation,
de configuration ou de vérification au démarrage de l'application.

**Cas d'usage typiques dans un contexte web :**

* Initialisation de données en base (ex: création d'un utilisateur admin par défaut, chargement de données de
  référence).
* Préchauffage de caches applicatifs (remplissage avec des valeurs par défaut).
* Vérification de la disponibilité de services externes critiques.
* Affichage d'informations de configuration ou d'état au démarrage dans les logs.
* Lancement de tâches de maintenance légères.

Il est important de noter que les `CommandLineRunner` s'exécutent *avant* que le serveur web embarqué (Tomcat, Jetty,
Undertow) ne soit prêt à accepter des requêtes.

L'interface `CommandLineRunner` possède une seule méthode à implémenter :

```java
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

Le paramètre `args` contient les arguments passés à l'application en ligne de commande.

## Implémentation Basique d'un CommandLineRunner

Pour créer un `CommandLineRunner`, il suffit de :

1. Créer une classe qui implémente l'interface `CommandLineRunner`.
2. Annoter cette classe avec `@Component` (ou une de ses spécialisations comme `@Service`, `@Repository`) pour qu'elle
   soit détectée et gérée par Spring.

**Exemple : Un simple message au démarrage**

Structure du projet :

```
fr/
└── formation/
    └── spring/
        ├── SpringWebApplication.java
        └── runner/
            └── FirstStartupRunner.java
```

`SpringWebApplication.java`:

```java
package fr.formation.spring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringWebApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringWebApplication.class, args);
        // À ce point, tous les CommandLineRunner auront été exécutés.
        System.out.println(
                "Application démarrée et prête (après les CommandLineRunner)."
        );
    }
}
```

`FirstStartupRunner.java`:

```java
package fr.formation.spring.runner;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component // Indique à Spring de gérer ce bean
public class FirstStartupRunner implements CommandLineRunner {

    // Utilisation de SLF4J pour le logging
    private static final Logger logger =
            LoggerFactory.getLogger(FirstStartupRunner.class);

    @Override
    public void run(String... args) throws Exception {
        logger.info("===== FirstStartupRunner en cours d'exécution... =====");
        // Simuler une tâche d'initialisation
        Thread.sleep(1000); // Pause d'1 seconde
        logger.info("Tâche de FirstStartupRunner terminée.");
    }
}
```

Lors du lancement de `SpringWebApplication`, vous verrez dans la console (avant le message "Application démarrée et
prête...") :

```
===== FirstStartupRunner en cours d'exécution... =====
Tâche de FirstStartupRunner terminée.
```

## Gestion de Plusieurs CommandLineRunner et leur Ordre

Il est possible d'avoir plusieurs `CommandLineRunner` dans une application. Par défaut, Spring ne garantit pas l'ordre
d'exécution de ces beans. Si l'ordre est important, on peut utiliser :

1. L'annotation `@Order(valeur)` où `valeur` est un entier (plus la valeur est petite, plus la priorité est haute).
2. Implémenter l'interface `org.springframework.core.Ordered`.

**Exemple : Deux runners avec un ordre défini**

`OrderedRunnerOne.java`:

```java
package fr.formation.spring.runner;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1) // S'exécutera en premier (valeur la plus basse)
public class OrderedRunnerOne implements CommandLineRunner {

    private static final Logger logger =
            LoggerFactory.getLogger(OrderedRunnerOne.class);

    @Override
    public void run(String... args) throws Exception {
        logger.info(">>> OrderedRunnerOne (Priorité 1) en cours...");
        // Logique spécifique
        logger.info(">>> OrderedRunnerOne terminé.");
    }
}
```

`OrderedRunnerTwo.java`:

```java
package fr.formation.spring.runner;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(2) // S'exécutera après OrderedRunnerOne
public class OrderedRunnerTwo implements CommandLineRunner {

    private static final Logger logger =
            LoggerFactory.getLogger(OrderedRunnerTwo.class);

    @Override
    public void run(String... args) throws Exception {
        logger.info(">>> OrderedRunnerTwo (Priorité 2) en cours...");
        // Logique spécifique
        logger.info(">>> OrderedRunnerTwo terminé.");
    }
}
```

En exécutant l'application, la sortie montrera `OrderedRunnerOne` avant `OrderedRunnerTwo`.

## Accès aux Arguments de la Ligne de Commande

Les arguments passés à l'application lors de son lancement (ex: `java -jar myapp.jar arg1 --option=valeur`) sont
accessibles via le paramètre `String... args` de la méthode `run()`.

**Exemple : Runner affichant les arguments**

`ArgumentDisplayRunner.java`:

```java
package fr.formation.spring.runner;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Component
@Order(10) // Pour qu'il s'exécute après les autres si présents
public class ArgumentDisplayRunner implements CommandLineRunner {

    private static final Logger logger =
            LoggerFactory.getLogger(ArgumentDisplayRunner.class);

    @Override
    public void run(String... args) throws Exception {
        logger.info("### ArgumentDisplayRunner Démarré ###");
        if (args.length > 0) {
            logger.info("Arguments reçus de la ligne de commande :");
            Arrays.stream(args).forEach(arg -> logger.info("- " + arg));
        } else {
            logger.info("Aucun argument reçu de la ligne de commande.");
        }
        logger.info("### ArgumentDisplayRunner Terminé ###");
    }
}
```

Pour tester, compilez votre application en JAR (`mvn package`) puis exécutez :
`java -jar target/votre-application.jar premierArg --mode=test "un autre argument"`

Vous verrez ces arguments loggés.

## Injection de Dépendances dans un CommandLineRunner

Un `CommandLineRunner` est un bean Spring comme un autre. Il peut donc bénéficier de l'injection de dépendances (
`@Autowired`) pour utiliser d'autres services ou composants de l'application. C'est ici que réside une grande partie de
sa puissance pour les tâches d'initialisation.

**Exemple : Initialisation de données avec un service**

Supposons un service simple `DataInitializationService`.

`DataInitializationService.java`:

```java
package fr.formation.spring.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service // Un bean de service Spring
public class DataInitializationService {

    private static final Logger logger =
            LoggerFactory.getLogger(DataInitializationService.class);

    public void initializeDefaultUser() {
        // Simule la création d'un utilisateur admin en base
        logger.info("Vérification/création de l'utilisateur admin par défaut...");
        // Logique de connexion à la base, vérification, création...
        // Par exemple: userRepository.findByUsername("admin").orElseGet(() -> {
        //    User admin = new User("admin", passwordEncoder.encode("password"));
        //    return userRepository.save(admin);
        // });
        logger.info("Utilisateur admin initialisé.");
    }

    public void loadReferenceData() {
        // Simule le chargement de données de référence
        logger.info("Chargement des données de référence (ex: pays, devises)...");
        // Logique de chargement...
        logger.info("Données de référence chargées.");
    }
}
```

`DatabaseInitializerRunner.java`:

```java
package fr.formation.spring.runner;

import fr.formation.spring.service.DataInitializationService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(5) // Ordre intermédiaire
public class DatabaseInitializerRunner implements CommandLineRunner {

    private static final Logger logger =
            LoggerFactory.getLogger(DatabaseInitializerRunner.class);

    // Injection du service
    private final DataInitializationService dataInitService;

    @Autowired // L'injection peut se faire via constructeur (recommandé)
    public DatabaseInitializerRunner(
            DataInitializationService dataInitService
    ) {
        this.dataInitService = dataInitService;
    }

    @Override
    public void run(String... args) throws Exception {
        logger.info("--- DatabaseInitializerRunner Démarré ---");

        dataInitService.initializeDefaultUser();
        dataInitService.loadReferenceData();

        logger.info("--- DatabaseInitializerRunner Terminé ---");
    }
}
```

Ce runner utilisera le `DataInitializationService` pour effectuer ses tâches. C'est une pratique courante pour séparer
les préoccupations : le runner orchestre, le service exécute la logique métier.

## Cas d'Utilisation Spécifiques au Contexte Web

Même si une application est "web" (destinée à servir des pages HTML, par exemple avec Thymeleaf), certaines tâches au
démarrage sont cruciales.

* **Initialisation de base de données :** S'assurer que les tables minimales existent, que des données de
  lookup (listes de pays, types de produits) sont présentes, qu'un utilisateur administrateur par défaut est créé.
* **Préchauffage de cache :** Si l'application utilise un cache (EhCache, Caffeine, Redis), le `CommandLineRunner` peut
  charger les données les plus fréquemment accédées pour éviter un premier accès lent pour les premiers utilisateurs.
  ```java
  // Dans un CacheWarmerRunner.java
  // @Autowired MyCacheService cacheService;
  // cacheService.loadFrequentlyAccessedProducts();
  // cacheService.loadRegionalSettings();
  ```
* **Vérification de configuration et de connectivité :** Avant de démarrer le serveur web, vérifier que les bases de
  données sont accessibles, que les services externes répondent, ou que des fichiers de configuration essentiels sont
  présents.
  ```java
  // Dans un ConfigCheckerRunner.java
  // try {
  //    dataSource.getConnection().close();
  //    logger.info("Connexion à la base de données réussie.");
  // } catch (SQLException e) {
  //    logger.error("Échec de la connexion à la base de données!", e);
  //    // On pourrait vouloir arrêter l'application ici
  //    // System.exit(1); ou lancer une exception qui arrête le boot
  // }
  ```
* **Nettoyage de ressources temporaires :** Supprimer des fichiers temporaires d'une exécution précédente.
* **Enregistrement d'informations au démarrage :** Logger des informations importantes sur l'environnement, les profils
  actifs, les versions des composants clés.

### Bonnes Pratiques et Conseils

1. **Responsabilité Unique :** Chaque `CommandLineRunner` devrait avoir une responsabilité claire et unique. Préférez
   plusieurs petits runners bien ordonnés à un seul runner monolithique.
2. **Utiliser `@Order` :** Si l'ordre d'exécution compte, utilisez systématiquement `@Order`. Donnez des valeurs
   espacées (ex: 10, 20, 30) pour faciliter l'insertion de nouveaux runners entre les existants.
3. **Gestion des Exceptions :** La méthode `run` peut lever une `Exception`. Si une exception non interceptée est levée,
   elle empêchera le démarrage complet de l'application. Gérez les exceptions de manière appropriée : loggez-les, et
   décidez si l'application peut continuer ou doit s'arrêter.
4. **Idempotence :** Si un runner effectue des modifications (ex: insertion en base), essayez de le rendre idempotent.
   C'est-à-dire que s'il est exécuté plusieurs fois, le résultat final reste le même (ex: vérifier si l'admin existe
   avant de le créer).
5. **Injection par Constructeur :** Préférez l'injection de dépendances par constructeur pour les runners. Cela rend les
   dépendances explicites et facilite les tests.
6. **Logging :** Utilisez abondamment le logging (`SLF4J` est un bon choix) pour tracer ce que font vos runners. C'est
   crucial pour le débogage et la compréhension du processus de démarrage.
7. **Profils Spring (`@Profile`) :** Vous pouvez conditionner l'exécution d'un `CommandLineRunner` à un profil Spring
   actif. Par exemple, un runner d'initialisation de données de test ne s'exécutera que si le profil "dev" ou "test" est
   actif.
   ```java
   // @Component
   // @Profile("dev")
   // public class DevDataInitializerRunner implements CommandLineRunner { ... }
   ```
8. **Alternative `ApplicationRunner` :** Spring Boot offre aussi `ApplicationRunner`. La différence principale est que
   sa méthode `run` prend un `ApplicationArguments` en paramètre, offrant un accès plus structuré aux arguments de la
   ligne de commande (options, non-options). Pour la plupart des cas d'usage d'initialisation, `CommandLineRunner` est
   suffisant.

## Exercices Pratiques

### Exercice 1 : Runner de Bienvenue Personnalisé

**Énoncé :**
Créez un `CommandLineRunner` nommé `WelcomeMessageRunner` qui affiche le message suivant dans les logs au démarrage de
l'application : "Bienvenue dans l'application [NomDeLApplication] ! Démarrage en cours...".
Le nom de l'application doit être récupéré depuis les propriétés de l'application (`application.properties` ou
`application.yml`) via la clé `spring.application.name`.

#### **Correction Exercice 1** {collapsible="true"}

1. Ajoutez dans `src/main/resources/application.properties`:
   ```properties
   spring.application.name=MonAppWebSuperCool
   ```

2. Créez `WelcomeMessageRunner.java`:
   ```java
   package fr.formation.spring.runner;

   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.boot.CommandLineRunner;
   import org.springframework.stereotype.Component;

   @Component
   public class WelcomeMessageRunner implements CommandLineRunner {

       private static final Logger logger = 
           LoggerFactory.getLogger(WelcomeMessageRunner.class);

       // Injection de la valeur depuis application.properties
       @Value("${spring.application.name}")
       private String applicationName;

       @Override
       public void run(String... args) throws Exception {
           logger.info(
               "Bienvenue dans l'application {} ! Démarrage en cours...", 
               applicationName
           );
       }
   }
   ```

### Exercice 2 : Vérificateur de Version Java

**Énoncé :**
Créez un `CommandLineRunner` nommé `JavaVersionCheckerRunner` qui vérifie la version de Java utilisée pour exécuter
l'application.

- Si la version est Java 17 ou supérieure, il logue un message d'information : "Version Java compatible (Java X.Y.Z)".
- Si la version est inférieure à Java 17, il logue un message d'avertissement : "ATTENTION : Version Java X.Y.Z
  obsolète. Java 17+ recommandé." et fait échouer le démarrage de l'application en lançant une `IllegalStateException`.
  Utilisez `@Order` pour que ce runner s'exécute en tout premier.

#### **Correction Exercice 2** {collapsible="true"}

Créez `JavaVersionCheckerRunner.java`:

```java
package fr.formation.spring.runner;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(Integer.MIN_VALUE) // Priorité la plus haute
public class JavaVersionCheckerRunner implements CommandLineRunner {

    private static final Logger logger =
            LoggerFactory.getLogger(JavaVersionCheckerRunner.class);

    // Version minimale requise (ex: 17)
    private static final int MINIMUM_JAVA_VERSION = 17;

    @Override
    public void run(String... args) throws Exception {
        logger.info("--- Vérification de la version Java ---");
        String javaVersion = System.getProperty("java.version");
        logger.info("Version Java détectée : {}", javaVersion);

        // Analyse simple de la version (pour l'exercice)
        // Une analyse plus robuste utiliserait Runtime.version() pour Java 9+
        int majorVersion;
        try {
            String[] parts = javaVersion.split("\\.");
            if (parts[0].equals("1")) { // Format 1.x.y (Java 8 et moins)
                majorVersion = Integer.parseInt(parts[1]);
            } else { // Format X.Y.Z (Java 9 et plus)
                majorVersion = Integer.parseInt(parts[0]);
            }
        } catch (NumberFormatException | ArrayIndexOutOfBoundsException e) {
            logger.error("Impossible de parser la version de Java : {}",
                    javaVersion);
            throw new IllegalStateException(
                    "Format de version Java non reconnu."
            );
        }

        if (majorVersion >= MINIMUM_JAVA_VERSION) {
            logger.info(
                    "Version Java compatible (Java {}).",
                    javaVersion
            );
        } else {
            String errorMessage = String.format(
                    "ATTENTION : Version Java %s obsolète. Java %d+ requis.",
                    javaVersion,
                    MINIMUM_JAVA_VERSION
            );
            logger.warn(errorMessage);
            // Fait échouer le démarrage de l'application
            throw new IllegalStateException(errorMessage);
        }
        logger.info("--- Vérification de la version Java terminée ---");
    }
}
```

**Note pour l'exercice 2 :** Pour tester la condition d'échec, vous devriez compiler et exécuter le JAR avec une JVM
plus ancienne (ex: Java 11). Dans un IDE, la JVM du projet est généralement celle configurée pour le projet.

### Exercice 3 : Initialisation de Données de Référence (Simulé)

**Énoncé :**

1. Créez une classe de service `ReferenceDataService` avec une méthode `loadCountries()` qui simule le chargement d'une
   liste de pays (loge simplement "Chargement de la liste des pays..." et "Liste des pays chargée.").
2. Créez un `CommandLineRunner` nommé `ReferenceDataLoaderRunner` qui :
    * Injecte `ReferenceDataService`.
    * Appelle la méthode `loadCountries()` du service.
    * Prend un argument optionnel en ligne de commande `--force-reload`.
    * Si `--force-reload` est présent, logue "Forçage du rechargement des données de référence." avant d'appeler
      `loadCountries()`.
    * Donnez-lui un `@Order` pour qu'il s'exécute après le `JavaVersionCheckerRunner` (si présent) mais avant d'autres
      runners potentiels (ex: `@Order(-2147483647 + 10)` ou une valeur positive si les autres ont des valeurs plus
      grandes).

#### **Correction Exercice 3** {collapsible="true"}

1. `ReferenceDataService.java`:
   ```java
   package fr.formation.spring.service;

   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.stereotype.Service;

   @Service
   public class ReferenceDataService {

       private static final Logger logger = 
           LoggerFactory.getLogger(ReferenceDataService.class);

       public void loadCountries() {
           logger.info("Chargement de la liste des pays...");
           // Simuler une opération qui prend du temps
           try {
               Thread.sleep(500); // 0.5 seconde
           } catch (InterruptedException e) {
               Thread.currentThread().interrupt();
           }
           logger.info("Liste des pays chargée.");
       }
   }
   ```

2. `ReferenceDataLoaderRunner.java`:
   ```java
   package fr.formation.spring.runner;

   import fr.formation.spring.service.ReferenceDataService;
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.CommandLineRunner;
   import org.springframework.core.annotation.Order;
   import org.springframework.stereotype.Component;
   import java.util.Arrays;

   @Component
   @Order(10) // S'assure qu'il s'exécute après les vérifs critiques
   public class ReferenceDataLoaderRunner implements CommandLineRunner {

       private static final Logger logger = 
           LoggerFactory.getLogger(ReferenceDataLoaderRunner.class);

       private final ReferenceDataService referenceDataService;

       @Autowired
       public ReferenceDataLoaderRunner(
           ReferenceDataService referenceDataService
       ) {
           this.referenceDataService = referenceDataService;
       }

       @Override
       public void run(String... args) throws Exception {
           logger.info("--- ReferenceDataLoaderRunner Démarré ---");

           boolean forceReload = Arrays.stream(args)
                                       .anyMatch("--force-reload"::equals);

           if (forceReload) {
               logger.info(
                   "Forçage du rechargement des données de référence."
               );
           } else {
               logger.info(
                   "Vérification des données de référence (pas de forçage)."
               );
           }

           // Ici, on pourrait ajouter une logique pour ne charger
           // que si nécessaire, sauf si forceReload est vrai.
           referenceDataService.loadCountries();
           
           logger.info("--- ReferenceDataLoaderRunner Terminé ---");
       }
   }
   ```

Pour tester avec l'argument : `java -jar target/votre-application.jar --force-reload`

### Conclusion

`CommandLineRunner` est un outil précieux dans Spring Boot pour exécuter des logiques d'initialisation ou de
configuration au démarrage d'une application web. En l'utilisant judicieusement, en respectant le principe de
responsabilité unique et en gérant l'ordre d'exécution, les développeurs peuvent s'assurer que leur application démarre
dans un état cohérent et prêt à l'emploi. Son intégration avec le système d'injection de dépendances de Spring le rend
particulièrement flexible et puissant.
