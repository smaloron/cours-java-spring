# Exercices-thymeleaf

## Exercice 1 : Afficher un message simple

*   **Objectif :** Afficher une simple chaîne de caractères passée par le contrôleur.
*   **Concepts Thymeleaf :** `th:text`, expression de variable `${...}`

*   **Contrôleur (`MessageController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;

    @Controller
    public class MessageController {

        @GetMapping("/message")
        public String showMessage(Model model) {
            String welcomeMessage = "Bienvenue sur notre site !";
            model.addAttribute("message", welcomeMessage);
            return "message-view"; // Nom du template HTML
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/message-view.html`) :**
    *   Crée un paragraphe `<p>` qui affiche la valeur de l'attribut `message` du modèle.

### Correction (`message-view.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Message Simple</title>
    </head>
    <body>
        <h1>Message d'accueil</h1>
        <p th:text="${message}">Message par défaut qui sera remplacé.</p>
    </body>
    </html>
```

---

## Exercice 2 : Afficher les propriétés d'un objet

*   **Objectif :** Afficher les différentes propriétés d'un objet Java.
*   **Concepts Thymeleaf :** Accès aux propriétés d'objet `${objet.propriete}`

*   **Classe Pojo (`User.java`) :**
    ```java
    // Créez cette classe simple
    public class User {
        private Long id;
        private String username;
        private String email;
        private boolean active;

        // Constructeur, Getters et Setters (générés ou écrits)
        public User(Long id, String username, String email, boolean active) {
            this.id = id;
            this.username = username;
            this.email = email;
            this.active = active;
        }

        // --- Getters ---
        public Long getId() { return id; }
        public String getUsername() { return username; }
        public String getEmail() { return email; }
        public boolean isActive() { return active; }
        // --- Setters (optionnels pour cet exo) ---
    }
    ```

*   **Contrôleur (`UserController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    // Assurez-vous d'importer ou de définir la classe User ci-dessus

    @Controller
    public class UserController {

        @GetMapping("/user-details")
        public String showUserDetails(Model model) {
            User user = new User(1L, "jdupont", "j.dupont@email.com", true);
            model.addAttribute("userDetails", user);
            return "user-details-view"; // Nom du template HTML
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/user-details-view.html`) :**
    *   Affiche l'ID, le nom d'utilisateur et l'email de l'utilisateur dans des paragraphes séparés.

### Correction (`user-details-view.html`) {collapsible="true"}
```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Détails Utilisateur</title>
    </head>
    <body>
        <h1>Détails de l'utilisateur</h1>
        <p>ID: <span th:text="${userDetails.id}">0</span></p>
        <p>Nom d'utilisateur: <span th:text="${userDetails.username}">username</span></p>
        <p>Email: <span th:text="${userDetails.email}">email@example.com</span></p>
        <!-- Bonus: Afficher le statut (actif/inactif) -->
        <p>Statut: <span th:text="${userDetails.active ? 'Actif' : 'Inactif'}">statut</span></p>
    </body>
    </html>
```

---

## Exercice 3 : Itérer sur une liste de chaînes

*   **Objectif :** Afficher les éléments d'une liste de chaînes de caractères.
*   **Concepts Thymeleaf :** Itération `th:each`

*   **Contrôleur (`ListController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import java.util.Arrays;
    import java.util.List;

    @Controller
    public class ListController {

        @GetMapping("/items")
        public String showItems(Model model) {
            List<String> items = Arrays.asList("Pomme", "Banane", "Orange", "Fraise");
            model.addAttribute("itemList", items);
            return "items-view"; // Nom du template HTML
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/items-view.html`) :**
    *   Crée une liste non ordonnée (`<ul>`) où chaque élément (`<li>`) affiche un item de `itemList`.

### Correction (`items-view.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Liste d'Items</title>
    </head>
    <body>
        <h1>Liste des Items</h1>
        <ul>
            <li th:each="item : ${itemList}" th:text="${item}">Item par défaut</li>
        </ul>
        <!-- Alternative si la liste est vide -->
         <p th:if="${#lists.isEmpty(itemList)}">Aucun item à afficher.</p>
    </body>
    </html>
```

---

## Exercice 4 : Itérer sur une liste d'objets dans un tableau

*   **Objectif :** Afficher une liste d'objets complexes dans un tableau HTML.
*   **Concepts Thymeleaf :** `th:each`, accès aux propriétés dans une boucle.

*   **Classe Pojo (`Product.java`) :**
    ```java
    // Créez cette classe simple
    public class Product {
        private String name;
        private double price;
        private int stock;

        // Constructeur, Getters et Setters
        public Product(String name, double price, int stock) {
            this.name = name;
            this.price = price;
            this.stock = stock;
        }
        // --- Getters ---
        public String getName() { return name; }
        public double getPrice() { return price; }
        public int getStock() { return stock; }
        // --- Setters (optionnels pour cet exo) ---
    }
    ```

*   **Contrôleur (`ProductController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import java.util.Arrays;
    import java.util.List;
    // Assurez-vous d'importer ou de définir la classe Product ci-dessus

    @Controller
    public class ProductController {

        @GetMapping("/products")
        public String showProducts(Model model) {
            List<Product> products = Arrays.asList(
                new Product("Ordinateur portable", 1200.50, 15),
                new Product("Souris sans fil", 25.99, 50),
                new Product("Clavier mécanique", 89.90, 30)
            );
            model.addAttribute("productList", products);
            return "products-view"; // Nom du template HTML
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/products-view.html`) :**
    *   Crée un tableau HTML (`<table>`) avec un en-tête (Nom, Prix, Stock).
    *   Pour chaque produit dans `productList`, ajoute une ligne (`<tr>`) avec les cellules (`<td>`) correspondantes affichant le nom, le prix et le stock.

### Correction (`products-view.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Liste des Produits</title>
        <style>
            table, th, td { border: 1px solid black; border-collapse: collapse; padding: 5px;}
            th { background-color: #f2f2f2; }
        </style>
    </head>
    <body>
        <h1>Liste des Produits</h1>
        <table th:if="${not #lists.isEmpty(productList)}">
            <thead>
                <tr>
                    <th>Nom</th>
                    <th>Prix (€)</th>
                    <th>Stock</th>
                </tr>
            </thead>
            <tbody>
                <tr th:each="product : ${productList}">
                    <td th:text="${product.name}">Nom produit</td>
                    <td th:text="${#numbers.formatDecimal(product.price, 1, 2, 'COMMA')}">0.00</td>
                    <td th:text="${product.stock}">0</td>
                </tr>
            </tbody>
        </table>
        <p th:if="${#lists.isEmpty(productList)}">Aucun produit à afficher.</p>
    </body>
    </html>
```
    
*Note :* Utilisation de `#numbers.formatDecimal` pour un formatage simple du prix.

---

## Exercice 5 : Affichage conditionnel avec `th:if`

*   **Objectif :** Afficher un élément HTML uniquement si une condition est vraie.
*   **Concepts Thymeleaf :** `th:if`

*   **Contrôleur (`ConditionalController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestParam;
    // Assurez-vous d'importer ou de définir la classe User (Exercice 2)

    @Controller
    public class ConditionalController {

        @GetMapping("/profile")
        public String showProfile(Model model, @RequestParam(required = false) boolean admin) {
            User user = new User(2L, "adminUser", "admin@test.com", true);
            model.addAttribute("userProfile", user);
            model.addAttribute("isAdmin", admin); // Ajoute le flag admin
            return "profile-view"; // Nom du template HTML
        }
    }
    ```
    *Accédez via `/profile?admin=true` ou `/profile` (admin sera false).*

*   **Template Thymeleaf à créer (`src/main/resources/templates/profile-view.html`) :**
    *   Affiche les informations de base de `userProfile`.
    *   Ajoute un bouton ou un lien "Panneau d'administration" qui ne s'affiche *que si* l'attribut `isAdmin` est vrai (`true`).

### Correction (`profile-view.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Profil Utilisateur</title>
    </head>
    <body>
        <h1>Profil Utilisateur</h1>
        <p>Nom d'utilisateur: <span th:text="${userProfile.username}">username</span></p>
        <p>Email: <span th:text="${userProfile.email}">email@example.com</span></p>

        <!-- Section conditionnelle -->
        <div th:if="${isAdmin}">
            <hr>
            <h2>Section Admin</h2>
            <p>Vous avez les droits administrateur.</p>
            <button>Accéder au Panneau d'administration</button>
        </div>

        <p th:unless="${isAdmin}">Vous n'êtes pas administrateur.</p>

    </body>
    </html>
```

*Note :* Ajout de `th:unless` pour montrer l'alternative.

---

## Exercice 6 : Utilisation des URL avec `th:href` et `@{...}`

*   **Objectif :** Créer des liens dynamiques en utilisant les expressions d'URL Thymeleaf.
*   **Concepts Thymeleaf :** `th:href`, syntaxe `@{...}`, variables de chemin.

*   **Contrôleur (`LinkController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.PathVariable; // Important
    import java.util.Arrays;
    import java.util.List;
    // Assurez-vous d'importer ou de définir la classe Product (Exercice 4)

    @Controller
    public class LinkController {

        @GetMapping("/product-list")
        public String showProductListLinks(Model model) {
             List<Product> products = Arrays.asList(
                new Product("P101", 10.0, 5), // Utilisons des IDs simples comme nom pour cet exemple
                new Product("P102", 20.0, 10),
                new Product("P103", 30.0, 15)
            );
            model.addAttribute("products", products);
            return "product-list-links"; // Nom du template HTML
        }

        // Contrôleur pour la page de détail (juste pour que le lien fonctionne)
        @GetMapping("/product/{productId}")
        public String showProductDetail(@PathVariable String productId, Model model) {
            model.addAttribute("productId", productId);
            // Dans une vraie appli, on chercherait le produit par ID
            return "product-detail-placeholder"; // Vue simple pour montrer que ça marche
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/product-list-links.html`) :**
    *   Affiche une liste de produits.
    *   Pour chaque produit, crée un lien (`<a>`) qui pointe vers `/product/{productId}` où `{productId}` est le nom (utilisé comme ID ici) du produit.
    *   Crée également un lien statique vers la page d'accueil (`/`).

### Correction (`product-list-links.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Liens Produits</title>
    </head>
    <body>
        <h1>Liste des Produits (avec liens)</h1>
        <ul>
            <li th:each="prod : ${products}">
                <a th:href="@{/product/{id}(id=${prod.name})}" th:text="${'Détails du produit ' + prod.name}">
                    Lien Produit
                </a>
                 <!-- Alternative: Concaténation directe -->
                 <!-- <a th:href="@{'/product/' + ${prod.name}}" th:text="${'Détails du produit ' + prod.name}">Lien Produit</a> -->
            </li>
        </ul>
        <hr>
        <a th:href="@{/}">Retour à l'accueil</a> <br/>
        <a th:href="@{/items}">Voir la liste des items (Exo 3)</a>
    </body>
    </html>
```
*   **Template pour la page de détail (`src/main/resources/templates/product-detail-placeholder.html`) :**
    
```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head><meta charset="UTF-8"><title>Détail Produit</title></head>
    <body>
        <h1>Détail du Produit</h1>
        <p>Affichage des détails pour le produit ID: <strong th:text="${productId}">ID</strong></p>
        <a th:href="@{/product-list}">Retour à la liste</a>
    </body>
    </html>
```

---

## Exercice 7 : Formulaire simple avec `th:object` et `th:field`

*   **Objectif :** Créer un formulaire HTML lié à un objet Java pour la soumission de données.
*   **Concepts Thymeleaf :** `th:object`, `th:field`, `th:action`, `th:method`

*   **Classe Pojo (`MessageForm.java`) :**
    ```java
    // Un objet simple pour contenir les données du formulaire
    public class MessageForm {
        private String sender;
        private String content;

        // Getters et Setters OBLIGATOIRES pour la liaison de formulaire
        public String getSender() { return sender; }
        public void setSender(String sender) { this.sender = sender; }
        public String getContent() { return content; }
        public void setContent(String content) { this.content = content; }
    }
    ```

*   **Contrôleur (`FormController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.ModelAttribute;
    import org.springframework.web.bind.annotation.PostMapping;
    // Assurez-vous d'importer ou de définir MessageForm

    @Controller
    public class FormController {

        // Afficher le formulaire vide
        @GetMapping("/contact")
        public String showContactForm(Model model) {
            model.addAttribute("messageForm", new MessageForm()); // Objet vide pour le formulaire
            return "contact-form"; // Nom du template HTML
        }

        // Traiter la soumission du formulaire
        @PostMapping("/submit-contact")
        public String submitContactForm(@ModelAttribute MessageForm messageForm, Model model) {
            // Ici, on traiterait les données (ex: sauvegarde en BDD)
            System.out.println("Expéditeur: " + messageForm.getSender());
            System.out.println("Message: " + messageForm.getContent());

            model.addAttribute("submittedSender", messageForm.getSender());
            model.addAttribute("submittedContent", messageForm.getContent());
            return "contact-success"; // Page de confirmation
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/contact-form.html`) :**
    *   Crée un formulaire (`<form>`) qui soumet les données à `/submit-contact` via POST.
    *   Lie le formulaire à l'objet `messageForm` en utilisant `th:object`.
    *   Crée des champs de saisie (`<input type="text">`, `<textarea>`) pour `sender` et `content` en utilisant `th:field`.
    *   Ajoute un bouton de soumission (`<button type="submit">`).

### Correction (`contact-form.html`) {collapsible="true"}

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Formulaire de Contact</title>
    </head>
    <body>
        <h1>Contactez-nous</h1>
        <form th:action="@{/submit-contact}" th:object="${messageForm}" method="post">
            <div>
                <label for="senderInput">Votre Nom:</label>
                <input type="text" id="senderInput" th:field="*{sender}" />
                 <!-- th:field génère id="sender", name="sender" et value="..." -->
            </div>
            <div>
                <label for="contentInput">Votre Message:</label>
                <textarea id="contentInput" th:field="*{content}" rows="4" cols="50"></textarea>
                 <!-- th:field génère id="content", name="content" et remplit le contenu -->
            </div>
            <div>
                <button type="submit">Envoyer</button>
            </div>
        </form>
    </body>
    </html>
```
  
* **Template pour la page de succès (`src/main/resources/templates/contact-success.html`) :**

```html
     <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head><meta charset="UTF-8"><title>Message Envoyé</title></head>
    <body>
        <h1>Message Envoyé !</h1>
        <p>Merci <strong th:text="${submittedSender}">Expéditeur</strong>,</p>
        <p>Votre message suivant a été reçu :</p>
        <blockquote th:text="${submittedContent}">Contenu du message</blockquote>
        <a th:href="@{/contact}">Envoyer un autre message</a>
    </body>
    </html>
```

---

## Exercice 8 : Utilisation des fragments Thymeleaf (`th:insert` ou `th:replace`)

*   **Objectif :** Créer des parties de page réutilisables (comme un header ou un footer) et les inclure dans d'autres templates.
*   **Concepts Thymeleaf :** `th:fragment`, `th:insert`, `th:replace`

*   **Contrôleur (`LayoutController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.GetMapping;

    @Controller
    public class LayoutController {

        @GetMapping("/home")
        public String homePage() {
            return "home-page"; // Template principal
        }

        @GetMapping("/about")
        public String aboutPage() {
            return "about-page"; // Autre template utilisant le layout
        }
    }
    ```

*   **Template Fragment à créer (`src/main/resources/templates/fragments/layout.html`) :**
    *   Crée un fichier contenant des fragments réutilisables.
    *   Définit un fragment `header` (par exemple, avec un `<h1>` et une navigation simple).
    *   Définit un fragment `footer` (par exemple, avec un copyright).

*   **Template Principal à créer (`src/main/resources/templates/home-page.html`) :**
    *   Inclut le fragment `header` de `fragments/layout.html`.
    *   Ajoute du contenu spécifique à la page d'accueil.
    *   Inclut le fragment `footer` de `fragments/layout.html`.


### Correction fragments {collapsible="true"}

**`fragments/layout.html`**

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
       <!-- Ce head peut aussi être un fragment si besoin -->
    </head>
    <body>

        <!-- Fragment Header -->
        <header th:fragment="header">
            <h1>Mon Super Site</h1>
            <nav>
                <a th:href="@{/home}">Accueil</a> |
                <a th:href="@{/about}">À Propos</a> |
                <a th:href="@{/products}">Produits (Exo 4)</a> |
                 <a th:href="@{/contact}">Contact (Exo 7)</a>
            </nav>
            <hr>
        </header>

        <!-- Fragment Footer -->
        <footer th:fragment="footer">
            <hr>
            <p>&copy; 2023 Mon Entreprise. Tous droits réservés.</p>
        </footer>

    </body>
    </html>
```

`home-page.html`
    
```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Accueil</title>
    </head>
    <body>
        <!-- Insérer le header ici -->
        <div th:insert="~{fragments/layout :: header}"></div>
        <!-- Alternative: th:replace remplace la div elle-même -->
        <!-- <header th:replace="~{fragments/layout :: header}"></header> -->

        <main>
            <h2>Bienvenue sur la page d'accueil !</h2>
            <p>Ceci est le contenu spécifique de la page d'accueil.</p>
        </main>

        <!-- Insérer le footer ici -->
        <div th:insert="~{fragments/layout :: footer}"></div>
        <!-- Alternative: th:replace remplace la div elle-même -->
        <!-- <footer th:replace="~{fragments/layout :: footer}"></footer> -->
    </body>
    </html>
```

**Template `about-page.html` (pour tester la réutilisabilité) :**

```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>À Propos</title>
    </head>
    <body>
        <div th:insert="~{fragments/layout :: header}"></div>

        <main>
            <h2>À Propos de Nous</h2>
            <p>Informations sur l'entreprise ou le site.</p>
        </main>

        <div th:insert="~{fragments/layout :: footer}"></div>
    </body>
    </html>
```

---

## Exercice 9 : Utilisation des objets utilitaires (`#numbers`, `#strings`, `#dates`)

*   **Objectif :** Utiliser les objets utilitaires intégrés de Thymeleaf pour formater des nombres, manipuler des chaînes ou formater des dates.
*   **Concepts Thymeleaf :** Objets utilitaires `#numbers`, `#strings`, `#dates`, `#calendars`

*   **Classe Pojo (`Event.java`) :**
    ```java
    import java.time.LocalDate;
    import java.util.Calendar;
    import java.util.Date;

    public class Event {
        private String name;
        private double ticketPrice;
        private String description;
        private LocalDate eventDate; // Utiliser java.time pour les dates modernes
        private Date legacyDate; // Pour montrer le formatage avec java.util.Date

        // Constructeur, Getters
        public Event(String name, double ticketPrice, String description, LocalDate eventDate) {
            this.name = name;
            this.ticketPrice = ticketPrice;
            this.description = description;
            this.eventDate = eventDate;
            // Créer une Date legacy à partir de LocalDate pour l'exemple
            this.legacyDate = java.sql.Date.valueOf(eventDate);
        }

        public String getName() { return name; }
        public double getTicketPrice() { return ticketPrice; }
        public String getDescription() { return description; }
        public LocalDate getEventDate() { return eventDate; }
        public Date getLegacyDate() { return legacyDate; }
    }
    ```

*   **Contrôleur (`UtilityController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import java.time.LocalDate;
    // Assurez-vous d'importer ou de définir Event

    @Controller
    public class UtilityController {

        @GetMapping("/event-details")
        public String showEventDetails(Model model) {
            Event event = new Event(
                "Concert Annuel",
                49.995, // Prix avec plus de décimales
                "   Un événement musical à ne pas manquer !   ", // Avec espaces
                LocalDate.of(2024, 10, 26)
            );
            model.addAttribute("event", event);
            return "event-details-view"; // Nom du template HTML
        }
    }
    ```

*   **Template Thymeleaf à créer (`src/main/resources/templates/event-details-view.html`) :**
    *   Affiche le nom de l'événement.
    *   Affiche le prix du billet formaté en devise (ex: €49.99) en utilisant `#numbers`.
    *   Affiche la description sans les espaces au début et à la fin, et en majuscules, en utilisant `#strings`.
    *   Affiche la date de l'événement formatée (ex: 26 octobre 2024) en utilisant `#dates` ou `#temporals` (pour `java.time`).

### Correction (`event-details-view.html`) {collapsible="true"}

    
```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Détails Événement</title>
    </head>
    <body>
        <h1>Détails de l'Événement</h1>
        <p>Nom: <strong th:text="${event.name}">Nom Événement</strong></p>

        <h2>Informations Utiles</h2>
        <ul>
            <li>
                Prix: <span th:text="${#numbers.formatCurrency(event.ticketPrice)}">0.00 €</span>
                 <!-- Par défaut, utilise Locale système. Peut forcer: #numbers.formatCurrency(event.ticketPrice, 'EUR') -->
                 <!-- Autre formatage: <span th:text="${#numbers.formatDecimal(event.ticketPrice, 1, 2)}">0.00</span> -->
            </li>
            <li>
                Description (nettoyée et majuscules):
                <strong th:text="${#strings.toUpperCase(#strings.trim(event.description))}">DESCRIPTION</strong>
                 <!-- Vérifier si vide: <span th:if="${#strings.isEmpty(event.description)}">Pas de description</span> -->
                 <!-- Tronquer: <span th:text="${#strings.abbreviate(event.description, 20)}">Desc...</span> -->
            </li>
             <li>
                Date (avec #temporals pour java.time):
                <span th:text="${#temporals.format(event.eventDate, 'dd MMMM yyyy', #locale.forLanguageTag('fr'))}">jj mois aaaa</span>
                <!-- Autres formats: 'dd/MM/yyyy', 'EEEE dd MMMM yyyy' (Jour de la semaine complet) -->
            </li>
             <li>
                Date (avec #dates pour java.util.Date):
                <span th:text="${#dates.format(event.legacyDate, 'dd-MMM-yyyy')}">jj-Mon-aaaa</span>
            </li>
        </ul>
    </body>
    </html>
```

---

## Exercice 10 : Affichage conditionnel avec `th:switch` / `th:case`

*   **Objectif :** Afficher différents contenus en fonction de la valeur d'une variable, similaire à un `switch` en Java.
*   **Concepts Thymeleaf :** `th:switch`, `th:case`

*   **Classe Enum (`UserRole.java`) :**
    ```java
    public enum UserRole {
        ADMIN, EDITOR, GUEST
    }
    ```

*   **Contrôleur (`SwitchController.java`) :**
    ```java
    import org.springframework.stereotype.Controller;
    import org.springframework.ui.Model;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestParam;
    // Assurez-vous d'importer ou de définir UserRole

    @Controller
    public class SwitchController {

        @GetMapping("/dashboard")
        public String showDashboard(@RequestParam(defaultValue = "GUEST") String role, Model model) {
            UserRole userRole;
            try {
                userRole = UserRole.valueOf(role.toUpperCase());
            } catch (IllegalArgumentException e) {
                userRole = UserRole.GUEST; // Valeur par défaut si invalide
            }
            model.addAttribute("userRole", userRole);
            return "dashboard-view"; // Nom du template HTML
        }
    }
    ```
    *Accédez via `/dashboard?role=ADMIN`, `/dashboard?role=EDITOR` ou `/dashboard` (role sera GUEST).*

*   **Template Thymeleaf à créer (`src/main/resources/templates/dashboard-view.html`) :**
    *   Utilise `th:switch` sur la variable `userRole`.
    *   Utilise `th:case` pour afficher un message spécifique si le rôle est `ADMIN`.
    *   Utilise `th:case` pour afficher un message différent si le rôle est `EDITOR`.
    *   Utilise `th:case="*"` (cas par défaut) pour afficher un message si le rôle est autre (`GUEST` ou inconnu).

### Correction (`dashboard-view.html`) {collapsible="true"}
    
```html
    <!DOCTYPE html>
    <html xmlns:th="http://www.thymeleaf.org">
    <head>
        <meta charset="UTF-8">
        <title>Tableau de Bord</title>
    </head>
    <body>
        <h1>Tableau de Bord</h1>
        <p>Votre rôle : <strong th:text="${userRole}">ROLE</strong></p>

        <div th:switch="${userRole}">
            <div th:case="${T(com.example.demo.UserRole).ADMIN}"> <!-- Chemin complet vers l'Enum ou import -->
                <h2>Accès Administrateur</h2>
                <p>Vous avez un accès complet au système.</p>
                <button>Gérer les utilisateurs</button>
            </div>
            <div th:case="${T(com.example.demo.UserRole).EDITOR}">
                <h2>Accès Éditeur</h2>
                <p>Vous pouvez créer et modifier du contenu.</p>
                <button>Créer un nouvel article</button>
            </div>
            <div th:case="*"> <!-- Cas par défaut -->
                <h2>Accès Invité</h2>
                <p>Bienvenue ! Vous pouvez consulter le contenu public.</p>
            </div>
        </div>
        <hr>
         <p>Tester d'autres rôles :
             <a th:href="@{/dashboard(role='ADMIN')}">Admin</a> |
             <a th:href="@{/dashboard(role='EDITOR')}">Editor</a> |
             <a th:href="@{/dashboard(role='GUEST')}">Guest</a> |
             <a th:href="@{/dashboard(role='INVALID')}">Invalide</a>
         </p>
    </body>
    </html>
```
    
*Note :* `T(com.example.demo.UserRole).ADMIN` est nécessaire pour accéder aux constantes de l'énumération. 
Remplacez `com.example.demo` par le package réel de votre énumération `UserRole`.
