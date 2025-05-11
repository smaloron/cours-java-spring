# La gestion des formulaires

**1. Introduction aux Formulaires dans Spring MVC**

Les formulaires HTML sont le moyen principal par lequel les utilisateurs interagissent et soumettent des données aux
applications web. Spring MVC fournit un ensemble robuste de fonctionnalités pour simplifier la gestion de ces
formulaires, depuis l'affichage initial jusqu'au traitement des données soumises et à la gestion des erreurs de
validation.

Le flux typique est le suivant :

1. **Requête GET :** L'utilisateur accède à une URL. Le contrôleur Spring prépare un objet (souvent un DTO vide) et le
   transmet à la vue (par exemple, une page Thymeleaf).
2. **Affichage du Formulaire :** La vue utilise l'objet fourni pour afficher un formulaire HTML vierge ou pré-rempli.
   Des balises spéciales (comme celles de Thymeleaf : `th:object`, `th:field`) facilitent la liaison entre les champs du
   formulaire et les propriétés de l'objet.
3. **Soumission POST/PUT :** L'utilisateur remplit le formulaire et le soumet. Le navigateur envoie une requête POST (ou
   PUT) au serveur avec les données du formulaire encodées dans le corps de la requête.
4. **Data Binding :** Spring MVC intercepte la requête et tente de "lier" (mapper) les données reçues aux propriétés
   d'un objet Java (généralement le même type de DTO utilisé pour l'affichage).
5. **Validation :** Si la validation est activée, Spring exécute les règles de validation définies sur l'objet DTO.
6. **Traitement Métier :**
    * Si la validation échoue, le contrôleur retourne généralement à la vue du formulaire, en y incluant l'objet DTO (
      qui contient maintenant les données soumises par l'utilisateur) et les informations sur les erreurs de validation.
      La vue affiche alors le formulaire pré-rempli avec les messages d'erreur.
    * Si la validation réussit, le contrôleur traite les données (par exemple, sauvegarde en base de données via une
      couche de service) et redirige généralement l'utilisateur vers une autre page (page de succès, liste mise à jour,
      etc.) en utilisant le pattern Post-Redirect-Get (PRG).

---

**2. Le Data Binding et les Data Transfer Objects (DTO)**

**Data Binding :** C'est le processus automatique par lequel Spring MVC mappe les paramètres d'une requête HTTP (venant
d'un formulaire, de l'URL, etc.) aux propriétés d'un objet Java, souvent appelé "Command Object" ou "Form Backing Bean".
Dans la pratique moderne, cet objet est très souvent un DTO. Spring utilise les noms des paramètres de la requête pour
trouver les propriétés correspondantes dans l'objet (via les setters ou l'accès direct aux champs si configuré).

**Data Transfer Objects (DTO) :** Un DTO est un objet simple (souvent un POJO - Plain Old Java Object) dont le but
principal est de transférer des données entre différentes couches d'une application (par exemple, entre le contrôleur et
la vue, ou entre le contrôleur et la couche service).

**Pourquoi utiliser des DTO pour les formulaires ?**

1. **Séparation des préoccupations :** Le DTO représente spécifiquement les données *attendues* du formulaire, qui
   peuvent différer de la structure de l'entité persistante (JPA Entity). Par exemple, un formulaire d'inscription peut
   demander une confirmation de mot de passe, qui n'existe pas dans l'entité `User`.
2. **Sécurité (Prévention du Mass Assignment/Over-Posting) :** En liant les données du formulaire à un DTO, on ne lie
   que les champs explicitement définis dans le DTO. Cela empêche un attaquant d'injecter des valeurs pour des champs de
   l'entité qui ne sont pas censés être modifiables via ce formulaire (par exemple, `isAdmin`, `accountBalance`). On
   expose uniquement ce qui est nécessaire.
3. **Flexibilité de la Vue :** Le DTO peut être adapté aux besoins spécifiques de la vue, en agrégeant des données ou en
   les formatant différemment de l'entité.
4. **Validation Ciblée :** Les contraintes de validation peuvent être appliquées directement sur le DTO, reflétant les
   règles métier spécifiques au contexte du formulaire.

**Exemple :**

Supposons une entité JPA `Product`:

```java
package fr.formation.spring.entity;

import jakarta.persistence.*;

import java.math.BigDecimal;

@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String sku; // Stock Keeping Unit

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    private boolean active; // Champ non exposé dans le formulaire de création

    // Getters et Setters...
}
```

On crée un DTO pour le formulaire de création de produit :

```java
package fr.formation.spring.dto;

import java.math.BigDecimal;

// DTO pour la création ou la mise à jour d'un produit
public class ProductDTO {

    private String sku;
    private String name;
    private String description;
    private BigDecimal price;

    // Constructeur vide nécessaire pour Spring MVC data binding
    public ProductDTO() {
    }

    // Getters et Setters...

    public String getSku() {
        return sku;
    }

    public void setSku(String sku) {
        this.sku = sku;
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

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }
}
```

**Contrôleur utilisant le DTO :**

```java
package fr.formation.spring.controller;

import fr.formation.spring.dto.ProductDTO;
import fr.formation.spring.service.ProductService; // Suppose un service
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import jakarta.validation.Valid; // Important pour la validation

@Controller
@RequestMapping("/products")
public class ProductController {

    private final ProductService productService;

    // Injection de dépendance du service (constructeur recommandé)
    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // Affiche le formulaire de création
    @GetMapping("/new")
    public String showCreateForm(Model model) {
        // Ajoute un DTO vide au modèle pour le data binding du formulaire
        model.addAttribute("productDTO", new ProductDTO());
        return "product-form"; // Nom de la vue Thymeleaf
    }

    // Traite la soumission du formulaire
    @PostMapping("/create")
    public String createProduct(
            @Valid @ModelAttribute("productDTO") ProductDTO productDTO,
            BindingResult bindingResult, // Doit suivre immédiatement l'objet validé
            Model model) {

        // Vérifie s'il y a des erreurs de validation
        if (bindingResult.hasErrors()) {
            // S'il y a des erreurs, retourne à la vue du formulaire
            // Le productDTO contient déjà les données soumises par l'utilisateur
            // Thymeleaf utilisera bindingResult pour afficher les erreurs
            return "product-form";
        }

        // Si la validation réussit, appelle le service pour créer le produit
        // Il faudra mapper le DTO vers l'entité Product ici (voir section Mapping)
        productService.createProductFromDTO(productDTO);

        // Redirige vers une page de succès ou la liste des produits (Pattern PRG)
        return "redirect:/products/list"; // Exemple de redirection
    }

    // Méthode pour afficher la liste (pour la redirection) - exemple simple
    @GetMapping("/list")
    public String listProducts(Model model) {
        // Logique pour récupérer et afficher la liste des produits...
        model.addAttribute("message", "Produit créé avec succès !"); // Optionnel
        // model.addAttribute("products", productService.findAll());
        return "product-list"; // Vue affichant la liste
    }
}
```

**Vue Thymeleaf (`product-form.html`):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Créer un Produit</title>
    <style>.error {
        color: red;
    }</style> <!-- Style simple pour les erreurs -->
</head>
<body>
<h1>Créer un Nouveau Produit</h1>

<!-- Le formulaire est lié à l'objet 'productDTO' du modèle -->
<!-- th:action spécifie l'URL de soumission -->
<form th:action="@{/products/create}" th:object="${productDTO}" method="post">

    <div>
        <label for="sku">SKU:</label>
        <!-- th:field lie l'input au champ 'sku' de productDTO -->
        <!-- Génère id, name et value automatiquement -->
        <input type="text" id="sku" th:field="*{sku}"/>
        <!-- Affiche les erreurs de validation pour le champ 'sku' -->
        <span class="error" th:if="${#fields.hasErrors('sku')}"
              th:errors="*{sku}">Erreur SKU</span>
    </div>

    <div>
        <label for="name">Nom:</label>
        <input type="text" id="name" th:field="*{name}"/>
        <span class="error" th:if="${#fields.hasErrors('name')}"
              th:errors="*{name}">Erreur Nom</span>
    </div>

    <div>
        <label for="description">Description:</label>
        <textarea id="description" th:field="*{description}"></textarea>
        <!-- Pas de validation spécifique ici, mais on pourrait en ajouter -->
        <span class="error" th:if="${#fields.hasErrors('description')}"
              th:errors="*{description}">Erreur Description</span>
    </div>

    <div>
        <label for="price">Prix:</label>
        <input type="number" step="0.01" id="price" th:field="*{price}"/>
        <span class="error" th:if="${#fields.hasErrors('price')}"
              th:errors="*{price}">Erreur Prix</span>
    </div>

    <div>
        <button type="submit">Créer Produit</button>
    </div>
</form>
</body>
</html>
```

---

**3. Validation des Données avec Bean Validation**

La validation des données soumises est cruciale pour garantir l'intégrité des données et la sécurité de l'application.
Spring s'intègre nativement avec l'API Bean Validation (JSR 380 / Jakarta Bean Validation). Hibernate Validator est
l'implémentation de référence, généralement incluse par défaut dans `spring-boot-starter-web`.

**Comment ça marche ?**

1. **Ajouter les dépendances :** Normalement incluses avec `spring-boot-starter-web`. Si ce n'est pas le cas, ajoutez
   `spring-boot-starter-validation`.
2. **Annoter le DTO :** Ajoutez des annotations de validation aux champs du DTO.
3. **Activer la validation dans le Contrôleur :** Annotez le paramètre DTO dans la méthode du contrôleur avec `@Valid` (
   ou `@Validated` qui est spécifique à Spring et offre des fonctionnalités de groupe).
4. **Gérer les erreurs :** Ajoutez un paramètre `BindingResult` juste après le paramètre DTO annoté. Spring y placera
   les résultats de la validation. Vérifiez `bindingResult.hasErrors()` pour savoir si la validation a échoué.
5. **Afficher les erreurs dans la Vue :** Utilisez les fonctionnalités de votre moteur de template (par exemple,
   `th:errors` et `#fields.hasErrors()` dans Thymeleaf) pour afficher les messages d'erreur à l'utilisateur.

**Annotations de Validation Courantes :**

* `@NotNull` : Ne doit pas être nul.
* `@NotEmpty` : Ne doit pas être nul et ne doit pas être vide (utile pour les Collections, Maps, ou tableaux). Pour les
  Strings, préférer `@NotBlank`.
* `@NotBlank` : Ne doit pas être nul et doit contenir au moins un caractère non blanc (trim appliqué). Idéal pour les
  chaînes de caractères obligatoires.
* `@Size(min=..., max=...)` : Pour la taille (String, Collection, Map, Array).
* `@Min(value=...)` : Valeur numérique minimale.
* `@Max(value=...)` : Valeur numérique maximale.
* `@Positive`, `@PositiveOrZero` : Pour les nombres positifs / positifs ou zéro.
* `@Negative`, `@NegativeOrZero` : Pour les nombres négatifs / négatifs ou zéro.
* `@Email` : Doit être une adresse email bien formée.
* `@Pattern(regexp=...)` : Doit correspondre à l'expression régulière Java spécifiée.
* `@Future`, `@Past`, `@FutureOrPresent`, `@PastOrPresent` : Pour les dates/heures.
* `@Digits(integer=..., fraction=...)` : Valide que le nombre a un nombre maximum de chiffres entiers et fractionnaires.
* `@AssertTrue`, `@AssertFalse` : Pour les booléens.

**Exemple avec Validation sur `ProductDTO` :**

```java
package fr.formation.spring.dto;

import java.math.BigDecimal;

import jakarta.validation.constraints.*; // Import des annotations

public class ProductDTO {

    @NotBlank(message = "Le SKU ne peut pas être vide.")
    @Size(min = 3, max = 50, message = "Le SKU doit contenir entre {min} et {max} caractères.")
    @Pattern(regexp = "^[a-zA-Z0-9-]+$", message = "Le SKU ne peut contenir que des lettres, chiffres et tirets.")
    private String sku;

    @NotBlank(message = "Le nom du produit est requis.")
    @Size(max = 100, message = "Le nom ne doit pas dépasser {max} caractères.")
    private String name;

    @Size(max = 500, message = "La description ne doit pas dépasser {max} caractères.")
    private String description; // Description peut être vide, donc pas @NotBlank

    @NotNull(message = "Le prix est requis.")
    @PositiveOrZero(message = "Le prix doit être positif ou nul.")
    @Digits(integer = 8, fraction = 2, message = "Le prix doit avoir au maximum {integer} chiffres avant la virgule et {fraction} après.")
    private BigDecimal price;

    // Getters et Setters...
}
```

**Points Clés :**

* L'attribut `message` permet de personnaliser les messages d'erreur. On peut utiliser des placeholders comme `{min}`,
  `{max}`, `{value}`, etc.
* Il est fortement recommandé d'externaliser les messages d'erreur dans des fichiers de propriétés (
  `messages.properties`) pour l'internationalisation (i18n). Spring Boot le configure souvent automatiquement. Le
  message défini dans l'annotation sert alors de code de message par défaut.
* Le contrôleur et la vue Thymeleaf montrés précédemment gèrent déjà l'affichage de ces erreurs grâce à `@Valid` et
  `BindingResult`.

**Exercice 1 : Ajout de Validation**

1. Créez un DTO `UserRegistrationDTO` avec les champs : `username` (String), `email` (String), `password` (String),
   `confirmPassword` (String), `acceptTerms` (boolean).
2. Ajoutez des annotations de validation appropriées :
    * `username`: non vide, taille entre 5 et 20 caractères.
    * `email`: non vide, format email valide.
    * `password`: non vide, taille minimale 8 caractères.
    * `confirmPassword`: non vide.
    * `acceptTerms`: doit être vrai (`@AssertTrue`).
3. Créez un contrôleur simple avec une méthode GET pour afficher un formulaire lié à ce DTO et une méthode POST pour le
   traiter, incluant `@Valid` et `BindingResult`.
4. Créez une vue Thymeleaf simple pour afficher le formulaire et les messages d'erreur.

**Correction Exercice 1**

**UserRegistrationDTO.java:**

```java
package fr.formation.spring.dto;

import jakarta.validation.constraints.*;

public class UserRegistrationDTO {

    @NotBlank(message = "Le nom d'utilisateur est requis.")
    @Size(min = 5, max = 20, message = "Le nom d'utilisateur doit faire entre {min} et {max} caractères.")
    private String username;

    @NotBlank(message = "L'adresse email est requise.")
    @Email(message = "Veuillez fournir une adresse email valide.")
    private String email;

    @NotBlank(message = "Le mot de passe est requis.")
    @Size(min = 8, message = "Le mot de passe doit contenir au moins {min} caractères.")
    private String password;

    @NotBlank(message = "La confirmation du mot de passe est requise.")
    private String confirmPassword; // Validation de correspondance sera faite via custom validator

    @AssertTrue(message = "Vous devez accepter les conditions d'utilisation.")
    private boolean acceptTerms;

    // Getters et Setters...
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

    public boolean isAcceptTerms() {
        return acceptTerms;
    }

    public void setAcceptTerms(boolean acceptTerms) {
        this.acceptTerms = acceptTerms;
    }
}
```

**UserController.java:**

```java
package fr.formation.spring.controller;

import fr.formation.spring.dto.UserRegistrationDTO;
import jakarta.validation.Valid;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


@Controller
@RequestMapping("/register")
public class UserController {

    private static final Logger log = LoggerFactory.getLogger(UserController.class);

    @GetMapping
    public String showRegistrationForm(Model model) {
        model.addAttribute("userDto", new UserRegistrationDTO());
        return "registration-form";
    }

    @PostMapping
    public String processRegistration(
            @Valid @ModelAttribute("userDto") UserRegistrationDTO userDto,
            BindingResult bindingResult,
            Model model) {

        // Ici, on ajoutera la validation pour password == confirmPassword

        if (bindingResult.hasErrors()) {
            log.warn("Erreurs de validation détectées.");
            // Retour au formulaire avec les erreurs
            return "registration-form";
        }

        // Logique métier (ex: enregistrer l'utilisateur)
        log.info("Inscription réussie pour : {}", userDto.getUsername());
        // Redirection vers une page de succès
        return "redirect:/register/success";
    }

    @GetMapping("/success")
    public String registrationSuccess() {
        return "registration-success"; // Vue simple de succès
    }
}
```

**registration-form.html (Thymeleaf):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Inscription</title>
    <style>.error {
        color: red;
        margin-left: 10px;
    }</style>
</head>
<body>
<h1>Formulaire d'Inscription</h1>
<form th:action="@{/register}" th:object="${userDto}" method="post">
    <div>
        <label for="username">Nom d'utilisateur:</label>
        <input type="text" id="username" th:field="*{username}"/>
        <span class="error" th:if="${#fields.hasErrors('username')}"
              th:errors="*{username}"></span>
    </div>
    <div>
        <label for="email">Email:</label>
        <input type="email" id="email" th:field="*{email}"/>
        <span class="error" th:if="${#fields.hasErrors('email')}"
              th:errors="*{email}"></span>
    </div>
    <div>
        <label for="password">Mot de passe:</label>
        <input type="password" id="password" th:field="*{password}"/>
        <span class="error" th:if="${#fields.hasErrors('password')}"
              th:errors="*{password}"></span>
    </div>
    <div>
        <label for="confirmPassword">Confirmer Mot de passe:</label>
        <input type="password" id="confirmPassword" th:field="*{confirmPassword}"/>
        <!-- Erreur de correspondance sera ajoutée via validateur custom -->
        <span class="error" th:if="${#fields.hasErrors('confirmPassword')}"
              th:errors="*{confirmPassword}"></span>
        <span class="error" th:if="${#fields.hasGlobalErrors()}"
              th:errors="*{global}"></span> <!-- Pour l'erreur globale -->
    </div>
    <div>
        <input type="checkbox" id="acceptTerms" th:field="*{acceptTerms}"/>
        <label for="acceptTerms">J'accepte les conditions d'utilisation</label>
        <span class="error" th:if="${#fields.hasErrors('acceptTerms')}"
              th:errors="*{acceptTerms}"></span>
    </div>
    <div>
        <button type="submit">S'inscrire</button>
    </div>
</form>
</body>
</html>
```

**registration-success.html (Thymeleaf):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Inscription Réussie</title>
</head>
<body>
<h1>Inscription Réussie !</h1>
<p>Votre compte a été créé avec succès.</p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

---

**4. Validateurs Personnalisés**

Parfois, les annotations de validation standard ne suffisent pas. Exemples courants :

* Vérifier que deux champs sont égaux (confirmation de mot de passe).
* Valider un champ en fonction de la valeur d'un autre champ.
* Effectuer une validation qui nécessite une requête en base de données (par exemple, vérifier l'unicité d'un email ou
  d'un nom d'utilisateur).

Bean Validation permet de créer ses propres contraintes. Cela se fait en trois étapes :

1. **Créer l'annotation de contrainte :** Une interface annotée avec `@Constraint`.
2. **Implémenter le validateur :** Une classe qui implémente
   `ConstraintValidator<VotreAnnotation, TypeDuChampAValider>`.
3. **Appliquer l'annotation :** Sur le champ ou la classe du DTO.

**Exemple 1 : Validation de Correspondance de Mots de Passe (Niveau Classe)**

Cette validation compare deux champs (`password` et `confirmPassword`), elle doit donc s'appliquer au niveau de la
classe DTO.

**1. L'annotation `PasswordMatches` :**

```java
package fr.formation.spring.validation.constraint;

import fr.formation.spring.validation.validator.PasswordMatchesValidator;
import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE}) // Applicable à une classe
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordMatchesValidator.class) // Lie à l'implémentation
@Documented
public @interface PasswordMatches {
    // Message d'erreur par défaut si la validation échoue
    String message() default "Les mots de passe ne correspondent pas.";

    // Groupes de validation (avancé, optionnel)
    Class<?>[] groups() default {};

    // Payload (avancé, optionnel)
    Class<? extends Payload>[] payload() default {};
}
```

**2. L'implémentation `PasswordMatchesValidator` :**

```java
package fr.formation.spring.validation.validator;

import fr.formation.spring.dto.UserRegistrationDTO; // DTO à valider
import fr.formation.spring.validation.constraint.PasswordMatches;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class PasswordMatchesValidator
        implements ConstraintValidator<PasswordMatches, UserRegistrationDTO> {

    @Override
    public void initialize(PasswordMatches constraintAnnotation) {
        // Peut être utilisé pour initialiser le validateur avec les attributs de l'annotation
    }

    @Override
    public boolean isValid(UserRegistrationDTO userDto, // L'objet DTO entier est passé
                           ConstraintValidatorContext context) {
        // Vérifie si les mots de passe correspondent
        // Gère les cas null pour éviter NullPointerException
        String password = userDto.getPassword();
        String confirmPassword = userDto.getConfirmPassword();

        boolean isValid = password != null && password.equals(confirmPassword);

        if (!isValid) {
            // Si invalide, désactive le message par défaut
            context.disableDefaultConstraintViolation();
            // Ajoute une violation spécifique au champ 'confirmPassword'
            context.buildConstraintViolationWithTemplate(
                            context.getDefaultConstraintMessageTemplate()) // Récupère le message de l'annotation
                    .addPropertyNode("confirmPassword") // Associe l'erreur à ce champ
                    .addConstraintViolation();
        }

        return isValid;
    }
}
```

**3. Appliquer l'annotation au DTO :**

```java
package fr.formation.spring.dto;

import fr.formation.spring.validation.constraint.PasswordMatches; // Importer l'annotation
import jakarta.validation.constraints.*;

// Appliquer l'annotation au niveau de la classe
@PasswordMatches(message = "La confirmation ne correspond pas au mot de passe.")
public class UserRegistrationDTO {
    // ... champs existants ...
    @NotBlank(message = "Le mot de passe est requis.")
    @Size(min = 8, message = "Le mot de passe doit contenir au moins {min} caractères.")
    private String password;

    @NotBlank(message = "La confirmation du mot de passe est requise.")
    private String confirmPassword;

    // ... reste des champs et getters/setters ...
}
```

Maintenant, lorsque `@Valid` est utilisé sur `UserRegistrationDTO`, `PasswordMatchesValidator` sera exécuté. L'erreur
sera associée au champ `confirmPassword` dans `BindingResult`.

**Exemple 2 : Validation d'Unicité (Niveau Champ, nécessite un Service)**

Cette validation vérifie si une valeur (par exemple, un email) existe déjà en base de données. Elle s'applique
généralement à un champ spécifique mais nécessite l'accès à la couche de service/repository.

**1. L'annotation `UniqueEmail` :**

```java
package fr.formation.spring.validation.constraint;

import fr.formation.spring.validation.validator.UniqueEmailValidator;
import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER}) // Applicable à un champ
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
@Documented
public @interface UniqueEmail {
    String message() default "Cette adresse email est déjà utilisée.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

**2. L'implémentation `UniqueEmailValidator` :**

Ce validateur a besoin d'accéder au `UserRepository` (ou un service). Spring peut injecter des dépendances dans les
validateurs.

```java
package fr.formation.spring.validation.validator;

import fr.formation.spring.repository.UserRepository; // Supposons un repository
import fr.formation.spring.validation.constraint.UniqueEmail;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.springframework.beans.factory.annotation.Autowired;
// import org.springframework.stereotype.Component; // Important pour l'injection

// @Component // Nécessaire pour que Spring gère l'injection de dépendance
// Si non utilisé, configurer ValidatorFactory pour connaitre le contexte Spring
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    // Injection du repository (ou service)
    // Note: Nécessite que le validateur soit un bean Spring (@Component)
    // ou une configuration spécifique du ValidatorFactory
    @Autowired
    private UserRepository userRepository;

    @Override
    public void initialize(UniqueEmail constraintAnnotation) {
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        // Si l'email est null ou vide, on considère valide (laisser @NotBlank gérer ça)
        if (email == null || email.isBlank()) {
            return true;
        }
        // Vérifie si l'email existe déjà en BDD
        // ATTENTION: Assurez-vous que userRepository est bien injecté !
        if (userRepository == null) {
            // Gérer le cas où l'injection n'a pas fonctionné (configuration)
            // Logguer une erreur sévère ou lancer une exception peut être approprié
            System.err.println("UserRepository non injecté dans UniqueEmailValidator !");
            // En production, ne pas retourner true ici, mais lever une exception ou logguer
            return true; // Pour l'exemple, mais dangereux
        }
        return !userRepository.existsByEmail(email); // Renvoie true si l'email N'EXISTE PAS
    }
}
```

**Configuration pour l'Injection dans les Validateurs:**

Pour que `@Autowired` fonctionne dans un `ConstraintValidator`, le validateur doit être géré par Spring. Le plus simple
est d'annoter le validateur avec `@Component`. Spring Boot configure généralement `LocalValidatorFactoryBean` pour
utiliser les beans Spring lors de la création des validateurs. Si ce n'est pas le cas, une configuration manuelle du
`Validator` bean est nécessaire.

**3. Appliquer l'annotation au champ DTO :**

```java
package fr.formation.spring.dto;

import fr.formation.spring.validation.constraint.PasswordMatches;
import fr.formation.spring.validation.constraint.UniqueEmail; // Importer
import jakarta.validation.constraints.*;

@PasswordMatches(message = "La confirmation ne correspond pas au mot de passe.")
public class UserRegistrationDTO {
    // ...
    @NotBlank(message = "L'adresse email est requise.")
    @Email(message = "Veuillez fournir une adresse email valide.")
    @UniqueEmail // Appliquer la nouvelle contrainte
    private String email;
    // ...
}
```

**Important :** Les validateurs nécessitant des accès BDD peuvent impacter les performances s'ils sont appelés très
fréquemment. Assurez-vous que les requêtes sous-jacentes sont optimisées (index sur le champ email).

**Exercice 2 : Validateur Personnalisé pour Prix Positif**

Bien qu'il existe `@Positive`, créez un validateur personnalisé `@PositivePrice` qui s'applique à un champ `BigDecimal`
et vérifie qu'il est strictement supérieur à zéro.

1. Créez l'annotation `PositivePrice`.
2. Créez le validateur `PositivePriceValidator`.
3. Appliquez `@PositivePrice` au champ `price` du `ProductDTO` à la place de `@PositiveOrZero` et `@NotNull`.

**Correction Exercice 2**

**PositivePrice.java:**

```java
package fr.formation.spring.validation.constraint;

import fr.formation.spring.validation.validator.PositivePriceValidator;
import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PositivePriceValidator.class)
@Documented
public @interface PositivePrice {
    String message() default "Le prix doit être strictement positif.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

**PositivePriceValidator.java:**

```java
package fr.formation.spring.validation.validator;

import fr.formation.spring.validation.constraint.PositivePrice;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import java.math.BigDecimal;

public class PositivePriceValidator
        implements ConstraintValidator<PositivePrice, BigDecimal> {

    @Override
    public void initialize(PositivePrice constraintAnnotation) {
    }

    @Override
    public boolean isValid(BigDecimal price, ConstraintValidatorContext context) {
        // Si null, on considère valide (laisser @NotNull ou @NotBlank gérer ça si besoin)
        // Ou retourner false si le prix est obligatoire ET doit être positif
        if (price == null) {
            // Décidez si null est une valeur invalide pour cette contrainte.
            // Ici, on suppose que @NotNull sera utilisé séparément si obligatoire.
            return true;
            // Alternative: Si le prix doit être OBLIGATOIREMENT positif:
            // return false;
        }
        // Vérifie si le prix est supérieur à zéro
        return price.compareTo(BigDecimal.ZERO) > 0;
    }
}
```

**ProductDTO.java (modifié):**

```java
package fr.formation.spring.dto;

import fr.formation.spring.validation.constraint.PositivePrice; // Importer

import java.math.BigDecimal;

import jakarta.validation.constraints.*;

public class ProductDTO {

    // ... autres champs ...

    @NotNull(message = "Le prix est requis.") // Garder @NotNull si le prix est obligatoire
    @PositivePrice // Utiliser notre validateur personnalisé
    @Digits(integer = 8, fraction = 2, message = "Le prix doit avoir au maximum {integer} chiffres avant la virgule et {fraction} après.")
    private BigDecimal price;

    // Getters et Setters...
}
```

---

**5. Gestion de l'Upload de Fichiers**

Spring MVC facilite la gestion des uploads de fichiers via l'interface `MultipartFile`.

**Étapes Clés :**

1. **Configuration :** Assurez-vous que la gestion multipart est activée. Avec Spring Boot, cela se fait souvent via
   `application.properties` (ou `.yml`) :
   ```properties
   # Activer la gestion multipart
   spring.servlet.multipart.enabled=true
   # Taille maximale d'un fichier uploadé (ex: 10MB)
   spring.servlet.multipart.max-file-size=10MB
   # Taille maximale totale de la requête multipart (ex: 12MB)
   spring.servlet.multipart.max-request-size=12MB
   # Optionnel: Seuil à partir duquel les fichiers sont écrits sur disque
   # spring.servlet.multipart.file-size-threshold=2KB
   # Optionnel: Répertoire temporaire pour le stockage
   # spring.servlet.multipart.location=/path/to/temp/dir
   ```
   Si vous n'utilisez pas Spring Boot, vous devrez déclarer un bean `MultipartConfigElement` ou
   `StandardServletMultipartResolver`.

2. **Formulaire HTML :** Le formulaire doit utiliser la méthode `POST` et avoir l'attribut
   `enctype="multipart/form-data"`. Ajoutez un champ `<input type="file" name="monFichier">`.

3. **Contrôleur :** Dans la méthode POST du contrôleur, ajoutez un paramètre de type `MultipartFile` annoté avec
   `@RequestParam("nomDuChampInput")`. Le nom dans `@RequestParam` doit correspondre à l'attribut `name` de l'input
   file.

4. **Traitement du Fichier :** L'objet `MultipartFile` reçu contient les informations et le contenu du fichier. Méthodes
   utiles :
    * `getOriginalFilename()`: Nom original du fichier côté client ( **Attention : ne jamais faire confiance à ce nom
      pour le stockage côté serveur !** Il peut contenir des caractères invalides ou des tentatives de path traversal
      `../../..`).
    * `getContentType()`: Le type MIME déclaré par le navigateur ( **Attention : peu fiable**, peut être falsifié par le
      client).
    * `getSize()`: Taille du fichier en octets.
    * `getBytes()`: Récupère le contenu du fichier sous forme de `byte[]` (Attention à la mémoire pour les gros
      fichiers).
    * `getInputStream()`: Obtient un `InputStream` pour lire le contenu (préférable pour les gros fichiers).
    * `transferTo(File dest)` ou `transferTo(Path dest)`: Méthode la plus simple et recommandée pour sauvegarder le
      fichier sur le disque serveur. Spring gère le flux.

**Exemple : Ajout d'une Image Produit**

**ProductDTO (ajout du champ):**

```java
package fr.formation.spring.dto;

// ... autres imports ...

import org.springframework.web.multipart.MultipartFile;
import fr.formation.spring.validation.constraint.ValidFile; // Validateur custom (voir plus bas)

public class ProductDTO {
    // ... autres champs ...

    @ValidFile(allowedTypes = {"image/jpeg", "image/png"}, maxSize = 5 * 1024 * 1024) // 5MB
    private MultipartFile productImage; // Champ pour le fichier uploadé

    // Getter et Setter pour productImage
    public MultipartFile getProductImage() {
        return productImage;
    }

    public void setProductImage(MultipartFile productImage) {
        this.productImage = productImage;
    }

    // ... reste des getters/setters ...
}
```

**product-form.html (ajout du champ file):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Créer un Produit</title>
    <style>.error {
        color: red;
    }</style>
</head>
<body>
<h1>Créer un Nouveau Produit</h1>

<!-- IMPORTANT: enctype="multipart/form-data" -->
<form th:action="@{/products/create}" th:object="${productDTO}" method="post"
      enctype="multipart/form-data">

    <!-- ... autres champs (sku, name, description, price) ... -->

    <div>
        <label for="productImage">Image du produit (JPG, PNG, max 5MB):</label>
        <!-- Note: L'input type="file" ne peut pas être lié avec th:field -->
        <!-- On utilise 'name' qui doit correspondre au champ dans le DTO -->
        <input type="file" id="productImage" name="productImage"/>
        <!-- Affichage des erreurs spécifiques au fichier -->
        <span class="error" th:if="${#fields.hasErrors('productImage')}"
              th:errors="*{productImage}">Erreur Fichier</span>
    </div>

    <div>
        <button type="submit">Créer Produit</button>
    </div>
</form>
</body>
</html>
```

**ProductController (modification de la méthode POST):**

```java
package fr.formation.spring.controller;

// ... autres imports ...

import org.springframework.web.multipart.MultipartFile;
import org.springframework.beans.factory.annotation.Value; // Pour injecter des propriétés

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID; // Pour générer des noms de fichiers uniques

@Controller
@RequestMapping("/products")
public class ProductController {

    private final ProductService productService;
    private final Path fileStorageLocation; // Chemin où stocker les fichiers

    private static final Logger log = LoggerFactory.getLogger(ProductController.class);


    // Injecter le chemin depuis application.properties
    public ProductController(ProductService productService,
                             @Value("${app.file.storage-location:/tmp/uploads}") String storagePath) {
        this.productService = productService;
        this.fileStorageLocation = Paths.get(storagePath).toAbsolutePath().normalize();

        // Créer le répertoire de stockage s'il n'existe pas
        try {
            Files.createDirectories(this.fileStorageLocation);
        } catch (IOException ex) {
            // Gérer l'erreur de création de répertoire (critique)
            log.error("Impossible de créer le répertoire de stockage: {}",
                    this.fileStorageLocation, ex);
            throw new RuntimeException(
                    "Impossible de créer le répertoire de stockage.", ex);
        }
    }

    // ... showCreateForm() et listProducts() restent similaires ...

    @PostMapping("/create")
    public String createProduct(
            @Valid @ModelAttribute("productDTO") ProductDTO productDTO,
            BindingResult bindingResult,
            Model model) {

        // La validation @ValidFile sur productDTO.productImage est exécutée avant

        if (bindingResult.hasErrors()) {
            log.warn("Echec de validation ou d'upload pour la création de produit.");
            return "product-form";
        }

        // Gérer le fichier uploadé SEULEMENT si la validation est OK
        MultipartFile file = productDTO.getProductImage();
        String storedFileName = null; // Nom du fichier stocké sur le serveur

        // Vérifier si un fichier a été réellement uploadé et n'est pas vide
        if (file != null && !file.isEmpty()) {
            try {
                // **Sécurité : Nettoyer et Générer un nom de fichier unique**
                String originalFileName = file.getOriginalFilename();
                // Nettoyage basique (plus robuste serait d'utiliser une lib)
                String cleanOriginalName = originalFileName.replaceAll("[^a-zA-Z0-9._-]", "");
                // Extraire l'extension
                String extension = "";
                int dotIndex = cleanOriginalName.lastIndexOf('.');
                if (dotIndex > 0 && dotIndex < cleanOriginalName.length() - 1) {
                    extension = cleanOriginalName.substring(dotIndex);
                }
                // Générer un nom unique pour éviter les collisions et l'exposition du nom original
                storedFileName = UUID.randomUUID().toString() + extension;

                // Construire le chemin complet de destination
                Path targetLocation = this.fileStorageLocation.resolve(storedFileName);

                // **Sécurité : Vérifier que le chemin final est bien dans le répertoire prévu**
                // Empêche les attaques de type "Path Traversal" (ex: ../../..),
                // même si resolve/normalize aident déjà.
                if (!targetLocation.getParent().equals(this.fileStorageLocation)) {
                    log.error("Tentative de Path Traversal détectée: {}", originalFileName);
                    // Ajouter une erreur au bindingResult et retourner au formulaire
                    bindingResult.rejectValue("productImage", "error.file.pathTraversal",
                            "Nom de fichier invalide.");
                    return "product-form";
                }


                // Sauvegarder le fichier
                log.info("Sauvegarde du fichier {} sous {}", originalFileName, targetLocation);
                file.transferTo(targetLocation);

            } catch (IOException ex) {
                log.error("Erreur lors de la sauvegarde du fichier {}",
                        file.getOriginalFilename(), ex);
                // Ajouter une erreur au bindingResult pour informer l'utilisateur
                bindingResult.rejectValue("productImage", "error.file.storage",
                        "Erreur lors de la sauvegarde du fichier.");
                return "product-form";
            }
        } else {
            log.info("Aucun fichier image fourni pour le produit.");
            // Gérer le cas où l'image est optionnelle ou obligatoire
            // Si obligatoire, une validation (@NotNull?) aurait dû échouer avant
        }

        // Si tout s'est bien passé (validation + sauvegarde fichier),
        // Appeler le service pour créer le produit
        // Il faudra transmettre 'storedFileName' au service pour le lier au produit
        productService.createProductFromDTO(productDTO, storedFileName);

        return "redirect:/products/list";
    }
}
```

**`application.properties` (ajout du chemin):**

```properties
# ... autres propriétés ...
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=12MB
# Chemin vers le répertoire de stockage des fichiers uploadés
# Utiliser des chemins absolus ou relatifs à l'exécution de l'application
# S'assurer que l'application a les droits d'écriture dans ce répertoire
app.file.storage-location=./uploads/product-images
```

**Validation du Fichier (Taille, Type MIME, Magic Numbers)**

La validation de base (taille) peut être faite avec les propriétés `spring.servlet.multipart.*`. Cependant, pour des
validations plus fines (type de fichier), il faut aller plus loin.

* **Taille :** La propriété `max-file-size` gère la taille maximale. Si dépassée, Spring lance une
  `MaxUploadSizeExceededException`, qui peut être gérée globalement via un `@ControllerAdvice`. On peut aussi vérifier
  `file.getSize()` dans le code si besoin.
* **Type (`Content-Type`) :** `file.getContentType()` retourne le type MIME envoyé par le navigateur. **Ce n'est PAS
  fiable.** Un utilisateur peut facilement modifier cette information. Ne l'utilisez que comme première indication ou
  pour des validations peu critiques.
* **Validation Robuste du Type :**
    * **Type MIME via Contenu (Magic Numbers / Bibliothèque) :** La méthode la plus fiable est d'analyser les premiers
      octets du fichier (les "magic numbers") pour déterminer son type réel. Des bibliothèques comme **Apache Tika**
      sont excellentes pour cela.
    * **Validation par Extension :** Moins fiable que l'analyse de contenu, mais peut être une première couche de
      filtrage.

**Exemple : Validateur Personnalisé `@ValidFile` (Taille et Type MIME/Magic Number)**

**1. Ajouter la dépendance Apache Tika:** (Maven)

```xml

<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-core</artifactId>
    <version>2.9.1</version> <!-- Vérifier la dernière version -->
</dependency>
```

**2. L'annotation `ValidFile` :**

```java
package fr.formation.spring.validation.constraint;

import fr.formation.spring.validation.validator.FileValidator;
import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = FileValidator.class)
@Documented
public @interface ValidFile {

    String message() default "Fichier invalide.";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    // Attributs personnalisés pour le validateur
    String[] allowedTypes() default {}; // Types MIME autorisés (ex: "image/jpeg")

    long maxSize() default Long.MAX_VALUE; // Taille max en octets
}
```

**3. L'implémentation `FileValidator` :**

```java
package fr.formation.spring.validation.validator;

import fr.formation.spring.validation.constraint.ValidFile;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import org.apache.tika.Tika; // Import Apache Tika
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.util.Arrays;
import java.util.List;

public class FileValidator implements ConstraintValidator<ValidFile, MultipartFile> {

    private static final Logger log = LoggerFactory.getLogger(FileValidator.class);
    private List<String> allowedTypes;
    private long maxSize;
    private final Tika tika = new Tika(); // Instance de Tika pour la détection MIME

    @Override
    public void initialize(ValidFile constraintAnnotation) {
        this.allowedTypes = Arrays.asList(constraintAnnotation.allowedTypes());
        this.maxSize = constraintAnnotation.maxSize();
    }

    @Override
    public boolean isValid(MultipartFile file, ConstraintValidatorContext context) {
        // Si le fichier est null ou vide, on considère valide
        // (laisser @NotNull gérer ça si le fichier est obligatoire)
        if (file == null || file.isEmpty()) {
            return true;
        }

        // 1. Validation de la taille
        if (file.getSize() > this.maxSize) {
            log.warn("Fichier trop volumineux: {} octets (max: {})",
                    file.getSize(), this.maxSize);
            updateContext(context, "La taille du fichier dépasse la limite autorisée ("
                    + formatSize(this.maxSize) + ").");
            return false;
        }

        // 2. Validation du type MIME via Apache Tika (basé sur le contenu)
        String detectedType = null;
        try (InputStream inputStream = file.getInputStream()) {
            // Tika détecte le type MIME à partir des premiers octets du flux
            detectedType = tika.detect(inputStream);
            log.debug("Type MIME détecté par Tika pour '{}': {}",
                    file.getOriginalFilename(), detectedType);
        } catch (IOException e) {
            log.error("Erreur lors de la lecture du fichier pour détection MIME: {}",
                    file.getOriginalFilename(), e);
            updateContext(context, "Erreur lors de la vérification du type de fichier.");
            return false; // Erreur de lecture, considérer invalide
        }

        // Vérifier si le type détecté est dans la liste des types autorisés
        if (this.allowedTypes.isEmpty() || detectedType == null ||
                !this.allowedTypes.contains(detectedType)) {
            log.warn("Type de fichier non autorisé: {} (autorisés: {})",
                    detectedType, this.allowedTypes);
            updateContext(context, "Type de fichier non autorisé. Types permis: "
                    + String.join(", ", this.allowedTypes));
            return false;
        }

        // (Optionnel) On pourrait ajouter une vérification Magic Number manuelle ici
        // si Tika ne suffit pas ou pour une double vérification, mais Tika est
        // généralement suffisant et plus complet.

        // Si toutes les validations passent
        return true;
    }

    // Helper pour mettre à jour le message d'erreur dans le contexte
    private void updateContext(ConstraintValidatorContext context, String message) {
        context.disableDefaultConstraintViolation(); // Désactive le message par défaut
        context.buildConstraintViolationWithTemplate(message) // Définit le nouveau message
                .addConstraintViolation(); // Ajoute la violation
    }

    // Helper pour formater la taille en KB/MB/etc. (simplifié)
    private String formatSize(long size) {
        if (size < 1024) return size + " B";
        int exp = (int) (Math.log(size) / Math.log(1024));
        char pre = "KMGTPE".charAt(exp - 1);
        return String.format("%.1f %sB", size / Math.pow(1024, exp), pre);
    }
}
```

**4. Appliquer l'annotation au DTO :** (Déjà montré dans l'exemple `ProductDTO`)

```java
    // Dans ProductDTO.java
@ValidFile(allowedTypes = {"image/jpeg", "image/png", "image/gif"},
        maxSize = 5 * 1024 * 1024) // 5MB, autorise JPG, PNG, GIF
private MultipartFile productImage;
```

Ce validateur personnalisé offre une sécurité bien meilleure que la simple vérification du `Content-Type` ou de
l'extension.

**Exercice 3 : Upload de CV (PDF uniquement)**

1. Modifiez le `UserRegistrationDTO` pour ajouter un champ `MultipartFile cvFile`.
2. Utilisez le validateur `@ValidFile` pour s'assurer que le fichier est :
    * Un PDF (`application/pdf`).
    * D'une taille maximale de 2MB.
3. Adaptez le formulaire `registration-form.html` pour inclure un champ d'upload pour le CV.
4. Adaptez la méthode `processRegistration` du contrôleur pour gérer (simplement logger le nom ou sauvegarder) le
   fichier CV *si* la validation réussit. N'oubliez pas de gérer les erreurs potentielles de sauvegarde.

**Correction Exercice 3**

**UserRegistrationDTO.java (ajout du champ):**

```java
package fr.formation.spring.dto;

import fr.formation.spring.validation.constraint.PasswordMatches;
import fr.formation.spring.validation.constraint.UniqueEmail;
import fr.formation.spring.validation.constraint.ValidFile; // Importer
import jakarta.validation.constraints.*;
import org.springframework.web.multipart.MultipartFile;

@PasswordMatches(message = "La confirmation ne correspond pas au mot de passe.")
public class UserRegistrationDTO {
    // ... autres champs ...

    @AssertTrue(message = "Vous devez accepter les conditions d'utilisation.")
    private boolean acceptTerms;

    // Ajout du champ CV
    @ValidFile(allowedTypes = {"application/pdf"}, // PDF uniquement
            maxSize = 2 * 1024 * 1024) // 2MB max
    private MultipartFile cvFile;

    // Getters et Setters...
    public MultipartFile getCvFile() {
        return cvFile;
    }

    public void setCvFile(MultipartFile cvFile) {
        this.cvFile = cvFile;
    }
    // ... reste des getters/setters ...
}
```

**registration-form.html (ajout du champ):**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<!-- ... head ... -->
<body>
<h1>Formulaire d'Inscription</h1>
<!-- IMPORTANT: enctype="multipart/form-data" -->
<form th:action="@{/register}" th:object="${userDto}" method="post"
      enctype="multipart/form-data">

    <!-- ... champs username, email, password, confirmPassword ... -->

    <div>
        <label for="cvFile">CV (PDF, max 2MB):</label>
        <input type="file" id="cvFile" name="cvFile"/>
        <span class="error" th:if="${#fields.hasErrors('cvFile')}"
              th:errors="*{cvFile}"></span>
    </div>

    <div>
        <input type="checkbox" id="acceptTerms" th:field="*{acceptTerms}"/>
        <label for="acceptTerms">J'accepte les conditions d'utilisation</label>
        <span class="error" th:if="${#fields.hasErrors('acceptTerms')}"
              th:errors="*{acceptTerms}"></span>
    </div>

    <div>
        <button type="submit">S'inscrire</button>
    </div>
</form>
</body>
</html>
```

**UserController.java (modification de processRegistration):**

```java
package fr.formation.spring.controller;

// ... imports ...

import org.springframework.web.multipart.MultipartFile;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.io.IOException;
import java.util.UUID;

import org.springframework.beans.factory.annotation.Value;

@Controller
@RequestMapping("/register")
public class UserController {

    private static final Logger log = LoggerFactory.getLogger(UserController.class);
    private final Path cvStorageLocation; // Chemin pour les CVs

    // Injecter le chemin pour les CVs
    public UserController(@Value("${app.cv.storage-location:/tmp/cvs}") String cvPath) {
        this.cvStorageLocation = Paths.get(cvPath).toAbsolutePath().normalize();
        try {
            Files.createDirectories(this.cvStorageLocation);
        } catch (IOException ex) {
            log.error("Impossible de créer le répertoire de stockage des CVs: {}",
                    this.cvStorageLocation, ex);
            throw new RuntimeException(
                    "Impossible de créer le répertoire de stockage des CVs.", ex);
        }
    }

    @GetMapping
    public String showRegistrationForm(Model model) {
        model.addAttribute("userDto", new UserRegistrationDTO());
        return "registration-form";
    }

    @PostMapping
    public String processRegistration(
            @Valid @ModelAttribute("userDto") UserRegistrationDTO userDto,
            BindingResult bindingResult,
            Model model) {

        // Validation custom de mot de passe (si implémentée) et @ValidFile sur cvFile
        // sont exécutées avant cette méthode grâce à @Valid

        if (bindingResult.hasErrors()) {
            log.warn("Erreurs de validation ou d'upload détectées.");
            return "registration-form";
        }

        // Gérer l'upload du CV si présent et valide
        MultipartFile cv = userDto.getCvFile();
        String storedCvFileName = null;

        if (cv != null && !cv.isEmpty()) {
            try {
                String originalFileName = cv.getOriginalFilename();
                String cleanOriginalName = originalFileName.replaceAll("[^a-zA-Z0-9._-]", "");
                String extension = ".pdf"; // On sait que c'est un PDF grâce au validateur
                storedCvFileName = UUID.randomUUID().toString() + extension;

                Path targetLocation = this.cvStorageLocation.resolve(storedCvFileName);

                if (!targetLocation.getParent().equals(this.cvStorageLocation)) {
                    log.error("Tentative de Path Traversal détectée pour CV: {}", originalFileName);
                    bindingResult.rejectValue("cvFile", "error.file.pathTraversal",
                            "Nom de fichier CV invalide.");
                    return "registration-form";
                }

                log.info("Sauvegarde du CV {} sous {}", originalFileName, targetLocation);
                cv.transferTo(targetLocation);

            } catch (IOException ex) {
                log.error("Erreur lors de la sauvegarde du CV {}",
                        cv.getOriginalFilename(), ex);
                bindingResult.rejectValue("cvFile", "error.file.storage",
                        "Erreur lors de la sauvegarde du CV.");
                return "registration-form";
            }
        } else {
            log.info("Aucun CV fourni pour l'utilisateur {}.", userDto.getUsername());
            // Gérer selon si le CV est optionnel ou requis
        }

        // Logique métier (ex: enregistrer l'utilisateur et le nom du fichier CV)
        log.info("Inscription réussie pour : {}. CV sauvegardé : {}",
                userDto.getUsername(), storedCvFileName != null ? storedCvFileName : "Non");

        // Redirection vers une page de succès
        return "redirect:/register/success";
    }

    @GetMapping("/success")
    public String registrationSuccess() {
        return "registration-success";
    }
}
```

**application.properties (ajout du chemin pour les CV):**

```properties
# ... autres propriétés ...
app.file.storage-location=./uploads/product-images
app.cv.storage-location=./uploads/cvs
```

---

**6. Mapping entre DTO et Entités (MapStruct & ModelMapper)**

Une fois les données du formulaire validées dans le DTO, il faut souvent les transférer vers une ou plusieurs entités de
domaine (par exemple, des entités JPA) avant de les persister. Inversement, lors de l'affichage d'un formulaire de
modification, il faut mapper une entité existante vers un DTO.

**Approche Manuelle :**
Écrire le code de mapping à la main dans la couche service.

```java
// Dans ProductService.java
public Product createProductFromDTO(ProductDTO dto, String imageFileName) {
    Product product = new Product();
    product.setSku(dto.getSku());
    product.setName(dto.getName());
    product.setDescription(dto.getDescription());
    product.setPrice(dto.getPrice());
    product.setActive(true); // Logique métier: nouveau produit est actif
    // product.setImageFileName(imageFileName); // Si l'entité a ce champ

    // return productRepository.save(product);
    return product; // Exemple simplifié
}

public ProductDTO convertToDto(Product product) {
    ProductDTO dto = new ProductDTO();
    dto.setSku(product.getSku());
    dto.setName(product.getName());
    dto.setDescription(product.getDescription());
    dto.setPrice(product.getPrice());
    // Ne pas mapper 'active' ou 'id' si non nécessaires dans le DTO
    return dto;
}
```

* **Avantages :** Simple pour des objets peu complexes, contrôle total.
* **Inconvénients :** Verbeux, répétitif, source d'erreurs (oublis, fautes de frappe), difficile à maintenir pour des
  objets complexes.

**Bibliothèques de Mapping Automatisé :**

Des bibliothèques comme MapStruct et ModelMapper automatisent ce processus.

**A. MapStruct**

MapStruct est un générateur de code basé sur des annotations. Il crée des implémentations de mapping **à la compilation
**.

* **Avantages :**
    * **Performances élevées :** Le code de mapping est du Java pur généré, pas de réflexion à l'exécution.
    * **Sécurité des types :** Les erreurs de mapping (types incompatibles, champs manquants non gérés) sont détectées à
      la compilation.
    * **Intégration facile :** Fonctionne bien avec Lombok, Spring.
    * **Clarté :** Les règles de mapping sont définies dans une interface dédiée.
* **Inconvénients :**
    * Nécessite une étape de compilation (intégration avec Maven/Gradle).
    * Peut sembler un peu plus verbeux au début pour définir les mappings complexes.

**Configuration MapStruct (Maven):**

```xml

<properties>
    <org.mapstruct.version>1.5.5.Final</org.mapstruct.version> <!-- Vérifier dernière version -->
</properties>

<dependencies>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
</dependency>
<!-- Autres dépendances Spring, JPA, etc. -->
</dependencies>

<build>
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.8.1</version> <!-- ou plus récent -->
        <configuration>
            <source>17</source> <!-- ou votre version Java -->
            <target>17</target>
            <annotationProcessorPaths>
                <path>
                    <groupId>org.mapstruct</groupId>
                    <artifactId>mapstruct-processor</artifactId>
                    <version>${org.mapstruct.version}</version>
                </path>
                <!-- Optionnel : si vous utilisez Lombok -->
                <!--
                <path>
                    <groupId>org.projectlombok</groupId>
                    <artifactId>lombok</artifactId>
                    <version>${lombok.version}</version>
                </path>
                <path>
                     <groupId>org.projectlombok</groupId>
                     <artifactId>lombok-mapstruct-binding</artifactId>
                     <version>0.2.0</version>
                </path>
                -->
            </annotationProcessorPaths>
            <!-- Optionnel: Pour intégration avec Spring -->
            <compilerArgs>
                <compilerArg>
                    -Amapstruct.defaultComponentModel=spring
                </compilerArg>
            </compilerArgs>
        </configuration>
    </plugin>
</plugins>
</build>
```

**Exemple d'Interface Mapper :**

```java
package fr.formation.spring.mapper;

import fr.formation.spring.dto.ProductDTO;
import fr.formation.spring.entity.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingTarget;
import org.mapstruct.Mappings;
import org.mapstruct.factory.Mappers; // Utile si non géré par Spring

// @Mapper rend cette interface détectable par MapStruct
// componentModel="spring" permet d'injecter ce mapper comme un bean Spring
@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Instance pour utilisation si non injecté par Spring (moins courant avec Spring)
    // ProductMapper INSTANCE = Mappers.getMapper(ProductMapper.class);

    // Mappe ProductDTO vers Product
    // Les champs avec le même nom sont mappés automatiquement.
    // On peut ignorer des champs ou spécifier des mappings différents.
    @Mappings({
            // On ignore l'ID car il est généré par la BDD pour une nouvelle entité
            @Mapping(target = "id", ignore = true),
            // On ignore 'active' car il sera défini par la logique métier, pas par le DTO
            @Mapping(target = "active", ignore = true)
            // Si les noms étaient différents: @Mapping(source = "dtoFieldName", target = "entityFieldName")
            // Exemple: @Mapping(source = "sku", target = "stockKeepingUnit")
    })
    Product productDtoToProduct(ProductDTO productDTO);

    // Mappe Product vers ProductDTO
    // Ici, les champs 'active' et 'id' de Product sont ignorés car non présents dans ProductDTO
    // (MapStruct ignore par défaut les champs cibles manquants)
    ProductDTO productToProductDto(Product product);

    // Méthode pour mettre à jour une entité Product existante depuis un DTO
    // Utilise @MappingTarget pour indiquer l'objet à mettre à jour
    @Mappings({
            @Mapping(target = "id", ignore = true), // Ne jamais mettre à jour l'ID depuis un DTO
            @Mapping(target = "active", ignore = true) // Ne pas laisser le DTO changer l'état actif
    })
    void updateProductFromDto(ProductDTO productDTO, @MappingTarget Product product);

}
```

**Utilisation dans le Service :**

```java
package fr.formation.spring.service;

import fr.formation.spring.dto.ProductDTO;
import fr.formation.spring.entity.Product;
import fr.formation.spring.mapper.ProductMapper; // Importer le mapper
import fr.formation.spring.repository.ProductRepository; // Supposons un repository
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper; // Injecter le mapper

    public ProductService(ProductRepository productRepository, ProductMapper productMapper) {
        this.productRepository = productRepository;
        this.productMapper = productMapper;
    }

    @Transactional // Assure la transactionnalité
    public Product createProductFromDTO(ProductDTO productDTO, String imageFileName) {
        // Utiliser le mapper pour convertir DTO -> Entité
        Product product = productMapper.productDtoToProduct(productDTO);

        // Appliquer la logique métier spécifique non gérée par le mapping
        product.setActive(true);
        // product.setImageFileName(imageFileName); // Assigner le nom du fichier sauvegardé

        // Sauvegarder l'entité
        return productRepository.save(product);
    }

    @Transactional(readOnly = true) // Transaction en lecture seule pour la récupération
    public ProductDTO getProductDTOById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Produit non trouvé")); // Gérer exception
        // Utiliser le mapper pour convertir Entité -> DTO
        return productMapper.productToProductDto(product);
    }

    @Transactional
    public Product updateProductFromDTO(Long id, ProductDTO productDTO) {
        Product existingProduct = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Produit non trouvé"));

        // Utiliser le mapper pour mettre à jour l'entité existante depuis le DTO
        productMapper.updateProductFromDto(productDTO, existingProduct);

        // Sauvegarder l'entité mise à jour
        return productRepository.save(existingProduct);
    }
}

```

**B. ModelMapper**

ModelMapper est une bibliothèque de mapping qui utilise la **réflexion à l'exécution**.

* **Avantages :**
    * **Configuration simple :** Moins de configuration de build nécessaire.
    * **Flexibilité :** Peut gérer des scénarios de mapping complexes avec une configuration fluide.
    * **Découverte automatique :** Tente de mapper automatiquement les champs par nom et type.
* **Inconvénients :**
    * **Performances :** Moins performant que MapStruct car basé sur la réflexion. L'impact est souvent négligeable pour
      la plupart des applications CRUD, mais peut compter sous forte charge.
    * **Sécurité des types :** Les erreurs de mapping sont détectées à l'exécution, pas à la compilation.
    * **Moins explicite :** La configuration peut parfois devenir complexe et moins facile à suivre que les interfaces
      MapStruct.

**Configuration ModelMapper (Maven):**

```xml

<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.1.1</version> <!-- Vérifier dernière version -->
</dependency>
```

**Configuration du Bean ModelMapper (dans une classe de configuration Spring):**

```java
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

        // Configuration recommandée pour éviter les mappings ambigus
        // STRRICT force les noms et types de source/destination à correspondre exactement.
        modelMapper.getConfiguration()
                .setMatchingStrategy(MatchingStrategies.STRICT)
                .setFieldMatchingEnabled(true) // Permet le matching sur les champs privés
                .setSkipNullEnabled(true); // Ne mappe pas les valeurs null de la source

        // Configuration spécifique si nécessaire (ex: ignorer un champ globalement)
        /*
        modelMapper.typeMap(ProductDTO.class, Product.class).addMappings(mapper -> {
            mapper.skip(Product::setId); // Ne jamais mapper l'ID depuis DTO vers Entité
            mapper.skip(Product::setActive);
        });
        */

        return modelMapper;
    }
}
```

**Utilisation dans le Service :**

```java
package fr.formation.spring.service;

import fr.formation.spring.dto.ProductDTO;
import fr.formation.spring.entity.Product;
import fr.formation.spring.repository.ProductRepository;
import org.modelmapper.ModelMapper; // Importer ModelMapper
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ProductServiceWithModelMapper {

    private final ProductRepository productRepository;
    private final ModelMapper modelMapper; // Injecter ModelMapper

    public ProductServiceWithModelMapper(ProductRepository productRepository, ModelMapper modelMapper) {
        this.productRepository = productRepository;
        this.modelMapper = modelMapper;
    }

    @Transactional
    public Product createProductFromDTO(ProductDTO productDTO, String imageFileName) {
        // Mapper DTO -> Entité
        // Le 2ème argument est la classe cible
        Product product = modelMapper.map(productDTO, Product.class);

        // Logique métier spécifique
        product.setActive(true);
        // product.setImageFileName(imageFileName);

        return productRepository.save(product);
    }

    @Transactional(readOnly = true)
    public ProductDTO getProductDTOById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Produit non trouvé"));
        // Mapper Entité -> DTO
        return modelMapper.map(product, ProductDTO.class);
    }

    @Transactional
    public Product updateProductFromDTO(Long id, ProductDTO productDTO) {
        Product existingProduct = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Produit non trouvé"));

        // Mapper le DTO sur l'entité existante
        // ModelMapper met à jour les champs de 'existingProduct' avec les valeurs de 'productDTO'
        // Attention: La configuration (ex: skipNullEnabled) est importante ici.
        // Il faut s'assurer de ne pas écraser l'ID ou d'autres champs sensibles.
        // Une configuration fine via TypeMap est souvent nécessaire pour les mises à jour.

        // Configuration spécifique pour la mise à jour (peut être dans le bean config)
        modelMapper.typeMap(ProductDTO.class, Product.class).addMappings(mapper -> {
            mapper.skip(Product::setId); // Assure que l'ID n'est pas écrasé
            mapper.skip(Product::setActive); // Ne modifie pas 'active' depuis le DTO
        });


        modelMapper.map(productDTO, existingProduct);

        return productRepository.save(existingProduct);
    }
}
```

**MapStruct vs ModelMapper : Lequel choisir ?**

* **Pour la performance et la sécurité de type à la compilation :** MapStruct est généralement préféré.
* **Pour la simplicité de configuration initiale et la flexibilité rapide :** ModelMapper peut être plus rapide à mettre
  en place pour des cas simples.
* **Pour les projets d'équipe importants :** La clarté et la détection précoce des erreurs de MapStruct sont des
  avantages significatifs.

Il est conseillé d'essayer les deux sur un petit module pour voir celui qui correspond le mieux aux besoins et aux
préférences de l'équipe.

---

**7. Sécurité des Formulaires Web**

Les formulaires sont une porte d'entrée majeure pour les attaquants. Il est **impératif** de comprendre et de mitiger
les risques associés.

**Principales Menaces et Mitigations :**

1. **Cross-Site Scripting (XSS)**
    * **Menace :** Un attaquant injecte du code malveillant (généralement JavaScript) dans les données soumises via un
      formulaire. Lorsque ces données sont ensuite affichées sans échappement sur une page web (par exemple, dans un
      commentaire, un nom de produit), le script s'exécute dans le navigateur d'un autre utilisateur, pouvant voler des
      cookies (sessions), rediriger l'utilisateur, défigurer la page, etc.
    * **Mitigation :**
        * **Échappement Systématique en Sortie (Output Encoding) :** C'est la défense la plus importante. **Toujours**
          échapper les données fournies par l'utilisateur avant de les afficher dans une page HTML. Les moteurs de
          templates comme Thymeleaf le font **par défaut** pour le contenu textuel (`th:text`, `th:field`, expressions
          `${...}`). Attention si vous utilisez `th:utext` (unescaped text) ou si vous construisez du HTML manuellement.
        * **Validation d'Entrée Stricte :** Refuser les entrées contenant des caractères potentiellement dangereux si le
          champ ne doit pas en contenir (par exemple, un champ "année" ne devrait contenir que des chiffres). Utiliser
          des listes blanches (autoriser uniquement certains caractères) plutôt que des listes noires (interdire
          certains caractères, facile à contourner).
        * **Content Security Policy (CSP) :** En-tête HTTP qui indique au navigateur quelles sources de contenu (
          scripts, styles, images) sont autorisées à être chargées et exécutées pour la page. Peut grandement limiter
          l'impact d'une injection XSS réussie. Spring Security facilite la configuration du CSP.
        * **Cookies `HttpOnly` :** Configurez les cookies de session (et autres cookies sensibles) avec l'attribut
          `HttpOnly`. Cela empêche JavaScript (y compris un script XSS) d'y accéder. Spring Boot le fait souvent par
          défaut pour le cookie de session.

2. **Cross-Site Request Forgery (CSRF)**
    * **Menace :** Un attaquant crée une page web malveillante (ou un email) qui contient une requête vers votre
      application (par exemple, via un formulaire caché ou une balise `<img>` pointant vers une URL d'action comme
      `/transfer?amount=1000&to=attacker`). Si un utilisateur authentifié sur votre application visite cette page
      malveillante, son navigateur enverra la requête à votre application *avec ses cookies d'authentification*. Votre
      application croira que la requête est légitime et exécutera l'action (transfert d'argent, suppression de compte,
      etc.) à l'insu de l'utilisateur.
    * **Mitigation :**
        * **Tokens Anti-CSRF (Synchronizer Token Pattern) :** C'est la défense standard.
            * Pour chaque session utilisateur, le serveur génère un token secret et imprévisible.
            * Ce token est inclus dans chaque formulaire HTML renvoyé à l'utilisateur (généralement dans un champ caché
              `<input type="hidden" name="_csrf" value="...">`).
            * Lorsqu'un formulaire est soumis, le serveur vérifie que le token reçu correspond à celui associé à la
              session de l'utilisateur.
            * Une requête forgée depuis un autre site ne connaîtra pas le token correct et sera rejetée.
        * **Spring Security :** Fournit une protection anti-CSRF robuste et activée par défaut pour les requêtes
          modifiant l'état (POST, PUT, DELETE). Il gère la génération, l'ajout aux formulaires (via intégration avec
          Thymeleaf, etc.) et la validation des tokens.
            * Thymeleaf ajoute automatiquement le champ caché `_csrf` si Spring Security est présent et la protection
              CSRF activée.
            * Pour les requêtes AJAX (JavaScript), le token doit être inclus dans un en-tête HTTP (par exemple
              `X-CSRF-TOKEN`). Spring Security fournit des mécanismes pour faciliter cela.
        * **Attribut `SameSite` pour les Cookies :** L'attribut `SameSite` sur les cookies (avec les valeurs `Strict` ou
          `Lax`) aide à prévenir l'envoi de cookies lors de requêtes cross-site, offrant une couche de défense
          supplémentaire contre CSRF. `Lax` est un bon équilibre par défaut. Spring Security configure souvent
          `SameSite=Lax` pour le cookie de session.

3. **Mass Assignment / Over-Posting**
    * **Menace :** Déjà évoquée avec les DTO. Si l'on lie directement les données du formulaire à une entité JPA, un
      attaquant pourrait ajouter des champs supplémentaires dans la requête POST (par exemple, `isAdmin=true`) qui
      seraient mappés à l'entité et potentiellement persistés, même si ces champs n'étaient pas présents dans le
      formulaire HTML original.
    * **Mitigation :**
        * **Utiliser des DTO :** C'est la meilleure défense. Le DTO définit explicitement les champs attendus et seuls
          ceux-ci seront peuplés lors du data binding.
        * **(Moins recommandé) Annotation `@InitBinder` :** Dans le contrôleur, on peut utiliser `@InitBinder` pour
          configurer le `WebDataBinder` et spécifier les champs autorisés (`setAllowedFields`) ou interdits (
          `setDisallowedFields`). C'est plus fastidieux et moins sûr que les DTO.

4. **Injection de Commandes / Fichiers Malveillants (Upload)**
    * **Menace :** Lors de l'upload de fichiers, un attaquant peut tenter :
        * **Path Traversal :** Soumettre un nom de fichier contenant `../` pour essayer d'écrire le fichier en dehors du
          répertoire d'upload prévu (par exemple, écraser des fichiers système ou de configuration).
        * **Upload de Fichiers Exécutables :** Uploader un fichier `.php`, `.jsp`, `.exe` qui pourrait être exécuté par
          le serveur si le répertoire d'upload est mal configuré ou accessible directement via le web.
        * **Déni de Service (DoS) :** Uploader des fichiers très volumineux ou un grand nombre de fichiers pour saturer
          l'espace disque ou les ressources serveur (CPU/mémoire lors du traitement).
        * **Fichiers "Bombes" (Zip Bomb, Image Bomb) :** Fichiers compressés ou images spécialement conçus qui
          consomment d'énormes ressources lors de leur décompression ou traitement.
    * **Mitigation :**
        * **Ne Jamais Faire Confiance au Nom de Fichier Original :** Toujours nettoyer et/ou générer un nom de fichier
          unique et sûr côté serveur. Supprimer les caractères spéciaux, les `../`.
        * **Validation Stricte du Type de Fichier :** Utiliser l'analyse de contenu (Magic Numbers, bibliothèques comme
          Tika), pas seulement l'extension ou le `Content-Type`. N'autoriser qu'une liste blanche de types nécessaires.
        * **Limites de Taille Strictes :** Configurer `max-file-size` et `max-request-size` à des valeurs raisonnables.
        * **Stockage Sécurisé :** Stocker les fichiers uploadés **en dehors** de la racine web (document root) du
          serveur. Servir les fichiers via un contrôleur dédié qui vérifie les permissions, plutôt que par accès direct.
        * **Configuration Serveur :** S'assurer que le serveur web n'est pas configuré pour exécuter des scripts depuis
          le répertoire d'upload.
        * **Scanner Antivirus :** Si possible, scanner les fichiers uploadés avec un antivirus.
        * **Traitement Asynchrone / Limitation :** Pour les traitements coûteux (redimensionnement d'image, analyse de
          PDF), les effectuer de manière asynchrone et limiter les ressources allouées pour éviter les DoS.

5. **Validation d'Entrée Insuffisante**
    * **Menace :** Ne pas valider correctement les données peut entraîner des erreurs applicatives, des corruptions de
      données, et ouvrir la voie à d'autres attaques (comme XSS ou même SQL Injection si les données validées sont
      utilisées de manière non sûre plus loin).
    * **Mitigation :**
        * **Valider TOUTES les données entrantes :** Utiliser Bean Validation (`@Valid`), les validateurs personnalisés.
        * **Valider côté Serveur :** La validation côté client (JavaScript) est utile pour l'UX mais ne doit **jamais**
          être la seule ligne de défense, car elle peut être facilement contournée. La validation côté serveur est *
          *obligatoire**.
        * **Valider le Type, le Format, la Longueur, la Plage, l'Appartenance à un Ensemble:** Soyez aussi spécifique
          que possible.

6. **Exposition d'Informations Sensibles dans les Messages d'Erreur**
    * **Menace :** Des messages d'erreur trop détaillés (par exemple, des stack traces complètes, des fragments de
      requêtes SQL) peuvent révéler des informations sur la structure de l'application, les versions des logiciels, ou
      les schémas de base de données, aidant un attaquant.
    * **Mitigation :**
        * **Messages d'Erreur Génériques :** Afficher des messages d'erreur conviviaux et génériques à l'utilisateur
          final.
        * **Journalisation Détaillée :** Logger les erreurs détaillées (stack traces, etc.) côté serveur uniquement,
          dans des fichiers de logs sécurisés, pour le débogage par les développeurs/ops.
        * **Gestion Globale des Exceptions :** Utiliser `@ControllerAdvice` et `@ExceptionHandler` pour intercepter les
          exceptions non gérées et renvoyer des pages d'erreur standardisées.

7. **Utilisation de HTTPS**
    * **Menace :** Si le formulaire est soumis via HTTP (non crypté), les données (y compris les mots de passe,
      informations personnelles) peuvent être interceptées en clair sur le réseau (attaque Man-in-the-Middle).
    * **Mitigation :**
        * **Utiliser HTTPS Partout :** Configurer le serveur pour utiliser TLS/SSL (HTTPS). Forcer toutes les connexions
          à utiliser HTTPS (par exemple, via une redirection de HTTP vers HTTPS ou l'en-tête HSTS - HTTP Strict
          Transport Security).

**En Résumé : La Sécurité des Formulaires est Multi-couches**

* **Validation d'entrée stricte** (côté serveur !)
* **Échappement en sortie** (Output Encoding)
* **Utilisation de DTO** (prévention Mass Assignment)
* **Protection anti-CSRF** (tokens)
* **Gestion sécurisée des uploads** (validation type/taille, stockage sûr, noms uniques)
* **HTTPS**
* **Configuration sécurisée du serveur et des dépendances**
* **Messages d'erreur contrôlés**
* **Mises à jour régulières** (Framework, serveur, bibliothèques)

---

**8. Bonnes Pratiques et Patterns**

1. **Pattern Post-Redirect-Get (PRG)**
    * **Problème :** Si un utilisateur soumet un formulaire (POST) et que le serveur répond directement avec la page de
      succès, que se passe-t-il si l'utilisateur rafraîchit la page (F5) ? Le navigateur va proposer de renvoyer la
      requête POST, entraînant potentiellement une double soumission (par exemple, créer deux fois le même produit,
      effectuer deux fois un paiement).
    * **Solution PRG :**
        1. L'utilisateur soumet le formulaire (requête **POST**).
        2. Le serveur traite la requête (validation, sauvegarde BDD, etc.).
        3. Si succès, le serveur ne renvoie pas directement de contenu HTML, mais une réponse de **Redirection** (HTTP
           302 ou 303) vers une URL différente (par exemple, la page de liste des produits, une page de succès dédiée).
        4. Le navigateur reçoit la redirection et effectue une nouvelle requête **GET** vers la nouvelle URL.
        5. Le serveur répond à la requête GET avec la page de succès/liste.
    * **Avantages :**
        * Empêche les doubles soumissions lors du rafraîchissement.
        * L'URL dans le navigateur après succès est celle de la page GET (plus propre, peut être mise en favori).
        * Sépare l'action de modification (POST) de l'action d'affichage (GET).
    * **Implémentation Spring :** Retourner une chaîne commençant par `redirect:` depuis la méthode du contrôleur POST.
      ```java
      @PostMapping("/create")
      public String createProduct(/*...*/) {
          if (bindingResult.hasErrors()) {
              return "product-form"; // Retour au formulaire si erreur
          }
          productService.createProductFromDTO(productDTO, storedFileName);
          // Redirection vers l'URL /products/list
          return "redirect:/products/list";
      }
      ```

2. **Utiliser les DTO Systématiquement :** Comme mentionné, pour la sécurité, la clarté et la séparation des
   préoccupations.

3. **Validation Côté Serveur Indispensable :** Ne jamais se fier à la validation côté client.

4. **Feedback Utilisateur Clair :**
    * Afficher les erreurs de validation clairement, près des champs concernés.
    * Utiliser des messages flash (Flash Attributes) pour afficher des messages de succès ou d'erreur après une
      redirection (PRG). Spring supporte les `RedirectAttributes` pour cela.
      ```java
      import org.springframework.web.servlet.mvc.support.RedirectAttributes;

      @PostMapping("/create")
      public String createProduct(
              @Valid @ModelAttribute("productDTO") ProductDTO productDTO,
              BindingResult bindingResult,
              RedirectAttributes redirectAttributes) { // Injecter RedirectAttributes

          if (bindingResult.hasErrors()) {
              return "product-form";
          }

          try {
               Product createdProduct = productService.createProductFromDTO(productDTO, /*...*/);
               // Ajouter un message flash qui survivra à la redirection
               redirectAttributes.addFlashAttribute("successMessage",
                   "Produit '" + createdProduct.getName() + "' créé avec succès !");
               return "redirect:/products/list";
          } catch (Exception e) {
               log.error("Erreur lors de la création du produit", e);
               // Ajouter un message d'erreur au modèle actuel (pas redirect)
               model.addAttribute("errorMessage", "Erreur serveur lors de la création.");
               // Ou si on redirige malgré l'erreur vers une page d'erreur
               // redirectAttributes.addFlashAttribute("errorMessage", "Erreur...");
               // return "redirect:/some-error-page";
               return "product-form"; // Retour au formulaire avec erreur générale
          }
      }

      // Dans la vue de destination (ex: product-list.html)
      // Afficher le message flash s'il existe
      <div th:if="${successMessage}" class="alert alert-success" th:text="${successMessage}">
          Message de succès
      </div>
      <div th:if="${errorMessage}" class="alert alert-danger" th:text="${errorMessage}">
          Message d'erreur
      </div>

      ```

5. **Contrôleurs Légers (Thin Controllers) :** Le rôle principal du contrôleur est de recevoir la requête, déléguer le
   traitement métier à une couche de service, et sélectionner la vue appropriée. Éviter de mettre de la logique métier
   complexe (calculs, accès BDD direct) dans le contrôleur.

6. **Internationalisation (i18n) :** Externaliser tous les textes visibles par l'utilisateur (libellés de formulaire,
   messages d'erreur, messages de succès) dans des fichiers de propriétés (par exemple, `messages_fr.properties`,
   `messages_en.properties`). Utiliser le `MessageSource` de Spring et les fonctionnalités i18n du moteur de template (
   Thymeleaf : `#{message.code}`). Bean Validation s'intègre bien avec cela pour les messages d'erreur.

7. **Tests Unitaires et d'Intégration :**
    * Tester les validateurs personnalisés unitairement.
    * Tester les mappings DTO<->Entité.
    * Tester les contrôleurs (par exemple avec `MockMvc`) pour vérifier le data binding, la validation, les
      redirections, et la sélection de vue.

---

**9. Conclusion**

La gestion des formulaires est une partie essentielle du développement web. Spring Framework, avec Spring MVC et Spring
Boot, offre un écosystème puissant et cohérent pour gérer l'ensemble du cycle de vie des formulaires, du rendu initial à
la validation robuste, en passant par le data binding et la gestion des uploads.

L'utilisation systématique des DTO, une validation rigoureuse côté serveur (incluant des validateurs personnalisés si
nécessaire), la compréhension et l'application des principes de sécurité (XSS, CSRF, uploads sécurisés, HTTPS), et
l'adoption de bonnes pratiques comme le pattern PRG sont fondamentales pour construire des applications web fiables,
maintenables et sécurisées. Les outils de mapping comme MapStruct ou ModelMapper peuvent grandement simplifier la
conversion entre les couches de données, améliorant la productivité et réduisant les erreurs.

En maîtrisant ces concepts et techniques, les développeurs peuvent créer des interactions utilisateur fluides et
sécurisées, qui sont au cœur de nombreuses applications web modernes.

---