# ControllerAdvice

Ok, voici une proposition de support de cours sur l'annotation `@ControllerAdvice` dans Spring Boot pour un contexte web MVC, en excluant la gestion des exceptions.

---

## Support de Cours : L'annotation @ControllerAdvice dans Spring Boot (Contexte Web MVC)

**Public Cible :** Développeurs Spring Boot ayant des bases en Spring MVC.
**Prérequis :** Connaissance de Spring Core, Spring MVC (`@Controller`, `@RequestMapping`, Modèle, Vues type Thymeleaf/JSP), gestion des formulaires.
**Objectif :** Comprendre et utiliser `@ControllerAdvice` pour centraliser des logiques transversales appliquées aux contrôleurs web, hors gestion des exceptions.

---

### 1. Introduction à @ControllerAdvice

Dans une application Spring Boot web basée sur Spring MVC, il est fréquent d'avoir besoin d'exécuter certaines logiques avant ou pendant le traitement des requêtes par plusieurs contrôleurs. Par exemple :

*   Ajouter des données communes au modèle pour qu'elles soient disponibles dans toutes les vues (ou un sous-ensemble de vues).
*   Configurer la manière dont les données des requêtes sont liées aux objets Java (data binding) de manière globale.

L'annotation `@ControllerAdvice` est une spécialisation de l'annotation `@Component`. Elle permet de déclarer une classe qui fournira des traitements transversaux applicables à un ensemble de contrôleurs (`@Controller`). Ces traitements sont "greffés" au cycle de vie de la requête gérée par Spring MVC.

**Points Clés :**

*   **Centralisation :** Évite la duplication de code dans plusieurs contrôleurs.
*   **Transversalité :** Applique des logiques communes à plusieurs points de l'application (contrôleurs ciblés).
*   **Spécialisation :** Destinée aux problématiques liées aux contrôleurs web.

Ce chapitre se concentre sur l'utilisation de `@ControllerAdvice` pour la gestion des **attributs de modèle globaux** et la **personnalisation du data binding**, en dehors de la gestion globale des exceptions (qui sera traitée séparément).

---

### 2. Cas d'Utilisation 1 : Ajout d'Attributs Globaux au Modèle avec @ModelAttribute

#### 2.1. Problématique

Souvent, plusieurs vues d'une application web nécessitent l'affichage d'informations communes : nom de l'utilisateur connecté, version de l'application, éléments de menu dynamiques, etc. Sans mécanisme de centralisation, il faudrait ajouter manuellement ces informations au `Model` dans chaque méthode de chaque `@Controller` concerné. C'est répétitif et source d'erreurs.

#### 2.2. Solution : `@ModelAttribute` dans un `@ControllerAdvice`

Une méthode annotée avec `@ModelAttribute` au sein d'une classe `@ControllerAdvice` sera exécutée *avant* les méthodes des contrôleurs ciblés (ceux définis par le scope du `@ControllerAdvice`, voir section 4). La valeur retournée par cette méthode est automatiquement ajoutée au `Model` de Spring MVC, la rendant accessible dans la vue.

#### 2.3. Exemple de Code

Supposons que nous voulions afficher la version de l'application et l'année courante sur toutes les pages.

**Étape 1 : Créer la classe `@ControllerAdvice`**

```java
package fr.formation.spring.advice;

import java.time.Year;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;

// Déclare cette classe comme un conseil pour les contrôleurs
// Par défaut, s'applique à tous les @Controller
@ControllerAdvice(basePackages = "fr.formation.spring.controllers") 
// Optionnel: Limite aux contrôleurs du package spécifié (voir section 4)
public class GlobalModelAttributesAdvice {

    // Injecte la version de l'application depuis application.properties
    @Value("${application.version:1.0.0}") // Fournit une valeur par défaut
    private String applicationVersion; 

    /**
     * Ajoute l'attribut 'appVersion' au modèle pour tous les contrôleurs ciblés.
     * Cette méthode est appelée avant l'exécution des méthodes @RequestMapping.
     * @return La version de l'application.
     */
    @ModelAttribute("appVersion") // Nom de l'attribut dans le modèle
    public String addAppVersionToModel() {
        return this.applicationVersion;
    }

    /**
     * Ajoute l'attribut 'currentYear' au modèle.
     * @return L'année courante.
     */
    @ModelAttribute("currentYear") // Nom de l'attribut dans le modèle
    public int addCurrentYearToModel() {
        return Year.now().getValue();
    }

    // On pourrait ajouter ici d'autres méthodes @ModelAttribute
    // pour des données globales (ex: menu de navigation, utilisateur connecté...)
}
```

**Étape 2 : Configurer la propriété dans `application.properties`**

```properties
# src/main/resources/application.properties
application.version=1.1.0-SNAPSHOT
```

**Étape 3 : Utiliser les attributs dans un contrôleur et une vue**

```java
package fr.formation.spring.controllers;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @GetMapping("/")
    public String home(Model model) {
        // Pas besoin d'ajouter 'appVersion' ou 'currentYear' ici !
        // Ils sont ajoutés automatiquement par GlobalModelAttributesAdvice.
        model.addAttribute("pageTitle", "Accueil");
        return "home"; // Nom de la vue (ex: home.html avec Thymeleaf)
    }
}
```

```html
<!-- src/main/resources/templates/home.html (Exemple avec Thymeleaf) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <!-- Utilisation de l'attribut 'pageTitle' ajouté dans le contrôleur -->
    <title th:text="${pageTitle}">Titre par défaut</title> 
</head>
<body>
    <h1>Bienvenue sur notre Application</h1>
    
    <footer>
        <!-- Utilisation des attributs ajoutés par @ControllerAdvice -->
        <p>
            Version: <span th:text="${appVersion}">?.?.?</span> - 
            Année: <span th:text="${currentYear}">????</span>
        </p>
    </footer>
</body>
</html>
```

#### 2.4. Conseils et Bonnes Pratiques

*   **Pertinence :** N'ajoutez que des attributs réellement globaux ou communs à un large ensemble de vues.
*   **Performance :** Soyez conscient que ces méthodes sont appelées pour chaque requête traitée par les contrôleurs ciblés. Évitez les calculs lourds ou les appels externes bloquants dans ces méthodes. Si les données changent peu, envisagez des mécanismes de mise en cache.
*   **Nommage :** Choisissez des noms d'attributs clairs et évitez les collisions avec les attributs ajoutés spécifiquement par les contrôleurs.

#### 2.5. Exercice Pratique 1

**Énoncé :**
Créez une nouvelle classe `UserDataAdvice` annotée avec `@ControllerAdvice`. Cette classe devra ajouter au modèle un attribut nommé `globalMessage` contenant la chaîne de caractères "Bienvenue sur notre site !". Vérifiez ensuite dans une page HTML (utilisant Thymeleaf ou JSP) que ce message est bien accessible. Limitez ce conseil aux contrôleurs du package `fr.formation.spring.controllers.user`.

**Correction :**

**Étape 1 : Créer le package et le contrôleur utilisateur (si inexistant)**

```java
// Créer le package fr.formation.spring.controllers.user

package fr.formation.spring.controllers.user;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
// Important: Ce contrôleur est dans le package ciblé
public class UserProfileController { 

    @GetMapping("/profile")
    public String userProfile() {
        // Le message 'globalMessage' sera ajouté par UserDataAdvice
        return "user/profile"; // Vue: templates/user/profile.html
    }
}
```

**Étape 2 : Créer la classe `UserDataAdvice`**

```java
package fr.formation.spring.advice;

import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;

// Cible uniquement les contrôleurs dans ce package spécifique
@ControllerAdvice(basePackages = "fr.formation.spring.controllers.user") 
public class UserDataAdvice {

    /**
     * Ajoute un message de bienvenue global pour les sections utilisateur.
     * @return Le message de bienvenue.
     */
    @ModelAttribute("globalMessage") // Nom de l'attribut dans le modèle
    public String addGlobalMessage() {
        return "Bienvenue sur notre site !";
    }
}
```

**Étape 3 : Créer la vue pour vérifier**

```html
<!-- src/main/resources/templates/user/profile.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Profil Utilisateur</title>
</head>
<body>
    <!-- Affichage du message ajouté par UserDataAdvice -->
    <h2 th:text="${globalMessage}">Message global ici</h2>

    <p>Contenu spécifique au profil utilisateur...</p>
</body>
</html>
```
En accédant à l'URL `/profile`, le message "Bienvenue sur notre site !" devrait s'afficher en haut de la page, prouvant que l'`UserDataAdvice` a bien fonctionné et a été correctement scopé. Un contrôleur en dehors du package `fr.formation.spring.controllers.user` n'aurait pas cet attribut `globalMessage` ajouté automatiquement.

---

### 3. Cas d'Utilisation 2 : Personnalisation du Data Binding avec @InitBinder

#### 3.1. Problématique

Lors de la soumission de formulaires HTML, Spring MVC tente de convertir les chaînes de caractères reçues en types de données Java appropriés pour les objets de commandes (form backing objects). Parfois, le format par défaut ne convient pas (ex: format de date spécifique `dd/MM/yyyy`), ou nous voulons appliquer une transformation systématique (ex: supprimer les espaces avant/après les chaînes de caractères). Devoir configurer cela dans chaque contrôleur serait redondant.

#### 3.2. Solution : `@InitBinder` dans un `@ControllerAdvice`

Une méthode annotée avec `@InitBinder` au sein d'une classe `@ControllerAdvice` permet de personnaliser le processus de data binding pour les contrôleurs ciblés. Cette méthode prend généralement un `WebDataBinder` en argument, sur lequel on peut enregistrer des éditeurs de propriétés personnalisés (`PropertyEditor`) ou configurer d'autres aspects du binding.

#### 3.3. Exemple de Code

Supposons que nous ayons des formulaires où les dates doivent être saisies au format `dd/MM/yyyy`.

**Étape 1 : Créer la classe `@ControllerAdvice` avec `@InitBinder`**

```java
package fr.formation.spring.advice;

import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.beans.propertyeditors.CustomDateEditor;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.InitBinder;

// Applique ce conseil à tous les contrôleurs (par défaut)
@ControllerAdvice 
public class GlobalBindingAdvice {

    /**
     * Configure le data binder pour tous les contrôleurs ciblés.
     * Est appelé avant que le data binding ne soit effectué pour une requête.
     * @param binder Le WebDataBinder pour la requête courante.
     */
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // Définit le format de date attendu pour tous les champs de type Date
        SimpleDateFormat dateFormat = new SimpleDateFormat("dd/MM/yyyy");
        dateFormat.setLenient(false); // Ne pas être tolérant aux dates invalides

        // Enregistre un éditeur personnalisé pour le type java.util.Date
        // Le deuxième argument 'true' signifie qu'une chaîne vide sera 
        // convertie en 'null'
        binder.registerCustomEditor(Date.class, 
                new CustomDateEditor(dateFormat, true));

        // On pourrait ajouter ici d'autres configurations globales du binder
        // binder.setDisallowedFields("id"); // Interdire le binding du champ 'id'
    }
}
```

**Étape 2 : Utiliser dans un contrôleur avec un formulaire**

```java
package fr.formation.spring.controllers;

import fr.formation.spring.model.Event; // Supposons une classe Event
import javax.validation.Valid; // Pour la validation éventuelle
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;

@Controller
public class EventController {

    @GetMapping("/event/new")
    public String showEventForm(Model model) {
        model.addAttribute("event", new Event()); // Objet pour le formulaire
        return "event-form"; // Nom de la vue du formulaire
    }

    @PostMapping("/event/save")
    public String saveEvent(
            @Valid @ModelAttribute("event") Event event, 
            BindingResult bindingResult, 
            Model model) {
            
        // Grâce à GlobalBindingAdvice, si l'utilisateur saisit "15/10/2024"
        // dans le champ correspondant à event.eventDate (de type Date),
        // Spring saura le convertir correctement.
            
        if (bindingResult.hasErrors()) {
            // S'il y a des erreurs (validation ou binding), retourne au formulaire
            return "event-form";
        }

        // Logique de sauvegarde de l'événement...
        System.out.println("Date de l'événement reçue : " + event.getEventDate());

        return "redirect:/"; // Redirige après succès
    }
}
```

```java
// fr/formation/spring/model/Event.java
package fr.formation.spring.model;

import java.util.Date;
import org.springframework.format.annotation.DateTimeFormat; // Utile pour l'affichage

public class Event {
    private String name;
    
    // Pas besoin d'annotation @DateTimeFormat ici pour la *réception* 
    // si @InitBinder est utilisé globalement.
    // @DateTimeFormat est plus utile pour le formatage à l'*affichage*.
    private Date eventDate; 

    // Getters and Setters...
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Date getEventDate() { return eventDate; }
    public void setEventDate(Date eventDate) { this.eventDate = eventDate; }
}
```

```html
<!-- src/main/resources/templates/event-form.html (Exemple avec Thymeleaf) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Nouvel Événement</title>
</head>
<body>
    <h1>Créer un événement</h1>
    
    <!-- Le tag 'th:object' lie le formulaire à l'objet 'event' du modèle -->
    <form action="#" th:action="@{/event/save}" th:object="${event}" method="post">
        <div>
            <label for="name">Nom:</label>
            <!-- Le tag 'th:field' génère les attributs name, id, value -->
            <input type="text" id="name" th:field="*{name}" />
            <!-- Affiche les erreurs de validation pour ce champ -->
            <span th:if="${#fields.hasErrors('name')}" th:errors="*{name}"></span> 
        </div>
        <div>
            <label for="eventDate">Date (jj/mm/aaaa):</label>
            <!-- Utilisation de th:field pour lier au champ Date -->
            <!-- Pour l'affichage initial, Spring utilisera le format par défaut ou
                 celui spécifié par @DateTimeFormat(pattern="...") sur le champ.
                 Pour la soumission, c'est @InitBinder qui prime. -->
            <input type="text" id="eventDate" 
                   th:field="*{eventDate}" placeholder="dd/MM/yyyy" />
            <span th:if="${#fields.hasErrors('eventDate')}" 
                  th:errors="*{eventDate}">Erreur Date</span>
        </div>
        <div>
            <button type="submit">Enregistrer</button>
        </div>
    </form>
</body>
</html>
```

#### 3.4. Conseils et Bonnes Pratiques

*   **Spécificité :** `@InitBinder` dans un `@ControllerAdvice` est idéal pour les règles de binding *vraiment* globales. Si une règle ne concerne qu'un seul contrôleur ou un seul type d'objet, il est préférable de placer la méthode `@InitBinder` directement dans le `@Controller` concerné.
*   **Éditeurs Personnalisés :** Pour des logiques de conversion complexes, vous pouvez créer vos propres classes implémentant `java.beans.PropertyEditorSupport`.
*   **Validation vs Binding :** `@InitBinder` intervient *avant* la validation (JSR 303/Bean Validation). Il prépare les données pour qu'elles soient dans le bon type avant que les validateurs ne s'exécutent.
*   **Attribut `value` de `@InitBinder`:** Vous pouvez spécifier les noms des attributs du modèle auxquels l'`@InitBinder` doit s'appliquer : `@InitBinder("event")` limiterait l'initialisation aux objets de commande nommés "event". C'est utile si différentes configurations de binder sont nécessaires pour différents objets au sein des mêmes contrôleurs.

#### 3.5. Exercice Pratique 2

**Énoncé :**
Créez (ou modifiez) un `GlobalBindingAdvice` pour qu'il enregistre un `StringTrimmerEditor`. Cet éditeur, fourni par Spring, permet de supprimer les espaces superflus au début et à la fin des chaînes de caractères soumises via les formulaires, et de convertir une chaîne composée uniquement d'espaces en `null`. Appliquez cet éditeur à tous les champs de type `String`. Testez avec un formulaire simple contenant un champ texte.

**Correction :**

**Étape 1 : Modifier/Créer `GlobalBindingAdvice`**

```java
package fr.formation.spring.advice;

import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.beans.propertyeditors.CustomDateEditor;
import org.springframework.beans.propertyeditors.StringTrimmerEditor; // Importer l'éditeur
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.InitBinder;

@ControllerAdvice
public class GlobalBindingAdvice {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        // Configuration précédente pour les dates (peut coexister)
        SimpleDateFormat dateFormat = new SimpleDateFormat("dd/MM/yyyy");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, 
                new CustomDateEditor(dateFormat, true));

        // **NOUVEAU : Enregistrer StringTrimmerEditor pour le type String**
        // Le constructeur prend un booléen: 
        // true = convertir les chaînes vides ("") ou pleines d'espaces en null
        // false = convertir uniquement les chaînes pleines d'espaces en null, 
        //         laisser les chaînes vides comme ""
        binder.registerCustomEditor(String.class, new StringTrimmerEditor(true)); 
    }
}
```

**Étape 2 : Créer un formulaire et un contrôleur de test**

```java
// fr/formation/spring/model/FormData.java
package fr.formation.spring.model;

public class FormData {
    private String comment;

    // Getters and Setters...
    public String getComment() { return comment; }
    public void setComment(String comment) { this.comment = comment; }
}
```

```java
// fr/formation/spring/controllers/FormTestController.java
package fr.formation.spring.controllers;

import fr.formation.spring.model.FormData;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;

@Controller
public class FormTestController {

    @GetMapping("/test-form")
    public String showTestForm(Model model) {
        model.addAttribute("formData", new FormData());
        return "test-form"; // Vue: templates/test-form.html
    }

    @PostMapping("/submit-form")
    public String submitTestForm(@ModelAttribute FormData formData, Model model) {
        // Grâce à StringTrimmerEditor(true) dans GlobalBindingAdvice:
        // Si l'utilisateur saisit "   Bonjour   ", formData.comment sera "Bonjour"
        // Si l'utilisateur saisit "      ", formData.comment sera null
        // Si l'utilisateur ne saisit rien "", formData.comment sera null

        System.out.println("Commentaire reçu : [" + formData.getComment() + "]");
        model.addAttribute("submittedComment", formData.getComment());
        return "form-result"; // Vue: templates/form-result.html
    }
}
```

**Étape 3 : Créer les vues**

```html
<!-- src/main/resources/templates/test-form.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Test Formulaire</title></head>
<body>
    <form action="#" th:action="@{/submit-form}" 
          th:object="${formData}" method="post">
        <label for="comment">Commentaire:</label>
        <input type="text" id="comment" th:field="*{comment}" />
        <button type="submit">Envoyer</button>
    </form>
</body>
</html>
```

```html
<!-- src/main/resources/templates/form-result.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head><title>Résultat</title></head>
<body>
    <h1>Commentaire Soumis</h1>
    <!-- Affiche 'null' si la chaîne était vide ou composée d'espaces -->
    <p>Valeur reçue après trim/conversion : 
       <span th:text="${submittedComment}">[valeur]</span>
    </p>
    <a th:href="@{/test-form}">Retour au formulaire</a>
</body>
</html>
```
En testant le formulaire `/test-form` avec différentes entrées (espaces avant/après, uniquement des espaces, vide), vous pourrez vérifier dans la console ou sur la page `form-result` que le `StringTrimmerEditor` configuré via `@ControllerAdvice` a bien fonctionné.

---

### 4. Cibler les Contrôleurs avec @ControllerAdvice

Par défaut, une classe `@ControllerAdvice` s'applique à **tous** les contrôleurs (`@Controller`) de l'application. Il est souvent nécessaire de restreindre son périmètre. L'annotation `@ControllerAdvice` fournit plusieurs attributs pour cela :

1.  **`basePackages`** (ou son alias `value`) :
    *   Type : `String[]`
    *   Usage : Spécifie les packages (et sous-packages) contenant les contrôleurs à cibler.
    *   Exemple : `@ControllerAdvice(basePackages = "fr.formation.spring.controllers.admin")`

2.  **`basePackageClasses`** :
    *   Type : `Class<?>[]`
    *   Usage : Spécifie des classes. Les packages de ces classes seront utilisés comme `basePackages`. Utile pour éviter les chaînes de caractères "en dur".
    *   Exemple : `@ControllerAdvice(basePackageClasses = AdminController.class)`

3.  **`assignableTypes`** :
    *   Type : `Class<?>[]`
    *   Usage : Cible les contrôleurs qui sont assignables (héritent ou implémentent) aux types spécifiés.
    *   Exemple : `@ControllerAdvice(assignableTypes = {ProductController.class, BaseUserController.class})`

4.  **`annotations`** :
    *   Type : `Class<? extends Annotation>[]`
    *   Usage : Cible les contrôleurs qui sont annotés avec les annotations spécifiées. Pratique pour créer des groupes logiques de contrôleurs.
    *   Exemple :
        ```java
        // Définir une annotation marqueur
        @Retention(RetentionPolicy.RUNTIME)
        @Target(ElementType.TYPE)
        public @interface SecuredZone {}

        // Appliquer l'annotation sur un contrôleur
        @Controller
        @SecuredZone
        public class AdminAreaController { /* ... */ }

        // Cibler les contrôleurs avec cette annotation
        @ControllerAdvice(annotations = SecuredZone.class)
        public class SecurityAdvice { /* ... @ModelAttribute, @InitBinder ... */ }
        ```

**Combinaison :** On peut combiner ces attributs. Un contrôleur est ciblé s'il correspond à *au moins une* des conditions spécifiées.

**Exemple Combiné :**

```java
@ControllerAdvice(
    basePackages = "fr.formation.spring.controllers.reporting", 
    annotations = InternalTool.class
)
public class ReportingAndInternalToolsAdvice {
    // Ce conseil s'appliquera aux contrôleurs:
    // 1. Situés dans le package 'fr.formation.spring.controllers.reporting' (et sous-packages)
    // OU
    // 2. Annotés avec @InternalTool (peu importe leur package)
}
```

---

### 5. Conseils et Bonnes Pratiques Générales pour @ControllerAdvice (hors exceptions)

*   **Principe de Responsabilité Unique :** Une classe `@ControllerAdvice` devrait idéalement se concentrer sur un aspect transversal spécifique (ex: gestion des dates, ajout d'infos utilisateur). Si vous avez plusieurs aspects très différents, envisagez plusieurs classes `@ControllerAdvice`, éventuellement avec des ciblages différents.
*   **Clarté du Ciblage :** Utilisez les attributs de ciblage (`basePackages`, `annotations`, etc.) pour limiter la portée de vos conseils. Un conseil global appliqué partout peut avoir des effets de bord inattendus. Le ciblage par annotation est souvent le plus flexible et le plus explicite.
*   **Éviter la Logique Métier :** Les classes `@ControllerAdvice` sont destinées à des préoccupations techniques transversales liées à MVC (modèle, binding). N'y placez pas de logique métier principale. Celle-ci appartient aux services.
*   **Documentation :** Commentez vos classes `@ControllerAdvice` pour expliquer clairement ce qu'elles font et quels contrôleurs elles ciblent.
*   **Tests :** Testez vos `@ControllerAdvice`, notamment les méthodes `@InitBinder` qui peuvent être sensibles. Les tests d'intégration avec MockMvc sont appropriés pour vérifier que les attributs sont bien ajoutés au modèle ou que le binding fonctionne comme prévu.
*   **Performance :** Soyez conscient de l'impact potentiel sur les performances, surtout pour les méthodes `@ModelAttribute` exécutées à chaque requête. Utilisez-les judicieusement.

---

### 6. Cas d'Utilisation Récapitulatifs (hors exceptions)

Voici quelques scénarios typiques où `@ControllerAdvice` (avec `@ModelAttribute` et `@InitBinder`) est particulièrement utile dans une application web Spring MVC :

*   **Ajouter des informations utilisateur :** Mettre le nom/prénom de l'utilisateur connecté dans le modèle pour l'afficher dans l'en-tête de toutes les pages.
*   **Ajouter des métadonnées de l'application :** Version, environnement (dev/prod), nom de l'application.
*   **Fournir des listes communes :** Remplir des listes déroulantes communes à plusieurs formulaires (ex: liste de pays, de catégories).
*   **Formatage de date global :** Assurer que toutes les dates saisies dans les formulaires respectent un format unique.
*   **Nettoyage des entrées String :** Appliquer un `trim()` systématique sur toutes les chaînes issues des formulaires.
*   **Configuration de binder spécifique :** Interdire le binding de certains champs sensibles (`id`, `password`) globalement ou pour un groupe de contrôleurs.

---

### 7. Conclusion

L'annotation `@ControllerAdvice` est un outil puissant dans l'écosystème Spring Boot MVC pour gérer les préoccupations transversales liées aux contrôleurs web. En utilisant judicieusement les méthodes `@ModelAttribute` et `@InitBinder` au sein de classes `@ControllerAdvice` bien ciblées, il est possible de réduire considérablement la duplication de code, d'améliorer la maintenabilité et de centraliser la configuration du comportement des contrôleurs pour des aspects comme l'enrichissement du modèle et la personnalisation du data binding. Maîtriser `@ControllerAdvice` est une étape importante pour écrire des applications web Spring Boot propres et efficaces. Le prochain chapitre abordera son utilisation pour la gestion globale des exceptions.

---