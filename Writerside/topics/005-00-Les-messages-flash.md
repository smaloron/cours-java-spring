# Les messages flash

Dans le développement d'applications web, il est fréquent de devoir afficher des messages de notification temporaires à
l'utilisateur après une action. Par exemple, après la soumission réussie d'un formulaire, on souhaite informer
l'utilisateur que "L'opération a été réalisée avec succès".

Le pattern **Post/Redirect/Get (PRG)** est une bonne pratique courante pour éviter les problèmes de double soumission de
formulaire. Après une requête POST (par exemple, la sauvegarde de données), l'application redirige l'utilisateur vers
une page via une requête GET (par exemple, la page de liste des éléments).

Cependant, cette redirection pose un défi : comment transmettre une information (comme un message de succès ou d'erreur)
de la requête POST initiale à la requête GET suivante, sans que cette information ne persiste indéfiniment ?

C'est là qu'interviennent les **messages flash** (ou "flash attributes"). Ce sont des attributs stockés temporairement (
généralement dans la session HTTP) avant la redirection et rendus disponibles *uniquement* pour la requête
*immédiatement* suivante (celle après la redirection). Une fois lus, ils sont automatiquement supprimés.

Ce mécanisme permet d'afficher des notifications contextuelles après une action, de manière propre et découplée.

## Fonctionnement Technique (Simplifié)

1. **Requête POST :** Le contrôleur traite la requête (ex: sauvegarde en base).
2. **Ajout du Message Flash :** Avant de déclencher la redirection, le contrôleur ajoute un ou plusieurs attributs "
   flash" via un mécanisme fourni par Spring. Ces attributs sont temporairement stockés, souvent dans la session HTTP
   associée à l'utilisateur.
3. **Redirection (Réponse 302) :** Le serveur renvoie une réponse de redirection au navigateur.
4. **Requête GET :** Le navigateur effectue une nouvelle requête GET vers l'URL de redirection.
5. **Récupération et Exposition :** Spring intercepte cette requête, récupère les attributs flash depuis le stockage
   temporaire (session), les ajoute au `Model` de la requête courante, puis les *supprime* du stockage temporaire.
6. **Rendu de la Vue :** La vue (ex: page Thymeleaf) peut accéder à ces attributs flash comme n'importe quel autre
   attribut du `Model` pour les afficher.
7. **Requêtes Suivantes :** Lors des requêtes ultérieures, ces attributs flash ne seront plus présents dans le `Model`
   ni dans la session.

## Implémentation dans Spring Boot

Spring MVC facilite grandement l'utilisation des messages flash grâce à l'interface `RedirectAttributes`.

### Injection de `RedirectAttributes`

Pour ajouter des messages flash dans une méthode de contrôleur qui effectue une redirection, il suffit d'ajouter un
paramètre de type `RedirectAttributes` à la signature de la méthode. Spring injectera automatiquement une instance
appropriée.

```java
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
// Importation de RedirectAttributes
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.springframework.web.servlet.view.RedirectView;

@Controller
@RequestMapping("/products")
public class ProductController {

    @PostMapping("/save")
    public RedirectView saveProduct(
            // Le nom de la variable est `productData` pour l'exemple,
            // mais il contiendrait les données du produit venant du formulaire.
            /* @ModelAttribute Product productData, */
            // Injection de RedirectAttributes
            RedirectAttributes redirectAttributes) {

        // Simulation d'une logique métier (sauvegarde, etc.)
        boolean success = performSaveOperation(/* productData */);

        if (success) {
            // Ajout d'un message flash en cas de succès
            String successMessage = "Produit ajouté avec succès !";
            // La clé "successMessage" sera utilisée dans la vue Thymeleaf.
            redirectAttributes.addFlashAttribute("successMessage", successMessage);
        } else {
            // Ajout d'un message flash en cas d'échec
            String errorMessage = "Erreur lors de l'ajout du produit.";
            redirectAttributes.addFlashAttribute("errorMessage", errorMessage);
        }

        // Redirection vers la page de liste des produits
        // Note : On peut aussi retourner "redirect:/products/list"
        return new RedirectView("/products/list");
    }

    // Méthode pour afficher la liste (cible de la redirection)
    // @GetMapping("/list") ... 

    // Méthode interne simulant l'opération
    private boolean performSaveOperation() {
        // Logique métier ici...
        // Retourne true pour simuler un succès dans cet exemple
        return true;
    }
}
```

**Explication :**

* `RedirectAttributes` est injecté dans la méthode `saveProduct`.
* `redirectAttributes.addFlashAttribute("key", value)` est la méthode clé. Elle stocke l'objet `value` sous le nom `key`
  de manière temporaire pour la prochaine requête après redirection.
* La méthode retourne soit un `RedirectView`, soit une chaîne de caractères préfixée par `"redirect:"` (ex:
  `"redirect:/products/list"`), ce qui déclenche la redirection.

### Types de Messages

Il est courant de vouloir afficher différents types de messages (succès, erreur, information, avertissement) avec des
styles visuels distincts. La manière la plus simple est d'utiliser des clés différentes pour chaque type :

* `successMessage`
* `errorMessage`
* `infoMessage`
* `warningMessage`

On ajoutera l'attribut flash correspondant à la situation.

## Consommation dans Thymeleaf

Les attributs flash ajoutés via `RedirectAttributes` sont automatiquement disponibles dans le `Model` de la requête GET
qui suit la redirection. On peut donc y accéder dans Thymeleaf comme n'importe quelle variable du modèle.

Il est crucial de vérifier l'existence de l'attribut avant de tenter de l'afficher, car il ne sera présent que lors de
cette *unique* requête post-redirection.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Liste des Produits</title>
    <!-- Inclure ici vos CSS pour styliser les messages -->
    <style>
        .alert {
            padding: 15px;
            margin-bottom: 20px;
            border: 1px solid transparent;
            border-radius: 4px;
        }

        .alert-success {
            color: #155724;
            background-color: #d4edda;
            border-color: #c3e6cb;
        }

        .alert-danger {
            color: #721c24;
            background-color: #f8d7da;
            border-color: #f5c6cb;
        }

        .alert-info {
            color: #0c5460;
            background-color: #d1ecf1;
            border-color: #bee5eb;
        }

        .alert-warning {
            color: #856404;
            background-color: #fff3cd;
            border-color: #ffeeba;
        }
    </style>
</head>
<body>

<h1>Liste des Produits</h1>

<!-- Affichage conditionnel du message de succès -->
<div th:if="${successMessage}" class="alert alert-success" role="alert">
    <p th:text="${successMessage}"></p>
</div>

<!-- Affichage conditionnel du message d'erreur -->
<div th:if="${errorMessage}" class="alert alert-danger" role="alert">
    <p th:text="${errorMessage}"></p>
</div>

<!-- Affichage conditionnel du message d'information -->
<div th:if="${infoMessage}" class="alert alert-info" role="alert">
    <p th:text="${infoMessage}"></p>
</div>

<!-- Affichage conditionnel du message d'avertissement -->
<div th:if="${warningMessage}" class="alert alert-warning" role="alert">
    <p th:text="${warningMessage}"></p>
</div>

<!-- Reste de la page (tableau des produits, etc.) -->
<table>
    <!-- ... contenu de la liste ... -->
</table>

</body>
</html>
```

**Explication :**

* `th:if="${successMessage}"` : Cet attribut Thymeleaf vérifie si la variable `successMessage` existe et n'est pas
  nulle (ou fausse, ou une collection/chaîne vide selon le contexte de l'expression). Le `div` ne sera rendu que si la
  condition est vraie.
* `th:text="${successMessage}"` : Affiche la valeur du message flash.
* Des classes CSS (`alert`, `alert-success`, `alert-danger`, etc. - inspirées de Bootstrap) sont utilisées pour le
  style. Il faut définir ces classes dans une feuille de style CSS.

## Factorisation et Bonnes Pratiques

Répéter les appels `redirectAttributes.addFlashAttribute(...)` dans de nombreuses méthodes de contrôleur peut devenir
fastidieux et violer le principe DRY (Don't Repeat Yourself). Plusieurs approches existent pour factoriser ce code.

### Méthode Utilitaire dans le Contrôleur

Une approche simple consiste à créer des méthodes privées au sein du contrôleur (ou d'une classe de base si plusieurs
contrôleurs partagent cette logique).

```java
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.springframework.web.servlet.view.RedirectView;

@Controller
@RequestMapping("/products")
public class ProductController {

    @PostMapping("/save")
    public RedirectView saveProduct(
            RedirectAttributes redirectAttributes) {

        boolean success = performSaveOperation();

        if (success) {
            // Appel de la méthode utilitaire pour le succès
            addFlashSuccessMessage(redirectAttributes, "Produit ajouté avec succès !");
        } else {
            // Appel de la méthode utilitaire pour l'erreur
            addFlashErrorMessage(redirectAttributes, "Erreur lors de l'ajout du produit.");
        }

        return new RedirectView("/products/list");
    }

    // ... (autres méthodes comme @GetMapping("/list"))

    // --- Méthodes utilitaires pour les messages flash ---

    private void addFlashSuccessMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("successMessage", message);
    }

    private void addFlashErrorMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("errorMessage", message);
    }

    private void addFlashInfoMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("infoMessage", message);
    }

    private void addFlashWarningMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("warningMessage", message);
    }

    // Méthode interne simulant l'opération
    private boolean performSaveOperation() {
        return true;
    }
}
```

### Composant Utilitaire (Service)

Pour une réutilisation plus large à travers l'application, on peut créer un composant Spring dédié.

```java
package fr.formation.spring.util;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Component // Déclare ce bean comme un composant Spring
public class FlashMessageHelper {

    public void addSuccessMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("successMessage", message);
    }

    public void addErrorMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("errorMessage", message);
    }

    public void addInfoMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("infoMessage", message);
    }

    public void addWarningMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("warningMessage", message);
    }
}
```

Et l'injecter dans les contrôleurs :

```java
package fr.formation.spring.controller;

import fr.formation.spring.util.FlashMessageHelper; // Importation
import org.springframework.beans.factory.annotation.Autowired; // Importation
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.springframework.web.servlet.view.RedirectView;

@Controller
@RequestMapping("/products")
public class ProductController {

    // Injection du helper par Autowired
    @Autowired
    private FlashMessageHelper flashHelper;

    @PostMapping("/save")
    public RedirectView saveProduct(
            RedirectAttributes redirectAttributes) {

        boolean success = performSaveOperation();

        if (success) {
            // Utilisation du helper injecté
            flashHelper.addSuccessMessage(redirectAttributes, "Produit ajouté avec succès !");
        } else {
            flashHelper.addErrorMessage(redirectAttributes, "Erreur lors de l'ajout du produit.");
        }

        return new RedirectView("/products/list");
    }

    // ... autres méthodes

    private boolean performSaveOperation() {
        return true;
    }
}
```

Cette seconde approche est généralement préférable car elle favorise la séparation des préoccupations et la
réutilisabilité.

### Bonnes Pratiques et Conseils

* **Consistance des Clés :** Utiliser des noms de clés cohérents dans toute l'application (`successMessage`,
  `errorMessage`, etc.).
* **Messages Clairs :** Rédiger des messages concis et compréhensibles pour l'utilisateur final. Penser à
  l'internationalisation si nécessaire (en utilisant les `MessageSource` de Spring).
* **Styling Centralisé :** Définir les styles CSS pour les messages dans un fichier CSS global pour une apparence
  uniforme.
* **Ne pas Abuser :** Les messages flash sont pour des notifications *temporaires* liées à une action venant d'être
  effectuée via redirection. Ne pas les utiliser pour des informations permanentes ou pour passer des données métier
  complexes.
* **Accessibilité :** Penser à rendre les messages accessibles (par exemple, en utilisant les attributs ARIA appropriés
  comme `role="alert"` dans le HTML).
* **Alternative :** Pour les erreurs de validation de formulaire *sans* redirection immédiate (rester sur la même page),
  le mécanisme de `BindingResult` de Spring est souvent plus adapté que les messages flash.

## Cas d'Utilisation Courants

* Confirmation de création, modification ou suppression d'une entité (ex: "Utilisateur créé avec succès").
* Notification d'échec d'une opération (ex: "Impossible de supprimer le produit car il est lié à des commandes").
* Message informatif après une action (ex: "Votre mot de passe a été réinitialisé. Veuillez vérifier vos emails.").
* Confirmation d'envoi de formulaire (ex: "Votre message a bien été envoyé.").

## Exercices Pratiques

### Exercice 1 : Message de Succès Simple

**Énoncé :**

1. Créez un contrôleur `TaskController` dans le package `fr.formation.spring.controller`.
2. Ajoutez une méthode GET pour afficher un formulaire simple à l'URL `/tasks/new` (vue `task-form.html`). Le formulaire
   aura un seul champ texte `description` et un bouton "Ajouter".
3. Ajoutez une méthode POST pour traiter la soumission du formulaire à l'URL `/tasks/save`.
4. Dans la méthode POST, simulez une sauvegarde réussie. Ajoutez un message flash de succès "Tâche ajoutée avec
   succès !" en utilisant `RedirectAttributes`.
5. Redirigez vers une page de liste (fictive pour l'instant) à l'URL `/tasks/list`.
6. Créez une méthode GET pour l'URL `/tasks/list` qui retourne une vue `task-list.html`.
7. Dans `task-list.html`, affichez le message flash de succès s'il est présent, en utilisant Thymeleaf et une classe CSS
   `alert-success`.

#### **Correction Exercice 1 :** {collapsible="true"}

**`TaskController.java`**

```java
package fr.formation.spring.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.springframework.web.servlet.view.RedirectView;

@Controller
@RequestMapping("/tasks")
public class TaskController {

    // Affiche le formulaire
    @GetMapping("/new")
    public String showTaskForm() {
        return "task-form"; // Nom de la vue Thymeleaf
    }

    // Traite la soumission du formulaire
    @PostMapping("/save")
    public RedirectView saveTask(
            // Récupère la donnée du formulaire (simplifié ici)
            @RequestParam String description,
            RedirectAttributes redirectAttributes) {

        System.out.println("Sauvegarde de la tâche : " + description);
        // Simulation d'un succès systématique pour cet exercice
        boolean success = true;

        if (success) {
            // Message de succès
            String successMsg = "Tâche '" + description + "' ajoutée avec succès !";
            redirectAttributes.addFlashAttribute("successMessage", successMsg);
        }
        // Pas de cas d'erreur dans cet exercice simple

        // Redirection vers la liste
        return new RedirectView("/tasks/list");
    }

    // Affiche la page de liste (cible de la redirection)
    @GetMapping("/list")
    public String showTaskList(Model model) {
        // Ici, on pourrait charger la vraie liste des tâches
        // model.addAttribute("tasks", taskService.findAll());
        return "task-list"; // Nom de la vue Thymeleaf
    }
}
```

**`src/main/resources/templates/task-form.html`**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Nouvelle Tâche</title>
</head>
<body>
<h1>Ajouter une nouvelle tâche</h1>
<form th:action="@{/tasks/save}" method="post">
    <div>
        <label for="desc">Description:</label>
        <input type="text" id="desc" name="description" required/>
    </div>
    <div>
        <button type="submit">Ajouter</button>
    </div>
</form>
</body>
</html>
```

**`src/main/resources/templates/task-list.html`**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Liste des Tâches</title>
    <style>
        .alert {
            padding: 10px;
            margin-bottom: 15px;
            border: 1px solid transparent;
            border-radius: 4px;
        }

        .alert-success {
            color: #155724;
            background-color: #d4edda;
            border-color: #c3e6cb;
        }
    </style>
</head>
<body>
<h1>Liste des Tâches</h1>

<!-- Affichage du message flash de succès -->
<div th:if="${successMessage}" class="alert alert-success" role="alert">
    <p th:text="${successMessage}"></p>
</div>

<!-- Ici viendrait l'affichage de la liste des tâches -->
<p>(Contenu de la liste des tâches...)</p>

</body>
</html>
```

### Exercice 2 : Gestion des Erreurs

**Énoncé :**

1. Modifiez la méthode `saveTask` du `TaskController` de l'exercice précédent.
2. Si le champ `description` soumis est vide ou contient uniquement des espaces, considérez cela comme une erreur.
3. Dans ce cas d'erreur, ajoutez un message flash d'erreur "La description ne peut pas être vide." avec la clé
   `errorMessage`. Ne sauvegardez pas la tâche et redirigez toujours vers `/tasks/list`.
4. Si la description est valide, ajoutez le message de succès comme précédemment.
5. Modifiez la vue `task-list.html` pour afficher *soit* le message de succès (avec la classe `alert-success`), *soit*
   le message d'erreur (avec une classe `alert-danger`), mais jamais les deux en même temps.

#### **Correction Exercice 2 :** {collapsible="true"}

**`TaskController.java` (Méthode `saveTask` modifiée)**

```java
    // ... (autres méthodes et imports identiques)

// Traite la soumission du formulaire
@PostMapping("/save")
public RedirectView saveTask(
        @RequestParam String description,
        RedirectAttributes redirectAttributes) {

    // Vérification simple de la description
    if (description == null || description.trim().isEmpty()) {
        // Cas d'erreur : description vide
        String errorMsg = "La description de la tâche ne peut pas être vide.";
        redirectAttributes.addFlashAttribute("errorMessage", errorMsg);
    } else {
        // Cas de succès : description valide (simulation)
        System.out.println("Sauvegarde de la tâche : " + description);
        String successMsg = "Tâche '" + description + "' ajoutée avec succès !";
        redirectAttributes.addFlashAttribute("successMessage", successMsg);
    }

    // Redirection vers la liste dans tous les cas (après ajout du message)
    return new RedirectView("/tasks/list");
}

// ... (méthode showTaskList identique)
```

**`src/main/resources/templates/task-list.html` (Modifiée)**

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Liste des Tâches</title>
    <style>
        .alert {
            padding: 10px;
            margin-bottom: 15px;
            border: 1px solid transparent;
            border-radius: 4px;
        }

        .alert-success {
            color: #155724;
            background-color: #d4edda;
            border-color: #c3e6cb;
        }

        /* Ajout du style pour les erreurs */
        .alert-danger {
            color: #721c24;
            background-color: #f8d7da;
            border-color: #f5c6cb;
        }
    </style>
</head>
<body>
<h1>Liste des Tâches</h1>

<!-- Affichage conditionnel: priorité à l'erreur s'il existe -->
<div th:if="${errorMessage}" class="alert alert-danger" role="alert">
    <p th:text="${errorMessage}"></p>
</div>

<!-- Affichage du succès SEULEMENT s'il n'y a pas de message d'erreur -->
<div th:if="${successMessage and not errorMessage}" class="alert alert-success" role="alert">
    <p th:text="${successMessage}"></p>
</div>

<!-- Ici viendrait l'affichage de la liste des tâches -->
<p>(Contenu de la liste des tâches...)</p>

</body>
</html>
```

*(Note: L'alternative `th:unless="${errorMessage}"` pourrait aussi être utilisée dans le second `div`)*

#### Exercice 3 : Factorisation avec un Helper

**Énoncé :**

1. Créez la classe `FlashMessageHelper` dans `fr.formation.spring.util` comme montré dans la section 5.2.
2. Injectez ce `FlashMessageHelper` dans le `TaskController`.
3. Modifiez la méthode `saveTask` pour utiliser les méthodes `addSuccessMessage` et `addErrorMessage` du helper au lieu
   d'appeler directement `redirectAttributes.addFlashAttribute`.

#### **Correction Exercice 3 :** {collapsible="true"}

**`FlashMessageHelper.java`**

```java
package fr.formation.spring.util;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Component
public class FlashMessageHelper {

    public void addSuccessMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("successMessage", message);
    }

    public void addErrorMessage(RedirectAttributes attributes, String message) {
        attributes.addFlashAttribute("errorMessage", message);
    }

    // On pourrait ajouter ici info et warning si besoin
    // public void addInfoMessage(...)
    // public void addWarningMessage(...)
}
```

**`TaskController.java` (Modifié pour utiliser le Helper)**

```java
package fr.formation.spring.controller;

import fr.formation.spring.util.FlashMessageHelper; // Import du Helper
import org.springframework.beans.factory.annotation.Autowired; // Import Autowired
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import org.springframework.web.servlet.view.RedirectView;

@Controller
@RequestMapping("/tasks")
public class TaskController {

    // Injection du Helper
    @Autowired
    private FlashMessageHelper flashHelper;

    @GetMapping("/new")
    public String showTaskForm() {
        return "task-form";
    }

    @PostMapping("/save")
    public RedirectView saveTask(
            @RequestParam String description,
            RedirectAttributes redirectAttributes) {

        if (description == null || description.trim().isEmpty()) {
            String errorMsg = "La description de la tâche ne peut pas être vide.";
            // Utilisation du helper pour le message d'erreur
            flashHelper.addErrorMessage(redirectAttributes, errorMsg);
        } else {
            System.out.println("Sauvegarde de la tâche : " + description);
            String successMsg = "Tâche '" + description + "' ajoutée avec succès !";
            // Utilisation du helper pour le message de succès
            flashHelper.addSuccessMessage(redirectAttributes, successMsg);
        }

        return new RedirectView("/tasks/list");
    }

    @GetMapping("/list")
    public String showTaskList(Model model) {
        return "task-list";
    }
}
```

*(Les vues `task-form.html` et `task-list.html` restent identiques à celles de l'exercice 2)*

## Conclusion

Les messages flash sont un outil essentiel dans les applications web Spring MVC suivant le pattern PRG. Ils permettent
de fournir un retour utilisateur clair et concis après une action, en survivant de manière élégante à une redirection
HTTP. Grâce à `RedirectAttributes` et l'intégration avec Thymeleaf, leur mise en œuvre est simple et directe. La
factorisation via des méthodes ou des composants utilitaires aide à maintenir un code propre et réutilisable. En
maîtrisant les messages flash, les développeurs peuvent améliorer significativement l'expérience utilisateur de leurs
applications web.

---