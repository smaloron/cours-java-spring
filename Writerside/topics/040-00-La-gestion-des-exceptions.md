# La gestion des exceptions

**Pourquoi Gérer les Exceptions ?**

Dans toute application, des erreurs peuvent survenir : une ressource non trouvée, une entrée utilisateur invalide, un
problème de base de données, etc. Une gestion inadéquate des exceptions conduit à :

* **Mauvaise Expérience Utilisateur (UX) :** Des pages d'erreur techniques (comme les traces de pile Java) peuvent
  effrayer ou frustrer les utilisateurs.
* **Failles de Sécurité :** Les traces de pile peuvent exposer des détails internes de l'application (versions de
  librairies, structure du code, requêtes SQL) exploitables par des attaquants.
* **Manque d'Information :** Sans gestion appropriée, il est difficile de diagnostiquer les problèmes survenus en
  production.

Spring Boot fournit plusieurs mécanismes pour intercepter et traiter ces erreurs de manière élégante et contrôlée dans
le contexte d'une application web MVC.

---

## Gestion par Défaut de Spring Boot : La "Whitelabel Error Page"

Lorsqu'une exception non interceptée se produit lors du traitement d'une requête web (ou si une ressource n'est pas
trouvée - erreur 404), Spring Boot, par défaut, redirige l'utilisateur vers une page d'erreur générique connue sous le
nom de "Whitelabel Error Page".

Cette page affiche des informations de base sur l'erreur (timestamp, statut HTTP, message d'erreur, chemin).

**Exemple de Scénario :**
Supposons un contrôleur simple :

```java
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DemoController {

    @GetMapping("/cause-error")
    public String causeError() {
        // Simule une erreur interne
        throw new RuntimeException("Something went wrong internally!");
    }

    @GetMapping("/hello")
    public String sayHello() {
        // Page qui fonctionne normalement
        return "welcome"; // Suppose une vue welcome.html
    }
}
```

Si l'utilisateur accède à `/cause-error`, il verra la "Whitelabel Error Page" indiquant un statut 500 (Internal Server
Error) et le message "Something went wrong internally!". Si une URL inexistante est accédée (ex: `/non-existent-page`),
une Whitelabel Page avec un statut 404 (Not Found) sera affichée.

**Limites :**
La Whitelabel Error Page n'est pas destinée à la production. Elle manque de personnalisation graphique et expose
potentiellement trop d'informations techniques.

---

## Pages d'Erreur Statiques Personnalisées

La manière la plus simple de remplacer la Whitelabel Error Page est de fournir des fichiers HTML statiques nommés selon
le code de statut HTTP.

**Fonctionnement :**
Spring Boot recherche des fichiers dans des emplacements spécifiques du classpath :

* `/static/error/<code>.html`
* `/public/error/<code>.html`
* `/resources/error/<code>.html`
* `/templates/error/<code>.html` (si Thymeleaf ou un autre moteur de template est utilisé et configuré pour cet
  emplacement)

Où `<code>` est le code de statut HTTP (ex: `404`, `500`).

**Exemple : Créer une page 404 personnalisée**

1. Créez le répertoire `src/main/resources/templates/error/` (si vous utilisez Thymeleaf dans
   `src/main/resources/templates/`).
2. Créez un fichier `404.html` dans ce répertoire :

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Page Non Trouvée</title>
    <link rel="stylesheet" th:href="@{/css/style.css}"/> <!-- Optionnel -->
</head>
<body>
<h1>Oops! Page Non Trouvée (Erreur 404)</h1>
<p>
    Désolé, la page que vous cherchez n'existe pas ou a été déplacée.
</p>
<p>
    <a th:href="@{/}">Retour à l'accueil</a>
</p>
</body>
</html>
```

3. Créez de même un fichier `500.html` pour les erreurs serveur internes.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Erreur Interne</title>
    <link rel="stylesheet" th:href="@{/css/style.css}"/> <!-- Optionnel -->
</head>
<body>
<h1>Oops! Une erreur s'est produite (Erreur 500)</h1>
<p>
    Nous rencontrons des difficultés techniques.
    Veuillez réessayer plus tard.
</p>
<p>
    <a th:href="@{/}">Retour à l'accueil</a>
</p>
</body>
</html>
```

Maintenant, en cas d'erreur 404 ou 500, Spring Boot affichera ces pages personnalisées au lieu de la Whitelabel Page.

**Avantages :** Simple à mettre en œuvre pour les erreurs HTTP courantes.
**Inconvénients :** Statique. Ne permet pas d'afficher des informations dynamiques sur l'erreur (sauf si on utilise
`ErrorController`, voir section 6) ni de gérer des exceptions Java spécifiques de manière différenciée.

---

## Gestion Centralisée avec `@ControllerAdvice` et `@ExceptionHandler`

Pour une gestion plus fine et centralisée des exceptions applicatives, Spring AOP (utilisé par Spring Boot) offre les
annotations `@ControllerAdvice` et `@ExceptionHandler`.

* `@ControllerAdvice`: Annotation de classe. Permet de regrouper des configurations globales (`@ExceptionHandler`,
  `@InitBinder`, `@ModelAttribute`) qui s'appliquent à plusieurs (ou tous les) contrôleurs.
* `@ExceptionHandler`: Annotation de méthode. Au sein d'une classe `@Controller` ou `@ControllerAdvice`, cette
  annotation déclare une méthode qui gérera les exceptions d'un type spécifique (ou de ses sous-types).

**Fonctionnement :**
Lorsqu'une exception est levée par une méthode d'un contrôleur, Spring recherche :

1. Une méthode `@ExceptionHandler` au sein du contrôleur lui-même capable de gérer ce type d'exception.
2. Si non trouvée, il cherche une méthode `@ExceptionHandler` dans une classe `@ControllerAdvice` capable de gérer ce
   type d'exception.
3. Si toujours non trouvée, l'exception remonte jusqu'au conteneur de servlets, qui déclenche le mécanisme d'erreur
   standard (menant potentiellement aux pages d'erreur statiques ou à la Whitelabel Page).

**Exemple : Gérer une exception métier spécifique**

1. **Créer une exception personnalisée :**

```java
package fr.formation.spring.exception;

// Exception spécifique si un produit demandé n'est pas trouvé
public class ProductNotFoundException extends RuntimeException {
    private final Long productId;

    public ProductNotFoundException(Long productId) {
        // Message d'erreur générique
        super("Product not found with ID: " + productId);
        this.productId = productId;
    }

    public Long getProductId() {
        return productId;
    }
}
```

2. **Lever l'exception dans un contrôleur :**

```java
package fr.formation.spring.controller;

import fr.formation.spring.exception.ProductNotFoundException;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/products")
public class ProductController {

    // Simule la récupération d'un produit
    @GetMapping("/{id}")
    public String getProductById(@PathVariable Long id, Model model) {
        if (id == 0) { // Simule un produit non trouvé
            throw new ProductNotFoundException(id);
        }
        // Logique pour récupérer le produit et l'ajouter au modèle
        model.addAttribute("productName", "Awesome Product " + id);
        return "product-details"; // Vue product-details.html
    }
}
```

3. **Créer le gestionnaire d'exceptions global (`@ControllerAdvice`) :**

```java
package fr.formation.spring.advice;

import fr.formation.spring.exception.ProductNotFoundException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.servlet.ModelAndView; // Important pour retourner une vue

import jakarta.servlet.http.HttpServletRequest; // Pour obtenir des infos requête

@ControllerAdvice // S'applique à tous les contrôleurs
public class GlobalExceptionHandler {

    private static final Logger log =
            LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Gère spécifiquement ProductNotFoundException
    @ExceptionHandler(ProductNotFoundException.class)
    public ModelAndView handleProductNotFound(
            HttpServletRequest request, // Info sur la requête originale
            ProductNotFoundException ex) { // L'exception capturée

        log.error("ProductNotFoundException occurred for ID {}: Request: {}",
                ex.getProductId(), request.getRequestURI(), ex);

        // Prépare le modèle et la vue à retourner
        ModelAndView modelAndView = new ModelAndView();
        // Ajoute des attributs pour la vue
        modelAndView.addObject("errorMessage", ex.getMessage());
        modelAndView.addObject("requestedId", ex.getProductId());
        modelAndView.addObject("requestedUrl", request.getRequestURL());

        // Définit la vue à afficher (ex: error/product-not-found.html)
        modelAndView.setViewName("error/product-not-found");

        return modelAndView;
    }

    // Gestionnaire générique pour d'autres exceptions (facultatif mais recommandé)
    @ExceptionHandler(Exception.class)
    public ModelAndView handleGenericException(
            HttpServletRequest request,
            Exception ex) {

        log.error("Unhandled exception occurred: Request: {}",
                request.getRequestURI(), ex);

        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("errorMessage",
                "An unexpected error occurred. Please try again later.");
        modelAndView.addObject("requestedUrl", request.getRequestURL());
        // Vue générique (ex: error/generic-error.html ou la 500.html)
        modelAndView.setViewName("error/generic-error");

        return modelAndView;
    }
}
```

4. **Créer la vue d'erreur spécifique (`src/main/resources/templates/error/product-not-found.html`) :**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Produit Non Trouvé</title>
</head>
<body>
<h1>Produit Non Trouvé</h1>
<p th:text="${errorMessage}">Message d'erreur par défaut</p>
<p>
    L'identifiant demandé était :
    <strong th:text="${requestedId}">ID inconnu</strong>
</p>
<p>URL demandée : <span th:text="${requestedUrl}">URL</span></p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

5. **Créer la vue d'erreur générique (`src/main/resources/templates/error/generic-error.html`) :**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Erreur Inattendue</title>
</head>
<body>
<h1>Une Erreur s'est Produite</h1>
<p th:text="${errorMessage}">Message d'erreur par défaut</p>
<p>URL demandée : <span th:text="${requestedUrl}">URL</span></p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

**Fonctionnement détaillé :**
Si l'URL `/products/0` est accédée :

1. `ProductController.getProductById` lève `ProductNotFoundException`.
2. Spring intercepte l'exception.
3. Il trouve `GlobalExceptionHandler.handleProductNotFound` via `@ControllerAdvice` et
   `@ExceptionHandler(ProductNotFoundException.class)`.
4. Cette méthode est exécutée. Elle logue l'erreur, crée un `ModelAndView`, y ajoute des données (message, ID, URL), et
   spécifie la vue `error/product-not-found`.
5. Spring Boot rend la vue `error/product-not-found.html` en utilisant les données du modèle.

**Gestion de plusieurs exceptions :**
Une méthode `@ExceptionHandler` peut gérer plusieurs types d'exceptions :

```java

@ExceptionHandler({OrderNotFoundException.class, InvoiceNotFoundException.class})
public ModelAndView handleResourceNotFound(HttpServletRequest request, Exception ex) {
    // ... logique commune pour ces exceptions ...
    ModelAndView mav = new ModelAndView();
    mav.addObject("errorMessage", "The requested resource was not found.");
    mav.addObject("details", ex.getMessage()); // Message spécifique de l'exception
    mav.setViewName("error/resource-not-found"); // Vue générique pour "non trouvé"
    return mav;
}
```

**Ordre de priorité :** Spring choisit le gestionnaire le plus spécifique. Un `@ExceptionHandler(Exception.class)` est
un gestionnaire "fourre-tout" qui ne sera appelé que si aucun gestionnaire plus précis n'est trouvé.

**Avantages :**

* Gestion centralisée et réutilisable du code de gestion d'erreur.
* Possibilité de retourner des vues spécifiques avec des données dynamiques.
* Contrôle fin sur la logique de gestion (log, notification, etc.).

**Inconvénients :**

* Ne gère pas nativement les erreurs qui surviennent *avant* d'atteindre un contrôleur (ex: erreur 404 pour une URL non
  mappée, erreur 400 pour une conversion de type échouée sur un paramètre de méthode avant l'appel de la méthode
  elle-même). Ces erreurs sont généralement gérées par le mécanisme d'`ErrorController` ou les pages statiques.

---

## Mapper les Exceptions à des Statuts HTTP avec `@ResponseStatus`

L'annotation `@ResponseStatus` peut être placée directement sur une classe d'exception personnalisée ou sur une méthode
`@ExceptionHandler`.

* **Sur une classe d'exception :** Indique le statut HTTP à retourner par défaut lorsque cette exception n'est pas gérée
  par un `@ExceptionHandler` spécifique qui définirait lui-même un statut ou une vue.

```java
package fr.formation.spring.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

// Associe cette exception au statut HTTP 404 Not Found
@ResponseStatus(value = HttpStatus.NOT_FOUND, reason = "Requested product does not exist")
public class ProductNotFoundException extends RuntimeException {
    // ... constructeur et méthodes ...
    public ProductNotFoundException(Long productId) {
        super("Product not found with ID: " + productId);
        // ...
    }
}
```

Si cette exception est levée et *non* interceptée par un `@ExceptionHandler`, Spring utilisera le statut `404` et la
`reason` (si fournie). Le corps de la réponse dépendra alors des autres mécanismes (page `404.html`, `ErrorController`,
Whitelabel Page).

* **Sur une méthode `@ExceptionHandler` :** Permet de forcer un statut HTTP spécifique pour la réponse, même si la
  méthode retourne un `ModelAndView`.

```java

@ControllerAdvice
public class GlobalExceptionHandler {

    // ... autres handlers ...

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST) // Force un statut 400
    public ModelAndView handleIllegalArgument(
            HttpServletRequest request,
            IllegalArgumentException ex) {

        log.warn("Bad request received: {} - Request: {}",
                ex.getMessage(), request.getRequestURI());

        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("errorMessage", "Invalid input: " + ex.getMessage());
        modelAndView.setViewName("error/bad-request"); // Vue spécifique pour 400
        return modelAndView;
    }
}
```

Ici, même si on retourne une vue (`error/bad-request`), la réponse HTTP aura bien le statut 400. Sans `@ResponseStatus`
ici, le statut serait par défaut 200 (OK) car on retourne une vue avec succès.

**Utilisation conjointe :** On peut avoir `@ResponseStatus` sur l'exception ET un `@ExceptionHandler`. L'annotation sur
le handler a priorité si elle est présente.

---

## Personnaliser le Comportement d'Erreur avec `ErrorController`

Pour un contrôle ultime sur le rendu de *toutes* les erreurs qui aboutissent au chemin `/error` (y compris les 404 non
mappées, les erreurs de conteneur, et les exceptions non gérées par `@ExceptionHandler` qui provoquent une redirection
vers `/error`), on peut implémenter l'interface `ErrorController` de Spring Boot.

Spring Boot fournit une implémentation par défaut : `BasicErrorController`. C'est elle qui gère l'affichage de la
Whitelabel Error Page ou le rendu des vues `error/<code>.html` si elles existent (en passant quelques attributs d'erreur
de base).

**Pourquoi personnaliser `ErrorController` ?**

* Pour utiliser une logique complexe afin de déterminer quelle vue afficher.
* Pour ajouter des attributs personnalisés au modèle de la page d'erreur, basés sur les détails de l'erreur récupérés
  via `ErrorAttributes`.
* Pour avoir un point central de gestion des erreurs HTTP génériques (4xx, 5xx) qui ne correspondent pas à des
  exceptions applicatives spécifiques.

**Exemple : Créer un `CustomErrorController`**

1. **Implémenter `ErrorController` :**

```java
package fr.formation.spring.controller;

import org.springframework.boot.web.servlet.error.ErrorController;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.boot.web.error.ErrorAttributeOptions;
import org.springframework.boot.web.servlet.error.ErrorAttributes;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.http.HttpStatus;

import jakarta.servlet.RequestDispatcher;
import jakarta.servlet.http.HttpServletRequest;

import java.util.Map;

@Controller
public class CustomErrorController implements ErrorController {

    private final ErrorAttributes errorAttributes;

    // Injecte ErrorAttributes pour obtenir les détails de l'erreur
    public CustomErrorController(ErrorAttributes errorAttributes) {
        this.errorAttributes = errorAttributes;
    }

    // Mappe les requêtes vers /error
    @RequestMapping("/error")
    public String handleError(HttpServletRequest request, Model model) {
        // Récupère les attributs d'erreur standards
        ServletWebRequest webRequest = new ServletWebRequest(request);
        Map<String, Object> errorDetails = this.errorAttributes.getErrorAttributes(
                webRequest,
                ErrorAttributeOptions.of( // Options pour inclure plus de détails
                        ErrorAttributeOptions.Include.MESSAGE,
                        ErrorAttributeOptions.Include.EXCEPTION, // Attention en prod !
                        ErrorAttributeOptions.Include.STACK_TRACE // Attention en prod !
                )
        );

        // Récupère le statut HTTP de l'erreur
        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        int statusCode = HttpStatus.INTERNAL_SERVER_ERROR.value(); // Défaut 500
        if (status != null) {
            statusCode = Integer.parseInt(status.toString());
        }

        // Ajoute les détails au modèle pour la vue
        model.addAttribute("statusCode", statusCode);
        model.addAttribute("timestamp", errorDetails.get("timestamp"));
        model.addAttribute("errorMessage", errorDetails.get("message"));
        model.addAttribute("errorDetails", errorDetails.get("error")); // Type d'erreur
        model.addAttribute("path", errorDetails.get("path"));
        // N'exposez JAMAIS la trace complète en production !
        // model.addAttribute("trace", errorDetails.get("trace"));

        // Logique pour choisir la vue en fonction du statut
        if (statusCode == HttpStatus.NOT_FOUND.value()) {
            // Utilise la vue 404.html (ou une autre vue spécifique)
            return "error/404"; // Assurez-vous que templates/error/404.html existe
        } else if (statusCode == HttpStatus.FORBIDDEN.value()) {
            // Utilise la vue 403.html
            return "error/403";
        }
        // Pour toutes les autres erreurs (500, etc.)
        // Utilise la vue 500.html ou une vue générique
        return "error/generic-error"; // Assurez-vous qu'elle existe
    }

    // getErrorPath() est déprécié, l'annotation @RequestMapping("/error") suffit
    // @Override
    // public String getErrorPath() {
    //     return "/error";
    // }
}
```

*Note :* `ErrorAttributes` permet d'accéder aux informations d'erreur collectées par Spring Boot.
`ErrorAttributeOptions` contrôle le niveau de détail inclus. Soyez prudent avec `EXCEPTION` et `STACK_TRACE` en
production.

2. **Créer les vues correspondantes** (ex: `error/404.html`, `error/403.html`, `error/generic-error.html` dans
   `src/main/resources/templates/`) et utiliser les attributs passés via le `Model`.

Exemple `error/generic-error.html` utilisant les attributs :

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Erreur</title>
</head>
<body>
<h1 th:text="'Erreur ' + ${statusCode} + ' : ' + ${errorDetails}">
    Erreur Générique
</h1>
<p>Timestamp: <span th:text="${timestamp}"></span></p>
<p>Chemin: <span th:text="${path}"></span></p>
<p>Message: <span th:text="${errorMessage}"></span></p>
<!-- Ne pas afficher la trace en production -->
<!-- <pre th:text="${trace}"></pre> -->
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

**Avantages :**

* Contrôle total sur le rendu de toutes les erreurs redirigées vers `/error`.
* Point unique pour gérer les erreurs HTTP standards (4xx, 5xx) de manière cohérente.
* Permet d'enrichir les pages d'erreur avec des informations dynamiques détaillées.

**Inconvénients :**

* Plus complexe à mettre en place que les pages statiques ou les `@ExceptionHandler` simples.
* `BasicErrorController` fait déjà beaucoup, la personnalisation n'est nécessaire que pour des besoins avancés.

---

## Bonnes Pratiques et Conseils

1. **Loggez Systématiquement les Exceptions :** Utilisez un framework de logging (Logback - par défaut dans Spring Boot,
   Log4j2, etc.) pour enregistrer les détails complets des exceptions (y compris la trace de pile) côté serveur. C'est
   crucial pour le débogage. Les `@ExceptionHandler` et `ErrorController` sont de bons endroits pour loguer.
2. **Ne Pas Exposer les Détails Techniques aux Utilisateurs :** Affichez des messages d'erreur clairs, conviviaux et non
   techniques. Ne montrez jamais de traces de pile ou de messages d'erreur bruts de la base de données sur la page
   rendue.
3. **Utilisez des Pages d'Erreur Personnalisées :** Remplacez toujours la Whitelabel Error Page en production par des
   pages qui correspondent à l'identité visuelle de votre application.
4. **Soyez Spécifique avec `@ExceptionHandler` :** Créez des gestionnaires pour les exceptions spécifiques que vous
   prévoyez et que vous pouvez traiter de manière particulière (ex: `ValidationException`, `ProductNotFoundException`).
5. **Ayez un Gestionnaire Global :** Implémentez un `@ExceptionHandler(Exception.class)` ou utilisez un
   `ErrorController` personnalisé comme filet de sécurité pour toutes les erreurs inattendues.
6. **Utilisez les Statuts HTTP Corrects :** Mappez les exceptions aux statuts HTTP appropriés (404 pour non trouvé, 400
   pour mauvaise requête, 403 pour interdit, 500 pour erreur serveur, etc.) en utilisant `@ResponseStatus` ou en les
   définissant dans votre `ErrorController`.
7. **Différenciez les Erreurs Client (4xx) et Serveur (5xx) :** Les erreurs 4xx indiquent généralement un problème avec
   la requête du client (données invalides, ressource inexistante). Les erreurs 5xx indiquent un problème interne au
   serveur. Votre logging et la communication à l'utilisateur peuvent différer. Loguez les 5xx comme des erreurs (
   `ERROR`), les 4xx peut-être comme des avertissements (`WARN`) ou infos (`INFO`) selon le cas.
8. **Centralisez avec `@ControllerAdvice` :** Préférez `@ControllerAdvice` pour la gestion globale plutôt que de
   dupliquer des `@ExceptionHandler` dans chaque contrôleur.
9. **Pensez à l'Internationalisation (i18n) :** Vos messages d'erreur affichés à l'utilisateur devraient être
   internationalisables si votre application supporte plusieurs langues.

---

## Cas d'Utilisation Typiques

* **Validation des Formulaires :** Une soumission de formulaire échoue à la validation (ex: champ email invalide). Une
  `BindException` ou `MethodArgumentNotValidException` est levée. Un `@ExceptionHandler` peut intercepter cette
  exception, renvoyer l'utilisateur au formulaire avec les messages d'erreur affichés près des champs concernés (Statut
  HTTP souvent 200 OK car on ré-affiche une page valide, mais avec des erreurs).
* **Ressource Non Trouvée :** Un utilisateur demande un produit avec un ID qui n'existe pas (`/products/999`). Le
  service lève une `ProductNotFoundException`. Un `@ExceptionHandler` intercepte, logue l'ID, et retourne une vue
  `error/product-not-found.html` avec un message convivial et un statut 404.
* **Accès Interdit :** Un utilisateur non administrateur tente d'accéder à une page d'administration. Spring Security (
  ou une logique manuelle) lève une `AccessDeniedException`. Un `@ExceptionHandler` ou la configuration de Spring
  Security redirige vers une page d'erreur 403 personnalisée.
* **Erreur Interne Inattendue :** Une erreur de connexion à la base de données survient. Une `DataAccessException` (ou
  une exception plus spécifique) est levée. Le gestionnaire `@ExceptionHandler(Exception.class)` ou le
  `CustomErrorController` intercepte, logue la trace de pile complète, et affiche une page d'erreur générique 500
  demandant à l'utilisateur de réessayer plus tard.
* **Conflit Métier :** Tentative de suppression d'une catégorie contenant encore des produits. Le service lève une
  `CategoryNotEmptyException`. Un `@ExceptionHandler` intercepte, retourne un statut 409 (Conflict) et affiche une page
  expliquant pourquoi la suppression est impossible.

---

## Exercices Pratiques

### Exercice 1 : Gérer une Nouvelle Exception Métier

**Énoncé :**

- Créez une nouvelle exception personnalisée `OrderProcessingException` dans le package `fr.formation.spring.
exception`. Elle prendra un `orderId` (type `String`) et un message dans son constructeur.
- Créez un `OrderController` simple avec une méthode GET `/orders/{orderId}`. Si `orderId` est égal à "INVALID",
  levez la `OrderProcessingException`. Sinon, affichez une vue de succès simple (`order-details.html`).
- Dans votre `GlobalExceptionHandler` (la classe `@ControllerAdvice`), ajoutez une méthode `@ExceptionHandler` pour
  intercepter `OrderProcessingException`.
- Cette méthode doit loguer l'erreur (niveau WARN) avec l'ID de commande.
- Elle doit retourner un `ModelAndView` pointant vers une nouvelle vue `error/order-error.html`.
- Passez l'ID de la commande et le message d'erreur de l'exception à la vue.
- Créez la vue `error/order-error.html` (Thymeleaf) affichant ces informations et un lien vers l'accueil.
- Testez en accédant à `/orders/VALID123` (devrait réussir) et `/orders/INVALID` (devrait montrer la page d'erreur).

#### **Correction Exercice 1** {collapsible="true"}

**`OrderProcessingException.java` :**

```java
    package fr.formation.spring.exception;

public class OrderProcessingException extends RuntimeException {
    private final String orderId;

    public OrderProcessingException(String orderId, String message) {
        super(message);
        this.orderId = orderId;
    }

    public String getOrderId() {
        return orderId;
    }
}

```

---

**`OrderController.java` :**

```java
    package fr.formation.spring.controller;

import fr.formation.spring.exception.OrderProcessingException;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{orderId}")
    public String getOrderById(@PathVariable String orderId, Model model) {
        if ("INVALID".equalsIgnoreCase(orderId)) {
            throw new OrderProcessingException(
                    orderId,
                    "Order processing failed due to invalid data."
            );
        }
        model.addAttribute("orderId", orderId);
        // Suppose l'existence de templates/order-details.html
        return "order-details";
    }
}
```

---

**Ajout dans `GlobalExceptionHandler.java` :**

```java
    // ... (imports et logger existants) ...

import fr.formation.spring.exception.OrderProcessingException; // Ajouter cet import

@ControllerAdvice
public class GlobalExceptionHandler {

    // ... (logger et autres handlers existants) ...

    @ExceptionHandler(OrderProcessingException.class)
    public ModelAndView handleOrderProcessingException(
            HttpServletRequest request,
            OrderProcessingException ex) {

        log.warn("OrderProcessingException for order {}: {} - Request: {}",
                ex.getOrderId(), ex.getMessage(), request.getRequestURI());

        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("errorMessage", ex.getMessage());
        modelAndView.addObject("failedOrderId", ex.getOrderId());
        modelAndView.addObject("requestedUrl", request.getRequestURL());
        // Définit la vue d'erreur spécifique
        modelAndView.setViewName("error/order-error");

        return modelAndView;
    }

    // ... (handleProductNotFound et handleGenericException) ...
}
```

---
**`src/main/resources/templates/error/order-error.html` :**

```html
    <!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Erreur Commande</title>
</head>
<body>
<h1>Erreur lors du Traitement de la Commande</h1>
<p>Commande ID: <strong th:text="${failedOrderId}">[inconnu]</strong></p>
<p>Message: <em th:text="${errorMessage}">[erreur]</em></p>
<p>URL: <span th:text="${requestedUrl}"></span></p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```
*   **`src/main/resources/templates/order-details.html` (pour le cas succès) :**
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Détails Commande</title>
</head>
<body>
<h1>Détails de la Commande</h1>
<p>Commande ID: <strong th:text="${orderId}">[inconnu]</strong> traitée avec succès.</p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

### Exercice 2 : Utiliser `@ResponseStatus` et une Page Statique

**Énoncé :**

- Créez une nouvelle exception `ResourceForbiddenException` dans `fr.formation.spring.exception`. Annotez cette
  classe avec `@ResponseStatus(HttpStatus.FORBIDDEN)`. Elle prendra juste un message dans son constructeur.
- Dans un contrôleur (par exemple, `DemoController`), créez une méthode GET `/admin-only` qui lève systématiquement
  cette `ResourceForbiddenException`.
- Créez une page d'erreur *statique* pour le code 403. Placez un fichier `403.html` dans
  `src/main/resources/templates/error/` (ou `src/main/resources/static/error/` si vous n'utilisez pas Thymeleaf pour les
  erreurs statiques). Cette page doit simplement afficher un message indiquant que l'accès est interdit.
- Assurez-vous qu'il n'y a *pas* de `@ExceptionHandler` spécifique pour `ResourceForbiddenException` dans votre
  `@ControllerAdvice` (pour laisser `@ResponseStatus` et la page statique agir).
- Testez en accédant à `/admin-only`. Vous devriez voir votre page `403.html` personnalisée.

#### **Correction Exercice 2** {collapsible="true"}

---

**`ResourceForbiddenException.java` :**

```java
    package fr.formation.spring.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.ResponseStatus;

@ResponseStatus(value = HttpStatus.FORBIDDEN, reason = "Access to this resource is forbidden")
public class ResourceForbiddenException extends RuntimeException {
    public ResourceForbiddenException(String message) {
        super(message);
    }
}
```

---

**Ajout dans `DemoController.java` (ou un autre contrôleur) :**

```java
    package fr.formation.spring.controller;

import fr.formation.spring.exception.ResourceForbiddenException;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DemoController {

    // ... (autres méthodes comme causeError, sayHello) ...

    @GetMapping("/admin-only")
    public String adminOnlyResource() {
        // Simule une ressource nécessitant des droits spécifiques
        throw new ResourceForbiddenException(
                "You do not have permission to access this page."
        );
    }
}
```

**`src/main/resources/templates/error/403.html` :**

```html
    <!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Accès Interdit</title>
    <style>body {
        font-family: sans-serif;
        padding: 20px;
    }</style>
</head>
<body>
<h1>Accès Interdit (Erreur 403)</h1>
<p>
    Désolé, vous n'avez pas les permissions nécessaires
    pour accéder à cette page.
</p>
<p>
    <a href="/">Retour à l'accueil</a>
</p>
</body>
</html>
```

**Vérification :** Assurez-vous qu'aucun `@ExceptionHandler(ResourceForbiddenException.class)` n'existe dans
`GlobalExceptionHandler`. Le `@ExceptionHandler(Exception.class)` générique ne devrait pas non plus interférer si la
gestion par statut HTTP via `ErrorController` ou page statique est prioritaire pour les erreurs 4xx/5xx.

### Exercice 3 : Personnaliser `ErrorController` pour les Erreurs 404

**Énoncé :**

- Implémentez (ou modifiez) le `CustomErrorController` vu dans la section 6.
- Assurez-vous qu'il injecte `ErrorAttributes`.
- Dans la méthode `handleError`, récupérez les attributs d'erreur (`timestamp`, `status`, `error`, `path`, `message`).
- Si le `statusCode` est 404 :
    - Ajoutez un attribut supplémentaire au modèle : `customMessage404` avec la valeur "La ressource demandée via l'URL
      fournie est introuvable.".
    - Retournez le nom de la vue `error/404-custom` (vous devrez créer ce fichier).
- Pour tous les autres codes d'erreur, retournez la vue `error/generic-error` (utilisez celle de la section 6 ou
  créez-en une simple).
- Créez la vue `error/404-custom.html` qui affiche les informations d'erreur standards *et* le message personnalisé
  `customMessage404`.
- Testez en accédant à une URL qui n'existe pas (ex: `/this-url-does-not-exist`). Vous devriez voir votre page
  `404-custom.html` avec toutes les informations.
- Testez en déclenchant une erreur 500 (ex: via `/cause-error` du `DemoController`). Vous devriez voir la page
  `generic-error.html`.

#### **Correction Exercice 3**

**`CustomErrorController.java` (potentiellement mis à jour) :**

```java
    package fr.formation.spring.controller;

// ... (imports existants de la section 6) ...

@Controller
public class CustomErrorController implements ErrorController {

    private final ErrorAttributes errorAttributes;

    public CustomErrorController(ErrorAttributes errorAttributes) {
        this.errorAttributes = errorAttributes;
    }

    @RequestMapping("/error")
    public String handleError(HttpServletRequest request, Model model) {
        ServletWebRequest webRequest = new ServletWebRequest(request);
        // Obtenir les attributs, inclure le message par défaut
        Map<String, Object> errorDetails = this.errorAttributes.getErrorAttributes(
                webRequest, ErrorAttributeOptions.defaults().including(
                        ErrorAttributeOptions.Include.MESSAGE)
        );

        Object status = request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        int statusCode = HttpStatus.INTERNAL_SERVER_ERROR.value();
        if (status != null) {
            statusCode = Integer.parseInt(status.toString());
        }

        model.addAttribute("statusCode", statusCode);
        model.addAttribute("timestamp", errorDetails.get("timestamp"));
        model.addAttribute("errorMessage", errorDetails.get("message"));
        model.addAttribute("errorDetails", errorDetails.get("error"));
        model.addAttribute("path", errorDetails.get("path"));

        // Logique spécifique pour 404
        if (statusCode == HttpStatus.NOT_FOUND.value()) {
            model.addAttribute(
                    "customMessage404",
                    "La ressource demandée via l'URL fournie est introuvable."
            );
            return "error/404-custom"; // Vue spécifique pour 404
        }

        // Vue par défaut pour les autres erreurs
        return "error/generic-error";
    }
}
```

---

**`src/main/resources/templates/error/404-custom.html` :**

```html
    <!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Page Non Trouvée (404)</title>
</head>
<body>
<h1>Erreur 404 - Page Non Trouvée</h1>
<p style="color: red; font-weight: bold;" th:text="${customMessage404}">
    Message spécifique 404.
</p>
<hr/>
<h2>Détails Techniques :</h2>
<p>Timestamp: <span th:text="${timestamp}"></span></p>
<p>Statut: <span th:text="${statusCode}"></span></p>
<p>Erreur: <span th:text="${errorDetails}"></span></p>
<p>Chemin: <span th:text="${path}"></span></p>
<p>Message Système: <span th:text="${errorMessage}"></span></p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

---

**`src/main/resources/templates/error/generic-error.html` (s'assurer qu'elle existe, comme celle de la section 6) :**

```html
    <!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Erreur Inattendue</title>
</head>
<body>
<h1>Une Erreur s'est Produite (<span th:text="${statusCode}"></span>)</h1>
<p>Timestamp: <span th:text="${timestamp}"></span></p>
<p>Chemin: <span th:text="${path}"></span></p>
<p>Erreur: <span th:text="${errorDetails}"></span></p>
<p>Message: <span th:text="${errorMessage}"></span></p>
<p><a th:href="@{/}">Retour à l'accueil</a></p>
</body>
</html>
```

---

## Conclusion

La gestion des exceptions est une partie intégrante du développement d'applications web robustes et conviviales. Spring
Boot offre une gamme d'outils flexibles, allant des simples pages d'erreur statiques à la gestion centralisée et
dynamique avec `@ControllerAdvice` et `@ExceptionHandler`, en passant par la personnalisation fine via
`ErrorController`.

Le choix de la stratégie dépendra de la complexité de l'application et du niveau de contrôle souhaité. Une combinaison
de ces approches est souvent la solution la plus efficace : utiliser des pages statiques ou un `ErrorController` pour
les erreurs HTTP génériques (404, 500), et des `@ExceptionHandler` dans un `@ControllerAdvice` pour les exceptions
métier spécifiques qui nécessitent une logique ou une présentation particulière.

En appliquant les bonnes pratiques (logging, messages clairs, statuts HTTP corrects), les développeurs peuvent
significativement améliorer la qualité et la maintenabilité de leurs applications Spring Boot.
