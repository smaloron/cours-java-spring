# La configuration

Spring Boot vise à simplifier le développement d'applications Spring autonomes et prêtes pour la production. Un aspect
clé de cette simplification est la gestion de la configuration. Plutôt que de nécessiter des fichiers XML complexes,
Spring Boot favorise une approche basée sur les conventions et permet de configurer l'application de manière flexible
via des fichiers de propriétés, des fichiers YAML, des variables d'environnement, des arguments de ligne de commande,
etc.

Ce support se concentre sur les méthodes les plus courantes : les fichiers `application.properties` et
`application.yml`, ainsi que sur les bonnes pratiques associées.

## Les Fichiers `application.properties`

### Présentation

Le fichier `application.properties` est le moyen le plus classique et le plus simple de fournir une configuration
externe à une application Spring Boot. Il se trouve généralement dans le répertoire `src/main/resources`.

La syntaxe est basée sur des paires clé-valeur, où chaque propriété est définie sur une nouvelle ligne.

```properties
# Configuration du port du serveur
server.port=8080
# Configuration personnalisée de l'application
app.name=Mon Application Spring
app.version=1.0.0
# Configuration de la base de données (exemple)
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
```

Les clés peuvent être hiérarchiques, en utilisant le point (`.`) comme séparateur. Spring Boot fournit de nombreuses
propriétés prédéfinies (comme `server.port` ou `spring.datasource.url`) que l'on peut surcharger.

### Accéder aux Valeurs de Configuration

Il existe principalement deux façons d'accéder aux valeurs définies dans `application.properties` :

#### Annotation `@Value`

L'annotation `@Value` permet d'injecter directement la valeur d'une propriété dans un champ d'un bean Spring.

```java
package fr.formation.spring.services;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class AppInfoService {

    // Injection de la valeur de la propriété 'app.name'
    @Value("${app.name}")
    private String applicationName;

    // Injection avec une valeur par défaut si la propriété n'existe pas
    @Value("${app.description:Application par défaut}")
    private String applicationDescription;

    // Injection du port du serveur
    @Value("${server.port}")
    private int serverPort;

    public void displayAppInfo() {
        System.out.println("Nom de l'application : " + applicationName);
        System.out.println("Description : " + applicationDescription);
        System.out.println("Port du serveur : " + serverPort);
    }
}
```

**Avantages :** Simple pour injecter quelques propriétés isolées.
**Inconvénients :**

* Peut devenir verbeux si de nombreuses propriétés sont nécessaires.
* Moins de sécurité de type (erreurs de frappe dans la clé non détectées à la compilation).
* Ne gère pas bien les structures de configuration complexes.

#### Annotation `@ConfigurationProperties` (Approche Type-Safe)

Cette approche est plus robuste et recommandée pour regrouper des propriétés liées. Elle mappe des blocs de propriétés
sur un objet Java (POJO).

**Étape 1 : Créer une classe de configuration**

```java
package fr.formation.spring.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

// Mappe toutes les propriétés commençant par 'app'
@Component // Rend ce bean géré par Spring
@ConfigurationProperties(prefix = "app")
public class AppProperties {

    private String name; // Correspond à app.name
    private String version; // Correspond à app.version
    private String description = "Description par défaut"; // Valeur par défaut

    // Getters et Setters sont nécessaires pour le binding
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getVersion() {
        return version;
    }

    public void setVersion(String version) {
        this.version = version;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }
}
```

**Étape 2 : Injecter et utiliser la classe de configuration**

```java
package fr.formation.spring.services;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import fr.formation.spring.config.AppProperties; // Import

@Service
public class AnotherAppInfoService {

    private final AppProperties appProperties;

    // Injection via le constructeur (recommandé)
    @Autowired
    public AnotherAppInfoService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }

    public void displayAppDetails() {
        System.out.println("--- Via @ConfigurationProperties ---");
        System.out.println("Nom App : " + appProperties.getName());
        System.out.println("Version App : " + appProperties.getVersion());
        System.out.println("Description App : " + appProperties.getDescription());
    }
}
```

**Avantages :**

* Type-safe (sécurité des types).
* Regroupement logique des propriétés.
* Meilleure lisibilité et maintenabilité.
* Supporte la validation (JSR-303/JSR-349) en ajoutant des annotations (`@NotNull`, `@Min`, etc.) sur les champs et la
  dépendance `spring-boot-starter-validation`.

## Les Fichiers `application.yml` ou `application.yaml`

### Présentation {id="pr-sentation_1"}

YAML (YAML Ain't Markup Language) est un format de sérialisation de données souvent utilisé pour les fichiers de
configuration en raison de sa lisibilité humaine. Spring Boot le supporte nativement (via la librairie SnakeYAML,
incluse dans `spring-boot-starter`).

Le fichier `application.yml` (ou `application.yaml`) se place également dans `src/main/resources`. S'il est présent en
même temps qu'`application.properties`, `application.yml` prend généralement la priorité pour les propriétés qui y sont
définies.

La syntaxe YAML est basée sur l'indentation (espaces, pas de tabulations !) pour représenter la structure hiérarchique.

```yaml
# Configuration du port du serveur
server:
  port: 8081 # Port différent de l'exemple .properties

# Configuration personnalisée de l'application
app:
  name: Mon Application YAML
  version: 1.1.0
  description: Application configurée via YAML
  # Structure imbriquée
  owner:
    name: Formation Team
    email: contact@formation.fr

# Configuration de la base de données (exemple)
spring:
  datasource:
    url: jdbc:h2:mem:yamldb
    driverClassName: org.h2.Driver
    username: sa
    password:

# Listes en YAML
app:
  # ... autres propriétés app ...
  features:
    - Feature A
    - Feature B
    - Feature C
```

### Accéder aux Valeurs de Configuration {id="acc-der-aux-valeurs-de-configuration_1"}

Les mécanismes d'accès (`@Value` et `@ConfigurationProperties`) fonctionnent **exactement de la même manière** avec les
fichiers YAML qu'avec les fichiers `properties`.

#### Exemple avec `@Value`

```java
// ... dans une classe Service ou Component ...

// Accès à une propriété simple
@Value("${app.name}")
private String applicationName;

// Accès à une propriété imbriquée
@Value("${app.owner.name}")
private String ownerName;

// Accès à un élément de liste (moins courant avec @Value)
@Value("${app.features[0]}")
private String firstFeature;
```

#### Exemple avec `@ConfigurationProperties`

Il faut adapter la classe Java pour refléter la structure YAML.

```java
package fr.formation.spring.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.List; // Pour la liste
import java.util.ArrayList; // Pour initialiser

@Component
@ConfigurationProperties(prefix = "app")
public class AppYamlProperties {

    private String name;
    private String version;
    private String description;
    private OwnerProperties owner; // Objet imbriqué
    private List<String> features = new ArrayList<>(); // Liste

    // Classe interne statique pour la structure imbriquée 'owner'
    public static class OwnerProperties {
        private String name;
        private String email;

        // Getters et Setters pour OwnerProperties
        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getEmail() {
            return email;
        }

        public void setEmail(String email) {
            this.email = email;
        }
    }

    // Getters et Setters pour AppYamlProperties
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    // ... autres getters/setters ...
    public OwnerProperties getOwner() {
        return owner;
    }

    public void setOwner(OwnerProperties owner) {
        this.owner = owner;
    }

    public List<String> getFeatures() {
        return features;
    }

    public void setFeatures(List<String> features) {
        this.features = features;
    }
}
```

L'injection et l'utilisation de `AppYamlProperties` se font de la même manière que pour `AppProperties`.

### Comparaison `.properties` vs `.yml`

* **`.properties`** :
    * Syntaxe simple clé=valeur.
    * Moins lisible pour les configurations hiérarchiques complexes.
    * Pas de support natif pour les listes (nécessite indexation `list[0]`, `list[1]`, etc.).
    * Très répandu dans l'écosystème Java historique.
* **`.yml`** :
    * Syntaxe basée sur l'indentation, plus lisible pour les structures.
    * Excellent support pour les listes et les objets imbriqués.
    * Moins verbeux.
    * Sensible aux erreurs d'indentation (utiliser des espaces !).
    * Format populaire dans les environnements Cloud Native (Kubernetes, Docker Compose).

**Conseil :** Pour des configurations simples, `.properties` suffit. Pour des configurations plus structurées ou
complexes, `.yml` est souvent préférable pour sa lisibilité.

## Gestion des Profils

Il est rare qu'une application utilise la même configuration dans tous les environnements (développement, test,
production). Spring Boot gère cela grâce aux **profils**.

Un profil est un identifiant (ex: `dev`, `test`, `prod`) qui permet d'activer un ensemble spécifique de configurations.

### Définir des Configurations par Profil

On peut créer des fichiers de configuration spécifiques à un profil en suivant la convention de nommage
`application-{profil}.properties` ou `application-{profil}.yml`.

* `application.properties` (ou `.yml`) : Contient la configuration commune ou par défaut.
* `application-dev.properties` (ou `.yml`) : Contient les surcharges ou ajouts pour l'environnement de développement.
* `application-prod.properties` (ou `.yml`) : Contient les surcharges ou ajouts pour l'environnement de production.

Les propriétés définies dans un fichier de profil spécifique (`application-dev.properties`) **écrasent** celles définies
dans le fichier de base (`application.properties`).

**Exemple :**

`application.properties`:

```properties
server.port=8080
app.name=Generic App
# Base de données par défaut (mémoire)
spring.datasource.url=jdbc:h2:mem:defaultdb
```

`application-dev.properties`:

```properties
# Surcharge le port pour le dev
server.port=9090
# Utilise une base de données de dev spécifique
spring.datasource.url=jdbc:h2:file:./devdb
logging.level.fr.formation.spring=DEBUG # Log plus verbeux en dev
```

`application-prod.properties`:

```properties
# Utilise une base de données de production (exemple PostgreSQL)
spring.datasource.url=jdbc:postgresql://prod-db-host:5432/prod_db
spring.datasource.username=prod_user
# Mot de passe à gérer de manière sécurisée (voir bonnes pratiques)
spring.datasource.password=${DB_PASSWORD} # Exemple via variable d'environnement
logging.level.fr.formation.spring=INFO # Log moins verbeux en prod
```

### Activer un Profil

Il existe plusieurs manières d'activer un ou plusieurs profils :

1. **Via `application.properties` / `application.yml` :**
   ```properties
   # Dans application.properties
   spring.profiles.active=dev
   ```
   ```yaml
   # Dans application.yml
   spring:
     profiles:
       active: dev
   ```
   *Note : Définir le profil actif dans le fichier de configuration principal est utile pour le développement, mais
   moins flexible pour les déploiements.*

2. **Via Variable d'Environnement :** (Méthode courante pour les déploiements)
   ```bash
   export SPRING_PROFILES_ACTIVE=prod
   java -jar my-app.jar
   ```

3. **Via Argument de Ligne de Commande :** (Priorité la plus haute)
   ```bash
   java -jar my-app.jar --spring.profiles.active=prod,email-notifications
   ```
   On peut activer plusieurs profils en les séparant par une virgule.

4. **Via l'IDE (Exemple : IntelliJ IDEA) :** Dans la configuration de lancement de l'application, on peut spécifier le
   profil actif dans les "VM options" (`-Dspring.profiles.active=dev`) ou dans les "Program arguments" (
   `--spring.profiles.active=dev`).

## Bonnes Pratiques de Configuration

1. **Externaliser la Configuration :** Ne jamais coder en dur des valeurs de configuration (URL de base de données, clés
   API, ports, etc.) dans le code source. Utiliser les fichiers `properties`/`yml` ou d'autres mécanismes externes.
2. **Utiliser les Profils :** Séparer systématiquement les configurations par environnement (`dev`, `test`, `staging`,
   `prod`).
3. **Sécuriser les Informations Sensibles :** Ne **jamais** commiter de mots de passe, clés API ou autres secrets dans
   le dépôt de code (Git). Utiliser :
    * Des variables d'environnement.
    * Des fichiers de configuration externes non versionnés.
    * Des solutions de gestion de secrets (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault).
    * Spring Cloud Config Server avec backend sécurisé.
4. **Préferer `@ConfigurationProperties` :** Pour les configurations structurées, utiliser `@ConfigurationProperties`
   pour la sécurité de type, la validation et la lisibilité.
5. **Nommage Cohérent :** Utiliser des préfixes clairs et cohérents pour les propriétés personnalisées (ex: `app.*`,
   `email.service.*`). Le format `kebab-case` (`mon-app.ma-propriete`) est courant dans `properties`/`yml` et se mappe
   automatiquement en `camelCase` (`monApp.maPropriete`) dans les classes `@ConfigurationProperties`.
6. **Valeurs par Défaut Sensées :** Fournir des valeurs par défaut raisonnables pour les propriétés, surtout pour
   l'environnement de développement.
7. **Documentation :** Documenter les propriétés de configuration personnalisées, par exemple via JavaDoc sur les
   classes `@ConfigurationProperties` ou dans un fichier `README.md`. L'utilisation du processeur d'annotations
   `spring-boot-configuration-processor` génère des métadonnées utiles pour l'autocomplétion dans les IDEs.
8. **Validation :** Ajouter la dépendance `spring-boot-starter-validation` et utiliser les annotations JSR-303 (
   `@NotNull`, `@Min`, `@Max`, `@Pattern`, etc.) sur les champs des classes `@ConfigurationProperties` pour valider la
   configuration au démarrage de l'application.

## Exercices Pratiques

### Exercice 1 : Configuration Simple avec `@Value`

**Énoncé :**

1. Créez un nouveau projet Spring Boot (via start.spring.io ou votre IDE) avec la dépendance "Web".
2. Dans `application.properties`, définissez les propriétés suivantes :
    * `welcome.message=Bonjour et bienvenue !`
    * `admin.email=admin@example.com`
3. Créez un `RestController` nommé `GreetingController`.
4. Injectez les valeurs des propriétés `welcome.message` et `admin.email` dans ce contrôleur en utilisant l'annotation
   `@Value`.
5. Créez un endpoint GET `/greeting` qui retourne la valeur de `welcome.message`.
6. Créez un endpoint GET `/admin-info` qui retourne la valeur de `admin.email`.
7. Testez vos endpoints.

#### Correction Exercice 1 {collapsible="true"}

`src/main/resources/application.properties`:

```properties
welcome.message=Bonjour et bienvenue !
admin.email=admin@example.com
server.port=8080 # Port optionnel
```

`src/main/java/fr/formation/spring/controllers/GreetingController.java`:

```java
package fr.formation.spring.controllers;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class GreetingController {

    // Injection du message d'accueil
    @Value("${welcome.message}")
    private String welcomeMessage;

    // Injection de l'email admin
    @Value("${admin.email}")
    private String adminEmail;

    // Endpoint pour le message d'accueil
    @GetMapping("/greeting")
    public String getGreeting() {
        return this.welcomeMessage;
    }

    // Endpoint pour l'info admin
    @GetMapping("/admin-info")
    public String getAdminInfo() {
        return "Email Admin : " + this.adminEmail;
    }
}
```

`src/main/java/fr/formation/spring/DemoApplication.java`: (Classique)

```java
package fr.formation.spring;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

*Pour tester, lancez l'application et accédez à `http://localhost:8080/greeting` et `http://localhost:8080/admin-info`
dans votre navigateur.*

---

### Exercice 2 : Configuration Structurée avec `@ConfigurationProperties` et YAML

**Énoncé :**

1. Supprimez (ou commentez) les propriétés de l'exercice 1 dans `application.properties`.
2. Créez un fichier `application.yml` dans `src/main/resources`.
3. Définissez une structure de configuration pour un service de notification :
   ```yaml
   notification:
     service:
       enabled: true
       default-sender: notification@myapp.com
       retry-attempts: 3
       supported-channels:
         - email
         - sms
   ```
4. Créez une classe de configuration `NotificationProperties` dans le package `fr.formation.spring.config` pour mapper
   cette structure en utilisant `@ConfigurationProperties(prefix = "notification.service")`. N'oubliez pas les
   getters/setters.
5. Assurez-vous que cette classe est scannée par Spring (utilisez `@Component` ou `@EnableConfigurationProperties`).
6. Créez un service `NotificationService` dans le package `fr.formation.spring.services`.
7. Injectez `NotificationProperties` dans `NotificationService` via le constructeur.
8. Créez une méthode dans `NotificationService` qui affiche les détails de configuration chargés (enabled, sender,
   attempts, channels).
9. Injectez `NotificationService` dans `GreetingController` et ajoutez un endpoint GET `/notification-config` qui
   appelle la méthode d'affichage du service et retourne un message simple (ex: "Config affichée dans la console").
10. Testez.

#### Correction Exercice 2 {collapsible="true"}

`src/main/resources/application.yml`:

```yaml
notification:
  service:
    enabled: true
    default-sender: notification@myapp.com # Note: kebab-case
    retry-attempts: 3 # Note: kebab-case
    supported-channels: # Note: kebab-case
      - email
      - sms

server:
  port: 8080 # Optionnel
```

`src/main/java/fr/formation/spring/config/NotificationProperties.java`:

```java
package fr.formation.spring.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;

@Component // Rend le bean géré par Spring
@ConfigurationProperties(prefix = "notification.service")
public class NotificationProperties {

    private boolean enabled;
    // Mappage automatique de kebab-case vers camelCase
    private String defaultSender;
    private int retryAttempts;
    private List<String> supportedChannels = new ArrayList<>();

    // --- Getters et Setters ---
    public boolean isEnabled() {
        return enabled;
    }

    public void setEnabled(boolean enabled) {
        this.enabled = enabled;
    }

    public String getDefaultSender() {
        return defaultSender;
    }

    public void setDefaultSender(String defaultSender) {
        this.defaultSender = defaultSender;
    }

    public int getRetryAttempts() {
        return retryAttempts;
    }

    public void setRetryAttempts(int retryAttempts) {
        this.retryAttempts = retryAttempts;
    }

    public List<String> getSupportedChannels() {
        return supportedChannels;
    }

    public void setSupportedChannels(List<String> supportedChannels) {
        this.supportedChannels = supportedChannels;
    }
}
```

`src/main/java/fr/formation/spring/services/NotificationService.java`:

```java
package fr.formation.spring.services;

import org.springframework.stereotype.Service;
import fr.formation.spring.config.NotificationProperties; // Import
import jakarta.annotation.PostConstruct; // Pour l'affichage au démarrage

@Service
public class NotificationService {

    private final NotificationProperties notificationProperties;

    // Injection par constructeur
    public NotificationService(NotificationProperties notificationProperties) {
        this.notificationProperties = notificationProperties;
    }

    // Méthode pour afficher la configuration
    public void displayConfig() {
        System.out.println("--- Configuration Notification ---");
        System.out.println("Service Activé : " + notificationProperties.isEnabled());
        System.out.println("Expéditeur par défaut : " + notificationProperties.getDefaultSender());
        System.out.println("Tentatives : " + notificationProperties.getRetryAttempts());
        System.out.println("Canaux supportés : " + notificationProperties.getSupportedChannels());
    }

    // Optionnel: Affiche la config au démarrage pour vérification facile
    @PostConstruct
    public void init() {
        displayConfig();
    }
}
```

`src/main/java/fr/formation/spring/controllers/GreetingController.java`: (Ajouter l'injection et l'endpoint)

```java
package fr.formation.spring.controllers;

// ... autres imports ...

import fr.formation.spring.services.NotificationService; // Import
import org.springframework.beans.factory.annotation.Autowired; // Import

@RestController
public class GreetingController {

    // ... (propriétés @Value de l'exo 1 peuvent rester ou être enlevées)

    private final NotificationService notificationService;

    // Injection de NotificationService (et autres si nécessaire)
    @Autowired
    public GreetingController(NotificationService notificationService) {
        this.notificationService = notificationService;
        // Initialiser les autres champs injectés si besoin
    }

    // ... (endpoints /greeting et /admin-info) ...

    // Nouvel endpoint pour afficher la config de notification
    @GetMapping("/notification-config")
    public String showNotificationConfig() {
        // Affiche dans la console serveur via le service
        this.notificationService.displayConfig();
        return "Configuration de notification affichée dans la console.";
    }
}
```

*Pour tester, lancez l'application. La configuration devrait s'afficher dans la console au démarrage (à cause
de `@PostConstruct`). Accédez ensuite à `http://localhost:8080/notification-config`. La configuration sera de nouveau
affichée dans la console.*

---

### Exercice 3 : Utilisation des Profils

**Énoncé :**

1. En partant de l'exercice 2, gardez `application.yml` comme fichier de base.
2. Créez un fichier `application-dev.yml`. Dedans, surchargez les propriétés suivantes :
    * `notification.service.default-sender` à `dev-sender@myapp.com`
    * Ajoutez une nouvelle propriété : `logging.level.fr.formation.spring=DEBUG`
3. Créez un fichier `application-prod.yml`. Dedans, surchargez :
    * `notification.service.enabled` à `false` (désactivé en prod pour cet exemple)
    * `notification.service.default-sender` à `prod-sender@myapp.com`
    * `notification.service.retry-attempts` à `5`
    * Ajoutez `logging.level.fr.formation.spring=WARN`
4. Modifiez la configuration de lancement de votre application pour activer le profil `dev`. Lancez l'application et
   vérifiez via l'endpoint `/notification-config` (et la console) que les valeurs de `dev` sont utilisées.
5. Arrêtez l'application. Modifiez la configuration de lancement pour activer le profil `prod`. Lancez et vérifiez que
   les valeurs de `prod` sont utilisées.
6. Que se passe-t-il si vous lancez l'application sans profil actif ? Quelles valeurs sont utilisées ?

#### Correction Exercice 3 {collapsible="true"}

`src/main/resources/application.yml`: (Fichier de base)

```yaml
notification:
  service:
    enabled: true
    default-sender: default-sender@myapp.com # Valeur par défaut
    retry-attempts: 3
    supported-channels:
      - email
      - sms

server:
  port: 8080

# Niveau de log par défaut
logging:
  level:
    fr.formation.spring: INFO
```

`src/main/resources/application-dev.yml`:

```yaml
notification:
  service:
    default-sender: dev-sender@myapp.com # Surcharge

logging:
  level:
    fr.formation.spring: DEBUG # Surcharge
```

*(Note : `enabled`, `retry-attempts`, `supported-channels` et `server.port` héritent des valeurs de `application.yml`)*

`src/main/resources/application-prod.yml`:

```yaml
notification:
  service:
    enabled: false # Surcharge
    default-sender: prod-sender@myapp.com # Surcharge
    retry-attempts: 5 # Surcharge

logging:
  level:
    fr.formation.spring: WARN # Surcharge
```

*(Note : `supported-channels` et `server.port` héritent de `application.yml`)*

**Activation des profils :**

* **Via IDE (Exemple IntelliJ) :** Edit Configurations -> Modify options -> Add VM options -> Mettre
  `-Dspring.profiles.active=dev` (ou `prod`).
* **Via Ligne de commande :** `java -jar target/votre-app.jar --spring.profiles.active=dev` (ou `prod`).

**Résultats attendus :**

* **Avec profil `dev` actif :**
    * Enabled: `true` (hérité)
    * Sender: `dev-sender@myapp.com` (surchargé par dev)
    * Attempts: `3` (hérité)
    * Channels: `[email, sms]` (hérité)
    * Logs `fr.formation.spring`: `DEBUG` (surchargé par dev)
* **Avec profil `prod` actif :**
    * Enabled: `false` (surchargé par prod)
    * Sender: `prod-sender@myapp.com` (surchargé par prod)
    * Attempts: `5` (surchargé par prod)
    * Channels: `[email, sms]` (hérité)
    * Logs `fr.formation.spring`: `WARN` (surchargé par prod)
* **Sans profil actif :**
    * Les valeurs du fichier de base `application.yml` sont utilisées.
    * Enabled: `true`
    * Sender: `default-sender@myapp.com`
    * Attempts: `3`
    * Channels: `[email, sms]`
    * Logs `fr.formation.spring`: `INFO`

---

## Conclusion

La gestion de la configuration est un pilier de Spring Boot, offrant flexibilité et robustesse. Que ce soit via les
fichiers `.properties` traditionnels ou les fichiers `.yml` plus structurés, Spring Boot facilite l'accès aux paramètres
de l'application. L'utilisation judicieuse de `@Value` pour des cas simples et de `@ConfigurationProperties` pour des
configurations complexes et typées, combinée à la puissance des profils pour la gestion des environnements, permet de
construire des applications maintenables et adaptées à différents contextes de déploiement. Le respect des bonnes
pratiques, notamment en matière de sécurité et d'externalisation, est essentiel pour des applications prêtes pour la
production.