# La gestion des formulaires

## Introduction aux Formulaires Web et Spring MVC

Les formulaires HTML sont le principal moyen pour les utilisateurs d'envoyer des données à une application web. Que ce
soit pour une inscription, une connexion, la soumission d'un commentaire ou la configuration de paramètres, les
formulaires sont omniprésents.

Spring MVC, un module clé du framework Spring, fournit un support robuste et flexible pour gérer le cycle de vie des
formulaires :

* Afficher un formulaire (souvent pré-rempli).
* Recevoir les données soumises par l'utilisateur.
* Convertir et lier ("binder") ces données à des objets Java (POJOs ou DTOs).
* Valider les données reçues.
* Traiter les données (les sauvegarder, effectuer une action).
* Gérer les erreurs et ré-afficher le formulaire si nécessaire.
* Rediriger l'utilisateur après un traitement réussi.

---

## Le Flux de Base : Affichage et Soumission

Le traitement d'un formulaire implique généralement deux interactions HTTP principales :

1. **Requête GET :** L'utilisateur demande la page contenant le formulaire. Le serveur renvoie le HTML du formulaire,
   éventuellement pré-rempli avec des données existantes ou des valeurs par défaut.
2. **Requête POST :** L'utilisateur remplit le formulaire et le soumet. Le navigateur envoie les données au serveur dans
   le corps de la requête POST. Le serveur traite ces données.

**Exemple Simple : Formulaire d'Inscription Utilisateur**

Supposons une entité `User` simple (nous introduirons les DTOs plus tard) :

```java
// fr.formation.spring.entity.User
package fr.formation.spring.entity;

// Supposons des annotations JPA pour la persistance (non détaillé ici)
public class User {
    private Long id;
    private String username;
    private String email;
    // Getters et Setters
    // ...
}
```

Un contrôleur Spring MVC pour gérer ce formulaire :

```java
// fr.formation.spring.controller.UserController
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import fr.formation.spring.entity.User; // Sera remplacé par DTO plus tard

@Controller
@RequestMapping("/users")
public class UserController {

    // Affiche le formulaire vide
    @GetMapping("/register")
    public String showRegistrationForm(Model model) {
        // Ajoute un nouvel objet User au modèle sous la clé "user"
        // Cet objet sera utilisé par Thymeleaf pour lier les champs
        model.addAttribute("user", new User());
        // Retourne le nom de la vue (template Thymeleaf)
        return "user-registration"; // Doit correspondre à src/main/resources/templates/user-registration.html
    }

    // Traite la soumission du formulaire
    @PostMapping("/register")
    public String processRegistration(
            @ModelAttribute("user") User user // Spring lie les données du formulaire à cet objet
    ) {
        // Logique de traitement (ex: sauvegarde en base de données)
        System.out.println("Nouvel utilisateur reçu:");
        System.out.println("Username: " + user.getUsername());
        System.out.println("Email: " + user.getEmail());

        // Pour l'instant, redirection simple vers une page de succès
        // Le pattern PRG sera abordé plus tard
        return "redirect:/users/registration-success";
    }

    @GetMapping("/registration-success")
    public String showSuccessPage() {
        return "registration-success"; // Template de succès
    }
}
```

Le template Thymeleaf (`src/main/resources/templates/user-registration.html`):

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Inscription</title>
    <meta charset="UTF-8">
</head>
<body>
<h1>Formulaire d'inscription</h1>
<!-- th:object lie le formulaire à l'objet 'user' du modèle -->
<form th:action="@{/users/register}" th:object="${user}" method="post">
    <div>
        <label for="username">Nom d'utilisateur:</label>
        <!-- th:field lie ce champ à la propriété 'username' de l'objet 'user' -->
        <!-- Génère id, name et value automatiquement -->
        <input type="text" id="username" th:field="*{username}"/>
    </div>
    <div>
        <label for="email">Email:</label>
        <input type="email" id="email" th:field="*{email}"/>
    </div>
    <div>
        <button type="submit">S'inscrire</button>
    </div>
</form>
</body>
</html>
```

**Points Clés:**

* `@Controller` : Marque la classe comme un contrôleur Web MVC.
* `@RequestMapping("/users")` : Définit un préfixe de chemin pour toutes les méthodes du contrôleur.
* `@GetMapping("/register")` : Gère les requêtes GET pour `/users/register`.
* `@PostMapping("/register")` : Gère les requêtes POST pour `/users/register`.
* `Model` : Interface pour passer des données du contrôleur à la vue.
* `model.addAttribute("user", new User())` : Met un objet `User` à disposition de la vue sous le nom `user`.
* `@ModelAttribute("user") User user` : Indique à Spring de créer une instance de `User`, de la peupler avec les données
  du formulaire correspondant aux noms de champs, et de la rendre disponible comme argument de la méthode. Le nom
  `"user"` doit correspondre à celui utilisé dans `addAttribute` et `th:object`.
* `th:action="@{/users/register}"` : Génère l'URL de soumission du formulaire.
* `th:object="${user}"` : Lie le formulaire à l'objet `user` passé dans le modèle.
* `th:field="*{username}"` : Raccourci Thymeleaf pour lier un champ de formulaire à une propriété de l'objet défini par
  `th:object`. Il génère les attributs `name`, `id` (si `id` n'est pas présent), et `value`. La syntaxe `*{...}` est une
  sélection relative à l'objet `th:object`.

---

## Data Transfer Objects (DTO) : Pourquoi et Comment

### Problématique

Utiliser directement les entités JPA (ou autres objets du domaine persistants) dans les formulaires pose plusieurs
problèmes :

1. **Couplage Fort :** La structure du formulaire devient directement liée à la structure de la base de données. Tout
   changement dans l'entité peut casser le formulaire, et vice-versa.
2. **Exposition de Données Internes :** L'entité peut contenir des champs non pertinents ou sensibles (ex: `id`,
   `dateCreation`, relations complexes) qui ne devraient pas être exposés ou modifiables via le formulaire.
3. **Problèmes de Validation :** Les contraintes de validation pour le formulaire peuvent différer de celles de l'entité
   persistante (ex: un champ "confirmer mot de passe" n'existe pas dans l'entité `User`).
4. **Sécurité (Mass Assignment) :** Un utilisateur malveillant pourrait injecter des valeurs pour des champs de l'entité
   non présents dans le formulaire HTML, mais qui existent dans l'objet `User` (ex: `isAdmin=true`). Voir la section
   Sécurité.
5. **Optimisation :** Les DTOs peuvent être conçus pour ne transporter que les données strictement nécessaires,
   réduisant la charge utile et améliorant potentiellement les performances, notamment dans les API REST.

### Solution : Le DTO

Un **Data Transfer Object (DTO)** est un objet simple (souvent un POJO) dont le seul but est de transporter des données
entre différentes couches ou composants d'une application (ex: entre le contrôleur et la vue, ou entre le frontend et le
backend via une API).

Dans le contexte des formulaires Spring MVC, le DTO sert d'intermédiaire entre le formulaire HTML et la logique
métier/persistance.

**Avantages :**

* **Découplage :** Sépare la représentation des données pour la vue/formulaire de la représentation des données pour la
  persistance.
* **Sécurité :** N'expose que les champs nécessaires et attendus du formulaire. Prévient le "Mass Assignment".
* **Flexibilité :** Permet d'adapter la structure des données aux besoins spécifiques du formulaire (ex: ajout de champs
  de confirmation, agrégation de données).
* **Validation Ciblée :** Permet d'appliquer des règles de validation spécifiques au contexte du formulaire.

### Exemple de DTO

Reprenons l'exemple de l'inscription. Nous créons un DTO spécifique :

```java
// fr.formation.spring.dto.UserRegistrationDto
package fr.formation.spring.dto;

// Pas d'annotations JPA ici. Validation sera ajoutée plus tard.
public class UserRegistrationDto {

    private String username;
    private String email;
    private String password;
    private String confirmPassword; // Champ spécifique au formulaire

    // Getters et Setters obligatoires pour le data binding
    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public String getConfirmPassword() {
        return confirmPassword;
    }

    public void setConfirmPassword(String confirmPassword) {
        this.confirmPassword = confirmPassword;
    }
}
```

Le contrôleur utilise maintenant ce DTO :

```java
// fr.formation.spring.controller.UserController (modifié)
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import fr.formation.spring.dto.UserRegistrationDto;
// Importer le service qui gère la logique métier (y compris le mapping)
// import fr.formation.spring.service.UserService;

@Controller
@RequestMapping("/users")
public class UserController {

    // Injection du service (exemple)
    // private final UserService userService;
    // public UserController(UserService userService) { this.userService = userService; }

    @GetMapping("/register")
    public String showRegistrationForm(Model model) {
        // On passe maintenant le DTO à la vue
        model.addAttribute("userDto", new UserRegistrationDto());
        return "user-registration";
    }

    @PostMapping("/register")
    public String processRegistration(
            // Spring lie les données du formulaire au DTO
            @ModelAttribute("userDto") UserRegistrationDto userDto
    ) {
        // Ici, on devrait appeler un service pour :
        // 1. Mapper le DTO vers l'entité User
        // 2. Effectuer la logique métier (ex: hasher le mot de passe)
        // 3. Sauvegarder l'entité User
        // Exemple simplifié :
        System.out.println("DTO reçu:");
        System.out.println("Username: " + userDto.getUsername());
        System.out.println("Email: " + userDto.getEmail());
        // !! Ne jamais logger les mots de passe en clair !!
        // userService.registerNewUser(userDto); // Appel au service

        return "redirect:/users/registration-success";
    }

    // ... méthode showSuccessPage inchangée ...
}

```

Le template Thymeleaf doit être adapté pour utiliser `userDto` et le champ `confirmPassword` :

```html
<!-- src/main/resources/templates/user-registration.html (modifié) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Inscription</title>
    <meta charset="UTF-8">
    <style>.error {
        color: red;
    }</style> <!-- Pour afficher les erreurs plus tard -->
</head>
<body>
<h1>Formulaire d'inscription</h1>
<!-- Utilise maintenant l'objet 'userDto' -->
<form th:action="@{/users/register}" th:object="${userDto}" method="post">
    <div>
        <label for="username">Nom d'utilisateur:</label>
        <input type="text" id="username" th:field="*{username}"/>
        <!-- Espace pour afficher les erreurs de validation -->
        <span class="error"
              th:if="${#fields.hasErrors('username')}"
              th:errors="*{username}">Erreur Username</span>
    </div>
    <div>
        <label for="email">Email:</label>
        <input type="email" id="email" th:field="*{email}"/>
        <span class="error"
              th:if="${#fields.hasErrors('email')}"
              th:errors="*{email}">Erreur Email</span>
    </div>
    <div>
        <label for="password">Mot de passe:</label>
        <input type="password" id="password" th:field="*{password}"/>
        <span class="error"
              th:if="${#fields.hasErrors('password')}"
              th:errors="*{password}">Erreur Password</span>
    </div>
    <div>
        <label for="confirmPassword">Confirmer le mot de passe:</label>
        <input type="password" id="confirmPassword" th:field="*{confirmPassword}"/>
        <span class="error"
              th:if="${#fields.hasErrors('confirmPassword')}"
              th:errors="*{confirmPassword}">Erreur Confirmation</span>
        <!-- Erreur globale (ex: mots de passe différents) -->
        <span class="error"
              th:if="${#fields.hasGlobalErrors()}"
              th:each="err : ${#fields.globalErrors()}" th:text="${err}">Erreur Globale</span>
    </div>
    <div>
        <button type="submit">S'inscrire</button>
    </div>
</form>
</body>
</html>
```

### Mapping DTO <-> Entité

Une fois le DTO reçu dans le contrôleur (ou dans une couche de service appelée par le contrôleur), il faut généralement
le convertir en une ou plusieurs entités du domaine pour la persistance ou la logique métier. Inversement, pour afficher
un formulaire de modification pré-rempli, il faut convertir une entité en DTO.

Plusieurs approches existent :

#### Mapping Manuel

Le plus simple, mais peut devenir verbeux et source d'erreurs pour des objets complexes.

```java
// Dans une classe de service, par exemple UserService
package fr.formation.spring.service;

import fr.formation.spring.dto.UserRegistrationDto;
import fr.formation.spring.entity.User;
// Importer PasswordEncoder, UserRepository etc.

public class UserService {

    // ... injections ...

    public User convertDtoToEntity(UserRegistrationDto dto) {
        User user = new User();
        user.setUsername(dto.getUsername());
        user.setEmail(dto.getEmail());
        // Le mot de passe doit être hashé avant d'être stocké
        // user.setPassword(passwordEncoder.encode(dto.getPassword()));
        // Ne pas mapper confirmPassword vers l'entité
        return user;
    }

    public UserRegistrationDto convertEntityToDto(User user) {
        UserRegistrationDto dto = new UserRegistrationDto();
        dto.setUsername(user.getUsername());
        dto.setEmail(user.getEmail());
        // Ne pas mapper le mot de passe (hashé) vers le DTO du formulaire
        // Les champs password et confirmPassword resteront nulls (ou vides)
        return dto;
    }

    // Méthode métier utilisant le mapping
    public void registerNewUser(UserRegistrationDto userDto) {
        // Validation métier supplémentaire (ex: email unique) si nécessaire

        User newUser = convertDtoToEntity(userDto);
        // Hashage du mot de passe
        // newUser.setPassword(passwordEncoder.encode(userDto.getPassword()));
        // Sauvegarde via le repository
        // userRepository.save(newUser);
    }
}

```

**Avantages :** Simple, pas de dépendance externe.
**Inconvénients :** Boilerplate, risque d'oubli ou d'erreur si les objets évoluent.

#### Mapping avec ModelMapper

Une librairie populaire qui utilise la réflexion pour mapper automatiquement les champs ayant le même nom.

**Dépendance (Maven) :**

```xml

<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.1.0</version> <!-- Vérifier la dernière version -->
</dependency>
```

**Configuration (Bean Spring) :**

```java
// fr.formation.spring.config.AppConfig
package fr.formation.spring.config;

import org.modelmapper.ModelMapper;
import org.modelmapper.convention.MatchingStrategies;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppConfig {

    @Bean
    public ModelMapper modelMapper() {
        ModelMapper modelMapper = new ModelMapper();
        // Stratégie stricte: ne mappe que si les noms ET types correspondent parfaitement
        // Ou utiliser LOOSE si les noms suffisent (plus permissif)
        modelMapper.getConfiguration()
                .setMatchingStrategy(MatchingStrategies.STRICT);
        return modelMapper;
    }
}
```

**Utilisation :**

```java
// Dans UserService, injecter ModelMapper
// private final ModelMapper modelMapper;
// public UserService(ModelMapper modelMapper /*, ... autres injections */) {
//     this.modelMapper = modelMapper;
//     // ...
// }

public User convertDtoToEntityWithModelMapper(UserRegistrationDto dto) {
    // Mappe les champs correspondants (username, email)
    User user = modelMapper.map(dto, User.class);
    // Traitements spécifiques (hashage mdp) toujours nécessaires
    return user;
}

public UserRegistrationDto convertEntityToDtoWithModelMapper(User user) {
    // Mappe les champs correspondants (username, email)
    UserRegistrationDto dto = modelMapper.map(user, UserRegistrationDto.class);
    return dto;
}
```

**Avantages :** Réduit le boilerplate, configuration flexible pour les cas complexes.
**Inconvénients :** Utilise la réflexion (peut avoir un léger impact sur les performances au démarrage et à
l'exécution), peut être moins évident à déboguer que le code manuel si les conventions ne suffisent pas.

#### Mapping avec MapStruct

Une autre librairie populaire qui fonctionne différemment : elle génère du code de mapping concret à la compilation via
un processeur d'annotations.

**Dépendances (Maven) :**

```xml

<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version> <!-- Vérifier la dernière version -->
</dependency>
<dependency>
<groupId>org.mapstruct</groupId>
<artifactId>mapstruct-processor</artifactId>
<version>1.5.5.Final</version>
<scope>provided</scope> <!-- Important : nécessaire seulement à la compilation -->
</dependency>
        <!-- Si utilisation avec Lombok -->
<dependency>
<groupId>org.projectlombok</groupId>
<artifactId>lombok-mapstruct-binding</artifactId>
<version>0.2.0</version>
</dependency>
```

**Configuration du plugin Maven pour le processeur d'annotations :**

```xml

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.8.1</version> <!-- ou plus récent -->
    <configuration>
        <source>17</source> <!-- ou votre version de Java -->
        <target>17</target>
        <annotationProcessorPaths>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version> <!-- si Lombok est utilisé -->
            </path>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.2.0</version>
            </path>
            <!-- autres processeurs d'annotations -->
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

**Interface de Mapping :**

```java
// fr.formation.spring.mapper.UserMapper
package fr.formation.spring.mapper;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.factory.Mappers;

import fr.formation.spring.dto.UserRegistrationDto;
import fr.formation.spring.entity.User;

// componentModel = "spring" permet l'injection du mapper comme un bean Spring
@Mapper(componentModel = "spring")
public interface UserMapper {

    // Instance pour utilisation sans injection Spring (moins courant)
    // UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

    // MapStruct ignore par défaut les champs qui n'existent pas dans la cible.
    // Ici, password et confirmPassword du DTO n'ont pas de correspondance
    // directe dans User (le password de User sera traité séparément).
    // On peut être explicite si nécessaire :
    @Mapping(target = "id", ignore = true) // Ne pas mapper l'ID du DTO vers l'entité
    @Mapping(target = "password", ignore = true)
    // Ne pas mapper le mdp DTO directement
    User dtoToEntity(UserRegistrationDto dto);

    // Pour la conversion inverse, on ignore aussi les champs password du DTO
    @Mapping(target = "password", ignore = true)
    @Mapping(target = "confirmPassword", ignore = true)
    UserRegistrationDto entityToDto(User user);
}
```

**Utilisation :**

```java
// Dans UserService, injecter le Mapper généré
// private final UserMapper userMapper;
// public UserService(UserMapper userMapper /*, ... autres */) {
//     this.userMapper = userMapper;
//     // ...
// }

public User convertDtoToEntityWithMapStruct(UserRegistrationDto dto) {
    User user = userMapper.dtoToEntity(dto);
    // Traitements spécifiques (hashage mdp) toujours nécessaires
    return user;
}

public UserRegistrationDto convertEntityToDtoWithMapStruct(User user) {
    UserRegistrationDto dto = userMapper.entityToDto(user);
    return dto;
}
```

**Avantages :** Très performant (pas de réflexion à l'exécution), type-safe (erreurs détectées à la compilation), bonne
intégration avec Spring et Lombok.
**Inconvénients :** Nécessite une configuration du build (processeur d'annotations), la "magie" se passe à la
compilation, ce qui peut être moins intuitif au début.

**Conseil :** Choisir entre ModelMapper et MapStruct dépend des préférences de l'équipe et des contraintes du projet.
MapStruct est souvent préféré pour les applications critiques en termes de performance et pour sa robustesse à la
compilation. ModelMapper est parfois plus rapide à mettre en place pour des besoins simples. Le mapping manuel reste une
option viable pour des cas très simples ou très spécifiques.

---

### Exercice 1 : Création et Mapping d'un DTO

**Objectif :** Créer un DTO pour un formulaire de création de produit et implémenter le mapping manuel vers une entité
`Product`.

**Énoncé :**

1. Créez une classe d'entité `fr.formation.spring.entity.Product` avec les champs : `id` (Long), `name` (String),
   `description` (String), `price` (double), `sku` (String, Stock Keeping Unit - un code interne).
2. Créez une classe DTO `fr.formation.spring.dto.ProductCreateDto` contenant uniquement les champs nécessaires pour le
   formulaire de création : `name`, `description`, `price`.
3. Créez une classe `fr.formation.spring.service.ProductService`.
4. Dans `ProductService`, écrivez une méthode `Product convertDtoToEntity(ProductCreateDto dto)` qui réalise le mapping
   manuel du DTO vers l'entité `Product`. Le champ `sku` de l'entité devra être généré (par exemple, une chaîne
   aléatoire ou basée sur le nom) et l'`id` sera `null` (généré par la persistance).

#### Correction Exercice 1 {collapsible="true"}

```java
// fr.formation.spring.entity.Product
package fr.formation.spring.entity;

// Supposons des annotations JPA
public class Product {
    private Long id;
    private String name;
    private String description;
    private double price;
    private String sku; // Code interne, non fourni par le formulaire

    // Getters et Setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }

    public String getSku() {
        return sku;
    }

    public void setSku(String sku) {
        this.sku = sku;
    }
}
```

```java
// fr.formation.spring.dto.ProductCreateDto
package fr.formation.spring.dto;

public class ProductCreateDto {
    private String name;
    private String description;
    private double price;

    // Getters et Setters
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public double getPrice() {
        return price;
    }

    public void setPrice(double price) {
        this.price = price;
    }
}
```

```java
// fr.formation.spring.service.ProductService
package fr.formation.spring.service;

import java.util.UUID; // Pour générer un SKU simple

import fr.formation.spring.dto.ProductCreateDto;
import fr.formation.spring.entity.Product;

public class ProductService {

    public Product convertDtoToEntity(ProductCreateDto dto) {
        Product product = new Product();
        product.setName(dto.getName());
        product.setDescription(dto.getDescription());
        product.setPrice(dto.getPrice());
        // L'ID est laissé à null (sera généré par JPA)
        // Génération simple d'un SKU (exemple)
        product.setSku("SKU-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase());
        return product;
    }

    // Méthode pour simuler l'utilisation
    public void createProduct(ProductCreateDto dto) {
        Product newProduct = convertDtoToEntity(dto);
        System.out.println("Produit à sauvegarder:");
        System.out.println("  Nom: " + newProduct.getName());
        System.out.println("  Prix: " + newProduct.getPrice());
        System.out.println("  SKU: " + newProduct.getSku());
        // Ici, on appellerait productRepository.save(newProduct);
    }
}
```

---

## Validation des Données

Il est crucial de valider les données soumises par les utilisateurs avant de les traiter. La validation garantit
l'intégrité des données, améliore l'expérience utilisateur (en fournissant des retours clairs sur les erreurs) et
renforce la sécurité (en rejetant les entrées malformées ou malveillantes).

### Introduction à Bean Validation (JSR 380)

Java fournit une API standard pour la validation des JavaBeans : **Bean Validation** (spécifiée par JSR 380, successeur
de JSR 303/349). Hibernate Validator est l'implémentation de référence et est inclus par défaut dans le
`spring-boot-starter-validation` (qui est une dépendance de `spring-boot-starter-web`).

Le principe est d'annoter les champs des objets (typiquement les DTOs) avec des **contraintes** de validation.

### Contraintes Standards

Bean Validation fournit un ensemble de contraintes prêtes à l'emploi :

* `@NotNull` : L'élément ne doit pas être nul.
* `@NotBlank` : L'élément ne doit pas être nul et doit contenir au moins un caractère non blanc (différent de
  `@NotEmpty` qui autorise les espaces). Applicable aux chaînes.
* `@NotEmpty` : L'élément ne doit pas être nul et sa taille/longueur doit être supérieure à 0. Applicable aux chaînes,
  collections, maps, tableaux.
* `@Size(min=..., max=...)` : La taille/longueur de l'élément doit être dans l'intervalle spécifié. Applicable aux
  chaînes, collections, maps, tableaux.
* `@Min(value=...)` : La valeur numérique doit être supérieure ou égale à la valeur spécifiée.
* `@Max(value=...)` : La valeur numérique doit être inférieure ou égale à la valeur spécifiée.
* `@Email` : La chaîne doit être une adresse e-mail bien formée.
* `@Pattern(regexp=...)` : La chaîne doit correspondre à l'expression régulière Java spécifiée.
* `@Past`, `@PastOrPresent` : La date doit être dans le passé / passé ou présent.
* `@Future`, `@FutureOrPresent` : La date doit être dans le futur / futur ou présent.
* `@AssertTrue`, `@AssertFalse` : L'élément booléen doit être vrai / faux.
* `@Valid` : Déclenche la validation récursive sur l'objet associé (utile pour les objets imbriqués).

### Intégration avec Spring MVC

Spring MVC s'intègre nativement avec Bean Validation. Pour activer la validation d'un objet lié à un formulaire :

1. Annotez les champs du DTO avec les contraintes de validation.
2. Dans la méthode du contrôleur qui reçoit l'objet (`@PostMapping`), ajoutez l'annotation `@Valid` devant le paramètre
   `@ModelAttribute`.
3. Ajoutez un paramètre de type `BindingResult` (ou `Errors`) *immédiatement après* le paramètre annoté `@Valid`. Cet
   objet collectera les erreurs de validation.

**Exemple (suite UserRegistrationDto):**

```java
// fr.formation.spring.dto.UserRegistrationDto (avec validation)
package fr.formation.spring.dto;

import jakarta.validation.constraints.*; // Import depuis jakarta.validation

public class UserRegistrationDto {

    @NotBlank(message = "Le nom d'utilisateur est obligatoire.")
    @Size(min = 3, max = 20, message = "Le nom d'utilisateur doit contenir entre 3 et 20 caractères.")
    private String username;

    @NotBlank(message = "L'email est obligatoire.")
    @Email(message = "Le format de l'email est invalide.")
    private String email;

    @NotBlank(message = "Le mot de passe est obligatoire.")
    @Size(min = 8, message = "Le mot de passe doit contenir au moins 8 caractères.")
    // On pourrait ajouter @Pattern pour la complexité (ex: majuscule, chiffre...)
    private String password;

    @NotBlank(message = "La confirmation du mot de passe est obligatoire.")
    private String confirmPassword;

    // Getters et Setters...
    // ... (inchangés)
}
```

**Contrôleur (modifié pour gérer la validation):**

```java
// fr.formation.spring.controller.UserController (modifié)
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult; // Important
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

import fr.formation.spring.dto.UserRegistrationDto;
import jakarta.validation.Valid; // Important

@Controller
@RequestMapping("/users")
public class UserController {

    // ... injection service ...

    @GetMapping("/register")
    public String showRegistrationForm(Model model) {
        if (!model.containsAttribute("userDto")) { // Evite d'écraser en cas d'erreur
            model.addAttribute("userDto", new UserRegistrationDto());
        }
        return "user-registration";
    }

    @PostMapping("/register")
    public String processRegistration(
            // 1. @Valid active la validation sur userDto
            @Valid
            // 2. @ModelAttribute lie les données et ajoute le DTO au modèle
            @ModelAttribute("userDto") UserRegistrationDto userDto,
            // 3. BindingResult collecte les erreurs (DOIT suivre @Valid DTO)
            BindingResult bindingResult
            // Model model // Plus nécessaire si on utilise @ModelAttribute
            // car il ajoute userDto au modèle automatiquement
    ) {
        // 4. Vérifier s'il y a des erreurs de validation
        if (bindingResult.hasErrors()) {
            // Si oui, retourner à la vue du formulaire
            // Spring + Thymeleaf ré-afficheront le formulaire
            // avec les données saisies et les messages d'erreur
            // grâce à l'objet userDto (ajouté au modèle par @ModelAttribute)
            // et à BindingResult (accessible par Thymeleaf via #fields)
            return "user-registration";
        }

        // S'il n'y a pas d'erreur de validation standard,
        // on peut continuer le traitement (ex: validation métier, sauvegarde)
        // userService.registerNewUser(userDto);
        System.out.println("Validation réussie pour : " + userDto.getUsername());

        // Redirection vers la page de succès (Pattern PRG)
        return "redirect:/users/registration-success";
    }

    // ... méthode showSuccessPage ...
}
```

### Affichage des Erreurs (Thymeleaf)

Thymeleaf fournit des utilitaires pour afficher facilement les erreurs de validation collectées dans `BindingResult`.
L'objet `fields` dans le contexte Thymeleaf permet d'accéder à ces informations.

Reprenons le template `user-registration.html` et ajoutons l'affichage des erreurs (déjà esquissé précédemment) :

```html
<!-- src/main/resources/templates/user-registration.html (avec erreurs) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Inscription</title>
    <meta charset="UTF-8">
    <style>
        .error {
            color: red;
            font-size: 0.9em;
        }

        label {
            display: block;
            margin-top: 10px;
        }

        input {
            margin-bottom: 5px;
        }

        button {
            margin-top: 15px;
        }
    </style>
</head>
<body>
<h1>Formulaire d'inscription</h1>
<form th:action="@{/users/register}" th:object="${userDto}" method="post">
    <div>
        <label for="username">Nom d'utilisateur:</label>
        <input type="text" id="username" th:field="*{username}"/>
        <!-- Affiche les erreurs spécifiques au champ 'username' -->
        <div class="error" th:if="${#fields.hasErrors('username')}">
            <p th:each="err : ${#fields.errors('username')}" th:text="${err}"></p>
        </div>
    </div>
    <div>
        <label for="email">Email:</label>
        <input type="email" id="email" th:field="*{email}"/>
        <!-- Affiche les erreurs spécifiques au champ 'email' -->
        <div class="error" th:if="${#fields.hasErrors('email')}">
            <p th:each="err : ${#fields.errors('email')}" th:text="${err}"></p>
        </div>
    </div>
    <div>
        <label for="password">Mot de passe:</label>
        <input type="password" id="password" th:field="*{password}"/>
        <!-- Affiche les erreurs spécifiques au champ 'password' -->
        <div class="error" th:if="${#fields.hasErrors('password')}">
            <p th:each="err : ${#fields.errors('password')}" th:text="${err}"></p>
        </div>
    </div>
    <div>
        <label for="confirmPassword">Confirmer le mot de passe:</label>
        <input type="password" id="confirmPassword" th:field="*{confirmPassword}"/>
        <!-- Affiche les erreurs spécifiques au champ 'confirmPassword' -->
        <div class="error" th:if="${#fields.hasErrors('confirmPassword')}">
            <p th:each="err : ${#fields.errors('confirmPassword')}" th:text="${err}"></p>
        </div>
    </div>

    <!-- Affiche les erreurs globales (non liées à un champ spécifique) -->
    <div class="error" th:if="${#fields.hasGlobalErrors()}">
        <p>Erreurs générales :</p>
        <ul>
            <li th:each="err : ${#fields.globalErrors()}" th:text="${err}"></li>
        </ul>
    </div>

    <div>
        <button type="submit">S'inscrire</button>
    </div>
</form>
</body>
</html>
```

**Explication Thymeleaf:**

* `#fields.hasErrors('fieldName')` : Vérifie s'il y a des erreurs pour le champ spécifié.
* `#fields.errors('fieldName')` : Récupère la liste des messages d'erreur pour ce champ.
* `th:each="err : ..."` : Itère sur la liste des erreurs.
* `th:text="${err}"` : Affiche le message d'erreur. Le message provient de l'attribut `message` de l'annotation de
  validation.
* `#fields.hasGlobalErrors()` : Vérifie les erreurs non associées à un champ spécifique (nous verrons comment les
  ajouter avec la validation personnalisée).
* `#fields.globalErrors()` : Récupère la liste des erreurs globales.

### Validation Personnalisée (Custom Validators)

Les contraintes standard ne couvrent pas tous les besoins. Bean Validation permet de créer ses propres contraintes. Cela
implique trois étapes :

1. **Créer une annotation de contrainte :** Définit le nom de l'annotation, le message d'erreur par défaut, et lie
   l'annotation à son implémentation de validation.
2. **Implémenter l'interface `ConstraintValidator` :** Contient la logique de validation réelle.
3. **Appliquer l'annotation** sur le champ ou la classe à valider.

#### Cas 1 : Validation Simple (Format spécifique)

**Objectif :** Créer une contrainte `@ValidPostalCode` pour valider qu'une chaîne représente un code postal français (5
chiffres).

**1. Annotation `@ValidPostalCode` :**

```java
// fr.formation.spring.validation.annotation.ValidPostalCode
package fr.formation.spring.validation.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import fr.formation.spring.validation.validator.PostalCodeValidator;

// @Target: Où peut-on appliquer l'annotation (champs, méthodes...)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER, ElementType.ANNOTATION_TYPE})
// @Retention: Quand l'annotation est-elle disponible (RUNTIME pour Bean Validation)
@Retention(RetentionPolicy.RUNTIME)
// @Constraint: Lie l'annotation à sa classe de validation
@Constraint(validatedBy = PostalCodeValidator.class)
@Documented // Inclure dans la Javadoc
public @interface ValidPostalCode {

    // Message d'erreur par défaut si la validation échoue
    String message() default "Le code postal doit contenir 5 chiffres.";

    // Groupes de validation (avancé, permet d'activer/désactiver des validations)
    Class<?>[] groups() default {};

    // Payload (avancé, permet d'attacher des métadonnées à la contrainte)
    Class<? extends Payload>[] payload() default {};
}
```

**2. Implémentation `PostalCodeValidator` :**

```java
// fr.formation.spring.validation.validator.PostalCodeValidator
package fr.formation.spring.validation.validator;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import fr.formation.spring.validation.annotation.ValidPostalCode;

// Implémente ConstraintValidator<Annotation, TypeDuChampValide>
public class PostalCodeValidator
        implements ConstraintValidator<ValidPostalCode, String> {

    private static final String POSTAL_CODE_PATTERN = "^[0-9]{5}$";

    @Override
    public void initialize(ValidPostalCode constraintAnnotation) {
        // Peut être utilisé pour initialiser le validateur
        // avec des paramètres de l'annotation (non nécessaire ici)
    }

    @Override
    public boolean isValid(String postalCode,
                           ConstraintValidatorContext context) {
        // La logique de validation :
        // null est considéré valide (utiliser @NotNull en plus si besoin)
        if (postalCode == null) {
            return true;
        }
        // Vérifie si la chaîne correspond au pattern (5 chiffres)
        return postalCode.matches(POSTAL_CODE_PATTERN);
    }
}
```

**3. Application sur un DTO :**

```java
// fr.formation.spring.dto.AddressDto (exemple)
package fr.formation.spring.dto;

import fr.formation.spring.validation.annotation.ValidPostalCode;
import jakarta.validation.constraints.NotBlank;

public class AddressDto {

    @NotBlank(message = "La rue est obligatoire.")
    private String street;

    @NotBlank(message = "La ville est obligatoire.")
    private String city;

    @NotBlank(message = "Le code postal est obligatoire.")
    @ValidPostalCode // Application de la contrainte personnalisée
    private String postalCode;

    // Getters et Setters...
}
```

Désormais, si le champ `postalCode` d'un `AddressDto` validé (`@Valid AddressDto`) ne contient pas exactement 5
chiffres, une erreur de validation sera générée avec le message défini dans `@ValidPostalCode`.

#### Cas 2 : Validation Inter-champs (Comparaison de mots de passe)

**Objectif :** Valider que les champs `password` et `confirmPassword` du `UserRegistrationDto` sont identiques. Ce type
de validation doit s'appliquer au niveau de la classe, car elle nécessite l'accès à plusieurs champs.

**1. Annotation `@PasswordsMatch` :**

```java
// fr.formation.spring.validation.annotation.PasswordsMatch
package fr.formation.spring.validation.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import fr.formation.spring.validation.validator.PasswordsMatchValidator;

// @Target(ElementType.TYPE) : L'annotation s'applique à une classe
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordsMatchValidator.class)
@Documented
public @interface PasswordsMatch {

    String message() default "Les mots de passe ne correspondent pas.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    // On peut ajouter des paramètres pour spécifier les noms des champs à comparer
    // String passwordFieldName() default "password";
    // String confirmPasswordFieldName() default "confirmPassword";
    // Mais pour simplifier, on va les coder en dur dans le validateur pour cet exemple.
}
```

**2. Implémentation `PasswordsMatchValidator` :**

```java
// fr.formation.spring.validation.validator.PasswordsMatchValidator
package fr.formation.spring.validation.validator;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import fr.formation.spring.dto.UserRegistrationDto; // Type spécifique ici
import fr.formation.spring.validation.annotation.PasswordsMatch;

// Valide un objet de type UserRegistrationDto
public class PasswordsMatchValidator
        implements ConstraintValidator<PasswordsMatch, UserRegistrationDto> {

    @Override
    public void initialize(PasswordsMatch constraintAnnotation) {
        // Initialisation si nécessaire
    }

    @Override
    public boolean isValid(UserRegistrationDto userDto,
                           ConstraintValidatorContext context) {
        String password = userDto.getPassword();
        String confirmPassword = userDto.getConfirmPassword();

        // Si les deux sont null ou vides, on considère que c'est ok
        // (la validation @NotBlank sur les champs gérera l'obligation)
        // Ou on pourrait décider que c'est invalide ici aussi.
        if (password == null || password.isEmpty() ||
                confirmPassword == null || confirmPassword.isEmpty()) {
            return true; // Laisser @NotBlank faire son travail sur les champs
        }

        boolean passwordsMatch = password.equals(confirmPassword);

        if (!passwordsMatch) {
            // Si on veut que l'erreur soit associée à un champ spécifique (ex: confirmPassword)
            // Désactive le message global par défaut
            // context.disableDefaultConstraintViolation();
            // context.buildConstraintViolationWithTemplate(
            //      context.getDefaultConstraintMessageTemplate()
            // )
                       // Associe à ce champ
            //        .addPropertyNode("confirmPassword") 
            //        .addConstraintViolation();

            // Par défaut (sans le code ci-dessus), 
            // l'erreur sera globale (class-level)
        }

        return passwordsMatch;
    }
}
```

**3. Application sur le DTO :**

```java
// fr.formation.spring.dto.UserRegistrationDto (modifié)
package fr.formation.spring.dto;

import jakarta.validation.constraints.*;
import fr.formation.spring.validation.annotation.PasswordsMatch; // Importer

// Appliquer l'annotation au niveau de la classe
@PasswordsMatch(message = "La confirmation doit être identique au mot de passe.")
public class UserRegistrationDto {

    @NotBlank(message = "Le nom d'utilisateur est obligatoire.")
    @Size(min = 3, max = 20, message = "...")
    private String username;

    @NotBlank(message = "L'email est obligatoire.")
    @Email(message = "Format email invalide.")
    private String email;

    @NotBlank(message = "Le mot de passe est obligatoire.")
    @Size(min = 8, message = "Le mot de passe doit faire 8 caractères min.")
    private String password;

    @NotBlank(message = "La confirmation du mot de passe est obligatoire.")
    private String confirmPassword;

    // Getters et Setters...
}
```

Maintenant, lors de la validation (`@Valid UserRegistrationDto`), après la validation des champs individuels, le
`PasswordsMatchValidator` sera exécuté. S'il retourne `false`, une erreur sera ajoutée à `BindingResult`. Par défaut, ce
sera une erreur globale, affichable dans Thymeleaf avec `#fields.globalErrors()`. Si on utilise `addPropertyNode` dans
le validateur, l'erreur sera liée au champ spécifié (ici, `confirmPassword`).

---

### Exercice 2 : Ajout de Validation Standard et Personnalisée

**Objectif :** Ajouter des contraintes de validation standard et une contrainte personnalisée à `ProductCreateDto`.

**Énoncé :**

1. Modifiez `ProductCreateDto` (de l'exercice 1) pour ajouter les contraintes suivantes :
    * `name` : obligatoire (`@NotBlank`), longueur entre 3 et 50 caractères (`@Size`).
    * `description` : longueur maximale de 200 caractères (`@Size`). Peut être vide.
    * `price` : doit être une valeur positive (`@Positive` ou `@DecimalMin("0.01")`).
2. Créez une annotation de validation personnalisée `@ValidSkuFormat` et son validateur `SkuFormatValidator`. Cette
   contrainte doit vérifier qu'une chaîne (le `sku`) commence par "SKU-" suivi de 8 caractères alphanumériques
   majuscules (ex: "SKU-A1B2C3D4").
3. *Bonus :* Modifiez le `ProductCreateDto` et le contrôleur (hypothétique) pour inclure un champ `sku` et appliquez la
   validation `@ValidSkuFormat`. Note : Dans l'exercice 1, le SKU était généré côté serveur, ce qui est souvent
   préférable. Cet ajout est pour l'exercice de validation.

#### **Correction Exercice 2** {collapsible="true"}

**1. `ProductCreateDto` avec validation standard :**

```java
// fr.formation.spring.dto.ProductCreateDto (modifié)
package fr.formation.spring.dto;

import jakarta.validation.constraints.*;

public class ProductCreateDto {

    @NotBlank(message = "Le nom du produit est obligatoire.")
    @Size(min = 3, max = 50, message = "Le nom doit faire entre 3 et 50 caractères.")
    private String name;

    @Size(max = 200, message = "La description ne doit pas dépasser 200 caractères.")
    private String description; // Pas @NotBlank, peut être vide/null

    // @Positive inclut 0. Si on veut strictement positif > 0.00 :
    @NotNull(message = "Le prix est obligatoire.") // Nécessaire car double (primitif) ne peut être null
    // Si Double (wrapper), @NotNull utile aussi.
    @DecimalMin(value = "0.01", message = "Le prix doit être positif.")
    private double price;

    // Getters et Setters...
}
```

**2. Annotation `@ValidSkuFormat` :**

```java
// fr.formation.spring.validation.annotation.ValidSkuFormat
package fr.formation.spring.validation.annotation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

import fr.formation.spring.validation.validator.SkuFormatValidator;

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SkuFormatValidator.class)
@Documented
public @interface ValidSkuFormat {
    String message() default "Format SKU invalide (doit être SKU-XXXXXXXX).";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

**3. Validateur `SkuFormatValidator` :**

```java
// fr.formation.spring.validation.validator.SkuFormatValidator
package fr.formation.spring.validation.validator;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import fr.formation.spring.validation.annotation.ValidSkuFormat;

public class SkuFormatValidator
        implements ConstraintValidator<ValidSkuFormat, String> {

    // Pattern: Commence par "SKU-", suivi de 8 caractères alphanumériques majuscules
    private static final String SKU_PATTERN = "^SKU-[A-Z0-9]{8}$";

    @Override
    public void initialize(ValidSkuFormat constraintAnnotation) {
    }

    @Override
    public boolean isValid(String sku, ConstraintValidatorContext context) {
        if (sku == null) {
            // Considéré valide ici, utiliser @NotBlank si obligatoire
            return true;
        }
        return sku.matches(SKU_PATTERN);
    }
}
```

**4. *Bonus* : Modification de `ProductCreateDto` et application :**

```java
// fr.formation.spring.dto.ProductCreateDto (avec SKU)
package fr.formation.spring.dto;

import jakarta.validation.constraints.*;
import fr.formation.spring.validation.annotation.ValidSkuFormat; // Importer

public class ProductCreateDto {

    @NotBlank(message = "Le nom du produit est obligatoire.")
    @Size(min = 3, max = 50, message = "Le nom doit faire entre 3 et 50 caractères.")
    private String name;

    @Size(max = 200, message = "La description ne doit pas dépasser 200 caractères.")
    private String description;

    @NotNull(message = "Le prix est obligatoire.")
    @DecimalMin(value = "0.01", message = "Le prix doit être positif.")
    private double price;

    @NotBlank(message = "Le SKU est obligatoire.") // Rend le SKU obligatoire pour la validation
    @ValidSkuFormat // Applique la validation de format personnalisée
    private String sku; // Champ ajouté pour le formulaire

    // Getters et Setters (y compris pour sku)...
    public String getSku() {
        return sku;
    }

    public void setSku(String sku) {
        this.sku = sku;
    }
    // ... autres getters/setters
}
```

**Contrôleur (hypothétique) utilisant la validation :**

```java
// fr.formation.spring.controller.ProductController (extrait)
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;

import fr.formation.spring.dto.ProductCreateDto;
import jakarta.validation.Valid;

@Controller
@RequestMapping("/products")
public class ProductController {

    // ... injection service ...

    @GetMapping("/new")
    public String showCreateForm(Model model) {
        model.addAttribute("productDto", new ProductCreateDto());
        return "product-create"; // Nom du template Thymeleaf
    }

    @PostMapping("/create")
    public String processCreation(
            @Valid @ModelAttribute("productDto") ProductCreateDto productDto,
            BindingResult bindingResult
    ) {
        if (bindingResult.hasErrors()) {
            // Les erreurs incluront maintenant celles de @NotBlank, @Size,
            // @DecimalMin et @ValidSkuFormat
            return "product-create"; // Retour au formulaire
        }

        // Logique de création si validation OK
        // productService.createProductFromDto(productDto); // Adapter le service
        System.out.println("Produit valide reçu: " + productDto.getName()
                + ", SKU: " + productDto.getSku());

        return "redirect:/products/success"; // Redirection PRG
    }

    // ... autres méthodes ...
}
```

---

## Gestion de l'Upload de Fichiers

Les formulaires permettent souvent aux utilisateurs d'uploader des fichiers (images, documents, etc.). Spring MVC
facilite la gestion de ces uploads.

### Configuration

Spring Boot nécessite une configuration minimale pour gérer les uploads multipart. Elle se fait généralement dans
`application.properties` ou `application.yml`.

**`application.properties` :**

```properties
# Activer la gestion multipart (généralement activé par défaut avec spring-web)
spring.servlet.multipart.enabled=true
# Taille maximale d'un fichier individuel (par défaut 1MB)
spring.servlet.multipart.max-file-size=10MB
# Taille maximale de la requête multipart complète (tous les fichiers + autres champs) (par défaut 10MB)
spring.servlet.multipart.max-request-size=10MB
# Emplacement temporaire pour stocker les fichiers pendant l'upload
# spring.servlet.multipart.location=/path/to/temp/dir
# Seuil à partir duquel les fichiers sont écrits sur disque (plutôt qu'en mémoire)
# spring.servlet.multipart.file-size-threshold=2KB
```

**Important :** Ajustez `max-file-size` et `max-request-size` selon les besoins de votre application. Des valeurs trop
élevées peuvent exposer le serveur à des attaques par déni de service (upload de fichiers très volumineux).

### Le Formulaire HTML

Le formulaire HTML doit être configuré spécifiquement pour l'upload :

1. L'attribut `method` doit être `post`.
2. L'attribut `enctype` **doit** être `multipart/form-data`.
3. Utiliser un champ `<input type="file">`.

**Exemple : Formulaire d'upload d'avatar**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Upload Avatar</title>
    <meta charset="UTF-8">
    <style>.error {
        color: red;
    }</style>
</head>
<body>
<h1>Changer l'avatar</h1>
<!-- method="post" et enctype="multipart/form-data" sont cruciaux -->
<form method="post" th:action="@{/profile/avatar/upload}" enctype="multipart/form-data">
    <div>
        <label for="avatarFile">Sélectionner un fichier image (max 10MB):</label>
        <!-- Le nom "avatarFile" sera utilisé dans le contrôleur -->
        <input type="file" id="avatarFile" name="avatarFile" accept="image/*"/>
        <!-- accept="image/*" est une indication pour le navigateur, PAS une sécurité -->
    </div>

    <!-- Affichage d'éventuelles erreurs (ex: fichier trop gros, type invalide) -->
    <div th:if="${errorMessage}" class="error">
        <p th:text="${errorMessage}"></p>
    </div>

    <div>
        <button type="submit">Uploader</button>
    </div>
</form>
</body>
</html>
```

### Le Contrôleur

Dans le contrôleur Spring, pour recevoir le fichier uploadé, on utilise un paramètre de type
`org.springframework.web.multipart.MultipartFile`. Le nom du paramètre doit correspondre à l'attribut `name` de l'
`<input type="file">`.

```java
// fr.formation.spring.controller.ProfileController (extrait)
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes; // Pour les messages flash

import java.io.IOException;
import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.UUID;

@Controller
@RequestMapping("/profile/avatar")
public class ProfileController {

    // !! IMPORTANT !! Définir un répertoire de stockage sécurisé
    // NE PAS stocker dans le classpath ou le répertoire web directement accessible
    // Idéalement, configurable via application.properties
    private final Path uploadStorageLocation = Paths.get("./uploads/avatars").toAbsolutePath().normalize();

    public ProfileController() {
        // Créer le répertoire de stockage s'il n'existe pas
        try {
            Files.createDirectories(this.uploadStorageLocation);
        } catch (Exception ex) {
            throw new RuntimeException(
                    "Impossible de créer le répertoire de stockage des uploads.", ex);
        }
    }

    @PostMapping("/upload")
    public String handleAvatarUpload(
            // @RequestParam lie le paramètre au champ "avatarFile" du formulaire
            @RequestParam("avatarFile") MultipartFile file,
            RedirectAttributes redirectAttributes // Pour passer des messages après redirection
    ) {

        // 1. Vérifications de base
        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Veuillez sélectionner un fichier à uploader.");
            return "redirect:/profile/avatar/change"; // Retour au formulaire (via redirection)
        }

        // !! SÉCURITÉ !! Nettoyer le nom du fichier
        // Ne jamais faire confiance au nom de fichier fourni par le client
        String originalFilename = file.getOriginalFilename();
        if (originalFilename == null || originalFilename.contains("..")) {
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Nom de fichier invalide.");
            return "redirect:/profile/avatar/change";
        }

        // Générer un nom de fichier unique pour éviter les écrasements et les conflits
        String fileExtension = "";
        int dotIndex = originalFilename.lastIndexOf('.');
        if (dotIndex > 0) {
            fileExtension = originalFilename.substring(dotIndex);
        }
        // Exemple : utilise un UUID pour garantir l'unicité
        String storedFilename = UUID.randomUUID().toString() + fileExtension;

        try {
            // Validation supplémentaire (taille, type) - voir section suivante

            // Construire le chemin de destination complet
            Path targetLocation = this.uploadStorageLocation.resolve(storedFilename);

            // Copier le fichier vers l'emplacement cible
            // StandardCopyOption.REPLACE_EXISTING est utile si, malgré l'UUID,
            // une collision rarissime arrivait, ou si on réutilise un nom existant.
            Files.copy(file.getInputStream(), targetLocation, StandardCopyOption.REPLACE_EXISTING);

            // Enregistrer le chemin `storedFilename` ou `targetLocation` dans la BD
            // associé à l'utilisateur, etc.
            // userService.updateAvatar(userId, storedFilename);

            redirectAttributes.addFlashAttribute("successMessage",
                    "Avatar uploadé avec succès : " + storedFilename);
            return "redirect:/profile"; // Rediriger vers le profil

        } catch (IOException ex) {
            ex.printStackTrace(); // Logguer l'erreur
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Erreur lors de l'upload du fichier '" + originalFilename + "'.");
            return "redirect:/profile/avatar/change";
        }
        // Gestion des erreurs de taille max (MaxUploadSizeExceededException)
        // peut être faite avec un @ControllerAdvice, voir Bonnes Pratiques.
    }

    // Méthode pour afficher le formulaire (non montrée)
    // @GetMapping("/change") public String showUploadForm(...) { ... }
}
```

**Points Clés et Sécurité :**

* `MultipartFile` : Interface de Spring pour représenter un fichier uploadé. Méthodes utiles :
    * `isEmpty()`: Vérifie si un fichier a été sélectionné.
    * `getOriginalFilename()`: Retourne le nom original du fichier **tel qu'envoyé par le client (non fiable !)**.
    * `getContentType()`: Retourne le type MIME déclaré par le client **(non fiable !)**.
    * `getSize()`: Retourne la taille en octets.
    * `getBytes()`: Retourne le contenu du fichier en tableau d'octets (attention à la mémoire pour les gros fichiers).
    * `getInputStream()`: Retourne un flux pour lire le contenu (préférable pour les gros fichiers).
    * `transferTo(File dest)` ou `transferTo(Path dest)`: Méthode pratique pour sauvegarder le fichier directement sur
      disque.
* **Nettoyage du Nom de Fichier :** Ne **jamais** utiliser `getOriginalFilename()` directement pour enregistrer le
  fichier. Il peut contenir des caractères invalides, des chemins relatifs (`../../..`) pour tenter d'écrire en dehors
  du répertoire prévu (Path Traversal), ou écraser des fichiers existants. **Générez toujours un nom de fichier unique
  et sûr côté serveur** (UUID, timestamp + nom aléatoire, hash du contenu...).
* **Stockage Sécurisé :** Ne stockez **jamais** les fichiers uploadés dans un répertoire directement accessible par le
  serveur web (comme `src/main/webapp` ou sous `static`). Stockez-les dans un répertoire dédié, en dehors de la racine
  web, et servez-les via un contrôleur spécifique si nécessaire (qui vérifiera les permissions).
* `RedirectAttributes` : Utilisé pour passer des messages (succès ou erreur) à la page vers laquelle on redirige après
  le traitement POST (pattern PRG). Ces attributs "flash" ne survivent qu'à une seule redirection.

### Validation des Fichiers Uploadés

Il est essentiel de valider les fichiers uploadés non seulement pour la taille, mais aussi pour leur type réel, afin
d'éviter que des fichiers malveillants (exécutables, scripts) ne soient stockés et potentiellement exécutés ou servis.

#### Validation de la Taille

* **Côté Serveur (Obligatoire) :** La configuration `spring.servlet.multipart.max-file-size` dans
  `application.properties` est la première ligne de défense. Si un fichier dépasse cette taille, Spring lèvera une
  `MaxUploadSizeExceededException` (ou `MultipartException`). Cette exception peut être gérée globalement avec un
  `@ControllerAdvice`.
* **Côté Contrôleur (Optionnel) :** On peut vérifier `file.getSize()` pour des logiques plus fines si nécessaire, mais
  la limite globale est généralement suffisante.

#### Validation du Type de Contenu (MIME Type)

La méthode `file.getContentType()` retourne le type MIME tel qu'envoyé par le navigateur (basé sur l'extension du
fichier ou les informations du système d'exploitation client). **Cette information n'est absolument pas fiable et peut
être facilement falsifiée.
<!-- ** Un fichier `virus.exe` peut être renommé `image.jpg` et le navigateur enverra `image/jpeg`.-->

On peut l'utiliser comme une première vérification rapide, mais elle ne doit **jamais** être la seule méthode de
validation de type.

```java
// Dans handleAvatarUpload, après la vérification isEmpty()
String contentType = file.getContentType();
if(contentType ==null||!contentType.

startsWith("image/")){
        // ATTENTION : Ceci est une vérification faible !
        redirectAttributes.

addFlashAttribute("errorMessage",
                          "Type de fichier non autorisé (basé sur l'info navigateur).");
// return "redirect:/profile/avatar/change"; // Commenté car non fiable
}
```

#### Validation par "Magic Number"

Une méthode beaucoup plus fiable consiste à lire les premiers octets du fichier (le "magic number" ou la "signature")
pour déterminer son type réel, indépendamment de son extension ou du type MIME déclaré.

Chaque format de fichier (JPEG, PNG, PDF, ZIP...) commence généralement par une séquence d'octets spécifique.

**Exemple Simplifié (Validation pour JPEG et PNG) :**

```java
// Dans handleAvatarUpload, après la vérification isEmpty() et nom de fichier

private boolean isValidImageType(MultipartFile file) throws IOException {
    byte[] signature = new byte[4]; // Lire les 4 premiers octets
    try (InputStream inputStream = file.getInputStream()) {
        int bytesRead = inputStream.read(signature);
        if (bytesRead < 4) {
            return false; // Fichier trop petit pour avoir une signature valide
        }
    }

    // Signatures connues (Hexadécimal)
    // JPEG: FF D8 FF E0 ou FF D8 FF E1 ou FF D8 FF DB
    // PNG:  89 50 4E 47 (soit .PNG en ASCII)

    // Conversion des octets lus en format comparable
    String magicNumberHex = bytesToHex(signature);

    System.out.println("Magic number lu: " + magicNumberHex); // Pour débogage

    // Vérification pour PNG
    if (magicNumberHex.startsWith("89504E47")) { // Comparaison Hex
        System.out.println("Type détecté : PNG");
        return true;
    }
    // Vérification pour JPEG
    if (magicNumberHex.startsWith("FFD8FFE0") ||
            magicNumberHex.startsWith("FFD8FFE1") ||
            magicNumberHex.startsWith("FFD8FFDB")) {
        System.out.println("Type détecté : JPEG");
        return true;
    }

    // Autres types d'images pourraient être ajoutés (GIF: 47494638, etc.)

    return false; // Type non reconnu ou non autorisé
}

// Helper pour convertir byte[] en chaîne Hexadécimale
private static final char[] HEX_ARRAY = "0123456789ABCDEF".toCharArray();

public static String bytesToHex(byte[] bytes) {
    char[] hexChars = new char[bytes.length * 2];
    for (int j = 0; j < bytes.length; j++) {
        int v = bytes[j] & 0xFF; // Masque pour obtenir la valeur non signée
        hexChars[j * 2] = HEX_ARRAY[v >>> 4];  // Premier quartet (4 bits de poids fort)
        hexChars[j * 2 + 1] = HEX_ARRAY[v & 0x0F]; // Second quartet (4 bits de poids faible)
    }
    return new String(hexChars);
}
```

```java
// Utilisation dans le contrôleur :

@PostMapping("/profile/avatar/upload")
public String handleAvatarUpload(
        @RequestParam("file") MultipartFile file,
        RedirectAttributes redirectAttributes
) {
    try {
        // Vérifie le type de l'image par contenu (MIME)
        if (!isValidImageType(file)) {
            redirectAttributes.addFlashAttribute(
                    "errorMessage",
                    "Type de fichier image invalide ou non supporté (vérifié par contenu)."
            );
            return "redirect:/profile/avatar/change";
        }

        // TODO : logique de sauvegarde du fichier
        // saveImage(file);

        redirectAttributes.addFlashAttribute(
                "successMessage",
                "Image enregistrée avec succès."
        );
        return "redirect:/profile/avatar";

    } catch (IOException ex) {
        // Gestion de l'erreur liée à la lecture/sauvegarde du fichier
        redirectAttributes.addFlashAttribute(
                "errorMessage",
                "Erreur lors de l'enregistrement de l'image."
        );
        return "redirect:/profile/avatar/change";
    }
}


```

**Conseil / Bonne Pratique : Utiliser une Librairie Dédiée**

Plutôt que de réimplémenter la détection de type par magic number, il est fortement recommandé d'utiliser des librairies
robustes et maintenues pour cela, comme **Apache Tika**.

**Dépendance Apache Tika (Maven) :**

```xml

<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
    <version>2.9.1</version> <!-- Vérifier la dernière version -->
</dependency>
```

**Utilisation avec Tika :**

```java
package fr.formation.spring.controller;

import org.apache.tika.Tika;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

@Controller
public class AvatarController {

    @PostMapping("/profile/avatar/upload")
    public String uploadAvatar(
            @RequestParam("file") MultipartFile file,
            RedirectAttributes redirectAttributes
    ) {
        try {
            Tika tika = new Tika();
            String detectedType = tika.detect(file.getInputStream());

            System.out.println("Tika a détecté le type: " + detectedType);

            List<String> allowedMimeTypes = Arrays.asList(
                    "image/jpeg", "image/png", "image/gif",
                    "image/bmp", "image/webp"
            );

            if (!allowedMimeTypes.contains(detectedType)) {
                redirectAttributes.addFlashAttribute(
                        "errorMessage",
                        "Type de fichier non autorisé : " + detectedType +
                                ". Types permis : JPEG, PNG, GIF, BMP, WEBP."
                );
                return "redirect:/profile/avatar/change";
            }

            // TODO : sauvegarde du fichier ici (nom unique + copy)

            redirectAttributes.addFlashAttribute(
                    "successMessage",
                    "Avatar enregistré avec succès."
            );
            return "redirect:/profile/avatar";

        } catch (IOException ex) {
            redirectAttributes.addFlashAttribute(
                    "errorMessage",
                    "Erreur lors de la lecture du fichier : " + ex.getMessage()
            );
            return "redirect:/profile/avatar/change";
        }
    }
}

```

Apache Tika est beaucoup plus fiable et couvre un éventail bien plus large de formats de fichiers que l'implémentation
manuelle. **C'est l'approche recommandée pour la validation de type de fichier.**

---

### Exercice 3 : Gestion et Validation d'Upload de Fichier

**Objectif :** Implémenter un endpoint de contrôleur pour uploader un document PDF, avec validation de la taille et du
type (en utilisant Apache Tika).

**Énoncé :**

1. Configurez la taille maximale de fichier à 5MB dans `application.properties`.
2. Créez une méthode de contrôleur `@PostMapping("/documents/upload")` dans une classe `DocumentController`.
3. Cette méthode doit accepter un `MultipartFile` nommé `documentFile`.
4. Elle doit vérifier que le fichier n'est pas vide.
5. Elle doit utiliser Apache Tika pour vérifier que le type MIME détecté est bien `application/pdf`.
6. Si les validations passent, elle doit générer un nom de fichier unique (UUID + ".pdf"), simuler la sauvegarde (en
   affichant un message en console avec le nom original et le nom de stockage), et rediriger vers une page de succès
   avec un message flash.
7. Si une validation échoue (vide ou mauvais type), elle doit rediriger vers la page du formulaire (supposée
   `/documents/new`) avec un message d'erreur flash approprié.
8. *Ne pas implémenter la sauvegarde réelle sur disque pour cet exercice, juste l'affichage en console.*

#### Correction Exercice 3 {collapsible="true"}

**1. `application.properties` :**

```properties
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
```

**2. Dépendance Tika (si pas déjà ajoutée) :**

```xml

<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
    <version>2.9.1</version>
</dependency>
```

**3. `DocumentController` :**

```java
// fr.formation.spring.controller.DocumentController
package fr.formation.spring.controller;

import org.apache.tika.Tika;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.io.IOException;
import java.io.InputStream;
import java.util.UUID;

@Controller
@RequestMapping("/documents")
public class DocumentController {

    // Supposons une méthode GET pour afficher le formulaire à /documents/new
    // @GetMapping("/new") public String showUploadForm() { return "document-upload"; }

    @PostMapping("/upload")
    public String handleDocumentUpload(
            @RequestParam("documentFile") MultipartFile file,
            RedirectAttributes redirectAttributes
    ) {

        // 1. Vérification fichier vide
        if (file.isEmpty()) {
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Veuillez sélectionner un fichier PDF.");
            return "redirect:/documents/new"; // Retour au formulaire
        }

        // 2. Validation du type avec Tika
        Tika tika = new Tika();
        String detectedType;
        try (InputStream inputStream = file.getInputStream()) {
            // On lit via le flux pour que Tika puisse analyser
            detectedType = tika.detect(inputStream);
        } catch (IOException e) {
            e.printStackTrace(); // Logguer l'erreur
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Erreur lors de la lecture du fichier.");
            return "redirect:/documents/new";
        }

        System.out.println("Type détecté par Tika : " + detectedType); // Debug

        if (!"application/pdf".equals(detectedType)) {
            redirectAttributes.addFlashAttribute("errorMessage",
                    "Fichier invalide. Seuls les documents PDF sont autorisés. Type détecté: " + detectedType);
            return "redirect:/documents/new";
        }

        // 3. Génération nom unique et "sauvegarde" simulée
        String originalFilename = file.getOriginalFilename();
        // Nettoyage basique pour l'affichage (pas pour le nom de stockage)
        originalFilename = (originalFilename != null) ?
                originalFilename.replaceAll("[^a-zA-Z0-9.\\-_]", "_") : "unknown";

        String storedFilename = UUID.randomUUID().toString() + ".pdf";

        System.out.println("Sauvegarde simulée :");
        System.out.println("  Nom original : " + originalFilename);
        System.out.println("  Nom de stockage : " + storedFilename);
        System.out.println("  Taille : " + file.getSize() + " octets");
        System.out.println("  Type validé : " + detectedType);
        // Ici, on copierait le fichier vers son emplacement de stockage final
        // try { Files.copy(...) } catch ...

        redirectAttributes.addFlashAttribute("successMessage",
                "Document PDF '" + originalFilename + "' uploadé avec succès sous le nom " + storedFilename);
        return "redirect:/documents/upload-success"; // Vers une page de succès
    }

    // Supposons une méthode GET pour la page de succès
    // @GetMapping("/upload-success") public String showSuccess() { return "document-success"; }

}
```

---

## Sécurité des Formulaires

Les formulaires web sont une cible privilégiée pour les attaquants. Il est impératif de comprendre les menaces courantes
et d'implémenter les mesures de sécurité appropriées.

### Attaques Courantes

#### Cross-Site Scripting (XSS)

* **Description :** L'attaquant injecte du code malveillant (généralement du JavaScript) dans une page web via un
  formulaire (ou un autre vecteur d'entrée comme les paramètres d'URL). Ce script est ensuite exécuté par le navigateur
  d'un utilisateur légitime qui visite la page compromise.
* **Types :**
    * **Reflected XSS :** Le script est injecté via une requête (ex: paramètre d'URL) et est immédiatement renvoyé et
      exécuté dans la réponse du serveur. `http://example.com/search?query=<script>alert('XSS')</script>`
    * **Stored XSS :** Le script malveillant est stocké de manière permanente sur le serveur (ex: dans un commentaire de
      blog, un profil utilisateur) et est exécuté chaque fois qu'un utilisateur affiche les données compromises. C'est
      le type le plus dangereux.
    * **DOM-based XSS :** La vulnérabilité se situe dans le code JavaScript côté client qui manipule le DOM de manière
      non sécurisée en utilisant des données contrôlées par l'utilisateur.
* **Impact :** Vol de sessions (cookies), redirection vers des sites malveillants, modification du contenu de la page,
  enregistrement des frappes clavier (keylogging).
* **Mitigation :** Voir ci-dessous (Encodage des sorties, Nettoyage des entrées, Content Security Policy).

#### Cross-Site Request Forgery (CSRF)

* **Description :** L'attaquant force le navigateur d'un utilisateur authentifié à envoyer une requête HTTP non désirée
  vers une application web sur laquelle l'utilisateur est déjà connecté. L'application ne peut pas distinguer une
  requête légitime d'une requête forgée si elle se base uniquement sur les cookies de session.
* **Exemple :** L'utilisateur est connecté à sa banque `mybank.com`. Il visite ensuite un site malveillant `evil.com`.
  Ce site contient une image ou un formulaire caché qui déclenche une requête POST vers
  `mybank.com/transfer?to=attacker&amount=1000`. Comme l'utilisateur est authentifié sur `mybank.com`, son navigateur
  envoie automatiquement les cookies de session, et la banque exécute le transfert.
* **Impact :** Actions non autorisées au nom de l'utilisateur (transferts d'argent, modification d'email/mot de passe,
  suppression de données).
* **Mitigation :** Voir ci-dessous (Tokens CSRF).

#### Mass Assignment / Over-Posting

* **Description :** Concerne le mécanisme de data binding qui mappe automatiquement les données d'une requête HTTP (
  formulaire, JSON) vers les propriétés d'un objet côté serveur. Si l'application utilise directement une entité du
  domaine (ex: Entité JPA `User` avec un champ `isAdmin`) et ne contrôle pas quels champs peuvent être liés, un
  attaquant peut ajouter des champs malveillants à la requête (ex: `isAdmin=true`) même s'ils ne sont pas présents dans
  le formulaire HTML. Le binder peut alors initialiser ou modifier des propriétés sensibles de l'objet.
* **Impact :** Élévation de privilèges, modification de données non autorisées.
* **Mitigation :** Utilisation de DTOs (Data Transfer Objects) qui n'exposent que les champs attendus du formulaire. Ne
  jamais binder directement sur des entités JPA ou des objets domaine complexes contenant des champs sensibles.

#### Parameter Tampering

* **Description :** L'attaquant modifie les paramètres envoyés au serveur qui ne sont pas censés être modifiés par l'
  utilisateur normal. Cela inclut :
    * Les valeurs des champs cachés (`<input type="hidden">`).
    * Les valeurs sélectionnées dans des listes déroulantes ou des boutons radio (ex: changer l'ID d'un produit à un
      produit auquel l'utilisateur n'a pas accès).
    * Les paramètres dans l'URL (query parameters).
* **Impact :** Accès non autorisé à des données, contournement de la logique métier, modification de prix, etc.
* **Mitigation :** Ne jamais faire confiance aux données venant du client. Toujours re-valider côté serveur :
    * Vérifier que l'utilisateur a le droit d'accéder/modifier l'objet identifié par un ID reçu.
    * Re-vérifier le prix d'un produit côté serveur, ne pas se fier au prix envoyé par le formulaire.
    * Valider que les valeurs sélectionnées dans les listes/radios sont bien parmi les options autorisées pour cet
      utilisateur.

### Stratégies de Mitigations

#### Protection CSRF (Spring Security)

* **Principe :** Utiliser le **Synchronizer Token Pattern**.
    1. Lorsqu'un formulaire est affiché (GET), le serveur génère un token CSRF unique et imprévisible, le stocke côté
       serveur (souvent dans la session HTTP), et l'inclut dans le formulaire sous forme de champ caché.
    2. Lorsque le formulaire est soumis (POST), le client renvoie ce token.
    3. Le serveur compare le token reçu avec celui stocké en session. S'ils correspondent, la requête est considérée
       comme légitime. Sinon, elle est rejetée.
* **Implémentation avec Spring Security :**
    * Spring Security active la protection CSRF par défaut pour les méthodes modifiant l'état (`POST`, `PUT`, `DELETE`).
    * Il s'attend à ce que le token soit présent dans la requête (soit comme paramètre `_csrf`, soit comme en-tête
      `X-CSRF-TOKEN`).
    * **Intégration Thymeleaf :** Si Spring Security et Thymeleaf sont utilisés, Thymeleaf ajoute automatiquement le
      champ caché `_csrf` aux formulaires en `POST` si l'attribut `th:action` est utilisé.
      ```html
      <!-- Si Spring Security CSRF est actif, Thymeleaf ajoutera ceci : -->
      <!-- <input type="hidden" name="_csrf" value="token-généré-par-le-serveur"/> -->
      <form th:action="@{/users/register}" th:object="${userDto}" method="post">
         <!-- ... champs du formulaire ... -->
      </form>
      ```
    * **Désactivation (NON RECOMMANDÉ pour les applications stateful) :** Si nécessaire (ex: API REST stateless), la
      protection peut être désactivée dans la configuration de sécurité, mais cela ouvre la porte aux attaques CSRF si
      d'autres mécanismes (ex: vérification de l'en-tête `Origin`) ne sont pas mis en place.
      ```java
      // @Configuration @EnableWebSecurity class SecurityConfig {
      //    @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
      //        http.csrf(csrf -> csrf.disable()); // Désactive CSRF
      //        // ... reste de la configuration ...
      //        return http.build();
      //    }
      // }
      ```
* **Conseil :** Laisser la protection CSRF activée par défaut pour les applications web traditionnelles avec sessions.
  S'assurer que tous les formulaires POST utilisent `th:action` ou incluent manuellement le token si nécessaire.

#### Validation Robuste (Côté Serveur)

* **Principe :** La validation côté client (JavaScript) est utile pour l'expérience utilisateur mais ne fournit **aucune
  sécurité**. Elle peut être facilement contournée. **Toute validation doit être impérativement dupliquée et renforcée
  côté serveur.**
* **Implémentation :**
    * Utiliser Bean Validation (`@Valid`, contraintes standard et personnalisées) sur les DTOs reçus par le contrôleur.
    * Vérifier les types de données, les longueurs, les formats, les plages de valeurs.
    * Valider les relations entre les champs (ex: date de fin après date de début).
    * Valider la logique métier (ex: l'email existe-t-il déjà ? Le stock est-il suffisant ?).
    * Valider les autorisations (ex: l'utilisateur a-t-il le droit de modifier cet objet spécifique ?). Cf. Parameter
      Tampering.

#### Utilisation de DTOs

* **Principe :** Comme mentionné précédemment, les DTOs agissent comme une couche intermédiaire qui n'expose que les
  champs strictement nécessaires au formulaire.
* **Implémentation :** Définir des DTOs spécifiques pour chaque formulaire (ou groupe de formulaires similaires). Mapper
  manuellement ou avec des outils (MapStruct, ModelMapper) entre le DTO et l'entité du domaine dans la couche service,
  en ne transférant que les champs pertinents et en appliquant la logique nécessaire (ex: hashage de mot de passe).
* **Avantage Sécurité :** Empêche les attaques de type Mass Assignment car seuls les champs définis dans le DTO peuvent
  être liés à partir de la requête.

#### Encodage des Sorties / Nettoyage des Entrées

* **Principe (Contre XSS) :**
    * **Encodage des Sorties (Output Encoding - Le plus important) :** Lorsque des données fournies par l'utilisateur
      sont affichées dans une page HTML, elles doivent être encodées pour que le navigateur les interprète comme du
      texte littéral et non comme du code HTML ou JavaScript. Par exemple, `<` devient `&lt;`, `>` devient `&gt;`, `"`
      devient `&quot;`.
    * **Nettoyage des Entrées (Input Sanitization) :** Si l'application doit accepter et stocker du HTML formaté par l'
      utilisateur (ex: éditeur de texte riche), l'encodage seul ne suffit pas. Il faut nettoyer (filtrer) le HTML
      entrant pour n'autoriser qu'un sous-ensemble sûr de balises et d'attributs, en supprimant tout code
      potentiellement dangereux (scripts, gestionnaires d'événements `onerror`, etc.).
* **Implémentation :**
    * **Thymeleaf :** Par défaut, Thymeleaf effectue automatiquement l'encodage HTML des expressions `th:text` et des
      valeurs de champ `th:field`, `th:value`, etc. C'est une protection XSS majeure intégrée. Utiliser `th:utext` (
      unescaped text) avec une extrême prudence et uniquement pour du HTML dont la sécurité a été vérifiée en amont.
    * **Nettoyage :** Utiliser des bibliothèques dédiées comme **OWASP Java HTML Sanitizer**. Configurer une politique
      de nettoyage stricte (whitelisting des éléments autorisés).
      ```java
      // Exemple avec OWASP Java HTML Sanitizer
      // PolicyFactory policy = new HtmlPolicyBuilder()
      //     .allowElements("p", "b", "i", "em", "strong", "a")
      //     .allowAttributes("href").onElements("a")
      //     .requireRelNofollowOnLinks()
      //     .toFactory();
      // String unsafeHtmlInput = "<p onclick='alert(\"XSS\")'>Hello <script>...</script></p><a href='javascript:bad()'>link</a>";
      // String safeHtmlOutput = policy.sanitize(unsafeHtmlInput);
      // // safeHtmlOutput contiendrait : "<p>Hello </p><a>link</a>" (selon la policy exacte)
      ```
* **Content Security Policy (CSP) :** Un en-tête HTTP (`Content-Security-Policy`) qui permet de définir des règles très
  précises sur les ressources (scripts, styles, images, etc.) que le navigateur est autorisé à charger et exécuter pour
  une page. C'est une défense en profondeur très efficace contre XSS et d'autres attaques par injection. Sa
  configuration peut être complexe mais est fortement recommandée.

#### HTTPS

* **Principe :** Chiffrer toute communication entre le navigateur de l'utilisateur et le serveur web en utilisant
  TLS/SSL.
* **Implémentation :** Configurer le serveur web (Tomcat, Nginx, Apache) pour utiliser un certificat TLS/SSL valide.
  Forcer la redirection de HTTP vers HTTPS.
* **Avantage Sécurité :** Protège les données des formulaires (identifiants, informations personnelles, tokens CSRF)
  contre l'écoute (eavesdropping) sur le réseau. Empêche certaines attaques de type "man-in-the-middle". C'est un
  prérequis fondamental pour toute application manipulant des données sensibles.

#### Mise à jour des Dépendances

* **Principe :** Les frameworks (Spring, Hibernate), les bibliothèques (Tika, ModelMapper), le serveur d'application et
  le JRE lui-même peuvent contenir des vulnérabilités de sécurité connues.
* **Implémentation :** Utiliser des outils d'analyse de dépendances (comme OWASP Dependency-Check, Snyk, Dependabot de
  GitHub) pour identifier les bibliothèques avec des vulnérabilités connues. Mettre régulièrement à jour les dépendances
  vers des versions corrigées. Suivre les annonces de sécurité des projets utilisés.

---

## Bonnes Pratiques et Patterns

Au-delà de la sécurité et des bases fonctionnelles, plusieurs bonnes pratiques et patterns améliorent la qualité, la
maintenabilité et la robustesse du code de gestion des formulaires.

### Utiliser les DTOs

* **Rappel :** Séparer les données de la vue/formulaire des données du domaine/persistance.
* **Avantages :** Découplage, sécurité (Mass Assignment), flexibilité, validation ciblée.
* **Conseil :** Créer des DTOs spécifiques pour chaque cas d'usage (création, modification, affichage) si leurs
  structures de données diffèrent significativement.

### Valider Systématiquement Côté Serveur

* **Rappel :** La validation côté client est cosmétique. La vraie validation se fait côté serveur.
* **Avantages :** Intégrité des données, sécurité.
* **Conseil :** Utiliser Bean Validation pour les validations structurelles et de format. Ajouter des validations métier
  dans la couche service (unicité, règles complexes, autorisations).

### Pattern Post-Redirect-Get (PRG)

* **Problème :** Après une soumission de formulaire réussie via POST, si l'utilisateur rafraîchit la page (F5), le
  navigateur peut proposer de renvoyer les mêmes données POST, conduisant à une double soumission (ex: créer deux fois
  le même objet).
* **Solution (PRG) :**
    1. Le client envoie la requête **POST** avec les données du formulaire.
    2. Le serveur traite la requête.
    3. Si le traitement réussit, au lieu de renvoyer directement une page HTML, le serveur renvoie une réponse de *
       *REDIRECT** (code 302 ou 303) vers une nouvelle URL (souvent une page de succès ou la vue de l'objet
       créé/modifié).
    4. Le navigateur suit cette redirection en effectuant une requête **GET** vers la nouvelle URL.
    5. Le serveur répond à cette requête GET avec la page de succès ou d'affichage.
* **Avantages :** Empêche la double soumission au rafraîchissement, sépare l'action de modification (POST) de l'action
  d'affichage (GET), URL plus propre dans le navigateur après l'action.
* **Implémentation :** Dans le contrôleur, après un traitement POST réussi, retourner une chaîne commençant par
  `redirect:`.
  ```java
  @PostMapping("/register")
  public String processRegistration(
      @Valid @ModelAttribute("userDto") UserRegistrationDto userDto,
      BindingResult bindingResult,
      RedirectAttributes redirectAttributes // Pour les messages flash
  ) {
      if (bindingResult.hasErrors()) {
          return "user-registration"; // Pas de redirect si erreur
      }

      try {
          // userService.registerNewUser(userDto);
          // Ajouter un message de succès pour la page redirigée
          redirectAttributes.addFlashAttribute("globalSuccessMessage",
              "Inscription réussie pour " + userDto.getUsername());
          // Rediriger vers une page de confirmation/succès (via GET)
          return "redirect:/users/registration-success"; // PRG en action!
      } catch (Exception e) {
          // Gérer l'erreur, potentiellement rediriger vers le formulaire avec erreur
          redirectAttributes.addFlashAttribute("globalErrorMessage",
              "Erreur lors de l'inscription.");
          return "redirect:/users/register"; // Ou retourner la vue directement
      }
  }

  @GetMapping("/registration-success")
  public String showSuccessPage(Model model) {
      // Le message flash ajouté via redirectAttributes est automatiquement
      // disponible dans le modèle de la requête GET suivant la redirection
      return "registration-success"; // Template qui affiche le message de succès
  }
  ```

### Garder les Contrôleurs Légers

* **Principe :** Le contrôleur doit agir comme un chef d'orchestre : recevoir la requête, extraire les données
  pertinentes (DTO), appeler les services métier appropriés pour la logique et la validation métier, puis sélectionner
  la vue ou la redirection.
* **À éviter dans les contrôleurs :** Logique métier complexe, accès direct aux repositories JPA, manipulation de bas
  niveau des requêtes/réponses (sauf cas spécifiques), construction manuelle de HTML/JSON.
* **Avantages :** Meilleure séparation des préoccupations, code plus testable (les services peuvent être testés
  unitairement sans le contexte web), code plus lisible et maintenable.
* **Implémentation :** Créer des classes de service (`@Service`) injectées dans les contrôleurs. Les services
  contiennent la logique métier, la coordination des repositories, le mapping DTO/Entité, etc.

### Gestion Centralisée des Erreurs

* **Problème :** Gérer toutes les exceptions possibles (validation, accès aux données, erreurs métier, upload trop
  volumineux) dans chaque méthode de contrôleur peut être répétitif et encombrant.
* **Solution :** Utiliser `@ControllerAdvice` et `@ExceptionHandler`. Une classe annotée `@ControllerAdvice` peut
  contenir des méthodes `@ExceptionHandler` qui capturent des exceptions spécifiques levées par n'importe quel
  contrôleur de l'application.
* **Exemple : Gestion de `MaxUploadSizeExceededException`**
  ```java
  // fr.formation.spring. MvcExceptionHandler
  package fr.formation.spring.exception;

  import org.springframework.web.bind.annotation.ControllerAdvice;
  import org.springframework.web.bind.annotation.ExceptionHandler;
  import org.springframework.web.multipart.MaxUploadSizeExceededException;
  import org.springframework.web.servlet.mvc.support.RedirectAttributes;
  import org.springframework.http.HttpStatus;
  import org.springframework.web.servlet.ModelAndView; // Alternative à Redirect
  import jakarta.servlet.http.HttpServletRequest;

  @ControllerAdvice
  public class MvcExceptionHandler {

      @ExceptionHandler(MaxUploadSizeExceededException.class)
      public String handleMaxSizeException(
          MaxUploadSizeExceededException exc,
          RedirectAttributes redirectAttributes,
          HttpServletRequest request // Pour obtenir l'URL d'origine
      ) {
          System.err.println("Erreur: Fichier trop volumineux ! Limite: " + exc.getMaxUploadSize());
          redirectAttributes.addFlashAttribute("errorMessage",
              "Le fichier dépasse la taille maximale autorisée.");

          // Essayer de rediriger vers la page d'où venait l'upload
          // Note: L'URL exacte peut être complexe à déterminer dans tous les cas
          String referer = request.getHeader("Referer");
          if (referer != null && !referer.isEmpty()) {
               return "redirect:" + referer;
          } else {
              // Fallback vers une page d'erreur générique ou l'accueil
              return "redirect:/error-upload";
          }

          // Alternative: retourner un ModelAndView vers une page d'erreur spécifique
          // ModelAndView mav = new ModelAndView("error-upload-size");
          // mav.addObject("maxSize", exc.getMaxUploadSize());
          // mav.setStatus(HttpStatus.PAYLOAD_TOO_LARGE);
          // return mav;
      }

      // Ajouter d'autres @ExceptionHandler pour d'autres types d'exceptions...
      // @ExceptionHandler(DataAccessException.class) public String handleDatabaseError(...)
      // @ExceptionHandler(Exception.class) public String handleGenericError(...)
  }
  ```
* **Avantages :** Centralise la logique de gestion des erreurs, nettoie le code des contrôleurs, permet de fournir des
  réponses d'erreur cohérentes (pages d'erreur spécifiques, redirections).

### Internationalisation (i18n) des Messages

* **Principe :** Rendre les messages affichés à l'utilisateur (labels de formulaire, messages d'erreur de validation,
  messages de succès/échec) traduisibles dans différentes langues.
* **Implémentation avec Spring Boot :**
    1. Créer des fichiers de propriétés pour chaque langue dans `src/main/resources`, en suivant la convention
       `messages_XX.properties` (où `XX` est le code langue, ex: `messages_fr.properties`, `messages_en.properties`) et
       un fichier par défaut `messages.properties`.
    2. Définir des clés et leurs traductions dans ces fichiers.
       ```properties
       # messages.properties (Défaut, ex: Anglais)
       label.username=Username
       label.password=Password
       error.required=This field is required.
       error.email.invalid=Invalid email format.
       validation.passwords.mismatch=Passwords do not match.
       ```
       ```properties
       # messages_fr.properties (Français)
       label.username=Nom d'utilisateur
       label.password=Mot de passe
       error.required=Ce champ est obligatoire.
       error.email.invalid=Format d'email invalide.
       validation.passwords.mismatch=Les mots de passe ne correspondent pas.
       ```
    3. Configurer `MessageSource` (Spring Boot le fait souvent automatiquement s'il trouve les fichiers
       `messages*.properties`).
    4. **Pour les messages de validation Bean Validation :** Les messages définis dans les annotations (
       `message = "..."`) peuvent être des clés de messages. Placez `{key}` dans l'attribut `message`. Bean Validation
       utilisera le `MessageSource` de Spring pour résoudre ces clés.
       ```java
       // Dans UserRegistrationDto
       @NotBlank(message = "{error.required}") // Utilise la clé du fichier properties
       private String username;

       @Email(message = "{error.email.invalid}")
       private String email;

       // Dans l'annotation @PasswordsMatch
       // String message() default "{validation.passwords.mismatch}";
       ```
    5. **Dans Thymeleaf :** Utiliser la syntaxe `#{key}` pour afficher des messages internationalisés.
       ```html
       <label for="username" th:text="#{label.username}">Username</label>
       <input type="text" id="username" th:field="*{username}" />
       <!-- Les erreurs de validation résolues via les clés s'affichent directement -->
       <div class="error" th:if="${#fields.hasErrors('username')}">
            <p th:each="err : ${#fields.errors('username')}" th:text="${err}"></p>
       </div>
       ```
* **Avantages :** Application adaptable à différents publics, gestion centralisée des textes.

---

## Conclusion

La gestion des formulaires est une partie essentielle du développement web. Spring Boot et Spring MVC, combinés à des
outils comme Thymeleaf, Bean Validation, et des bibliothèques de mapping (ModelMapper, MapStruct), fournissent un
écosystème puissant et cohérent pour gérer l'ensemble du cycle de vie des formulaires, de l'affichage à la validation,
en passant par la sécurité et le traitement des données.

Maîtriser l'utilisation des DTOs, implémenter une validation robuste côté serveur, comprendre et mitiger les risques de
sécurité (XSS, CSRF, Mass Assignment), gérer correctement les uploads de fichiers et appliquer les bonnes pratiques
comme le pattern PRG sont des compétences clés pour développer des applications web fiables et sécurisées avec Spring.
